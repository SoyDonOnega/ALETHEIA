# Changelog

All notable changes to ALETHEIA are documented here.
This project follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [0.1] — 2026-06-23

First release of **ALETHEIA** — a single-file, offline visual HTML editor for studying the documents you build from your second brain.

### Editing
- Open any `.html` file and edit it visually: select, move, resize, rotate.
- Multi-select via marquee (drag on empty canvas) or Ctrl/⌘+click; all move and format operations apply to the full selection.
- Selection handles and the rotation line follow elements live during a drag; rotation pivots around the true element centre (Shift to snap 15°).
- Flow-preserving edits — removing or floating content leaves an invisible spacer so the rest of the page never shifts.

### Inserting
- Text boxes, images, tables, coloured sticky notes and links.
- **21 study marks & icons** — outline-only, non-scaling strokes: Check, X mark, Star, Important, Question; Circle, Rectangle, Bracket, Brace, Speech bubble; Arrow, Arrow connector, Line, Plus; Idea, Bookmark, Flag, Pin, Search, Clock, Key.
- Centred, editable text inside shapes and notes (double-click to type a label in the middle).

### Annotating
- Pencil, highlighter and an **area eraser** with three size presets; each tool has its own cursor (pencil glyph, tinted highlighter, ring eraser).
- Unified six-colour palette shared by notes and the highlighter.

### Formatting
- Format ribbon (bold, italic, underline, text align, fill colour, stroke colour, shadow, rotate ±90°, flip horizontal) and the Properties panel (font, size, colour, background, opacity, border-radius, position, rotation) all apply to the **complete multi-selection** at once.

### Reflection notes
- Docked study panel with Notion-style `/` blocks: Basic (text, H1–H3, to-do, bulleted, numbered, quote, divider, code, table) and Study (Callout, Toggle, Key term, Highlight, Question/A).
- **Markdown import and export**, Clear button, and an optional bring-your-own-key AI helper (Anthropic / OpenAI / compatible).
- Opening the panel scales the page down proportionally instead of reflowing it — editing is identical with the panel open or closed.

### Paste & export
- **System clipboard paste** — Ctrl/⌘+V accepts OS clipboard images (inserted as data-URI `<img>` objects) or plain text (inserted as a text box).
- **Filename-preserving export** — downloaded as `<original-filename>-editado.html`; editor UI stripped, content, annotations and notes kept; file is self-contained.

### Links & navigation
- Links open in a **new window** (and export with `target="_blank"`), so a click never replaces the document.
- Loading a new file while one is already open fully replaces the canvas and resets editor state.

### Interface
- Typographic identity (the wordmark `ALETHEIA`), dark mode for the editor chrome, three-group Insert ribbon (Objects · Annotate · Reflection).
- Full keyboard set with **Mac ⌘** support (Ctrl elsewhere): `Shift+E`/`Shift+B` modes, `Ctrl/⌘+B/I/U` formatting, undo/redo, copy/paste/duplicate, nudge.
