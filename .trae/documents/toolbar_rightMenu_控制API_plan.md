# toolbar 工具栏与右侧菜单显隐控制 API 实现计划

## 摘要

在 `onlyoffice-web-comp` 组件中新增两套独立的显隐控制 API（toolbar / rightMenu），各提供 `get / set / toggle` 三件套，风格与现有 `setReadOnly / setTheme` 一致。支持本地同源模式（直接操作 iframe 内 DOM + 触发 `layout:changed`）和 CDN 跨域模式（扩展 bridge 消息通道下发指令），与 `setReadOnly` 的双路径架构对齐。

## 当前状态分析

### 既有架构（已探索确认）

1. **门面层 `OnlyOfficeManager`**（`src/components/onlyoffice-web-comp/core/onlyoffice-manager.ts`）：面向业务，方法委托底层 `EditorManager`。已有 `getReadOnly/setReadOnly/toggleReadOnly`、`getTheme/setTheme/toggleTheme` 三件套模式。

2. **底层 `EditorManager`**（`src/components/onlyoffice-web-comp/core/editor-manager.ts`）：
   - 通过 `getEditorFrameWindow()` 获取 iframe `contentWindow`，访问 `Common.NotificationCenter`、`Asc.editor`、DOM。
   - `setReadOnly` 实现分双路径：本地走 `syncEditingRights`（`asc_setRestriction` + `syncShellEditingDisable` 触发 `editing:disable`）；CDN 走 `syncCrossOriginReadOnly` → `setCrossOriginReadOnly`。
   - CDN 判断：`isOnlyOfficeCdnMode()`。

3. **父页侧 bridge**（`src/components/onlyoffice-web-comp/internal/editor/runtime-bridge.ts`）：
   - `CROSS_ORIGIN_BRIDGE_MESSAGE.EDITOR_SET_READONLY = "editor:set-readonly"`
   - `BridgeSession` 有 `pendingReadOnly: boolean | null`，hello 握手时下发。
   - `setCrossOriginReadOnly(frameEditorId, readOnly)` 函数：设 `pendingReadOnly`，bridgeReady 时 postToIframe。

4. **iframe 侧 bridge 脚本**（`scripts/assets/onlyoffice/onlyoffice-cross-origin-bridge.js`）：
   - `BRIDGE_MESSAGE.EDITOR_SET_READONLY`（L9）
   - `applyReadOnlyUI(readOnly)`（L824-L837）：**已有现成模式**——操作 `#toolbar`/`#right-menu` 的 `style.display` + 触发 `Common.NotificationCenter.trigger("layout:changed", "toolbar"/"rightmenu")`。这是本计划本地模式直接参照的模板。
   - `setReadOnly(readOnly)`（L907）：调 `asc_setRestriction` + `applyReadOnlyUI`
   - `scheduleReadOnly(readOnly, retries)`（L933）：重试包装
   - 消息处理（L1917）：`message.type === EDITOR_SET_READONLY → scheduleReadOnly`

5. **导出**：`core/index.ts` 已 `export * from "./editor-manager"` 和 `./onlyoffice-manager"`，新增公共方法自动导出，无需改动。

### 关键 DOM 元素（iframe 内）
- `#toolbar`：工具栏容器
- `#right-menu`：右侧菜单容器
- 显隐控制：`element.style.display = "none"/""` + `Common.NotificationCenter.trigger("layout:changed", "toolbar"/"rightmenu")` 重算布局

## 拟定修改

### 修改 1：iframe 侧 bridge 脚本新增消息处理
**文件**：`scripts/assets/onlyoffice/onlyoffice-cross-origin-bridge.js`

**What/Why**：CDN 跨域模式下，父页通过 postMessage 下发 toolbar/rightMenu 显隐指令，iframe 侧需接收并执行。参照现有 `EDITOR_SET_READONLY` 的完整模式（消息常量 → apply 函数 → schedule 重试 → 消息分支）。

**How**：
1. `BRIDGE_MESSAGE`（L5-L10）新增两个常量：
   ```js
   EDITOR_SET_TOOLBAR_VISIBLE: "editor:set-toolbar-visible",
   EDITOR_SET_RIGHT_MENU_VISIBLE: "editor:set-right-menu-visible",
   ```
