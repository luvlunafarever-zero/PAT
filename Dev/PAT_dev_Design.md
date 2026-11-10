# Prompt Assembly Tool (PAT) 详细设计文档

> 本文档对应 **v2.0**。v2.0 将模板编辑从「Settings Tab」迁移至「模板管理」全屏浮层界面，并新增模板排序、配置导出/导入等功能。

## 1. 系统概述

**Prompt Assembly Tool (PAT)** 是一个单文件静态 HTML 小工具，旨在帮助用户保存、管理和复用常用的 Prompt 模板。通过将用户输入动态嵌入预设模板的 `${user_input}` 占位符中，快速生成结构化的完整 Prompt。

### 1.1 技术架构
- **单文件模式**：HTML 结构、CSS 样式、JavaScript 逻辑全部内联在一个 `.html` 文件中，无需构建工具或服务器环境，支持 `file://` 协议直接打开。
- **数据持久化**：使用浏览器 `localStorage` 存储用户自定义的模板配置和内容。
- **多语言支持 (i18n)**：采用集中式常量 + 属性标记的轻量级方案，无需外部依赖；CJK 字体栈由 CSS 变量 `--font-sans-cjk` 集中管理。
- **全屏浮层页面 (mode-page)**：模板管理 / 排序 / 导出导入采用同一套「浮层页面」机制（半透明遮罩 + 中央卡片 + 进入动画），主界面仅保留「问题」输入页。

---

## 2. 核心数据结构

### 2.1 模板配置对象 (TEMPLATE_CONFIGS)

每个模板由一个配置对象描述，存储在全局数组 `TEMPLATE_CONFIGS` 中，并持久化到 `localStorage`。

```javascript
{
  key: string,            // localStorage 中存储模板内容的键名 (如 'LS_TEMPLATE_CONTENT_CN3')
  tabId: string,          // 模板唯一标识 (如 'settings3')，用于界面事件路由、选择记忆等
  name: string,           // 模板显示名称 (用户可编辑，最多30字符)
  defaultContent: string, // 「重置所有模板」时恢复的兜底内容
  editTime: string        // 最近一次保存时间戳 (格式 "YYYY-MM-DD HH:mm"，
                          //   仅当模板内容经确认保存成功后写入；空字符串表示从未保存过)
}
```

> **v2.0 变更**：配置对象中移除了 `textareaId` / `countId` / `resetAction` / `saveAction` 字段（v1.2 中这些字段用于给每个 Settings Tab 生成独立的 DOM 与按钮；v2.0 中所有模板共用管理界面内的同一个编辑器，不再需要）。新增 `editTime` 字段。

### 2.2 localStorage 键值设计

| 键名 | 内容 | 说明 |
|------|------|------|
| `LS_TEMPLATE_CONFIGS_META` | JSON 字符串 | 存储 `TEMPLATE_CONFIGS` 数组（模板元数据） |
| `LS_TEMPLATE_CONTENT_CN` | 字符串 | 第1个模板的实际内容 |
| `LS_TEMPLATE_CONTENT_CN2` | 字符串 | 第2个模板的实际内容 |
| `LS_TEMPLATE_CONTENT_CN{n}` | 字符串 | 后续新增模板的实际内容（`getNextTemplateNumber()` 自动采番，如 `CN3`） |

历史常量 `LS_TAB_NAME_PREFIX`（模板名单独存储的键名前缀）已不再用于实际读写——模板名统一内嵌在 `TEMPLATE_CONFIGS` 中，常量仅保留结构约定。

**旧数据兼容**：`loadTemplateConfigs()` 读取到缺失 `editTime` 字段的旧配置时，自动补充为空字符串，不做迁移提示。

---

## 3. 核心模块与处理逻辑

### 3.1 页面初始化流程

页面加载完成后 (`DOMContentLoaded`)，按以下顺序执行初始化：

