# Docker SDK 接入层

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

Portainer 后端通过 **Docker 官方 Go SDK**（`github.com/docker/docker/client`）与 Docker Daemon 交互。本文档覆盖：SDK 依赖、客户端工厂（连接管理）、SSRF 防护、代理拦截层（所有 Docker API 请求的公共通道）、快照服务、以及 handler 直连 SDK 的标准模式。

---

## 1. SDK 依赖清单（`go.mod`）

### 直接依赖

| 依赖 | 版本 | go.mod 行号 | 用途 |
|---|---|---|---|
| `github.com/docker/docker` | v28.5.2+incompatible | 24 | Docker Go SDK 核心（`client`、`api/types`） |
| `github.com/docker/cli` | v28.5.1+incompatible | 22 | Docker CLI 库（compose 部署的 `command.DockerCli`、`docker stack deploy` 复刻） |
| `github.com/docker/compose/v2` | v2.40.3 | 23 | Compose v2 部署库（进程内 API，见 compose.md） |
| `github.com/compose-spec/compose-go/v2` | v2.9.1 | 18 | Compose 文件解析/校验（见 compose.md） |

### 间接依赖（Docker 生态，`// indirect`）

| 依赖 | go.mod 行号 | 用途 |
|---|---|---|
| `docker/go-connections` | 146 | socket/TLS 连接工具 |
| `docker/go-units` | 149 | 存储单位换算 |
| `docker/docker-credential-helpers` | 145 | registry 凭据存储 |
| `docker/distribution` | 144 | registry 协议（digest 查询） |
| `novln/docker-parser` | 273 | 镜像名分层解析 |
| `moby/docker-image-spec` | 250 | 镜像规范类型 |

---

## 2. 客户端工厂（`api/docker/client/client.go`，212 行）

本版本中 `ClientFactory` 是**具体 struct 而非接口**，`CreateClient` 返回 SDK 的具体类型 `*client.Client`。单例在 `api/cmd/portainer/main.go:469` 创建，注入到代理工厂、各 handler、快照服务、升级服务。

### 2.1 类型与函数映射表

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `ClientFactory` struct | 33-36 | 持有 `signatureService`（Agent 通信签名）与 `reverseTunnelService`（Edge 反向隧道）两个依赖 |
| `NewClientFactory` | 39-44 | 简单构造器 |
| `CreateClient` | 50-72 | 入口分发函数，按环境类型与 URL 协议路由到 4 个创建函数（见 2.2） |
| `createLocalClient` | 74-79 | 本地 unix socket / Windows npipe 连接 |
| `createTCPClient` | 81-98 | 远程 TCP 连接（可叠加强制 https） |
| `createAgentClient` | 100-132 | Agent / Edge Agent 连接（注入签名头） |
| `NodeNameTransport` struct | 134-136 | 包装 `http.Transport` 的自定义 RoundTripper |
| `NodeNameTransport.RoundTrip` | 138-186 | 拦截 Agent 聚合的 `/images/json` 响应，提取节点名（见第 4 节） |
| `httpClient` | 188-212 | 构造带 SSRF 防护的底层 `*http.Client`（见第 5 节） |
| 常量 `defaultDockerRequestTimeout` | 26 | 默认超时 60s |
| 常量 `dockerClientVersion` | 27 | `"1.37"`，**已闲置**——三条创建路径均改用 `WithAPIVersionNegotiation()` 动态协商 |

### 2.2 CreateClient 分发逻辑（50-72 行）

```
CreateClient(endpoint, nodeName, timeout)
├─ AzureEnvironment            → 返回 errUnsupportedEnvironmentType（23 行定义）
├─ AgentOnDockerEnvironment    → createAgentClient(endpoint, endpoint.URL, ...)
├─ EdgeAgentOnDockerEnvironment
│    ├─ reverseTunnelService.TunnelAddr(endpoint)  // 取反向隧道本地地址
│    └─ createAgentClient(endpoint, "http://"+tunnelAddr, ...)
├─ URL 前缀 unix:// 或 npipe:// → createLocalClient(endpoint)
└─ 其余                          → createTCPClient(endpoint, timeout)
```

### 2.3 五种连接方式对比

