# 隐藏 app-title 展示行 - 实施计划

## 1. 需求分析

用户要求去除界面中 `app-title` 这一行的显示，即编辑器顶部的应用标题栏区域。

## 2. 代码库调研结论

### 2.1 `#app-title` 元素定位
- `app-title` 是一个 **HTML 元素 ID**（非 CSS class），通过 jQuery `$("#app-title")` 方式在 JavaScript 中操作
- 该元素由压缩后的 JS (`main/app.js`、`forms/app.js`、`main/ie/app.js`) 动态渲染，不在静态 HTML 模板中

### 2.2 现有 show/hide 机制
在各编辑器的 `app.js` 中已存在通过配置参数控制显示/隐藏的逻辑：
```js
var d = /title:(?:(true|show)|(false|hide))/.exec(s);
d && (d[1] ? $("#app-title").show() : d[2] && $("#app-title").hide())
```
即 SDK 使用方可以通过 `title:hide` 或 `title:false` 配置项来隐藏，但这不是默认行为。

### 2.3 布局中的引用（高度计算）
JS 布局代码使用 `#app-title` 高度进行窗口布局计算，例如：
```js
var e = ($("#app-title").length > 0 ? $("#app-title").height() : 0) + $("#toolbar").height() + 2
```
**关键点**：当元素存在但设置 `display:none` 时，jQuery `.height()` 返回 `0`，与代码中已有的 fallback 分支逻辑兼容，不会导致布局错误。

### 2.4 受影响的编辑器
已确认存在 `#app-title` 引用的 JS 文件位置：
- Document Editor（文档编辑器）：`main/app.js`、`main/ie/app.js`、`forms/app.js`
- Spreadsheet Editor（电子表格编辑器）：`main/app.js`、`main/ie/app.js`
- Presentation Editor（演示文稿编辑器）：`main/app.js`、`main/ie/app.js`

Embed 版本 (`embed/app-all.js`) 和 Mobile 版本中 **未发现** `#app-title` 引用，但为了保持一致性仍建议同步修改其对应 CSS 文件（如果将来引入或继承主编辑器样式可以立即生效）。

## 3. 实施方式：CSS 覆盖（推荐）

采用 **CSS 规则覆盖** 方案：在对应编辑器的 CSS 文件末尾追加 `#app-title{display:none!important}` 规则。

**方案对比**：
| 方案 | 优点 | 缺点 |
|------|------|------|
| **CSS 覆盖** | 不侵入 JS 压缩代码、无语法风险、可回滚、维护成本低 | 元素仍存在于 DOM（但不影响布局和性能） |
| 修改压缩 JS | 从 DOM 中彻底移除 | 压缩代码修改语法风险极高、后续升级极易失效 |

**结论**：CSS 覆盖方案风险远低于修改压缩 JS，且效果完全等价（视觉上不再显示）。

## 4. 需修改的文件清单（共 7 个文件）

| # | 文件路径 | 对应编辑器 |
|---|---------|-----------|
| 1 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/resources/css/app.css` | 文档编辑器主界面 |
| 2 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/embed/resources/css/app-all.css` | 文档编辑器嵌入版 |
| 3 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/forms/resources/css/app-all.css` | 文档编辑器表单版 |
| 4 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/resources/css/app.css` | 电子表格主界面 |
| 5 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/embed/resources/css/app-all.css` | 电子表格嵌入版 |
| 6 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/resources/css/app.css` | 演示文稿主界面 |
| 7 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/embed/resources/css/app-all.css` | 演示文稿嵌入版 |

**Mobile 版无需处理**：经搜索，移动版基于 Framework7，`mobile/index.html` 及移动版 CSS 中未发现 `#app-title` 引用。

## 5. 修改步骤

对上述 7 个 CSS 文件，在文件 **末尾** 追加以下 CSS 规则：
```css
#app-title{display:none!important}
```

> 注：所有目标 CSS 文件均为已压缩的单行格式，追加规则无需换行，直接拼接在文件末尾最后一个字符之后即可，保持与压缩格式一致。

## 6. 潜在问题与风险处理

### 6.1 布局间距残留
- **风险**：如果 `#app-title` 的父容器有硬编码的 padding-top 或 margin-top 预留标题栏空间，隐藏后顶部可能出现空白间隙。
- **处理**：若出现此问题，额外检查父容器样式（优先查看 `.toolbar` 或 `#toolbar` 外层容器），追加对应 `padding-top:0!important` 或 `margin-top:0!important` 规则。

### 6.2 其他元素定位依赖标题栏高度
- **风险**：极少数弹窗或浮层的绝对定位可能假设标题栏存在固定高度。
- **处理**：JS 代码已做 `length>0 ? height() : 0` 防御性判断，且 `display:none` 的 `.height()` 返回 0，与已有 fallback 完全一致，理论上无风险；若实测异常，可在对应 CSS 中补充调整。

### 6.3 深色/对比度主题一致性
- **风险**：追加的规则只影响元素显示/隐藏，不涉及颜色或主题变量。
- **处理**：`#app-title{display:none!important}` 不涉及任何主题变量，对所有主题（浅色、深色、高对比度深色、灰色主题）效果完全一致。

## 7. 验证步骤

1. 启动编辑器，打开任一编辑器（文档 / 表格 / 演示）
2. 确认顶部标题栏区域不再渲染，工具栏直接从顶部开始
3. 切换深色主题 / 高对比度主题，确认效果不变
4. 打开窗口或弹出面板（如查找替换、设置等），确认位置不再有额外偏移
5. 验证 embed 模式和 forms 模式（forms 路径仅 doc editor 有）同样生效
