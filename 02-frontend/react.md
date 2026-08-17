# ⚛️ React Complete

> React is a library for keeping a tree of UI in sync with state. Everything else — hooks, Fiber, Suspense, Server Components — exists to make that sync fast, interruptible, and composable.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [Elements, Components, and the Virtual DOM](#2-elements-components-vdom)
3. [Fiber Architecture](#3-fiber-architecture)
4. [Reconciliation](#4-reconciliation)
5. [Hooks — How They Actually Work](#5-hooks-internals)
6. [The Complete Hook Reference](#6-hook-reference)
7. [State Management](#7-state-management)
8. [Rendering Performance](#8-rendering-performance)
9. [Concurrent React](#9-concurrent-react)
10. [Server Components](#10-server-components)
11. [Data Fetching](#11-data-fetching)
12. [Forms](#12-forms)
13. [Component Patterns](#13-component-patterns)
14. [Testing](#14-testing)
15. [Common Bugs and Their Causes](#15-common-bugs)
16. [Interview Section](#16-interview-section)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Mental Model

🧠 **`UI = f(state)`.** You describe what the UI should look like for a given state. React figures out the minimal DOM operations to get there.

```
   ┌──────────┐   render    ┌──────────────┐  reconcile  ┌──────────┐
   │  STATE   │────────────▶│ React Element│────────────▶│   DOM    │
   │          │             │ tree (VDOM)  │   (diff)    │          │
   └──────────┘             └──────────────┘             └──────────┘
        ▲                                                      │
        │                        events                        │
        └──────────────────────────────────────────────────────┘
```

**The two-phase commit:**

```
   ┌─────────────────── RENDER PHASE ────────────────────┐
   │  Pure. Interruptible. Can be thrown away.           │
   │                                                     │
   │  • Call your components                             │
   │  • Build the new Fiber tree                         │
   │  • Diff against the current tree                    │
   │  • Collect a list of effects                        │
   │                                                     │
   │  ⚠️ NO side effects here — React may run this twice, │
   │     pause it, or discard it entirely.               │
   └───────────────────────┬─────────────────────────────┘
                           ▼
   ┌─────────────────── COMMIT PHASE ────────────────────┐
   │  Synchronous. Cannot be interrupted.                │
   │                                                     │
   │  1. Before mutation: getSnapshotBeforeUpdate        │
   │  2. Mutation: apply DOM changes                     │
   │  3. Layout: useLayoutEffect, refs attached          │
   │     ── browser paints ──                            │
   │  4. Passive: useEffect (async, after paint)         │
   └─────────────────────────────────────────────────────┘
```

This split is the whole reason for the "render must be pure" rule and for Strict Mode's double-rendering.

---

## 2. Elements, Components, VDOM

### 2.1 Elements are plain objects

```jsx
<div className="box">Hello</div>
// compiles (React 17+ automatic runtime) to
_jsx('div', { className: 'box', children: 'Hello' })
// which produces
{
  $$typeof: Symbol.for('react.element'),
  type: 'div',
  key: null,
  ref: null,
  props: { className: 'box', children: 'Hello' },
}
```

An element is a **description**, not an instance. It's cheap to create and immutable.

⚠️ **`type` identity matters enormously:**

```jsx
// ❌ New component type on EVERY render → React unmounts and remounts
//    the whole subtree, losing all state and DOM nodes
function Parent() {
  function Child() { return <input />; }    // new function identity each render
  return <Child />;
}

// ✅ Stable identity
function Child() { return <input />; }
function Parent() { return <Child />; }
```

### 2.2 Why a Virtual DOM

Direct DOM manipulation is not slow because of one write — it's slow because of *layout recalculation* and because manual code does redundant writes. The VDOM lets React batch all changes and compute a minimal patch set. The tradeoff is memory and diff cost; frameworks like Svelte and Solid skip it by compiling to targeted updates instead.

---

## 3. Fiber Architecture

A Fiber is a unit of work — a JavaScript object representing a component instance plus its position in the tree.

```js
const fiber = {
  type,                // 'div' | Component function | Symbol (Fragment etc.)
  key,
  stateNode,           // the DOM node or class instance
  
  // The tree, as a linked list — enables pause/resume without recursion
  return,              // parent
  child,               // first child
  sibling,             // next sibling

  pendingProps,        // incoming
  memoizedProps,       // from the last render
  memoizedState,       // hooks linked list lives here
  updateQueue,

  flags,               // Placement | Update | Deletion | Passive ...
  lanes,               // priority bitmask
  alternate,           // the other tree (double buffering)
};
```

### 3.1 Why a Linked List?

The old stack reconciler recursed. You can't pause recursion. Fiber's `child`/`sibling`/`return` pointers let React walk the tree in a loop, check `shouldYield()` between units, and resume exactly where it left off.

```
        App
         │ child
         ▼
       Header ──sibling──▶ Main ──sibling──▶ Footer
         │                   │
         ▼ child             ▼ child
       Logo                List
                             │
                             ▼
                          Item ──sibling──▶ Item

   Traversal: App → Header → Logo → (no child, no sibling, return)
              → Header.sibling = Main → List → Item → Item
              → back up → Footer → done
```

### 3.2 Double Buffering

```
   ┌──────────────┐  alternate  ┌──────────────┐
   │   CURRENT    │◀───────────▶│   WORK IN    │
   │    tree      │             │   PROGRESS   │
   │  (on screen) │             │    tree      │
   └──────────────┘             └──────────────┘
          ▲                            │
          │      commit: swap pointers │
          └────────────────────────────┘
```

React builds the new tree in the alternate, then swaps a single pointer at commit. If the render is abandoned, the alternate is simply discarded — the screen never showed a partial state.

### 3.3 Lanes (Priority)

```
   SyncLane            ── discrete input: click, keypress
   InputContinuousLane ── drag, scroll, hover
   DefaultLane         ── normal setState, network responses
   TransitionLanes     ── startTransition (interruptible)
   RetryLanes          ── Suspense retries
   IdleLane            ── offscreen, lowest
```

Lanes are a bitmask, so React can express "work at these several priorities" and batch them. Higher-priority work interrupts lower-priority in-progress renders.

---

## 4. Reconciliation

React's diff is O(n) because of two assumptions:

1. **Different element types produce different trees** — don't try to match them.
2. **Keys tell you which children are stable across renders.**

```
   Same position, same type      →  update props in place, keep state
   Same position, DIFFERENT type →  unmount old subtree entirely, mount new
   Lists                         →  match by key
```

```jsx
// Type change destroys everything below
{isLoggedIn ? <div><Profile/></div> : <span><Profile/></span>}
// div → span: Profile unmounts and remounts, all its state lost
```

### 4.1 Keys — the rules

```jsx
// ❌ Index keys with a reorderable/filterable list
{items.map((item, i) => <Row key={i} item={item} />)}

// ✅ Stable identity from the data
{items.map((item) => <Row key={item.id} item={item} />)}
```

**Why index keys break things:**

```
   Before: [A(key=0), B(key=1), C(key=2)]
   Delete A
   After:  [B(key=0), C(key=1)]

   React sees: key 0's props changed A→B, key 1's changed B→C, key 2 deleted.
   Result: it MUTATES existing DOM nodes rather than removing the first.
   Any internal state (input text, focus, animation, scroll) stays with the
   WRONG item. Also 2 updates instead of 1 deletion.
```

Index keys are fine only when the list is static, never reordered, never filtered, and items have no internal state.

**Keys as a remount tool:**

```jsx
// Force a full reset when the user changes
<UserProfile key={userId} userId={userId} />
```

This is the idiomatic way to reset all state in a subtree.

---

## 5. Hooks Internals

⚙️ Hooks are a **linked list stored on the fiber**, walked in the same order every render. That's the entire reason for the rules of hooks.

```js
fiber.memoizedState → hook1 → hook2 → hook3 → null

// each hook:
{ memoizedState, baseState, queue, next }
```

```jsx
function Component() {
  const [a, setA] = useState(1);      // hook1
  const [b, setB] = useState(2);      // hook2
  useEffect(() => {}, []);            // hook3
}

// Render 1: builds the list in call order
// Render 2+: walks the list in call order, position = identity
```

**Why conditional hooks break:**

```jsx
function Bad({ cond }) {
  if (cond) {
    const [a] = useState(1);    // sometimes hook1, sometimes absent
  }
  const [b] = useState(2);      // hook1 or hook2 — MISMATCH
}
```

`b` would read `a`'s slot. React detects the count mismatch and throws.

### 5.1 A Minimal useState

```js
let hooks = [];
let cursor = 0;

function useState(initial) {
  const i = cursor++;
  hooks[i] = hooks[i] ?? (typeof initial === 'function' ? initial() : initial);
  const setState = (next) => {
    hooks[i] = typeof next === 'function' ? next(hooks[i]) : next;
    scheduleRerender();          // cursor resets to 0 before the next render
  };
  return [hooks[i], setState];
}
```

This ~10-line model explains: why order matters, why the initializer can be a function (lazy init), why the setter accepts an updater function, and why setters are stable across renders.

### 5.2 Closures Over Stale State

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);      // ❌ `count` is captured at effect creation = 0
    }, 1000);                    //    forever sets 0 + 1 = 1
    return () => clearInterval(id);
  }, []);                        // empty deps → effect never re-runs
}

// ✅ Functional update — never reads a captured value
setCount((c) => c + 1);

// ✅ Or include the dep and let the effect re-create
useEffect(() => { ... }, [count]);
```

This is the #1 React bug. The rule: **if an effect needs the current value of something, either use a functional update or list it in deps.**

---

## 6. Hook Reference

### useState

```jsx
const [state, setState] = useState(initial);

useState(() => expensiveInit());     // lazy init — runs only on mount
setState(newValue);
setState((prev) => derive(prev));    // updater — use when based on previous

// Bailout: setting the SAME value (Object.is) skips the re-render.
// But React may still render once more before bailing out.
```

### useReducer

```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init);

// Use when: multiple related values, next state depends on previous,
// or the update logic is complex enough to test independently.
function reducer(state, action) {
  switch (action.type) {
    case 'add':    return { ...state, items: [...state.items, action.item] };
    case 'remove': return { ...state, items: state.items.filter(i => i.id !== action.id) };
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}
```

`dispatch` is guaranteed stable, so it never needs to be in a dep array — a real advantage over multiple `useState` setters when passing callbacks down.

### useEffect / useLayoutEffect / useInsertionEffect

```
   commit DOM mutations
        ▼
   useInsertionEffect   ← CSS-in-JS libraries inject <style> here
        ▼
   refs attached
        ▼
   useLayoutEffect      ← SYNCHRONOUS, blocks paint. Measure & mutate DOM.
        ▼
   ── BROWSER PAINTS ──
        ▼
   useEffect            ← async, after paint. Everything else.
```

```jsx
useEffect(() => {
  const ac = new AbortController();
  fetchData({ signal: ac.signal }).then(setData).catch(ignoreAbort);
  return () => ac.abort();        // cleanup: runs before next effect AND on unmount
}, [url]);
```

**Dependency rules:**

| Value | In deps? |
|---|---|
| props, state, derived values | ✅ |
| functions defined in the component | ✅ (or move out / wrap in `useCallback`) |
| `useState` setters | ❌ stable |
| `dispatch` from `useReducer` | ❌ stable |
| refs (`ref.current`) | ❌ (mutating a ref doesn't re-render) |
| module-level constants | ❌ |

⚠️ **You often don't need an effect.** React's docs are emphatic about this:

```jsx
// ❌ Effect to derive state
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(first + ' ' + last); }, [first, last]);

// ✅ Just compute during render
const fullName = first + ' ' + last;

// ❌ Effect to reset state on prop change
useEffect(() => { setSelection(null); }, [items]);

// ✅ Use a key to remount
<List key={categoryId} items={items} />

// ❌ Effect for an event response
useEffect(() => { if (submitted) showToast(); }, [submitted]);

// ✅ Do it in the handler
function handleSubmit() { await save(); showToast(); }
```

Effects are for **synchronizing with external systems**: subscriptions, DOM APIs, network, timers, analytics.

### useMemo / useCallback

```jsx
const sorted = useMemo(() => items.sort(cmp), [items]);        // memoize a VALUE
const handler = useCallback((id) => select(id), [select]);      // memoize a FUNCTION
// useCallback(fn, deps) === useMemo(() => fn, deps)
```

They pay off only when:
1. The computation is genuinely expensive, **or**
2. The value is a dep of another hook, **or**
3. The value is a prop to a `React.memo` child.

Otherwise you're adding allocation and comparison cost for nothing. The React Compiler (React 19) automates this — when you adopt it, most manual memoization can be deleted.

### useRef

```jsx
const inputRef = useRef(null);          // DOM access
const prevValue = useRef();             // mutable value that survives renders
const renderCount = useRef(0);          // does NOT trigger re-render

useEffect(() => { prevValue.current = value; });   // "previous value" pattern
```

⚠️ Never read or write `ref.current` during render — it makes render impure and breaks concurrent features. Do it in effects or handlers.

### useContext

```jsx
const ThemeContext = createContext(defaultValue);
<ThemeContext value={theme}>          {/* React 19: no .Provider needed */}
  <App />
</ThemeContext>
const theme = useContext(ThemeContext);
```

⚠️ **Every consumer re-renders when the value's identity changes**, regardless of `memo`. Two fixes:

```jsx
// 1. Memoize the value
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);

// 2. Split contexts — state vs dispatch
<StateContext value={state}>
  <DispatchContext value={dispatch}>   {/* stable — consumers never re-render */}
```

### React 18/19 additions

```jsx
const id = useId();                              // SSR-safe unique IDs
const [isPending, startTransition] = useTransition();
const deferred = useDeferredValue(value);
const width = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot);
useImperativeHandle(ref, () => ({ focus: () => inputRef.current.focus() }), []);
useDebugValue(value);

// React 19
const value = use(promise);                      // suspends; also reads context conditionally
const [state, formAction, isPending] = useActionState(action, initialState);
const { pending } = useFormStatus();             // inside a <form> descendant
const optimistic = useOptimistic(state, reducer);
```

---

## 7. State Management

### 7.1 The Decision Tree

```
              Where does this state belong?
                        │
        ┌───────────────┼───────────────────┐
        ▼               ▼                   ▼
   Can it be       Used by one       Used by many
   DERIVED?        component?        components?
        │               │                   │
        ▼               ▼                   ▼
   Just compute    useState /         Is it SERVER data?
   it in render    useReducer                │
                                  ┌──────────┴──────────┐
                                  ▼                     ▼
                                 YES                   NO
                                  │                     │
                                  ▼                     ▼
                         TanStack Query /        How often does it change?
                         SWR / RSC                      │
                         (cache, dedupe,        ┌───────┴────────┐
                          revalidate)           ▼                ▼
                                             rarely          frequently
                                                │                │
                                                ▼                ▼
                                            Context      Zustand/Jotai/Redux
                                                        (external store, avoids
                                                         context re-render fan-out)
```

**The single most important insight:** most "global state" is actually *server cache*, and it should be handled by a data-fetching library, not Redux. Once you remove server data, the remaining truly-global client state is usually tiny (theme, auth session, modal stack).

### 7.2 Library Comparison

| Library | Model | Re-render granularity | Boilerplate | Best for |
|---|---|---|---|---|
| `useState` | Local | Component | None | Default choice |
| Context | Tree injection | All consumers | Low | Rarely-changing globals |
| Zustand | External store + selectors | Per-selector | Very low | Most client global state |
| Jotai/Recoil | Atomic | Per-atom | Low | Fine-grained, derived graphs |
| Redux Toolkit | Single store + reducers | Per-selector | Medium | Large teams, time-travel debugging, strict conventions |
| TanStack Query | Server cache | Per-query | Low | **All server data** |
| XState | State machine | Per-machine | Medium | Complex flows with illegal transitions |

```jsx
// Zustand — minimal and effective
import { create } from 'zustand';

const useStore = create((set, get) => ({
  count: 0,
  increment: () => set((s) => ({ count: s.count + 1 })),
  reset: () => set({ count: 0 }),
}));

// Selector: this component re-renders ONLY when count changes
const count = useStore((s) => s.count);
```

---

## 8. Rendering Performance

### 8.1 When Does a Component Re-render?

```
   A component re-renders when:
   1. Its own state changes
   2. Its parent re-renders            ← the big one
   3. A context it consumes changes
   4. Its props change AND it's wrapped in memo
```

⚠️ **Point 2 is unconditional.** By default, a parent re-render re-renders *all* descendants, regardless of whether props changed. This is usually fine — rendering is cheap. It becomes a problem in large trees or with expensive children.

### 8.2 The Children Prop Trick

Before reaching for `memo`, restructure:

```jsx
// ❌ ExpensiveTree re-renders on every count change
function App() {
  const [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(c => c + 1)}>
      {count}
      <ExpensiveTree />
    </div>
  );
}

// ✅ ExpensiveTree is created by the PARENT, so its element identity
//    is unchanged when Wrapper re-renders → React bails out
function App() {
  return <Wrapper><ExpensiveTree /></Wrapper>;
}
function Wrapper({ children }) {
  const [count, setCount] = useState(0);
  return <div onClick={() => setCount(c => c + 1)}>{count}{children}</div>;
}
```

This costs nothing and requires no memoization. Prefer it.

### 8.3 React.memo

```jsx
const Row = memo(function Row({ item, onSelect }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

// ⚠️ memo is USELESS if props aren't referentially stable
<Row item={item} onSelect={(id) => select(id)} />   // new function every render!

// ✅
const onSelect = useCallback((id) => select(id), []);

// Custom comparison (rarely needed, easy to get wrong)
const Row = memo(Component, (prev, next) => prev.item.id === next.item.id);
```

### 8.4 List Virtualization

Rendering 10,000 rows creates 10,000 DOM nodes. Virtualization renders only the visible window.

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function List({ items }) {
  const parentRef = useRef(null);
  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 5,                     // render a few extra above/below
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((v) => (
          <div key={items[v.index].id}
               style={{ position: 'absolute', top: 0, left: 0, width: '100%',
                        height: v.size, transform: `translateY(${v.start}px)` }}>
            <Row item={items[v.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 8.5 Code Splitting

```jsx
const Dashboard = lazy(() => import('./Dashboard'));

<Suspense fallback={<Skeleton />}>
  <Dashboard />
</Suspense>

// Preload on intent — feels instant
<Link onMouseEnter={() => import('./Dashboard')} to="/dashboard">
```

### 8.6 Profiling Workflow

```
1. React DevTools → Profiler → record an interaction
2. Flamegraph: wide bars = slow components
3. Ranked chart: sorted by render time
4. "Why did this render?" (enable in settings) → shows which prop changed
5. Fix the actual cause; re-measure
```

⚠️ Always profile a **production build**. Development React is 2-5× slower and includes double-rendering in Strict Mode.

---

## 9. Concurrent React

### 9.1 Automatic Batching

```jsx
// React 17: batched only inside React event handlers
// React 18+: batched EVERYWHERE
setTimeout(() => {
  setA(1);        // ┐
  setB(2);        // ├─ one re-render
  setC(3);        // ┘
}, 0);

flushSync(() => setD(4));   // opt out when you must read the DOM immediately
```

### 9.2 Transitions

```jsx
const [isPending, startTransition] = useTransition();

function handleChange(e) {
  setQuery(e.target.value);                    // urgent — input stays responsive
  startTransition(() => {
    setResults(filterHugeList(e.target.value)); // interruptible
  });
}
```

```
   Without transition:
   type 'a' → block 300ms rendering results → type 'b' → block 300ms ...
   Input feels frozen.

   With transition:
   type 'a' → input updates instantly → start rendering results
   type 'b' → ABANDON that render, input updates instantly → restart
   Only the final render completes.
```

`useDeferredValue` is the same idea when you don't own the setter:

```jsx
const deferredQuery = useDeferredValue(query);
const results = useMemo(() => filter(deferredQuery), [deferredQuery]);
```

### 9.3 Suspense

```jsx
<Suspense fallback={<Skeleton />}>
  <ProfileDetails />          {/* throws a promise while loading */}
  <Suspense fallback={<PostsSkeleton />}>
    <ProfilePosts />          {/* nested boundary = independent loading */}
  </Suspense>
</Suspense>
```

Suspense turns loading states into a *declarative tree property* rather than per-component `if (isLoading)` branches. Combined with streaming SSR, the server sends the shell immediately and streams each boundary's content as it resolves.

---

## 10. Server Components

```
   ┌───────────────── SERVER COMPONENT ─────────────────┐
   │  • Runs ONCE, on the server, at request/build time │
   │  • Can be async; can read DB/filesystem directly   │
   │  • Zero bytes shipped to the client                │
   │  • No state, no effects, no browser APIs           │
   │  • Can import and render Client Components         │
   └────────────────────┬───────────────────────────────┘
                        │  serialized RSC payload
                        ▼
   ┌───────────────── CLIENT COMPONENT ─────────────────┐
   │  'use client'                                      │
   │  • Hydrates in the browser                         │
   │  • state, effects, event handlers, browser APIs    │
   │  • Cannot import a Server Component (but CAN       │
   │    render one passed as `children`)                │
   └────────────────────────────────────────────────────┘
```

```jsx
// app/page.jsx — Server Component by default
import { db } from '@/lib/db';
import LikeButton from './LikeButton';

export default async function Page() {
  const posts = await db.post.findMany();     // direct DB access, no API layer
  return (
    <ul>
      {posts.map((p) => (
        <li key={p.id}>
          {p.title}
          <LikeButton postId={p.id} />        {/* client island */}
        </li>
      ))}
    </ul>
  );
}
```

```jsx
// LikeButton.jsx
'use client';
import { useState } from 'react';
export default function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? '♥' : '♡'}</button>;
}
```

### Server Actions

```jsx
// Runs on the server, callable from the client — no API route needed
async function createPost(formData) {
  'use server';
  const title = formData.get('title');
  await db.post.create({ data: { title } });
  revalidatePath('/posts');
}

<form action={createPost}>
  <input name="title" />
  <SubmitButton />
</form>
```

⚠️ Server Actions are public HTTP endpoints. **Always authenticate and validate inside them** — the fact that a form is only rendered for admins does not stop anyone from calling the action directly.

### The composition rule

```jsx
// ❌ Client Component importing a Server Component
'use client';
import ServerThing from './ServerThing';   // becomes a Client Component

// ✅ Pass it through as children from a Server parent
<ClientWrapper>
  <ServerThing />        {/* stays server-rendered */}
</ClientWrapper>
```

---

## 11. Data Fetching

### 11.1 Why not useEffect

```jsx
// ❌ The naive version has: no cache, no dedupe, a race condition,
//    no retry, waterfalls, and refetch-on-every-mount
useEffect(() => {
  fetch(`/api/user/${id}`).then(r => r.json()).then(setUser);
}, [id]);
```

The race: change `id` fast and an older, slower response can arrive last and overwrite the newer data.

```jsx
// Minimum viable fix
useEffect(() => {
  let cancelled = false;
  const ac = new AbortController();
  fetch(url, { signal: ac.signal })
    .then(r => { if (!r.ok) throw new Error(r.statusText); return r.json(); })
    .then(d => { if (!cancelled) setData(d); })
    .catch(e => { if (e.name !== 'AbortError' && !cancelled) setError(e); });
  return () => { cancelled = true; ac.abort(); };
}, [url]);
```

### 11.2 TanStack Query

```jsx
const { data, isLoading, error, refetch } = useQuery({
  queryKey: ['user', id],
  queryFn: ({ signal }) => fetch(`/api/user/${id}`, { signal }).then(r => r.json()),
  staleTime: 5 * 60_000,        // don't refetch for 5 min
  gcTime: 10 * 60_000,          // keep in cache 10 min after unused
  retry: 3,
});

// Mutations with optimistic update + rollback
const mutation = useMutation({
  mutationFn: updateUser,
  onMutate: async (newUser) => {
    await queryClient.cancelQueries({ queryKey: ['user', id] });
    const previous = queryClient.getQueryData(['user', id]);
    queryClient.setQueryData(['user', id], newUser);       // optimistic
    return { previous };
  },
  onError: (_err, _new, ctx) => {
    queryClient.setQueryData(['user', id], ctx.previous);  // rollback
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['user', id] }),
});
```

You get for free: caching, deduplication, background refetch, stale-while-revalidate, retry with backoff, pagination/infinite scroll, offline handling, and request cancellation.

---

## 12. Forms

### 12.1 Controlled vs Uncontrolled

| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | The DOM |
| Re-render per keystroke | ✅ | ❌ |
| Instant validation | Easy | Harder |
| Performance on big forms | Can suffer | Better |
| Code | `value` + `onChange` | `ref` / `FormData` |

```jsx
// Uncontrolled — often the better default for large forms
function Form() {
  function handleSubmit(e) {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.currentTarget));
    submit(data);
  }
  return (
    <form onSubmit={handleSubmit}>
      <input name="email" defaultValue="" required type="email" />
      <button>Send</button>
    </form>
  );
}
```

### 12.2 React Hook Form + Zod

```jsx
const schema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'At least 8 characters'),
});

