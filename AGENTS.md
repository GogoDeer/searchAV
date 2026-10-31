# AGENTS.md

This file provides architectural context, development guidelines, and operating conventions for AI coding assistants and autonomous agents working in this repository.

## Project overview

This project is a Tampermonkey/Greasemonkey userscript that scans web pages for Japanese AV (adult video) IDs ("番号"), highlights them with visual cues, and provides rich interactive popup menus containing search links, cover images, metadata, and preview videos. The Chinese name translates to "根据番号快速搜索" (Quick Search By AV ID).

## Build, test, and development

There is no compilation step, build system, or test suite. This is a pure userscript executed by browser userscript managers (Tampermonkey, Violentmonkey, Greasemonkey).

- **Main script (production entry)**: `根据番号快速搜索.user.js` (~4280+ lines)
- **Local fast development loader**: `根据番号快速搜索.local-loader.user.js`
- **Installation URL**: https://greasyfork.org/zh-CN/scripts/423350
- **Local development workflow**:
  1. In Chrome / Edge extension settings, enable **"Allow access to file URLs"** for Tampermonkey.
  2. Install `根据番号快速搜索.local-loader.user.js` once in Tampermonkey. It dynamically references `file:///D:/Dev/projects/searchAV/根据番号快速搜索.user.js` via `@require`.
  3. Edit `根据番号快速搜索.user.js` locally; simply refresh any tested webpage to see changes take effect immediately without copy-pasting.

## Architecture

The main userscript is structured as an IIFE executed at `document-idle` via `@run-at document-idle`. Core legacy functions live within the root IIFE scope, while newer complex capabilities (such as `IconButtonUtils`) are encapsulated into isolated sub-modules to prevent scope pollution and minimize upstream merge conflicts.

### Initialization & Lifecycle
1. **Exclusion Check**: Match URL against userscript `@exclude` rules and path checks (LLM/AI services, developer tools, shopping sites).
2. **Global State**: Initialize `window.qxin` for communication with `@require`-ed scripts.
3. **Settings Loading**: Read user configuration from `GM_getValue("_setting")` and `GM_getValue("_setting2")`.
4. **Pattern Setup**: Compile regular expressions for standard IDs (`oRegExp`), uncensored/amateur IDs (`oRegExp_wuma`), magnet links (`oRegExp_Magnet`), and exclusion patterns.
5. **Site-specific Overrides**: Adjust regexes on specific domains (`dmm.co.jp`, `attackers.net`, `mgstage.com`, `javlibrary.com`).
6. **Preview Table**: Build `cid` lookup table for DMM preview video streaming.
7. **DOM Matching**: Call `findAVID()` to scan DOM nodes, then `addStyle()` to inject CSS into the page.
8. **SPA & MutationObserver Guard**:
   - Automatically polls URL changes (`popstate`, `pushState`, Next.js / React 18 / Vue routers) with debounce to re-scan dynamically loaded pages.
   - Monitors DOM additions with `MutationObserver`, guarded by `try...finally` and SVG class/tag type checks to avoid breaking on non-HTMLElement nodes.

### Core subsystems

- **AV ID Detection & Replacement** (`findAVID` → `findAndReplaceDOMTextFun` / `findAndReplaceDOMTextFun_Wuma`): Uses `findAndReplaceDOMText` to scan text nodes and wrap detected IDs in `<sav-id>` custom elements without breaking existing links or inline tags.
- **Button Group Container** (`displayMode`): Bundles the search icon (✈), Jellyfin/Emby server lookup button, and custom search icons into an inline-flex `.sav-btn-group` container behind the ID to ensure vertical alignment and tight spacing.
- **Resource Checking & Icon Buttons** (`IconButtonUtils`):
  - Injects action icons (🎬 Jable, 🎞️ 123AV, 🌱 Nyaa/Sukebei) based on `setting.list_all`.
  - **Strategy Pattern (`SITE_STRATEGIES`)**: Autonomous per-site validation rules, selector logic, classes, and tooltip copy.
  - **Quality Filtering Rules**: Nyaa torrent checks require `seeders >= 1` and `size >= 2GiB`.
  - **In-Memory Domain Circuit Breaker**: Trips on anti-bot challenge (CF 403/503), immediately short-circuiting subsequent requests to that domain on the current page into warning state to prevent 403 request flooding and IP bans.
  - **Per-Domain Concurrency Queue**: Total concurrency 4, per-domain concurrency 2; eliminates starvation caused by slow or hanging sites.
  - **Viewport Lazy Loading**: Uses `IntersectionObserver` (300px prefetch margin) to only trigger network validation when icons enter view.
  - **Cloudflare & Anti-Bot Defense**: Dynamically uses native `navigator.userAgent` to avoid clearance cookie mismatch; renders prominent warning styles (`sav-warn-*`) instead of misleading false-negatives (disabled/grayscale), guiding users to pass the challenge manually.
  - **Tiered Cache Isolation (`_external_link_cache_v2`)**: 24-hour TTL for positive confirmed resources, 3-hour TTL for negative results (404 / 0 seeders) to quickly pick up new releases, while strictly avoiding persisting transient challenge, rate-limited, or network error states.