```
1. applyTexts()              → 应用 i18n 文字到静态 HTML 元素
2. loadTemplateConfigs()     → 从 localStorage 加载模板配置（含 editTime 兼容补全）
3. 元素引用初始化             → 管理界面 / 排序界面 / 导出导入界面的元素引用与滚动监听
4. 管理界面编辑区事件         → 自动增高、字数统计、脏状态更新 (updateManagerDirtyState)
5. 管理界面名称输入框事件      → 名称字数统计
6. renderSettingsTabs()      → 初始化各模板在 localStorage 中的内容 + 渲染管理界面列表
7. setupTextarea()           → 为主输入框 (main) 绑定自动增高和字数统计事件
8. updateIndicator()         → 初始化 Tab 滑动指示器位置（含窗口 resize 监听）
```

> `renderSettingsTabs()` 的函数名沿用自 v1.2（当时负责渲染动态 Settings Tab），v2.0 中其职责变为：① 确保每个模板在 `localStorage` 中有内容（为空则写入 `defaultContent`）；② 渲染管理界面左侧列表并选中当前编辑项（或第一项）。

### 3.2 i18n 多语言处理模块

#### 3.2.1 设计原则
- **集中定义**：所有界面文字定义在 `TEXTS` 常量对象中。
- **属性标记**：静态 HTML 元素通过 `data-i18n` 等属性标记，无需修改 DOM 结构。
- **保留原文**：HTML 中保留默认语言文字，作为 fallback 和代码可读性保障。
- **字体统一管理**：CJK 字体集中于 `:root` 的 `--font-sans-cjk` 变量，发布其他语言版本时仅需修改此变量（及 `<html lang>`）。

#### 3.2.2 标记类型

| 属性 | 替换目标 | 使用场景 |
|------|----------|----------|
| `data-i18n="key"` | `el.textContent` | 普通文本（按钮、标签） |
| `data-i18n-html="key"` | `el.innerHTML` | 含 HTML 标签的文本（`<br>`、`<font>`） |
| `data-i18n-aria="key"` | `el.setAttribute('aria-label', ...)` | 无障碍标签 |

#### 3.2.3 带变量消息处理

对于包含动态数据（如模板名称、导入数量）的消息，使用 `{name}` / `{count}` 占位符：

```javascript
// TEXTS 定义
snackbarArgMsg_TemplateCreated: "新模板【{name}】已创建"

// 运行时替换
function formatText(str, vars) {
  return str.replace(/\{(\w+)\}/g, (_, key) => vars[key] ?? '');
}
showSnackbar(formatText(TEXTS.snackbarArgMsg_TemplateCreated, { name: config.name }));
```

#### 3.2.4 动态生成内容的处理

对于 JS 动态生成的 HTML（如管理界面列表项、排序卡片），在模板字符串中直接引用 `TEXTS`：

```javascript
card.innerHTML = `
  <button aria-label="${TEXTS.aria_MoveTop} ${cfg.name}">…</button>
`;
```

### 3.3 Tab 管理模块（v2.0 调整）

- **v1.2 模式（已废弃）**：`main` 标签页 + 每个模板一个动态生成的 Settings Tab（`createTabButton` / `createTabContent`），通过滑动动画切换。
- **v2.0 模式**：`.tabs-wrapper` 中仅保留静态的 `main`（问题输入）标签；所有模板的新增/删除/重命名/内容编辑集中在「模板管理」全屏浮层界面中完成（见 3.4）。
- Tab 切换框架（事件委托 + Grid 堆叠 + `translateX` 动画 + 滑动指示器 `updateIndicator`）仍保留在代码中，`checkTemplateBeforeSwitchTabs()` 也随之保留——目前 `currentTab` 恒为 `'main'`，不会产生实际提示。

### 3.4 模板管理界面模块（mgr-overlay，v2.0 新增）

#### 3.4.1 界面结构

管理界面是 `.mgr-overlay.mode-page` 全屏浮层（z-index: 150），半透明遮罩 + 背景模糊，面板与主卡片观感一致但层级更深：

