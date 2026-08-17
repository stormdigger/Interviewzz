# 🟨 JavaScript Deep Dive

> From the engine up. This book explains *why* JavaScript behaves the way it does, not just what to type.

---

## 📑 Table of Contents

1. [Mental Model: What JavaScript Actually Is](#1-mental-model)
2. [The Engine: Parsing, Compilation, Optimization](#2-the-engine)
3. [Types and Coercion](#3-types-and-coercion)
4. [Execution Contexts, Scope, Closures](#4-execution-contexts-scope-closures)
5. [The `this` Binding System](#5-the-this-binding-system)
6. [Prototypes and Object Orientation](#6-prototypes-and-object-orientation)
7. [The Event Loop](#7-the-event-loop)
8. [Promises and Async/Await](#8-promises-and-asyncawait)
9. [Memory Management and Leaks](#9-memory-management-and-leaks)
10. [Modules](#10-modules)
11. [Iterators, Generators, Symbols](#11-iterators-generators-symbols)
12. [Proxies and Metaprogramming](#12-proxies-and-metaprogramming)
13. [Functional Patterns](#13-functional-patterns)
14. [Performance Engineering](#14-performance-engineering)
15. [Modern JavaScript (ES2020→)](#15-modern-javascript)
16. [Interview Section](#16-interview-section)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Mental Model

🧠 **The one-paragraph version:** JavaScript is a single-threaded, garbage-collected language with first-class functions, prototypal inheritance, and dynamic types. It never blocks — instead, slow work is handed to the host environment (browser or Node), which calls back into JavaScript when finished. Almost every JavaScript oddity traces back to one of four decisions: dynamic typing with aggressive coercion, function-scoped `var` (later patched by `let`/`const`), prototypes instead of classes, and the run-to-completion event loop.

```
┌──────────────────────────────────────────────────────────────────┐
│                       THE RUNTIME                                │
│                                                                  │
│   ┌────────────────────────┐        ┌───────────────────────┐    │
│   │      JS ENGINE         │        │   HOST ENVIRONMENT    │    │
│   │      (V8, JSC, SM)     │        │   (Browser / Node)    │    │
│   │                        │        │                       │    │
│   │  ┌──────────────────┐  │        │  ┌─────────────────┐  │    │
│   │  │   Call Stack     │  │        │  │  Web APIs /     │  │    │
│   │  │  ┌────────────┐  │  │        │  │  libuv          │  │    │
│   │  │  │  frame     │  │  │───────▶│  │  - timers       │  │    │
│   │  │  ├────────────┤  │  │        │  │  - fetch/http   │  │    │
│   │  │  │  frame     │  │  │        │  │  - fs           │  │    │
│   │  │  └────────────┘  │  │        │  │  - DOM events   │  │    │
│   │  └──────────────────┘  │        │  └────────┬────────┘  │    │
│   │                        │        │           │           │    │
│   │  ┌──────────────────┐  │        │           ▼           │    │
│   │  │      Heap        │  │        │  ┌─────────────────┐  │    │
│   │  │  objects,        │  │        │  │  Task Queues    │  │    │
│   │  │  closures,       │  │        │  │  ┌───────────┐  │  │    │
│   │  │  strings         │  │        │  │  │microtasks │  │  │    │
│   │  └──────────────────┘  │        │  │  ├───────────┤  │  │    │
│   │                        │        │  │  │ macrotasks│  │  │    │
│   └────────────────────────┘        │  │  └───────────┘  │  │    │
│              ▲                      │  └────────┬────────┘  │    │
│              │                      └───────────┼───────────┘    │
│              │      EVENT LOOP                  │                │
│              └──────────────────────────────────┘                │
│         (when stack empty, pull next task and run it)            │
└──────────────────────────────────────────────────────────────────┘
```

**Read that diagram until it's automatic.** Nearly every async interview question is answered by tracing a value through it.

---

## 2. The Engine

### 2.1 Pipeline

Modern engines are not simple interpreters. V8's pipeline:

```
Source
  │
  ▼
┌──────────┐   Scanner produces tokens; parser produces AST.
│ Parser   │   Lazy parsing: function bodies are only pre-parsed
│          │   until actually called (big startup win).
└────┬─────┘
     ▼
┌──────────┐   Ignition: AST → bytecode. Fast to produce,
│Interpreter│  compact in memory. Starts executing immediately.
│ (Ignition)│
└────┬─────┘
     │  profiling data: which functions are hot,
     │  what types actually flow through
     ▼
┌──────────┐   Sparkplug (baseline) → Maglev (mid) → TurboFan (top).
│ Optimizing│  Each tier compiles to machine code with speculative
│ Compilers │  assumptions based on observed types.
└────┬─────┘
     ▼
┌──────────┐   If an assumption breaks (a string arrives where the
│Deopt     │   compiler assumed a number), bail back to bytecode
│          │   and re-profile. Deopt is expensive.
└──────────┘
```

### 2.2 Hidden Classes (Shapes / Maps)

⚙️ JavaScript objects are dictionaries in the spec but engines refuse to implement them that way — dictionary lookups are too slow. Instead V8 assigns each object a **hidden class** describing its layout.

```js
function Point(x, y) {
  this.x = x;   // transition: C0 → C1 (adds x at offset 0)
  this.y = y;   // transition: C1 → C2 (adds y at offset 1)
}

const a = new Point(1, 2);   // shape C2
const b = new Point(3, 4);   // shape C2  ← same shape, fast

const c = new Point(5, 6);
c.z = 7;                     // shape C2 → C3, c now differs from a,b
```

```
     Hidden class transition tree

            C0 {}
             │ add x
             ▼
            C1 {x}
             │ add y
             ▼
            C2 {x, y}  ◀── a, b live here
             │ add z
             ▼
            C3 {x, y, z}  ◀── c lives here
```

**Consequences you can act on:**

| Do | Don't | Why |
|---|---|---|
| Initialize all fields in the constructor | Add properties later, conditionally | Keeps one shape |
| Always assign in the same order | `if (cond) o.a = 1; else o.b = 2;` | Order creates different shapes |
| Use `null` placeholders for optional fields | `delete obj.prop` | `delete` can force dictionary mode |
| Keep arrays homogeneous | `[1, 'a', {}, 2.5]` | Element kinds degrade irreversibly |

### 2.3 Inline Caches

At each property access site, the engine caches "last time, this shape had `x` at offset 0."

```
obj.x   ──▶ IC state progression:

  uninitialized  →  monomorphic  →  polymorphic  →  megamorphic
                     (1 shape)      (2-4 shapes)    (5+ shapes)
                     ⚡ fastest      🙂 fine          🐢 hash lookup
```

Passing objects of many different shapes to the same function is what pushes call sites megamorphic. This is the actual mechanism behind "monomorphic code is faster."

### 2.4 Element Kinds

Arrays have their own lattice, and transitions only go **downward**:

```
  PACKED_SMI_ELEMENTS      [1, 2, 3]           ⚡ fastest
        │
        ├─────────────▶ HOLEY_SMI_ELEMENTS     [1, , 3]
        ▼                      │
  PACKED_DOUBLE_ELEMENTS  [1.5, 2.5]           │
        │                      │               │
        ├─────────────▶ HOLEY_DOUBLE_ELEMENTS  ◀┘
        ▼                      │
  PACKED_ELEMENTS         [1, 'a', {}]         │
        │                      │               │
        └─────────────▶ HOLEY_ELEMENTS   ◀─────┘  🐢 slowest
```

⚠️ `new Array(1000)` creates a holey array immediately. `arr[1000] = x` on a length-3 array does too. Prefer `Array.from({length: n}, ...)` or push in order.

---

## 3. Types and Coercion

### 3.1 The Type System

```
                    JavaScript Values
                          │
        ┌─────────────────┴─────────────────┐
        ▼                                   ▼
   PRIMITIVES                            OBJECTS
   (immutable, copied by value)          (mutable, copied by reference)
        │                                   │
   ┌────┼────┬────┬────┬────┬────┐    ┌────┼─────┬──────┬───────┐
   ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼     ▼      ▼       ▼
undefined  boolean  bigint  symbol  Object Array Function Date  Map/Set
      null      number   string
```

`typeof` results — memorize the table, including the two liars:

| Value | `typeof` | Note |
|---|---|---|
| `undefined` | `"undefined"` | |
| `null` | `"object"` | ⚠️ **historical bug**, kept for compatibility |
| `true` | `"boolean"` | |
| `42` | `"number"` | |
| `42n` | `"bigint"` | |
| `"hi"` | `"string"` | |
| `Symbol()` | `"symbol"` | |
| `{}` / `[]` / `new Date()` | `"object"` | ⚠️ arrays need `Array.isArray` |
| `function(){}` | `"function"` | Technically an object |
| undeclared variable | `"undefined"` | Doesn't throw — only safe use of typeof |

### 3.2 The Abstract Operations

Coercion is not chaos — it's three spec algorithms.

**ToPrimitive(input, hint)**

```
        ToPrimitive(obj, hint)
                │
     ┌──────────┴───────────┐
     │ Symbol.toPrimitive?  │──yes──▶ call it, done
     └──────────┬───────────┘
                │ no
                ▼
     ┌──────────────────────┐
     │  hint === "string"?  │
     └──────┬────────┬──────┘
        yes │        │ no ("number" or "default")
            ▼        ▼
    [toString,   [valueOf,
     valueOf]     toString]
            │        │
            └────┬───┘
                 ▼
    try each in order; first one returning
    a primitive wins. Both fail → TypeError
```

**Why `{} + []` is confusing:** in expression position, `{}` is coerced via ToPrimitive("default") → `"[object Object]"`, `[]` → `""`, result `"[object Object]"`. At the start of a statement, `{}` is a *block*, and `+[]` is unary plus on `[]` → `0`. The value depends on parsing, not coercion.

**ToNumber highlights:**

| Input | Result |
|---|---|
| `""` / `"  "` | `0` |
| `"12"` | `12` |
| `"12px"` | `NaN` |
| `"0x1F"` | `31` |
| `null` | `0` |
| `undefined` | `NaN` |
| `true` / `false` | `1` / `0` |
| `[]` | `0` (→ `""` → `0`) |
| `[5]` | `5` (→ `"5"` → `5`) |
| `[1,2]` | `NaN` (→ `"1,2"` → `NaN`) |
| `{}` | `NaN` |

**Falsy values — the complete list (8 items):**
```
false    0    -0    0n    ""    null    undefined    NaN
```
Everything else is truthy. Including `"0"`, `"false"`, `[]`, `{}`, `function(){}`.

### 3.3 Equality

```
        a == b   (loose)
             │
   ┌─────────┴──────────┐
   │ same type?         │──yes──▶ use === semantics
   └─────────┬──────────┘
             │ no
             ▼
   null == undefined ────────────▶ true (special-cased)
   null/undefined vs anything ───▶ false
   number vs string ─────────────▶ ToNumber(string)
   boolean vs anything ──────────▶ ToNumber(boolean) first
   object vs primitive ──────────▶ ToPrimitive(object)
```

The famous puzzle:

```js
[] == ![]        // true
// ![]        → false        (arrays are truthy)
// [] == false → [] == 0     (boolean→number)
// ToPrimitive([]) → ""      → 0 == 0 → true
```

**Three equality operators:**

| Operator | Name | `NaN` vs `NaN` | `+0` vs `-0` |
|---|---|---|---|
| `==` | loose | `false` | `true` |
| `===` | strict | `false` | `true` |
| `Object.is()` | SameValue | `true` | `false` |

🏭 **Rule:** always `===`. The single legitimate `==` use is `x == null` to check "null or undefined."

---

## 4. Execution Contexts, Scope, Closures

### 4.1 What Happens When a Function Is Called

```
   Creation phase                      Execution phase
   ─────────────────                   ────────────────
   1. Create Environment Record        4. Run statements top to bottom
   2. Bind params + `arguments`        5. Assignments happen
   3. Hoist declarations:                 (TDZ ends when let/const
      - var    → initialized undefined     declaration is evaluated)
      - let/const → uninitialized (TDZ)
      - function decls → fully hoisted
```

```js
console.log(a);   // undefined   (var: hoisted + initialized)
console.log(b);   // ReferenceError (let: hoisted, in TDZ)
console.log(f);   // [Function: f] (fully hoisted)
console.log(g);   // undefined   (var holding a function expression)

var a = 1;
let b = 2;
function f() {}
var g = function () {};
```

⚠️ **TDZ is not "not hoisted."** `let` bindings *are* hoisted — they're created but marked uninitialized. This is observable:

```js
let x = 'outer';
{
  console.log(x);  // ReferenceError, NOT 'outer'
  let x = 'inner'; // the inner x shadows from the block's start
}
```

### 4.2 The Scope Chain

```
     ┌───────────────────────────────────┐
     │   Global Environment              │
     │   { console, window, myGlobal }   │
     └────────────────▲──────────────────┘
                      │ [[OuterEnv]]
     ┌────────────────┴──────────────────┐
     │   outer() Environment             │
     │   { count: 0 }                    │
     └────────────────▲──────────────────┘
                      │ [[OuterEnv]]
     ┌────────────────┴──────────────────┐
     │   inner() Environment             │
     │   { temp: 5 }                     │
     └───────────────────────────────────┘

  Lookup walks UP this chain. Never sideways, never down.
  The chain is fixed at DEFINITION time (lexical), not call time.
```

### 4.3 Closures

🧠 **Definition:** a closure is a function together with the environment records it references. When a function is created, it stores `[[Environment]]` pointing at the scope where it was *defined*. Because that pointer keeps the environment alive, the variables outlive the call that created them.

```js
function counter() {
  let count = 0;                    // lives in counter's env
  return {
    inc: () => ++count,             // both closures share
    get: () => count,               // the SAME env record
  };
}

const c = counter();
c.inc(); c.inc();
c.get();   // 2
```

```
       Heap
   ┌─────────────────────┐
   │ counter env         │
   │   count: 2          │◀──┐
   └─────────────────────┘   │
              ▲               │
              │               │
   ┌──────────┴────┐   ┌─────┴──────────┐
   │ inc function  │   │ get function   │
   │ [[Env]] ──────┘   │ [[Env]] ───────┘
   └───────────────┘   └────────────────┘

   Note: ONE shared environment, not one per closure.
```

**The classic loop bug:**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);   // 3, 3, 3
}

for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);   // 0, 1, 2
}
```

Why: `var` creates **one** binding in the function scope; all three closures point at it, and by the time timers run it's `3`. `let` in a `for` head creates a **fresh binding per iteration**, with the spec explicitly copying the value forward (`CreatePerIterationEnvironment`).

**Practical closure patterns:**

```js
// 1. Private state (module pattern)
const store = (() => {
  const data = new Map();                    // truly private
  return {
    set: (k, v) => data.set(k, v),
    get: (k) => data.get(k),
  };
})();

// 2. Memoization
const memoize = (fn, keyFn = JSON.stringify) => {
  const cache = new Map();
  return (...args) => {
    const key = keyFn(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
};

// 3. Once
const once = (fn) => {
  let called = false, result;
  return (...args) => {
    if (!called) { called = true; result = fn(...args); }
    return result;
  };
};

// 4. Debounce — fire after quiet period
const debounce = (fn, ms) => {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), ms);
  };
};

// 5. Throttle — fire at most once per interval
const throttle = (fn, ms) => {
  let last = 0, timer;
  return (...args) => {
    const now = Date.now();
    const remaining = ms - (now - last);
    if (remaining <= 0) {
      clearTimeout(timer); timer = null;
      last = now;
      fn(...args);
    } else if (!timer) {
      timer = setTimeout(() => {
        last = Date.now(); timer = null;
        fn(...args);
      }, remaining);
    }
  };
};
```

🎤 **Debounce vs throttle** — the classic follow-up:

```
Events:    ●●●●●    ●        ●●●●●●●●●●
           
Debounce:       ▲       ▲              ▲
           (fires after silence — search-as-you-type, resize end)

Throttle:  ▲    ▲    ▲    ▲    ▲    ▲
           (fires at fixed rate — scroll handlers, rate limiting)
```

---

## 5. The `this` Binding System

`this` is determined **at call time** by how the function is invoked, not where it's defined. Except arrow functions, which have no `this` of their own.

### 5.1 The Resolution Order

```
     Is it an arrow function?
              │
      yes ────┴──── no
       │            │
       ▼            ▼
  Lexical this   Called with `new`?
  (from enclosing        │
   scope at         yes ─┴─ no
   definition)       │      │
                     ▼      ▼
             new instance   .call/.apply/.bind?
                                 │
                            yes ─┴─ no
                             │      │
                             ▼      ▼
                       given value  Called as obj.method()?
                                          │
                                     yes ─┴─ no
                                      │      │
                                      ▼      ▼
                                    obj    strict mode?
                                                │
                                          yes ──┴── no
                                           │        │
                                           ▼        ▼
                                       undefined  globalThis
```

### 5.2 The Lost-`this` Trap

```js
const user = {
  name: 'Ada',
  greet() { return `Hi ${this.name}`; },
};

user.greet();                    // "Hi Ada"       ✅
const g = user.greet;
g();                             // "Hi undefined" ⚠️ lost binding
setTimeout(user.greet, 0);       // "Hi undefined" ⚠️ same reason

// Fixes:
setTimeout(user.greet.bind(user), 0);
setTimeout(() => user.greet(), 0);
```

The call `user.greet()` is really "get the function, then call it with receiver `user`." Assigning to `g` drops the receiver.

### 5.3 Implementing bind/call/apply

Interviewers ask for these. Know them cold.

```js
Function.prototype.myCall = function (ctx, ...args) {
  ctx = ctx ?? globalThis;
  const key = Symbol('fn');
  ctx[key] = this;               // `this` is the function being called
  const result = ctx[key](...args);
  delete ctx[key];
  return result;
};

Function.prototype.myApply = function (ctx, args = []) {
  return this.myCall(ctx, ...args);
};

Function.prototype.myBind = function (ctx, ...bound) {
  const fn = this;
  function boundFn(...args) {
    // if called with `new`, ignore ctx and use the fresh instance
    return fn.apply(this instanceof boundFn ? this : ctx, [...bound, ...args]);
  }
  boundFn.prototype = Object.create(fn.prototype ?? null);
  return boundFn;
};
```

⚠️ Arrow functions ignore `call`/`apply`/`bind` entirely for `this`. They also have no `arguments`, no `prototype`, and cannot be used with `new`.

---

## 6. Prototypes and Object Orientation

### 6.1 The Chain

Every object has an internal `[[Prototype]]` link. Property lookup walks it.

```js
class Animal { speak() { return 'generic'; } }
class Dog extends Animal { speak() { return 'woof'; } }
const d = new Dog();
```

```
      d ──────────────▶ Dog.prototype ──────▶ Animal.prototype ──────▶ Object.prototype ──▶ null
      │                  { speak, ctor }       { speak, ctor }         { toString, ... }
      │                        ▲                     ▲
      │                        │                     │
      └─ own props        Dog.__proto__ ──────▶ Animal (constructor fn)
                          (class inheritance links constructors too)

  d.speak()  → not own → found on Dog.prototype   → 'woof'
  d.toString() → walks 3 links → Object.prototype → works
  d.nope     → walks to null → undefined
```

### 6.2 Classes Are Syntax Over Prototypes — Mostly

```js
// These are close, not identical:
class A { m() {} }

function A2() {}
A2.prototype.m = function () {};
```

Differences that matter:

| Feature | `class` | `function` constructor |
|---|---|---|
| Callable without `new` | ❌ TypeError | ✅ (silently broken) |
| Hoisted & usable | ❌ TDZ | ✅ |
| Methods enumerable | ❌ non-enumerable | ✅ enumerable |
| Body strict mode | ✅ always | Depends |
| `super` | ✅ | Manual `Parent.call(this)` |
| Private fields `#x` | ✅ | ❌ (closures only) |

### 6.3 Modern Class Features

```js
class Account {
  #balance = 0;                        // truly private — not accessible outside
  static #count = 0;                   // private static
  static { /* static init block */ }

  constructor(owner) {
    this.owner = owner;
    Account.#count++;
  }

  get balance() { return this.#balance; }          // getter
  set balance(v) {
    if (v < 0) throw new RangeError('negative');
    this.#balance = v;
  }

  deposit(amount) { this.#balance += amount; return this; }  // chainable

  static #validate(x) { return x > 0; }            // private static method
  static create(owner) { return new Account(owner); }

  // Brand check — works even on subclasses
  static isAccount(obj) {
    try { obj.#balance; return true; } catch { return false; }
  }
}
```

⚠️ `#private` is enforced by the engine at compile time — unlike `_convention`, it cannot be reached by `obj['#balance']`, reflection, or `Object.keys`.

### 6.4 Object Creation Comparison

| Method | Prototype | Use case |
|---|---|---|
| `{}` / object literal | `Object.prototype` | Data bags |
| `Object.create(p)` | `p` | Explicit prototype control |
| `Object.create(null)` | `null` | Safe dictionaries (no `__proto__`, no inherited keys) |
| `new Ctor()` | `Ctor.prototype` | Traditional OO |
| `class` | `Class.prototype` | Modern OO |
| `structuredClone(o)` | same as source | Deep copy incl. Map/Set/Date/cycles |

🏭 Use `Object.create(null)` or `Map` for user-controlled keys. A plain object lets an attacker send the key `__proto__` and cause prototype pollution.

---

## 7. The Event Loop

### 7.1 The Algorithm

This is the single highest-yield diagram in JavaScript.

```
┌──────────────────────────────────────────────────────────┐
│                      EVENT LOOP TICK                     │
│                                                          │
│  1. Is the call stack empty?  ── no ──▶ keep running JS  │
│           │ yes                                          │
│           ▼                                              │
│  2. Take ONE task from the MACROTASK queue and run it    │
│     (timers, I/O, DOM events, script evaluation)         │
│           │                                              │
│           ▼                                              │
│  3. Drain the ENTIRE MICROTASK queue                     │
│     (promise callbacks, queueMicrotask, MutationObserver)│
│     ⚠️ microtasks added during drain also run — this can │
│        starve the loop forever                           │
│           │                                              │
│           ▼                                              │
│  4. (Browser) Is it time to render?                      │
│     requestAnimationFrame callbacks → style → layout     │
│     → paint → composite                                  │
│           │                                              │
│           └────────────▶ back to 1                       │
└──────────────────────────────────────────────────────────┘
```

**Key rule:** one macrotask, then *all* microtasks. Not one microtask.

### 7.2 Trace This

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve()
  .then(() => console.log('3'))
  .then(() => console.log('4'));

queueMicrotask(() => console.log('5'));

(async () => {
  console.log('6');
  await null;
  console.log('7');
})();

console.log('8');
```

<details>
<summary>Answer + reasoning</summary>

**Output: `1, 6, 8, 3, 5, 7, 4, 2`**

| Step | Queue action |
|---|---|
| `1` | sync |
| `setTimeout` | → macrotask queue |
| `.then(3)` | → microtask queue |
| `queueMicrotask(5)` | → microtask queue (after 3) |
| `6` | sync — async fn body runs synchronously until first `await` |
| `await null` | rest of function → microtask queue (after 5) |
| `8` | sync |
| — | **stack empty, drain microtasks** |
| `3` | runs; its `.then(4)` → microtask queue (end) |
| `5` | runs |
| `7` | runs |
| `4` | runs |
| — | **microtasks empty → next macrotask** |
| `2` | runs |

</details>

### 7.3 Node.js Phases

Node's loop (libuv) has more structure than the browser's:

```
   ┌───────────────────────────┐
┌─▶│         timers            │  setTimeout, setInterval callbacks
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │    pending callbacks      │  deferred I/O errors (e.g. TCP ECONNREFUSED)
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │     idle, prepare         │  internal
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐      ┌─────────────────┐
│  │          poll             │◀─────│  incoming I/O   │
│  │  (retrieve I/O events;    │      └─────────────────┘
│  │   may BLOCK here)         │
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
│  │          check            │  setImmediate callbacks
│  └────────────┬──────────────┘
│  ┌────────────▼──────────────┐
└──│      close callbacks      │  socket.on('close')
   └───────────────────────────┘

  Between EVERY phase: drain process.nextTick queue, then microtasks.
  nextTick has HIGHER priority than promises.
```

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

// nextTick, promise, then timeout/immediate in nondeterministic order
// (from the main module — inside an I/O callback, immediate always wins)
```

⚠️ `setTimeout` vs `setImmediate` order from the main module is genuinely nondeterministic — it depends on how long process startup took relative to the timer threshold.

### 7.4 Starvation

```js
// ❌ Blocks rendering and all I/O forever
function starve() { Promise.resolve().then(starve); }

// ✅ Yields to the macrotask queue, allowing render + I/O
function polite() { setTimeout(polite, 0); }
```

🏭 For CPU-heavy work in the browser, chunk with `scheduler.yield()` (where available), `setTimeout(0)`, or move to a Web Worker. In Node, use `worker_threads` for CPU-bound tasks — otherwise a single request can block the whole server.

---

## 8. Promises and Async/Await

### 8.1 State Machine

```
                    ┌──────────┐
                    │ PENDING  │
                    └────┬─────┘
             resolve()   │   reject()
          ┌──────────────┴──────────────┐
          ▼                             ▼
   ┌─────────────┐              ┌─────────────┐
   │  FULFILLED  │              │  REJECTED   │
   │  (value)    │              │  (reason)   │
   └─────────────┘              └─────────────┘

   Settled state is PERMANENT and IMMUTABLE.
   Calling resolve twice: second call is ignored.
```

### 8.2 Building a Promise From Scratch

Knowing this removes all mystery.

```js
const PENDING = 'pending', FULFILLED = 'fulfilled', REJECTED = 'rejected';

class MyPromise {
  #state = PENDING;
  #value = undefined;
  #callbacks = [];

  constructor(executor) {
    const resolve = (value) => this.#settle(FULFILLED, value);
    const reject  = (reason) => this.#settle(REJECTED, reason);
    try { executor(resolve, reject); }
    catch (err) { reject(err); }
  }

  #settle(state, value) {
    if (this.#state !== PENDING) return;              // settle once

    // Resolving with a thenable adopts its state
    if (state === FULFILLED && value && typeof value.then === 'function') {
      value.then(
        (v) => this.#settle(FULFILLED, v),
        (e) => this.#settle(REJECTED, e),
      );
      return;
    }

    this.#state = state;
    this.#value = value;
    // ALWAYS async — this is why .then never runs synchronously
    queueMicrotask(() => {
      this.#callbacks.forEach((cb) => cb());
      this.#callbacks = [];
    });
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handle = () => {
        try {
          if (this.#state === FULFILLED) {
            resolve(typeof onFulfilled === 'function'
              ? onFulfilled(this.#value)
              : this.#value);                          // pass-through
          } else {
            if (typeof onRejected === 'function') {
              resolve(onRejected(this.#value));        // recovery!
            } else {
              reject(this.#value);                     // propagate
            }
          }
        } catch (err) { reject(err); }
      };
      this.#state === PENDING
        ? this.#callbacks.push(handle)
        : queueMicrotask(handle);
    });
  }

  catch(onRejected) { return this.then(undefined, onRejected); }

  finally(onFinally) {
    return this.then(
      (v) => { onFinally(); return v; },
      (e) => { onFinally(); throw e; },
    );
  }

  static resolve(v) {
    return v instanceof MyPromise ? v : new MyPromise((res) => res(v));
  }
  static reject(e) { return new MyPromise((_, rej) => rej(e)); }
}
```

Three insights this code makes obvious:
1. `.then` always returns a **new** promise → chaining works.
2. Callbacks are always deferred via `queueMicrotask` → never synchronous.
3. A `catch` handler that returns normally **recovers** the chain.

### 8.3 Combinators

| Method | Resolves when | Rejects when | Result |
|---|---|---|---|
| `all` | all fulfill | **any** rejects (fail fast) | array of values |
| `allSettled` | all settle | never | array of `{status, value/reason}` |
| `race` | first **settles** | first settles as rejection | that value/reason |
| `any` | first **fulfills** | all reject | value / `AggregateError` |

```
Input:   P1 ──✓──▶
         P2 ────✗────▶
         P3 ──────✓──────▶

all        ✗ at P2's rejection (fail fast)
allSettled ✓ after P3 with all three outcomes
race       ✓ at P1 (first to settle)
any        ✓ at P1 (first to fulfill)
```

### 8.4 Async/Await Mechanics

`async function` is a generator over promises. Every `await` splits the function:

```js
async function f() {
  console.log('A');          // synchronous — runs on the caller's stack
  const x = await g();       // suspend; remainder queued as a microtask
  console.log('B', x);       // resumed in a microtask
  return x * 2;              // resolves f's promise
}
```

```
   Caller stack           Microtask queue
   ────────────           ───────────────
   f() called
   'A' logged
   g() called, awaits ──▶  [resume f with g's value]
   caller continues...
   (stack empties)
                      ◀──  resume f: log 'B', resolve f's promise
```

### 8.5 Practical Async Patterns

```js
// ── Sequential vs parallel ─────────────────────────────
// ❌ 3 seconds
const a = await fetchA();  const b = await fetchB();  const c = await fetchC();

// ✅ 1 second
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// ⚠️ Subtle: start early, await later
const pa = fetchA(), pb = fetchB();   // both in flight now
const ra = await pa, rb = await pb;

// ── Timeout wrapper ────────────────────────────────────
const withTimeout = (promise, ms) => {
  const ac = new AbortController();
  const timer = setTimeout(() => ac.abort(), ms);
  return Promise.race([
    promise,
    new Promise((_, rej) =>
      ac.signal.addEventListener('abort', () =>
        rej(new Error(`Timeout after ${ms}ms`)))),
  ]).finally(() => clearTimeout(timer));
};

// ── Retry with exponential backoff + jitter ────────────
async function retry(fn, { attempts = 3, baseMs = 300, factor = 2 } = {}) {
  let lastErr;
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (err) {
      lastErr = err;
      if (i === attempts - 1) break;
      const backoff = baseMs * factor ** i;
      const jitter = Math.random() * backoff;   // avoid thundering herd
      await new Promise((r) => setTimeout(r, backoff + jitter));
    }
  }
  throw lastErr;
}

// ── Concurrency limiter (pool) ─────────────────────────
async function pool(tasks, limit = 5) {
  const results = new Array(tasks.length);
  let cursor = 0;
  const workers = Array.from({ length: Math.min(limit, tasks.length) },
    async () => {
      while (cursor < tasks.length) {
        const i = cursor++;
        results[i] = await tasks[i]();
      }
    });
  await Promise.all(workers);
  return results;
}

// ── Async iteration ────────────────────────────────────
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next);
    const { items, nextUrl } = await res.json();
    yield* items;                 // yield each item
    next = nextUrl;
  }
}
for await (const item of paginate('/api/items')) { /* ... */ }
```

⚠️ **`forEach` does not await.**
```js
// ❌ returns immediately, unhandled rejections
items.forEach(async (i) => { await save(i); });

// ✅ sequential
for (const i of items) await save(i);

// ✅ parallel
await Promise.all(items.map((i) => save(i)));
```

### 8.6 Error Handling

```js
// Rejection propagates through the chain until a handler is found
fetchUser()
  .then((u) => fetchOrders(u.id))   // if this throws...
  .then((o) => render(o))            // ...this is SKIPPED
  .catch((e) => showError(e))        // ...and this catches
  .finally(() => hideSpinner());     // always runs

// catch placement matters
p.then(onOk).catch(onErr);   // catches errors from p AND from onOk
p.then(onOk, onErr);         // catches errors from p ONLY
```

🏭 Always attach a global handler:
```js
// Browser
window.addEventListener('unhandledrejection', (e) => {
  report(e.reason); e.preventDefault();
});
// Node — an unhandled rejection terminates the process by default
process.on('unhandledRejection', (reason) => { log(reason); process.exit(1); });
```

---

## 9. Memory Management and Leaks

### 9.1 Generational GC

```
        ┌────────────────── HEAP ──────────────────┐
        │                                          │
        │  YOUNG GENERATION (new space, small)     │
        │  ┌──────────┐  ┌──────────┐              │
        │  │  From    │──│    To    │  Scavenger:  │
        │  │  space   │  │  space   │  copy live   │
        │  └──────────┘  └──────────┘  objects,    │
        │       most objects die here   swap spaces│
        │                    │                     │
        │        survived 2 scavenges              │
        │                    ▼                     │
        │  OLD GENERATION (large)                  │
        │  ┌────────────────────────────────────┐  │
        │  │ Mark-Compact:                      │  │
        │  │  1. Mark reachable from roots      │  │
        │  │  2. Sweep unreachable              │  │
        │  │  3. Compact to reduce fragmentation│  │
        │  │  (incremental + concurrent to      │  │
        │  │   avoid long pauses)               │  │
        │  └────────────────────────────────────┘  │
        └──────────────────────────────────────────┘
```

**Generational hypothesis:** most objects die young. So collect the young space often and cheaply; the old space rarely and expensively.

**Roots:** global object, current call stack, active closures' environments, and handles held by the host.

### 9.2 The Five Common Leaks

| # | Leak | Symptom | Fix |
|---|---|---|---|
| 1 | **Accidental globals** | `function f(){ x = 1 }` in sloppy mode | `'use strict'` |
| 2 | **Forgotten timers** | `setInterval` on an unmounted component | `clearInterval` in cleanup |
| 3 | **Detached DOM** | Node removed but still in a JS array/closure | Null the reference |
| 4 | **Unremoved listeners** | Handler closes over a big object | `removeEventListener` / `AbortController` |
| 5 | **Unbounded caches** | Map grows forever | LRU eviction, `WeakMap`, or TTL |

```js
// Detached DOM leak
const cache = [];
function add() {
  const el = document.createElement('div');
  document.body.append(el);
  cache.push(el);                 // ⚠️ keeps element alive after removal
}
el.remove();                      // removed from DOM, NOT from memory

// AbortController cleans up many listeners at once
const ac = new AbortController();
el.addEventListener('click', onClick, { signal: ac.signal });
el.addEventListener('scroll', onScroll, { signal: ac.signal });
ac.abort();                       // removes BOTH
```

### 9.3 Weak References

```js
// WeakMap: key held weakly — entry vanishes when key is collected
const metadata = new WeakMap();
metadata.set(domNode, { clicks: 0 });   // no leak when domNode is removed

// WeakRef + FinalizationRegistry (advanced, use sparingly)
const registry = new FinalizationRegistry((key) => {
  console.log(`${key} was collected`);
});
```

| Structure | Keys | Iterable | Use |
|---|---|---|---|
| `Map` | any | ✅ | General key→value |
| `WeakMap` | objects only | ❌ | Attaching data to objects you don't own |
| `Set` | any | ✅ | Uniqueness |
| `WeakSet` | objects only | ❌ | "Have I seen this object?" |

### 9.4 Finding Leaks

```
1. DevTools → Memory → Heap Snapshot
2. Perform the suspect action (open/close a modal 10×)
3. Force GC (🗑 icon)
4. Take a second snapshot
5. Comparison view → sort by "Delta"
6. Objects that grow every cycle = your leak
7. Click one → "Retainers" panel → shows the chain keeping it alive
```

---

## 10. Modules

### 10.1 ESM vs CommonJS

| Aspect | ESM (`import`) | CJS (`require`) |
|---|---|---|
| Loading | Static, parsed before execution | Dynamic, runtime |
| Binding | **Live bindings** (references) | Value copy at require time |
| Async | Top-level `await` supported | Synchronous only |
| Tree-shakeable | ✅ (static analysis) | ❌ (mostly) |
| Circular deps | Handled via hoisted bindings (may hit TDZ) | Partial exports object |
| `this` at top level | `undefined` | `module.exports` |
| File extension | `.mjs` or `"type":"module"` | `.cjs` or default |

```js
// LIVE BINDING — this is the key difference
// counter.mjs
export let count = 0;
export const inc = () => ++count;

// main.mjs
import { count, inc } from './counter.mjs';
console.log(count);  // 0
inc();
console.log(count);  // 1  ← updated! CJS would still show 0
```

### 10.2 The Module Lifecycle

```
   ┌──────────────┐
   │ CONSTRUCTION │  Fetch, parse, build module record.
   │              │  Discover imports → recurse (build graph).
   └──────┬───────┘
   ┌──────▼───────┐
   │ INSTANTIATION│  Allocate memory for exports.
   │              │  Wire imports to exports (live bindings).
   │              │  Values not yet computed → TDZ.
   └──────┬───────┘
   ┌──────▼───────┐
   │  EVALUATION  │  Run module bodies, depth-first,
   │              │  each exactly once. Fill in the values.
   └──────────────┘
```

This three-phase design is why `import` is hoisted, why circular imports sometimes work, and why tree-shaking is possible.

---

## 11. Iterators, Generators, Symbols

### 11.1 The Iteration Protocol

```js
// Anything with Symbol.iterator works in for...of, spread, destructuring
const range = {
  from: 1, to: 5,
  [Symbol.iterator]() {
    let cur = this.from, last = this.to;
    return {
      next: () => cur <= last
        ? { value: cur++, done: false }
        : { value: undefined, done: true },
    };
  },
};
[...range];                       // [1,2,3,4,5]
for (const n of range) {}         // works
const [a, b] = range;             // works
```

### 11.2 Generators

Generators are resumable functions — the only place JavaScript can pause mid-execution.

```js
function* gen() {
  const a = yield 1;      // pauses; `a` = value passed to next()
  const b = yield a + 1;
  return a + b;
}

const it = gen();
it.next();      // { value: 1, done: false }
it.next(10);    // { value: 11, done: false }   a = 10
it.next(20);    // { value: 30, done: true }    b = 20
```

```
     Caller                    Generator
       │                           │
   next()  ──────────────────────▶ run to first yield
       │ ◀──────────  {value:1}    (paused)
   next(10) ─────────────────────▶ a=10, run to next yield
       │ ◀──────────  {value:11}   (paused)
   next(20) ─────────────────────▶ b=20, run to return
       │ ◀── {value:30, done:true} (finished)
```

**Practical uses:**

```js
// Infinite lazy sequences
function* naturals() { let n = 1; while (true) yield n++; }
function* take(it, n) { let i = 0; for (const v of it) { if (i++ >= n) return; yield v; } }
[...take(naturals(), 5)];  // [1,2,3,4,5]

// Delegation
function* inner() { yield 1; yield 2; }
function* outer() { yield 0; yield* inner(); yield 3; }  // 0,1,2,3

// Async generators for streams
async function* readLines(stream) {
  let buffer = '';
  for await (const chunk of stream) {
    buffer += chunk;
    const lines = buffer.split('\n');
    buffer = lines.pop();
    yield* lines;
  }
  if (buffer) yield buffer;
}
```

### 11.3 Well-Known Symbols

| Symbol | Controls |
|---|---|
| `Symbol.iterator` | `for...of`, spread |
| `Symbol.asyncIterator` | `for await...of` |
| `Symbol.toPrimitive` | Coercion (overrides valueOf/toString) |
| `Symbol.toStringTag` | `Object.prototype.toString.call(x)` |
| `Symbol.hasInstance` | `instanceof` behaviour |
| `Symbol.for(k)` | Global symbol registry (cross-realm) |

```js
class Money {
  constructor(cents) { this.cents = cents; }
  [Symbol.toPrimitive](hint) {
    if (hint === 'number') return this.cents;
    if (hint === 'string') return `$${(this.cents / 100).toFixed(2)}`;
    return `Money(${this.cents})`;
  }
  get [Symbol.toStringTag]() { return 'Money'; }
}
const m = new Money(1250);
+m;             // 1250
`${m}`;         // "$12.50"
m + '';         // "Money(1250)"
String(Object.prototype.toString.call(m));  // "[object Money]"
```

---

## 12. Proxies and Metaprogramming

```js
const validated = (target, schema) => new Proxy(target, {
  get(obj, prop, receiver) {
    if (!(prop in obj)) throw new ReferenceError(`No property ${String(prop)}`);
    return Reflect.get(obj, prop, receiver);
  },
  set(obj, prop, value, receiver) {
    const rule = schema[prop];
    if (rule && !rule(value)) throw new TypeError(`Invalid ${String(prop)}`);
    return Reflect.set(obj, prop, value, receiver);
  },
  deleteProperty() { throw new Error('Immutable'); },
});

const user = validated({ age: 30 }, { age: (v) => Number.isInteger(v) && v >= 0 });
user.age = -5;    // TypeError
user.nope;        // ReferenceError
```

**All 13 traps:** `get`, `set`, `has`, `deleteProperty`, `apply`, `construct`, `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `preventExtensions`, `getOwnPropertyDescriptor`, `defineProperty`, `ownKeys`.

🏭 This is exactly how Vue 3 reactivity, Immer drafts, and many ORM lazy-loaders work. Cost: proxied access is meaningfully slower than direct access — never proxy a hot path.

Always use `Reflect` inside traps: it forwards `receiver` correctly so getters on prototypes see the right `this`.

---

## 13. Functional Patterns

```js
// Composition
const pipe  = (...fns) => (x) => fns.reduce((v, f) => f(v), x);
const compose = (...fns) => (x) => fns.reduceRight((v, f) => f(v), x);

// Currying
const curry = (fn) => {
  const curried = (...args) =>
    args.length >= fn.length ? fn(...args) : (...rest) => curried(...args, ...rest);
  return curried;
};
const add3 = curry((a, b, c) => a + b + c);
add3(1)(2)(3);  // 6

// Deep clone with cycle handling
function deepClone(obj, seen = new WeakMap()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (seen.has(obj)) return seen.get(obj);              // cycles

  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof RegExp) return new RegExp(obj.source, obj.flags);
  if (obj instanceof Map) {
    const m = new Map(); seen.set(obj, m);
    obj.forEach((v, k) => m.set(deepClone(k, seen), deepClone(v, seen)));
    return m;
  }
  if (obj instanceof Set) {
    const s = new Set(); seen.set(obj, s);
    obj.forEach((v) => s.add(deepClone(v, seen)));
    return s;
  }

  const clone = Array.isArray(obj) ? [] : Object.create(Object.getPrototypeOf(obj));
  seen.set(obj, clone);
  for (const key of Reflect.ownKeys(obj)) {             // includes symbols
    clone[key] = deepClone(obj[key], seen);
  }
  return clone;
}
// Built-in alternative: structuredClone(obj) — handles cycles, Map, Set, Date,
// but NOT functions, DOM nodes, or prototypes.

// Flatten with depth
const flatten = (arr, depth = 1) =>
  depth < 1 ? arr.slice()
    : arr.reduce((acc, v) => acc.concat(Array.isArray(v) ? flatten(v, depth - 1) : v), []);

// Group by (now built in: Object.groupBy / Map.groupBy)
const groupBy = (arr, keyFn) =>
  arr.reduce((acc, x) => { (acc[keyFn(x)] ??= []).push(x); return acc; }, {});

// EventEmitter
class Emitter {
  #handlers = new Map();
  on(ev, fn)   { (this.#handlers.get(ev) ?? this.#handlers.set(ev, new Set()).get(ev)).add(fn); return () => this.off(ev, fn); }
  off(ev, fn)  { this.#handlers.get(ev)?.delete(fn); }
  once(ev, fn) { const w = (...a) => { this.off(ev, w); fn(...a); }; return this.on(ev, w); }
  emit(ev, ...a) { this.#handlers.get(ev)?.forEach((fn) => fn(...a)); }
}
```

---

## 14. Performance Engineering

### 14.1 Rules That Actually Matter

| Rule | Why |
|---|---|
| Measure before optimizing | Intuition about JS perf is usually wrong |
| Keep object shapes stable | Avoids megamorphic ICs and dictionary mode |
| Avoid `delete` on hot objects | Forces dictionary mode |
| Keep arrays packed & homogeneous | Element-kind transitions are one-way |
| Avoid `try/catch` in the hottest loop | (Much less true in modern V8, but still measurable) |
| Batch DOM reads and writes | Prevents layout thrashing |
| Move CPU work off the main thread | Workers keep the UI at 60fps |

### 14.2 Layout Thrashing

```js
// ❌ Forced synchronous layout on every iteration — O(n) reflows
for (const el of elements) {
  el.style.height = el.offsetHeight + 10 + 'px';  // write then read then write...
}

// ✅ Read all, then write all — 1 reflow
const heights = elements.map((el) => el.offsetHeight);   // READ phase
elements.forEach((el, i) => { el.style.height = heights[i] + 10 + 'px'; }); // WRITE
```

```
   ❌  R W R W R W R W    →  layout recomputed 4×
   ✅  R R R R W W W W    →  layout recomputed 1×
```

### 14.3 Measuring

```js
performance.mark('start');
doWork();
performance.mark('end');
performance.measure('work', 'start', 'end');
console.table(performance.getEntriesByType('measure'));

// Long tasks (>50ms) block interaction
new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) console.warn('Long task', entry.duration);
}).observe({ entryTypes: ['longtask'] });
```

---

## 15. Modern JavaScript

```js
// Optional chaining + nullish coalescing
const city = user?.address?.city ?? 'Unknown';
const port = config.port ?? 3000;      // 0 is preserved (unlike ||)
obj.method?.();                        // only calls if it exists
arr?.[0];

// Logical assignment
opts.retries ??= 3;    // assign if null/undefined
flags.debug ||= false;
cache.hit &&= cache.hit + 1;

// Array methods that don't mutate (ES2023)
arr.toSorted();  arr.toReversed();  arr.toSpliced(1, 2);  arr.with(0, 'x');
arr.at(-1);      // last element
arr.findLast((x) => x > 3);
arr.flat(Infinity);

// Object/Map grouping (ES2024)
Object.groupBy(people, (p) => p.dept);
Map.groupBy(people, (p) => p.manager);   // object keys allowed

// Structured clone
const copy = structuredClone(complexObject);

// Top-level await (ESM only)
const config = await fetch('/config.json').then((r) => r.json());

// Error cause
throw new Error('Query failed', { cause: originalError });

// Promise.withResolvers (ES2024)
const { promise, resolve, reject } = Promise.withResolvers();

// Numeric separators + BigInt
const billion = 1_000_000_000;
const huge = 9007199254740993n;

// String methods
'abc'.replaceAll('a', 'z');
'  x '.trimStart().trimEnd();
'5'.padStart(3, '0');    // "005"
```

---

## 16. Interview Section

### 🎤 Conceptual

<details>
<summary><b>Q1. Explain the event loop to a non-JS engineer.</b></summary>

JavaScript has one thread that runs your code. When you ask for something slow — a network call, a timer, a file read — JavaScript doesn't wait. It hands the job to the runtime and immediately continues with the next line. When the job finishes, the runtime puts a callback in a queue.

The event loop is a simple rule: *when the call stack is empty, take the next callback from the queue and run it.*

There are two queues with different priority. Microtasks (promise callbacks) drain completely after every single macrotask (timers, I/O, events). That's the whole model. Almost every async question is answered by asking "which queue does this land in, and is the stack empty?"
</details>

<details>
<summary><b>Q2. What is a closure and why does it matter?</b></summary>

A closure is a function bundled with the variable environment where it was defined. When JavaScript creates a function, it stores a hidden pointer to the scope it was born in. That pointer keeps the scope alive after the outer function returns.

It matters because it's the mechanism behind private state, module encapsulation, memoization, and every callback that "remembers" something. It's also the mechanism behind React hooks — `useState` returns a setter that closes over the fiber's slot.

The cost: closures retain memory. A closure over a large object keeps that object alive as long as the closure lives.
</details>

<details>
<summary><b>Q3. `null` vs `undefined`?</b></summary>

`undefined` means "no value has been assigned" — the language produces it: uninitialized variables, missing parameters, missing properties, functions with no return. `null` means "intentionally empty" — a programmer assigns it.

Practically: default parameters trigger on `undefined` but not `null`; `JSON.stringify` drops `undefined` properties but keeps `null`; `typeof null === "object"` is a bug from 1995. Use `== null` to check for both.
</details>

<details>
<summary><b>Q4. How does prototypal inheritance differ from classical?</b></summary>

Classical inheritance copies structure from a class blueprint into an instance. Prototypal inheritance links objects to other objects — nothing is copied. Lookup walks the link chain at access time.

Consequences: you can modify a prototype at runtime and every existing instance sees the change immediately. You can create objects from other objects without any class. `class` syntax in JS is sugar; underneath, `class Dog extends Animal` just sets `Dog.prototype.__proto__ = Animal.prototype`.
</details>

<details>
<summary><b>Q5. What is the TDZ?</b></summary>

The Temporal Dead Zone is the span between when a `let`/`const` binding is created (at scope entry) and when its declaration is evaluated. Accessing it in that window throws a `ReferenceError`.

It exists to catch use-before-declare bugs that `var` silently swallowed as `undefined`, and to make `const` sound — a `const` can't be observed before its one and only assignment.
</details>

<details>
<summary><b>Q6. Deep vs shallow copy — how would you deep-copy safely?</b></summary>

Shallow copy (`{...o}`, `Object.assign`, `slice`) copies one level; nested objects are shared references.

For deep copy, `structuredClone()` is the built-in answer — it handles cycles, `Map`, `Set`, `Date`, `ArrayBuffer`. It cannot clone functions, DOM nodes, or class prototypes.

`JSON.parse(JSON.stringify(x))` is the common hack and it silently destroys `undefined`, functions, `Symbol`, `Date` (→string), `NaN`/`Infinity` (→null), and throws on cycles.

For full control, write a recursive clone with a `WeakMap` to track visited nodes (see §13).
</details>

<details>
<summary><b>Q7. What causes memory leaks in JS?</b></summary>

Anything that keeps a reference the program no longer needs: accidental globals, timers never cleared, event listeners never removed, detached DOM nodes still in a JS array, and unbounded caches.

The GC is precise — it collects everything unreachable. So every leak is a bug where you're still reachable. Diagnose with two heap snapshots around a repeated action and read the retainer chain.
</details>

<details>
<summary><b>Q8. Explain `async`/`await` in terms of promises.</b></summary>

`async` marks a function as returning a promise. `await` suspends the function until a promise settles, then resumes it in a microtask with the resolved value — or throws the rejection reason at the await site, so `try/catch` works.

Mechanically it's a state machine: the compiler splits the function at each `await` and registers the remainder as a `.then` callback. The body up to the first `await` runs synchronously on the caller's stack — a detail people miss.
</details>

<details>
<summary><b>Q9. Event delegation — what and why?</b></summary>

Attach one listener to a common ancestor instead of one per child, then use `event.target` to figure out which child was hit. It works because DOM events bubble up.

Benefits: constant memory regardless of child count, and it automatically handles elements added later. Essential for large lists and dynamic content.

```js
list.addEventListener('click', (e) => {
  const item = e.target.closest('[data-id]');
  if (item && list.contains(item)) handle(item.dataset.id);
});
```
</details>

<details>
<summary><b>Q10. What is prototype pollution?</b></summary>

If code merges attacker-controlled JSON into an object without guarding keys, an attacker sends `{"__proto__": {"isAdmin": true}}`. The merge writes to `Object.prototype`, and now *every* object in the program appears to have `isAdmin === true`.

Defenses: reject `__proto__`/`constructor`/`prototype` keys, use `Object.create(null)` or `Map` for user-keyed data, and `Object.freeze(Object.prototype)` in hardened environments.
</details>

### 🎤 Output prediction

<details>
<summary><b>P1</b></summary>

```js
console.log(typeof typeof 1);
```
**`"string"`** — inner `typeof 1` is `"number"`, `typeof "number"` is `"string"`.
</details>

<details>
<summary><b>P2</b></summary>

```js
const obj = { a: 1 };
const copy = { ...obj, b: { c: 2 } };
const copy2 = { ...copy };
copy2.b.c = 99;
console.log(copy.b.c);
```
**`99`** — spread is shallow; `b` is the same object in both.
</details>

<details>
<summary><b>P3</b></summary>

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));
for (let j = 0; j < 3; j++) setTimeout(() => console.log(j));
```
**`3 3 3 0 1 2`** — see §4.3.
</details>

<details>
<summary><b>P4</b></summary>

```js
const o = {
  name: 'a',
  regular() { return this?.name; },
  arrow: () => this?.name,
};
console.log(o.regular(), o.arrow());
```
**`a undefined`** — the arrow captured `this` from the module/global scope where the object literal was written, not from `o`.
</details>

<details>
<summary><b>P5</b></summary>

```js
console.log([1,2,3] + [4,5,6]);
```
**`"1,2,34,5,6"`** — `+` on two objects coerces both to primitives; arrays' `toString` joins with commas, then string concatenation.
</details>

<details>
<summary><b>P6</b></summary>

```js
console.log(0.1 + 0.2 === 0.3);
console.log(0.1 + 0.2);
```
**`false`**, **`0.30000000000000004`** — IEEE-754 binary floats cannot represent 0.1 or 0.2 exactly. Compare with an epsilon: `Math.abs(a - b) < Number.EPSILON`. For money, use integer cents or a decimal library.
</details>

<details>
<summary><b>P7</b></summary>

```js
async function f() {
  console.log(1);
  await console.log(2);
  console.log(3);
}
console.log(0); f(); console.log(4);
```
**`0 1 2 4 3`** — the body runs synchronously through the `await` *expression*; only the continuation is deferred.
</details>

<details>
<summary><b>P8</b></summary>

```js
const s = new Set([1, 1, 2, NaN, NaN, -0, 0]);
console.log(s.size);
```
**`3`** — Set uses SameValueZero: duplicates removed, `NaN` equals itself, `-0` and `0` are the same. Contents: `{1, 2, NaN}`.
</details>

<details>
<summary><b>P9</b></summary>

```js
function f() { return { a: 1 }; }
function g() {
  return
  { a: 1 };
}
console.log(f(), g());
```
**`{a:1} undefined`** — automatic semicolon insertion puts a `;` right after `return` in `g`.
</details>

<details>
<summary><b>P10</b></summary>

```js
const arr = [10, 1, 5, 100];
console.log(arr.sort());
```
**`[1, 10, 100, 5]`** — default sort converts to strings and compares lexicographically. Always pass a comparator: `arr.sort((a,b) => a-b)`.
</details>

### 🎤 Implementation drills

Implement from scratch, no libraries:

1. `Promise.all`, `Promise.race`, `Promise.allSettled`, `Promise.any`
2. `Array.prototype.map/filter/reduce/flat`
3. `debounce` (with leading/trailing options and `cancel`)
4. `throttle`
5. `deepEqual`
6. `curry`
7. `EventEmitter`
8. An LRU cache in O(1)
9. `bind` supporting `new`
10. A promise-based concurrency pool
11. `deepClone` with cycles
12. A simple virtual-DOM diff

```js
// Promise.all from scratch — the canonical answer
Promise.myAll = (promises) => new Promise((resolve, reject) => {
  const items = [...promises];
  const results = new Array(items.length);
  let remaining = items.length;
  if (remaining === 0) return resolve([]);
  items.forEach((p, i) => {
    Promise.resolve(p).then(
      (v) => { results[i] = v; if (--remaining === 0) resolve(results); },
      reject,                                    // first rejection wins
    );
  });
});
```

---

## 17. Cheat Sheet

```
╔═══════════════════════════════════════════════════════════════════════╗
║                      JAVASCRIPT — ONE PAGE                            ║
╠═══════════════════════════════════════════════════════════════════════╣
║ FALSY (8): false 0 -0 0n "" null undefined NaN                        ║
║ typeof null === "object"    Array.isArray() for arrays                ║
║ === always; == only as `x == null`                                    ║
╠═══════════════════════════════════════════════════════════════════════╣
║ EVENT LOOP:  1 macrotask → ALL microtasks → maybe render → repeat     ║
║   macro: setTimeout, setInterval, I/O, events, setImmediate(node)     ║
║   micro: promises, queueMicrotask, MutationObserver                   ║
║   node extra: process.nextTick runs BEFORE promises                   ║
╠═══════════════════════════════════════════════════════════════════════╣
║ `this`:  arrow→lexical | new→instance | .call→given                   ║
║          obj.m()→obj | bare→undefined(strict)/global(sloppy)          ║
╠═══════════════════════════════════════════════════════════════════════╣
║ SCOPE:  var→function, hoisted=undefined                               ║
║         let/const→block, hoisted but TDZ                              ║
║         closure = fn + its defining environment                       ║
╠═══════════════════════════════════════════════════════════════════════╣
║ PROMISE COMBINATORS                                                   ║
║   all        → all fulfill | fail fast                                ║
║   allSettled → never rejects                                          ║
║   race       → first to settle                                        ║
║   any        → first to fulfill | AggregateError                      ║
╠═══════════════════════════════════════════════════════════════════════╣
║ PERF:  stable shapes • packed arrays • no delete • batch DOM R/W      ║
║        workers for CPU • measure with performance.mark                ║
╠═══════════════════════════════════════════════════════════════════════╣
║ LEAKS: globals • timers • listeners • detached DOM • unbounded cache  ║
║        WeakMap/WeakSet for object-keyed data                          ║
╠═══════════════════════════════════════════════════════════════════════╣
║ MODERN: ?. ?? ??= at(-1) toSorted() structuredClone()                 ║
║         Object.groupBy() Promise.withResolvers() error cause          ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [TypeScript](typescript.md) · [React](../02-frontend/react.md) · [Node.js](../03-backend/nodejs.md) · [Browser & Performance](../02-frontend/browser-performance.md)
