# Image（镜像）操作

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

镜像领域的特点：**没有独立的代理文件**（`api/http/proxy/factory/docker/` 下无 `images.go`），代理逻辑集中在 `transport.go` 的 `proxyImageRequest`（536-551 行），只做三件事——registry 凭据替换、prune 管理员限制、其余透传。镜像也不是 Portainer RC 管理的资源类型。复杂的镜像业务（拉取编排、registry 匹配、新旧状态检测）在 `api/docker/images/` 服务层。

---

## 1. 操作 → 代码映射总表

### 1.1 代理路径（`api/http/proxy/factory/docker/transport.go`）

| Docker API | 拦截处理 | 行号 |
|---|---|---|
| `POST /images/create?fromImage=...`（pull） | `replaceRegistryAuthenticationHeader`（凭据替换） | 540-541 → 553-559 |
| `POST /images/{name}/push` | 同上 | 545-546 → 553-559 |
| `POST /images/prune` | `administratorOperation`（仅管理员） | 542-543 |
| 其余（`GET /images/json` 列表、`GET /images/{name}/json` inspect、`GET /images/{name}/history`、`POST /images/{name}/tag`、`DELETE /images/{name}`、`GET /images/get` 导出等） | **直接透传**（无 RC 装饰） | 549-550 |

`replaceRegistryAuthenticationHeader`（553-559 行）= `decorateRegistryAuthenticationHeader`（改写认证头）+ `decorateGenericResourceCreationOperation`（对拉取的流式响应实际不产生装饰效果，因为响应非 201 JSON）。

### 1.2 原生 handler 路径（`api/http/handler/docker/images/`）

| 端点 | 处理函数 | 文件:行号 | 说明 |
|---|---|---|---|
| `GET /docker/{id}/images` | `imagesList` | `images_list.go:45-103` | 唯一原生端点：聚合节点名与使用状态（见 §5） |
| 路由注册 | `NewHandler` | `handler.go:21-33` | `AuthenticatedAccess` + `CheckEndpointAuthorization` |

### 1.3 服务层（`api/docker/images/`）

| 文件 | 职责 | 关键函数 |
|---|---|---|
| `puller.go` | 服务端 SDK 拉取 | `Puller.Pull`（29-51） |
| `registry.go` | registry 匹配与凭据 | `EncodedRegistryAuth`（64-88）、`findBestMatchRegistry`（103-143） |
| `image.go` | 镜像名解析模型 | `ParseImage`（93-127）、`hubLink`（129-172） |
| `status.go` | 镜像新旧（outdated）检测 | `ContainerImageStatus`（116）、`ServiceImageStatus`（168） |
| `digest.go` | 远端 digest 查询 | `(*DigestClient).RemoteDigest`（44-92） |

### 1.4 代理层 build（`api/http/proxy/factory/docker/build.go`，129 行）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `buildOperation` | 37-129 | 把 `/build` 请求体统一改写成 tar 归档（见 §4） |
| `postDockerfileRequest` | 19-21 | JSON 提交 Dockerfile 内容的 body 结构 |

---

## 2. Registry 凭据注入链（pull / push 的核心）

**设计动机**：前端不掌握 registry 密码，只传"我想用哪个 registry"的引用；服务端把引用换成存储的真实凭据，并顺便校验用户对该 registry 的访问权——**防止用户借代理头读到无权 registry 的凭据**。

### 2.1 三层调用链

```
proxyImageRequest (transport.go:536)
  └─ replaceRegistryAuthenticationHeader (553-559)
       ├─ decorateRegistryAuthenticationHeader (transport.go:561-610)
       │    ① 读 X-Registry-Auth 头 —— 前端传的是 base64 的 {"registryId": N}
       │    ② base64 解码（573）
       │    ③ 空 JSON {} → 删除头（586-590，兼容匿名拉取的旧行为）
       │    ④ 无 registryId → 不动（593-595）
       │    ⑤ 否则调 createRegistryAuthenticationHeader 生成真实凭据
       │       重新 base64（URL 编码变体）写回头（597-607）
       └─ decorateGenericResourceCreationOperation
```

### 2.2 真实凭据构造（`api/http/proxy/factory/docker/registry.go`）

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `portainerRegistryAuthenticationHeader` | 25-28 | 前端传的头结构（仅 registryId） |
| `createRegistryAuthenticationHeader` | 30-67 | registryID==0 → 匿名 DockerHub（35-38）；否则在 `accessContext.registries` 中找 ID 匹配且 `security.AuthorizedRegistryAccess` 有权的 registry（44-52）；`registryutils.PrepareRegistryCredentials` 解密存储的密码并确保 token 有效（58）；返回 `{username, password, serveraddress}` |

