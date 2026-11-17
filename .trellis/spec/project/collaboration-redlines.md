# 协作红线与 AI 协作流程

> 来源：`SKILL.md` §0（优先于后续所有章节）、§13、§15。
> 这一份是**硬约束**，不是建议。违反红线的改动即使功能正确也不合格。

## 1. 凭据与登录方式

- **未经当次明确确认**，不得：
  - 修改服务器密码；
  - 切换 / 关闭 / 开启 SSH 认证方式；
  - 轮换任何凭据：root 口令、数据库 / Redis 口令、`SECRET_KEY`、`DATA_ENCRYPTION_KEY`、第三方 Key。
- 确需变更时，必须在**同一条消息**里写清六件事：
  1. 改了什么；
  2. 旧凭据何时失效；
  3. 新凭据如何获得；
  4. 用户下次登录步骤；
  5. 回滚办法；
  6. 备份位置。
- 绝不把口令、私钥、可复用登录信息写进仓库、文档、提交信息或任何长期文件。
- 服务器具体登录信息只保留在本地受控位置，**不进入 SKILL.md / 仓库 / spec**。

代码侧对应约束：
- 本地配置放 `backend/env/.env.dev`（该文件不提交真实密码/Key；模板见 `backend/env/.env.example`）。
- 操作日志已自动脱敏：`app/core/router_class.py` 的 `_SENSITIVE_KEYS` 覆盖 password / token / api_key / secret / authorization 等，请求体与响应体都过 `_redact_sensitive`。**新增凭据类字段名时要同步进这个集合**，否则明文入库。
- 测试固定注入假密钥（`backend/tests/conftest.py`：`SECRET_KEY="unit-test-secret-key-not-a-real-secret"`），不要为了跑测试把真实 Key 写进测试文件。

## 2. 不可逆动作（一律先说明影响 + 取得当次确认）

- 部署 / 重建镜像
- 合并 `master`
- `git push`
- 删除服务器数据或团队
- 重启服务器
- 清理历史提交

确认是**当次**的：上一轮批准过部署，不代表这一轮可以直接部署。

## 3. 登录 / 连接排障顺序

按序执行，不要跳步：

1. 先用可用凭据**实测能否连接**，区分「服务不可用」和「客户端凭据缺失」；
2. 再查服务端对应来源 IP 的认证日志；
3. 再查客户端认证方式、密钥路径、agent 身份；
4. **不要急着重启服务器**——先证明重启能解决什么。

## 4. AI 协作流程（SKILL.md §15）

1. 先读 `SKILL.md` 与覆盖文档（实际文件是 `docs/COVERAGE-PLAN.md`，SKILL.md 里写的 `docs/TEST-COVERAGE.md` 不存在），确认现有覆盖和已知缺口；
2. 修改前先查现有组件 / 套件，避免重复造轮子（见 [../guides/code-reuse-thinking-guide.md](../guides/code-reuse-thinking-guide.md)）；
3. 改完执行对应验证，**至少「单元/接口 + 类型/构建」两类**；
4. 涉及生产写操作、部署、master、凭据时先获得当次明确确认；
5. 不把「页面能打开」当成「功能通过」；写操作要真实提交并清理测试数据；
6. 交付时说明三件事：**改了什么、验证了什么、未覆盖什么**。

## 5. 生产环境注意

- 生产地址 `https://service.fastapiadmin.com`，健康检查 `/api/v1/monitor/health/check`，Web `/web/`。
- 演示模式由系统参数 `demo_enable` 控制（后台「参数管理」），`RequestLogMiddleware` 会拦截非 GET 写操作并返回「演示环境，禁止操作」。**这是配置行为，不是代码故障**；测写接口前先确认环境。
- 部署前必须确认并备份：后端源码、`docker/nginx/web`、数据库 dump；备份目录 `/home/FastapiAdmin/backups/release-<时间戳>/`。
- 不把生产凭据、dump、密钥写入仓库。

详见 [release-and-artifacts.md](./release-and-artifacts.md)。

## 反模式

| 反模式 | 为什么不行 |
| --- | --- |
| 「顺手」把 SSH 改成密码登录方便调试 | 属凭据/认证方式变更，必须当次确认 + 六件事说明 |
| 把服务器口令写进 `.env.example` 或 spec 方便下次用 | 凭据进仓库 = 泄露，且会被提交历史永久保留 |
| 连接失败直接重启服务器 | 未证明重启能解决什么；可能掩盖凭据缺失真因 |
| 上一轮批准过 push，这一轮直接 push | 确认是当次的，不跨轮延续 |
| 演示环境写接口返回「禁止操作」就改代码绕过 | 那是 `demo_enable` 配置行为，改代码属于误判 |
