# Volume（卷）操作

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

卷领域同样**没有专属 HTTP handler**（不存在 `api/http/handler/volumes/` 目录），CRUD 走代理路径。卷的三个特点：① RC 标识是「卷名_dockerID」复合格式（卷没有 UUID）；② 创建时有 bind 限制与重名预检；③ 支持**卷文件浏览**（volume browse，走 Agent `/v2/browse` 通道）。

---

## 1. 操作 → 代码映射总表

### 1.1 代理路径（`api/http/proxy/factory/docker/transport.go` 的 `proxyVolumeRequest`，378-395 行）

| Docker API | 拦截处理 | 行号 |
|---|---|---|
| `POST /volumes/create` | `decorateVolumeResourceCreationOperation`（bind 限制 + 重名预检 + 建 RC） | 383 → volumes.go:153-207 |
| `POST /volumes/prune` | `administratorOperation`（仅管理员） | 386 |
| `GET /volumes`（列表） | `volumeListOperation`（补 ResourceID + RC 装饰/过滤） | 389 → volumes.go:41-79 |
| 其余 `/volumes/{name}`（inspect/delete/update 等） | `restrictedVolumeOperation` | 393 → volumes.go:236-253 |

### 1.2 卷文件浏览（Agent 通道，transport.go 的 `proxyAgentRequest`，185-242 行）

| 请求 | 拦截处理 | 行号 |
|---|---|---|
| `/v2/browse`（无 `volumeID` 参数，即宿主文件系统浏览） | 要求管理员 | 189-193 |
| `/v2/browse?volumeID=xxx`（卷内文件浏览） | 用卷名构造 resourceID → `restrictedResourceOperation(..., volumeBrowseRestrictionCheck=true)` | 196-204 |

`volumeBrowseRestrictionCheck`（transport.go:622-631 行）：非管理员且端点设置 `AllowVolumeBrowserForRegularUsers=false` 时返回 403。浏览请求最终透传给 **Agent 进程**（本仓库不含 Agent 端实现）。

### 1.3 SDK 直接调用点

| 场景 | 位置:行号 | SDK 调用 | 说明 |
|---|---|---|---|
| 重名预检 | `volumes.go:190` | `VolumeInspect` | 创建装饰逻辑内部（见 §3.2） |
| RC 鉴权辅助 | `transport.go:692-739` | — | 卷不走 UUID 解析（resourceID 直接用卷名，见 §2） |
| Dashboard 汇总 | `dashboard.go:110-121` | `VolumeList` | 逐卷 `VolumeResourceControlGetter` 过滤 |
| 环境快照 | `pkg/snapshot/docker.go:243-253` | `VolumeList` | 记录 VolumeCount 与 `SnapshotRaw.Volumes` |
| Docker ID 获取 | `volumes.go:264-305` | `client.Info()`（兜底） | 见 §4 |
| 容器重建 | `api/docker/container.go` | `VolumeInspect` | 挂载卷沿用（间接经 ContainerInspect） |

> 未发现业务代码直接调 `VolumeCreate`——卷的创建完全由代理层透传 Docker HTTP API 完成。

---

## 2. 卷的资源 ID 体系（与容器/网络最大的差异）

**问题**：容器/网络有 UUID 作 RC 标识，而**卷没有 UUID**，且不同 Docker 主机上可以有同名卷。解决方案（transport.go:708-714 行注释说明）：RC 标识 = `卷名_dockerID`。

| 函数 | 文件:行号 | 逻辑说明 |
|---|---|---|
| `getVolumeResourceID` | volumes.go:255-262 | 拼接 `卷名_dockerID`；dockerID 经 `getDockerID` 获取 |
| `decorateVolumeResponseWithResourceID` | volumes.go:104-117 | 列表/inspect 响应中把 `Name` 转成 Portainer resourceID 写入 `ResourceID` 字段——前端据此判断卷的归属 |
| `getDockerID` | volumes.go:264-305 | **三级缓存**获取 Docker/Swarm 集群 ID：① 内存缓存（带锁）→ ② 快照（`snapshotService.FillSnapshotData` + `snapshot.FetchDockerID`）→ ③ 现场 `client.Info()`（Swarm 用 `info.Swarm.Cluster.ID`，否则 `info.ID`） |
| `VolumeResourceControlID` | `api/uac/volumes.go:26-28` | uac 包的版本：直接用卷名（注释说明与代理层 `getVolumeResourceID` 的差异是待解决的 TODO） |
| `VolumeResourceControlGetter` | `api/uac/volumes.go:8-22` | 泛型 RC 获取器，Dashboard/快照过滤用 |

