# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Wallit Creations marketing site — a single self-contained static page. There is **no build step, no package manager, and no test suite**. The entire site ships as one file: `index.html` (~2.7MB). The only other files are `vercel.json`, `README.md`, and `.gitignore`.

## Serving / deploying

- **Local preview:** open `index.html` directly, or serve the directory (`python3 -m http.server`). Note that `DecompressionStream` (used to gunzip assets) requires a secure context — `file://` and `localhost` both qualify, so both work.
- **Deploy:** Vercel serves `index.html` as a static file. Framework preset is **Other** (no build command, no output directory). `vercel.json` only sets `cleanUrls` + `trailingSlash`.

## Architecture: the bundle format

`index.html` is a **generated artifact**, not hand-authored source. It embeds the real app plus every image/font/script as base64 (often gzip-compressed) and unpacks them client-side. The `<head>`/bootstrap wrapper is human-readable; the three payload `<script>` blocks are giant minified base64 and must not be edited by hand.

The bootstrap script (top of `<body>`) runs on `DOMContentLoaded` and:
1. Reads three payload script tags by type:
   - `__bundler/manifest` — `{ uuid: { data (base64), mime, compressed } }` map of assets.
   - `__bundler/template` — the real page's full HTML, with asset UUIDs used as placeholder strings.
   - `__bundler/ext_resources` — maps external resource IDs → asset UUIDs (exposed to the app as `window.__resources`).
2. Decodes each asset (gunzipping via `DecompressionStream` when `compressed`), creates a blob URL per UUID.
3. String-replaces every UUID in the template with its blob URL, strips `integrity`/`crossorigin` attrs (blob URLs get a null origin and would fail SRI), injects `window.__resources`, then re-parses the template and swaps in its `<documentElement>`.
4. Re-creates `<script>` tags via `createElement` (DOMParser scripts are inert) and awaits `src` loads in order so the inner app's dependencies (React → ReactDOM → Babel → `text/babel`) execute correctly.

The inner app inside `__bundler/template` is **React transpiled in-browser by Babel** (`text/babel` scripts). Errors during unpacking/render surface in a fixed `#__bundler_err` overlay and the console (`[bundler]` / `[bundle]` prefixes).

## Editing the site

Because `index.html` is bundler output, meaningful content/design changes are not made by editing the base64 payloads. Two practical paths:

- **Small, surgical tweaks** (e.g. the responsive-CSS commit in history): edit the human-readable wrapper, or decode → modify → re-encode a specific asset. Verify by loading the page and watching for the `#__bundler_err` overlay.
- **Substantive changes:** regenerate `index.html` from the original source with the same external bundler tool that produced it (not present in this repo). If asked to make large edits, confirm with the user where the un-bundled source lives rather than trying to rewrite the base64 in place.