2. 在 `applyReadOnlyUI`（L824）附近新增两个独立函数（不耦合只读逻辑）：
   ```js
   function applyToolbarVisible(visible) {
     try {
       var toolbar = document.getElementById("toolbar");
       if (toolbar) { toolbar.style.display = visible ? "" : "none"; }
       var common = window.Common;
       if (common && common.NotificationCenter && common.NotificationCenter.trigger) {
         common.NotificationCenter.trigger("layout:changed", "toolbar");
       }
     } catch (error) { /* best effort */ }
   }

   function applyRightMenuVisible(visible) {
     try {
       var rightMenu = document.getElementById("right-menu");
       if (rightMenu) { rightMenu.style.display = visible ? "" : "none"; }
       var common = window.Common;
       if (common && common.NotificationCenter && common.NotificationCenter.trigger) {
         common.NotificationCenter.trigger("layout:changed", "rightmenu");
       }
     } catch (error) { /* best effort */ }
   }
   ```
3. 在 `scheduleReadOnly`（L933）附近新增两个重试包装（DOM 可能尚未就绪）：
   ```js
   function scheduleToolbarVisible(visible, retries) {
     var toolbar = document.getElementById("toolbar");
     if ((toolbar && window.Common) || retries <= 0) {
       applyToolbarVisible(visible);
       return;
     }
     window.setTimeout(function () {
       scheduleToolbarVisible(visible, retries - 1);
     }, 50);
   }

   function scheduleRightMenuVisible(visible, retries) {
     var rightMenu = document.getElementById("right-menu");
     if ((rightMenu && window.Common) || retries <= 0) {
       applyRightMenuVisible(visible);
       return;
     }
     window.setTimeout(function () {
       scheduleRightMenuVisible(visible, retries - 1);
     }, 50);
   }
   ```
4. 消息处理（L1917 附近）新增两个分支：
   ```js
   if (message.type === BRIDGE_MESSAGE.EDITOR_SET_TOOLBAR_VISIBLE) {
     scheduleToolbarVisible(!!message.visible, 20);
     return;
   }
   if (message.type === BRIDGE_MESSAGE.EDITOR_SET_RIGHT_MENU_VISIBLE) {
     scheduleRightMenuVisible(!!message.visible, 20);
     return;
   }
   ```

### 修改 2：父页侧 bridge 扩展消息通道
**文件**：`src/components/onlyoffice-web-comp/internal/editor/runtime-bridge.ts`

**What/Why**：父页侧需提供下发 toolbar/rightMenu 显隐指令的函数，与 `setCrossOriginReadOnly` 对齐。

**How**：
1. `CROSS_ORIGIN_BRIDGE_MESSAGE`（L597-L602）新增：
   ```ts
   EDITOR_SET_TOOLBAR_VISIBLE: "editor:set-toolbar-visible",
   EDITOR_SET_RIGHT_MENU_VISIBLE: "editor:set-right-menu-visible",
   ```
2. `BridgeSession` 类型（L1088-L1098）新增字段：
   ```ts
   pendingToolbarVisible: boolean | null;
   pendingRightMenuVisible: boolean | null;
   ```
3. `registerCrossOriginBridge`（L1479）初始化新字段为 `null`。
4. 新增两个函数（参照 `setCrossOriginReadOnly` L1512-L1533）：
   ```ts
   export function setCrossOriginToolbarVisible(frameEditorId: string, visible: boolean) {
     const session = sessions.get(frameEditorId);
     if (!session) return false;
     session.pendingToolbarVisible = visible;
     if (!session.bridgeReady) return true;
     postToIframe(session, {
       type: CROSS_ORIGIN_BRIDGE_MESSAGE.EDITOR_SET_TOOLBAR_VISIBLE,
       frameEditorId, visible,
     });
     session.pendingToolbarVisible = null;
     return true;
   }

   export function setCrossOriginRightMenuVisible(frameEditorId: string, visible: boolean) {
     const session = sessions.get(frameEditorId);
     if (!session) return false;
     session.pendingRightMenuVisible = visible;
     if (!session.bridgeReady) return true;
     postToIframe(session, {
       type: CROSS_ORIGIN_BRIDGE_MESSAGE.EDITOR_SET_RIGHT_MENU_VISIBLE,
       frameEditorId, visible,
     });
     session.pendingRightMenuVisible = null;
     return true;
   }
   ```
