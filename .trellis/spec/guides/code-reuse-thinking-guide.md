# 复用思考指南

> 目的：写新代码前先问「这是不是已经有了」。本仓库已经沉淀了 Fa 组件库、hooks 开发套件、CRUD/响应/权限基建——**重复造轮子是这里最常见的一致性 bug 来源**。

## 1. 先搜再写

```bash
# 前端：组件 / hook / 工具
ls frontend/web/src/components/{forms,tables,modal,display,containers,cards,actions}
ls frontend/web/src/hooks/core
grep -rn "export function\|export const" frontend/web/src/utils/index.ts
grep -rn "关键词" frontend/web/src --include="*.vue" --include="*.ts"

# 后端：基建 / 工具 / 既有模块
ls backend/app/core backend/app/utils
grep -rn "def .*关键词" backend/app
```

| 问题 | 是 → 怎么做 |
| --- | --- |
| 已有相似组件/hook？ | 用它或扩展 props，不要另写一个 |
| 别的页面已有同样写法？ | 照那个样板（`views/module_system/position`、`dict`） |
| 这段逻辑会被第二处用？ | 放进 `utils/`（前端）或 `app/utils`、`app/core`（后端） |
| 正在从别的文件复制代码？ | **停**，抽共享 |

## 2. 本仓库已有的「唯一来源」（不要另起一套）

### 前端

| 需求 | 用现成的 | 位置 |
| --- | --- | --- |
| 列表数据/分页/缓存/去重 | `useTable` | `hooks/core/useTable.ts` |
| 列显隐/排序/持久化 | `useTableColumns` | 同上 |
| 多选/批删状态 | `useTableSelection` | 同上 |
| 弹窗 + 表单 + 提交 | `useCrudDialog` + `useCrudForm`（省每页约 60 行） | 同上 |
| 导入导出弹窗状态 | `useImportExport` | 同上 |
| 删除/批删/状态切换确认 | `confirmDelete` / `confirmBatchDelete` / `confirmToggleStatus` | `hooks/core/useConfirm.ts` |
| 权限判断 | `v-hasPerm` / `useAuth().hasAuth` / `checkPerm` | `directives/permission`、`utils/checkPerm.ts` |
| 表格/表单/弹窗/空态/状态标签 | `FaTable`、`FaForm`、`FaSearchBar`、`FaDialog`、`FaEmptyState`、`FaStatusTag`… | `components/**` |
| 操作列渲染 | `renderTableOperationCell` + `TableOperationAction` | `@utils` |
| 状态列统一映射 | `resolveStatusColumns` | `@utils` |
| 查询参数清洗 | `cleanEmptyArrayParams`、`stripPaginationParams` | `@utils` |
| 时间/数字/文件大小格式化 | `formatDateTime` 等共享函数 | `@utils` |
| 消息提示 | 拦截器 + `utils/message.ts`（80ms 同级别合并） | **页面不要再叠同义提示** |
| 刷新用户/字典/配置/路由 | `refreshAppCaches(opts)` | `store/index.ts` |
| 颜色/字号/间距/圆角/阴影 | `design/tokens.json` → 生成文件 → 语义 class | **禁止裸 hex / 随手值** |

### 后端

| 需求 | 用现成的 | 位置 |
| --- | --- | --- |
| CRUD 查询/分页/软删除 | `CRUDBase[Model, Create, Update]` | `core/base_crud.py` |
| 通用字段/索引/命名约定 | `ModelMixin` | `core/base_model.py` |
| 跨方言 JSON 列 | `JSONType` | `core/base_model.py`（**不要硬编码 JSONB**） |
| 响应信封/分页结果 | `ResponseSchema[T]`、`SuccessResponse`、`PageResultSchema[T]` | `common/response.py`、`core/base_schema.py` |
| 业务错误 | `CustomException(msg, code)` + `RET` | `core/exceptions.py`、`common/enums.py` |
| HTTP 状态推导 | `_CODE_TO_HTTP_STATUS` / `resolve_http_status` | **新增 RET 码必须登记，不要另写映射** |
| 数据权限过滤 | `Permission.filter_query` | `core/permission.py` |
| 查询模型 → 条件字典 | `search_to_dict` + `json_schema_extra={"q": op}` | `utils/common_util.py` |
| 时间/手机号/邮箱类型 | `DateTimeStr`、`DateStr`、`TimeStr`、`Telephone`、`Email` | `core/validator.py` |
| 时区转换 | `to_local_datetime` / 响应编码器 | **不要手动 +8 小时** |
| 操作日志 + 脱敏 | `OperationLogRoute` + `_SENSITIVE_KEYS` | `core/router_class.py` |
| 仅超管 | `@require_superadmin` | `core/exceptions.py` |
| Redis 访问 | `RedisCURD` + `RedisInitKeyConfig` | `core/redis_crud.py`、`common/enums.py` |
| 日志 | `logger`（`{}` 占位符） | `core/logger.py` |

