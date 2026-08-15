# 后端总体架构

> 行号基线：分支 `docs` / commit `269738a01`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

## 1. 入口与启动流程

唯一入口：`api/cmd/portainer/main.go`（738 行）。

### 1.1 main() 主循环（`api/cmd/portainer/main.go:675-702`）

```
main() → logs.ConfigureLogger → initCLI → for { buildServer → server.Start }
```

`for` 循环意味着 HTTP server 意外退出后会自动重建（正常关机靠 `shutdownCtx` 取消）。`server.Start` 返回即进入下一轮 `buildServer`。

### 1.2 buildServer() 初始化顺序（`api/cmd/portainer/main.go:374-673`）

按执行顺序的关键步骤（省略次要赋值）：

| 步骤 | 代码位置 | 说明 |
|---|---|---|
| 1. feature flags 解析 | `main.go:375-377` | `pkg/featureflags/featureflags.go:9` 定义支持列表 |
| 2. FIPS/SSRF 防护 | `main.go:391-422` | CE 强制非 FIPS；SSRF AllowList 来自 DB 并包装全局 `http.DefaultTransport`；go-git 的 http/ssh 传输替换为 SSRF 安全版、禁用 file 协议（`api/git/ssrf_transport.go`） |
| 3. 数据库 | `main.go:400` → `initDataStore`（`main.go:96-171`） | boltdb 打开 → `store.Init()`；新库写入 Version（SchemaVersion/Edition/InstanceID），旧库走 `MigrateData()` |
| 4. 版本一致性检查 | `main.go:407` | DB schema 与 server 版本不匹配直接 Fatal |
| 5. 认证类服务 | `main.go:429-449` | APIKey、JWT（会话超时可配）、LDAP、OAuth、Git、ECDSA 数字签名 |
| 6. Edge/SSL/密钥对 | `main.go:451-465` | EdgeStacks 服务、SSL 服务（证书动态重载）、签名密钥对（存在则加载，否则生成并落盘） |
| 7. 隧道与客户端工厂 | `main.go:467-478` | chisel 反向隧道服务、Docker 客户端工厂、K8s 客户端工厂、授权服务、K8s token 缓存 |
| 8. 部署器 | `main.go:487-493` | Compose（进程内 compose v2）、Swarm、Kubernetes（libkubectl）三套部署器 |
| 9. PendingActions | `main.go:495-498` | 注册 3 个处理器：清 NAP 策略、删 K8s registry secrets、PostInit 环境 |
| 10. 快照服务 | `main.go:500-505` | 创建并立即启动循环（详见 §5） |
| 11. 代理管理器 | `main.go:507` | 注入全部依赖到 proxy factory |
| 12. 本地环境初始化 | `main.go:516` | goroutine 等待 admin 创建信号后注册本地 Docker/K8s 环境（`api/internal/endpointutils/endpoint_setup.go:25`） |
| 13. 管理员密码/setup token | `main.go:518-569` | `--admin-password` flag 或 setup token 初始化首个管理员 |
| 14. 隧道服务器 | `main.go:571-573` | chisel server 监听 `0.0.0.0:8000`（默认，`api/cli/defaults.go:8-9`） |
| 15. 调度器 | `main.go:575-588` | GitOps SourceScheduler `ReconcileAll`；Swarm 栈状态对账每分钟跑 |
| 16. 升级/PostInit 迁移 | `main.go:597-623` | 升级服务（DB 版本变更时自动升级镜像）、PostInit 迁移器 |
| 17. 恢复中断部署 | `main.go:625-630` | 把重启时卡在 Deploying 状态的栈重置为 Error（`recoverStaleDeployingStacks`，`main.go:707-738`） |
| 18. 组装 http.Server | `main.go:632-672` | 全部服务作为 struct 字段注入 |

### 1.3 默认端口

`api/cli/defaults.go:5-23`（Windows 版在 `defaults_windows.go`）：

