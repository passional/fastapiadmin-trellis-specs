# Authentication and Secrets

## 1. Scope / Trigger

Use this spec for login, refresh, logout, OAuth/WeChat login, password or
account-recovery changes, REST/SSE/WebSocket authentication, token storage,
environment variables, or outbound provider credentials.

## 2. Current authentication contract

The backend issues HS256 JWTs whose payload is deliberately small:

```python
JWTPayloadSchema(sub=session_id, is_refresh=False, exp=...)
JWTPayloadSchema(sub=session_id, is_refresh=True, exp=...)
```

`backend/app/api/v1/module_system/auth/service.py` stores the access token,
refresh token, and a full session record under Redis keys derived from the
session ID. `backend/app/core/dependencies.py::_authenticate` verifies the
signature/type, requires the Redis session, rejects a session whose cached
`user_status` was already disabled, then reloads a non-deleted user from SQL.
It does **not** re-check the reloaded user's current `status`. Logout deletes
all three Redis keys selected by the token in its request body.
With `TOKEN_SLIDING_EXPIRE=True`, JWT `exp` is skipped for access-token
authentication and Redis TTL is the effective expiry boundary.

| Channel | Current carrier and validation | Current status |
|---|---|---|
| REST access token | `Authorization: Bearer <access JWT>` via `OAuth2Schema` and `_authenticate` | Verified in source; missing/invalid credentials return HTTP 401 |
| Refresh | `POST /system/auth/token/refresh` with a JSON **string** body; refresh JWT and Redis session are checked | Web sends the string correctly; App currently sends `{refresh_token: ...}`, a contract gap |
| Logout | Access token in the header; current Web/App send that token again as a JSON string body | Header authenticates, but the backend does not require the body token to match it; the body independently chooses the Redis session to delete |
| AI WebSocket | preferred `Sec-WebSocket-Protocol: access_token, access_token.<jwt>`; query `?token=` fallback | Server supports both; current Web and App consumers use the query fallback |
| System-chat / transfer WebSocket | query `?token=` then shared `_authenticate` | Authenticated, but the token is exposed in the URL |
| Health SSE | no auth dependency on `/common/health/stream` | Public and includes DB/Redis, disk, uptime, and timestamp status |

Evidence: `backend/app/core/security.py`,
`backend/app/core/dependencies.py`, `backend/app/core/base_schema.py`,
`backend/app/api/v1/module_ai/chat/controller.py`,
`backend/app/api/v1/module_system/chat/controller.py`,
`backend/app/api/v1/module_storage/transfer/controller.py`,
`backend/app/api/v1/module_common/health/controller.py`,
`frontend/web/src/utils/http/index.ts`, and
`frontend/app/src/http/adapters/alova.ts`.

Web stores tokens in `localStorage` when “remember me” is enabled and otherwise
in `sessionStorage` (`frontend/web/src/utils/auth/index.ts`). App stores them
through synchronous uni-app storage (`frontend/app/src/store/userStore.ts` and
`frontend/app/src/utils/storage.ts`). These are current client contracts, not
proof of protection from script compromise; any XSS in the same origin can
read browser-managed tokens.

## 3. Password and secret rules

`backend/app/utils/password_util.py` hashes passwords with PBKDF2-HMAC-SHA256,
a random 16-byte salt, and 600,000 iterations. Services hash before create,
change, reset, import-default, OAuth, and WeChat-created user persistence; the
model stores a hash in `sys_user.password`. New code must call `PwdUtil` rather
than compare or persist plaintext. `verify_password` returns `False` for a
malformed stored hash.

Schema validation currently accepts 6-128 characters, while
`PwdUtil.check_password_strength` exists but is not called by the user schemas.
Do not claim upper/lower/digit strength enforcement. A stronger policy must be
introduced consistently for register, change, reset, import, OAuth/WeChat
bootstrap behavior, Web, and App, with migration/compatibility tests.

Secrets are settings, never response fields or logs. Relevant owners include
`SECRET_KEY`, database/Redis passwords, `OAUTH_*_CLIENT_SECRET`,
`WX_MINI_APP_SECRET`, and `OPENAI_API_KEY` in
`backend/app/config/setting.py`. The checked-in `SECRET_KEY` has a development
default despite its comment saying it has no default. Production startup must
override it; do not infer enforcement that is absent. Empty optional provider
secrets disable those provider flows through explicit credential checks.

