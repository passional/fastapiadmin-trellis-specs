# API 层与类型规范

> 事实来源：`src/api/module_system/dict.ts`（样板）、`src/utils/http/index.ts`（request/拦截器）、`src/types/global.d.ts`、`src/enums/api/result.enum.ts`。

## 1. API 层写法（照抄 dict.ts）

```ts
import { request } from "@utils";

const API_PATH = "/system/dict";

const DictAPI = {
  listDictType(query: DictPageQuery) {
    return request<ApiResponse<PageResult<DictTable>>>({
      url: `${API_PATH}/type/list`,
      method: "get",
      params: query,
    });
  },
  createDictType(body: DictForm) {
    return request<ApiResponse>({ url: `${API_PATH}/type/create`, method: "post", data: body });
  },
  updateDictType(id: number, body: DictForm) {
    return request<ApiResponse>({ url: `${API_PATH}/type/update/${id}`, method: "put", data: body });
  },
  deleteDictType(body: number[]) {
    return request<ApiResponse>({ url: `${API_PATH}/type/delete`, method: "delete", data: body });
  },
  exportDictType(body: DictPageQuery) {
    return request<Blob>({ url: `${API_PATH}/type/export`, method: "post", data: body, responseType: "blob" });
  },
};

export default DictAPI;
```

规则：
- **对象字面量方法 + `request`**（来自 `@utils`），默认导出 `<资源>API`；不要每个方法单独 export，也不要新建 axios 实例。
- 文件顶部 `const API_PATH = "/<域>/<资源>"`，URL 与后端 controller **逐字对齐**（后端 `/api/v1` 前缀由 `baseURL` 提供，API_PATH 里不写）。
- 泛型：列表 `ApiResponse<PageResult<T>>`、详情/下拉 `ApiResponse<T>` / `ApiResponse<T[]>`、写操作 `ApiResponse`（无 data）、下载 `Blob` + `responseType: "blob"`。
- GET 用 `params`，POST/PUT/PATCH/DELETE 用 `data`（**批量删除是 DELETE 带 body**，与后端一致）。
- 方法命名 `<动作><资源>`：`list/detail/create/update/delete/batch/export/import/option...`。
- 类型定义**跟 API 同文件底部导出**（`DictPageQuery` / `DictTable` / `DictForm`），页面从 API 文件 import type。

## 2. request 与响应形状

`request` 是 AxiosInstance（`utils/http/index.ts:245`）：
- `baseURL = VITE_APP_BASE_API`（`/api/v1`），timeout 取 `VITE_API_TIMEOUT` 或 15000；
- `paramsSerializer` 用 `qs.stringify(params, { indices: false })`（数组参数不带下标）；
- 请求拦截器自动加 `Bearer <accessToken>`；`headers.Authorization = NO_AUTH_FLAG`（`"no-auth"`）或 `skipAuth: true` 可跳过；
- **响应拦截器返回完整 `AxiosResponse`**，所以取值是 `res.data`（信封）→ `res.data.data`（业务数据）。

信封与业务码：

```ts
interface ApiResponse<T = any> { code: number; data: T; msg: string; status_code: number; success: boolean }
```

- `ResultEnum.SUCCESS = 0`（对应后端 `RET.OK.code = 0`）；非 0 → 拦截器 `ElMessage.error(msg)` 并 reject。
- `10401 TOKEN_EXPIRED` → 静默刷新 token 后重放请求；`10403 UNAUTHORIZED` → 无权限处理。
- **非 GET 且非 login/logout 的成功响应会自动弹 `ElMessage.success(msg)`**；页面不要再手动弹同义成功提示（`utils/message.ts` 会做 80ms 合并，但叠加仍是噪音）。可用 `showSuccessMessage: false` / `showErrorMessage: false` 关闭。
- blob 直通；下载失败时拦截器会解析 Blob 里的 JSON 错误体优先展示后端 msg（401 除外，留给续期）。
- 取消请求（`ERR_CANCELED`）静默处理，不弹提示。

