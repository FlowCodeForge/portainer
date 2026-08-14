# Compose（堆栈）操作

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

Portainer 的 Compose 能力构建在 **Stack（堆栈）** 体系之上：一个 Stack 是"一份 compose 文件 + 环境变量 + 部署状态"的 Portainer 数据库实体，部署执行则委托给 **docker compose v2 的进程内 Go API**（`github.com/docker/compose/v2` 的 `compose.NewComposeService`），**不是**调用 docker-compose 二进制。Swarm 栈另有一条独立链路（复刻 `docker stack deploy`）。

---

## 1. 核心类型定义（`api/portainer.go`）

### 1.1 Stack struct（1269-1326 行）

| 字段 | 行号 | 说明 |
|---|---|---|
| `Type StackType` | 1275 | 1=Swarm、2=Compose、3=Kubernetes |
| `SwarmID` | 1279 | Swarm 栈的集群 ID；Compose 栈为空 |
| `EntryPoint` | 1284 | compose 文件相对路径 |
| `Env` | 1286 | 界面配置的环境变量（`[]portainer.Pair`） |
| `Status` / `DeploymentStartStatus` | 1290-1294 | 部署状态机 |
| `ProjectPath` | 1296 | 磁盘上栈文件目录（Portainer 数据目录下） |
| `AdditionalFiles` | 1306 | 额外 compose 文件（override） |
| `AutoUpdate` | 1308 | Git webhook / polling 自动更新配置 |
| `Namespace` | 1322 | K8s 栈专用 |
| `DeploymentStatus` | 1325 | 最近一次部署的结果 |

### 1.2 类型与状态常量

| 常量 | 行号 | 值 |
|---|---|---|
| `DockerSwarmStack` | 2422-2430 | 1（`docker stack`） |
| `DockerComposeStack` | 2422-2430 | 2（`docker-compose`） |
| `KubernetesStack` | 2422-2430 | 3 |
| 状态常量 | 2433-2437 | Active=1 / Inactive=2 / Deploying=3 / Error=4 |

### 1.3 类型区分的运行时处理

每个操作 handler 内 `switch stack.Type` 分发：Swarm 栈走 `SwarmStackManager`，Compose 栈走 `ComposeStackManager`，K8s 栈在 start/stop/migrate 中直接拒绝（`stack_start.go:57-59`、`stack_stop.go:54-56`、`stack_migrate.go:73-75`）。

栈在 Docker 侧通过标签识别（`api/docker/consts/labels.go:4-5`）：Compose 栈 `com.docker.compose.project`、Swarm 栈 `com.docker.stack.namespace`——用于栈名查重、RC 继承、代理过滤。

---

## 2. HTTP Handler（`api/http/handler/stacks/`）

### 2.1 路由表（`handler.go:57-96` 的 `NewHandler`）

| 端点 | 方法 | 处理函数 | 行号 |
|---|---|---|---|
| `/stacks/create/{type}/{method}` | POST | `stackCreate` | 66-67 |
| `/stacks` | GET | `stackList` | 68-69 |
| `/stacks/{id}` | GET / DELETE / PUT | `stackInspect` / `stackDelete` / `stackUpdate` | 70-79 |
| `/stacks/{id}/associate` | PUT（Admin） | `stackAssociate` | 74-75 |
| `/stacks/name/{name}` | DELETE | `stackDeleteKubernetesByName` | 76-77 |
| `/stacks/{id}/git`、`/git/redeploy` | POST/PUT | `stackUpdateGit` / `stackGitRedeploy` | 80-83 |
| `/stacks/{id}/file` | GET | `stackFile` | 84-85 |
| `/stacks/{id}/migrate` | POST | `stackMigrate` | 86-87 |
| `/stacks/{id}/start`、`/stop` | POST | `stackStart` / `stackStop` | 88-91 |
| `/stacks/webhooks/{webhookID}` | POST（公开） | `webhookInvoke` | 92-93 |

### 2.2 生命周期 handler 详解