### 2.3 拉取进度如何返回前端

`POST /images/create` 的响应是 Docker daemon 的 **chunked JSON 进度流**（每行 `{"status":...,"id":...,"progressDetail":...}`）。Portainer 代理**不做任何缓冲或改写，直接透传**（`executeDockerRequest` → `HTTPTransport.RoundTrip`），前端 axios 流式读取逐行解析。

> **不使用 hijack 也不使用 websocket**——这两者仅用于容器 exec/attach 交互式控制台（见 container.md §5）。

---

## 3. 服务端 SDK 拉取与 registry 匹配（`api/docker/images/`）

### 3.1 拉取器（`puller.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `Puller.Pull` | 29-51 | ① `registryClient.EncodedRegistryAuth(img)` 解析镜像所属 registry 并取 base64 认证头（失败则匿名继续，32-38）；② `client.ImagePull(ctx, img.FullName(), image.PullOptions{RegistryAuth})`（40）；③ `io.ReadAll(out)` 把进度流读完——**Docker SDK 要求消费完输出才保证拉取完成**（48） |

调用方：容器重建（`api/docker/container.go:104-110`）、Swarm 栈部署等。

### 3.2 registry 匹配（`registry.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `registriesCache` | 16 | 5 分钟结果缓存 |
| `EncodedRegistryAuth` | 64-88 | 查缓存 → 未命中读库 `findBestMatchRegistry` → 生成认证头 |
| `EncodedCertainRegistryAuth` | 90-96 | 先 `EnsureRegTokenValid` 刷新 token 再 `GetRegistryAuthHeader` |
| `findBestMatchRegistry` | 103-143 | 匹配优先级：① DockerHub 且镜像路径以 `<username>/` 开头；② registry.URL 出现在镜像名中；③ 兜底第一个 DockerHub registry；`cmp.Or` 取第一个非空（135） |

### 3.3 镜像名解析模型（`image.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `ParseImage` | 93-127 | 用 `go.podman.io/image/v5/docker/reference` 的 `ParseNormalizedNamed`+`TagNameOnly` 规范化镜像名，拆出 Domain/Path/Tag/Digest |
| `hubLink` | 129-172 | 按 domain 生成 Docker Hub/GHCR/Quay/GitLab 等的外链（前端跳转用） |
| `IsLocalImage` / `IsDanglingImage` / `IsNoTagImage` | 175-189 | 无 RepoDigests 即本地构建 / 无 tag 判定 |

---

## 4. 镜像构建（`/build`）

### 4.1 代理入口（transport.go）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `proxyBuildRequest` | 491-501 | `/build/prune` 仅管理员（492-494）；Git remote 构建先 `updateDefaultGitBranch`（见 docker-sdk.md §3.4）；最后 `interceptAndRewriteRequest(request, buildOperation)` 先改写再透传 |
| `updateDefaultGitBranch` | 503-534 | remote 以 `.git` 结尾时 `ssrf.CheckURL` 防护 → gitService 取 `LatestCommitID` → 改写 remote 为 `repo.git#<commit>` —— 固定 commit 保证可复现构建并防 SSRF |

### 4.2 请求体归一化（`build.go:37-129` 的 `buildOperation`）

按 `Content-Type` 分三种分支，统一产出 `application/x-tar`：

| Content-Type | 场景 | 处理 | 行号 |
|---|---|---|---|
| 空 | 前端上传无扩展名 Dockerfile | 读 body 原文 → `archive.TarFileInBuffer(body, "Dockerfile", 0600)` 打成单文件 tar | 51-60 |
| `application/json` | Dockerfile 内容以 JSON `{content}` 提交 | 解析 `postDockerfileRequest` 后同样打 tar | 62-72 |
| `multipart/form-data` | Dockerfile + 附加文件（上限 32MB） | 遍历 `MultipartForm.File` 逐个放入 tar；名为 `blob` 的文件重命名为 `Dockerfile`；清空 Form 防重复消费 | 75-113 |
| 其他 | — | 不改写 | 120-121 |

