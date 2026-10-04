# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

«Полка» — a client-only FB2 / FB2.ZIP / EPUB reader PWA, deployed as static files to GitHub Pages (branch `main`, root folder; `.nojekyll` disables Jekyll). Books never leave the user's browser. UI text, comments, and README are in Russian — keep new user-facing strings and code comments in Russian to match.

There is no build step, package manager, linter, or test suite. To run locally, serve the folder over HTTP (the service worker only registers in a secure context, which includes `localhost`):

```sh
python3 -m http.server 8000   # then open http://localhost:8000/
```

## Release rule

Whenever `index.html` (or any cached asset) changes for a release, bump `VERSION` in `sw.js` (e.g. `polka-0.3.0` → `polka-0.4.0`). The service worker serves own files cache-first, so without a version bump users keep the old cached copy. The README's feature list is labeled with the version too.

## Architecture

Everything lives in `index.html` (~1850 lines): inline `<style>`, markup, and one `'use strict'` inline `<script>`. The only external dependencies are JSZip 3.10.1 from cdnjs and Google Fonts (Literata, PT Serif, Golos Text); `sw.js` caches those hosts at runtime for offline use (`RUNTIME_HOSTS`). If you add another CDN host, add it there too.

The script is split by `/* ===== Section ===== */` banners:

- **Storage** — `DB` wraps IndexedDB `polka-reader` (v1) with two stores keyed by `id`: `meta` (title, authors, cover thumbnail, `pos: {ch, r}`, `progress`, `done`, `bookmarks`, `quotes`, etc.) and `files` (the raw original `ArrayBuffer`). Falls back to in-memory Maps if IndexedDB is unavailable (`#memWarn` is shown). Changing store layout requires bumping the DB version and handling `onupgradeneeded`.
- **Settings** — `S` object merged from `DEFAULTS` and `localStorage['polka-settings']`; `applySettings()` pushes them into CSS.
- **Parsing** — `parseBook(buf)` detects format: FB2 (incl. windows-1251 via `decodeXml`, zipped FB2 via JSZip) → `parseFb2Text`, EPUB → `parseEpub`. Both produce a common book shape `{ title, authors, lang, cover, chapters: [{title, html, isNotes}], toc }` with chapters as sanitized HTML strings. EPUB sanitizing uses the `BAD` tag blocklist and `KEEP` attribute allowlist; links go through `safeUrl`. Element lookup is namespace-agnostic via `lnOf` (strips `fb-` prefixes).
- **Post-processing** — `finalize(book)` splits oversized chapters (>60k chars into ~30k parts via `splitInto`), builds `idMap` (element id → chapter index, used for footnotes/TOC/internal links), and computes `len`/`cum`/`total` for whole-book progress. Books are re-parsed from the stored raw file each time they are opened.
- **Library** — shelf rendering with search/sort/filter state `L` (persisted in `localStorage['polka-library']`), `addFiles` (dedupe by name+size, drag-and-drop, cover `thumb`; files named `polka-backup*.zip` go to `importBackup`), per-book sheet (`openBookSheet`). A book is done if `meta.done === true` or progress ≥ 99.5% unless `done === false` (`isDone`).
- **Reader** — runtime state in `R`. Only one chapter is in the DOM at a time (`renderChapter` into `#flow`). Paged mode uses CSS multi-column layout with page width `R.W` + gap `R.G`, paging by `translateX`; scroll mode scrolls `#viewport`. `position(where)` accepts `'end'`, `{id}`, `{off, len}` (character offset into chapter text, used by search/bookmarks via `buildNodes`/`locate`/`offsetRange`), or `{ratio}`. `R.tok` guards against stale async renders. Position is saved debounced via `savePos()` → `writePos()`. Footnotes open in a sheet (`showNote`); bookmarks, quotes, search, TOC, and settings are bottom sheets (`openSheet`/`closeSheets`). Bookmarks and quotes store `{ch, off}` — `off` is a character offset into the chapter's text (same as `chapterText(ch)`); selection points map to offsets via `pointOffset`. Quotes are painted with the CSS Custom Highlight API (`paintQuotes`, `polka-quote`); search uses `polka-find`.
- **Backup** — `exportBackup`/`importBackup`: a JSZip archive with `polka-backup.json` (settings, library state, all `meta` records) plus `books/<id>` raw files. Import adds missing books and merges existing ones (`mergeMeta`: newer position wins, bookmarks/quotes unioned by id).
- **Init** — opens DB, requests persistent storage, relayouts after fonts load, registers `sw.js` and toasts when an update is installed.

`sw.js`: network-first for navigations (updates cached `index.html`), cache-first for same-origin and `RUNTIME_HOSTS` requests; old caches are deleted on activate.
