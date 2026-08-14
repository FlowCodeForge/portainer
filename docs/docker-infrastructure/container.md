# Container（容器）操作

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

容器操作绝大多数走**代理路径**（透传 Docker Engine API + 拦截改写），少量复杂编排走**原生 handler 路径**。公共机制（代理分发、RC、安全设置）见 [docker-sdk.md](./docker-sdk.md)。

---

## 1. 操作 → 代码映射总表

### 1.1 代理路径（`api/http/proxy/factory/docker/transport.go` 的 `proxyContainerRequest`，269-315 行）

| Docker API | 拦截处理 | 处理函数（同目录 `containers.go`） | 行号 |
|---|---|---|---|
| `POST /containers/create` | 创建装饰（安全限制 + 建私有 RC） | `decorateContainerCreationOperation` | 273-274 → 172-273 |
| `POST /containers/prune` | 仅管理员 | `administratorOperation` | 276-277 |
| `GET /containers/json`（列表） | 列表重写 + RC 装饰/过滤 + 标签黑名单 | `containerListOperation` | 279-280 → 57-89 |
| `GET /containers/{id}/json`（inspect） | 单对象重写 | `containerInspectOperation` | 289-290 → 93-110 |
| `POST /containers/{id}/update` | 更新装饰（设备映射限制） | `decorateContainerUpdateOperation` | 293-294 → 275-315 |
| `DELETE /containers/{id}` | 受限删除 + 清理 RC | `executeGenericResourceDeletionOperation` | 302-303 |
| 其他 `/containers/{id}/{action}`（start/stop/kill/restart/pause/unpause/wait/rename/resize/logs/stats/top/attach/exec/archive 等） | 仅做 RC 访问控制，请求/响应透传 | `restrictedResourceOperation` | 297 |
| `POST /exec/{id}/start` 等 | 先 SDK 反查 exec 所属容器再做 RC 校验 | `proxyExecRequest` | 317-332 |

> 关键结论：**启动/停止/杀掉/重启/删除/日志/stats 等操作 Portainer 不改写请求体**，仅做访问控制后透传给 Docker daemon；只有 create / 列表 / inspect / update 有请求或响应改写。

### 1.2 原生 handler 路径（`api/http/handler/docker/containers/handler.go:33-38`）

| 端点 | 处理函数 | 文件 | 行号 |
|---|---|---|---|
| `GET /docker/{id}/containers/{cid}/gpus` | `containerGpusInspect` | `container_gpus_inspect.go` | 36-87 |
| `POST /docker/{id}/containers/{cid}/recreate` | `recreate` | `recreate.go` | 28-65 |

中间件链：`bouncer.AuthenticatedAccess` + `middlewares.CheckEndpointAuthorization`。

### 1.3 后台 SDK 调用点（非 HTTP 请求触发）

| 场景 | 位置 | SDK 调用 |
|---|---|---|
| 容器重建 | `api/docker/container.go:72` `Recreate` | 见第 4 节 |
| 状态统计 | `api/docker/stats/container_stats.go:25` `CalculateContainerStats` | `ContainerList` + 并发 `ContainerInspect` |
| Dashboard 汇总 | `api/http/handler/docker/dashboard.go:50-170` | `ContainerList(All:true)`（72 行）+ `ImageList`/`Info`/`ServiceList`/`VolumeList`/`NetworkList` |
| 栈名查重 | `api/http/handler/stacks/handler.go:168-211` | `ServiceList`（Swarm 标签）+ `ContainerList`（Compose 标签） |
| Webhook 服务更新 | `api/http/handler/webhooks/webhook_execute.go:65` | `ServiceInspectWithRaw` → 改 spec → `ServiceUpdate` |
| 环境快照 | `pkg/snapshot/docker.go:134-229` | `ContainerList(All:true)` + 逐容器 `ContainerInspect` |
| 远程栈部署 | `api/stacks/deployments/deployer_remote.go:206-348` | 见 compose.md |
| 自身升级 | `api/internal/upgrade/upgrade_docker.go` | `ImageList` / `DistributionInspect` 验证镜像 |