```
manager-header
├── 标题「模板管理」+ 模板数量 count-chip
├── 「重置所有模板」⭯ 图标按钮（危险色）
├── 「导出 / 导入」入口按钮 (btnOpenIo)
└── 返回按钮 (closeManager)
manager-body
├── tpl-list-panel（左侧）
│   ├── add-row：排序入口按钮 (btnOpenSort) + 「新增模板」按钮 (btnCreateNewTpl)
│   └── tpl-list：模板列表项（名称+未保存红点 / 字数 + editTime）
└── tpl-editor（右侧）
    ├── editor-header：名称输入框（只读/编辑两态）、字数 0/30、
    │                    编辑名称✎ / 保存名称✓ / 删除模板🗑 / 放弃编辑↩ 按钮
    ├── settings-section：字数统计+说明、`${user_input}` 插入按钮、「回退」「保存」按钮
    └── textarea：模板内容编辑区（auto-grow）
```

#### 3.4.2 打开 / 关闭流程

```
打开 (openManager):
1. 强制 mode-page；双 rAF 后添加 .active 触发进入动画
2. 锁定 body 滚动；遮罩滚动回顶部
3. applyMgrTplListPanelHeight() → 大屏(>900px)时计算左侧列表面板高度范围，
   写入 --mgr-list-panel-min-height / --mgr-list-panel-max-height
   （最小高度：与空编辑区底部齐平；最大高度：避免列表过长导致整页滚动；仅打开时计算一次）
4. renderManagerList() 并选中第一个模板

关闭 (closeManager / ESC):
1. beforeLeaveCurrentTemplate() 守卫：名称编辑中 → 阻止并提示；
   内容有未保存变动 → 暂存草稿并提示（不阻止）
2. 移除 .active、恢复 body 滚动、清除面板高度变量、刷新回到顶部按钮
```

**点击遮罩不关闭**（防止误触丢失上下文），仅通过返回按钮或 ESC 关闭。

#### 3.4.3 列表渲染与选中 (renderManagerList / selectTemplateInManager)

- 列表项：模板名（含未保存红点 `.dirty-dot`）、内容字数、`editTime`。
- 脏判断：当前编辑项以编辑区内容 ≠ 基线内容判定；非当前项以 `unsavedDrafts[tabId] !== undefined` 判定。
- 点击列表项切换模板：同样经过 `beforeLeaveCurrentTemplate()` 守卫（阻止名称编辑中切换、暂存上一模板草稿、提示未保存）。
- 加载逻辑：编辑区填充 **草稿**（若有）或 **基线内容**（最近一次已保存内容，无保存内容时用 `defaultContent` 兜底）。

#### 3.4.4 草稿与未保存机制

- `currentBaselineContent`：当前编辑模板的基线（最近一次已保存内容），由 `getTemplateBaselineContent()` 提供。
- `unsavedDrafts`（仅内存，不持久化）：以 `tabId` 为 key 暂存各模板的未保存内容；草稿为空时清除（空内容不作为有效草稿）。
- `storeCurrentDraftIfDirty()`：对比编辑区与基线（去首尾空白），有变动且非空时写入草稿。
- 组装 Prompt 时只读取 `localStorage` 中已保存的版本——草稿不会影响组装结果（未保存提示中也包含此说明）。

#### 3.4.5 模板名称内联编辑

```
1. 点击 ✎ (enterNameEditMode)：输入框进入编辑态（.editing 加粗描边），
   ✎ → ✓（保存），🗑（删除）位置替换为 ↩（放弃）
2. 保存 (saveTemplateName)：trim 后校验非空、≤30 字符 → 写回配置并持久化
3. 放弃 (revertTemplateName)：恢复 originalNameBeforeEdit
4. 名称编辑期间禁止切换模板 / 关闭管理界面 / 进入排序或导出导入界面
   （isTemplateNameEditingBlocking 统一拦截）
```

