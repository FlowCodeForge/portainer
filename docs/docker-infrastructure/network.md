# Network（网络）操作

> 行号基线：分支 `docs` / commit `4097b69bc`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

网络领域**没有专属 HTTP handler**（本版本不存在 `api/http/handler/networks/` 目录）——所有 `/networks/*` 请求走代理路径，由代理拦截层执行访问控制与响应改写。网络是标准的 RC 管理资源类型（`NetworkResourceControl`），含"系统网络"特殊处理。

---

## 1. 操作 → 代码映射总表

### 1.1 代理路径（`api/http/proxy/factory/docker/transport.go` 的 `proxyNetworkRequest`，402-429 行）

| Docker API | 拦截处理 | 行号 |
|---|---|---|
| `POST /networks/create` | `decorateGenericResourceCreationOperation(request, "Id", NetworkResourceControl)`：透传创建 → 201 后从响应 `Id` 建私有 RC → RC 注入响应 | 406-407 |
| `GET /networks`（列表） | `networkListOperation`（RC 装饰/过滤） | 410 → networks.go:38-58 |
| `POST /networks/{id}/connect`、`/disconnect` | `restrictedResourceOperation`（RC 校验） | 412-416 |
| `GET /networks/{id}`（inspect） | `networkInspectOperation` | 418-419 → networks.go:62-77 |
| `DELETE /networks/{id}` | `executeGenericResourceDeletionOperation`：受限校验 → 删除成功后删 RC | 421-423 |
| 其余（如 `POST /networks/{id}` 更新） | `restrictedResourceOperation` | 427-428 |

路由注册：`prefixProxyFuncMap` 中 `"networks"` 键（transport.go:104）。

### 1.2 SDK 直接调用点（业务代码）

| 场景 | 位置:行号 | SDK 调用 | 说明 |
|---|---|---|---|
| 容器重建 | `api/docker/container.go:131 / 156 / 216` | `NetworkDisconnect` / `NetworkConnect` | 唯一的非代理 Network SDK 使用处（见 [container.md](./container.md) §4.2） |
| Dashboard 汇总 | `api/http/handler/docker/dashboard.go:125-130` | `NetworkList` | 用 `uac.NetworkResourceControlGetter` 过滤后返回数量 |
| 环境快照 | `pkg/snapshot/docker.go:255-264` | `NetworkList` | 存 `SnapshotRaw.Networks`（无计数字段） |
| Swarm 栈部署器 | `pkg/libstack/swarm/swarm.go` | `NetworkCreate`(347) / `NetworkList`(267) / `NetworkInspect`(306) / `NetworkRemove`(581) | 见 [compose.md](./compose.md) §7 |

> 未发现业务代码直接调 `NetworkCreate` 做常规创建——网络的创建完全由代理层透传 Docker HTTP API 完成。

---

## 2. 代理层逻辑详解（`api/http/proxy/factory/docker/networks.go`，103 行）

### 2.1 函数映射表

| 函数 | 行号 | 逻辑说明 |
|---|---|---|
| `getInheritedResourceControlFromNetworkLabels` | 22-34 | 网络无独立 RC 时的继承查找：`NetworkInspect` 取 `Labels` → `getStackResourceIDFromLabels` 找 `com.docker.stack.namespace`（Swarm）或 `com.docker.compose.project`（Compose）标签 → 返回**所属 Stack 的 RC**（资源继承） |
| `networkListOperation` | 38-58 | 解析 NetworkList 响应为 JSON 数组，以 `Id` 为标识、`selectorNetworkLabels`（101-103）取 Labels，调 `applyAccessControlOnResourceList` 装饰/过滤后重写响应 |
| `networkInspectOperation` | 62-77 | 单网络按 RC 校验：`applyAccessControlOnResource` 要么返回装饰后的网络，要么改写为 403 |
| `findSystemNetworkResourceControl` | 81-94 | **系统网络特殊处理**：名为 `bridge`/`host`/`ingress`/`nat`/`none` 的网络自动生成 system RC——管理员可见、普通用户隐藏/受限装饰 |

### 2.2 网络创建的完整流程

```
POST /api/endpoints/{id}/docker/networks/create
  ① proxyNetworkRequest 命中 create 分支（transport.go:406）
  ② 透传给 Docker daemon
  ③ 响应 201 → decorateGenericResourceCreationResponse（transport.go:796-818）
     a. 从响应 JSON 的 "Id" 字段取网络 ID
     b. createPrivateResourceControl（access_control.go:110-123）
        —— 为创建者生成私有 RC（仅本人与所属团队可见）
     c. decorateObject（access_control.go:339-348）
        —— 把 RC 写进响应的 Portainer.ResourceControl 字段
```

### 2.3 网络删除的完整流程

```
DELETE /api/endpoints/{id}/docker/networks/{id}
  ① executeGenericResourceDeletionOperation（transport.go:838-860）
  ② 先走 restrictedResourceOperation 鉴权（见 [docker-sdk.md](./docker-sdk.md) §3.4）
  ③ 透传删除，响应 204/200 成功
  ④ 从数据库删除该网络对应的 RC 记录（848-857）
```

---

## 3. RC 校验中的网络特有逻辑

`restrictedResourceOperation`（transport.go:612-690）对网络的特殊分支（`getDockerResourceUUID`，692-739 行）：

- **NetworkResourceControl 分支**（695-699 行）：URL 中的网络标识可能是**名字而非 UUID**，先用 SDK `NetworkInspect` 把名字解析成真实 ID 再查 RC；
- 仍查不到则走 `getInheritedResourceControlFromServiceOrStack`（access_control.go:125-142）找继承 RC。

---

## 4. 权限矩阵（标准 Portainer RBAC）

授权码定义（`api/portainer.go:2523-2527`）：

| 授权码 | 说明 |
|---|---|
| `OperationDockerNetworkList` | 网络列表 |
| `OperationDockerNetworkInspect` | 网络详情 |
| `OperationDockerNetworkCreate` | 创建网络 |
| `OperationDockerNetworkConnect` | 容器接入网络 |
| `OperationDockerNetworkDisconnect` | 容器断开网络 |

角色映射（`api/internal/authorization/authorizations.go`）：

| 角色 | 权限 | 行号 |
|---|---|---|
| 标准用户 | 全部 5 项 | 67-71 |
| 只读用户 | 仅 List / Inspect | 177-178 |
| Edge 相关角色 | 见 | 261-265 |

即：普通用户默认可创建/连接网络，只读用户仅可查看。

---

## 5. 与其他领域的协作

- **Container**：容器创建/重建时的 `NetworkConnect`/`NetworkDisconnect`（[container.md](./container.md)）。
- **Compose**：Compose 栈部署自动创建 `project_default` 网络时**不带** Portainer RC——但网络会带 `com.docker.compose.project` 标签，从而继承栈级 RC（本文 §2.1 继承机制）；Swarm 栈部署器的 external 网络校验与 overlay 网络创建见 [compose.md](./compose.md) §7。
- **RC 继承链**：网络 → Swarm service 标签 → Swarm/Compose stack 标签（README「公共机制」）。
