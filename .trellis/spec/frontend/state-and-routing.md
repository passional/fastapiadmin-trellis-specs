# 状态管理与路由

> 事实来源：`src/store/index.ts`、`src/store/modules/*.store.ts`、`src/router/{routes,guards,MenuProcessor,route-loader,refresh}.ts`、`src/utils/auth/index.ts`。

## 1. 状态分层：什么放哪

| 类别 | 放哪 | 例 |
| --- | --- | --- |
| 页面局部 UI 状态 | 组件内 `ref`/`reactive` | 搜索表单、弹窗开关（`useCrudDialog`）、列显隐 |
| 列表服务端数据 | **`useTable`**（含缓存/去重/分页） | 不要塞进 pinia |
| 跨页面共享的会话/配置 | pinia store | 用户信息、菜单、工作栏、字典、站点配置、界面设置 |
| 令牌 | **`utils/auth`（直接操作 localStorage/sessionStorage）** | 不走 pinia persist，也不走 `@utils/storage` 封装 |
| 列设置持久化 | `useTableColumns` 自动存 localStorage | key = `table-${route.path}` |
| iframe 路由 | `IframeRouteManager` → sessionStorage | F5 后恢复 |

判断口径：**只有一个页面用 → 局部状态；跨页面/跨刷新要活下来 → store 或专用持久化**。不要为了「方便」把列表数据提升到全局 store。

## 2. Pinia stores

`src/store/index.ts` 创建 pinia 并装 `pinia-plugin-persistedstate`，统一 re-export 所有 store；组件里 `import { useXxxStore } from "@stores"`。

| store | 职责 | 持久化 |
| --- | --- | --- |
| `user.store` | 用户信息、登录态、token 刷新 | `key:"user"`，**pick 仅 `language/isLogin/searchHistory/rememberMe`**（令牌不在此） |
| `worktab.store` | 多标签 opened/当前标签、KeepAlive include/exclude 来源 | `key:"worktab"`（全量） |
| `menu.store` | 后端菜单列表、路由生成输入 | pick `homePath/menuWidth` |
| `setting.store` | 布局/主题/页签/水印/工具栏显隐 | 部分字段（persist + useStorage） |
| `dict.store` | 字典缓存 | `persist: true` |
| `config.store` | 站点配置 | `key:"config"`，pick `configData/isConfigLoaded` |
| `notice.store` | 公告 | `key:"notice"`，pick `readIds` |
| `app.store` | 应用级状态 | `persist: true` |
| `table.store` | 表格相关全局状态 | `key:"tableStore"` |

规则：
- 新增 store 放 `store/modules/<名>.store.ts`，命名 `use<名>Store`，并在 `store/index.ts` re-export。
- 持久化**显式写 `key` + `pick`**，只持久化真正需要跨刷新的字段；不要把整个服务端响应塞进 localStorage。
- 在组件外（守卫、工具函数）用 store 要传实例：`useUserStore(store)`，或 `useUserStoreHook()`。
- 刷新缓存统一走 **`refreshAppCaches(opts)`**（`store/index.ts`）：可选 `dictTypes / refreshUser / refreshRoutes / refreshConfig / refreshNotice / clearTags / clearDictBefore`，内部 `Promise.allSettled` 并发拉取，再 `refreshMenuAndRoutes(router)`。**不要在各页面自己拼一套刷新逻辑。**
- 改 `worktab.store` 时**不要删 `clearAll()` 里写 `keepAliveExclude` 的那段**（登出驱逐缓存的唯一手段，详见 [keepalive-and-connections.md](./keepalive-and-connections.md)）。

## 3. 令牌与鉴权存储

`utils/auth/index.ts`：
- key：`access_token` / `refresh_token` / `rememberMe`。
- `rememberMe=true` → localStorage；否则 sessionStorage，并清掉另一端残留。
- **刻意绕过 `@utils/storage` 封装**（注释说明：需要在两端间切换）。改这块要同时考虑「记住我」切换与登出清理。
- 守卫里有 `checkStorageHealth()`：localStorage 异常直接登出跳 `/login`。

