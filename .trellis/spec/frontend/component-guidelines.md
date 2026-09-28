# Frontend Component Guidelines

## Shared Vue shape

Use Vue 3 Composition API and TypeScript SFCs. Existing components put
`defineOptions`/contracts at the top of `<script setup lang="ts">`, keep derived
state in `computed`, and clean up listeners/connections on unmount. Name routed
or keep-alive components explicitly with `defineOptions({ name: ... })`.

For a routed Web page the name is a contract, not a label: `KeepAlive` and the
worktab cache match by **component name**, so `defineOptions({ name })` **must
equal the backend menu's `route_name`**. If they differ, `include`/`exclude`
silently miss the page and neither caching nor eviction works. See
[Routing and Caching](./routing-and-caching.md).

For reusable contracts:

- Define a local `Props` interface and use
  `withDefaults(defineProps<Props>(), ...)` for defaults.
- Use typed `defineEmits<Emits>()`; tuple payloads document each event.
- Use `defineModel<T>()` for true two-way value contracts and named models,
  rather than manually mutating props.
- Type public slots with `defineSlots` when their payload matters.
- Keep internal refs private; expose only deliberate public operations through
  `defineExpose`.

`frontend/web/src/components/modal/fa-dialog/index.vue` demonstrates typed
props/events, computed `modelValue`, attribute forwarding, keyboard cleanup,
slots, and a minimal exposed ref. `frontend/app/src/components/StatusBadge.vue`
demonstrates a small typed mobile display component.

## Admin Web components

Build on Element Plus and the existing `Fa*` wrappers before introducing a new
primitive. Reuse `FaTable`, `FaForm`, `FaSearchBar`, `FaDialog`/`FaDrawer`, and
their configuration types for CRUD screens. The user page at
`frontend/web/src/views/module_system/user/index.vue` shows the local pattern:
data-driven table/form/description configurations, named slots for exceptional
fields, typed API records, and reusable CRUD/table hooks.

Global reusable components belong under the semantic categories in
`src/components/`. Page-specific pieces stay under the view. Keep component
logic out of table cell/template expressions when it can be a typed helper or
computed configuration.

Styling combines Tailwind utilities, project `fa-*` classes/tokens, SCSS, and
Element Plus overrides. Preserve the global import order documented in
`frontend/web/src/main.ts`. Prefer existing CSS variables and utilities;
component-local SCSS is appropriate for nontrivial visuals. Avoid new raw color
systems, deep selectors without a third-party boundary, and global styles from
an otherwise local component.

## Uni-app components and pages

Use uni-app primitives (`view`, `text`) and Wot UI `wd-*` components so the code
works beyond H5. Use UnoCSS and Wot semantic tokens (`wot-text-*`, `wot-bg-*`)
for theme-aware styling; use `rpx`/safe-area support for mobile geometry. The
tickets page at
`frontend/app/src/subPages/module_system/tickets/index.vue` shows Wot controls,
touch actions, `definePage`, translated labels, loading/empty/end states, and
safe-area spacing.

Put platform differences behind uni-app conditional compilation comments and
test every changed target. `frontend/app/src/components/GlobalDialog.vue` shows
the Alipay-specific branch and component options needed for cross-platform
style isolation. Do not reach directly for browser-only DOM APIs in shared App
code.

Wot/Vue/Pinia/local utilities are auto-imported according to
`frontend/app/vite.config.ts`; API calls remain explicit imports. Prefer the
global toast/dialog/loading/message composables and components over per-page
infrastructure.

## Accessibility and interaction

The repository does not have a dedicated accessibility test suite, so review
interaction manually. Preserve native/Wot/Element controls, visible focus and
keyboard behavior, labels for form controls, readable empty/loading/error
states, and non-color-only status text. `FaDialog` supports Escape and
Ctrl/Cmd+Enter; do not break those behaviors when wrapping it. For touch UI,
maintain adequate targets and safe-area spacing.

## Avoid

- untyped prop bags or string-array emits on new reusable components;
- mutating a prop instead of emitting/updating a model;
- copying a page-only component into the global component tree;
- replacing established Fa/Wot wrappers with a second design system;
- App components that only work in a browser without conditional handling;
- large templates containing server-state, validation, and formatting logic
  that already has a local hook or configuration abstraction.