---

## 2. 代理层核心逻辑（`api/http/proxy/factory/docker/containers.go`，315 行）

### 2.1 函数映射表

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `getInheritedResourceControlFromContainerLabels` | 33-53 | 容器无独立 RC 时的继承查找：`ContainerInspect` 取标签 → 有 `com.docker.swarm.service.id` 查服务级 RC → 否则按栈标签拼 Stack 资源 ID 查栈级 RC |
| `containerListOperation` | 57-89 | 容器列表响应改写核心（流程见 2.2） |
| `containerInspectOperation` | 93-110 | 单容器：打 IsPortainer 标记（107）+ `applyAccessControlOnResource` RC 装饰/403（109） |
| `selectorContainerLabelsFromContainerListOperation` | 131-135 | 列表响应从顶层 `Labels` 取标签 |
| `selectorContainerLabelsFromContainerInspectOperation` | 116-125 | inspect 响应从 `Config.Labels` 取标签 |
| `filterContainersWithBlackListedLabels` | 139-170 | 按 Settings 的 `BlackListedLabels` 剔除标签完全匹配的容器 |
| `decorateContainerCreationOperation` | 172-273 | **普通用户创建容器的安全限制**（7 项设置、8 个 HostConfig 字段，见 2.3） |
| `decorateContainerUpdateOperation` | 275-315 | 更新时额外校验 Devices（308-310），最终进 `restrictedResourceOperation` |

### 2.2 容器列表响应改写流程（`containerListOperation`，57-89 行）

```
① 解析响应 JSON 数组
② 构造 resourceOperationParameters{resourceIdentifierAttribute: "Id",
   resourceType: ContainerResourceControl, labelsObjectSelector: 列表标签选择器}
③ applyAccessControlOnResourceList → 管理员: 全量装饰 RC / 普通用户: 过滤
④ filterContainersWithBlackListedLabels → 剔除黑名单标签容器
⑤ applyPortainerContainers → 给 Portainer 自身容器打 IsPortainer: true
⑥ RewriteResponse 重写响应
```

其中第 ⑤ 步实现在 `portainer.go`（同目录，47 行）：init 时（9-16 行）取 `os.Hostname()`（容器化部署下即容器 ID 前 12 位），`applyPortainerContainer(s)`（18/36 行）比对响应中容器 Id 前 12 位，命中则标记。

### 2.3 创建安全限制（`decorateContainerCreationOperation`，172-273 行）

定义 `PartialContainer` 局部结构体（173-187 行）只解析 `HostConfig` 中的安全敏感字段。管理员跳过全部限制；普通用户按端点 `SecuritySettings`（205 行读取）逐项校验（220-258 行），命中即 403 + 预定义错误（24-31 行）：

| 安全设置 | 禁止的 HostConfig 字段 | 说明 |
|---|---|---|
| `AllowPrivilegedModeForRegularUsers` | `Privileged` | 特权模式 |
| `AllowHostNamespaceForRegularUsers` | `PidMode == "host"` | 宿主 PID 命名空间 |
| `AllowDeviceMappingForRegularUsers` | `Devices` | 设备映射 |
| `AllowSysctlSettingForRegularUsers` | `Sysctls` | 内核参数 |
| `AllowSecurityOptForRegularUsers` | `SecurityOpt` | 安全选项（如 apparmor） |
| `AllowContainerCapabilitiesForRegularUsers` | `CapAdd` / `CapDrop` | Linux capabilities |
| `AllowBindMountsForRegularUsers` | `Binds`（以 `/` 开头）、`Mounts` 中 `Type=="bind"` | 宿主路径 bind 挂载 |

校验通过后重置 request.Body（260 行）→ 执行真实请求（263 行）→ 201 成功则 `decorateGenericResourceCreationResponse`：从响应取 `Id` → 为创建者建私有 RC → 把 RC 写入响应 `Portainer.ResourceControl` 字段。

