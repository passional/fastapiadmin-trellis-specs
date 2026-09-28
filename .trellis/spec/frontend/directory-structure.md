# Frontend Directory Structure

## Separate applications

Each frontend directory is an independent package with its own lockfile and
commands. Run package scripts from that package directory; there is no root
frontend workspace that validates all three.

### Admin Web (`frontend/web`)

```text
src/
├── api/module_*/           # typed Axios endpoint groups by backend area
├── assets/                 # bundled static assets
├── components/             # reusable Fa* components grouped by UI role
├── config/                 # app-level configuration
├── constants/              # shared constant values
├── directives/             # global directives, incl. permission/ (v-hasPerm)
├── enums/                  # shared runtime enums
├── hooks/core/             # reusable Web composables
├── layouts/                # admin shell and layout-local components/hooks
├── locales/langs/          # i18n entries (zh.json / en.json)
├── mock/                   # local mock data
├── plugins/                # single Vue plugin-registration entry
├── router/                 # static shell + dynamic menu → route pipeline
├── store/modules/          # Pinia stores
├── styles/                 # global tokens, Element Plus overrides, layouts
├── types/                  # cross-feature component/router/store types
├── utils/                  # HTTP and other shared helpers
├── views/module_*/         # routed feature pages and page-local components
├── App.vue                 # root component
└── main.ts                 # app bootstrap
```

The `router/` directory is more than a route list: `routes.ts` is the static
shell, while `guards.ts` → `MenuProcessor.ts` → `route-loader.ts` is the dynamic
pipeline (`ComponentLoader` path→component, `RouteTransformer` menu→record,
`RouteRegistry` add/remove routes), with `refresh.ts` and `index.ts` wiring it
together. See [Routing and Caching](./routing-and-caching.md).

Keep backend-aligned names across API and views: for example
`src/api/module_system/user.ts` serves
`src/views/module_system/user/index.vue`. There is one known, existing
inconsistency to record as-is, not to "fix": the storage API layer lives under
`src/api/module_storage/` while its views live under
`src/views/module_task/storage/`. Match whichever side an existing feature
already uses rather than renaming either tree. Put components reused across
features in the category-based `src/components/` tree; put page-only components
under that view's `components/`. Put composables reused across the app in
`src/hooks/core/`; layout-only composables stay beside the layout, as in
`src/layouts/fa-settings-panel/composables/`.

`src/plugins/index.ts` is the single Vue plugin-registration entry. It preserves
the required Store → Router → Directives → error handler → terminal → i18n →
CodeMirror order. Do not call `app.use()` from arbitrary features.

### Uni-app client (`frontend/app`)

```text
src/
├── api/module_*/           # typed Alova endpoint groups
├── components/             # globally reusable mobile components
├── composables/            # auto-imported shared composition functions
├── http/                   # Alova adapter, interceptors, request types
├── layouts/                # default and tabbar page layouts
├── pages/                  # primary package pages
├── subPages/               # lazy-loaded feature subpackage pages
├── router/                 # @wot-ui/router setup/guards
├── store/                  # Pinia stores and custom persistence plugin
├── styles/                 # shared SCSS/theme styles
├── types/                  # ambient and framework declarations
├── utils/                  # cross-platform helpers
├── pages.json              # generated/consumed page manifest
└── manifest.json           # uni-app application manifest
```

Normal top-level tab/login screens live in `pages/`; feature screens live in
`subPages/` to preserve package splitting. Define route metadata with
`definePage()` in the SFC; `pages.config.ts` and the pages plugin generate page
configuration. Use the `@/` alias. Do not put application code in
`src/uni_modules/`; it is vendored plugin code and excluded from TypeScript and
ESLint checks.

`vite.config.ts` auto-imports Vue/VueUse/Pinia/uni-app/Wot/Alova APIs and local
`composables`, `store`, and `utils`, but intentionally does not auto-import
`src/api`; API modules are always explicit imports. Generated
`auto-imports.d.ts`, `components.d.ts`, and `uni-pages.d.ts` are outputs, not
places for hand-written types.

### Documentation (`frontend/docs`)

VitePress configuration and theme code live under `.vitepress/`; Markdown is
under `src/`. Chinese guides live in `src/guide/` and their English mirrors in
`src/en/guide/`. Navigation/sidebar entries are explicit in
`.vitepress/config.mts`, so a new document must update both the file set and the
matching locale navigation where applicable. Shared interactive documentation
components live in `.vitepress/components/` and composables in
`.vitepress/composables/`.

## Naming and anti-patterns

- Vue components and new TS types use PascalCase; composables use `useXxx`;
  normal variables/functions use camelCase; constants use UPPER_SNAKE_CASE.
  Lower-camel legacy interfaces such as `deptTreeType` and
  `searchSelectDataType` remain in both clients' `module_system/user.ts`; do
  not copy that naming into new contracts.
- Reusable Web component directories use semantic category + kebab-case paths
  and `Fa` component names; routed pages usually use `index.vue`.
- App pages/subpages use directory-based `index.vue` and stable route names in
  `definePage`.
- Do not import code across `frontend/web`, `frontend/app`, and
  `frontend/docs`; share API behavior by matching contracts, not filesystem
  imports.
- Do not create a second HTTP client, store bootstrap, global style entry, or
  route registry inside a feature.

## Reference files

- `frontend/web/src/main.ts`
- `frontend/web/src/plugins/index.ts`
- `frontend/web/src/views/module_system/user/index.vue`
- `frontend/app/src/main.ts`
- `frontend/app/vite.config.ts`
- `frontend/app/src/subPages/module_system/tickets/index.vue`
- `frontend/docs/.vitepress/config.mts`