5. hello 握手分支（L1437-L1444，`session.pendingReadOnly !== null` 之后）追加下发 pending：
   ```ts
   if (session.pendingToolbarVisible !== null) {
     postToIframe(session, {
       type: CROSS_ORIGIN_BRIDGE_MESSAGE.EDITOR_SET_TOOLBAR_VISIBLE,
       frameEditorId,
       visible: session.pendingToolbarVisible,
     });
     session.pendingToolbarVisible = null;
   }
   if (session.pendingRightMenuVisible !== null) {
     postToIframe(session, {
       type: CROSS_ORIGIN_BRIDGE_MESSAGE.EDITOR_SET_RIGHT_MENU_VISIBLE,
       frameEditorId,
       visible: session.pendingRightMenuVisible,
     });
     session.pendingRightMenuVisible = null;
   }
   ```

### 修改 3：EditorManager 底层新增显隐控制
**文件**：`src/components/onlyoffice-web-comp/core/editor-manager.ts`

**What/Why**：底层编辑器管理器提供 toolbar/rightMenu 显隐控制，本地模式直接操作 DOM，CDN 模式走 bridge。与 `setReadOnly` 双路径架构对齐。

**How**：
1. 类属性（L175 附近，`private readOnly = false;` 之后）新增：
   ```ts
   private toolbarVisible = true;
   private rightMenuVisible = true;
   ```
2. 新增 CDN 同步方法（参照 `syncCrossOriginReadOnly` L337）：
   ```ts
   private syncCrossOriginToolbarVisible(visible: boolean, retries = 10) {
     if (setCrossOriginToolbarVisible(this.containerId, visible)) return true;
     // …参照 syncCrossOriginReadOnly 的重试逻辑
   }
   private syncCrossOriginRightMenuVisible(visible: boolean, retries = 10) {
     if (setCrossOriginRightMenuVisible(this.containerId, visible)) return true;
     // …同上
   }
   ```
   需 import `setCrossOriginToolbarVisible` / `setCrossOriginRightMenuVisible`。
3. 新增本地 apply 方法（参照 iframe 侧 `applyReadOnlyUI` 模式）：
   ```ts
   private applyLocalToolbarVisible(visible: boolean) {
     const win = this.getEditorFrameWindow();
     const doc = win?.document;
     const toolbar = doc?.getElementById("toolbar");
     if (toolbar) { toolbar.style.display = visible ? "" : "none"; }
     win?.Common?.NotificationCenter?.trigger?.("layout:changed", "toolbar");
   }
   private applyLocalRightMenuVisible(visible: boolean) {
     const win = this.getEditorFrameWindow();
     const doc = win?.document;
     const rightMenu = doc?.getElementById("right-menu");
     if (rightMenu) { rightMenu.style.display = visible ? "" : "none"; }
     win?.Common?.NotificationCenter?.trigger?.("layout:changed", "rightmenu");
   }
   ```
4. 新增公共方法（同步，无需 await——不涉及文档保存）：
   ```ts
   getToolbarVisible() { return this.toolbarVisible; }
   setToolbarVisible(visible: boolean) {
     if (this.toolbarVisible === visible) return;
     this.toolbarVisible = visible;
     if (isOnlyOfficeCdnMode()) {
       this.syncCrossOriginToolbarVisible(visible);
       return;
     }
     this.applyLocalToolbarVisible(visible);
   }
   toggleToolbarVisible() {
     this.setToolbarVisible(!this.toolbarVisible);
     return this.toolbarVisible;
   }

   getRightMenuVisible() { return this.rightMenuVisible; }
   setRightMenuVisible(visible: boolean) {
     if (this.rightMenuVisible === visible) return;
     this.rightMenuVisible = visible;
     if (isOnlyOfficeCdnMode()) {
       this.syncCrossOriginRightMenuVisible(visible);
       return;
     }
     this.applyLocalRightMenuVisible(visible);
   }
   toggleRightMenuVisible() {
     this.setRightMenuVisible(!this.rightMenuVisible);
     return this.rightMenuVisible;
   }
   ```
   需确认 `isOnlyOfficeCdnMode` 已 import（setReadOnly 已用，确认存在）。

### 修改 4：OnlyOfficeManager 门面层委托
**文件**：`src/components/onlyoffice-web-comp/core/onlyoffice-manager.ts`

