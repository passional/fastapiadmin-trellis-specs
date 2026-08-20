# Frontend Development Guidelines

This repository has three frontend surfaces with separate dependency trees and
commands:

- `frontend/web/`: Vue 3 admin SPA, Element Plus, Axios, Pinia, Tailwind/SCSS.
- `frontend/app/`: Vue 3 uni-app client, Wot UI, Alova, Pinia, UnoCSS.
- `frontend/docs/`: VitePress bilingual documentation site.

Do not transfer a Web-only library or convention to the App, or vice versa.
Source, package scripts, TypeScript/ESLint configs, and build configs are
authoritative; parts of `frontend/docs/src/guide/` describe older layouts.

## Guides

| Guide | Use it for |
|---|---|
| [Directory Structure](./directory-structure.md) | Web, App, Docs, and feature placement |
| [Component Guidelines](./component-guidelines.md) | Vue SFC contracts, UI systems, styling, and platform behavior |
| [Hook Guidelines](./hook-guidelines.md) | Web hooks, App composables, lifecycle, and async ownership |
| [State Management](./state-management.md) | Pinia, persistence, local state, and server state |
| [Type Safety](./type-safety.md) | Strict TypeScript, API types, globals, and generated declarations |
| [Quality Guidelines](./quality-guidelines.md) | Package-specific lint, type, test, build, and review checks |

For a cross-client feature, read the full guide set and verify the Web and App
implementations independently.