#### 3.4.6 内容保存 (checkAndOpenSaveDialog → confirmDialog)

```
1. 内容校验：
   - 空内容 → 恢复基线内容并提示（不弹对话框）
   - 缺少 ${user_input} → 阻止保存并提示
2. 通过校验 → openDialog('save')，正文显示模板名
3. 确认后：
   a. safeSetItem(config.key, content) 写入模板内容
   b. 更新基线、清除该模板草稿
   c. 写入编辑时间戳 editTime = formatEditTimestamp()（"YYYY-MM-DD HH:mm"），
      随 TEMPLATE_CONFIGS 一并持久化（列表上的最近保存时间即此字段）
   d. 刷新列表与脏状态
```

#### 3.4.7 内容回退 (checkAndRequestRevert)

v1.2 中单模板的「重置」（恢复 `defaultContent`）已取消，替换为「回退」：

```
1. 判断是否有未保存变动（存在草稿，或编辑区内容 ≠ 基线）：
   - 无变动 → 直接提示"没有未保存的修改"
   - 有变动 → openDialog('revert')
2. 确认回退：编辑区恢复为基线内容、清除草稿、刷新列表与脏状态
```

#### 3.4.8 新增 / 删除 / 重置所有模板

- **新增 (handleAddNewTab → executeAddNewTab)**：弹出 `add_tab` 确认框 → `getNextTemplateNumber()` 采番生成配置（`key: LS_TEMPLATE_CONTENT_CN{n}`、`tabId: settings{n}`、`name: 模板{n}`、`defaultContent: ''`、`editTime: ''`）→ 推入配置数组并持久化 → 初始化内容键 → 刷新列表并选中新模板。
- **删除 (confirmDeleteFromManager → deleteTemplate)**：`delete_confirm` 确认框（含模板名）→ 删除 `localStorage` 内容键（清理残留数据）→ 从数组移除 → 清除相关草稿 → 刷新列表。
- **重置所有 (resetToDefaultConfigs → executeResetToDefault)**：`reset_all` 确认框（富文本提示默认配置需在文件的 `DEFAULT_TEMPLATE_CONFIGS` 中手工设定）→ 删除全部模板内容键 → 用 `DEFAULT_TEMPLATE_CONFIGS` 深拷贝覆盖配置 → 清空草稿/基线/当前编辑项 → 退出名称编辑态 → 重新渲染。

### 3.5 模板排序模块（sort-overlay，v2.0 新增）

- **入口与守卫**：管理界面左侧排序按钮；模板少于 2 个时提示 `snackbar_TooFewToSort` 不打开；打开前执行 `beforeLeaveCurrentTemplate()`（与切换模板同规则）。
- **结构**：与管理页同机制的浮层页面（独立 `.sort-overlay`，z-index: 150），点击遮罩不关闭；卡片显示序号徽章、模板名、内容字数与三个移动按钮：
  - 置顶（琥珀色）：仅 `idx >= INT_TOP_THRESHOLD (=4)`（即第 5 个）起启用，前 4 个用上移即可；
  - 上移（绿色）/ 下移（玫红色）；首/尾项对应按钮 `disabled` 置灰（不隐藏，保持按钮位置恒定）。
- **移动与持久化**：`moveTemplate()` / `moveTemplateToTop()` 直接交换/移动 `TEMPLATE_CONFIGS` 数组元素后立即 `saveTemplateConfigs()`——排序结果即数组顺序，修改即时生效且无需确认。
- **FLIP 动画**：先记录各卡片位置（First），重渲染后按位移反向偏移（Invert），再以过渡动画归位（Play）；`isSortAnimating` 作为动画期间渲染锁。
- **焦点保持**：移动后焦点转移到新位置的同方向按钮（到边界则取反向），支持键盘连续操作。
- **关闭**：按钮或 ESC；关闭时以新顺序重渲染管理列表，并恢复进入排序前选中的模板项（`pendingSortSelectedTabId`，含最小化滚动定位）。

