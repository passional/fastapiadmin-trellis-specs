# Code Generation and Routing

## 1. Scope / Trigger

Use this contract when changing generator schemas, Jinja templates, generated
paths, permissions, menu creation, preview/ZIP/local output, or the plugin route
loader.

The generator targets the backend plugin tree and Admin Web only. It does not
generate uni-app or docs code.

## 2. Signatures

Important API signatures from
`backend/app/api/v1/module_generator/gencode/controller.py`:

```text
PUT   /generator/gencode/update/{table_id}   GenTableSchema
GET   /generator/gencode/preview/{table_id}  -> path-to-content map
PATCH /generator/gencode/batch/output        list[str] -> code.zip
POST  /generator/gencode/output/{table_name} -> local files + menu records
POST  /generator/gencode/sync_db/{table_name}
```

`GenTableSchema` owns `table_name`, `class_name`, `package_name`,
`module_name`, `business_name`, `function_name`, optional main/sub-table
fields, `parent_menu_id`, and `columns`. Its validators normalize package names
to `module_xxx`, remove that prefix from `module_name`, and normalize
multi-segment `business_name`. See
`backend/app/api/v1/module_generator/gencode/schema.py`.

## 3. Contracts

### Templates and outputs

`Jinja2TemplateUtil.get_template_list()` is the output set:

| Template | Output |
|---|---|
| `python/{controller,service,crud,schema,model,__init__}.py.jinja2` | `backend/app/plugin/{package_name}/{module_name}/...` |
| `ts/api.ts.jinja2` | `frontend/web/src/api/{package_name}/{module_name}.ts` |
| `vue/index.vue.jinja2` | `frontend/web/src/views/{package_name}/{module_name}/index.vue` |

Main/sub-table generation adds only model, schema, and `__init__` for the child.
The template list, mapping, and context live together in
`backend/app/api/v1/module_generator/gencode/jinja2_template_util.py`; the
actual sources live in `backend/templates/`.

Local output has one additional scaffolding behavior: after each template
write it creates `backend/app/plugin/{package_name}/__init__.py` when missing.
That package marker is not part of preview or batch ZIP output; only the
feature-level `__init__.py` comes from the registered template set. Account for
this current output-set difference when testing or using a ZIP for a brand-new
package.

Change the Jinja source first. A patch to a previously generated plugin/page
does not update preview, ZIP, or the next generated feature.

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

The `generate_code` docstring still mentions an extra `business_path` in route
and component paths, while the executable code uses only package + module.
Treat the executable mapping above as current behavior. Do not reproduce the
stale comment in new code.

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

Evidence: `backend/app/api/v1/module_generator/gencode/service.py` and
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

## 5. Good / Base / Bad Cases

- Good: update a template, context/type mapping, preview expectations, ZIP
  paths, local-output paths, menu/component/permission assertions, then
  generate a representative slice.
- Base: change template formatting without changing fields or paths; render
  every template and run backend/Web format/type checks.
- Bad: edit only `frontend/web/src/views/module_xxx/...`, or add a template
  without registering it in both the template list and path mapping.

## 6. Tests Required

- Validate `GenTableSchema` normalization for package/module/business names and
  invalid main/sub configurations.
- Render all templates for a representative table and assert the exact output
  file set, API prefix, component path, and permission prefix.
- Assert preview and ZIP use the same registered template paths, and that local
  output writes those paths plus only its documented package `__init__.py`
  scaffold.
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
