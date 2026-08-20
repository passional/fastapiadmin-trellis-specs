# Backend Quality Guidelines

## Toolchain and style

`backend/pyproject.toml` is authoritative. The project requires Python 3.12+,
uses Ruff for formatting/import/lint rules, and pytest with asyncio auto mode.
Ruff uses four-space indentation, double quotes, a configured 200-character
line length, and rules from FAST/F/E/W/I/B/C4/UP. Do not claim Black, Flake8,
Pylint, MyPy, Poetry, or a 100-character limit as repository requirements;
older documentation still mentions some of them but the active configuration
does not.

Run from `backend/`:

```bash
uv run ruff check .
uv run pytest
```

Use `uv run ruff format .` only when formatting is in scope, and review the
result because it is a writing command. `ruff check` is configured with
`fix=true`, so also inspect `git diff` after it runs.

## Repository contribution workflow

`CONTRIBUTING.md` asks contributors to discuss large changes in an issue first,
use a `feature/xxx` or `bugfix/xxx` branch, write conventional commit messages,
and target pull requests at `dev`. These are repository-wide collaboration
rules; the active `backend/pyproject.toml` commands above remain the backend
quality gate.

## Implementation review

For a normal API feature, verify the whole flow:

1. Controller has typed `Annotated` inputs, permission dependency, injected
   request session, `response_model`, and the common response wrapper.
2. Service owns business validation and returns a typed output schema.
3. CRUD uses `CRUDBase` where possible and respects soft deletion, audit
   fields, data-permission conditions, eager loading, and request transaction
   ownership.
4. Model and schema constraints agree; date serialization uses the documented
   Python-versus-JSON modes.
5. The area router mounts the controller, or the plugin satisfies dynamic
   discovery rules.
6. Errors do not leak internal details. The feature must not add secrets to
   logs/audit payloads; for credential or private-content write routes, account
   for the existing unredacted `OperationLogRoute` gap documented in the
   logging guide rather than claiming the default audit path is safe.

Prefer the working slice in `backend/app/api/v1/module_system/dept/` and the
extension slice in `backend/app/plugin/module_example/demo/` over code copied
from generated templates under `backend/templates/`.

## Tests

Tests live in `backend/tests/`. `conftest.py` configures a temporary SQLite DB,
an in-memory mocked Redis surface, a reduced lifespan, a shared `TestClient`,
authentication headers, and `assert_route`. `test_main.py` is the reference for
asserting both HTTP status and response data.

`assert_route` intentionally treats non-404 responses—and even application
exceptions—as proof that a route is mounted. It is useful for broad route
coverage in `test_api_module_system.py`, but it is not sufficient for new
business behavior. Add focused assertions for success data, validation,
authorization, conflicts, soft deletion, and state changes when those behaviors
change. Use pytest async support for direct async service/CRUD tests.

## Forbidden shortcuts

- Skipping service/CRUD layers by putting domain SQL in controllers.
- Calling `commit()` in ordinary request CRUD methods.
- Adding `# noqa`, broad Ruff ignores, or catch-all exceptions instead of
  addressing a local issue.
- Depending on a real developer database, Redis, clock, or network in the
  default test suite.
- Treating generated code or outdated documentation as stronger evidence than
  current source and configuration.

## Final check

Run the two commands above, review the backend diff, and inspect any changed
route with a representative success and failure test. For migrations, also
review the generated Alembic operations before applying them.
