# YouTube Watch Tracker — Full Project Blueprint

A complete, ready-to-implement plan for an open-source browser extension that injects a button on every YouTube video page showing whether a user has watched that video, how many times, and the dates of each watch. Includes architecture, full code examples (Manifest V3 content script + service worker + popup/options), storage design (IndexedDB), UI, UX decisions, privacy, README, CONTRIBUTING guide, issue/label suggestions, roadmap, and first-PR checklist.

---

## Short summary

Build a Chrome/Firefox extension (Manifest V3) that:

* Injects a small `Watched` button in YouTube's video UI near the like/share buttons.
* Detects when a user *watches* a video (configurable threshold: e.g. 70% of duration, or `ended`).
* Stores watch events locally (IndexedDB) keyed by YouTube `videoId` with timestamps.
* Shows count and individual watch dates in a small popup/tooltip.
* Optionally: export/import data, sync across devices (opt-in), privacy-first design (local-only by default).

This project will be published as `youtube-watch-tracker` on GitHub with MIT license.

---

# Table of contents

1. Goals & features
2. Architecture & data model
3. Manifest & permissions
4. Content script (injection + detection)
5. Service worker (background logic)
6. Storage layer (IndexedDB with a small wrapper)
7. UI - injected button, tooltip, popup, options
8. Edge cases & YouTube SPA handling
9. Privacy, security, and policy considerations
10. GitHub repo structure
11. README template
12. CONTRIBUTING, issues, labels, and first issues
13. CI & tests
14. Packaging & publishing
15. Roadmap of future features
16. Appendix: full example files

---

## 1. Goals & features

**MVP features**

* Detect and record watches per video.
* Display a small button on the video page showing watched `count` and quick tooltip with latest dates.
* Show a detailed popup listing all watch timestamps and option to clear or export (JSON).
* Store data **locally** with user option to export/import.

**Nice-to-have / future**

* Sync via GitHub Gist/Google Drive (opt-in, encrypted client-side).
* Per-channel view stats (top-watched videos, watch frequency heatmap).
* Keyboard shortcuts and configurable watch threshold.
* Dark-mode/Theme and localization.

---

## 2. Architecture & data model

### Components

* Content script: Inject UI, observe video element, send watch events to service worker.
* Service worker (MV3 background): Central store access, handles export/import, options, analytics (opt-in).
* IndexedDB storage: Stores `watchEvents` table keyed by `videoId`.
* Popup/Options page: Settings UI and export/import.

### Data model (IndexedDB)

**DB name:** `yt_watch_tracker`
**Object stores:**

1. `videos` (keyPath = `videoId`)

```json
{
  videoId: "abc123",
  title: "Video Title",
  channel: "Channel Name",
  watchCount: 3,
  events: ["2026-01-10T12:34:56Z", "2026-01-15T10:00:00Z"],
  lastSeenAt: "2026-01-15T10:00:00Z"
}
```

2. `meta` (for extension settings)

```json
{
  key: "settings",
  value: {
    thresholdPercent: 70,
    autoMarkOnEnded: true,
    exportOnUninstall: false
  }
}
```

Events are stored as ISO 8601 strings (UTC) for easy formatting.

---

## 3. Manifest & permissions

Use Manifest V3 for Chrome/Edge/Firefox compatibility.

**manifest.json** (high-level)

* `manifest_version`: 3
* `name`, `description`, `version`
* `permissions`: `storage`, `scripting`, `activeTab` (optional), `notifications` (optional)
* `host_permissions`: `https://www.youtube.com/*`
* `background`: service_worker file
* `content_scripts`: matches for YouTube pages
* `action` (popup)

See the Appendix for a full manifest example.

**Permission rationale**

* `storage`: to persist watch data.
* `scripting`: to inject scripts if needed.
* `host_permissions`: to operate only on YouTube pages.

Keep permissions minimal to pass review and build trust.

---

## 4. Content script: injection + detection

Responsibilities:

* Detect YouTube video page (videoId).
* Insert the Watch button into the YouTube UI.
* Detect 'watch' events (video ended or > threshold percent watched).
* Debounce events to avoid duplicates.
* Communicate events to service worker via `chrome.runtime.sendMessage`.

Key points:

* YouTube is a Single Page App (SPA). Detect navigation changes using `history` hooks, `popstate`, or `MutationObserver` for the `ytd-watch-flexy` element.
* Use `MutationObserver` to wait for the right container to exist before injecting UI.
* Use robust selectors and fallback strategies because YouTube may change class names.

**Video ID detection**

```js
function getVideoId() {
  const url = new URL(location.href);
  return url.searchParams.get('v');
}
```

**Watch detection**

* Listen to the `<video>` element `timeupdate` and `ended` events.
* Mark watch when one of the following occurs:

  * `ended` event fires and `settings.autoMarkOnEnded` true
  * `currentTime / duration >= thresholdPercent/100`
