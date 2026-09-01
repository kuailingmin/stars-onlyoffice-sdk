# OnlyOffice 编辑器：背景白色为主 + 选中状态紫色 #7c3aed 实施方案

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans 或 superpowers:subagent-driven-development 来按任务分步执行。步骤使用 `- [ ]` 复选框语法便于追踪。

## 一、需求描述

两大修改，作用于三个编辑器（文字 / 表格 / 幻灯片）：

1. **背景灰色 → 白色为主**：工具栏、侧边栏、状态栏、画布外围等大面积区域改为纯白或极浅灰，界面干净。
2. **所有选中/按下/焦点/勾选状态颜色 → 紫色 `#7c3aed`**：文本选择高亮、按钮按下、焦点边框、行/列标题选中、输入框聚焦、复选框/单选框勾选、动画面板选中条目等全部改成紫色 `#7c3aed`。

**不改动的内容**：
- 深色主题（`theme-dark` / `theme-contrast-dark` / `theme-night`）— 保持黑色调
- 标题栏品牌色（文字蓝 `#446995` / 表格绿 `#3A8056` / 幻灯片红 `#B75B44`）
- 画布内部文档内容背景（已是白色，不动）
- `--highlight-fill-button-*` 系列（填充/背景色按钮，黄色系，有语义用途，不改紫色）
- `highlight-header-button-pressed`（标题栏按色，对主界面灰色影响小且在深色主题语义不同，保留）

## 二、代码库调研结论

### 主题结构

每个编辑器的 `app.css` / `app-all.css` 内含以下主题块（顺序固定）：

| 主题块 | 色调 | 是否处理 | 处理内容 |
|---|---|---|---|
| `:root` （DEFAULT）| 浅灰 | ✅ | 背景白 + 选中紫 |
| `.theme-classic-light` | 浅灰 | ✅ | 背景白 + 选中紫 |
| `.theme-gray` | 浅灰 | ✅ | 背景白 + 选中紫 |
| `.theme-white` | 品牌蓝 | ✅ | 选中紫（背景已接近白） |
| `.theme-dark` | 深灰 | ❌ | 保持原样 |
| `.theme-contrast-dark` | 深黑 | ❌ | 保持原样 |
| `.theme-night` | 深夜蓝 | ❌ | 保持原样 |

### CSS 文件清单（共 7 个，每个文件 7 个主题块）

1. `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/resources/css/app.css`
2. `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/embed/resources/css/app-all.css`
3. `onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/forms/resources/css/app-all.css`
4. `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/resources/css/app.css`
5. `onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/embed/resources/css/app-all.css`
6. `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/resources/css/app.css`
7. `onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/embed/resources/css/app-all.css`

> 注：不修改 mobile/help/vendor CSS。

---

## 三、A 部分：背景灰色 → 白色为主（替换映射表）

作用域：默认 `:root` + `.theme-classic-light` + `.theme-gray` 三个浅色主题块。

### A.1 默认 `:root` 背景替换（26 条）

| # | CSS 变量 | 旧值 | 新值 | 说明 |
|---|---|---|---|---|
| A1 | `--background-toolbar` | `#f7f7f7` | `#ffffff` | 工具栏主背景 |
| A2 | `--background-toolbar-additional` | `#efefef` | `#ffffff` | 工具栏附加面板 |
| A3 | `--background-pane` | `var(--background-toolbar)` | `#ffffff` | 侧边面板 |
| A4 | `--background-pane-additional` | `#efefef` | `#ffffff` | 侧边面板附加区 |
| A5 | `--canvas-background` | `#eee` 或 `#eeeeee` | `#ffffff` | 画布外围（大面积） |
| A6 | `--canvas-page-border` | `#ccc` 或 `#cccccc` | `#e6e6e6` | 页边框（稍淡） |
| A7 | `--canvas-ruler-margins-background` | `#d9d9d9` | `#f5f5f5` | 标尺边距 |
| A8 | `--canvas-cell-title-background` | `#f7f7f7` | `#ffffff` | 表格行列标题背景 |
| A9 | `--canvas-cell-title-background-hover` | `#dfdfdf` | `#f5f5f5` | 行列标题悬停 |
| A10 | `--canvas-cell-title-background-selected` | `#cfcfcf` | `#e6e6e6` | 行列标题选中（后续再叠紫色选中） |
| A11 | `--canvas-anim-pane-background` | `#f7f7f7` | `#ffffff` | 动画面板背景 |
| A12 | `--highlight-button-hover` | `#e0e0e0` | `#f0f0f0` | 按钮悬停（浅灰反馈保留） |
| A13 | `--highlight-button-pressed` | `#cbcbcb` | `#e6e6e6` | 按钮按下背景 |
| A14 | `--highlight-button-pressed-hover` | `#bababa` | `#d9d9d9` | 按下+悬停 |
| A15 | `--border-toolbar` | `#cbcbcb` | `#e6e6e6` | 工具栏分隔边框 |
| A16 | `--border-divider` | `#dfdfdf` | `#ededed` | 细分隔线 |
| A17 | `--canvas-ruler-border` | `#cbcbcb` | `#e6e6e6` | 标尺边框 |
| A18 | `--canvas-cell-title-border` | `#d8d8d8` | `#e6e6e6` | 行列标题边框 |
| A19 | `--canvas-cell-title-border-hover` | `#c9c9c9` | `#d9d9d9` | 行列标题悬停边框 |
| A20 | `--canvas-cell-title-border-selected` | `#bbb` | `#cccccc` | 行列标题选中边框 |
| A21 | `--canvas-scroll-thumb` | `#f7f7f7` | `#ffffff` | 滚动条滑块 |
| A22 | `--canvas-scroll-thumb-border` | `#cbcbcb` | `#e6e6e6` | 滑块边框 |
| A23 | `--canvas-scroll-thumb-target` | `#c0c0c0` | `#e0e0e0` | 目标滑块 |
| A24 | `--canvas-scroll-arrow-pressed` | `#f7f7f7` | `#ffffff` | 滚动条箭头按下（背景） |
| A25 | `--canvas-scroll-thumb-target-pressed` | `#f7f7f7` | `#ffffff` | 目标滑块按下（背景） |
| A26 | `--highlight-comment-hover` | `#EBEAEB` | `#f5f5f5` | 评论悬停高亮 |