## 3. 分页契约

```ts
interface PageQuery extends BaseQueryParams { page_no: number; page_size: number }
interface PageResult<T = any> { items: T[]; total: number; page_no: number; page_size: number; has_next: boolean }
```

- 与后端 `PaginationQueryParam` / `PageResultSchema` 一一对应（`page_size` 后端上限 100）。
- `order_by` 传 **JSON 字符串**：`'[{"id":"asc"}]'`。
- 时间范围传数组 `["start","end"]`（`BaseQueryParams.created_time/updated_time`），后端 `search_to_dict` 转 between。
- `useTable` 内部分页状态叫 `current/size`，由 `paginationKey` 映射到 `page_no/page_size`——**不要在页面里自己改这套映射**。
- 查询参数清洗用共享工具：`cleanEmptyArrayParams`、`stripPaginationParams`（`@utils`），不要每页手写。

## 4. 类型组织

| 类型放哪 | 内容 |
| --- | --- |
| `src/types/global.d.ts`（全局声明，无需 import） | `ApiResponse`、`PageResult`、`PageQuery`、`BaseQueryParams`、`UserByQueryParams`、`BaseType`、`BaseFormType`、`CommonType`、`BatchType`、`OptionType`、`ExcelResult` |
| `src/api/module_x/xxx.ts` 底部 | 该资源的 `XxxTable`（extends `BaseType`）、`XxxForm`（extends `BaseFormType`）、`XxxPageQuery`（extends `PageQuery`） |
| `src/types/{component,router,store,config,ai}/` | 组件 props / 路由记录 / store 状态等跨模块类型 |
| 页面/组件内 | 只服务本文件的局部类型（如 `PositionSearchForm`） |
| `src/types/auto-imports.d.ts`、`components.d.ts` | **构建产物，禁止手改** |

规则：
- 表格行类型继承 `BaseType`（带 `id?/uuid?/created_time?/created_by?/...`），表单类型继承 `BaseFormType`（只带 `id?`）——审计字段不要每个资源重复声明。
- 字段可选性与后端 OutSchema 对齐：后端 `str | None` → 前端 `?: string`；渲染时给兜底（`row.name ?? "—"`）。
- 批量状态用全局 `BatchType { ids: number[]; status: number }`。
- 下拉选项用 `OptionType { value, label, children? }`。
- **不要用 `any` 兜底**；确实未知时用 `unknown` + 收窄。现存少量 `as unknown as X` 是历史写法，新代码避免。
- 改类型必须跑 `pnpm ts:check`（`vue-tsc --noEmit --skipLibCheck`）。

## 5. 跨端对齐检查点

- URL / 方法 / 参数位置（params vs data）与后端 controller 一致。
- 信封字段 `code/msg/data/status_code/success` 与后端 `ResponseSchema` 一致；改字段名两端同步。
- 权限标识逐字一致（含嵌套子域的四段式，如 `module_task:storage:node:query`）。
- 状态枚举语义一致（`0` 启用 / `1` 停用；公告 `0/1/2`；菜单类型 `1/2/3/4`）。
- 时间是**后端已格式化的本地时区字符串**（`DateTimeStr`），前端用 `formatDateTime`（`@utils`）展示，**不要再做时区加减**。

## 6. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| 页面里直接 `axios.get(...)` | 走 API 层 + `request` |
| `res.data` 当业务数据用 | 信封是 `res.data`，业务数据是 `res.data.data`（或用 `useTable` 自动解包） |
| 每个资源重复声明审计字段 | 继承 `BaseType` / `BaseFormType` |
| 写操作成功后再 `ElMessage.success("保存成功")` | 拦截器已弹后端 msg；需要额外提示用 `onSubmitSuccess` 做业务动作 |
| 前端自己给时间 +8 小时 | 后端已按 `TIMEZONE` 输出 |
| 手改 `auto-imports.d.ts` / `components.d.ts` | 由插件生成 |
