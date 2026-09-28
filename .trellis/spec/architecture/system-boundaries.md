# System Boundaries and Module Loading

## 1. Scope / Trigger

Use this contract when adding or moving a module, route, page, menu, plugin,
documentation section, or shared behavior between applications.

FastApiAdmin is one repository containing four independently built runtimes:

| Runtime | Ownership | Build/runtime evidence |
|---|---|---|
| Backend | `backend/` FastAPI, SQLAlchemy, Redis, scheduler, code generator | `backend/main.py`, `backend/app/__init__.py`, `backend/app/api/v1/routers.py`, `backend/pyproject.toml` |
| Admin Web | `frontend/web/` Vue/Vite/Axios/Pinia | `frontend/web/package.json`, `frontend/web/vite.config.ts`, `frontend/web/src/main.ts` |
| App | `frontend/app/` uni-app/Alova/Pinia | `frontend/app/package.json`, `frontend/app/vite.config.ts`, `frontend/app/src/main.ts` |
| Docs | `frontend/docs/` VitePress | `frontend/docs/package.json`, `frontend/docs/.vitepress/config.mts`, `frontend/docs/src/` |

There is no root workspace that builds or type-checks all four. Do not import
source across frontend packages; share behavior through explicit transport
contracts and mirrored types.

## 2. Signatures and registration points

First-party backend routes are assembled in exactly one place: the
`DOMAIN_CONTROLLERS` table in `backend/app/api/v1/routers.py` ("唯一路由事实来源").
Each domain key is a public prefix and each value is the list of controllers
included below it:

```python
# backend/app/api/v1/routers.py
DOMAIN_CONTROLLERS: dict[str, list[APIRouter]] = {
    "/system": [AuthRouter, UserRouter, RoleRouter, MenuRouter, DeptRouter, ...],
    "/monitor": [CacheRouter, HealthRouter, OnlineRouter, ServerRouter],
    "/task": [CronJobRouter, StorageBrowseRouter, StorageTransferRouter, ...],
    "/ai": [ChatRouter],
    "/generator": [GenRouter],
    "/common": [FileRouter],
}
```

`backend/app/__init__.py:register_routers` imports `api_v1` from that table and
also calls `dynamic_router.init_app(app)`. A new first-party feature therefore
needs its own feature package and one new entry in `DOMAIN_CONTROLLERS`; there
is no per-area `__init__.py` aggregation step.

Only extensions below `backend/app/plugin/module_*/**/controller.py` use
dynamic discovery. `backend/app/core/discover.py` imports every top-level
`APIRouter` in each controller and maps `module_example` to `/example`.
`backend/app/plugin/module_example/demo/controller.py` is the executable
example.

`plugin.toml` is metadata only; neither first-party registration nor dynamic
discovery reads it at runtime. It lives at the **area/package** level and is
present for first-party areas too (`backend/app/modules/system/plugin.toml`,
`backend/app/modules/{ai,common,generator,monitor,task}/plugin.toml`), not below
each feature directory.

## 3. Contracts

### Backend feature ownership

- First-party vertical slices live at
  `backend/app/modules/<area>/<feature>/` with `controller.py`, `service.py`,
  `crud.py`, `model.py`, and `schema.py`. Canonical slice:
  `backend/app/modules/system/dept/`.
- Dynamically discovered extensions live at
  `backend/app/plugin/module_<package>/<feature>/` and require importable
  directory names plus `controller.py`.
- Framework behavior belongs in `backend/app/core/`; shared protocol objects in
  `backend/app/common/`; feature rules remain in the feature service/CRUD.

Evidence: `backend/app/modules/system/dept/`,
`backend/app/plugin/module_example/demo/`, and
`backend/app/core/discover.py`.

### Discovery is fail-fast and runs only at startup

`DynamicRouterRegistry._build()` scans `backend/app/plugin/module_*` once and
caches the result. It raises `RuntimeError` and **aborts application startup**
when any plugin controller cannot be imported, rather than logging and skipping
it; `check_route_conflicts()` likewise halts startup when the same
method+path is registered twice (parameter names are normalized before
comparison). Duplicate `APIRouter` objects inside a single controller are
de-duplicated by object identity during the scan.

Because discovery only runs during startup, after generating or moving a module
you **must restart the backend**. Uvicorn dev reload does not re-run discovery;
new modules silently 404 until the process restarts.

### Web menu-to-component boundary

Backend menu records carry `route_path`, `route_name`, `component_path`, and
permission fields (`backend/app/modules/system/menu/schema.py` and `model.py`).
Web converts them in `frontend/web/src/router/MenuProcessor.ts`.
`component_path="module_system/user/index"` must resolve to a real file below
`frontend/web/src/views/` through the glob in
`frontend/web/src/router/route-loader.ts`.

Changing only the database menu record or only the view path is incomplete.
The route loader shows an error component for a missing page in production; it
does not repair the contract.

### Route cache invariants (cross-layer)