### A.2 `.theme-classic-light` 背景补充（10 条）

| # | CSS 变量 | 旧值 | 新值 |
|---|---|---|---|
| C1 | `--background-toolbar` | `#f1f1f1` | `#ffffff` |
| C2 | `--background-toolbar-additional` | `#f1f1f1` | `#ffffff` |
| C3 | `--canvas-background` | `#e2e2e2` | `#ffffff` |
| C4 | `--canvas-cell-title-background` | `#f1f1f1` | `#ffffff` |
| C5 | `--canvas-cell-title-background-hover` | `#d6d6d6` | `#f5f5f5` |
| C6 | `--canvas-cell-title-background-selected` | `#c1c1c1` | `#e6e6e6` |
| C7 | `--canvas-anim-pane-background` | `#f1f1f1` | `#ffffff` |
| C8 | `--canvas-scroll-thumb` | `#f1f1f1` | `#ffffff` |
| C9 | `--canvas-scroll-arrow-pressed` | `#f1f1f1` | `#ffffff` |
| C10 | `--canvas-scroll-thumb-target-pressed` | `#f1f1f1` | `#ffffff` |

### A.3 `.theme-gray` 背景补充（10 条）

同 A.2 C1-C10（`.theme-gray` 的灰色变体与 classic-light 类似，但精确值为 `#f7f7f7`/`#efefef`/`#eee`，A.1 部分已覆盖；theme-gray 独特的地方是 `--highlight-header-button-pressed:#cbcbcb` 改成 `#e6e6e6` 即可。若其他变量已在 A.1 命中则自动生效）。

### A.4 `.theme-white` 背景 — 略

`.theme-white` 的 `--background-toolbar` 已是 `#f3f3f3`（接近纯白），且有自己的品牌蓝选中色系，背景不做修改，但**紫色选中部分**（B 部分）仍要覆盖。

---

## 四、B 部分：选中状态改为紫色 #7c3aed（替换映射表）

作用域：默认 `:root` + `.theme-classic-light` + `.theme-gray` + `.theme-white` 四个浅色主题块。
**`-dark` / `-contrast-dark` / `-night` 三个深色主题完全不动。**

为保证文字在紫色背景上仍可读，按下/勾选用 `#7c3aed`（紫色），焦点/边框用同色，**需要同时按下相关的文字颜色改白色**：
- 文字选中高亮的底色是**半透明叠加**，所以对于 `--highlight-text-select` 这种用来给选中文本填色的变量，直接用 `#7c3aed` 会使选中文字变黑看不清。因此**对底色变量**，我们**用紫色的半透明版本** `rgba(124,58,237,0.35)`；而**对边框/选中条填充**这种实心色，用纯 `#7c3aed`。

### B.1 默认 `:root` 选中状态紫色替换（36 条）