### 2.4 exec 的 RC 校验（`proxyExecRequest`，transport.go 317-332 行）

exec 相关 URL（`/exec/{id}/start`）**不含容器 ID**，无法直接从 URL 做资源控制。处理方式（326 行）：

```go
// 现场 CreateClient + SDK ContainerExecInspect 反查 exec 所属的容器 ID
// 再对该容器执行 restrictedResourceOperation
```

这是代理层直接消费 SDK 做鉴权辅助的典型场景。

---

## 3. 原生 handler 详解

### 3.1 GPU 检测（`container_gpus_inspect.go:36-87`）

1. `dockerClientFactory.CreateClient`（49 行，支持 `X-PortainerAgent-Target` 定向节点）；
2. `cli.ContainerInspect`（54 行）取 `HostConfig.DeviceRequests`；
3. 查找 `Driver == "nvidia"` 或 `Capabilities[0][0] == "gpu"` 的设备请求（63-73 行）；
4. 返回 `{"gpus": "none"|"all"|"id:0,1,..."}`——`Count == -1` 表示 all，否则拼接 DeviceIDs。

### 3.2 容器重建端点（`recreate.go:28-65`）

- 读取 `RecreatePayload{PullImage}`（19-22 行）；
- 校验 `AuthorizedEndpointOperation`（44 行）；
- 调 `containerService.Recreate(ctx, endpoint, containerID, payload.PullImage, "", agentTargetHeader)`（50 行，详见第 4 节）；
- **成功后迁移旧容器的关联数据**：
  - `updateWebhook`（86-98 行）：把 webhook 的 ResourceID 改绑到新容器 ID；
  - `createResourceControl`（67-84 行）：把旧容器私有 RC 改绑到新容器 ID；
- 异步 `images.EvictImageStatus`（58-62 行）按三个 key（容器 ID、compose 标签、swarm service 标签）清理镜像状态缓存。

---

## 4. 容器重建服务（`api/docker/container.go:239` 行，`ContainerService`）

### 4.1 类型定义

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `ContainerService` struct | 21-24 | 持有 clientFactory 与 dataStore |
| `NewContainerService` | 26 | 构造器；`api/http/server.go` 中实例化 |
| `applyVersionConstraint` | 35-55 | Docker API < 1.44 时清空 `EndpointsConfig` 中的 MAC 地址（`clearMacAddrs` 57-69 行）——旧版本 Docker 重用已分配 MAC 会报错 |
| `Recreate` | 72-239 | 九步重建流程（见下） |

### 4.2 Recreate 九步流程（72-239 行）

前置：`CreateClient`（73）→ `ContainerInspectWithRaw(containerId, true)`（81，`InsertDefaults=true` 保证 spec 完整）→ `images.ParseImage` 解析/替换镜像 tag（87-102，`imageTag` 非空时 `img.WithTag` 并更新 `container.Config.Image`）。

```
① 可选强制拉镜像        104-110  images.NewPuller(cli, ...).Pull（见 image.md §3.2）
② 停止旧容器            114      cli.ContainerStop
③ 旧容器改名 {name}-old  120      cli.ContainerRename
④ 断开旧容器全部网络     129-141  逐网络 cli.NetworkDisconnect；记下第一个网络
                                 （作为创建时网络以保留 IP 配置）
⑤ 注册回滚 defer        143-164  失败时：恢复旧名、重连网络、重启旧容器
⑥ 创建新容器（同名）     166-185  applyVersionConstraint → cli.ContainerCreate
                                 沿用旧 Config/HostConfig + 仅第一个网络
   并注册第二个 defer    187-201  创建后续步骤失败时删除新容器
⑦ 补连其余网络          209-219  逐个 cli.NetworkConnect
                                 （Docker 创建时只能连一个网络，见注释引用 moby#17750）
⑧ 启动新容器            223      cli.ContainerStart
⑨ 删除旧容器            229      cli.ContainerRemove；置 restore=false 禁用回滚
返回                     233-238  ContainerInspectWithRaw 返回新容器详情
```