| 函数 | 文件:行号 | 逻辑说明 |
|---|---|---|
| `stackCreate` | `stack_create.go:19-76` | 按 `{type}` 分发 swarm/standalone/kubernetes（66-73）；先 `userCanManageStacks`（handler.go:134-151，端点 `AllowStackManagementForRegularUsers=false` 时非管理员拒绝）+ `AuthorizedEndpointOperation`；`createComposeStack`（78-89）再按 `{method}` 分发 string/repository/file；`decorateStackResponse`（117-156）——管理员建 AdministratorsOnly RC，普通用户建私有 RC（126-129），RC ID 为 `{endpointID}_{stackName}`（`stackutils.ResourceControlID`，`api/stacks/stackutils/util.go:35-37`） |
| `createComposeStackFromFileContent` | `create_compose_stack.go:147-178` | 规范化栈名（154）→ `ensureUniqueComposeStackName`（100-130，按容器 `com.docker.compose.project` 标签查重；Swarm 环境查 service 的 `com.docker.stack.namespace`，实现于 `handler.go:168-211` `checkUniqueStackNameInDocker`）→ `stackbuilders.BuildAndAsyncDeploy`（172） |
| `createComposeStackFromGitRepository` | `create_compose_stack.go:263` 起 | 默认 compose 文件名 `filesystem.ComposeFileDefaultName`（271-273）；webhook ID 唯一性检查（280-288） |
| `stackStart` | `stack_start.go:39-162` | 状态机检查（Active/Deploying/Error 返回 409，63-78）→ Docker 内查重（106-113）→ RC 鉴权 → `startStack`（164-203）：Compose 栈 `DeployComposeStack`（191）、Swarm 栈 `DeploySwarmStack`（199）；失败置 `StackStatusError` 并追加 `DeploymentStatus`（128-141），成功置 Active（144-147） |
| `stackStop` + `stopStack` | `stack_stop.go:36-131 + 133-154` | Compose 走 `UndeployComposeStack`（142，即 compose down）；Swarm 走 `SwarmStackManager.Remove`（150） |
| `stackUpdate` | `stack_update.go:92-155` | DB 事务内 `updateStackInTx`（157-237，Deploying 状态返回 409；RC 鉴权 190-201）；K8s 栈内联部署（137-146）；Docker 栈异步 `go stackDeploy(...)`（152）；`updateAndDeployStack`（239-251）分发到 `updateSwarmStack`（336）/ `updateComposeStack`（256）/ `updateKubernetesStack` |
| `stackDelete` | `stack_delete.go:39-145` | 支持 `?external=true`（147-186，管理员专用，删除不在 DB 中的外部 Swarm 栈）；正常删除按序调 teardown 三步（115/131/136，见 §8） |
| `stackMigrate` | `stack_migrate.go:55-188` | 校验后把 stack 的 EndpointID/SwarmID/Name 指向目标环境（126-141）→ 目标环境查重（143-151）→ `migrateStack`（190-195）在目标端点部署（`CreateComposeStackDeploymentConfigTx`/`CreateSwarmStackDeploymentConfigTx` + `.Deploy`，197-250）→ 成功后清理原环境（160）并更新 RC 的 ResourceID（169-175） |
| `webhookInvoke` | `webhook_invoke.go:27-57` | 公开端点（无认证，靠 webhookID 不可猜）；Deploying 时 409；调 `deployments.RedeployWhenChanged`（47，见 §9） |
| `stackFile` | `stack_file.go:41-124` | 读 compose 文件内容（118，`FileService.GetFileContent`）；git 栈若 URL/文件路径已改但未 redeploy 则 409（114-116、129-136 `gitStackPendingRedeploy`） |

### 2.3 创建流程的构建器模式（`api/stacks/stackbuilders/`）

| 函数/类型 | 文件:行号 | 逻辑说明 |
|---|---|---|
| `stackBuildProcess` 接口 | `director.go:16-27` | 构建器抽象 |
| `BuildAndAsyncDeploy` | `director.go:37-54` | 先存 DB（Status=Deploying）再 goroutine 异步部署，15 分钟超时；`deploy`（106-146）完成后更新最终状态 |
| `BuildAndDeploy` | `director.go:70-104` | 同步部署（K8s 栈内联部署用） |
| `initComposeDeployment` / `initSwarmDeployment` | `stack_builder.go:118-127 / 129-138` | 创建对应 DeploymentConfig |

---

## 3. Compose 部署调用链（四层）

```
HTTP handler（§2）
   │
   ▼
层1 门面：api/stacks/deployments/deployer.go（StackDeployer）
   │
   ▼
层2 管理器：api/exec/compose_stack.go（ComposeStackManager）
   │
   ▼
层3 部署器：pkg/libstack/compose/（ComposeDeployer，docker compose v2 进程内 API）
   │           ├─ pkg/libstack/dockercli.go（WithCli：构造 command.DockerCli）
   │           └─ composeplugin.go createProject（compose-go 解析）
   ▼
Docker Daemon（经 Portainer 代理 URL）
```

