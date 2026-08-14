# Docker 基础设施层分析文档

> 代码基线：Portainer CE 2.44.0（`api/portainer.go:2116`），分支 `docs`，commit `4097b69bc`。
> 文中所有路径均**相对仓库根目录**（本仓库 Go 后端直接位于 `api/` 与 `pkg/` 下，不存在 `portainer/api` 双层嵌套）。
> 所有行号格式为「定义起始行 – 结束行」，行号可能随后续提交漂移，定位时请以函数名为准、行号为辅。

## 文档导航

按研究目标划分，共 6 个领域文档：

| 文档 | 研究目标 | 覆盖内容 |
|---|---|---|
| [docker-sdk.md](./docker-sdk.md) | Docker SDK | go.mod 依赖、客户端工厂、5 种连接方式、SSRF 防护、代理拦截层（Transport）、快照服务 |
| [container.md](./container.md) | Container | 容器 CRUD 代理拦截、创建安全限制、容器重建（Recreate）、exec/attach WebSocket、GPU 检测、统计 |
| [image.md](./image.md) | Image | 镜像列表/拉取/推送/构建/导出、registry 凭据注入链、镜像新旧状态检测 |
| [network.md](./network.md) | Network | 网络创建/删除/连接代理拦截、系统网络资源控制、RC 继承链 |
| [volume.md](./volume.md) | Volume | 卷创建/删除代理拦截、bind 限制、重名预检、卷资源 ID 体系、卷文件浏览 |
| [compose.md](./compose.md) | Compose | Stack 类型体系、堆栈全生命周期（创建/部署/停止/删除/迁移/Git 自动更新）、compose-go 解析、Swarm 栈部署 |

## 总体架构

Portainer 对 Docker 的所有操作分为**两条路径**，这是理解全部文档的前提：

```
                        ┌────────────────────────────────────────────────────────┐
                        │                     浏览器前端                          │
                        └───────┬────────────────────────────────┬───────────────┘
                                │                                │
              路径① 代理路径                路径② 原生 handler 路径
        /api/endpoints/{id}/docker/**            /api/docker/{id}/**
        (Docker Engine API 风格 URL)         (Portainer 自有业务端点)
                                │                                │
                                ▼                                ▼
              api/http/handler/handler.go:234-243      api/http/handler/handler.go:227-228
              (StripPrefix 后交给 EndpointProxyHandler) (StripPrefix 后交给 DockerHandler)
                                │                                │
                                ▼                                ▼
              api/http/handler/endpointproxy/          api/http/handler/docker/handler.go:43-55
              proxy_docker.go:14                        /dashboard、/containers/*、/images/*
              (权限校验 + Edge 隧道处理)                  (WithEndpoint + dockerOnly 中间件)
                                │                                │
                                ▼                                │
              api/http/proxy/manager.go:39-52            │
              (按 endpointID 缓存代理实例)                 │
                                │                         │
                                ▼                         │
              api/http/proxy/factory/docker.go:19        │
              ┌─ unix/npipe → newDockerLocalProxy ─┐     │
              └─ TCP/TLS/Agent/Edge → HTTP 反代 ───┤     │
                                                 ▼       ▼
                        api/http/proxy/factory/docker/transport.go:138
                        ProxyDockerRequest（核心拦截器，979 行）
                        ① 剥离 /v1.xx 版本前缀
                        ② Agent/Edge 环境注入签名头
                        ③ 按 URL 首段查 prefixProxyFuncMap 分发（RBAC / RC 注入 / label 过滤 / registry 头改写）
                        ④ 未命中前缀 → adminOnlyRoutes 检查 → 兜底透传
                                │
                                ▼
                     ┌──────────┴──────────┐
                     ▼                     ▼
            Docker Daemon 直连     旁路 SDK 客户端
            (HTTPTransport)       api/docker/client/client.go:50 CreateClient
                                  (exec 反查、RC UUID 解析、卷重名预检等
                   ▲              鉴权辅助场景；以及原生 handler 的主链路)
                   │
        ┌──────────┴───────────┬─────────────────────┬──────────────────┐
        ▼                      ▼                     ▼                  ▼
   unix socket/npipe      TCP + 双向 TLS         Agent(签名头)     Edge Agent 反向隧道
                                              (reverse proxy   (reverseTunnelService
                                               聚合多节点)       TunnelAddr 取本地地址)
```

另有两条**后台链路**同样消费 Docker SDK：

- **快照链路**：`api/internal/snapshot/snapshot.go`（定时调度）→ `api/docker/snapshot.go`（门面）→ `pkg/snapshot/docker.go`（SDK 采集容器/镜像/网络/卷/版本信息）。
- **Compose 部署链路**：`api/http/handler/stacks/`（HTTP 层）→ `api/stacks/deployments/deployer.go`（门面）→ `api/exec/compose_stack.go`（ComposeStackManager）→ `pkg/libstack/compose/`（docker compose v2 进程内 API）。

