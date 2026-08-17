# ▲ Next.js & Rendering Strategies

> Next.js is a rendering strategy toolkit. The whole framework is about answering one question per route: **where and when does this HTML get built?**

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Rendering Spectrum](#2-the-rendering-spectrum)
3. [App Router Architecture](#3-app-router-architecture)
4. [File Conventions](#4-file-conventions)
5. [Data Fetching and Caching](#5-data-fetching-and-caching)
6. [Server Actions](#6-server-actions)
7. [Streaming and Suspense](#7-streaming-and-suspense)
8. [Routing Patterns](#8-routing-patterns)
9. [Middleware and Edge](#9-middleware-and-edge)
10. [Images, Fonts, Metadata](#10-images-fonts-metadata)
11. [Authentication](#11-authentication)
12. [Performance](#12-performance)
13. [Deployment and Self-Hosting](#13-deployment)
14. [Interview Section](#14-interview-section)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. Mental Model

```
   For each route, answer three questions:

   1. WHEN is the HTML produced?
      build time · request time · never (client only)

   2. WHERE does the code run?
      build machine · Node server · edge runtime · browser

   3. HOW LONG is the result valid?
      forever · N seconds · until revalidated by tag/path · never cached
```

Everything in Next.js — `generateStaticParams`, `revalidate`, `cache`, `dynamic`, `use client` — is a knob on those three axes.

---

## 2. The Rendering Spectrum

```
 BUILD TIME ◀───────────────────────────────────────────────▶ REQUEST TIME
                                                        
 ┌─────────┐   ┌─────────┐   ┌──────────┐   ┌─────────┐   ┌──────────┐
 │   SSG   │   │   ISR   │   │Streaming │   │   SSR   │   │   CSR    │
 │         │   │         │   │   SSR    │   │         │   │          │
 │ Built   │   │ Built + │   │ Shell    │   │ Built   │   │ Empty    │
 │ once at │   │ rebuilt │   │ now,     │   │ per     │   │ shell +  │
 │ deploy  │   │ every N │   │ rest     │   │ request │   │ fetch in │
 │         │   │ seconds │   │ streams  │   │         │   │ browser  │
 └─────────┘   └─────────┘   └──────────┘   └─────────┘   └──────────┘
   Fastest      Fast +        Fast TTFB      Always        Worst TTFB
   Cheapest     fresh         + fresh        fresh         + SEO
   Stalest                                   Slowest TTFB
```

| Strategy | TTFB | Freshness | Server cost | SEO | Use for |
|---|---|---|---|---|---|
| **SSG** | ⚡ CDN | Stale until rebuild | ~0 | ✅ | Marketing, docs, blogs |
| **ISR** | ⚡ CDN | ≤ revalidate window | Low | ✅ | Product pages, articles |
| **SSR** | 🐢 | Always fresh | High | ✅ | Dashboards, personalized |
| **Streaming SSR** | ⚡ shell | Fresh | High | ✅ | Slow data + good UX |
| **CSR** | ⚡ shell, slow content | Fresh | ~0 | ❌ | Behind-login apps |
| **PPR** | ⚡ static shell | Dynamic holes fresh | Medium | ✅ | Mixed pages (the future default) |

### 2.1 Partial Prerendering (PPR)

The synthesis of static and dynamic in **one response**:

```
   ┌──────────────────────────────────────────┐
   │  STATIC SHELL — served instantly from CDN │
   │  ┌────────────────────────────────────┐  │
   │  │ Header, nav, layout, product info  │  │
   │  ├────────────────────────────────────┤  │
   │  │ ░░░ dynamic hole ░░░               │  │  streamed in
   │  │ <Suspense> user cart </Suspense>   │  │  at request time
   │  ├────────────────────────────────────┤  │
   │  │ Footer                             │  │
   │  └────────────────────────────────────┘  │
   └──────────────────────────────────────────┘
```

```jsx
export const experimental_ppr = true;

export default function Page() {
  return (
    <>
      <StaticHeader />               {/* prerendered into the shell */}
      <Suspense fallback={<CartSkeleton />}>
        <Cart />                     {/* dynamic hole, streamed */}
      </Suspense>
    </>
  );
}
```

---

## 3. App Router Architecture

```
   Request
      │
      ▼
   ┌──────────────┐
   │  middleware  │  runs on EVERY matching request, before cache
   └──────┬───────┘  auth redirects, rewrites, headers, A/B
          ▼
   ┌──────────────┐
   │ Route match  │
   └──────┬───────┘
          ▼
   ┌────────────────────────────────────────────┐
   │  layout.tsx (root)      ← persists across  │
   │   └ layout.tsx (nested)   navigation, does │
   │      └ template.tsx       NOT re-render    │
   │         └ page.tsx                         │
   │  wrapped by: error.tsx, loading.tsx,       │
   │              not-found.tsx                 │
   └────────────────────┬───────────────────────┘
                        ▼
              RSC payload + HTML stream
                        ▼
                  Client hydrates
                  only 'use client' islands
```

**Layouts persist. Templates remount.**

```
   Navigate /dashboard/a → /dashboard/b

   layout.tsx    ── state PRESERVED, effects NOT re-run
   template.tsx  ── remounted, state reset, effects re-run
   page.tsx      ── re-rendered
```

Use `template.tsx` when you need enter/exit animations or per-navigation state resets.

---

## 4. File Conventions

```
app/
├── layout.tsx              # root layout — must render <html> and <body>
├── page.tsx                # route: /
├── loading.tsx             # auto-wraps page in <Suspense>
├── error.tsx               # 'use client' — error boundary for this segment
├── global-error.tsx        # catches root layout errors
├── not-found.tsx           # 404 for notFound()
├── template.tsx            # like layout, but remounts every navigation
├── route.ts                # API endpoint (cannot coexist with page.tsx)
├── default.tsx             # fallback for parallel routes
│
├── (marketing)/            # ROUTE GROUP — no URL segment
│   ├── layout.tsx          #   separate layout for this group
│   ├── about/page.tsx      #   → /about
│   └── pricing/page.tsx    #   → /pricing
│
├── blog/
│   ├── [slug]/page.tsx     # dynamic:      /blog/hello
│   ├── [...tags]/page.tsx  # catch-all:    /blog/a/b/c
│   └── [[...opt]]/page.tsx # optional:     /blog  AND  /blog/a/b
│
├── @modal/                 # PARALLEL ROUTE — rendered as a prop
│   └── (.)photo/[id]/      # INTERCEPTING — (.)same level, (..)up one,
│       └── page.tsx        #   (...)from root
│
└── api/
    └── users/route.ts      # export GET, POST, PUT, PATCH, DELETE
```

### Route handlers

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(req: NextRequest) {
  const page = Number(req.nextUrl.searchParams.get('page') ?? 1);
  const users = await db.user.findMany({ skip: (page - 1) * 20, take: 20 });
  return NextResponse.json(users, {
    headers: { 'Cache-Control': 's-maxage=60, stale-while-revalidate=300' },
  });
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const parsed = CreateUserSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json({ errors: parsed.error.flatten() }, { status: 400 });
  }
  const user = await db.user.create({ data: parsed.data });
  return NextResponse.json(user, { status: 201 });
}

// Dynamic segment params (Next 15: params is a Promise)
export async function GET(_req: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  ...
}
```

---

## 5. Data Fetching and Caching

### 5.1 The Four Caches

```
   ┌─────────────────────────────────────────────────────────┐
   │ 1. REQUEST MEMOIZATION      (server, per-request)        │
   │    Same fetch() URL+options during one render → 1 call.  │
   │    Automatic. Lets you fetch the same data in multiple   │
   │    components without prop drilling.                     │
   ├─────────────────────────────────────────────────────────┤
   │ 2. DATA CACHE               (server, persistent)         │
   │    fetch() results across requests and deployments.      │
   │    Controlled by `cache` and `next.revalidate`.          │
   ├─────────────────────────────────────────────────────────┤
   │ 3. FULL ROUTE CACHE         (server, build/request time) │
   │    Rendered HTML + RSC payload for static routes.        │
   ├─────────────────────────────────────────────────────────┤
   │ 4. ROUTER CACHE             (client, in-memory)          │
   │    RSC payloads for visited/prefetched routes.           │
   │    Makes back/forward instant.                           │
   └─────────────────────────────────────────────────────────┘
```

⚠️ **Next 15 changed the defaults:** `fetch` is no longer cached by default, and route handlers are dynamic by default. Opt *in* to caching now, rather than opting out.

```ts
// Explicit control
fetch(url)                                       // not cached (Next 15 default)
fetch(url, { cache: 'force-cache' })             // cached indefinitely
fetch(url, { cache: 'no-store' })                // never cached, makes route dynamic
fetch(url, { next: { revalidate: 60 } })         // ISR: revalidate after 60s
fetch(url, { next: { tags: ['products'] } })     // taggable for on-demand purge

// Segment-level config
export const dynamic = 'force-dynamic';     // 'auto' | 'force-dynamic' | 'error' | 'force-static'
export const revalidate = 3600;             // seconds, or false
export const fetchCache = 'default-cache';
export const runtime = 'nodejs';            // or 'edge'
export const preferredRegion = ['iad1'];
```

### 5.2 On-Demand Revalidation

```ts
'use server';
import { revalidateTag, revalidatePath } from 'next/cache';

export async function updateProduct(id: string, data: FormData) {
  await db.product.update({ where: { id }, data: parse(data) });
  revalidateTag('products');            // purge every fetch tagged 'products'
  revalidatePath(`/products/${id}`);    // purge one route
}
```

This is the pattern that makes ISR practical: serve from cache indefinitely, purge precisely when the data actually changes.

### 5.3 Caching non-fetch data

```ts
import { unstable_cache } from 'next/cache';
import { cache } from 'react';

// React cache: per-request memoization for any function (DB calls, etc.)
export const getUser = cache(async (id: string) => db.user.findUnique({ where: { id } }));

// unstable_cache: persistent cross-request cache for non-fetch sources
export const getProducts = unstable_cache(
  async (category: string) => db.product.findMany({ where: { category } }),
  ['products-by-category'],                     // key parts
  { revalidate: 3600, tags: ['products'] },
);
```

### 5.4 Avoiding Waterfalls

```jsx
// ❌ Sequential — 300ms total
const user = await getUser(id);
const posts = await getPosts(id);

// ✅ Parallel — 150ms
const [user, posts] = await Promise.all([getUser(id), getPosts(id)]);

// ✅ Better still — preload pattern kicks off the fetch before awaiting
export const preload = (id: string) => { void getUser(id); };

// In a layout:
preload(id);                      // starts now, no await
const other = await getOther();   // runs in parallel
const user = await getUser(id);   // already in flight, hits the memo cache
```

---

## 6. Server Actions

```tsx
// app/actions.ts
'use server';
import { z } from 'zod';
import { auth } from '@/lib/auth';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

const Schema = z.object({ title: z.string().min(1).max(200), body: z.string() });

export async function createPost(prevState: State, formData: FormData) {
  // 1. AUTHENTICATE — this is a public HTTP endpoint
  const session = await auth();
  if (!session) return { error: 'Unauthorized' };

  // 2. VALIDATE
  const parsed = Schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { fieldErrors: parsed.error.flatten().fieldErrors };

  // 3. AUTHORIZE
  if (!can(session.user, 'post:create')) return { error: 'Forbidden' };

  // 4. MUTATE
  const post = await db.post.create({ data: { ...parsed.data, authorId: session.user.id } });

  // 5. REVALIDATE + REDIRECT
  revalidatePath('/posts');
  redirect(`/posts/${post.id}`);
}
```

```tsx
'use client';
import { useActionState } from 'react';
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}

export function PostForm() {
  const [state, formAction] = useActionState(createPost, {});
  return (
    <form action={formAction}>
      <input name="title" aria-describedby="title-error" />
      {state.fieldErrors?.title && <p id="title-error">{state.fieldErrors.title}</p>}
      <SubmitButton />
    </form>
  );
}
```

🔒 **The three rules of Server Actions:**
1. They are **public POST endpoints**. Anyone can call them with any payload.
2. Never trust arguments — validate every one, every time.
3. Auth checks go **inside** the action, not in the component that renders the form.

Progressive enhancement is free: `<form action={serverAction}>` works before JS loads.

---

## 7. Streaming and Suspense

```jsx
export default async function Page() {
  const critical = await getCriticalData();      // blocks the shell — keep it fast

  return (
    <>
      <Header data={critical} />

      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />              {/* slow — streams in when ready */}
      </Suspense>

      <Suspense fallback={<RecsSkeleton />}>
        <Recommendations />      {/* independent boundary, arrives separately */}
      </Suspense>
    </>
  );
}
```

```
   Timeline without streaming:
   0ms ──────────────────────────── 2000ms  [blank page]  → full HTML

   Timeline with streaming:
   0ms ──── 200ms  shell + header + skeletons (visible, interactive)
            600ms  reviews chunk arrives, replaces skeleton
           2000ms  recommendations arrive
```

`loading.tsx` is sugar for wrapping the whole page in one boundary. Nested boundaries give much better perceived performance.

---

## 8. Routing Patterns

### 8.1 Parallel Routes

```
app/
├── layout.tsx           # receives @team and @analytics as props
├── @team/page.tsx
├── @analytics/page.tsx
└── page.tsx             # the `children` slot
```

```jsx
export default function Layout({ children, team, analytics }) {
  return <>{children}<aside>{team}{analytics}</aside></>;
}
```

Each slot has independent loading and error states — one slow panel doesn't block the others.

### 8.2 Intercepting Routes (the modal pattern)

```
app/
├── feed/page.tsx
├── photo/[id]/page.tsx          # full page — direct visit / refresh
└── @modal/(.)photo/[id]/page.tsx # intercepted — modal over the feed
```

```
   Click a photo in the feed  → URL changes to /photo/1, renders the MODAL
   Refresh the page           → renders the FULL PAGE
   Share the link             → recipient gets the FULL PAGE
```

This is exactly how Instagram's photo modal works, and it's genuinely hard to build without framework support.

### 8.3 Navigation APIs

```tsx
'use client';
import { useRouter, usePathname, useSearchParams, useParams } from 'next/navigation';

const router = useRouter();
router.push('/a');        // add to history
router.replace('/a');     // replace current entry
router.refresh();         // re-fetch server data, KEEP client state
router.back(); router.forward();
router.prefetch('/a');

// Link prefetches on viewport entry (static) or hover (dynamic)
<Link href="/about" prefetch={true}>About</Link>
```

---

## 9. Middleware and Edge

```ts
// middleware.ts — at the project root
import { NextResponse, type NextRequest } from 'next/server';

export function middleware(req: NextRequest) {
  const token = req.cookies.get('session')?.value;

  if (!token && req.nextUrl.pathname.startsWith('/dashboard')) {
    const url = new URL('/login', req.url);
    url.searchParams.set('from', req.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  const res = NextResponse.next();
  res.headers.set('x-nonce', crypto.randomUUID());
  return res;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg)$).*)'],
};
```

⚠️ **Middleware runs on the edge runtime.** No Node APIs, no `fs`, no native database drivers, no heavy crypto. Keep it under a few milliseconds — it's on the critical path of every request. Do a cheap cookie/JWT presence check there and full session verification in the page or action.

| | Node runtime | Edge runtime |
|---|---|---|
| APIs | Full Node | Web standard subset |
| Cold start | ~100-500 ms | ~0-5 ms |
| Location | Regional | Global PoPs |
| Bundle limit | Large | ~1-4 MB |
| DB drivers | Any (TCP) | HTTP-based only |

---

## 10. Images, Fonts, Metadata

```jsx
import Image from 'next/image';

<Image
  src="/hero.jpg" alt="Hero" width={1200} height={600}
  priority                          // LCP image — preloads, skips lazy loading
  sizes="(max-width: 768px) 100vw, 50vw"
  placeholder="blur" blurDataURL={base64}
  quality={80}
/>
```

`next/image` gives automatic AVIF/WebP conversion, responsive `srcset`, lazy loading, and — critically — reserved space from `width`/`height`, which prevents layout shift (CLS).

```ts
// next/font — self-hosted, zero layout shift, no external request
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'], display: 'swap', variable: '--font-inter' });
```

```ts
// Static metadata
export const metadata: Metadata = {
  title: { default: 'Site', template: '%s | Site' },
  description: '...',
  openGraph: { images: ['/og.png'] },
  robots: { index: true, follow: true },
};

// Dynamic metadata — deduped with the page's own fetch
export async function generateMetadata({ params }): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  return { title: post.title, description: post.excerpt };
}

// Static params for SSG
export async function generateStaticParams() {
  const posts = await getPosts();
  return posts.map((p) => ({ slug: p.slug }));
}
export const dynamicParams = true;   // render unlisted params on demand
```

---

## 11. Authentication

```
   ┌──────────────────────────────────────────────────────┐
   │  Middleware      → cheap check: does a cookie exist?  │
   │                    redirect if obviously logged out   │
   ├──────────────────────────────────────────────────────┤
   │  Layout/Page     → verify the session properly        │
   │                    (DB/JWT), get the user             │
   ├──────────────────────────────────────────────────────┤
   │  Server Action / → re-verify EVERY time. Never assume │
   │  Route Handler     the caller went through the UI.    │
   ├──────────────────────────────────────────────────────┤
   │  Data layer      → authorize the specific resource    │
   │                    (does this user own this row?)     │
   └──────────────────────────────────────────────────────┘
```

```ts
// lib/dal.ts — Data Access Layer, the recommended pattern
import 'server-only';                  // build error if imported client-side
import { cache } from 'react';

export const verifySession = cache(async () => {
  const cookie = (await cookies()).get('session')?.value;
  const session = await decrypt(cookie);
  if (!session?.userId) redirect('/login');
  return { userId: session.userId, role: session.role };
});

export const getPost = cache(async (id: string) => {
  const { userId } = await verifySession();
  const post = await db.post.findUnique({ where: { id } });
  if (post.authorId !== userId) return null;    // authorize the resource
  return post;
});
```

⚠️ Never pass secrets or full user objects from a Server Component into a Client Component — the whole props object is serialized into the HTML and readable by anyone. Pass only what the client genuinely needs.

---

## 12. Performance

| Lever | Impact |
|---|---|
| Push `'use client'` down the tree | Smaller JS bundle |
| `next/dynamic` with `ssr: false` for heavy client-only libs | Smaller initial bundle |
| Nested `<Suspense>` instead of one `loading.tsx` | Better perceived speed |
| `Promise.all` / preload pattern | Kills waterfalls |
| `priority` on the LCP image | Faster LCP |
| `next/font` | Zero CLS, no font request |
| Static where possible (`generateStaticParams`) | CDN-served |
| `@next/bundle-analyzer` | Finds the 300 KB date library you forgot |

```jsx
const Chart = dynamic(() => import('./Chart'), {
  loading: () => <Skeleton />,
  ssr: false,                          // skip SSR for browser-only libs
});
```

```bash
ANALYZE=true next build          # with @next/bundle-analyzer
next build --profile             # React profiling in production build
```

---

## 13. Deployment

### Self-hosting with Docker

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build                    # requires output: 'standalone' in next.config

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

```js
// next.config.js
module.exports = {
  output: 'standalone',        // bundles only needed node_modules — ~100 MB image
};
```

⚠️ **Env vars:** `NEXT_PUBLIC_*` are inlined into the client bundle **at build time** — they are public and frozen at build. Runtime secrets must not use that prefix and are only readable on the server.

| Feature | Vercel | Self-hosted |
|---|---|---|
| ISR / on-demand revalidation | ✅ | ✅ (needs shared cache across replicas) |
| Image optimization | ✅ | ✅ (needs sharp; consider a CDN instead) |
| Edge middleware | ✅ global | Runs in Node |
| Multi-replica cache coherence | Automatic | Configure a custom cache handler (Redis) |

---

## 14. Interview Section

<details>
<summary><b>Q1. SSR vs SSG vs ISR vs CSR — when do you pick each?</b></summary>

SSG builds HTML at deploy time: fastest and cheapest, served from CDN, but stale until you rebuild. Use for content that changes on your schedule — marketing, docs.

ISR is SSG plus a revalidation window or on-demand purge. You get CDN speed with bounded staleness. Use for product pages and articles — anything with many pages where a full rebuild would be too slow.

SSR renders per request. Always fresh, personalized, but every request costs server time and TTFB suffers. Use for dashboards and anything user-specific.

CSR ships an empty shell and fetches in the browser. Bad TTFB and SEO, but zero server cost. Fine behind a login wall where SEO is irrelevant.

In practice you mix them per route, and PPR is converging on doing it within a single route.
</details>

<details>
<summary><b>Q2. What are Server Components and what do they change?</b></summary>

Components that execute only on the server and never ship their code to the client. They can be async and hit the database directly, then serialize their rendered output into the tree.

Two concrete wins: bundle size, since a markdown parser or heavy date library used only server-side costs zero client bytes; and eliminated waterfalls, since you fetch next to where data is used without a client round trip.

The constraints are real: no state, no effects, no browser APIs, no event handlers. And the composition rule — a Client Component can't import a Server Component, though it can render one passed as `children`.
</details>

<details>
<summary><b>Q3. Explain Next.js caching layers.</b></summary>

Four of them. Request memoization dedupes identical `fetch` calls within a single render pass — automatic, per-request. The Data Cache persists `fetch` results across requests and deployments, controlled by `cache` and `next.revalidate`. The Full Route Cache stores rendered HTML and RSC payload for static routes. The Router Cache is client-side, holding RSC payloads for visited and prefetched routes so back/forward is instant.

Next 15 flipped the defaults so `fetch` and route handlers are uncached unless you opt in — because the old implicit caching was the single biggest source of "why is my data stale" bugs.

`revalidateTag` and `revalidatePath` purge the server caches on demand, which is what makes ISR practical at scale.
</details>

<details>
<summary><b>Q4. How does streaming work and why does it help?</b></summary>

The server sends the HTML shell immediately, with placeholders where Suspense boundaries are, then streams each boundary's real content as its data resolves. React on the client swaps them in.

It helps because TTFB and first paint stop being hostage to your slowest query. A page with a 2-second recommendations call previously showed nothing for 2 seconds; now the header and product info are visible and interactive in 200 ms.

The tradeoff: you can't set HTTP status codes or headers after the stream has started, so errors after the shell must be handled with error boundaries rather than a 500.
</details>

<details>
<summary><b>Q5. What's the security model of Server Actions?</b></summary>

A Server Action compiles to a public POST endpoint with a generated ID. Anyone who can find that ID can invoke it with arbitrary arguments — the form UI is not a gate.

So every action must independently authenticate the caller, validate every argument (with a schema, not manual checks), and authorize the specific resource being touched. Rendering the form only for admins protects nothing.

Next mitigates some risk with origin checks and non-guessable action IDs, but those are defense in depth, not the control.
</details>

<details>
<summary><b>Q6. `layout.tsx` vs `template.tsx`?</b></summary>

Both wrap child routes. A layout persists across navigations within its segment — it doesn't remount, so its state survives and its effects don't re-run. That's what makes navigation feel fast and keeps sidebar scroll positions.

A template creates a new instance on every navigation, so state resets and effects re-run. Use it when you need enter animations or per-page state that must not leak between routes.

Layout is the default; reach for template only when you need the remount.
</details>

<details>
<summary><b>Q7. When should logic go in middleware?</b></summary>

Only cheap, request-shaping work: redirect if a session cookie is missing, rewrite for A/B tests or i18n, set headers. It runs on every matching request before the cache, on the edge runtime, so it's on the critical path of everything.

What doesn't belong: database queries, full session verification, heavy crypto, anything needing Node APIs. Do the cheap presence check in middleware and the authoritative verification in the page, action, or data layer.
</details>

<details>
<summary><b>Q8. How do you avoid request waterfalls in the App Router?</b></summary>

Three techniques. Use `Promise.all` when a component needs several independent pieces of data. Use the preload pattern — call the fetch function without awaiting it early (in the layout), so it's in flight by the time a child awaits it and hits the request memo cache. And use Suspense boundaries so a slow fetch renders its own fallback instead of blocking siblings.

The anti-pattern is a chain of parent-awaits-then-renders-child-that-awaits, which serializes latency down the tree.
</details>

<details>
<summary><b>Q9. What does hydration mean and what causes hydration errors?</b></summary>

Hydration is React attaching event listeners and building its internal tree over server-rendered HTML, expecting the client's first render to produce identical markup.

Errors come from anything non-deterministic between server and client: `Date.now()`, `Math.random()`, `localStorage` or `window` reads during render, browser extensions injecting DOM, locale-dependent formatting, and invalid HTML nesting (a `<div>` inside a `<p>`) that the browser silently reshapes.

Fixes: move browser-only reads into an effect, use `suppressHydrationWarning` for genuinely unavoidable cases like timestamps, or render client-only content with `dynamic(..., { ssr: false })`.
</details>

<details>
<summary><b>Q10. How do you self-host Next.js properly?</b></summary>

Set `output: 'standalone'` so the build emits a minimal server with only the node_modules it needs, then build a multi-stage Docker image running as a non-root user.

The parts people miss: with multiple replicas, the ISR cache is per-instance by default, so revalidation on one pod doesn't affect the others — you need a shared cache handler backed by Redis or S3. Image optimization needs `sharp` and CPU headroom, or you point at a CDN's image service instead. And `NEXT_PUBLIC_*` variables are baked in at build time, so runtime configuration has to come through server-only env vars.
</details>

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        NEXT.JS — ONE PAGE                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ Per route ask: WHEN built? WHERE run? HOW LONG valid?                ║
║   SSG(build) · ISR(build+revalidate) · SSR(request) · CSR(browser)   ║
║   PPR = static shell + streamed dynamic holes                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ FILES: layout(persists) template(remounts) page loading error        ║
║        route.ts(API) not-found default                               ║
║   (group)  [param]  [...all]  [[...opt]]  @slot  (.)intercept        ║
╠══════════════════════════════════════════════════════════════════════╣
║ CACHES: request memo · Data Cache · Full Route · Router(client)      ║
║   Next 15: fetch NOT cached by default — opt in                      ║
║   revalidateTag / revalidatePath to purge                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ SERVER ACTIONS = PUBLIC POST ENDPOINTS                               ║
║   authenticate + validate + authorize INSIDE every action            ║
╠══════════════════════════════════════════════════════════════════════╣
║ STREAMING: nested <Suspense> beats one loading.tsx                   ║
║ WATERFALLS: Promise.all · preload pattern · Suspense boundaries      ║
╠══════════════════════════════════════════════════════════════════════╣
║ 'use client' as LOW in the tree as possible                          ║
║ 'server-only' package to guard secrets                               ║
║ NEXT_PUBLIC_* is baked in at BUILD time and is PUBLIC                ║
╠══════════════════════════════════════════════════════════════════════╣
║ MIDDLEWARE: edge runtime, every request, keep it <5ms                ║
║   cookie presence check only — real auth in the data layer           ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [React](react.md) · [Browser & Performance](browser-performance.md) · [API Design](../03-backend/api-design.md) · [AppSec](../07-security/appsec.md)
