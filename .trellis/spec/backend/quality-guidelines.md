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
uv run main.py run --env=dev  # start the app (dev)
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
5. The controller is registered in `DOMAIN_CONTROLLERS`
   (`backend/app/api/v1/routers.py`), or the plugin satisfies dynamic discovery
   rules. Remember that discovery runs only at startup: restart the backend
   after adding or generating a module, otherwise it silently 404s.
6. Errors do not leak internal details. The feature must not add secrets to
   logs/audit payloads. `OperationLogRoute` redacts known sensitive keys in the
   request body, form fields, and response body (see the logging guide), but
   redaction is key-based and free-text content is captured verbatim.

Prefer the working slice in `backend/app/modules/system/dept/` and the
extension slice in `backend/app/plugin/module_example/demo/` over code copied
from generated templates under `backend/templates/`.

## Code generation

The generator is `backend/app/modules/generator/gencode/`, rendering
`backend/templates/{python,ts,vue}/*.jinja2`. Generated code must be reviewed and
then the backend restarted.

Jinja2 pitfall: a `{% for %}` loop with a `{% set %}` accumulator flag leaks
scope (for-block isolation), so loop-local flags do not survive outside the
loop. Compute such booleans with a single filter expression instead, as
`templates/python/schema.py.jinja2` does:

```jinja2
{% if columns | selectattr('python_type', 'equalto', 'date') | list | length > 0 %}
```

This exact pattern is a real fix: an accumulator-based version dropped a needed
validator import from the generated `schema.py`.

## Tests

Tests live in `backend/tests/`, currently `conftest.py`, `test_main.py`, and
`test_migrations.py`. `conftest.py` configures a temporary SQLite DB, an
in-memory mocked Redis surface, a reduced lifespan, a shared `TestClient`,
authentication headers, and the `assert_route` helper. `test_main.py` is the
reference for asserting both HTTP status and response data (it checks the
`/monitor/health/check/` envelope).

`assert_route` intentionally treats non-404 responses—and even application
exceptions—as proof that a route is mounted. It currently has **zero call
sites** (`conftest.py:240`) after `test_api_module_system.py` was removed, and it
is never sufficient for new business behavior. Add focused assertions for
success data, validation, authorization, conflicts, soft deletion, and state
changes when those behaviors change. Use pytest async support for direct async
service/CRUD tests.

`test_migrations.py` is the create_all-versus-metadata drift guard. It asserts
that the test layout (`init_db`) produces no leaked migration files under
`versions/`, and that an explicit `autogenerate` run produces zero new
revisions—catching schema drift between the fresh-install bootstrap path and the
model metadata. Treat a failure here as a real model/migration inconsistency,
not a flaky test.

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

Run the checks above, review the backend diff, and inspect any changed
route with a representative success and failure test. For migrations, also
review the generated Alembic operations before applying them.
