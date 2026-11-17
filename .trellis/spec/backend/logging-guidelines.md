# 日志规范

> 事实来源：`app/core/logger.py`、`app/core/router_class.py`、`app/core/exceptions.py`、`app/config/path_conf.py`。
> 日志库：**loguru**（标准库 logging 经 `InterceptHandler` 全量重定向）。

## 1. 基本用法

```python
from app.core.logger import logger

logger.info("字典缓存已刷新: type={}", dict_type)     # ✅ {} 占位符
logger.warning("子部门数据权限计算失败，降级为本部门范围: {}", e)
logger.exception("导入字典失败")                      # ✅ 自动带堆栈
logger.opt(exception=exc).error("[数据库异常] {} {}", method, path)
```

- **占位符用 `{}`（loguru 风格），不要用 `%s`，也不要用 f-string 拼消息**。f-string 会让消息在日志级别被过滤时仍然求值，且破坏结构化聚合。
  - 现存代码里有少量 f-string 历史写法（如 `database.py` 的连接失败日志），新代码统一 `{}`。
- 异常日志用 `logger.exception(...)` 或 `logger.opt(exception=exc)`，不要手动 `traceback.format_exc()` 拼字符串。
- 不要 `print()`；不要新建 `logging.getLogger(__name__)` 绕过统一配置（会被 InterceptHandler 接管，但丢失 loguru 的 sink/格式）。

## 2. 级别口径

| 级别 | 用在哪 |
| --- | --- |
| DEBUG | 开发期细节；`settings.LOGGER_LEVEL` 默认 DEBUG，生产按需调高 |
| INFO | 正常生命周期节点：启动初始化完成、迁移已应用、种子导入/跳过、缓存刷新 |
| WARNING | **客户端可修正的业务失败（4xx）**、降级行为（如数据权限降级为本部门）、请求被演示模式/黑名单拦截、僵尸迁移版本清理 |
| ERROR | **服务端故障（5xx）**、未捕获异常、数据库异常、插件导入失败、破坏性迁移被拦截 |

关键约定（`exceptions.py::custom_exception_handler`）：`CustomException` 按推导出的 HTTP 状态分级——**4xx → warning，5xx → error**。不要把每个参数校验失败都打 ERROR，那会淹没真故障信号。

## 3. 输出与格式

`setup_logger()`（模块导入即执行）：

- 两个 sink：`sys.stdout` + 文件 `backend/logs/fastapiadmin.log`（`path_conf.LOG_DIR`）。
- 格式：`时间 | 级别 | 模块:函数:行号 - 消息 | cid=xxxxxxxx`。
- 文件 rotation `00:00`（每天零点）、retention 30 天、compression `gz`、encoding utf-8。
- 生产（`ENVIRONMENT=prod`）关闭 `backtrace` / `diagnose`，避免泄露变量值。
- **链路 ID**：`ContextVar` 存 `correlation_id`，中间件设置，`_context_patcher` 自动往每条日志追加 ` | cid=<前8位>`。跨请求排查按 cid grep。

排查线上问题的第一入口就是 `backend/logs/fastapiadmin.log`。典型case：前端静态托管 500（`check_dir=False` 启动不报错、首次请求才炸）只能靠这个日志定位。

## 4. 第三方降噪

`setup_logger()` 把以下 logger 压到 WARNING（dev reload 下会刷屏）：
`apscheduler*`、`alembic.runtime.migration`、`alembic.autogenerate`、`alembic.runtime.plugins`、`asyncio`、`websockets`、`sqlalchemy`、`aiomysql`、`tzlocal`。

新增噪音大的第三方库时按同样方式加进这个列表，**不要在业务代码里到处 setLevel**。

## 5. 操作日志（落库，不是 loguru）

`OperationLogRoute`（`app/core/router_class.py`）自动记录写操作到 `sys_operation_log`：

- 触发方法由 `settings.OPERATION_RECORD_METHOD` 配置。
- `BackgroundTask` 异步写库，失败仅 `logger.exception("操作日志写入失败: path={}", ...)`，不影响响应。
- 请求体 / 响应体都过 `_redact_sensitive`；`_SENSITIVE_KEYS` 含 password / passwd / old_password / new_password / confirm_password / captcha_key / token / access_token / refresh_token / api_key / apikey / secret / client_secret / secret_key / authorization → `******`。
- 请求参数 JSON 超 2000 字符 → 记「请求参数过长」；multipart 过滤 `UploadFile`；非 JSON 响应体记 `{}`。

**新增凭据类字段名时必须同步进 `_SENSITIVE_KEYS`**，否则明文入库等于泄露（登录/刷新响应含 access_token，历史上已因此脱敏响应体）。

## 6. 什么不能进日志

- 口令、私钥、token、第三方 Key、`SECRET_KEY` / `DATA_ENCRYPTION_KEY`（红线，见 [../project/collaboration-redlines.md](../project/collaboration-redlines.md)）。
- 生产环境的异常 `data`（全局处理器已置 None，不要自己绕过）。
- 完整请求体里未脱敏的敏感字段。
- 用户隐私大字段（手机号/邮箱）非必要不整条打印。

## 7. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| `logger.info(f"用户{user_id}登录")` | `logger.info("用户登录: id={}", user_id)` |
| `logger.error("参数错误")` | 4xx 用 warning |
| `print(...)` 调试残留 | 删掉或改 logger.debug |
| 手动拼 traceback | `logger.exception` / `logger.opt(exception=...)` |
| 新敏感字段不脱敏就落操作日志 | 同步 `_SENSITIVE_KEYS` |
| 在业务代码里给第三方库 setLevel | 统一在 `setup_logger()` 降噪列表 |