function Form() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm({ resolver: zodResolver(schema) });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} aria-invalid={!!errors.email} />
      {errors.email && <span role="alert">{errors.email.message}</span>}
      <button disabled={isSubmitting}>Submit</button>
    </form>
  );
}
```

RHF keeps inputs uncontrolled, so typing doesn't re-render the whole form — a large win on forms with many fields.

---

## 13. Component Patterns

```jsx
// ── Compound components — shared implicit state ──────────
const TabsContext = createContext(null);

function Tabs({ children, defaultValue }) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);
  return <TabsContext value={value}>{children}</TabsContext>;
}
Tabs.List = ({ children }) => <div role="tablist">{children}</div>;
Tabs.Tab = ({ value, children }) => {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button role="tab" aria-selected={active === value}
            onClick={() => setActive(value)}>{children}</button>
  );
};
Tabs.Panel = ({ value, children }) => {
  const { active } = useContext(TabsContext);
  return active === value ? <div role="tabpanel">{children}</div> : null;
};

// Usage reads like HTML
<Tabs defaultValue="a">
  <Tabs.List><Tabs.Tab value="a">A</Tabs.Tab><Tabs.Tab value="b">B</Tabs.Tab></Tabs.List>
  <Tabs.Panel value="a">Content A</Tabs.Panel>