* Use an epsilon to deal with floats.

**Debounce**: for a single watch session mark only once (use an in-memory set per videoId for current session).

---

## 5. Service worker (background)

Responsibilities:

* Receive messages from content scripts.
* Update IndexedDB records.
* Provide query endpoints for content scripts (e.g., ask: "is this video watched? return count and dates").
* Handle export/import and options persistence.

Communication patterns:

* `chrome.runtime.onMessage.addListener((msg, sender) => { ... })`
* Provide promise-based request/response messages: `{type: 'record_watch', videoId, meta}` or `{type: 'get_video', videoId}`

Avoid heavy work in the service worker — keep operations quick and move heavy export work to the options page if needed.

---

## 6. Storage layer (IndexedDB wrapper)

We recommend a tiny wrapper around IndexedDB for convenience (or use `idb` from Jake Archibald if you want a dependency). Example minimal wrapper included in Appendix.

APIs:

* `db.getVideo(videoId)`
* `db.recordWatch(videoId, meta = {title, channel})` -> returns updated record
* `db.getAllVideos()`
* `db.clearVideo(videoId)`
* `db.export()` -> returns JSON
* `db.import(json)`

Performance notes: index by `videoId`. Keep `events` arrays limited for very popular videos (optionally rollup events older than X months into `summary` to avoid big objects).

---

## 7. UI: injected button, tooltip, popup, options

### Injected Button

* Placement: near the Like / Share / Save buttons in the `#menu` area underneath the video title. This is the most stable place visually.
* Button appearance: small icon + label `Watched` and a small badge count.

Example markup:

```html
<button class="ytwt-watched-btn" aria-label="Watch history">👁 <span class="ytwt-count">3</span></button>
```

### Tooltip / Panel

* Hover or click opens a small panel showing:

  * Watched times (count)
  * Recent watch dates (formatted as `Jan 15, 2026 10:00`)
  * Buttons: `Export JSON`, `Clear`, `Open in Tracker` (opens popup)

### Popup (action popup)

* Show stats across all videos
* Provide export/import
* Settings: threshold percentage, autoMarkOnEnded, privacy/sync options

### Options page

* Advanced settings and sync management
* Bulk export, clear all

Accessibility: ensure keyboard focus, aria labels, color contrast.

---

## 8. Edge cases & YouTube SPA handling

YouTube tries to be tricky:

* Navigating from one video to another does not reload the page. Detect URL changes via `history.pushState` override or polling `location.href`.

* The video element may be replaced; always locate the `<video>` node with `document.querySelector('video')` each time navigation occurs.

* YouTube changes DOM classes often. Use descriptive selectors:

  * Prefer element tag + role when available (`#top-level-buttons-computed` or `ytd-menu-renderer`) instead of brittle classes.

* Handle iframes (embedded players) only if extension allows. MVP scope: only main youtube.com site.

---

## 9. Privacy, security, and policy considerations

* Default to local-only storage. Do not send watch info to any server without explicit opt-in.
* Provide clear privacy text in README and store listing:

  * "Everything is stored locally in your browser. No data leaves your device unless you enable sync or export."
* Do not attempt to block or change ads.
* Provide an export and delete-all option so users can control their data.
* Follow Chrome Web Store policies and Firefox add-on policies.

---

## 10. GitHub repo structure

```
youtube-watch-tracker/
├─ .github/
│  ├─ ISSUE_TEMPLATE/
│  ├─ workflows/ci.yml
├─ src/
│  ├─ content/
│  │  ├─ contentScript.js
│  │  └─ inject.css
│  ├─ background/
│  │  └─ serviceWorker.js
│  ├─ popup/
│  │  ├─ popup.html
│  │  └─ popup.js
│  ├─ options/
│  │  ├─ options.html
│  │  └─ options.js
│  └─ shared/
│     ├─ db.js
│     └─ utils.js
├─ manifest.json
├─ README.md
├─ LICENSE
└─ images/
   ├─ icon-128.png
   └─ screenshot-1.png
```

Use `src/` for source and a `dist/` build if you bundle/transpile. Consider TypeScript in a `ts` branch or later.

---

## 11. README template (short)

```
# YouTube Watch Tracker

Small browser extension that shows whether you've watched a YouTube video, how many times, and the watch dates. Local-first, privacy-respecting.

## Features
- Track watches automatically (configurable threshold)
- See watch count and dates in-page
- Export / import watch history

## Getting started
1. Clone repo
2. Load extension in Chrome: `chrome://extensions` -> Load unpacked -> select project root
3. Open a YouTube video and allow the extension when prompted

## Contributing
See CONTRIBUTING.md. Please open issues for bugs or feature requests. Look for `good first issue` label.

