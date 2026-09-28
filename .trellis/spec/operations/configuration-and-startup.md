# Configuration and Startup

## 1. Scope / Trigger

Use this contract when adding or changing an environment key, secret, database
or Redis dependency, startup task, seed, scheduler job, migration, or runtime
module gate.

## 2. Signatures

Settings are a case-sensitive Pydantic `BaseSettings` model
(`backend/app/config/setting.py`):

```python
SettingsConfigDict(
    env_file=ENV_DIR / f".env.{os.getenv('ENVIRONMENT', 'dev')}",
    extra="ignore",
    case_sensitive=True,
)
```

The supported file convention is `backend/env/.env.dev` or `.env.prod`, copied
from `backend/env/.env.example`. Process environment variables override file
values. `docker/docker-compose.yaml` sets selected backend variables before the
Python process starts.

`backend/env/.env.example` instructs users to set `ENVIRONMENT`, `SECRET_KEY`,
`DATABASE_PASSWORD`, `REDIS_PASSWORD`, and `OPENAI_API_KEY`, but the file has no
`SECRET_KEY` entry — the header comment asks for a key the template never
declares. Add it when refreshing the template.

The backend CLI is run from `backend/`:

```text
uv run main.py run --env=dev
uv run main.py revision --env=dev
uv run main.py upgrade --env=dev
```

## 3. Contracts

### First run

1. Copy `backend/env/.env.example` to `.env.dev` and set the mandatory values
   (`DATABASE_PASSWORD`, `REDIS_PASSWORD`, `OPENAI_API_KEY`; also `SECRET_KEY`).
   `DATABASE_TYPE` supports `mysql` / `postgres` / `sqlite`; `sqlite` is the
   zero-config local option (no host/credentials needed).
2. `uv sync && uv run main.py run --env=dev` — startup applies migrations and
   imports the flat `backend/sql/*.json` seed data (menu/role/user/dept/dict/
   param); in `dev` it also auto-generates and applies migrations when models
   changed. No manual `create_all` / seed step is required.
3. Frontend web: `cd frontend/web && pnpm install && pnpm dev` (App H5 debug uses
   `pnpm dev:h5`).
4. Default seed accounts: `super` / `admin` / `user`, password `123456`
   (hashed with **PBKDF2-HMAC-SHA256**, 600k iterations — not bcrypt). Change it
   immediately after deployment.
5. URLs: Web `http://localhost:{VITE_PORT}` (`frontend/web/.env` ships 5180),
   backend `http://localhost:8001`, Swagger `http://localhost:8001/docs`, API
   prefix `/api/v1`.
6. Docker one-shot deploy: `./deploy.sh` at the repo root (see
   `docker/README.md`).

### Environment ownership

| Concern | Backend/runtime keys | Other owners |
|---|---|---|
| Runtime/profile | `ENVIRONMENT`, `DEBUG`, `SERVER_HOST`, `SERVER_PORT`, `WORKERS`, `TRUSTED_PROXY_HOPS`, `LOGGER_LEVEL` | `backend/app/config/setting.py`, `backend/env/.env.example` |
| Database | `DATABASE_TYPE/HOST/PORT/USER/PASSWORD/NAME`, pool keys | Compose maps MySQL values into backend keys |
| Redis | `REDIS_HOST/PORT/DB_NAME/USER/PASSWORD`, `REDIS_HEALTH_CHECK_INTERVAL`, `REDIS_DEFAULT_CACHE_TTL` | Compose Redis service and backend environment |
| Auth/security | `SECRET_KEY`, token expiry, `SESSION_MAX_LIFETIME_SECONDS`, `LOGIN_RATE_LIMIT_*`, `DATA_ENCRYPTION_KEY`/`DATA_ENCRYPTION_OLD_KEYS`, `PASSWORD_MIN/MAX_LENGTH`, `PASSWORD_IMPORT_DEFAULT`, `CAPTCHA_MIN_VERIFY_SECONDS`, `ALLOWED_HOSTS`, `PROD_CORS_ORIGINS`/`CORS_EXPOSE_HEADERS`, OAuth/WX secrets | Nginx host/TLS configuration is separate |
| Scheduler | `SCHEDULER_ALLOW_CODE_EXEC` | `backend/app/core/ap_scheduler.py` |
| HTTP client | `HTTPX_DEFAULT_TIMEOUT`, `GZIP_MIN_SIZE`, `GZIP_COMPRESS_LEVEL` | `backend/app/utils/` / middleware |
| AI | `OPENAI_BASE_URL`, `OPENAI_API_KEY`, `OPENAI_MODEL` | Per-user model configs are also stored in Redis |
| Clients | `VITE_API_BASE_URL`, `VITE_APP_BASE_API`, `VITE_APP_WS_ENDPOINT`, `VITE_PORT`, app base paths | Each frontend package owns its env/type declarations |

`AUTOFETCH` is an alias for `AUTOFLUSH` (higher precedence, kept for legacy env
names). Add a key to `Settings` and every relevant example/deployment mapping;
add a frontend key to that package's `ImportMetaEnv` declaration. Never assume a
similarly named Compose variable is consumed automatically.

### CLI environment selection

Each `backend/main.py` command sets `os.environ["ENVIRONMENT"] = env.value`
**before** importing `settings` (run/revision/upgrade), and `get_settings()`
defaults `ENVIRONMENT` to `dev`. Consequently `--env=dev|prod` reliably loads
`backend/env/.env.dev` / `.env.prod`; the settings model never constructs a
`None`-suffixed env path, and there is no import-ordering defect. Docker avoids the issue entirely because Compose sets
`ENVIRONMENT` before Python starts. New entry points must set the environment
before importing any settings consumer; keep subprocess tests for `run`,
`revision`, and `upgrade`.

