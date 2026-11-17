# CRUD 开发套件（hooks/core）

> 事实来源：`src/hooks/core/{useTable,useTableColumns,useTableSelection,useCrudDialog,useCrudForm,useImportExport,useConfirm,useAuth}.ts`。
> 这些 hook 由 AutoImport 自动导入（`dirs: ["./src/hooks/core"]`），**无需手写 import**（现有文件显式 import 也允许，跟随即可）。
> 目标：CRUD 页面不重复造轮子。样板 `views/module_system/position/index.vue`、`dict/index.vue`。

## 1. useTable —— 列表数据

```ts
const {
  data, loading, error, isEmpty,
  pagination, handleSizeChange, handleCurrentChange,
  searchParams, replaceSearchParams, resetSearchParams,
  getData, fetchData, refreshData, refreshCreate, refreshUpdate, refreshRemove,
  cancelRequest,
} = useTable({
  core: {
    apiFn: PositionAPI.listPosition,
    apiParams: { page_no: 1, page_size: 10 },
    columnsFactory: resolveStatusColumns<PositionTable>(() => [
      { type: "selection", width: 48, fixed: "left" },
      { type: "globalIndex", width: 56, label: "序号" },
      { prop: "name", label: "岗位名称", minWidth: 100, showOverflowTooltip: true },
      { prop: "status", label: "状态", width: 88,
        status: { 0: { type: "success", text: "启用" }, 1: { type: "danger", text: "停用" } } },
      { prop: "created_time", label: "创建时间", width: 190,
        formatter: (row) => formatDateTime(row.created_time), sortable: true },
    ]),
  },
});
```

配置分组：`core`（apiFn / apiParams / excludeParams / immediate / columnsFactory / paginationKey）、`transform`（dataTransformer / responseAdapter）、`performance`（enableCache / cacheTime / debounceTime / maxCacheSize）、`hooks`（成功/失败回调）。泛型由 `apiFn` 推导记录类型，**不要手写记录泛型**。

### ⚠️ 命名陷阱：getData vs fetchData

| 返回名 | 实际行为 |
| --- | --- |
| `getData` | **回到第一页**并查询（搜索 / 重置页码场景） |
| `fetchData` | **当前页**加载（错误已内部处理） |
| `getDataDebounced` | 防抖版 getData |

### 刷新语义（按场景选，不要一律 refreshData）

| 方法 | 用途 |
| --- | --- |
| `refreshData` | 全量刷新：清空所有缓存（手动刷新按钮） |
| `refreshSoft` | 轻量刷新：仅清当前搜索条件缓存，保持分页（定时刷新） |
| `refreshCreate` | 新增后：回第一页 + 清分页缓存 |
| `refreshUpdate` | 更新后：保持当前页，仅清当前搜索缓存 |
| `refreshRemove` | 删除后：智能处理页码，避免落在空页 |

### 内建行为（不要重复实现）

- 请求去重：**仅单实例内部**（`inFlightDedupePromise` + key），同参进行中请求复用 Promise。**不做跨实例去重**——不同接口同参会串台（PR #516 教训）。
- KeepAlive 联动：`onDeactivated` 取消请求并置 `tableViewActive`，失活期间 `fetchData` 直接返回当前快照**不发 HTTP**；`onActivated` 恢复。
- 对外 `getData`/`fetchData` 经 `fetchDataQuiet` 吞掉 reject，**页面不要再套 try/catch 重复弹错误**；错误通过 `error` 暴露给 `FaTable`。
- 分页字段默认 `current/size`，通过 `core.paginationKey` 映射到后端 `page_no/page_size`。
- 缓存策略见 `CacheInvalidationStrategy`（`@utils`）。

## 2. useTableColumns / useTableSelection

- `useTableColumns`：列显隐、拖拽排序、增删改；与 `columnsFactory` 配合；**列设置自动持久化到 localStorage，key 为 `table-${route.path}`**。导出 `getColumnVisibility` / `getColumnChecks` 供表头复用同一套规则（`visible` 优先，否则 `checked`，默认展示）。特殊列 `selection/expand/index` 用占位 prop（`__selection__` 等）避免与业务列冲突。
- `useTableSelection<T>()` → `{ selectedRows, selectedIds, batchDeleting, onTableSelectionChange, clearSelection }`；`selectedIds` 自动过滤 null/NaN。批量删除把 `batchDeleting` 接给 `FaTableHeaderLeft` 的 `:delete-loading`。

## 3. useCrudDialog + useCrudForm —— 弹窗表单