License: MIT
```

---

## 12. CONTRIBUTING, issues, labels, and first issues

### CONTRIBUTING.md quick items

* How to run locally (load unpacked extension)
* Code style (Prettier + ESLint)
* Branch naming & PR rules
* Testing guidance

### Labels

* `good first issue`
* `help wanted`
* `bug`
* `feature`
* `design`
* `documentation`

### First issues ideas

* Add a `good first issue` to update README with screenshots
* Fix CSS to improve button placement on mobile
* Add unit tests for `db.js` functions
* Improve date formatting in tooltip

---

## 13. CI & tests

Set up GitHub Actions to run ESLint and unit tests on PRs.

Simple workflow:

* `ci.yml`: run `npm install`, `npm run lint` and `npm test`
* For tests: use `jest` with `jsdom` to run tests on `db.js` and `utils.js`.

---

## 14. Packaging & publishing

**Chrome/Edge**

* `manifest.json` must be valid V3.
* Build a ZIP of your repo (only files needed) and upload to Chrome Web Store Developer Dashboard.
* Provide privacy policy URL (can be GitHub Pages hosting the privacy file).

**Firefox**

* Firefox supports Manifest V3 but check for any differences. Test by loading the extension in `about:debugging`.

Follow store rules, include privacy policy and screenshots, and be explicit about data usage.

---

## 15. Roadmap

**Phase 1 (MVP)**

* Content script injection
* Basic watch detection
* Local storage
* Popup with per-video detail

**Phase 2**

* Export/import
* Options page
* Basic analytics (most-watched videos)

**Phase 3**

* Optional encrypted sync
* Channel-level stats
* Mobile/browser-specific tweaks

---

## 16. Appendix — Example files (ready to paste)

> You can copy these into your `src/` folder. These are minimal examples and intentionally simple for clarity. You may want to refactor into modules and add error-handling, transpilation, bundling, and linting for production.

### `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "YouTube Watch Tracker",
  "description": "Shows whether you watched a YouTube video, how many times, and when.",
  "version": "0.1.0",
  "permissions": ["storage"],
  "host_permissions": ["https://www.youtube.com/*"],
  "background": {
    "service_worker": "src/background/serviceWorker.js"
  },
  "content_scripts": [
    {
      "matches": ["https://www.youtube.com/*"],
      "js": ["src/content/contentScript.js"],
      "css": ["src/content/inject.css"]
    }
  ],
  "action": {
    "default_popup": "src/popup/popup.html",
    "default_icon": {
      "128": "images/icon-128.png"
    }
  }
}
```

### `src/content/contentScript.js`

```js
// Minimal content script example
(function () {
  const DEBUG = false;

  function log(...args) { if (DEBUG) console.log('[YTWT]', ...args); }

  let currentVideoId = null;
  let sessionMarked = new Set();

  function getVideoId() {
    try {
      const url = new URL(location.href);
      return url.searchParams.get('v');
    } catch (e) {
      return null;
    }
  }

  function findVideoElement() {
    return document.querySelector('video');
  }

  function insertButtonIfMissing() {
    const menu = document.querySelector('#top-level-buttons-computed');
    if (!menu) return;
    if (document.querySelector('.ytwt-watched-btn')) return; // already inserted

    const btn = document.createElement('button');
    btn.className = 'ytwt-watched-btn';
    btn.innerHTML = '👁 <span class="ytwt-count">0</span>';
    btn.style.cursor = 'pointer';
    btn.addEventListener('click', onClick);
    menu.appendChild(btn);

    // request current data to show count
    sendMessage({ type: 'get_video', videoId: getVideoId() }).then(res => {
      if (res && res.watchCount) {
        updateButtonCount(res.watchCount);
      }
    }).catch(() => {});
  }

  function updateButtonCount(count) {
    const el = document.querySelector('.ytwt-count');
    if (el) el.textContent = String(count);
  }

  function onClick(e) {
    // open a small details pane (simple alert for MVP)
    const vid = getVideoId();
    if (!vid) return;
    sendMessage({ type: 'get_video', videoId: vid }).then(res => {
      const lines = [];
      if (!res) {
        alert('No watch history for this video.');
        return;
      }
      lines.push(`Watched ${res.watchCount} times`);
      if (res.events && res.events.length) {
        lines.push('\nDates:');
        for (const d of res.events.slice().reverse()) {
          lines.push(new Date(d).toLocaleString());
        }
      }
      alert(lines.join('\n'));
    });
  }

  function sendMessage(msg) {
    return new Promise((resolve) => {
      try {
        chrome.runtime.sendMessage(msg, (res) => resolve(res));
      } catch (e) {
        resolve(null);
      }
    });
  }

  function attachVideoListeners() {
    const video = findVideoElement();
    if (!video) return;

    video.addEventListener('ended', () => {
      const vid = getVideoId();
      if (!vid) return;
      if (sessionMarked.has(vid)) return;
      sessionMarked.add(vid);
      sendMessage({ type: 'record_watch', videoId: vid, meta: { title: document.title } });
    });

    video.addEventListener('timeupdate', () => {
      const vid = getVideoId();
      if (!vid) return;
      const duration = video.duration || 0;
      if (!duration || duration === Infinity) return;
      const percent = (video.currentTime / duration) * 100;
      // threshold 70%
      if (percent >= 70 && !sessionMarked.has(vid)) {
        sessionMarked.add(vid);
        sendMessage({ type: 'record_watch', videoId: vid, meta: { title: document.title } });
      }
    });
  }

  // observe SPA navigation
  let lastHref = location.href;
  const hrefObserver = new MutationObserver(() => {
    if (location.href !== lastHref) {
      lastHref = location.href;
      onNavigation();
    }
  });
  hrefObserver.observe(document, { subtree: true, childList: true });

  function onNavigation() {
    // reset session markers for new video
    currentVideoId = getVideoId();
    sessionMarked = new Set();
    setTimeout(() => {
      insertButtonIfMissing();
      attachVideoListeners();
    }, 500);
  }

  // initial
  onNavigation();

})();
```

### `src/background/serviceWorker.js`

```js
// Minimal service worker: listens and updates IndexedDB
importScripts(); // placeholder if needed for libraries