### 3.1 层 1：StackDeployer 门面（`api/stacks/deployments/deployer.go`）

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `BaseStackDeployer` 接口 | 14-19 | 部署动作抽象 |
| `StackDeployer` | 21-24 | 含 Remote（EE 用） |
| `DeployComposeStack` | 56-74 | 可选先 `composeStackManager.Pull`（63-67，forcePullImage 时）再 `composeStackManager.Up`（69-73，带 ForceRecreate/Prune）；全程互斥锁（57-58） |
| `UndeployComposeStack` | 76-81 | → `Down` |
| `DeploySwarmStack` | 49-54 | → `swarmStackManager.Deploy` |
| `DeployKubernetesStack` | 83-106 | K8s 分支 |

### 3.2 层 2：ComposeStackManager（`api/exec/compose_stack.go`，216 行）

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `ComposeStackManager` struct | 18-21 | 字段：`libstack.Deployer` + `proxy.Manager` |
| `NewComposeStackManager` | 24-29 | 构造器。接口定义在 `api/portainer.go:1739-1746`（Up/Down/Pull/Run/NormalizeStackName/ComposeSyntaxMaxVersion） |
| `ComposeSyntaxMaxVersion` | 32-34 | 返回支持的 compose 语法最高版本 |
| `Up` | 37-68 | ① `fetchEndpointProxy` 取**指向该环境的本地代理 URL**（38）——compose 部署流量仍经过 Portainer 的代理与鉴权层；② `createEnvFile` 生成合并 env 文件（47）；③ `stackutils.GetStackFilePaths`（52，`util.go:21-32`，EntryPoint + AdditionalFiles 拼绝对路径）；④ `deployer.Deploy`（53-64）：WorkingDir=ProjectPath、EnvFilePath、Host=代理 URL、ProjectName=stack.Name、Registries=registry 凭证、ForceRecreate/AbortOnContainerExit/RemoveOrphans 选项 |
| `Run` | 71-102 | 一次性任务（`docker compose run`），调 `deployer.Run` |
| `Down` | 105-122 | `deployer.Remove`（按 project name，`RemoveOrphans: true`） |
| `Pull` | 126-150 | `deployer.Pull`（仅拉镜像不启动） |
| `NormalizeStackName` | 153-155 | 替换栈名中不支持的字符 |
| `createEnvFile` | 159-183 | 见 §5 |
| `copyDefaultEnvFile` | 186-204 | 拷贝 compose 文件同目录的默认 `.env`（打不开则静默跳过） |
| `copyConfigEnvVars` | 207-215 | 追加界面配置的环境变量 |

### 3.3 层 3：ComposeDeployer（`pkg/libstack/compose/`）

#### compose.go

| 函数/类型 | 行号 | 逻辑说明 |
|---|---|---|
| `ComposeDeployer` struct | 9-18 | 持有 `createComposeServiceFn`（默认 `compose.NewComposeService`，来自 `github.com/docker/compose/v2/pkg/compose`） |

#### composeplugin.go

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `withComposeService` | 38-72 | 通用骨架：`libstack.WithCli` 创建 DockerCli → 构造 compose service → `createProject` 加载项目 → 支持 `COMPOSE_PARALLEL_LIMIT` 环境变量设置并发上限（58-68）→ 执行回调 |
| `Deploy` | 75-114 | `addServiceLabels`（254-276，打 `com.docker.compose.project` 等标签 + Edge 栈标签 `io.portainer.edge_stack_id`）→ `WithoutUnnecessaryResources` → 组装 `api.UpOptions`（ForceRecreate/RemoveOrphans/IgnoreOrphans 从 options 与 `COMPOSE_REMOVE_ORPHANS`/`COMPOSE_IGNORE_ORPHANS` 环境变量读取，86-96）→ `composeService.Build`（102）→ `composeService.Up`（106） |
| `Run` | 117-151 | `RunOneOffContainer` 一次性运行服务（先把目标服务移入 DisabledServices 再 Create，129-134） |
| `Remove` | 154-168 | `composeService.Down` |
| `Pull` | 171-181 | `composeService.Pull` |
| `Config` | 191-207 | `project.MarshalYAML` 输出解析后的 compose 配置 |
| `GetExistingEdgeStacks` | 209-252 | 按 Edge 标签列出已部署 Edge 栈 |
| `createProject` | 278-359 | **compose-go 解析核心**，见 §6 |

#### dockercli.go（`WithCli`，32-100 行）