| # | CSS 变量 | 旧值 | 新值（紫色体系） | 说明 |
|---|---|---|---|---|
| B1 | `--highlight-text-select` | `#3494fb` | `rgba(124,58,237,0.35)` | 文字选中高亮（半透明底，不遮文字） |
| B2 | `--highlight-button-pressed` | `#cbcbcb` | `#7c3aed` | 工具栏按钮按下（实心紫） |
| B3 | `--highlight-button-pressed-hover` | `#bababa` | `#8b5cf6` | 按下+悬停，稍亮紫 |
| B4 | `--border-control-focus` | `#848484` | `#7c3aed` | 所有输入控件焦点边框 |
| B5 | `--border-button-pressed-focus` | `#848484` | `#7c3aed` | 按钮按下+焦点组合边框 |
| B6 | `--border-preview-select` | `#888` | `#7c3aed` | 预览模式选中边框 |
| B7 | `--border-toolbar-active-panel-top` | `var(--background-toolbar)` | `#7c3aed` | 顶部激活面板顶边（标签指示） |
| B8 | `--border-toolbar-active-tab` | `var(--background-toolbar)` | `#7c3aed` | 激活标签边 |
| B9 | `--canvas-cell-title-background-selected` | `#cfcfcf` | `#7c3aed` | 表格行列标题选中背景 |
| B10 | `--canvas-cell-title-border-selected` | `#bbb` | `#5b21b6` | 行列标题选中边框（更深紫，有层次） |
| B11 | `--canvas-anim-pane-item-fill-selected` | `#cbcbcb` | `#7c3aed` | 动画面板选中条目填充 |
| B12 | `--canvas-scroll-thumb-pressed` | `#adadad` | `#7c3aed` | 滚动条滑块按下 |
| B13 | `--canvas-scroll-thumb-border-pressed` | `#adadad` | `#7c3aed` | 滑块按下边框 |
| B14 | `--canvas-select-all-icon` | `#999` | `#7c3aed` | 全选图标颜色 |
| B15 | `--icon-normal-pressed` | `#444` | `#ffffff` | 按钮按下的图标颜色改白（紫底上黑图标看不清） |
| B16 | `--text-normal-pressed` | `rgba(0,0,0,0.8)` | `#ffffff` | 按钮按下的文字颜色改白 |
| B17 | `--text-link-active` | `#445799` | `#7c3aed` | 激活链接色 |
| B18 | `--shadow-control-focus` | `inset 0 0 0 1px var(--border-control-focus)` | 保持不变（跟随 B4 自动变紫） | — |
| B19 | `--shadow-control-pressed-focus` | 复合 | 保持不变（跟随 B4 自动变） | — |
| B20 | `--chb-border-normal-focus` | `var(--border-control-focus)` | 跟随 B4 自动变 | — |
| B21 | `--chb-border-checked-focus` | `var(--border-control-focus)` | 跟随 B4 自动变 | — |
| B22 | `--chb-border-checked` | `var(--border-regular-control)` | **覆盖为** `#7c3aed` | 复选框勾选中边框 |
| B23 | `--chb-border-checked-hover` | `var(--border-regular-control)` | **覆盖为** `#8b5cf6` | 复选框勾选+悬停边框 |
| B24 | `--rb-border-normal-focus` | `var(--border-control-focus)` | 跟随 B4 自动变 | — |
| B25 | `--rb-border-checked` | `var(--border-regular-control)` | **覆盖为** `#7c3aed` | 单选框选中边框 |
| B26 | `--rb-border-checked-hover` | `var(--border-regular-control)` | **覆盖为** `#8b5cf6` | 单选框选中+悬停边框 |
| B27 | `--rb-border-checked-focus` | `var(--border-control-focus)` | 跟随 B4 自动变 | — |
| B28 | `--slider-thumb-background-active` | `var(--background-normal)` | **覆盖为** `#7c3aed` | 滑块条按下 |
| B29 | `--canvas-sheet-view-select-all-icon` | `#3d664e` | `#7c3aed` | 表格全选按钮图标 |
| B30 | `--highlight-category-button-pressed` | `var(--highlight-button-pressed)` | 跟随 B2 自动变 | — |
| B31 | `--highlight-header-input-pressed` | `var(--highlight-header-button-pressed)` | 保持（头部有语义） | — |
| B32 | `--border-fill-input-focused` | `var(--border-control-focus)` | 跟随 B4 自动变 | — |
| B33 | `--shadow-control-primary-pressed-focus` | 复合 | 保持不变 | — |
| B34 | `--chb-background-checked` | `var(--background-normal)` | 保持不变（选中背景通过边框颜色+勾选图标表现） | — |
| B35 | `--chb-background-checked-hover` | `var(--background-normal)` | 保持不变 | — |
| B36 | `--rb-background-checked` | `var(--background-normal)` | 保持不变 | — |

**压缩格式下需要做的真实替换（精确匹配，忽略跟随/不变项，共约 23 条实替换）：**

