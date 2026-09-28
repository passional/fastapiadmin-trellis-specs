# Code Generation and Routing

## 1. Scope / Trigger

Use this contract when changing generator schemas, Jinja templates, generated
paths, permissions, menu creation, preview/ZIP/local output, or the plugin route
loader.

The generator targets the backend plugin tree and Admin Web only. It does not
generate uni-app or docs code.

## 2. Signatures

Important API signatures from
`backend/app/modules/generator/gencode/controller.py`:

```text
GET   /generator/gencode/list                 PaginationQueryParam -> page of GenTableOutSchema
GET   /generator/gencode/db/list              paginated database tables
POST  /generator/gencode/import               import table definitions
GET   /generator/gencode/detail/{table_id}    GenTableOutSchema
POST  /generator/gencode/create               -> bool
PUT   /generator/gencode/update/{table_id}    GenTableSchema
DELETE /generator/gencode/delete              -> None
GET   /generator/gencode/preview/{table_id}   -> path-to-content map
PATCH /generator/gencode/batch/output         list[str] -> code.zip
POST  /generator/gencode/output/{table_name}  -> local files + menu records
POST  /generator/gencode/sync_db/{table_name}
GET   /generator/gencode/sync_db/preview/{table_name} -> GenSyncPreviewSchema
```

`sync_db/preview/{table_name}` is the diff preview contract for a table sync and
is the route a client should call before `sync_db/{table_name}`.

`GenTableSchema` owns `table_name`, `class_name`, `package_name`,
`module_name`, `business_name`, `function_name`, optional main/sub-table
fields, `parent_menu_id`, and `columns`. Its validators normalize package names
to `module_xxx`, remove that prefix from `module_name`, and normalize
multi-segment `business_name`. See
`backend/app/modules/generator/gencode/schema.py`.

## 3. Contracts

### Templates and outputs

`Jinja2TemplateUtil.get_template_list()` is the main-table output set:

| Template | Output |
|---|---|
| `python/{controller,service,crud,schema,model}.py.jinja2` | `backend/app/plugin/{package_name}/{module_name}/...` |
| `python/__init__.py.jinja2` | `backend/app/plugin/{package_name}/{module_name}/__init__.py` |
| `python/package_init.py.jinja2` | `backend/app/plugin/{package_name}/__init__.py` |
| `python/plugin.toml.jinja2` | `backend/app/plugin/{package_name}/plugin.toml` |
| `ts/api.ts.jinja2` | `frontend/web/src/api/{package_name}/{module_name}.ts` |
| `vue/index.vue.jinja2` | `frontend/web/src/views/{package_name}/{module_name}/index.vue` |

Main/sub-table generation adds only model, schema, and `__init__` for the
child. The template list, mapping, and context live together in
`backend/app/modules/generator/gencode/jinja2_template_util.py`; the actual
sources live in `backend/templates/{python,ts,vue}/`.

Because the package-level `package_init.py.jinja2` and `plugin.toml.jinja2` are
registered templates, the package `__init__.py` **and** the package
`plugin.toml` are part of preview, batch ZIP, and local output. Local output
additionally runs a scaffolding safety net after each template write: it ensures
`backend/app/plugin/{package_name}/__init__.py` and
`backend/app/plugin/{package_name}/{module_name}/__init__.py` exist, creating
them when missing so the package hierarchy stays importable. This scaffold is
not a separate registered template; it only fills gaps.

Change the Jinja source first. A patch to a previously generated plugin/page
does not update preview, ZIP, or the next generated feature.

### Jinja2 scoping pitfall

Jinja `{% set %}` inside a `{% for %}` loop does not leak into the outer scope
(for-block isolation), so accumulator booleans set per iteration are lost.
Compute such flags with a filter expression instead, e.g.
`{% set has_date_validator = columns | selectattr('python_type', 'equalto', 'date') | list | length > 0 %}`
(see `backend/templates/python/schema.py.jinja2`). This was a real bug: a
missing validator import was omitted from the generated `schema.py` because the
flag was computed with a leaked loop variable.

### Restart after codegen

Generated modules are only discovered at application startup (see
`architecture/system-boundaries.md`). After generating or moving a module you
**must restart the backend**; uvicorn dev reload does not re-run
`core/discover.py`, so the new routes silently 404 until the process restarts.

### Route, component, and permission coupling

For `package_name=module_example` and `module_name=demo`:

- dynamic backend container: `/example`, from
  `backend/app/core/discover.py`;