| 步骤 | 行号 | 逻辑说明 |
|---|---|---|
| 创建 CLI | 39-50 | `command.NewDockerCli` + `Initialize`（**全局互斥锁串行化**——docker cli 全局状态非并发安全） |
| 注入头 | 58-64 | 自定义 HTTP 头（Swarm 场景 `X-PortainerAgent-ManagerOperation`） |
| registry 凭据 | 68-74 | 把 Portainer registry 凭证写入 `cli.ConfigFile().AuthConfigs` |
| 凭据隔离 | 95-97 | 有内联凭证时清空全局 credsStore，防止 Docker Desktop 凭证存储干扰 |

---

## 4. 部署时序（Compose 栈 `Up` 全景）

```
stackUpdate / stackStart (handler)
  → StackDeployer.DeployComposeStack (deployer.go:56)
  → ComposeStackManager.Up (compose_stack.go:37)
      ├─ fetchEndpointProxy → 代理 URL（部署流量过 Portainer 鉴权层）
      ├─ createEnvFile → stack.env（.env + 界面变量合并）
      └─ ComposeDeployer.Deploy (composeplugin.go:75)
           ├─ WithCli → DockerCli（注入 registry 凭据）
           ├─ createProject → compose-go 解析（§6）
           ├─ addServiceLabels（栈识别标签）
           ├─ composeService.Build（有 build 段的服务）
           └─ composeService.Up（创建网络/卷/容器并启动）
```

---

## 5. 环境变量体系（.env 与界面变量合并）

**优先级**（`createEnvFile`，compose_stack.go:159-183）：**界面 env > .env 文件**。

```
① stack.Env 为空 → 返回 ""（不生成 env 文件，compose 自行读 .env）
② 在 ProjectPath 下写 stack.env（0600 权限）
③ copyDefaultEnvFile（186-204）：拷贝 compose 文件同目录的 .env 内容
④ copyConfigEnvVars（207-215）：追加界面配置的 stack.Env
   —— 后写入覆盖先写入，实现"界面变量优先"
⑤ compose-go 加载时 WithEnvFiles(stack.env)（composeplugin.go:309）
```

校验期与部署期的一致性由 `api/stacks/stackutils/env.go` 的 `BuildEnvMap`（23-58 行）保证：优先级 OS env < `.env` 文件（`dotenv.Read`，31 行）< stack.Env（经 `dotenv.ParseWithLookup` 按 dotenv 规则解析，47-50 行）。

---

## 6. compose 文件解析（`createProject`，composeplugin.go:278-359）

| 步骤 | 行号 | 逻辑说明 |
|---|---|---|
| 工作目录 | 279-287 | 取第一个 compose 文件所在目录；`options.ProjectDir` 非空时覆盖（相对路径基准） |
| env 文件 | 289-292 | `EnvFilePath`（stack.env）作为 env file 传入 |
| COMPOSE_ 变量 | 294-299 | 收集进程环境中所有 `COMPOSE_` 前缀变量 |
| ProjectOptions | 301-323 | `cli.NewProjectOptions` 组合选项：`WithWorkingDirectory`、`WithName(ProjectName)`、`WithoutEnvironmentResolution`（延迟插值）、`WithResolvedPaths`（除非 `--no-path-resolution`）、`WithEnv(PortainerEnvVars() + composeEnvVars + options.Env)`、`WithEnvFiles`、`COMPOSE_ENV_FILES` 环境变量支持（310-319）、`WithDotEnv`、`WithDefaultProfiles`、`WithConfigFileEnv` |
| 加载 | 328 | `projectOptions.LoadProject(ctx)` —— compose-go 完整解析/合并多个 compose 文件/插值 |
| 相对路径修正 | 333-340 | service 的 `env_file` 相对路径补全为绝对路径（compose 路径处理 workaround） |
| 环境插值 | 343-347 | `WithServicesEnvironmentResolved(true)` 解析服务级环境变量 |
| bind-mount hash | 349-356 | 可选为每个 service 加 bind mount 内容 hash 标签（检测 bind 挂载文件变更） |

---

## 7. Swarm 栈部署（`pkg/libstack/swarm/swarm.go`）

Swarm 栈**不用 compose v2 库**，而是复刻 `docker stack deploy` 的逻辑：

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `SwarmDeployer` | 86-89 | 部署器类型 |
| `deployStack` | 179-253 | `Info` 校验 manager（180-187）→ `getConfig`（639-679，用 docker/cli 的 `composeloader.Load` 解析 + 校验镜像引用）→ 可选 prune 孤儿 service（197-207）→ `convert.Networks` + `validateExternalNetworks`（209-213，external 网络必须是 swarm scope，`NetworkInspect` 298-318）→ `createNetworks`（215，默认 overlay 驱动，320-353）→ secrets/configs 创建或更新（219-235）→ `convert.Services` + `deployServices`（237-252，SDK `ServiceCreate`/`ServiceUpdate`，`ForceUpdate` 递增实现 force-recreate，496-501） |
| `Remove` | 112-174 | 按命名空间标签列出并删除 services/secrets/configs/networks 后 `waitOnTasks` 等待任务终结 |
| `WaitForStatus` | 818-895 | 每秒轮询聚合状态 |

