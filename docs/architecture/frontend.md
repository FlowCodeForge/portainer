# 前端总体架构

> 行号基线：分支 `docs` / commit `269738a01`；行号可能随后续提交漂移，**以函数名为准、行号为辅**。公共约定见 [README.md](./README.md)。

Portainer 前端是 **AngularJS 1.8 + React 17 的混合应用**（`package.json:70,122`），处于长期迁移过程中：路由、模块加载、登录态由 AngularJS（UI-Router）持有；新功能页面全部用 React 编写，经 `r2a` 桥接注册为 AngularJS 组件。无独立前端仓库/子目录，全部在 `app/` 下。

## 1. 顶层结构与入口

| 位置 | 职责 |
|---|---|
| `app/index.js` | JS 入口：定义 AngularJS 主模块 `portainer`，注入 `@uirouter/react-hybrid` 与 5 个子模块（`portainer.app`、`portainer.agent`、`azureModule`、`portainer.docker`、`portainer.kubernetes`、`portainer.edge`），末尾 `require.context` 自动加载全部非测试 js（`app/index.js:72-77`） |
| `app/index.html` | HTML 入口：`ng-app="portainer"`，两个命名 ui-view（`sidebar` 与 `content`）承载全部页面 |
| `app/portainer/` | 核心模块：用户/环境/设置/模板/注册表/RBAC（AngularJS 遗留为主） |
| `app/docker/`、`app/kubernetes/`、`app/edge/`、`app/agent/`、`app/azure/` | 按平台分的 AngularJS 模块，每个都有 `__module.js` 注册路由状态树 |
| `app/react/` | **React 新代码**，按域分：`portainer/`、`docker/`、`kubernetes/`、`edge/`、`azure/`、`sidebar/`、`components/`（共享组件，别名 `@@/*`）、`hooks/` |
| `app/react-tools/` | 桥接工具：`react2angular.tsx`、HOC 包装（`withUIRouter` 等）、react-query 配置 |
| `app/assets/` | 样式（含 CSS 变量主题 `theme.css`）与图标 |

路由状态树集中在各模块的 `__module.js`：状态 `'content@': { component: 'xxxView' }` 即 React 视图，`templateUrl + controller` 为旧 AngularJS 页面，二者在同一状态树中混用（例：`app/docker/__module.js:73-95`，configs 列表用 React、详情用 templateUrl）。

## 2. AngularJS ↔ React 桥接（核心机制）

### 2.1 r2a（react2angular）

`app/react-tools/react2angular.tsx:37-47` 把 React 组件包装成带 `<` 单向绑定的 AngularJS component，controller 内 `ReactDOM.render` 到宿主元素（`react2angular.tsx:61-68`）。

### 2.2 惯用包装链

```ts
r2a(withUIRouter(withReactQuery(withCurrentUser(View))), [props])
```

四个 HOC 都在 `app/react-tools/`：`withUIRouter.tsx`（注入 UI-Router 服务）、`withReactQuery.tsx`（QueryClientProvider）、`withCurrentUser.tsx`（UserProvider）、`withI18nSuspense.tsx`。注册示例：`app/portainer/react/views/environments.ts:9-20`。

### 2.3 React 视图的模块化注册

每个 AngularJS 模块挂一个 `react/` 桥接目录（如 `app/docker/react/views/configs.ts:9-14`），把 `app/react/docker/...` 下的视图组件注册成 `configsListView` 之类的 AngularJS 组件，供路由 `component:` 引用。侧边栏整体已是 React（`app/react/sidebar/`）。

反向桥接也存在：AngularJS 服务可直接调用 React 的 axios 函数（`app/portainer/services/angularToReact.ts` 的 `useAxios` 包装），旧服务正逐步改为转发（`app/docker/services/containerService.js`）。

## 3. 状态管理

- **React 侧**：`@tanstack/react-query` v4（服务端状态，`package.json:58`）+ 少量 zustand（本地 UI 状态，如 `app/react/sidebar/sidebarStore.ts`）。
- QueryClient 工厂 `app/react-tools/react-query.ts:71-90`：`staleTime: 20`、`networkMode: 'offlineFirst'`、Mutation/QueryCache 统一 `notifyError`。
- 约定：错误用 `withError(fallbackMessage)` 元信息（`react-query.ts:14-20`）；成功后失效用 `withInvalidate`（`react-query.ts:24-41`）；query key 工厂集中在各域 `queries/query-keys.ts`。

## 4. API 客户端

### 4.1 React 侧（统一 axios 实例）

`app/react/portainer/services/axios/axios.ts:50-55`：`Axios.create({ baseURL: 'api' })`。拦截器：

| 拦截器 | 位置 | 作用 |
|---|---|---|
| 缓存刷新事件 | `axios.ts:56-59` | mutation 后清 K8s 缓存 |
| dockerMaxAPIVersion | `axios.ts:125` | 限制 Docker API 版本 |
| agentInterceptor | `axios.ts:107-123` | `/docker/` 请求注入 `X-PortainerAgent-Target` 与 `X-PortainerAgent-ManagerOperation` 头 |
| 可选缓存适配器 | `axios.ts:69-99` `updateAxiosAdapter` | 用户开启 `UseCache` 时挂 `axios-cache-interceptor`，按响应头 `X-Portainer-Cache` 判定（5 分钟） |
| 401 响应 | `axios.ts:130-146` | 非 docker/gitlab 代理路由跳 `/logout` |

