# FastapiAdmin 项目规范总入口

> 本目录是 `.trellis/spec/`，供 `trellis-implement` / `trellis-check` 子代理按任务 jsonl 清单自动加载。
> 写代码前先读对应层的 spec；跨端功能还要读 [guides/](./guides/index.md)。

## 权威来源与冲突处理

| 优先级 | 来源 | 管什么 |
| --- | --- | --- |
| 1 | 根目录 [`SKILL.md`](../../SKILL.md) | **仓库作者的开发规则**（协作红线、工程约定、踩坑记录）。规则性条款以它为准 |
| 2 | [`docs/UI-ENGINEERING-SPEC.md`](../../docs/UI-ENGINEERING-SPEC.md) | 前端工程契约（token、状态闭环、布局、组件用法），评审以它判合格 |
| 3 | [`docs/DESIGN-SPEC.md`](../../docs/DESIGN-SPEC.md) + [`design/tokens.json`](../../design/tokens.json) | 品牌与视觉唯一来源 |
| 4 | 本目录 `.trellis/spec/` | 面向 AI 的可执行摘要 + 代码实例索引 |
| 5 | 代码现状 | **事实**（端口、路径、字段名、默认值）以代码为准 |

冲突处理：
- spec 与 SKILL.md 的**规则**冲突 → 以 SKILL.md 为准，回改 spec。
- spec / SKILL.md 与代码的**事实**冲突 → 以代码为准，在 spec 标注漂移并提醒回改 SKILL.md。
- `UI-ENGINEERING-SPEC.md` 与 `DESIGN-SPEC.md` 冲突 → 工程实现以前者为准，并回改后者，不保留两套说法。

## 层路由

| 层 | 入口 | 什么时候读 |
| --- | --- | --- |
| 跨端工程规则 | [project/index.md](./project/index.md) | 任何任务开始前（红线、命令、全栈流程、测试、产物/部署） |
| 后端 | [backend/index.md](./backend/index.md) | 改 `backend/`（FastAPI + SQLAlchemy + Alembic + Redis） |
| 前端 | [frontend/index.md](./frontend/index.md) | 改 `frontend/web`（Vue3 + Element Plus + Tailwind4 + Pinia）；`frontend/app`、`frontend/docs` 见该层说明 |
| 思考指南 | [guides/index.md](./guides/index.md) | 跨层数据流、复用判断、AI 评审结果核验 |

## 命令速查

```bash
# 后端（cd backend，优先 uv）
uv run main.py run --env=dev            # 启动；端口取 env/.env.dev 的 SERVER_PORT
uv run main.py revision -m "说明"        # 生成 Alembic 迁移
uv run main.py upgrade --env=dev        # 应用到 head
UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q

# 前端 web（cd frontend/web）
pnpm dev / pnpm ts:check / pnpm lint / pnpm test / pnpm build:prod
```

细节与首次启动步骤见 [project/dev-workflow.md](./project/dev-workflow.md)。

## 当前已知漂移（事实以代码为准）

| SKILL.md 说法 | 代码现状 | 影响 |
| --- | --- | --- |
| 后端端口「通常 8001」 | `backend/app/config/setting.py` 默认 `SERVER_PORT=8001`，但本地 `backend/env/.env.dev` 设为 **8003**；`frontend/web/.env.development` 代理目标也是 **8003** | 本地联调以 `.env.dev` 为准，改端口必须两端同步 |
| `docs/TEST-COVERAGE.md` | 该文件不存在，实际是 [`docs/COVERAGE-PLAN.md`](../../docs/COVERAGE-PLAN.md) | 查覆盖矩阵/缺口读 COVERAGE-PLAN.md |
| 「生产 `DEMO_ENABLE=true`」像是环境变量 | 实为**系统参数** `demo_enable`（`SysParamKey`，存 Redis/参数管理），由 `app/core/middlewares.py::RequestLogMiddleware` 拦截非 GET 请求 | 演示模式开关在后台「参数管理」，不在 `.env` |

## 维护约定

- 每条规则都要能指向真实文件或可复现命令；不写放之四海皆准的空话。
- 踩坑修复后，把「症状 → 根因 → 防回退规则」补进对应层 spec（走 `trellis-update-spec`）。
- 不留模板占位符与空标题；每条规则都要能落到真实文件或可复现命令。
