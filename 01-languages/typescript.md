# 🔷 TypeScript Mastery

> TypeScript is a language for describing the *shape of possible values*. Once you see the type system as a small functional language that runs at compile time, everything clicks.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Type Lattice](#2-the-type-lattice)
3. [Structural Typing](#3-structural-typing)
4. [Narrowing and Control Flow Analysis](#4-narrowing-and-control-flow-analysis)
5. [Generics](#5-generics)
6. [Conditional Types](#6-conditional-types)
7. [Mapped Types](#7-mapped-types)
8. [Template Literal Types](#8-template-literal-types)
9. [Variance](#9-variance)
10. [Declaration Merging and Modules](#10-declaration-merging-and-modules)
11. [Utility Types — Complete Reference](#11-utility-types)
12. [Type-Level Programming](#12-type-level-programming)
13. [Practical Patterns](#13-practical-patterns)
14. [Configuration & Compiler](#14-configuration--compiler)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. Mental Model

🧠 **TypeScript is a set-theory language.** A type is a *set of values*. Everything else follows:

| Type concept | Set concept |
|---|---|
| `A extends B` | A ⊆ B (A is assignable to B) |
| `A \| B` | union A ∪ B |
| `A & B` | intersection A ∩ B |
| `never` | ∅ (empty set) |
| `unknown` | universal set |
| `any` | escape hatch — disables set reasoning |

```
                    ┌─────────────────────────────┐
                    │         unknown             │  everything
                    │  ┌───────────────────────┐  │
                    │  │  {} (non-null object  │  │
                    │  │      or primitive)    │  │
                    │  │  ┌─────────┬────────┐ │  │
                    │  │  │ string  │ number │ │  │
                    │  │  │ ┌─────┐ │ ┌────┐ │ │  │
                    │  │  │ │"a"  │ │ │ 42 │ │ │  │  literal types
                    │  │  │ └─────┘ │ └────┘ │ │  │
                    │  │  └─────────┴────────┘ │  │
                    │  └───────────────────────┘  │
                    │        null   undefined     │
                    │           ┌───────┐         │
                    │           │ never │         │  nothing
                    │           └───────┘         │
                    └─────────────────────────────┘

  `any` is not in this picture — it opts OUT of the lattice entirely.
```

⚠️ **The counterintuitive part:** a *narrower* type has *more* members in its object shape. `{a: 1, b: 2}` is a subtype of `{a: number}` because the set of objects with both properties is smaller than the set with just `a`.

---

## 2. The Type Lattice

### 2.1 Top Types: `unknown` vs `any`

```ts
let u: unknown = getData();
let a: any     = getData();

u.foo;              // ❌ Error — must narrow first
a.foo;              // ✅ (silently unsafe)

if (typeof u === 'string') u.toUpperCase();   // ✅ narrowed

const s1: string = u;   // ❌ unknown not assignable
const s2: string = a;   // ✅ any is assignable to everything
```

| | `any` | `unknown` |
|---|---|---|
| Assign anything **to** it | ✅ | ✅ |
| Assign it **to** anything | ✅ | ❌ (only `unknown`/`any`) |
| Access properties | ✅ (unsafe) | ❌ until narrowed |
| Purpose | escape hatch | safe boundary type |

🏭 **Rule:** all external data (`JSON.parse`, `fetch`, `catch (e)`) is `unknown`. Narrow it with a validator before it enters your code.

### 2.2 Bottom Type: `never`

`never` is the type with no values. It appears when:

```ts
function fail(msg: string): never { throw new Error(msg); }   // never returns
function loop(): never { while (true) {} }                    // never terminates

type Impossible = string & number;                            // ∅
type Filtered = Exclude<'a' | 'b', 'a' | 'b'>;                // never

// Exhaustiveness checking — the killer app for never
type Shape =
  | { kind: 'circle'; r: number }
  | { kind: 'square'; side: number }
  | { kind: 'rect'; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case 'circle': return Math.PI * s.r ** 2;
    case 'square': return s.side ** 2;
    case 'rect':   return s.w * s.h;
    default: {
      const _exhaustive: never = s;    // ✅ compiles only if all cases handled
      throw new Error(`Unhandled: ${JSON.stringify(_exhaustive)}`);
    }
  }
}
```

Add a fourth variant to `Shape` and the `default` branch fails to compile with a precise error. This is the single most valuable pattern in the language for maintaining large codebases.

⚠️ `never` in a union vanishes: `string | never` is `string`. This is why `Exclude` works.

### 2.3 Literal Types and Widening

```ts
let a = 'hello';           // widened to string  (mutable → could change)
const b = 'hello';         // literal type "hello" (immutable)

let c: 'hello' = 'hello';  // explicitly narrow

const obj = { x: 'a' };    // { x: string }  — properties widen
const obj2 = { x: 'a' } as const;  // { readonly x: "a" }

// const assertions
const tuple = [1, 2] as const;              // readonly [1, 2]
const arr   = [1, 2];                       // number[]

// Preserving literals in functions
function f<T extends string>(x: T): T { return x; }
const r = f('abc');        // "abc", not string
```

---

## 3. Structural Typing

TypeScript compares types by **shape**, not by name (unlike Java/C#).

```ts
interface Point { x: number; y: number; }
class Vec { constructor(public x: number, public y: number) {} }

const p: Point = new Vec(1, 2);   // ✅ compatible — same shape
const q: Point = { x: 1, y: 2 };  // ✅ same shape
```

### 3.1 Excess Property Checking

An exception exists for object *literals* assigned directly:

```ts
const a: Point = { x: 1, y: 2, z: 3 };   // ❌ Error: 'z' does not exist

const tmp = { x: 1, y: 2, z: 3 };
const b: Point = tmp;                     // ✅ no error — not a fresh literal
```

This is a deliberate typo-catching heuristic, not a soundness rule.

### 3.2 Branded / Nominal Types

Sometimes you *want* nominal typing — a `UserId` shouldn't be assignable to an `OrderId` even though both are strings.

```ts
declare const brand: unique symbol;
type Brand<T, B> = T & { readonly [brand]: B };

type UserId  = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;

const asUserId = (s: string): UserId => s as UserId;

function getUser(id: UserId) {}
getUser(asUserId('u1'));         // ✅
getUser('u1');                   // ❌ plain string rejected
declare const oid: OrderId;
getUser(oid);                    // ❌ wrong brand
```

🏭 Brands cost nothing at runtime (the symbol property never exists) and prevent a whole class of ID-mixup bugs.

---

## 4. Narrowing and Control Flow Analysis

TypeScript tracks types through control flow. The narrowing operations:

```ts
// 1. typeof
function f(x: string | number) {
  if (typeof x === 'string') x.toUpperCase();   // string
  else x.toFixed(2);                            // number
}

// 2. instanceof
if (err instanceof HttpError) err.statusCode;

// 3. in operator
type A = { a: string }; type B = { b: number };
function g(v: A | B) { if ('a' in v) v.a; else v.b; }

// 4. Truthiness
function h(s?: string) { if (s) s.length; }      // string

// 5. Equality
function i(x: string | number, y: string | boolean) {
  if (x === y) { /* both narrowed to string */ }
}

// 6. Discriminated union (best pattern)
type Result<T> =
  | { ok: true;  value: T }
  | { ok: false; error: Error };

function unwrap<T>(r: Result<T>): T {
  if (r.ok) return r.value;      // narrowed by literal discriminant
  throw r.error;
}

// 7. Type predicate (user-defined guard)
function isString(x: unknown): x is string {
  return typeof x === 'string';
}

// 8. Assertion function
function assertDefined<T>(v: T): asserts v is NonNullable<T> {
  if (v == null) throw new Error('Expected value');
}
declare const maybe: string | undefined;
assertDefined(maybe);
maybe.length;                    // ✅ narrowed for the rest of the scope

// 9. Discriminating on array length (TS 4.6+)
type Args = [string] | [string, number];
function j(args: Args) {
  if (args.length === 2) args[1].toFixed();   // narrowed to [string, number]
}
```

### 4.1 Narrowing Traps

```ts
// ⚠️ Narrowing is lost across closures/callbacks
function f(x?: string) {
  if (x) {
    setTimeout(() => x.length);   // ❌ x could be reassigned before it runs
  }
}
// Fix: capture in a const
function g(x?: string) {
  if (x) { const s = x; setTimeout(() => s.length); }  // ✅
}

// ⚠️ Narrowing lost after a function call on a mutable property
type Box = { v?: string };
function h(b: Box) {
  if (b.v) { mutate(); b.v.length; }  // ❌ mutate() could clear it
}
```

---

## 5. Generics

### 5.1 Constraints and Inference

```ts
// Constraint: T must have a length
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

// keyof constraint — type-safe property access
function pluck<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const name = pluck({ name: 'Ada', age: 36 }, 'name');   // string
pluck({ name: 'Ada' }, 'nope');                          // ❌ error

// Multiple type params with defaults
function make<T, U = T>(a: T, b: U): [T, U] { return [a, b]; }

// Inference from return position
function create<T>(): T { return {} as T; }
const x: string = create();     // T inferred as string
```

### 5.2 Inference Rules Worth Knowing

```ts
// Inference happens per-parameter, left to right, and picks the
// "best common supertype" for multiple candidates.
declare function pick<T>(a: T, b: T): T;
pick('a', 1);     // T = string | number

// `const` type parameter (TS 5.0) — preserves literals without `as const`
declare function route<const T extends readonly string[]>(paths: T): T;
const r = route(['a', 'b']);   // readonly ["a", "b"]

// NoInfer (TS 5.4) — block a position from contributing to inference
declare function withDefault<T>(values: T[], fallback: NoInfer<T>): T;
withDefault([1, 2, 3], 4);       // T = number
withDefault(['a'], 'x');         // T = string, "x" doesn't widen T
```

### 5.3 Generic Classes and Interfaces

```ts
interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  save(entity: Omit<T, 'id'>): Promise<T>;
  update(id: ID, patch: Partial<T>): Promise<T>;
  delete(id: ID): Promise<void>;
}

class InMemoryRepo<T extends { id: string }> implements Repository<T> {
  #items = new Map<string, T>();
  async findById(id: string) { return this.#items.get(id) ?? null; }
  async findAll(filter?: Partial<T>) {
    return [...this.#items.values()].filter((item) =>
      !filter || Object.entries(filter).every(([k, v]) => item[k as keyof T] === v));
  }
  async save(entity: Omit<T, 'id'>) {
    const item = { ...entity, id: crypto.randomUUID() } as unknown as T;
    this.#items.set(item.id, item);
    return item;
  }
  async update(id: string, patch: Partial<T>) {
    const cur = this.#items.get(id);
    if (!cur) throw new Error('Not found');
    const next = { ...cur, ...patch };
    this.#items.set(id, next);
    return next;
  }
  async delete(id: string) { this.#items.delete(id); }
}
```

---

## 6. Conditional Types

The type-level `if`:

```ts
type IsString<T> = T extends string ? true : false;
type A = IsString<'hi'>;    // true
type B = IsString<42>;      // false
```

### 6.1 Distribution

⚙️ **Critical rule:** a conditional type over a *naked* type parameter distributes over unions.

```ts
type ToArray<T> = T extends any ? T[] : never;
type R = ToArray<string | number>;
// distributes: ToArray<string> | ToArray<number>
// = string[] | number[]      NOT (string | number)[]

// Block distribution by wrapping in a tuple
type ToArrayNo<T> = [T] extends [any] ? T[] : never;
type R2 = ToArrayNo<string | number>;   // (string | number)[]
```

This is exactly how `Exclude` works:

```ts
type Exclude<T, U> = T extends U ? never : T;
type X = Exclude<'a' | 'b' | 'c', 'a'>;
//  = (a extends a ? never : a) | (b extends a ? never : b) | (c extends a ? never : c)
//  = never | 'b' | 'c'
//  = 'b' | 'c'          ← never disappears in unions
```

### 6.2 `infer`

`infer` declares a type variable to be filled by pattern matching.

```ts
type ElementType<T> = T extends (infer U)[] ? U : never;
type E = ElementType<string[]>;        // string

type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type Parameters<T> = T extends (...args: infer P) => any ? P : never;
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;   // recursive

// Multiple infers
type FirstArg<T> = T extends (a: infer A, ...rest: any[]) => any ? A : never;

// infer with constraint (TS 4.7)
type FirstNumeric<T> = T extends [infer U extends number, ...any[]] ? U : never;

// String pattern matching
type Split<S extends string, D extends string> =
  S extends `${infer Head}${D}${infer Tail}`
    ? [Head, ...Split<Tail, D>]
    : [S];
type Parts = Split<'a.b.c', '.'>;   // ["a", "b", "c"]
```

---

## 7. Mapped Types

The type-level `for` loop over keys.

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] };
type MyRequired<T> = { [K in keyof T]-?: T[K] };     // -? removes optionality
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };
type Mutable<T>    = { -readonly [K in keyof T]: T[K] };  // -readonly removes it
```

```
      Mapped type anatomy

      { readonly [K in keyof T as NewKey]?: NewValue }
         │        │   │     │        │      │
         │        │   │     │        │      └── value transform
         │        │   │     │        └── key remapping (TS 4.1+)
         │        │   │     └── source of keys
         │        │   └── the key variable
         │        └── iterate
         └── modifier: readonly / -readonly / ? / -?
```

### 7.1 Key Remapping

```ts
// Generate getters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type G = Getters<{ name: string; age: number }>;
// { getName: () => string; getAge: () => number }

// Filter keys by value type
type PickByType<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};
type Strings = PickByType<{ a: string; b: number; c: string }, string>;
// { a: string; c: string }        ← `never` key removes the entry

// Remove keys
type OmitByType<T, V> = {
  [K in keyof T as T[K] extends V ? never : K]: T[K];
};
```

### 7.2 Deep Variants

```ts
type DeepPartial<T> = T extends object
  ? T extends Function | Date | RegExp ? T
  : { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

---

## 8. Template Literal Types

String manipulation at the type level.

```ts
type Greeting = `hello ${string}`;
const g1: Greeting = 'hello world';   // ✅
const g2: Greeting = 'hi world';      // ❌

// Combinatorial expansion
type Size = 'sm' | 'md' | 'lg';
type Variant = 'primary' | 'danger';
type ClassName = `btn-${Variant}-${Size}`;
// "btn-primary-sm" | "btn-primary-md" | ... 6 combinations

// Intrinsics: Uppercase, Lowercase, Capitalize, Uncapitalize
type Upper = Uppercase<'abc'>;        // "ABC"

// Typed event names
type EventMap = { click: MouseEvent; keydown: KeyboardEvent };
type Handlers = { [K in keyof EventMap as `on${Capitalize<K>}`]: (e: EventMap[K]) => void };
// { onClick: (e: MouseEvent) => void; onKeydown: (e: KeyboardEvent) => void }

// Typed deep paths — very powerful
type Path<T> = T extends object
  ? { [K in keyof T & string]: T[K] extends object ? K | `${K}.${Path<T[K]>}` : K }[keyof T & string]
  : never;

type Config = { db: { host: string; port: number }; debug: boolean };
type P = Path<Config>;   // "db" | "db.host" | "db.port" | "debug"

type PathValue<T, P extends string> =
  P extends `${infer K}.${infer Rest}`
    ? K extends keyof T ? PathValue<T[K], Rest> : never
    : P extends keyof T ? T[P] : never;

declare function get<T, P extends Path<T>>(obj: T, path: P): PathValue<T, P>;
declare const cfg: Config;
const port = get(cfg, 'db.port');    // number ✅
get(cfg, 'db.nope');                 // ❌ compile error
```

---

## 9. Variance

How type relationships propagate through containers.

```
   If  Dog <: Animal   (Dog is a subtype of Animal)

   COVARIANT      F<Dog> <: F<Animal>       → readonly containers, return types
   CONTRAVARIANT  F<Animal> <: F<Dog>       → function parameters
   INVARIANT      no relationship           → mutable containers (in sound systems)
   BIVARIANT      both directions           → TS method params (unsound, historical)
```

```ts
class Animal { name = ''; }
class Dog extends Animal { bark() {} }

// Covariant: arrays of subtypes are assignable (TS allows, technically unsound)
const dogs: Dog[] = [new Dog()];
const animals: Animal[] = dogs;      // ✅ TS allows
animals.push(new Animal());          // 💥 dogs now contains a non-Dog

// Contravariant parameters — with strictFunctionTypes
type Handler<T> = (x: T) => void;
let animalHandler: Handler<Animal> = (a) => {};
let dogHandler: Handler<Dog> = (d) => d.bark();

dogHandler = animalHandler;   // ✅ safe: handles Animals, so handles Dogs
animalHandler = dogHandler;   // ❌ unsafe: would call .bark() on a plain Animal
```

⚠️ **Method syntax is bivariant, property syntax is contravariant:**

```ts
interface A { m(x: Dog): void; }        // bivariant (legacy, unsafe)
interface B { m: (x: Dog) => void; }    // contravariant (checked under strictFunctionTypes)
```

Prefer property syntax for callbacks you want checked strictly.

### 9.1 Explicit Variance Annotations (TS 4.7+)

```ts
interface Producer<out T> { get(): T; }        // covariant
interface Consumer<in T> { set(v: T): void; }  // contravariant
interface Box<in out T> { get(): T; set(v: T): void; }  // invariant
```

These both document intent and speed up the compiler on large generic types.

---

## 10. Declaration Merging and Modules

```ts
// Interfaces merge; type aliases do not
interface Window { myApp: App; }        // augments the global Window
interface Window { myOther: Other; }    // merges

// Module augmentation
declare module 'express' {
  interface Request { user?: { id: string; role: string }; }
}

// Global augmentation
declare global {
  interface Array<T> { last(): T | undefined; }
  var __DEV__: boolean;
}

// Namespace + function merging
function greet(name: string) { return `Hi ${name}`; }
namespace greet { export const version = '1.0'; }
greet('a'); greet.version;
```

| | `interface` | `type` |
|---|---|---|
| Declaration merging | ✅ | ❌ |
| Unions | ❌ | ✅ |
| Tuples / primitives | ❌ | ✅ |
| Mapped / conditional | ❌ | ✅ |
| `implements` | ✅ | ✅ (object types) |
| Error messages | Often nicer | Can be expanded/noisy |
| Performance on large objects | Slightly better (cached) | Recomputed |

🏭 **Rule:** `interface` for public object contracts you may extend; `type` for everything else.

---

## 11. Utility Types

### Built-in reference

| Utility | Definition | Example |
|---|---|---|
| `Partial<T>` | all optional | `Partial<{a:1}>` → `{a?:1}` |
| `Required<T>` | all required | removes `?` |
| `Readonly<T>` | all readonly | |
| `Record<K,V>` | object with keys K | `Record<'a'\|'b', number>` |
| `Pick<T,K>` | keep keys K | `Pick<User,'id'\|'name'>` |
| `Omit<T,K>` | drop keys K | `Omit<User,'password'>` |
| `Exclude<T,U>` | union minus | `Exclude<'a'\|'b','a'>` → `'b'` |
| `Extract<T,U>` | union intersect | `Extract<'a'\|'b','a'>` → `'a'` |
| `NonNullable<T>` | remove null/undefined | |
| `Parameters<F>` | param tuple | |
| `ReturnType<F>` | return type | |
| `ConstructorParameters<C>` | ctor params | |
| `InstanceType<C>` | instance type | |
| `Awaited<T>` | unwrap promises recursively | |
| `ThisParameterType<F>` | `this` param | |
| `NoInfer<T>` | block inference | TS 5.4 |

### Custom utilities worth having

```ts
// Make some keys optional, keep the rest
type PartialBy<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;

// Make some keys required
type RequiredBy<T, K extends keyof T> = Omit<T, K> & Required<Pick<T, K>>;

// Exactly one of a set of keys
type OneOf<T, K extends keyof T = keyof T> =
  K extends any ? Required<Pick<T, K>> & Partial<Record<Exclude<keyof T, K>, never>> : never;

type Auth = OneOf<{ token: string; apiKey: string; cookie: string }>;
const a1: Auth = { token: 'x' };                  // ✅
const a2: Auth = { token: 'x', apiKey: 'y' };     // ❌

// Union to intersection (classic contravariance trick)
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void ? I : never;

// Prettify — force the editor to expand a computed type
type Prettify<T> = { [K in keyof T]: T[K] } & {};

// Nullable everything
type Nullable<T> = { [K in keyof T]: T[K] | null };

// Entries type
type Entries<T> = { [K in keyof T]: [K, T[K]] }[keyof T][];

// Tuple length arithmetic
type Length<T extends readonly any[]> = T['length'];
type BuildTuple<N extends number, R extends any[] = []> =
  R['length'] extends N ? R : BuildTuple<N, [...R, any]>;
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]['length'];
type Seven = Add<3, 4>;   // 7
```

---

## 12. Type-Level Programming

The type system is Turing-complete. Use responsibly.

```ts
// Recursive JSON type
type Json = string | number | boolean | null | Json[] | { [k: string]: Json };

// Type-safe query builder
type Table = { users: { id: number; name: string; email: string } };
type Columns<T extends keyof Table> = keyof Table[T];

declare function select<T extends keyof Table, C extends Columns<T>>(
  table: T, columns: C[],
): Pick<Table[T], C>[];

const rows = select('users', ['id', 'name']);   // { id: number; name: string }[]
select('users', ['nope']);                       // ❌

// Typed state machine — illegal transitions are compile errors
type Transitions = {
  idle:    'loading';
  loading: 'success' | 'error';
  success: 'idle';
  error:   'loading' | 'idle';
};
type State = keyof Transitions;

declare function transition<S extends State>(from: S, to: Transitions[S]): Transitions[S];
transition('idle', 'loading');    // ✅
transition('idle', 'success');    // ❌
```

⚠️ **Cost warning:** deep recursive types blow up compile times and hit the recursion limit (~50 levels, ~1M instantiations). If your IDE gets slow, look for a recursive conditional type first. `tsc --generateTrace` will find it.

---

## 13. Practical Patterns

### 13.1 Runtime Validation at Boundaries

Types vanish at runtime. Anything crossing a boundary must be validated.

```ts
// With Zod — one schema, both the runtime check and the type
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  role: z.enum(['admin', 'user', 'guest']),
  createdAt: z.coerce.date(),
});
type User = z.infer<typeof UserSchema>;   // derived, never drifts

async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return UserSchema.parse(await res.json());   // throws on mismatch
}

// Hand-rolled guard when you can't add a dependency
function isUser(v: unknown): v is User {
  return typeof v === 'object' && v !== null &&
    typeof (v as any).id === 'string' &&
    typeof (v as any).email === 'string';
}
```

### 13.2 The Result Type

```ts
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

const Ok  = <T>(value: T): Result<T, never> => ({ ok: true, value });
const Err = <E>(error: E): Result<never, E> => ({ ok: false, error });

async function safeFetch<T>(url: string): Promise<Result<T, string>> {
  try {
    const res = await fetch(url);
    if (!res.ok) return Err(`HTTP ${res.status}`);
    return Ok(await res.json() as T);
  } catch (e) {
    return Err(e instanceof Error ? e.message : 'Unknown');
  }
}

const r = await safeFetch<User>('/api/me');
if (r.ok) console.log(r.value.email);   // narrowed
else console.error(r.error);
```

Advantage over exceptions: the error path is in the type signature, so the compiler forces you to handle it.

### 13.3 Builder with Type-Level State

```ts
type Built<T> = { [K in keyof T]-?: T[K] };

class QueryBuilder<T = {}> {
  private parts: Record<string, unknown> = {};

  from(table: string): QueryBuilder<T & { from: string }> {
    this.parts.from = table;
    return this as any;
  }
  where(cond: string): QueryBuilder<T & { where: string }> {
    this.parts.where = cond;
    return this as any;
  }
  // build only exists when `from` was called
  build(this: QueryBuilder<{ from: string }>): string {
    return `SELECT * FROM ${this.parts.from}` +
      (this.parts.where ? ` WHERE ${this.parts.where}` : '');
  }
}

new QueryBuilder().from('users').where('id=1').build();   // ✅
new QueryBuilder().where('id=1').build();                  // ❌ no `from`
```

### 13.4 Typing React Props

```tsx
// Polymorphic component — `as` prop with correct element props
type PolymorphicProps<E extends React.ElementType, P> = P & {
  as?: E;
} & Omit<React.ComponentPropsWithoutRef<E>, keyof P | 'as'>;

type ButtonOwnProps = { variant?: 'primary' | 'ghost'; size?: 'sm' | 'lg' };

function Button<E extends React.ElementType = 'button'>(
  { as, variant = 'primary', size = 'sm', ...rest }: PolymorphicProps<E, ButtonOwnProps>,
) {
  const Tag = as ?? 'button';
  return <Tag className={`btn-${variant}-${size}`} {...rest} />;
}

<Button onClick={() => {}} />;                 // ✅ button props
<Button as="a" href="/x" />;                   // ✅ anchor props
<Button as="a" onClick={() => {}} nope={1} />; // ❌ unknown prop
```

---

## 14. Configuration & Compiler

### 14.1 The tsconfig That Matters

```jsonc
{
  "compilerOptions": {
    // ── Strictness (turn ALL of these on) ────────────────
    "strict": true,                        // enables the block below
    "noUncheckedIndexedAccess": true,      // arr[i] is T | undefined  ⭐
    "exactOptionalPropertyTypes": true,    // ?: means absent, not undefined
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitReturns": true,
    "useUnknownInCatchVariables": true,    // catch (e: unknown)

    // ── Modules ─────────────────────────────────────────
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "target": "es2022",
    "lib": ["es2023", "dom", "dom.iterable"],
    "verbatimModuleSyntax": true,          // explicit `import type`
    "isolatedModules": true,               // safe for esbuild/swc

    // ── Output ──────────────────────────────────────────
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true,
    "skipLibCheck": true,                  // big build-time win

    // ── Interop ─────────────────────────────────────────
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  }
}
```

⭐ `noUncheckedIndexedAccess` is the highest-value non-default flag. It turns `arr[0]` into `T | undefined`, which is *the truth*, and catches a large class of real bugs.

### 14.2 What `strict` Turns On

| Flag | Catches |
|---|---|
| `noImplicitAny` | Untyped parameters |
| `strictNullChecks` | `null`/`undefined` misuse — the biggest one |
| `strictFunctionTypes` | Unsafe function assignability |
| `strictBindCallApply` | Wrong args to bind/call/apply |
| `strictPropertyInitialization` | Class fields never assigned |
| `noImplicitThis` | `this` of type `any` |
| `alwaysStrict` | Emits `'use strict'` |
| `useUnknownInCatchVariables` | `catch (e)` typed `unknown` |

### 14.3 Performance

```bash
tsc --noEmit --extendedDiagnostics     # where is time going
tsc --generateTrace ./trace            # then open in Perfetto UI
```

| Symptom | Usual cause |
|---|---|
| Slow IDE hovers | Deep conditional/recursive type |
| Slow whole-project build | Missing `skipLibCheck`, huge union types |
| Memory blowup | Recursive template literal explosion |

Fix: replace giant unions with branded strings, add `Prettify` sparingly, split into project references.

---

## 15. Interview Section

<details>
<summary><b>Q1. `interface` vs `type` — when do you use each?</b></summary>

They overlap heavily. `interface` can declaration-merge and is slightly cheaper for the compiler to cache; `type` can express unions, tuples, primitives, mapped and conditional types.

My rule: `interface` for object contracts that consumers might extend or that get published in a library's public API. `type` for everything else — unions, function types, and any computed type.
</details>

<details>
<summary><b>Q2. `any` vs `unknown` vs `never`.</b></summary>

`any` disables type checking on that value — it's an escape hatch and it spreads. `unknown` is the safe top type: you can assign anything to it but must narrow before use. `never` is the bottom type — no value inhabits it; it means "this can't happen."

Practically: `unknown` at every boundary (API responses, `catch`, `JSON.parse`), `never` for exhaustiveness checks, and `any` only with a comment explaining why.
</details>

<details>
<summary><b>Q3. Explain conditional type distribution.</b></summary>

When a conditional type's checked type is a naked type parameter and you pass a union, TypeScript distributes the conditional over each union member and recombines the results.

`T extends U ? X : Y` with `T = A | B` becomes `(A extends U ? X : Y) | (B extends U ? X : Y)`.

That's how `Exclude` works: members that match become `never`, and `never` disappears from unions. To prevent distribution, wrap both sides in tuples: `[T] extends [U] ? X : Y`.
</details>

<details>
<summary><b>Q4. What is a discriminated union and why is it the best pattern in TS?</b></summary>

A union of object types that all share a common literal-typed field. Checking that field narrows the whole object.

It's the best pattern because it makes illegal states unrepresentable and gives you exhaustiveness checking for free via `never`. Instead of `{ loading: boolean; data?: T; error?: E }` — which permits `loading: true` with `data` set — you write three variants where each state carries exactly the data it has.
</details>

<details>
<summary><b>Q5. Structural vs nominal typing — and how do you get nominal in TS?</b></summary>

TypeScript compares shapes, so any object with the right properties is assignable. That's convenient but lets you pass a `UserId` string where an `OrderId` is expected.

For nominal behavior, use branding: intersect the primitive with a phantom property keyed by a `unique symbol`. It's erased at runtime and costs nothing, but the compiler now treats the two as distinct.
</details>

<details>
<summary><b>Q6. What does `keyof` do with an index signature?</b></summary>

`keyof { [k: string]: T }` is `string | number` — because JS coerces numeric keys to strings. `keyof { [k: number]: T }` is `number`. For a normal object type it's the union of literal key names.

Related gotcha: `Object.keys(o)` returns `string[]`, not `(keyof T)[]`, because the object might have extra properties at runtime (structural typing). Casting is common but technically unsound.
</details>

<details>
<summary><b>Q7. Why is `strictFunctionTypes` needed and where doesn't it apply?</b></summary>

Function parameters should be contravariant: a handler accepting `Animal` is safely usable where an `Animal`-or-narrower handler is expected, but not the reverse. Without the flag, TS checks parameters bivariantly and lets you assign a `Handler<Dog>` where `Handler<Animal>` is expected, which crashes when a non-Dog arrives.

It does not apply to method-syntax declarations (`m(x: T): void`) — those stay bivariant for backward compatibility with things like `Array.prototype.sort`. Use property syntax (`m: (x: T) => void`) if you want the strict check.
</details>

<details>
<summary><b>Q8. How do you type an API response safely?</b></summary>

Never type-assert it. `as User` is a lie the compiler accepts. Instead treat the response as `unknown` and run a validator — Zod, Valibot, or a hand-written type predicate — and derive the TypeScript type from the schema so they can't drift apart.

The principle: types are compile-time only, so the only real safety at a runtime boundary is a runtime check.
</details>

<details>
<summary><b>Q9. When is `as` acceptable?</b></summary>

Three cases: narrowing to a more specific type you've genuinely proven (after a validator), `as const` for literal preservation, and interop with untyped libraries at a single controlled boundary.

Every other use is a suppressed error. `as unknown as X` is a double assertion, which is the compiler telling you the two types have nothing in common.
</details>

<details>
<summary><b>Q10. What is `satisfies` and how does it differ from `as` and from an annotation?</b></summary>

```ts
const a: Record<string, string> = { x: 'a' };     // a.x is string, keys widened
const b = { x: 'a' } as Record<string, string>;   // same, plus no checking
const c = { x: 'a' } satisfies Record<string, string>;  // checked AND narrow
```

`satisfies` validates that the expression conforms to a type *without* widening the inferred type. So `c.x` is the literal `"a"` and `Object.keys(c)` knows the exact keys, while still erroring if you write a non-string value. It's the right tool for config objects.
</details>

### 🎤 Type challenges

Solve these without looking:

```ts
// 1. Implement Pick
type MyPick<T, K extends keyof T> = ?

// 2. Implement Exclude
type MyExclude<T, U> = ?

// 3. Get the last element type of a tuple
type Last<T extends any[]> = ?

// 4. Concat two tuples
type Concat<A extends any[], B extends any[]> = ?

// 5. Deep readonly
type DeepReadonly<T> = ?

// 6. Turn a union into a tuple (hard)
type UnionToTuple<T> = ?
```

<details>
<summary>Solutions</summary>

```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };

type MyExclude<T, U> = T extends U ? never : T;

type Last<T extends any[]> = T extends [...any[], infer L] ? L : never;

type Concat<A extends any[], B extends any[]> = [...A, ...B];

type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;

// UnionToTuple relies on UnionToIntersection to pop one member at a time
type UnionToIntersection<U> =
  (U extends any ? (x: U) => void : never) extends (x: infer I) => void ? I : never;
type LastOfUnion<U> =
  UnionToIntersection<U extends any ? (x: U) => void : never> extends (x: infer L) => void ? L : never;
type UnionToTuple<T, R extends any[] = []> =
  [T] extends [never] ? R : UnionToTuple<Exclude<T, LastOfUnion<T>>, [LastOfUnion<T>, ...R]>;
```
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                      TYPESCRIPT — ONE PAGE                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ TYPES ARE SETS                                                       ║
║   A extends B  ≈  A ⊆ B      never = ∅      unknown = everything     ║
║   union = ∪    intersection = ∩    any = opt out                     ║
╠══════════════════════════════════════════════════════════════════════╣
║ NARROWING: typeof · instanceof · in · truthiness · === literal       ║
║            x is T (predicate) · asserts x is T (assertion)           ║
║   ⚠️ narrowing is lost inside callbacks — capture to a const          ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONDITIONAL:  T extends U ? X : Y      distributes over naked unions ║
║   block with [T] extends [U]                                         ║
║   infer to pattern-match: T extends (infer U)[] ? U : never          ║
╠══════════════════════════════════════════════════════════════════════╣
║ MAPPED:  { readonly [K in keyof T as NewK]?: NewV }                  ║
║   -readonly / -? remove modifiers                                    ║
║   key → never removes the entry                                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ VARIANCE: params contravariant, returns covariant                    ║
║   method syntax = bivariant (legacy) | property syntax = strict      ║
╠══════════════════════════════════════════════════════════════════════╣
║ ALWAYS:  strict: true + noUncheckedIndexedAccess                     ║
║          unknown at boundaries + runtime validation (Zod)            ║
║          discriminated unions + never for exhaustiveness             ║
║          satisfies for config objects                                ║
║ NEVER:   `as` to silence errors • any without a comment              ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [JavaScript](javascript.md) · [React](../02-frontend/react.md) · [API Design](../03-backend/api-design.md)
