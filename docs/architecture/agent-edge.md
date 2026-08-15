# Agent 与 Edge 架构

> 行号基线：分支 `docs` / commit `269738a01`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

**重要前提**：Agent 二进制在独立仓库 `portainer/agent`，不在本仓库。本文以 server 侧代码为准描述协议与交互；agent 内部实现（监听端口、socket 转发、`edge_agent.db`）仅在前端安装脚本快照中可见（`app/react/edge/components/EdgeScriptForm/__snapshots__/scripts.test.ts.snap:29-40`：`-e EDGE=1 -e EDGE_ID=... -v portainer_agent_data:/data --privileged`）。

## 1. 环境类型体系（`api/portainer.go:2308-2324`）

| 枚举值 | 含义 | 连接方式 |
|---|---|---|
| `DockerEnvironment` (1) | 直连 Docker API | unix socket / npipe / TCP(±TLS) |
| `AgentOnDockerEnvironment` (2) | Docker 集群上的 Agent | HTTP(S) → agent，带签名头 |
| `AzureEnvironment` (3) | Azure ACI | Azure API（agent 不参与） |
| `EdgeAgentOnDockerEnvironment` (4) | Docker 上的 Edge Agent | 反向隧道（§4） |
| `KubernetesLocalEnvironment` (5) | 本地 K8s | kubeconfig |
| `AgentOnKubernetesEnvironment` (6) | K8s 上的 Agent | `https://<endpoint.URL>/kubernetes` + 签名头 |
| `EdgeAgentOnKubernetesEnvironment` (7) | K8s 上的 Edge Agent | 隧道上的 `/kubernetes` |

平台枚举 `AgentPlatformDocker=1`、`AgentPlatformKubernetes=2`（`api/portainer.go:2212-2218`）。

## 2. Agent 通信协议（server → agent）

### 2.1 协议头（`api/portainer.go:2131-2151`）

| 头 | 作用 |
|---|---|
| `Portainer-Agent` | agent 响应头，携带版本号 |
| `Portainer-Agent-Platform` | agent 响应头，平台类型 |
| `X-PortainerAgent-Target` | **请求头**：Swarm 集群中定向到指定节点名 |
| `X-PortainerAgent-Signature` | **请求头**：server 签名，agent 校验来源 |
| `X-PortainerAgent-PublicKey` | **请求头**：server 公钥（hex），供 agent 验签 |
| `X-PortainerAgent-EdgeID` | Edge agent 检入时上报的身份 |
| `X-PortainerAgent-SA-Token` | K8s 代理场景携带的用户 ServiceAccount token |

### 2.2 签名机制

- 签名服务 `crypto.ECDSAService`（`api/crypto/ecdsa.go:23-39`）：ECDSA P-256，密钥对启动时生成并落盘（`api/cmd/portainer/main.go:320-342`）。
- 固定签名消息 `"Portainer-App"`（`api/portainer.go:2149-2151`）；**若设置 `AGENT_SECRET` 环境变量则直接对 secret 签名**（`ecdsa.go:110-113`），用于多实例共享信任。
- 注入点：Docker 代理 transport、agent transport、K8s agent/edge transport、Docker/K8s 客户端工厂、WebSocket proxy（`api/http/proxy/factory/docker/transport.go:144-152` 等六处）。

### 2.3 多节点路由

server 创建 Docker 客户端时按节点名设置 `X-PortainerAgent-Target`（`api/docker/client/client.go:111-118`）；各 docker handler 从请求头读取该值（`api/http/handler/docker/utils/get_client.go:20`）。前端在容器列表选择节点时经 `agentInterceptor` 注入同一头（见 frontend.md §4.1）。

## 3. Edge key

生成（`api/chisel/key.go:10-24`）：

```
key = base64( portainer_instance_url | tunnel_server_addr:port | tunnel_server_fingerprint | endpoint_ID )
```

