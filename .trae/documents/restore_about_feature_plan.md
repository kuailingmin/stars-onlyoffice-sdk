# 还原"关于"功能交互 - 实施计划

## 1. 需求概述

用户要求还原界面中的"关于"(About) 功能交互，使 About 按钮和面板在编辑器中重新可见可用。

## 2. 当前状态分析

About 功能通过 `customization.about` 配置项控制，当前被**三层机制**禁用：

### 2.1 API 默认配置（api.js）
- 文件：`onlyoffice/9.4.0-develop/web-apps/apps/api/documents/api.js`
- 第 992 行：`DocsAPI.DocEditor.defaultConfig.customization.about` 设为 `false`

### 2.2 桌面模式覆盖（api.js）
- 同文件第 593 行：当检测到桌面环境 (`window.AscDesktopEditor`) 时，强制 `customization.about = false`

### 2.3 编辑器主程序强制覆盖（6 个 JS 文件）
每个编辑器的 `app.js` 和 `ie/app.js` 在应用自定义配置前，都有一行代码**强制**将 about 设为 false：
```js
this.appOptions.customization&&(this.appOptions.customization.about=!1)
```
这会覆盖任何外部传入的 `about: true` 配置。

涉及文件：
| 编辑器 | 主版本 | IE 兼容版 |
|--------|--------|-----------|
| 文档编辑器 | `documenteditor/main/app.js` | `documenteditor/main/ie/app.js` |
| 电子表格 | `spreadsheeteditor/main/app.js` | `spreadsheeteditor/main/ie/app.js` |
| 演示文稿 | `presentationeditor/main/app.js` | `presentationeditor/main/ie/app.js` |

### 2.4 显示逻辑链路
- `mapCustomizationElements` 将 `about` 映射到 `button#left-btn-about` DOM 元素
- `Common.Utils.applyCustomization()` 读取 `customization.about` 值：为 `true` 则显示按钮，为 `false` 则 `.hide()`
- 按钮点击后通过 `aboutShowHide` 函数控制 `Common.Views.About` 视图的显示/隐藏

## 3. 修改方案

将以下 7 个位置中的 `about` 从 `false` 改为 `true`，恢复完整的 About 交互链路。

### 3.1 修改 api.js（2 处）

**文件**：`onlyoffice/9.4.0-develop/web-apps/apps/api/documents/api.js`

| 位置 | 当前代码 | 修改为 | 说明 |
|------|----------|--------|------|
| 第 593 行 | `_config.editorConfig.customization.about = false;` | `_config.editorConfig.customization.about = true;` | 桌面模式不再禁用 About |
| 第 992 行 | `about: false,` | `about: true,` | 默认配置启用 About |

### 3.2 修改 6 个编辑器 JS 文件（各 1 处）

在每个文件中，将以下代码片段：
```
this.appOptions.customization.about=!1
```
替换为：
```
this.appOptions.customization.about=!0
```

| # | 文件路径 |
|---|---------|
| 1 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/app.js` |
| 2 | `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/ie/app.js` |
| 3 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/app.js` |
| 4 | `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/ie/app.js` |
| 5 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/app.js` |
| 6 | `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/ie/app.js` |

## 4. 风险评估

- **低风险**：修改的是布尔值赋值，不涉及语法结构变化
- **`!0` 等价于 `true`**：在压缩 JS 中 `!0` 是 `true` 的标准写法，保持压缩格式一致性
- **向后兼容**：SDK 使用方仍可通过在 `editorConfig.customization` 中显式设置 `about: false` 来覆盖默认行为
- **不影响布局**：About 按钮位于左侧菜单面板，显示/隐藏不影响主编辑区布局

## 5. 验证步骤

1. 启动任一编辑器（文档 / 表格 / 演示）
2. 检查左侧菜单面板出现 "About" 按钮（`#left-btn-about`）
3. 点击 About 按钮，确认弹出关于对话框（显示版本号、公司信息等）
4. 点击关闭，确认对话框正常关闭
5. 在桌面模式（如果有 `AscDesktopEditor` 环境）下同样验证 About 可见
