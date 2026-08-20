# Frontend State Management

## State ownership order

Choose the narrowest owner:

1. Plain constants and derived values in the feature module.
2. `ref`/`reactive`/`computed` inside a component for view-only state.
3. A composable for reusable stateful behavior or server-list orchestration.
4. URL/route state for navigation, shareable filters, and record identity.
5. Pinia only for application-wide state used across routes or layouts.

Do not put dialog visibility, a single page's form, or ordinary list rows in a
global store.

## Admin Web

`frontend/web/src/store/index.ts` creates the only Pinia instance, installs
`pinia-plugin-persistedstate`, and exports stores. Stores are feature-oriented
files in `src/store/modules/` and predominantly use Pinia setup-store syntax
with `ref`, `computed`, and actions. Use the shared `store` instance when a
store is accessed outside component setup.

Persistence is opt-in in each store. For example, settings declare a stable
`persist` key and `localStorage`; do not persist transient loading, errors,
socket objects, DOM handles, or request promises. Authentication tokens are
owned by `src/utils/auth`, while `useUserStore` keeps the reactive login/user,
permissions, route, and lock state synchronized. Logout/reset must clear
dependent menu, work-tab, router, chat, and credential state as the current
user store does.

Use `useTable` for page server state. `refreshAppCaches()` is the coordinated
path for refreshing user/config/notice/dictionary/route caches after changes;
do not scatter competing application bootstrap refreshes.

## Uni-app client

`frontend/app/src/main.ts` creates the App's independent Pinia instance. Stores
in `src/store/` currently use option-store syntax. The custom plugin in
`src/store/persist.ts` restores state by `$patch` and persists every store by ID
except `temp` and `theme`.

Because persistence is broad by default, keep module-scoped in-flight flags and
timestamps outside state, as `src/store/configStore.ts` does. Never store
non-serializable `Method`, Promise, socket, component, or platform handle values
in Pinia. `src/store/userStore.ts:isLoggingIn` is a current legacy exception: it
is transient but is still included in the broadly persisted `appUserInfo`
state, so do not use it as the model for new request flags. Tokens use the typed
`Storage` wrapper and constants; user profile is the persisted `appUserInfo`
state. `clearAll()` is the credential/profile reset boundary.

Use `useListPage`/Alova for paginated server data and local refs for page
filters/forms. Do not mirror every API response into Pinia. Promote data only
when multiple pages need a shared durable cache (system config and ticket
statistics are current examples), and document invalidation/refresh behavior.

## Derived and async state

- Use `computed`/getters for derived data and never mutate state from a getter.
- Keep actions as the owner of multi-field transitions and remote side effects.
- Expose loading flags that prevent duplicate submissions; always reset them in
  `finally`.
- Treat persisted state as a startup cache, then refresh from the server when
  correctness requires it.
- When authentication expires, let the HTTP/auth boundary clear state and
  navigate once; both clients de-duplicate concurrent 401 refresh flows.

## Avoid

- importing the Web store or storage code into the App;
- a second Pinia instance or persistence plugin;
- direct `$state` replacement (the App persistence plugin intentionally uses
  `$patch`);
- persisted secrets beyond the existing token storage contract;
- keeping stale route/menu/permission state after logout;
- duplicate global toasts/errors from both store actions and the HTTP layer.