## 3. 什么时候抽象 / 什么时候不

**抽象**：同样代码出现 3 次以上；逻辑复杂到会有 bug；多个模块都要用；是契约（字段名、状态语义、错误码）。
**不抽象**：只用一次；一行 trivial；抽象后比重复更难懂。

契约类东西**即使只出现两次也要收敛到唯一来源**——因为它们会漂移：
- 权限标识、菜单类型、状态枚举、分页字段名、响应信封字段；
- 敏感字段脱敏集合；
- 设计 token 与 Logo 副本。

## 4. 本仓库真实的「不对称机制」陷阱

两套机制产出同一份东西时，结构性变更只会经过自动那条，手工那条静默漂移。

| 不对称 | 表现 | 防法 |
| --- | --- | --- |
| `modules/` 手工注册 vs `plugin/` 自动发现 | 新模块建在 `modules/` 却以为会自动生效 → 静默 404 | `modules/` 必须改 `api/v1/routers.py::DOMAIN_CONTROLLERS`；`plugin/` 才走 `discover.py` |
| 三份 dist 副本 | 源码修了，`docker/nginx/web/dist`、`backend/dist` 还是旧的 | `pnpm build:prod` + `rsync -a --delete` 同步 + **产物 grep 特征** |
| token 三端生成文件 | 只改 Web 生成文件 | 改 `design/tokens.json` + `node scripts/design-tokens/generate.mjs`，检查三端 diff |
| Logo 多副本 | 只改一端 | `node scripts/check-brand-consistency.mjs`（sha256 比对 web/app/docs/backend 副本） |
| 代码生成模板 python/ts/vue 三套 | 改了后端契约，模板没跟 → 生成产物与手写模块不一致 | 改模板后**实际生成一次并启动验证** |
| 中英词条 / 中英文档 | 只补 zh | `locales/langs/{zh,en}.json`、`frontend/docs/src/{guide,en/guide}` 同步 |
| web 与 app 两套 API 层 | 跨端需求只改 web | 同时评估 `frontend/web` 与 `frontend/app` |

## 5. 枚举/字面量新增值：搜全所有分支

Python 的 `if/elif/else` 与 TS 的 `switch` 都没有穷尽检查。给枚举加值时，**所有按它分支的地方都要显式补**，否则新值静默落到 `else` 拿错默认。

本仓库要一起查的位置（以菜单类型 `1 目录 / 2 菜单 / 3 按钮 / 4 外链` 为例）：

```bash
grep -rn "MenuTypeEnum\|menu_type" backend/app frontend/web/src backend/sql
```

- 后端：`common/enums.py`、`modules/system/menu/{schema,service}.py`、`scripts/initialize.py`（递归表判定）
- 前端：`enums/system/menu.enum.ts`、`router/MenuProcessor.ts`（目录 component 置 undefined 的分支）、菜单管理页面表单（类型切换会改变字段顺序）
- 种子：`backend/sql/sys_menu.json`

同理适用于 `RET` 业务码（新增必须登记 `_CODE_TO_HTTP_STATUS`）、`data_scope`（1/2/3）、公告状态（0/1/2）。

## 6. 批量修改后

1. 回顾：所有实例都改到了吗？
2. 再 grep 一遍旧值/旧名（含 `backend/sql`、`backend/templates`、dist 副本）。
3. 问：这是否该抽成共享？
4. 跑验证：后端 `ruff check` + `pytest`；前端 `pnpm ts:check` + `pnpm test` + `pnpm build`。

## 7. 提交前清单

- [ ] 搜过现有组件/hook/utils/基建，没有重复实现
- [ ] 没有该共享却复制粘贴的逻辑
- [ ] 常量/枚举/契约只有一处定义
- [ ] 相似页面遵循同一结构（FaTableHeader + FaTable + FaDialog）
- [ ] 枚举新增值已补全所有分支（含种子与模板）
- [ ] 多副本产物/词条/Logo 已同步
- [ ] 没有把「故意设计的守卫」当重复代码删掉（先读注释）
