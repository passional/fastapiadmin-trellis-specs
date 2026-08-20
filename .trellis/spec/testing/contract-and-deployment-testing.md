# Contract and Deployment Testing

## 1. Scope / Trigger

Use this spec when a change crosses backend/Web/App, modifies response or
pagination fields, authentication, SSE/WebSocket protocols, generator
templates, environment/startup, migrations, Docker, Nginx, or public paths.

## 2. REST and authentication contracts

Normal JSON endpoints must agree on the five-field envelope `code`, `msg`,
`data`, `status_code`, and `success`. Backend/Web pagination uses `items`,
`total`, `page_no`, `page_size`, and `has_next`; App's ambient `list` is a known
mismatch. Contract tests should use representative success, empty, validation,
401, 403, and business-error responses and compare the adapters' exposed
values, not merely OpenAPI types.

Authentication assertions include exact carriers and bodies:

- REST access: `Authorization: Bearer <access JWT>`;
- refresh/logout: current backend consumes a JSON string body;
- refresh issues replacement JWTs and extends the Redis session, but current
  validation does not compare against stored token values, so old signed JWTs
  remain usable while their JWT/session conditions pass;
- logout invalidates the session selected by its JSON-string body, but does not
  require that token/session to match the authenticated header; assert both the
  normal same-session case and current mismatch/refresh-token behavior;
- user deletion, user disable, and role disable are separate revocation cases:
  current access observes deletion but not a post-login status change, while
  refresh checks current status and data-scope role queries ignore role
  status/deletion;
- role/menu revocation behavior is tested against an already logged-in
  session, because permissions are cached in Redis;
- operation logs are searched for sentinel passwords/tokens after every
  credential endpoint test.

Do not reuse `backend/tests/conftest.py::assert_route` for these contracts;
assert exact status, envelope, state, and absence of secret leakage.

## 3. Realtime contracts

Each channel is a separate protocol:

| Channel | Required regression points |
|---|---|
| Health SSE | event name `health`, immediate event, payload fields, interval behavior with controlled time, disconnect cleanup, intended public/auth boundary |
| AI chat WebSocket | subprotocol and query compatibility policy, invalid handshake, JSON/schema errors, chunks, `[DONE]`, disconnect, and stop observed before natural completion |
| System chat WebSocket | invalid-token close behavior, `ping`/`pong`, `message`/`read`/`presence` discriminants, recipient isolation, reconnect policy |
| Transfer WebSocket | invalid-token close, `ping`/`pong`, creator-only `task_update`, disconnect cleanup |

The repository currently has no dedicated realtime test suite. In particular,
the AI controller cannot read a stop message while awaiting generation, and
system-chat/transfer call close before accept for invalid auth. Tests must
assert client-observed behavior rather than assuming comments describe a
working protocol. Query tokens must not appear in captured proxy/application
logs when a safer carrier is introduced.

## 4. Generator/template regression strategy

There are currently no generator tests. Changes under `backend/templates/` or
`backend/app/api/v1/module_generator/gencode/` should add focused tests that:

1. persist representative `GenTableSchema` input and load the resulting
   `GenTableOutSchema`, including normalized package, module, business name,
   permissions, and a main/sub-table case;
2. render every path registered by
   `Jinja2TemplateUtil.get_template_list()` into a temporary output root;
3. assert the exact Python/TS/Vue file set, backend router/API prefix, Web API
   path, component path, and permission strings;
4. compare preview and ZIP registered outputs and account explicitly for the
   local-only package `__init__.py` scaffold;
5. assert invalid configuration, per-template preview error, partial ZIP
   failure with `X-Skipped-Tables`, all-failed behavior, overwrite, and the
   current file-before-menu-conflict behavior;
6. run Ruff on generated Python and Web type-check/build on generated TS/Vue
   when output syntax/types change.

Use temporary directories and controlled DB/menu fixtures. Never run local
generation into the working plugin/Web tree as a test.

## 5. Deployment and configuration smoke checks

No checked-in CI workflow or end-to-end browser suite currently owns these
checks. Run only the relevant commands/tools available in the target
environment and record exact evidence. A deployment-sensitive change should
verify:

- settings load with production-like required DB/Redis/JWT/OAuth values and
  secrets are absent from rendered config/log output;
- Alembic upgrade runs against the supported target database before the app
  serves traffic;
- `docker compose --env-file <test-env> config` resolves required variables;
- the backend image contains or deliberately mounts `main.py` and `app/`;
- Nginx config parses and serves built Docs/Web/App artifacts;
- direct `/common/health/live` and `/ready` plus their public `/api/v1/...`
  equivalents behave correctly, including DB/Redis failure;
- REST, SSE, and all WebSocket upgrades work through Nginx with the same public
  prefix and authentication policy;
- one backend replica is used until process-local realtime/cancellation state
  and scheduler execution have a distributed design and tests.

The current Compose healthcheck targets a nonexistent bare `/common/health`,
the backend image relies on the source bind mount, frontend artifacts are not
built by `deploy.sh`, and the public `/api/v1` proxy boundary lacks an
integration test. These are current gaps; a smoke test should expose them, not
encode them as successful expectations.

## 6. Change-to-test matrix

| Changed contract | Tests that must move together |
|---|---|
| Envelope/pagination field | Backend response + Web Axios consumer + App Alova consumer/type |
| Login/refresh/logout | Backend Redis/JWT + both client queues/storage + audit redaction |
| Permission/data scope | 401/403 + exact permission + own/dept/outside row + bulk mutation |
| WebSocket payload/carrier | Backend handshake/message + each consuming client + proxy log/upgrade |
| Upload/content | Backend bytes/path/permission + client upload parsing + static serving policy |
| Generator template/path | schema/context + exact output set + preview/ZIP/local + generated lint/type |
| Docker/Nginx/config | compose/config + image contents + live/ready + public REST/SSE/WS smoke |
