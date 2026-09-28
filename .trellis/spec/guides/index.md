# FastApiAdmin Thinking Guides

Thinking guides are short pre-change checklists: they ask **what must be
considered** before implementation. They do not define payloads, paths, or
commands. Concrete implementation rules belong in the backend, frontend,
architecture, operations, security, and testing specs.

| Guide | Use it when |
|---|---|
| [Code Reuse Thinking Guide](./code-reuse-thinking-guide.md) | Adding a helper, module, API wrapper, template output, or configuration value |
| [Cross-Layer Thinking Guide](./cross-layer-thinking-guide.md) | A change crosses the backend, Web, App, docs, generated code, or deployment boundary |

## FastApiAdmin triggers

- A backend schema, response envelope, pagination field, or permission string
  changes.
- A REST or WebSocket endpoint is consumed by both `frontend/web/` and
  `frontend/app/`.
- A `module_*` feature, menu record, generated page, or plugin route is added.
- A route pipeline, layout route outlet, `KeepAlive` cache, or worktab store
  change is made, or a page's `defineOptions({ name })` / menu `route_name` moves
  (see `../frontend/routing-and-caching.md`).
- Code is generated or a module is added/modified — the backend must be
  restarted because dynamic discovery runs only at startup
  (see `../architecture/code-generation-and-routing.md`).
- A Jinja template, output path, environment key, proxy path, startup step, or
  migration changes.
- Authentication, permission/data-scope, password/secret, upload, rendered
  content, or audit-log behavior changes.
- Frontend source changes but the deployment artifacts (`frontend/web/dist`,
  `docker/nginx/web/dist`, `backend/dist`, App H5) must be rebuilt and synced
  (see `../operations/deployment-topology.md` and
  `../testing/contract-and-deployment-testing.md`).
- Documentation claims a feature is optional or deployable in a way the
  runtime must enforce.

Use the checklists to find the affected contract, then follow:

- `../backend/index.md`
- `../frontend/index.md`
- `../architecture/system-boundaries.md`
- `../architecture/http-and-realtime-contracts.md`
- `../architecture/code-generation-and-routing.md`
- `../frontend/routing-and-caching.md`
- `../operations/configuration-and-startup.md`
- `../operations/deployment-topology.md`
- `../security/index.md`
- `../testing/index.md`
- `../../../SKILL.md` (repo-root fastapiadmin-dev skill)