build 参数（`t`/`dockerfile`/`remote`/`buildargs`/`nocache` 等）全部经查询字符串透传。**输出流**：daemon 返回 chunked JSON 构建日志流，代理不改写直接透传，前端 `jsonObjectsToArrayHandler`（`app/react/docker/images/queries/useBuildImageMutation.ts:117`）按行拆数组渲染。

四种构建方式（上传/URL/Dockerfile 内容/内容+文件）的前端定义：`app/react/docker/images/queries/useBuildImageMutation.ts:8-69`（注释映射）与 126-288。

---

## 5. 镜像列表原生 handler（`api/http/handler/docker/images/images_list.go:45-103`）

此端点绕过代理直连 Docker/Agent，意义在于**聚合节点信息与使用状态**——代理透传的 `/images/json` 做不到这两点：

```
① utils.GetClient 建客户端（46）
② 构造 nodeNames map 通过 NodeNamesCtxKey{} 塞进 context（51-54）
   → 传输层 NodeNameTransport.RoundTrip（client.go:138-186）在响应中
     提取 Agent 附加的 Portainer.Agent.NodeName 填充该 map
③ cli.ImageList(ctx, image.ListOptions{})（56）
④ withUsage=true 时 cli.ContainerList(All:true) 收集 imageUsageSet（66-78）
   —— 标记每个镜像是否被容器使用
⑤ 对无 RepoTags 但有 RepoDigests 的镜像补 "repo:<none>" 伪 tag（82-86）
⑥ NodeName 按 "{imageID}-{index}" 从 map 取回（94，与传输层写入端约定一致）
⑦ 返回自定义 ImageResponse{created,nodeName,id,size,tags,used}（结构 20-30）
```

---

## 6. 镜像新旧状态检测（outdated，`api/docker/images/status.go`）

支持 UI 上"镜像有新版本"提示：

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `statusCache` | 38 | 24 小时结果缓存 |
| `ContainersImageStatus` | 46 | errgroup 并发（限 8）对多容器求聚合状态 |
| `ContainerImageStatus` | 116 | `ContainerInspect` 提取 `sha256:` imageID → `ImageInspectWithRaw` 收集 RepoDigests/RepoTags → `checkStatus`（207-265）比对本地与远端 digest：命中即 `Updated`，否则 `Outdated`；无 digest 则 `Skipped`。结果写缓存 |
| `ServiceImageStatus` | 168 | 按 `com.docker.swarm.service.id` 标签 `ContainerList` 找服务容器；存在 `created` 状态容器（Swarm 正在替换任务）返回 `Preparing`（189-196） |
| `EvictImageStatus` | 291 | 供容器重建/服务强更后清缓存（按容器 ID、compose 标签、swarm service 标签三个 key） |
| `(*DigestClient).RemoteDigest` | digest.go:44-92 | `go.podman.io/image/v5/docker.GetDigest` 发 HEAD 请求取远端 digest；5 秒超时（19），带 registry 凭据（61-73），DockerHub 失败回退匿名（79-84） |

---

## 7. 镜像导入/导出

| 操作 | 实现方式 | 位置 |
|---|---|---|
| **导出**（`docker save`） | `GET /api/endpoints/{id}/docker/images/get?names=...` 代理透传 tar 流；前端 `responseType: 'blob'` 下载，从 `Content-Disposition` 提取文件名；支持 `X-PortainerAgent-Target` 指定节点 | 前端 `app/react/docker/images/queries/useExportImageMutation.ts:20-58`（`getImagesNamesForDownload` 60-71 取第一个 tag，`<none>` 时退回 image ID） |
| **导入**（`docker load`） | **未实现**——前端无 `images/load` 调用（全仓搜索无结果），代理层对 `POST /images/load` 也无专门处理（`images` 前缀 default 分支透传） | — |
| 容器文件打包 | `GET/PUT /containers/{id}/archive` 走容器通配 → `restrictedResourceOperation` 校验后透传 | transport.go:297 |

---

## 8. 与其他领域的协作

- **Container**：`Puller.Pull` 被容器重建调用（[container.md](./container.md) §4.2 步骤①）；`EvictImageStatus` 被重建/强更调用清理缓存。
- **Compose**：`ComposeStackManager.Up` 的 forcePullImage 先走 `deployer.Pull`（compose.md）；Swarm 服务强更（`webhook_execute.go:119-125`）可选预拉镜像。
- **NodeNameTransport**：镜像列表专用的传输层增强（docker-sdk.md §4）。
