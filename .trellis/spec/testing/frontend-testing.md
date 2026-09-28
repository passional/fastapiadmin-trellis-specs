# Frontend Testing

## 1. Admin Web

`frontend/web/` uses Vitest 4, jsdom, `@vitejs/plugin-vue`, and Vue Test Utils.
`vitest.config.ts` includes `src/**/*.{test,spec}.{ts,js}` and maps the full
alias set used by `vite.config.ts`: `@`, `@views`, `@imgs`, `@icons`, `@utils`,
`@stores`, `@plugins`, `@styles`, `@api`, `@fa_imgs`. Run from `frontend/web/`:

```bash
pnpm test        # vitest run
pnpm ts:check    # vue-tsc --noEmit --skipLibCheck (canonical type gate)
pnpm lint
pnpm build:prod
```

`pnpm type-check` (`vue-tsc --noEmit`) also exists but omits `--skipLibCheck`;
`ts:check` is the SKILL-prescribed canonical command. Use it when reporting a
type gate.

Two test files currently exist:

- `src/__tests__/smoke.spec.ts` (small) dynamically imports `MenuTypeEnum` and
  the code-generator enum modules. It proves those runtime modules load and
  selected enum values match — nothing more.
- `src/__tests__/route-invariants.spec.ts` (added in the Sept router refactor)
  is the canonical executable contract for the routing/KeepAlive invariants
  described in `frontend/routing-and-caching.md`. It asserts:
  - global single-layer `KeepAlive`: static directory routes with children must
    keep `component: undefined`, so Vue Router's depth-skip renders the leaf;
  - the multi-level `Dashboard` / `Fastlink` directories stay component-less
    (regression anchor for the removed `NestedRouterParent` shell);
  - every leaf record is renderable (`component` / `link` / `iframe`);
  - `backend/sql/sys_menu.json` directory nodes (`type=CATALOG`) and any node
    with non-button children must not carry `component_path`.

Neither file covers component, store, HTTP, auth, accessibility, or user-flow
behavior. The router invariants are the exception: that area **is** covered.

Use Vitest according to the owner:

- Pure serializer/normalizer/menu/protocol helpers: table-driven unit tests
  with exact inputs and outputs.
- Pinia store: fresh active Pinia per test, mocked API/storage/router, assert
  state and persisted side effects; clear local/session storage between cases.
- HTTP/auth: mock Axios transport; assert `Authorization`, no-auth behavior,
  five-field envelope handling, concurrent 401 single-refresh queue, replay,
  refresh failure cleanup, and one user-visible error.
- Component: Vue Test Utils mount with explicit plugins/stubs; assert emitted
  events, loading/empty/error/permission variants, cleanup, and accessible
  interaction rather than internal refs.
- Router/menu: assert exact backend component/menu path mapping, unknown routes,
  directory-no-component, and permission-driven visibility. UI hiding is not
  backend authorization.

Do not snapshot entire generated pages or large Element Plus DOM trees. Assert
stable contracts and user-visible behavior. Reset fake timers, mocks, global
listeners, storage, and Pinia/router state after each test.

## 2. Uni-app App

`frontend/app/package.json` currently has lint, type-check, and platform build
scripts but no `test` script, Vitest/Jest dependency, or checked-in test files.
`@dcloudio/uni-automator` is installed, but no automated test command/config is
defined. Do not run or document `pnpm test` for this package and do not claim
App automated coverage.

Current executable gates from `frontend/app/` are:

```bash
pnpm lint
pnpm type-check
pnpm build:h5
```

Build the actual changed platform too, for example `pnpm build:mp-weixin`,
when the change uses platform conditionals, storage, login, upload, WebSocket,
manifest permissions, or uni APIs. Record manual verification for H5/device or
mini-program behavior until a runner is deliberately added.

App verification must respect its distinct contracts:

- Alova returns business `data` directly; do not copy Web `.data.data` tests.
- The 401 refresh queue, `ignoreAuth`/`authRole`, uni storage cleanup, and
  `reLaunch` to login require platform API mocks or manual failure-path checks.
- Upload returns the raw uni upload response and callers parse its string
  `data`; ordinary Alova response assertions do not cover it.
- WebSocket clients need connect/open/message/error/close and reconnect cleanup
  checks on each supported platform.

If automated App tests are introduced, add an explicit package script and
config in the same change, start with extracted transport/store/composable
logic, and document which uni globals are simulated versus exercised by
uni-automator on a real target.

## 3. Cross-client cases

For a shared backend change, keep Web and App expectations separate:

| Contract | Web assertion | App assertion |
|---|---|---|
| Success data | Axios response consumer reads `.data.data` | Alova consumer receives business `data` |
| Pagination | `items`, `total`, `page_no`, `page_size`, `has_next` | Current ambient `list` mismatch must be adapted/fixed, not copied silently |
| Access auth | Axios injects Bearer token | Alova injects Bearer token unless `ignoreAuth` |
| Refresh body | JSON string in current Web/backend contract | Current App object body is a known failing mismatch |
| Realtime | Browser WebSocket / `EventSource` (SSE) behavior | uni-app socket API/platform behavior |

Type-check success is necessary but does not prove runtime envelope, storage,
retry, upload, or realtime behavior.

## 4. Built-artifact (dist) verification

The deployment chain serves built artifacts, not `src/`. There are four dist
output locations (all gitignored, so a fresh checkout has none of them), and the
same build must be synced between the web-serving copies:

| Artifact | Produced by | Served / consumed |
|---|---|---|
| `frontend/web/dist` | `pnpm build:prod` | canonical build output |
| `docker/nginx/web/dist` | `rsync -a --delete` from `frontend/web/dist` | Nginx `/web` alias `/usr/share/nginx/html/web/dist` |
| `backend/dist` | `rsync -a --delete` from `frontend/web/dist` | `path_conf.FRONTEND_DIST_DIR`, mounted by `register_frontend` (`app/__init__.py`) |
| `docker/nginx/app/dist/build/h5` | `pnpm build:h5` in `frontend/app` | Nginx `/app` alias |

Procedure after a frontend change that ships to deployment:

1. rebuild (`pnpm build:prod`),
2. `rsync -a --delete` the output into `docker/nginx/web/dist` and
   `backend/dist`,
3. verify at the **artifact** level, not just `src/`.

Minification drops function names and comments, but property accesses and
message strings survive, so grep the bundle for feature strings and invariants:

```bash
# locate the chunk that owns a feature
grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js
# confirm the WebSocket guard uses !== WebSocket.CLOSED (not only === OPEN)
grep -oE '.{0,60}readyState.{0,60}' docker/nginx/web/dist/js/<chunk> | grep -i websocket
```

Rule: **"source fixed" ≠ "deployment fixed"**. When feedback says a fix is
"still broken in production", grep the dist artifacts for the feature string
first, then go to source. Docker deployments additionally need the image
rebuilt and re-transferred — the running container serves the baked-in copy.

This is the verification contract; the surrounding topology and the
`register_frontend` same-path pitfall are documented in
`operations/deployment-topology.md`.