# 组件与页面规范

> 事实来源：`src/components/` 实际清单、`src/views/module_system/{position,dict}/index.vue`、`docs/UI-ENGINEERING-SPEC.md`（工程契约，评审以它判合格）。
> **写页面前先找现成组件**，不要手写 ElTable/ElDialog 堆业务页。

## 1. 公共组件清单（FaXxx，自动注册）

| 分类 | 组件 |
| --- | --- |
| forms | `FaForm`、`FaSearchBar`、`FaSelectV2`、`FaCascader`、`FaAutocomplete`、`FaInputTag`、`FaTimeSelect`、`FaSlider`、`FaRate`、`FaMention`、`FaWangEditor`、`FaFileUpload`、`FaExcelImport`、`FaExcelExport`、`FaButtonTable`、`FaButtonMore`、`FaFilterCard`、`FaDragVerify`、`FaPuzzleCaptcha` |
| tables | `FaTable`、`FaTableV2`、`FaTableHeader`、`FaTableHeaderLeft`、`FaPagination`、`FaTableSelect`、`FaDataTableCard` |
| modal | `FaDialog`、`FaDrawer`、`FaImportDialog`、`FaExportDialog` |
| display | `FaDescriptions`、`FaStatusTag`、`FaEmptyState`、`FaPageHeader`、`FaStatistic`、`FaResultPage`、`FaJsonPretty`、`FaMarkdownRenderer`、`FaCalendar`、`FaCarousel` |
| containers | `FaWorkbench`、`FaSidePanel`（主从工作区） |
| cards / charts | `FaStatCard`、`FaStatsCard`、`FaCardGrid`、`FaCardList`、`FaDataListCard`、`FaLineChartCard`、`FaBarChartCard`、`FaDonutChartCard`、`FaProgressCard`、`FaImageCard`、`FaTimelineListCard` |
| actions / navigation | `FaCopyButton`、`FaIconButton`、`FaUpload`、`FaSizeSelect`、`FaAnchor`、`FaGuide`、`FaIntervalTab`、`FaMenuTreeTable` 等 |
| 其它 | `FaSvgIcon`（图标）、exception / skeleton / banners / media / text-effect / base / others |

组件目录形态：`components/<分类>/fa-xxx/index.vue`（部分带 `types.ts` / `composables/`）。
页面私有子组件放**同目录 `components/`**（如 `views/module_system/dict/components/DictDataPanel.vue`），也会被自动注册；新建优先 `FaXxx` 命名。

## 2. 页面骨架

列表页标准结构（`UI-ENGINEERING-SPEC.md` §2.2，样板 `views/module_system/position/index.vue`）：

```vue
<template>
  <div class="fa-full-height">
    <FaSearchBar v-model="searchForm" :items="searchItems" include-audit
                 @search="..." @reset="..." />
    <ElCard class="fa-table-card">
      <FaTableHeader v-model:columns="columnChecks" v-model:showSearchBar="showSearchBar"
                     :loading="loading" @refresh="refreshData">
        <template #left>
          <FaTableHeaderLeft :remove-ids="selectedIds"
            :perm-create="['module_system:position:create']"
            :perm-export="['module_system:position:export']"
            :perm-delete="['module_system:position:delete']"
            :perm-patch="['module_system:position:patch']"
            @add="handleAdd" @export="openExport" @delete="handleBatchDelete" />
        </template>
      </FaTableHeader>
      <FaTable :loading="loading" :error="error" :data="data" :columns="columns"
               :pagination="pagination" @retry="getData"
               @selection-change="onTableSelectionChange"
               @pagination:size-change="handleSizeChange"
               @pagination:current-change="handleCurrentChange" />
    </ElCard>
    <FaDialog v-model="dialogVisible.visible" :title="dialogVisible.title"
              :form-mode="dialogVisible.type" :confirm-loading="submitLoading"
              @confirm="handleSubmit()" @cancel="handleCloseDialog" @close="handleCloseDialog">
      <FaForm ref="dataFormRef" v-model="formData" :items="formItems" :rules="rules" ... />
    </FaDialog>
  </div>
</template>

<script setup lang="ts">
defineOptions({ name: "Position", inheritAttrs: false });   // ★ name 必须等于菜单 route_name
</script>
```

