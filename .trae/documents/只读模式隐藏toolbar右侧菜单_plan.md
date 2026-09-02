# 只读模式隐藏 toolbar 和右侧菜单 — bridge 扩展方案

## 摘要

在 `onlyoffice-cross-origin-bridge.js` 中扩展只读模式的 UI 控制：当编辑器处于只读模式时隐藏 `#toolbar`（工具板块）和 `#right-menu`（右侧菜单），切换为编辑模式时恢复显示。复用现有 `setReadOnly` → `syncControllerReadOnly` 链路，外围 `getReadOnly()/setReadOnly()/toggleReadOnly()` API 无需改动。初始化只读由外围传 `editorConfig.mode:"view"` 实现，bridge 监听 SDK `document:ready` 事件在文档就绪后主动隐藏 UI。

---

## 一、现状分析（基于 Phase 1 探索）

### 1.1 现有只读切换链路（已完整可用）

```
外围 manager.toggleReadOnly()
  → manager.setReadOnly(!readOnly)              [onlyoffice-manager.ts:233]
  → editor.setReadOnly(readOnly)                [runtime-bridge.ts]
  → postMessage {type:"editor:set-readonly", readOnly}
  → bridge scheduleReadOnly(readOnly, 20)       [bridge:878]
  → bridge setReadOnly(readOnly)                [bridge:852]
    → editor.asc_setRestriction(ASC_RESTRICTION_VIEW/NONE)  [bridge:858-866]
    → editor.asc_setCanSendChanges(!readOnly)               [bridge:868-870]
    → syncControllerReadOnly(readOnly)                      [bridge:871]
      → controller.mode.isEdit = !readOnly                  [bridge:829]
      → controller.mode.canEdit = !readOnly                [bridge:830]
      → controller.editorConfig.mode.isEdit/canEdit 同步    [bridge:837-838]
      → NotificationCenter.trigger("editing:disable", readOnly) [bridge:845]
```

### 1.2 现有问题

| 问题 | 现状 | 影响 |
|---|---|---|
| `syncControllerReadOnly` 不控制 UI | 只设置 `mode` + 触发 `editing:disable`，不操作 `#toolbar`/`#right-menu` DOM | 切换只读/编辑时 toolbar 和右侧菜单始终显示，不随模式变化 |
| bridge 不感知 SDK ready | bridge 没有 `document:ready` 监听，初始化只读时无法主动隐藏 UI | 即使外围传 `mode:"view"` 以只读加载，toolbar/右侧菜单仍显示 |
| `getReadOnly` 在外围读本地状态 | `onlyoffice-manager.ts:216` 读 `this.readOnly`，不从编辑器查询 | 与编辑器实际限制状态可能不一致（可接受，本次不修改外围） |

### 1.3 关键 SDK 事件（已从源码确认）

- `document:ready` — 文档就绪，`Common.NotificationCenter.trigger("document:ready")` 触发
- `editing:disable` — 编辑禁用，bridge 的 `syncControllerReadOnly` 触发
- `layout:changed` — 布局变更，触发后 SDK 调用 `vlayout.doLayout()`/`hlayout.doLayout()` 重算布局
- DOM：`<div id="toolbar" data-layout-name="toolbar">`、`<div id="right-menu" data-layout-name="rightMenu">`

### 1.4 已确认的决策点（用户已答复）

1. **只读模式 → toolbar 和右侧菜单隐藏；编辑模式 → 显示**（与之前 left-menu 逻辑一致）
2. **在 bridge 内扩展**（与现有 setReadOnly 的 postMessage 机制一致，bridge 能直接访问 iframe 内 DOM 和 controller）
3. **初始化只读由外围传 `editorConfig.mode:"view"`**（SDK 原生只读加载，不改 SDK app.js）

---

## 二、拟议修改

### 文件 1：`onlyoffice/9.4.0-develop/web-apps/vendor/onlyoffice-cross-origin-bridge.js`（SDK 仓库，主修改）

#### 修改点 A：扩展 `syncControllerReadOnly`（L824-850）

在 `syncControllerReadOnly` 函数末尾（触发 `editing:disable` 之后）追加 `applyReadOnlyUI(readOnly)` 调用，使每次只读状态同步都同步 UI 显隐。

