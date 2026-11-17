# 跨端工程规则（project）

> 来源：根目录 `SKILL.md`（仓库作者开发规则）+ 仓库实际脚本/配置。
> 这一层管「不管改哪一端都要遵守」的事：协作红线、开发流程、测试、产物与部署。

## 文件索引

| 文件 | 内容 | 什么时候读 |
| --- | --- | --- |
| [collaboration-redlines.md](./collaboration-redlines.md) | 凭据/不可逆动作红线、登录排障顺序、AI 协作流程与交付口径 | **每个任务开始前**；涉及服务器、部署、master、凭据时必读 |
| [dev-workflow.md](./dev-workflow.md) | 工程地图、常用命令、首次启动、新增业务功能全栈流程、提交规范 | 跑服务、建模块/页面、提交前 |
| [testing-guidelines.md](./testing-guidelines.md) | 用例设计六类覆盖、测试资产、`e2e_` 测试数据规范、执行命令、浏览器自动化 | 写/改测试，验收功能 |
| [release-and-artifacts.md](./release-and-artifacts.md) | 三份 dist 产物与重建同步、产物级验证、生产部署与备份 | 改完前端源码要上线；排查「源码改了线上没变」 |

## 与其它层的关系

- 后端具体写法 → [../backend/index.md](../backend/index.md)
- 前端具体写法 → [../frontend/index.md](../frontend/index.md)
- 跨层数据流怎么想 → [../guides/cross-layer-thinking-guide.md](../guides/cross-layer-thinking-guide.md)

## 本层最高优先级的三条

1. **未经当次明确确认，不动凭据、不做不可逆动作**（部署/重建镜像、合并 master、push、删数据、重启服务器）。
2. **源码已修 ≠ 部署已修**：前端改完必须重建并同步三份 dist，验证要核到产物特征字符串。
3. **「页面能打开」不等于「功能通过」**：写操作要真实提交、真实校验、清理测试数据。
