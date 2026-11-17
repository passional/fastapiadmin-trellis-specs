# 跨层思考指南

> 目的：动手前把数据流与边界契约想清楚。本仓库的 bug 大多发生在**层与层之间**：后端改了字段前端没跟、菜单名与组件名不一致、源码修了产物没修、SQLite 过了 PG 炸。

## 1. 先画数据流

### 业务数据（写 → 读 → 展示）

```
前端表单(FaForm + XxxForm 类型)
  → api/module_x/xxx.ts (request, data/params)
  → controller (AuthPermission + XxxCreateSchema 校验)
  → service (业务规则 + Permission 数据权限 + 缓存失效)
  → crud (CRUDBase, flush 不 commit)  ← 事务边界在 db_getter
  → model (ModelMixin, UTC 时间, JSONType)
  ← OutSchema (DateTimeStr → 本地时区字符串)
  ← ResponseSchema[T] 信封 {code,msg,data,status_code,success}
  ← request 拦截器 (code!==0 → 提示+reject；非 GET 成功自动 toast)
  ← useTable (解包 items/total，current/size ↔ page_no/page_size)
  ← FaTable (loading/empty/error+retry 状态闭环)
```

### 路由与缓存（菜单 → 页面实例）

```
backend/sql/sys_menu.json / 后台「菜单管理」
  → 菜单接口 → menu.store
  → MenuProcessor.mapMenuNode (route_path/route_name/component_path/keep_alive；目录 component=undefined)
  → ComponentLoader (import.meta.glob 匹配 src/views|layouts)
  → RouteTransformer → RouteRegistry.addRoute
  → guards.beforeEach (登录/权限/404 回落)
  → layouts/fa-page-content (唯一 KeepAlive，key=叶子 path)
  → include/exclude 按「组件名」匹配 ← defineOptions({ name }) 必须等于 route_name
  → worktab.store (opened → include；clearAll → keepAliveExclude 驱逐)
```

### 交付（源码 → 线上）

```
frontend/web/src → pnpm build:prod → frontend/web/dist
  → rsync -a --delete → docker/nginx/web/dist (nginx /web)
  → rsync -a --delete → backend/dist (app.frontend("/") 托管)
  → docker 重新打镜像/重传 → 线上
```

对每个箭头问三件事：**数据是什么格式？谁负责校验/转换？这里可能出什么错？**

## 2. 边界清单（本仓库真实易错点）

| 边界 | 契约 | 典型事故 |
| --- | --- | --- |
| schema ↔ model | 字段名/可选性/长度/枚举一致；审计字段继承 `BaseSchema`/`BaseType` | OutSchema 漏字段，前端拿到 undefined |
| service ↔ crud | 数据权限在 **Service**；CRUD 只 flush | Service 忘过滤 → 越权 |
| controller ↔ 前端 api | URL 逐字一致；GET 用 params、写用 data；DELETE 带 body 批删 | 404 / 参数收不到 |
| 响应信封 ↔ `ApiResponse` | `code/msg/data/status_code/success`；`code=0` 成功 | 前端把信封当业务数据（少一层 `.data`） |
| 分页 | `page_no/page_size/items/total/has_next`；`order_by` 是 **JSON 字符串**；page_size ≤ 100 | 排序不生效 / 分页字段对不上 |
| 业务码 ↔ HTTP | `RET.code` → `_CODE_TO_HTTP_STATUS`；10401/10403 对应前端 `ResultEnum` | 新码未登记 → 落 400 安全网 |
| 权限标识 | `module_<域>:<资源>:<操作>`，嵌套子域四段（`module_task:storage:node:query`）；后端 `AuthPermission` + `sys_menu` + 前端 `v-hasPerm` 三处一致 | 按钮不显示 / 接口 403 |
| 时间 | 库存 UTC；出口 `DateTimeStr`/`to_local_datetime` 按 `TIMEZONE`；前端只展示不再加减；Excel 导出前转 naive | 手动 +8 小时叠加；openpyxl 报时区错 |
| 菜单 ↔ 组件名 | `route_name` 唯一且等于 `defineOptions({ name })`；目录 `component=undefined` | KeepAlive include/exclude 落空、重复挂载 |
| 命名不对称 | 后端 `modules/task/storage` ↔ URL `/task/storage` ↔ 前端 `api/module_storage` ↔ `views/module_task/storage` | 只按目录名推断，改漏一处 |
| 端口/代理 | `backend/env/.env.dev SERVER_PORT`（当前 8003）↔ `VITE_API_BASE_URL` ↔ `VITE_APP_WS_ENDPOINT`（直连不走代理） | 改一端 → 全 404 / WS 连不上 |
| 方言 | `JSONType` 三方言；`uq` 不命名；迁移/约束语义要在 PG 验 | SQLite 过了 PG 炸；迁移被 DROP 守卫删 |
| 枚举语义 | 状态 0/1；菜单 1/2/3/4；公告 0/1/2；data_scope **1/2/3**（model docstring 的 1–5 是历史描述） | 新值落到 else 默认分支 |
| 演示模式 | 系统参数 `demo_enable`（非环境变量）→ `RequestLogMiddleware` 拦非 GET | 把「演示环境，禁止操作」当代码故障 |
| src ↔ dist | 三份产物 + 移动端 h5；`--delete` 同步；核特征字符串 | 源码已修线上未修 |
| 连接资源 ↔ 缓存生命周期 | `onActivated`/`onDeactivated`；WS 守卫 `!== CLOSED`；close 前摘回调 | 登出后连接泄漏、重复「连接成功」 |