**修改前**（L824-850 关键结构）：
```js
function syncControllerReadOnly(readOnly) {
  try {
    var common = window.Common;
    var controller = common && common.Controllers && common.Controllers.Main;
    if (controller && controller.mode) {
      controller.mode.isEdit = !readOnly;
      controller.mode.canEdit = !readOnly;
    }
    if (controller && controller.editorConfig && controller.editorConfig.mode) {
      controller.editorConfig.mode.isEdit = !readOnly;
      controller.editorConfig.mode.canEdit = !readOnly;
    }
    if (common && common.NotificationCenter && common.NotificationCenter.trigger) {
      common.NotificationCenter.trigger("editing:disable", readOnly);
    }
  } catch (error) {
    /* best effort */
  }
}
```

**修改后**：在 `trigger("editing:disable", readOnly)` 后追加一行 `applyReadOnlyUI(readOnly);`（在 try 块内，trigger 之后）。

#### 修改点 B：新增 `applyReadOnlyUI(readOnly)` 函数

在 `syncControllerReadOnly` 函数定义之前（L824 前）插入新函数。负责操作 DOM 显隐 + 触发布局重算：

```js
function applyReadOnlyUI(readOnly) {
  try {
    var toolbar = document.getElementById("toolbar");
    var rightMenu = document.getElementById("right-menu");
    var display = readOnly ? "none" : "";
    if (toolbar) { toolbar.style.display = display; }
    if (rightMenu) { rightMenu.style.display = display; }
    var common = window.Common;
    if (common && common.NotificationCenter && common.NotificationCenter.trigger) {
      // 触发布局重算，让编辑区填充 toolbar 隐藏后的空间
      common.NotificationCenter.trigger("layout:changed", "toolbar");
      common.NotificationCenter.trigger("layout:changed", "rightmenu");
    }
  } catch (error) {
    /* best effort */
  }
}
```

设计要点：
- 用 `getElementById` 而非 jQuery（bridge 不依赖 `$`，更稳妥）
- 只读 → `display:none`；编辑 → `display:""`（恢复默认，不写死 `block`/`flex`，让 SDK 原 CSS 决定）
- 触发 `layout:changed` 两次（toolbar + rightmenu），对应 SDK 的 `onLayoutChanged` 分支处理（documenteditor app.js:544227 处 `switch(t){case"rightmenu":...default:vlayout.doLayout()}`）

#### 修改点 C：新增 `installReadOnlyUIPatch()` 函数 + 在 IIFE 末尾调用

在 `installPrintFramePatch` 函数定义之后（L821 后，`syncControllerReadOnly` 之前）插入新函数。负责监听 SDK `document:ready`，在文档就绪后检测只读状态并隐藏 UI：

```js
function installReadOnlyUIPatch() {
  if (window.__ONLYOFFICE_READONLY_UI_PATCHED__) {
    return;
  }

  function applyInitialReadOnlyUI() {
    try {
      var common = window.Common;
      var controller = common && common.Controllers && common.Controllers.Main;
      // mode.isEdit === false 表示只读（外围以 mode:"view" 加载）
      var isReadOnly = !(controller && controller.mode && controller.mode.isEdit);
      applyReadOnlyUI(isReadOnly);
    } catch (error) {
      /* best effort */
    }
  }

  function tryListen(attempt) {
    try {
      var common = window.Common;
      var nc = common && common.NotificationCenter;
      if (nc && typeof nc.on === "function") {
        nc.on("document:ready", applyInitialReadOnlyUI);
        // 兜底：若 SDK 已 ready（迟加载），也立即检测一次
        applyInitialReadOnlyUI();
        return;
      }
    } catch (error) {
      /* best effort */
    }
    if (attempt > 0) {
      window.setTimeout(function () { tryListen(attempt - 1); }, 100);
    }
  }

  // 轮询等待 Common.NotificationCenter 就绪（SDK 加载有先后）
  tryListen(50); // 最多重试 50 次 ≈ 5 秒
  window.__ONLYOFFICE_READONLY_UI_PATCHED__ = true;
}
```