要点：
- 工具栏与表格同宽；分页在卡片底部右侧；**新增是唯一主按钮**（solid primary），导入/导出/批删/更多用 plain/默认。
- 搜索栏高级项默认收起，`Ctrl/Cmd + Enter` 触发搜索。
- **`useTable()` 的 `error` 必须接到 `FaTable` 并提供 `@retry`**。
- 主从工作区用 `FaWorkbench` + `FaSidePanel`（样板 `dict/index.vue`）：左右两张独立卡片、间距 **16px**、**固定宽度不提供拖拽**；左栏宽度按页面配置（字典 380 / 用户 260 / 参数 240 / 文件浏览 216 / SQL 320 / SSH 320）；`≤1024px` 自动上下堆叠（`side-height` 默认 320）；右侧 `min-width: 0`，禁止页面级横向滚动。
  > 旧版 DESIGN-SPEC 的「12px 热区/可拖拽」已废弃，不要按旧描述新增拖拽。
- 详情展示用 `FaDescriptions`（`FaDialog` 的 `form-mode="detail"` 分支），不要另写一套只读表单。

## 3. 表格契约（UI spec §3）

- 列配置集中在 `columnsFactory`，**不在模板里堆 `ElTableColumn`**。
- 选择列宽 48、序号列 56；时间列 `YYYY-MM-DD` 120 / `YYYY-MM-DD HH:mm` 150，带秒只用于详情/日志。
- 文本列必须 `showOverflowTooltip` 或 `minWidth`。
- 状态列统一 `resolveStatusColumns` / `FaStatusTag`，同一状态全站同色同文案。
- 操作列：`prop="operation"`、`label="操作"`、`fixed="right"`、`align="center"`、`class-name="operation-column"`，用 `renderTableOperationCell` 渲染；默认宽 180–200（单图标紧凑页 120–160，同页统一）；超过 2 个次操作折叠进 `FaButtonMore`；**删除永远单独可见且 `type="danger"`**。
- `ElTable` 没有 `loading` prop——不要把 `:loading` 写到 `ElTable`/`ElCard`，用 `FaTable` 的 `loading` 或 `v-loading`。
- 移动端（`≤800px`）优先 `FaTable` 的 `mobile-cards`；自定义卡片列表**必须自己补错误态与重试**。

## 4. 状态闭环（UI spec §5，每个异步区域至少覆盖）

| 状态 | 落点 |
| --- | --- |
| loading | 首屏骨架（`FaTable` 内部 `loading && isEmpty`）；局部 `v-loading`；按钮 loading |
| empty | `FaEmptyState`：图标 + 标题 + 一句说明 + 可选主操作；说明「为什么空/下一步」，不要只写「暂无数据」 |
| error | 持久错误态 + 后端 message + 重试按钮，`role="alert"`；toast 只做瞬时补充 |
| success | toast 或状态标签，不弹阻塞对话框 |
| permission | 无权限隐藏入口或给说明，不静默 403 |

- 刷新时保留旧数据 + 表格 loading，不闪空；失败时若已有旧数据，保留旧数据并 toast，**不要清空成「暂无数据」**。
- 禁止：`catch {}` 不提示、失败后显示 0/「-」/「不存在」、失败后仍渲染空状态、用 `console.error` 代替用户反馈（生产构建还会 drop console）。

## 5. 样式与 token（UI spec §1）

单一来源链：`design/tokens.json` →（`node scripts/design-tokens/generate.mjs`）→ `src/styles/core/_brand-tokens.generated.scss`（**禁止手改**）+ app/docs 对应生成文件；Web 语义补充只放 `src/styles/core/_tokens.scss`；工具类映射在 `src/styles/tailwind.css` 的 `@theme`。

使用优先级：语义 class（`text-g-700`、`bg-(--default-box-color)`、`border-(--default-border)`）> 语义变量（`var(--fa-text-1)`、`var(--fa-radius-md)`）> Element Plus 变量（`var(--el-color-primary)`）> 原始色值（**仅 Logo/插画/图表系列色**，业务 UI 禁止）。

