# 后端质量规范与已知坑

> 事实来源：`backend/pyproject.toml`（ruff / pytest / coverage）、`backend/pyrightconfig.json`、`SKILL.md` §8/§12、各模块代码注释里的守卫约定。

## 1. 工具口径

| 工具 | 配置 | 命令 |
| --- | --- | --- |
| ruff | `line-length = 200`、`indent-width = 4`、双引号、select `FAST,F,E,W,I,B,C4,UP`；ignore `E501,B008,C901,B904,B009,B010,E712,W191,W293` | `cd backend && ruff check`（`fix = true`，会自动修） |
| pytest | `asyncio_mode = "auto"`、`asyncio_default_fixture_loop_scope = "function"` | `UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q` |
| coverage | `[tool.coverage.run]` 口径仅「支持范围」（无服务协议的 FTP/SFTP/OBS/OSS/COS 已剔除） | `pytest --cov=app` |
| pyright | `backend/pyrightconfig.json` | 编辑器/CI 类型检查 |

- `B008` 被 ignore 是因为 FastAPI 依赖注入需要在默认值里调用 `Depends()`/`Security()`——**这是框架惯例，不要为了消 lint 改成别的写法**。
- `tests/conftest.py` 允许 `E402`（必须先设置环境变量再 import 应用）。
- 异步用例不需要 `@pytest.mark.asyncio`（auto 模式）。

## 2. 必须遵守的模式

- 分层与契约见 [api-conventions.md](./api-conventions.md)；错误写法见 [error-handling.md](./error-handling.md)；日志见 [logging-guidelines.md](./logging-guidelines.md)。
- 类型注解齐全：`Mapped[...]`、`Annotated[..., Depends/Security/Query/Body/Path]`、返回类型；Schema 用 Pydantic v2（`field_validator` / `model_validator` / `ConfigDict(from_attributes=True)`）。
- 泛型基建优先：`CRUDBase[Model, Create, Update]`、`ResponseSchema[T]`、`PageResultSchema[T]`、`DateTimeStr`。
- 文档字符串写中文，说明参数/返回/异常（沿用现有模块风格）。
- `core/` 不得在导入期依赖业务模块——需要时用**函数内延迟导入**（`permission.py`、`router_class.py` 注释「守卫不变式 3」）。

## 3. 禁止模式

| 禁止 | 原因 |
| --- | --- |
| controller 写业务 / 直接查库 | 串层，权限与事务边界失控 |
| CRUD `commit` | 事务在 `db_getter`，只 `flush` |
| Service 列表/详情不过数据权限 | 越权 |
| `CustomException(status_code=...)` | 构造器不接受；HTTP 由 code 推导 |
| 宽泛 `except` 吞掉 `CustomException` / 原始异常 | 4xx/5xx 分类失效，监控失明 |
| JSON 列硬编码 `JSONB` | 破坏三方言兼容；用 `JSONType` |
| 给 unique 约束命名 | MySQL 存量索引名不匹配 → 迁移被 DROP 守卫删除 |
| 存本地时间 / 手动 +8 小时 | 时区契约：存 UTC，展示走 `DateTimeStr` / `to_local_datetime` |
| `print` / f-string 日志 / 明文敏感字段 | 见 logging-guidelines |
| 事件循环里用同步引擎 | 阻塞；同步引擎只给调度器/生成器 Inspector |
| 绕过 husky/lint-staged 提交 | 前端 hook 会格式化，绕过导致风格漂移 |

## 4. 已知坑（踩过，勿回退）

### 4.1 代码生成器：Jinja2 for 块作用域

**症状**：生成的 `schema.py` 漏 `validator` import，产物 `NameError`，后端起不来。
**根因**：Jinja2 `{% for %}` 块作用域隔离，循环内 `{% set %}` 累计的布尔标志在循环外仍是初值。
**规则**：用过滤器一次性计算。现行正确写法（`backend/templates/python/schema.py.jinja2:8`）：

```jinja2
{% set has_date_validator = columns | selectattr('python_type', 'equalto', 'date') | list | length > 0 %}
```

模板清单：`templates/python/{__init__,controller,crud,model,package_init,plugin.toml,schema,service}.jinja2`、`templates/ts/api.ts.jinja2`、`templates/vue/index.vue.jinja2`。**改模板后必须实际生成一次并启动后端验证产物**。

### 4.2 生成/修改模块后必须重启后端

`core/discover.py` 只在启动时扫描 `app/plugin/module_*` 并缓存；dev reload 只监听文件变更。不重启 → 新模块静默 404。

### 4.3 前端静态托管路径必须同源

`app/__init__.py::register_frontend` 的**检查与挂载都要用 `path_conf.FRONTEND_DIST_DIR`**（`backend/dist`）。曾检查用 path_conf、挂载硬编码 `frontend/web/dist` → `check_dir=False` 启动不报错、**首次请求才 500**，只能看 `backend/logs/fastapiadmin.log` 定位。

### 4.4 时间与 Excel

- Excel 导出前把 aware datetime 转 naive（openpyxl 不支持时区）。
- 导入模板中文性别映射：男→0、女→1、未知→2。
- 空/损坏 Excel 返回 **400**，不允许 500。
- 相关回归：`tests/test_datetime_display_timezone.py`。

### 4.5 SQL 控制台既有约束

`app/modules/ops/dbconsole/service.py`：结果最多 **200 行**（`_MAX_ROWS`，多取一行判 truncated）、敏感列打码、查询 **30s 超时**、多语句/未确认写操作有守卫。**不要为了「显示方便」放宽这些约束或给裸 DATETIME 加 8 小时。**

### 4.6 调度器 reconcile

`reconcile_node_jobs` 启动时刷新已存在 job。改 `node.func` / `trigger` 后必须确认**旧 job 参数被替换**，否则「数据库已改、调度器仍报旧原因」。回归见 `tests/test_scheduler_reconcile.py`。

### 4.7 演示模式不是故障

系统参数 `demo_enable` 开启时，`RequestLogMiddleware` 拦截非 GET（白名单路径/IP 除外）返回「演示环境，禁止操作」。测写接口前先确认环境。

## 5. 提交前自检清单

- [ ] `ruff check` 通过
- [ ] `UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q` 通过（涉及迁移/方言语义时加跑 PG）
- [ ] 新接口：权限标识 + `sys_menu` 种子 + 前端 API/按钮三处对齐
- [ ] 改 model：迁移已生成并应用；破坏性变更走手工 revision + 审查
- [ ] 新模块/生成代码：后端已重启验证路由生效
- [ ] 列表/详情接口过了数据权限
- [ ] 错误只抛 `CustomException(code=...)`，新 RET 码已登记映射
- [ ] 日志用 `{}`，敏感字段已脱敏
- [ ] 新测试数据用 `e2e_` 前缀并已清理
- [ ] `docs/COVERAGE-PLAN.md` 覆盖状态已同步
