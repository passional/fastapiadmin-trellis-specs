# 后端接口与分层契约

> 事实来源：`app/modules/system/dict/`（标准样板）、`app/core/base_schema.py`、`app/common/response.py`、`app/core/dependencies.py`、`app/core/router_class.py`。
> 规则来源：`SKILL.md` §3、§12、§14。

## 1. 四层职责（不要串层）

```
controller.py  →  service.py  →  crud.py  →  model.py
   绑参数/权限      业务规则/数据权限    SQL/软删除     ORM 映射
   包响应信封      抛 CustomException   flush 不 commit
        schema.py 横跨各层：Create / Update / Out / QueryParam
```

| 层 | 必须做 | 禁止做 |
| --- | --- | --- |
| controller | 声明权限、绑定参数、调 Service、`SuccessResponse` 包装 | 写业务判断、直接查库、自己拼错误响应 |
| service | 业务校验、**数据权限过滤**、缓存失效、抛业务异常 | 返回裸 ORM 对象给 controller（要转 OutSchema） |
| crud | 构造查询、软删除、`flush` | `commit`（事务在 HTTP 层）、按 `created_id` 做权限过滤 |
| schema | 字段约束、`field_validator`、查询操作符标记 | 在 schema 里查库 |

## 2. controller 写法（照抄 dict 样板）

```python
DictRouter = APIRouter(route_class=OperationLogRoute, prefix="/dict", tags=["字典管理"])

@DictRouter.get("/type/list", summary="查询字典类型",
                response_model=ResponseSchema[PageResultSchema[DictTypeOutSchema]])
async def get_type_list_controller(
    auth: Annotated[AuthSchema, Security(AuthPermission(["module_system:dict_type:query"]))],
    page: Annotated[PaginationQueryParam, Depends()],
    search: Annotated[DictTypeQueryParam, Query()],
    db: Annotated[AsyncSession, Depends(db_getter)],
) -> JSONResponse:
    result = await DictTypeService(auth, db).page(
        page_no=page.page_no, page_size=page.page_size,
        search=search, order_by=page.order_by,
    )
    return SuccessResponse(data=result, msg="查询字典类型列表成功")
```

要点：
- `route_class=OperationLogRoute` **必带**（自动操作日志 + 脱敏，见 §7）。
- 每个端点都要 `summary`（中文）与 `response_model=ResponseSchema[T]`。
- 创建端点加 `status_code=status.HTTP_201_CREATED`。
- 路径参数用 `Annotated[int, Path(description=..., ge=1)]`；批量删除用 `Body` 收 `list[int]`（DELETE 带 body 是本项目既定写法）。
- 需要 Redis 的端点注入 `redis: Annotated[Redis, Depends(redis_getter)]`，并作为**方法参数**传给 Service（不是每个端点都用 Redis）。
- 只需鉴权不需要 auth 对象时，用 `dependencies=[Security(AuthPermission([...]))]`（见 `monitor/online/controller.py`）。

## 3. 权限标识

- 基线三段式：`module_<域>:<资源>:<操作>`，如 `module_system:dict_type:create`。
- **嵌套子域会多一段**：`module_task:storage:node:query`（`task/storage/node`）。不是所有标识都三段，前后端必须**逐字一致**。
- 操作词沿用现有集合：`query` / `detail` / `create` / `update` / `delete` / `patch` / `export` / `import`。
- `AuthPermission` 内部：超管直接放行；支持 `*` / `*:*:*` 通配。
- 仅超管可用的 Service 方法加 `@require_superadmin`（`app/core/exceptions.py`），非超管抛 403「仅平台管理员可操作」。
- 前端对应：按钮 `v-hasPerm="'module_system:dict_type:create'"`，代码内 `useAuth().hasAuth(...)`。
- 标识必须同时写进 `sys_menu`（后台「菜单管理」按钮权限，种子在 `backend/sql/sys_menu.json`），否则前端按钮不显示。

## 4. 响应信封（前后端唯一契约）

`app/common/response.py`：

```python
class ResponseSchema[T](BaseModel):
    code: int = RET.OK.code          # 0
    msg: str = RET.OK.msg
    data: T | None = None
    status_code: int = 200           # HTTP 状态，写进 body 供前端判断
    success: bool = True
```

- 成功：`return SuccessResponse(data=..., msg="...")`；无数据时 `SuccessResponse(msg="...")`。
- 失败：**抛 `CustomException`**，由全局处理器转 `ErrorResponse`，不要在 controller 里手拼错误 JSON。
- 分页 `data` 统一为 `PageResultSchema[T]`：`items` / `total` / `page_no` / `page_size` / `has_next`。
- 前端全局类型 `ApiResponse<T>` / `PageResult<T>`（`frontend/web/src/types/global.d.ts`）字段与上面一一对应，**改字段名必须两端同步**。
- 流式/文件/重定向用 `StreamResponse` / `RedirectContentResponse` 等既有封装。

## 5. Service / CRUD / Schema 写法

Service（实例承载 auth 上下文）：

```python
class DictTypeService:
    def __init__(self, auth: AuthSchema, db: AsyncSession) -> None:
        self.auth = auth
        self.db = db

    async def detail(self, id: int) -> DictTypeOutSchema:
        obj = await DictTypeCRUD(self.auth, self.db).get_or_404(id=id)
        return DictTypeOutSchema.model_validate(obj)
```

> 历史写法 `XxxService.method_service(auth=...)` 已废弃，统一 `XxxService(auth, db).method(...)`。

CRUD：

