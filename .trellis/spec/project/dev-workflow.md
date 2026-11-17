# 工程地图、命令与全栈开发流程

> 来源：`SKILL.md` §1–§2、§5–§6、§10 + 仓库实际配置。
> 事实性数值（端口、路径）以代码为准；与 SKILL.md 不一致处已标注。

## 1. 工程地图

| 目录 | 说明 | 技术栈 |
| --- | --- | --- |
| `backend/` | 后端服务，入口 `main.py` | FastAPI + SQLAlchemy(async) + Alembic + Redis + uv |
| `frontend/web/` | 管理后台（pnpm 包，Node >= 20） | Vue3 + Vite + TS + Element Plus + Tailwind4 + Pinia |
| `frontend/app/` | 小程序 / 移动端，**独立约定** | uni-app + wot-ui（skills 在 `frontend/app/.agents/skills/`） |
| `frontend/docs/` | 文档站，中英双语 `src/guide` 与 `src/en/guide` | VitePress |
| `docker/` | 部署（backend / mysql / nginx / redis） | docker-compose |
| `design/` + `scripts/design-tokens/` | 设计 token 源与生成脚本 | `tokens.json` → 三端生成文件 |
| `docs/` | 工程/品牌规范、覆盖计划 | `UI-ENGINEERING-SPEC.md`、`DESIGN-SPEC.md`、`COVERAGE-PLAN.md` |
| `REQUIRE/` | 业务需求原始材料 | 需求文档 / 流程图 |
| 根目录 | `SKILL.md`（作者开发规则）、`README.md`、`CONTRIBUTING.md`、`deploy.sh` | 做大功能前先读 |

三个前端包各自用 pnpm 管理自己的依赖，**不要在根目录跑 pnpm**。

## 2. 常用命令

后端（`cd backend`，优先 uv）：

```bash
uv sync                                  # 安装依赖
uv run main.py run --env=dev             # 启动（dev 默认；DEBUG=True 自动 reload）
uv run main.py revision -m "说明"         # 生成 Alembic 迁移（autogenerate）
uv run main.py upgrade --env=dev         # 应用迁移到 head
UV_CACHE_DIR=/tmp/uv-cache-test uv run --no-sync pytest -q
ruff check                               # 后端 lint（CONTRIBUTING.md 要求）
```

前端 web（`cd frontend/web`）：

```bash
pnpm i                # 安装
pnpm dev              # 开发
pnpm ts:check         # vue-tsc 类型检查（改完 TS/Vue 必跑）
pnpm lint             # eslint + prettier + stylelint
pnpm test             # vitest run
pnpm build            # vue-tsc --noEmit && vite build
pnpm build:prod       # 仅构建（产物同步用这个，见 release-and-artifacts.md）
pnpm commit           # git-cz 交互式生成提交信息
```

`frontend/app`：`pnpm dev:mp-weixin`（微信开发者工具导入 `dist/dev/mp-weixin`）、`pnpm dev:h5`。
`frontend/docs`：`pnpm dev`。

## 3. 端口与代理（易错，务必两端一致）

| 项 | 位置 | 当前值 |
| --- | --- | --- |
| 后端端口默认 | `backend/app/config/setting.py` `SERVER_PORT` | 8001 |
| 后端端口本地实际 | `backend/env/.env.dev` `SERVER_PORT` | **8003** |
| 前端端口 | `frontend/web/.env` `VITE_PORT` | 5180 |
| 请求前缀 | `frontend/web/.env` `VITE_APP_BASE_API` | `/api/v1` |
| 代理目标 | `frontend/web/.env.development` `VITE_API_BASE_URL` | **http://127.0.0.1:8003** |
| AI WebSocket | `frontend/web/.env.development` `VITE_APP_WS_ENDPOINT` | **ws://localhost:8003**（直连，不走代理） |

- SKILL.md 写「通常 8001」指的是代码默认值；**本地联调以 `.env.dev` 为准**。
- 改后端端口必须同步改前端代理目标与 WS endpoint，否则表现为「接口全 404 / 连不上」而不是报错。
- vite 把 `VITE_APP_BASE_API` 前缀代理到 `VITE_API_BASE_URL`（见 `vite.config.ts`），本地开发无 CORS 问题。