</Tabs>

// ── Custom hooks — reuse LOGIC, not markup ───────────────
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return debounced;
}

function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try { return JSON.parse(localStorage.getItem(key)) ?? initial; }
    catch { return initial; }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(value)); } catch {}
  }, [key, value]);
  return [value, setValue];
}

function useMediaQuery(query) {
  return useSyncExternalStore(
    (cb) => { const m = matchMedia(query); m.addEventListener('change', cb);
              return () => m.removeEventListener('change', cb); },
    () => matchMedia(query).matches,
    () => false,                       // server snapshot
  );
}

function useIntersectionObserver(ref, options) {
  const [entry, setEntry] = useState(null);
  useEffect(() => {
    if (!ref.current) return;
    const obs = new IntersectionObserver(([e]) => setEntry(e), options);
    obs.observe(ref.current);
    return () => obs.disconnect();
  }, [ref, options?.threshold, options?.rootMargin]);
  return entry;
}

// ── Error boundary (still class-only) ────────────────────
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  componentDidCatch(error, info) { logToService(error, info.componentStack); }
  render() {
    if (this.state.error) {
      return this.props.fallback?.(this.state.error,
             () => this.setState({ error: null })) ?? <p>Something went wrong.</p>;
    }
    return this.props.children;
  }
}
```

⚠️ Error boundaries do **not** catch: event handler errors, async errors (`setTimeout`, promises), SSR errors, or errors in the boundary itself. Use `try/catch` for those.

---

## 14. Testing

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

test('submits the form with entered values', async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(<LoginForm onSubmit={onSubmit} />);

  // Query by ACCESSIBLE ROLE — tests what users perceive
  await user.type(screen.getByRole('textbox', { name: /email/i }), 'a@b.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.click(screen.getByRole('button', { name: /sign in/i }));

  await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
    email: 'a@b.com', password: 'secret123',
  }));
});
```

