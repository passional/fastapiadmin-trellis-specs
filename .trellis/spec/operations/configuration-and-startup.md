# Configuration and Startup

## 1. Scope / Trigger

Use this contract when adding or changing an environment key, secret, database
or Redis dependency, startup task, seed, scheduler job, migration, or runtime
module gate.

## 2. Signatures

Settings are a case-sensitive Pydantic `BaseSettings` model:

```python
SettingsConfigDict(
    env_file=ENV_DIR / f".env.{os.getenv('ENVIRONMENT')}",
    extra="ignore",
    case_sensitive=True,
)
```

The supported file convention is `backend/env/.env.dev` or `.env.prod`, copied
from `backend/env/.env.example`. Process environment variables override file
values. `docker/docker-compose.yaml` sets selected backend variables before the
Python process starts.

The backend CLI is run from `backend/`:

```text
uv run main.py run --env=dev
uv run main.py revision --env=dev
uv run main.py upgrade --env=dev
```

## 3. Contracts

### Environment ownership

| Concern | Backend/runtime keys | Other owners |
|---|---|---|
| Runtime/profile | `ENVIRONMENT`, `DEBUG`, `SERVER_HOST`, `SERVER_PORT`, `LOGGER_LEVEL` | `backend/app/config/setting.py`, `backend/env/.env.example` |
| Database | `DATABASE_TYPE/HOST/PORT/USER/PASSWORD/NAME`, pool keys | Compose maps MySQL values into backend keys |
| Redis | `REDIS_HOST/PORT/DB_NAME/USER/PASSWORD` | Compose Redis service and backend environment |
| Auth/security | `SECRET_KEY`, token expiry, `ALLOWED_HOSTS`, `PROD_CORS_ORIGINS`, OAuth/WX secrets | Nginx host/TLS configuration is separate |
| AI | `OPENAI_BASE_URL`, `OPENAI_API_KEY`, `OPENAI_MODEL` | Per-user model configs are also stored in Redis |
| Clients | `VITE_API_BASE_URL`, `VITE_APP_BASE_API`, `VITE_APP_WS_ENDPOINT`, app base paths | Each frontend package owns its env/type declarations |

Add a key to `Settings` and every relevant example/deployment mapping; add a
frontend key to that package's `ImportMetaEnv` declaration. Never assume a
similarly named Compose variable is consumed automatically.

`backend/main.py` imports the global `settings` object before any command sets
`ENVIRONMENT`. Consequently, `run --env=...` uses the already-created object;
`revision` and `upgrade` clear the factory cache but do not rebind that global
object before Alembic imports it. When `ENVIRONMENT` was not already set by the
parent process, the settings model also computes an `.env.None` path instead of
`.env.dev`/`.env.prod`. Docker avoids this specific ordering bug because Compose
sets `ENVIRONMENT` before Python starts. Treat CLI `--env` selection as a
current defect, not as proof that the requested file was loaded. New entry
points must set the environment before importing any settings consumer, and a
fix needs subprocess tests for `run`, `revision`, and `upgrade`.

### Secret and logging safety

- Production must override the development `SECRET_KEY`. It signs JWTs in
  `backend/app/core/security.py` and derives the Fernet key for storage-source
  passwords in `backend/app/api/v1/module_storage/core/encrypt.py`; changing it
  invalidates tokens and can make stored credentials undecryptable.
- System-default AI credentials come from env, but user model configurations
  currently store `api_key` as JSON in Redis for seven days and return it to
  the authenticated owner. Evidence:
  `backend/app/api/v1/module_ai/chat/{schema.py,service.py}`.
- `OperationLogRoute` currently captures form/JSON request bodies and JSON
  response bodies without field redaction. Existing audited routes can persist
  login passwords and returned JWTs, refresh/logout tokens, AI model API keys,
  and system/AI chat content. The 2,000-character replacement applies only to
  the serialized request; responses have no equivalent cap. Treat this as an
  existing data-exposure gap, not only a risk for future endpoints. Evidence:
  `backend/app/core/router_class.py`, the auth and AI chat controllers, and
  `backend/app/api/v1/module_system/chat/controller.py`.
- Do not log connection URIs, authorization headers, query-string JWTs,
  decrypted storage credentials, or full third-party responses containing
  secrets.

### Startup and migrations

`backend/app/init_app.py:lifespan` performs, in order:

1. `InitializeData.init_db()` checks the DB, runs metadata `create_all`, and
   seeds empty core tables from `backend/sql/data/`.
2. Redis connects and is stored on `app.state`.
3. parameter and dictionary caches initialize.
4. APScheduler starts and registers the operation-log cleanup job.
5. Shutdown stops the scheduler, Redis, and DB engine.

`create_all` only creates missing tables; it does not safely migrate an
existing schema. Production schema changes require reviewed Alembic revisions
under `backend/app/alembic/versions/` and an explicit `upgrade` before serving
new code. The versions directory currently contains only `__init__.py`, and
the Docker command does not run Alembic. This is a current rollout gap that
must be resolved for the first production migration.

## 4. Validation & Error Matrix

| Condition | Expected/Current behavior |
|---|---|
| Wrong-case env key | Ignored because settings are case-sensitive |
| Unknown env key | Ignored (`extra="ignore"`) |
| CLI `--env` without pre-set `ENVIRONMENT` | Global settings were already created; requested `.env.*` is not reliably loaded |
| Unsupported database type | URI property raises `ValueError` |
| Default `SECRET_KEY` in production | Application can start but is insecure; deployment validation must reject it |
| `SECRET_KEY` rotated with stored credentials | Storage password decryption can fail |
| DB unavailable | Startup fails |
| Redis auth/timeout | Connection helper returns no client; later startup work cannot be considered ready |
| Seed table already has rows | Entire table seed is skipped |
| Model changed without migration | `create_all` does not alter the existing table |
| Missing `ENABLED_MODULES` | No module is disabled; README claim has no runtime effect |

## 5. Good / Base / Bad Cases

- Good: add a typed setting, example value, Compose/secret wiring, startup
  validation, safe log behavior, and tests for missing/invalid values.
- Base: add an optional non-secret setting with a safe default and document
  where it is consumed.
- Bad: add a lowercase env key, rely on `--env` after importing settings,
  rotate `SECRET_KEY` without credential migration, or use `create_all` as an
  upgrade strategy.

## 6. Tests Required

- Instantiate settings with isolated process env and temporary `.env.dev` /
  `.env.prod` files; assert case, precedence, parsing, and invalid values.
- Assert production validation rejects the development `SECRET_KEY` once such
  a gate is introduced.
- Generate and review Alembic output; test upgrade (and downgrade where safe)
  against an existing schema, not only an empty database.
- Test startup failure/readiness for unavailable DB and Redis.
- Assert operation logs redact password/token/api-key fields before adding new
  credential endpoints.
- Run `uv run ruff check .` and `uv run pytest` from `backend/`.

## 7. Wrong vs Correct

Wrong:

```python
from app.config.setting import settings
os.environ["ENVIRONMENT"] = env
```

Correct ordering for new entry points:

```python
os.environ["ENVIRONMENT"] = env
from app.config.setting import get_settings

settings = get_settings()
```
