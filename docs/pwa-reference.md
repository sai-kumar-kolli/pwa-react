# Progressive Web Apps — Learning Notes

## Table of Contents

* [What is a PWA?](#what-is-a-pwa)
* [Service Worker](#service-worker)

  * [What it is](#what-it-is)
  * [Scope](#scope)
  * [Lifecycle](#lifecycle)
  * [Lifecycle hooks](#lifecycle-hooks)
  * [Registration](#registration)
  * [Detecting waiting state](#detecting-waiting-state)
* [Caching](#caching)
* [Workbox](#workbox)
* [PWA Installability](#pwa-installability)
* [Key Gotchas](#key-gotchas)
* [Thread and Resource Management](#thread-and-resource-management)

## What is a PWA?

A PWA is a web app that uses modern browser APIs to behave like a native app. It's not a binary — it's a spectrum of capabilities. The three core pillars are:

* **Service Worker** — a JavaScript file running in a separate worker context (not the page’s main thread), acting as a programmable network proxy
* **Web App Manifest** — a JSON file that tells the browser how the app should be displayed when “installed” (name, icons, start URL, display mode)

  Example minimal `manifest.json`:

  ```json
  {
    "name": "My PWA",
    "short_name": "PWA",
    "start_url": "/",
    "display": "standalone",
    "icons": [
      {
        "src": "/icon-192.png",
        "sizes": "192x192",
        "type": "image/png",
        "purpose": "any"
      }
    ]
  }
  ```
* **HTTPS** — required for service workers to function (localhost is exempt during development)

A service worker without a manifest is “just” a powerful runtime layer (caching/offline/routing/background-ish events). A manifest without a service worker is “just” metadata. Together they’re what usually convinces a browser to treat your site like an app.

**Clarification (Accuracy Detail)**
Some browsers can consider a site installable based on manifest + other heuristics even without a service worker, but service workers are typically expected for a real “app-like” experience and are required for offline/network interception. ([MDN Web Docs][1])

---

## Service Worker

### What it is

A service worker runs in a separate worker context from the main page — not on the page’s main JS thread and not on the UI thread. It has no access to `window` or the DOM. The global scope is `self`, not `window`.

```javascript
// Regular browser JS
window.addEventListener(...)

// Service worker — self is the global scope
self.addEventListener(...)
// or just
addEventListener(...) // implicitly self
```

### Scope

A service worker can only intercept requests within its scope, determined by where the file is served from (and optionally the `Service-Worker-Allowed` header). A service worker at `/sw.js` can control the whole origin. A service worker at `/app/sw.js` only controls requests under `/app/`.

This is why `sw.js` often goes in `public/` (for Vite/React setups) — it needs to be served at the root URL if you want it to control the whole site.

**Clarification (Accuracy Detail)**
“Public/” is a build-tool convention (Vite, CRA, etc.), not a browser rule. The browser only cares about the URL path it’s served from.

### Lifecycle

```
Register → Install → Waiting → Activate → Idle → (wake on event)
```

**Install** — fires when the service worker is first registered *and whenever a new SW version is discovered*. The page is not yet under SW control. This is the setup phase — pre-cache assets here. If install fails, the SW is discarded entirely and never reaches activate.

**Waiting** — after install, if an older SW is still controlling open tabs, the new SW waits. It won't activate until all tabs controlled by the old SW are gone. One service worker controls all tabs *within the same scope* — not one per tab.

**Activate** — fires when the SW takes control. The old SW is gone. This is the cleanup phase — delete caches from previous versions here. By the time activate fires it's safe to delete old caches because nothing is using them anymore.

**Idle** — after activation the SW sits idle. Browsers may terminate idle service workers aggressively (often within seconds to tens of seconds); the timing is implementation-specific. This is why a SW is not a long-running process — it's more like an event-driven function. ([Stack Overflow][2])

**Fetch/Push/Message** — the SW wakes up on demand when these events fire, handles the event, then returns to idle.

### Lifecycle hooks

```javascript
// Hold a lifecycle phase open until async work is done
event.waitUntil(somePromise)

// Take ownership of a network request
event.respondWith(somePromise) // promise must resolve to a Response
```

`event.waitUntil` is for lifecycle phases (install, activate) — keep the SW alive while async work runs. Without it, the browser may consider the phase complete before your async work finishes.

`event.respondWith` is for the fetch event — the browser waits for your response because it literally can't proceed without one.

### Registration

```javascript
// In main.jsx — register after page load to avoid competing with initial render
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registered:', reg))
      .catch(err => console.log('SW registration failed:', err))
  })
}
```

Why wait for `load`? SW registration consumes resources. Registering during page load can compete with your critical assets loading.

**Clarification (Accuracy Detail)**
This is a performance tradeoff, not a correctness requirement. Many apps register earlier for faster “time-to-offline,” but doing it after `load` is a common conservative choice.

### Detecting waiting state

```javascript
navigator.serviceWorker.register('/sw.js').then(reg => {
  if (reg.waiting) {
    // New SW installed but waiting — other tabs still open
    // Good place to show "New version available" UI
  }

  reg.addEventListener('updatefound', () => {
    const newSW = reg.installing
    newSW.addEventListener('statechange', () => {
      console.log('SW state:', newSW.state)
      // installing → installed → activating → activated
    })
  })
})
```

---

## Caching

### Browser cache vs Cache Storage

|               | Browser Cache                 | Cache Storage API                       |
| ------------- | ----------------------------- | --------------------------------------- |
| Controlled by | Server (via response headers) | You (via JavaScript)                    |
| Location      | Managed by browser on disk    | Managed by browser on disk, per origin  |
| Eviction      | Server-defined expiry         | Browser evicts under disk pressure      |
| Access        | Automatic                     | Explicit — you must read/write manually |

Cache Storage is part of the origin's storage bucket alongside localStorage, IndexedDB, and Cookies. Visible in DevTools → Application → Cache Storage.

**The browser does not consult Cache Storage automatically for navigation or subresource requests.** You must explicitly check it in your fetch handler.

### Caching strategies

| Strategy               | Pattern                                       | Use case                          |
| ---------------------- | --------------------------------------------- | --------------------------------- |
| Cache First            | Serve from cache, fallback to network         | Static assets (JS, CSS, fonts)    |
| Network First          | Try network, fallback to cache                | API calls where freshness matters |
| Stale While Revalidate | Serve cache immediately, update in background | Content where speed > freshness   |
| Cache Only             | Only cache, never network                     | Fully offline assets              |
| Network Only           | Never cache                                   | Real-time data, analytics         |

### Pre-caching vs Runtime caching

**Pre-caching** — cache assets during the install phase before the SW takes control. Assets are guaranteed available from the first controlled page load. Used for the app shell — the minimum assets needed to render something.

**Runtime caching** — cache responses as they happen in the fetch handler. Assets get cached on first visit and served from cache on subsequent visits. Used for everything else — JS chunks, images, API responses.

### Manual implementation

```javascript
const CACHE_NAME = 'v1'

// Install — pre-cache app shell
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      return cache.addAll(['/', '/offline.html'])
    })
  )
})

// Activate — clean up old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames
          .filter(name => name !== CACHE_NAME)
          .map(name => caches.delete(name))
      )
    })
  )
})

// Fetch — serve from cache, runtime cache network responses
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cachedResponse => {
      if (cachedResponse) return cachedResponse

      return fetch(event.request).then(networkResponse => {
        return caches.open(CACHE_NAME).then(cache => {
          cache.put(event.request, networkResponse.clone()) // clone — Response body is a stream, can only be read once
          return networkResponse
        })
      }).catch(() => {
        if (event.request.destination === 'document') {
          return caches.match('/offline.html')
        }
      })
    })
  )
})
```

### Why response cloning matters

A Response object has a body stream — once read it's consumed and gone. If you pass the same response to both the cache and the browser, one of them gets an empty stream.

```javascript
const clone = networkResponse.clone() // two independent streams
cache.put(request, clone)             // cache gets one stream
return networkResponse                // browser gets the other
```

---

## Workbox

Workbox is Google's library that provides pre-built implementations of caching strategies and automates the boilerplate of service worker management.

### Why it exists

Manual service workers require you to:

* Know all asset filenames upfront (including Vite's hashed names)
* Write caching strategy logic per request type
* Handle cache versioning and cleanup manually
* Deal with edge cases in offline fallbacks

Workbox solves all of this.

### vite-plugin-pwa

The recommended way to use Workbox with Vite. It integrates Workbox into the build pipeline, automatically generates a precache manifest with all asset filenames including hashes, and handles SW registration.

```javascript
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate', // skipWaiting — auto activate new SW
      manifest: {
        name: 'My App',
        short_name: 'App',
        start_url: '/',
        display: 'standalone',
        background_color: '#ffffff',
        theme_color: '#000000',
        icons: [
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png', purpose: 'any' },
          { src: '/icon-192.png', sizes: '192x192', type: 'image/png', purpose: 'maskable' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png', purpose: 'any' },
          { src: '/icon-512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' }
        ]
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png}'] // files to precache
      }
    })
  ]
})
```

`registerType: 'autoUpdate'` — skips waiting and claims all clients immediately. Good for development. In production use `'prompt'` and show a "New version available" UI so users control when updates happen.

> **Tip:** `clients.claim()` ensures the active SW takes control of open pages immediately, while `skipWaiting()` lets the new SW activate without waiting; Workbox handles both for you.

### What Workbox generates

```javascript
self.skipWaiting()
clientsClaim()

// Precache manifest — auto-generated from dist folder
precacheAndRoute([
  { url: 'index.html', revision: 'abc123' },         // revision hash for non-hashed files
  { url: 'assets/index-SguGrCfz.js', revision: null } // null — hash already in filename
])

cleanupOutdatedCaches() // equivalent to manual activate cleanup

// SPA routing — all navigation requests serve index.html
registerRoute(new NavigationRoute(createHandlerBoundToURL('index.html')))
```

Revision hashes are used for files without hashed filenames (like `index.html`). Workbox uses them to detect changes and invalidate the precache entry. Files with hashed filenames (Vite output) don't need revision hashes — the hash in the filename already serves that purpose.

---

## PWA Installability

### What makes an app installable

The browser (not the OS) decides installability. **Chrome’s install prompt criteria** commonly include:

* **HTTPS** — or localhost for development
* **A web app manifest** with:

  * `name` or `short_name`
  * `start_url`
  * `display` set to `standalone`, `fullscreen`, or `minimal-ui`
  * At least one icon of 192x192 and one of 512x512 (PNG recommended)
* **A service worker** (often expected for the “app” experience; historically required by Chrome to be considered installable in many flows)

**Clarification (Accuracy Detail)**
Across browsers, a service worker is not universally a hard requirement for “installability” as a concept, but it is central to the PWA experience and commonly expected by install-promotion heuristics. ([MDN Web Docs][1])

Chrome also applies a user engagement heuristic — it won't show the install prompt on first visit. Override this in development with `chrome://flags/#bypass-app-banner-engagement-checks`.

💡 Use **Lighthouse** (DevTools → Audits or the CLI) to verify installability, performance, and other PWA best practices. ([Chrome for Developers][3])

### How PWA installation differs from native

Native (.dmg/.exe):

* Copies binaries to disk
* OS registers the app (file associations, start menu, registry)
* Direct hardware/OS API access
* Developer controls update cycle

PWA:

* Browser creates a shortcut/app entry pointing to a URL
* App still runs inside a sandboxed browser engine (just without address bar)
* No “app binaries” copied — assets typically live in Cache Storage / HTTP cache
* Updates silently on next visit via service worker
* Same security sandbox as any website

The "installation" is the browser saying: "I'll give this URL its own window, its own taskbar entry, and hide the fact that it's a browser."

### Manifest icon purpose

Separate `any` and `maskable` into distinct entries — combining them is discouraged:

```json
{ "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
{ "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "maskable" }
```

`any` — used as-is on platforms that don't mask icons
`maskable` — safe zone icon, OS can apply its own shape mask (circle, squircle etc)

### Richer PWA Install UI

Chrome shows an enhanced install prompt with app preview when screenshots are provided:

```json
"screenshots": [
  { "src": "/screenshot-wide.png", "sizes": "1280x720", "type": "image/png", "form_factor": "wide" },
  { "src": "/screenshot-mobile.png", "sizes": "390x844", "type": "image/png", "form_factor": "narrow" }
]
```

Without screenshots, Chrome falls back to a basic install dialog. Functional but less polished.

---

## Key Gotchas

**Service worker doesn't control the page on first load.** The SW registers after your registration code runs — all initial requests (JS bundles, CSS) may happen before the SW is active. The SW only starts intercepting from the next navigation/load that it controls. Practical implication: your app may only become “fully cached” after multiple loads, depending on whether you precache bundles or rely on runtime caching.

**`cache.addAll` makes fresh network requests.** Pre-caching in install isn't intercepting the requests that already happened. It's making new separate requests to populate the cache. Your server can see two requests for `'/'` on first load — one from the browser, one from the SW — because the SW isn't yet controlling the page when install runs; they're independent fetches.

**Install fires on every new SW version.** If you change `sw.js` by even one character, the browser treats it as a new SW and fires install again. This is how cache versioning works.

**Both old and new cache exist during waiting.** The new SW creates its new cache during install, but can't delete the old cache until activate — because the old SW may still be using it. Two versions can coexist on disk during the waiting phase.

**Offline blank page problem.** If your app shell's JS bundles aren't cached and the user goes offline, `index.html` loads from cache but JS bundles fail. React never boots, leaving a blank page. Solution: pre-cache the bundles (Workbox handles this) or serve a standalone `offline.html` for document requests that has no JS dependencies.

**`offline.html` must be pre-cached, not runtime cached.** If it's only runtime cached it might not be there when you need it. Pre-cache it in install to guarantee availability.

**`event.respondWith` with `offline.html` for all requests breaks things.** If you return `offline.html` for JS/CSS requests, the browser tries to execute HTML as JavaScript and everything breaks. Gate the offline fallback on `event.request.destination === 'document'`.

**Codespaces proxy intercepts manifest and icon requests.** By default, the manifest is fetched **without credentials (no cookies)** even if it’s same-origin. If your port is private and requires auth cookies, the manifest request can be redirected to login. Fix: make the port public or configure the manifest link with `crossorigin="use-credentials"` if you truly need credentialed fetching. ([web.dev][4])

**`purpose: 'any maskable'` is discouraged.** Separating into two icon entries with individual purpose values is the correct approach.

* **Background Sync / Periodic Sync** — these extra service worker APIs let you defer actions until the device is online or at specific intervals; treat them as optional enhancements rather than core requirements.

---

## Thread and Resource Management

Service workers don’t get a dedicated OS thread each. Browsers maintain thread pools internally and schedule work across them. A SW only needs CPU time while actively handling an event — install, fetch, push. When idle it can be suspended/terminated and restarted later.

Browsers may terminate idle service workers aggressively (often within seconds to tens of seconds) to reclaim memory. They restart cold when needed. This is why service workers must be stateless across restarts — you can't rely on in-memory state between events. ([Stack Overflow][2])

For push notifications when the browser is closed: on desktop Chrome, background processes may remain running if background execution is enabled, allowing push to wake a SW. If you fully exit/force-quit (or background running is disabled), delivery may be delayed until the browser starts again.

Tab discarding: Chrome's memory saver can evict background tab memory under pressure. The SW and the tab have separate lifecycles — a discarded tab can reload cold later, and the SW can still be woken by events depending on browser/platform behavior.

**Clarification (Accuracy Detail)**
The “SW can still wake up even if the tab is discarded” part is platform- and permission-dependent (push, sync, and background execution policies vary), so treat it as “can happen,” not “guaranteed.”

[1]: https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Guides/Making_PWAs_installable?utm_source=chatgpt.com "Making PWAs installable - Progressive web apps | MDN"
[2]: https://stackoverflow.com/questions/42733835/service-worker-termination-by-a-timeout-timer-was-canceled-because-devtools-is-a?utm_source=chatgpt.com "Service Worker termination by a timeout timer was ..."
[3]: https://developer.chrome.com/docs/lighthouse/pwa/installable-manifest?utm_source=chatgpt.com "Web app manifest does not meet the installability requirements"
[4]: https://web.dev/articles/add-manifest?utm_source=chatgpt.com "Add a web app manifest | Articles"
