# Deployment Topology

## 1. Scope / Trigger

Use this contract for Docker images, Compose services, Nginx routes, frontend
artifacts, public URL prefixes, health checks, WebSockets, replicas, deploy
scripts, or production rollout changes.

## 2. Signatures and topology

The checked-in Compose topology is:

```text
client -> Nginx :80/:443
              -> static docs `/`, Admin Web `/web`, App H5 `/app`
              -> backend:8001 for `/api/v1` (HTTP + WebSocket upgrade)
backend -> MySQL + Redis
```

Evidence: `docker/docker-compose.yaml`, `docker/nginx/nginx.conf`,
`frontend/web/vite.config.ts`, and `frontend/app/vite.config.ts`.

Expected probe semantics from
`backend/app/api/v1/module_common/health/controller.py`:

| Direct backend route | Meaning | Healthy status |
|---|---|---|
| `/common/health/live` | Process is alive | 200 |
| `/common/health/ready` | DB and Redis are ready | 200; otherwise 503 |
| `/common/health/check` | Basic status/version/uptime | 200 |

The public form must be verified through the configured `/api/v1` proxy.

## 3. Contracts and current gaps

### Backend image/source contract

`docker/backend/Dockerfile` copies only `requirements.txt`; it does not copy
`backend/main.py` or `backend/app/`. The Compose bind mount
`../backend:/home` is therefore required for the current image to run. The
README advice to remove that mount for production is not executable until the
Dockerfile copies application source (and the resulting immutable image is
tested).

Do not remove the mount as an isolated hardening change. Either preserve the
current source-mounted deployment or first implement and verify a complete
image artifact.

### Frontend artifacts and deploy script

Nginx mounts `docker/nginx/{web,app,docs}` and expects built `dist` content.
The checked-in `deploy.sh` builds only Docker images; it has no frontend build
function and does not parse `--build-frontend`/`--skip-frontend`. The Docker
README describes those flags and `BUILD_WEB`, but `docker/.env.example` and the
script do not implement them. Build and copy frontend artifacts explicitly, or
add one tested implementation before documenting automatic builds.

### Public path/probe gaps

- Compose probes `/common/health`, but the backend exposes `/check`, `/live`,
  and `/ready` below `/common/health`; there is no bare health handler.
- Backend tests call direct paths without `/api/v1`, while Nginx proxies the
  unmodified `/api/v1` URI and backend uses `ROOT_PATH=/api/v1`. This boundary
  lacks an integration test; verify it instead of assuming prefix stripping.
- Nginx `/` serves VitePress and `/docs` is therefore a docs-site path, while
  the backend also registers Swagger at direct `/docs`. Docker README claims
  `/docs` is Swagger. Treat public documentation URLs as unresolved until the
  reverse-proxy routes are made explicit and tested.
- `NGINX_SERVER_NAME` is documented as an env variable, but
  `docker/nginx/nginx.conf` hard-codes `service.fastapiadmin.com` and Compose
  does not template it.

These are known deployment mismatches, not preferred conventions.

### Realtime and replica safety

System-chat and storage-transfer WebSocket managers keep
`user_id -> set[WebSocket]` in process memory. Transfer cancellation also uses
an in-process registry. Evidence:
`backend/app/api/v1/module_system/chat/ws_manager.py`,
`backend/app/api/v1/module_storage/transfer/ws_manager.py`, and
`backend/app/api/v1/module_storage/transfer/registry.py`.

APScheduler starts inside every application process, although its default
jobstore is Redis (`backend/app/core/ap_scheduler.py`). Starting multiple
backend replicas can duplicate scheduler listeners/system job execution and
split WebSocket presence/pushes. Deploy one backend process/replica unless a
coordinated pub/sub, distributed connection/cancellation model, and scheduler
leader/worker design is implemented and tested.

The split WebSocket state follows directly from the process-local dictionaries.
Duplicate scheduler execution is an operational inference from starting an
independent scheduler in every process against a shared job store; the
repository has no multi-process test proving safe coordination. Keep that
distinction explicit when evaluating a future replica design.

### Operational safety

Compose exposes MySQL and Redis host ports by default and mounts mutable source
into the backend. Restrict these in production deliberately. Docker JSON logs
rotate at 10 MB x 3 files per service, while application file logs also rotate
daily under `backend/logs/`.

`deploy.sh` calls `docker system prune -a -f` during a full deploy and through
`clean`; this removes unused Docker resources host-wide, not only this project.
Do not run it on a shared host without accepting that scope.

## 4. Validation & Error Matrix

| Condition | Operational result / required response |
|---|---|
| Required MySQL/Redis password absent | Compose interpolation fails (`:?`) |
| Backend bind mount removed with current image | `main.py`/application source is absent |
| Bare `/common/health` probe | 404; container cannot become healthy |
| DB/Redis down | `/ready` returns 503; `/live` should remain 200 |
| Missing frontend `dist` | Nginx serves missing/404 content |
| Missing TLS files | Nginx config/start fails |
| Public prefix not stripped/matched | `/api/v1/...` requests return 404 |
| Multiple backend replicas | Presence, pushes, cancellation, and scheduler behavior diverge/duplicate |
| Query-string WebSocket JWT | Token can appear in proxy/access logs; use AI subprotocol where the client supports it |

## 5. Good / Base / Bad Cases

- Good: build immutable backend and frontend artifacts, run migration as an
  explicit release step, start one backend replica, verify live/ready plus REST
  and WebSocket paths through Nginx, and retain rollback artifacts.
- Base: use the current source-mounted single-instance Compose deployment after
  fixing/overriding the health probe and supplying prebuilt static assets.
- Bad: remove the backend mount because the README says to, trust unimplemented
  frontend build flags, or scale replicas without realtime/scheduler design.

## 6. Tests Required

- `docker compose --env-file .env config` with required variables supplied; no
  secrets in committed output.
- Build the backend image, inspect that the chosen source strategy provides
  `/home/main.py`, and run the container without relying on an undocumented
  host state.
- Run `nginx -t`; verify `/web`, `/app` (if enabled), docs, REST, SSE, and each
  WebSocket upgrade through the public origin.
- Probe direct `/common/health/live` and `/ready`, then their public equivalents;
  simulate DB and Redis failure and assert live/ready divergence.
- Apply Alembic upgrade before application startup in a production-like test.
- Assert a single backend process or add distributed tests before increasing
  replicas.

## 7. Wrong vs Correct

Wrong with the current Dockerfile:

```yaml
# Removing the only application-source provider leaves the image incomplete.
backend:
  volumes: []
```

Correct current-state decision:

```text
Keep ../backend:/home, or first COPY the backend source into the image and prove
the image boots, migrates, serves health/API/WebSocket traffic, and can roll back.
```
