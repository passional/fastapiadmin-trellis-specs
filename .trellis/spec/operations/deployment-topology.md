# Deployment Topology

## 1. Scope / Trigger

Use this contract for Docker images, Compose services, Nginx routes, frontend
artifacts (`dist`), public URL prefixes, health checks, realtime streams,
replicas, deploy scripts, or production rollout changes.

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

Backend health routes (`backend/app/modules/monitor/health/controller.py`):

| Public route | Meaning | Healthy status |
|---|---|---|
| `/api/v1/monitor/health/check` | Real-time DB/Redis connectivity + version/uptime | 200 |
| `/api/v1/monitor/health/stream` | SSE frame every 30s with DB/Redis status | 200 |

Only these two routes exist: there is no liveness/readiness split and no health
endpoints outside the `/monitor` domain. Compose's backend healthcheck already
probes
`http://localhost:8001/api/v1/monitor/health/check` with
`X-Forwarded-Proto: https` (`docker/docker-compose.yaml:138-147`).

## 3. Contracts and current gaps

### Backend image/source contract

`docker/backend/Dockerfile` installs `requirements.txt`, then does
`COPY ./backend/ .`, so the image ships `main.py` and `app/` itself. The Compose
backend bind mount `../backend:/home` is **commented out**
(`docker/docker-compose.yaml:122-130`); only the upload directory
`./backend/static/upload:/home/static/upload` is mounted, to persist user files.
Removing the commented mount is safe — it exists only as an optional dev
hot-reload convenience, and production runs the image's own source.

### Frontend artifacts, dist sync, and deploy script

Nginx mounts `docker/nginx/{web,app,docs}` and expects built `dist` content.
All `dist` directories are generated artifacts, gitignored, and absent in a
fresh checkout (`.gitignore` ignores `dist`, `docker/nginx/{web,app,docs}/dist`).
The deployment chain has three web artifact locations plus the App H5 build:

| Artifact | Serving entry |
|---|---|
| `frontend/web/dist` | `pnpm build:prod` output (`vite.config.ts` `outDir: "dist"`) |
| `docker/nginx/web/dist` | docker `/web` (`nginx.conf:103-106`, `alias /usr/share/nginx/html/web/dist`) |
| `backend/dist` | integrated hosting via `app.frontend("/")` (`path_conf.FRONTEND_DIST_DIR`, `backend/app/__init__.py:125-129`) |
| `docker/nginx/app/dist/build/h5` | docker `/app` (`nginx.conf:110-113`) |

Build once and sync with `--delete` (minified chunk filenames carry a content
hash; without `--delete` stale chunks linger):

```bash
cd frontend/web && pnpm build:prod
cd ../.. && rsync -a --delete frontend/web/dist/ docker/nginx/web/dist/ \
              && rsync -a --delete frontend/web/dist/ backend/dist/
```

Verify the **built artifact**, not only `src/` (function names/comments are lost
to minify, but property accesses like `WebSocket.CLOSED` and message strings
survive):

```bash
grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js
grep -oE '.{0,60}readyState.{0,60}' <chunk> | grep -i websocket
```

`register_frontend` pitfall: the existence check and the `app.frontend()` mount
must use the same `path_conf.FRONTEND_DIST_DIR`. It mounts with
`check_dir=False`, so a mismatch fails only on the **first request**, and the
error surfaces only in `backend/logs/fastapiadmin.log`.

The checked-in `deploy.sh` (repo root) builds Docker images only; it has no
frontend build function and does not parse `--build-frontend`/`--skip-frontend`.
The Docker README describes those flags and `BUILD_WEB`, but neither
`docker/.env.example` nor the script implements them. Build and copy frontend
artifacts explicitly, or add one tested implementation before documenting
automatic builds. Neither a Windows batch deploy script nor a nested
`docker/deploy.sh` exists; only the repo-root script does.

### Public path/probe gaps

- Nginx now **applies** rate limiting: `location /api/v1` uses
  `limit_req zone=api_limit burst=60 nodelay;` and `limit_conn conn_limit 100;`
  (`docker/nginx/nginx.conf:116-118`), on top of the declared
  `api_limit`/`conn_limit` zones.
- Nginx `/` serves the docs site (`root /usr/share/nginx/html/docs/dist`); the
  backend also registers Swagger at direct `/docs`. Docker README claims `/docs`
  is Swagger. Treat public documentation URLs as unresolved until the
  reverse-proxy routes are made explicit and tested.
- `NGINX_SERVER_NAME` is documented as an env variable, but
  `docker/nginx/nginx.conf` hard-codes `service.fastapiadmin.com` in both server
  blocks and Compose does not template it.

