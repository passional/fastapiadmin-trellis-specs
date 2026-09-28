# Untrusted Content and Audit Data

## 1. Scope / Trigger

Use this spec for multipart uploads, storage/transfer sources, downloads,
static serving, rich HTML/Markdown, operation/login logs, request/response
logging, exports, and any schema containing passwords, tokens, provider keys,
personal data, or private content.

## 2. Common file boundary

`/common/file/upload` and `/common/file/download` require exact permissions
`module_common:file:upload` and `module_common:file:download` plus an
authenticated user. `backend/app/utils/upload_util.py` currently:

- rejects path separators/NUL in original filenames;
- sanitizes names and generates a timestamp/machine/random-suffix filename;
- blocks a dangerous-extension set and allows only `settings.ALLOWED_EXTENSIONS`;
- checks `UploadFile.size` when supplied;
- reads content, performs limited magic-byte detection, confines the resolved
  destination under `UPLOAD_FILE_PATH`, and writes in chunks;
- delegates downloads to `backend/app/modules/common/file/service.py`, which
  normalizes the requested path and rejects paths outside the upload root.

Preserve those checks, but do not overstate them. The current content-type
validator only logs an extension/magic mismatch and returns `True`; unknown
content also passes. The pre-read size check does not enforce actual bytes when
`file.size` is absent or wrong, and the entire upload is read once before the
chunked write. SVG is allowed and may contain active content when served from
the application origin. Download confinement is now canonical:
`Path(file_path).resolve().is_relative_to(settings.UPLOAD_FILE_PATH.resolve())`
(`backend/app/modules/common/file/service.py:52-56`), which closes the old
`startswith` sibling-prefix bypass. Keep a guard test for sibling-prefix paths.

Storage source and transfer uploads have separate services under
`backend/app/modules/task/storage/`; common-upload validation is not inherited
automatically. Inspect every entry point rather than assuming `UploadUtil`
protects it.

For a new/changed upload, assert all of:

| Case | Required behavior |
|---|---|
| Allowed extension + matching safe content | Saved below the intended type directory with generated name |
| Original filename with `../`, backslash, slash, or NUL | Rejected before file read/write; no file created |
| Resource `target_path` with traversal/NUL/absolute input | Assert current behavior explicitly: `..`/NUL reject only after the full upload read, while leading separators are normalized away rather than rejected |
| Dangerous final extension or multiple extensions | Dangerous/unsupported **final** extension is rejected; earlier suffixes are stripped from the generated basename but are not independently validated |
| Download outside root, including sibling-prefix path | Rejected by `Path.resolve().is_relative_to(upload_root)`; keep a guard test that a same-prefix sibling directory cannot pass |
| Claimed image/office extension with mismatched bytes | Reject once strict validation is implemented; current warning-only behavior is a documented gap |
| Actual bytes exceed `MAX_FILE_SIZE` | Streaming read aborts and partial file is removed once byte-count enforcement is implemented |
| SVG/HTML-capable content | Served from a sandboxed/separate origin or rejected unless an explicit active-content policy is tested |

## 3. HTML and rendered content

`NoticeCreateSchema.notice_content` is sanitized with Bleach through
`backend/app/utils/xss_util.py`; update inherits the same validator. The
allowlist includes rich tags, selected attributes, and CSS properties and
strips comments/disallowed tags. Add sanitizer tests for script tags, event
attributes, `javascript:`/dangerous URL schemes, image/video sources, style
properties, and create/update parity before changing the allowlist.

This guarantee applies only when data passes through that schema. Notice
creation is currently reached only through the validated service path
(`NoticeCRUD.create` in `backend/app/modules/system/notice/service.py`); the
old workflow notice handler that wrote a plain dict was removed, so no known
bypass exists today. Trace direct CRUD/import/background writers as well as
HTTP schemas before claiming all notice rows are sanitized — any new writer
that calls `NoticeCRUD.create` with a raw dict reintroduces the bypass.

This is notice-specific. Ticket/chat/AI/generated or other text fields do not
automatically call `sanitize_html`. Vue interpolation escapes text, but any
`v-html`, rich editor, Markdown renderer, WebView, exported HTML, or stored
content reused by another client creates a separate sink. Trace source ->
storage -> every renderer; sanitize at the owning trust boundary and keep
output encoding/sandboxing at the sink.

## 4. Operation-log exposure

`OperationLogRoute` in `backend/app/core/router_class.py` captures write
request bodies/forms and JSON responses, truncates only after 2,000 serialized
request characters, and writes `request_payload` plus `response_json` to
`sys_operation_log`. Detail and export routes are permission-protected. The
stored values **are redacted**: `_SENSITIVE_KEYS` and `_redact_sensitive`
replace credential fields with `******` recursively and case-insensitively
across the JSON request body, form fields, and the JSON response body
(`router_class.py:15-41,90,97,112`). `_SENSITIVE_KEYS` currently covers
`password`, `passwd`, `old_password`, `new_password`, `confirm_password`,
`captcha_key`, `token`, `access_token`, `refresh_token`, `api_key`, `apikey`,
`secret`, `client_secret`, `secret_key`, and `authorization`. This closes the
former leak of login passwords and login/refresh returned tokens.

Residual caveat: redaction is key-name based, so secret material carried in a
free-text field (for example AI/chat message content) is not detected. Treat
any new route that passes secrets through non-credential-named fields as a
gap to fix.

Do not put a secret into a route and assume some path list prevents capture;
logging is configured per method by `OPERATION_RECORD_METHOD`.

The rule for new code is redact before serialization, recursively and
case-insensitively; extend `_SENSITIVE_KEYS` for any new credential field. For
known credential endpoints, prefer metadata/policy that suppresses request and
response bodies entirely. Apply the same policy to form bodies, JSON,
query/path logging, exception data, application logs, exports, and WebSocket
URLs. Redaction must occur before the background task receives `log_data`.

## 5. Good / Base / Bad cases

- Good: operation-log tests submit unique sentinel secrets, then query the
  persisted row/export/log capture and assert no sentinel appears anywhere.
  This is now a **regression guard on implemented redaction**, not a request to
  build it.
- Base: ordinary non-sensitive writes retain useful route, actor, status,
  duration, and safe identifiers without full bodies.
- Bad: masking only the UI detail view, truncating secrets after storage,
  logging full provider responses, adding a new credential-named field without
  extending `_SENSITIVE_KEYS`, or returning an upload URL before enforcing
  storage and serving policy.

Wrong (would persist credentials):

```python
oper_param["body"] = json.loads(payload.decode())
```

Current redacted behavior:

```python
oper_param["body"] = _redact_sensitive(json.loads(payload.decode()))
```

Regression tests should cover nested dict/list, mixed-case field names,
form-data fields, the JSON response body, and false positives (non-sensitive
keys left intact). Old retained operation logs remain readable until a
deliberate cleanup/rotation decision.
