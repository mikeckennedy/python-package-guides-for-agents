# EasyMDE — Comprehensive Reference (v2.21.0)

> A drop-in JavaScript replacement for `<textarea>` that turns it into a beautiful, embeddable Markdown editor with toolbar, side-by-side preview, autosaving, image upload, spell checking, and keyboard shortcuts.
> EasyMDE is a maintained fork of SimpleMDE. It bundles CodeMirror 5 (the editor engine), the `marked` Markdown parser, and Font Awesome icons.

This reference is generated from the **EasyMDE 2.21.0 source** (`src/js/easymde.js`, `types/easymde.d.ts`, and the official `README.md`). Although this repository is mainly for Python packages, EasyMDE is a common front-end choice for Python web apps (Flask, Django, FastAPI, Quart) that need rich Markdown editing — drop it next to your `flask_reference.md` so the agent has correct option names, defaults, and toolbar/keyboard semantics on the JS side.

> Version: 2.21.0 | License: MIT | Runtime: Browser (ES5-friendly), bundled `marked@^4`, `codemirror@^5.65`, `codemirror-spell-checker@1.1.2`

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Constructor & Options](#constructor--options)
  - [Top-level Options](#top-level-options)
  - [autosave](#autosave)
  - [blockStyles](#blockstyles)
  - [insertTexts](#inserttexts)
  - [parsingConfig](#parsingconfig)
  - [renderingConfig](#renderingconfig)
  - [promptTexts](#prompttexts)
  - [overlayMode](#overlaymode)
  - [Image Upload Options](#image-upload-options)
  - [imageTexts](#imagetexts)
  - [errorMessages](#errormessages)
- [Instance Methods](#instance-methods)
- [Static Methods (Toolbar Actions)](#static-methods-toolbar-actions)
- [Toolbar Customization](#toolbar-customization)
  - [Built-in Toolbar Buttons](#built-in-toolbar-buttons)
  - [Custom Buttons & Dropdowns](#custom-buttons--dropdowns)
- [Keyboard Shortcuts](#keyboard-shortcuts)
- [Status Bar](#status-bar)
- [Event Handling (CodeMirror)](#event-handling-codemirror)
- [Image Upload](#image-upload)
  - [Server Endpoint Contract](#server-endpoint-contract)
  - [Custom Upload Function](#custom-upload-function)
- [Autosave](#autosave-1)
- [Markdown Rendering Pipeline](#markdown-rendering-pipeline)
- [Common Patterns](#common-patterns)
  - [Flask / Django / FastAPI form integration](#flask--django--fastapi-form-integration)
  - [Sanitizing preview output](#sanitizing-preview-output)
  - [Custom previewRender (e.g., server-side render)](#custom-previewrender-eg-server-side-render)
  - [Init in a hidden tab / modal](#init-in-a-hidden-tab--modal)
  - [Cleanly destroying an instance](#cleanly-destroying-an-instance)

---

## Installation

### Via npm

```bash
npm install easymde
```

```js
import EasyMDE from 'easymde';
import 'easymde/dist/easymde.min.css';
```

### Via CDN

```html
<link rel="stylesheet" href="https://unpkg.com/easymde/dist/easymde.min.css">
<script src="https://unpkg.com/easymde/dist/easymde.min.js"></script>
```

Or jsDelivr:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/easymde/dist/easymde.min.css">
<script src="https://cdn.jsdelivr.net/npm/easymde/dist/easymde.min.js"></script>
```

The script exposes a global `EasyMDE` constructor when loaded via `<script>`.

### Font Awesome (icons)

EasyMDE auto-injects Font Awesome 4 from `maxcdn.bootstrapcdn.com/font-awesome/latest/...` if it doesn't detect that stylesheet on the page. Set `autoDownloadFontAwesome: false` to opt out (e.g., when you ship FA yourself or use a CSP that blocks the CDN).

---

## Quick Start

```html
<textarea id="editor"></textarea>

<script>
  const easyMDE = new EasyMDE({
    element: document.getElementById('editor'),
    autofocus: true,
    placeholder: 'Type your markdown here...',
    spellChecker: false,
  });

  // Submit the form? The textarea is auto-synced.
  // Manual access:
  document.getElementById('save').addEventListener('click', () => {
    console.log(easyMDE.value());
  });
</script>
```

If you don't pass `element`, EasyMDE attaches to the **first `<textarea>` on the page**.

---

## Constructor & Options

```ts
new EasyMDE(options?: EasyMDE.Options)
```

All options are optional. EasyMDE merges nested option objects (`autosave`, `blockStyles`, `insertTexts`, `promptTexts`, `imageTexts`, `errorMessages`, `shortcuts`, `iconClassMap`) with built-in defaults, so you can override only the keys you care about.

### Top-level Options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `element` | `HTMLElement` | first `<textarea>` on page | Source textarea to attach to. |
| `autoDownloadFontAwesome` | `boolean` | `undefined` (auto-detect) | Force-load (`true`) or block (`false`) the Font Awesome CDN stylesheet. When `undefined`, EasyMDE injects it only if no FA stylesheet is detected. |
| `autofocus` | `boolean` | `false` | Focus the editor on init. |
| `autoRefresh` | `boolean \| { delay: number }` | – | Useful when initializing inside a hidden DOM node. With `{ delay: 300 }`, polls every 300 ms and calls CodeMirror's `refresh()` once visible. |
| `autosave` | `AutoSaveOptions` | – | See [autosave](#autosave). |
| `blockStyles` | `BlockStyleOptions` | see below | Marker characters used by toggleBold/toggleItalic/toggleCodeBlock. |
| `direction` | `'ltr' \| 'rtl'` | `'ltr'` | Text direction. |
| `errorCallback` | `(message: string) => void` | `(m) => alert(m)` | Called with image-upload error messages. |
| `errorMessages` | `ImageErrorTextsOptions` | see below | Error message templates for image upload. |
| `forceSync` | `boolean` | `false` | If `true`, syncs editor changes to the source textarea on every keystroke (instead of on form submit only). |
| `hideIcons` | `ToolbarButton[]` | – | Icons to remove from the default toolbar. Cannot be combined meaningfully with a custom `toolbar` array. |
| `iconClassMap` | `Record<string, string>` | see source | Map of toolbar button name → CSS classes for its `<i>` icon. |
| `imagesPreviewHandler` | `(src: string) => string` | – | Transform `![](src)` → final `<img src="...">` URL for in-editor image previews. |
| `indentWithTabs` | `boolean` | `true` | If `false`, indent with spaces. |
| `initialValue` | `string` | – | Sets the editor's initial content (overridden by an autosaved value if one is found). |
| `inputStyle` | `'textarea' \| 'contenteditable'` | `textarea` desktop / `contenteditable` mobile | CodeMirror input mode. `contenteditable` is required for `nativeSpellcheck`. |
| `insertTexts` | `InsertTextOptions` | see below | Wrapping text used by drawLink/drawImage/drawTable/drawHorizontalRule. |
| `lineNumbers` | `boolean` | `false` | Show CodeMirror line numbers. |
| `lineWrapping` | `boolean` | `true` | Soft-wrap long lines. |
| `maxHeight` | `string` | `undefined` | Fixed height (CSS value like `'500px'`). When set, `minHeight` is ignored and forced to equal `maxHeight`. |
| `minHeight` | `string` | `'300px'` | Minimum height of the composition area before auto-grow. |
| `nativeSpellcheck` | `boolean` | `true` | Enable browser-native spellcheck (requires `inputStyle: 'contenteditable'`). |
| `onToggleFullScreen` | `(entering: boolean) => void` | – | Fires when fullscreen mode toggles. |
| `overlayMode` | `OverlayModeOptions` | – | Layer a CodeMirror overlay mode on top of the Markdown mode. See [overlayMode](#overlaymode). |
| `parsingConfig` | `ParsingOptions` | `{ highlightFormatting: true }` | Markdown parsing settings used **while editing**. See [parsingConfig](#parsingconfig). |
| `placeholder` | `string` | – | Placeholder text shown when empty. |
| `previewClass` | `string \| string[]` | `'editor-preview'` | Class(es) applied to the preview pane. |
| `previewImagesInEditor` | `boolean` | `false` | Render in-editor previews for images on their own line. |
| `previewRender` | `(plainText, previewEl?) => string \| null` | calls `easyMDE.markdown(text)` | Custom Markdown→HTML renderer for the preview. Return `null` to leave `previewEl.innerHTML` untouched (e.g., when you control it via vdom). |
| `promptTexts` | `PromptTexts` | see below | Prompt strings shown when `promptURLs` is `true`. |
| `promptURLs` | `boolean` | `false` | Pop a `prompt()` for the URL when inserting links/images instead of inserting placeholder syntax. |
| `renderingConfig` | `RenderingOptions` | – | Markdown rendering settings used **for the preview**. See [renderingConfig](#renderingconfig). |
| `scrollbarStyle` | `string` | `'native'` | CodeMirror scrollbar style; `'null'` hides them. |
| `shortcuts` | `Shortcuts` | see [Keyboard Shortcuts](#keyboard-shortcuts) | Override or unbind keyboard shortcuts. |
| `showIcons` | `ToolbarButton[]` | – | Names of non-default icons to add to the default toolbar. |
| `sideBySideFullscreen` | `boolean` | `true` | If `false`, side-by-side mode does **not** force fullscreen. |
| `spellChecker` | `boolean \| (opts) => void` | `true` | Built-in spell checker (uses `codemirror-spell-checker`). Pass a function to install a CodeMirrorSpellChecker-compatible alternative. |
| `status` | `boolean \| Array<string \| StatusBarItem>` | `['autosave','lines','words','cursor']` (plus `'upload-image'` prepended when `uploadImage: true`) | Status bar config. `false` hides it. See [Status Bar](#status-bar). |
| `styleSelectedText` | `boolean` | `true` | Adds the `CodeMirror-selectedtext` class to selected lines. |
| `syncSideBySidePreviewScroll` | `boolean` | `true` | Sync scroll between editor and side-by-side preview. |
| `tabSize` | `number` | `2` | Tab size in spaces. |
| `theme` | `string` | `'easymde'` | CodeMirror theme name. |
| `toolbar` | `false \| Array<'\|' \| ToolbarButton \| ToolbarIcon \| ToolbarDropdownIcon>` | default toolbar | `false` hides toolbar. See [Toolbar Customization](#toolbar-customization). |
| `toolbarButtonClassPrefix` | `string` | – | Prefix added to each toolbar button's class (e.g., `'mde'` → `'mde-bold'`). |
| `toolbarTips` | `boolean` | `true` | Show `title=` tooltips on toolbar buttons. |
| `unorderedListStyle` | `'*' \| '-' \| '+'` | `'*'` | Marker used by toggleUnorderedList. |
| `uploadImage` | `boolean` | `false` | Enable drag/drop, paste, and upload-image button. See [Image Upload](#image-upload). |

### `autosave`

```ts
interface AutoSaveOptions {
  enabled?: boolean;
  delay?: number;          // ms between saves; default 10000
  submit_delay?: number;   // ms before resaving on assumed-failed submit; defaults to `delay` or 10000
  uniqueId: string;        // REQUIRED — used as the localStorage key suffix ("smde_" + uniqueId)
  timeFormat?: { locale?: string | string[]; format?: Intl.DateTimeFormatOptions };
  text?: string;           // prefix for the auto-saved status text (default "Autosaved: ")
}
```

`uniqueId` is mandatory; without it, autosave silently no-ops (logs to console). For backward compatibility, `unique_id` is mapped to `uniqueId` if present.

Default `timeFormat`:

```js
{ locale: 'en-US', format: { hour: '2-digit', minute: '2-digit' } }
```

### `blockStyles`

```ts
interface BlockStyleOptions {
  bold?: '**' | '__';   // default '**'
  italic?: '*' | '_';   // default '*'
  code?: '```' | '~~~'; // default '```'
}
```

### `insertTexts`

Each value is a `[before, after]` pair wrapping the cursor/selection.

```js
// Defaults (from src/js/easymde.js)
{
  link:           ['[', '](#url#)'],
  image:          ['![', '](#url#)'],
  uploadedImage:  ['![](#url#)', ''],
  table:          ['', '\n\n| Column 1 | Column 2 | Column 3 |\n| -------- | -------- | -------- |\n| Text     | Text     | Text     |\n\n'],
  horizontalRule: ['', '\n\n-----\n\n'],
}
```

The README documents `link`, `image`, `table`, and `horizontalRule` as user-overrideable. The `#url#` placeholder is replaced by the URL prompt result (or left in place when `promptURLs: false`).

### `parsingConfig`

Used by the **CodeMirror Markdown mode while editing** (not the preview).

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `allowAtxHeaderWithoutSpace` | `boolean` | `false` | Recognize `#Heading` without a space. |
| `strikethrough` | `boolean` | `true` | Process GFM `~~strikethrough~~`. |
| `underscoresBreakWords` | `boolean` | `false` | Treat `_` as a word delimiter. |
| `highlightFormatting` | `boolean` | `true` | Internally forced `true` so toggleCodeBlock can detect code-block types. |

### `renderingConfig`

Used by the **preview** Markdown renderer (default: `marked`).

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `singleLineBreaks` | `boolean` | `true` | When `true`, `marked.breaks = true` (GFM single line breaks). |
| `codeSyntaxHighlighting` | `boolean` | `false` | Plug `highlight.js` into `marked` for fenced code blocks. |
| `hljs` | `any` | `window.hljs` | Inject a specific `highlight.js` instance. |
| `markedOptions` | `marked.MarkedExtension` | `{}` | Passed to `marked.use()`. Other `renderingConfig` keys take precedence (e.g., `breaks`). |
| `sanitizerFunction` | `(html: string) => string` | – | Post-process the rendered HTML (e.g., DOMPurify). |

> The preview pipeline always (1) calls `marked.parse()`, (2) runs your `sanitizerFunction` if any, (3) adds `target="_blank"` to anchors, (4) strips list-style on checkbox `<li>`s.

### `promptTexts`

```js
{ image: 'URL of the image:', link: 'URL for the link:' }
```

Only used when `promptURLs: true`.

### `overlayMode`

```ts
interface OverlayModeOptions {
  mode: CodeMirror.Mode<any>;
  combine?: boolean; // default true
}
```

When `combine` is `true` (default), classes from the overlay are **added to** the Markdown-mode classes. Set `false` to **replace** them. EasyMDE auto-defaults `combine` to `true` if you set `overlayMode` without specifying it.

### Image Upload Options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `uploadImage` | `boolean` | `false` | Enable drag/drop, paste, and upload-image button. |
| `imageMaxSize` | `number` | `2097152` (2 MB) | Max bytes; checked client-side before upload. **Always re-validate on the server.** |
| `imageAccept` | `string` | `'image/png, image/jpeg, image/gif, image/avif'` | Comma-separated MIME allowlist. (Note: README mentions only PNG/JPEG, but the source default also includes `image/gif` and `image/avif`.) |
| `imageUploadEndpoint` | `string` | – | URL to POST images to. See [server contract](#server-endpoint-contract). |
| `imageUploadFunction` | `(file, onSuccess, onError) => void` | – | Custom uploader. When set, `imageMaxSize`, `imageAccept`, `imageUploadEndpoint`, and `imageCSRFToken` become inert. |
| `imagePathAbsolute` | `boolean` | `false` | If `true`, treat the returned `filePath` as absolute (no `window.location.origin` prefix). |
| `imageCSRFToken` | `string` | – | CSRF token to send with the upload. |
| `imageCSRFName` | `string` | `'csrfmiddlewaretoken'` | Field/header name for the token. |
| `imageCSRFHeader` | `boolean` | `false` | If `true`, send the CSRF token as an HTTP header; otherwise as a form field. |
| `imageInputName` | `string` | `'image'` | Name of the multipart field used by the built-in uploader (note: the form field key the server sees is hard-coded to `image` by `formData.append('image', file)` in the source — `imageInputName` is configured but only relevant for the underlying file `<input>`). |
| `imageTexts` | `ImageTextsOptions` | see below | Status bar messages during upload. |
| `errorMessages` | `ImageErrorTextsOptions` | see below | Error message templates. |
| `errorCallback` | `(msg: string) => void` | `alert` | Where to deliver formatted error messages. |

### `imageTexts`

```js
{
  sbInit:        'Attach files by drag and dropping or pasting from clipboard.',
  sbOnDragEnter: 'Drop image to upload it.',
  sbOnDrop:      'Uploading image #images_names#...',
  sbProgress:    'Uploading #file_name#: #progress#%',
  sbOnUploaded:  'Uploaded #image_name#',
  sizeUnits:     ' B, KB, MB',
}
```

Placeholders `#image_name#`, `#image_size#`, `#image_max_size#`, `#file_name#`, `#progress#`, `#images_names#` are interpolated at runtime.

### `errorMessages`

```js
{
  noFileGiven:   'You must select a file.',
  typeNotAllowed:'This image type is not allowed.',
  fileTooLarge:  'Image #image_name# is too big (#image_size#).\nMaximum file size is #image_max_size#.',
  importError:   'Something went wrong when uploading the image #image_name#.',
}
```

---

## Instance Methods

Constructed from the source in `src/js/easymde.js`.

| Method | Signature | Description |
| --- | --- | --- |
| `value()` | `(): string` | Return current Markdown text. |
| `value(val)` | `(val: string): EasyMDE` | Set Markdown text. Re-renders the preview if currently active. Returns `this`. |
| `codemirror` | `CodeMirror.Editor` (property) | The underlying CodeMirror 5 instance. Use it for `on('change', ...)` etc. |
| `markdown(text)` | `(text: string): string` | Render Markdown → HTML using the configured `marked`/`highlight.js`/sanitizer pipeline. |
| `toTextArea()` | `(): void` | Tear down editor UI and restore the original `<textarea>`. Clears autosaved value if autosave is configured. |
| `cleanup()` | `(): void` | Remove document-level keydown listener registered by EasyMDE. Use when the editor is being destroyed but the textarea is reused. |
| `isPreviewActive()` | `(): boolean` | True when the preview-only pane is showing. |
| `isSideBySideActive()` | `(): boolean` | True when side-by-side mode is on. |
| `isFullscreenActive()` | `(): boolean` | Reads CodeMirror's `fullScreen` option. |
| `clearAutosavedValue()` | `(): void` | Removes `localStorage['smde_<uniqueId>']`. No-op (with console log) if `uniqueId` not configured. |
| `autosave()` | `(): void` | Manually trigger an autosave write (also called automatically on a timer when `autosave.enabled`). |
| `updateStatusBar(itemName, content)` | `(string, string): void` | Update the text content of a status bar item by class name. |
| `openBrowseFileWindow(onSuccess?, onError?)` | `(fn?, fn?): void` | Programmatically open the file picker and start an upload. |
| `uploadImage(file, onSuccess?, onError?)` | `(File, fn?, fn?): void` | Upload one image via `imageUploadEndpoint`. Defaults `onSuccess` to insert the resulting Markdown image at the cursor. |
| `uploadImages(files, onSuccess?, onError?)` | `(FileList, fn?, fn?): void` | Upload many. Calls `uploadImage` per file. |
| `uploadImageUsingCustomFunction(fn, file)` | `(fn, File): void` | Run the user-provided `imageUploadFunction` and wire status bar / error handling around it. |
| `uploadImagesUsingCustomFunction(fn, files)` | `(fn, FileList): void` | Same, batched. |
| `setPreviewMaxHeight()` | `(): void` | Recompute preview pane height (called internally when `maxHeight` is set). |
| `createSideBySide()` | `(): HTMLElement` | Build the side-by-side preview pane. (Internal — exposed but rarely useful.) |
| `createToolbar(items)` | `(items): HTMLElement` | Build the toolbar from an items array. (Internal.) |
| `createStatusbar(status)` | `(status): HTMLElement` | Build the status bar. (Internal.) |
| `render(el?)` | `(HTMLElement?): void` | Render the editor into `el` (or the configured / first `<textarea>`). Called automatically by the constructor. |

The **action methods** below are exposed both as instance methods (no-args) and statics (`(editor) => void`):

```js
easyMDE.toggleBold();
easyMDE.toggleItalic();
easyMDE.toggleStrikethrough();
easyMDE.toggleBlockquote();
easyMDE.toggleHeadingSmaller();
easyMDE.toggleHeadingBigger();
easyMDE.toggleHeading1();
easyMDE.toggleHeading2();
easyMDE.toggleHeading3();
easyMDE.toggleHeading4();
easyMDE.toggleHeading5();
easyMDE.toggleHeading6();
easyMDE.toggleCodeBlock();
easyMDE.toggleUnorderedList();
easyMDE.toggleOrderedList();
easyMDE.toggleCheckList();
easyMDE.cleanBlock();
easyMDE.drawLink();
easyMDE.drawImage();
easyMDE.drawUploadedImage();
easyMDE.drawTable();
easyMDE.drawHorizontalRule();
easyMDE.undo();
easyMDE.redo();
easyMDE.togglePreview();
easyMDE.toggleSideBySide();
easyMDE.toggleFullScreen();
easyMDE.getState();   // returns CodeMirror state-flags object (bold, italic, code, list, ...)
```

---

## Static Methods (Toolbar Actions)

Each accepts the editor instance and is suitable for use as a custom toolbar item's `action`:

```js
EasyMDE.toggleBold(editor);
EasyMDE.toggleItalic(editor);
EasyMDE.toggleStrikethrough(editor);
EasyMDE.toggleBlockquote(editor);
EasyMDE.toggleHeadingSmaller(editor);
EasyMDE.toggleHeadingBigger(editor);
EasyMDE.toggleHeading1(editor);
EasyMDE.toggleHeading2(editor);
EasyMDE.toggleHeading3(editor);
EasyMDE.toggleHeading4(editor);
EasyMDE.toggleHeading5(editor);
EasyMDE.toggleHeading6(editor);
EasyMDE.toggleCodeBlock(editor);
EasyMDE.toggleUnorderedList(editor);
EasyMDE.toggleOrderedList(editor);
EasyMDE.toggleCheckList(editor);
EasyMDE.cleanBlock(editor);
EasyMDE.drawLink(editor);
EasyMDE.drawImage(editor);
EasyMDE.drawUploadedImage(editor);
EasyMDE.drawTable(editor);
EasyMDE.drawHorizontalRule(editor);
EasyMDE.undo(editor);
EasyMDE.redo(editor);
EasyMDE.togglePreview(editor);
EasyMDE.toggleSideBySide(editor);
EasyMDE.toggleFullScreen(editor);
```

> **Note** the type definitions advertise `toggleHeading4/5/6` on the static surface. They exist as instance methods on the prototype and are usable through CodeMirror keymaps / `bindings`, but **only `toggleHeading1/2/3` are bound as static `EasyMDE.toggleHeadingN`** in the source (`EasyMDE.toggleHeading1 = toggleHeading1` etc.). To use 4–6, call `editor.toggleHeading4()` / wire a custom toolbar `action: (e) => e.toggleHeading4()`.

---

## Toolbar Customization

`toolbar` accepts:

- `false` to hide the toolbar entirely
- `undefined` to use the default set (see below)
- An array of strings (built-in names), `'|'` separators, or `ToolbarIcon` / `ToolbarDropdownIcon` objects

```ts
interface ToolbarIcon {
  name: string;
  action: string | ((editor: EasyMDE) => void);  // a URL opens in a new tab; a function runs
  className: string;
  title: string;
  noDisable?: boolean;   // stays clickable in preview/sidebyside
  noMobile?: boolean;    // hidden on mobile
  icon?: string;         // optional inline SVG/HTML overriding the <i class="className">
  attributes?: { [k: string]: string };  // extra HTML attrs (id, data-*, etc.)
}

interface ToolbarDropdownIcon {
  name: string;
  children: Array<ToolbarIcon | ToolbarButton>;  // 1+
  className: string;
  title: string;
  noDisable?: boolean;
  noMobile?: boolean;
}
```

### Built-in Toolbar Buttons

`★` = enabled in the **default** toolbar.

| Name | Action | Default Class | Default Title | Default? |
| --- | --- | --- | --- | --- |
| `bold` | toggleBold | `fa fa-bold` | Bold | ★ |
| `italic` | toggleItalic | `fa fa-italic` | Italic | ★ |
| `strikethrough` | toggleStrikethrough | `fa fa-strikethrough` | Strikethrough | |
| `heading` | toggleHeadingSmaller | `fa fa-header fa-heading` | Heading | ★ |
| `heading-smaller` | toggleHeadingSmaller | `fa fa-header fa-heading header-smaller` | Smaller Heading | |
| `heading-bigger` | toggleHeadingBigger | `fa fa-header fa-heading header-bigger` | Bigger Heading | |
| `heading-1` | toggleHeading1 | `fa fa-header fa-heading header-1` | Big Heading | |
| `heading-2` | toggleHeading2 | `fa fa-header fa-heading header-2` | Medium Heading | |
| `heading-3` | toggleHeading3 | `fa fa-header fa-heading header-3` | Small Heading | |
| `code` | toggleCodeBlock | `fa fa-code` | Code | |
| `quote` | toggleBlockquote | `fa fa-quote-left` | Quote | ★ |
| `unordered-list` | toggleUnorderedList | `fa fa-list-ul` | Generic List | ★ |
| `ordered-list` | toggleOrderedList | `fa fa-list-ol` | Numbered List | ★ |
| `check-list` | toggleCheckList | `fa fa-check-square-o` | Check List | ★ |
| `clean-block` | cleanBlock | `fa fa-eraser` | Clean block | |
| `link` | drawLink | `fa fa-link` | Create Link | ★ |
| `image` | drawImage | `fa fa-image` | Insert Image | ★ |
| `upload-image` | drawUploadedImage | `fa fa-image` | Import an image | |
| `table` | drawTable | `fa fa-table` | Insert Table | |
| `horizontal-rule` | drawHorizontalRule | `fa fa-minus` | Insert Horizontal Line | |
| `preview` | togglePreview | `fa fa-eye` | Toggle Preview | ★ (noDisable) |
| `side-by-side` | toggleSideBySide | `fa fa-columns` | Toggle Side by Side | ★ (noDisable, noMobile) |
| `fullscreen` | toggleFullScreen | `fa fa-arrows-alt` | Toggle Fullscreen | ★ (noDisable, noMobile) |
| `guide` | URL → markdownguide.org | `fa fa-question-circle` | Markdown Guide | ★ (noDisable) |
| `undo` | undo | `fa fa-undo` | Undo | (noDisable) |
| `redo` | redo | `fa fa-repeat fa-redo` | Redo | (noDisable) |

> Source-of-truth note: the README lists the heading icon class as `fa fa-header`, but the source's `iconClassMap` actually sets `'fa fa-header fa-heading'` (FA4 + FA5 fallback) — same idea, more compatible.

> When `toolbar` is `undefined`, EasyMDE iterates `toolbarBuiltInButtons` in source order, inserting `'|'` for each `separator-N` key. Add non-default buttons via `showIcons: ['code', 'table']` or omit defaults via `hideIcons: ['guide', 'heading']`.

### Custom Buttons & Dropdowns

```js
const easyMDE = new EasyMDE({
  toolbar: [
    'bold',
    'italic',
    '|',
    {
      name: 'star',
      action: (editor) => {
        const cm = editor.codemirror;
        cm.replaceSelection('⭐ ' + cm.getSelection());
      },
      className: 'fa fa-star',
      title: 'Insert star',
      attributes: { 'data-test': 'star-btn' },
    },
    {
      name: 'headings',
      className: 'fa fa-header',
      title: 'Headings',
      children: [
        { name: 'h1', action: EasyMDE.toggleHeading1, className: 'fa fa-header header-1', title: 'H1' },
        { name: 'h2', action: EasyMDE.toggleHeading2, className: 'fa fa-header header-2', title: 'H2' },
        { name: 'h3', action: EasyMDE.toggleHeading3, className: 'fa fa-header header-3', title: 'H3' },
      ],
    },
    'preview',
  ],
});
```

A toolbar item with a string `action` opens that URL in a new tab (this is how the `guide` button works).

---

## Keyboard Shortcuts

Defaults (defined in `src/js/easymde.js`):

| Action key | Default shortcut |
| --- | --- |
| `toggleBold` | `Cmd-B` |
| `toggleItalic` | `Cmd-I` |
| `drawLink` | `Cmd-K` |
| `toggleHeadingSmaller` | `Cmd-H` |
| `toggleHeadingBigger` | `Shift-Cmd-H` |
| `toggleHeading1` | `Ctrl+Alt+1` |
| `toggleHeading2` | `Ctrl+Alt+2` |
| `toggleHeading3` | `Ctrl+Alt+3` |
| `toggleHeading4` | `Ctrl+Alt+4` |
| `toggleHeading5` | `Ctrl+Alt+5` |
| `toggleHeading6` | `Ctrl+Alt+6` |
| `cleanBlock` | `Cmd-E` |
| `drawImage` | `Cmd-Alt-I` |
| `toggleBlockquote` | `Cmd-'` |
| `toggleOrderedList` | `Cmd-Alt-L` |
| `toggleUnorderedList` | `Cmd-L` |
| `toggleCheckList` | `Shift-Cmd-L` |
| `toggleCodeBlock` | `Cmd-Alt-C` |
| `togglePreview` | `Cmd-P` |
| `toggleSideBySide` | `F9` |
| `toggleFullScreen` | `F11` |

EasyMDE auto-translates `Cmd` ↔ `Ctrl` for the user's platform via `fixShortcut`.

Override or unbind:

```js
new EasyMDE({
  shortcuts: {
    toggleOrderedList: 'Ctrl-Alt-K', // change
    toggleCodeBlock:   null,         // unbind
    drawTable:         'Cmd-Alt-T',  // bind an action that has no default
  },
});
```

The full list of bindable action keys: `toggleBold`, `toggleItalic`, `toggleStrikethrough`, `drawLink`, `drawImage`, `toggleHeadingSmaller`, `toggleHeadingBigger`, `toggleHeading1`–`toggleHeading6`, `toggleBlockquote`, `toggleOrderedList`, `toggleUnorderedList`, `toggleCheckList`, `toggleCodeBlock`, `togglePreview`, `cleanBlock`, `drawTable`, `drawHorizontalRule`, `undo`, `redo`, `toggleSideBySide`, `toggleFullScreen`.

---

## Status Bar

`status` controls the bottom bar. Default when omitted: `['autosave', 'lines', 'words', 'cursor']` (with `'upload-image'` prepended automatically when `uploadImage: true`).

Pass `false` to hide it, or an array mixing built-in names and custom items:

```ts
interface StatusBarItem {
  className: string;                                 // unique CSS class for lookup
  defaultValue: (element: HTMLElement) => void;     // runs once on init
  onUpdate:     (element: HTMLElement) => void;     // runs on CodeMirror update
}
```

Built-in items: `autosave`, `lines`, `words`, `cursor`, `upload-image`.

```js
new EasyMDE({
  status: [
    'lines',
    'words',
    'cursor',
    {
      className: 'keystrokes',
      defaultValue: (el) => { el.dataset.k = '0'; el.textContent = '0 keystrokes'; },
      onUpdate:     (el) => {
        const n = Number(el.dataset.k) + 1;
        el.dataset.k = n;
        el.textContent = `${n} keystrokes`;
      },
    },
  ],
});
```

Update a custom item programmatically:

```js
easyMDE.updateStatusBar('keystrokes', '999 keystrokes');
```

---

## Event Handling (CodeMirror)

EasyMDE doesn't expose its own event bus — use the embedded CodeMirror instance:

```js
const easyMDE = new EasyMDE({ element: textarea });

easyMDE.codemirror.on('change', () => {
  console.log('content:', easyMDE.value());
});

easyMDE.codemirror.on('blur', () => { /* ... */ });
easyMDE.codemirror.on('cursorActivity', () => { /* ... */ });
```

See [CodeMirror 5 events](https://codemirror.net/5/doc/manual.html#events) for the full list (`change`, `changes`, `beforeChange`, `cursorActivity`, `keyHandled`, `inputRead`, `electricInput`, `beforeSelectionChange`, `viewportChange`, `swapDoc`, `gutterClick`, `gutterContextMenu`, `focus`, `blur`, `scroll`, `refresh`, `optionChange`, `scrollCursorIntoView`, `update`, `renderLine`, `mousedown`, `dblclick`, `touchstart`, `contextmenu`, `keydown`, `keypress`, `keyup`, `cut`, `copy`, `paste`, `dragstart`, `dragenter`, `dragover`, `dragleave`, `drop`).

---

## Image Upload

Two modes: built-in HTTP POST (default) or your own function.

```js
const easyMDE = new EasyMDE({
  uploadImage: true,
  imageUploadEndpoint: '/api/upload',
  imageMaxSize: 5 * 1024 * 1024,
  imageAccept: 'image/png, image/jpeg, image/webp',
  imageCSRFToken: document.querySelector('meta[name=csrf-token]').content,
  imageCSRFHeader: true,
  imageCSRFName: 'X-CSRF-Token',
  errorMessages: {
    fileTooLarge: '#image_name# is #image_size# — max is #image_max_size#.',
  },
});
```

Triggers when this is enabled:

- Drag and drop a file onto the editor
- Paste image data from the clipboard
- Click the `upload-image` toolbar button

### Server Endpoint Contract

POST is `multipart/form-data` with the field `image` (and the CSRF field if `imageCSRFToken && !imageCSRFHeader`).

**Success (HTTP 200):**

```json
{ "data": { "filePath": "/uploads/abc.png" } }
```

The editor inserts `![](<origin>/uploads/abc.png)` (or the bare `filePath` if `imagePathAbsolute: true`).

**Error:**

```json
{ "error": "fileTooLarge" }
```

Recognized error keys map to `errorMessages` entries: `noFileGiven` (HTTP 400), `typeNotAllowed` (HTTP 415), `fileTooLarge` (HTTP 413), `importError`. Any other string in `error` is alerted verbatim — useful for server-generated messages.

### Custom Upload Function

```js
new EasyMDE({
  uploadImage: true,
  imageUploadFunction: async (file, onSuccess, onError) => {
    try {
      const fd = new FormData();
      fd.append('file', file);
      const res = await fetch('/s3-presign-and-upload', { method: 'POST', body: fd });
      const { url } = await res.json();
      onSuccess(url);                    // inserts ![](url)
    } catch (e) {
      onError('Upload failed');          // shown via errorCallback (default: alert)
    }
  },
});
```

When `imageUploadFunction` is set, **`imageMaxSize`, `imageAccept`, `imageUploadEndpoint`, `imageCSRFToken` are not consulted** by the built-in path — validate these yourself.

---

## Autosave

```js
new EasyMDE({
  autosave: {
    enabled: true,
    uniqueId: 'post-edit-42',
    delay: 5000,         // every 5s
    submit_delay: 8000,
    text: 'Saved at ',
  },
});
```

- Persists to `localStorage` under the key `smde_<uniqueId>`.
- On init, if a saved value exists for `uniqueId`, it overrides `initialValue`.
- On the surrounding `<form>`'s `submit` event, the saved value is **cleared**.
- If localStorage is unavailable (e.g., Safari Private Mode quotas), autosave silently no-ops with a console message.
- The `autosave` status item updates a DOM node with `id="autosaved"` if one exists on the page.

To clear manually:

```js
easyMDE.clearAutosavedValue();
```

---

## Markdown Rendering Pipeline

`easyMDE.markdown(text)` and the default `previewRender` apply, in order:

1. `marked.use(markedOptions)` then `marked.parse(text)`
2. `breaks` flag flipped on/off based on `renderingConfig.singleLineBreaks`
3. If `renderingConfig.codeSyntaxHighlighting` and an `hljs` is available, `markedOptions.highlight` is set to `hljs.highlight` / `hljs.highlightAuto`
4. `renderingConfig.sanitizerFunction(html)` runs if provided (`this` is the EasyMDE instance)
5. `addAnchorTargetBlank()` adds `target="_blank"` to bare `<a>` tags
6. `removeListStyleWhenCheckbox()` strips the bullet on `<li>`s containing `<input type="checkbox">`

Override the entire pipeline with a custom `previewRender`:

```js
new EasyMDE({
  previewRender: (plainText, previewEl) => myParser(plainText, previewEl),
});
```

Return `null` to skip overwriting `previewEl.innerHTML` (useful when you control the DOM elsewhere).

---

## Common Patterns

### Flask / Django / FastAPI form integration

EasyMDE writes back to the source `<textarea>` on form submit, so the Markdown lands in `request.form['body']` (Flask), `request.POST['body']` (Django), or your form model field on whatever framework. With `forceSync: true`, the textarea is updated continuously (handy for SPA-style validation).

```html
<form method="post" action="/posts">
  {{ csrf_token }}
  <textarea id="body" name="body"></textarea>
  <button type="submit">Save</button>
</form>

<script>
  new EasyMDE({
    element: document.getElementById('body'),
    autosave: { enabled: true, uniqueId: 'post-draft', delay: 3000 },
    forceSync: true,
    spellChecker: false, // disable for languages other than English
    uploadImage: true,
    imageUploadEndpoint: '/api/upload-image',
    imageCSRFToken: document.querySelector('meta[name=csrf-token]').content,
    imageCSRFHeader: true,
    imageCSRFName: 'X-CSRFToken',
  });
</script>
```

### Sanitizing preview output

```js
import DOMPurify from 'dompurify';

new EasyMDE({
  renderingConfig: {
    sanitizerFunction: (html) => DOMPurify.sanitize(html, {
      USE_PROFILES: { html: true },
    }),
  },
});
```

### Custom previewRender (e.g., server-side render)

```js
new EasyMDE({
  previewRender: (text, el) => {
    fetch('/render', { method: 'POST', body: text })
      .then(r => r.text())
      .then(html => { el.innerHTML = html; });
    return 'Rendering…';   // shown immediately
  },
});
```

### Init in a hidden tab / modal

```js
new EasyMDE({
  element: document.querySelector('#tab-2 textarea'),
  autoRefresh: { delay: 300 },
});
```

`autoRefresh` polls visibility and calls `codemirror.refresh()` once the editor enters layout — without it, the editor often renders with collapsed height.

### Cleanly destroying an instance

```js
let easyMDE = new EasyMDE({ ... });

// Later, e.g., when navigating away in an SPA:
easyMDE.toTextArea();      // teardown UI + restore the original textarea
easyMDE.cleanup();         // remove document-level keydown listener
easyMDE = null;
```

Always call `toTextArea()` (and `cleanup()` when relevant) before discarding the reference, especially with autosave enabled — it cancels the timer and removes the saved value from `localStorage`.
