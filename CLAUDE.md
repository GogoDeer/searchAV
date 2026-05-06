# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

This is a Tampermonkey/Greasemonkey userscript that scans web pages for Japanese AV (adult video) IDs ("番号"), marks them with visual cues, and provides popup menus with search links, cover images, metadata, and preview videos. The Chinese name translates to "Quick Search By AV ID."

## Build, test, and development

There is no build step, test suite, or dev server. This is a single userscript file loaded directly by a userscript manager (Tampermonkey/Greasemonkey) in the browser.

- **Entry point**: `根据番号快速搜索.user.js` — the full userscript (~3763 lines)
- **Installation URL**: https://greasyfork.org/zh-CN/scripts/423350
- **Local test flow**: Edit the `.user.js` file, then manually copy-paste into Tampermonkey's editor, or configure Tampermonkey to load from a local file. No automated reload.
- **No linting or formatting tools** are configured in the project.

## Architecture

The entire application is a single IIFE that runs at `document-idle` via `@run-at document-idle`. It is **not** modularized — all functions are defined in the IIFE scope.

### Initialization order
1. Check URL against exclusion patterns (skip pages with `detail|product` in URL unless on supported AV sites)
2. Initialize `window.qxin` global for communication with `@require`-ed scripts
3. Load settings from `GM_getValue("_setting")` and `GM_getValue("_setting2")`
4. Set up regex patterns for AV ID detection, exclusion lists, and magnet link detection
5. Apply site-specific regex overrides based on `window.location.hostname` (dmm.co.jp, attackers.net, mgstage.com, javlibrary.com)
6. Build `cid` lookup table for preview video URL construction
7. Call `findAVID()` to scan the DOM, then `addStyle()` to inject CSS

### Core subsystems

- **AV ID detection** (`findAVID` → `findAndReplaceDOMTextFun` / `findAndReplaceDOMTextFun_Wuma`): Uses the `findAndReplaceDOMText` library to walk text nodes and match regex patterns. Detected IDs are wrapped in `<sav-id>` custom elements. Different functions handle standard IDs (javbus-sourced) vs. amateur/uncensored IDs (javdb-sourced) vs. FC2 IDs.

- **Regex patterns**: `oRegExp` (standard IDs), `oRegExp_wuma` (amateur/uncensored IDs), `oRegExp_Magnet` (magnet links). Extensive exclusion regexes prevent false positives on tech terms, CPU models, camera models, etc.

- **Info fetching** (`getInfo`, `getInfo_wuma`, `getInfo_fc2`): Fetches AV metadata (title, actors, tags, cover image) from JavBus, JavDB, or FC2Hub via `GM_xmlhttpRequest`. Results are cached in `GM_setValue("avInfo2", ...)`.

- **Popup menu** (`createPattenr`): Builds a DOM popup with search buttons, cover image, actor links, tags, and optional preview video. Positioning logic in `settingPostion`.

- **Preview videos** (`getVideo`, `queryDMMVideoURL`, `getVideoURLFC2`): Fetches short preview clips primarily from DMM (for standard IDs) and FC2 (for FC2 IDs), with fallback to javspyl.tk.

- **Translation** (`googleTrans`, `baiduTrans`): Translates AV titles via Google Translate or Baidu Translate API (requires API key).

- **External integrations**: Jellyfin/Emby local media server lookup, qBittorrent remote download, sehuatang (色花堂) forum search.

### Key files

| File | Purpose |
|------|---------|
| `根据番号快速搜索.user.js` | Main userscript — all logic |
| `标签翻译对照列表.js` | Tag translation mapping (Japanese → Chinese) |
| `findAndReplaceDOMText 0.4.6.js` | Third-party DOM text find-and-replace library |
| `README.md` | Full Chinese documentation with settings reference |
| `更新日志.md` | Changelog (Chinese) |
| `待做事项.md` | TODO list (Chinese) |

### @require dependencies (loaded at runtime via Greasy Fork CDN)
- `findAndReplaceDOMText v0.4.6` — DOM text scanning
- `MD5 函数` — MD5 hash function
- `av番号特征(tag)对照表` — Additional tag translation data

### Greasemonkey API usage
`GM_getValue`, `GM_setValue`, `GM_deleteValue`, `GM_addStyle`, `GM_xmlhttpRequest`, `GM_setClipboard`, `GM_registerMenuCommand`, `GM_openInTab`

### Settings system
User configuration is stored via `GM_getValue("_setting")` as a JSON object. The `@grant GM_registerMenuCommand` adds a "自定义搜索" (Custom Search) menu command in Tampermonkey that opens a settings editor (`savBoxEdit`). Hidden internal state is stored in `_setting2`.

## Code conventions

- Variable names: Chinese-facing (pingyin-like) naming — `oRegExp`, `timerGetInfo`, `avInfo`, `javdbTime`
- DOM custom elements created by the script use the `<sav-*>` prefix (e.g., `<sav-id>`, `<sav-box>`, `<sav-video>`)
- All CSS classes use the `sav` prefix (e.g., `.savMenubtn`, `.savVideoClose`)
- Comments and documentation are in Chinese
- No transpilation, no module system — plain ES5/ES6 JS with some `async/await`
