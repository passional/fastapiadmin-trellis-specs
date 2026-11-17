# 前端产物验证与生产部署

> 来源：`SKILL.md` §9、§13 + `docker/nginx/`、`backend/app/__init__.py`、`deploy.sh`。
> 核心命题：**源码已修 ≠ 部署已修。**

## 1. 为什么必须核到产物

`dist` 是生成物、不入库（`.gitignore`），但仓库内存在**多份预构建副本**并直接被打进部署。改完前端源码若不重建同步，线上仍跑旧行为。

真实教训：AI chat 重复「连接成功」的修复，源码 `e39a369b` 已修，但三份 dist 仍停在修复前，导致漏判一轮。

## 2. 三份产物与各自服务入口

| 产物 | 服务入口 |
| --- | --- |
| `frontend/web/dist` | 构建输出（`pnpm build:prod` 的 outDir，见 `vite.config.ts` `build.outDir`） |
| `docker/nginx/web/dist` | docker 部署的 `/web`（`nginx.conf` 的 `alias /usr/share/nginx/html/web/dist`） |
| `backend/dist` | 一体化部署时由 `app.frontend("/")` 托管（`app/config/path_conf.py` 的 `FRONTEND_DIST_DIR`） |

移动端同理：`docker/nginx/app/dist/build/h5`（nginx 的 `/app`）。

后端挂载代码（`backend/app/__init__.py:125`）：

```python
def register_frontend(app: FastAPI) -> None:
    if path_conf.FRONTEND_DIST_DIR.exists():
        app.frontend("/", directory=str(path_conf.FRONTEND_DIST_DIR), check_dir=False)
```

**检查与挂载必须用同一个 `path_conf.FRONTEND_DIST_DIR`**。曾因检查用 path_conf、挂载硬编码 `frontend/web/dist` 导致 500：`check_dir=False` 时启动不报错，**首次请求才炸**，必须看 loguru 日志（`backend/logs/fastapiadmin.log`）才能定位。

## 3. 重建 + 同步

```bash
cd frontend/web && pnpm build:prod          # 输出 frontend/web/dist
cd ../.. && rsync -a --delete frontend/web/dist/ docker/nginx/web/dist/ \
              && rsync -a --delete frontend/web/dist/ backend/dist/
```

- **必须带 `--delete`**：产物文件名带 content hash，不带会把旧 chunk 留在目录里，浏览器可能仍加载旧文件。
- 只改 Web 生成文件而不同步另外两份 = 违规（同理适用于 design token 生成物，见 `docs/UI-ENGINEERING-SPEC.md` §1.1）。
- docker 部署还要**重新打镜像 / 重传 `docker` 目录**才算生效——属不可逆动作，需当次确认。

## 4. 产物级验证（不能只看 `src/`）

minify 后函数名 / 注释会丢，但**属性访问**（如 `WebSocket.CLOSED`）和**提示文案字符串**会保留，可用作特征：

```bash
# 1) 定位 chunk
grep -l "ai/chat/ws" docker/nginx/web/dist/js/*.js

# 2) 核守卫特征（-oE 重复上限 255，所以用 .{0,60} 而不是 .*）
grep -oE '.{0,60}readyState.{0,60}' <chunk> | grep -i websocket
```

判读示例：
- 旧包：`if (ws?.readyState === WebSocket.OPEN) return;` → `CLOSED` 命中 **0** 次；
- 修复后：`if (ws && ws.readyState !== WebSocket.CLOSED) return;` → 命中。

**收到「源码没改好」的线上反馈时，第一步先核 dist 产物特征字符串，再回源码。**

## 5. 生产部署与备份

部署前必须确认并备份：

1. 后端源码；
2. `docker/nginx/web`；
3. 数据库 dump。

- 备份目录：`/home/FastapiAdmin/backups/release-<时间戳>/`。
- 部署后验证清单：health（`/api/v1/monitor/health/check`）、web（`/web/`）、登录、关键列表、调度器日志。
- 生产 `https://service.fastapiadmin.com`；演示模式（系统参数 `demo_enable`）下写操作 / 导出被拦截是**配置行为**。
- 不把生产凭据、dump、密钥写入仓库。
- 部署 / 重建镜像 / 重启服务器属不可逆动作：先说明影响并取得**当次**确认（见 [collaboration-redlines.md](./collaboration-redlines.md)）。

## 6. 检查清单

改完前端源码准备上线：

- [ ] `pnpm build:prod` 成功（含类型检查则用 `pnpm build`）
- [ ] `rsync -a --delete` 同步到 `docker/nginx/web/dist` 与 `backend/dist`
- [ ] 移动端改动同步 `docker/nginx/app/dist/build/h5`
- [ ] 在**产物**里 grep 到本次修复的特征字符串
- [ ] docker 部署已重新打镜像 / 重传（并取得确认）
- [ ] 部署后 health / web / 登录 / 关键列表 / 调度器日志逐项验过
- [ ] 备份位置已告知用户