### 3.6 模板配置导出 / 导入模块（io-overlay，v2.0 新增）

独立浮层页面（`.io-overlay`，z-index: 155，位于管理/排序遮罩之上），仅从管理界面进入；点击遮罩不关闭。

#### 3.6.1 导出 (ioExportConfigs)

```javascript
{
  "version": "2.0",                 // 取自文件内 version 常量
  "exportedAt": "…±HH:MM",          // 本地时区 ISO 时间（toIsoLocalString）
  "templates": [                    // 仅导出 name / content，数组顺序即模板顺序
    { "name": "模板名", "content": "模板内容（含 ${user_input}）" }
  ]
}
```

导出结果写入 JSON 编辑区（等宽字体，`spellcheck="false"`）并提示点击「复制」。不导出 `key` / `tabId` 等 localStorage 实现细节。

#### 3.6.2 导入 (ioRequestImport → ioExecuteImport)

点击「导入」按钮触发 `ioRequestImport()`，全部校验**同步、顺序**执行，且为**快速失败（fail-fast）**：任一步失败立即终止并以 Snackbar 提示（不弹出确认框），消息文案对应 `TEXTS` 中的 `snackbar_Io*` 键。只有全部通过才会进入确认对话框。

**第 1 步：非空检查**
- 判定：`ioJsonTextarea.value.trim()` 为空
- 失败消息：`请先输入符合格式要求的 JSON 内容`（并将焦点设回编辑区）

**第 2 步：JSON 解析**
- 判定：`JSON.parse(text)` 成功
- 失败消息：`JSON 解析失败，请检查格式`（任何语法错误：缺引号、尾逗号、非法字符等）

**第 3 步：提取模板数组**
- 提取优先级：
  1. 解析结果本身是数组 → 直接采用（顶层数组结构 `[...]`）；
  2. 解析结果是对象且其 `templates` 属性为数组 → 采用（`{"templates":[...]}` 结构）;
  3. 均不满足 → 失败。
- 提取结果为**空数组**（`length === 0`）同样判为失败
- 失败消息：`未找到模板数组：需 {"templates":[...]} 结构（或顶层数组）`
- **不参与校验的字段**：导出时写入的 `version` / `exportedAt`（及任何其它附加字段）仅作信息展示，不做任何检查

**第 4 步：逐条校验**（按数组顺序，提示信息中的 `{index}` 从 **1** 开始计数）

| 顺序 | 检查项 | 判定条件 | 失败消息 |
|------|--------|----------|----------|
| ① | 条目为普通对象 | `item !== null && typeof item === 'object'` | `第 {index} 条无效：需含非空 name(≤30字符) 与字符串 content` |
| ② | name 有效 | `typeof item.name === 'string'` 且 trim 后非空 | 同上 |
| ③ | content 为字符串 | `typeof item.content === 'string'`（**允许空字符串**） | 同上 |
| ④ | name 长度 | `item.name.trim().length ≤ 30` | `第 {index} 条模板名称超过30字符` |

- ①②③ 共用一条失败消息：结构缺失 / name 非字符串或为空 / content 非字符串，任一不满足都会报出该消息（无法区分具体是哪一项）；④ 为独立的长度检查消息
- **遇到首个无效条目即终止**：立即返回，不继续检查后续条目，也不汇总所有错误条目
- 校验不对数据做任何就地修改（原数组保留；name 的去除首尾空白在导入执行时才生效）

**第 5 步：暂存与确认**
- 全部通过后：`pendingIoImportList = list`，打开 `io_import_confirm` 对话框，正文为：
  `将导入 {count} 个新模板（不会覆盖已有模板）。确定要导入吗？`（`{count}` 取数组长度）
- 用户取消则不发生任何数据变更，下次导入需重新点击并重新执行全套校验

