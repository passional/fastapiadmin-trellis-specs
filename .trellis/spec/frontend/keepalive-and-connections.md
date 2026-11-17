# KeepAlive 缓存与连接类资源（踩过坑，勿回退）

> 来源：`SKILL.md` §7、§4 + 代码现状：`src/layouts/fa-page-content/index.vue`、`src/store/modules/worktab.store.ts`、`src/router/{MenuProcessor,route-loader}.ts`、`src/views/module_ai/chat/index.vue`、`src/hooks/core/useTable.ts`。
> 这一份是**事故复盘沉淀的契约**。改动这里任何一条都要先理解为什么，不要「简化」回去。

## 1. 路由出口缓存只能是单层

唯一 KeepAlive 在 `src/layouts/fa-page-content/index.vue`：

```vue
<KeepAlive :include="keepAliveInclude" :exclude="keepAliveExclude">
  <component :is="Component" :key="routeViewCacheKey(router)" />
</KeepAlive>
```

```ts
function routeViewCacheKey(r: RouteLocationNormalizedLoaded): string {
  return r.path;          // 叶子路由 path；query 变化不新建实例
}
```

硬约束：
- **只能有一层 KeepAlive**。目录路由的 `component` 必须保持 `undefined`（`MenuProcessor.mapMenuNode` / `RouteTransformer.handleNormalRoute`），靠 vue-router 的 RouterView **深度跳级**直达叶子页面。
- **不要给它加 `:max`**：缓存集合已由 include/exclude 精确表达，再叠 LRU 会在标签仍打开时挤掉最早页面，切回时无谓重挂载、请求次数不可预测。
- **KeepAlive 必须常驻**，不能按当前路由 `meta.keepAlive` 做 `v-if` 开关：卸载 KeepAlive 会连同已缓存实例一起销毁，每次切回都重新挂载（接口重复请求）。叶子取舍只由 include/exclude 表达。
- 缓存键是叶子 `path`：同组件不同 path（含动态参数）各自一份实例；**query 变化不重挂载**。要改键逻辑就改这个函数，别在页面里 hack。
- 开发期守卫：该文件 `watch(route.path)` 在 DEV 下检查 `matched[1]` 是否意外带了 `components`——若带了，说明中间层挂了组件，深度跳级失效、include/exclude 全部落空。**不要为了让守卫闭嘴而给目录挂组件。**

历史事故：曾给目录挂壳组件 `NestedRouterParent`，壳实例随缓存同时存活多份、同一页面被重复挂载，切换菜单时接口重复请求。

## 2. include / exclude 的来源与语义

- `include` 来自 worktab 的 `opened`（多标签模式下生效），按**组件名**匹配（不是路由名）。
- **`opened` 为空时 include 为 `undefined`——KeepAlive 对 undefined include 不做 prune**。不要以为「标签清了缓存就没了」。
- KeepAlive 内部缓存**无法从外部直接清空**，唯一手段是改变 `include`/`exclude` 触发内部 prune。
- 登出场景：`worktab.store.ts::clearAll()` 把待删标签的组件名写入 `keepAliveExclude` 驱逐旧实例；下次 `openTab` 的 `removeKeepAliveExclude` 自动移出，恢复正常缓存。**改 store 时勿删这段。**
- include/exclude 判定用 `!== false`，与静态路由显式 `keepAlive: false`、后端布尔字段对齐。
- 组件名解析：`resolveOutletComponentName(path)` 回解 matched 中第一个带 `components` 的后代，取 `comp.name ?? comp.__name`。所以页面 `defineOptions({ name })` 必须等于菜单 `route_name`。

## 3. 有副作用的页面必须写 activated / deactivated

**`onUnmounted` 只在缓存被驱逐时触发，不能作为唯一清理点。**

标准写法（`views/module_ai/chat/index.vue`）：

```ts
onBeforeUnmount(() => {
  window.removeEventListener("resize", syncSidebarForViewport);
});
onUnmounted(disconnectWebSocket);

// KeepAlive 缓存切换：离开视图即断开，回到视图按需重连。
// 缺失 deactivated 时被缓存实例会持续占用 WebSocket（登出后旧实例的连接也不会释放）。
onActivated(() => {
  if (!isConnected.value) connectWebSocket();
});
onDeactivated(disconnectWebSocket);
```

适用资源：WebSocket、SSE、`setInterval`/`setTimeout` 轮询、全局事件监听（window/document）、第三方图表实例、IntersectionObserver 等。

