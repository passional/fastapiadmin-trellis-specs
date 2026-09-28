# Frontend Type Safety

## Compiler contracts

Both applications use TypeScript strict mode. Web additionally enables
`noUncheckedIndexedAccess`, `noImplicitOverride`, isolated modules, and no emit
in `frontend/web/tsconfig.json`. App extends strict Vue/uni-app settings in
`frontend/app/tsconfig.base.json` and supplies uni platform/Wot types in
`tsconfig.json`. Use the configured `@/` and package-specific aliases rather
than fragile relative paths across feature roots.

Run each package's type check after changes. For Web, `pnpm ts:check`
(`vue-tsc --noEmit --skipLibCheck`) is the canonical command; `pnpm type-check`
runs without `--skipLibCheck` and is stricter. Do not weaken strictness or add a
broad ambient declaration to hide a local error.

## API and feature types

API modules co-locate request, response, and record interfaces with the endpoint
object. `frontend/web/src/api/module_system/user.ts` and
`frontend/app/src/api/module_system/user.ts` are representative. Keep backend
snake_case field names in transport types; map to a UI-only shape only when the
component benefits from it.

Shared Web router/component/store types live below `frontend/web/src/types/` and
are exported by the relevant index. Project-wide response, pagination, audit,
and base-record types are ambient in `frontend/web/src/types/global.d.ts`.
App has its independent ambient equivalents in
`frontend/app/src/types/global.d.ts`. Update each client deliberately when a
shared backend contract changes; do not assume the two ambient files are linked.

The current pagination types demonstrate why: backend `PageResultSchema` and
Web use `items`, while App's ambient `PageResult` and `useListPage` use `list`,
without an adapter that renames the field. Treat this as an existing contract
gap to resolve in any affected feature, not as a convention to reproduce.

Keep a type local to a component/composable when it is not a cross-feature
contract. Export reusable component configuration types from the component or a
nearby `types.ts`, as `FaForm` and modal components do. Use `import type` for
type-only dependencies.

## Vue contracts and inference

Type props, emitted payloads, models, slots, template refs, API records, and
configuration arrays. Preserve literal unions with `as const` where useful.
Prefer inference from typed source functions: Web `useTable` infers record and
params from its `apiFn`; App `useListPage<T>` takes an explicit record type.
Use `unknown` plus narrowing at untrusted/error boundaries.

The Web ESLint config currently permits explicit `any`, and legacy/generic UI
code uses it. This is compatibility, not a recommendation. For new business
code use concrete interfaces, generics, `Record<string, unknown>`, or
`unknown`; keep `any` limited to framework escape hatches that cannot be typed
reasonably. Avoid non-null assertions such as `row.id!` unless a preceding
contract guarantees the value.

## Generated and runtime boundaries

Do not hand-edit App `auto-imports.d.ts`, `components.d.ts`, or
`uni-pages.d.ts`; their generators are configured in
`frontend/app/vite.config.ts`. Likewise, respect Web auto-import/component
generation instead of declaring duplicate globals.

Neither frontend uses a general Zod/Yup-style runtime schema layer. Compile-time
interfaces do not validate network data. The HTTP clients validate the common
HTTP status and business `code` fields to decide success, but they do not
runtime-schema-validate the rest of the envelope or payload. Use explicit
guards when consuming optional, third-party, persisted, decoded JSON, upload,
or WebSocket data. Examples include defensive array checks in stores and
response parsing in both HTTP adapters.

## Avoid

- redefining `ApiResponse`, `PageResult`, or base audit types inside a page;
- casting an entire response to the desired type without checking a genuinely
  untrusted shape;
- editing generated declaration files;
- importing runtime values when only a type is required;
- broad index signatures that erase known fields;
- changing transport field casing independently from the backend schema.