These are known deployment mismatches, not preferred conventions.

### Realtime and replica safety

Storage-transfer progress is now **SSE**, not WebSocket:
`GET /api/v1/task/storage/transfer/stream` (`response_class=EventSourceResponse`),
served by `backend/app/modules/task/storage/transfer/{controller,sse_manager}.py`
with the shared core helper `backend/app/core/sse_manager.py`. The legacy
system-chat channel is gone and the WebSocket manager was replaced by
`backend/app/core/sse_manager.py`; reference the SSE manager only. The frontend
consumer is
`frontend/web/src/api/module_storage/transfer.ts` via `createSSEClient`
(`@utils/sse`); health SSE uses `backend/app/modules/monitor/health/controller.py`.

SSE connection state and transfer cancellation remain in-process registries.
APScheduler starts inside every application process, although its default
jobstore is Redis (`backend/app/core/ap_scheduler.py`). Starting multiple
backend replicas can duplicate scheduler listeners/system job execution and
split SSE presence/pushes. Deploy one backend process/replica unless a
coordinated pub/sub, distributed connection/cancellation model, and scheduler
leader/worker design is implemented and tested.

The split realtime state follows directly from the process-local registries.
Duplicate scheduler execution is an operational inference from starting an
independent scheduler in every process against a shared job store; the
repository has no multi-process test proving safe coordination. Keep that
distinction explicit when evaluating a future replica design.

### Operational safety

Compose exposes MySQL and Redis host ports by default. Restrict these in
production deliberately. Docker JSON logs rotate at 10 MB x 3 files per service,
while application file logs also rotate daily under `backend/logs/`.

`deploy.sh` cleanup runs only `docker image prune -f` and
`docker builder prune -f` (`deploy.sh:116-121`), with an explicit comment
rejecting `-a` because it would delete other projects' unused images on a shared
host. The script no longer git-pulls; it expects code to be uploaded, loads
`docker/.env`, and uses `PROJECT_NAME="FastapiAdmin"`.

## 4. Validation & Error Matrix

| Condition | Operational result / required response |
|---|---|
| Required MySQL/Redis password absent | Compose interpolation fails (`:?`) |
| Backend bind mount absent | Normal: image ships source via `COPY ./backend/ .` |
| `/api/v1/monitor/health/check` probe | 200 when DB and Redis are reachable, else 503-equivalent body |
| DB/Redis down | `/monitor/health/check` reports `redis_status`/`db_status` != 1 |
| Missing frontend `dist` | Nginx / `register_frontend` serve missing/404 content |
| Missing TLS files | Nginx config/start fails |
| Public prefix not stripped/matched | `/api/v1/...` requests return 404 |
| Multiple backend replicas | SSE presence/cancellation and scheduler behavior diverge/duplicate |
| Query-string WebSocket JWT | Token can appear in proxy/access logs; Web uses the AI subprotocol, App still uses `?token=` |

## 5. Good / Base / Bad Cases

- Good: build immutable backend and frontend artifacts, sync `dist` with
  `--delete`, start one backend replica, verify health/check plus REST and SSE
  paths through Nginx, and retain rollback artifacts.
- Base: use the current single-instance Compose deployment and supply prebuilt
  static assets.
- Bad: trust unimplemented frontend build flags, ship un-synced `dist`
  directories, skip `--delete` on the artifact sync, or scale replicas without
  realtime/scheduler design.

## 6. Tests Required

- `docker compose --env-file .env config` with required variables supplied; no
  secrets in committed output.
- Build the backend image and run the container without a host source mount;
  confirm `/home/main.py` and `app/` come from the image.
- Run `nginx -t`; verify `/web`, `/app` (if enabled), docs, REST, and SSE through
  the public origin.
- Probe `/api/v1/monitor/health/check` and `/api/v1/monitor/health/stream`;
  simulate DB and Redis failure and assert status reporting.
- Assert `backend/dist` and `docker/nginx/web/dist` match `frontend/web/dist`
  after sync (e.g. compare hashed chunk names) and that a feature string
  (e.g. `ai/chat/ws`) is present in the built artifact.
- Assert a single backend process or add distributed tests before increasing
  replicas.

## 7. Wrong vs Correct

Wrong (assuming a removed host mount is the only source provider):

```yaml
# The image already contains source; a missing ../backend:/home mount is not an error.
backend:
  volumes:
    - ./backend/static/upload:/home/static/upload
```

Correct current-state decision:

```text
Keep the source COPY in docker/backend/Dockerfile. The ../backend:/home mount is
commented out and optional (dev hot reload only); production runs image-bundled
source. After any frontend change, rebuild and rsync --delete all three web dist
targets before re-verifying the deployed artifact.
```