| 连接方式 | 创建函数 | SDK 选项 | 特殊处理 |
|---|---|---|---|
| unix socket | `createLocalClient` | `WithHost` + `WithAPIVersionNegotiation` | 无自定义 http.Client（走 SDK 默认） |
| Windows 命名管道 | `createLocalClient` | 同上 | 同上 |
| TCP（明文） | `createTCPClient` | `WithHost` + `WithAPIVersionNegotiation` + `WithHTTPClient` | 自定义 http.Client 带 SSRF 防护 |
| TCP + TLS | `createTCPClient` | 同上 + `WithScheme("https")` | `endpoint.TLSConfig.TLS` 为真时强制 https |
| Agent / Edge Agent | `createAgentClient` | 同 TCP + `WithHTTPHeaders` | 注入签名三头（见下） |

**Agent 签名头注入**（`createAgentClient`，111-118 行）：

```go
headers := map[string]string{
    portainer.PortainerAgentPublicKeyHeader: signatureService.EncodedPublicKey(), // 公钥
    portainer.PortainerAgentSignatureHeader: signature,                          // 签名
}
if nodeName != "" {
    headers[portainer.PortainerAgentTargetHeader] = nodeName // Swarm 定向节点
}
```

签名由 `signatureService.CreateSignature(portainer.PortainerAgentSignatureMessage)` 生成。

---

## 3. 代理拦截层（`api/http/proxy/`）

代理层是路径①（见 README 架构图）的实现，所有 `/api/endpoints/{id}/docker/**` 请求经它拦截、改写后透传给 Daemon。

### 3.1 文件职责映射

| 文件 | 行数 | 职责 |
|---|---|---|
| `api/http/proxy/manager.go` | 92 | 代理实例生命周期与缓存 |
| `api/http/proxy/factory/factory.go` | 61 | 总工厂：按环境类型分流 docker/kubernetes/azure |
| `api/http/proxy/factory/docker.go` | 130 | Docker 代理构建（本地 socket vs HTTP 反代） |
| `api/http/proxy/factory/docker.go`（`docker_unix.go` / `docker_windows.go`） | — | 平台特定 socket transport |
| `api/http/proxy/factory/reverse_proxy.go` | — | 单宿主反向代理 + 请求头白名单剥离（注意：在 factory 目录，不在 docker 子目录） |
| `api/http/proxy/factory/docker/transport.go` | 979 | **核心拦截器**（RBAC / RC / 改写分发） |
| `api/http/handler/endpointproxy/proxy_docker.go` | — | HTTP 入口（权限校验 + Edge 隧道 + 取缓存代理） |

### 3.2 代理生命周期（`manager.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `Manager` struct | 20-24 | 内含 `endpointProxies sync.Map` 按 endpointID 缓存代理 |
| `CreateAndRegisterEndpointProxy` | 39-52 | 调工厂建代理并写入缓存 |
| `GetEndpointProxy` | 65-72 | 取缓存代理 |
| `DeleteEndpointProxy` | 77-83 | 删代理并清理 k8s 客户端缓存 |

### 3.3 代理构建（`factory/docker.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `newDockerProxy` | 19-25 | URL 前缀分流：unix/npipe → 本地代理；否则 HTTP 反代 |
| `newDockerLocalProxy` | 27-34 | 构造本地 socket 代理（Linux `docker_unix.go:15`，Windows 对应 npipe） |
| `newDockerHTTPProxy` | 36-89 | Edge 环境先把 URL 替换为反向隧道地址；构造 `docker.TransportParameters`（53-59 行）；按 TLS/Edge 选择 `ssrf.NewInternalTransport` 或 `ssrf.NewTransport` 作内层传输；最后挂到 `NewSingleHostReverseProxyWithHostHeader` |
| `dockerLocalProxy.ServeHTTP` | 97-130 | 强制改写 `r.URL.Scheme/Host` 为 `http://unixsocket`，调 `transport.ProxyDockerRequest(r)` 后手工回写响应 |

`api/http/proxy/factory/reverse_proxy.go` 的 `createRewriteFn`（53-78 行）按 `allowedHeaders` 白名单（14-38 行）**剥离非白名单请求头**，防止客户端伪造内部头。

### 3.4 核心拦截器（`transport.go`）