**What/Why**：业务调用入口，与 `getReadOnly/setReadOnly/toggleReadOnly` 并列。

**How**：在 `toggleReadOnly`（L233）之后、`getLanguage`（L237）之前新增：
```ts
getToolbarVisible() { return this.editor.getToolbarVisible(); }
setToolbarVisible(visible: boolean) { this.editor.setToolbarVisible(visible); }
toggleToolbarVisible() { return this.editor.toggleToolbarVisible(); }

getRightMenuVisible() { return this.editor.getRightMenuVisible(); }
setRightMenuVisible(visible: boolean) { this.editor.setRightMenuVisible(visible); }
toggleRightMenuVisible() { return this.editor.toggleRightMenuVisible(); }
```

### 修改 5：导出
**文件**：`src/components/onlyoffice-web-comp/core/index.ts`
无需修改——已 `export * from "./editor-manager"` 和 `./onlyoffice-manager"`，新增公共方法自动从组件入口导出。需确认顶层 `src/components/onlyoffice-web-comp/index.ts` 是否 re-export `core`（若存在则同样自动透出）。

## 假设与决策

1. **同步方法**：`setToolbarVisible/setRightMenuVisible` 设计为同步（非 async），因为仅操作 DOM + postMessage 即发即弃，不涉及文档快照保存（`setReadOnly` 为 async 是因为切只读前需 `captureDocumentSnapshot`）。`toggle` 返回新值。
2. **状态独立**：`toolbarVisible`/`rightMenuVisible` 与 `readOnly` 状态解耦，互不影响。`setReadOnly` 的 `applyReadOnlyUI` 会联动改 toolbar/right-menu display，但不会回写这两个状态字段——这是已知行为，业务若先 `setReadOnly(true)` 再 `getToolbarVisible()` 仍返回 true（状态未同步）。可接受：readOnly 是编辑权限语义，toolbarVisible 是 UI 显隐语义，二者正交。
3. **重试策略**：iframe 侧 `scheduleToolbarVisible/scheduleRightMenuVisible` 复用 `scheduleReadOnly` 的 20 次 ×50ms 重试模式，应对 DOM 未就绪场景。
4. **bridge 握手时序**：pending 状态在 hello 握手时下发，与 `pendingReadOnly` 同模式，确保 bridge 就绪前调用的显隐指令不丢失。
5. **不动 setReadOnly 既有逻辑**：不修改 `applyReadOnlyUI`，避免影响只读联动。新增的 `applyToolbarVisible/applyRightMenuVisible` 是独立函数。
6. **stars-onlyoffice-sdk 项目副本**：`scripts/assets/onlyoffice/onlyoffice-cross-origin-bridge.js` 是 web-comp 项目的源文件；stars-onlyoffice-sdk 里的 `vendor/onlyoffice-cross-origin-bridge.js` 是 SDK 静态资源副本，本计划不修改 SDK 副本（由 web-comp 构建流程同步）。

## 验证步骤

1. **类型检查**：`pnpm tsc --noEmit` 确认无类型错误。
2. **本地模式验证**：启动 `pnpm dev`，打开 Word/Excel/PPT 文档：
   - `manager.setToolbarVisible(false)` → 工具栏隐藏，编辑区上移无错位
   - `manager.setToolbarVisible(true)` → 工具栏恢复
   - `manager.toggleToolbarVisible()` → 切换并返回新状态
   - 同理验证 `setRightMenuVisible` / `toggleRightMenuVisible`
   - 三个编辑器（documenteditor/spreadsheeteditor/presentationeditor）均验证
3. **CDN 跨域模式验证**：`OnlyOfficeManager.registerStaticResource({ cdnOrigin: ... })` 后创建编辑器，重复步骤 2，确认 bridge 通道下发生效。
4. **与 setReadOnly 交互验证**：`setReadOnly(true)` 后 `setToolbarVisible(true)` 应能单独显示工具栏（只读但工具栏可见）；反之亦然。确认二者无互相破坏。
5. **bridge 握手时序验证**：CDN 模式下，bridge 未就绪时调用 `setToolbarVisible(false)`，确认 bridgeReady 后工具栏正确隐藏（pending 不丢失）。
6. **回归验证**：`setReadOnly` / `setTheme` / `setLanguage` 原有行为不受影响。
