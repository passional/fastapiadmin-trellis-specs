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
- refresh uses rotation with replay detection: the submitted refresh token is
  compared against the value stored for the session, and a mismatch revokes the
  session, so a replayed/older refresh token does not silently issue new
  credentials — assert both the normal rotation and the revoke-on-mismatch path;
- logout invalidates the session selected by its JSON-string body, but does not
  require that token/session to match the authenticated header; assert both the
  normal same-session case and current mismatch behavior;
- user deletion, user disable, and role disable are separate revocation cases:
  current access observes deletion but not a post-login status change, while
  refresh checks current status and data-scope role queries ignore role
  status/deletion;
- role/menu revocation behavior is tested against an already logged-in
  session, because permissions are cached in Redis;
- operation logs are searched for sentinel passwords/tokens after every
  credential endpoint test. This is a **regression guard on implemented
  redaction**, not a known gap: `OperationLogRoute` redacts `_SENSITIVE_KEYS`
  from the request JSON body, form fields, and the JSON response body.

Do not reuse `backend/tests/conftest.py::assert_route` for these contracts;
assert exact status, envelope, state, and absence of secret leakage.

## 3. Realtime contracts

Each channel is a separate protocol:

| Channel | Required regression points |
|---|---|
| Health SSE (`GET /api/v1/monitor/health/stream`) | event name `health`, immediate event, payload fields, interval behavior with controlled time, disconnect cleanup, intended public/auth boundary |
| AI chat WebSocket (`/api/v1/ai/chat/ws`) | handshake auth via `Sec-WebSocket-Protocol` (`["access_token", "access_token."+jwt]`) for Web and `?token=` compatibility for App, invalid handshake, close code 4001 (auth) and 4003 (permission), JSON/schema errors, chunks, `[DONE]`, disconnect, and stop observed before natural completion |
| Transfer SSE (`GET /api/v1/task/storage/transfer/stream`) | event stream framing, creator-only progress delivery, disconnect cleanup |

The repository currently has no dedicated realtime test suite. In particular,
the AI controller cannot read a stop message while awaiting generation, so
"stop" before natural completion is not observable as designed. Tests must
assert client-observed behavior rather than assuming comments describe a
working protocol. The query-token compatibility path for App must not leak into
proxy/application logs; Web already uses the subprotocol carrier.

## 4. Generator/template regression strategy

There are currently no generator tests. Changes under `backend/templates/` or
`backend/app/modules/generator/gencode/` should add focused tests that:

1. persist representative `GenTableSchema` input and load the resulting
   `GenTableOutSchema`, including normalized package, module, business name,
   permissions, and a main/sub-table case;
2. render every path registered by
   `Jinja2TemplateUtil.get_template_list()` into a temporary output root. The
   registered python set now includes `package_init.py.jinja2` and
   `plugin.toml.jinja2` alongside the feature-level
   `controller/service/crud/schema/model/__init__` templates;
3. assert the exact Python/TS/Vue file set, backend router/API prefix, Web API
   path, component path, and permission strings;
4. compare preview and ZIP registered outputs. The package `__init__.py`
   (`package_init.py.jinja2`) is a registered template, so it **is** in
   preview/ZIP; local generation additionally writes a module-level
   `__init__.py` that is not a registered template — account for that
   difference explicitly;
5. assert invalid configuration, per-template preview error, partial ZIP
   failure with `X-Skipped-Tables`, all-failed behavior, overwrite, and the
   current file-before-menu-conflict behavior, plus the `sync_db/preview`
   endpoint contract;
6. run Ruff on generated Python and Web type-check/build on generated TS/Vue
   when output syntax/types change.

Use temporary directories and controlled DB/menu fixtures. Never run local
generation into the working plugin/Web tree as a test.

Jinja2 pitfall to keep in mind when writing template tests: `{% for %}` +
`{% set %}` accumulator booleans leak scope (for-block isolation), so derive
flags with a one-shot filter expression (e.g.
`columns | selectattr('python_type','equalto','date') | list | length > 0`)
instead of accumulating inside a loop.

## 5. Deployment and configuration smoke checks

No checked-in CI workflow or end-to-end browser suite currently owns these
checks. Run only the relevant commands/tools available in the target
environment and record exact evidence. A deployment-sensitive change should
verify:

- settings load with production-like required DB/Redis/JWT/OAuth values and
  secrets are absent from rendered config/log output;
- startup auto-migration runs against the supported target database before the
  app serves traffic (`scripts/initialize.py`: empty DB → `create_all` + stamp
  head; existing DB → `alembic upgrade head`; dev additionally autogenerates
  with a destructive-DROP guard), and `test_migrations.py` stays green;
- `docker compose --env-file <test-env> config` resolves required variables;
- the backend image contains `main.py` and `app/` via `COPY ./backend/ .`; the
  compose bind mount `../backend:/home` is commented out, so the image copy is
  the source of truth;
- Nginx config parses and serves built Docs/Web/App artifacts;
- `GET /api/v1/monitor/health/check` succeeds over the real route (only
  `/monitor/health/check` and `/monitor/health/stream` exist — there is no
  `/common/health`, no `/live`, and no `/ready`), including DB/Redis failure;
- REST, SSE, and all WebSocket upgrades work through Nginx with the same public
  prefix and authentication policy; `/api/v1` applies `limit_req` and
  `limit_conn`;
- one backend replica is used until process-local realtime/cancellation state
  and scheduler execution have a distributed design and tests.

The current Compose healthcheck targets `http://localhost:8001/api/v1/monitor/health/check`
with `X-Forwarded-Proto: https`. `deploy.sh` does not build frontend artifacts
(they must be built and synced by hand, see below) and its cleanup runs
`docker image prune -f` + `docker builder prune -f` only — it deliberately does
not run `docker system prune -a`. The public `/api/v1` proxy boundary lacks an
integration test. These are current gaps; a smoke test should expose them, not
encode them as successful expectations.

Frontend deployment artifacts are covered by the dist verification procedure in
`frontend-testing.md` §4: build (`pnpm build:prod`), `rsync -a --delete` the
output (`frontend/web/dist`) into the web-serving copies `docker/nginx/web/dist`
and `backend/dist`, rebuild the App H5 into `docker/nginx/app/dist/build/h5`,
then grep the built chunks for feature strings/invariants. `register_frontend`
must check and mount the same
`path_conf.FRONTEND_DIST_DIR`; a mismatch only fails on the first request and is
visible in `backend/logs/fastapiadmin.log`.

## 6. Change-to-test matrix

| Changed contract | Tests that must move together |
|---|---|
| Envelope/pagination field | Backend response + Web Axios consumer + App Alova consumer/type |
| Login/refresh/logout | Backend Redis/JWT + both client queues/storage + audit redaction |
| Permission/data scope | 401/403 + exact permission + own/dept/outside row + bulk mutation |
| SSE/WebSocket payload/carrier | Backend handshake/message + each consuming client + proxy log/upgrade |
| Upload/content | Backend bytes/path/permission + client upload parsing + static serving policy |
| Generator template/path | schema/context + exact output set + preview/ZIP/local + generated lint/type |
| Docker/Nginx/config | compose/config + image contents + health check/stream + public REST/SSE/WS smoke |
| Built web/App artifact | `pnpm build:prod`/`build:h5` + `rsync --delete` + grep of `docker/nginx/web/dist` and `backend/dist` |