- 主题色可切换（7 套），**禁止把 `#409eff`/`#909399`/`#f56c6c`/`#e6a23c` 写进模板**。
- 字号档位 11/12/13/14/15/18/22/28：正文 14（`text-md`）、紧凑表格 13（`text-base`）、辅助 12（`text-sm`）、微标签 11（`text-xs`）。不要 `text-[14px]`；`text-[Npx]` 只允许一次性图表/画布并注明原因。
- 间距 4/8/12/16/20/24（`gap-1..6`/`p-*`），禁止 5/7/9/10/14/18px 随手值。
- 圆角 6/8/12/16/pill：卡片 12、输入/按钮 8、小标签 6。
- 阴影只用 `--fa-shadow-card/pop/modal`，不叠自定义阴影。
- Element Plus 内部色值只通过 `--el-*` 主题变量覆盖，不在业务页写 EP 内部色。
- **改 token 必须跑生成脚本并检查三端 diff**；只改 Web 生成文件属违规。

## 6. i18n 与图标

- 文案走 `$t('key')`（模板）/ `$t(...)`（脚本，来自 `@/locales`），**同时补 `locales/langs/zh.json` 与 `en.json`**，键一致；禁止只补一端。
- 状态文案集中映射（公告 0 草稿 / 1 已发布 / 2 已归档），页面、移动端、详情、筛选共用语义。
- 日期/数字/文件大小用共享格式化函数（`formatDateTime` 等，`@utils`），不在模板手拼。
- 确认文案要说明对象与后果：「确定将选中的 3 条公告转为草稿吗？」，不用「确定吗？」。
- 图标：`FaSvgIcon` + iconify（`ri:` / `ep:` / `line-md:`），如 `<FaSvgIcon icon="ri:more-2-fill" />`。

## 7. A11y（UI spec §6）

- 图标按钮必须有 `aria-label` 或可见文字。
- 表单控件有 `<label>`；无可见 label 时 `aria-label` 必填；**禁止 placeholder-only 字段**。
- 可点击 `div/span/li/td` 必须 `role="button"` + `tabindex="0"` + `@keydown.enter` + `@keydown.space.prevent`（见 dict 页面的类型行）；能用 `button` 就不用 div。
- 焦点可见（`:focus-visible`），禁止全局 `outline: none` 无替代；弹窗焦点进出遵循 Element Plus 默认，不要破坏。
- 状态不能只靠颜色，必须同时有文字/图标；图片有 `alt`（装饰图 `alt=""`）；动画尊重 `prefers-reduced-motion`。

## 8. 响应式（UI spec §7）

| 断点 | 规则 |
| --- | --- |
| `>1024` | 主从左右布局；表格完整列 |
| `≤1024` | 工作区上下堆叠；首页保持 16/8 栏；Banner 隐藏插画 |
| `≤800` | 侧边栏抽屉；表格转移动卡片；弹窗全宽/接近全宽；分页精简 |

唯一移动断点常量 `MOBILE_BREAKPOINT = 800`（`@utils/constants`）。**1280 / 800 两档必须实际检查**，不只看 CSS；任何页面不得出现页面级横向滚动。

## 9. 反模式

| 反模式 | 正确做法 |
| --- | --- |
| 手写 `ElTable` + `ElTableColumn` 堆列表 | `useTable` + `FaTable` + `columnsFactory` |
| 手写 `ElDialog` + 表单提交逻辑 | `useCrudDialog`/`useCrudForm` + `FaDialog` + `FaForm` |
| 页面里写裸 hex / `text-[14px]` / 随手间距 | token / 语义 class |
| 手改 `_brand-tokens.generated.scss` | 改 `design/tokens.json` + 跑生成脚本 |
| 只补 zh.json | zh/en 同步 |
| 失败后清空数据显示「暂无数据」 | 保留旧数据 + 错误态 + 重试 |
| 可点击 div 无键盘支持 | role/tabindex/keydown |
| 主从工作区加拖拽分割线 | 固定宽度 + 16px 间距 |
