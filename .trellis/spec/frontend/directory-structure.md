# 前端目录结构（frontend/web）

> 事实来源：`frontend/web/src/` 实际布局、`vite.config.ts`（alias / AutoImport / Components）、`package.json`。

## 1. src 布局

```
frontend/web/src/
├── api/module_x/xxx.ts        # ★ API 层，与后端模块对应（对象字面量 + request）
├── views/module_x/            # ★ 业务页面
│   ├── <功能>/index.vue
│   └── <功能>/components/     # 页面私有子组件（自动注册）
├── components/                # ★ 公共组件 FaXxx，按形态分子目录
│   ├── forms/                 # FaForm / FaSearchBar ...
│   ├── tables/                # FaTable / FaTableHeader / FaTableHeaderLeft / FaButtonTable / FaButtonMore
│   ├── modal/                 # FaDialog / FaDrawer ...
│   ├── display/               # FaDescriptions / FaStatusTag / FaEmptyState ...
│   ├── containers/ cards/ charts/ navigation/ media/ banners/ actions/
│   ├── base/ exception/ skeleton/ text-effect/ others/
├── hooks/core/                # ★ 开发套件（useTable/useCrudForm/...），AutoImport 自动导入
├── layouts/                   # 布局；fa-page-content/index.vue 是唯一 KeepAlive 出口
├── router/                    # routes.ts 静态壳 + guards/MenuProcessor/route-loader/refresh 动态管道
├── store/modules/*.store.ts   # pinia stores（user/worktab/menu/setting/dict/config/notice/app/table）
├── directives/                # permission（v-hasPerm）、business（ripple/highlight）
├── locales/langs/{zh,en}.json # i18n 词条（两份必须同步）
├── utils/                     # http(request)/auth/storage/navigation/download/sse/message/table/query/form/ui...
├── types/                     # global.d.ts（全局契约类型）、component/router/store/config/ai、auto-imports.d.ts、components.d.ts
├── styles/                    # tailwind.css(@theme)、core/(_tokens.scss、_brand-tokens.generated.scss...)、element-plus/、layouts/、pages/
├── enums/ constants/ config/ plugins/ assets/ mock/
├── __tests__/*.spec.ts        # vitest
├── App.vue / main.ts / env.d.ts
```

## 2. 路径别名（`vite.config.ts`）

| 别名 | 指向 |
| --- | --- |
| `@` | `src` |
| `@views` | `src/views` |
| `@api` | `src/api` |
| `@utils` | `src/utils` |
| `@stores` | `src/store` |
| `@plugins` | `src/plugins` |
| `@styles` | `src/styles` |
| `@imgs` / `@icons` / `@fa_imgs` | `src/assets/images` / `.../svg` / `src/assets/fa_imgs` |

现有文件混用 `@/hooks/core/useCrudForm` 与 `@utils`、`@stores`——**跟随所在文件的既有写法**，不要为了统一而大面积改 import。

## 3. 自动导入（不要手写多余的 import）

`AutoImport`（`vite.config.ts:179`）：
- presets：`vue`、`vue-router`、`pinia`、`@vueuse/core`、`vue-i18n`、`axios`默认导出；
- `element-plus/es` 显式列表：`ElScrollbar`、`ElInput`、`ElMessageBox`、`ElNotification`、`ElMessage`、`ElSwitch`、`ElAvatar`、`ElButton`；
- `dirs: ["./src/hooks/core"]` → **hooks 自动导入**；
- `vueTemplate: true`，产物声明 `src/types/auto-imports.d.ts`，并生成 `.eslintrc-auto-import.json`。

`Components`（`vite.config.ts:216`）：
- `dirs: ["src/components", "src/layouts", "src/**/components"]` → **FaXxx 公共组件、布局组件、页面私有 components 都自动注册**；
- `ElementPlusResolver()` → 模板里直接用 `ElTable`/`ElCard` 等；
- 产物声明 `src/types/components.d.ts`。

实务口径（与 SKILL.md §4 一致）：
- `ref`/`computed`/`watch`/`onMounted`/`onActivated` 等 vue API **无需 import**；
- Element Plus 组件模板内直接用；
- `ElMessage` 等按现有文件习惯**可显式 import**（很多页面这么做，跟随即可）；
- 类型（`type DictForm`）仍需显式 import；
- 自动生成文件（`auto-imports.d.ts`/`components.d.ts`）**不要手改**。

## 4. 模块组织与命名

| 对象 | 约定 | 例 |
| --- | --- | --- |
| API 目录 | `api/module_<域>/<资源>.ts` | `api/module_system/dict.ts` |
| API 对象 | `<资源>API`，默认导出；方法 `<动作><资源>` | `DictAPI.listDictType` |
| 页面目录 | `views/module_<域>/<功能>/index.vue` | `views/module_system/position/index.vue` |
| 页面私有组件 | 同目录 `components/`，仍用 `Fa` 前缀 | `dict/components/DictDataPanel.vue`、`user/components/FaDeptTree.vue` |
| 公共组件 | `components/<形态>/fa-xxx/index.vue`，名 `FaXxx` | `components/tables/fa-table/index.vue` |
| 组件 name | `defineOptions({ name })` **等于菜单 route_name** | `name: "Position"` |
| store | `store/modules/<名>.store.ts`，`use<名>Store` | `useWorktabStore` |
| hook | `hooks/core/use<名>.ts` | `useTable` |

> 页面私有组件命名不完全统一（有 `FaDeptTree.vue` 也有 `DictDataPanel.vue`）；**新建时优先 `FaXxx`**，改旧文件跟随原名。

### 命名不对称陷阱（存储模块）

后端 `modules/task/storage/` → URL `/task/storage/...` → 前端 **API 在 `api/module_storage/`**，**页面在 `views/module_task/storage/`**。改存储相关功能时三处都要查，别只按目录名推断。

## 5. 环境与构建

| 文件 | 关键项 |
| --- | --- |
| `.env` | `VITE_APP_BASE_API=/api/v1`、`VITE_PORT=5180` |
| `.env.development` | `VITE_API_BASE_URL=http://127.0.0.1:8003`（代理目标）、`VITE_APP_WS_ENDPOINT=ws://localhost:8003`（直连不走代理） |
| `vite.config.ts` | `build.outDir="dist"`、`target="es2024"`、生产 terser（`drop_console`、`pure_funcs: console.log/info`）、gzip 压缩 |

- 后端端口以 `backend/env/.env.dev` 的 `SERVER_PORT` 为准（当前 8003，代码默认 8001）；改端口两端同步。
- 生产构建会 **drop console**，所以 `console.log` 调试信息线上不存在——不要依赖它做线上排查。
- 产物有三份副本要同步，见 [../project/release-and-artifacts.md](../project/release-and-artifacts.md)。

## 6. 另外两个包

- `frontend/app`（uni-app）：改 UI 遵循 `frontend/app/.agents/skills/wot-ui-*`；调试 `pnpm dev:mp-weixin` / `pnpm dev:h5`；产物 `dist/build/h5` 对应 nginx `/app`。**跨端需求要同时评估 web 与 app，API 层各自维护。**
- `frontend/docs`（VitePress）：`src/guide/` 与 `src/en/guide/` **中英两份同步改**。
