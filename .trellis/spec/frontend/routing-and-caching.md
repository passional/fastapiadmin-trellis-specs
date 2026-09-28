# Frontend Routing and Route Caching

This guide owns the Admin Web route pipeline and the single-layer `KeepAlive`
cache contract. These are load-bearing invariants, not preferences: violating
them causes duplicate mounts, duplicate API requests, leaked WebSocket
connections, and stale side effects across logout/login. The executable contract
is `frontend/web/src/__tests__/route-invariants.spec.ts`; run `pnpm test` in
`frontend/web` after touching routing, the app layout, or the worktab store.

## Route pipeline

Two sources feed one router:

1. **Static shell** — `frontend/web/src/router/routes.ts` declares the root
   `Layout` record (`ROOT_LAYOUT_ROUTE_NAME = "RootLayout"`), login/exception
   pages, and the built-in `Dashboard` / `Fastlink` multi-level directories. The
   root record must carry the Layout component; every record below it that has
   `children` must leave `component` undefined.
2. **Dynamic business routes** — loaded from backend menus and registered at
   runtime:
   - `router/guards.ts` runs `beforeEach`: login check, dynamic-route loading,
     permission check, and the `CatchAll404` fallback that re-resolves the target
     after registration (a `replace` re-navigation — see the race note below).
   - `router/MenuProcessor.ts` converts backend `MenuTable` records into
     `AppRouteRecord`s (`mapMenuNode`), merging backend, frontend-builtin, and
     mixed menu modes.
   - `router/route-loader.ts` holds the dynamic pipeline:
     `ComponentLoader` (menu `component_path` → `import.meta.glob` module),
     `RouteTransformer` (menu → `RouteRecordRaw`), and `RouteRegistry` (batch
     `addRoute`/`removeRoute` under the root Layout).
   - `router/refresh.ts` owns route reset/refresh after login, logout, and menu
     changes.

New pages are not reachable until they are configured in 菜单管理
(`backend/sql/sys_menu.json`): set `route_path`, `route_name`, `component_path`
(relative to `src/views/`, e.g. `system/user/index`), and `keep_alive`. A page
whose menu is missing simply 404s — the file existing under `src/views/` is not
enough.

## Deep-skip rendering (NestedRouterParent removal)

The router relies on Vue Router's **depth-skip**: a `RouterView` skips intermediate
records that have no `component` and renders the first matched descendant that
does — the real leaf page. The single hard precondition is:

> A record that has `children` (a directory/catalog) **must not** have a
> `component`. Directory routes must keep `component: undefined`.

`MenuProcessor.mapMenuNode` enforces this when a menu node is `CATALOG` or has
children, and `RouteTransformer.handleNormalRoute` does the same. Runtime
fallbacks warn in development: `warnInvalidRouteConfig` in `route-loader.ts` and
the `watch` guard in `layouts/fa-page-content/index.vue`.

Historical note (do not reintroduce): directories used to mount a `NestedRouterParent`
shell component (`ROUTE_COMPONENT_NESTED_PARENT`) and `RouteMeta.remountOnFullPath`.
That shell broke depth-skip: the outlet rendered the shell, the leaf was nested
and remounted per navigation, and every menu switch re-fired the page's API
requests. `NestedRouterParent`, `ROUTE_COMPONENT_NESTED_PARENT`, and
`remountOnFullPath` were removed (commit `24589ad9`). Do not reference them as
existing APIs.

## Single-layer route-view cache

- `KeepAlive` exists in **exactly one place**: the route outlet in
  `frontend/web/src/layouts/fa-page-content/index.vue`. `route-invariants.spec.ts`
  asserts this; do not add a second cache layer (including in a nested directory
  shell).
- The cache key is the **leaf route `path`** (`routeViewCacheKey(router)` returns
  `r.path`). Same component under different paths gets separate instances; a query
  change on the same path does **not** remount. Keep the key a path, not a full
  location.
- Matching for `include` / `exclude` is by **component name**, so the outlet
  resolves the rendered leaf component name (`resolveOutletComponentName`) rather
  than using the route name.
- **Never add `:max`** to the `KeepAlive`. The cache set is already expressed
  exactly by `include`/`exclude`; an extra LRU layer evicts still-open tabs,
  causing needless remounts and unpredictable request counts.

## include / exclude semantics

`include` comes from the worktab store's `opened` list
(`frontend/web/src/store/modules/worktab.store.ts`), and only applies in
multi-tab mode:

- If `showWorkTab` is false, `include` is `undefined` and there is no per-tab
  pruning.
