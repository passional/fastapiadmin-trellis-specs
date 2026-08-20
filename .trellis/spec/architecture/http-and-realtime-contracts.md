# HTTP and Realtime Contracts

## 1. Scope / Trigger

Use this contract for any endpoint, response field, auth flow, pagination
shape, SSE event, WebSocket message, or frontend API wrapper change.

## 2. Signatures

Ordinary backend JSON responses use:

```python
class ResponseSchema[T](BaseModel):
    code: int
    msg: str
    data: T | None
    status_code: int
    success: bool
```

Controllers declare `response_model=ResponseSchema[...]` and return
`SuccessResponse`; global handlers return the same envelope through
`ErrorResponse`. The definitions are in `backend/app/common/response.py`; the
handlers are in `backend/app/core/exceptions.py`. File, redirect, HTML, SSE,
and streaming endpoints are explicit exceptions.

Public HTTP URLs are built as:

```text
{origin}{VITE_APP_BASE_API}{area prefix}{feature prefix}{endpoint}
```

Both frontend envs use `VITE_APP_BASE_API=/api/v1`. Backend routers themselves
are registered without that prefix (`/system/auth`, `/common/health`, etc.).
`Settings.ROOT_PATH` describes the external mount for OpenAPI/ASGI scope; it
does not register a second set of `/api/v1/...` routes. The reverse proxy or
ASGI server must therefore make the public prefix and backend path agree.

## 3. Contracts

### REST adapter behavior

| Consumer | Base | Successful call returns | Evidence |
|---|---|---|---|
| Admin Web | Axios `baseURL=VITE_APP_BASE_API` | `AxiosResponse<ApiResponse<T>>`; call sites normally read `.data.data` | `frontend/web/src/utils/http/index.ts`, `frontend/web/src/api/module_system/auth.ts` |
| App | Alova base is `VITE_API_BASE_URL + VITE_APP_BASE_API` | business `data` directly after envelope validation | `frontend/app/src/http/adapters/alova.ts`, `frontend/app/src/api/module_system/auth.ts` |

Do not copy Web `.data.data` access into App code. Both ambient `ApiResponse`
types mirror the five-field backend envelope, but the adapters expose different
return values.

Backend and Web pagination use `items`, `total`, `page_no`, `page_size`, and
`has_next` (`backend/app/core/base_schema.py` and
`frontend/web/src/types/global.d.ts`). App's ambient `PageResult` still says
`list`; this is a known mismatch, not a contract to extend. A changed App list
feature must align or adapt it explicitly.

Pydantic/FastAPI validates body/path/query input. Validation failures are HTTP
422 in the normal error envelope. `CustomException` controls business code,
HTTP status, and message; production suppresses its `data`. Web and App both
treat business `code != 0` as failure. Evidence:
`backend/app/core/exceptions.py`, `frontend/web/src/enums/api/result.enum.ts`,
and `frontend/app/src/http/adapters/alova.ts`.

### Realtime protocol matrix

| Channel | Client input | Server output | Authentication / lifecycle |
|---|---|---|---|
| AI chat `/ai/chat/ws` | JSON `{message, session_id?, files?}` or `{action:"stop", session_id?}` | text chunks; `[DONE]`; intended `[STOPPED]`; plain-text validation/error messages | Server supports subprotocols `access_token` and `access_token.<jwt>`; current Web and App clients use query `?token=` |
| System chat `/system/chat/ws` | `ping` only; message mutation remains REST | `pong` or JSON `{type:"message",data}`, `{type:"read",...}`, `{type:"presence",...}` | query `?token=`; server requests close `4001` before accepting an invalid handshake; Web stops reconnecting only if it observes `4001` |
| Storage transfer `/storage/transfer/ws` | `ping` only | `pong` or JSON `{type:"task_update",data:{...}}` | query `?token=`; server requests close `4001` before accepting an invalid handshake; no frontend consumer is currently present |
| Health SSE `/common/health/stream` | GET stream | event `health`, immediately then every 30 seconds | ordinary HTTP route; payload contains dependency status, disk, uptime, timestamp |