### 4.2 认证方式

**前端不设置 `Authorization` / `X-API-Key` 头**——浏览器侧走 cookie 会话（登录 `POST /api/auth`，`app/portainer/rest/auth.js:1-15`；会话服务 `app/portainer/services/authentication.js`）。API Key 仅用于外部程序直连后端。

### 4.3 AngularJS 侧

- `$resource` 工厂集中在各模块 `rest/` 目录（`app/portainer/rest/*.js`），endpoint 常量在 `app/constants.ts:2-26` 经 `ng-constants.ts` 注册。
- 业务服务工厂包一层：`app/portainer/services/api/endpointService.js:3-16`。
- 未用 Restangular；`$http` 拦截器在 `app/config.js:11-26`。

## 5. 主题与 UI 组件

- **无外部组件库**（无 MUI/AntD）：自研 `app/react/components/`（datatables、form-components、modals 等，别名 `@@/*`）+ Bootstrap 3 遗留 + Tailwind 工具类。
- 主题由 `<html theme="dark|highcontrast">` 属性驱动：`app/react/portainer/services/applyTheme.ts:5-23`（auto 模式监听 `prefers-color-scheme`）；CSS 变量在 `app/assets/css/theme.css`；Tailwind 自定义变体 `th-dark:`/`th-highcontrast:`（`tailwind.config.js:33-38`）。
- CE/BE 功能开关：`app/react/portainer/feature-flags/feature-flags.service.ts:3`（`isBE`）+ `BEFeatureIndicator` 组件。

## 6. 构建

- Webpack 5（`webpack/webpack.common.js`，dev/prod 配置在旁边），入口 `./app`，输出 `dist/public`，`[name].[contenthash]`。
- devServer：端口 `8999`（`webpack.common.js:109`），`/api` 代理到 `http://localhost:9000` 且 `ws: true`（`webpack.common.js:110-116`）。
- `.html`（除 index.html）经 ngtemplate-loader 编入模板缓存（`webpack.common.js:38-55`）；CSS Modules（`webpack.common.js:77-95`）；svg 走 @svgr。
- TS 别名（`tsconfig.json:27-32`）：`@@/*` → `app/react/components/*`、`@/*` → `app/*`、`@api/*` → `app/react/portainer/generated-api/portainer/*`。
- 单包仓库（非 pnpm workspace），包名 `@portainer/ce`，`packageManager: pnpm@10`。测试 Vitest + React Testing Library；组件开发 Storybook（6006，MSW mock `/api`）。
- i18n：i18next 已接入（`app/i18n.ts:6-16`）但处早期阶段，翻译文件在 `translations/en/translation.json`，React 代码暂未大规模使用 `useTranslation`。

## 7. 类型定义与后端 DTO 同步

- **有 codegen**：`pnpm generate-api`（`package.json:31`）用 `@hey-api/openapi-ts` 从 `api/docs/openapi.yaml` 生成到 `app/react/portainer/generated-api/portainer/`（`openapi-ts.config.ts:4-9`），插件含 typescript/sdk/client-axios/zod。生成代码从主 tsconfig 排除，用独立 `tsconfig.generated.json`。
- hey-api client 复用 §4.1 的同一 axios 实例（`configure-hey-api.ts:5-10`）。
- 存量代码仍有大量手写 `types.ts`（各特性目录）；Docker/K8s 上游 DTO 直接用社区类型包 `docker-types`、`kubernetes-types`。

## 8. 功能模块目录归属

| 域 | AngularJS | React |
|---|---|---|
| Docker | `app/docker/{views,components,services,models}` | `app/react/docker/`（containers、images、networks、volumes、configs、events、host、swarm、stacks、DashboardView…） |
| Kubernetes | `app/kubernetes/{views,components,services,rest,converters,helm,ingress,...}` | `app/react/kubernetes/`（applications、cluster、namespaces、ingresses、configs、volumes、helm、datatables…） |
| Edge | `app/edge/{components,rest,services}`（视图已全 React） | `app/react/edge/{edge-devices,edge-groups,edge-jobs,edge-stacks}` |
| 核心（用户/环境/设置） | `app/portainer/views/*`、`components/*` | `app/react/portainer/*`（environments、users、settings、registries、custom-templates…） |
| Agent 卷/文件浏览 | `app/agent/` | — |
| Azure ACI | — | `app/azure/index.ts`（全 React） |

**特性级拆分约定**：每个特性一个目录，内含 `ListView/`、`ItemView/`（或 `DetailsView/`）、`CreateView/`、`components/`、`queries/`（query-keys.ts + useXxx.ts）、`*.service.ts`、`types.ts`。例：`app/react/kubernetes/applications/`、`app/react/portainer/environments/`。
