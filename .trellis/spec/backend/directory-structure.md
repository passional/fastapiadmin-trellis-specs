# 后端目录结构与模块组织

> 事实来源：`backend/app/` 实际布局、`app/api/v1/routers.py`、`app/core/discover.py`。

## 1. 目录布局

```
backend/
├── main.py                  # typer CLI：run / revision / upgrade
├── alembic.ini              # 迁移配置（versions 在 app/alembic/versions）
├── pyproject.toml           # uv 依赖 + ruff + pytest + coverage 配置
├── env/                     # .env.example / .env.dev / .env.prod（不提交真实凭据）
├── sql/*.json               # 种子数据：菜单/角色/用户/部门/字典/参数
├── templates/{python,ts,vue}/  # 代码生成器 jinja2 模板
├── tests/                   # pytest（conftest 自建临时库 + mock Redis）
├── logs/fastapiadmin.log    # loguru 输出
├── static/{image,swagger,upload}/
└── app/
    ├── __init__.py          # create_app：异常→中间件→路由→静态→docs→前端挂载
    ├── api/v1/routers.py    # ★ 全部路由的唯一注册处
    ├── common/              # constant / enums(RET 等) / response / request / dataclasses
    ├── config/              # setting.py（Settings）/ path_conf.py / db_url.py
    ├── core/                # 基建（见下表）
    ├── modules/<域>/<功能>/  # ★ 常规业务模块
    ├── plugin/module_xxx/   # ★ 可插拔插件（动态发现）
    ├── scripts/initialize.py# 启动时迁移 + 种子数据
    ├── alembic/             # env.py + versions/
    └── utils/               # crypto/password/excel/upload/xss/common_util 等
```

`app/core/` 基建职责：

| 文件 | 职责 |
| --- | --- |
| `database.py` | 异步引擎/会话工厂；同步引擎仅供 APScheduler JobStore 与代码生成器 Inspector |
| `base_model.py` | `MappedBase` / `ModelMixin`（通用字段 + 索引 + 命名约定 + `JSONType`） |
| `base_crud.py` | `CRUDBase[Model, Create, Update]`：get/page/create/update/delete/软删除 |
| `base_schema.py` | `BaseSchema` / `BaseQueryParam` / `AuthSchema` / `PageResultSchema` / `PaginationQueryParam` |
| `dependencies.py` | `db_getter`（请求级事务）、`redis_getter`、`AuthPermission`、WebSocket 握手认证 |
| `permission.py` | `Permission` 数据权限过滤（data_scope 1/2/3） |
| `exceptions.py` | `CustomException`、`RET.code → HTTP` 映射、全局异常注册、`require_superadmin` |
| `router_class.py` | `OperationLogRoute`：自动操作日志 + 敏感字段脱敏 |
| `validator.py` | `DateTimeStr`/`DateStr`/`TimeStr`/`Telephone`/`Email` 等自定义类型、`to_local_datetime` |
| `redis_crud.py` | `RedisCURD` 封装 |
| `ap_scheduler.py` | 定时任务调度器 |
| `sse_manager.py` | SSE 连接管理 |
| `discover.py` | `app/plugin/module_*` 动态路由发现（启动时执行并缓存） |
| `logger.py` | loguru 配置 + 标准库 logging 重定向 + 链路 cid |
| `security.py` | JWT 签发/校验、OAuth2 schema |
| `middlewares.py` | CORS、请求日志、演示模式/IP 黑名单拦截 |

## 2. 业务模块组织（固定五件套）

`app/modules/<域>/<功能>/`，每个功能目录固定文件划分：

| 文件 | 职责 |
| --- | --- |
| `controller.py` | 路由层：声明 `XxxRouter = APIRouter(route_class=OperationLogRoute, prefix=..., tags=[...])`，只做参数绑定 + 调 Service + 包响应 |
| `service.py` | 业务层：`XxxService(auth, db)` 实例方法承载用户上下文；**数据权限过滤在这里** |
| `crud.py` | 数据层：`XxxCRUD(CRUDBase[Model, CreateSchema, UpdateSchema])` |
| `model.py` | ORM：继承 `ModelMixin`，显式 `__tablename__` |
| `schema.py` | Pydantic：`XxxCreateSchema` / `XxxUpdateSchema` / `XxxOutSchema` / `XxxQueryParam` |

参考实现：`app/modules/system/dict/`（一个域里两个功能共用一套文件）、`app/modules/system/position/`（单功能最小样板）。

现有域：`system`（auth/user/role/menu/dept/position/dict/params/notice/ticket/versions/log）、`monitor`（cache/health/online/server）、`task`（cronjob/{job,node}、storage/{browse,node,transfer,workflow}）、`ai`（chat/tools）、`generator`（gencode）、`common`（file）、`ops`（ssh/dbconsole）。

