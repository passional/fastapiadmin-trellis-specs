# 思考指南（guides）

> 目的：在动手前把「没想到」的那部分想清楚。本仓库大多数事故不是不会写代码，而是**漏了某个边界**或**重复造了已有的轮子**。
> 这一层是跨包通用的思考清单，具体写法看 [backend](../backend/index.md) / [frontend](../frontend/index.md) / [project](../project/index.md)。

## 指南索引

| 指南 | 用途 | 什么时候读 |
| --- | --- | --- |
| [code-reuse-thinking-guide.md](./code-reuse-thinking-guide.md) | 先搜再写：Fa 组件 / hooks 套件 / utils / 基建的唯一来源 | 要新建组件、hook、工具函数、常量、模板时 |
| [cross-layer-thinking-guide.md](./cross-layer-thinking-guide.md) | 全栈数据流与边界契约（菜单→路由→缓存、model→响应→表格、src→dist） | 功能跨 3 层以上、改字段/枚举/契约、排查跨端不一致 |

## 触发清单：什么时候该想跨层

- [ ] 功能同时动 `backend/app/modules/**` 与 `frontend/web/src/{api,views}/**`
- [ ] 改了 schema 字段、分页字段、响应信封、业务码（`RET`）
- [ ] 改了枚举语义（状态 0/1、菜单类型 1/2/3/4、公告 0/1/2、data_scope 1/2/3）
- [ ] 新增/改名路由、菜单 `route_name`、页面 `defineOptions({ name })`
- [ ] 改了权限标识（含嵌套子域的四段式）
- [ ] 改了时间/时区/Excel 导入导出
- [ ] 改了端口、代理目标、WS endpoint
- [ ] 改完前端源码准备上线（三份 dist）
- [ ] 页面有 WebSocket / SSE / 定时器 / 全局监听（KeepAlive 生命周期）

→ 读 [cross-layer-thinking-guide.md](./cross-layer-thinking-guide.md)

## 触发清单：什么时候该想复用

- [ ] 要写一个列表页 / 弹窗表单 / 导入导出 / 确认框
- [ ] 要写一个工具函数（格式化、查询参数清洗、权限判断、下载）
- [ ] 要定义常量、状态映射、颜色、字号、间距
- [ ] 要在多个文件里读同一个字段 / 写同一段校验
- [ ] 要改代码生成模板（python / ts / vue 三套）
- [ ] 要新建模块（`modules/` 手工注册 vs `plugin/` 自动发现，两条路径不对称）
- [ ] 要改 Logo / token（多副本同步）

→ 读 [code-reuse-thinking-guide.md](./code-reuse-thinking-guide.md)

## 改动前必做：先搜

> **改任何值之前，先全库搜一遍。** 这一个习惯能挡掉大部分「忘了同步 X」。

```bash
# 例：要改权限标识 / 端口 / 状态语义 / 提示文案
grep -rn "module_system:dict_type" backend/app frontend/web/src backend/sql
grep -rn "8001\|8003" backend/env backend/app/config frontend/web/.env*
grep -rn "连接成功" frontend/web/src            # N 次弹窗 = N 个实例或 N 次重入
grep -rn "keepAliveExclude" frontend/web/src
```

搜索范围要包含**种子数据**（`backend/sql/*.json`）、**生成模板**（`backend/templates/`）、**产物副本**（`docker/nginx/web/dist`、`backend/dist`）——它们不在 `src/` 里，但同样承载契约。

## 核验 AI 交叉评审结果

评审意见（含 AI 评审）**必须对着真实代码核过再排优先级**：

- [ ] 说「用户输入可能恶意」→ 先看数据来源：是外部请求，还是内部种子 JSON / 系统参数 / 已鉴权上下文？
- [ ] 说「缺校验」→ 校验是否已在 Pydantic schema / `AuthPermission` / 全局异常处理器里做了？
- [ ] 说「行为变更是 bug」→ 先读代码注释：本仓库大量守卫是**故意设计**（如 dev 自动迁移的 DROP 拦截、KeepAlive 对 undefined include 不 prune、WS 守卫用 `!== CLOSED`、`uq` 特意不命名）。
- [ ] 说「测试有问题」→ 心里删掉被测功能，测试还会过吗？会过就是同义反复。
- [ ] 说「线上没修好」→ 先核 dist 产物特征字符串，再回源码。

常见误报模式：
1. **信任边界混淆**：把内部种子数据 / 系统参数当外部不可信输入。
2. **忽略设计注释**：把注释里写明的取舍当 bug（本仓库注释密度高，先读注释）。
3. **变量误读**：没追到定义就下结论（如 KeepAlive 按**组件名**匹配，不是路由名；`getData` 回第一页而 `fetchData` 是当前页）。
4. **方言/环境混淆**：拿 SQLite 行为推断 PostgreSQL，或拿本地 `.env.dev` 推断生产。

## 用法

1. 动手前：扫一遍相关指南的触发清单。
2. 编码中：觉得重复或复杂 → 查复用指南；觉得跨端 → 查跨层指南。
3. 修完 bug：把「症状 → 根因 → 防回退规则」补进对应层 spec（走 `trellis-update-spec`），而不是只留在这次对话里。

**核心原则：想清楚 30 分钟，省掉调试 3 小时。**