Executable owners are
`backend/app/api/v1/module_ai/chat/controller.py`,
`backend/app/api/v1/module_system/chat/{controller.py,service.py,ws_manager.py}`,
`backend/app/api/v1/module_storage/transfer/{controller.py,engine.py,ws_manager.py}`,
and `backend/app/api/v1/module_common/health/controller.py`.

Client examples are `frontend/app/src/composables/useAiChat.ts` and
`frontend/web/src/api/module_system/chat.ts`. The AI server supports the
subprotocol form so browser clients can keep JWTs out of URLs/access logs, but
`frontend/web/src/views/module_ai/chat/index.vue` and the App composable both
currently use the supported query fallback. Do not describe the safer carrier
as deployed until the Web client is changed and tested; uni-app cannot rely on
the browser WebSocket subprotocol API.

### Current AI stop limitation

The AI controller receives a request and then awaits the entire streaming
`async for` before calling `receive_text()` again. Although `ChatService` checks
`stop_event` between chunks and the controller contains an active-stop branch,
the same connection cannot read `{action:"stop"}` while that loop is running.
In current execution, a stop message is processed only after generation has
finished and `[DONE]` has been sent, at which point it normally receives the
idle response instead of `[STOPPED]`. Treat `[STOPPED]` as an unreachable
intended branch until receiving and generation are coordinated concurrently.

## 4. Validation & Error Matrix

| Condition | Boundary behavior |
|---|---|
| Invalid REST body/query/path | HTTP 422, envelope `success=false`; first validation message becomes `msg` |
| `CustomException` | Declared HTTP status and business code; `data` hidden in production |
| SQL integrity/connection failure | Mapped by global exception handlers; details hidden in production |
| Missing AI WebSocket token | Server attempts text before `accept()` and then closes; do not assume the text reaches the client |
| Malformed AI JSON/schema | Connection remains open; error text is sent |
| `stop` sent while AI generation is active | Not read until generation completes; current concurrency gap |
| `stop` while idle | Plain text "当前没有正在进行的生成任务" |
| Invalid system-chat/transfer token | Server calls `close(4001)` before `accept()`; observable browser close code/reconnect behavior needs a handshake test |
| Unknown pushed JSON kind | Current clients ignore or must narrow it; add the type union before emitting a new kind |

## 5. Good / Base / Bad Cases

- Good: a field is added to its backend Pydantic schema, Web transport type,
  App transport type where consumed, realtime union/decoder where relevant,
  and behavior tests.
- Base: a Web-only endpoint updates the backend and Web wrapper while leaving
  App untouched after confirming it has no consumer.
- Bad: returning a raw dict from a normal controller, changing `items` to
  `list` for one client, or treating all WebSocket channels as one protocol.

## 6. Tests Required

- Backend: assert HTTP status and all five envelope fields, plus invalid and
  permission cases. Do not settle for `status_code != 404`.
- Web: run type-check/tests and assert Axios interceptor behavior or the API
  wrapper's `.data.data` consumer.
- App: run type-check and assert Alova returns business `data`, including 401
  refresh and non-zero business code behavior.
- WebSocket: test authenticated and rejected handshakes, the client-observed
  close code, malformed input, heartbeat, terminal/close behavior, and every
  new push discriminant. An AI stop test must prove the server observes `stop`
  before the stream naturally completes.
- Proxy-sensitive changes: test both direct backend paths and public
  `/api/v1/...` paths through Nginx.

The current repository has no dedicated WebSocket contract suite; any protocol
change should add one rather than rely only on manual testing.

## 7. Wrong vs Correct

Wrong App consumer:

```ts
const response = await AuthAPI.login(body)
const token = response.data.data.access_token
```

Correct App consumer (Alova already unwraps `data`):

```ts
const result = await AuthAPI.login(body)
const token = result.access_token
```