## 3. 定契约再写代码

每个边界明确：输入格式、输出格式、可能的错误、**谁负责校验**。

本仓库的校验归属（不要重复也不要漏）：
- 字段格式/长度/枚举 → **Pydantic schema**（`field_validator` / `Field(ge/le/max_length)`）
- 身份与操作权限 → **`AuthPermission`**（controller）
- 数据行权限 → **`Permission.filter_query`**（service）
- 业务规则（重复、状态流转、关联约束）→ **service**（抛 `CustomException`）
- 约束冲突/连接故障分类 → **全局异常处理器**
- 前端只做交互级校验与展示兜底，**不重复实现业务规则**

## 4. 真实事故复盘（防回退）

| 症状 | 根因（跨层） | 规则 |
| --- | --- | --- |
| 登出→登录后多条 ws + 多条「连接成功」 | WS 守卫只挡 `OPEN`，CONNECTING 期重入覆盖旧引用 | 守卫用 `ws && ws.readyState !== WebSocket.CLOSED`；close 前摘回调 |
| 源码已修，线上仍重复提示 | 三份 dist 停在修复前 | 重建 + `rsync --delete` + 产物 grep 特征 |
| 切换菜单接口重复请求 | 目录路由挂了壳组件，KeepAlive 多层实例存活 | 目录 `component=undefined`；单层 KeepAlive；不加 `:max` |
| 标签清了但页面还活着 | undefined include 不 prune | 登出靠 `clearAll()` 写 `keepAliveExclude` 驱逐 |
| 生成产物 NameError、后端起不来 | Jinja2 for 块作用域，循环内 set 的标志循环外仍是初值 | 用 `selectattr` 过滤器一次性计算 |
| 启动正常、首次请求 500 | `register_frontend` 检查用 path_conf、挂载硬编码路径 | 检查与挂载同源 `FRONTEND_DIST_DIR`；看 loguru 日志 |
| 迁移永远应用不上 | 给 unique 命名 → 与 MySQL 存量索引名不匹配 → autogen 出 DROP → 被守卫删除 | `uq` 保持匿名；破坏性变更手工 revision + 审查 |
| 「数据库已改、调度器仍报旧原因」 | `reconcile_node_jobs` 未替换旧 job 参数 | 改 node.func/trigger 后确认旧 job 被替换 |
| 新模块接口静默 404 | `discover.py` 只在启动时扫描并缓存 | 生成/改模块后重启后端 |

## 5. 清单

实现前：
- [ ] 画出完整数据流（含菜单→路由→缓存、src→dist）
- [ ] 标出所有边界与格式
- [ ] 决定校验归属（schema / 权限 / service / 全局处理器）
- [ ] 确认枚举/契约的唯一来源，搜过所有消费方（含 `backend/sql`、`backend/templates`）

实现后：
- [ ] 边界值测过（null / 空 / 非法 / min-1 / max+1）
- [ ] 每个边界的错误路径验过（4xx vs 5xx 分类正确）
- [ ] 数据往返一致（创建→详情→列表→编辑→删除）
- [ ] 权限三路径验过（未登录 / 无权限 / 数据权限越权）
- [ ] 消费方都走共享类型/工具，没有本地重复解析
- [ ] 前端 `pnpm ts:check` + `pnpm test` + `pnpm build`；后端 `ruff check` + `pytest`（涉迁移加 PG）
- [ ] 要上线：三份 dist 同步并核到产物特征
- [ ] 测试数据 `e2e_` 前缀已清理；`docs/COVERAGE-PLAN.md` 已同步

## 6. 什么时候写流程文档

功能跨 3 层以上、数据格式复杂、涉及多端（web + app）、或这类功能以前出过 bug → 在任务 `design.md` 里画清数据流与边界，并把沉淀出的契约补进对应层 spec。
