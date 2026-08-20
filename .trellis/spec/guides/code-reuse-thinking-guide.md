# Code Reuse Thinking Guide

Use this checklist before adding a second implementation of an existing idea.

## Search the real owners

- [ ] Is this backend behavior already owned by `backend/app/core/`,
  `backend/app/common/`, or a feature `service.py`/`crud.py`?
- [ ] Is there already a Web hook/API/store under `frontend/web/src/` or an App
  composable/API/store under `frontend/app/src/` with the same responsibility?
- [ ] Is the apparent duplication intentional because Web uses Axios while App
  uses Alova and uni-app APIs?
- [ ] Is this a generated shape owned by `backend/templates/` rather than by a
  generated file below `backend/app/plugin/` or `frontend/web/src/`?
- [ ] Is this value repeated in settings, env examples, Compose, Nginx, and
  frontend build config?

Reference owners:

- HTTP clients: `frontend/web/src/utils/http/index.ts` and
  `frontend/app/src/http/adapters/alova.ts`
- Shared backend response contract: `backend/app/common/response.py`
- Code generator templates and path mapping: `backend/templates/` and
  `backend/app/api/v1/module_generator/gencode/jinja2_template_util.py`
- Runtime configuration: `backend/app/config/setting.py`, `backend/env/`, and
  `docker/docker-compose.yaml`

## Decide deliberately

- [ ] Reuse a feature-local pattern before promoting it to `core`, `common`,
  `utils`, a global hook, or a global component.
- [ ] Keep Web/App implementations separate when platform APIs differ, but
  compare their transport fields and error behavior side by side.
- [ ] When a generated output changes, update its Jinja template and every
  preview/download/local-output mapping; do not patch only one generated file.
- [ ] When a module changes, search its backend router assembly, Web/App API and
  view paths, menu seed/configuration, and permission strings.
- [ ] When an env key changes, search exact spelling across backend settings,
  examples, Compose, deploy scripts, Nginx, and frontend `ImportMetaEnv` types.

## Avoid premature reuse

- [ ] Are two similar functions actually using different response semantics?
  Web returns an Axios response; App's Alova adapter returns business `data`.
- [ ] Is a helper hiding a cross-layer contract that should instead be explicit
  in a schema or type?
- [ ] Would abstraction force imports between the independent
  `frontend/web/`, `frontend/app/`, and `frontend/docs/` packages?

Implementation details belong in the architecture specs linked from
[the guide index](./index.md).

