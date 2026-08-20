# System Boundaries and Module Loading

## 1. Scope / Trigger

Use this contract when adding or moving a module, route, page, menu, plugin,
documentation section, or shared behavior between applications.

FastApiAdmin is one repository containing four independently built runtimes:

| Runtime | Ownership | Build/runtime evidence |
|---|---|---|
| Backend | `backend/` FastAPI, SQLAlchemy, Redis, scheduler, code generator | `backend/main.py`, `backend/app/init_app.py`, `backend/pyproject.toml` |
| Admin Web | `frontend/web/` Vue/Vite/Axios/Pinia | `frontend/web/package.json`, `frontend/web/vite.config.ts`, `frontend/web/src/main.ts` |
| App | `frontend/app/` uni-app/Alova/Pinia | `frontend/app/package.json`, `frontend/app/vite.config.ts`, `frontend/app/src/main.ts` |
| Docs | `frontend/docs/` VitePress | `frontend/docs/package.json`, `frontend/docs/.vitepress/config.mts`, `frontend/docs/src/` |

There is no root workspace that builds or type-checks all four. Do not import
source across frontend packages; share behavior through explicit transport
contracts and mirrored types.

## 2. Signatures and registration points

First-party backend routes are assembled explicitly:

```python
# backend/app/api/v1/module_system/__init__.py
system_router = APIRouter(prefix="/system")
system_router.include_router(UserRouter)
```

`backend/app/init_app.py:register_routers` imports and includes every current
first-party area router: common, monitor, system, AI, generator, task, and
storage. A new first-party area therefore needs both its area `__init__.py` and
the app-level registration point.

Only extensions below `backend/app/plugin/module_*/**/controller.py` use
dynamic discovery. `backend/app/core/discover.py` imports every top-level
`APIRouter` in each controller and maps `module_example` to `/example`.
`backend/app/plugin/module_example/demo/controller.py` is the executable
example. `plugin.toml` is metadata only; neither first-party registration nor
dynamic discovery reads it.

## 3. Contracts

### Backend feature ownership

- First-party vertical slices live at
  `backend/app/api/v1/module_<area>/<feature>/` and are included explicitly.
- Dynamically discovered extensions live at
  `backend/app/plugin/module_<package>/<feature>/` and require importable
  directory names plus `controller.py`.
- Framework behavior belongs in `backend/app/core/`; shared protocol objects in
  `backend/app/common/`; feature rules remain in the feature service/CRUD.

Evidence: `backend/app/api/v1/module_system/dept/`,
`backend/app/plugin/module_example/demo/`, and
`backend/app/core/discover.py`.

### Web menu-to-component boundary

Backend menu records carry `route_path`, `route_name`, `component_path`, and
permission fields (`backend/app/api/v1/module_system/menu/schema.py` and
`model.py`). Web converts them in `frontend/web/src/router/MenuProcessor.ts`.
`component_path="module_system/user/index"` must resolve to a real file below
`frontend/web/src/views/` through the glob in
`frontend/web/src/router/route-loader.ts`.

Changing only the database menu record or only the view path is incomplete.
The route loader shows an error component for a missing page in production; it
does not repair the contract.

### Documentation boundary

Chinese and English files are separate source trees. Navigation is explicit in
`frontend/docs/.vitepress/config.mts`; adding or moving a guide requires the
matching source and sidebar/nav update. `frontend/docs/src/guide/` and
`frontend/docs/src/en/guide/` are the reference pair.

### Current module-optionality mismatch

`README.md` and `README.en.md` describe an `ENABLED_MODULES` switch, but no such
setting or runtime check exists. `backend/app/init_app.py` imports all current
first-party routers, `backend/app/api/v1/module_system/__init__.py` always
includes chat, and startup always initializes the scheduler. The `optional`
flag in `plugin.toml` does not disable code.

Treat all current first-party modules as loaded. Do not promise module
trimming, conditional menus, or conditional startup until one executable gate
controls router imports, initialization, and clients and has tests.

## 4. Validation & Error Matrix

| Condition | Current behavior | Required handling for changes |
|---|---|---|
| First-party router omitted from an `__init__.py` | Endpoint is absent/404 | Assert the final app route exists |
| Plugin path is not importable or controller raises | Discovery logs and skips it | Fix path/import and test discovery; do not rely on `plugin.toml` |
| Menu `component_path` has no matching Vue file | Web renders a route warning component | Verify the exact glob key and route registration |
| Duplicate dynamic `APIRouter` object | Deduplicated by object identity | Export one intended router per controller |
| Docs nav points to a missing locale file | Build/link may be broken | Build docs and verify both locale paths |
| `ENABLED_MODULES` is edited only in docs | No runtime effect | Reject the claim or implement a complete runtime gate |

## 5. Good / Base / Bad Cases

- Good: a new dynamic feature adds an importable plugin controller, matching
  Web API/view/menu fields, permissions, and discovery tests.
- Base: a backend-only first-party endpoint updates its area router and app
  registration and has no frontend page.
- Bad: adding `optional = true` to `plugin.toml` and assuming the route or
  scheduler is disabled.

## 6. Tests Required

- Backend: construct `create_app()` and assert the intended route/prefix; for a
  plugin, exercise `DynamicRouterRegistry._build()` with the controller.
- Web: type-check, then verify `ComponentLoader` resolves the exact
  `component_path` and dynamic route registration does not show the error
  component.
- Docs: run the VitePress build and verify every changed locale link exists.
- A future enable/disable feature must assert absent REST and WebSocket routes,
  skipped startup work, hidden menus, and disabled client entry points.

Current route tests in `backend/tests/test_api_module_system.py` mostly prove
non-404 behavior; add stronger assertions for new module-loading semantics.

## 7. Wrong vs Correct

Wrong:

```python
# Adding metadata does not register or disable a runtime module.
# plugin.toml: optional = true
```

Correct:

```python
# First-party route: include it through the existing explicit assembly chain.
area_router.include_router(FeatureRouter)
# Dynamic plugin: place an APIRouter in app/plugin/module_xxx/**/controller.py.
```

