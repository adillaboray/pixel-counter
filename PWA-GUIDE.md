# PWA Strategy Guide — Pixel Counter

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser / PWA Shell                      │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌────┐  │
│  │ counter  │  │ update-popup │  │install-popup │  │manifest │
│  │  .html   │  │  (UI layer)  │  │ (UI layer)   │  │ .json  │
│  └────┬─────┘  └──────┬───────┘  └──────┬───────┘  └────┘  │
│       │               │                      │              │
│  ┌────▼───────────────▼──────────────────────▼───────────┐  │
│  │              Service Worker (sw.js)                    │  │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐  │  │
│  │  │ Install  │  │ Activate │  │  Fetch (cache-first  │  │  │
│  │  │ (cache   │  │ (clean   │  │  + stale-while-     │  │  │
│  │  │ assets)  │  │ old caches)│  └─────────┬───────────┘  │  │
│  │  └──────────┘  └──────────┘  │           │              │
│  │                              │           ▼              │
│  │  ┌───────────────────────────┴──────────────────────┐  │
│  │  │           Cache Storage                          │  │
│  │  │  pixel-counter-v{N}                              │  │
│  │  │  ├─ /                     (HTML)                 │  │
│  │  │  ├─ /index.html         (HTML)                 │  │
│  │  │  ├─ /manifest.json        (PWA manifest)         │  │
│  │  │  ├─ /icon.svg             (app icon)             │  │
│  │  │  └─ /sw.js                (service worker)       │  │
│  │  └──────────────────────────────────────────────────┘  │
│  └────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────┘
```

---

## Update Deployment Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Server as Web Server
    participant Browser
    participant SW_Old as Old Service Worker
    participant SW_New as New Service Worker
    participant Cache as Cache Storage
    participant Popup as Update Popup

    Dev->>Server: 1. Edit assets (HTML/CSS/JS)
    Dev->>Server: 2. Bump CACHE_VERSION in sw.js
    Dev->>Server: 3. Deploy all files

    Browser->>Server: 4. Visit page
    Server-->>Browser: 5. Serve page + updated sw.js

    Browser->>SW_Old: 6. Detect sw.js byte difference
    SW_Old->>SW_New: 7. Install new SW in background

    SW_New->>Cache: 8. Create pixel-counter-v{N+1}
    SW_New->>Cache: 9. Cache all ASSETS in new cache

    Note over SW_New: install event complete

    SW_New->>Browser: 10. statechange → 'installed'
    Browser->>Popup: 11. Show "Update Available" popup
    Note over Popup: User sees blinking NEW badge

    User->>Popup: 12. Click UPDATE button
    Popup->>SW_New: 13. postMessage({ type: 'SKIP_WAITING' })
    SW_New->>SW_New: 14. self.skipWaiting()

    Note over SW_New: activate event fires

    SW_New->>Cache: 15. Delete old pixel-counter-v{N}
    SW_New->>SW_Old: 16. Take control (clients.claim())
    SW_New->>Browser: 17. controllerchange event
    Browser->>Browser: 18. window.location.reload()
    Note over Browser: Page loads with fresh assets from new cache
```

---

## Install Prompt Workflow

```mermaid
sequenceDiagram
    participant Browser
    participant Page as index.html
    participant Prompt as Install Popup
    participant App as Installed PWA

    Browser->>Page: 1. Detect manifest.json + SW registered
    Browser->>Browser: 2. Fire beforeinstallprompt event
    Browser->>Page: 3. event.preventDefault() (store event)
    Page->>Prompt: 4. Show install popup (top of screen)
    Note over Prompt: Cyan border, blinking INSTALL badge

    User->>Prompt: 5. Click YES
    Prompt->>Browser: 6. deferredPrompt.prompt()
    Browser->>User: 7. Show native install dialog
    User->>Browser: 8. Accept install
    Browser->>App: 9. Add to home screen / standalone window
    Note over App: App launches with splash screen, no browser chrome

    User->>Prompt: 5b. Click NAH
    Prompt->>Prompt: 6b. Hide popup, discard prompt
    Note over Prompt: Won't show again this session
```

### Dismiss behavior
- **YES** → fires native browser install prompt; on accept, app is installed
- **NAH** → hides popup and discards the deferred event (won't reappear until next page visit)
- Popup is positioned at the **top** of the screen (update popup is at the bottom) so both can coexist

### Requirements for the prompt to fire
| Condition | Detail |
|---|---|
| HTTPS or localhost | Required for all PWA features |
| Valid manifest.json | Must have name, icons, display, start_url |
| Active service worker | Must be registered and activated |
| User engagement | User must interact with the page before prompt |

---

## Caching Strategy

### On Install (Pre-cache)
When a new service worker installs, it creates a versioned cache and pre-caches all critical assets:

| Asset | Purpose |
|---|---|
| `/` | Root HTML entry point |
| `/index.html` | Explicit HTML fallback |
| `/manifest.json` | PWA install manifest |
| `/icon.svg` | App icon |
| `/sw.js` | Service worker itself |

### On Fetch (Cache-first + Stale-While-Revalidate)

```
         ┌──────────┐
         │  Request  │
         └────┬─────┘
              │
              ▼
       ┌──────────────┐
       │  caches.match │
       └──────┬───────┘
              │
         ┌────┴────┐
         │         │
         ▼         ▼
    ┌────────┐ ┌────────────┐
    │ Cached │ │ Not Cached │
    └───┬────┘ └─────┬──────┘
        │            │
        │            ▼
        │      ┌──────────┐
        │      │ fetch()  │
        │      └────┬─────┘
        │           │
        │      ┌────┴─────┐
        │      │          │
        │      ▼          ▼
        │ ┌────────┐ ┌────────┐
        │ │ 200 OK │ │ Fail   │
        │ │(cache+)│ │ 404    │
        │ │ return │ └────────┘
        │ └────────┘
        │
        │ (background)
        ▼
  ┌──────────────┐
  │ fetch(net)   │
  │ (silent)     │
  │ update cache │
  └──────────────┘
```

The cache is checked **first** and the response is returned **immediately** — no network wait. A network fetch runs **silently in the background** (left branch) to refresh the cache for the next visit. If the resource isn't cached (right branch), the handler waits for the network response and caches it on success. This guarantees instant loading from the home screen even when fully offline.

---

## Versioning Contract

### Deploying an Update

```
sw.js (line 1):
────────────────────────────────────────────────────
const CACHE_VERSION = 0.3;   ← BUMP THIS on every deploy
────────────────────────────────────────────────────
```

| Step | Action |
|---|---|
| 1 | Edit HTML, CSS, or other assets |
| 2 | Increment `CACHE_VERSION` in `sw.js` |
| 3 | Deploy everything to the server |
| 4 | Browser detects changed `sw.js` bytes |
| 5 | Update popup appears → user clicks UPDATE |
| 6 | New SW activates with fresh cache, old cache deleted |

### Why byte comparison works
Browsers detect service worker updates by comparing `sw.js` bytes. Bumping `CACHE_VERSION` changes the file content, which triggers the browser's update check on every navigation (or every 24h in Chrome). No additional polling or build tools needed.

---

## Key Files

| File | Role |
|---|---|
| `index.html` | App shell + SW registration + update popup + install popup UI |
| `sw.js` | Service worker: install, activate, fetch, messaging |
| `manifest.json` | PWA metadata for install prompt |
| `icon.svg` | App icon (512×512, pixel art) |

---

## Update Popup Behavior

| Trigger | Condition |
|---|---|
| **Appears** | New SW installs while old SW is still active (waiting state) |
| **UPDATE** | Sends `SKIP_WAITING` → new SW activates → page reloads |
| **✕ (dismiss)** | Hides popup until next page visit (SW still waiting) |
| **Next visit** | Waiting SW activates on next navigation automatically |

The popup uses the same 16-bit pixel aesthetic (`Press Start 2P` font, blinking `NEW` badge, double border) to match the app's visual identity.

---

## Install Popup Behavior

| Trigger | Condition |
|---|---|
| **Appears** | `beforeinstallprompt` fires (browser determines PWA installable) |
| **YES** | Calls `deferredPrompt.prompt()` → native install dialog → app installed on accept |
| **NAH** | Hides popup, discards deferred prompt (won't show again until next page visit) |
| **Position** | Top center of screen (update popup is bottom center — no overlap) |

### Visual identity
- **Border**: cyan (`#0ff`) with cyan double-shadow to distinguish from the update popup (white)
- **Badge**: cyan background with `INSTALL` label, blinking animation
- **YES button**: cyan background, black text
- **NAH button**: transparent, dim text
- The popup is positioned at the **top** of the screen so both popups can coexist without overlapping