```
1.  --highlight-text-select:#3494fb              →  --highlight-text-select:rgba(124,58,237,0.35)
2.  --highlight-button-pressed:#cbcbcb           →  --highlight-button-pressed:#7c3aed
3.  --highlight-button-pressed-hover:#bababa     →  --highlight-button-pressed-hover:#8b5cf6
4.  --border-control-focus:#848484               →  --border-control-focus:#7c3aed
5.  --border-button-pressed-focus:#848484        →  --border-button-pressed-focus:#7c3aed
6.  --border-preview-select:#888                 →  --border-preview-select:#7c3aed
7.  --canvas-cell-title-background-selected:#cfcfcf   →  --canvas-cell-title-background-selected:#7c3aed
8.  --canvas-cell-title-border-selected:#bbb          →  --canvas-cell-title-border-selected:#5b21b6
9.  --canvas-anim-pane-item-fill-selected:#cbcbcb     →  --canvas-anim-pane-item-fill-selected:#7c3aed
10. --canvas-scroll-thumb-pressed:#adadad             →  --canvas-scroll-thumb-pressed:#7c3aed
11. --canvas-scroll-thumb-border-pressed:#adadad      →  --canvas-scroll-thumb-border-pressed:#7c3aed
12. --canvas-select-all-icon:#999                     →  --canvas-select-all-icon:#7c3aed
13. --icon-normal-pressed:#444                        →  --icon-normal-pressed:#ffffff
14. --text-normal-pressed:rgba(0,0,0,0.8)             →  --text-normal-pressed:#ffffff
15. --text-link-active:#445799                        →  --text-link-active:#7c3aed
16. --chb-border-checked:var(--border-regular-control)    →  --chb-border-checked:#7c3aed
17. --chb-border-checked-hover:var(--border-regular-control) → --chb-border-checked-hover:#8b5cf6
18. --rb-border-checked:var(--border-regular-control)     →  --rb-border-checked:#7c3aed
19. --rb-border-checked-hover:var(--border-regular-control) → --rb-border-checked-hover:#8b5cf6
20. --slider-thumb-background-active:var(--background-normal) → --slider-thumb-background-active:#7c3aed
21. --canvas-sheet-view-select-all-icon:#3d664e          →  --canvas-sheet-view-select-all-icon:#7c3aed
22. --border-toolbar-active-panel-top:var(--background-toolbar) → --border-toolbar-active-panel-top:#7c3aed
23. --border-toolbar-active-tab:var(--background-toolbar)    → --border-toolbar-active-tab:#7c3aed
```

### B.2 `.theme-classic-light` 选中紫色补充（+1 差异）

`.theme-classic-light` 的独特差异（区别于默认）：
- `--highlight-button-pressed:#7d858c` → `#7c3aed`
- `--highlight-button-pressed-hover:#7d858c` → `#8b5cf6`
- `--canvas-cell-title-background-selected:#c1c1c1` → `#7c3aed`
- `--canvas-cell-title-border-selected:#929292` → `#5b21b6`
- `--canvas-anim-pane-item-fill-selected:#7d858c` → `#7c3aed`
- `--border-button-pressed-focus:#444444` → `#7c3aed`
- `--border-preview-select:#848484` → `#7c3aed`
- `--canvas-select-all-icon:#82878f` → `#7c3aed`
- `--icon-normal-pressed:#fff` → **保持 `#fff`**（classic-light 按下已经白色，无需改）
- `--text-normal-pressed:#fff` → **保持 `#fff`**

其余 B.1 中的 34 条在 classic-light 主题里要么与默认相同、要么已经是变量引用，会被 B.1 的替换命中。

### B.3 `.theme-gray` 选中紫色补充（与默认几乎相同）

`.theme-gray` 的选中颜色变量与 `:root` 默认完全一致，**B.1 的 23 条替换会在 theme-gray 主题块里再次命中**（因为文件里重复定义了每个主题块的变量），所以无需额外维护。唯一需要确认的：
- `--highlight-header-button-pressed:#cbcbcb` → 这个属于头部区域语义，不改紫色。

### B.4 `.theme-white` 选中紫色覆盖（品牌蓝 → 紫色）

`.theme-white` 默认使用品牌蓝 `#4473CA` / `#1D4FAF` 作为选中/焦点色，把这两个替换为紫色：