**第 6 步：确认后执行导入 (ioExecuteImport)**
- 导入列表整体按**新模板**处理，不做合并：
  - 逐条经 `getNextTemplateNumber()` 采番，生成 `key: LS_TEMPLATE_CONTENT_CN{n}` / `tabId: settings{n}`；
  - 显示名称取 `name.trim()`；
  - 导入的 `content` 原样写入内容键，同时作为该模板的 `defaultContent`；
  - `editTime` 置空（视为尚未经过编辑保存）
- 循环结束后一次性 `saveTemplateConfigs()` 持久化元数据、刷新管理界面列表，并提示 `已导入 {count} 个新模板`

**规则与边界说明**：
- 导入为纯追加：**不检查重名**，不覆盖、不合并现有同名模板（同名模板将作为多个独立条目并存）；导入数据内部的多条同名记录也会各自导入
- **不强制检查 `${user_input}` 标记**：缺少该标记的模板也能成功导入；但后续在管理界面编辑该模板时，未补上标记将无法保存；若直接用于组装，结果即模板原文（输入内容不会被嵌入）
- 校验时机仅在「导入」按钮点击时执行一次；「导出」与编辑区的其它操作不触发校验

#### 3.6.3 辅助操作

- 清除（橡皮擦图标）：直接清空编辑区，不弹确认。
- 复制：仅在有内容时执行 `clipboard.writeText`，复用主界面的复制成功/失败提示。

### 3.7 Prompt 组装流程

#### 3.7.1 触发入口

点击「组装 Prompt」按钮 → `assemblePrompt()`：主输入框为空（去空白后）时提示并聚焦，不弹对话框。

#### 3.7.2 模板选择对话框 (tplSelectOverlay，v2.0 独立化)

v1.2 中模板选择共用通用确认对话框（`select_template` action），v2.0 拆分为独立对话框：

- **布局**：标题居左、「取消/确认」按钮位于标题行右侧（模板极多时无需滚动到底部操作）；卡片宽度随视口分档限制，与主容器响应式宽度一致（≤900px 时默认 90%）。
- **定位**：打开时测量 `#main-textarea` 视口顶边，钳制到 `[24, 视口高度 40%]` 后写入 `--tpl-select-top`，使卡片上边沿大致对齐输入框上边沿；测量失败时回退静态值 `191px`。
- **选择记忆**：以 `lastSelectedTemplateTabId` 记忆，重新打开时恢复选中（按 `tabId` 而非索引，排序后不会选错模板）。
- **交互**：点击遮罩仅抖动动画不关闭（与通用对话框一致），通过取消/确认/ESC 关闭。

#### 3.7.3 处理流程

```
1. 用户选中模板后点击「确认」（未选中则提示"请先选择一个模板"）
2. 从 localStorage 读取选中模板的内容（仅已保存版本）：
   - 内容为空 → 提示"模板内容为空"
3. 将模板中所有 ${user_input} 替换为主输入框内容：
   content.split('${user_input}').join(userInput)
4. 记忆选中模板、关闭选择对话框、打开 Drawer 展示结果
```

> v1.2 中主输入框的「嵌入模板后字数预览」（括号内各模板预估）已在当时停用（`updateExtraCount` 注释保留），v2.0 仅显示输入字数与 Token 换算说明。

### 3.8 Dialog 交互模块

#### 3.8.1 配置表驱动 (DIALOG_PRESETS)

v1.2 的 `openDialog` 按 action 展开 if-else 分支；v2.0 收敛为 `DIALOG_PRESETS` 配置表——每种对话框声明 `title` / `messageText`(或 `messageHtml`) / `showInput` / `showDeleteBtn` / `showChips` / `onOpen`，未登记的 action 回退到默认 `confirm` 预设。新增对话框类型只需在表中加一条配置。

当前 action 一览：

