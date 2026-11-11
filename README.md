# Prompt Assembly Tool (PAT)

**Prompt Assembly Tool** 是一个用于保存和复用`常用提示词`的html小工具。

通过将用户输入嵌入到预设的模板中，减少复用内容的重复输入。

适用的提示词场景例如：翻译，探讨方案，给AI的固定规则...etc.


## ✨ 功能特性

### 核心功能

- **Prompt 组装（问题主页）**
  - 输入问题内容后，点击「组装 Prompt」打开独立的模板选择对话框（Chip 单选）
  - 组装结果在底部 Drawer 中展示，一键复制到剪贴板
  - 模板选择对话框记忆上次选中的模板（按模板标识记忆，排序后不会记错）
  - 主输入框支持带确认的「清除」按钮（输入为空时点击不响应）

- **模板管理界面**（v2.0 核心变化）
  - 通过主界面左上角的「模板管理」FAB 进入全屏浮层页面：左侧模板列表 + 右侧模板编辑器
  - 列表项显示模板名、内容字数、最近保存时间（`editTime`）；存在未保存修改时显示红点标记
  - 支持新增、删除模板；模板名可在编辑器中内联重命名（最多30个字符）
  - 未保存的编辑内容以草稿形式暂存于内存中（不持久化），切换模板时给出未保存提示
  - 模板内容保存时必须包含 `${user_input}` 嵌入标记；编辑器提供在光标位置插入该标记的专用按钮
  - 「回退」可将编辑内容还原为最近一次已保存的内容（v1.2 中针对单个模板的「重置」已取消）
  - 「重置所有模板」移至管理页头部（⭯ 按钮），重置为文件内置的 `DEFAULT_TEMPLATE_CONFIGS`

- **模板排序**（v2.0 新增）
  - 置顶（第5个模板起启用）/ 上移 / 下移，每次移动即时持久化
  - 排序决定「组装 Prompt」时模板选择对话框中的模板顺序
  - 模板少于2个时不开放排序

- **模板配置导出 / 导入**（v2.0 新增）
  - 导出：将全部模板（名称 + 内容）导出为 JSON，含 `version`、`exportedAt`（本地时区时间）字段
  - 导入：逐条校验结构（`name` 非空且 ≤30 字符、`content` 为字符串）后追加导入，**不会覆盖**现有模板
  - 支持 `{"templates":[...]}` 与顶层数组两种 JSON 结构
  - 用于跨浏览器 / 跨系统同步和分享模板数据（`localStorage` 无法跨环境共享）

- **数据持久化**
  - 使用 `localStorage` 自动保存模板配置和内容
  - 自动兼容旧版本数据（缺失 `editTime` 字段时自动补充为空值）

- **多语言支持 (i18n)**
  - 界面文字集中定义在 `TEXTS` 常量中，便于多语言发布
  - 支持带变量消息的 `{name}` 等占位符动态替换
  - CJK 字体通过 CSS 变量 `--font-sans-cjk` 集中管理，切换语言版本时只需修改该变量
  - 无需构建工具，保持单文件静态 HTML 模式

