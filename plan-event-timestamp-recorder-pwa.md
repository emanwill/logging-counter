# Plan: Event Timestamp Recorder PWA

## Context

We're building a personal-use PWA (per [project-brief.md](/Users/eric/dev/personal/logging-counter/project-brief.md)) that lets the user tap one of up to 8 configurable buttons on an Android phone to log timestamps for observed events, and export those timestamps as CSV. The goal is a self-contained, framework-free, installable home-screen app that works offline.

Through Q&A we settled the following decisions that go beyond the brief:

- **Log persists to localStorage** (not in-memory only) — survives reloads/restarts.
- **Buttons show a live press count** (the "counter" in the repo name) where count = number of entries currently in the persisted log for that event.
- **Navigation** = small header icons on the main screen (gear → Settings, list → Logs) with back arrows on the sub-screens.
- **Color picker** = a fixed palette of ~8 accessible swatches.
- **PWA scope** = `index.html` + `manifest.json` + `sw.js`, icons as inline data URIs in the manifest (no separate icon files).
- **First-launch empty state** = friendly CTA pointing at the gear icon.
- **Undo** = a per-row × delete button on the Logs screen (with confirm).
- **Button layout** = adaptive grid; buttons always stretch to fill the available vertical space.
- **Log display** = local time, human-readable; CSV stays ISO 8601 UTC.
- **CSV** = header row `timestamp,event`, RFC-4180 quoted.
- **Historical entries are immutable**: renaming/disabling a button does not retroactively change or delete old log entries.

## Files to create

All paths are relative to `/Users/eric/dev/personal/logging-counter/`:

- `index.html` — the whole app: HTML, CSS, JS inline.
- `manifest.json` — PWA manifest with inline SVG data-URI icons.
- `sw.js` — minimal cache-first service worker.

## Architecture (in `index.html`)

### State model (localStorage-backed)

Two keys:

- `lc.config` — array of exactly 8 slot objects:
  ```js
  { enabled: boolean, label: string, eventName: string, color: string }
  ```
  Default: 8 disabled slots with sensible placeholder labels and rotating palette colors.
- `lc.log` — array of entries `{ ts: ISOString, event: string }`, appended in chronological order.

A small `store` module wraps `localStorage` reads/writes with JSON parsing and a single in-memory cache to avoid re-parsing on every render.

### Screens

Three `<section>` elements (`#screen-main`, `#screen-logs`, `#screen-settings`). A `data-screen` attribute on `<body>` controls which is visible via CSS (`section { display: none } body[data-screen="main"] #screen-main { display: flex }`). No hash routing — pure DOM toggling.

### Main screen

- Header bar: title, list icon (→ Logs), gear icon (→ Settings).
- Grid container with CSS variables for `--cols` and `--rows`. JS computes both from the enabled-button count:
  - 1 → 1×1
  - 2 → 2×1
  - 3–4 → 2×2
  - 5–6 → 2×3
  - 7–8 → 2×4
- Each button: large centered label, count badge (top-right corner), background tinted to the configured color, foreground auto-picked black/white based on luminance.
- Press handler: append entry to `lc.log`, increment count display in place (no full re-render), brief CSS scale animation, and `navigator.vibrate?.(15)` for haptic feedback if available.
- Empty state (no enabled buttons): centered text "No buttons configured — tap ⚙ to add some."

### Logs screen

- Header: back arrow, title "Logs".
- Toolbar: **Export CSV** button, **Clear** button (calls `confirm()` first).
- Scrollable list, newest first. Each row: local-time string (e.g. `2026-05-23 14:32:01`), event name, small × delete button (also `confirm()` before deleting).
- Empty state if zero entries.

### Settings screen

- Header: back arrow, title "Settings".
- 8 slot cards. Each card:
  - Enable toggle (checkbox styled as a switch).
  - Label input (shown on the button).
  - Event name input (recorded in the log).
  - Color swatch row — 8 swatches; tapping one sets it and shows a check on the selection.
- All edits save to `lc.config` on `change`/`input` (no explicit Save button). When user returns to main, layout re-renders to reflect the new config.

### CSV export

- Filename: `events-${YYYY-MM-DD}.csv` from `new Date()`.
- Content: `timestamp,event\n` header, then one row per log entry. Values containing `,`, `"`, or newlines are wrapped in `"` with internal `"` doubled (RFC 4180).
- Download via `Blob` + temporary `<a download>` click.

### Color palette

8 visually distinct, accessible swatches:
`#e53935` (red), `#fb8c00` (orange), `#fdd835` (yellow), `#43a047` (green), `#00897b` (teal), `#1e88e5` (blue), `#8e24aa` (purple), `#546e7a` (slate).

### Styling

- Mobile-first, viewport meta `width=device-width, initial-scale=1, viewport-fit=cover`.
- CSS uses `100dvh` for full-height layouts and respects `env(safe-area-inset-*)` for notched devices.
- System font stack, no web fonts.
- Dark-friendly neutral background; button colors carry the visual identity.

## `manifest.json`

```json
{
  "name": "Event Recorder",
  "short_name": "Events",
  "start_url": "./index.html",
  "scope": "./",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#111",
  "theme_color": "#111",
  "icons": [
    { "src": "data:image/svg+xml;utf8,<svg ...192...>", "sizes": "192x192", "type": "image/svg+xml", "purpose": "any maskable" },
    { "src": "data:image/svg+xml;utf8,<svg ...512...>", "sizes": "512x512", "type": "image/svg+xml", "purpose": "any maskable" }
  ]
}
```

Icon SVGs will be simple: a colored rounded square with a stylized tally-mark glyph.

## `sw.js`

Minimal cache-first shell:

- `install`: precache `['./', './index.html', './manifest.json']`.
- `activate`: clean up old cache versions.
- `fetch`: respond from cache, fall back to network, then update cache on network success (stale-while-revalidate for same-origin GETs).
- Cache name embeds a version constant so a bumped version invalidates old caches.

Registration in `index.html`:
```js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => navigator.serviceWorker.register('./sw.js'));
}
```

## Verification

Run locally with a static server (file:// won't register the SW):

```bash
cd /Users/eric/dev/personal/logging-counter
python3 -m http.server 8000
```

Then in Chrome/Chrome-Android (or DevTools device emulation):

1. **Settings flow** — enable 3 slots, set labels/event names/colors, return to main. Verify the grid shows 3 buttons with the chosen colors and labels.
2. **Logging** — tap each button several times. Counts increment in place; haptic vibration fires on a real device.
3. **Persistence** — full page reload. Config, log, and counts all survive.
4. **Logs screen** — entries appear newest-first with local-time format. Tap × on a row → confirm → row disappears and the corresponding button count decrements by 1.
5. **Export CSV** — file downloads as `events-YYYY-MM-DD.csv`. Open in a spreadsheet: 2 columns, header row, ISO 8601 UTC timestamps, event names preserved (including any with commas/quotes).
6. **Clear** — confirm prompt appears; accepting wipes the log and resets all button counts to 0.
7. **Rename/disable immutability** — log a press as "Coffee", rename slot to "Tea", log another press. CSV contains both "Coffee" and "Tea" entries. Disable the slot — main screen hides it, but the historical entries remain in the Logs screen and CSV.
8. **Empty state** — disable every slot; main screen shows the CTA.
9. **PWA install** — Chrome on Android shows "Add to Home Screen". After install, launch from home screen → opens standalone (no browser chrome), works with airplane mode on (service worker serves cached shell).
10. **Lighthouse PWA audit** — should pass installability checks.