---

## 3. 代理层逻辑详解（`api/http/proxy/factory/docker/volumes.go`，305 行）

### 3.1 函数映射表

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `getInheritedResourceControlFromVolumeLabels` | 25-37 | `VolumeInspect` 后按栈标签（`com.docker.stack.namespace` / `com.docker.compose.project`）找继承 RC |
| `volumeListOperation` | 41-79 | 解析 `Volumes` 数组 → 逐卷 `decorateVolumeResponseWithResourceID` → `applyAccessControlOnResourceList` 过滤/装饰 → 重写响应 |
| `volumeInspectOperation` | 83-102 | 单卷同上 |
| `CheckVolumeBodyRestrictions` | 128-151 | 若 `DriverOpts.type == "bind"` 返回 `ErrBindMountsForbidden`（403）——通过卷 driver 绕过 bind 限制的防护 |
| `decorateVolumeResourceCreationOperation` | 153-207 | 创建的定制逻辑（见 §3.2） |
| `decorateVolumeCreationResponse` | 209-234 | 201 后建私有 RC + 装饰响应 |
| `restrictedVolumeOperation` | 236-253 | GET 走 inspect 装饰；DELETE 走 `executeGenericResourceDeletionOperation`（删 RC）；其他方法走 `restrictedResourceOperation` |
| `getVolumeResourceID` / `getDockerID` | 255-262 / 264-305 | 见 §2 |

### 3.2 卷创建的完整流程（`decorateVolumeResourceCreationOperation`，153-207 行）

```
① bind 限制（159-178 行）
   非管理员 且 端点 AllowBindMountsForRegularUsers=false
   → CheckVolumeBodyRestrictions：DriverOpts.type == "bind" → 403
   （防止用户用 local driver + bind opts 绕过容器创建时的 bind 限制）

② 重名预检（180-195 行）
   读请求头 X-PortainerAgent-VolumeName（前端预检卷名）
   → SDK VolumeInspect 检查同名卷是否已存在
   → 存在则直接返回 409（不透传创建请求）

③ 透传创建（197-206 行）
   → 201 成功 → decorateVolumeCreationResponse
     （卷名 + dockerID 拼资源 ID → 建私有 RC → 装饰响应）
```

> 为什么需要重名预检：卷创建是幂等的（同名卷已存在时 Docker 不报错而是复用），而 Portainer 需要为"新建"卷建立 RC——预检避免把别人已建的卷"认领"成自己的。

### 3.3 卷删除流程

```
DELETE /api/endpoints/{id}/docker/volumes/{name}
  ① restrictedVolumeOperation（volumes.go:236-253）→ DELETE 分支
  ② executeGenericResourceDeletionOperation（transport.go:838-860）
     a. restrictedResourceOperation 鉴权（resourceID = 卷名_dockerID）
     b. 透传删除，204/200 成功
     c. 删除对应 RC 记录
```

单测参考：`api/http/proxy/factory/docker/volumes_test.go`。

---

## 4. getDockerID 三级缓存（volumes.go:264-305 行）

卷资源 ID 依赖 dockerID，但每次创建 RC 都远程调 `Info()` 代价高，故：

```
① Transport 结构体内的内存缓存（sync.Mutex 保护，dockerID 字段）
② snapshotService 快照
   FillSnapshotData(endpoint) → snapshot.FetchDockerID
   （api/internal/snapshot/snapshot.go:304-316）
③ 现场 SDK client.Info()
   Swarm 集群 → info.Swarm.Cluster.ID
   单机       → info.ID
```

---

## 5. 与其他领域的协作

- **Container**：容器挂载卷的 RC 继承（容器通过 stack 标签继承栈级 RC，卷同理）；容器重建沿用挂载配置（[container.md](./container.md) §4）。
- **Network**：同构的代理拦截模式（create 装饰 / list 重写 / delete 清理），差异在卷的复合资源 ID，见 [network.md](./network.md)。
- **Compose**：compose 栈声明的外部卷（external volumes）不建 RC，靠栈标签继承；`docker compose down` 默认不删卷，详见 [compose.md](./compose.md)。
- **Agent/Edge**：卷文件浏览走 Agent `/v2/browse` 通道（本文 §1.2），Edge 环境经反向隧道。
