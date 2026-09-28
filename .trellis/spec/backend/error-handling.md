# Backend Error Handling

## Public contract

All JSON endpoints use the envelope defined in
`backend/app/common/response.py`:

```text
{ code, msg, data, status_code, success }
```

Controllers return `SuccessResponse`; failures are converted to
`ErrorResponse` by the handlers registered from
`backend/app/core/exceptions.py`. Do not return ad-hoc dictionaries or expose
FastAPI's default error body for a new API.

## Domain failures

Raise `CustomException` from services, CRUD helpers, dependencies, and domain
utilities when the caller can receive a meaningful business failure. Set
`msg`, business `code`, HTTP `status_code`, and optional diagnostic `data` only
as needed. Examples:

- `backend/app/modules/system/dept/service.py` rejects duplicate names,
  duplicate codes, empty deletion sets, and deletion of parents with children.
- `backend/app/core/dependencies.py` uses explicit 401 and 403 exceptions for
  authentication and authorization.
- `backend/app/core/base_crud.py:get_or_404` supplies the shared missing-object
  behavior.

Let exceptions cross the controller boundary. Controllers should not wrap each
service call in a generic `try/except`; the registered handlers preserve the
response contract and logging. When catching is necessary for cleanup or
context, preserve the chain with `raise ... from e`. Re-raise an existing
`CustomException` unchanged before a generic catch, as `CRUDBase.update` does,
so its business message is not replaced.

## Global mappings

`handle_exception()` currently maps:

- `CustomException` to its declared business and HTTP status.
- Starlette `HTTPException` to the common error envelope.
- request validation to HTTP 422 using the first Pydantic message and full
  validation errors in `data`.
- response validation to HTTP 500.
- SQLAlchemy `IntegrityError` to conflict-oriented messages (with a
  connection-text special case); other `SQLAlchemyError` instances currently
  return HTTP 400 with the exception class in `msg` and details only outside
  production. Do not assume every database connectivity failure maps to 503.
- `ValueError` to HTTP 400.
- all other exceptions to a generic HTTP 500 message.

Production deliberately removes `CustomException.data` and SQL exception
details. Do not bypass that behavior by embedding SQL, constraint details,
paths, stack traces, credentials, or tokens in `msg`.

## Transactions and background work

The `db_getter` request transaction rolls back automatically when an exception
escapes. Do not catch a write failure and then return success. Background tasks
that own their own session must also own their error boundary; for example,
`backend/app/core/router_class.py:_write_operation_log_async` logs audit-write
failure without breaking the completed response.

Middleware may translate only failures it can recover from. The request guard
in `backend/app/core/middlewares.py` returns `ErrorResponse` for blocked demo/IP
requests. Its fallback does not establish a general pattern of swallowing
unexpected exceptions in services.

## Validation boundary

Put structural constraints in Pydantic schemas (`Field`, `field_validator`,
`model_validator`) and rules requiring database/domain state in services.
`backend/app/modules/system/dept/schema.py` validates shape and codes;
`DeptService` checks uniqueness and tree relationships. Do not duplicate the
same validation in controllers.

## Avoid

- `except Exception: pass` around business or persistence work.
- returning status 200 with an improvised error object.
- exposing exception strings to clients from unknown exceptions.
- manually committing after a partially handled request error.
- logging an error and raising a second unrelated error that loses the cause.

Verify error changes with response-body assertions in pytest, including the
HTTP status and common envelope fields—not only that the route exists.