**Query priority (Testing Library):**
```
1. getByRole            ← accessible + resilient. Default choice.
2. getByLabelText       ← form fields
3. getByPlaceholderText
4. getByText
5. getByDisplayValue
6. getByTestId          ← last resort
```

Mock the network at the boundary with MSW, not by stubbing `fetch`:

```js
const server = setupServer(
  http.get('/api/user/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'Ada' })),
);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## 15. Common Bugs

| Symptom | Cause | Fix |
|---|---|---|
| Infinite re-render loop | `setState` during render, or an object/array in deps | Move to an effect/handler; memoize the dep |
| Stale value inside a callback | Closure captured an old render's variable | Functional update or add to deps |
| List items lose input state on reorder | Index keys | Stable `key` from data |
| Effect runs twice on mount | Strict Mode (dev only) | Make effects idempotent with cleanup |
| `memo` doesn't help | Props are new objects/functions each render | `useCallback`/`useMemo` on the props |
| Context change re-renders everything | Value identity changes | `useMemo` the value; split contexts |
| "Cannot update during render" | `setState` called in the render body | Move to an event handler or effect |
| Layout flicker | `useEffect` mutating layout after paint | `useLayoutEffect` |
| Race condition on fast typing | Unordered async responses | `AbortController` + cancelled flag, or TanStack Query |
| Memory leak warning | Subscription not cleaned up | Return a cleanup from the effect |

### Why Strict Mode double-invokes

In development, React 18+ mounts, unmounts, and remounts every component, and calls render functions twice. This is deliberate: it surfaces effects that aren't idempotent and render functions that aren't pure — the exact bugs that break under concurrent rendering. If double-invocation breaks your code, your code has a real bug that would eventually appear in production.

---

## 16. Interview Section

<details>
<summary><b>Q1. What is the virtual DOM and is it actually faster?</b></summary>

The VDOM is a lightweight JS object tree describing the UI. On state change, React builds a new tree and diffs it against the previous one, producing a minimal set of DOM operations.

It is *not* inherently faster than optimal hand-written DOM code — you could always beat it manually. What it buys is that you get near-optimal updates *declaratively*, without tracking which nodes need changing. The real cost it avoids is redundant layout-triggering writes.

Frameworks like Svelte and Solid skip the VDOM entirely by compiling templates into fine-grained update instructions, which is genuinely faster, at the cost of a compile step and less runtime flexibility.
</details>

<details>
<summary><b>Q2. Explain reconciliation and the role of keys.</b></summary>

Reconciliation is how React decides what changed. A general tree-diff is O(n³), so React uses two heuristics to get O(n): different element types mean different trees (so it discards and rebuilds rather than trying to match), and keys identify which children are the same element across renders.

Without keys React matches children by position. With a stable key from your data, React can tell that an item moved rather than that two items' contents changed — so it moves one DOM node instead of mutating several, and internal state (input values, focus, animations) follows the right item.

Index keys defeat this whenever the list is reordered, filtered, or has items inserted at the front.
</details>

<details>
<summary><b>Q3. Why do the rules of hooks exist?</b></summary>

Hooks are stored as a linked list on the fiber and matched to calls purely by **call order**. There's no name or identifier attached — the third `useState` call gets the third slot.

So if a hook is inside a condition or loop, the order can change between renders and a hook reads another hook's state. React detects the count mismatch and throws, but the underlying issue is positional identity.

The second rule — only call hooks from React functions — exists because hooks need a "currently rendering fiber" to attach to, which only exists during a React render.
</details>

<details>
<summary><b>Q4. `useEffect` vs `useLayoutEffect`.</b></summary>

Both run after the DOM is mutated. `useLayoutEffect` runs *synchronously before the browser paints*, so it blocks paint; `useEffect` runs asynchronously after paint.

Use `useLayoutEffect` when you must measure the DOM and mutate it before the user sees anything — tooltip positioning, scroll restoration, measuring an element to size another. Anything visible you do in `useEffect` risks a flicker frame.

Use `useEffect` for everything else, because blocking paint hurts responsiveness. Note `useLayoutEffect` doesn't run during SSR and warns about it.
</details>

<details>
<summary><b>Q5. What is a closure/stale state bug in React?</b></summary>

Every render creates new function instances that close over that render's props and state. If you capture a value in an effect with an empty dep array, or in a `setInterval` callback, that function keeps seeing the value from the render where it was created — forever.

Two fixes: use the functional updater form (`setCount(c => c + 1)`), which never reads a captured value, or list the dependency so the effect re-creates with fresh values. A ref works when you genuinely need the latest value without re-subscribing.
</details>

<details>
<summary><b>Q6. When should you use `useMemo` and `useCallback`?</b></summary>

Only when there's a measured reason: the computation is genuinely expensive, the value feeds another hook's dep array, or it's a prop to a memoized child. Memoization isn't free — it allocates, stores, and compares on every render.

The most common mistake is wrapping everything, which adds overhead without benefit. The second most common is wrapping a callback but not memoizing the child, so the stable function accomplishes nothing.

React 19's compiler makes most of this automatic, which is a strong signal that manual memoization was always more of a workaround than a design.
</details>

<details>
<summary><b>Q7. How does Context work and what's its performance problem?</b></summary>

Context lets you pass a value to any descendant without threading props. Consumers subscribe to the nearest provider above them.

The problem: when the provider's value changes *by identity*, every consumer re-renders — `React.memo` doesn't help, because context isn't a prop. So an inline object `value={{user, setUser}}` re-renders all consumers on every provider render.

Mitigations: memoize the value, split state and dispatch into separate contexts (dispatch is stable, so those consumers never re-render), or use an external store with selectors — Zustand, Redux — where each subscriber only re-renders if its selected slice changed.
</details>

<details>
<summary><b>Q8. What problem do concurrent features solve?</b></summary>

Before React 18, rendering was synchronous and uninterruptible. A big render blocked the main thread, so typing into an input that filtered a large list felt frozen.

Concurrent rendering makes the render phase interruptible. `startTransition` marks an update as low-priority, so React can abandon that render when a higher-priority one (a keystroke) arrives, and restart it later. The input stays responsive and only the final result is committed.

The prerequisites are what forced the "render must be pure" rule — React may render your component multiple times or discard the work entirely.
</details>

<details>
<summary><b>Q9. Server Components — what and why?</b></summary>

Components that run only on the server and never ship their code to the browser. They can be async and access the database or filesystem directly, then serialize their output into the client tree.

Two wins: bundle size (a markdown renderer or date library used only in a Server Component costs 0 KB on the client), and eliminated waterfalls (fetch data next to where it's used, on the server, with no client round-trip).

The cost is a genuinely new mental model — no state, no effects, no browser APIs — plus a strict composition rule: a Client Component can't import a Server Component, though it can render one passed as `children`.
</details>

<details>
<summary><b>Q10. Why does Strict Mode run effects twice?</b></summary>

To surface non-idempotent effects. React deliberately mounts, unmounts, and remounts each component in development, so an effect that subscribes without cleaning up, or fetches without cancellation, breaks visibly and immediately.

This isn't arbitrary strictness — future features (offscreen rendering, state preservation across unmounts) genuinely will mount and unmount components, so the double-invoke simulates that. If your effect breaks, it has a real bug.

It only happens in development; production mounts once.
</details>

<details>
<summary><b>Q11. Controlled vs uncontrolled inputs?</b></summary>

Controlled means React state is the source of truth: `value` plus `onChange`, so every keystroke re-renders. Uncontrolled means the DOM holds the value and you read it with a ref or `FormData` on submit.

Controlled gives you instant validation, conditional formatting, and the ability to drive the input from elsewhere. Uncontrolled is simpler and much faster for large forms, since typing doesn't re-render.

React Hook Form is popular precisely because it keeps inputs uncontrolled while still giving a controlled-feeling API.
</details>

<details>
<summary><b>Q12. How would you optimize a slow React list?</b></summary>

In order:

1. **Profile first** in a production build to confirm what's actually slow.
2. **Virtualize** if there are many rows — this is usually the whole fix, since it caps DOM nodes at what's visible.
3. **Fix keys** — index keys cause unnecessary DOM mutation.
4. **Restructure with `children`** so expensive subtrees aren't re-created by parent state changes.
5. **`memo` the row** plus `useCallback` on its handlers.
6. **Move expensive filtering/sorting** into `useMemo`, or off the main thread entirely.
7. **`useDeferredValue`** so filtering doesn't block typing.

Note that 1–4 come before memoization. Memoizing a badly structured tree usually just moves the cost around.
</details>

---

## 17. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                          REACT — ONE PAGE                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ UI = f(state).  Render phase = PURE, interruptible.                  ║
║ Commit phase = sync, side effects allowed.                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ RE-RENDER TRIGGERS: own state · parent renders · context changes     ║
║   (props alone do NOT — that's what memo adds)                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ HOOKS = linked list on the fiber, matched BY CALL ORDER              ║
║   → never in conditions/loops. Never below an early return.          ║
║ Stable identities (no deps needed): setState, dispatch, refs         ║
╠══════════════════════════════════════════════════════════════════════╣
║ EFFECTS are for EXTERNAL SYSTEMS only.                               ║
║   Derived value?  → compute in render                                ║
║   Reset on prop?  → use a key                                        ║
║   Event response? → do it in the handler                             ║
║ Always return cleanup. Effects must be idempotent.                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ KEYS: stable id from data. Index keys break reorder/filter/insert.   ║
║ <X key={id} /> is also the idiomatic way to RESET a subtree.         ║
╠══════════════════════════════════════════════════════════════════════╣
║ PERF ORDER: profile → virtualize → keys → children-prop →            ║
║             memo+useCallback → useMemo → useDeferredValue            ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONCURRENT: startTransition (interruptible) · useDeferredValue       ║
║             Suspense boundaries · automatic batching · flushSync     ║
╠══════════════════════════════════════════════════════════════════════╣
║ STATE: derived→compute · local→useState · server→TanStack Query      ║
║        global rare→Context · global hot→Zustand/Redux selectors      ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONTEXT re-renders ALL consumers on value identity change.           ║
║   → useMemo the value, split state/dispatch contexts                 ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [JavaScript](../01-languages/javascript.md) · [TypeScript](../01-languages/typescript.md) · [Next.js](nextjs.md) · [Browser & Performance](browser-performance.md)
