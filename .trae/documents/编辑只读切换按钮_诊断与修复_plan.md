# 编辑/只读切换按钮：仅 Word 显示的原因诊断与修复方案

## 摘要

用户观察到：OnlyOffice SDK 的标题栏右侧「编辑/只读模式切换按钮」(即 `customization.layout.header.editMode` 配置项对应的按钮) **只在 Word(documenteditor) 界面显示**，而在 Excel(spreadsheeteditor) 和 PPT(presentationeditor) 界面不显示。

经代码级诊断，根本原因是：**`appOptions.canSwitchMode` 属性仅在 documenteditor 中被赋值，在 spreadsheeteditor / presentationeditor 中从未赋值（值为 `undefined`）**。该属性是标题栏模式切换按钮渲染条件的共同门槛，因此 Excel/PPT 中按钮永远走 `hide()` 分支被隐藏。

本文档先给出完整诊断结论，随后提供可选的修复方案，待用户确认后再执行。

---

## 一、诊断结论（根本原因）

### 1.1 标题栏模式切换按钮的渲染逻辑

三个编辑器的 `apps/{documenteditor,spreadsheeteditor,presentationeditor}/main/app.js` 中，标题栏 `#slot-btn-edit-mode` 插槽的渲染逻辑**完全相同**，结构如下：

```js
c && d.isEdit && d.canSwitchMode ? (
    // 条件1：渲染 btnPDFMode（PDF 模式按钮，caption=textComment，value="comment"）
    p.btnPDFMode = new Common.UI.Button({...}),
    p.btnPDFMode.render(f.find("#slot-btn-edit-mode")),
    ...
) : h && d.isEdit && d.canSwitchMode ? (
    // 条件2：渲染 btnDocMode（文档模式切换按钮——真正的"编辑/只读切换"）
    p.btnDocMode = new Common.UI.Button({
        iconCls: "..." + (d.isReviewOnly ? "btn-ic-review" : "btn-edit"),
        caption: d.isReviewOnly ? p.textReview : p.textEdit,   // "审阅" 或 "编辑"
        value:   d.isReviewOnly ? "review" : "edit",
        visible: d.isReviewOnly || !d.canReview,
        menu: true,
        ...
    }),
    p.btnDocMode.render(f.find("#slot-btn-edit-mode")),
    ...
) : f.find("#slot-btn-edit-mode").hide()   // 否则：隐藏
```

**关键点**：条件1和条件2**都依赖 `d.canSwitchMode`**。其中 `d` 即 `this.appOptions`。

> 注：`btnDocMode`（条件2分支）才是用户看到的「编辑/只读切换按钮」——它带有"编辑/审阅"文案和切换菜单；`btnPDFMode`（条件1分支）是 PDF 场景的评论按钮。

### 1.2 `canSwitchMode` 在三个编辑器中的赋值差异（核心证据）

| 编辑器 | `canSwitchMode` 出现次数 | 是否赋值 | 赋值语句 |
|---|---|---|---|
| documenteditor (Word) | 4 | ✅ 有 | `this.appOptions.canSwitchMode = this.appOptions.isEdit` |
| spreadsheeteditor (Excel) | 2 | ❌ 无 | （仅在按钮条件中读取，从未赋值） |
| presentationeditor (PPT) | 2 | ❌ 无 | （仅在按钮条件中读取，从未赋值） |

- **documenteditor** 中 `canSwitchMode` 出现 4 次：
  - 2 次在标题栏按钮条件中读取（条件1 `c&&d.isEdit&&d.canSwitchMode`、条件2 `h&&d.isEdit&&d.canSwitchMode`）
  - 1 次赋值：`this.appOptions.canComments=!1),this.appOptions.canSwitchMode=this.appOptions.isEdit,this.appOptions.canSubmitForms=...`
  - 1 次在 `onDocModeApply` 回调中读取：`if(this.appOptions.canSwitchMode||e){...}`

- **spreadsheeteditor / presentationeditor** 中 `canSwitchMode` 仅出现 2 次，**都是标题栏按钮条件中的读取**，**没有任何赋值语句**。因此 `this.appOptions.canSwitchMode` 始终为 `undefined`（falsy）。

### 1.3 因果链