- When `opened` is **empty**, the computed `include` is `undefined`. `KeepAlive`
  does **not** prune on an `undefined` include. Never assume "closing tabs clears
  the cache" — closing a tab must go through the exclude path below.
- Otherwise `include` is the set of component names for open tabs whose
  `keepAlive !== false`, plus the current route as a fallback.

Eviction happens only by changing `include`/`exclude`:

- Closing a single tab (`removeTab`) and batch closes (`removeLeft`/`removeRight`/
  `removeOthers`/`removeAll`) mark removed tabs via `markTabsToRemove` →
  `pushExcludeIfLastSiblingOfName`, which pushes the component name into
  `keepAliveExclude`. A name is only excluded once no remaining tab shares it
  (same `route_name` may be open under multiple paths).
- `openTab` calls `removeKeepAliveExclude(name)`, so re-opening a tab restores its
  cache eligibility.
- Logout: `worktab.store.ts clearAll()` keeps fixed tabs, then **replaces**
  `keepAliveExclude` with the component names of the tabs being dropped, forcing
  `KeepAlive` to prune old instances (an `undefined` include would not). The next
  `openTab` removes them again. Do not delete this logic when editing the store —
  without it, cached instances keep their WebSocket connections and timers alive
  across sessions.
- The outlet also adds `meta.keepAlive === false` pages to `exclude` so they are
  never cached.

`keepAliveExclude` is persisted with the worktab store, so the eviction survives a
reload until the next `openTab`.

## defineOptions name == menu route_name

`KeepAlive` matches by component name. A routed page's
`defineOptions({ name: "..." })` **must equal** the backend menu's `route_name`.
If they diverge, the page is neither cached nor excludable and the include/exclude
sets silently miss it. Component-name matching was unified across
`layouts/fa-page-content/index.vue`, `router/routes.ts`, and the view components
in commit `2d393247`; keep new pages on that contract. See
[Component Guidelines](./component-guidelines.md).

## Side-effectful pages and connections

Pages that own a WebSocket, timers, or global listeners must implement
`onActivated`/`onDeactivated`:

- `onDeactivated` releases resources (disconnect, clear timers).
- `onActivated` restores them as needed.
- `onUnmounted` fires only when the cache is evicted, so it is **not** the sole
  cleanup point. A cached-but-deactivated page keeps living: missing
  `onDeactivated` leaks its WebSocket connection after logout.

Connection guards (see `frontend/web/src/views/module_ai/chat/index.vue`):

- The WebSocket re-entry guard must be
  `if (ws && ws.readyState !== WebSocket.CLOSED) return` — cover
  `CONNECTING`/`OPEN`/`CLOSING`, not just `OPEN`. Guarding only `OPEN` leaks
  re-entries during the handshake: the new socket overwrites the old reference
  while the leaked one still connects and shows its own toast (this produced
  duplicate sockets and duplicate "连接成功" toasts after logout → login).
- Before an intentional `close()`, detach `onopen`/`onmessage`/`onerror`/`onclose`
  first, so close-race callbacks cannot fire toasts or state updates after the
  owner has decided to disconnect.

## Debugging duplicate popups / connections / requests

Recurring "N popups", "N connections", or "N API calls" bugs almost always come
from extra component instances or extra lifecycle re-entries. Work the chain:

1. Grep the exact toast/message string to find its unique source. N popups means
   N live instances or N re-entries, not a UI glitch.
2. Check the `KeepAlive` `include`/`exclude` computation and the logout → login
   navigation chain. `guards.ts` re-navigates with `replace` when the original
   navigation was caught by `CatchAll404` (F5 / dynamic-route timing), which adds
   a race window in which a page can mount twice.
3. Check whether a directory route accidentally has a `component` (its shell
   instance keeps the inner page nested and remounts it).
4. Only then check the backend menu data (`backend/sql/sys_menu.json`): confirm
   `route_name` uniqueness and `keep_alive` values. `route-invariants.spec.ts`
   asserts that catalog nodes and nodes with children do not set `component_path`.

## Avoid

- adding a second `KeepAlive`, or a `:max` on the existing one;
- giving a directory/catalog route a `component` (breaks depth-skip);
- assuming an empty `opened` prunes the cache, or clearing `keepAliveExclude`
  manually;
- deleting the `clearAll()` exclude-eviction logic in `worktab.store.ts`;
- naming a page component differently from its menu `route_name`;
- guarding WebSocket re-entry on `OPEN` alone, or closing without detaching
  handlers;
- treating `onUnmounted` as the only cleanup point for a keep-alive page.