| action | 用途 | 特殊元素 |
|--------|------|----------|
| `save` | 保存模板内容确认 | 无 |
| `revert` | 回退模板内容确认 | 无 |
| `edit_tab_name` | 编辑模板名（保留自 v1.2；v2.0 改名走管理界面内联编辑） | 输入框 + 删除按钮 |
| `delete_confirm` | 删除模板确认 | 无 |
| `add_tab` | 新增模板确认 | 无 |
| `reset_all` | 重置所有模板确认 | 富文本提示 |
| `io_import_confirm` | 导入确认（显示导入条数） | 无 |
| `clear_confirm` | 清除主输入框确认 | 无 |
| `confirm` | 默认通用确认 | 无 |

> 模板选择（`select_template`）已独立为 3.7.2 的专用对话框。

#### 3.8.2 状态变量

- `currentDialogAction`：当前对话框类型，供确认回调分派。
- `currentDialogTabId`：当前操作关联的模板 `tabId`。
- `pendingIoImportList`：导入确认前暂存的待导入列表。

#### 3.8.3 交互细节

- 点击遮罩触发抖动动画（不关闭）。
- 「取消」/「确认」/ ESC 关闭。

#### 3.8.4 ESC 关闭优先级（按键时的顶层优先）

```
Drawer → 确认对话框 → 模板选择对话框 → 导出/导入页 → 排序页 → 模板管理页
```

导出/导入页与排序页关闭后返回管理页，保持 `body` 滚动锁定；仅退出管理页时恢复。

### 3.9 Drawer 结果展示模块

#### 3.9.1 打开流程

```
1. 添加 .active 类触发滑入动画
2. 禁止 body 滚动
3. 自动调整 textarea 高度
4. 更新字数统计
5. 初始化"回到顶部"按钮显示状态
```

#### 3.9.2 复制功能

使用 `navigator.clipboard.writeText()` API：
- 成功：显示"已复制" Snackbar
- 失败：显示"复制失败，请手动复制" Snackbar

#### 3.9.3 关闭方式

- 点击关闭按钮
- 点击遮罩层
- 按 `ESC` 键

### 3.10 辅助功能模块

#### 3.10.1 主输入框清除

「清除」按钮（橡皮擦图标）→ `requestClearMainTextarea()`：输入为空时不响应；否则弹出 `clear_confirm` 确认，确认后清空、触发 `input` 事件更新统计并重新聚焦。

#### 3.10.2 `${user_input}` 标记插入

编辑区说明区的 `${user_input}` 按钮（`btnInsertUserInputMark`）：
- 按钮 `onmousedown` 阻止默认（避免编辑区失焦），点击时要求编辑区正处于聚焦状态才执行；
- 在光标处插入标记（存在选中文本时替换选区），光标置于标记末尾；
- 触发 `input` 事件，走统一的字数统计/自动增高/脏状态更新路径。

#### 3.10.3 Snackbar 消息提示

```
1. 设置消息文本
2. 重置计时条动画（通过 cloneNode 替换）
3. 添加 .active 类触发滑入动画
4. 4秒后自动滑出
```

#### 3.10.4 回到顶部 FAB（多实例分层）

各浮层的遮罩层级高于页面，单一 FAB 会被遮挡，因此按层级设置 4 个独立的回到顶部 FAB：

| FAB | 归属 | z-index | 滚动监听对象 |
|-----|------|---------|--------------|
| `fabTop` | 主界面 | 100 | `window` |
| `mgrFabTop` | 模板管理页 | 160 | `.mgr-overlay` |
| `sortFabTop` | 排序页 | 160 | `.sort-overlay` |
| `ioFabTop` | 导出/导入页 | 165 | `.io-overlay` |

`toggleFab()` 按 **导出/导入 > 排序 > 管理 > 主界面** 的互斥优先级，决定当前由哪个 FAB 响应滚动显示（滚过 100px 显示）。

主界面左上角绿色小 FAB（复用 `fab-reset-default` 样式）在 v1.2 中是「重置为默认配置」入口，v2.0 中改为「模板管理」入口（`openManager('page')`）；「重置所有模板」移入管理页头部。

#### 3.10.5 Textarea 自动增高