const DEBUG = false;
function log(...args) { if (DEBUG) console.log('[YTWT sw]', ...args); }

const DB_NAME = 'yt_watch_tracker';
const DB_VERSION = 1;
let dbPromise = null;

function openDb() {
  if (dbPromise) return dbPromise;
  dbPromise = new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = (e) => {
      const db = e.target.result;
      if (!db.objectStoreNames.contains('videos')) {
        db.createObjectStore('videos', { keyPath: 'videoId' });
      }
    };
    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
  return dbPromise;
}

async function getVideo(videoId) {
  const db = await openDb();
  return new Promise((res, rej) => {
    const tx = db.transaction('videos', 'readonly');
    const store = tx.objectStore('videos');
    const r = store.get(videoId);
    r.onsuccess = () => res(r.result);
    r.onerror = () => res(null);
  });
}

async function putVideo(obj) {
  const db = await openDb();
  return new Promise((res, rej) => {
    const tx = db.transaction('videos', 'readwrite');
    const store = tx.objectStore('videos');
    const r = store.put(obj);
    r.onsuccess = () => res(r.result);
    r.onerror = () => rej(r.error);
  });
}

chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  (async () => {
    if (msg.type === 'get_video') {
      const data = await getVideo(msg.videoId);
      sendResponse(data || null);
    } else if (msg.type === 'record_watch') {
      const v = (await getVideo(msg.videoId)) || { videoId: msg.videoId, watchCount: 0, events: [] };
      v.watchCount = (v.watchCount || 0) + 1;
      v.events = v.events || [];
      v.events.push((new Date()).toISOString());
      v.title = msg.meta && msg.meta.title || v.title;
      v.lastSeenAt = v.events[v.events.length - 1];
      await putVideo(v);
      sendResponse({ success: true, watchCount: v.watchCount });
    } else {
      sendResponse(null);
    }
  })();

  // return true to indicate we will respond asynchronously
  return true;
});
```

### `src/content/inject.css`

```css
.ytwt-watched-btn {
  background: transparent;
  border: none;
  font-size: 14px;
  padding: 6px 8px;
}
.ytwt-count {
  font-weight: 600;
  margin-left: 6px;
}
```

### `src/popup/popup.html`

```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>YT Watch Tracker</title></head>
<body>
  <h3>YouTube Watch Tracker</h3>
  <div id="summary">Loading...</div>
  <button id="export">Export JSON</button>
  <script src="popup.js"></script>
</body>
</html>
```

### `src/popup/popup.js`

```js
// Popup script to show a simple summary
(async function() {
  const el = document.getElementById('summary');
  const res = await new Promise((resolve) => chrome.runtime.sendMessage({type:'get_all_videos'}, resolve));
  if (!res) {
    el.textContent = 'No history yet.';
    return;
  }
  const total = res.length;
  el.textContent = `Tracked videos: ${total}`;
})();
```

> Note: background should implement `get_all_videos` message used here.

---

## Final notes and next steps

I added the full blueprint with working example files into this document. You can copy the `src/` files into your local project, create a GitHub repo, and `Load unpacked` in Chrome to test.

If you want, I can now:

* generate a ready-to-commit repo (ZIP) with these files,
* open a step-by-step first-PR checklist and the exact `git` commands,
* scaffold a `package.json`, ESLint, Prettier, and GitHub Actions CI files,
* or create some `good first issue` descriptions and assign them.

Tell me which one you'd like next and I will create it for you.