## 两条路径的分工

| 维度 | 路径① 代理路径 | 路径② 原生 handler 路径 |
|---|---|---|
| URL 形态 | `/api/endpoints/{id}/docker/containers/json` 等 | `/api/docker/{id}/containers/{cid}/recreate` 等 |
| 本质 | Portainer 拦截/改写后透传 Docker Engine API | Portainer 自己实现的业务端点 |
| 覆盖操作 | 绝大多数容器/镜像/网络/卷 CRUD | 需要"读-改-写"编排或多 SDK 调用组合的操作 |
| 典型例子 | 容器列表（RC 过滤）、容器创建（安全限制）、镜像 pull（凭据替换） | 容器重建、GPU 检测、镜像列表（聚合节点名与使用状态）、堆栈管理 |
| 实现位置 | `api/http/proxy/factory/docker/`（`transport.go` + 各资源文件） | `api/http/handler/docker/`、`api/http/handler/stacks/` 等 |

## 目录速查表

| 目录/文件 | 职责 | 所属文档 |
|---|---|---|
| `api/docker/client/client.go` | Docker SDK 客户端工厂（所有连接方式） | docker-sdk |
| `api/docker/consts/labels.go` | Compose/Swarm 标签常量（贯穿全领域） | docker-sdk |
| `api/http/proxy/manager.go` | 代理实例缓存管理 | docker-sdk |
| `api/http/proxy/factory/` | 代理工厂（docker/kubernetes/azure 分流） | docker-sdk |
| `api/http/proxy/factory/docker/` | Docker 代理拦截器（RBAC/RC/改写核心） | docker-sdk（总览）+ 各领域 |
| `api/docker/container.go` | ContainerService（容器重建等业务） | container |
| `api/docker/images/` | 镜像解析/拉取/registry 匹配/状态检测 | image |
| `api/docker/stats/container_stats.go` | 容器状态统计 | container |
| `pkg/snapshot/docker.go` | 快照 SDK 采集器 | docker-sdk |
| `api/http/handler/docker/` | 原生 Docker handler（dashboard/containers/images） | container / image |
| `api/http/handler/stacks/` | 堆栈 HTTP handler | compose |
| `api/exec/compose_stack.go` | ComposeStackManager | compose |
| `api/exec/swarm_stack.go` | SwarmStackManager | compose |
| `pkg/libstack/` | compose v2 / swarm 的进程内部署封装 | compose |
| `api/stacks/stackutils/` | compose 文件校验、env 解析、路径工具 | compose |

## 全领域公共机制

以下机制被多个领域文档引用，统一在此说明：

### 1. 资源控制（Resource Control，简称 RC）

Portainer 的 RBAC 核心数据模型（定义于 `api/portainer.go` 的 `ResourceControl` struct）。代理层对每个资源操作做三类处理：

- **装饰（decorate）**：管理员看到全部资源，每个对象附加 `Portainer.ResourceControl` 字段；
- **过滤（filter）**：普通用户只保留有权限的资源；
- **受限操作（restricted）**：单资源写操作（启动/停止/删除/连接网络等）先查 RC，无权限返回 403。

查找优先级（`api/http/proxy/factory/docker/access_control.go:281-321` 的 `findResourceControl`）：

```
资源自身 ID → Swarm service 标签继承 → Swarm stack 标签继承
→ Compose stack 标签继承 → io.portainer.accesscontrol.* 标签动态创建
```

### 2. 标签常量（`api/docker/consts/labels.go:4-7`）

```go
ComposeStackNameLabel = "com.docker.compose.project"   // Compose 栈识别标签
SwarmStackNameLabel   = "com.docker.stack.namespace"   // Swarm 栈识别标签
SwarmServiceIDLabel   = "com.docker.swarm.service.id"  // Swarm 服务标签
SwarmNodeIDLabel      = "com.docker.swarm.node.id"     // Swarm 节点标签
```

这些标签贯穿：栈名查重、RC 继承、代理列表过滤、快照统计、镜像状态缓存失效。

### 3. Agent / Edge Agent 通信头

Agent 环境下所有请求（代理路径与 SDK 路径一致）注入三个头（`api/portainer.go` 中的常量）：

- `X-PortainerAgent-Public-Key`：Portainer 服务端公钥；
- `X-PortainerAgent-Signature`：消息签名，Agent 校验来源合法性；
- `X-PortainerAgent-Target`（可选）：Swarm 集群中定向到指定节点名。
