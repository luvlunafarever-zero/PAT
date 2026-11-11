# Prompt Assembly Tool (PAT)

**Prompt Assembly Tool** is a small HTML utility for saving and reusing frequently used prompts.

It embeds user input into preset templates, reducing repeated typing when reusing content.

Suitable prompt scenarios include translation, discussing solutions, fixed rules for AI, etc.


## ✨ Features

### Core Features

- **Prompt Assembly (Question Home Page)**
  - After entering question content, click **Assemble Prompt** to open an independent template selection dialog (single selection via chips).
  - The assembled result is displayed in the bottom Drawer and can be copied to the clipboard with one click.
  - The template selection dialog remembers the last selected template (remembered by template key, so sorting will not break the memory).
  - The main input field supports a **Clear** button with confirmation (clicking it when the input is empty does nothing).

- **Template Management Screen** (core change in v2.0)
  - Enter the full-screen overlay page through the green **Template Management** FAB at the top-left of the main screen: template list on the left + template editor on the right.
  - List items display template name, content character count, and last saved time (`editTime`); a red dot is shown when there are unsaved changes.
  - Supports adding and deleting templates; template names can be renamed inline in the editor (up to 30 characters).
  - Unsaved edited content is temporarily stored in memory as a draft (not persisted), and an unsaved-changes prompt is shown when switching templates.
  - Template content must contain the `${user_input}` embedded marker when saved; the editor provides a dedicated button to insert this marker at the cursor position.
  - **Revert** can restore the edited content to the last saved content (the per-template **Reset** from v1.2 has been removed).
  - **Reset all templates** has been moved to the header of the management page (⭯ button), resetting to the built-in `DEFAULT_TEMPLATE_CONFIGS` in the file.

- **Template Sorting** (new in v2.0)
  - Pin to top (enabled from the 5th template onward) / move up / move down; each move is persisted immediately.
  - Sorting determines the template order in the template selection dialog when using **Assemble Prompt**.
  - Sorting is disabled when there are fewer than 2 templates.

- **Template Export / Import** (new in v2.0)
  - Export: exports all templates (name + content) as JSON, including `version` and `exportedAt` (local timezone) fields.
  - Import: validates each item structure (`name` is non-empty and ≤30 characters, `content` is a string) before appending; **it does not overwrite** existing templates.
  - Supports both `{"templates":[...]}` and top-level array JSON structures.
  - Useful for syncing and sharing template data across browsers / systems (`localStorage` cannot be shared across environments).

- **Data Persistence**
  - Uses `localStorage` to automatically save template configurations and contents.
  - Automatically maintains compatibility with old-version data (if the `editTime` field is missing, it is automatically filled with an empty value).

- **Multi-language Support (i18n)**
  - UI text is centrally defined in the `TEXTS` constant, making multi-language releases easier.
  - Supports dynamic replacement of placeholders such as `{name}` in messages with variables.
  - CJK fonts are centrally managed through the CSS variable `--font-sans-cjk`; switching language versions only requires changing this variable.
  - No build tools are required; remains a single-file static HTML app.

### Built-in Sample Templates
Default templates can be customized.
1. **AI-xChk (example)** — a verification template for checking answers from other AI assistants.
2. **Empty Template 1** — a blank template for users to fill in themselves.

## 🚀 Usage

### Run Directly
This project is a single-HTML-file app. No dependencies are required. Open it in a browser and bookmark it for quick access.

### Basic Workflow
1. **Enter question**: Enter the content you want to ask AI on the **Question** tab.
2. **Select template**: Click **Assemble Prompt**, then choose a target template in the popup selection dialog.
3. **Get result**: View the fully assembled prompt in the bottom Drawer.
4. **Copy and use**: Click the copy button and paste it into an LLM conversation.

### Template Management
- **Enter management screen**: Click the green **Template Management** FAB at the top-left of the main screen (outside the left edge of the card).
- **Add template**: In the management screen, click **+ New Template** at the top-left and confirm, then fill it in with the editor on the right.
- **Rename**: Click the pencil icon in the editor name row → edit the name (≤30 characters) → click save (✓) or cancel (↩).
- **Edit content**: Edit template content in the editor, then click **Save** (`${user_input}` is required; use the top-right button to insert it quickly).
- **Revert**: Click **Revert** to discard unsaved changes and restore the last saved content.
- **Delete template**: Click the trash icon in the editor name row, then confirm deletion (irreversible).
- **Template sorting**: Click the sort button to the left of the **New Template** button, adjust order using the buttons on the right side of cards; changes take effect immediately.
- **Reset all templates**: Click the ⭯ button in the management page header and confirm to reset to the built-in default configuration (⚠ current custom content will be cleared).
- **Export / Import**: Use the export/import buttons in the management page header, operate in the JSON editing area, then use the corresponding buttons (Export / Import / Copy / Clear).

## 📁 Project Structure