1. Excel/PPT 的 `appOptions` 初始化逻辑中**遗漏了 `canSwitchMode` 赋值**
2. → `this.appOptions.canSwitchMode === undefined`（falsy）
3. → 标题栏按钮条件 `h && d.isEdit && d.canSwitchMode` 恒为 `false`（条件1同理）
4. → 渲染逻辑走到 `else` 分支：`f.find("#slot-btn-edit-mode").hide()`
5. → 按钮被隐藏，用户看不到「编辑/只读切换」交互

而 Word 中 `canSwitchMode = isEdit`，当处于编辑模式（`isEdit=true`）时条件成立，`btnDocMode` 被渲染并显示。

### 1.4 补充说明

- 三个编辑器的 `index.html` header 模板都包含相同的按钮插槽 `<div class="hedset" data-layout-name="header-editMode"><div class="btn-slot" id="slot-btn-edit-mode"></div></div>`，即 DOM 容器都存在，差异纯粹在 JS 渲染条件。
- OnlyOffice 官方文档中 `customization.layout.header.editMode`（默认 `true`）描述为"是否在标题中显示用于切换编辑器模式的按钮"，该配置在三个编辑器中均受支持；按钮不显示是 SDK 内部 `canSwitchMode` 赋值缺失所致，而非配置项不支持。

---

## 二、可选修复方案（待用户确认是否执行）

> 用户诉求为「先诊断原因，再决定是否修复」。以下修复方案需用户确认后再实施。

### 2.1 修复思路

在 spreadsheeteditor 和 presentationeditor 的 `appOptions` 初始化逻辑中，**补充 `canSwitchMode` 赋值**，使其与 documenteditor 保持一致：

```js
this.appOptions.canSwitchMode = this.appOptions.isEdit
```

### 2.2 需要修改的文件

需修改 4 个压缩 `app.js` 文件（main + ie 各两个编辑器），并同步外部项目副本（如 `~/Downloads/onlyoffice-web-comp-main`）：

1. `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/app.js`
2. `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/ie/app.js`
3. `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/app.js`
4. `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/ie/app.js`

### 2.3 修改方法

在 spreadsheeteditor / presentationeditor 的 `appOptions` 初始化处（需定位 `setAppOptions` 或等价的 `this.appOptions.isEdit` 赋值之后的位置），插入：

```js
this.appOptions.canSwitchMode=this.appOptions.isEdit,
```

参照 documenteditor 中该语句的位置（紧跟 `canComments` 处理之后、`canSubmitForms` 之前），在 Excel/PPT 中寻找等价的 `appOptions` 初始化序列并插入。

### 2.4 注意事项

- `app.js` 为压缩单行文件，修改需精确匹配字符串、避免破坏语法（参考此前 Hook 注入的经验：需用 `node --check` 验证语法，注意 `$` 转义问题）。
- 需同步至外部项目副本（用户存在两份独立副本：`~/Desktop/klm/stars-onlyoffice-sdk` 与 `~/Downloads/onlyoffice-web-comp-main`）。
- 修改后需硬刷新浏览器验证。
- `btnDocMode` 的 `visible` 为 `d.isReviewOnly || !d.canReview`；Excel/PPT 无审阅权限体系，需确认按钮显示后的实际可见性与菜单项是否符合预期（可能需要同步调整 `visible` 条件）。

---

## 三、假设与决策

1. **假设**：用户当前三种编辑器以相同的 `editorConfig.mode`（如均为 `edit`）打开文档进行对比，排除了因 Excel/PPT 以 `view` 模式打开而 Word 以 `edit` 模式打开导致的观察差异。
2. **决策点**：修复方案是否执行，由用户在阅读诊断结论后决定。若仅需要诊断结论，则无需修改任何代码。
3. **未修改项**：本诊断阶段未对任何代码文件做修改（仅执行只读的搜索/读取操作）。

---

## 四、验证步骤（修复执行后适用）

1. **语法校验**：对修改后的 4 个 `app.js` 执行 `node --check <file>`，确认无语法错误。
2. **注入标记确认**：`grep -oE 'canSwitchMode=this\.appOptions\.isEdit' <file>` 确认赋值语句已存在于 spreadsheeteditor / presentationeditor。
3. **运行验证**：重启 Next.js 服务，硬刷新浏览器，分别用 Excel 和 PPT 打开文档：
   - 编辑模式下：标题栏右侧应出现「编辑/只读切换按钮」
   - 只读模式下：按钮行为与 Word 一致
4. **回归验证**：确认 Word 文档编辑器的按钮行为未受影响。
5. **同步验证**：确认外部项目副本（`~/Downloads/onlyoffice-web-comp-main`）同步生效。
