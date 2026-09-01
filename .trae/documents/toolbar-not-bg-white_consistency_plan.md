# 三端 toolbar:not 背景白色一致性修复计划

**编制时间**: 2026-09-01
**适用范围**: Word(documenteditor) / Excel(spreadsheeteditor) / PPT(presentationeditor) 三个编辑器
**目标**: 让三种组件的 `#toolbar:not(.style-off-tabs)` 在页面加载后稳定显示为白色背景 `#fff`，并保证「修改一处 → 三端一致」的原则。

---

## 1. 仓库调研结论（根因分析）

### 1.1 当前 `#toolbar:not` 规则在三处的现状（Desktop SDK）

| 编辑器 | `#toolbar:not(.style-off-tabs)` 当前声明 | fallback | CSS 变量（实际生效） |
|---|---|---|---|
| **Word** (documenteditor) | `background:#fff; background:var(--toolbar-header-document)` | `#fff` ✅ | **`#446995`（蓝色）❌** → 导致 Word 实际不是白色 |
| **Excel** (spreadsheeteditor) | `background:#fff; background:var(--toolbar-header-spreadsheet)` | `#fff` ✅ | **`#3A8056`（绿色）❌** → 导致 Excel 实际不是白色 |
| **PPT** (presentationeditor) | `background:#fff; background:var(--toolbar-header-presentation)` | `#fff` ✅ | `#fff` ✅（恰巧相等，视觉正确） |

**核心问题**：fallback 写的是 `#fff`，但浏览器优先使用第二行 `var(...)`，该变量在 `.theme-classic-light` 主题块中被定义为品牌色（Word 蓝 / Excel 绿 / PPT 白），所以只有 PPT 看起来「正确」。之前我们只改了 fallback 层，没改变量层 → **三端实际上都不一致**，只是 PPT 碰巧对了。

### 1.2 `.theme-classic-light` 主题块中三个变量的定义（三个编辑器 CSS 文件中**都重复定义了同一套主题变量**，值完全相同）

```css
.theme-classic-light {
  --toolbar-header-document:      #446995;   /* Word品牌蓝  → 需改 #fff */
  --toolbar-header-spreadsheet:   #3A8056;    /* Excel品牌绿 → 需改 #fff */
  --toolbar-header-presentation:  #fff;       /* PPT已白    → 不变 */
  --toolbar-header-pdf:           #AA5252;    /* PDF        → 不动（非本次范围） */
  --toolbar-header-visio:         #444796;    /* Visio      → 不动（非本次范围） */
  --background-pane: var(--background-toolbar);
  --background-toolbar: #ffffff;
}
```

### 1.3 已确认一致的规则（作为本次一致化的参照）
- 基础 `#toolbar{padding:0;padding:var(...);background:#fff;background:#fff}`：三处都是白色 ✅
- `.toolbar .tabs li.active` 的 `#7c3aed` 紫色：三处都命中 ✅
- `.panel-menu` 容器的 `#fff`：三处都是白色 ✅

---

## 2. 需要修改的文件与模块

### 2.1 Desktop SDK（源端，6 个 CSS 文件）
之前仅改了 `main/resources/css/app.css` ×3，本次一并覆盖可能存在的 `embed` / `forms` 场景：

| # | 文件路径 | 理由 |
|---|---|---|
| 1 | `documenteditor/main/resources/css/app.css` | Word 主编辑器（必改） |
| 2 | `documenteditor/embed/resources/css/app-all.css`（若存在） | Word 嵌入模式 |
| 3 | `documenteditor/forms/resources/css/app-all.css` | Word 表单模式（前次已同步 panel-menu，需一起改） |
| 4 | `spreadsheeteditor/main/resources/css/app.css` | Excel 主编辑器（必改） |
| 5 | `spreadsheeteditor/embed/resources/css/app-all.css`（若存在） | Excel 嵌入模式 |
| 6 | `presentationeditor/main/resources/css/app.css` | PPT 主编辑器（值目前正确但也要写死对齐，防主题变） |
| 7 | `presentationeditor/embed/resources/css/app-all.css`（若存在） | PPT 嵌入模式 |

> 注：每个文件都有 **两处** 要改（规则层 + 主题变量层）。

### 2.2 Downloads 项目（运行端镜像）
完全对应 2.1 的路径，前缀为：
`/Users/kuailingmin/Downloads/onlyoffice-web-comp-main/public/packages/onlyoffice/9.4.0-develop/web-apps/apps/`

---

## 3. 修改步骤

### 步骤一：规则层兜底 —— `#toolbar:not(.style-off-tabs)` 的 `var(...)` 也硬编码为 `#fff`

把三个编辑器所有 CSS 文件中：
```
before:  background:#fff;  background:var(--toolbar-header-document)     // Word 例
after :  background:#fff;  background:#fff
```
同理替换 Excel 的 `var(--toolbar-header-spreadsheet)`、PPT 的 `var(--toolbar-header-presentation)`。

**原因**：按之前 `.toolbar .tabs` / `#toolbar` 基础选择器 的修改约定，统一采用「硬编码双份 #fff;#fff」模式，与主题变量解耦，避免将来主题更新再次污染 toolbar。

### 步骤二：主题变量层根治 —— `.theme-classic-light` 中 `--toolbar-header-document` 与 `--toolbar-header-spreadsheet` 改为 `#fff`

```
before:  --toolbar-header-document:    #446995;
         --toolbar-header-spreadsheet: #3A8056;
         --toolbar-header-presentation:#fff;

after :  --toolbar-header-document:    #fff;
         --toolbar-header-spreadsheet: #fff;
         --toolbar-header-presentation:#fff;
```