| # | 旧值 | 新值 | 说明 |
|---|---|---|---|
| T1 | `--border-button-pressed-focus:#4473CA` | `#7c3aed` | 焦点边框系列 |
| T2 | `--border-control-focus:#4473CA` | `#7c3aed` | |
| T3 | `--border-fill-input-focused:#4473CA` | `#7c3aed` | |
| T4 | `--border-preview-select:#4473CA` | `#7c3aed` | |
| T5 | `--chb-border-checked-focus:#4473CA` | `#7c3aed` | |
| T6 | `--chb-border-normal-focus:#4473CA` | `#7c3aed` | |
| T7 | `--highlight-primary-dialog-button-pressed:#1D4FAF` | `#7c3aed` | 主按钮按下（Dialog OK） |
| T8 | `--rb-border-checked-focus:#4473CA` | `#7c3aed` | |
| T9 | `--rb-border-normal-focus:#4473CA` | `#7c3aed` | |
| T10 | `--slider-thumb-background-active:#1D4FAF` | `#7c3aed` | 滑块条按下 |

`.theme-white` 中一些与默认主题相同的灰色选中色（如 `--canvas-cell-title-background-selected:#cfcfcf`、`--highlight-button-pressed:#E1E1E1`）会被 B.1 的替换命中。

---

## 五、修改步骤（按文件 × 两步替换）

对下面 7 个文件依次执行：**① A 部分背景替换 → ② B 部分紫色选中替换**。

每个文件跑完脚本后做校验：
```bash
# 校验：替换后仍有多少灰色
grep -oE "#(f7f7f7|efefef|eeeeee|d9d9d9|cbcbcb|dfdfdf|bababa|cfcfcf|adadad|c0c0c0|dfdfdf)" file.css | wc -l
# 修改前 ~50-100 次，修改后应 < 5 次（仅存于未改的 dark 主题）

# 校验：紫色值出现次数
grep -oE "7c3aed|8b5cf6|5b21b6" file.css | wc -l
# 应 ≥ 50 次（每个浅色主题 ~15 次 × 4 主题 ≈ 60+）
```

### 文件清单与预期替换计数

| 文件 | 目标替换数（A + B ≈ 合计） |
|---|---|
| documenteditor/main/resources/css/app.css | ~90-110 |
| documenteditor/embed/resources/css/app-all.css | ~90-110 |
| documenteditor/forms/resources/css/app-all.css | ~90-110 |
| presentationeditor/main/resources/css/app.css | ~90-110 |
| presentationeditor/embed/resources/css/app-all.css | ~90-110 |
| spreadsheeteditor/main/resources/css/app.css | ~95-115（表格多 canvas-cell 相关） |
| spreadsheeteditor/embed/resources/css/app-all.css | ~95-115 |

---

## 六、执行细节（实际 Python 脚本）

把这整个脚本保存为 `_replace_colors.py`，然后对 7 个文件**各跑一次**（或一次性跑所有文件）：

