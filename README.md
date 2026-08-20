# fastapiadmin-trellis-specs

FastAPI Admin 项目的 Trellis 工程规范模板库。

## 目录结构

spec 位于 `.trellis/spec/`，按层组织：

- `architecture/` — 系统边界、HTTP/Realtime 契约、代码生成与路由
- `backend/` — 目录结构、错误处理、数据库、日志、质量
- `frontend/` — 目录结构、组件、hook、状态管理、类型安全、质量
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
- 输入：

```
gh:passional/fastapiadmin-trellis-specs
```

- 引导会自动下载 `.trellis/spec` 到项目，并把 registry 源写回 `.trellis/config.yaml`

方式二：手动配置（已 init 的项目）

编辑 `.trellis/config.yaml`：

```yaml
registry:
  spec:
    source: https://github.com/passional/fastapiadmin-trellis-specs
```

然后同步：

```bash
trellis update
```

## 维护

修改后推送到本仓库，目标项目执行 `trellis update` 即可获取更新（hash 追踪，本地已改的 spec 不会被覆盖）。