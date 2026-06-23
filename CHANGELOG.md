# Changelog

All notable changes to ALETHEIA are documented here.
This project loosely follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [1.0.0] — 2026-06

First public release of **ALETHEIA** — a single-file, offline visual HTML editor for studying the documents you build from your second brain. (It grew out of an earlier internal prototype, "Visual HTML Editor".)

### Editing
- Open any `.html` file and edit it visually: select, move, resize, rotate, multi-select, marquee, group and z-order.
- Selection handles and the rotation line follow elements live during a drag; rotation pivots around the true element centre.
- Flow-preserving edits — removing or floating content leaves an invisible spacer so the rest of the page never shifts.

### Inserting
- Text boxes, images, tables, coloured sticky notes and links.
- **21 study marks & icons** (check, X, star, question, idea, bookmark, flag, pin, search, clock, key, arrows, brackets, speech bubble, …) — outline-only, with non-scaling strokes.
- Centred, editable text inside shapes and notes (double-click to type a label in the middle).

### Annotating
- Pencil, highlighter and an **area eraser** with three size presets (each tool has its own pointer; the eraser is a ring of the chosen size and removes only the area you pass over).
- Unified colour palette across notes and the highlighter.

### Reflection notes
- Docked study panel with Notion-style `/` blocks (headings, to-dos, callouts, toggles, key terms, highlight, question, …).
- **Markdown import and export**, plus a Clear button, and an optional bring-your-own-key AI helper.
- Opening the panel scales the page down proportionally instead of reflowing it — editing is identical with the panel open or closed.

### Links, export & shortcuts
- Links open in a **new window** (and export with `target="_blank"`), so a click never replaces the document.
- Clean HTML export: editor UI is stripped; content, annotations and notes are kept; the file is self-contained.
- Full keyboard set with **Mac ⌘** support (Ctrl elsewhere): `Shift+E`/`Shift+B` modes, `Ctrl/⌘+B/I/U` formatting, undo/redo, copy/paste/duplicate, nudge, group and z-order.

### Interface
- Typographic identity (the wordmark `ALETHEIA`), dark mode for the chrome, and a three-group Insert ribbon (Objects · Annotate · Reflection).