- feature router: `/demo`, from the generated controller template;
- frontend API base: `/example/demo`, from `backend/templates/ts/api.ts.jinja2`;
- generated view/component path: `module_example/demo/index`;
- permissions: `module_example:demo:{query,detail,create,update,delete,patch,export,import,download}`;
- generated menu route currently: `/module_example/demo`.

These values are assembled in
`Jinja2TemplateUtil.prepare_context/get_file_name`,
`GenTableService.generate_code`, the Python/TS/Vue templates, and the Web
component loader. Update and test them as one contract.

The `generate_code` docstring still describes pages as
`/{module_xxx}/{module_name}/{business_path}` and the component as
`module_xxx/module_name/business_path/index`, but the executable mapping uses
only package + module: `_route_path = f"/{route_seg}/{_mn}"` and
`_component_path = f"{_pn}/{_mn}/index"`. `business_path`/`business_file` are
still computed and passed into the Jinja context for template compatibility but
are no longer a directory level ("不再额外使用业务名作为目录层级"). Treat the
executable mapping above as current behavior; do not reproduce the stale
docstring in new code.

### Write and failure behavior

- Preview renders every output into an in-memory map. Individual render errors
  become `"渲染错误: ..."` values rather than failing the whole preview.
- Batch ZIP skips failed tables, continues with successful ones, and sets
  `X-Skipped-Tables`; it fails only if zero files were produced.
- Local output calls `write_text` unconditionally and can overwrite existing
  generated files. Despite the "可跳过覆盖" docstring, there is no overwrite
  guard.
- Local output writes files before creating directory/menu/button records. This
  avoids menu rows when rendering fails, but a later menu conflict can leave
  files on disk; there is no filesystem rollback.
- The preview controller advertises `ResponseSchema[GenTableOutSchema]`, while
  the service actually returns a path-to-rendered-content mapping. Because the
  controller returns a `JSONResponse`, the live payload follows the mapping,
  but the declared OpenAPI response is stale. Do not generate a client from
  that preview schema until the declaration is aligned.

Evidence: `backend/app/modules/generator/gencode/service.py` and
`controller.py`.

## 4. Validation & Error Matrix

| Condition | Current behavior |
|---|---|
| Empty table name/list | `CustomException` |
| Missing package/module/function | Rejected before local generation |
| Invalid slug characters | Normalized by `GenTableSchema` validators |
| Missing main/sub table pair or FK | Hint/invalid configuration; generation is blocked by service validation |
| Missing template mapping | `ValueError` from `get_file_name` |
| Preview template render error | Error text stored under the intended output path |
| Preview OpenAPI client generation | Declared response type does not match the live path-to-content map |
| Some ZIP tables fail | Successful ZIP plus `X-Skipped-Tables` |
| All ZIP tables fail | `CustomException` |
| Existing local file | Overwritten |
| Existing function menu | Menu creation fails after files may already have been written |
| New module generated but backend not restarted | Routes silently 404; must restart |

## 5. Good / Base / Bad Cases

- Good: update a template, context/type mapping, preview expectations, ZIP
  paths, local-output paths, menu/component/permission assertions, then
  generate a representative slice and restart the backend.
- Base: change template formatting without changing fields or paths; render
  every template and run backend/Web format/type checks.
- Bad: edit only `frontend/web/src/views/module_xxx/...`, or add a template
  without registering it in both the template list and path mapping, or expect
  generated routes to appear without a restart.

## 6. Tests Required

- Validate `GenTableSchema` normalization for package/module/business names and
  invalid main/sub configurations.
- Render all templates for a representative table and assert the exact output
  file set (including package-level `__init__.py` and `plugin.toml`), API
  prefix, component path, and permission prefix.
- Assert preview and ZIP use the same registered template paths, and that local
  output writes those paths plus only its documented `__init__.py` scaffold
  safety net.
- Assert partial ZIP failure and `X-Skipped-Tables` behavior.
- Run Ruff on generated Python and Web type-check/lint on generated TS/Vue.
- For local output, use a temporary repository root and assert overwrite/menu
  conflict behavior explicitly before changing it.

There are currently no dedicated generator tests under `backend/tests/`; this
is a coverage gap, not permission to change templates without regression tests.

## 7. Wrong vs Correct

Wrong:

```text
Fix the generated controller.py and index.vue only.
```

Correct:

```text
Fix backend/templates/*, update Jinja context/path mappings when needed, then
assert preview == ZIP == local-output paths and regenerate the sample.
```