上层包装：`api/exec/swarm_stack.go` 的 `SwarmStackManager`（struct 22-25 行）：`Deploy`（39-89，部署后 `WaitForStatus` 等待最多 30 秒确认没有早期失败，80-88）、`Remove`（121-140）、`CheckRunningStatus`（93-118）。接口：`api/portainer.go:2106-2111`。

---

## 8. 删除与清理（`api/stacks/teardown/teardown.go`）

| 函数/接口 | 行号 | 逻辑说明 |
|---|---|---|
| `Service` 接口 | 23-34 | 三步清理抽象 |
| `RemoveResources` | 61-102 | 按 stack.Type 分发：Compose 栈 `UndeployComposeStack`；Swarm 栈 `SwarmStackManager.Remove` |
| `DeleteRecords` | 104-121 | 删除 stack 数据库记录 + RC |
| `RemoveFiles` | 123-129 | 删除 ProjectPath 目录 |

---

## 9. Git 自动更新（`api/stacks/deployments/deploy.go`）

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `RedeployWhenChanged` | 39-62 | 入口 |
| `redeployWhenChanged` | 64-118 | webhook 触发异步 / polling 经 singleflight 去重（防并发重复部署） |
| `redeployWhenChangedSecondStage` | 120-277 | 拉取 git 比较 hash（153，`update.UpdateGitObject`）；有变更则按 stack.Type 重新部署（211-244 `redeployStack`）并更新 `CurrentDeploymentInfo` |
| `ReconcileSwarmStackStatus` | 285-341 | 启动时把已恢复的 Error 状态 Swarm 栈翻回 Active |

---

## 10. Compose 文件校验（`api/stacks/stackutils/validation.go`）

校验用 **compose-go v2**（`github.com/compose-spec/compose-go/v2/loader`，13-14 行导入）：

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `IsValidStackFile` | 25-72 | `composeloader.LoadWithContext(..., WithSkipValidation)` 解析后，对非管理员用户按端点安全设置逐项拒绝：bind-mount（39-44）、privileged（46-48）、`pid: host`（50-52）、devices（54-56）、sysctls（58-60）、security-opt（62-64）、cap_add/cap_drop（66-68）——与容器创建的安全限制（7 项设置、8 个 HostConfig 字段）对应，见 [container.md](./container.md) §2.3 |
| `ValidateComposeURLs` | 77-101 | SSRF 防护：检查 build context URL 与镜像 registry 主机名 |
| `ValidateEdgeStackComposeContent` | 106-116 | Edge 栈内容校验（同上） |
| `extractImageRegistry` | 154-167 | 从镜像名提取 registry 主机名 |
| `ValidateStackFiles` | 169-194 | 遍历所有 compose 文件调 `IsValidStackFile`；由部署配置在部署前触发（`deployment_compose_config.go:67-72`、`deployment_swarm_config.go:64-72`，均只对非管理员） |

---

## 11. 远程栈部署（EE 机制，CE 中不可达）

`api/stacks/deployments/deployer_remote.go`：`RemoteStackDeployer`（接口 38-49 行）通过在目标环境运行 `portainer/compose-unpacker` 临时容器执行部署。命令构建于 `compose_unpacker_cmd_builder.go`（`buildDeployCmd` 68 行、`buildUndeployCmd` 92 行、`buildSwarmDeployCmd` 138 行等）。

容器编排细节（`deployer_remote.go:206-348` 的 `remoteStack`）：`ImagePull` 拉 unpacker 镜像（243）→ `ContainerCreate`（272-283，挂载 compose 目录与 docker socket）→ defer `ContainerRemove`（284-288）→ `ContainerStart`（290）→ `ContainerWait`（NotRunning 条件，294-301）→ `ContainerLogs` + `stdcopy.StdCopy` 收集部署日志（305-316）→ `ContainerInspect` 查退出码判定成败（318-323）。`createDockerClient`（351）用 1 小时超时。

> CE 中 `IsRelativePathStack` 恒为 false（`api/stacks/stackutils/util.go:47-51`），相对路径栈（即远程部署）仅 EE 使用，此处仅作架构说明。
