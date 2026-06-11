# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the official configuration repository for [Venera](https://github.com/venera-app), a comic/manga reader app. Each `.js` file (except `_template_.js` and `_venera_.js`) is a **comic source** — a configuration that teaches the Venera app how to scrape, search, and display comics from a specific website.

These JS files run inside Venera's **custom JavaScript engine** (not Node.js, not a browser). All interaction with the outside world goes through a native bridge via the global `sendMessage()` function.

## Repository structure

```
_template_.js       # Template for creating new comic sources (with inline documentation)
_venera_.js         # Type definitions / code completion helper for IDEs (the "standard library")
index.json          # Registry of all available sources (name, fileName, key, version)
*.js                # Individual comic source configs (e.g., nhentai.js, jm.js, manga_dex.js)
.github/workflows/  # CI: auto-purges jsDelivr CDN when .js/.json files change on main
```

## Creating or editing a comic source

1. Copy `_template_.js` and rename it.
2. The class must extend `ComicSource` and export nothing — the engine finds the class by scanning for `extends ComicSource`.
3. Every source has the required fields: `name`, `key` (unique), `version` (semver), `minAppVersion`, `url` (CDN URL for auto-update, pattern: `https://cdn.jsdelivr.net/gh/venera-app/venera-configs@main/<file>.js`).
4. After adding a new source, register it in `index.json`.

### Source lifecycle methods (all optional except where noted)

| Section | Methods | Purpose |
|---|---|---|
| `explore[]` | `load(page)` / `loadNext(next)` | Browse/discovery pages. Supports `multiPartPage`, `multiPageComicList`, `mixed` types |
| `category` | `parts[]` + `categoryComics.load()` + `ranking.load()` | Category browsing and ranking pages |
| `search` | `load(keyword, options, page)` | Full-text search |
| `comic` | `loadInfo(id)` **(required)**, `loadEp(comicId, epId)` **(required)**, `loadThumbnails`, `loadComments`, `sendComment`, `starRating`, `likeComic`, `onClickTag`, `link` | Single comic detail, chapter images, comments, ratings |
| `favorites` | `addOrDelFavorite`, `loadFolders`, `loadComics`, `addFolder`, `deleteFolder` | Favorites/follows sync (multi-folder optional) |
| `account` | `login(account, pwd)`, `loginWithWebview`, `loginWithCookies`, `logout` | Authentication |
| `settings` | User-configurable settings (input, select, switch, callback) | Per-source configuration UI |

## Core APIs (from `_venera_.js`)

All APIs live in the global scope of the engine. These are the main ones:

- **`Network`** — HTTP client. `Network.get(url, headers)`, `Network.post(url, headers, data)`, etc. Returns `{status, headers, body}` where `body` is a string. Also `Network.fetchBytes()` for binary responses. Cookie management: `Network.setCookies/getCookies/deleteCookies(url)`.
- **`HtmlDocument`** — HTML parser. `new HtmlDocument(htmlString)` then `.querySelector(query)`, `.querySelectorAll(query)`, `.getElementById(id)`. Returns `HtmlElement` objects with `.text`, `.attributes`, `.children`, `.innerHTML`, `.querySelector()`, etc. Call `.dispose()` when done.
- **`Convert`** — Encoding/crypto. `encodeUtf8/decodeUtf8`, `encodeGbk/decodeGbk`, `encodeBase64/decodeBase64`, `md5`, `sha1`, `sha256`, `sha512`, `hmac/hmacString`, `decryptAesEcb/decryptAesCbc/decryptAesCfb/decryptAesOfb`, `decryptRsa`, `hexEncode`.
- **`Comic(props)`**, **`ComicDetails(props)`**, **`Comment(props)`** — Data constructors for returning structured data to the app.
- **`ImageLoadingConfig(props)`** — Config for per-image request customization (custom headers, URL rewriting, post-processing).
- **`Image`** — Image manipulation (copyRange, copyAndRotate90, fillImageAt). Only usable in `modifyImage` scripts.
- **`PageJumpTarget`** — For inter-page navigation (e.g., tag click → search page).
- **`UI`** — `showMessage`, `showDialog`, `showInputDialog`, `showSelectDialog`, `showLoading/cancelLoading`, `launchUrl`.
- **`APP`** — `APP.version`, `APP.locale`, `APP.platform`.
- **`console`** — `console.log/warn/error` (works but output goes to app logs, not a terminal).
- **`setTimeout(callback, delay)`**, **`setInterval(callback, delay)`** — Timers.
- **`randomInt(min, max)`**, **`randomDouble(min, max)`**, **`createUuid()`** — Random/ID generation.
- **`compute(funcString, ...args)`** — Offload computation to a background isolate.
- **`setClipboard(text)`**, **`getClipboard()`** — Clipboard access.
- **`fetch(url, options)`** — Browser-like fetch API (since app 1.2.0).

### ComicSource base class methods

- **`this.loadSetting(key)`** — Read a user-configured setting value.
- **`this.loadData(key)`**, **`this.saveData(key, data)`**, **`this.deleteData(key)`** — Persistent key-value storage scoped to this source.
- **`this.isLogged`** — Boolean, whether the user is currently authenticated.
- **`this.translate(key)`** — i18n via the `translation` dictionary.

## Patterns and conventions

- **Update URL format**: `https://cdn.jsdelivr.net/gh/venera-app/venera-configs@main/<filename>.js`
- **Error signaling**: Throw specific strings for known conditions (e.g., `throw 'Login expired'` triggers auto-re-login; `throw 'Invalid status code: ${res.status}'` for network errors).
- **Settings for changeable endpoints**: Use `settings` with domain/API URL fields instead of hardcoding — sites change domains frequently.
- **Japanese/Chinese sites**: Many sources use `Convert.decodeGbk()` for GBK-encoded responses or `Convert.decodeUtf8()` for UTF-8.
- **Image decryption**: Some sources (e.g., jm.js, hitomi.js) require client-side image decryption via `Convert.decryptAesEcb` or similar.
- **CDN purge**: The GitHub Action on `main` branch push auto-purges changed `.js`/`.json` files from jsDelivr cache so users get updates quickly.
- **Code completion**: The `/** @type {import('./_venera_.js')} */` JSDoc comment at the top of each config file enables IDE autocompletion for the Venera APIs.
