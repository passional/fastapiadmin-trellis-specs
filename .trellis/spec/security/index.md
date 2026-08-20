# Security Specifications

These specs describe FastApiAdmin's executable security boundaries. They
separate verified controls from current gaps and from rules for new changes;
comments or README claims are not proof that a control is active.

| Spec | Covers |
|---|---|
| [Authentication and Secrets](./authentication-and-secrets.md) | JWT/Redis sessions, REST and realtime token carriers, password handling, OAuth, and configuration secrets |
| [Authorization and Data Scope](./authorization-and-data-scope.md) | Permission strings, superuser behavior, row-level scope, public routes, and authorization tests |
| [Untrusted Content and Audit Data](./untrusted-content-and-audit-data.md) | Upload/download validation, HTML sanitization, operation-log exposure, and sensitive-data verification |

Use all three for authentication, account recovery, permissions, uploads,
rich content, operation logs, WebSockets, or third-party integrations. Also
read `../architecture/http-and-realtime-contracts.md` when a transport contract
changes and `../testing/contract-and-deployment-testing.md` for required
negative-path assertions.

