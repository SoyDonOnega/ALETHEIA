# ALETHEIA — Developer Manual

> Audience: an AI (or engineer) that needs to understand the **complete current state** of this project and **how every element works**, well enough to extend or fix it without re-reading the whole source first.
>
> **Document status** — Sources: the live source `ALETHEIA.html` (single file). Coverage: all subsystems, functions, IDs, storage keys and export semantics were read from source, not memory. Currency: reflects the build as of 2026‑06‑23 (ALETHEIA rebrand + typographic logo, 21 study shapes/icons, area eraser with 3 size presets, distinct tool cursors, unified note/highlighter palette, study slash‑blocks, centred editable text in notes/shapes, links open in a new window, Reflection Markdown import/export + clear + **proportional canvas scaling**, Mac ⌘ shortcuts, Align/Order/Group buttons removed). Confidence: high; line numbers are intentionally omitted (they drift) — navigate by the function names and section banners (`// === ... ===`) quoted throughout.

---

## 1. What this project is

A **single‑file, offline, vanilla‑JS visual editor** for arbitrary HTML documents, branded **ALETHEIA** (Heidegger's *aletheia*, "unconcealment", + *IA*). You open an `.html` file you generated (e.g. study material), and edit/annotate it visually — "operate HTML like PowerPoint": select, move, resize, rotate, restyle, insert objects and study icons, annotate (highlighter/pencil/eraser), take side notes (Reflection panel, optionally AI‑assisted, Markdown in/out), then download the edited HTML.

- **No build step, no framework, no dependencies, no network** (except optional AI calls). Pure HTML + CSS + one IIFE of JavaScript.
- Works from `file://` (double‑click) — everything runs in the browser; files never leave the device.
- **Framing: a study tool.** Inserted "shapes" are study‑oriented marks and icons (check, X, star, idea, bookmark, pin…); the Reflection panel is for note‑taking with Markdown round‑trip.
- Aesthetic of the editor chrome: black/white/grey, system font, no accent colours (a house rule). The brand is **typographic** — the wordmark `ALETHEIA` and the letter `A` as the icon. *User content* (coloured notes, highlighter) may be coloured — that is content, not chrome.

---

## 2. Repository layout

```
aletheia/
├── ALETHEIA.html      ← THE PRODUCT (standalone, self-contained). Edit this.
├── README.md          ← project overview (the "why", quick start)
├── DEVELOPER.md       ← this file (architecture & internals)
├── CHANGELOG.md       ← version history
├── LICENSE            ← MIT
├── docs/              ← Manual_ALETHEIA_User_V1_EN.docx + screenshots
└── examples/          ← a sample study HTML to open and try
```

### Branding integration (single-file invariant)
The app loads **zero external assets**, so the brand is **inlined**: the favicon is a `data:image/svg+xml;base64,…` letter‑`A` mark in `<head>` (dark‑mode aware), and the ribbon and welcome hero render the wordmark as the plain caps text `ALETHEIA`. The identity is **typographic** — there is no pictorial logo. The macOS Finder icon of `ALETHEIA.html` can be set via its resource fork (`icns`, flag `C`); this is OS metadata and does not travel to non‑Mac filesystems.

---

## 3. High-level architecture

```
┌───────────────────────────────────────────────────────────────┐
│ EDITOR CHROME (parent document) — never exported               │
│  • #ribbon  (tabs: Home / Insert / Format ; mode toggle; theme)│
│  • #app (flex):  #canvas-wrap (flex:1)  |  #reflection-panel    │
│  • #panel (floating Properties)                                 │
│  • popups: #shapes-popup #table-popup #note-menu #highlight-menu│
│            #pen-menu #eraser-menu #slash-menu                   │
│  • #iframe-overlay  (transparent; captures pointer in Edit mode)│
└───────────────────────────────────────────────────────────────┘
        │ loads user HTML via iframe.contentDocument.write()
        ▼
┌───────────────────────────────────────────────────────────────┐
│ #content-iframe  → THE DOCUMENT (the user's HTML = "the canvas")│
│  this is what gets edited and exported                          │
└───────────────────────────────────────────────────────────────┘
```

- The loaded page lives in an **iframe**; that iframe *is* the canvas. The editor edits `iframeDoc` (`iframe.contentDocument`), never the parent DOM as "the document".
- The whole app is **one IIFE** `(function(){ ... })();` at the bottom of the file. All state and functions are closure‑scoped (nothing on `window`). Function declarations are hoisted, so call‑order within the IIFE doesn't matter.
- `iframeDoc` is the module‑level handle to the current iframe document. It is **re‑assigned on every load and every undo/redo** (history is restored by rewriting the iframe), so listeners that must survive history navigation are (re)bound inside `setupIframeListeners()`.
- **Reflection scaling:** when the Reflection panel is open, the iframe is shrunk with a CSS `transform: scale()` instead of being narrowed, so the loaded HTML never reflows. A module‑level `canvasScale` (1 when closed) carries the factor; all pointer math divides by it. See §8.10 and §12.

### Two modes (`switchMode(toEdit)`, state `editMode`)
- **Browse** (`#btn-mode-nav`, `editMode=false`): `#iframe-overlay` is hidden → the page is live. Links open in a **new window** (see §8.9). The user scrolls, clicks links, selects text natively.
- **Edit** (`#btn-mode-edit`, `editMode=true`): `#iframe-overlay` (`.active`, `z-index:10`, fills the iframe) **captures all pointer events**. Mouse interactions are interpreted by the overlay's `mousedown/mousemove/mouseup/dblclick/wheel` handlers (select, drag, resize, rotate, marquee, draw, erase). Wheel is forwarded to the iframe for scrolling.

---

## 4. Core mental model & invariants

Read this before touching anything.

1. **Editor objects vs native content.**
   - *Editor objects* are elements the editor inserted, tagged `data-editor-object="<type>"`. Types: `shape`, `textbox`, `image`, `table`, `note`, `group`. They are **absolutely positioned** (free‑canvas / PPTX‑style). Deleting/moving them never reflows the page.
   - *Native content* = the original page's elements (in normal flow). They are selectable/editable too, but they reflow if removed from flow.
2. **Flow preservation (the spacer rule).** Whenever a **flow** element is removed from flow (on delete, or when `toAbsolute()` floats it for group), the editor leaves an **invisible same‑footprint spacer** (`<div class="__editor-spacer__">`, identical margins, `visibility:hidden`) in its place, so the rest of the page stays put. See `flowSpacer()`, `removeEl()`, `toAbsolute()`.
3. **Reflection notes are chrome, not document.** The Reflection panel lives in the parent. Its content is **injected into the export** as `<aside data-editor-reflection>` at download time and **round‑tripped back** into the panel on load — but it is not part of the live iframe document while editing. It also exports/imports as Markdown independently (§8.10).
4. **History = full‑document snapshots.** Undo/redo serialize the *entire* iframe document to an HTML string (`getCleanHTML()`), and restore by `document.write()`. Any DOM mutation becomes undoable by calling `saveState()` afterward; selection/handles are cleaned before serialization.
5. **Export strips editor artifacts but keeps annotations and object text.** See §9.
6. **Coordinate spaces matter — including the Reflection scale.** See §12.

---

## 5. State model (module-level variables)

| Variable | Meaning |
|---|---|
| `iframeDoc` | current iframe document (re-assigned on load/restore) |
| `loaded`, `editMode` | a file is loaded / Edit mode active |
| `selectedEl` | the "primary" selected element (drives the Properties panel + handles) |
| `selectedEls[]` | full multi-selection (marquee / Ctrl+click); `selectedEl` is its last member |
| `isDragging`, `isResizing`, `isRotating` | active manipulation gesture |
| `dragStart`, `resizeStart`, `rotateStart`, `resizeHandle`, `resizePlaceholder` | gesture scratch state |
| `pendingSelect` | deferred click: decide on mouseup whether it was a click (select) or a drag (marquee) |
| `isMarquee`, `marqueeStart`, `marqueeEl` | rubber-band selection state |
| `history[]`, `historyIndex` | undo stack of `getCleanHTML()` strings (cap 60) |
| `clipboardEl`, `clipboardSource`, `pasteCount` | copy/paste |
| `drawTool` (`null\|'pen'\|'marker'\|'eraser'`), `penColor/penWidth`, `markerColor/markerWidth`, `eraserWidth`, `curStroke/curPts`, `erasing` | annotation tools (`markerColor` default `#FFE45C`; `eraserWidth` presets 14/28/48) |
| `slashOpen`, `slashStart`, `slashIndex`, `slashDoc`, `slashOffset` | slash-command menu |
| `gridVisible`, `zoomLevel`, `canvasScale`, `lastClickedCell`, `selectedTableCells[]` | misc. `canvasScale`<1 only while Reflection scales the canvas; `zoomLevel = canvasScale*100` keeps resize math in sync |

---

## 6. UI surface (ribbon, panels, popups)

### Ribbon tabs and their buttons (IDs)
Group labels are hidden by CSS (`.ribbon-group-label{display:none}`); `.ribbon-group` blocks are separated by a thin right border. Buttons identify by icon + text + `title`.

- **Home**: `btn-upload`, `btn-download` (primary; stays solid‑dark when disabled — no transparent hover), `btn-undo`, `btn-redo`, `btn-copy`, `btn-duplicate`, `btn-delete`, `btn-grid-toggle`. Mode toggle: `btn-mode-nav`, `btn-mode-edit`. Theme: `btn-theme`.
- **Insert** — **three separated groups**:
  1. Objects: `btn-insert-textbox`, `btn-insert-shape`, `btn-insert-image`, `btn-insert-table`, `btn-insert-note`, `btn-insert-link`.
  2. Annotate: `btn-insert-pencil`, `btn-insert-highlight`, `btn-insert-eraser`.
  3. `btn-insert-reflection`.
  Picking any *object/annotation* insert tool deactivates an active draw tool (`if(drawTool) setDrawTool(null)`).
- **Format**: text style `btn-bold`/`btn-italic`/`btn-underline` (letter‑only buttons **B/I/U** — no duplicate icon; shortcut `Ctrl/⌘+B/I/U`); *text* align `btn-align-left/center/right`; style `btn-fill-color` (paint‑bucket icon), `btn-stroke-color` (bold frame icon), `btn-shadow` (offset‑squares icon); rotate `btn-rotate-left`/`btn-rotate-right`/`btn-flip-h`. **Align, Distribute, Order and Group/Ungroup ribbon buttons were removed** — group/order remain **keyboard‑only** (§10); `alignObjects`/`distributeObjects` still exist in code but have no UI.

### Panels / popups (element IDs)
- `#panel` — floating **Properties** panel (font, size, colours, opacity, border, shadow, position/size, rotation, image replace, table toolbar `#section-table`). Draggable by `#panel-drag-handle`. Positioned by `positionPanelNearElement()` which is **`canvasScale`‑aware**.
- `#reflection-panel` — docked right notes panel. Header buttons: `#reflection-import-md` (`↑ md`), `#reflection-export-md` (`↓ md`), `#reflection-clear` (`Clear`), `#reflection-close` (`×`). Body `#reflection-body`, resize `#reflection-resize`, AI section, hidden `#reflection-import-input` file picker.
- Popups (class `.menu-popup`): `#shapes-popup`, `#table-popup`, `#note-menu`, `#highlight-menu`, `#pen-menu`, `#eraser-menu`, `#slash-menu`. All closed by `closeAllPopups()` and an outside‑click listener. (`#align-menu` and `#order-menu` were removed.)

---

## 7. Object & annotation model (what lives in the iframe)

| Marker | What it is | Selectable? | Exported? |
|---|---|---|---|
| `[data-editor-object="shape"]` | inserted study mark/icon (outline‑only SVG, `vector-effect:non-scaling-stroke`); contains a centred editable label | yes | yes |
| `[data-editor-object="textbox"]` | inserted text box | yes | yes |
| `[data-editor-object="image"]` | inserted image (data‑URI) | yes | yes |
| `[data-editor-object="table"]` | inserted table | yes | yes |
| `[data-editor-object="note"]` | coloured sticky note (flex‑centred); text lives in inner `.__note-text__` | yes | yes |
| `[data-editor-object="group"]` | wrapper grouping ≥2 objects | yes (whole group) | yes |
| `.__shape-label__` / `.__shape-text__` | a shape's centred text layer (`label` is `inset:0` flex, `pointer-events:none`; `text` is the editable inner block) | no (transparent to selection) | **yes** (content) |
| `.__note-text__` | a note's centred editable text block | no (selects the note) | **yes** (content) |
| `<a data-editor-link target="_blank" rel="noopener…">` | wrapper added by the Link tool around an object | transparent to selection | yes |
| `#annotation-pen-layer` | one SVG holding all freehand strokes; `<path data-annot="pen">` (opaque) and `<path data-annot="marker">` (translucent, `stroke-opacity:.45`) | no (pointer‑events:none) | **yes** |
| `.__editor-spacer__` | invisible flow placeholder (invariant 2) | no | **yes** |
| `<aside data-editor-reflection>` | reflection notes, injected at export, removed after | n/a | **yes** (export only) |
| `.__editor-*` (handles, grid, marquee, styles, placeholders) | transient editor UI inside the iframe | n/a | **no** |

`findSelectable(el)` walks up from the clicked node and returns the nearest `[data-editor-object]`; if none, the raw element is used (native elements are selectable too). It skips `.__editor-*` UI, the grid, and handle elements.

---

## 8. Subsystems & how each works

### 8.1 Load / import — `loadFile`, `loadIntoIframe`, `initAfterLoad`
`btn-upload` / drag‑drop → `FileReader` → `loadIntoIframe(html)` → `iframeDoc.open(); write(html); close()`. On load it hides the welcome overlay, enables Download, resets history with one `saveState()`, calls `setupIframeListeners()`, and **round‑trips reflection notes**: if the loaded doc contains `[data-editor-reflection]`, its inner HTML is moved into `#reflection-body`, removed from the doc, and the panel opens.

### 8.2 setupIframeListeners() — runs on every load AND every undo/redo
Injects `<style id="__editor-styles__">` (handle styles + `[contenteditable=true]{outline:none}`) and `<style id="__no-hscroll__">`. Binds the iframe `click` handler that governs link navigation (§8.9), the in‑iframe `keydown` shortcuts (§10), `initSlashMenu(iframeDoc, iframeOffset)`, and the horizontal‑scroll killer. **Anything that must keep working after undo must be (re)bound here**, because the iframe document is replaced on restore.

### 8.3 Selection
- **Click**: overlay `mousedown` → convert to iframe‑internal coords (`(clientX-rect.left)/canvasScale`) → `elementFromPoint` → `findSelectable`. Native/element clicks are **deferred** (`pendingSelect`) and resolved on `mouseup` (press‑drag → marquee, press‑release → select). Editor objects select immediately so they can be dragged.
- **Ctrl/Cmd+click**: toggles membership in `selectedEls`.
- **Marquee** (`marqueeSelect`): press‑drag on empty canvas draws `#__editor-marquee__`; on release selects objects **fully enclosed** (PPTX semantics) — and **only `[data-editor-object]` objects**. All marquee coords are `canvasScale`‑divided.
- `selectElement` / `deselectElement` manage outline, handles, the Properties panel, and table toolbar.

### 8.4 Move / resize / rotate / smart guides / handles
- **Drag**: overlay `mousemove` while `isDragging` updates `style.left/top` for every element in `selectedEls` (delta `(clientX-dragStart.x)/canvasScale`). It calls **`repositionOverlayHandles()`** so the handles/rotation line follow live, and `showSmartGuides`. On `mouseup` it rebuilds handles (`addResizeHandles`).
- **Resize**: 8 handles (`addResizeHandles`); corner handles honour **Shift = lock aspect**. Divides deltas by `zoomLevel/100` (= `canvasScale`). Flow elements get a placeholder during resize then return to flow with explicit size; handles refresh on mouseup.
- **Rotate**: rotate handle; **Shift = snap to 15°**. The pivot centre is computed in **parent‑viewport coords** (`iframeRect + internalCentre*canvasScale`) so it matches `e.clientX/Y` (this also fixes the historical pivot‑offset bug). `rotateEl(deg)` for the ±90° buttons; flip via `scaleX(-1)`.
- **Handles** live inside the iframe. For positioned editor objects they are appended as children (move with the element). For **flow** elements they live in an external `#__editor-handles-overlay__` positioned in iframe‑document coords — `repositionOverlayHandles()` keeps that overlay tracking the element during drag, and gestures rebuild it after.

### 8.5 Arrange — `// === ARRANGE ... ===`
- `groupSelected` / `ungroupSelected` — wrap/unwrap in `div[data-editor-object="group"]`; **Shift+G** toggles via `groupOrUngroup` (keyboard‑only — no button).
- Z‑order `orderZ('front'|'back'|'forward'|'backward')` using `getMaxZ`/`getMinZ`; **Shift+]/Shift+[** for forward/backward (keyboard‑only — no button).
- `alignObjects(edge)` and `distributeObjects('h'|'v')` still exist (spacer‑safe via `toAbsolute()`) but are **not wired to any UI** after the Align/Order removal.

### 8.6 Insert
- **Text box** — absolute `div[data-editor-object=textbox]`, transparent background, thin border; default text "Type here…".
- **Shapes** (`SHAPES[]`, `insertShape`) — **21 study marks/icons**, outline‑only (no fill until the user adds one), strokes kept crisp with `vector-effect:non-scaling-stroke`. Order: **marks** (Check, X mark, Star, Important, Question) → **enclose** (Circle, Rectangle, Bracket [single `[`], Brace, Speech bubble) → **connect** (Arrow, Arrow connector, Line, Plus) → **study icons** (Idea, Bookmark, Flag, Pin, Search, Clock, Key). Each inserted shape gets a centred editable label (`.__shape-label__` → `.__shape-text__`).
- **Image** — file → `fileToBase64` → `<img data-editor-object=image>`. Replace via `#btn-replace-img`.
- **Table** — `#table-popup` grid picker → `insertTable(rows,cols)`; full cell editing (`enterTableCellEditMode`, add/del row/col, merge/split).
- **Note** — `#note-menu` swatches (unified 6‑colour palette) → `insertNote(color)` → flex‑centred sticky `div[data-editor-object=note]` with inner `.__note-text__`.

### 8.7 Text editing + Notion-style slash menu
- Double‑click a text element → `contentEditable=true`, overlay off, edit; blur → `contentEditable=false` + `saveState()`. The injected outline‑only rule means editing never reflows the box.
  - **Notes/shapes** edit their **inner centred text block** (`.__note-text__` / `.__shape-text__`), not the container (avoids Safari flex+contenteditable quirks; keeps text centred).
  - **Default placeholders** ("Note", "Type here…") are **selected on edit** so the first keystroke (or `/`) replaces them.
- **Slash menu** (`initSlashMenu(doc, offsetFn)`, `insertBlock`, `blockNode`): typing `/` at start‑of‑line / after whitespace in *any* editable (iframe text boxes, table cells, notes, shape labels, **and** the parent Reflection panel) opens `#slash-menu` at the caret, positioned with `slashOffset` (iframe rect, or `{0,0}` for the parent). Blocks: **Basic** — Text, H1/H2/H3, To‑do, Bulleted, Numbered, Quote, Divider, Code, Table; **Study** — Callout, Toggle (`<details>`), Key term (`<dl>`), Highlight (`<mark>`), Question (Q/A). Keyboard ↑/↓/Enter/Tab/Esc; mouse items use `mousedown`+`preventDefault` to keep the editable focused.

### 8.8 Annotation tools — `// === ANNOTATION TOOLS ===`
One model, three tools via `setDrawTool('pen'|'marker'|'eraser'|null)` (toggle; click again or **Esc** = off). All strokes go into one SVG `#annotation-pen-layer` (`pointer-events:none`, `z-index:9000`). All draw coords are iframe‑document coords (`(clientX-rect.left)/canvasScale + scroll`).
- **Distinct cursors** (`cursorFor`/`applyDrawCursor`): pencil glyph; highlighter glyph **tinted with the active colour**; eraser is a **ring sized to `eraserWidth`**. The marker cursor refreshes when its colour changes.
- **Pencil** — opaque `<path data-annot="pen">`, colour via `#pen-menu` swatches (4) + size slider `#pen-size`.
- **Highlighter (marker)** — translucent band `<path data-annot="marker" stroke-opacity="0.45" stroke-width="16">`, colour via `#highlight-menu` (the **same 6‑colour palette as notes**).
- **Eraser** — `#eraser-menu` offers **three size presets (14/28/48)** shown as **circle previews**; picking one applies it and **closes the menu**. `eraseAt(docX,docY)` is **area/segment‑level**: `parseStrokePath` parses each stroke's polyline, drops the points within radius `eraserWidth/2` (sampling segments), and rewrites the `d` as the surviving sub‑paths (or removes the stroke if nothing survives). It only touches pen/marker strokes — never notes or objects.
- Drawing is in the overlay `mousedown/move/up` via `startStroke`/`extendStroke`/`eraseAt`; each finished stroke or erase ends with `saveState()`.

### 8.9 Hyperlinks — `btn-insert-link`, `setObjectLink`
`window.prompt` for a URL. Text selected in an editable → `execCommand('createLink')` then the new anchors get `target="_blank" rel="noopener noreferrer"`. Else the selected object is wrapped in `<a data-editor-link target="_blank" rel="noopener…">` (or updated/removed). **Navigation handling** (iframe `click` listener): in **Edit** mode all link clicks are blocked; in **Browse** mode external links are opened in a **new window** via `window.open(href,'_blank')` (so a click never replaces the iframe document), while in‑page `#…` anchors stay native (TOC jumps). Exports keep `target="_blank"`.

### 8.10 Reflection panel — `// === REFLECTION ===`
- Toggled by `btn-insert-reflection`; flex sibling of `#canvas-wrap`. Closed by `#reflection-close`.
- **Proportional scaling — `applyReflectionScale()` / `canvasScale`.** Opening the panel does **not** narrow the iframe (which would reflow the HTML). Instead the iframe keeps its full design width and is shrunk with `transform: scale(s)` (origin top‑left), `s = canvasWrapWidth / appWidth`; `canvasScale=s`, `zoomLevel=s*100`. Closing restores width/height/transform and `canvasScale=1`. Called on open/close, panel resize, and window resize. **Because every pointer path divides by `canvasScale`, editing is identical with or without the panel; with the panel closed it is a strict no‑op.**
- **Resize**: drag `#reflection-resize` (left edge) → width clamped [260,760]; persisted to `localStorage['vhe_reflection_w']`; recomputes the scale live.
- **Editing**: `#reflection-body` is contenteditable, slash‑enabled, `:empty` placeholder.
- **Persistence**: debounced autosave to `localStorage['vhe_reflection_notes']`; embedded in export as `<aside data-editor-reflection>` and round‑tripped on load.
- **Markdown export** (`#reflection-export-md`, `reflectionToMarkdown()`): walks the body to Markdown — headings, lists, **to‑dos `- [ ]`/`- [x]`**, blockquotes, code, tables, `**bold**`/`*italic*`/`<u>`/`` `code` ``/`==highlight==`/links, plus the study blocks; downloads `reflection-notes.md`.
- **Markdown import** (`#reflection-import-md` → `#reflection-import-input`, `markdownToHtml()`): converts Markdown → HTML and **appends** it to the notes (with a divider; never destroys existing notes).
- **Clear** (`#reflection-clear`): empties the notes after a confirm.

### 8.11 AI assistant — `// === REFLECTION AI ASSISTANT ===`, `aiAction`, `aiCall`
Collapsible section in the Reflection panel. Settings in `localStorage`: `vhe_ai_provider` (`anthropic`|`openai`|`compatible`), `vhe_ai_key`, `vhe_ai_base`, `vhe_ai_model`. Actions: **Summarize notes**, **Explain selection**, **Generate questions**.
- Anthropic: `POST https://api.anthropic.com/v1/messages` with `x-api-key`, `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`; default model `claude-haiku-4-5-20251001`.
- OpenAI/compatible: `POST {base||https://api.openai.com/v1}/chat/completions` with `Authorization: Bearer`.
- **Offline reality**: the editor stays offline; the AI *call* needs internet. The key lives only in the browser. From `file://` some providers may refuse CORS — serving the file locally fixes it. Failures are surfaced inline and never block the editor.

### 8.12 Format / properties — `updatePanel` + panel listeners
Font family/size/colour, background + opacity, border, shadow toggle, X/Y/W/H/rotation inputs, image replace, table toolbar. Ribbon `btn-bold/italic/underline` map to inline styles via `toggleBlockFormat(el,'B'|'I'|'U')`; *text* align sets `text-align`. Fill/Stroke/Shadow buttons open colour inputs / toggle shadow.

### 8.13 History — `saveState`, `getCleanHTML`, `undo`/`redo`, `restoreFromHistory`
`saveState()` pushes `getCleanHTML()` (cap 60). `restoreFromHistory()` deselects, `document.write`s the snapshot, re‑runs `setupIframeListeners`. Reflection‑panel edits do **not** go through this stack (they autosave to localStorage). **Object** undo/redo is bound to `Ctrl/⌘+Z` etc. but is **skipped while typing in an editable**, so native text undo works there.

### 8.14 Grid — `btn-grid-toggle`
Toggles a fixed `#__editor-grid__` overlay (20px grid). Removed from export.

---

## 9. Export semantics (`getCleanHTML` → download `documento-editado.html`)

**Stripped** (transient editor UI): resize/rotate handles + rotation line, resize placeholders, `#__editor-grid__`, all `contenteditable` attributes, all `data-editor-selected`, the injected anchor‑click‑blocker helper.
**Injected for export then removed from the live DOM**: `<aside data-editor-reflection>` with the panel notes.
**Kept (content/layout)**: every editor object, **the centred text of notes/shapes** (`.__note-text__`/`.__shape-label__`/`.__shape-text__`), `<a data-editor-link target="_blank">`, the `#annotation-pen-layer` SVG (pen + marker strokes), and all `.__editor-spacer__` placeholders.
**Known leak (harmless)**: the injected `__editor-styles__`/`__no-hscroll__` blocks ship as dead CSS.

---

## 10. Keyboard shortcuts

Bound in **two** places — the parent `document` keydown and the in‑iframe keydown (in `setupIframeListeners`) — so they fire regardless of focus. Both use `const cmd = e.ctrlKey || e.metaKey`, so **on Mac everything works with ⌘** (Ctrl on Win/Linux).

| Keys | Action | Notes |
|---|---|---|
| Shift+E / **Shift+B** | Edit / **Browse** mode | Browse moved from Shift+N → **Shift+B** |
| **Ctrl/⌘+B / I / U** | bold / italic / underline | on a selected block; while editing text the browser formats natively (handler stands down) |
| Shift+G | group ↔ ungroup toggle | keyboard‑only (no button) |
| Shift+] / Shift+[ | bring forward / send backward | detected as `}` / `{`; keyboard‑only |
| Ctrl/⌘+Z | undo | skipped while typing (native text undo) |
| Ctrl/⌘+Shift+Z · Ctrl/⌘+Y | redo | skipped while typing |
| Ctrl/⌘+C / V | copy / paste object | |
| Ctrl/⌘+D | duplicate | `preventDefault` always (kills Safari's "add bookmark"); duplicates only if something is selected |
| Delete / Backspace | delete selection | guarded by `isTypingInEditable` |
| Arrows (Shift = 10px) | nudge selection | whole multi‑selection moves; ignored with ⌘/Ctrl held |
| Esc | exit draw tool → else deselect + close popups | also closes the slash menu |

Slash menu (when open): ↑/↓ navigate, Enter/Tab insert, Esc close.

---

## 11. localStorage keys
`vhe_theme` (`light`|`dark`), `vhe_reflection_w` (panel width), `vhe_reflection_notes` (notes HTML), `vhe_ai_provider`, `vhe_ai_key`, `vhe_ai_base`, `vhe_ai_model`. All writes are wrapped in try/catch (Safari/`file://` may block storage; the editor degrades silently).

### Dark mode (`#btn-theme`)
Toggles `html.dark`, redefining the chrome CSS variables to dark values; persisted in `vhe_theme`, restored by `applyTheme`. **Editor chrome only** — the loaded document inside the iframe keeps its own colours. The favicon and the brand SVGs carry their own `prefers-color-scheme` switch.

---

## 12. Coordinate systems (read before touching pointer code)

Four things are in play:
1. **Parent viewport** — `e.clientX/Y` on the overlay (the overlay is in the parent).
2. **Iframe viewport** — `clientX − iframeRect.left`, `clientY − iframeRect.top`. `iframeDoc.elementFromPoint` and any iframe‑element `getBoundingClientRect()` use **this** space (unaffected by the parent's CSS transform).
3. **Iframe document** — iframe‑viewport + scroll. Stroke `d` coords and absolute `style.left/top` math use this.
4. **The Reflection scale `canvasScale`** — when the panel is open the iframe is CSS‑scaled by `s<1` (origin top‑left). So:
   - **parent → iframe‑internal**: `(clientX − iframeRect.left) / canvasScale` (this is applied at *every* pointer entry: mousedown, mousemove draw/marquee/drag, dblclick, table‑cell hit‑test).
   - **iframe‑internal → parent**: `iframeRect.left + internalX * canvasScale` (used by `positionPanelNearElement` and the rotate pivot).
   - Things drawn **inside** the iframe (handles, marquee, smart guides, strokes) stay in internal coords and scale visually for free.
   With the panel closed `canvasScale=1`, so all of the above are no‑ops.

Rules of thumb: hit‑testing/marquee = iframe‑internal (divide by scale); drawing/positioning on elements = iframe‑document; popups in the parent add `iframeOffset()` and multiply by scale.

---

## 13. Known constraints, tradeoffs & gotchas

- **Free‑canvas vs responsive**: manipulated objects become absolutely positioned (px). Exported HTML is intentionally non‑responsive for those.
- **Spacers persist in the export** (deleting flow content leaves invisible placeholders so nothing shifts). By design.
- **Reflection scaling** keeps the HTML from reflowing, but relies on every pointer path being `canvasScale`‑aware. If you add a new pointer handler, divide its parent→iframe conversion by `canvasScale`.
- **Eraser is area/segment‑level** with three preset radii — not pixel‑level and not free radius.
- **Highlighter is a translucent overlay band**, not true multiply (overlay isolation). Text stays readable through 45% ink.
- **Align/Distribute live in code but have no UI** (buttons removed); Group/Order are keyboard‑only.
- **AI is online‑only**; local models are out of scope.
- **`document.write` + `execCommand`** are used (deprecated but universally supported); acceptable for an offline single‑file tool.
- **Markdown export** covers the blocks the slash menu produces; hand‑pasted exotic HTML may convert imperfectly (never crashes).

---

## 14. How to test

There is no test runner. Verified workflow:
- Serve the folder with a static server (`python3 -m http.server`) and drive the page with a headless browser / preview tool. (Pure functions like `markdownToHtml` and the eraser's `parseStrokePath` math can be unit‑checked with `jsc`/Node; element‑id and dangling‑reference integrity can be grep‑checked.)
- `examples/` holds a sample study page (bring your own for wider coverage). Per page: load → render check → enter Edit → insert each object type + a few study icons → type into a note/shape (centred) → select a native element + nudge → toggle Reflection (check **nothing reflows**, and select/drag/draw still land under the cursor) → export → assert no editor artifacts and **zero console errors**.
- Manual checks that matter: editing/deleting **does not reflow** the rest (spacer invariant); marquee selects only objects; undo/redo round‑trips; `⌘Z` while typing is native text undo; links open in a new window (Browse mode); Markdown export → import round‑trips; the Properties panel and rotate pivot stay glued to the object **with Reflection open**.

---

## 15. Roadmap / backlog (decision tree for the next agent)

**Shipped**: modes; select (click/ctrl/marquee); move/resize/rotate (parent‑coord pivot) with live‑following handles; group + z‑order (keyboard); insert (text / 21 study shapes+icons with centred labels / image / table / coloured notes); table editing; text editing + slash blocks (basic + study); pencil/highlighter/area‑eraser with distinct cursors and unified palette; hyperlinks that open in a new window; docked Reflection panel + AI + Markdown import/export + clear + **proportional canvas scaling**; undo/redo; grid; clean export with reflection round‑trip; ALETHEIA branding (inline favicon + macOS icon); Mac ⌘ shortcuts.

**Candidate next steps** (each independently shippable):
- **P1**: whole‑document autosave + "restore session"; "select all objects"; change note/shape colour after insert; re‑expose Align/Distribute behind a small menu if wanted.
- **P2**: multi‑object resize; snap‑to‑object guides; explicit canvas zoom UI (reuse `canvasScale`); "Save to file" via File System Access API; PDF/PNG export.
- **P3**: layers panel; format painter; comment pins; in‑document search; strip the dead `__editor-styles__` CSS from exports; pixel‑level eraser.

When extending: keep it a single offline vanilla file; tag new inserted things with `data-editor-object`; leave a `flowSpacer` whenever you pull flow content out of flow; **divide any new parent→iframe pointer math by `canvasScale`**; end every mutating action with `saveState()`; re‑bind iframe listeners inside `setupIframeListeners`; verify against a spread of real pages with zero console errors.