```python
class DictTypeCRUD(CRUDBase[DictTypeModel, DictTypeCreateSchema, DictTypeUpdateSchema]):
    def __init__(self, auth: AuthSchema, db: AsyncSession) -> None:
        super().__init__(model=DictTypeModel, auth=auth, db=db)
```

Schema 四件套（`dict/schema.py`）：

```python
class DictTypeCreateSchema(BaseModel):
    dict_name: str = Field(..., min_length=1, max_length=64, description="字典名称")
    status: int = Field(default=0, ge=0, le=1, description="状态(0:正常 1:停用)")

    @field_validator("dict_name")
    @classmethod
    def validate_dict_name(cls, value: str): ...

class DictTypeUpdateSchema(DictTypeCreateSchema): ...          # 继承即可
class DictTypeOutSchema(DictTypeCreateSchema, BaseSchema):     # 带审计字段
    model_config = ConfigDict(from_attributes=True)

class DictTypeQueryParam(BaseQueryParam):
    dict_name: str | None = Field(None, max_length=100, json_schema_extra={"q": "like"})
    dict_type: str | None = Field(None, max_length=100, json_schema_extra={"q": "eq"})
```

查询参数约定：
- `json_schema_extra={"q": "<op>"}` 标记操作符，`app/utils/common_util.py::search_to_dict` 会转成 `("op", value)` 交给 `CRUDBase._build_conditions`；支持比较符与 `like/in/between/date/month/None/not None`（完整语义见 [database-guidelines.md](./database-guidelines.md) §3）。
- `BaseQueryParam` 自带 `created_time` / `updated_time` 范围（前端传 `["start","end"]`，自动转 `between`）；需要创建人/更新人过滤再继承 `UserByQueryParam`。
- Service 里统一 `search=search_to_dict(search)`，不要手写 `model_dump`。

分页参数 `PaginationQueryParam`：`page_no`(≥1，默认1)、`page_size`(1–100，默认10)、`order_by`（**JSON 字符串**，如 `[{"id":"asc"}]`，validator 会解析；None → `[{"id":"asc"}]`）。Service 里换算 `offset = (page_no - 1) * page_size`。

## 6. 统一业务契约（SKILL.md §14，前后端共用）

| 项 | 契约 |
| --- | --- |
| 用户账号 | **3–32 位**，字母开头，仅字母/数字/`_ . -` |
| 角色编码 | **2–64 位**，字母开头，仅字母/数字/下划线 |
| 角色数据权限 | `1` 仅本人 / `2` 本部门及以下 / `3` 全部（`core/permission.py` 实现口径） |
| 通用状态 | `0` 启用 / `1` 停用 |
| 菜单类型 | `1` 目录 / `2` 菜单 / `3` 按钮 / `4` 外链 |
| 公告类型 | `1` 通知 / `2` 公告；状态 `0/1/2` |
| 存储对象 key | 不接受 `..`；绝对路径会被规范化为桶内路径 |

> 注意：`base_model.py` 的 docstring 里还留着 data_scope 1–5 的历史描述，**实际实现与前端契约是 1/2/3**（`Permission.DATA_SCOPE_SELF/DEPT_AND_CHILD/ALL`）。以 1/2/3 为准。

## 7. 操作日志（OperationLogRoute）

`app/core/router_class.py`：
- 按 `settings.OPERATION_RECORD_METHOD` 配置的方法记录请求/响应，`BackgroundTask` 异步落库（失败仅告警，不影响响应）。
- 请求体与响应体都过 `_redact_sensitive`，`_SENSITIVE_KEYS` 含 password/token/api_key/secret/authorization 等 → `******`。**新增凭据类字段名要同步进集合**。
- 请求参数超 2000 字符记为「请求参数过长」；multipart 过滤 `UploadFile`。
- 非 JSON 响应体记 `{}`。

## 8. 时间与消息契约（SKILL.md §12，已验证勿回退）

- 数据库统一存 **UTC**；接口/页面按 `settings.TIMEZONE`（默认 `Asia/Shanghai`）展示。
- 输出走 `app/core/validator.py::to_local_datetime` 或 `DateTimeStr`（Pydantic 序列化时自动转本地时区字符串）；**裸 datetime** 由 `app/common/response.py` 的 `_JSON_DATETIME_CUSTOM_ENCODER` 兜底转换。
- 不要为了「显示方便」给裸 DATETIME 手动 +8 小时（SQL 控制台已有约束）。
- Excel 导出前必须把 aware datetime 转 naive（openpyxl 不支持时区）；导入模板中文性别映射 男→0 / 女→1 / 未知→2；空/损坏 Excel 返回 **400 不是 500**。
- 调度器：`reconcile_node_jobs` 启动时刷新已存在 job；改 `node.func`/`trigger` 后必须确认旧 job 参数被替换，否则「数据库已改、调度器仍报旧原因」。

## 9. 与前端对齐的检查点

- URL 必须与前端 `api/module_x/xxx.ts` 里的 `API_PATH` 逐字一致。
- **命名不对称陷阱**：后端 `modules/task/storage/` → URL `/task/storage/...` → 前端 API 目录却是 `api/module_storage/`，页面在 `views/module_task/storage/`。改存储相关接口时三处都要查。
- 分页字段 `page_no/page_size/items/total/has_next` 两端一致；前端 `useTable` 默认映射 `current/size`，由 `paginationKey` 适配。
- 权限标识、菜单 `route_name` 与前端 `defineOptions({ name })` 必须一致（KeepAlive 按组件名匹配）。
