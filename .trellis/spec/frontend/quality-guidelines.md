# 前端质量规范与验收

> 事实来源：`frontend/web/package.json`、`vitest.config.ts`、`eslint.config.mjs`、`.husky/*`、`docs/UI-ENGINEERING-SPEC.md` §9/§10、`scripts/check-brand-consistency.mjs`。

## 1. 验收命令（改完必跑）

```bash
cd frontend/web
pnpm ts:check        # vue-tsc --noEmit --skipLibCheck（改完 TS/Vue 必跑）
pnpm type-check      # vue-tsc --noEmit（UI spec 评审要求单独跑，便于定位）
pnpm test            # vitest run
pnpm build           # vue-tsc --noEmit && vite build
pnpm lint            # eslint --fix + prettier --write + stylelint --fix（会写文件）
```

口径（`UI-ENGINEERING-SPEC.md` §10）：
- `pnpm build` 已含 `vue-tsc --noEmit`，但**评审仍要求单独跑 type-check**。
- `pnpm lint` 会写文件，提交前按需执行，**不要把它当作唯一验证**。
- 上线构建用 `pnpm build:prod`，然后同步三份 dist（见 [../project/release-and-artifacts.md](../project/release-and-artifacts.md)）。

改了设计 token，先在**仓库根目录**执行：

```bash
node scripts/design-tokens/generate.mjs      # 生成三端 token 文件，检查 diff
node scripts/check-brand-consistency.mjs     # Logo 哈希一致、无旧色值、无旧间距类
```

`check-brand-consistency.mjs` 会比对 `design/assets/logo.{svg,png}`、`favicon.ico` 与 web/app/docs/backend 各副本的 sha256——**Logo 改动必须同步所有副本**，只改一端会红。

## 2. 测试

- 框架 vitest（`vitest.config.ts`），用例在 `src/__tests__/*.spec.ts`（15 个）：
  `route-invariants`（路由不变式）、`http-error-message`、`message-dedupe`（消息合并）、`permission-check`、`query-params`、`fa-table-error`（表格错误态）、`form-validators`、`column-i18n`、`responsive-span`、`data-url` 等。
- 改到对应契约（路由/权限/消息/表格状态/查询参数/i18n/响应式）**必须补或改这些 spec**，不要只靠手点。
- 前端行覆盖未启用 all-source，**不能当全站覆盖率**（见 `docs/COVERAGE-PLAN.md`）。
- UI 验收要真实交互：空值校验、成功创建、按钮/弹窗/富文本/下拉；写操作真实提交并清理 `e2e_` 数据（见 [../project/testing-guidelines.md](../project/testing-guidelines.md)）。
- 浏览器自动化用隔离临时 profile，按字段名定位，EP 下拉/富文本用真实鼠标点击。

## 3. 评审清单（UI spec §9，PR 必过）

- [ ] 颜色/字号/间距/圆角来自 token 或语义 class，无新增裸十六进制 / `text-[Npx]`
- [ ] 列表有 loading / empty / error+retry；`error` 已从 `useTable` 接到 `FaTable`
- [ ] 操作列 `operation` / `fixed=right` / `center` / `nowrap`，使用共享操作单元格
- [ ] 表单 2 列或明确全宽；无 placeholder-only 字段；无 label 时有 `aria-label`
- [ ] 弹窗无 body 滚动条；长内容内部滚动；底部按钮固定
- [ ] 危险操作二次确认；批量操作有 loading 和结果反馈
- [ ] 图标按钮有可访问名；可点击 div 有键盘支持
- [ ] **1280 / 800、浅色/深色实际检查**；无页面横向滚动
- [ ] 新文案同时补 zh/en；状态语义与后端一致
- [ ] `pnpm type-check`、`pnpm test`、`pnpm build` 通过

额外（本仓库特有）：
- [ ] `defineOptions({ name })` 与菜单 `route_name` 一致
- [ ] 有副作用页面实现了 `onActivated`/`onDeactivated`
- [ ] 权限标识与后端 `AuthPermission` + `sys_menu` 逐字一致
- [ ] 改 store 没删 `worktab.clearAll()` 的 `keepAliveExclude` 写入
- [ ] 要上线：三份 dist 已重建同步，并在产物里核到特征字符串

## 4. 禁止模式

| 禁止 | 原因 / 正确做法 |
| --- | --- |
| 手写 `ElTable`/`ElDialog` 堆 CRUD 页 | 用 `useTable`/`FaTable`/`useCrudForm`/`FaDialog` 套件 |
| 裸 hex（`#409eff` 等）、`text-[14px]`、随手间距 | token / 语义 class（主题可切换 7 套） |
| 手改 `_brand-tokens.generated.scss`、`auto-imports.d.ts`、`components.d.ts` | 都是生成物；改源 + 跑生成脚本 |
| 只补 `zh.json` | zh/en 必须同步 |
| `catch {}` 静默失败 / `console.error` 当用户反馈 | 状态闭环；生产构建 drop console |
| 失败后清空数据显示「暂无数据」 | 保留旧数据 + 错误态 + 重试 |
| 给 KeepAlive 加 `:max` / 第二层缓存 / 目录挂壳组件 | 见 [keepalive-and-connections.md](./keepalive-and-connections.md) |
| WS 守卫只挡 `OPEN`；close 前不摘回调 | 连接泄漏 + 误导提示 |
| 只用 `onUnmounted` 清理连接/定时器 | 缓存期间不触发 |
| 页面再弹一次成功/错误提示 | 拦截器 + `utils/message.ts` 已处理合并 |
| 绕过 husky / lint-staged（`--no-verify`） | 风格漂移，commitlint 失效 |
| 只改 `src/` 就认为线上修好 | 必须重建同步 dist 并核产物 |

## 5. 提交规范

- `frontend/web/.husky/pre-commit` → `pnpm run lint:lint-staged`（eslint/prettier/stylelint 按文件类型分工，见 `package.json` 的 `lint-staged`）。
- `commit-msg` → `commitlint --edit`（`commitlint.config.cjs`）；type 用 feat/fix/refactor/chore/docs/test 等。
- 交互式提交：`pnpm commit`（git-cz / cz-git）。
- 分支 `feature/xxx` / `bugfix/xxx`；贡献 PR 提到 `dev`；合并 `master` 需当次确认。

## 6. 已知历史债务

`docs/UI-ENGINEERING-SPEC.md` §11 列了「不在本次改动中伪装完成」的历史债务。改旧页面时**至少让本次改动的区域符合规范**，不做无关的全页重写，也不要把未完成的债务标成已完成。