| 函数/变量 | 行号 | 逻辑说明 |
|---|---|---|
| `Transport` struct | 39-50 | 自定义 `http.RoundTripper`，持有内层 `HTTPTransport`、endpoint、dataStore、签名/隧道服务、`dockerClientFactory`、snapshotService、dockerID 缓存 |
| `RoundTrip` | 94-96 | 委托 `ProxyDockerRequest` |
| `prefixProxyFuncMap` | 98-112 | URL 首段 → 处理函数路由表：build/configs/containers/exec/images/networks/nodes/secrets/services/swarm/tasks/v2(agent)/volumes |
| `adminOnlyRoutes` + `isAdminOnlyRoute` | 119-134 | plugins 相关 8 类操作仅管理员 |
| `ProxyDockerRequest` | 138-169 | **总入口**：① 正则剥 `/v1.xx` 版本前缀；② Agent/Edge 环境注入签名头（144-152）；③ 取首段查路由表分发；④ 未命中查 adminOnly；⑤ 兜底 `executeDockerRequest` 透传 |
| `executeDockerRequest` | 171-183 | 执行内层 RoundTrip；Edge 环境成功后 `UpdateLastActivity` 刷新隧道活跃时间 |
| `restrictedResourceOperation` | 612-690 | **RBAC 核心**（详见下） |
| `getDockerResourceUUID` | 692-739 | 用 SDK inspect 把资源"名字"换成真实 UUID 再查 RC（NetworkInspect/ContainerInspect/ServiceInspectWithRaw/ConfigInspectWithRaw/SecretInspectWithRaw） |
| `rewriteOperationWithLabelFiltering` | 744-761 | 执行请求 + 200 时对响应做列表改写 + `BlackListedLabels` 标签过滤 |
| `rewriteOperation` | 765-776 | 同上，无标签黑名单 |
| `interceptAndRewriteRequest` | 778-784 | 先改写请求体再透传（build 用） |
| `decorateGenericResourceCreationResponse` | 796-818 | 创建成功（201）后从响应抽资源 ID → 建私有 RC → 注入响应 `Portainer.ResourceControl` 字段 |
| `decorateGenericResourceCreationOperation` | 820-836 | 上述操作的入口封装 |
| `executeGenericResourceDeletionOperation` | 838-860 | 先做受限校验，删除成功（204/200）后同步删除 RC 记录 |
| `administratorOperation` | 877-888 | 非管理员直接 AccessDenied |
| `executeRequestAndRewriteResponse` | 862-873 | 执行请求，200 时调用回调改写响应 |
| `createOperationContext` | 929-961 | 从 JWT 取 tokenData、读全部 RC，组装 `restrictedDockerOperationContext`（isAdmin/userID/userTeamIDs/resourceControls） |
| `updateDefaultGitBranch` | 503-534 | `/build` 的 remote 指向 git 仓库时：SSRF 校验 → 查最新 commitID → 改写 remote 为 `url#commitID` 固定构建版本 |

**`restrictedResourceOperation`（612-690 行）鉴权流程**——所有单资源写操作的权限闸门：

```
① 管理员 → 直接放行
② 普通用户：
   a. 卷浏览器类操作 → 检查端点设置 AllowVolumeBrowserForRegularUsers（622-631 行）
   b. 读用户团队成员关系 + 全部 RC（633-643 行）
   c. 按 resourceID 直查 RC
   d. 查不到 → getDockerResourceUUID（692-739）用 SDK inspect 把名字换成 UUID 再查
   e. 仍查不到 → getInheritedResourceControlFromServiceOrStack
      （access_control.go:125-142，沿 service/stack 标签继承链查找）
   f. 任何一关不满足 → WriteAccessDeniedResponse（403）
```

### 3.5 访问控制引擎（`api/http/proxy/factory/docker/access_control.go`，348 行）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `newResourceControlFromPortainerLabels` | 42-108 | 支持 `io.portainer.accesscontrol.teams/users/public` 三个标签**即时创建** RC |
| `applyAccessControlOnResource` | 144-181 | 单对象：命中 RC 且有权 → 装饰返回；无 RC 且管理员 → 原样；否则 403 |
| `applyAccessControlOnResourceList` | 183-189 | 列表：管理员 → `decorateResourceList`（191-231，全量装饰）；普通用户 → `filterResourceList`（233-279，只保留可访问项） |
| `findResourceControl` | 281-321 | RC 查找优先级链（见 README「公共机制」） |
| `getStackResourceIDFromLabels` | 323-337 | 从 `com.docker.stack.namespace` / `com.docker.compose.project` 标签拼 Stack 资源 ID |
| `decorateObject` | 339-348 | 把 RC 写入响应 JSON 的 `Portainer.ResourceControl` 字段——**前端资源控制信息的来源** |