```
PAT_html/
├── Dev/                      # Files under development
│   ├── PAT_dev.html          # Development version (editable)
│   └── PAT_dev_Design.md     # Detailed design document
├── Release/                  # Multi-language release versions
│   ├── PAT.en.html           # English version
│   └── PAT.ja.html           # Japanese version
├── .clinerules               # Cline rules
├── PAT_html.code-workspace   # VS Code workspace configuration
├── README.md                 # Documentation (Chinese)
└── README.en.md              # Documentation (English)
```

## 🛠️ Tech Stack

- **HTML5**
- **CSS3** - Vuetify-style design
- **JavaScript (ES6+)** - interaction and logic
- **localStorage** - data persistence

### Main UI Components
| Component | Description |
|------|------|
| Overlay Page | Full-screen overlay page: Template Management / Template Sorting / Export & Import (closed only by the back button or ESC; clicking the overlay background does not close it) |
| Dialog | Confirmation dialog, driven by the `DIALOG_PRESETS` configuration table |
| Chip Group | Single-selection cards in the template selection dialog |
| Drawer | Bottom drawer for displaying the assembled result |
| Snackbar | Top message notification |
| FAB | Floating action button (scroll page back to top / open Template Management; overlays have their own back-to-top FAB) |
| Tooltip | Hover tooltip |

## 🌐 Multi-language Support (i18n)

- All UI text is centrally defined in the `TEXTS` constant.
- The CJK font stack is centrally managed by the CSS variable `--font-sans-cjk`.
- Some CSS widths and size values may be affected by different language UIs.
    - css: `.dialog-card-select-tpl .tpl-select-actions .v-btn-text`

### Releasing Another Language Version

1. Copy the HTML file and change `<html lang="zh-CN">` to the target language code (for example, `ja`).
2. Translate all values in the `TEXTS` constant into the target language.
3. Change the `--font-sans-cjk` variable to the target language's CJK font stack (for example, for Japanese: `'Hiragino Sans', 'Yu Gothic', 'Noto Sans JP', 'Meiryo'`).
4. If default template contents need translation, modify the `DEFAULT_TEMPLATE_CONTENT_*` constants and `DEFAULT_TEMPLATE_CONFIGS`.

```javascript
const TEXTS = {
  pageTitle: "Prompt Assembler",
  txt_MainTab_Title: "Question",        // Change to the translation for the target language
  btn_MainTab_Assemble: "Assemble Prompt", // Change to the translation for the target language
  // ... other texts
};
```

### Messages with Variables

Some messages contain dynamic variables (such as template name, count, etc.), represented by placeholders like `{name}` and `{count}`:

```javascript
// Defined in TEXTS
snackbarArgMsg_TemplateCreated: "New template {name} created",

// Used in code
showSnackbar(formatText(TEXTS.snackbarArgMsg_TemplateCreated, { name: newConfig.name }));
```

## ⚙️ Customizing Default Templates

To modify default template content, edit the following parts in the file:

```javascript
// Set each template's meta information and defaultContent in DEFAULT_TEMPLATE_CONFIGS
// v2.0 structure: key / tabId / name / defaultContent / editTime
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

> **Notes**:
> - Default template names reference `TEXTS.param_defaultTemplateNameX`; update these values together when changing language versions.
> - New templates are automatically named `{param_newTemplateDefaultName}{number}` (for example, "Template 3"); the number is automatically assigned based on existing storage keys.
> - **Reset all templates** only restores the content defined in `DEFAULT_TEMPLATE_CONFIGS`; if `defaultContent` is empty, the template content will be empty after reset.

## 🔒 Notes

1. **Browser compatibility**: Modern browsers are recommended (Chrome, Edge, Firefox).
2. **localStorage restrictions**: Some browsers may restrict localStorage access under the `file://` protocol.
3. **Template saving**: Template content must contain the `${user_input}` marker before it can be saved.
4. **Template name**: Must not exceed 30 characters (including validation of imported data).
5. **Reset all templates**: Clears all custom templates and restores the default configuration. Use with caution; exporting a backup first is recommended.
6. **Data security**: All data is stored only in the local browser and is not uploaded to any server.

## 📄 Version History

| Version | Description |
|------|------|
| v2.0 | ① Added full-screen overlay **Template Management** screen (template list + editor, inline renaming, unsaved drafts with red dot marker, edit timestamps); ② Added template sorting screen (pin to top / move up / move down, immediate persistence); ③ Added template export/import screen (JSON append import without overwrite); ④ Added clear button with confirmation to the main input field; ⑤ Made the template selection dialog independent (aligned to input field position, remembers last selection); ⑥ Changed the former Settings Tab editing approach to the Template Management screen, and changed per-template **Reset** to **Revert**; ⑦ Increased template name length limit from 20 to 30 characters |
| v1.2 | Added multi-language support (i18n); UI text is centrally managed in the `TEXTS` constant |
| v1.1 | Initial version |

---

*Prompt Assembly Tool - Make prompt reuse more efficient*