- 生成时机：管理员预置环境时（`api/http/handler/endpoints/endpoint_create.go:364-398`），host 会经 `ParseHostForEdge` 校验并拒绝 localhost（`api/internal/edge/url.go:10-32`）。
- 存储于 `Endpoint.EdgeKey`（`api/portainer.go:480`）；agent 侧持有 key 即完成「环境绑定」。
- 解除关联时清空 EdgeID 并把 key 中 `http://host:9000` 重写为 `https://host:9443`（`endpoint_association_delete.go:48-82`）。

## 4. 反向隧道（chisel）

server 侧实现在 `api/chisel/`，基于 `github.com/jpillora/chisel v1.11.6`（`go.mod:37`）。

### 4.1 隧道生命周期

```
用户请求 Edge 环境 → proxyManager 需要代理
  → TunnelAddr(endpoint)              api/chisel/tunnel.go:125（阻塞等待隧道就绪）
    → Open(endpoint)                  api/chisel/tunnel.go:34
       · 校验 Edge 环境、非 AsyncMode、已信任（UserTrusted）
       · 在动态端口段 49152-65535 分配本地端口（tunnel.go:22-25）
       · chisel.AddUser(user, pass, "^R:0.0.0.0:<port>$")   ← 反向授权
       · 凭据用 EdgeID 派生密钥加密后放入状态响应（tunnel.go:231-240）
  → 下次 agent 检入 GET /edge/status 时，响应携带 port + credentials
  → agent 主动向 server:8000 发起 chisel 连接，把 server 本地端口反向暴露
  → server 的请求经 http://127.0.0.1:<port> 穿透到 agent 背后的 Docker/K8s API
```

### 4.2 关键常量与循环

| 项 | 值 | 位置 |
|---|---|---|
| 隧道服务器监听 | `0.0.0.0:8000` | `api/cli/defaults.go:8-9`；启动 `main.go:571-573` |
| 空闲超时 | 4m30s | `api/chisel/service.go:21-26` |
| 巡检间隔 | 10s（空闲隧道先打快照再关闭） | `service.go:234-247` |
| ping 超时 / 保活 | 8s / 25s | `service.go:21-26` |
| WebSocket 会话保活 | 1h（`KeepTunnelAlive`） | `api/portainer.go:2166-2167`；`websocket/proxy.go:89-92` |

快照补打：Edge 环境隧道打开时把 `endpoint.URL` 临时指向 `tcp://127.0.0.1:<tunnelPort>` 采集快照（`service.go:330-339`）。

## 5. 心跳与轮询（拉取式模型）

- **检入端点**：`GET /api/endpoints/{id}/edge/status`，注册于 `api/http/handler/endpointedge/handler.go:34`，PublicAccess（无 JWT/API Key）。
- 身份校验：`AuthorizedEdgeEndpointOperation`（`api/http/security/bouncer.go:183-199`）比对 `X-PortainerAgent-EdgeID` 头与库中 `Endpoint.EdgeID`；首次信任（TrustOnFirstConnect）走 `TrustedEdgeEnvironmentAccess:203-218`，配合 `UserTrusted` 字段实现「待信任设备 / Waiting Room」。
- 默认轮询间隔 5s（`DefaultEdgeAgentCheckinIntervalInSeconds`，`api/portainer.go:2154-2155`）；每环境可覆盖，生效值计算在 `EffectiveCheckinInterval`（`api/internal/edge/endpoint.go:32-42`），随状态响应下发。
- 心跳是内存 `sync.Map`（`api/dataservices/endpoint/endpoint.go:144-154`），检入时先查再 `UpdateHeartbeat`（`endpointedge_status_inspect.go:86-103`）。
- 在线判定：距上次检入 ≤ `interval*2 + 20s`（`api/http/handler/endpoints/endpoint_list.go:16-17`）。
- 响应有 ETag 缓存（FNV-32a + `If-None-Match`/304，`endpointedge_status_inspect.go:294-349`），避免每次检入重算 stacks/jobs。

