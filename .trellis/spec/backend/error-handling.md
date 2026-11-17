# 错误处理规范

> 事实来源：`app/core/exceptions.py`（业务异常 + 全局处理器 + 码表）、`app/common/enums.py::RET`、`app/common/response.py::ErrorResponse`。
> 这是本项目收敛过的契约（代码注释里的 t16 / t19 约定），**不要回退**。

## 1. 唯一业务错误写法

```python
from app.core.exceptions import CustomException
from app.common.enums import RET

raise CustomException(msg="字典类型已存在", code=RET.DATAEXIST.code)
```

硬约束：
- **构造器不接受 `status_code`**。HTTP 状态由 `code` 经 `resolve_http_status()` 查 `_CODE_TO_HTTP_STATUS` 推导，`status_code` 只是只读派生属性。从结构上杜绝「业务码 + 显式状态码」两个事实来源。
- 默认 `code=RET.EXCEPTION.code`（-1）→ HTTP **400**（客户端可修正），不是 500。
- 可带 `data`（调试上下文）；**生产环境全局处理器会把 data 置 None**，不外泄 SQL 字段/约束名。
- 「资源不存在」用 `CRUDBase.get_or_404()`（内部抛 `RET.NOT_FOUND` → 404），不要自己判空再抛 500。

## 2. 业务码 → HTTP 映射（唯一事实来源）

`_CODE_TO_HTTP_STATUS` 摘要：

| RET 成员 | code | HTTP |
| --- | --- | --- |
| `EXCEPTION`（默认业务异常） | -1 | 400 |
| `BAD_REQUEST` / `PARAMERR` | 400 / 4103 | 400 |
| `UNAUTHORIZED` / `INVALID_TOKEN` / `EXPIRED_TOKEN` / `INVALID_CREDENTIALS` / `TOKEN_EXPIRED` | 401 / 4504 / 4505 / 4506 / 10401 | 401 |
| `FORBIDDEN` / `NO_PERMISSION` | 403 / 10403 | 403 |
| `NOT_FOUND` | 404 | 404 |
| `CONFLICT` / `DATAEXIST` | 409 / 4003 | 409 |
| `UNPROCESSABLE_ENTITY` | 422 | 422 |
| `TOO_MANY_REQUESTS` / `RATE_LIMIT_EXCEEDED` | 429 / 4503 | 429 |
| `TIMEOUT` | 4502 | 504 |
| `SERVERERR` / `INTERNAL_SERVER_ERROR` | 4500 / 500 | 500 |

- 未登记的 code 落到 `_DEFAULT_BUSINESS_STATUS = 400`，这只是「新增码忘登记」的安全网，**不是正常路径**。
- **新增 `RET` 成员必须同步登记进 `_CODE_TO_HTTP_STATUS`**。
- `10401` / `10403` 是前后端约定的认证类业务码，对应前端 `ResultEnum`（`frontend/web/src/enums/api/result.enum.ts`），前端据此跳登录/提示无权限。

## 3. 意外异常：不要包装，让它冒泡

```python
# ✅ 需要运维上下文时：先记日志再原样 raise（保留原始异常类型）
try:
    ...
except Exception:
    logger.exception("导入字典失败")
    raise

# ✅ 宽泛 except 若确实要转业务错误，必须先放行 CustomException
except CustomException:
    raise
except Exception as e:
    raise CustomException(msg="...", code=RET.SERVERERR.code)

# ❌ 把未知异常一律包成 CustomException → 全局处理器失去分类能力，5xx 变 400
```

全局处理器（`handle_exception(app)`，在 `create_app` 里注册）已经统一兜底：

| 异常 | 处理 |
| --- | --- |
| `CustomException` | 按码表返回；**4xx 记 warning，5xx 记 error**（避免校验失败淹没真故障）；生产隐藏 data |
| `RequestValidationError` | 取第一条 msg（剥掉 `Value error` 前缀）→ **422**，data 带 errors |
| `ResponseValidationError` | → 500「服务器响应格式错误」 |
| `IntegrityError` | 按驱动文案分类：Duplicate entry → 409「数据重复，请检查唯一字段」；foreign key constraint → 409「存在关联数据，无法删除」；cannot be null → 409「必填字段缺失」；连接类 → 503 |
| 其它 `SQLAlchemyError` | 连接类（`InterfaceError`/`DisconnectionError`/文案含 connect、gone away、timed out 等）→ **503**；其余 → **500**。**不再返回 400**（把服务端故障说成客户端错误会让监控失明） |
| `ValueError` | → 400，msg 原样 |
| `Exception` | → 500「服务器内部错误」，细节只进日志/堆栈 |

客户端只拿通用文案，驱动原文/堆栈只进日志（`logger.opt(exception=...)`）。

## 4. 权限类错误

- 接口级：`AuthPermission` 校验失败抛 `RET.NO_PERMISSION`（10403 → 403）。
- Service 级仅超管：`@require_superadmin`（`exceptions.py`），非超管抛「仅平台管理员可操作」。
- 认证失效：`dependencies.py::_authenticate` 抛 `RET.UNAUTHORIZED` / `INVALID_CREDENTIALS`；WebSocket 握手用 `websocket_authenticate`，失败同样抛 `CustomException`。

## 5. 前端怎么消费

- `frontend/web/src/utils/http/index.ts` 拦截器按 HTTP 状态 + 业务 code 分支，`HttpError` 带 code/data/url/method。
- `401` / `10401` → 刷新 token 或跳登录；`403` / `10403` → 无权限提示。
- 错误提示统一走 `utils/message.ts`（80ms 内同级别合并），**页面不要再叠加同义提示**。

## 6. 反模式

| 反模式 | 为什么不行 |
| --- | --- |
| `CustomException(msg=..., status_code=500)` | 构造器不接受该参数；HTTP 由 code 推导 |
| 新增 RET 码不登记映射 | 落到 400 安全网，语义错 |
| `except Exception: raise CustomException(...)` 吞掉原始异常 | 数据库连接故障被说成客户端错误，监控失明 |
| 宽泛 except 不放行 `CustomException` | 业务码被二次包装，404/409 变 400 |
| 在 controller 里 `return ErrorResponse(...)` | 错误出口应统一由异常处理器产生 |
| 把驱动原文/堆栈返回给客户端 | 生产信息泄露；细节只进日志 |
| 空/损坏 Excel 返回 500 | 契约要求 400（客户端可修正） |