```python
#!/usr/bin/env python3
import sys, re

FILES = [
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/resources/css/app.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/embed/resources/css/app-all.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/forms/resources/css/app-all.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/resources/css/app.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/embed/resources/css/app-all.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/resources/css/app.css",
    "/Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk/onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/embed/resources/css/app-all.css",
]

# PART A + PART B 合并为一个替换列表
# 注意 1：顺序有要求 —— 先长字符串再短（避免 A 改完 B 的精确匹配失效）
# 注意 2：一次只读一次文件，全部替换一次性做（否则 B 依赖的 "cfcfcf" 已被 A 改没）
#   → 解决方法：A.10 是把 #cfcfcf → #e6e6e6，而 B.9 是把 #cfcfcf → #7c3aed
#   → 我们希望 "选中背景" 最终落在 B.9 的紫上，所以 B.9 必须在 A.10 之后（或先 A 改所有非选中灰，再 B 改选中色）
#
# 最佳顺序：
#   Phase 1: 所有"非选中含义"的背景灰色变白（A 部分中，不与 B 冲突的条目）
#   Phase 2: 所有"选中/焦点/按下/勾选"紫色替换（B 部分）— 覆盖原灰色色值
#   Phase 3: A 部分中 B 没涉及的剩余灰色（如 --canvas-scroll-thumb:#f7f7f7 → #ffffff 等）

REPLACEMENTS_ORDERED = [
    # ================ Phase A1: 背景类灰色 → 白色（不触碰选中专用） =================
    # 工具栏/面板
    ('--background-toolbar:#fff',                     '--background-toolbar:#ffffff'),
    ('--background-toolbar-additional:#efefef',          '--background-toolbar-additional:#ffffff'),
    ('--background-pane:#fff',                        '--background-pane:#ffffff'),
    ('--background-pane-additional:#efefef',             '--background-pane-additional:#ffffff'),
    # canvas 外围/边框/标尺
    ('--canvas-background:#eee',                         '--canvas-background:#ffffff'),
    ('--canvas-background:#eeeeee',                      '--canvas-background:#ffffff'),
    ('--canvas-page-border:#ccc',                        '--canvas-page-border:#e6e6e6'),
    ('--canvas-page-border:#cccccc',                     '--canvas-page-border:#e6e6e6'),
    ('--canvas-ruler-margins-background:#d9d9d9',        '--canvas-ruler-margins-background:#f5f5f5'),
    ('--canvas-ruler-border:#cbcbcb',                    '--canvas-ruler-border:#e6e6e6'),
    # 行标题（未 hover 未 selected 的常态/hover 态，不碰 selected）
    ('--canvas-cell-title-background:#f7f7f7',           '--canvas-cell-title-background:#ffffff'),
    ('--canvas-cell-title-background-hover:#dfdfdf',     '--canvas-cell-title-background-hover:#f5f5f5'),
    ('--canvas-cell-title-border:#d8d8d8',               '--canvas-cell-title-border:#e6e6e6'),
    ('--canvas-cell-title-border-hover:#c9c9c9',         '--canvas-cell-title-border-hover:#d9d9d9'),
    # 动画面板背景（不碰 item-fill-selected）
    ('--canvas-anim-pane-background:#f7f7f7',            '--canvas-anim-pane-background:#ffffff'),
    # 按钮 hover（不碰 pressed）
    ('--highlight-button-hover:#e0e0e0',                 '--highlight-button-hover:#f0f0f0'),
    # 分隔边框
    ('--border-toolbar:#cbcbcb',                         '--border-toolbar:#e6e6e6'),
    ('--border-divider:#dfdfdf',                         '--border-divider:#ededed'),
    # 滚动条（非 pressed 态）
    ('--canvas-scroll-thumb:#f7f7f7',                    '--canvas-scroll-thumb:#ffffff'),
    ('--canvas-scroll-thumb-border:#cbcbcb',             '--canvas-scroll-thumb-border:#e6e6e6'),
    ('--canvas-scroll-thumb-target:#c0c0c0',             '--canvas-scroll-thumb-target:#e0e0e0'),
    ('--canvas-scroll-arrow-pressed:#f7f7f7',            '--canvas-scroll-arrow-pressed:#ffffff'),
    ('--canvas-scroll-thumb-target-pressed:#f7f7f7',     '--canvas-scroll-thumb-target-pressed:#ffffff'),
    # 评论悬停
    ('--highlight-comment-hover:#EBEAEB',                '--highlight-comment-hover:#f5f5f5'),
    # theme-classic-light 独有背景灰
    ('--background-toolbar:#f1f1f1',                     '--background-toolbar:#ffffff'),
    ('--background-toolbar-additional:#f1f1f1',          '--background-toolbar-additional:#ffffff'),
    ('--canvas-background:#e2e2e2',                      '--canvas-background:#ffffff'),
    ('--canvas-cell-title-background:#f1f1f1',           '--canvas-cell-title-background:#ffffff'),
    ('--canvas-cell-title-background-hover:#d6d6d6',     '--canvas-cell-title-background-hover:#f5f5f5'),
    ('--canvas-anim-pane-background:#f1f1f1',            '--canvas-anim-pane-background:#ffffff'),
    ('--canvas-scroll-thumb:#f1f1f1',                    '--canvas-scroll-thumb:#ffffff'),
    ('--canvas-scroll-arrow-pressed:#f1f1f1',            '--canvas-scroll-arrow-pressed:#ffffff'),
    ('--canvas-scroll-thumb-target-pressed:#f1f1f1',     '--canvas-scroll-thumb-target-pressed:#ffffff'),
    ('--highlight-button-hover:#d8dadc',                 '--highlight-button-hover:#f0f0f0'),

    # ================ Phase B: 选中/按下/焦点/勾选 → 紫色 =================
    # 文本选中高亮（半透明底 → 不遮文字）
    ('--highlight-text-select:#3494fb',                          '--highlight-text-select:rgba(124,58,237,0.35)'),
    # 工具栏按钮按下（实心紫 + 配套图标/文字改白）
    ('--highlight-button-pressed:#cbcbcb',                       '--highlight-button-pressed:#7c3aed'),
    ('--highlight-button-pressed:#7d858c',                       '--highlight-button-pressed:#7c3aed'),   # theme-classic-light
    ('--highlight-button-pressed-hover:#bababa',                 '--highlight-button-pressed-hover:#8b5cf6'),
    ('--highlight-button-pressed-hover:#7d858c',                 '--highlight-button-pressed-hover:#8b5cf6'), # theme-classic-light
    ('--icon-normal-pressed:#444',                               '--icon-normal-pressed:#ffffff'),
    ('--text-normal-pressed:rgba(0,0,0,0.8)',                    '--text-normal-pressed:#ffffff'),
    # 焦点/选中边框（输入/按钮/预览）
    ('--border-control-focus:#848484',                           '--border-control-focus:#7c3aed'),
    ('--border-button-pressed-focus:#848484',                    '--border-button-pressed-focus:#7c3aed'),
    ('--border-button-pressed-focus:#444444',                    '--border-button-pressed-focus:#7c3aed'), # classic-light
    ('--border-preview-select:#888',                             '--border-preview-select:#7c3aed'),
    ('--border-preview-select:#848484',                          '--border-preview-select:#7c3aed'),  # classic-light
    # 顶部激活标签指示条
    ('--border-toolbar-active-panel-top:var(--background-toolbar)', '--border-toolbar-active-panel-top:#7c3aed'),
    ('--border-toolbar-active-tab:var(--background-toolbar)',       '--border-toolbar-active-tab:#7c3aed'),
    # 行/列标题选中（紫色系）
    ('--canvas-cell-title-background-selected:#cfcfcf',         '--canvas-cell-title-background-selected:#7c3aed'),
    ('--canvas-cell-title-background-selected:#c1c1c1',         '--canvas-cell-title-background-selected:#7c3aed'), # classic-light
    ('--canvas-cell-title-border-selected:#bbb',                '--canvas-cell-title-border-selected:#5b21b6'),
    ('--canvas-cell-title-border-selected:#929292',             '--canvas-cell-title-border-selected:#5b21b6'),  # classic-light
    # 动画面板选中条目
    ('--canvas-anim-pane-item-fill-selected:#cbcbcb',           '--canvas-anim-pane-item-fill-selected:#7c3aed'),
    ('--canvas-anim-pane-item-fill-selected:#7d858c',           '--canvas-anim-pane-item-fill-selected:#7c3aed'), # classic-light
    # 滚动条按下
    ('--canvas-scroll-thumb-pressed:#adadad',                   '--canvas-scroll-thumb-pressed:#7c3aed'),
    ('--canvas-scroll-thumb-border-pressed:#adadad',            '--canvas-scroll-thumb-border-pressed:#7c3aed'),
    # 全选图标
    ('--canvas-select-all-icon:#999',                           '--canvas-select-all-icon:#7c3aed'),
    ('--canvas-select-all-icon:#82878f',                        '--canvas-select-all-icon:#7c3aed'), # classic-light
    ('--canvas-sheet-view-select-all-icon:#3d664e',             '--canvas-sheet-view-select-all-icon:#7c3aed'),
    # 链接激活
    ('--text-link-active:#445799',                              '--text-link-active:#7c3aed'),
    # 复选框勾选边框
    ('--chb-border-checked:var(--border-regular-control)',      '--chb-border-checked:#7c3aed'),
    ('--chb-border-checked-hover:var(--border-regular-control)','--chb-border-checked-hover:#8b5cf6'),
    # 单选框勾选边框
    ('--rb-border-checked:var(--border-regular-control)',       '--rb-border-checked:#7c3aed'),
    ('--rb-border-checked-hover:var(--border-regular-control)', '--rb-border-checked-hover:#8b5cf6'),
    # 滑块条按下
    ('--slider-thumb-background-active:var(--background-normal)', '--slider-thumb-background-active:#7c3aed'),

    # ================ Phase B+.theme-white 品牌蓝 → 紫色 =================
    ('--border-button-pressed-focus:#4473CA',                   '--border-button-pressed-focus:#7c3aed'),
    ('--border-control-focus:#4473CA',                          '--border-control-focus:#7c3aed'),
    ('--border-fill-input-focused:#4473CA',                     '--border-fill-input-focused:#7c3aed'),
    ('--border-preview-select:#4473CA',                         '--border-preview-select:#7c3aed'),
    ('--chb-border-checked-focus:#4473CA',                      '--chb-border-checked-focus:#7c3aed'),
    ('--chb-border-normal-focus:#4473CA',                       '--chb-border-normal-focus:#7c3aed'),
    ('--highlight-primary-dialog-button-pressed:#1D4FAF',       '--highlight-primary-dialog-button-pressed:#7c3aed'),
    ('--rb-border-checked-focus:#4473CA',                       '--rb-border-checked-focus:#7c3aed'),
    ('--rb-border-normal-focus:#4473CA',                        '--rb-border-normal-focus:#7c3aed'),
    ('--slider-thumb-background-active:#1D4FAF',                '--slider-thumb-background-active:#7c3aed'),

    # ================ Phase A2: A 部分中冲突于 B 的补齐（用 B 覆盖后已无需再补；留空防御） =================
    # （注：A 部分中 --canvas-cell-title-background-selected / 等都已在 B 中覆盖为紫；
    # 若还有 #cfcfcf 残留（非 selected 含义），用下面的兜底。但 selected 先被 B 改完不会命中这里。）
]

for fpath in FILES:
    with open(fpath, 'r', encoding='utf-8') as fh:
        content = fh.read()
    count = 0
    for old, new in REPLACEMENTS_ORDERED:
        occurrences = content.count(old)
        if occurrences > 0:
            content = content.replace(old, new)
            count += occurrences
    with open(fpath, 'w', encoding='utf-8') as fh:
        fh.write(content)
    # 统计报告
    gray_left = len(re.findall(r'#(f7f7f7|efefef|eeeeee|d9d9d9|cbcbcb|dfdfdf|bababa|cfcfcf|adadad|c0c0c0|d8d8d8|dfdfdf)', content))
    purple_hits = len(re.findall(r'7c3aed|8b5cf6|5b21b6|124,58,237', content))
    name = '/'.join(fpath.split('/')[-5:])
    print(f"✔ {name}: 做了 {count} 处替换；残留浅灰 ~{gray_left} 个；紫色出现 ~{purple_hits} 次")
```