## 4. Current gaps and required rules

- `POST /system/user/password/forget` is unauthenticated and resets a password
  using only `username` plus a new password. Treat this as account takeover
  exposure until a one-time, expiring, rate-limited proof flow is implemented.
- Login/OAuth/captcha controllers document desired endpoint-specific limits,
  but no backend limiter is registered. The checked-in Nginx config declares
  `limit_req_zone`/`limit_conn_zone` but never applies `limit_req` or
  `limit_conn` in a server/location. Do not report endpoint-, username-, IP-,
  or application-level brute-force controls as implemented.
- Slider completion marks the supplied captcha key verified; the App comment
  accurately notes that its `x` value is not validated. It is not proof of a
  human challenge by itself.
- OAuth state is random, stored in Redis, single-use, and provider-bound, but
  the frontend redirect URL is accepted from the caller and success redirects
  put access and refresh tokens in the query string. The default
  `OAUTH_ALLOWED_HOSTS=["*"]` constrains neither host construction nor the
  frontend redirect. Require an explicit production allowlist and avoid
  bearer tokens in redirects before describing the flow as hardened.
- Query-string WebSocket JWTs can reach proxy/access logs and monitoring.
  Prefer the implemented AI subprotocol carrier where the client supports it;
  for uni-app/system-chat/transfer, a carrier change is a protocol migration
  and must retain/test only an intentionally bounded compatibility window.
- Health SSE exposes infrastructure status publicly. Preserve only the fields
  intentionally required by operators, or add an auth/network boundary before
  adding hostnames, error details, versions with known risk, or identifiers.
- Access authentication checks the Redis session but does not compare the JWT
  with the value stored under the access-token key. Refresh likewise does not
  compare the submitted token with the current stored refresh token. Therefore
  refresh issues replacement JWTs but does not revoke an older signed
  access/refresh JWT while its JWT/session conditions still pass. Preserve this
  as a tested current behavior or add stored-token comparison/revocation tests
  when hardening it.
- Logout authenticates the header session but decodes the body token
  independently; it does not compare session IDs and does not require the body
  to be an access token. A caller with one valid access token and another
  session's signed access/refresh token can select the latter session for
  deletion. Require header/body session equality (or remove the redundant body)
  and cover mismatch/type cases before treating logout as session-bound.
- Disabling a user after login does not immediately invalidate access:
  `_authenticate` checks the cached login-time `user_status`, then reloads the
  SQL user without checking its current `status`. Deletion is observed because
  the SQL query filters `is_deleted`; refresh also checks current `status`.
  Add a current-status check and active-session regression test when hardening
  account revocation.
- Sliding access activity extends access/refresh-token keys but not the
  `USER_SESSION` key. A session still reaches its original session TTL unless
  the refresh flow extends it; do not promise indefinite activity-based
  sliding sessions from the current code.

## 5. Verification matrix

| Case | Assertion point |
|---|---|
| Valid access token + live Redis session | HTTP succeeds and resolves the current, non-deleted SQL user; current enabled status is not re-checked yet |
| Refresh JWT used as access token | HTTP 401; `_authenticate` rejects `is_refresh=True` |
| Signed token after logout / missing Redis session | HTTP 401 even if JWT signature and `exp` are valid |
| User disabled after login | Refresh fails; access currently continues until session/JWT invalidation because access checks cached status |
| User deleted after login | Access and refresh fail because current SQL lookup excludes deleted users |
| Refresh body object instead of JSON string | Contract test must expose 422/current mismatch rather than silently accepting both shapes |
| Logout header/body from different sessions or body is refresh JWT | Contract test exposes the current independent-body behavior until equality/type checks are added |
| WebSocket missing/invalid token | Client-observed handshake/close behavior is asserted; no authenticated manager registration occurs |
| Secret absent/default in production-like config | Startup/config test fails before serving traffic once enforcement is added |
| Password change | Old hash verifies before change, new hash differs from plaintext and uses a fresh salt, old password no longer verifies |

Wrong App refresh shape under the current backend:

```ts
AuthAPI.refreshToken({ refresh_token: refreshToken })
```

Current backend-compatible shape:

```ts
http.Post("/system/auth/token/refresh", JSON.stringify(refreshToken))
```

If the backend is deliberately changed to an object schema, update both
clients, OpenAPI/types, operation-log redaction, and contract tests together.