监听 `input` 事件，动态设置 `height = scrollHeight`，实现内容自适应高度。适用于：主输入框、管理界面模板编辑区、导出导入 JSON 编辑区、Drawer 结果区（打开时调整）。

#### 3.10.6 Ripple 波纹效果

按钮点击时，在点击位置创建圆形波纹元素，通过 CSS 动画扩散后移除。`.v-btn`（实心）与 `.v-btn-text`（文字）各用一套涟漪样式。

---

## 4. 错误处理与边界情况

### 4.1 localStorage 访问限制

部分浏览器在 `file://` 协议下限制 `localStorage` 访问。通过 `safeSetItem` / `safeGetItem` 封装，捕获异常并降级处理：

```javascript
function safeSetItem(key, value) {
  try {
    localStorage.setItem(key, value);
    return true;
  } catch (e) {
    console.warn('Failed to write to localStorage:', e);
    return false;
  }
}
```

### 4.2 渲染 / 动画锁

- `isRendering`：防止快速点击导致的重复渲染冲突（新增/删除/重置/导入等）。
- `isSortAnimating`：排序 FLIP 动画期间的移动操作锁。

### 4.3 编辑状态守卫

- 名称编辑中：阻止切换模板、关闭管理/排序/导出导入页面（`isTemplateNameEditingBlocking`）。
- 内容有未保存变动：切换/离开前写入草稿并给出提示（不阻止流程）；组装始终使用已保存内容。
- 草稿仅存内存，页面刷新后丢失（属预期行为）。

### 4.4 内容校验

- 模板内容保存：必须非空且包含 `${user_input}`；空内容自动恢复基线。
- 模板名称：非空、≤30 字符（编辑器输入 `maxlength=30`；导入数据同步校验）。
- 导入数据：链式校验「非空 → 可解析 → 模板数组存在且非空 → 逐条（普通对象 + name 非空 + content 为字符串 + name ≤30字符）」，快速失败并由 Snackbar 精确报出出错条目序号（从 1 计）；`version` / `exportedAt` 等元数据字段不参与校验；导入为纯追加，不去重、不覆盖（详见 3.6.2）。

---

## 5. 开发者日志规范

所有 `console.warn` / `console.error` 使用英文，便于开发者调试：

| 场景 | 日志内容 |
|------|----------|
| localStorage 写入失败 | `Failed to write to localStorage (possibly restricted by file: protocol):` |
| localStorage 读取失败 | `Failed to read from localStorage (possibly restricted by file: protocol):` |
| 配置解析失败 | `Failed to parse TEMPLATE_CONFIGS, using default configuration` |
| 模板内容删除失败 | `Failed to remove template content from localStorage:` |
| 复制到剪贴板失败 | `Failed to copy:` |

---

## 6. 多语言发布指南

发布其他语言版本的步骤：

1. 复制 `PAT_dev.html`（或最新 Release 文件），重命名为目标语言版本（如 `PAT_v2.0_en.html`）。
2. 修改 `<html lang="zh-CN">` 为目标语言代码。
3. 修改 `TEXTS` 常量中的所有值为目标语言翻译。
4. 修改 `:root` 中的 `--font-sans-cjk` 为目标语言 CJK 字体栈（例：日语版改为 `'Hiragino Sans', 'Yu Gothic', 'Noto Sans JP', 'Meiryo'`）。
5. 修改 `DEFAULT_TEMPLATE_CONTENT_*` 常量与 `DEFAULT_TEMPLATE_CONFIGS`（如需翻译默认模板内容与默认模板名 `param_defaultTemplateNameX`）。

**注意**：
- HTML 中的原始文字保留为默认语言，无需修改。
- 带变量的消息保持 `{name}` / `{count}` / `{index}` 占位符不变，仅翻译固定文字部分。
- 模板名上限为 **30 字符**（编辑器与导入校验共用此规则，各语言提示信息中的字符数需同步）。
- `editTime` 时间戳格式（`YYYY-MM-DD HH:mm`）为展示用途，无需翻译。