---

## 七、注意事项与风险处理

### 1. 不修改深色主题
- `.theme-dark`、`.theme-contrast-dark`、`.theme-night` 三个主题块**绝对不碰**。
- 我们做的精确值替换（`#f7f7f7` / `#cbcbcb` / `#cfcfcf` / `#848484` / `#444` 等）在深色主题里出现频率极低 —— 深色主题用的是 `#333`、`#404040`、`#606060`、`#888`、`#707070`，唯一重叠的是 `#888` 的 `--border-preview-select` 和 `#cfcfcf`（theme-dark 里不用 `#cfcfcf`）。**为了安全**，执行完后 grep 一下 `theme-dark` 的 `#7c3aed` 数量，如果 > 0，说明误伤了，需要把深色主题块的 `#7c3aed` 改回原深色值。

### 2. 改完必须清理缓存
之前报错的教训：只有清缓存才能保证浏览器加载新的 CSS。
- DevTools 开着 + 勾 "Disable cache" + Cmd+Shift+R

### 3. 回滚方式
所有修改在 7 个 CSS 文件里，一键回滚：
```bash
cd /Users/kuailingmin/Desktop/klm/stars-onlyoffice-sdk
git checkout -- onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/main/resources/css/app.css onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/embed/resources/css/app-all.css onlyoffice/9.4.0-develop/web-apps/apps/documenteditor/forms/resources/css/app-all.css onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/main/resources/css/app.css onlyoffice/9.4.0-develop/web-apps/apps/presentationeditor/embed/resources/css/app-all.css onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/main/resources/css/app.css onlyoffice/9.4.0-develop/web-apps/apps/spreadsheeteditor/embed/resources/css/app-all.css
```