## 4. 路由：静态壳 + 动态管道

```
routes.ts        静态壳：Layout、登录、401/403/404/500、redirect、dashboard、fastlink、iframe 占位
                 （首屏即注册，不依赖菜单权限）
guards.ts        beforeEach：存储健康 → 登录校验 → 动态路由加载 → 权限检查 → 404 回落
                 afterEach：前进回退不刷新、NProgress/loading 关闭
MenuProcessor    后端 MenuTable → AppRouteRecord（mapMenuNode）；getMenuList 依 useAppMode 分支
route-loader     ComponentLoader（路径→组件）→ RouteTransformer（菜单→路由记录）→ RouteRegistry（批量 addRoute）
refresh.ts       refreshMenuAndRoutes / refreshState（路由初始化失败 → 跳 /500）
```

- 组件映射用 `import.meta.glob("/src/{views,layouts}/**/*.vue", { eager: true })`；后端菜单 `component_path` 存相对路径（如 `system/user/index`），`ComponentLoader` 去 glob 表匹配。**新页面必须放在 `src/views` 或 `src/layouts` 下**，否则匹配不到。
- 动态路由挂在 `ROOT_LAYOUT_ROUTE_NAME` 下（`RouteRegistry`）。
- 重复注册会 `console.warn("[RouteRegistry] 路由已注册，跳过重复注册")`。
- 公开路径正则：`/401 /403 /404 /500 /redirect /login`。

## 5. 新增页面必须配菜单

业务路由来自后端菜单，**不是前端写死**。在后台「菜单管理」配置（种子 `backend/sql/sys_menu.json`）：

| 字段 | 要求 |
| --- | --- |
| `route_path` | 路由路径（子级相对，父级绝对，`joinAbsolutePath` 拼接） |
| `route_name` | **全局唯一**，且必须等于页面 `defineOptions({ name })` |
| `component_path` | 相对 `src/views` 的路径，如 `system/position/index` |
| `keep_alive` | 布尔；写入 `meta.keepAlive`（`item.keep_alive ?? true`） |
| 菜单类型 | 1 目录 / 2 菜单 / 3 按钮 / 4 外链 |
| 按钮权限 | 权限标识（`module_x:资源:操作`），供 `v-hasPerm` |

**目录路由的 `component` 必须保持 `undefined`**（`MenuProcessor.mapMenuNode` / `RouteTransformer.handleNormalRoute`），靠 RouterView 深度跳级直达叶子页面。给目录挂壳组件会导致重复挂载/重复请求（历史事故，见 keepalive 文档）。

## 6. 组件名匹配（KeepAlive / 工作栏）

- KeepAlive 的 `include`/`exclude` 按**组件 name** 匹配，不是路由 name。
- `fa-page-content/index.vue::resolveOutletComponentName` 会回解 matched 中第一个带 `components` 的后代，取 `comp.name ?? comp.__name`。
- 所以页面必须 `defineOptions({ name: "Position" })` 且与菜单 `route_name` 一致；`<script setup>` 不写 name 时推断名可能与菜单不一致 → 缓存/排除失灵。
- 布局组件同样显式命名（如 `FaPageContent`）。

## 7. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| 列表数据放全局 store | `useTable`（自带缓存/去重/失活停请求） |
| 令牌塞进 pinia persist | `utils/auth` 双端存储 + rememberMe |
| 页面各自拼刷新用户/字典/路由 | `refreshAppCaches` |
| 前端写死业务路由 | 后端菜单 + 动态管道 |
| 目录路由挂壳组件 | `component: undefined`，深度跳级 |
| 页面不写 `defineOptions({ name })` | 必须写且等于 `route_name` |
| 新页面放 `src/views` 之外 | glob 匹配不到，菜单配了也白屏 |
| 删 `worktab.clearAll()` 的 exclude 写入 | 登出后旧实例无法驱逐 |
