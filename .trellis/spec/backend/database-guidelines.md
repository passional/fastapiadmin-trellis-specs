# 数据库规范（ORM / 查询 / 迁移）

> 事实来源：`app/core/base_model.py`、`app/core/base_crud.py`、`app/core/database.py`、`app/core/permission.py`、`app/scripts/initialize.py`、`app/alembic/env.py`。
> 支持方言：**MySQL / PostgreSQL / SQLite**（`settings.DATABASE_TYPE`）。生产 PG，本地当前 mysql，测试默认 sqlite。

## 1. 模型：继承 ModelMixin

```python
class DictTypeModel(ModelMixin):
    __tablename__: str = "sys_dict_type"
    __table_args__: dict[str, str] = {"comment": "字典类型表"}

    dict_name: Mapped[str] = mapped_column(String(100), nullable=False, index=True, comment="字典名称")
    dict_type: Mapped[str] = mapped_column(String(255), nullable=False, index=True, unique=True, comment="字典类型")
    status: Mapped[int] = mapped_column(Integer, default=0, nullable=False, comment="状态(0:启动 1:停用)")
```

`ModelMixin` 自动提供：`id`(自增主键)、`uuid`(唯一)、`is_deleted`、`created_time`/`updated_time`(UTC，`DateTime(timezone=True)`)、`created_id`/`updated_id`、`deleted_time`/`deleted_id`，以及两个复合索引：

```
ix_<table>_status_deleted    (status, is_deleted)
ix_<table>_created_deleted   (created_time, is_deleted)
```

规则：
- 表名显式写 `__tablename__`，约定 `<域前缀>_<实体>`（`sys_` / `task_` 等）；不写则默认类名小写。
- 每列写 `comment`（中文），状态列注明取值含义。
- 时间列一律 `DateTime(timezone=True)` + `datetime.now(UTC)`，**不要存本地时间**。
- JSON 列统一用 `app/core/base_model.py` 的 **`JSONType`**（`JSONB().with_variant(JSON(), "sqlite", "mysql")`）。**禁止直接硬编码 `JSONB`**——会破坏三方言兼容承诺，SQLite/MySQL 下 create_all 与 autogenerate 直接失败。
- 关系加载：一对多推荐 `selectin`，多对一 `joined`；查询时用 `CRUDBase` 的 `preload=[...]` 传加载选项，不要在异步会话里触发懒加载（会抛 `MissingGreenlet`）。

## 2. 命名约定（autogenerate 可靠性）

`base_model.py::NAMING_CONVENTION`：

```python
{"ix": "ix_%(column_0_label)s",
 "ck": "ck_%(table_name)s_%(constraint_name)s",
 "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
 "pk": "pk_%(table_name)s"}
```

- `ix`/`ck`/`fk`/`pk` **显式命名**：匿名 FK 在 MySQL 落库为 `ibfk_1` 这类自动名，跨方言渲染/对比/batch 重建都不可靠。
- **`uq` 特意不配置**：MySQL 下 `unique=True` 渲染为列名索引（如 `code`），一旦命名成 `uq_sys_dept_code` 会与存量库不匹配，autogenerate 生成 `remove_index + add_uq`，又被 dev 启动的 DROP 拦截机制删掉，导致迁移永远无法自动应用。**不要给 unique 约束加命名约定**。

## 3. 事务边界与 CRUD 用法

- **事务在 HTTP 层**：`dependencies.py::db_getter` 用 `async with async_db_session() as session, session.begin()`，一个请求内所有 SQL 共享同一事务（读也走这个事务）。
- `CRUDBase` 只 `flush`，**绝不 `commit`**。
- 需要独立事务的后台任务用 `async with async_db_session() as session, session.begin():`（见 `router_class.py::_write_operation_log_async`）。
- 同步引擎（`create_sync_engine`）只给 APScheduler JobStore 和代码生成器 Inspector（经 `asyncio.to_thread`）；**请求侧一律异步**。

`CRUDBase[Model, CreateSchema, UpdateSchema]` 可用方法：

| 方法 | 说明 |
| --- | --- |
| `get(**kwargs)` / `get_or_404(id=..., msg=...)` | 单条；查不到抛 `CustomException(code=RET.NOT_FOUND)` → HTTP 404 |
| `exists(**kwargs)` / `count(**kwargs)` | 用 COUNT，不加载整行 |
| `get_list(search=..., order_by=..., preload=..., load_columns=...)` | 不分页列表，默认 `[{"id":"asc"}]` |
| `page(offset, limit, order_by, search=..., out_schema=...)` | COUNT + 数据两趟；COUNT 复用 WHERE 但不带 loading options |
| `create(data)` / `update(id, data)` | 自动填充 `created_id`/`updated_id` |
| `delete(ids)` | 模型有软删除三字段则软删，否则硬删 |
| `set(ids, **kwargs)` | 批量改字段（如状态） |
| `clear()` | 清空（慎用） |

查询条件：`**kwargs` 按 `字段名=值` 传；操作符用元组，由 `_build_conditions` + `_resolve_condition` 解析：

| 操作符 | 例 | 语义 |
| --- | --- | --- |
| 比较 | `("ge", 1)`、`("ne", 0)` | `eq == ne != gt > ge >= lt < le <=` |
| `like` | `("like", "张")` | `LIKE '%张%'`（自动包 `%`） |
| `in` | `("in", [1,2])` | **空集合 → `false()` 永假条件**（不是忽略） |
| `between` | `("between", [a, b])` | 需恰好 2 元素 |
| `date` / `month` | `("date", "2026-11-17")` | 当天 `[0:00, 次日0:00)` / 当月 `[1号, 下月1号)` |
| `None` / `not None` | `("None", None)` | IS NULL / IS NOT NULL |

