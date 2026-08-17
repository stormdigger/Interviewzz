# 🐍 Python Deep Dive

> Python's design is unusually coherent once you see the data model. Nearly every "magic" feature — operators, iteration, context managers, attribute access — is one protocol expressed through dunder methods.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Object Model](#2-the-object-model)
3. [The Data Model — Dunder Protocols](#3-the-data-model)
4. [Names, Scope, Closures](#4-names-scope-closures)
5. [Mutability and Copying](#5-mutability-and-copying)
6. [Sequences, Dicts, Sets — Internals](#6-collections-internals)
7. [Iterators and Generators](#7-iterators-and-generators)
8. [Decorators](#8-decorators)
9. [Classes, MRO, Metaclasses](#9-classes-mro-metaclasses)
10. [The GIL and Concurrency](#10-the-gil-and-concurrency)
11. [asyncio](#11-asyncio)
12. [Memory Management](#12-memory-management)
13. [Type Hints](#13-type-hints)
14. [Error Handling](#14-error-handling)
15. [Performance](#15-performance)
16. [Modern Python (3.10→3.13)](#16-modern-python)
17. [Interview Section](#17-interview-section)
18. [Cheat Sheet](#18-cheat-sheet)

---

## 1. Mental Model

🧠 **Everything is an object with an identity, a type, and a value.** Variables are not boxes holding values — they are *names bound to objects*. Assignment never copies; it rebinds a name.

```
   x = [1, 2, 3]
   y = x
   
   NAMESPACE (dict)          HEAP
   ┌──────────────┐         ┌──────────────────────┐
   │ 'x' ─────────┼────────▶│ list object          │
   │ 'y' ─────────┼────────▶│  refcount: 2         │
   └──────────────┘         │  type: list          │
                            │  items: [1, 2, 3]    │
                            └──────────────────────┘

   y.append(4)   →  BOTH x and y see [1,2,3,4]  (same object)
   y = [9]       →  rebinds 'y' only; x unchanged
```

This single picture explains mutable default arguments, aliasing bugs, `is` vs `==`, and how function arguments are passed.

### Python's arg passing: "call by object reference"

```python
def f(lst, num):
    lst.append(4)      # mutates the caller's object    → visible outside
    num += 1           # rebinds a local name           → NOT visible
    lst = [99]         # rebinds a local name           → NOT visible

a, b = [1,2,3], 10
f(a, b)
print(a, b)            # [1, 2, 3, 4] 10
```

---

## 2. The Object Model

Every object has three attributes:

| Attribute | Accessor | Mutable? |
|---|---|---|
| **identity** | `id(x)` — memory address in CPython | never |
| **type** | `type(x)` | almost never |
| **value** | the contents | depends on the type |

```python
a = 256; b = 256
a is b            # True  — CPython caches small ints (-5..256)

a = 257; b = 257
a is b            # False in a REPL; often True in a script (compiler folds constants)

s1 = "hello"; s2 = "hello"
s1 is s2          # True — string interning for identifier-like literals
```

⚠️ **Never use `is` for value comparison.** Only for `None`, `True`, `False`, and sentinel objects. The small-int cache and interning are implementation details that change between versions and contexts.

### The truthiness protocol

```
   bool(x)
      │
      ├── __bool__ defined?  ──▶ use it
      ├── __len__ defined?   ──▶ len(x) != 0
      └── neither            ──▶ True
```

Falsy: `False`, `None`, `0`, `0.0`, `0j`, `Decimal(0)`, `""`, `()`, `[]`, `{}`, `set()`, `range(0)`, and any object whose `__len__` returns 0.

---

## 3. The Data Model

⚙️ **This is the heart of Python.** Syntax is sugar over dunder methods.

| You write | Python calls |
|---|---|
| `a + b` | `a.__add__(b)`, then `b.__radd__(a)` |
| `a[k]` | `a.__getitem__(k)` |
| `a[k] = v` | `a.__setitem__(k, v)` |
| `len(a)` | `a.__len__()` |
| `x in a` | `a.__contains__(x)` → falls back to iteration |
| `for x in a` | `iter(a)` → `a.__iter__()` → `__next__()` |
| `a()` | `a.__call__()` |
| `with a:` | `a.__enter__()` / `a.__exit__()` |
| `a.b` | `a.__getattribute__('b')` → `__getattr__` on failure |
| `str(a)` / `f"{a}"` | `a.__str__()` → falls back to `__repr__` |
| `repr(a)` | `a.__repr__()` |
| `a == b` | `a.__eq__(b)` |
| `hash(a)` | `a.__hash__()` |
| `a < b` | `a.__lt__(b)` |
| `-a` | `a.__neg__()` |
| `del a[k]` | `a.__delitem__(k)` |
| `await a` | `a.__await__()` |

### 3.1 Building a full-protocol class

```python
from __future__ import annotations
from dataclasses import dataclass
from functools import total_ordering
import math

@total_ordering                  # fills in <=, >, >= from __eq__ and __lt__
@dataclass(frozen=True, slots=True)
class Vector:
    x: float
    y: float

    # ── Arithmetic ────────────────────────────────
    def __add__(self, other: Vector) -> Vector:
        if not isinstance(other, Vector):
            return NotImplemented        # lets Python try other.__radd__
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, k: float) -> Vector:
        return Vector(self.x * k, self.y * k)

    __rmul__ = __mul__                   # 3 * v works too

    def __neg__(self) -> Vector:
        return Vector(-self.x, -self.y)

    # ── Comparison ────────────────────────────────
    def __lt__(self, other: Vector) -> bool:
        return abs(self) < abs(other)

    # ── Conversion ────────────────────────────────
    def __abs__(self) -> float:
        return math.hypot(self.x, self.y)

    def __bool__(self) -> bool:
        return bool(abs(self))

    def __repr__(self) -> str:           # unambiguous, for developers
        return f"Vector({self.x!r}, {self.y!r})"

    def __str__(self) -> str:            # readable, for users
        return f"({self.x}, {self.y})"

    def __format__(self, spec: str) -> str:
        if spec.endswith('p'):           # polar format
            return f"<{abs(self):{spec[:-1]}}, {math.atan2(self.y, self.x):{spec[:-1]}}>"
        return f"({self.x:{spec}}, {self.y:{spec}})"

    # ── Container ─────────────────────────────────
    def __iter__(self):
        yield self.x; yield self.y       # enables tuple unpacking: a, b = v

    def __len__(self) -> int:
        return 2

    def __getitem__(self, i: int) -> float:
        return (self.x, self.y)[i]

v = Vector(3, 4)
abs(v)              # 5.0
v + Vector(1, 1)    # Vector(4, 5)
3 * v               # Vector(9, 12)
a, b = v            # unpacking via __iter__
f"{v:.2f}"          # "(3.00, 4.00)"
```

### 3.2 `__repr__` vs `__str__`

```
   print(x)  /  str(x)  /  f"{x}"     →  __str__, falls back to __repr__
   repr(x)   /  REPL echo  /  f"{x!r}" →  __repr__, falls back to <Class object at 0x...>
   Inside a container: [x]             →  ALWAYS __repr__
```

🏭 Rule: always define `__repr__` — it's what you see in logs and debuggers. Define `__str__` only when the user-facing form differs.

### 3.3 The `__eq__` / `__hash__` contract

```python
class Point:
    def __init__(self, x, y): self.x, self.y = x, y
    def __eq__(self, other):
        return isinstance(other, Point) and (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))     # MUST match __eq__'s fields
```

**Rules:**
1. `a == b` ⟹ `hash(a) == hash(b)` (the reverse need not hold — collisions are fine).
2. Defining `__eq__` sets `__hash__ = None`, making instances unhashable, unless you define `__hash__` too.
3. Hash on **immutable** fields only. A mutable key silently gets lost in its dict.

```python
class Bad:
    def __init__(self, v): self.v = v
    def __eq__(self, o): return self.v == o.v
    def __hash__(self): return hash(self.v)

k = Bad(1); d = {k: 'x'}
k.v = 2                # hash changed while in the dict
d[k]                   # KeyError — looks in the wrong bucket
```

---

## 4. Names, Scope, Closures

### 4.1 LEGB

```
   ┌───────────────────────────────────────────┐
   │ B — Builtins   (print, len, ...)          │
   │  ┌──────────────────────────────────────┐ │
   │  │ G — Global (module level)            │ │
   │  │  ┌─────────────────────────────────┐ │ │
   │  │  │ E — Enclosing (outer functions) │ │ │
   │  │  │  ┌────────────────────────────┐ │ │ │
   │  │  │  │ L — Local (current func)   │ │ │ │
   │  │  │  └────────────────────────────┘ │ │ │
   │  │  └─────────────────────────────────┘ │ │
   │  └──────────────────────────────────────┘ │
   └───────────────────────────────────────────┘

   Lookup goes L → E → G → B, then NameError.
```

⚠️ **Assignment anywhere in a function makes the name local for the entire function** — determined at compile time.

```python
x = 10
def f():
    print(x)      # UnboundLocalError!
    x = 20        # this line makes x local throughout f
```

```python
counter = 0
def inc():
    global counter        # rebind module-level name
    counter += 1

def outer():
    n = 0
    def inner():
        nonlocal n        # rebind the enclosing function's name
        n += 1
    inner(); return n     # 1
```

### 4.2 The Late-Binding Closure Trap

```python
# ❌ All three print 2
fns = [lambda: i for i in range(3)]
[f() for f in fns]        # [2, 2, 2]

# ✅ Default arg captures the value at definition time
fns = [lambda i=i: i for i in range(3)]
[f() for f in fns]        # [0, 1, 2]

# ✅ Or use a factory
def make(i): return lambda: i
fns = [make(i) for i in range(3)]
```

Closures capture *variables*, not values. In the broken version, all lambdas share the same cell for `i`.

### 4.3 The Mutable Default Argument

```python
# ❌ The default list is created ONCE, at function definition
def add(item, target=[]):
    target.append(item)
    return target

add(1)      # [1]
add(2)      # [1, 2]   ← surprise!

# ✅
def add(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

This is the single most-asked Python gotcha. The reason: default values are evaluated once when `def` executes and stored in `func.__defaults__`.

---

## 5. Mutability and Copying

| Immutable | Mutable |
|---|---|
| `int`, `float`, `complex`, `bool` | `list` |
| `str`, `bytes` | `dict` |
| `tuple`* | `set` |
| `frozenset` | `bytearray` |
| `range` | most user classes |

\* A tuple's *bindings* are immutable, but a mutable element inside it can change:

```python
t = ([1], 2)
t[0].append(3)      # ✅ works — [1,3]
t[0] += [4]         # 💥 TypeError raised AND the list IS mutated
                    # because += calls __iadd__ (mutates) then tries to
                    # rebind t[0] (fails). Both happen.
```

### Copying

```python
import copy
original = {'a': [1, 2], 'b': {'c': 3}}

alias    = original                    # same object
shallow  = copy.copy(original)         # or dict(original), original.copy()
deep     = copy.deepcopy(original)     # handles cycles via a memo dict

original['a'].append(9)
alias['a']    # [1,2,9]
shallow['a']  # [1,2,9]  ← nested object is shared
deep['a']     # [1,2]    ← fully independent
```

```
   original ──┬──▶ {'a': [1,2], 'b': {...}}
   alias ─────┘         │        │
                        │        │
   shallow ──▶ {new dict}┘────────┘   (top level new, contents shared)

   deep ─────▶ {new dict}
                 │      │
                 ▼      ▼
              [new]  {new}                (everything new)
```

Control deep-copy behavior with `__deepcopy__`/`__copy__` on your classes.

---

## 6. Collections Internals

### 6.1 list

A dynamic array of pointers with over-allocation.

```
   list object
   ┌──────────────────┐
   │ ob_size:  4      │  logical length
   │ allocated: 8     │  capacity
   │ ob_item ─────────┼──▶ [ptr][ptr][ptr][ptr][ _ ][ _ ][ _ ][ _ ]
   └──────────────────┘

   Growth pattern: 0, 4, 8, 16, 25, 35, 46, 58, 72, 88, ...
   (roughly newsize + newsize/8 + 6)  → amortized O(1) append
```

| Operation | Complexity | Note |
|---|---|---|
| `append`, `pop()` | O(1) amortized | |
| `insert(0,x)`, `pop(0)` | **O(n)** | shifts everything — use `deque` |
| `x in lst` | O(n) | use `set` for membership |
| `lst[i]` | O(1) | |
| `sort()` | O(n log n) | Timsort, stable, adaptive to runs |
| `len` | O(1) | stored |

### 6.2 dict

Since 3.6, dicts are **compact and insertion-ordered** — a sparse index array plus a dense entry array.

```
   Compact dict layout

   indices (sparse, small ints)      entries (dense, in insertion order)
   ┌────┬────┬────┬────┬────┬────┐   ┌──────────────────────────────┐
   │ -1 │  0 │ -1 │  2 │  1 │ -1 │   │ 0: hash, key 'a', value 1    │
   └────┴────┴────┴────┴────┴────┘   │ 1: hash, key 'b', value 2    │
        │         │    │             │ 2: hash, key 'c', value 3    │
        └─────────┴────┴────────────▶└──────────────────────────────┘

   Benefits: ~25% less memory, and iteration order = insertion order,
   because it just walks the dense array.
```

Lookup: hash the key → probe the index array (open addressing with a perturbation sequence) → follow to the entry → compare hashes, then `==`.

| Operation | Average | Worst |
|---|---|---|
| `d[k]`, `k in d`, `d[k]=v`, `del` | O(1) | O(n) with adversarial collisions |

⚠️ Mutating a dict while iterating raises `RuntimeError`. Iterate a copy: `for k in list(d)`.

### 6.3 set

Same hash table machinery, values omitted.

```python
a = {1,2,3}; b = {2,3,4}
a | b     # union         {1,2,3,4}
a & b     # intersection  {2,3}
a - b     # difference    {1}
a ^ b     # symmetric     {1,4}
a <= b    # subset
```

O(1) membership is why "does this exist?" checks should almost always use a set, not a list.

### 6.4 The `collections` module

```python
from collections import defaultdict, Counter, deque, namedtuple, OrderedDict, ChainMap

# defaultdict — no KeyError, factory supplies missing values
graph = defaultdict(list)
graph['a'].append('b')                  # no setdefault needed

# Counter — multiset with counting helpers
c = Counter("mississippi")
c.most_common(2)                        # [('i', 4), ('s', 4)]
c + Counter("pi")                       # combines counts

# deque — O(1) at BOTH ends, optional max length
d = deque([1,2,3], maxlen=3)
d.appendleft(0)                         # drops 3 from the right
d.rotate(1)

# namedtuple — lightweight immutable record
Point = namedtuple('Point', 'x y')
p = Point(1, 2); p.x; p._replace(x=5)

# ChainMap — layered lookup without merging
config = ChainMap(cli_args, env_vars, defaults)
```

`heapq` and `bisect` round out the toolkit:

```python
import heapq, bisect
h = [5,1,3]; heapq.heapify(h)           # min-heap in place, O(n)
heapq.heappush(h, 0); heapq.heappop(h)  # O(log n)
heapq.nlargest(3, data, key=len)
# max-heap: push negated values

sorted_list = [1,3,5,7]
bisect.insort(sorted_list, 4)           # keeps it sorted, O(n) insert
i = bisect.bisect_left(sorted_list, 5)  # O(log n) search
```

---

## 7. Iterators and Generators

### 7.1 The protocol

```
   for x in obj:
        │
        ▼
   iter(obj) ──▶ obj.__iter__()  (or __getitem__ fallback)
        │
        ▼
   iterator object with __next__()
        │
        ├── returns value ──▶ loop body
        └── raises StopIteration ──▶ loop ends
```

```python
class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self):
        cur = self.n
        while cur > 0:
            yield cur
            cur -= 1
```

⚠️ **Iterators are consumed.** A generator can only be walked once.

```python
g = (x*x for x in range(3))
list(g)     # [0, 1, 4]
list(g)     # []  ← exhausted
```

### 7.2 Generators

```python
def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Memory: O(1) regardless of how many you take
from itertools import islice
list(islice(fib(), 10))       # [0,1,1,2,3,5,8,13,21,34]
```

**Generators as coroutines** (`send`, `throw`, `close`):

```python
def averager():
    total = count = 0
    average = None
    while True:
        value = yield average       # receives via .send()
        total += value; count += 1
        average = total / count

avg = averager()
next(avg)          # prime it — runs to the first yield
avg.send(10)       # 10.0
avg.send(20)       # 15.0
avg.close()
```

**Pipelines** — composable, lazy, constant memory:

```python
def read_lines(path):
    with open(path) as f:
        yield from f

def parse(lines):
    for line in lines:
        yield line.strip().split(',')

def filter_errors(rows):
    for row in rows:
        if row[2] == 'ERROR':
            yield row

# Processes a 10 GB file in a few MB of RAM
for row in filter_errors(parse(read_lines('huge.log'))):
    handle(row)
```

### 7.3 itertools

```python
from itertools import (chain, cycle, repeat, count, islice, tee,
                       groupby, product, permutations, combinations,
                       accumulate, zip_longest, pairwise, batched)

chain([1,2],[3,4])                   # 1 2 3 4
islice(count(10, 2), 3)              # 10 12 14
accumulate([1,2,3,4])                # 1 3 6 10
list(pairwise([1,2,3]))              # [(1,2),(2,3)]        3.10+
list(batched(range(7), 3))           # [(0,1,2),(3,4,5),(6,)]  3.12+
product('ab', repeat=2)              # aa ab ba bb
combinations([1,2,3], 2)             # (1,2) (1,3) (2,3)

# groupby needs SORTED input — it groups CONSECUTIVE items
data = sorted(people, key=lambda p: p.dept)
for dept, group in groupby(data, key=lambda p: p.dept):
    print(dept, list(group))
```

---

## 8. Decorators

A decorator is a function that takes a function and returns a replacement.

```python
@log
def f(): ...
# is exactly
def f(): ...
f = log(f)
```

### 8.1 The full template

```python
import functools, time, logging

def retry(attempts=3, delay=1.0, backoff=2.0, exceptions=(Exception,)):
    """Decorator factory — takes config, returns a decorator."""
    def decorator(func):
        @functools.wraps(func)              # preserves __name__, __doc__, __wrapped__
        def wrapper(*args, **kwargs):
            wait = delay
            for attempt in range(1, attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == attempts:
                        raise
                    logging.warning("Attempt %d/%d failed: %s", attempt, attempts, e)
                    time.sleep(wait)
                    wait *= backoff
        return wrapper
    return decorator

@retry(attempts=5, exceptions=(ConnectionError,))
def fetch(url): ...
```

⚠️ **Always use `functools.wraps`.** Without it the wrapper's `__name__` becomes `"wrapper"`, docstrings vanish, and introspection/Sphinx/pytest fixtures break.

### 8.2 Useful stdlib decorators

```python
from functools import cache, lru_cache, cached_property, singledispatch, partial

@cache                              # unbounded memo, 3.9+
def fib(n): return n if n < 2 else fib(n-1) + fib(n-2)

@lru_cache(maxsize=128)             # bounded LRU
def expensive(x): ...
expensive.cache_info()              # hits, misses, size
expensive.cache_clear()

class Circle:
    def __init__(self, r): self.r = r
    @cached_property                # computed once, stored in the instance __dict__
    def area(self): return 3.14159 * self.r ** 2
    @property                       # computed each access
    def diameter(self): return 2 * self.r

@singledispatch                     # type-based dispatch
def render(obj): return str(obj)
@render.register
def _(obj: list): return ", ".join(map(render, obj))
@render.register
def _(obj: dict): return "{...}"
```

⚠️ `@cache` on a method keeps `self` alive forever — a real memory leak in long-running services. Cache module-level functions instead, or use `cached_property`.

### 8.3 Class decorators and stacking

```python
@dataclass(frozen=True, slots=True, kw_only=True)
class Config:
    host: str
    port: int = 8080

# Stacking applies bottom-up
@a
@b
@c
def f(): ...
# f = a(b(c(f)))
```

---

## 9. Classes, MRO, Metaclasses

### 9.1 Attribute Lookup Order

```
   obj.attr
      │
      ▼
   type(obj).__getattribute__(obj, 'attr')
      │
      ├─1─▶ data descriptor on type(obj) or its MRO?  (has __set__/__delete__)
      │        yes ──▶ call its __get__  ⭐ HIGHEST PRIORITY
      │
      ├─2─▶ obj.__dict__['attr']?
      │        yes ──▶ return it
      │
      ├─3─▶ non-data descriptor or plain class attr on the MRO?
      │        yes ──▶ __get__ if descriptor, else return value
      │
      └─4─▶ type(obj).__getattr__('attr')  (last-resort hook)
               else AttributeError
```

This order explains why `@property` (a data descriptor) always wins over an instance attribute of the same name, while a plain method (non-data descriptor) can be shadowed by one.

### 9.2 Descriptors

```python
class Validated:
    """Reusable typed + validated attribute."""
    def __set_name__(self, owner, name):     # 3.6+: learns its own name
        self.private = f"_{name}"
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None: return self          # accessed on the class
        return getattr(obj, self.private)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f"{self.name} must be numeric")
        if value < 0:
            raise ValueError(f"{self.name} must be non-negative")
        setattr(obj, self.private, value)

class Product:
    price = Validated()
    weight = Validated()
    def __init__(self, price, weight):
        self.price, self.weight = price, weight

Product(-5, 1)     # ValueError
```

`property`, `classmethod`, `staticmethod`, and `functools.cached_property` are all descriptors. Understanding descriptors means understanding all of them at once.

### 9.3 MRO and C3 Linearization

```
        A
       / \
      B   C
       \ /
        D          class D(B, C)
```

```python
class A:
    def m(self): print("A")
class B(A):
    def m(self): print("B"); super().m()
class C(A):
    def m(self): print("C"); super().m()
class D(B, C):
    def m(self): print("D"); super().m()

D().m()            # D B C A     ← NOT D B A C A
D.__mro__          # (D, B, C, A, object)
```

⚙️ **`super()` does not mean "my parent."** It means "the next class in the MRO of `type(self)`." That's why `B.m` calls `C.m` here — `B` alone doesn't know about `C`, but `D`'s MRO does. This is what makes cooperative multiple inheritance work.

C3 rules: a class precedes its parents, parents keep their declared order, and the result is consistent for every subclass. When no valid linearization exists, Python raises `TypeError` at class creation.

### 9.4 `__slots__`

```python
class Point:
    __slots__ = ('x', 'y')      # no per-instance __dict__
    def __init__(self, x, y): self.x, self.y = x, y
```

| | Regular | `__slots__` |
|---|---|---|
| Memory per instance | ~56 B + dict (~104 B) | ~48 B |
| Attribute access | dict lookup | array offset — faster |
| Add new attributes | ✅ | ❌ `AttributeError` |
| Weak references | ✅ | only with `'__weakref__'` in slots |

Use for classes with many instances (millions of points, graph nodes). `@dataclass(slots=True)` gives it for free.

### 9.5 Metaclasses

```python
class RegistryMeta(type):
    registry = {}
    def __new__(mcls, name, bases, namespace, **kwargs):
        cls = super().__new__(mcls, name, bases, namespace)
        if bases:                                # skip the base class itself
            mcls.registry[name.lower()] = cls
        return cls

class Plugin(metaclass=RegistryMeta): ...
class JsonPlugin(Plugin): ...
RegistryMeta.registry        # {'jsonplugin': <class JsonPlugin>}
```

```
   Class creation order:
   1. Body executes into a namespace dict
   2. metaclass.__prepare__(name, bases)   → namespace type
   3. metaclass.__new__(mcls, name, bases, ns)  → the class object
   4. metaclass.__init__(cls, ...)
   5. Later: MyClass(...)  →  metaclass.__call__  →  cls.__new__ → cls.__init__
```

🏭 **You almost never need a metaclass.** `__init_subclass__` covers most registry/validation use cases with far less complexity:

```python
class Plugin:
    registry = {}
    def __init_subclass__(cls, /, name=None, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry[name or cls.__name__.lower()] = cls

class CsvPlugin(Plugin, name='csv'): ...
```

---

## 10. The GIL and Concurrency

### 10.1 What the GIL Is

The Global Interpreter Lock is a mutex ensuring only one thread executes Python bytecode at a time in a CPython process.

```
   4 CPU cores, 4 Python threads, CPU-bound work:

   Core 1  ████░░░░████░░░░████░░░░     only one thread
   Core 2  ░░░░████░░░░░░░░░░░░░░░░     holds the GIL
   Core 3  ░░░░░░░░░░░░████░░░░░░░░     at any moment
   Core 4  ░░░░░░░░░░░░░░░░░░░░████

   Total throughput ≈ 1 core (plus switching overhead)
```

**But the GIL is released during I/O:**

```
   4 threads doing network I/O:

   T1  [GIL]──wait for socket────────[GIL]
   T2      [GIL]──wait for socket────────
   T3          [GIL]──wait for socket────
   T4              [GIL]──wait──────────

   Near-perfect overlap → threads ARE the right tool for I/O
```

It's also released inside C extensions that opt in: NumPy, Pillow, `hashlib`, compression, and most database drivers.

### 10.2 Choosing a Concurrency Model

```
                    What is the bottleneck?
                             │
             ┌───────────────┴────────────────┐
             ▼                                ▼
         I/O-BOUND                       CPU-BOUND
    (network, disk, DB)              (math, parsing, ML)
             │                                │
      ┌──────┴──────┐                         ▼
      ▼             ▼                  multiprocessing /
  thousands      dozens               ProcessPoolExecutor
  of tasks       of tasks             (or NumPy / Rust ext /
      │             │                  release the GIL in C)
      ▼             ▼
   asyncio      threading
```

| | `threading` | `multiprocessing` | `asyncio` |
|---|---|---|---|
| Parallel CPU | ❌ (GIL) | ✅ | ❌ |
| Memory per unit | ~8 MB stack | full process (~30 MB+) | ~KB per task |
| Practical scale | hundreds | ~n_cores | tens of thousands |
| Data sharing | shared memory (needs locks) | pickle/IPC | shared memory, single thread |
| Preemptive | ✅ | ✅ | ❌ cooperative — one blocking call stalls all |

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# I/O-bound
with ThreadPoolExecutor(max_workers=32) as ex:
    results = list(ex.map(fetch_url, urls))

# CPU-bound
with ProcessPoolExecutor() as ex:
    results = list(ex.map(heavy_compute, chunks))
```

### 10.3 Threading Primitives

```python
import threading

lock = threading.Lock()             # basic mutual exclusion
rlock = threading.RLock()           # re-entrant: same thread can acquire twice
sem = threading.Semaphore(5)        # limit concurrency to 5
event = threading.Event()           # one-to-many signal
cond = threading.Condition()        # wait for a predicate
barrier = threading.Barrier(3)      # all three arrive before any proceeds
local = threading.local()           # per-thread storage

with lock:                          # always use `with` — exception-safe
    shared_counter += 1
```

**Deadlock avoidance:** always acquire multiple locks in a globally consistent order, and prefer `lock.acquire(timeout=...)` in code that can retry.

### 10.4 Free-Threaded Python

Python 3.13 ships an experimental build (`--disable-gil`, PEP 703) where the GIL can be turned off. Single-threaded performance takes a hit and C extensions must be updated. Worth knowing about for interviews; not yet the default deployment choice.

---

## 11. asyncio

### 11.1 The Model

```
   ┌───────────────────────────────────────────────────────┐
   │                     EVENT LOOP                        │
   │                                                       │
   │   ready queue:  [task A]  [task B]  [task C]          │
   │                     │                                 │
   │                     ▼  run until it awaits            │
   │              ┌─────────────┐                          │
   │              │  Coroutine  │── await socket read ──┐  │
   │              └─────────────┘                       │  │
   │                     ▲                              ▼  │
   │                     │                     ┌───────────┐│
   │                     └── resume when ready │  selector ││
   │                                           │ (epoll/   ││
   │                                           │  kqueue)  ││
   │                                           └───────────┘│
   └───────────────────────────────────────────────────────┘

   ONE thread. Concurrency comes from tasks yielding at `await`.
```

### 11.2 Core API

```python
import asyncio

async def fetch(session, url):
    async with session.get(url) as resp:
        return await resp.json()

async def main():
    # ── Run concurrently ────────────────────────
    results = await asyncio.gather(*[fetch(s, u) for u in urls])
    # return_exceptions=True → failures come back as exception objects

    # ── TaskGroup (3.11+) — preferred: structured concurrency ──
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(fetch(s, url1))
        t2 = tg.create_task(fetch(s, url2))
    # exits only when ALL finish; any failure cancels the siblings
    # and raises an ExceptionGroup

    # ── Timeouts ────────────────────────────────
    async with asyncio.timeout(5):          # 3.11+
        await slow_operation()

    # ── Concurrency limit ───────────────────────
    sem = asyncio.Semaphore(10)
    async def limited(u):
        async with sem:
            return await fetch(s, u)

    # ── Run blocking code without stalling the loop ──
    result = await asyncio.to_thread(blocking_io_call, arg)

    # ── Queues for producer/consumer ────────────
    q = asyncio.Queue(maxsize=100)
    await q.put(item); item = await q.get(); q.task_done()
    await q.join()

asyncio.run(main())
```

### 11.3 The Cardinal Sin

```python
# ❌ Blocks the ENTIRE loop — every other task freezes
async def bad():
    time.sleep(5)                 # blocking
    requests.get(url)             # blocking
    heavy_cpu_computation()       # blocking

# ✅
async def good():
    await asyncio.sleep(5)
    async with aiohttp.ClientSession() as s: await s.get(url)
    await asyncio.to_thread(heavy_cpu_computation)      # or a process pool
```

🏭 In production, enable `asyncio.run(main(), debug=True)` in staging — it warns about coroutines that block the loop for more than 100 ms.

### 11.4 Cancellation

```python
task = asyncio.create_task(work())
task.cancel()
try:
    await task
except asyncio.CancelledError:
    ...                      # re-raise unless you're the one who cancelled

# Protect cleanup from cancellation
async def work():
    try:
        await do_stuff()
    except asyncio.CancelledError:
        await asyncio.shield(cleanup())    # cleanup completes even so
        raise                              # ALWAYS re-raise
```

⚠️ Swallowing `CancelledError` breaks timeouts and shutdown. Catch it only to clean up, then re-raise.

---

## 12. Memory Management

### 12.1 Reference Counting + Cycle GC

```python
import sys
a = []
sys.getrefcount(a)     # 2 — one for `a`, one for the getrefcount argument
```

Reference counting frees objects immediately at zero — deterministic and cache-friendly, but it cannot free cycles:

```python
a = {}; b = {}
a['b'] = b; b['a'] = a       # refcount never reaches 0
del a, b                     # leaked without the cycle collector
```

The generational cycle collector handles those:

```
   Gen 0 ──survives──▶ Gen 1 ──survives──▶ Gen 2
   (scanned often)      (less often)        (rarely)

   Thresholds default to (700, 10, 10):
   gen0 runs after 700 net allocations; gen1 after 10 gen0 runs; etc.
```

```python
import gc
gc.collect()             # force a full collection
gc.disable()             # safe ONLY if you create no cycles (rare)
gc.set_threshold(2000, 20, 20)
gc.get_referrers(obj)    # who is keeping this alive
```

🏭 A classic production win: `gc.freeze()` after startup in a pre-forking server (Gunicorn/uWSGI) moves startup objects to a permanent generation so copy-on-write memory isn't dirtied by GC writes in every worker.

### 12.2 Profiling Memory

```python
import tracemalloc
tracemalloc.start()
snap1 = tracemalloc.take_snapshot()
run_workload()
snap2 = tracemalloc.take_snapshot()
for stat in snap2.compare_to(snap1, 'lineno')[:10]:
    print(stat)
```

Other tools: `memray` (best-in-class, allocation flamegraphs), `objgraph` (reference chains), `psutil` (RSS over time).

---

## 13. Type Hints

Hints are erased at runtime by default; they exist for tooling.

```python
from typing import (Optional, Union, Literal, TypedDict, Protocol, TypeVar,
                    Generic, Callable, Iterator, overload, Final, ClassVar,
                    Annotated, NewType, Self, ParamSpec, TypeAlias)
from collections.abc import Sequence, Mapping, Iterable

# Modern syntax (3.10+)
def f(x: int | None = None) -> list[str]: ...

# Type variables and generics (3.12+ syntax)
def first[T](items: Sequence[T]) -> T | None:
    return items[0] if items else None

class Stack[T]:
    def __init__(self) -> None: self._items: list[T] = []
    def push(self, item: T) -> None: self._items.append(item)
    def pop(self) -> T: return self._items.pop()

# Protocol — structural typing, no inheritance required
class Comparable(Protocol):
    def __lt__(self, other: Self, /) -> bool: ...

def maximum[T: Comparable](items: Iterable[T]) -> T:
    return max(items)

# TypedDict — typed dict shapes
class UserDict(TypedDict):
    id: int
    name: str
    email: NotRequired[str]

# Literal + exhaustiveness
Method = Literal['GET', 'POST', 'PUT', 'DELETE']

# NewType — cheap nominal typing
UserId = NewType('UserId', int)

# Annotated — metadata for validators (this is how FastAPI/Pydantic work)
Age = Annotated[int, Field(ge=0, le=150)]

# ParamSpec — preserve a decorated function's signature
P = ParamSpec('P'); R = TypeVar('R')
def logged(fn: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return fn(*args, **kwargs)
    return wrapper

# overload — different signatures for different inputs
@overload
def parse(x: str) -> str: ...
@overload
def parse(x: int) -> int: ...
def parse(x): return x
```

🏭 Run `mypy --strict` or `pyright` in CI. Hints without a checker are just comments.

---

## 14. Error Handling

```python
class AppError(Exception):
    """Base for all application errors — lets callers catch one thing."""

class NotFound(AppError): ...
class ValidationError(AppError):
    def __init__(self, field: str, message: str):
        self.field, self.message = field, message
        super().__init__(f"{field}: {message}")

try:
    risky()
except ValidationError as e:
    handle(e.field)
except (NotFound, TimeoutError) as e:
    retry()
except AppError:
    raise                                 # re-raise, preserving traceback
else:
    commit()                              # runs only if NO exception
finally:
    cleanup()                             # always runs
```

**Exception chaining:**

```python
try:
    parse(raw)
except ValueError as e:
    raise ConfigError("bad config") from e     # __cause__ — explicit chain
    # raise ConfigError(...) from None         # suppress the original
```

**Exception groups (3.11+):**

```python
try:
    async with asyncio.TaskGroup() as tg: ...
except* ValueError as eg:          # handles ValueErrors in the group
    ...
except* TypeError as eg:           # and TypeErrors, independently
    ...
```

### Context managers

```python
from contextlib import contextmanager, suppress, ExitStack, asynccontextmanager

@contextmanager
def timed(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.3f}s")

with suppress(FileNotFoundError):
    os.remove(path)

with ExitStack() as stack:                      # dynamic number of contexts
    files = [stack.enter_context(open(p)) for p in paths]

# Class-based
class Transaction:
    def __enter__(self):
        self.conn = connect(); self.conn.begin(); return self.conn
    def __exit__(self, exc_type, exc, tb):
        if exc_type: self.conn.rollback()
        else: self.conn.commit()
        self.conn.close()
        return False                # False → propagate; True → suppress
```

---

## 15. Performance

### 15.1 Rules

| Rule | Gain |
|---|---|
| Use the right data structure (set vs list membership) | 100–1000× |
| Move loops into C (`sum`, `map`, comprehensions, NumPy) | 2–100× |
| Avoid attribute lookups in tight loops (bind to a local) | 10–30% |
| Use generators for large data | memory, not speed |
| `join` instead of `+=` for strings | O(n²) → O(n) |
| `__slots__` for many instances | 40–50% memory |
| Cache with `functools.cache` | depends |

```python
# String building
s = ''.join(parts)                        # ✅ O(n)
# not: for p in parts: s += p             # ❌ O(n²)

# Local binding in a hot loop
append = result.append                    # avoid re-lookup each iteration
for x in data: append(x * 2)

# Comprehensions beat explicit loops (specialized bytecode)
squares = [x*x for x in range(n)]
```

### 15.2 Profiling

```bash
python -m cProfile -s cumtime script.py       # function-level
python -m timeit -s "setup" "statement"       # microbenchmarks
py-spy top --pid 1234                          # sampling, on a LIVE process
py-spy record -o flame.svg --pid 1234          # flamegraph, no restart needed
```

```python
# line_profiler for line-by-line
@profile
def slow(): ...
# kernprof -l -v script.py
```

🏭 `py-spy` is the production tool — it attaches to a running process with no code change and near-zero overhead.

---

## 16. Modern Python

```python
# ── Structural pattern matching (3.10) ──────────────────
match command:
    case {'action': 'move', 'x': int(x), 'y': int(y)}:
        move(x, y)
    case ['rotate', angle] if angle % 90 == 0:
        rotate(angle)
    case Point(x=0, y=0):
        origin()
    case str() as s:
        parse(s)
    case _:
        raise ValueError

# ── Walrus operator (3.8) ───────────────────────────────
while (chunk := f.read(8192)):
    process(chunk)
if (m := re.match(pat, s)) is not None:
    use(m.group(1))

# ── f-string power ──────────────────────────────────────
f"{value=}"           # "value=42"   — debugging
f"{x:>10.2f}"         # alignment + precision
f"{n:,}"              # 1,234,567
f"{ratio:.1%}"        # "45.3%"
f"{d:%Y-%m-%d}"       # date formatting

# ── Dict/set operations ─────────────────────────────────
merged = defaults | overrides                # 3.9
d |= updates

# ── dataclasses ─────────────────────────────────────────
from dataclasses import dataclass, field, asdict, replace
@dataclass(frozen=True, slots=True, kw_only=True, order=True)
class Point:
    x: float
    y: float
    tags: list[str] = field(default_factory=list, compare=False)

# ── pathlib over os.path ────────────────────────────────
from pathlib import Path
p = Path('data') / 'file.txt'
p.read_text(); p.exists(); p.stem; list(Path('.').rglob('*.py'))

# ── enum ────────────────────────────────────────────────
from enum import Enum, StrEnum, auto, Flag
class Status(StrEnum):        # 3.11 — behaves like a str
    ACTIVE = 'active'
    BANNED = 'banned'

class Perm(Flag):
    READ = auto(); WRITE = auto(); EXEC = auto()
p = Perm.READ | Perm.WRITE
Perm.READ in p                # True
```

---

## 17. Interview Section

<details>
<summary><b>Q1. Explain the GIL. Does it make Python useless for concurrency?</b></summary>

The GIL is a mutex letting only one thread run Python bytecode at a time in CPython. It exists because CPython's reference counting isn't thread-safe, and a single global lock was simpler and faster for single-threaded code than fine-grained locking.

It doesn't make Python useless for concurrency — it makes *threads* useless for CPU parallelism specifically. The GIL is released during I/O waits and inside many C extensions (NumPy, hashlib, DB drivers), so threads work well for I/O-bound workloads. For CPU parallelism you use `multiprocessing`, a C extension, or a native library. For high-concurrency I/O, asyncio scales far better than threads because tasks cost kilobytes instead of megabytes.

Python 3.13 ships an experimental free-threaded build (PEP 703) that removes it, at some single-threaded cost.
</details>

<details>
<summary><b>Q2. `is` vs `==`?</b></summary>

`==` calls `__eq__` and compares values. `is` compares identity — whether the two names point to the same object.

Use `is` only for singletons: `None`, `True`, `False`, and sentinel objects. Using it for values appears to work because CPython caches small integers (-5 to 256) and interns some strings, but those are implementation details that break unpredictably.
</details>

<details>
<summary><b>Q3. Why is the mutable default argument a problem?</b></summary>

Default values are evaluated once, when the `def` statement runs, and stored on the function object. So a mutable default like `[]` is shared by every call that doesn't pass the argument, and mutations persist across calls.

The fix is `def f(x=None)` then `if x is None: x = []` inside the body, which creates a fresh object per call.
</details>

<details>
<summary><b>Q4. Shallow vs deep copy.</b></summary>

Shallow copy creates a new outer container whose elements are the *same objects* as the original's. Deep copy recursively copies everything, using a memo dict so shared references and cycles are preserved rather than duplicated or infinite-looping.

`copy.copy` vs `copy.deepcopy`. Deep copy is expensive; often the better answer is immutability, so no copy is needed.
</details>

<details>
<summary><b>Q5. How do decorators work?</b></summary>

`@d` above `def f` is exactly `f = d(f)`. A decorator receives the function object and returns a replacement — usually a closure wrapping the original.

Always apply `functools.wraps` to the wrapper so `__name__`, `__doc__`, `__module__`, and `__wrapped__` survive; otherwise debugging, docs tooling, and frameworks that introspect signatures break.

A decorator that takes arguments is one level deeper: a function returning a decorator.
</details>

<details>
<summary><b>Q6. What does `yield` do, and what's a generator good for?</b></summary>

`yield` turns a function into a generator function. Calling it doesn't run the body — it returns a generator object. Each `next()` runs until the next `yield`, then freezes the entire frame: locals, instruction pointer, everything.

Generators give lazy evaluation and constant memory over arbitrarily large sequences. They also compose into pipelines where each stage pulls from the previous, so a 10 GB file processes in a few MB. With `send()`, they can also receive values, which is how coroutines were built before `async`/`await`.
</details>

<details>
<summary><b>Q7. Explain the MRO and what `super()` actually does.</b></summary>

The MRO is the linear order Python searches for attributes, computed by the C3 algorithm: a class precedes its parents, parents keep their declared order, and the ordering is monotonic across subclasses.

`super()` does not mean "my superclass." It means "the next class after me in the MRO of `type(self)`." That's the key insight — in a diamond, `B.method` calling `super().method()` can land on `C`, a class `B` knows nothing about. This is what makes cooperative multiple inheritance possible, and why every class in such a hierarchy must call `super()`.
</details>

<details>
<summary><b>Q8. What are `__slots__` and when do you use them?</b></summary>

`__slots__` replaces the per-instance `__dict__` with a fixed array of descriptors. Instances shrink by roughly half and attribute access becomes an array offset instead of a dict lookup.

The cost: you can't add attributes not listed, and you lose `__dict__` and (unless declared) weak-reference support. Worth it when you have very many instances — nodes, points, records. `@dataclass(slots=True)` is the ergonomic way to get it.
</details>

<details>
<summary><b>Q9. How does Python manage memory?</b></summary>

Two mechanisms. Reference counting frees objects the instant their count hits zero — deterministic and prompt, but blind to cycles. A generational cycle collector periodically scans for unreachable cycles, with three generations scanned at decreasing frequency on the theory that most objects die young.

Below that, small objects come from `pymalloc`'s arena/pool allocator rather than `malloc`, which is why freed memory often isn't returned to the OS.
</details>

<details>
<summary><b>Q10. `@staticmethod` vs `@classmethod` vs instance method?</b></summary>

Instance methods take `self` and operate on an instance. `@classmethod` takes `cls` and operates on the class — the main use is alternative constructors (`dict.fromkeys`, `datetime.now`), and it respects inheritance, so a subclass calling it gets its own class. `@staticmethod` takes neither; it's a plain function namespaced inside the class for organization.

Prefer `@classmethod` over `@staticmethod` for factories precisely because of the inheritance behavior.
</details>

<details>
<summary><b>Q11. Why must `__hash__` and `__eq__` agree?</b></summary>

Hash-based containers find a candidate bucket by hash, then confirm with `==`. If two equal objects hash differently, they land in different buckets and the container will store both, breaking set semantics and dict lookups.

Also: a hash must be stable for an object's lifetime as a key, so it must be computed from immutable fields. Defining `__eq__` without `__hash__` makes the class unhashable, which is Python protecting you from exactly this bug.
</details>

<details>
<summary><b>Q12. List vs tuple — beyond "mutable vs immutable"?</b></summary>

Semantically, lists are homogeneous sequences of varying length; tuples are heterogeneous fixed-size records where position has meaning. That's the intent behind the choice.

Mechanically, tuples are hashable (usable as dict keys), slightly smaller, don't over-allocate, and CPython caches small tuples for reuse. Constant tuples are also folded at compile time. But the main reason to reach for a tuple is signaling "this is a fixed record" — and for that, a `NamedTuple` or frozen dataclass is usually clearer still.
</details>

### 🎤 Output prediction

<details>
<summary><b>P1</b></summary>

```python
def f(a, b=[]):
    b.append(a); return b
print(f(1)); print(f(2))
```
**`[1]`** then **`[1, 2]`** — shared mutable default.
</details>

<details>
<summary><b>P2</b></summary>

```python
print([lambda: i for i in range(3)][0]())
```
**`2`** — late binding; the loop finished before any lambda ran.
</details>

<details>
<summary><b>P3</b></summary>

```python
a = [1, 2, 3]; b = a[:]; c = a
a.append(4)
print(b, c)
```
**`[1, 2, 3] [1, 2, 3, 4]`** — `b` is a copy, `c` is an alias.
</details>

<details>
<summary><b>P4</b></summary>

```python
print(0.1 + 0.2 == 0.3)
print(round(2.5), round(3.5))
```
**`False`**, then **`2 4`** — binary float representation; and Python uses banker's rounding (round-half-to-even).
</details>

<details>
<summary><b>P5</b></summary>

```python
t = ([1, 2], 3)
try: t[0] += [4]
except TypeError: pass
print(t)
```
**`([1, 2, 4], 3)`** — `+=` mutated the list via `__iadd__`, *then* the tuple assignment failed. Both happened.
</details>

<details>
<summary><b>P6</b></summary>

```python
print({True: 'a', 1: 'b', 1.0: 'c'})
```
**`{True: 'c'}`** — `True == 1 == 1.0` and they hash identically, so it's one key; the first key object is kept but the value is overwritten twice.
</details>

<details>
<summary><b>P7</b></summary>

```python
class A:
    x = []
a, b = A(), A()
a.x.append(1)
print(b.x)
```
**`[1]`** — `x` is a class attribute shared by all instances. Mutating it affects everyone.
</details>

<details>
<summary><b>P8</b></summary>

```python
def gen():
    try: yield 1; yield 2
    finally: print('cleanup')
g = gen(); next(g); del g
```
**`cleanup`** — closing a generator throws `GeneratorExit` at the yield point, running `finally` blocks.
</details>

### 🎤 Implementation drills

1. LRU cache with `OrderedDict` (and from scratch with a dict + doubly linked list)
2. A context manager both as a class and with `@contextmanager`
3. A retry decorator with exponential backoff
4. A rate limiter (token bucket)
5. Flatten an arbitrarily nested list, iteratively
6. A generator-based CSV/log processing pipeline
7. Merge sorted iterators lazily (like `heapq.merge`)
8. A thread-safe singleton
9. `deepcopy` with cycle handling
10. An async worker pool with a bounded queue

---

## 18. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        PYTHON — ONE PAGE                             ║
╠══════════════════════════════════════════════════════════════════════╣
║ NAMES bind to OBJECTS. Assignment rebinds, never copies.             ║
║ Mutable default args are created ONCE → use None sentinel            ║
║ Closures capture VARIABLES not values → use default-arg trick        ║
║ `is` only for None/True/False/sentinels                              ║
╠══════════════════════════════════════════════════════════════════════╣
║ COMPLEXITY                                                           ║
║   list:  append O(1) | insert(0)/pop(0) O(n) | `in` O(n)             ║
║   dict/set: get/set/in O(1) avg — ordered since 3.7                  ║
║   deque: O(1) both ends       heapq: push/pop O(log n)               ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONCURRENCY                                                          ║
║   I/O + many tasks  → asyncio                                        ║
║   I/O + few tasks   → threading                                      ║
║   CPU-bound         → multiprocessing / NumPy / C ext                ║
║   GIL released during I/O and in C extensions                        ║
║   NEVER block inside async — use asyncio.to_thread                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ DATA MODEL: syntax → dunder                                          ║
║   len→__len__  x in y→__contains__  with→__enter__/__exit__          ║
║   for→__iter__/__next__   f()→__call__   a+b→__add__/__radd__        ║
║   __eq__ and __hash__ MUST agree, on immutable fields                ║
╠══════════════════════════════════════════════════════════════════════╣
║ MRO: C3 linearization. super() = NEXT IN MRO, not "parent"           ║
║ Lookup: data descriptor > instance dict > class attr > __getattr__   ║
╠══════════════════════════════════════════════════════════════════════╣
║ MEMORY: refcount (immediate) + generational cycle GC                 ║
║   __slots__ halves instance size; gc.freeze() before fork            ║
║   profile with tracemalloc / memray / py-spy                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ MODERN: match/case · walrus := · f"{x=}" · dict | merge              ║
║         dataclass(frozen,slots,kw_only) · pathlib · StrEnum          ║
║         TaskGroup · asyncio.timeout · ExceptionGroup/except*         ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [FastAPI](../03-backend/fastapi.md) · [Data Engineering](../08-data-ai/data-engineering.md) · [DSA Patterns](../04-dsa/00-patterns.md)