- **Info Fetching** (`getInfo`, `getInfo_wuma`, `getInfo_fc2`): Fetches metadata (title, actors, tags, cover image) from JavBus, JavDB, or FC2Hub via `GM_xmlhttpRequest`. Results are cached in `GM_setValue("avInfo2", ...)`.
- **Popup Menu** (`createPattenr`): Generates a floating HUD with direct search buttons, cover image preview, actor links, tags, and translation tools.
- **Preview Videos** (`getVideo`, `queryDMMVideoURL`, `getVideoURLFC2`): Fetches and streams short preview clips from DMM or FC2 with fallback to third-party mirrors.
- **Translation** (`googleTrans`, `baiduTrans`): Translates Japanese titles to Chinese using Google Translate or Baidu Translate API.
- **External Integrations**: Jellyfin/Emby local media server lookup, remote qBittorrent magnet dispatch, Sehuatang (色花堂) forum search integration.

### Key files

| File | Purpose |
|------|---------|
| `根据番号快速搜索.user.js` | Main production userscript containing all logic (~4280+ lines) |
| `根据番号快速搜索.local-loader.user.js` | Fast development userscript that `@require`s the local file for instant hot-reload |
| `标签翻译对照列表.js` | Tag dictionary (Japanese → Simplified Chinese) |
| `findAndReplaceDOMText 0.4.6.js` | Third-party DOM text scanning utility |
| `README.md` | Comprehensive Chinese documentation and user configuration reference |
| `更新日志.md` | Detailed version history and changelog (Chinese) |
| `待做事项.md` | Pending feature backlog and known issues (Chinese) |
| `AGENTS.md` | Autonomous agent operating guide and architectural reference |

### @require runtime dependencies (Greasy Fork CDN)
- `findAndReplaceDOMText v0.4.6` — DOM text scanning without tag corruption
- `MD5 函数` — MD5 hash function for cache key hashing
- `av番号特征(tag)对照表` — Additional tag mapping dictionary

### Greasemonkey / Tampermonkey API usage
- **Network**: `GM_xmlhttpRequest` (cross-origin HTTP requests bypassing CORS)
- **Storage**:
  - `_setting`: User configuration JSON (custom search lists, UI toggles, Jellyfin/Emby URLs, qBittorrent host)
  - `_setting2`: Hidden internal runtime state (e.g., Sehuatang forum formhash)
  - `avInfo2`: Metadata and cover cache
  - `_external_link_cache_v2`: Online playback / torrent availability cache (24h TTL)
- **DOM & UI**: `GM_addStyle`, `GM_openInTab`, `GM_setClipboard`, `GM_registerMenuCommand`

## Code conventions & agent guidelines

- **Upstream Merge Compatibility**: When implementing new features or bug fixes, always encapsulate logic into self-contained helper modules or functions (such as `IconButtonUtils`) within `根据番号快速搜索.user.js`. Keep call sites in legacy routines (`displayMode`, `findAVID`) minimal to avoid merge conflicts when rebasing onto `upstream/main`.
- **External Resource Checking & Tri-state Modeling**: When probing external websites (e.g. video streams, torrents):
  - **Tri-state Classification**: Never model results as a crude binary boolean. Differentiate between *Positive*, *Negative (404/empty)*, and *Transient / Intercepted (Cloudflare 403/503 challenge, HTTP 429 rate limit, network timeout)*.
  - **UI State Invariant**: Transient or challenge states MUST render warning cues (`sav-warn-*`) and explanatory guidance, NEVER misleading false-negative disabled/grayscale styles.
  - **Cache Purity**: Only cryptographically or semantically trusted results can be cached with long TTL. Never persist transient errors or anti-bot challenge states into persistent storage.
- **Scope Evaluation Policy**: Distinguish between uncontrolled scope creep and high-value developer/user convenience extensions (such as cache-clearing buttons in settings or synchronizing local-loader configurations). Valuable ergonomics that aid operability should be preserved and welcomed rather than mechanically reverted.
- **CSS Scoping**: All custom CSS classes injected into host pages MUST carry the `sav-` prefix (e.g., `.sav-btn-group`, `.sav-icon-btn`, `.sav-has-video`) or be strictly scoped within `.sav-btn-group` to prevent style leaks into host websites.
- **Language**: User communications, changelog entries (`更新日志.md`), and in-script comments must be in Simplified Chinese (简体中文).
- **Naming Conventions**: Match existing codebase conventions — pinyin-style or descriptive camelCase (e.g., `oRegExp`, `displayMode`, `avInfo`, `savBoxEdit`).
- **No Build Tools**: Maintain raw ES5/ES6+ vanilla JavaScript with standard `async/await`. Do not introduce bundlers, transpilers, or external npm dependencies.
