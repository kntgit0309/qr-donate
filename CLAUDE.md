# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Static, single-purpose Vietnamese-language donation page. Two HTML files, one fallback config, one QR image — no build step, no package manager, no server. Open `index.html` directly in a browser, or serve the directory with anything (`python3 -m http.server`, `npx serve`, GitHub Pages, etc.).

- `index.html` — public landing page. Renders QR + bank info, then reveals a download link after the user clicks "Tôi đã chuyển khoản".
- `admin.html` — browser-only editor for the live config. Talks directly to JSONBin from the client.
- `config.json` — fallback config used only if the JSONBin fetch fails.
- `qr.png` — committed QR image (the page actually loads `qrUrl` from config, which currently points at `img.vietqr.io`; `qr.png` is not referenced by the HTML).

## Architecture: how config flows

The runtime config (download URL, QR image URL, bank name/account/owner) is **not** read from `config.json` in production. The flow is:

1. `index.html` (`loadConfig()`, ~line 140) fetches `https://api.jsonbin.io/v3/b/<JSONBIN_ID>/latest` with `cache: "no-store"` and a `_=${Date.now()}` query param to defeat any intermediate cache.
2. If that request fails, it falls back to `config.json` from the same origin.
3. `admin.html` writes to the same JSONBin record via `PUT /v3/b/<JSONBIN_ID>`, authenticated with an `X-Master-Key` the operator pastes in and which is persisted in `localStorage` under `qr_donate_jsonbin_masterkey`. The master key never leaves the browser.

Consequence: edits made through `admin.html` go live immediately for every visitor, with no rebuild and no commit. `config.json` only matters as a fallback and as the seed used by `admin.html`'s "Tạo bin mới" flow.

## The JSONBIN_ID is duplicated — keep them in sync

`JSONBIN_ID` is hardcoded as a `const` at the top of the `<script>` block in **both** files and must match:

- `index.html:137`
- `admin.html:179`

If you change the bin (e.g., point to a new JSONBin record), update both. The placeholder string `REPLACE_WITH_YOUR_BIN_ID` is a sentinel checked by `admin.html`'s `route()` to show the "create a new bin" UI; preserve that string exactly if you ever reset the project.

The expected JSON shape stored in JSONBin (and mirrored in `config.json`):

```json
{ "downloadUrl": "", "qrUrl": "", "bankName": "", "bankAccount": "", "bankOwner": "" }
```

`admin.html`'s `saveConfig()` writes exactly these five string fields — adding a new field requires editing both the form in `admin.html` and the DOM-update code in `index.html`'s `loadConfig()`.

## Conventions

- All user-facing copy is Vietnamese. Keep it that way unless explicitly asked.
- Vanilla JS only, no framework, no bundler, no transpilation. Don't introduce a build step for a change that doesn't need one.
- Styles are inline `<style>` blocks per page using a shared dark-theme CSS-variable palette (`--bg`, `--card`, `--accent`, etc.). Reuse those variables rather than hardcoding colors.
- The cache-busting on the JSONBin fetch (`?_=${Date.now()}` plus `cache: "no-store"`) is load-bearing — past commits specifically added it (`26462cd Bypass browser cache on JSONBin fetch`). Don't remove it when refactoring.

## Development workflow

- No tests, no linter, no CI configured. "Run" the project by opening the HTML in a browser; for fetches to behave normally serve over HTTP (`python3 -m http.server 8000`).
- Branch policy for this environment: develop on `claude/add-claude-documentation-dvHUD` and push there. Never push elsewhere without explicit permission.
