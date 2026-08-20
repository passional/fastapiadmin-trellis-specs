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
- delegates downloads to `module_common/file/service.py`, which normalizes the
  requested path and rejects paths it considers outside the upload root.

Preserve those checks, but do not overstate them. The current content-type
validator only logs an extension/magic mismatch and returns `True`; unknown
content also passes. The pre-read size check does not enforce actual bytes when
`file.size` is absent or wrong, and the entire upload is read once before the
chunked write. SVG is allowed and may contain active content when served from
the application origin. Download confinement uses a string `startswith`
comparison rather than `Path.is_relative_to`, so sibling-prefix paths require
an explicit regression test.

Storage source and transfer uploads have separate services under
`backend/app/api/v1/module_storage/`; common-upload validation is not inherited
automatically. Inspect every entry point rather than assuming `UploadUtil`
protects it.

For a new/changed upload, assert all of:

| Case | Required behavior |
|---|---|
| Allowed extension + matching safe content | Saved below the intended type directory with generated name |
| Original filename with `../`, backslash, slash, or NUL | Rejected before file read/write; no file created |
| Resource `target_path` with traversal/NUL/absolute input | Assert current behavior explicitly: `..`/NUL reject only after the full upload read, while leading separators are normalized away rather than rejected |
| Dangerous final extension or multiple extensions | Dangerous/unsupported **final** extension is rejected; earlier suffixes are stripped from the generated basename but are not independently validated |
| Download outside root, including sibling-prefix path | Reject after canonical containment is implemented; current string-`startswith` check needs a regression that exposes the sibling-prefix gap |
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

This guarantee applies only when data passes through that schema. The workflow
handler in
`backend/app/api/v1/module_task/workflow/flows/handlers/builtin_nodes.py`
calls `NoticeCRUD.create` with a plain dict, so stored workflow-created notice
content bypasses the input validator. Trace direct CRUD/import/background
writers as well as HTTP schemas before claiming all notice rows are sanitized.

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
`sys_operation_log`. Detail and export routes are permission-protected, but
the stored values are not redacted.

Current high-risk examples include:

- login form password and login response access/refresh tokens;
- refresh/logout body tokens and refreshed token response;
- register/change/reset/forget-password bodies;
- OAuth/provider/model configuration values or future secret-bearing forms.

`_PUBLIC_WRITE_PATHS` does not disable logging; it only participates in route
initialization and currently adds no security dependency. Never put a secret
into a route and assume that set prevents capture.

The rule for new code is allowlist/redact before serialization, recursively and
case-insensitively, for fields such as `password`, `old_password`,
`new_password`, `confirm_password`, `access_token`, `refresh_token`,
`authorization`, `secret`, and `api_key`. For known credential endpoints,
prefer metadata/policy that suppresses request and response bodies entirely.
Apply the same policy to form bodies, JSON, query/path logging, exception data,
application logs, exports, and WebSocket URLs. Redaction must occur before the
background task receives `log_data`.

## 5. Good / Base / Bad cases

- Good: operation-log tests submit unique sentinel secrets, then query the
  persisted row/export/log capture and assert no sentinel appears anywhere.
- Base: ordinary non-sensitive writes retain useful route, actor, status,
  duration, and safe identifiers without full bodies.
- Bad: masking only the UI detail view, truncating secrets after storage,
  logging full provider responses, or returning an upload URL before enforcing
  storage and serving policy.

Wrong:

```python
oper_param["body"] = json.loads(payload.decode())
```

Correct direction:

```python
oper_param["body"] = redact_sensitive(json.loads(payload.decode()))
```

The helper itself needs nested dict/list, mixed-case field, form-data, and
false-positive tests. A security fix is incomplete if old retained operation
logs remain readable without a deliberate cleanup/rotation decision.
