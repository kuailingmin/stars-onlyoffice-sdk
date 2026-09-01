# 隐藏 app-title 展示行 - 实施计划

## 1. 需求概述

去除编辑器界面顶部 `#app-title`（应用标题栏）这一行的展示，使其在界面中不渲染。

## 2. 当前状态分析

### 2.1 元素定位
- `app-title` 是 HTML 元素 **ID**（`id="app-title"`），通过 jQuery `$("#app-title")` 在压缩后的 JS 中动态创建和操作
- 该元素由各编辑器的 `main/app.js`、`main/ie/app.js`、`forms/app.js` 在运行时渲染，不在静态 HTML 模板中

### 2.2 布局兼容性
JS 布局代码在计算工具栏/编辑区高度时已内置防御逻辑：
```js
var e = ($("#app-title").length > 0 ? $("#app-title").height() : 0) + $("#toolbar").height() + 2
```
当元素 `display:none` 时，jQuery `.height()` 返回 `0`，与 fallback 分支完全一致，**不会出现布局错位**。

### 2.3 现有隐藏机制
JS 中已支持通过配置参数 `title:hide` / `title:false` 控制，但这需要 SDK 调用方显式传参，不是默认行为。

## 3. 实施方案

采用 **CSS 覆盖** 方案（而非修改压缩 JS）：

| 方案 | 风险评估 |
|------|---------|
| ✅ CSS 追加 `#app-title{display:none!important}` | 无语法风险、不侵入压缩 JS、可随时回滚 |
| ❌ 修改压缩 JS 删除渲染逻辑 | 单行百万字符，修改语法风险极高，后续升级失效 |

## 4. 需修改的文件（7 个 CSS 文件）

| # | 文件路径 | 对应编辑器 |
|---|---------|-----------|
| 1 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/resources/css/app.css` | 文档编辑器主界面 |
| 2 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/embed/resources/css/app-all.css` | 文档编辑器嵌入版 |
| 3 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/forms/resources/css/app-all.css` | 文档编辑器表单版 |
| 4 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/resources/css/app.css` | 电子表格主界面 |
| 5 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/embed/resources/css/app-all.css` | 电子表格嵌入版 |
| 6 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/resources/css/app.css` | 演示文稿主界面 |
| 7 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/embed/resources/css/app-all.css` | 演示文稿嵌入版 |

**Mobile 版无需处理**：移动版（`mobile/index.html`）基于 Framework7，未发现 `#app-title` 引用。

## 5. 修改步骤

对上述 7 个 CSS 文件，在**文件末尾**追加以下规则（因文件为压缩单行格式，无需换行）：
```css
#app-title{display:none!important}
```

## 6. 风险与处理

| 风险 | 处理方式 |
|------|---------|
| 顶部残留空白间隙（父容器硬编码 padding-top） | 若出现则追加父容器 `padding-top:0!important` 覆盖 |
| 其他浮层/弹窗对标题栏高度的依赖 | JS 代码已内置 `length>0 ? height() : 0` 防御，实际返回 0 与默认 fallback 一致 |
| 深色/高对比度主题 | 本规则只控制显隐不涉及颜色，所有主题效果一致 |

## 7. 验证步骤

1. 启动文档 / 表格 / 演示任一编辑器
2. 确认顶部标题栏（`#app-title` 区域）不再渲染，工具栏直接从顶部起
3. 切换深色、高对比度、灰色主题，效果一致
4. 打开查找替换、设置等弹窗面板，位置无偏移
5. 验证 embed 模式和 forms 模式（文档）同样生效
