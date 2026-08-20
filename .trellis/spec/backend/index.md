# Backend Development Guidelines

The backend is the Python 3.12+ FastAPI application under `backend/`. These
guides describe the current async SQLAlchemy, Pydantic v2, Loguru, and pytest
patterns. `backend/README.md` is useful supporting documentation; source and
`backend/pyproject.toml` are authoritative when older docs disagree.

## Guides

| Guide | Use it for |
|---|---|
| [Directory Structure](./directory-structure.md) | Runtime boundaries, business slices, routers, and plugins |
| [Database Guidelines](./database-guidelines.md) | Models, async sessions, CRUD, queries, and Alembic |
| [Error Handling](./error-handling.md) | Domain failures and the uniform HTTP error envelope |
| [Logging Guidelines](./logging-guidelines.md) | Loguru, correlation IDs, operation audit logs, and sensitive data |
| [Quality Guidelines](./quality-guidelines.md) | Ruff, pytest, review expectations, and verification |

Read all five guides for a new backend feature. The concrete reference slice is
`backend/app/api/v1/module_system/dept/`; the dynamically discovered plugin
example is `backend/app/plugin/module_example/demo/`.
