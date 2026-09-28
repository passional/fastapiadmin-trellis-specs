# Cross-Layer Thinking Guide

Use this checklist before changing data or behavior that crosses an application
boundary.

## Map the affected surfaces

- [ ] Which backend controller/schema/service owns the input and output?
- [ ] Is the consumer Admin Web, uni-app, VitePress, generated code, Nginx, or
  more than one of them?
- [ ] Does the change affect the public `/api/v1` prefix, the backend's direct
  route, a menu `component_path`, or a generated filesystem path?
- [ ] Does it use ordinary JSON, a file/blob, SSE, or one of the distinct
  WebSocket protocols?
- [ ] Who validates at each boundary, and what HTTP status, business `code`,
  close code, sentinel, or error text reaches the consumer?

## Contract checks

- [ ] For REST, are `code`, `msg`, `data`, `status_code`, and `success` still
  aligned with both frontend clients?
- [ ] For pagination, does the consumer use backend/Web `items`, not the App's
  legacy ambient `list` declaration?
- [ ] For WebSocket, have authentication carrier, inbound messages, outbound
  message kinds, heartbeat, terminal markers, and close behavior all been
  checked?
- [ ] If a control message must interrupt a stream, is the server receiving it
  concurrently rather than waiting for the producer loop to finish?
- [ ] For a menu/page, does `component_path` resolve through the Web
  `import.meta.glob`, and do route and permission prefixes match the backend?
- [ ] For generated code, do preview, ZIP download, local output, menu creation,
  and templates still agree?
- [ ] For an "optional" module, is there an actual runtime enable/disable gate,
  or only documentation/metadata?

## Runtime and rollout checks

- [ ] Are environment keys present before settings are imported?
- [ ] Does startup require database, Redis, seed data, scheduler, or an Alembic
  step?
- [ ] Do Nginx proxy paths, frontend base paths, Docker health probes, and
  backend direct routes agree?
- [ ] Does the feature keep process-local connections or cancellation state
  that makes multiple backend replicas unsafe?
- [ ] Could request/audit logs expose a password, token, API key, or storage
  credential?
- [ ] Do tests assert good/base/bad behavior at every changed boundary,
  including exact auth carrier, permission, stored state, and secret absence?

Read the concrete contracts in `../architecture/`, `../operations/`,
`../security/`, and `../testing/` before implementation. Current evidence
includes `backend/app/__init__.py` (app factory + lifespan),
`frontend/web/src/router/route-loader.ts`,
`frontend/app/src/http/adapters/alova.ts`, and `docker/nginx/nginx.conf`.