---

## 4. NodeNameTransport：Agent 聚合响应的节点名提取（134-186 行）

**问题背景**：Swarm 集群的 Agent 会聚合多节点数据返回镜像列表，但 Docker API 响应结构中没有"该镜像属于哪个节点"的字段，Agent 只能在每个镜像对象上附加 `Portainer.Agent.NodeName` 私有字段——该字段会污染传给前端的响应，需要在传输层摘出来。

**实现逻辑**（`RoundTrip`，138-186 行）：

1. 仅当响应为 200、ContentLength > 0 且请求路径以 `/images/json` 结尾时介入，否则直接透传；
2. 读取整个响应 body 并重新包装（`io.NopCloser(bytes.NewReader(body))`，160 行），保证下游仍可读；
3. 把 body 反序列化为匿名结构体切片（162-169 行）——内嵌 SDK 的 `image.Summary` + Agent 附加字段；
4. 从 request context 取出 `NodeNamesCtxKey{}`（30 行定义）对应的 map，以 `"镜像ID-下标"` 为 key 写入各镜像的节点名（175-183 行）；
5. handler 事后按相同 key 约定取回节点名（写入端见 `api/http/handler/docker/images/images_list.go:51-54`，读取端 94 行）。

> 用「镜像ID-下标」而非纯 ID 做 key：同一镜像可能出现在多个节点，仅凭 ID 无法区分。

---

## 5. SSRF 防护传输层（`pkg/libhttp/ssrf/builder.go`）

`httpClient`（`client.go:188-212`）为所有 TCP/Agent 连接构造底层传输：

| 场景 | 处理 |
|---|---|
| TLS 开启 | `crypto.CreateTLSConfigurationFromDisk` 从磁盘加载证书 → `ssrf.NewTransport(tlsConfig)` + `Protocols = ssrf.HTTP1Only()`（50 行）限制 HTTP/1 |
| TLS 关闭 | `ssrf.NewTransport(nil)` |
| 所有路径 | 统一包成 `NodeNameTransport`，超时默认 60s（`timeout` 参数可覆盖） |

SSRF transport 的作用：Portainer 允许用户自行配置环境 URL（即 Docker API 地址），该地址属于用户可控输入，直接请求会产生 SSRF 风险（如访问云元数据地址 169.254.169.254）。`ssrf.NewTransport` 在连接层拦截目标 IP/协议，限制为合法的 Docker 端点。

---

## 6. 快照服务（三层结构）

```
api/internal/snapshot/snapshot.go (调度与持久化)
        │ 调用
        ▼
api/docker/snapshot.go (门面，实现 portainer.DockerSnapshotter 接口)
        │ 委托
        ▼
pkg/snapshot/docker.go (SDK 采集器，全部直接调 SDK)
```

### 6.1 门面（`api/docker/snapshot.go`，31 行）

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `Snapshotter` struct | 11-13 | 仅持有 `clientFactory` |
| `NewSnapshotter` | 16-20 | 构造器；`api/cmd/portainer/main.go:227` 实例化，230 行注入 `snapshot.NewService` |
| `CreateSnapshot` | 23-31 | `CreateClient` 建客户端（defer 关闭）→ 委托 `pkg/snapshot.CreateDockerSnapshot(cli)` |

### 6.2 SDK 采集器（`pkg/snapshot/docker.go`，351 行）

