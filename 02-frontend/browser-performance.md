# 🌐 Browser Internals & Web Performance

> To make a page fast you must know what the browser is doing between "user clicks a link" and "user can interact." This book walks that path and shows where to intervene at each step.

---

## 📑 Table of Contents

1. [The Journey of a Request](#1-the-journey-of-a-request)
2. [Browser Architecture](#2-browser-architecture)
3. [The Critical Rendering Path](#3-the-critical-rendering-path)
4. [The Pixel Pipeline](#4-the-pixel-pipeline)
5. [Core Web Vitals](#5-core-web-vitals)
6. [Network Optimization](#6-network-optimization)
7. [Caching](#7-caching)
8. [JavaScript Performance](#8-javascript-performance)
9. [Rendering Performance](#9-rendering-performance)
10. [Image and Font Optimization](#10-image-and-font-optimization)
11. [Web Workers & Off-Main-Thread](#11-web-workers)
12. [Storage APIs](#12-storage-apis)
13. [Measurement](#13-measurement)
14. [Performance Budget Playbook](#14-performance-budget-playbook)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. The Journey of a Request

🎤 *"What happens when you type a URL and press Enter?"* — the most common systems question in frontend interviews. Here is the full answer.

```
 1. URL PARSING
    Scheme, host, port, path, query, fragment.
    Browser checks HSTS preload list → may force https:// before any request.

 2. DNS RESOLUTION                                        ~0-120 ms
    browser cache → OS cache → hosts file → router →
    ISP resolver → root (.) → TLD (.com) → authoritative NS
    Optimization: <link rel="dns-prefetch"> , DNS over HTTPS

 3. TCP HANDSHAKE                                         1 RTT
    SYN → SYN-ACK → ACK
    (QUIC/HTTP3 fuses this with TLS → 0-1 RTT)

 4. TLS HANDSHAKE                                         1-2 RTT
    ClientHello (SNI, ALPN, cipher list)
    ServerHello + certificate
    Key exchange (ECDHE) → session keys
    TLS 1.3 = 1 RTT; resumption/0-RTT = 0 RTT
    Optimization: <link rel="preconnect">  (does DNS+TCP+TLS early)

 5. HTTP REQUEST
    GET / HTTP/2 with headers + cookies

 6. SERVER PROCESSING                                     TTFB dominated here
    Routing, DB queries, render, cache lookup

 7. RESPONSE STREAMS BACK
    Browser starts parsing before the full document arrives.
    The PRELOAD SCANNER runs ahead of the parser, discovering
    <img>, <script>, <link> and starting those fetches early.

 8. PARSING → DOM + CSSOM → RENDER TREE → LAYOUT → PAINT → COMPOSITE

 9. SUBRESOURCES fetched, JS executes, hydration, interactivity
```

**Where the time actually goes** on a typical page:

```
   DNS+TCP+TLS  ████                        ~15%
   TTFB         ████████                    ~25%   ← server + network
   Download     ███                         ~10%
   Parse/Exec   ██████████████              ~40%   ← usually JavaScript
   Render       ███                         ~10%
```

The single biggest lever on most sites is *shipping less JavaScript*.

---

## 2. Browser Architecture

Modern browsers are multi-process for security and stability:

```
   ┌──────────────────────────────────────────────────────────┐
   │                     BROWSER PROCESS                      │
   │        UI, address bar, tabs, network, storage           │
   └───────┬──────────────────┬──────────────────┬────────────┘
           │                  │                  │
   ┌───────▼──────┐   ┌───────▼──────┐   ┌───────▼──────┐
   │  RENDERER    │   │  RENDERER    │   │     GPU      │
   │  (per site,  │   │              │   │   process    │
   │  sandboxed)  │   │              │   │              │
   │              │   │              │   └──────────────┘
   │ ┌──────────┐ │   └──────────────┘   ┌──────────────┐
   │ │  MAIN    │ │                      │   NETWORK    │
   │ │  THREAD  │ │  ← your JS, DOM,     │   process    │
   │ │          │ │    layout, paint     └──────────────┘
   │ ├──────────┤ │                      ┌──────────────┐
   │ │Compositor│ │  ← scroll, transform │   UTILITY    │
   │ │ thread   │ │    WITHOUT main      │  (audio,     │
   │ ├──────────┤ │                      │   plugins)   │
   │ │  Raster  │ │                      └──────────────┘
   │ │ threads  │ │
   │ ├──────────┤ │
   │ │ Workers  │ │
   │ └──────────┘ │
   └──────────────┘
```

⚙️ **The key insight:** the *main thread* does JavaScript, DOM, style, layout, and paint. If JS blocks it, nothing else on that list can happen — the page freezes. The *compositor thread* can still scroll and animate `transform`/`opacity` independently, which is why those two properties are the golden path for animation.

Site isolation gives each origin its own renderer process, which is the primary defense against Spectre-style cross-origin data leaks.

---

## 3. The Critical Rendering Path

```
   HTML bytes
       │ tokenize + parse
       ▼
     ┌─────┐
     │ DOM │◀──── blocked by synchronous <script>
     └──┬──┘
        │
   CSS bytes                       JS
       │ parse                      │
       ▼                            │
    ┌───────┐                       │
    │ CSSOM │◀──── render blocking ─┤ script execution WAITS
    └───┬───┘                       │ for CSSOM (JS can read styles)
        │                           │
        └──────────┬────────────────┘
                   ▼
            ┌─────────────┐
            │ RENDER TREE │   visible nodes only
            │             │   (no head, no display:none)
            └──────┬──────┘
                   ▼
              ┌────────┐   geometry: position + size of every box
              │ LAYOUT │   ("reflow")
              └───┬────┘
                  ▼
              ┌───────┐    fill in pixels into layers
              │ PAINT │
              └───┬───┘
                  ▼
            ┌────────────┐  assemble layers on the GPU
            │ COMPOSITE  │
            └────────────┘
```

### 3.1 Render-Blocking Resources

| Resource | Blocks parsing? | Blocks rendering? |
|---|---|---|
| `<script src>` | ✅ | ✅ |
| `<script src defer>` | ❌ | ❌ (runs before `DOMContentLoaded`, in order) |
| `<script src async>` | ❌ | ⚠️ can interrupt (runs whenever it lands) |
| `<script type="module">` | ❌ (deferred by default) | ❌ |
| `<link rel="stylesheet">` | ❌ | ✅ |
| `<link media="print">` | ❌ | ❌ |
| Inline `<script>` | ✅ | ✅ (and waits for pending CSSOM) |

```html
<!-- ✅ The canonical fast head -->
<head>
  <meta charset="utf-8">
  <link rel="preconnect" href="https://api.example.com" crossorigin>
  <style>/* critical above-the-fold CSS inlined */</style>
  <link rel="preload" as="font" type="font/woff2" href="/f.woff2" crossorigin>
  <link rel="stylesheet" href="/rest.css" media="print" onload="this.media='all'">
  <script src="/app.js" defer></script>
</head>
```

### 3.2 Resource Hints

```html
<link rel="dns-prefetch" href="//cdn.com">        <!-- DNS only,       ~20ms saved -->
<link rel="preconnect"  href="//cdn.com" crossorigin>  <!-- DNS+TCP+TLS, ~100-300ms -->
<link rel="preload" as="font" href="/f.woff2" crossorigin>  <!-- fetch NOW, high priority -->
<link rel="prefetch" href="/next-page.js">        <!-- fetch for a FUTURE navigation, idle -->
<link rel="modulepreload" href="/mod.js">         <!-- preload + parse an ES module -->
```

⚠️ `preload` without using the resource within ~3 seconds triggers a console warning and wastes bandwidth. Use it only for resources the initial render genuinely needs and that the preload scanner cannot discover (fonts referenced from CSS, dynamically imported LCP images).

---

## 4. The Pixel Pipeline

Every visual change goes through some suffix of this pipeline. Doing less means being faster.

```
   JS/CSS  ──▶ Style ──▶ Layout ──▶ Paint ──▶ Composite
                            │         │          │
   ┌────────────────────────┴─────────┴──────────┴────────┐
   │ Changing `width`, `top`, `font-size`                 │
   │   → Style + Layout + Paint + Composite   💸 EXPENSIVE│
   ├──────────────────────────────────────────────────────┤
   │ Changing `background-color`, `box-shadow`, `color`   │
   │   → Style + Paint + Composite            💰 MEDIUM   │
   ├──────────────────────────────────────────────────────┤
   │ Changing `transform`, `opacity`, `filter`            │
   │   → Composite ONLY                       ✅ CHEAP     │
   │   Runs on the COMPOSITOR THREAD — no main thread!    │
   └──────────────────────────────────────────────────────┘
```

**Animate only `transform` and `opacity`.** Everything else risks jank.

```css
/* ❌ triggers layout on every frame */
.slide { transition: left 300ms; }
.slide.active { left: 100px; }

/* ✅ compositor-only */
.slide { transition: transform 300ms; will-change: transform; }
.slide.active { transform: translateX(100px); }
```

⚠️ `will-change` promotes an element to its own compositor layer, which costs GPU memory. Apply it just before the animation and remove it after; never put it on hundreds of elements or in a global rule.

### 4.1 Layout Thrashing

```js
// ❌ Forced synchronous layout — reflow on EVERY iteration
boxes.forEach(box => {
  box.style.width = box.offsetWidth + 10 + 'px';   // write → read → write → ...
});

// ✅ Batch: read all, then write all → one reflow
const widths = boxes.map(b => b.offsetWidth);       // READ phase
boxes.forEach((b, i) => b.style.width = widths[i] + 10 + 'px');  // WRITE phase
```

**Properties that force synchronous layout when read after a write:**

```
   offsetTop/Left/Width/Height    scrollTop/Left/Width/Height
   clientTop/Left/Width/Height    getComputedStyle()
   getBoundingClientRect()        getClientRects()
   window.scrollX/Y               element.focus()
```

`requestAnimationFrame` is the right place for coordinated read-then-write work.

---

## 5. Core Web Vitals

```
   ┌──────────────┬────────────┬──────────────┬────────────────────┐
   │ Metric       │  Good      │  Needs work  │  Poor              │
   ├──────────────┼────────────┼──────────────┼────────────────────┤
   │ LCP  loading │  ≤ 2.5 s   │  ≤ 4.0 s     │  > 4.0 s           │
   │ INP  interact│  ≤ 200 ms  │  ≤ 500 ms    │  > 500 ms          │
   │ CLS  stability│ ≤ 0.1     │  ≤ 0.25      │  > 0.25            │
   ├──────────────┼────────────┼──────────────┼────────────────────┤
   │ TTFB         │  ≤ 800 ms  │  ≤ 1800 ms   │  > 1800 ms         │
   │ FCP          │  ≤ 1.8 s   │  ≤ 3.0 s     │  > 3.0 s           │
   └──────────────┴────────────┴──────────────┴────────────────────┘

   Scored at the 75th percentile of real users, per device class.
```

### 5.1 LCP — Largest Contentful Paint

The render time of the largest text block or image in the viewport.

```
   LCP breaks down into four parts:

   [ TTFB ][ Resource load delay ][ Resource load time ][ Render delay ]
      │            │                      │                    │
   server      discovery              download            main thread
   + network   was late              too big              busy / blocked
```

| Sub-part | Fix |
|---|---|
| TTFB | CDN, caching, faster server, streaming |
| Load delay | `fetchpriority="high"`, preload, get it out of CSS/JS discovery |
| Load time | Modern formats (AVIF/WebP), correct `sizes`, compression |
| Render delay | Less blocking JS/CSS, avoid client-side-only rendering of the hero |

```html
<img src="hero.avif" fetchpriority="high" alt="" width="1200" height="600">
<!-- NEVER lazy-load the LCP image -->
```

### 5.2 INP — Interaction to Next Paint

Replaced FID in March 2024. Measures the **full** latency of the worst interaction: from input to the next frame painted.

```
   Click                                                    Paint
     │                                                        │
     ├──── Input delay ────┼──── Processing ────┼── Presentation ──┤
           (main thread          (your event         (style, layout,
            busy with            handlers run)        paint, composite)
            other work)
```

**Fixes by sub-part:**

```js
// 1. Input delay — break up long tasks so the handler can start
async function processAll(items) {
  for (const chunk of chunks(items, 50)) {
    process(chunk);
    await scheduler.yield();        // or: await new Promise(r => setTimeout(r));
  }
}

// 2. Processing — do the minimum for visual feedback, defer the rest
button.onclick = () => {
  showSpinner();                             // immediate feedback
  requestIdleCallback(() => heavyWork());    // defer the rest
};

// 3. Presentation — yield BEFORE the expensive DOM update so
//    the browser can paint the feedback first
button.onclick = async () => {
  setActive(true);
  await new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r)));
  renderBigList();
};
```

⚠️ **Any task over 50 ms is a "long task"** and directly threatens INP. Third-party scripts, hydration, and large `JSON.parse` calls are the usual culprits.

### 5.3 CLS — Cumulative Layout Shift

```
   Layout shift score = impact fraction × distance fraction
   CLS = sum of the worst 5-second session window
```

| Cause | Fix |
|---|---|
| Images without dimensions | Always set `width`/`height` or `aspect-ratio` |
| Ads/embeds/iframes | Reserve space with a min-height container |
| Web fonts (FOUT/FOIT) | `font-display: optional` or `size-adjust` metric overrides |
| Content injected above existing content | Reserve space, or insert below the fold |
| Animating `top`/`height` | Animate `transform` instead |

```css
img, video { aspect-ratio: attr(width) / attr(height); max-width: 100%; height: auto; }
.ad-slot { min-height: 250px; contain: layout; }

@font-face {
  font-family: 'Inter'; src: url(/inter.woff2) format('woff2');
  font-display: swap;
  size-adjust: 107%;        /* match the fallback's metrics → near-zero shift */
  ascent-override: 90%;
}
```

Note: shifts within 500 ms of a user interaction are excluded — the metric targets *unexpected* movement.

---

## 6. Network Optimization

### 6.1 Protocol Evolution

```
   HTTP/1.1  ┌──────────────────────────────────────────────┐
             │ 1 request per connection at a time            │
             │ Head-of-line blocking                         │
             │ Browsers open ~6 connections per host         │
             │ → domain sharding was a real optimization     │
             └──────────────────────────────────────────────┘

   HTTP/2    ┌──────────────────────────────────────────────┐
             │ Multiplexed streams over ONE TCP connection   │
             │ Binary framing, header compression (HPACK)    │
             │ Server push (deprecated — rarely helped)      │
             │ ⚠️ Still TCP head-of-line blocking on loss     │
             │ → domain sharding now HURTS                   │
             └──────────────────────────────────────────────┘

   HTTP/3    ┌──────────────────────────────────────────────┐
             │ QUIC over UDP                                 │
             │ Per-stream flow control — packet loss on one  │
             │ stream doesn't stall the others               │
             │ 0-1 RTT handshake, connection migration       │
             │ (survives WiFi → cellular switch)             │
             └──────────────────────────────────────────────┘
```

### 6.2 Compression

| Method | Ratio (text) | CPU | Use |
|---|---|---|---|
| gzip | ~70% | Low | Universal fallback |
| brotli (q=11) | ~75-80% | High to compress | **Static assets, precompressed at build** |
| brotli (q=4-5) | ~72% | Low | Dynamic responses |
| zstd | ~76% | Very low | Increasingly supported |

⚠️ Never compress already-compressed formats (jpg, png, webp, mp4, woff2) — you burn CPU and can make them larger.

### 6.3 Bundle Optimization

```js
// Route-level splitting
const Dashboard = lazy(() => import('./Dashboard'));

// Component-level for heavy, rarely-used widgets
const Editor = lazy(() => import('./RichTextEditor'));

// Interaction-triggered — load on intent
button.addEventListener('mouseenter', () => import('./Modal'), { once: true });

// Tree shaking requires ESM + no side effects
import { debounce } from 'lodash-es';    // ✅ shakeable
import _ from 'lodash';                  // ❌ pulls in everything
```

```json
// package.json — tells bundlers files are side-effect free
{ "sideEffects": false }
```

**Bundle budget for a typical content site:**

```
   HTML          ≤  15 KB
   CSS           ≤  25 KB   (critical inlined)
   JS (initial)  ≤ 150 KB   compressed  ← the number that matters
   Images (LCP)  ≤ 150 KB
   Fonts         ≤ 100 KB   (2 weights, woff2, subset)
   ─────────────────────────
   Total initial ≤ 500 KB
```

---

## 7. Caching

### 7.1 HTTP Cache Headers

```
   Cache-Control: public, max-age=31536000, immutable
                  │       │                 │
                  │       │                 └── never revalidate; content-hashed
                  │       └── seconds fresh
                  └── any cache may store (vs private = browser only)

   Cache-Control: no-cache          → store, but revalidate every time
   Cache-Control: no-store          → never store (sensitive data)
   Cache-Control: max-age=0, must-revalidate
   Cache-Control: s-maxage=600      → CDN-specific TTL (overrides max-age)
   Cache-Control: stale-while-revalidate=86400  → serve stale, refresh in background
```

### 7.2 The Two-Tier Strategy

```
   ┌──────────────────────────────────────────────────────────┐
   │ HTML                                                     │
   │   Cache-Control: no-cache                                │
   │   → always revalidated, so deploys take effect instantly │
   ├──────────────────────────────────────────────────────────┤
   │ Hashed assets   app.a1b2c3.js                            │
   │   Cache-Control: public, max-age=31536000, immutable     │
   │   → cached forever; a new deploy = a new filename        │
   └──────────────────────────────────────────────────────────┘
```

This gives you instant deploys and permanent asset caching simultaneously.

### 7.3 Validators

```
   Request:  If-None-Match: "abc123"
   Response: 304 Not Modified          ← no body, saves bandwidth

   ETag: strong validator (content hash)
   Last-Modified: weak validator (1-second granularity)
```

### 7.4 Service Worker Strategies

```js
// Cache-first — for immutable assets
if (url.pathname.startsWith('/static/')) {
  e.respondWith(caches.match(e.request).then(r => r ?? fetch(e.request)));
}

// Network-first with cache fallback — for HTML
e.respondWith(
  fetch(e.request)
    .then(res => { cache.put(e.request, res.clone()); return res; })
    .catch(() => caches.match(e.request) ?? caches.match('/offline.html'))
);

// Stale-while-revalidate — best UX for semi-fresh data
e.respondWith(
  caches.match(e.request).then(cached => {
    const fresh = fetch(e.request).then(res => { cache.put(e.request, res.clone()); return res; });
    return cached ?? fresh;          // instant if cached, updates in background
  })
);
```

---

## 8. JavaScript Performance

### 8.1 The Cost of JavaScript

```
   200 KB of JavaScript on a mid-tier Android phone:

   Download (4G)   ████            ~1.0 s
   Parse           ██              ~0.3 s
   Compile         ███             ~0.4 s
   Execute         ███████         ~1.0 s
   ──────────────────────────────────────
   Total           ~2.7 s of main-thread time

   The same 200 KB of images: download only, decoded off-thread.
   → Byte for byte, JavaScript is the most expensive resource you ship.
```

### 8.2 Breaking Up Long Tasks

```js
// The modern API
if ('scheduler' in window && 'yield' in scheduler) {
  await scheduler.yield();
}

// scheduler.postTask with priorities
scheduler.postTask(() => importantWork(), { priority: 'user-blocking' });
scheduler.postTask(() => analytics(),     { priority: 'background' });

// Universal fallback
const yieldToMain = () => new Promise(r => setTimeout(r, 0));

async function processLargeArray(items) {
  let lastYield = performance.now();
  for (const item of items) {
    process(item);
    if (performance.now() - lastYield > 45) {   // stay under the 50ms threshold
      await yieldToMain();
      lastYield = performance.now();
    }
  }
}
```

### 8.3 Efficient DOM Work

```js
// ❌ n reflows
items.forEach(i => list.appendChild(createRow(i)));

// ✅ 1 reflow — DocumentFragment is not in the live tree
const frag = document.createDocumentFragment();
items.forEach(i => frag.appendChild(createRow(i)));
list.appendChild(frag);

// ✅ Event delegation — 1 listener instead of n
list.addEventListener('click', e => {
  const row = e.target.closest('[data-id]');
  if (row) handle(row.dataset.id);
});

// ✅ Passive listeners — tells the browser you won't preventDefault,
//    so scrolling doesn't wait for your handler
window.addEventListener('scroll', onScroll, { passive: true });

// ✅ IntersectionObserver instead of scroll math
new IntersectionObserver(entries => {
  entries.forEach(e => e.isIntersecting && load(e.target));
}, { rootMargin: '200px' }).observe(el);

// ✅ CSS containment — scope layout/paint work to a subtree
// .card { contain: content; }         /* layout + paint + style */
// .list { content-visibility: auto; contain-intrinsic-size: 0 500px; }
```

`content-visibility: auto` is remarkably effective: the browser skips rendering work entirely for off-screen elements, often cutting initial render time by half on long pages.

---

## 9. Rendering Performance

### 9.1 The Frame Budget

```
   60 fps → 16.67 ms per frame

   ┌───────────────────────────────────────────────┐
   │ Your JS  │  Style │ Layout │ Paint │ Composite│
   │  ~8 ms   │  2 ms  │  2 ms  │ 2 ms  │  2 ms    │
   └───────────────────────────────────────────────┘
              ↑ browser housekeeping needs ~6ms,
                so your budget is realistically ~10ms
```

Miss the budget and the frame is dropped — that's jank.

### 9.2 Scroll Performance Checklist

```
✅ passive: true on scroll/touch listeners
✅ transform/opacity only in scroll-linked animations
✅ IntersectionObserver instead of scroll handlers where possible
✅ Virtualize lists over ~100 items
✅ content-visibility: auto on long pages
✅ Debounce/throttle expensive scroll work
❌ No getBoundingClientRect() in a scroll handler
❌ No position: fixed with box-shadow over scrolling content
❌ No unnecessary will-change (each layer costs GPU memory)
```

---

## 10. Image and Font Optimization

### 10.1 Images

```html
<!-- Modern formats with fallback + responsive sizing -->
<picture>
  <source type="image/avif" srcset="hero-400.avif 400w, hero-800.avif 800w" sizes="(max-width:600px) 100vw, 50vw">
  <source type="image/webp" srcset="hero-400.webp 400w, hero-800.webp 800w" sizes="(max-width:600px) 100vw, 50vw">
  <img src="hero-800.jpg" alt="…" width="800" height="450"
       loading="lazy" decoding="async" fetchpriority="auto">
</picture>
```

| Format | Size vs JPEG | Support | Use |
|---|---|---|---|
| AVIF | -50% | Good | Best compression; slower encode |
| WebP | -30% | Universal | Safe default |
| JPEG | baseline | Universal | Fallback |
| PNG | large | Universal | Only when lossless/transparency needed |
| SVG | tiny | Universal | Icons, logos, diagrams |

⚠️ `loading="lazy"` on the LCP image *delays* it and hurts your score. Lazy-load below-the-fold only.

### 10.2 Fonts

```css
@font-face {
  font-family: 'Inter';
  src: url('/inter-var.woff2') format('woff2-variations');
  font-weight: 100 900;              /* one file, all weights */
  font-display: swap;
  unicode-range: U+0000-00FF;        /* subset: Latin only */
}
```

| `font-display` | Block period | Swap period | Result |
|---|---|---|---|
| `auto` | ~3 s | — | Browser default (usually block) |
| `block` | 3 s | infinite | FOIT — invisible text, bad |
| `swap` | 0 | infinite | FOUT — fallback shows, then swaps (CLS risk) |
| `fallback` | 100 ms | 3 s | Compromise |
| `optional` | 100 ms | 0 | **Zero CLS** — uses the font only if nearly instant |

**Best practice stack:** self-host, woff2 only, subset to needed glyphs, variable font when using 3+ weights, `preload` the one critical face, and use `size-adjust`/`ascent-override` to match fallback metrics so the swap causes no shift.

---

## 11. Web Workers

```js
// main.js
const worker = new Worker('/worker.js', { type: 'module' });
worker.postMessage({ cmd: 'parse', payload: bigString });
worker.onmessage = (e) => render(e.data);

// Transferable objects — zero-copy ownership transfer
worker.postMessage(arrayBuffer, [arrayBuffer]);   // buffer is now unusable on main
```

| Worker type | Purpose |
|---|---|
| **Dedicated Worker** | CPU work off the main thread — parsing, crypto, image processing |
| **Shared Worker** | One instance shared across tabs of the same origin |
| **Service Worker** | Network proxy — offline, caching, push, background sync |
| **Worklets** | Tiny, high-performance hooks into rendering/audio pipelines |

Workers cannot touch the DOM. Structured clone is used for messages, so large objects have a serialization cost — use `ArrayBuffer` transfers or `SharedArrayBuffer` for big data.

Good candidates: JSON parsing over ~100 KB, search/filter over large datasets, image manipulation, syntax highlighting, diffing, encryption, and PDF generation.

---

## 12. Storage APIs

| API | Capacity | Sync? | Persistence | Use |
|---|---|---|---|---|
| `localStorage` | ~5-10 MB | **Sync (blocks!)** | Until cleared | Small prefs only |
| `sessionStorage` | ~5-10 MB | Sync | Per tab | Per-session UI state |
| **IndexedDB** | ~50%+ of disk | Async | Until cleared | Structured/large data |
| Cache Storage | Large | Async | Until cleared | HTTP responses (SW) |
| Cookies | 4 KB | Sync | Configurable | Auth tokens (HttpOnly) |
| OPFS | Large | Async | Until cleared | High-performance file I/O |

⚠️ **`localStorage` is synchronous and blocks the main thread.** Reading 1 MB of JSON from it on startup is a common, invisible cause of poor INP and slow FCP. Use IndexedDB (via `idb-keyval` for a simple API) for anything non-trivial.

```js
navigator.storage.persist();                    // request durable storage
const { usage, quota } = await navigator.storage.estimate();
```

---

## 13. Measurement

### 13.1 Lab vs Field

| | Lab (synthetic) | Field (RUM) |
|---|---|---|
| Tools | Lighthouse, WebPageTest, DevTools | CrUX, `web-vitals` + your analytics |
| Pros | Reproducible, debuggable, pre-deploy | Real devices, networks, users |
| Cons | One device/network; misses reality | Aggregate, harder to debug |
| Use | Catch regressions in CI | Know what users experience |

You need both. Lab tells you *why*; field tells you *whether it matters*.

### 13.2 Collecting Real Metrics

```js
import { onLCP, onINP, onCLS, onTTFB, onFCP } from 'web-vitals';

function send(metric) {
  navigator.sendBeacon('/analytics', JSON.stringify({
    name: metric.name,
    value: metric.value,
    rating: metric.rating,                  // good | needs-improvement | poor
    id: metric.id,
    navigationType: metric.navigationType,
    attribution: metric.attribution,        // ⭐ which element/script caused it
  }));
}

onLCP(send); onINP(send); onCLS(send); onTTFB(send); onFCP(send);
```

The `attribution` field is the difference between "our LCP is 4 s" and "our LCP is 4 s because of the hero image on the product page, delayed by the analytics script."

### 13.3 Performance APIs

```js
// Long tasks
new PerformanceObserver(list => {
  for (const e of list.getEntries()) {
    console.warn('Long task', e.duration, e.attribution);
  }
}).observe({ type: 'longtask', buffered: true });

// Navigation timing breakdown
const [nav] = performance.getEntriesByType('navigation');
console.table({
  dns: nav.domainLookupEnd - nav.domainLookupStart,
  tcp: nav.connectEnd - nav.connectStart,
  tls: nav.secureConnectionStart ? nav.connectEnd - nav.secureConnectionStart : 0,
  ttfb: nav.responseStart - nav.requestStart,
  download: nav.responseEnd - nav.responseStart,
  domInteractive: nav.domInteractive,
  loadComplete: nav.loadEventEnd,
});

// Custom marks
performance.mark('feature-start');
performance.measure('feature', 'feature-start');
```

---

## 14. Performance Budget Playbook

```
   ┌─── STEP 1: MEASURE ─────────────────────────────────┐
   │ Field data first (CrUX / RUM). Find the worst        │
   │ metric at p75, and which pages/devices drive it.     │
   ├─── STEP 2: DIAGNOSE ────────────────────────────────┤
   │ LCP bad?  → break into TTFB / delay / load / render  │
   │ INP bad?  → find the long tasks, check third parties │
   │ CLS bad?  → DevTools "Layout Shift Regions" overlay  │
   ├─── STEP 3: FIX THE BIGGEST ONE ─────────────────────┤
   │ Usually: too much JS, unoptimized LCP image, or a    │
   │ third-party script blocking the main thread.         │
   ├─── STEP 4: PREVENT REGRESSION ──────────────────────┤
   │ CI budget: fail the build if initial JS > 150 KB     │
   │ Lighthouse CI on every PR                            │
   │ Alert on p75 field metrics                           │
   └─────────────────────────────────────────────────────┘
```

```json
// lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance": ["error", { "minScore": 0.9 }],
        "largest-contentful-paint": ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift": ["error", { "maxNumericValue": 0.1 }],
        "total-byte-weight": ["error", { "maxNumericValue": 500000 }]
      }
    }
  }
}
```

### Third-party scripts

Usually the largest single win available.

```html
<!-- Load after the page is interactive -->
<script>
  addEventListener('load', () => {
    const s = document.createElement('script');
    s.src = 'https://analytics.example.com/t.js'; s.async = true;
    document.body.append(s);
  });
</script>
```

Better still: move tag managers and analytics into a web worker with Partytown, or replace client-side analytics with server-side collection entirely.

---

## 15. Interview Section

<details>
<summary><b>Q1. What happens when you type a URL and hit Enter?</b></summary>

Nine stages. The browser parses the URL and checks HSTS. DNS resolves the host through browser, OS, and resolver caches before hitting the hierarchy. TCP handshakes in one round trip, then TLS in one or two more — HTTP/3 fuses these. The request goes out; the server's processing time dominates TTFB.

The response streams back, and crucially the browser starts parsing before it's complete. The preload scanner runs ahead of the main parser to start subresource fetches early. HTML becomes the DOM, CSS becomes the CSSOM — CSS is render-blocking and also blocks any script that follows it, because scripts can query computed styles.

DOM plus CSSOM produce the render tree, which goes through layout, paint, and composite. Then subresources load, JavaScript executes, and the page becomes interactive.

The interesting follow-up is where you'd optimize: `preconnect` for the handshakes, `defer` for scripts, inlined critical CSS, and shipping less JavaScript, which is usually the dominant cost.
</details>

<details>
<summary><b>Q2. Explain the Core Web Vitals and how you'd fix each.</b></summary>

LCP measures loading — when the largest element in the viewport renders. Fix by breaking it into TTFB, resource discovery delay, download time, and render delay, then attacking whichever dominates. Usually it's an unoptimized hero image or a client-rendered hero.

INP measures responsiveness — the worst interaction's full latency from input to next paint. Fix by breaking up long tasks, yielding to the main thread, doing minimum work in handlers, and auditing third-party scripts.

CLS measures visual stability. Fix by always reserving space: dimensions on images, min-height on ad slots, and font-display strategies with metric overrides.

They're measured at p75 of real users, so lab scores can look fine while field data is poor — usually because of slow devices you never test on.
</details>

<details>
<summary><b>Q3. `async` vs `defer` vs neither.</b></summary>

No attribute: the parser stops, fetches, executes, then resumes. Blocking and usually wrong.

`async`: fetched in parallel, executed as soon as it lands — which can interrupt parsing at an unpredictable point, and execution order between multiple async scripts isn't guaranteed. Use for independent scripts like analytics.

`defer`: fetched in parallel, executed after parsing completes, in document order, before `DOMContentLoaded`. This is the right default for application code.

Module scripts are deferred by default. `async` on a module behaves like async.
</details>

<details>
<summary><b>Q4. What is layout thrashing?</b></summary>

Forcing the browser to recompute layout synchronously many times in one frame. It happens when you interleave DOM writes and reads of layout-dependent properties — `offsetWidth`, `getBoundingClientRect`, `scrollTop`.

The browser normally batches style and layout work, but reading a layout property requires an up-to-date answer, so a pending write must be flushed first. In a loop that means n reflows instead of one.

The fix is batching: read everything into variables, then write everything. `requestAnimationFrame` is a good place to coordinate that, and `FastDom`-style libraries automate it.
</details>

<details>
<summary><b>Q5. Why animate only `transform` and `opacity`?</b></summary>

Because they're handled entirely by the compositor. Changing them skips style, layout, and paint, and they can be applied on the compositor thread — so the animation keeps running smoothly even if the main thread is busy with JavaScript.

Changing geometric properties like `width` or `left` triggers layout, which cascades into paint and composite, on the main thread, every frame. At 60 fps you have about 10 ms of budget, and layout on a complex page can exceed that alone.

You promote an element with `will-change: transform` or a 3D transform, but each layer costs GPU memory, so it's not free and shouldn't be applied broadly.
</details>

<details>
<summary><b>Q6. How would you make a site with 10,000 list items fast?</b></summary>

Virtualize — render only the visible window plus a small overscan, absolutely positioned inside a container sized to the full scroll height. That caps DOM nodes at roughly 20 regardless of dataset size, which fixes memory, layout time, and scroll performance simultaneously.

Supporting measures: `content-visibility: auto` if virtualization isn't feasible, stable keys so the framework moves nodes instead of rebuilding them, event delegation instead of per-row listeners, passive scroll listeners, and moving any filtering or sorting into a `useMemo` or a worker.

If the data itself is large, paginate or use infinite scroll so you're not shipping 10,000 records to the client at all.
</details>

<details>
<summary><b>Q7. Explain the browser cache and how you'd design a caching strategy.</b></summary>

Two tiers. HTML gets `no-cache`, meaning it's stored but revalidated on every request — so deploys take effect immediately, and a 304 costs almost nothing. Static assets get content-hashed filenames plus `max-age=31536000, immutable`, so they're cached permanently and a new deploy simply produces a new URL.

That combination gives instant deploys and maximum cache hit rates at the same time, with no cache-busting query strings.

Beyond that: `s-maxage` for CDN-specific TTLs, `stale-while-revalidate` to serve instantly while refreshing in the background, and ETags for cheap revalidation. A service worker adds offline support and lets you implement stale-while-revalidate for navigations too.
</details>

<details>
<summary><b>Q8. What is the preload scanner?</b></summary>

A secondary, lightweight parser that scans ahead in the raw HTML while the main parser is blocked — typically by a synchronous script. It finds `<img>`, `<script>`, `<link>`, and starts those downloads early.

It's why blocking scripts aren't as catastrophic as they'd otherwise be, and it's also why resources it *can't* see are late: images set via CSS `background-image`, fonts referenced from within stylesheets, or anything injected by JavaScript. Those are exactly the cases where `preload` earns its cost.
</details>

<details>
<summary><b>Q9. How do you diagnose a slow page you've never seen?</b></summary>

Start with field data if it exists — CrUX or RUM — to learn which metric is bad, on which pages, and on which device class. Lab-only debugging on a fast laptop misses most real problems.

Then reproduce in the lab with throttling matched to the affected users: 4× CPU slowdown and Slow 4G. Run Lighthouse for the summary, then the DevTools Performance panel for the detail — the main-thread flamechart shows long tasks and what's in them, and the network waterfall shows discovery delays and blocking chains.

Then it's a triage: is it too many bytes, too much main-thread work, or too many round trips? The fix follows from which. And I'd check third-party scripts early, since they're frequently the single largest contributor and the easiest to defer.
</details>

<details>
<summary><b>Q10. `localStorage` vs IndexedDB vs cookies.</b></summary>

Cookies are sent with every matching request, so they're limited to 4 KB and belong to auth tokens — with `HttpOnly`, `Secure`, and `SameSite` set.

`localStorage` is a simple synchronous string key-value store, around 5-10 MB. The synchronous part matters: reading a large blob blocks the main thread and quietly damages INP and startup time. It's fine for a theme preference, wrong for application data.

IndexedDB is asynchronous, transactional, indexed, and can hold hundreds of megabytes. Its raw API is unpleasant, so most people use a wrapper like `idb`. It's the right choice for any meaningful client-side data, offline support, or caching structured records.
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              BROWSER & WEB PERFORMANCE — ONE PAGE                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ PATH: URL→DNS→TCP→TLS→HTTP→parse→DOM+CSSOM→render tree→             ║
║       layout→paint→composite                                         ║
║ MAIN THREAD does JS+DOM+style+layout+paint. Block it = frozen page.  ║
╠══════════════════════════════════════════════════════════════════════╣
║ CORE WEB VITALS (p75 of real users)                                  ║
║   LCP ≤2.5s   INP ≤200ms   CLS ≤0.1   TTFB ≤800ms                    ║
║   LCP = TTFB + load delay + load time + render delay                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ PIXEL PIPELINE                                                       ║
║   width/top/font-size → Layout+Paint+Composite  💸                   ║
║   color/box-shadow    → Paint+Composite         💰                   ║
║   transform/opacity   → Composite ONLY          ✅  ANIMATE THESE     ║
╠══════════════════════════════════════════════════════════════════════╣
║ SCRIPTS: defer = parallel fetch, ordered, after parse ← DEFAULT      ║
║          async = parallel fetch, runs whenever ← analytics only      ║
║          neither = blocks parsing ← avoid                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ HINTS: dns-prefetch < preconnect < preload < prefetch(future nav)    ║
╠══════════════════════════════════════════════════════════════════════╣
║ CACHE: HTML → no-cache                                               ║
║        hashed assets → max-age=31536000, immutable                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ BUDGET: initial JS ≤150KB compressed · total ≤500KB                  ║
║   JS is the most expensive byte you can ship (parse+compile+exec)    ║
╠══════════════════════════════════════════════════════════════════════╣
║ QUICK WINS: fetchpriority on LCP img · never lazy-load LCP           ║
║   width/height on all images · font-display:optional · defer 3rd     ║
║   party · passive listeners · content-visibility:auto · virtualize   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [React](react.md) · [Next.js](nextjs.md) · [CSS](css.md) · [JavaScript](../01-languages/javascript.md)