The Admin Web has a single page-level `KeepAlive`, mounted exactly once in
`frontend/web/src/layouts/fa-page-content/index.vue`. Architecture-level rules
that other layers depend on:

- The cache key is the leaf route `path` (`routeViewCacheKey`).
- Directory routes must keep `component: undefined`
  (`MenuProcessor.mapMenuNode`; `RouteTransformer.handleNormalRoute`). A
  directory with a component renders as a shell and breaks deep-skip rendering.
- `include` is derived from the worktab `opened` list; an empty `opened` means
  `include` is undefined, so nothing is pruned.
- Eviction is expressed through `keepAliveExclude` in
  `frontend/web/src/store/modules/worktab.store.ts` (`clearAll()` pushes closed
  component names, `openTab()` calls `removeKeepAliveExclude`).
- Never add `:max` to the `KeepAlive`; LRU eviction would drop still-open tabs.
- `defineOptions({ name })` must equal the backend menu `route_name`, because
  `include`/`exclude` match on component name.

The full ownership, pitfalls, and debugging procedure for these invariants live
in the frontend layer spec `frontend/routing-and-caching.md`; the executable
contract is `frontend/web/src/__tests__/route-invariants.spec.ts`.

### Documentation boundary

Chinese and English files are separate source trees. Navigation is explicit in
`frontend/docs/.vitepress/config.mts`; adding or moving a guide requires the
matching source and sidebar/nav update. `frontend/docs/src/guide/` and
`frontend/docs/src/en/guide/` are the reference pair.

### Current module-optionality mismatch

`README.md` and `README.en.md` describe an `ENABLED_MODULES` switch, but no such
setting or runtime check exists. `backend/app/__init__.py:register_routers`
always imports the full `DOMAIN_CONTROLLERS` table and always initializes the
scheduler. The `optional` flag in `plugin.toml` does not disable code.

Treat all current first-party modules as loaded. Do not promise module
trimming, conditional menus, or conditional startup until one executable gate
controls router imports, initialization, and clients and has tests.

## 4. Validation & Error Matrix

| Condition | Current behavior | Required handling for changes |
|---|---|---|
| First-party controller omitted from `DOMAIN_CONTROLLERS` | Endpoint is absent/404 | Assert the final app route exists |
| Plugin path is not importable or controller raises | Discovery raises `RuntimeError` and aborts startup | Fix path/import and test discovery; do not rely on `plugin.toml` |
| Duplicate method+path across routers | `check_route_conflicts()` raises `RuntimeError` and aborts startup | Export one intended router per path; do not shadow by registration order |
| Duplicate `APIRouter` object in one controller | De-duplicated by object identity | Export one intended router per controller |
| Menu `component_path` has no matching Vue file | Web renders a route warning component | Verify the exact glob key and route registration |
| Directory route carries a component | Deep-skip rendering fails; `warnInvalidRouteConfig` logs `目录节点不应挂组件` | Keep `component: undefined` for directories |
| Docs nav points to a missing locale file | Build/link may be broken | Build docs and verify both locale paths |
| `ENABLED_MODULES` is edited only in docs | No runtime effect | Reject the claim or implement a complete runtime gate |

## 5. Good / Base / Bad Cases

- Good: a new dynamic feature adds an importable plugin controller, matching
  Web API/view/menu fields, permissions, and discovery tests.
- Base: a backend-only first-party endpoint adds a controller to
  `DOMAIN_CONTROLLERS` and has no frontend page.
- Bad: adding `optional = true` to `plugin.toml` and assuming the route or
  scheduler is disabled; or generating code and expecting the route to appear
  without restarting the backend.

## 6. Tests Required

- Backend: construct `create_app()` and assert the intended route/prefix; for a
  plugin, exercise `DynamicRouterRegistry._build()` with the controller, and
  assert startup fails on an unimportable plugin.
- Web: type-check, then verify `ComponentLoader` resolves the exact
  `component_path` and dynamic route registration does not show the error
  component; `route-invariants.spec.ts` is the existing executable contract for
  directory/KeepAlive shape.
- Docs: run the VitePress build and verify every changed locale link exists.
- A future enable/disable feature must assert absent REST and WebSocket routes,
  skipped startup work, hidden menus, and disabled client entry points.

Current backend tests are `backend/tests/conftest.py`, `test_main.py`, and
`test_migrations.py`. `assert_route` is defined in `conftest.py` but has no call
sites; add stronger assertions for new module-loading semantics.

## 7. Wrong vs Correct

Wrong:

```python
# Adding metadata does not register or disable a runtime module.
# plugin.toml: optional = true
```

Correct:

```python
# First-party route: add the controller to DOMAIN_CONTROLLERS.
DOMAIN_CONTROLLERS["/system"].append(FeatureRouter)
# Dynamic plugin: place an APIRouter in app/plugin/module_xxx/**/controller.py,
# then restart the backend so discovery re-runs.
```