## 4. 首次启动（新人跑通）

1. 复制 `backend/env/.env.example` 为 `.env.dev`，按文件头注释改必改项（`DATABASE_PASSWORD` / `REDIS_PASSWORD` / `OPENAI_API_KEY`）。`DATABASE_TYPE` 支持 `mysql` / `postgres` / `sqlite`——本地体验用 sqlite 可零配置起跑（当前 `.env.dev` 是 mysql）。
2. `uv sync && uv run main.py run --env=dev`：**启动时自动执行迁移并导入 `backend/sql/*.json` 种子数据**（菜单/角色/用户/字典等），无需手动建表导数。dev 环境检测到模型变更还会自动生成并应用迁移（`app/scripts/initialize.py`）。
3. 前端：`cd frontend/web && pnpm install && pnpm dev`。
4. 默认账号：`super` / `admin` / `user`，密码均 `123456`（种子 bcrypt 哈希已校验；部署后应立即修改）。
5. 访问：Web `http://localhost:5180`；后端 Swagger `http://localhost:<SERVER_PORT>/docs`；API 前缀 `/api/v1`。
6. Docker 一键部署：根目录 `./deploy.sh`（详见 `docker/README.md`）。

## 5. 新增一个业务功能（全栈流程）

1. **后端建模块**：参考 `backend/app/modules/system/dict/`；标准 CRUD 表可先用后台「代码生成」可视化产出再调整。
2. **注册路由**：`backend/app/api/v1/routers.py` 的 `DOMAIN_CONTROLLERS`（唯一路由事实来源）。
3. **迁移 + 菜单权限**：`uv run main.py revision -m "..."` + `upgrade`；菜单/按钮权限写入 `sys_menu`（后台「菜单管理」配置 `route_path` / `route_name` / `component_path` / `keep_alive` / 按钮权限）。
4. **前端**：新建 `frontend/web/src/api/module_x/xxx.ts` 与 `views/module_x/` 页面，复用 `useTable` / `FaTable` 套件。
5. **权限标识对齐**：前后端都用 `模块:资源:操作` 三段式，页面按钮加 `v-hasPerm`。
6. **验证**：后端 `pytest`；前端 `pnpm ts:check`、`pnpm lint`、`pnpm test`。
7. **要上线还要重建同步 dist**：见 [release-and-artifacts.md](./release-and-artifacts.md)。

> ⚠️ 生成/修改后端代码后**必须重启后端**：dev reload 只在文件变更时重载，而动态路由发现（`app/core/discover.py`）只在启动时执行一次并缓存。不重启新模块不生效且会静默失败。

跨端需求（web + 小程序）要**同时评估** `frontend/web` 与 `frontend/app` 两套代码，API 层各自维护。

## 6. 提交规范

- husky + commitlint + git-cz：`frontend/web/.husky/pre-commit` 跑 `pnpm run lint:lint-staged`，`commit-msg` 跑 `commitlint --edit`。
- type 用 `feat` / `fix` / `refactor` / `chore` / `docs` / `test` 等；`pnpm commit` 交互式生成。
- lint-staged 会自动格式化暂存文件，**不要绕过 hook**（`--no-verify`）。
- 分支：功能分支 `feature/xxx`、修复 `bugfix/xxx`；贡献 PR 提到 `dev` 分支（`CONTRIBUTING.md`）。合并 `master` 属不可逆动作，需当次确认。

## 7. 本地环境已知边界

- 当前可用的对象存储测试服务只有 **MinIO S3**；FTP/FTPS、SFTP、OBS、OSS、COS 没有测试环境，不作为集成验收项。
- 文档站改动记得**中英两份**（`frontend/docs/src/guide/` 与 `src/en/guide/`）。
- 小程序 UI 遵循 wot-ui 约定（`frontend/app/.agents/skills/wot-ui-v2` 等）。