### 内置示例模板
可自定义修改默认模板
1. **AI-xChk(例** — 用于检查其他 AI 回答的验证模板
2. **空模板1** — 空白模板，供用户自行填写

## 🚀 使用方法

### 直接运行
本项目为单HTML文件APP，无需安装依赖，浏览器打开放入收藏夹即可方便打开。

### 基本workflow
1. **输入问题**：在「问题」标签页输入想问 AI 的内容
2. **选择模板**：点击「组装 Prompt」，在弹出的选择对话框中选中目标模板
3. **获取结果**：在底部 Drawer 中查看组装后的完整 Prompt
4. **复制使用**：点击复制按钮，粘贴到 LLM 对话框中使用

### 模板管理
- **进入管理界面**：点击主界面左上角（卡片左边线外侧）的绿色「模板管理」FAB
- **新增模板**：管理界面左上角点击「+ 新增模板」并确认，随后在右侧编辑器中填写
- **重命名**：编辑器名称行的铅笔图标 → 修改名称（≤30字符）→ 点击保存（✓）或放弃（↩）
- **编辑内容**：在编辑器编辑区修改模板内容后点击「保存」（需包含 `${user_input}`，可用右上角按钮快速插入）
- **回退**：点击「回退」放弃未保存的修改，恢复到最近一次已保存内容
- **删除模板**：编辑器名称行的垃圾桶图标，确认后删除（不可恢复）
- **模板排序**：点击「新增模板」按钮左侧的排序按钮，通过卡片右侧按钮调整顺序，修改即时生效
- **重置所有模板**：管理页头部 ⭯ 按钮，确认后重置为文件内的默认配置（⚠ 当前自定义内容将被清除）
- **导出 / 导入**：管理页头部导出/导入按钮，在 JSON 编辑区操作后使用对应按钮（导出 / 导入 / 复制 / 清除）

## 📁 项目结构

```
PAT_html/
├── Dev/                      # 开发中的文件
│   ├── PAT_dev.html          # 开发版本（可编辑）
│   └── PAT_dev_Design.md     # 详细设计文档
├── Release/                  # 多语言发布版本
│   ├── PAT.en.html           # 英文版
│   └── PAT.ja.html           # 日文版
├── .clinerules               # Cline规则
├── PAT_html.code-workspace   # VS Code 工作区配置
└── README.md                 # 说明文件
```

## 🛠️ 技术栈

- **HTML5**
- **CSS3** - Vuetify 设计风格
- **JavaScript (ES6+)** - 交互&逻辑
- **localStorage** - 数据持久化

### 主要 UI 组件
| 组件 | 说明 |
|------|------|
| Overlay Page | 全屏浮层页面：模板管理 / 模板排序 / 导出导入（仅通过返回按钮或 ESC 关闭，点击遮罩不关闭） |
| Dialog | 确认对话框，由 `DIALOG_PRESETS` 配置表驱动 |
| Chip 组 | 模板选择对话框中的单选卡片 |
| Drawer | 底部抽屉，展示组装结果 |
| Snackbar | 顶部消息提示 |
| FAB | 浮动操作按钮（页面回到顶部 / 模板管理入口；各浮层有独立的回到顶部 FAB） |
| Tooltip | 悬浮提示信息 |

## 🌐 多语言支持 (i18n)

- 所有界面文字集中定义在 `TEXTS` 常量中。
- CJK 字体栈集中管理于 CSS 变量 `--font-sans-cjk`。
- 部分css宽度和尺寸数值可能收到不同语言UI表现得影响。
    - css:`.dialog-card-select-tpl .tpl-select-actions .v-btn-text`

### 发布其他语言版本

1. 复制 HTML 文件并将 `<html lang="zh-CN">` 修改为目标语言代码（如 `ja`）。
2. 修改 `TEXTS` 常量中的全部值为对应语言翻译。
3. 修改 `--font-sans-cjk` 变量为目标语言的 CJK 字体栈（如日语版改为 `'Hiragino Sans', 'Yu Gothic', 'Noto Sans JP', 'Meiryo'`）。
4. 如需翻译默认模板内容，修改 `DEFAULT_TEMPLATE_CONTENT_*` 常量及 `DEFAULT_TEMPLATE_CONFIGS`。

```javascript
const TEXTS = {
  pageTitle: "Prompt Assembler",
  txt_MainTab_Title: "问题",        // 改为对应语言的翻译
  btn_MainTab_Assemble: "组装 Prompt", // 改为对应语言的翻译
  // ... 其他文字
};
```

### 带变量的消息

部分消息包含动态变量（如模板名称、数量等），使用 `{name}`、`{count}` 等占位符表示：

```javascript
// TEXTS 中定义
snackbarArgMsg_TemplateCreated: "新模板 {name} 已创建",

// 代码中使用
showSnackbar(formatText(TEXTS.snackbarArgMsg_TemplateCreated, { name: newConfig.name }));
```

## ⚙️ 自定义默认模板

如需修改默认模板内容，可编辑文件中的以下部分：

```javascript
// 在 DEFAULT_TEMPLATE_CONFIGS 中设定各模板的元信息与 defaultContent
// v2.0 结构：key / tabId / name / defaultContent / editTime
const DEFAULT_TEMPLATE_CONFIGS = [
  {
    key: 'LS_TEMPLATE_CONTENT_CN',
    tabId: 'settings',
    name: TEXTS.param_defaultTemplateName0,
    defaultContent: DEFAULT_TEMPLATE_CONTENT_0,
    editTime: ''
  },
  // ...
];
```

> **注意**：
> - 默认模板名称引用 `TEXTS.param_defaultTemplateNameX`，修改语言版本时请同步更新这些值。
> - 新增模板自动命名为 `{param_newTemplateDefaultName}{编号}`（如「模板3」），编号按现有存储键自动采番。
> - 「重置所有模板」仅恢复到 `DEFAULT_TEMPLATE_CONFIGS` 定义的内容；若 `defaultContent` 为空，重置后该模板内容为空。

## 🔒 注意事项

1. **浏览器兼容性**：建议使用现代浏览器（Chrome、Edge、Firefox）
2. **localStorage 限制**：部分浏览器在 `file://` 协议下可能限制 localStorage 访问
3. **模板保存**：模板内容必须包含 `${user_input}` 标记才能保存
4. **模板名称**：不得超过30个字符（含导入数据校验）
5. **重置所有模板**：会清除全部自定义模板并恢复默认配置，请谨慎操作；建议先使用导出功能备份
6. **数据安全**：所有数据仅存储在本地浏览器中，不会上传到服务器

## 📄 版本历史

| 版本 | 说明 |
|------|------|
| v2.0 | ①新增「模板管理」全屏浮层界面（模板列表 + 编辑器、内联重命名、未保存草稿与红点标记、编辑时间戳）；②新增模板排序界面（置顶/上移/下移，即时持久化）；③新增模板导出/导入界面（JSON 追加导入，不覆盖）；④主输入框新增带确认的清除按钮；⑤模板选择对话框独立化（位置对齐输入框、记忆上次选择）；⑥原 Settings Tab 编辑方式改为模板管理界面，单个模板的「重置」改为「回退」；⑦模板名长度上限 20 → 30 字符 |
| v1.2 | 新增多语言支持 (i18n)，界面文字集中管理于 `TEXTS` 常量 |
| v1.1 | 初期版本 |

---

*Prompt Assembly Tool - 让 Prompt 复用更高效*