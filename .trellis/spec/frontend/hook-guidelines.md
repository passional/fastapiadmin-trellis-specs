# Frontend Hook and Composable Guidelines

## Naming and placement

All shared composition functions use a `useXxx` name and return reactive state
plus explicit actions. Web composables live under
`frontend/web/src/hooks/core/` (or next to the owning layout/feature); App
composables live under `frontend/app/src/composables/`. Keep a composable local
until more than one consumer or a clear framework-level responsibility exists.

The Web barrel `frontend/web/src/hooks/index.ts` exports only selected public
hooks. Import non-exported specialized hooks from their exact path. App local
composables are discovered by the auto-import plugin, but explicit imports are
also used where they improve clarity or avoid collisions (`useShare` is
explicitly excluded from auto-import).

## Web patterns

`frontend/web/src/hooks/core/useTable.ts` is the standard server-list
abstraction. Give it a typed API function and let it infer params/record types;
use its pagination, loading, stale-result suppression, request de-duplication,
cache, and KeepAlive behavior rather than duplicating page-level fetch state.
Its `AbortController` is not passed to `apiFn`, so `cancelRequest()` currently
prevents an obsolete response from being committed but does not abort the
underlying Axios request; transport cancellation requires an explicit API
contract change. Configure columns through `useTableColumns` and
dialog/loading behavior through `useCrudDialog`, `useCrudForm`, and
`useLoading` where their existing contract fits.

Do not add a page-level `try/catch` solely to repeat an error already owned by
the Axios or hook layer. `useTable` exposes quiet public fetch functions because
it already commits error state. If a caller needs domain-specific recovery,
use the hook callback and preserve the original error semantics.

Use VueUse for established browser/reactivity concerns (`useWindowSize`,
throttle/timeouts) instead of writing new event machinery. Clean up raw event
listeners, timers, AbortControllers, sockets, and observers in the matching
lifecycle. Hooks used by KeepAlive pages must consider both activated and
deactivated states, as `useTable` does.

For a page kept alive by the route cache, `onUnmounted` fires only when the
instance is evicted — it is **not** the sole cleanup point. Pages that own a
WebSocket, timers, or global listeners must implement `onActivated`/`onDeactivated`:
release the resource on deactivate and restore it on activate, as
`frontend/web/src/views/module_ai/chat/index.vue` does. A cached-but-deactivated
page stays alive, so a missing `onDeactivated` leaks its connection.

When guarding a WebSocket connection, cover the handshake phase:
`if (ws && ws.readyState !== WebSocket.CLOSED) return` (blocking only `OPEN` leaks
re-entries while `CONNECTING`). Before an intentional `close()`, detach
`onopen`/`onmessage`/`onerror`/`onclose` first so close-race callbacks cannot fire
toasts or state updates. See [Routing and Caching](./routing-and-caching.md) for
the full contract.

## App patterns

`frontend/app/src/composables/useListPage.ts` is the standard paginated-list
flow. It wraps Alova `usePagination`, disables stale response caching, exposes
`list/total/loading/error`, and owns pull-refresh cleanup. Pass a fetcher that
returns an Alova `Method`; reset searches with `toFirst()` and drive initial and
platform page events from the page.

There is a current cross-layer mismatch to resolve when touching this flow:
`useListPage` reads `PageResult.list`, while
`backend/app/core/base_schema.py:PageResultSchema` and the Web ambient type use
`items`; the App Alova response adapter does not rename it. Verify the live API
response and normalize or align the contract deliberately rather than copying
the existing `list` assumption into a new endpoint.

Use uni-app lifecycle hooks (`onLoad`, `onShow`, `onPullDownRefresh`,
`onReachBottom`) for page behavior and Vue lifecycle hooks for component-owned
resources. A socket composable must expose and invoke cleanup;
`frontend/app/src/composables/useAiChat.ts` exposes `close()` and distinguishes
user closure from unexpected disconnection.

The HTTP layer already unwraps successful data and globally reports most
errors. Empty catches are acceptable only where a comment identifies that
ownership, as in `frontend/app/src/pages/mine/index.vue`; do not show a second
toast for the same failure. Use `meta.silent` only when the feature intentionally
renders an inline error.

## API of a new composable

- Type input options and the return contract; prefer generics when the source
  function determines the data type.
- Keep mutable state private where possible and expose computed/readonly views.
- Make initial loading behavior explicit (`immediate` or page-owned trigger).
- Prevent duplicate concurrent work when multiple lifecycle events can overlap.
- Define who displays errors and who resets loading/refresh indicators.
- Release global listeners, timers, sockets, and pending requests.

Avoid hidden singleton state unless cross-instance coalescing is the explicit
contract, raw Promise rejections from lifecycle callbacks, and one-off copies of
the existing table/list/auth/theme composables.
