# Frontend Testing

## 1. Admin Web

`frontend/web/` uses Vitest 4, jsdom, `@vitejs/plugin-vue`, and Vue Test Utils.
`vitest.config.ts` includes `src/**/*.{test,spec}.{ts,js}` and maps `@`,
`@utils`, and `@stores`. Run from `frontend/web/`:

```bash
pnpm test
pnpm type-check
pnpm build
```

The only current test file is `src/__tests__/smoke.spec.ts`; it dynamically
imports `MenuTypeEnum` and code-generator enum modules. It proves those runtime
modules load and selected enum values match, not component, router, store, HTTP,
auth, accessibility, or user-flow coverage.

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
  and permission-driven visibility. UI hiding is not backend authorization.

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
| Realtime | Browser WebSocket behavior | uni-app socket API/platform behavior |

Type-check success is necessary but does not prove runtime envelope, storage,
retry, upload, or realtime behavior.