设计要点：
- 用 `__ONLYOFFICE_READONLY_UI_PATCHED__` 标志防重复安装（与现有 `installFetchProxy`/`installPrintFramePatch` 风格一致）
- `tryListen` 轮询等待 `Common.NotificationCenter` 可用（bridge 注入时 SDK 可能尚未初始化 Common）
- 监听 `document:ready` + 兜底立即检测一次（防止 SDK 已 ready 才注入 bridge 的情况）
- `isReadOnly = !controller.mode.isEdit`：与 `syncControllerReadOnly` 的 `controller.mode.isEdit = !readOnly` 对应

#### 修改点 D：在 IIFE 末尾调用 `installReadOnlyUIPatch()`

在 L1889（`installPrintFramePatch();` 之后）、L1890（`})();` 之前）追加一行：

```js
  installFetchProxy();
  installXhrProxy();
  installNamedDownloadPatch(100);
  installPrintFramePatch();
  installReadOnlyUIPatch();   // ← 新增
})();
```

### 文件 2：`/Users/kuailingmin/Downloads/onlyoffice-web-comp-main/public/packages/onlyoffice/9.4.0-develop/web-apps/vendor/onlyoffice-cross-origin-bridge.js`（外部副本 1）

用 `cp` 命令从 SDK 仓库同步覆盖（参考之前同步 app.js 的方式）。

### 文件 3：`/Users/kuailingmin/Downloads/onlyoffice-web-comp-main/scripts/assets/onlyoffice/onlyoffice-cross-origin-bridge.js`（外部副本 2）

同样用 `cp` 同步。注意：此前副本若与副本 1 内容不同（如已有多版本定制），需先 `diff` 确认再决定覆盖还是逐处修改。

### 不需要修改的文件

| 文件 | 原因 |
|---|---|
| 三种编辑器 `main/app.js` 和 `ie/app.js` | 不改 SDK app.js，bridge 直接操作 DOM + 监听 document:ready |
| `apps/api/documents/api.js` | 不暴露 DocEditor.prototype 方法（用户选 bridge 内扩展） |
| 外围 `onlyoffice-manager.ts` / `editor-manager.ts` | 已有 getReadOnly/setReadOnly/toggleReadOnly，无需改动 |

---

## 三、假设与决策

1. **假设**：外围初始化时统一传 `editorConfig.mode:"view"`，三种编辑器（document/spreadsheet/presentation）均以只读加载。`controller.mode.isEdit` 在 view 模式下为 `false`。
2. **假设**：SDK 的 `document:ready` 事件在文档加载完成后一定会触发（已从源码确认 L877370 `on("document:ready", ...)`）。
3. **决策**：`applyReadOnlyUI` 用 `display:""` 恢复编辑模式（而非写死 `block`/`flex`），让 SDK 原 CSS 决定显示方式，避免破坏布局。
4. **决策**：不修改 `getReadOnly` 的外围实现（读本地 `this.readOnly` 状态），本次只补充 UI 显隐逻辑；若后续需要从编辑器查询真实只读状态，再考虑新增 `editor:get-readonly` 消息。
5. **决策**：`toggleReadOnly` 在外围 `onlyoffice-manager.ts:233` 已实现为 `setReadOnly(!this.readOnly)`，调用现有链路即可，bridge 内不需要新增 `toggle-readonly` 消息。

---

## 四、验证步骤

1. **语法校验**：`node --check onlyoffice/9.4.0-develop/web-apps/vendor/onlyoffice-cross-origin-bridge.js`
2. **副本同步校验**：`diff` 确认两份外部副本与 SDK 仓库一致；`node --check` 校验两份副本
3. **运行验证**：
   - 重启 Next.js 服务，硬刷新浏览器
   - 外围以 `mode:"view"` 打开任意 Word/Excel/PPT 文档
   - 预期：加载完成后 toolbar 和右侧菜单隐藏，编辑区填充
4. **切换验证**：
   - 调用 `manager.toggleReadOnly()`（或 `manager.setReadOnly(false)`）切到编辑模式
   - 预期：toolbar 和右侧菜单恢复显示
   - 再调用 `manager.toggleReadOnly()` 切回只读
   - 预期：toolbar 和右侧菜单再次隐藏
5. **回归验证**：
   - 确认编辑模式下 toolbar 所有功能正常（按钮可点击、tab 切换正常）
   - 确认右侧菜单在编辑模式下可正常打开/关闭
   - 确认布局无错位（toolbar 隐藏后编辑区顶部对齐，无空白）