| 函数 | 行号 | 使用的 SDK 调用 | 逻辑说明 |
|---|---|---|---|
| `CreateDockerSnapshot` | 29-75 | — | 先 `cli.Ping` 探活（失败整个快照失败）；随后依次采集 info → Swarm（nodes/services）→ containers → images → volumes → networks → version，**每步独立容错**（失败仅 log.Warn 不中断）；最后打时间戳 |
| `dockerSnapshotInfo` | 77-90 | `cli.Info` | 取 Swarm.ControlAvailable、DockerVersion、NCPU、MemTotal；原始 info 存 `SnapshotRaw.Info` |
| `dockerSnapshotNodes` | 92-110 | `cli.NodeList` | 聚合各节点 NanoCPUs/MemoryBytes 得集群总 CPU/内存与 NodeCount |
| `dockerSnapshotSwarmServices` | 112-132 | `cli.ServiceList` | 按 `com.docker.stack.namespace` 标签去重统计 StackCount |
| `dockerSnapshotContainers` | 134-229 | `cli.ContainerList(All:true)` + 逐容器 `cli.ContainerInspect` | 最重的一步（96 行）：统计 Compose 栈数；对每个 running 容器取 Env 与 GPU 信息（HostConfig.DeviceRequests 中 driver=nvidia 或 capabilities 含 gpu → GpuUseAll/GpuUseList）；Swarm 跨节点 inspect 失败降级跳过；调 `stats.CalculateContainerStats`；容器列表存 `SnapshotRaw.Containers` |
| `dockerSnapshotImages` | 231-241 | `cli.ImageList` | ImageCount + `SnapshotRaw.Images` |
| `dockerSnapshotVolumes` | 243-253 | `cli.VolumeList` | VolumeCount + `SnapshotRaw.Volumes` |
| `dockerSnapshotNetworks` | 255-264 | `cli.NetworkList` | `SnapshotRaw.Networks`（无计数字段） |
| `dockerSnapshotVersion` | 266-276 | `cli.ServerVersion` | `SnapshotRaw.Version`；`isPodman`（343-351）通过 Components 名字判断是否 Podman 兼容引擎 |
| `DockerSnapshotDiagnostics` | 279-305 | `cli.ContainerLogs`（Tail=5, stderr）+ stdcopy 解码 | Edge Agent 诊断专用：采集自身容器错误日志，配合 DNS/Telnet 探测 |

### 6.3 调度与持久化（`api/internal/snapshot/snapshot.go`，347 行）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `Service` struct | 21-28 | 聚合 dataStore、间隔 channel、Docker/K8s snapshotter、pendingActions |
| `SnapshotEndpoint` | 135-158 | Agent 环境先探测 Agent 版本；按类型分流 K8s/Docker |
| `snapshotDockerEndpoint` | 183-200 | 调 `CreateSnapshot` → `validateContainerEngineCompatibility`（202-210，校验 Docker/Podman 引擎选项匹配）→ 写库 |
| `startSnapshotLoop` + `snapshotEndpoints` | 212-264 | 定时循环；跳过 Edge/Azure（`SupportDirectSnapshot` 124 行——Edge 快照由 Agent 反向触发） |
| `updateEndpointStatus` | 266-301 | 快照后更新环境 Up/Down 状态并执行 pending actions |
| `FetchDockerID` | 304-316 | 从快照取 Swarm Cluster ID（Swarm）或 info.ID（单机），供卷资源 ID 构造（volume.md） |
| `FillSnapshotData` | 318-346 | 把库中快照（可选含 raw）回填到 endpoint 对象 |

接口定义：`api/portainer.go:2098-2103`（`SnapshotService`）。

---

## 7. Handler 直连 SDK 的标准模式

原生 handler 统一持有 `*dockerclient.ClientFactory`，通过工具函数取客户端：

### 通用工具（`api/http/handler/docker/utils/get_client.go`，28 行）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `GetClient` | 14-28 | 从 request context 取 endpoint（`middlewares.FetchEndpoint`）→ 读 `X-PortainerAgent-Target` 头作为 nodeName（Swarm 多节点定向）→ `CreateClient(endpoint, agentTargetHeader, nil)` 返回 `*client.Client` |

### 使用该模式的调用方

| 调用方 | 位置 | 说明 |
|---|---|---|
| 镜像列表 handler | `api/http/handler/docker/images/images_list.go:46` | 见 image.md |
| GPU 检测 handler | `api/http/handler/docker/containers/container_gpus_inspect.go:49` | 见 container.md |
| Dashboard handler | `api/http/handler/docker/dashboard.go` | ContainerList/ImageList/Info/ServiceList/VolumeList/NetworkList 汇总 |
| 堆栈 handler | `api/http/handler/stacks/handler.go:38` | 栈名查重等（见 compose.md） |
| Webhook 执行 | `api/http/handler/webhooks/webhook_execute.go:65` | ServiceInspect → 改 spec → ServiceUpdate |
| 升级服务 | `api/internal/upgrade/upgrade.go:36` | 自检镜像存在性（DistributionInspect） |

典型生命周期：`GetClient` 取客户端 → 调 SDK 方法 → `logs.CloseAndLogErr(cli)` 关闭（SDK client 持有底层连接，需显式关闭）。
