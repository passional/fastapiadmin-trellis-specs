# Frontend Quality Guidelines

## Package-specific checks

Run commands from the package being changed.

Admin Web (`frontend/web/`):

```bash
pnpm type-check
pnpm test
pnpm build
```

`pnpm lint` runs ESLint with `--fix`, Prettier with `--write`, and Stylelint with
`--fix`; it mutates files. Use it when formatting/lint fixes are in scope and
review the diff afterward. The active configs are `eslint.config.mjs`,
`.prettierrc.yaml`, `.stylelintrc.cjs`, and `tsconfig.json`.

Uni-app client (`frontend/app/`):

```bash
pnpm lint
pnpm type-check
pnpm build:h5
```

Also build the changed deployment target (for example `pnpm build:mp-weixin`)
when platform conditionals, manifests, routing, uploads, storage, or platform
APIs change. Active conventions come from `eslint.config.mjs`,
`tsconfig.base.json`, `tsconfig.json`, `uno.config.ts`, and `vite.config.ts`.

Documentation (`frontend/docs/`):

```bash
pnpm build
```

Check both Chinese and English route/sidebar links when changing localized
docs. Do not report a root-level frontend check as validation for all packages.

## Repository contribution workflow

`CONTRIBUTING.md` asks contributors to discuss large changes in an issue first,
use a `feature/xxx` or `bugfix/xxx` branch, write conventional commit messages,
and target pull requests at `dev`. Web and App also carry package-local
Commitlint configs; Web's `commitlint.config.cjs` lists the accepted types,
while App extends `@commitlint/config-conventional`.

## Tests and evidence

Web uses Vitest + jsdom and Vue Test Utils; tests match
`src/**/*.{test,spec}.{ts,js}`. The existing
`frontend/web/src/__tests__/smoke.spec.ts` verifies importable runtime enums but
is intentionally small. Add focused tests for pure transformations, routing,
stores, composables, HTTP retry/error behavior, or components when changing
those contracts.

App currently has no package test script. Type-check, lint, target builds, and
manual platform verification are the available gate; do not claim automated
App test coverage. Extract complex pure logic so it can be tested if a runner is
introduced, and exercise loading/empty/error/auth-expiry behavior manually.

## Review checklist

- The change is in the correct Web/App/Docs package and uses its established UI,
  HTTP, router, store, and styling stack.
- API paths, methods, params/body placement, and types match the backend.
- Server errors are displayed once; loading and refresh indicators reset in
  `finally` or their owning hook.
- Components have typed contracts and clean up global resources.
- Global state is justified and persistence excludes transient/non-serializable
  data.
- Web desktop/mobile layouts or App H5/target platform remain usable, including
  keyboard/touch, safe areas, empty states, and theme modes.
- No generated declaration, vendored `uni_modules`, lockfile, or broad config was
  changed accidentally.

## Formatting realities

Web formats with two spaces, double quotes, semicolons, 100-column Prettier
width, and trailing commas where supported. App's Uni Helper ESLint style uses
the existing no-semicolon/single-quote form seen in source. Do not reformat one
package to resemble the other.

## Avoid

- relying on an outdated example in `frontend/docs/src/guide/` when the current
  source/config differs;
- disabling strict TypeScript or lint rules for a feature-level problem;
- browser-only assumptions in shared uni-app code;
- duplicate HTTP clients, design systems, route bootstraps, or persistence
  layers;
- reporting success without reviewing auto-fix/build-generated changes and the
  final git diff.
