# 后端开发规范（backend/）

> 技术栈：FastAPI + SQLAlchemy 2.x(async) + Alembic + Redis + uv，Python 3.12（`backend/.python-version`）。
> 规则来源：根目录 `SKILL.md` §3、§8、§12、§14 + 代码现状。事实以代码为准。

## 文件索引

| 文件 | 内容 | 什么时候读 |
| --- | --- | --- |
| [directory-structure.md](./directory-structure.md) | `app/` 布局、modules/plugin 模块组织、`plugin.toml`、路由注册、动态发现与重启 | 建新模块 / 找文件放哪 |
| [api-conventions.md](./api-conventions.md) | controller/service/crud/schema 四层契约、权限标识、响应信封、分页与查询参数、统一业务契约 | 写接口 / 改 schema / 对前端联调 |
| [database-guidelines.md](./database-guidelines.md) | `ModelMixin`/`CRUDBase`、软删除、事务边界、数据权限、迁移、命名与跨方言 JSON | 改 model / 写查询 / 生成迁移 |
| [error-handling.md](./error-handling.md) | `CustomException`、`RET` 业务码 → HTTP 映射、全局处理器、超管装饰器 | 抛业务错误 / 排查状态码 |
| [logging-guidelines.md](./logging-guidelines.md) | loguru 用法、`{}` 占位符、链路 cid、脱敏、日志文件与降噪 | 加日志 / 查线上问题 |
| [quality-guidelines.md](./quality-guidelines.md) | ruff/pytest 口径、禁止模式、已知坑（代码生成模板、时区、Excel、调度器） | 提交前自检 / 踩坑排查 |

## 后端三条铁律

1. **业务错误只有一种写法**：`raise CustomException(msg=..., code=...)`。HTTP 状态由 `code` 经映射表推导，构造器不接受 `status_code`。
2. **事务边界在 HTTP 层**：`db_getter` 里 `session.begin()`，CRUD 只 `flush` 不 `commit`；数据权限过滤是 **Service 层**的责任，忘了过滤就是越权。
3. **改 model 必须生成并应用 Alembic 迁移**；新增/改动模块后**必须重启后端**（动态路由发现只在启动时执行）。

## 快速定位

| 要找什么 | 去哪 |
| --- | --- |
| 全部路由注册 | `backend/app/api/v1/routers.py`（`DOMAIN_CONTROLLERS`） |
| 标准 CRUD 参考实现 | `backend/app/modules/system/dict/`（controller/service/crud/model/schema） |
| 基建 | `backend/app/core/`（base_crud / base_model / base_schema / dependencies / exceptions / permission / router_class / validator / redis_crud / ap_scheduler / sse_manager / discover） |
| 配置 | `backend/app/config/setting.py` + `backend/env/.env.dev`（模板 `.env.example`） |
| 种子数据 | `backend/sql/*.json`（菜单/角色/用户/部门/字典/参数） |
| 启动初始化 | `backend/app/scripts/initialize.py` |
| 代码生成模板 | `backend/templates/{python,ts,vue}/*.jinja2` |
| 日志文件 | `backend/logs/fastapiadmin.log` |

跨端流程与命令见 [../project/dev-workflow.md](../project/dev-workflow.md)。