Service 层统一用 `search_to_dict(QueryParam)` 生成这个字典（自动合并 `_start`/`_end` 时间对为 between、按 `json_schema_extra={"q": op}` 转元组）。

`include_deleted=True` 才会查软删除行；默认自动加 `is_deleted == False`。

## 4. 软删除

- 判定：模型同时有 `is_deleted` / `deleted_time` / `deleted_id` 三字段（`_supports_soft_delete`）。
- 删除写入 `is_deleted=True`、`deleted_time=datetime.now(UTC)`、`deleted_id=当前用户`。
- **清理测试数据时**：先走 API 删除（软删），再按需硬删遗留软删行（见 [../project/testing-guidelines.md](../project/testing-guidelines.md)）。
- 不要手写 `DELETE FROM` 绕过软删除语义。

## 5. 数据权限（Service 层责任）

`CRUDBase` 只自动填充 `created_id`/`updated_id`，**不按这些字段过滤**。过滤由 Service 层用 `app/core/permission.py::Permission` 完成：

```python
query = select(Model)
query = await Permission(Model, self.auth, self.db).filter_query(query)
```

实现口径（`Permission` 常量）：`1` 仅本人、`2` 本部门及以下、`3` 全部；超管不过滤。
- 模型没有 `created_id` 时不过滤（返回 None）。
- `UserModel` 特例：按 `dept_id` / `id` 过滤而非 `created_id`。
- 子部门计算失败会**降级为仅本部门**并打 warning（最小授权，不静默扩大范围）。

> **Service 忘记过滤 = 越权风险。** 新增列表/详情接口时必须显式过一遍数据权限，评审重点查这条。

## 6. 迁移（Alembic）

命令：

```bash
cd backend
uv run main.py revision -m "说明" --env=dev   # autogenerate
uv run main.py upgrade --env=dev              # 应用到 head
```

启动时自动做的事（`app/scripts/initialize.py::init_db`）：

1. **空库判定**：以 `sys_menu` 为哨兵表；不存在 → `create_tables()` 按模型建全表，若仓库已有迁移历史则 `alembic stamp head`。
2. 非空库 → 先清理「僵尸 revision」（库里有、仓库脚本已不存在的版本号），再 `upgrade head`。
3. **dev 环境**额外 autogenerate：无变更则 `env.py::process_revision_directives` 拦截不产文件；有变更则生成并自动应用。
4. **破坏性 DROP 守卫**：新迁移的 `upgrade` 段若含 `op.drop_*` 或手写 `DROP`，文件会被**直接删除并中止自动应用**，日志提示改用手工 `revision` + 审查 + `upgrade`。（`downgrade` 段的 DROP 是正常回滚，不拦。）
5. 种子数据：按依赖顺序导入 `backend/sql/*.json`，**表已有数据则整体跳过**（幂等）；`sys_menu`/`sys_dept` 递归处理 children。

现状与注意：
- `app/alembic/versions/` 当前**有已提交的迁移脚本**（2 个）。`initialize.py` 注释里「本仓库不提交迁移历史」是历史口径，以实际目录为准。
- 改了 model **必须**生成并应用迁移；dev 自动生成只是便利，**含 DROP 的变更必须手工生成并审查**。
- `env.py` 用 `include_object` 排除非业务表，`compare_server_default=True`；MySQL 下若出现误报可按注释移除该项。
- 涉及迁移 / 方言特有语义（JSONB、约束、索引名）时，**必须在 PostgreSQL 上验**：`TEST_DB_TYPE=postgres UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q tests/test_alembic_migrations_unit.py tests/test_alembic_offline.py`。SQLite 会掩盖差异。
- 测试期间不要往 `versions/` 写文件（`conftest.py` 有快照守卫比对污染）。

## 7. Redis

- 走 `app/core/redis_crud.py::RedisCURD`，key 前缀用 `app/common/enums.py::RedisInitKeyConfig`（如 `USER_SESSION`、`SYSTEM_CONFIG`）。
- 会话结构契约见 `base_schema.py::SessionInfoSchema`；改字段要同步认证、在线用户等消费方。
- 缓存失效由 Service 负责（如字典改动后清字典缓存），CRUD 不管缓存。

## 8. 反模式

| 反模式 | 后果 / 正确做法 |
| --- | --- |
| JSON 列写 `JSONB` | SQLite/MySQL 建表失败；用 `JSONType` |
| 给 unique 约束命名（`uq_...`） | 与存量 MySQL 索引名不匹配 → 迁移被 DROP 守卫删掉，永远应用不上 |
| CRUD 里 `commit` | 破坏请求级事务；只 `flush` |
| Service 列表不过数据权限 | 越权；用 `Permission.filter_query` |
| 异步会话里懒加载关系 | `MissingGreenlet`；用 `preload` |
| 存本地时间 / 手动 +8 小时 | 时区契约破坏；存 UTC，展示走 `DateTimeStr` |
| 只靠 dev 自动迁移做破坏性变更 | 被守卫删除且静默；手工 revision + 审查 |
| 只在 SQLite 验迁移 | PG 上才暴露的漂移漏网 |