- deactivated：释放/暂停资源；activated：按需恢复（**先判状态再重建**，避免重复连接）。
- `useTable` 已内建这套：`onDeactivated` 取消请求并置 `tableViewActive`，失活期间 `fetchData` 直接返回快照不发 HTTP。**页面不要再叠一层自己的轮询绕过它。**
- 参考实现：`module_ai/chat`、`module_monitor/server`、`module_monitor/cache`。

## 4. WebSocket 守卫必须覆盖握手期

```ts
// ✅ CONNECTING / OPEN / CLOSING 期间一律拒绝重入
if (ws && ws.readyState !== WebSocket.CLOSED) return;

// ❌ 只挡 OPEN：CONNECTING 期间重入会创建新连接并覆盖旧引用，
//    泄漏的连接照样握手成功并弹「连接成功」
if (ws?.readyState === WebSocket.OPEN) return;
```

事故：AI chat 登出→登录后出现多条 ws + 多条「连接成功」提示，根因就是只挡 OPEN。

主动断开：**先摘回调再 close**，避免关闭竞态触发提示或状态回调：

```ts
const disconnectWebSocket = () => {
  if (ws) {
    const socket = ws;
    ws = null;
    socket.onopen = null;
    socket.onmessage = null;
    socket.onerror = null;
    socket.onclose = null;
    socket.close(1000, "用户主动断开");
  }
  isConnected.value = false;
  connectionStatus.value = "disconnected";
  finishLoadingMessages();
};
```

发送前判 `ws?.readyState === WebSocket.OPEN`（这里判 OPEN 是对的——发送需要已建立；**重入守卫**才必须用 `!== CLOSED`）。

## 5. WebSocket 鉴权（AI chat）

- 前端：`new WebSocket(url, ["access_token", `access_token.${token}`])`，token 经 **`Sec-WebSocket-Protocol`** 传递，**不出现在 URL / 服务端 access log**。
- URL：`new URL("/api/v1/ai/chat/ws", VITE_APP_WS_ENDPOINT)`，**直连后端不走 vite 代理**。
- 后端：`app/core/dependencies.py::get_websocket_token` 优先解析子协议里的 `access_token.<jwt>`；小程序等无法自定义子协议的客户端可用 `?token=` 兜底。`websocket_authenticate` 在握手阶段校验，失败抛 `CustomException`，并返回需回显的子协议。
- 改鉴权方式要两端同步，且保持「token 不进 URL」这条。

## 6. 排查「重复弹窗 / 重复连接 / 重复请求」的固定路径

1. **先 grep 提示文案**定位全库唯一来源（N 次弹窗 = N 个实例或 N 次重入）。
2. 查 KeepAlive `include`/`exclude` 计算，以及**登出→登录导航链**（登录守卫 404 → replace 重定向会叠加竞态窗口）。
3. 查菜单配置 `backend/sql/sys_menu.json`：确认 `route_name` 唯一、`keep_alive` 正确，排除后端因素。
4. 查页面是否缺 `onDeactivated`、WS 守卫是否只挡 OPEN。
5. **线上反馈还要核 dist 产物**（源码修了不代表三份 dist 修了）：见 [../project/release-and-artifacts.md](../project/release-and-artifacts.md)。

## 7. 反模式速查

| 反模式 | 后果 |
| --- | --- |
| 给路由出口 KeepAlive 加 `:max` | 标签还开着就被 LRU 挤掉，切回重挂载 |
| 按 `meta.keepAlive` 给 KeepAlive 加 `v-if` | 卸载即销毁全部缓存实例 |
| 目录路由挂壳组件 | 多份壳实例存活，页面重复挂载、接口重复请求 |
| 第二层 KeepAlive / 页面内再包一层 | 缓存语义混乱，include/exclude 落空 |
| 以为清空标签就清空缓存 | undefined include 不 prune |
| 删 `clearAll()` 写 exclude 的逻辑 | 登出后旧实例无法驱逐，连接泄漏 |
| 只用 `onUnmounted` 清理 ws/定时器 | 缓存期间不触发，资源一直占用 |
| WS 守卫只挡 `OPEN` | CONNECTING 期重入 → 连接泄漏 + 重复提示 |
| close 前不摘回调 | 关闭竞态触发「连接失败」等误导提示 |
| token 拼进 WS URL | 进 access log，凭据泄露 |
| 页面 `name` 与菜单 `route_name` 不一致 | 缓存/排除失灵 |
