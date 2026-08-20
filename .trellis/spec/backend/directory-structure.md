# Backend Directory Structure

## Runtime boundaries

The backend runs from `backend/`; commands and imports assume that directory is
the working directory. `backend/main.py` owns the Typer commands and constructs
the FastAPI application. `backend/app/init_app.py` owns lifespan setup and the
registration order for exceptions, middleware, routers, static assets, docs,
and the optional built web application.

```text
backend/
├── main.py                 # run/revision/upgrade CLI and app factory
├── app/
│   ├── api/v1/             # first-party API modules
│   ├── common/             # response envelopes, enums, shared data objects
│   ├── config/             # settings and filesystem paths
│   ├── core/               # DB/session, auth, CRUD, middleware, logging
│   ├── plugin/             # dynamically discovered extension modules
│   ├── scripts/            # initialization routines
│   ├── utils/              # reusable, mostly stateless helpers
│   └── alembic/            # migration environment and versions
└── tests/                  # pytest fixtures and API tests
```

## Business slices

First-party features are vertically sliced below
the `backend/app/api/v1/` tree in a `module_<area>/<feature>` slice. A full data-backed slice normally
contains:

- `controller.py`: FastAPI boundary only—typed request inputs, permission and
  DB dependencies, response model, and `SuccessResponse`.
- `service.py`: business rules, orchestration, output-schema conversion, and
  `CustomException` decisions.
- `crud.py`: a small feature-specific subclass of
  `backend/app/core/base_crud.py:CRUDBase`; add direct SQL only when the base
  operations cannot express the query.
- `model.py`: SQLAlchemy 2.0 mapped model.
- `schema.py`: Pydantic request, query, and output schemas.

`backend/app/api/v1/module_system/dept/` demonstrates the complete flow. A
feature without persistence may omit `crud.py` and `model.py`; examples include
the health and monitor subdomains. Do not create empty layers merely to match
the template.

Area routers are assembled explicitly in each `module_*/__init__.py`, then in
`backend/app/init_app.py`. For example,
`backend/app/api/v1/module_system/__init__.py` mounts `DeptRouter` below
`/system`, while `DeptRouter` adds `/dept`.

## Controllers and dependencies

Use an `APIRouter` with `route_class=OperationLogRoute`. Declare inputs with
`Annotated` and FastAPI's `Body`, `Path`, `Query`, `Depends`, or `Security`.
Authenticated endpoints inject both `AuthSchema` through `AuthPermission` and
`AsyncSession` through `db_getter`. Always declare `response_model` using
`ResponseSchema[...]`, including `ResponseSchema[None]` for empty success data.

Keep controllers thin. `backend/app/api/v1/module_system/dept/controller.py`
constructs `DeptService(auth, db)`, delegates once, and returns
`SuccessResponse`; it does not contain uniqueness checks, tree traversal, or
SQL.

## Plugins

Extension code belongs below `backend/app/plugin/` in a `module_<name>` directory, not inside a
first-party area. `backend/app/core/discover.py` scans
`module_*/**/controller.py`, imports top-level `APIRouter` instances, and maps
`module_example` to the `/example` container prefix. Every import-path segment
must be a valid Python identifier and should contain `__init__.py`.

Use `backend/app/plugin/module_example/demo/` as the structural example.
`plugin.toml` is optional metadata and does not install dependencies.

## Naming and placement

- Python modules and directories use `snake_case`; top-level areas use the
  `module_` prefix.
- Mapped classes, services, schemas, and CRUD classes use `PascalCase` plus the
  local suffix (`DeptModel`, `DeptService`, `DeptOutSchema`, `DeptCRUD`).
- Router variables use the feature name plus `Router` (`DeptRouter`).
- Put cross-feature framework behavior in `app/core/`; put shared protocol
  objects such as response envelopes and enums in `app/common/`; keep generic
  transformations in `app/utils/`.

Avoid business-specific helpers in `app/core/` or `app/utils/`, controllers
that execute SQL directly, manual imports of plugin routers in `init_app.py`,
and new top-level backend packages that bypass the existing app factory.

## Reference files

- `backend/main.py`
- `backend/app/init_app.py`
- `backend/app/api/v1/module_system/__init__.py`
- `backend/app/api/v1/module_system/dept/controller.py`
- `backend/app/api/v1/module_system/dept/service.py`
- `backend/app/core/discover.py`
- `backend/app/plugin/module_example/demo/controller.py`