**双 defer 回滚设计**：两个 defer 按 LIFO 生效——若 ⑦⑧ 失败，先执行 defer ②（删除新容器）再执行 defer ①（恢复旧容器）；全部成功后 ⑨ 显式关掉回滚开关。

---

## 5. exec / attach 控制台（WebSocket 通道）

普通日志/stats 走代理透传（chunked 流），但**交互式终端**需要 WebSocket + TCP hijack：

### 5.1 文件映射（`api/http/handler/websocket/`）

| 函数 | 文件:行号 | 逻辑说明 |
|---|---|---|
| 路由注册 `/websocket/exec`、`/websocket/attach` | `handler.go:35-42` | 顶层挂载于 `api/http/handler/handler.go:283-284`（`/api/websocket`） |
| `websocketExec` | `exec.go:40` | 校验 execID 十六进制 + endpointId 参数 + `AuthorizedEndpointOperation` 鉴权 |
| `handleExecRequest` | `exec.go:80` | Agent/Edge 环境转 WebSocket 反代；本地环境升级连接后 hijack |
| `proxyAgentWebsocketRequest` / `proxyEdgeAgentWebsocketRequest` | `proxy.go:32 / 18` | `websocketproxy.NewProxy` 反代 WebSocket；Director 注入签名头与 `X-PortainerAgent-Target`（82-87）；Edge 走反向隧道地址并 `KeepTunnelAlive`（89-92） |
| `initDial` | `initdial.go:12` | 支持 unix/npipe/TLS socket 直连 daemon 的 TCP 连接建立 |
| `createExecStartRequest` | `exec.go:117` | 构造 `POST /exec/{id}/start`，头 `Connection: Upgrade`、`Upgrade: tcp`，payload `{Tty:true, Detach:false}` |
| `hijackExecStartOperation` | `exec.go:99` | `ws.HijackRequest(websocketConn, conn, execStartRequest)` 在 WebSocket 与 hijacked TCP 间双向泵数据 |
| `websocketAttach` / `createAttachStartRequest` | `attach.go:37 / 128-139` | 同 exec 模式，请求为 `POST /containers/{id}/attach?stdin=1&stdout=1&stderr=1&stream=1`；`hijackAttachStartOperation`（95）额外对 TCP 设 KeepAlive（110-118） |

### 5.2 完整流程（本地环境）

```
前端 containerConsoleController.js:95-104
  ① POST /endpoints/{id}/docker/containers/{cid}/exec（走代理，restrictedResourceOperation 校验）
  ② 用返回的 execId 建 WebSocket: /api/websocket/exec?endpointId=...&id=...
服务端
  ③ 鉴权 → http 升级为 WebSocket
  ④ initDial 直连 Docker daemon socket
  ⑤ 向 daemon 发 POST /exec/{id}/start（Upgrade: tcp）→ hijack 出双向 TCP
  ⑥ ws.HijackRequest 在 WebSocket ↔ TCP 间泵数据
```

---

## 6. 容器状态统计（`api/docker/stats/container_stats.go`，139 行）

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `DockerClient` 接口 | 21-23 | 最小接口（仅 `ContainerInspect`），便于单测 mock |
| `CalculateContainerStats` | 25-85 | 非 Swarm：对 `ContainerList` 结果逐个并发（信号量限 5，34/43 行）`ContainerInspect`，统计 running/stopped/healthy/unhealthy；NotFoundError 容忍跳过（50-56，注释 BE-12567：容器可能在 inspect 前已消失） |
| `getContainerStatus` | 87-110 | 按 `State.Status` 与 `State.Health.Status` 分类 |
| `CalculateContainerStatsForSwarm` | 114-138 | Swarm 轻量版：直接按列表 `State` 与 `Status` 字符串（含 `(healthy)`）近似统计，避免跨节点 inspect |

被 dashboard handler（`dashboard.go:149`）与快照（`pkg/snapshot/docker.go`）共用。
