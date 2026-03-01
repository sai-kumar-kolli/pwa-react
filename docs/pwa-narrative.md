# PWA — From React Developer to Understanding the Platform

> Written from the perspective of a frontend developer who spent years in React land and wanted to understand what actually makes a web app a PWA — not just how to tick the checklist.

---

## Table of Contents

* [The Wrong Question](#the-wrong-question)
* [The Thing Nobody Explains About Service Workers](#the-thing-nobody-explains-about-service-workers)

  * [Where it actually lives](#where-it-actually-lives)
  * [The thread pool reality](#the-thread-pool-reality)
  * [One SW per origin, not per tab](#one-sw-per-origin-not-per-tab)
* [The Lifecycle — A State Machine](#the-lifecycle--a-state-machine)
* [Caching — Two Separate Systems](#caching--two-separate-systems)
* [The First Load Problem — A Trace](#the-first-load-problem--a-trace)
* [The Offline Problem — What Actually Breaks](#the-offline-problem--what-actually-breaks)
* [Response Cloning — The Stream Problem](#response-cloning--the-stream-problem)
* [Workbox — What It Actually Solves](#workbox--what-it-actually-solves)
* [Installability — How the Browser Decides](#installability--how-the-browser-decides)
* [The Codespaces Gotcha — A Real Example of Browser Internals](#the-codespaces-gotcha--a-real-example-of-browser-internals)
* [Push Notifications — The Architecture (Not Yet Implemented)](#push-notifications--the-architecture-not-yet-implemented)
* [Mental Models That Actually Stuck](#mental-models-that-actually-stuck)
* [What I Don't Know Yet](#what-i-dont-know-yet)
* [The One-Sentence Summaries](#the-one-sentence-summaries)

## The Wrong Question

The first instinct is to ask: *"what do I need to add to make this a PWA?"*

That's the wrong question. The right question is: *"what does the browser need to consider this app self-sufficient enough to live outside a tab?"*

That reframe changes everything. You're not adding features to a webpage. You're convincing the browser that your app can survive on its own — offline, without a URL bar, without a server always being there.

Three things convince it:

```text
┌─────────────────────────────────────────────────────────┐
│                    PWA = Three Pillars                  │
│                                                         │
│   HTTPS              Manifest            Service Worker │
│   ──────             ────────            ─────────────  │
│   Security           Metadata            Capability     │
│   requirement        "here's how         "I can work    │
│   (localhost         to display          without the    │
│   is exempt)         my app"             network"       │
│                                                         │
│   Without HTTPS → SW won't register                     │
│   Without Manifest → not installable                    │
│   Without SW → no offline, no background capabilities   │
└─────────────────────────────────────────────────────────┘
```

A manifest without a service worker is just metadata. A service worker without a manifest is just a caching layer. You need both.

Minimal manifest example:

```json
{
  "name": "My PWA",
  "short_name": "App",
  "start_url": "/",
  "display": "standalone",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" }
  ]
}
```

---

## The Thing Nobody Explains About Service Workers

Everyone says "service worker runs in a separate thread." That's true but incomplete. The more important thing is *why* and *what that actually means*.

**Clarification (Accuracy Detail)**
MDN is great for practical behavior and constraints, but the authoritative rules live in the W3C Service Workers specification. Reading both gives you the best mental model.

### Where it actually lives

```text
┌──────────────────────────────────────────────────────────────┐
│                         Browser Process                      │
│                                                              │
│  ┌─────────────────┐    ┌──────────────────────────────────┐ │
│  │   Main Thread   │    │         Worker Threads           │ │
│  │                 │    │                                  │ │
│  │  - DOM          │    │  ┌────────────┐ ┌─────────────┐  │ │
│  │  - React        │    │  │  SW Tab 1  │ │  SW Tab 2   │  │ │
│  │  - Your JS      │    │  │  (idle)    │ │  (active)   │  │ │
│  │  - window       │    │  └────────────┘ └─────────────┘  │ │
│  │                 │    │                                  │ │
│  └─────────────────┘    │  Thread Pool (shared, limited)   │
│                         └──────────────────────────────────┘ │
│                                                              │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                          OS Layer                            │
│                                                              │
│   OS sees: "Chrome is using N threads"                       │
│   OS does NOT know about tabs, SWs, or React                 │
│   Thread allocation is entirely Chrome's internal business   │
└──────────────────────────────────────────────────────────────┘
```

The OS gives Chrome threads and says "do what you want." Chrome decides internally how to slice those threads across tabs, service workers, V8, rendering — everything. The OS just sees one process called Chrome (plus whatever extra processes Chrome spawns).

### The thread pool reality

You open 30 tabs. Does that mean 30 service workers all running simultaneously on 30 threads? No.

```text
Reality:
┌──────────────────────────────────────────────────────────────┐
│                    Chrome Thread Pool                        │
│                                                              │
│  Active tab       → Full thread, full resources              │
│  Recent tabs      → Memory kept, CPU throttled               │
│  Background tabs  → Deprioritized, may be discarded          │
│  Discarded tabs   → Memory evicted, state serialized         │
│                    (reloads cold when you switch back)       │
│                                                              │
│  Heuristic: most tabs are idle most of the time              │
│  Same logic cloud providers use to oversell capacity         │
└──────────────────────────────────────────────────────────────┘
```

A service worker is closer to a serverless function than a server. It wakes up, handles an event, dies. Browsers terminate idle service workers aggressively (often within seconds to tens of seconds) to reclaim memory. The exact timing is implementation-specific and not standardized. This is intentional — SWs must behave as stateless across restarts.

**Clarification (Accuracy Detail)**
“Stateless” here means *don’t rely on in-memory state surviving*. If you need state, store it in IndexedDB / Cache Storage / etc.

### One SW per origin, not per tab

This trips everyone up.

```
Tab 1  ─┐
Tab 2  ──┼──→  Single Service Worker  →  Single Cache Storage
Tab 3  ─┘

All tabs (within the same scope) share the same SW and the same cache.
Cache warmed by Tab 1 is immediately available to Tab 2 and Tab 3.
```

**Clarification (Accuracy Detail)**
It’s really **one active SW per scope**, not strictly “per origin.” One origin can have multiple SWs if they control different scopes (e.g. `/app/` and `/admin/`).

Note that "origin" really means "scope"; a service worker at `/app/sw.js` won't control `/other/` even on the same domain.

This is why the waiting phase exists — you can't have two different SW versions running simultaneously for the same scope. The new one waits until every tab using the old one is closed (unless you force it).

---

## The Lifecycle — A State Machine

Don't think of it as a sequence of functions. Think of it as a state machine the browser controls, with hooks it gives you to participate.

```
  navigator.serviceWorker.register('/sw.js')
                │
                ▼
         ┌─────────────┐
         │  INSTALLING │◄─── install event fires
         │             │     your hook: event.waitUntil()
         └────┬────────┘     if promise rejects → SW discarded
              │
              ▼
         ┌─────────────┐
         │   WAITING   │◄─── new SW waiting for old tabs to close
         │             │     no event — browser manages this
         └────┬────────┘     skipWaiting() forces past this
              │
              ▼
         ┌─────────────┐
         │  ACTIVATING │◄─── activate event fires
         │             │     your hook: event.waitUntil()
         └────┬────────┘     old SW is gone, safe to clean up
              │
              ▼
         ┌─────────────┐
         │   ACTIVE    │◄─── SW is now controlling pages
         │             │
         └────┬────────┘
              │
         ┌────┴──────────────────┐
         ▼                         ▼
    ┌─────────┐               ┌─────────┐
    │  IDLE   │◄──────────────│ ACTIVE  │
    │         │  event done   │ handling│
    │ killed  │               │ fetch/  │
    └─────────┘               │ push/   │
                              │ message │
                              └─────────┘
```

### The phase responsibilities

```
┌──────────┬────────────────────────────────┬──────────────────────┐
│  Phase   │  Your responsibility            │  Your hook          │
├──────────┼────────────────────────────────┼──────────────────────┤
│ Install  │  Pre-cache app shell           │  event.waitUntil()   │
│          │  (setup before taking control) │                      │
├──────────┼────────────────────────────────┼──────────────────────┤
│ Waiting  │  Nothing — browser managed     │  skipWaiting()       │
│          │  (safety buffer)               │  to force past it    │
├──────────┼────────────────────────────────┼──────────────────────┤
│ Activate │  Delete old caches             │  event.waitUntil()   │
│          │  (cleanup after old SW gone)   │                      │
├──────────┼────────────────────────────────┼──────────────────────┤
│ Fetch    │  Serve from cache or network   │  event.respondWith() │
│          │  (the actual work)             │                      │
└──────────┴────────────────────────────────┴──────────────────────┘
```

### Why `event.waitUntil` exists

The browser does not automatically extend service worker lifecycle events for asynchronous work unless you explicitly signal it using `event.waitUntil()`.

```
Without event.waitUntil:
install handler returns → browser moves to activate
                                    ↑
                          cache.addAll still running
                          activate fires before cache is ready
                          ← broken

With event.waitUntil:
install handler returns → browser holds install phase open
                          waits for your promise to resolve
                          then moves to activate
                          ← correct
```

---

## Caching — Two Separate Systems

This is the mental model most people never get clear:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Two Cache Systems                          │
│                                                                 │
│  Browser HTTP Cache              Cache Storage API              │
│  ─────────────────               ────────────────               │
│  Controlled by: server           Controlled by: you             │
│  Via: Cache-Control headers      Via: JavaScript                │
│  You can't override it           You write and read it          │
│  Browser reads it automatically  Browser never consults it      │
│                                  automatically for requests     │
│                                  unless you tell it to          │
│                                                                 │
│  Same origin's storage:                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  localhost:5173                                         │    │
│  │  ├── localStorage                                       │    │
│  │  ├── IndexedDB                                          │    │
│  │  ├── Cache Storage  ← this is what SW uses              │    │
│  │  └── Cookies                                            │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**The browser never automatically reads from Cache Storage for navigation or subresource requests.** You must explicitly check it in the fetch handler. This surprises people — you can spend hours pre-caching assets and wonder why everything still goes to the network. Because you never wrote the read logic.

### Caching strategies — pick per request type

```
Request type          Strategy              Reasoning
─────────────         ──────────            ──────────
JS/CSS bundles   →    Cache First     →     Hashed filenames = safe forever
HTML pages       →    Network First   →     Want fresh, fallback if offline
API responses    →    Network First   →     Always want current data
Images/fonts     →    Cache First     →     Rarely change, expensive to fetch
Real-time data   →    Network Only    →     Never cache (prices, live scores)
```

### Pre-cache vs Runtime cache

```
Pre-cache (install phase):
  - You know the files upfront
  - Cached before SW takes control
  - Guaranteed available from first controlled load
  - Used for: app shell, offline fallback page

Runtime cache (fetch handler):
  - Cached as requests happen naturally
  - Available from second visit onwards
  - Used for: everything else — JS chunks, images, API responses

The combination:
  Install  → cache '/', '/offline.html' (guaranteed)
  Runtime  → cache everything else as it's requested (opportunistic)
```

---

## The First Load Problem — A Trace

This is the most important thing to understand about service workers. Walk through it exactly:

```
FIRST LOAD (times are illustrative):
──────────
  t=0ms   Browser requests '/'
  t=50ms  Server responds with index.html
  t=51ms  Browser parses HTML, finds script tags
  t=52ms  Browser requests main.js, index.css...
  t=200ms All assets loaded
  t=200ms 'load' event fires
  t=201ms Registration code runs
  t=202ms Browser fetches sw.js
  t=210ms install fires → cache.addAll(['/']) runs
            ↑ This makes a FRESH request for '/'
            ↑ Server receives 2 requests for '/' — one from browser, one from SW
  t=250ms activate fires → old cache cleanup runs
  t=251ms SW is now active

  BUT: The JS/CSS that loaded at t=52ms?
       SW wasn't registered yet. Not intercepted. Not cached.
       SW only controls from the NEXT page load.
```

```
SECOND LOAD:
───────────
  t=0ms   Browser requests '/'
  t=0ms   SW intercepts → cache hit → returns instantly
  t=1ms   Browser parses HTML, finds script tags
  t=2ms   Browser requests main.js → SW intercepts → NOT in cache → network
  t=50ms  Network responds → SW runtime caches it → returns to browser
  t=51ms  Browser requests index.css → SW intercepts → NOT in cache → network
  ...
  Second load: SW active, but JS bundles not yet cached (unless you precached them with Workbox).
```

```
THIRD LOAD:
──────────
  t=0ms   Browser requests '/' → cache hit
  t=1ms   Browser requests main.js → cache hit
  t=1ms   Browser requests index.css → cache hit
  t=2ms   App fully loaded from cache. Server not hit.

  This is the load your users experience from third visit onwards.
```

The insight: **your app only becomes fully cached from the third load.** This is why Workbox's precaching matters — it injects a build-generated asset manifest into the service worker so it can cache all assets (including JS bundles) during install, collapsing "third load behavior" to "second load."

---

## The Offline Problem — What Actually Breaks

Go offline between second and third load. Here's what happens:

```
OFFLINE — without Workbox precaching:
──────────────────────────────────────
  '/'         → cache hit ✓    (pre-cached in install)
  main.js     → cache miss     (runtime cached? depends on second load)
  index.css   → cache miss

  If JS bundles not cached:
  index.html loads → React tries to boot → main.js fails
  → blank page

  If you serve offline.html for ALL failed requests:
  index.html loads → requests main.js → SW returns offline.html
  → browser tries to execute HTML as JavaScript
  → everything explodes
  → still blank page, but now with errors
```

The fix requires two things working together:

```
1. Gate offline.html to document requests only:
   event.request.destination === 'document'
   → HTML pages get offline.html
   → JS/CSS requests fail with network errors

2. But then index.html loads, JS fails to load, blank page anyway

Real fix: Workbox precaches ALL assets including JS bundles
→ collapse to third-load behavior from second visit
→ everything available offline from second visit
```

This is the complexity that makes manual service workers painful and Workbox necessary in production.

---

## Response Cloning — The Stream Problem

A Response is an object, but its body is a stream — like a pipe. Once you read the body, it's gone.

```
Without cloning:
────────────────
  fetch() returns response (stream)
       │
       ├──→ cache.put(response)     ← body consumed here
       │
       └──→ return response         ← empty pipe, browser gets nothing
                                       broken silently

With cloning:
─────────────
  fetch() returns response (stream)
       │
       ├── response.clone() ────→ clone (independent stream)
       │                              │
       │                              └──→ cache.put(clone)   ← cache gets full response
       │
       └──→ return response   ← browser gets full response
```

```javascript
fetch(event.request).then(networkResponse => {
  caches.open('v1').then(cache => {
    cache.put(event.request, networkResponse.clone()) // clone for cache
  })
  return networkResponse // original for browser
})
```

---

## Workbox — What It Actually Solves

Workbox isn't magic. It solves specific, concrete problems you hit when doing this manually:

```
Problem                           Workbox solution
───────                           ────────────────
Hashed filenames change           Build tool scans dist/, injects
every build — can't hardcode      all filenames automatically

Writing cache strategy            Pre-built strategies:
logic for every route             CacheFirst, NetworkFirst,
is verbose and error-prone        StaleWhileRevalidate etc.

Cache versioning — knowing        Revision hashes for non-hashed
when to invalidate an entry       files, null for hashed filenames

SPA routing — /dashboard          NavigationRoute automatically
served index.html for all         serves index.html for all
navigation requests               navigation requests

Boilerplate                       Single plugin, handles
                                  registration + generation
```

### What Workbox generates (annotated)

```javascript
self.skipWaiting()      // ← registerType: 'autoUpdate'
                        //   skip waiting, take control immediately

clientsClaim()          // ← claim all open tabs immediately
                        //   pairs with skipWaiting

precacheAndRoute([
  {
    url: 'index.html',
    revision: 'abc123'  // ← hash generated by Workbox
                        //   index.html has no hash in filename
                        //   Workbox tracks changes via this revision
  },
  {
    url: 'assets/index-SguGrCfz.js',
    revision: null      // ← null because hash IS the filename
                        //   SguGrCfz changes when content changes
                        //   no separate revision needed
  }
])

cleanupOutdatedCaches() // ← your manual activate cleanup, automated

registerRoute(
  new NavigationRoute(
    createHandlerBoundToURL('index.html')
  )
)                       // ← serve index.html for /dashboard, /profile etc.
                        //   React Router handles the actual routing
```

---

## Installability — How the Browser Decides

```
Browser's decision tree:
────────────────────────

Is this HTTPS (or localhost)?
  NO  → not installable. Stop.
  YES → continue

Does a service worker successfully control the page and register a fetch handler?
  NO  → not installable. Stop.
  YES → continue

Does a valid manifest exist?
  NO  → not installable. Stop.
  YES → check manifest fields

Manifest has name/short_name?         YES → continue
Manifest has start_url?               YES → continue
Manifest display is standalone/       YES → continue
  fullscreen/minimal-ui?
Manifest includes a 192×192 icon?     YES → continue
Manifest includes a 512×512 icon?     YES → installable ✓

Chrome additionally checks:
  User engagement heuristic → override with chrome://flags
                               #bypass-app-banner-engagement-checks

💡 Run Lighthouse (DevTools → Audits or the CLI) to validate installability and other PWA best practices.
```

### PWA installation vs native installation

```
Native (.dmg/.exe):                   PWA:
───────────────────                   ────
User downloads installer              User visits website
Installer copies binaries to disk     Browser checks criteria
OS registers app                      Browser shows install prompt
App gets direct OS API access         Browser creates shortcut → URL
Developer controls updates            Updates silently, next visit
Uninstall removes binaries            Uninstall removes app entry/shortcut
                                      (origin storage may persist until cleared)

Key insight: PWA "installation" is the browser saying
"I'll give this URL its own window and hide that it's a browser"
The app still runs in a sandboxed browser engine.
It has the same permissions as any website.
```

---

## The Codespaces Gotcha — A Real Example of Browser Internals

Running a PWA in GitHub Codespaces exposed something most developers never see:

```
Why did manifest.json get intercepted but main.js didn't?

main.js request:
  Sec-Fetch-Dest: script
  Cookie: <github session cookie>     ← cookie attached, proxy allows through

manifest.json request:
  Sec-Fetch-Dest: manifest
  Cookie: (none)                      ← no cookie attached

Why no cookie on manifest?
  Browser behavior: manifest is typically fetched with credentials mode "omit"
  It's a configuration file, not a user resource
  Browser deliberately omits cookies in that mode

Why did this cause a 302?
  Port was Private → proxy requires auth for every request
  Manifest arrives with no cookie → proxy redirects to sign-in
  Fix: set port to Public → proxy skips auth check entirely
```

The broader lesson: **authentication and cookies are tied to request context, not just the origin.** Same URL, different behavior depending on who makes the request and what credentials they carry. You'll hit this in production with CDNs, API gateways, and authentication middleware.

---

## Push Notifications — The Architecture (Not Yet Implemented)

Understanding the flow without implementing it:

```
Your Server
    │
    │  POST push message
    ▼
Push Service (Google FCM / Apple APNs / Mozilla)
    │
    │  OS-level notification
    ▼
OS Notification Daemon
    │
    │  wakes up browser background agent
    ▼
Browser Background Process (thin, may remain running)
    │
    │  spins up service worker
    ▼
Service Worker (push event fires)
    │
    │  shows notification
    ▼
User sees notification

Key: your server never talks directly to the user's device.
It talks to the push service which owns the OS-level channel.
```

When browser is "closed": on desktop Chrome, background processes may remain running to support push and background tasks if background execution is enabled. If the browser is fully exited, push delivery may be delayed until the browser restarts. On mobile the behavior varies; quitting the browser often tears down the channel until the next foreground launch.

---

## Mental Models That Actually Stuck

**Service worker as event-driven worker**
Not a server that runs continuously. It behaves like a serverless function: it wakes up for events, handles work, and can be terminated between events. Stateless across restarts by design. The browser is the platform that invokes it.

**Browser as installer**
For PWAs, the browser is the middleman between your web app and the OS. The OS doesn't know about your PWA — it just sees Chrome created a shortcut/app entry. Chrome decides installability, Chrome manages the installed entry, Chrome handles updates.

**Cache Storage as write-only until you add read logic**
You can pre-cache everything perfectly and still have every request go to the network — because you never told the fetch handler to check the cache. Caching and serving from cache are two separate responsibilities.

**Thread pool as apartment building**
The OS is the landlord — gives Chrome a unit (threads). What Chrome does inside that unit is entirely its business. The landlord doesn't know about tabs, service workers, or React. Chrome runs its own scheduler inside the space the OS gave it.

**Response as pipe, not object**
A Response isn't data sitting in memory. Its body is a stream. Once water flows through the pipe, the pipe is empty. Clone before consuming.

---

## What I Don't Know Yet

These are the edges of current understanding — not gaps, just the next frontier:

* **Push API implementation** — the architecture is clear, the implementation is next
* **Background Sync / Periodic Sync** — queuing offline actions and replaying when online (the Background Sync API and its periodic cousin)
* **Workbox routing in depth** — per-route strategy configuration for complex apps
* **PWA on iOS** — Safari's limited and quirky PWA support, what's actually possible post iOS 16.4
* **Web App Window Controls Overlay** — customizing the title bar area on desktop PWAs
* **File System Access API** — how PWAs can interact with the file system within the sandbox

---

## The One-Sentence Summaries

If you had to explain each concept to a colleague in one sentence:

| Concept            | The sentence                                                                                                                                                                                                                           |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Service Worker     | A proxy that runs in a separate thread, intercepts your network requests, and can respond from cache — but only after the page that registered it has been reloaded (because the SW can’t control the initial load that registers it). |
| Install phase      | Your one chance to set up before taking control — if it fails, the whole SW is discarded.                                                                                                                                              |
| Waiting phase      | The new SW knows it's better but waits patiently while the old one finishes its shift (unless you force activation).                                                                                                                   |
| Activate phase     | The new SW's first act in power is cleaning up the previous regime's mess.                                                                                                                                                             |
| Cache Storage      | A programmatic cache you fully control — but the browser will never consult it for requests unless your fetch handler does.                                                                                                            |
| Response cloning   | A network response body is a pipe, not a bucket — clone it before you try to use it twice.                                                                                                                                             |
| Workbox            | The part that knows your asset filenames change every build and handles it so you don't have to.                                                                                                                                       |
| PWA installability | The browser deciding your app is self-sufficient enough to live outside a tab.                                                                                                                                                         |
| PWA installation   | The browser creating an app entry/shortcut to a URL and hiding the fact that it's still a browser.                                                                                                                                     |
