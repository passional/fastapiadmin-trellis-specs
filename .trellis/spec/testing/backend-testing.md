# Backend Testing

## 1. Current topology and commands

Backend tests live in `backend/tests/`; `backend/pyproject.toml` configures
pytest with asyncio auto mode and function-scoped async fixture loops. Run from
`backend/`:

```bash
uv run pytest
uv run ruff check .
```

The checked-in suite is small and deliberately close to the bootstrap surface:

- `test_main.py` calls `GET /monitor/health/check/` and asserts the response
  status plus the `success`/`code` envelope fields.
- `test_migrations.py` is the migration-consistency guard (see §2).
- `conftest.py` owns application setup, SQLite, mocked Redis, shared
  `TestClient`, admin login, and `assert_route`.

There are still no dedicated backend security, WebSocket/SSE, generator,
upload, or service/CRUD test files. `test_migrations.py` is the one dedicated
suite beyond the health smoke test; the cases recommended below are coverage
targets, not claims of existing tests.

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
| `auth_headers` | Session-scoped login/token for `admin` / `admin123` |
| `versions_before_layout` | Session-scoped snapshot of `ALEMBIC_VERSION_DIR` taken at conftest import, consumed by the migration guard |
| Redis dict | Module-global; `flush*` exists but no automatic per-test flush |
| DB rows | No automatic rollback/reset between tests |

Tests can therefore depend on ordering or leak mutations. Until isolation is
improved, use unique values/IDs, restore modified shared seed data, and avoid
changing the session admin's password/status in a test that other tests need.
For new focused suites, prefer function-scoped transaction rollback plus a
Redis reset fixture, or explicitly reset both stores around each case. Never
point default tests at a developer MySQL/PostgreSQL/Redis instance.

The Redis double implements only a selected command surface (string, hash, and
expiry commands). When production code adds a Redis operation, extend the double
with realistic bytes/TTL/NX behavior and test it; an unconstrained `AsyncMock`
can otherwise make an unsupported call appear successful. Use a real disposable
Redis integration test only when semantics such as TTL, atomicity, pub/sub, or
scripting matter.

### Migration-consistency guard

`test_migrations.py` exists because startup has two schema paths: a fresh
install bootstraps with `create_all()` + `alembic stamp head`, while an existing
install runs `alembic upgrade head`. If the models and the bootstrap product
drift, the two paths produce different schemas and nothing notices — dev-only
auto-migration is an implicit guard that only runs on the developer's machine.

The guard asserts two things:

1. the test layout (`init_db` during `TestClient` startup) must not write any
   new file into `versions/` — a leaked revision is a drift signal;
2. an explicit `alembic revision --autogenerate` against the bootstrapped
   database must produce **zero** output, proving `create_all` output ≡ model
   `metadata` under the SQLite dialect. MySQL dialect DDL rendering is covered
   by the dev-startup autogenerate zero-diff check instead.

`versions_before_layout` (session fixture) supplies the pre-layout filename set
so both guards can diff against it.

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

`assert_route` (defined in `conftest.py`) is only a mount smoke helper: with no
`expected_status` it accepts any non-404, and it catches application exceptions.
Do not use it as evidence for validation, authorization, persistence, or
response correctness. It currently has **zero call sites** in the suite — it is
available but unused, so it is not equivalent to existing route coverage.

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
| Auth/session | token type, signature/session/cached-vs-current user status, replacement-token issuance, refresh-token replay revocation on mismatch, logout same/mismatched-session behavior, 401/403 distinction |
| Upload/download | path confinement (`Path.is_relative_to`), extension/content/actual size, cleanup, permission, no partial file |
| Logging | correlation/audit metadata and sentinel-secret absence from logs/DB/export (operation-log request+response redaction regression guard) |
| Migration/model | `test_migrations.py` stays green, plus upgrade result and representative CRUD on the supported target dialect |

Tests must not swallow unexpected server exceptions to obtain a green route
check. If a failure case is expected, assert its public envelope and unchanged
state.