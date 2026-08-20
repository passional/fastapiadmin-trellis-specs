# Backend Testing

## 1. Current topology and commands

Backend tests live in `backend/tests/`; `backend/pyproject.toml` configures
pytest with asyncio auto mode and function-scoped async fixture loops. Run from
`backend/`:

```bash
uv run pytest
uv run ruff check .
```

The checked-in suite is small:

- `test_main.py` asserts status plus envelope fields for readiness/check.
- `test_api_module_system.py` provides broad system-route/data smoke coverage.
- `conftest.py` owns application setup, SQLite, mocked Redis, shared
  `TestClient`, admin login, and `assert_route`.

There are no dedicated backend security, WebSocket, generator, upload, or
service/CRUD test files at bootstrap time. Recommended cases below are
coverage targets, not claims of existing tests.

## 2. Fixture behavior and isolation

`conftest.py` sets database environment variables before importing the app,
creates one temporary SQLite filename, initializes schema/data once in a
reduced lifespan, replaces Redis with an `AsyncMock` backed by a module-level
dict, disables captcha, and resets the admin hash to `admin123`.

The effective lifetimes are:

| Resource | Lifetime / behavior |
|---|---|
| SQLite file and app | One per pytest process/session |
| `_api_client` | Session-scoped `TestClient` and lifespan |
| `test_client` | Function fixture returning the same client |
| `auth_headers` | Session-scoped login/token |
| Redis dict | Module-global; `flush*` exists but no automatic per-test flush |
| DB rows | No automatic rollback/reset between tests |

Tests can therefore depend on ordering or leak mutations. Until isolation is
improved, use unique values/IDs, restore modified shared seed data, and avoid
changing the session admin's password/status in a test that other tests need.
For new focused suites, prefer function-scoped transaction rollback plus a
Redis reset fixture, or explicitly reset both stores around each case. Never
point default tests at a developer MySQL/PostgreSQL/Redis instance.

The Redis double implements only a selected command surface. When production
code adds a Redis operation, extend the double with realistic bytes/TTL/NX
behavior and test it; an unconstrained `AsyncMock` can otherwise make an
unsupported call appear successful. Use a real disposable Redis integration
test only when semantics such as TTL, atomicity, pub/sub, or scripting matter.

## 3. Test levels and patterns

### HTTP behavior

Use `TestClient` for route, middleware, dependency, response-envelope, and
lifespan behavior. Assert the exact status and business contract:

```python
response = test_client.get("/system/user/current/info", headers=auth_headers)
assert response.status_code == 200
body = response.json()
assert body["success"] is True
assert body["code"] == 0
assert body["status_code"] == 200
assert body["data"]["username"] == "admin"
```

`assert_route` is only a mount smoke helper: with no `expected_status` it
accepts any non-404, and it catches application exceptions. Do not use it as
evidence for validation, authorization, persistence, or response correctness.
Several existing route tests intentionally provide wrong refresh/logout body
shapes and merely prove the route exists.

Every changed behavior needs good/base/bad cases where applicable:

- good: valid input, exact output, and persisted state;
- base: optional/default/empty-list boundary;
- bad: invalid schema (422), unauthenticated (401), unauthorized (403),
  missing/conflict/business error, and no unintended state change.

### Async service/CRUD behavior

Pytest asyncio auto mode permits `async def` tests and async fixtures without
per-test decorators. Inject an `AsyncSession`, controlled `AuthSchema`, and
Redis double directly when testing transaction, query, data-scope, or retry
logic. The request session uses `session.begin()` in `db_getter`; ordinary CRUD
flushes rather than committing. Assert rollback/commit at the owning boundary,
soft-delete filters, audit IDs, and data-scope filters.

SQLite does not prove MySQL/PostgreSQL-specific types, SQL, locking, JSON/date
semantics, or Alembic operations. A dialect-sensitive change needs a disposable
target-database integration job in addition to the default fast suite.

## 4. Required ownership by change

| Change | Minimum assertions |
|---|---|
| Controller/schema | method/path, all five envelope fields, valid/invalid body/query/path, auth and exact permission |
| Service/CRUD | success and domain conflict, DB state, rollback, soft deletion, data scope, batch IDs |
| Auth/session | token type, signature/session/cached-vs-current user status, replacement-token issuance and old-token acceptance, logout same/mismatched-session behavior, 401/403 distinction |
| Upload/download | path confinement, extension/content/actual size, cleanup, permission, no partial file |
| Logging | correlation/audit metadata and sentinel-secret absence from logs/DB/export |
| Migration/model | upgrade result and representative CRUD on the supported target dialect |

Tests must not swallow unexpected server exceptions to obtain a green route
check. If a failure case is expected, assert its public envelope and unchanged
state.
