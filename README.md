# fastapiadmin-trellis-specs

FastAPI Admin 项目的 Trellis 工程规范模板库。

## 目录结构

spec 位于 `.trellis/spec/`，按层组织（24 份）：

- `index.md` — 总入口：权威来源排序、层路由、命令速查、已知漂移
- `project/` — 跨端工程规则：协作红线、开发流程与命令、测试规范、dist 产物与部署备份
- `backend/` — 目录结构、接口与分层契约、数据库、错误处理、日志、质量
- `frontend/` — 目录结构、API 与类型、组件、CRUD 套件、状态与路由、KeepAlive 与连接、质量
- `guides/` — 跨层 / 代码复用思考指南

## 本次刷新（2026-11-17）

- **来源**：`passional/xian-news`（FastapiAdmin 派生仓库）commit `2e477fe8`。以该仓库根目录 `SKILL.md` 为最高权威，全部规则对照真实代码核验；前端工程契约引用 `docs/UI-ENGINEERING-SPEC.md` 与 `docs/DESIGN-SPEC.md`，不复制成两套说法。
- **结构由旧 7 层收敛为 4 层**：旧 `architecture/`、`operations/`、`security/`、`testing/` 四层（15 份）**未合并进新树**，完整保留在分支 `archive/pre-overwrite-2026-11-17`（commit `5f686a4d`），需要时从该分支取回。
- 文件对应关系：旧 `frontend/{hook-guidelines,state-management,type-safety,routing-and-caching}.md` → 新 `frontend/{crud-hooks,state-and-routing,api-and-types,keepalive-and-connections}.md`；新增 `backend/api-conventions.md` 与 `project/` 整层。
- ⚠️ **部分事实是派生仓库的本地口径**（后端端口 8003 而非默认 8001、覆盖文档为 `docs/COVERAGE-PLAN.md`、演示模式是系统参数 `demo_enable`、`data_scope` 实现为 1/2/3、`plugin.toml` 在域根、权限标识存在四段式）。spec 内已逐条标注漂移；导入其它 FastapiAdmin 项目时请按该项目代码核对，不要直接当通用默认值。
- **下游同步**：层分类变化较大，建议先看 diff 再 `trellis update`；本地已改的 spec 会进冲突流程，要强制对齐用 `trellis update -f`。

## 用法（在目标 FastAPI Admin 项目导入）

方式一：`trellis init` 引导

```bash
trellis init
```

- 在 spec template 选择器里选 `custom (enter a registry source)`
- 输入（**注意必须带 `/.trellis/spec` 子目录**）：

```
gh:passional/fastapiadmin-trellis-specs/.trellis/spec
```

- 引导会自动下载 spec 到项目 `.trellis/spec/`，并把 registry 源写回 `.trellis/config.yaml`

> 为什么带子目录：`trellis update` 会把 source 指向的**目录内容**拷入项目 `.trellis/spec/`。指到仓库根会把 README、`.trellis/` 一起搬进去，产生 `.trellis/spec/.trellis/spec/` 双层嵌套。

方式二：手动配置（已 init 的项目）

编辑 `.trellis/config.yaml`：

```yaml
registry:
  spec:
    source: gh:passional/fastapiadmin-trellis-specs/.trellis/spec
```

然后同步：

```bash
trellis update
```

## 维护

修改后推送到本仓库，目标项目执行 `trellis update` 即可获取更新（hash 追踪，本地已改的 spec 不会被静默覆盖，会进入冲突流程）。

- 想让目标项目对齐 registry 最新版（本地改动可丢弃时用）：`trellis update -f` 强制覆盖并重建 hash 基线。
- 反向工作流（先在目标项目改 spec、再回流本仓库）：把项目 `.trellis/spec/` 拷回本仓库提交 push，然后在项目里跑一次 `trellis update`。若新增了 spec 文件，注意 update 会跳过"本地已存在且与 registry 内容一致"的文件而不登记 hash——删掉该本地文件再跑一次 update，让它走新文件写入路径完成登记。