### plugin.toml 放在「域根」，不是每个功能目录

实际清单（`find backend/app -name plugin.toml`）：

```
app/modules/{ai,common,generator,monitor,system,task}/plugin.toml
app/modules/task/storage/plugin.toml      # 嵌套子域也可有自己的清单
app/plugin/module_example/plugin.toml
```

字段：`name` / `title` / `version` / `description` / `optional` / `tags`。示例（`app/modules/system/plugin.toml`）：

```toml
name = "system"
title = "系统"
version = "1.0.0"
description = "系统核心功能（用户/角色/权限/部门/字典/参数/通知/日志等）"
optional = true
tags = ["system", "auth", "rbac"]
```

> SKILL.md 说「模块根放 plugin.toml」——这里的「模块」指**域**（如 `modules/system`），不是 `modules/system/dict`。新功能目录不需要单独建清单。

## 3. 两种新模块放法

| 放法 | 位置 | 路由注册方式 |
| --- | --- | --- |
| 常规业务 | `app/modules/<域>/<功能>/` | **手工**加进 `app/api/v1/routers.py` 的 `DOMAIN_CONTROLLERS` |
| 可插拔 / 示例 | `app/plugin/module_<name>/<功能>/` | **自动发现**（`core/discover.py`），无需改 routers.py |

`app/plugin/` 发现规则（`discover.py` 文档字符串）：
- 顶级目录名必须以 `module_` 开头，映射为路由前缀 `/<去掉 module_ 的部分>`（`module_example` → `/example`）；
- 控制器文件必须叫 `controller.py`，模块顶层定义 `APIRouter` 实例；
- 每级目录名须为合法 Python 标识符，且有 `__init__.py`；
- **任何插件导入失败都会中止启动**（故意设计：半成品 router 会让故障推迟成「某模块突然 404」，比启动即失败难定位）。

## 4. 路由注册（唯一事实来源）

`app/api/v1/routers.py`：

```python
DOMAIN_CONTROLLERS: dict[str, list[APIRouter]] = {
    "/system": [AuthRouter, UserRouter, RoleRouter, MenuRouter, ...],
    "/monitor": [CacheRouter, HealthRouter, OnlineRouter, ServerRouter],
    "/task": [CronJobRouter, CronJobNodeRouter, StorageBrowseRouter, ...],
    "/ai": [ChatRouter],
    "/generator": [GenRouter],
    "/common": [FileRouter],
    "/ops": [SshRouter, DbConsoleRouter],
}
```

最终 URL = `/api/v1` + 域前缀 + controller 自身 prefix + 端点路径。
例：`DictRouter(prefix="/dict")` 的 `/type/list` → `/api/v1/system/dict/type/list`。

## 5. 命名约定

| 对象 | 约定 | 例 |
| --- | --- | --- |
| 域 / 功能目录 | 小写单词，功能目录用单数名词 | `system/dict`、`task/cronjob/node` |
| Router 变量 | `<功能>Router` | `DictRouter`、`StorageNodeRouter` |
| Service / CRUD | `<实体>Service` / `<实体>CRUD` | `DictTypeService`、`DictTypeCRUD` |
| Model | `<实体>Model`，表名 `<域前缀>_<实体>` | `DictTypeModel` → `sys_dict_type` |
| Schema | `<实体>CreateSchema` / `UpdateSchema` / `OutSchema` / `QueryParam` | `DictTypeCreateSchema` |
| 权限标识 | `module_<域>:<资源>:<操作>`，嵌套子域多一段 | `module_system:dict_type:create`、`module_task:storage:node:query` |

## 6. 必须重启后端的场景

`core/discover.py` 的 `DynamicRouterRegistry._build()` 带缓存，只在启动时扫描一次；dev 的 `reload` 只监听文件变更。因此：

- 代码生成器产出新模块后；
- 新增 / 移动 `app/plugin/module_*` 目录后；
- 改 `plugin.toml` 或模块结构后。

**不重启 = 新模块不生效且静默失败**（没有报错，只是接口 404）。

## 7. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| 在 controller 里写业务逻辑 / 直接查库 | controller 只绑参数 + 调 Service；查询走 CRUD |
| 新功能建在 `modules/` 却指望自动发现 | `modules/` 必须手工注册进 `DOMAIN_CONTROLLERS` |
| 给每个功能目录都建 `plugin.toml` | 清单放域根 |
| 插件目录名不带 `module_` 前缀 | 会被 discover 跳过并打 error 日志 |
| 在事件循环里用同步引擎 | 同步引擎只给 APScheduler / 生成器 Inspector（经 `asyncio.to_thread`） |
| `core/` 导入期依赖业务模块 | 用延迟导入（见 `permission.py`、`router_class.py` 里的注释「守卫不变式 3」） |
