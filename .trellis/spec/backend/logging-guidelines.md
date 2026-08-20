# Backend Logging Guidelines

## Logging stack

Use the shared Loguru `logger` imported from `backend/app/core/logger.py`.
`setup_logger()` removes default sinks, patches each record with a correlation
ID, writes to stdout and `logs/fastapiadmin.log`, rotates daily, retains 30
archives, and intercepts standard-library/Uvicorn logs. Do not configure a new
logger or sink inside a feature module.

The format already supplies timestamp, level, module, function, line, message,
and the first eight characters of `cid`. `CorrelationIdMiddleware` in
`backend/app/core/middlewares.py` accepts or creates `X-Correlation-ID`, adds it
to the response, and resets the context after the request. Avoid manually
adding timestamps, module names, or correlation IDs to each message.

## Levels and exception context

- `debug`: high-volume diagnostic state useful during development; respect the
  configured logger level.
- `info`: application lifecycle and successful subsystem initialization, as in
  `backend/app/init_app.py`.
- `warning`: recoverable degradation, blocked requests, or malformed optional
  state. `RequestLogMiddleware` uses it for IP/demo-mode rejection.
- `error`: a failed operation or handled infrastructure/domain failure that
  needs attention.
- `logger.exception`: use inside an active exception handler when the traceback
  materially helps diagnosis. `_write_operation_log_async` is the reference.

Loguru brace interpolation markers are the predominant local pattern:

```python
logger.error("数据库连接失败: {}", error)
```

Some legacy f-strings and standard logging interpolation remains; do not spread
mixed formatting in new code.

## What to record

Record lifecycle transitions, unexpected infrastructure failures, failed
background work, security-relevant denial, and identifiers needed to locate an
operation. Include stable context such as HTTP method/path, model or job ID,
protocol, and failure category. Avoid logging every successful CRUD call; the
route audit layer handles write-operation history.

`OperationLogRoute` in `backend/app/core/router_class.py` records configured
write methods, route summary, actor, path, method, IP, elapsed time, response
code, request payload, and JSON response body in a background task. The request
payload is replaced once its serialized form exceeds 2,000 characters; the
response body currently has no equivalent size cap. New normal API routers
should keep `route_class=OperationLogRoute` so they participate in this existing
audit flow, and changes to large-response endpoints must account for that
unbounded response capture.

## Sensitive data

Never intentionally add passwords, access/refresh tokens, authorization
headers, storage credentials, private chat content, raw uploaded files, or
secrets to ordinary logs. This is not yet fully enforced: the operation route
captures non-file form/JSON requests and JSON responses without field
redaction. Existing login/refresh/logout, AI model configuration, and system/AI
chat write routes can therefore persist passwords, returned JWTs, refresh
tokens, API keys, or message content in `sys_operation_log`. Fix or explicitly
exclude/redact these paths before treating operation logs as safe; length-only
replacement is not redaction.

Production handling suppresses `CustomException.data` and SQL diagnostic
details; keep user-facing messages free of internal SQL and stack details.
Prefer record IDs over entire Pydantic models or ORM objects.

## Avoid

- `print()` in runtime feature code (Alembic's CLI progress output is a
  migration-only exception).
- adding per-module files or calling `logging.basicConfig` outside
  `core/logger.py`.
- logging and silently swallowing a failure that should roll back a request.
- adding new unbounded payload logs (the operation route's response capture is
  a current exception to review), or high-frequency scheduler polling at
  info/debug level; the logger intentionally raises APScheduler's threshold to
  WARNING.

When changing logging or middleware, test that `X-Correlation-ID` is preserved
and that client responses do not expose internal exception details.