| 用途 | 默认值 | flag |
|---|---|---|
| HTTP API | `:9000` | `--bind` |
| HTTPS API | `:9443` | `--bind-https` |
| Edge 隧道服务器 | `0.0.0.0:8000` | `--tunnel-addr` / `--tunnel-port` |
| 数据目录 | `/data` | `--data` |

## 2. 目录职责清单

### 2.1 `api/` 顶层

| 目录 | 职责 | 代表文件 |
|---|---|---|
| `api/adminmonitor/` | 实例未初始化管理员超时监控（默认 5 分钟后禁用实例） | `admin_monitor.go:28` |
| `api/agent/` | 探测 Agent 版本/平台的客户端辅助 | `version.go:18` |
| `api/apikey/` | API Key 服务（哈希存储 + LRU 缓存） | `service.go` |
| `api/backup/` | 备份归档（可加密）与恢复 | `backup.go:38` |
| `api/chisel/` | Edge 反向隧道服务器（封装 jpillora/chisel） | `service.go:150` |
| `api/cli/` | CLI flags 与默认值 | `cli.go:28` |
| `api/crypto/` | ECDSA 签名、AES、TLS 配置 | `ecdsa.go:23` |
| `api/database/` | 数据库连接工厂（仅支持 boltdb）+ Version 模型 | `database.go:11` |
| `api/dataservices/` | 数据服务接口 + 每实体一个子包 | `interface.go:41` |
| `api/datastore/` | Store 门面：初始化、备份、迁移编排 | `migrate_data.go:19` |
| `api/docker/` | Docker 客户端工厂、快照、容器/镜像/统计服务 | `client/client.go:50` |
| `api/exec/` | 栈部署器实现（compose/swarm/k8s） | `kubernetes_deploy.go:105` |
| `api/filesystem/` | FileService：栈文件、TLS、密钥对落盘 | `filesystem.go:105` |
| `api/git/` | Git 服务（go-git）+ SSRF 安全传输 | `ssrf_transport.go` |
| `api/gitops/` | GitOps Source 轮询调度（新功能） | `scheduling/scheduler.go:41` |
| `api/http/` | HTTP 服务器、路由、中间件、代理、安全 | `server.go:121` |
| `api/jwt/` | JWT 签发校验 + kubeconfig token | `jwt.go:51` |
| `api/kubernetes/` | K8s 快照、kubeconfig 访问、客户端工厂 | `cli/client.go:56` |
| `api/ldap/`、`api/oauth/` | 外部认证 | `ldap.go:72`、`oauth.go:33` |
| `api/pendingactions/` | 环境不可用时暂存的操作 | `pendingactions.go:54` |
| `api/scheduler/` | robfig/cron 封装 | `scheduler.go:32` |
| `api/stacks/` | 栈部署/回滚/拆除编排 | `deployments/deploy.go:39` |
| `api/uac/` | 基于资源控制的列表过滤（读路径 RBAC） | `uac.go:13` |
| `api/ws/` | WebSocket 底层（hijack 双向流） | `hijack.go:26` |
| `api/portainer.go` | 核心模型（2787 行）：Endpoint/Stack/User/Role/ResourceControl 等全部在此 | `Endpoint:449` |

### 2.2 `api/internal/`（真正的 internal，仅 10 包）

| 包 | 职责 |
|---|---|
| `authorization/` | RBAC 默认角色授权矩阵（4 类默认角色）与动态计算（`authorizations.go:25`） |
| `edge/` | Edge 组/栈工具与缓存：`edgestacks/service.go:34`（BuildEdgeStack）、`endpoint.go:32`（EffectiveCheckinInterval） |
| `endpointutils/` | 环境类型判断、本地环境初始化（`endpoint_setup.go:25`） |
| `snapshot/` | 快照循环（§5） |
| `ssl/` | 证书生成/重载 |
| `upgrade/` | 实例自动升级（docker/k8s 双实现） |
| `nodes/`、`randomstring/`、`registryutils/`、`testhelpers/` | 工具包 |

