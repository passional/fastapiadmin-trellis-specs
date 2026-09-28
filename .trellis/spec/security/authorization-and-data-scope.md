# Authorization and Data Scope

## 1. Scope / Trigger

Use this spec for controllers, permission strings, menus/roles, superuser
behavior, data-scope changes, CRUD/bulk operations, current-user endpoints,
WebSocket actions, exports/imports, and generated modules.

## 2. Permission contract

Protected management endpoints use a concrete dependency such as:

```python
auth: Annotated[
    AuthSchema,
    Security(AuthPermission(["module_system:user:update"])),
]
```

The established permission signature is `module_<area>:<resource>:<action>`;
generated examples include
`module_example:demo:{query,detail,create,update,delete,patch,export,import,download}`.
Backend code only ever uses the `module_` prefix (`module_system:dept:query`,
`module_ai:chat:query`, `module_monitor:dashboard:query`); the older
`sys:user:add` style examples are stale and must not be copied. The string must
agree across the controller, menu/button records, session permission
collection, Web directives/buttons, and generator templates.

The Web button directive is `v-hasPerm`
(`frontend/web/src/directives/permission/index.ts`). It accepts a single
permission string or an array of strings, throws if given any other shape, and
delegates the decision to `useAuth().hasAuth` (backend any-of semantics). When
the check fails it removes the element from its parent; it is a UI convenience
only and never replaces the server-side `AuthPermission` dependency.

`AuthPermission` in `backend/app/core/dependencies.py` implements **any-of**
semantics for a list, returns 403 when the authenticated non-superuser owns
none of the requested strings, and bypasses checks for `is_superuser`. It also
bypasses when the dependency itself requests `*` or `*:*:*`; those wildcards
are endpoint configuration, not user-granted wildcard matching. Use the
narrowest explicit action and do not add a wildcard to make a failing test
pass.

Permissions and menu IDs are collected from enabled roles/menus at login and
stored in the Redis session (`LoginService._collect_permissions` and
`_build_session_dict`). `_authenticate` reloads user status from SQL but uses
the session's permission list. Therefore role/menu changes may remain stale
for existing sessions unless they are explicitly invalidated/refreshed; test
the intended revocation behavior when changing role assignment.

Account state is asymmetric: a deleted user stops authenticating because the
SQL reload filters `is_deleted`, but a user disabled after login can continue
using access tokens because `_authenticate` checks cached `user_status` rather
than the reloaded SQL `status`. Refresh does check current status. Do not claim
immediate disable revocation until the access path is fixed and tested.

## 3. Row-level data scope

`backend/app/core/permission.py` defines the executable scopes:

| Value | Meaning | Filter behavior |
|---|---|---|
| `1` | Self | `UserModel.id == user.id`, otherwise `created_id == user.id` |
| `2` | Department and children | `UserModel.dept_id IN (...)`, a creator's department where the relationship exists, otherwise self fallback |
| `3` | All | no row filter |

Superusers bypass the filter. `CRUDBase._build_conditions` applies the
condition to `get`, `count`, `get_list`, and `page`, so ordinary reads and the
read-before-write path in `update` are scoped. Models without `created_id`
cannot be scoped by this mechanism.

Current limitations must remain visible:

- `RoleDeptsModel` and its comment mention custom scope value `5`, but
  `Permission` implements only 1/2/3. Do not claim custom-department scope.
- The live role query in `Permission._load_user_data_scopes` (called by
  `_filter_by_data_scope`) does not filter
  `RoleModel.status` or `RoleModel.is_deleted`. Disabled/soft-deleted associated
  roles can therefore still influence row scope, including an unfiltered
  `data_scope=3`; add status/deletion filters and regression cases before
  treating role disable/delete as row-scope revocation.
- `CRUDBase.delete`, `set`, `restore`, and `clear` build direct UPDATE/DELETE
  statements and do not apply `_permission_condition`. A controller permission
  still protects the action, but row-level data scope is not automatically
  enforced for those bulk paths.
- Department traversal catches every exception and falls back to the current
  department. This is fail-narrower for that lookup, but the exception is not
  surfaced for diagnosis.
- Services that issue custom SQL outside `CRUDBase._build_conditions` own the
  row filter explicitly.

For a scoped mutation, first resolve targets through a scoped query or add the
same permission condition to the mutation statement. Never authorize only the
collection endpoint and then mutate caller-supplied IDs globally.

## 4. Public versus authenticated routes

Authentication is dependency-driven; `WHITE_API_LIST_PATH` in settings is
used by demo-mode middleware and is not a global authentication allowlist.
Do not infer that a path is protected or public from that list. Inspect the
controller dependencies.

Examples:

- Authenticated self-service: `/system/user/current/info`, current update,
  password change.
- Explicit action permission: user list/detail/admin reset, file
  upload/download, operation-log detail/export.
- Public by current controller definition: login/refresh/captcha,
  registration, password-forget, OAuth callbacks, health routes/SSE
  (`/monitor/health/check`, `/monitor/health/stream`).
- Realtime: REST route dependencies do not protect a WebSocket; the AI chat
  endpoint performs handshake authentication via `websocket_authenticate`
  (4001 on auth failure, 4003 when the permission check fails) before accepting
  the connection or processing messages. Storage transfer progress is SSE
  (`/task/storage/transfer/stream`), authenticated per request.

## 5. Good / Base / Bad cases

- Good: test an ordinary user with the exact action permission against own,
  same-department, child-department, and outside-scope records, then repeat the
  mutation's direct-ID/bulk path.
- Base: an authenticated self-service endpoint derives the target ID from
  `auth.user.id`, not from a body/path ID.
- Bad: UI-only button hiding, a menu permission without a controller
  dependency, a WebSocket accepted before `_authenticate`, or a bulk delete
  that trusts submitted IDs after only checking `module_x:y:delete`.

Wrong:

```python
await DemoCRUD(auth, db).delete(ids=data.ids)
```

Correct direction for scoped data (exact implementation may live in CRUD or
service):

```python
allowed = await DemoCRUD(auth, db).get_list(search={"id": ("in", data.ids)})
await DemoCRUD(auth, db).delete(ids=[item.id for item in allowed])
```

The security test must also prove outside-scope IDs were untouched; a 200
response alone is insufficient.

## 6. Required assertions

- Missing/malformed token -> 401; authenticated but missing action -> 403.
- One of multiple requested permissions is enough, because current semantics
  are any-of; use separate dependencies if all-of behavior is required.
- Superuser succeeds and an ordinary user is limited. Test deleted-user access,
  newly-disabled access versus refresh, and disabled/deleted role data scope as
  separate current behaviors; do not collapse them into one revocation claim.
- Permission/menu edits have an explicit active-session expectation.
- List, detail, update, delete, batch set, restore, import/export, and custom
  SQL each receive row-scope coverage when they expose scoped data.
- Permission strings in controller/menu/Web/generator output are compared as
  exact strings, not suffix-only checks.
