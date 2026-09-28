# fastapiadmin-trellis-specs

FastAPI Admin 项目的 Trellis 工程规范模板库。

## 目录结构

spec 位于 `.trellis/spec/`，按层组织：

- `architecture/` — 系统边界、HTTP/Realtime 契约、代码生成与路由
- `backend/` — 目录结构、错误处理、数据库、日志、质量
- `frontend/` — 目录结构、组件、hook、状态管理、类型安全、路由与缓存、质量
- `guides/` — 跨层 / 代码复用思考指南
- `operations/` — 配置与启动、部署拓扑
- `security/` — 认证与密钥、授权与数据范围、非可信内容与审计
- `testing/` — 后端 / 前端 / 契约与部署测试

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