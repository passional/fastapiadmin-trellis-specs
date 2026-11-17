# 前端开发规范（frontend/）

> 三个独立 pnpm 包，约定不同，**不要混用**：
> - `frontend/web`：管理后台（Vue3 + Vite + TS + Element Plus + Tailwind4 + Pinia）——本层主要对象
> - `frontend/app`：小程序/移动端（uni-app + wot-ui），约定在 `frontend/app/.agents/skills/`（`wot-ui-v2`、`wot-ui-cli`、`create-wot-ui-theme`、`wot-ui-unocss-preset-guide`、`starter-cleaner`）
> - `frontend/docs`：VitePress 文档站，中英双语 `src/guide` 与 `src/en/guide`，**改动必须两份同步**

## 权威文档

| 文档 | 管什么 |
| --- | --- |
| 根目录 `SKILL.md` §4/§7/§8/§9 | 作者开发规则：API/页面/权限/i18n/路由约定、KeepAlive 踩坑、dist 产物 |
| [`docs/UI-ENGINEERING-SPEC.md`](../../../docs/UI-ENGINEERING-SPEC.md) | **工程契约**：token、页面骨架、表格/表单契约、反馈状态、A11y、响应式、评审清单（§9）、验收命令（§10） |
| [`docs/DESIGN-SPEC.md`](../../../docs/DESIGN-SPEC.md) + `design/tokens.json` | 品牌与视觉唯一来源 |

冲突时工程实现以 `UI-ENGINEERING-SPEC.md` 为准，并回改 `DESIGN-SPEC.md`，不保留两套说法。

## 文件索引

| 文件 | 内容 | 什么时候读 |
| --- | --- | --- |
| [directory-structure.md](./directory-structure.md) | `src/` 布局、路径别名、自动导入、三包边界 | 找文件放哪 / 新建页面 |
| [api-and-types.md](./api-and-types.md) | API 层写法、`ApiResponse`/`PageResult` 全局类型、类型组织与命名 | 写/改 API、定义类型 |
| [component-guidelines.md](./component-guidelines.md) | FaXxx 公共组件、页面私有组件、token/样式、i18n、图标、A11y、状态闭环 | 写页面 / 写组件 |
| [crud-hooks.md](./crud-hooks.md) | `useTable`/`useCrudForm`/`useCrudDialog`/`useTableColumns`/`useImportExport`/`useConfirm`/`useAuth` | 写 CRUD 列表/弹窗 |
| [state-and-routing.md](./state-and-routing.md) | Pinia stores 与持久化、动态路由管道、菜单配置、组件名匹配 | 改 store / 加路由 / 配菜单 |
| [keepalive-and-connections.md](./keepalive-and-connections.md) | 单层 KeepAlive 契约、include/exclude、WebSocket/定时器/SSE 生命周期与守卫 | 有副作用的页面、排查重复请求/重复连接 |
| [quality-guidelines.md](./quality-guidelines.md) | 验收命令、评审清单、禁止模式、dist 产物验证入口 | 提交前 / 上线前 |

## 前端三条铁律

1. **CRUD 页面复用现成套件**：`useTable` + `FaTable` + `FaSearchBar` + `FaTableHeader`，弹窗 `useCrudDialog`/`useCrudForm` + `FaDialog` + `FaForm`。**不要手写 ElTable/ElDialog 堆页面**。
2. **`defineOptions({ name })` 必须与菜单 `route_name` 一致**：KeepAlive 与工作栏按组件名匹配，不一致则缓存/排除失灵。
3. **有副作用的页面必须实现 `onActivated`/`onDeactivated`**：`onUnmounted` 只在缓存被驱逐时触发，不能作为唯一清理点。

## 快速定位

| 要找什么 | 去哪 |
| --- | --- |
| 标准列表页样板 | `src/views/module_system/position/index.vue` |
| 主从工作区样板 | `src/views/module_system/dict/index.vue`（`FaWorkbench` + `FaSidePanel`） |
| API 样板 | `src/api/module_system/dict.ts` |
| 公共组件 | `src/components/{forms,tables,modal,display,charts,cards,containers,...}` |
| 开发套件 hooks | `src/hooks/core/`（自动导入，无需 import） |
| 全局类型 | `src/types/global.d.ts`（`ApiResponse`/`PageResult`/`BaseType`/`BatchType`…） |
| 请求封装 | `src/utils/http/index.ts`（`request` 经 `@utils` 导出） |
| 路由管道 | `src/router/{routes,guards,MenuProcessor,route-loader,refresh}.ts` |
| 唯一 KeepAlive 出口 | `src/layouts/fa-page-content/index.vue` |
| i18n 词条 | `src/locales/langs/{zh,en}.json` |
| 测试 | `src/__tests__/*.spec.ts`（vitest） |