### 状态响应内容（`endpointedge_status_inspect.go:47-60`）

`status`（IDLE/REQUIRED）、`port` + `credentials`（隧道指令）、`schedules`（Edge jobs）、`stacks`（版本列表）、`checkinInterval`。

## 6. Edge stacks 与 Edge jobs 下发

| 环节 | 端点 | 代码位置 |
|---|---|---|
| agent 轮询得知栈版本变化 | `GET /edge/status`（ stacks 列表） | `endpointedge_status_inspect.go:270` `buildEdgeStacks` |
| agent 拉取栈内容 | `GET /api/endpoints/{id}/edge/stacks/{stackId}` | `endpointedge/endpointedge_stack_inspect.go:36`；payload 类型 `api/edge/edge.go:11`（`StackPayload`：DirEntries + EntryFileName，兼容旧版 StackFileContent） |
| agent 回报部署状态 | `PUT /api/edge_stacks/{id}/status` | `edgestacks/edgestack_status_update.go:60`（旧版本上报忽略 `:125-127`；鉴权用 EdgeID 头 `:76-78`） |
| server 侧栈服务 | — | `api/internal/edge/edgestacks/service.go:34`（BuildEdgeStack/PersistEdgeStack/DeleteRecords）；更新时 `Version++` 并清空各环境状态 |
| Edge jobs 下发 | 随 `GET /edge/status` 的 `schedules`（cron + base64 脚本） | `endpointedge_status_inspect.go:224-268` |
| agent 回传 job 日志 | `POST /api/endpoints/{id}/edge/jobs/{jobID}/logs` | `endpointedge/endpointedge_job_logs.go:39`（落盘并置 LogsStatus=Collected） |

Edge groups：动态组按 Tag 计算成员（`api/internal/edge/edgegroup.go:11-43`）；`EndpointInEdgeGroup` 要求 Edge 且 `UserTrusted`（`endpoint.go:44-67`）。

## 7. K8s 环境差异

- kube client 分派（`api/kubernetes/cli/client.go:252-262` `CreateConfig`）：普通 agent → `https://<endpoint.URL>/kubernetes`（`buildAgentConfig:311`）；Edge agent → `TunnelAddr` 的 `http://127.0.0.1:<port>/kubernetes`（`buildEdgeConfig:340`）；均注入签名+公钥头。
- HTTP 代理同样三分（`api/http/proxy/factory/kubernetes.go:12-21`）：local/agent/edge，Edge 走隧道 HTTP 反代。
- K8s 代理额外携带 `X-PortainerAgent-SA-Token`（`edge_transport.go:38-44`），token 由 server 按用户 ServiceAccount 签发与缓存（`proxy/factory/kubernetes/token.go:24`、`token_cache.go:10`）。
- Edge stack 对 K8s 取 `ManifestPath` 作入口文件，未启用 manifest namespace 时强制 `DefaultNamespace`（`endpointedge_stack_inspect.go:87-98`）。
- exec/attach WebSocket 对 Edge 环境统一走 `proxyEdgeAgentWebsocketRequest` 并 `KeepTunnelAlive` 1h（`websocket/proxy.go:18-30`、`pod.go:95-101`）。

## 8. 异步模式（AsyncMode）

`Endpoint.Edge.AsyncMode`（`api/portainer.go:1174`）为离线防火墙场景设计（agent 只能出站轮询，不能保持隧道）。**CE 部分支持**：

- 隧道对 AsyncMode 环境报错（`api/chisel/tunnel.go:39-41` `ErrAsyncEnv`）。
- AsyncMode 环境的日志收集被拒（`edgejobs/edgejob_tasklogs_collect.go:78-80`："Async Edge Endpoints are not supported in Portainer CE"）。
- 列表过滤与在线判定考虑三间隔取最小值（`endpoints/filter.go:489-491`）。
- 完整的命令/快照轮询端点（`/api/endpoints/:id/async`）属 EE，CE 无此路由。