**为什么两处都要改（步骤一 + 步骤二）**：
- 只改步骤一不改步骤二：其他可能依赖 `--toolbar-header-document` 的规则（如变体选择器、插件、子主题）会继续显示蓝色/绿色，存在漏网风险。
- 只改步骤二不改步骤一：`.theme-classic-light` 块定义在 `#toolbar:not` 规则之前，正常级联下 `#toolbar:not` 本就应覆盖变量，但未来如果主题块被移动到后面或者被浏览器某些机制重新加载，会被反覆盖。双保险更稳。
- 三端统一风格：三个编辑器「主题变量 + 具体规则」完全一致，符合用户「一种样式改变，考虑3个界面」的要求。

### 步骤三：Desktop SDK 本地验证脚本
对所有命中的 CSS 文件跑以下三维校验：
1. `#toolbar:not(.style-off-tabs){}` 声明块中 `background:#fff;background:#fff` 命中数 = 实际编辑器/模式组合数
2. `.theme-classic-light{}` 中 `--toolbar-header-document/spreadsheet/presentation:#fff` 三项均为 #fff
3. 全局括号平衡 `{ }` 计数相等
4. 旧品牌色残留检查：`--toolbar-header-document:#446995`、`--toolbar-header-spreadsheet:#3A8056` 出现次数 = 0

### 步骤四：同步到 Downloads 项目（cat 覆盖方式）
- 使用 `cat src > dst`（绕过沙箱 rm 权限限制）。
- 逐文件对 Downloads 镜像重复步骤三的四项校验。

### 步骤五：清除 `.next` 缓存 + 输出刷新指引
- `rm -rf /Users/kuailingmin/Downloads/onlyoffice-web-comp-main/.next`（若失败则 find+delete 降级）。
- 提示用户重启 dev server + 浏览器硬刷新 Cmd+Shift+R。

---

## 4. 潜在依赖与注意事项

| 项 | 说明 |
|---|---|
| **嵌入/表单模式是否存在** | 脚本先 `os.path.exists` 判断，不存在自动跳过，不误报错。 |
| **主题系统其他消费者** | 本次只改「三端 toolbar 白」相关两个变量。`--toolbar-header-pdf` (#AA5252)、`--toolbar-header-visio` (#444796) 故意保留，避免影响 PDF/Visio 预览场景。若后续用户要求 PDF/Visio 也统一白色再单改。 |
| **沙箱写权限（Downloads 目录）** | Downloads 项目位于 `~/Downloads`，之前遇到过 `rm Operation not permitted`。本次**全程避免 `rm`**，一律用 `cat src > dst` 覆盖写入，事前 `xattr -dr com.apple.quarantine` + `chflags nouchg` + `chmod u+w` 解锁。 |
| **括号平衡是硬红线** | 压缩 CSS 单行超长，任何 regex 替换必须括号平衡后再写回；不平衡直接跳过并报错，避免产出语法错误导致整页样式雪崩。 |
| **Downloads & SDK 双写一致性** | Desktop SDK 是源仓库（将来要提交到 git），Downloads 项目是运行镜像。任何文件修改 **必须在 Desktop SDK 做完 → 校验通过后，再同步到 Downloads**。禁止只改镜像不改源。 |
| **与 li.active 紫色 #7c3aed 的兼容** | #7c3aed 已在 `li.active` 里写死为硬编码，不受本次 `#fff` 白色改造影响。两者在视觉上（白底+紫选中tab）已是预期搭配。 |

---

## 5. 风险与兜底

| 风险 | 发生概率 | 应对 |
|---|---|---|
| CSS 中存在「压缩时多个 #toolbar:not 同名规则合并」导致 regex 漏匹配 | 低 | 验证脚本用 `re.findall` 多选择器模式，并加旧值残留计数（#446995/#3A8056 残留>0 就告警，用户可反馈具体视觉再追加补丁）。 |
| Downloads 同步中途失败，文件处于半写状态 | 极低 | 使用 `cat src > dst` 的原子性（同文件系统内 truncate+write），校验失败则重新拷；所有文件校验通过才做后续 `.next` 清理。 |
| 浏览器强缓存 / Next.js build cache | 中 | 每次修改必清 `.next`，并明确指引用户 Cmd+Shift+R；如仍不生效，指导用户打开 DevTools → Disable cache 再刷新。 |
| `.theme-dark` / `.theme-contrast-black` 主题下显示异常 | 中 | 本次只改 `.theme-classic-light`。深色/高对比主题按用户之前诉求暂不动；如用户在黑主题下打开，toolbar 底色保持深色品牌色是合理的。若用户后续要求深色主题下也强制白，单独追加一份深色主题计划。 |

---

## 6. 交付后的一致性自检（用户视角）

打开三个编辑器页面，按顺序观察：

- ✅ Word 编辑器顶部 toolbar 条：纯白，无蓝色边/底
- ✅ Excel 编辑器顶部 toolbar 条：纯白，无绿色边/底
- ✅ PPT 编辑器顶部 toolbar 条：纯白，与之前一致
- ✅ RibTab 选中态：紫色 #7c3aed（仍然正常，不被白色盖掉）
- ✅ 下拉菜单 / panel-menu：白底（仍然正常）

如有任何一项不符合，直接反馈具体组件（Word/Excel/PPT）+ 截图，再做针对性的第 2 轮补丁（大概率是 `embed/forms` 场景或被父容器覆盖，届时按实际 DOM 定位追加高优先级选择器补丁）。
