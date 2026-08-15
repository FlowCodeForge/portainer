# Portainer 整体架构分析文档

> 代码基线：Portainer CE 2.44.0（`api/portainer.go:2116`），分支 `docs`，commit `269738a01`。
> 文中所有路径均**相对仓库根目录**（Go 后端在 `api/` 与 `pkg/`，前端在 `app/`，均无额外嵌套层）。
> 所有行号格式为「定义起始行 – 结束行」，行号可能随后续提交漂移，定位时请以函数名为准、行号为辅。
> Docker 领域的细节（代理拦截、RBAC、Compose 部署等）见 [../docker-infrastructure/README.md](../docker-infrastructure/README.md)，本文不重复展开。

## 文档导航

按研究目标划分，共 3 个领域文档：

| 文档 | 研究目标 | 覆盖内容 |
|---|---|---|
| [backend.md](./backend.md) | 后端总体 | 入口与启动流程、目录职责清单、HTTP API 层（路由/中间件/认证）、数据层（boltdb/迁移）、后台任务、Kubernetes 攻面、横切机制 |
| [frontend.md](./frontend.md) | 前端总体 | AngularJS + React 混合架构、r2a 桥接、UI Router 路由、axios 客户端与拦截器、主题机制、目录归属、OpenAPI 类型生成 |
| [agent-edge.md](./agent-edge.md) | Agent 与 Edge | 环境类型体系、Agent 签名协议、Edge key 格式、chisel 反向隧道、心跳轮询、Edge stacks/jobs 下发、异步模式 |

## 系统组成

Portainer 由三个运行时角色组成（本仓库只包含其中两个的代码）：

```
┌────────────────────────────────────────────────────────────────────┐
│                       浏览器（AngularJS + React SPA）                │
│                     app/ 打包为静态资源，由 server 提供              │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ HTTP(S) /api/**（cookie 会话或 API Key）
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                     Portainer Server（本仓库）                      │
│  api/cmd/portainer/main.go 唯一入口                                 │
│  ├─ HTTP:9443/9000   api/http/   路由 + 中间件 + REST handler        │
│  ├─ boltdb           api/database/ + api/dataservices/ + api/datastore/ │
│  ├─ 快照循环          api/internal/snapshot/（默认 5 分钟）           │
│  ├─ GitOps 轮询       api/gitops/scheduling/（Source 变更自动重部署） │
│  └─ 隧道服务器:8000   api/chisel/（Edge 反向隧道入口）                │
└───────┬──────────────────────────────────┬─────────────────────────┘
        │ ①直连 Docker API                 │ ②经 Agent（签名头）
        │   unix/npipe/TCP+TLS             │   Agent 每 Swarm/K8s 节点一个
        ▼                                  ▼
┌──────────────────┐        ┌────────────────────────────────┐
│  Docker Daemon   │        │  Portainer Agent（独立仓库      │
│  /var/run/       │        │  portainer/agent，非本仓库）    │
│  docker.sock     │        │  Edge 变体：主动连 server:8000  │
└──────────────────┘        │  建立反向隧道，轮询 :9443 检入   │
                            └────────────────────────────────┘
```

关键事实：

- **单二进制**：本仓库唯一的 Go main 包是 `api/cmd/portainer`（`api/cmd/portainer/main.go:675`）。Agent 二进制已拆分到独立仓库 `portainer/agent`，本仓库只保留与其通信的协议头常量（`api/portainer.go:2131-2151`）和客户端辅助函数（`api/agent/version.go:18`）。
- **三种连接形态**（`EndpointType` 枚举，`api/portainer.go:2308-2324`）：直连（Docker socket / K8s local）、Agent 中转、Edge Agent（反向隧道 + 拉取式检入）。
- **拉取式 Edge 模型**：server 从不主动推送 Edge 任务。Edge Agent 按间隔轮询 `GET /api/endpoints/{id}/edge/status`（默认 5 秒，`api/portainer.go:2154-2155`），由响应携带隧道指令、Edge stacks 与 Edge jobs。

## 各文档的公共前提

### 1. 分层约定

后端遵循「handler → service → dataservices → boltdb」的单向依赖：

| 层 | 位置 | 职责 |
|---|---|---|
| HTTP 层 | `api/http/handler/<domain>/` | 参数解析、权限（经 Bouncer）、调 service、写响应 |
| 业务服务 | `api/<domain>/`、`api/internal/`、`pkg/lib*/` | 编排多客户端调用、领域逻辑 |
| 数据服务 | `api/dataservices/<entity>/` | 每个实体一个 bucket 服务，泛型 CRUD |
| 存储 | `api/database/boltdb/` | 连接、加密、事务、对象序列化 |

### 2. 版本与迁移

- 当前 `APIVersion = "2.44.0"`（`api/portainer.go:2116`）。
- 迁移按 semver 版本链注册（`api/datastore/migrator/migrator.go:202` `initMigrations`），每版本一个 `migrate_*.go` 文件；执行前自动备份、失败自动回滚（`api/datastore/migrate_data.go:19-61`）。
- 需要容器客户端的复杂迁移走 PostInit 通道（`api/datastore/postinit/`），经 PendingActions 机制延迟到环境可用时执行（`api/pendingactions/pendingactions.go:54`）。

### 3. 术语对照

Portainer 正在把「endpoint」重命名为「environment」，代码中两者混用：数据库 bucket、Go 类型、旧 API 路径用 endpoint；新 API（`/api/gitops/*`）与前端用 environment。本文按代码原文引用。