### 2.3 `pkg/`（可复用库，不含业务模型）

`libhelm`（Helm SDK，CE 内置非 EE 专属）、`libkubectl`（kubectl 语义：apply/delete/drain/restart/describe）、`libstack`（compose v2 / swarm 进程内部署）、`libhttp`（request/response/error 封装 + SSRF）、`libcrypto`、`libkompose`、`liboras`、`featureflags`、`fips`、`snapshot`（快照采集器接口与 Docker/K8s 实现）、`schedule`、`registryhttp`、`networking`、`validate`、`retry`、`build`（构建元数据注入）、`metrics`、`libpolicy`、`libprometheus`、`librand`、`endpoints`、`edge`、`authorization`、`testhelpers`。

## 3. HTTP API 层

### 3.1 两级路由

- **顶层自研前缀分发**：`api/http/handler/handler.go:205-292` 的 `ServeHTTP` 按 URL 前缀 switch，`http.StripPrefix("/api", ...)` 后转交子 handler。特殊优先级：`/api/endpoints` 且路径含 `/edge/` 优先走 EdgeHandler（`handler.go:207-208`）；含 `/docker/`、`/kubernetes/`、`/azure/`、`/agent/` 的走 EndpointProxyHandler（`handler.go:234-246`）；兜底 `case "/"` 是静态文件（`handler.go:289-290`）。
- **子 handler 内 gorilla/mux**：每个子包的 `handler.go` 用 `NewHandler` 集中注册，REST 风格 + `.Methods(...)`。例：`api/http/handler/endpoints/handler.go:52-91`。

### 3.2 中间件链（全局）

`api/http/server.go:342-351` 按由内到外的顺序包裹：

```
handler.Handler → adminMonitor.WithRedirect → offlineGate.WaitingMiddleware(1min)
→ WithPanicLogger → WithSlowRequestsLogger → csrf.WithProtect
```

| 中间件 | 位置 | 职责 |
|---|---|---|
| adminMonitor | `api/adminmonitor/admin_monitor.go:106` | 实例未初始化超过 5 分钟（`server.go:143`）时所有请求重定向到初始化页 |
| offlineGate | `api/http/offlinegate/offlinegate.go:14` | 备份期间阻断 API，最多等 1 分钟 |
| panic/slow logger | `api/http/middlewares/` | recover 记日志、慢请求告警 |
| CSRF | `api/http/csrf/csrf.go:13` | 基于 TrustedOrigins；cookie 会话的写请求校验 Origin |

### 3.3 认证与授权（RequestBouncer）

`api/http/security/bouncer.go`，在 `server.go:124` 创建：

- **访问级别包装器**（每个路由选用其一）：`PublicAccess:95`、`AdminAccess:105`、`RestrictedAccess:118`、`TeamLeaderAccess:132`、`AuthenticatedAccess:145`、`EdgeComputeOperation:563`。
- **认证链**（`mwAuthenticatedUser:223-232`）：token 查找顺序 `X-API-KEY` 头 → cookie → Bearer JWT；API Key 与 Bearer 同时出现拒绝（`bouncer.go:313-317`）。
- **环境级校验**：`AuthorizedEndpointOperation:156`、Edge 环境 `AuthorizedEdgeEndpointOperation:184`（校验 `X-PortainerAgent-EdgeID` 与库中一致）、首次信任 `TrustedEdgeEnvironmentAccess:203`。
- **登录限流**：`security.NewRateLimiter(10, 1s, 1h)`（`server.go:129`），作用于 `/auth` 与 `/auth/oauth/validate`。
- **mux 中间件**：`WithEndpoint`（环境注入 context，`api/http/middlewares/endpoint.go:27`）、`CheckEndpointAuthorization`、泛型 `WithItem`、`Deprecated`、featureflag 开关。

### 3.4 直通代理（`/api/endpoints/{id}/...`）