### Secret and logging safety

- Production must override the development `SECRET_KEY`. It signs JWTs in
  `backend/app/core/security.py` and, via HKDF, derives the primary Fernet key
  used by `backend/app/utils/crypto_util.py` for storage-source passwords and
  AI `api_key` values. Changing it invalidates tokens and can make stored
  credentials undecryptable; `DATA_ENCRYPTION_KEY` (or `DATA_ENCRYPTION_OLD_KEYS`
  during rotation) is the intended, decoupled master key.
- User AI model configurations store `api_key` encrypted in Redis and return it
  only to the authenticated owner. Evidence:
  `backend/app/modules/ai/chat/{schema.py,service.py}`.
- `OperationLogRoute` **redacts sensitive fields** before persisting operation
  logs. `_SENSITIVE_KEYS` / `_redact_sensitive` (`backend/app/core/router_class.py`)
  cover the JSON request body, form fields, and the JSON response body —
  including login/refresh responses that carry `access_token`/`refresh_token`.
  Matches are replaced with `******`. The 2,000-character `请求参数过长`
  replacement still applies only to the serialized request.
- Do not log connection URIs, authorization headers, query-string JWTs,
  decrypted storage credentials, or full third-party responses containing
  secrets.

### Startup and migrations

`backend/app/__init__.py:lifespan` performs, in order:

1. `InitializeData().init_db()` (`backend/app/scripts/initialize.py`) applies
   migrations and imports seed data.
2. `init_agno_tables()` creates the AI session tables
   (`backend/app/modules/ai/chat/crud.py`).
3. Redis connects and is stored on `app.state`.
4. parameter and dictionary caches initialize.
5. APScheduler starts (`SchedulerUtil.init_scheduler`), registering the
   operation-log cleanup job.
6. Shutdown stops the scheduler, Redis, and DB engine.

`init_db()` runs on **every** startup:

- Empty DB (sentinel `sys_menu` missing) → `create_tables()`; if the repo has
  migration history, `alembic stamp head`; then seed from `backend/sql/*.json`.
- Existing DB → clear unresolvable zombie `alembic_version` rows, then
  `command.upgrade(cfg, "head")`.
- `ENVIRONMENT == DEV` → additionally autogenerate a revision, and if models
  changed, apply it. A generated revision whose `upgrade()` section contains
  destructive `op.drop_*` / `op.execute(... DROP ...)` is deleted and
  auto-application is aborted with instructions to run
  `main.py revision --env=dev` manually, review, then `upgrade`.

The Alembic versions directory intentionally holds only `__init__.py`; schema is
brought up to date from models plus startup migrations, so store-bought
deployments do not need an external migration step. `docker/backend/Dockerfile`
CMD is `python main.py run --env=prod`, so containers auto-migrate on boot.

## 4. Validation & Error Matrix

| Condition | Expected/Current behavior |
|---|---|
| Wrong-case env key | Ignored because settings are case-sensitive |
| Unknown env key | Ignored (`extra="ignore"`) |
| CLI `--env` without pre-set `ENVIRONMENT` | Loads the requested `.env.{dev,prod}`; `ENVIRONMENT` is set before settings import and defaults to `dev` |
| Unsupported database type | URI property raises `ValueError` |
| Default `SECRET_KEY` in production | Application can start but is insecure; deployment validation must reject it |
| `SECRET_KEY` rotated with stored credentials | Fernet decryption can fail unless `DATA_ENCRYPTION_KEY`/`DATA_ENCRYPTION_OLD_KEYS` is used for rotation |
| DB unavailable | Startup fails |
| Redis auth/timeout | Connection helper returns no client; later startup work cannot be considered ready |
| Empty DB on startup | Tables created from metadata, stamped `head` when migration history exists, then seeded |
| Existing DB on startup | `alembic upgrade head` applied automatically |
| Zombie `alembic_version` row | Unresolvable revision rows are deleted before upgrade |
| dev autogen emits destructive `DROP` | Revision file deleted, auto-apply aborted, manual revision/upgrade required |
| Seed table already has rows | Entire table seed is skipped (idempotent) |
| Missing `ENABLED_MODULES` | No module is disabled; README claim has no runtime effect |

## 5. Good / Base / Bad Cases

- Good: add a typed setting, example value, Compose/secret wiring, startup
  validation, safe log behavior, and tests for missing/invalid values.
- Base: add an optional non-secret setting with a safe default and document
  where it is consumed.
- Bad: add a lowercase env key, reorder an entry point so settings import
  precedes `ENVIRONMENT`, rotate `SECRET_KEY` without a decoupled
  `DATA_ENCRYPTION_KEY`, or rely on an external migration step that startup
  already performs.

## 6. Tests Required

- Instantiate settings with isolated process env and temporary `.env.dev` /
  `.env.prod` files; assert case, precedence, parsing, and invalid values.
- Assert production validation rejects the development `SECRET_KEY` once such
  a gate is introduced.
- Generate and review Alembic output; test upgrade (and downgrade where safe)
  against an existing schema, not only an empty database.
- Test startup failure/readiness for unavailable DB and Redis.
- Assert operation logs redact password/token/api-key fields; the redaction is
  already implemented, so new credential endpoints must not bypass it.
- Run `uv run ruff check .` and `uv run pytest` from `backend/`.

## 7. Wrong vs Correct

Wrong (ordering that reads settings before the environment is selected):

```python
from app.config.setting import settings
os.environ["ENVIRONMENT"] = env
```

Correct ordering, as used by every command in `backend/main.py`:

```python
os.environ["ENVIRONMENT"] = env
from app.config.setting import get_settings

settings = get_settings()
```