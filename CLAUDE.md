# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

«Полка» — a client-only FB2 / FB2.ZIP / EPUB reader PWA, deployed as static files to GitHub Pages (branch `main`, root folder; `.nojekyll` disables Jekyll). Books never leave the user's browser. UI text, comments, and README are in Russian — keep new user-facing strings and code comments in Russian to match.

There is no build step, package manager, linter, or test suite. To run locally, serve the folder over HTTP (the service worker only registers in a secure context, which includes `localhost`):

```sh
python3 -m http.server 8000   # then open http://localhost:8000/
```

The planned work, in order, is in `ROADMAP.md`; update it when a stage ships or the plan changes.

## Release rule

Whenever `index.html` (or any cached asset) changes for a release, bump `VERSION` in `sw.js` (e.g. `polka-0.6.0` → `polka-0.7.0`). The service worker serves own files cache-first, so without a version bump users keep the old cached copy. The README's feature list is labeled with the version too.

## Architecture

Everything lives in `index.html` (~2750 lines): inline `<style>`, markup, and one `'use strict'` inline `<script>`. The only external dependencies are JSZip 3.10.1 from cdnjs and Google Fonts (Literata, PT Serif, Golos Text); `sw.js` caches those hosts at runtime for offline use (`RUNTIME_HOSTS`). If you add another CDN host, add it there too. The dictionary calls the ru.wiktionary.org API (CORS via `origin=*`); it is deliberately not in `RUNTIME_HOSTS`, so lookups are online-only.

The script is split by `/* ===== Section ===== */` banners:

- **Storage** — `DB` wraps IndexedDB `polka-reader` (v3) with four stores keyed by `id`: `meta` (title, authors, `series: {name, num}`, `tags`, cover thumbnail, `pos: {ch, r}`, `progress`, `done`, `readMs`, `bookmarks`, `quotes`, etc.), `files` (the raw original `ArrayBuffer`), `stats` (one record per local day: `{id: 'YYYY-MM-DD', ms, chars, books: {bookId: ms}}`), and `words` (saved vocabulary; shape documented at the top of the Vocabulary section). Falls back to in-memory Maps if IndexedDB is unavailable (`#memWarn` is shown). Changing store layout requires bumping the DB version and handling `onupgradeneeded`.
- **Settings** — `S` object merged from `DEFAULTS` and `localStorage['polka-settings']`; `applySettings()` pushes them into CSS. Text settings (`TEXT_KEYS`) can be overridden per book in `meta.text`; always read them through `eff(k)`, and write changes to `bookText()` when it exists. `applyTheme()` (re-run every minute) resolves the `schedule` theme and the warm filter (`#warm` overlay) from `darkFrom`/`darkTo`. A custom font lives in the `files` store under `CUSTOM_FONT_ID` and is registered as the `PolkaCustom` FontFace (`FONTS.custom`).
- **Parsing** — `parseBook(buf)` detects format: FB2 (incl. windows-1251 via `decodeXml`, zipped FB2 via JSZip) → `parseFb2Text`, EPUB → `parseEpub`. Both produce a common book shape `{ title, authors, lang, series, cover, chapters: [{title, html, isNotes}], toc }` with chapters as sanitized HTML strings. EPUB sanitizing uses the `BAD` tag blocklist and `KEEP` attribute allowlist; links go through `safeUrl`. Element lookup is namespace-agnostic via `lnOf` (strips `fb-` prefixes).
- **Post-processing** — `finalize(book)` splits oversized chapters (>60k chars into ~30k parts via `splitInto`), builds `idMap` (element id → chapter index, used for footnotes/TOC/internal links), and computes `len`/`cum`/`total` for whole-book progress. Books are re-parsed from the stored raw file each time they are opened.
- **Library** — shelf rendering with search/sort/filter/tag state `L` (persisted in `localStorage['polka-library']`), `addFiles` (dedupe by name+size, drag-and-drop, cover `thumb`; files named `polka-backup*.zip` go to `importBackup`), per-book sheet (`openBookSheet`). A book is done if `meta.done === true` or progress ≥ 99.5% unless `done === false` (`isDone`).
- **Reader** — runtime state in `R`. Only one chapter is in the DOM at a time (`renderChapter` into `#flow`). Paged mode uses CSS multi-column layout with page width `R.W` + gap `R.G`, paging by `translateX` (in spread mode, `spreadOn()`, one step shows two columns); scroll mode scrolls `#viewport`. `position(where)` accepts `'end'`, `{id}`, `{off, len}` (character offset into chapter text, used by search/bookmarks via `buildNodes`/`locate`/`offsetRange`), or `{ratio}`. `R.tok` guards against stale async renders. Position is saved debounced via `savePos()` → `writePos()`. Footnotes open in a sheet (`showNote`); bookmarks, quotes, search, TOC, and settings are bottom sheets (`openSheet`/`closeSheets`). Bookmarks and quotes store `{ch, off}` — `off` is a character offset into the chapter's text (same as `chapterText(ch)`); selection points map to offsets via `pointOffset`. Quotes are painted with the CSS Custom Highlight API (`paintQuotes`, `polka-quote`); search uses `polka-find`.
  - Autoscroll (`AS`): speed in lines per minute; scroll mode advances `viewport.scrollTop` from `requestAnimationFrame`, paged mode calls `next()` on a timer; opening a sheet, tapping the page or hiding the tab pauses it, and a screen wake lock is held while it runs.
  - Reading stats: `savePos()` calls `statTick()`, which adds the time since the previous reading action (capped at `IDLE_CAP`) and forward movement in characters (ignored above `JUMP`; `renderChapter` with an object target resets the baseline). `ST.speed` (chars/ms over 30 days) drives the "time left" labels.
  - Dictionary: `showDict` → `lookupWord` (exact title, lowercase, English `stems`, then a prefix-matched search hit) → `parseWikt` extracts the «Значение» lists per language (h1) and part of speech (h2); `lookupArticle` follows word-form pages («форма … слова X») to the lemma via `formOf`; `pickLang` prefers the book's language. Saved words store plain-text definitions so cards and the dictionary fallback work offline.
- **Vocabulary** — `WL` caches the `words` store (`loadWords`). Words with a position are underlined in the text (`polka-word` highlight; `wordAt` opens the dictionary on tap). `renderWords` lists them, `startReview`/`grade` run spaced repetition (interval × ease, «не помню» re-queues the card), `exportWords` writes an Anki CSV.
- **Statistics** — `renderStats` builds the library stats sheet (tiles, 14-day bar chart, streaks).
- **Backup** — `exportBackup`/`importBackup`: a JSZip archive with `polka-backup.json` (settings, library state, all `meta` records) plus `books/<id>` raw files. Import adds missing books and merges existing ones (`mergeMeta`: newer position wins, bookmarks/quotes unioned by id); per-day stats take the max of each field so re-importing is idempotent; words are unioned by id, later `edited` wins; the custom font travels as `fonts/custom`.
- **Init** — opens DB, requests persistent storage, relayouts after fonts load, registers `sw.js` and toasts when an update is installed.

`sw.js`: network-first for navigations (updates cached `index.html`), cache-first for same-origin and `RUNTIME_HOSTS` requests; old caches are deleted on activate.