`api/http/handler/endpointproxy/handler.go:27-37` 把 docker/kubernetes/azure/agent 四类子路径转交 `proxy.Manager`（按 endpointID 缓存代理实例）。代理工厂按环境类型分流（`api/http/proxy/factory/factory.go:47-59`）：

- `AzureEnvironment` → azure 代理；
- 三种 K8s 类型 → kubernetes 代理（agent/edge/local 三种 transport）;
- 其余（Docker 全家族）→ docker 代理（transport 层做 RBAC 资源过滤，详见 docker-infrastructure 文档）。

另有 `NewGitlabProxy`（`factory.go:62-64`）供 GitLab API 代理。

### 3.5 WebSocket

`api/http/handler/websocket/handler.go:36-43`：`/websocket/exec|attach|pod|kubernetes-shell`，均 AuthenticatedAccess。底层 hijack 双向流在 `api/ws/hijack.go:26`；对 Agent 的请求注入签名头（`websocket/proxy.go:77-84`）。

## 4. 数据层

### 4.1 存储引擎与连接

- 仅支持 boltdb：`api/database/database.go:11-20`。
- 连接层 `api/database/boltdb/db.go`：可选 AES 加密（密钥来自 `/run/secrets/<name>` 的 SHA-256，`api/cmd/portainer/main.go:348-372`）、`UpdateTx:215`/`ViewTx:224` 事务、通用对象 CRUD（`CreateObject:385` 等）。

### 4.2 Store 与数据服务

- `api/datastore/datastore.go:19` `NewStore`；`services.go:88` `initServices()` 构造 30+ bucket 服务（每实体一个 `api/dataservices/<entity>/` 子包）。
- 事务边界：`dataStore.UpdateTx(fn)` / `ViewTx(fn)`（`datastore.go:79-89`）。
- 接口：`api/dataservices/interface.go:9-55`（`DataStoreTx` 与 `DataStore`）。

### 4.3 迁移机制

| 环节 | 位置 | 说明 |
|---|---|---|
| 入口 | `api/datastore/migrate_data.go:19-61` | `MigrateData()`：防重入检查 → 全量备份 → `FailSafeMigrate` → 失败恢复备份 |
| 版本注册表 | `api/datastore/migrator/migrator.go:202-288` | `initMigrations()` 按 semver 注册版本链（1.21 → 2.45.0），每版本一个 `migrate_*.go` 文件 |
| 同版本重跑 | `migrator.go:176-185` | 同一 schema 版本内新增迁移函数按 `MigratorCount` 计数全部重跑（注释 `migrator.go:187-193`） |
| 回滚 | `main.go:117-124` + `migrate_data.go:138-151` | `--rollback` flag 恢复上一份备份 |
| PostInit 迁移 | `api/datastore/postinit/migrate_post_init.go:49` | 需要 Docker/K8s 客户端的迁移，经 PendingActions 在环境可用后执行 |

## 5. 后台任务一览

无统一的 controller 目录，各服务自带循环：

| 任务 | 位置 | 触发方式 |
|---|---|---|
| 环境快照 | `api/internal/snapshot/snapshot.go:212` `startSnapshotLoop` | ticker，间隔来自 flag/settings（默认 5m，`api/portainer.go:2152-2153`）；每轮对非 Edge/Azure 环境采集并更新 Up/Down；环境恢复时顺带执行 pending actions |
| Edge 缺失快照补打 | `api/internal/snapshot/snapshot.go:55-82` | HTTPS server 启动前，对缺快照的 Edge 环境开隧道补采 |
| Swarm 栈状态对账 | `main.go:586-588` → `deployments.ReconcileSwarmStackStatus` | 每分钟 |
| GitOps Source 轮询 | `api/gitops/scheduling/scheduler.go:41` | 每个 Source 按自身 `Interval` 建 cron job；变更时 `RedeployWhenChanged`（`api/stacks/deployments/deploy.go:39`）自动重部署 |
| chisel 隧道巡检 | `api/chisel/service.go:234-247` | 10s 间隔；空闲超 4m30s 的隧道先打快照再关闭 |
| 管理员监控 | `api/adminmonitor/admin_monitor.go:37` | 5 分钟超时 |
| MOTD 拉取 | `api/motd/service.go`（`server.go:221-222` 启动） | 定期 |
| JWT 撤销表清理 | `bouncer.go:83` | 每小时 |
| 本地环境初始化 | `main.go:516` | 一次性 goroutine，等 admin 创建信号 |

