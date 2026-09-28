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

`backend/app/modules/system/auth/service.py` stores the access token, refresh
token, and a full session record under Redis keys derived from the session ID.
`backend/app/core/dependencies.py::_authenticate` verifies the signature/type,
requires the Redis session, rejects a session whose cached `user_status` was
already disabled, then reloads a non-deleted user from SQL. It does **not**
re-check the reloaded user's current `status`. It also does not compare the
presented JWT with the value stored under the access-token Redis key. Logout
deletes all three Redis keys selected by the token in its request body.
With `TOKEN_SLIDING_EXPIRE=True`, JWT `exp` is skipped for access-token
authentication and Redis TTL is the effective expiry boundary.

Sliding activity renews the `USER_SESSION` key (the survival key) **and** the
refresh-token key, but is bounded by `SESSION_MAX_LIFETIME_SECONDS` (absolute
cap). A session whose `created_at` age exceeds the cap is deleted and rejected
with 401 `"会话超过最大存活时长，请重新登录"`; legacy sessions without
`created_at` are backfilled from the current time rather than killed. See
`backend/app/core/dependencies.py:124-156`.

Refresh tokens rotate **with replay detection**: `LoginService.refresh_token`
compares the submitted `refresh_token` with the stored value under the
refresh-token key. A mismatch (or missing stored value) deletes the session,
refresh, and access keys, logs `"检测到疑似 refresh token 重放，已撤销会话"`,
and rejects with `"刷新凭证已失效，请重新登录"`. See
`backend/app/modules/system/auth/service.py:449-462`.

| Channel | Current carrier and validation | Current status |
|---|---|---|
| REST access token | `Authorization: Bearer <access JWT>` via `OAuth2Schema` and `_authenticate` | Verified in source; missing/invalid credentials return HTTP 401 |
| Refresh | `POST /system/auth/token/refresh` with a JSON **string** body; refresh JWT, Redis session, and stored-refresh-token equality are checked | Web sends the string correctly; App currently sends `{refresh_token: ...}`, a contract gap. Mismatch revokes the session (replay detection) |
| Logout | Access token in the header; current Web/App send that token again as a JSON string body | Header authenticates, but the backend does not require the body token to match it; the body independently chooses the Redis session to delete |
| AI WebSocket | preferred `Sec-WebSocket-Protocol: access_token, access_token.<jwt>`; query `?token=` fallback | Server supports both; Web now uses the subprotocol carrier (`views/module_ai/chat/index.vue`), App still uses the query fallback because mini-program WebSockets cannot set subprotocols |
| Storage transfer SSE | `GET /task/storage/transfer/stream` (`EventSourceResponse`), authenticated per request | No token in the URL; the WebSocket carrier of the removed transfer WS is gone |
| Health SSE | no auth dependency on `/monitor/health/stream` | Public and includes DB/Redis, disk, uptime, and timestamp status |

Evidence: `backend/app/core/security.py`,
`backend/app/core/dependencies.py`, `backend/app/core/base_schema.py`,
`backend/app/modules/ai/chat/controller.py`,
`backend/app/modules/task/storage/transfer/controller.py`,
`backend/app/modules/monitor/health/controller.py`,
`frontend/web/src/utils/http/index.ts`,
`frontend/web/src/views/module_ai/chat/index.vue`, and
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

- `POST /system/user/password/forget` remains unauthenticated
  (`backend/app/modules/system/user/controller.py`), but
  `UserService.forget_password` (`backend/app/modules/system/user/service.py`)
  **no longer sets a password**. It logs the request and returns a uniform,
  anti-enumeration response; unauthenticated callers cannot change any
  password. Password changes require an authenticated change/reset flow. Do not
  describe this endpoint as account-takeover exposure any more; if a real
  self-service reset is added, it must use a one-time, expiring,
  rate-limited proof flow.
