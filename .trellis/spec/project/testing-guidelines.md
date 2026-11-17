# 测试用例与测试数据规范

> 来源：`SKILL.md` §11 + `backend/tests/conftest.py`、`backend/pyproject.toml`、`docs/COVERAGE-PLAN.md`。
> 覆盖矩阵、执行记录与剩余缺口在 **`docs/COVERAGE-PLAN.md`**（SKILL.md 写的 `docs/TEST-COVERAGE.md` 不存在）。改测试状态时同步更新该文档。

## 1. 用例设计：新增功能至少覆盖六类

| 类别 | 要求 |
| --- | --- |
| 等价类 | 合法 / 非法 / 空值 / null / 格式错误 |
| 边界值 | `min-1 / min / min+1 / max-1 / max / max+1` |
| 输入域 | 类型、长度、枚举、关联字段、状态流转 |
| 正交组合 | 角色 × 状态、数据权限 × 状态、协议 × 分片参数等 |
| 接口 | 正常路径、非法输入、越权、重复提交、批量部分失败 |
| UI | 空值校验、成功创建、按钮 / 弹窗 / 富文本 / 下拉等真实交互 |

统一契约的边界值直接取 [../backend/api-conventions.md](../backend/api-conventions.md) 第 6 节（账号 3–32 位、角色编码 2–64 位等），不要自己另定一套。

## 2. 已有测试资产（先找现成的，别新建重复文件）

后端 `backend/tests/`（约 199 个测试文件）：

| 资产 | 用途 |
| --- | --- |
| `conftest.py` | 共享 fixture：`test_client`（session 级 TestClient）、`assert_route`；自建临时库 + mock Redis |
| `test_input_domain_boundaries.py` | 输入域 / 边界值 / 正交组合参数化用例 |
| `test_base_crud_conditions.py`、`test_base_crud_edges.py` | CRUD 基类条件构造与边界 |
| `test_scheduler_*.py`、`test_zz_scheduler_lifecycle.py` | 调度器分支、事件、reconcile、生命周期 |
| `test_storage_*.py`、`test_local_storage_adapter.py` | 存储工厂 / 节点 / 浏览 / 工作流 / 本地适配器（含路径穿越） |
| `test_ops_ssh_*.py`、`test_dbconsole_*.py` | SSH 与 SQL 控制台（含 SQL 守卫） |
| `test_datetime_display_timezone.py` | 时区展示契约 |
| `test_alembic_migrations_unit.py`、`test_alembic_offline.py` | 迁移守卫 / 离线渲染 |
| `test_integration_*_pg.py` | **真实 PostgreSQL 集成**（SSH / 存储 / 调度器） |

前端 `frontend/web/src/__tests__/`（15 个 spec）：`route-invariants`、`http-error-message`、`message-dedupe`、`permission-check`、`query-params`、`fa-table-error`、`form-validators`、`column-i18n`、`responsive-span`、`data-url` 等。

## 3. 测试环境口径

- `conftest.py` 默认用 **SQLite 临时库**（`tempfile` 建 `.db`），开箱即跑；`TEST_DB_TYPE=postgres` 切真实 PG（`TEST_DB_HOST/PORT/USER/PASSWORD/NAME` 可覆盖）。
- **SQLite 与 PG 在类型 / 约束 / 迁移上行为不一致**：涉及迁移或数据库特有语义时必须在 PG 上验（对应 `test_integration_*_pg.py`）。
- 测试固定注入：`POOL_SIZE=1`、`MAX_OVERFLOW=1`、假 `SECRET_KEY`、`CAPTCHA_ENABLE=False`。
- Redis 用 dict 版 mock（支持 get/set/delete/exists/keys/ttl/expire），登录写入的 session 可在后续请求读到。
- pytest 配置：`asyncio_mode = "auto"`（`backend/pyproject.toml`），异步用例不需要手写 `@pytest.mark.asyncio`。
- `conftest.py` 在 import 时刻快照 `alembic/versions/` 目录，供迁移守卫比对 dev 自动迁移是否污染仓库——**不要在测试里往 versions/ 写文件**。

## 4. 测试数据规范

- 统一 **`e2e_` 前缀**，每次运行生成唯一后缀（避免并发/重跑串数据）。
- 由 AI **创建、使用、清理**；不要依赖用户手工准备测试数据。
- 清理顺序：先 API 删除 → 再按需硬删除遗留的软删除行 → 清理对象存储桶/对象。
- 不把真实密码、SSH 私钥、第三方 Key 写进测试文件或提交记录。
- 对象存储只有 **MinIO S3** 可测；FTP/FTPS、SFTP、OBS、OSS、COS 无测试环境，不作为验收项（已从覆盖率分母剔除）。

## 5. 执行命令

```bash
# 后端（--no-sync 避免测试时改动锁文件；独立 cache 目录避免污染）
cd backend && UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q

# 需要真实 PG 语义时
cd backend && TEST_DB_TYPE=postgres UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q tests/test_integration_storage_node_pg.py

# 前端
cd frontend/web && pnpm test && pnpm ts:check && pnpm build
```

验收至少跑「单元/接口 + 类型/构建」两类；只跑其中一类不算验证完成。

## 6. 浏览器自动化

- 使用**隔离的临时浏览器 profile**；启用 `--password-store=basic --use-mock-keychain --disable-sync`，不要复用用户日常浏览器 / 钥匙串。
- 动态表单按**字段名**定位，不要写死输入框索引（菜单类型切换会改变字段顺序）。
- Element Plus 下拉 / 富文本优先用**真实鼠标坐标点击**，而不是只改 DOM value（只改 value 不触发组件内部状态）。
- 写操作要真实提交并断言结果，再把 `e2e_` 数据清理干净；「页面能打开」不是通过标准。

## 7. 反模式

| 反模式 | 后果 |
| --- | --- |
| 只在 SQLite 上验迁移 / JSONB / 约束 | PG 上才暴露的漂移漏网（`JSONType` 降级为 JSON 掩盖差异） |
| 测试数据用固定名（如 `test_user`） | 重跑 / 并发互相污染，失败不可复现 |
| 断言「接口返回 200」就结束 | 没验业务 code / data 结构 / 权限路径，等于没测 |
| 把真实 Key 写进测试让它跑通 | 凭据入库，违反红线 |
| 改了测试状态不更新 `docs/COVERAGE-PLAN.md` | 覆盖矩阵与实际脱节，下一轮重复劳动 |