## 6. Kubernetes 攻面

| 能力 | 位置 |
|---|---|
| 客户端工厂 | `api/kubernetes/cli/client.go:56` `NewClientFactory`（依赖签名、隧道、dataStore）；`GetPrivilegedKubeClient:115`、`CreateKubeClientFromKubeConfig:171` |
| KubeClient 资源操作 | `api/kubernetes/cli/` 按资源分文件（deployment/namespace/secret/pod/ingress/rbac 等），接口在 `api/portainer.go:1883` |
| 集群内落地资源 | `api/kubernetes/cli/naming.go:7-18`：namespace `portainer`、ConfigMap `portainer-config`、SA `portainer-sa-clusteradmin` 与 `portainer-sa-user-<instanceID>-<userID>`、kubectl shell pod 前缀 |
| Namespace 访问策略 | 存 `portainer-config` ConfigMap（`cli/access.go:36`） |
| HTTP 层 | 原生 API `/api/kubernetes/**`（`handler/kubernetes/handler.go:35-169`，约 120 条路由）；代理 `/api/endpoints/{id}/kubernetes/**` + 按用户 RBAC token 替换（`proxy/factory/kubernetes/token.go:24`） |
| 清单部署 | `api/exec/kubernetes_deploy.go:105` 用 `pkg/libkubectl`（apply/delete/drain/restart） |
| Helm | **CE 内置**：`pkg/libhelm/manager.go:9` 以库方式用 helm v4（`go.mod:65`），路由 `handler/helm/handler.go:43-87` |
| 快照 | `api/kubernetes/snapshot.go` |
| kubeconfig 下载 | `api/kubernetes/kubeclusteraccess_service.go` + `api/jwt/jwt_kubeconfig.go` |

## 7. 横切机制

| 机制 | 位置 | 说明 |
|---|---|---|
| SSRF 防护 | `pkg/libhttp/ssrf` + `main.go:411-422` | AllowList 来自 DB；包装全局 transport 与 go-git 传输 |
| 文件存储 | `api/filesystem/filesystem.go:105` | 栈文件按版本落盘（`GetStackProjectPathByVersion:157`）、TLS 证书、签名密钥对、Edge 栈目录；`JoinPaths:87` 防路径穿越 |
| 文件上传 | `api/http/handler/upload/handler.go:24-26` | 仅 TLS 证书上传（`POST /upload/tls/{ca|cert|key}`，AdminAccess） |
| 备份/恢复 | `api/backup/backup.go:38` | 加密归档；HTTP 入口 `handler/backup/` |
| 数字签名 | `api/crypto/ecdsa.go:23-39` | Server→Agent 信任：固定消息 `"Portainer-App"` 签名；设 `AGENT_SECRET` 时直接签 secret |
| RBAC 读过滤 | `api/uac/uac.go:13` | 列表接口按 ResourceControl 过滤（直接/继承/label 三类） |
| 日志 | zerolog，`api/logs/log.go`；`main.go:676-682` 配置 | |
| Feature flags | `pkg/featureflags/featureflags.go:9,45` | |
| 平台升级 | `api/internal/upgrade/upgrade.go:64` | DB 版本变更触发 docker/k8s 镜像自动升级 |
| OpenAPI 文档 | `handler.go:81-202`（swag 注解）→ `Makefile:129-130` 生成 `api/docs/swagger.yaml` / `openapi.yaml` | 前端类型生成的输入（见 frontend.md §7） |