- Login rate limiting **is implemented**: `LoginService._check_login_rate_limit`
  uses the Redis fixed-window key `login_rate_limit:{ip}` with
  `LOGIN_RATE_LIMIT_WINDOW_SECONDS` / `LOGIN_RATE_LIMIT_MAX_ATTEMPTS`
  (`backend/app/modules/system/auth/service.py:155-168`, called from
  `authenticate_user`). It short-circuits for `unknown`/loopback IPs and fails
  open on Redis errors. Nginx additionally applies
  `limit_req zone=api_limit burst=60 nodelay` and
  `limit_conn conn_limit 100` on `/api/v1`
  (`docker/nginx/nginx.conf`). A global `core/rate_limiter.py` /
  `fastapi-limiter` limiter was added and then removed — do not describe it as
  existing.
- Slider completion binds the captcha key to an issuer fingerprint (IP + UA),
  enforces `CAPTCHA_MIN_VERIFY_SECONDS` as a minimum residence time before
  verification, and consumes the key one time
  (`CaptchaService` in `backend/app/modules/system/auth/service.py`). It still
  is not cryptographic proof of a human challenge by itself — the App comment
  correctly notes the slider `x` value is not independently validated.
- OAuth state is random, stored in Redis, single-use, and provider-bound, but
  the frontend redirect URL is accepted from the caller and success redirects
  put access and refresh tokens in the query string. The default
  `OAUTH_ALLOWED_HOSTS=["*"]` constrains neither host construction nor the
  frontend redirect. Require an explicit production allowlist and avoid
  bearer tokens in redirects before describing the flow as hardened.
- The AI WebSocket prefers the `Sec-WebSocket-Protocol` carrier so the JWT does
  not enter URL/access logs; the App (uni-app) client still sends `?token=`
  because mini-program WebSockets cannot set subprotocols. Query-string JWTs
  can reach proxy/access logs — treat the App fallback as a bounded
  compatibility path and migrate it only through a tested protocol change.
- Health SSE (`/monitor/health/stream`) exposes infrastructure status publicly.
  Preserve only the fields intentionally required by operators, or add an
  auth/network boundary before adding hostnames, error details, versions with
  known risk, or identifiers.
- Access authentication checks the Redis session but does not compare the JWT
  with the value stored under the access-token key. Refresh, by contrast, now
  compares the submitted token with the stored refresh token and revokes the
  whole session on mismatch (replay detection, see §2). The remaining access
  nuance means a signed access JWT stays usable while its session and cached
  status conditions pass; preserve this as tested current behavior or add
  stored-token comparison/revocation tests when hardening it.
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
- Sliding access activity now renews the `USER_SESSION` survival key **and** the
  refresh-token key, bounded by `SESSION_MAX_LIFETIME_SECONDS`. A session can no
  longer slide indefinitely: once its absolute age cap is reached,
  `_authenticate` deletes it and returns 401. Legacy sessions without
  `created_at` are backfilled rather than force-expired.

## 5. Verification matrix

| Case | Assertion point |
|---|---|
| Valid access token + live Redis session | HTTP succeeds and resolves the current, non-deleted SQL user; current enabled status is not re-checked yet |
| Refresh JWT used as access token | HTTP 401; `_authenticate` rejects `is_refresh=True` |
| Signed token after logout / missing Redis session | HTTP 401 even if JWT signature and `exp` are valid |
| User disabled after login | Refresh fails; access currently continues until session/JWT invalidation because access checks cached status |
| User deleted after login | Access and refresh fail because current SQL lookup excludes deleted users |
| Refresh body object instead of JSON string | Contract test must expose 422/current mismatch rather than silently accepting both shapes |
| Refresh with a token that differs from the stored refresh token | Session is revoked (access/refresh/session keys deleted) and the caller gets an auth failure; the old refresh token can no longer be reused |
| Repeated login attempts from one IP beyond `LOGIN_RATE_LIMIT_MAX_ATTEMPTS` in the window | Request is rejected with `"登录尝试过于频繁，请稍后再试"`; loopback/unknown IPs are exempt |
| Sliding request near session age `SESSION_MAX_LIFETIME_SECONDS` | `USER_SESSION` and refresh keys are renewed while under the cap; at/over the cap the session is deleted and returns 401 |
| Logout header/body from different sessions or body is refresh JWT | Contract test exposes the current independent-body behavior until equality/type checks are added |
| AI WebSocket missing/invalid token | Handshake closes 4001 (auth); authenticated-but-unauthorized chat closes 4003 (permission); no registration occurs |
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