### 4. 若紫色"太亮/太暗"，调整方案
- 觉得 `#7c3aed` 太刺眼：把纯色变量换成 `#6d28d9`（稍暗）
- 觉得文字选中标太淡：把 `rgba(124,58,237,0.35)` 改为 `rgba(124,58,237,0.5)`
- 不用重写方案：直接对 7 个文件做一次 `sed` 批量即可

---

## 八、验证清单（改完后在浏览器检查）

### 背景变白
- [ ] 文字编辑器：工具栏纯白、左侧菜单背景白、画布外围白
- [ ] 表格编辑器：工具栏白、行号列标题背景白、网格外白、底部标签栏白
- [ ] 幻灯片编辑器：工具栏白、动画面板白、底部状态栏白
- [ ] 按钮 hover 仍有浅灰反馈（不突兀就行）
- [ ] 深色主题切换后正常（没有出现白底黑字异常）

### 选中/按下变紫色
- [ ] 在文档里拖动选中文字 → 高亮底色是紫色半透明
- [ ] 工具栏按钮按下 → 按钮背景紫色，图标/文字变白
- [ ] 点击一个输入框 → 聚焦边框是紫色
- [ ] 勾选复选框 / 单选框 → 选中边框紫色
- [ ] 表格编辑器：点击行号列标题 → 选中背景紫色，边框深紫
- [ ] 表格编辑器：拖曳滚动条 → 滚动条按下时紫色
- [ ] 设置 → 切到 theme-classic-light / theme-gray / theme-white → 以上效果一致
- [ ] 切到 theme-dark / theme-contrast-dark → **仍然正常深色，没有紫色干扰**

---

## 九、修改任务列表

```
- [ ] 任务 1：执行批量替换 Python 脚本，修改 7 个 CSS 文件
- [ ] 任务 2：替换计数校验 —— 每个文件 grep 灰色残留 < 5；紫色出现 > 50
- [ ] 任务 3：深色主题安全校验 —— grep `theme-dark` 区段不应有 #7c3aed
- [ ] 任务 4：浏览器打开 3 个编辑器，逐条跑"八、验证清单"
- [ ] 任务 5：如有颜色深度微调，根据反馈再跑一次 sed 替换
```
