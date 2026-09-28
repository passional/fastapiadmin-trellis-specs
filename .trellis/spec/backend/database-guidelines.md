# Database Guidelines

## Stack and ownership

The persistence layer uses SQLAlchemy 2.0 with `AsyncSession`. Engine and
session factories live in `backend/app/core/database.py`; request handlers get
their session only through `backend/app/core/dependencies.py:db_getter`.
SQLite, MySQL, and PostgreSQL URLs are supported by settings.

The HTTP transaction boundary is the request dependency:

```python
async with async_db_session() as session, session.begin():
    yield session
```

Consequently, normal CRUD and service methods call `flush()` when generated
values are needed but do not call `commit()`. `CRUDBase` states and implements
this contract. Explicit commits exist only in self-owned transaction flows such
as storage transfer
(`backend/app/modules/task/storage/transfer/engine.py`) and the log cleanup job
(`backend/app/modules/system/log/service.py:151`); treat those as specialized
transaction owners, not as the pattern for ordinary request services. There is
no `commit()` anywhere under `backend/app/modules/ai/`.

## Models

Use SQLAlchemy's typed mapping (`Mapped[T]` and `mapped_column`). Normal
business tables inherit `ModelMixin` and, when audit users are needed,
`UserMixin` from `backend/app/core/base_model.py`. These supply integer `id`,
UUID, soft-delete timestamps, UTC create/update timestamps, and audit user
relationships.

Declare an explicit snake_case table name with its domain prefix. Current
examples are `sys_dept` in `backend/app/modules/system/dept/model.py` and the
storage tables below `backend/app/modules/task/storage/`. Use snake_case columns, typed nullable
unions, database constraints, field comments, `ForeignKey`, and
`relationship(..., back_populates=...)`. Add indexes for demonstrated access
paths. `ModelMixin` defines common status/deletion and creation indexes, but a
model's explicit `__table_args__` replaces those defaults; review the effective
table arguments rather than assuming both sets are present.

Do not introduce an alternative declarative base, untyped legacy `Column`
models, naive datetime defaults, or hard deletes for a model that supports the
standard soft-delete fields.

## Schemas and query operators

Keep request/query/output contracts in the feature's `schema.py`:

- Create schemas inherit `BaseModel` and apply `Field` constraints plus focused
  validators.
- Update schemas commonly inherit the create schema when PUT requires the same
  shape.
- Output schemas combine the feature schema with `BaseSchema` and optionally
  `UserBySchema`, then set `ConfigDict(from_attributes=True)`.
- Query schemas inherit `BaseQueryParam` and optionally `UserByQueryParam`.
  Each searchable field declares its operator through
  `json_schema_extra={"q": "like"}` / `"eq"`; date ranges are inherited.

Convert query schemas with `search_to_dict()` in the service before passing
them into CRUD. `backend/app/modules/system/dept/schema.py` and
`service.py` are the reference pair. For ORM writes use
`model_dump(mode="python")` semantics so native date/time values survive; for
JSON or Redis use `model_dump(mode="json")`. The date serializers are defined
in `backend/app/core/validator.py` and explained in `backend/README.md`.

## CRUD and loading

Prefer a typed `CRUDBase[Model, CreateSchema, UpdateSchema]` subclass. The base
implementation provides get/get-or-404, count/list/page, audit population,
soft delete/restore, bulk set, ordering, permission conditions, and loader
options. It automatically excludes soft-deleted rows unless
`include_deleted=True`.

Use `get_or_404()` for required objects and `exists()`/`count()` when the row is
not needed. For list filters use operator tuples produced by
`search_to_dict()` rather than hand-built string SQL. Pass relationship names
through `preload`; the base selects `joinedload` for scalar relationships and
`selectinload` for collections and automatically loads `created_by` and
`updated_by` where present.

If custom SQL is necessary, keep it in the feature CRUD/service and use
SQLAlchemy expressions with the injected session. Never interpolate user input
into SQL strings or create a fresh request-local session inside a controller.

## Migrations and initialization

`backend/app/scripts/initialize.py` runs on **every** startup from `lifespan()`.
It is the schema-bring-up owner, not a first-run-only seeder:

- Empty database (sentinel `sys_menu` table absent): `create_tables()` builds
  the full schema from the models, then `alembic stamp head` records the
  baseline when the repository has migration history.
- Existing database: `command.upgrade(cfg, "head")` applies pending migrations.
- Before upgrading it self-heals zombie `alembic_version` rows whose revision
  scripts no longer exist in the repository.
- In `ENVIRONMENT == DEV` it additionally autogenerates a migration, applies it
  when one is produced, and **deletes and aborts** any generated revision whose
  `upgrade` section contains destructive `op.drop_*` calls or a hand-written
  `op.execute("... DROP ...")`.

Seed data is imported after migration from flat JSON files under
`backend/sql/*.json` (`SCRIPT_DIR = BASE_DIR / "sql"`), one file per table
(`sys_menu.json`, `sys_role.json`, `sys_user.json`, `sys_dept.json`,
`sys_dict_*.json`, `sys_param.json`, ...). Tables that already contain rows are
skipped, so seeding is idempotent. The JSON files sit directly in
`backend/sql/`; there is no nested `data/` subdirectory.

The migration environment in `backend/app/alembic/env.py` discovers mapped
models, compares types and server defaults, and suppresses empty revisions.

From `backend/`:

```bash
uv run main.py run --env=dev       # start (applies migrations + seed)
uv run main.py revision --env=dev  # generate a revision
uv run main.py upgrade --env=dev   # apply revisions
```

Review every generated revision before applying it. Keep generated files under
`backend/app/alembic/versions/`; the directory currently contains no historical
revision beyond `__init__.py`, so do not invent a manual baseline or edit
initialization SQL as a substitute for a migration without an explicit rollout
decision. Docker containers run `python main.py run --env=prod`, so they apply
migrations automatically on start.

## Verification

For model/query changes run from `backend/`:

```bash
uv run ruff check .
uv run pytest
```

Add a behavior assertion, not only a route-exists assertion, when changing
constraints, permissions, soft deletion, pagination, or transaction behavior.