```ts
const { dialogVisible, openDialog, closeDialog } = useCrudDialog();
// openDialog("create") / openDialog("update", "修改XX") / openDialog("detail", "XX详情")

const crud = useCrudForm<DeptForm>({
  formData: ref({ id: undefined, name: "", status: 0 }),
  initialFormData: { id: undefined, name: "", status: 0 },
  dialogVisible,                 // 来自 useCrudDialog()
  dataFormRef,                   // FaForm 的 ref
  formRenderKey,                 // 用于 :key 强制重渲染
  detailApi: DeptAPI.detailDept,
  createApi: DeptAPI.createDept,
  updateApi: DeptAPI.updateDept,
  onCreateSuccess: async () => { await refreshCreate(); },
  onUpdateSuccess: async () => { await refreshUpdate(); },
});
// crud.handleOpenDialog("create") / ("update", row.id) / crud.handleSubmit() / crud.handleCloseDialog()
```

- `useCrudForm` 统一管理「对话框 + 表单 + 提交」，省掉每页约 60 行重复代码；`submitLoading` 接 `FaDialog` 的 `:confirm-loading`。
- `detailApi` 形状是 `(id) => Promise<{ data: { data?: T } }>`（即 AxiosResponse → 信封 → payload），与 API 层一致。
- 关闭时自动 `resetFields()` + `clearValidate()` + 回填 `initialFormData`。
- `formRenderKey` 变化用于强制重建 `FaForm`（切换 create/update/detail 时避免残留校验态）。
- 详情分支用 `FaDescriptions` + `detailFormData`，不要另写只读表单。
- 提交成功后的额外业务动作（刷字典、刷配置）放 `onSubmitSuccess`，**不要在里面再弹成功提示**（拦截器已弹）。

## 4. useImportExport / useConfirm

- `useImportExport()` → `{ importVisible, exportVisible, openImport, openExport }`，配 `FaImportDialog` / `FaExportDialog`（或 `FaExcelImport` / `FaExcelExport`）。
- `useConfirm` 导出函数式确认：`confirmDelete(message?)`、`confirmBatchDelete(count, names?)`（自动拼「前 5 条…等 N 条」）、`confirmToggleStatus("enable"|"disable")`。**危险操作必须二次确认**，用这些封装而不是每页手写 `ElMessageBox.confirm` 配置。
- 确认文案要说明对象与后果（见 component-guidelines §6）。

## 5. useAuth / v-hasPerm —— 权限

```vue
<ElButton v-hasPerm="['module_system:dict_type:create']" type="primary">新增</ElButton>
```

```ts
const { hasAuth } = useAuth();
if (hasAuth("module_system:position:update")) { ... }
```

- `v-hasPerm`（`src/directives/permission/index.ts`）支持字符串或数组（**数组任一匹配即放行**）；无权限时**直接从 DOM 移除元素**；传空/非法值会 throw 提示。
- 内部委托 `useAuth().hasAuth` → `utils/checkPerm.ts`（superuser / ROLE_ROOT / 通配符 / perms），并兼容路由 `meta.authList` 短标识（后端模式）。
- `useAuth` 在无 setup 上下文（指令 mounted）时不取 route，**不要在指令外假设它一定拿得到当前路由**。
- 标识必须与后端 `AuthPermission([...])` 及 `sys_menu` 按钮权限逐字一致。
- 表格行内操作用 `TableOperationAction.perm` 字段（见 position 的 `buildPositionRowActions`），由 `renderTableOperationCell` 统一处理，不要自己 v-if。

## 6. 其它常用 hook

`useChart`（图表）、`useTheme`（主题）、`useLayoutHeight` / `useTableHeight`（高度自适应）、`useLoading`、`useCommon`（`getMainScrollEl` 等）、`useAppMode` / `useAppBootstrap`（应用模式与启动）、`useSiteConfig`、`useHeaderBar`、`useFastEnter`、`useCeremony`。**先查 `hooks/core/` 有没有现成的**，再考虑新写。

## 7. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| 页面自己 `ref([])` + `onMounted` 拉列表 + 手写分页 | `useTable` |
| 搜索后调 `fetchData` 期望回第一页 | 用 `getData`（回第一页）；`fetchData` 是当前页 |
| 删除后 `refreshData` 全量刷 | `refreshRemove`（智能页码） |
| 页面 try/catch 再弹一次错误 | `error` 接 `FaTable` + `@retry`；拦截器已提示 |
| 跨页面依赖 useTable 去重 | 去重只在单实例内，跨实例会串台 |
| 手写 ElMessageBox 删除确认 | `confirmDelete` / `confirmBatchDelete` |
| 列设置自己存 localStorage | `useTableColumns` 已按 `table-${route.path}` 持久化 |
| 在失活页面里起定时请求 | 见 [keepalive-and-connections.md](./keepalive-and-connections.md) |
