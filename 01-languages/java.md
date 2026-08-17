# ☕ Java Deep Dive

> Java's power in interviews comes from the JVM. Know the memory model, garbage collection, and concurrency primitives and you can answer almost anything.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [JVM Architecture](#2-jvm-architecture)
3. [Memory Layout](#3-memory-layout)
4. [Garbage Collection](#4-garbage-collection)
5. [Type System and Generics](#5-type-system-and-generics)
6. [Collections Framework](#6-collections-framework)
7. [equals, hashCode, and Contracts](#7-equals-hashcode-contracts)
8. [Concurrency](#8-concurrency)
9. [The Java Memory Model](#9-the-java-memory-model)
10. [Streams and Functional Java](#10-streams-and-functional-java)
11. [Exceptions](#11-exceptions)
12. [Modern Java (8→21+)](#12-modern-java)
13. [Performance](#13-performance)
14. [Interview Section](#14-interview-section)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. Mental Model

🧠 Java is a statically typed, class-based, garbage-collected language that compiles to portable bytecode and is JIT-compiled to machine code at runtime. The two things that make Java behave the way it does: **everything except primitives is a reference to a heap object**, and **the JVM is a sophisticated runtime that optimizes based on what your program actually does.**

```
   Foo.java
      │  javac  (compile once)
      ▼
   Foo.class  (bytecode — platform independent)
      │
      ▼
   ┌───────────────────────────────────────────────────────┐
   │                        JVM                            │
   │  ┌──────────────┐   ┌──────────────┐  ┌────────────┐  │
   │  │ ClassLoader  │──▶│ Runtime Data │◀─│  Execution │  │
   │  │  Bootstrap   │   │    Areas     │  │   Engine   │  │
   │  │  Platform    │   │  heap/stack/ │  │ interpreter│  │
   │  │  Application │   │  metaspace   │  │  + C1/C2   │  │
   │  └──────────────┘   └──────────────┘  │  JIT + GC  │  │
   │                                       └────────────┘  │
   └───────────────────────────────────────────────────────┘
                          │
                          ▼
                  native machine code
```

### Pass by value — always

```java
void mutate(StringBuilder sb, int n) {
    sb.append("!");        // mutates the object the reference points to → visible
    sb = new StringBuilder("new");   // rebinds the LOCAL copy → not visible
    n = 99;                          // local copy → not visible
}
```

Java passes *references by value*. The reference is copied; the object is not.

---

## 2. JVM Architecture

### 2.1 Class Loading

```
   LOADING      → find the .class bytes, create a Class object
       ▼
   LINKING
     ├ Verification  → bytecode is well-formed and safe
     ├ Preparation   → static fields get DEFAULT values (0/null/false)
     └ Resolution    → symbolic refs → direct refs (may be lazy)
       ▼
   INITIALIZATION → run static initializers and static field assignments
                    (<clinit>, thread-safe, exactly once)
```

**Delegation model:** each loader asks its parent first.

```
   Bootstrap (java.base, native)
        ▲
   Platform (JDK modules)
        ▲
   Application (your classpath)
        ▲
   Custom (web app, plugin isolation)
```

This prevents a malicious `java.lang.String` on your classpath from replacing the real one.

### 2.2 JIT Compilation

```
   Method invocation count / loop back-edge count grows
        │
        ▼
   Interpreted ──▶ C1 (client, fast compile, light optimization, profiling)
                        │
                        ▼  becomes "hot" (~10k invocations)
                   C2 (server, slow compile, aggressive optimization)
                        │
                        ▼ assumption invalidated
                   Deoptimize → back to interpreter, re-profile
```

**Key optimizations C2 performs:**

| Optimization | What it does |
|---|---|
| Inlining | Copies small method bodies into callers — enables everything else |
| Escape analysis | Object never leaves the method → allocate on stack or eliminate |
| Loop unrolling | Fewer branch checks |
| Dead code elimination | Removes provably unused work |
| Monomorphic dispatch | Virtual call with one observed type → direct call + guard |
| Lock elision | Removes locks on thread-confined objects |

⚠️ This is why microbenchmarks lie. Use JMH — it handles warmup, dead-code elimination, and constant folding.

---

## 3. Memory Layout

```
   ┌──────────────────────── JVM MEMORY ─────────────────────────┐
   │                                                             │
   │  PER THREAD                    SHARED                       │
   │  ┌──────────────────┐   ┌───────────────────────────────┐   │
   │  │  JVM Stack       │   │            HEAP               │   │
   │  │  ┌────────────┐  │   │  ┌─────────────────────────┐  │   │
   │  │  │ frame      │  │   │  │  YOUNG GENERATION       │  │   │
   │  │  │  locals    │  │   │  │  ┌──────┬─────┬─────┐   │  │   │
   │  │  │  operands  │──┼───┼─▶│  │ Eden │ S0  │ S1  │   │  │   │
   │  │  │  ref to CP │  │   │  │  └──────┴─────┴─────┘   │  │   │
   │  │  ├────────────┤  │   │  ├─────────────────────────┤  │   │
   │  │  │ frame      │  │   │  │  OLD GENERATION         │  │   │
   │  │  └────────────┘  │   │  │  (tenured objects)      │  │   │
   │  │  StackOverflow   │   │  └─────────────────────────┘  │   │
   │  │  if too deep     │   │   OutOfMemoryError: Java heap │   │
   │  ├──────────────────┤   ├───────────────────────────────┤   │
   │  │  PC Register     │   │   METASPACE (native memory)   │   │
   │  ├──────────────────┤   │   class metadata, method code │   │
   │  │  Native Stack    │   ├───────────────────────────────┤   │
   │  └──────────────────┘   │   CODE CACHE (JIT output)     │   │
   │                         ├───────────────────────────────┤   │
   │                         │   String pool (in heap 7+)    │   │
   │                         └───────────────────────────────┘   │
   └─────────────────────────────────────────────────────────────┘
```

| Region | Holds | Failure mode |
|---|---|---|
| Stack | Frames: locals, partial results, operand stack | `StackOverflowError` |
| Heap | All objects and arrays | `OutOfMemoryError: Java heap space` |
| Metaspace | Class metadata (native mem, auto-grows) | `OutOfMemoryError: Metaspace` |
| Code cache | JIT-compiled native code | Full → falls back to interpretation |

### Object header

```
   64-bit JVM, compressed oops:

   ┌────────────────┬──────────────┬──────────────────┐
   │  Mark word     │  Klass ptr   │   Fields...      │
   │  (8 bytes)     │  (4 bytes)   │                  │
   │  hash, GC age, │  → class     │  padded to an    │
   │  lock state    │    metadata  │  8-byte boundary │
   └────────────────┴──────────────┴──────────────────┘

   Minimum object: 16 bytes. An empty object costs 16 B;
   an Integer costs 16 B to hold 4 bytes of data.
```

This is why `int[]` vastly outperforms `List<Integer>` for numeric work — the latter is an array of pointers to 16-byte boxes scattered across the heap.

---

## 4. Garbage Collection

### 4.1 The Generational Cycle

```
   1. New objects → EDEN
   
      Eden [████████] S0 [    ] S1 [    ]   Old [        ]
   
   2. Eden fills → MINOR GC (stop-the-world, short)
      Live objects copied to S0; Eden cleared
   
      Eden [        ] S0 [██  ] S1 [    ]   Old [        ]
   
   3. Next minor GC: live from Eden AND S0 → S1; ages incremented
   
      Eden [        ] S0 [    ] S1 [███ ]   Old [        ]
   
   4. Age > threshold (default 15) → PROMOTED to Old
   
      Eden [        ] S0 [    ] S1 [    ]   Old [███     ]
   
   5. Old fills → MAJOR/FULL GC (expensive)
```

**Weak generational hypothesis:** most objects die young. So collect Eden constantly and cheaply (copying collectors are O(live), not O(heap)), and touch the old generation rarely.

### 4.2 Collector Comparison

| Collector | Flag | Pause | Throughput | Best for |
|---|---|---|---|---|
| Serial | `-XX:+UseSerialGC` | High | Good | Tiny heaps, containers with 1 CPU |
| Parallel | `-XX:+UseParallelGC` | High | **Best** | Batch jobs, throughput over latency |
| **G1** (default) | `-XX:+UseG1GC` | ~10-200 ms | Good | General purpose, heaps 4-64 GB |
| ZGC | `-XX:+UseZGC` | **<1 ms** | Slightly lower | Low-latency services, huge heaps |
| Shenandoah | `-XX:+UseShenandoahGC` | <10 ms | Slightly lower | Low latency, RedHat ecosystem |
| Epsilon | `-XX:+UseEpsilonGC` | N/A | N/A | No-op; for benchmarking/short jobs |

**G1's model:** the heap is split into ~2048 equal regions, each dynamically labelled Eden, Survivor, Old, or Humongous. G1 collects the regions with the most garbage first — hence "Garbage First" — and targets a pause goal you set with `-XX:MaxGCPauseMillis=200`.

```
   G1 heap regions

   ┌──┬──┬──┬──┬──┬──┬──┬──┐
   │E │E │O │S │E │O │H │H │   E=Eden  S=Survivor
   ├──┼──┼──┼──┼──┼──┼──┼──┤   O=Old    H=Humongous (>50% of a region)
   │O │E │E │O │S │E │O │  │
   └──┴──┴──┴──┴──┴──┴──┴──┘
```

### 4.3 Reference Types

```java
Strong    Object o = new Object();          // never collected while reachable
Soft      SoftReference<T>                  // collected only under memory pressure → caches
Weak      WeakReference<T>                  // collected at next GC → WeakHashMap, listeners
Phantom   PhantomReference<T>               // for cleanup after finalization; get() always null
```

### 4.4 Diagnostics

```bash
# Unified logging (JDK 9+)
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# Heap dump on OOM — non-negotiable in production
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps

# Live inspection
jcmd <pid> GC.heap_info
jcmd <pid> Thread.print          # thread dump — deadlock detection
jcmd <pid> VM.native_memory summary
jmap -histo:live <pid> | head -30
```

Analyze heap dumps with **Eclipse MAT** (dominator tree → what's actually retaining memory) or VisualVM. For continuous production profiling, **JFR** (`-XX:StartFlightRecording`) costs ~1% overhead.

---

## 5. Type System and Generics

### 5.1 Type Erasure

Generics exist only at compile time. At runtime `List<String>` and `List<Integer>` are both `List`.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
a.getClass() == b.getClass();          // true

// Consequences:
new T[10];                             // ❌ can't create generic arrays
if (x instanceof List<String>) {}      // ❌ can't check parameterized types
class A { void f(List<String> l){} void f(List<Integer> l){} }  // ❌ same erasure
```

The compiler inserts casts and bridge methods; the JVM never sees the type parameters. This preserved backward compatibility when generics arrived in Java 5.

### 5.2 Wildcards — PECS

```
   PRODUCER EXTENDS, CONSUMER SUPER

   List<? extends Number>   →  you READ Numbers out (producer)
                               you CANNOT add (except null)

   List<? super Integer>    →  you WRITE Integers in (consumer)
                               you can only read as Object
```

```java
// Copying: src produces, dst consumes
static <T> void copy(List<? super T> dst, List<? extends T> src) {
    for (T t : src) dst.add(t);
}

List<? extends Number> nums = List.of(1, 2, 3);
Number n = nums.get(0);      // ✅ read
nums.add(4);                 // ❌ compiler can't know the actual element type

List<? super Integer> sink = new ArrayList<Number>();
sink.add(42);                // ✅ write
Object o = sink.get(0);      // only Object guaranteed
```

### 5.3 Bounded Type Parameters

```java
<T extends Comparable<? super T>> T max(List<? extends T> list)

// Multiple bounds — class first, then interfaces
<T extends Number & Comparable<T> & Serializable> void f(T t)

// Recursive bound — the "self type" pattern, used by Enum
abstract class Builder<T extends Builder<T>> {
    @SuppressWarnings("unchecked")
    T self() { return (T) this; }
    T name(String n) { this.name = n; return self(); }
}
```

---

## 6. Collections Framework

```
                        Iterable
                            │
                       Collection ─────────────────┐
                     ┌──────┼──────┐               │
                     ▼      ▼      ▼               ▼
                   List    Set   Queue           Map (not a Collection!)
                     │      │      │               │
      ┌──────────────┤      ├──────┴───┐     ┌─────┼────────┬──────────┐
      ▼        ▼     ▼      ▼          ▼     ▼     ▼        ▼          ▼
  ArrayList Linked Vector HashSet  ArrayDeque HashMap Tree  Linked  Concurrent
            List          LinkedHS  Priority        Map    HashMap   HashMap
                          TreeSet    Queue
```

### 6.1 Complexity Table

| Structure | get | add | contains | remove | Ordering | Null? |
|---|---|---|---|---|---|---|
| `ArrayList` | O(1) | O(1)* | O(n) | O(n) | insertion | ✅ |
| `LinkedList` | O(n) | O(1) ends | O(n) | O(1) w/ iterator | insertion | ✅ |
| `ArrayDeque` | — | O(1) ends | O(n) | O(1) ends | insertion | ❌ |
| `HashMap` | O(1) | O(1) | O(1) | O(1) | none | 1 null key |
| `LinkedHashMap` | O(1) | O(1) | O(1) | O(1) | insertion/access | ✅ |
| `TreeMap` | O(log n) | O(log n) | O(log n) | O(log n) | sorted | ❌ key |
| `HashSet` | — | O(1) | O(1) | O(1) | none | ✅ one |
| `TreeSet` | — | O(log n) | O(log n) | O(log n) | sorted | ❌ |
| `PriorityQueue` | O(1) peek | O(log n) | O(n) | O(log n) poll | heap | ❌ |

\* amortized; growth is 1.5× for `ArrayList`

🏭 **`ArrayDeque` beats `LinkedList` for queues and stacks** in almost every real case — better cache locality, less allocation. And `Stack` (a legacy synchronized `Vector` subclass) should never be used in new code.

### 6.2 HashMap Internals

```
   table (array of buckets, always a power of 2)

   [0] → null
   [1] → Node(k1,v1) → Node(k9,v9)          collision chain
   [2] → null
   [3] → TreeNode (red-black tree)          ≥8 nodes in one bucket
   ...                                       AND table.length ≥ 64
   [15]→ Node(k7,v7)

   index = (n - 1) & hash          where
   hash  = h ^ (h >>> 16)          ("spreading" — mixes high bits into low,
                                    because the mask only uses low bits)
```

**Resize:** when `size > capacity × loadFactor` (default 0.75), capacity doubles and every entry is rehashed. Because capacity is a power of two, each entry either stays at index `i` or moves to `i + oldCapacity` — a cheap split.

**Treeification:** a bucket with ≥8 entries converts to a red-black tree, bounding worst-case lookup at O(log n) instead of O(n). This was added as a defense against hash-collision DoS attacks.

⚠️ **Never use `HashMap` from multiple threads.** In Java 7 a concurrent resize could create a circular list and spin a CPU at 100% forever. Java 8 fixed the infinite loop but concurrent use still loses updates and corrupts state. Use `ConcurrentHashMap`.

### 6.3 Fail-Fast Iteration

```java
for (String s : list) {
    if (cond) list.remove(s);      // 💥 ConcurrentModificationException
}

// Correct options:
list.removeIf(s -> cond);                     // ✅ best
Iterator<String> it = list.iterator();        // ✅ explicit
while (it.hasNext()) { if (cond) it.remove(); }
```

`modCount` is incremented on structural change; the iterator compares it each step. This is best-effort bug detection, not a thread-safety guarantee.

---

## 7. equals, hashCode, Contracts

```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;                       // fast path
        if (!(o instanceof Point p)) return false;        // pattern matching, 16+
        return x == p.x && y == p.y;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
    @Override public String toString() { return "Point[x=%d, y=%d]".formatted(x, y); }
}

// Java 16+: a record gives you all three, correctly, for free
public record Point(int x, int y) {}
```

### The equals contract

| Property | Requirement |
|---|---|
| Reflexive | `x.equals(x)` is true |
| Symmetric | `x.equals(y)` ⟺ `y.equals(x)` |
| Transitive | `x=y` and `y=z` ⟹ `x=z` |
| Consistent | Repeated calls give the same answer |
| Null | `x.equals(null)` is false |

### The hashCode contract

1. Equal objects **must** have equal hash codes.
2. Unequal objects *should* have different hash codes (for performance, not correctness).
3. The value must not change while the object is a key in a hash structure.

⚠️ **The classic symmetry break:**

```java
class Point { ... }
class ColorPoint extends Point {
    Color color;
    @Override public boolean equals(Object o) {
        return o instanceof ColorPoint cp && super.equals(o) && color == cp.color;
    }
}
Point p = new Point(1,2);
ColorPoint cp = new ColorPoint(1,2,RED);
p.equals(cp);    // true
cp.equals(p);    // false   ← contract violated
```

There is no way to extend an instantiable class, add a value field, and preserve the contract. Use composition instead — this is Effective Java Item 10.

---

## 8. Concurrency

### 8.1 Thread States

```
        ┌─────┐  start()   ┌──────────┐
        │ NEW ├───────────▶│ RUNNABLE │◀────────────┐
        └─────┘            └────┬─────┘             │
                                │                   │
             ┌──────────────────┼───────────────┐   │
             │ wait for lock    │ wait()/join() │   │ notify()/
             ▼                  ▼               ▼   │ timeout/
        ┌─────────┐       ┌─────────┐    ┌──────────┴──┐ lock
        │ BLOCKED │       │ WAITING │    │TIMED_WAITING│ acquired
        └─────────┘       └─────────┘    └─────────────┘
                                │
                                ▼
                          ┌────────────┐
                          │ TERMINATED │
                          └────────────┘
```

### 8.2 Synchronization

```java
// intrinsic lock (monitor)
synchronized (lockObject) { /* critical section */ }
public synchronized void m() {}          // locks `this`
public static synchronized void s() {}   // locks the Class object

// explicit lock — more control
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* work */ } finally { lock.unlock(); }    // ALWAYS unlock in finally

if (lock.tryLock(1, TimeUnit.SECONDS)) { ... }   // deadlock avoidance

// read/write — many readers OR one writer
private final ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();   // concurrent reads
rw.writeLock().lock();  // exclusive

// StampedLock — optimistic reads, best throughput for read-heavy data
long stamp = sl.tryOptimisticRead();
int v = value;
if (!sl.validate(stamp)) { stamp = sl.readLock(); try { v = value; } finally { sl.unlockRead(stamp); } }
```

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Release | Automatic | Manual (`finally`) |
| Timeout | ❌ | ✅ `tryLock` |
| Interruptible | ❌ | ✅ |
| Fairness option | ❌ | ✅ (slower) |
| Multiple conditions | ❌ (one wait set) | ✅ `newCondition()` |
| Bytecode support | ✅ (JIT can elide/biased-lock) | Library |

### 8.3 The Concurrency Toolkit

```java
// ── Executors ─────────────────────────────────────────
ExecutorService pool = Executors.newFixedThreadPool(8);
// Better: explicit ThreadPoolExecutor so you control the queue
var pool = new ThreadPoolExecutor(
    4, 16, 60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1000),          // BOUNDED — never unbounded
    new ThreadPoolExecutor.CallerRunsPolicy()); // backpressure

// Java 21: virtual threads — one per task is fine
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i -> executor.submit(() -> handle(i)));
}

// ── Atomics — lock-free via CAS ───────────────────────
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
counter.updateAndGet(x -> x * 2);
counter.compareAndSet(expected, newValue);

LongAdder adder = new LongAdder();     // better than AtomicLong under high contention
adder.increment(); adder.sum();        // striped cells reduce cache-line contention

// ── Concurrent collections ────────────────────────────
ConcurrentHashMap<K,V> map = new ConcurrentHashMap<>();
map.computeIfAbsent(k, key -> expensive(key));    // atomic
map.merge(k, 1, Integer::sum);

CopyOnWriteArrayList<T>       // read-heavy, write-rare (listener lists)
BlockingQueue<T> q = new LinkedBlockingQueue<>(1000);
q.put(item);   // blocks when full   → natural backpressure
q.take();      // blocks when empty

// ── Coordination ──────────────────────────────────────
CountDownLatch latch = new CountDownLatch(3);   // one-shot: wait for N
CyclicBarrier barrier = new CyclicBarrier(3);   // reusable rendezvous
Semaphore sem = new Semaphore(10);              // permit-based limiting
Phaser phaser = new Phaser(3);                  // dynamic, multi-phase

// ── CompletableFuture — async composition ─────────────
CompletableFuture.supplyAsync(() -> fetchUser(id), pool)
    .thenCompose(user -> fetchOrdersAsync(user.id()))   // flatMap
    .thenApply(this::summarize)                          // map
    .thenCombine(otherFuture, this::merge)               // zip
    .orTimeout(3, TimeUnit.SECONDS)
    .exceptionally(ex -> fallback())
    .thenAccept(this::render);
```

⚠️ **`Executors.newFixedThreadPool` uses an unbounded queue.** Under overload it accumulates tasks until OOM instead of shedding load. Always construct `ThreadPoolExecutor` directly with a bounded queue and a rejection policy.

### 8.4 Virtual Threads (Java 21)

```
   PLATFORM THREADS (traditional)      VIRTUAL THREADS (Loom)

   Thread ──1:1──▶ OS thread          Thread ──M:N──▶ carrier threads
   ~1 MB stack each                    ~few KB, grows on heap
   ~thousands max                      ~millions
   Blocking wastes an OS thread        Blocking UNMOUNTS from the carrier
```

```java
Thread.startVirtualThread(() -> { ... });

// Structured concurrency (preview) — child tasks are scoped and cancelled together
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user  = scope.fork(() -> fetchUser(id));
    var order = scope.fork(() -> fetchOrders(id));
    scope.join().throwIfFailed();
    return new Result(user.get(), order.get());
}
```

⚠️ Virtual threads make blocking I/O cheap again — thread-per-request comes back. But `synchronized` blocks *pin* a virtual thread to its carrier (improved in later JDKs); prefer `ReentrantLock` in code that runs on virtual threads. Also, thread pools of virtual threads are pointless — create one per task.

---

## 9. The Java Memory Model

The JMM defines when one thread's writes become visible to another. Without it, the compiler, JIT, and CPU are all free to reorder.

```
   Thread A                     Thread B
   ────────                     ────────
   data = 42;                   while (!ready) {}      ← may loop FOREVER:
   ready = true;                use(data);               `ready` can be
                                                         cached in a register
   Without synchronization, B may:
     • never see ready == true  (no visibility guarantee)
     • see ready == true but data == 0  (reordering)
```

### 9.1 happens-before

If action A *happens-before* action B, then A's effects are visible to B.

| Rule | Guarantee |
|---|---|
| Program order | Within one thread, earlier statements happen-before later ones |
| Monitor lock | Unlock happens-before a subsequent lock of the same monitor |
| `volatile` | A write happens-before every subsequent read of that field |
| Thread start | `t.start()` happens-before everything in `t` |
| Thread join | Everything in `t` happens-before `t.join()` returns |
| Final fields | Correct construction publishes finals safely |
| Transitivity | A hb B and B hb C ⟹ A hb C |

### 9.2 volatile

```java
private volatile boolean running = true;   // ✅ guarantees visibility
```

`volatile` gives **visibility** and prevents reordering across the access. It does **not** give atomicity for compound operations:

```java
volatile int count;
count++;             // ❌ still a race: read, add, write are three steps
```

Use `AtomicInteger` or a lock for that.

### 9.3 Double-Checked Locking

The canonical example of why the JMM matters:

```java
// ❌ BROKEN without volatile
class Singleton {
    private static Singleton instance;
    static Singleton get() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```
The write `instance = new Singleton()` is three steps: allocate, construct, assign. The JIT may reorder assign before construct — so another thread sees a non-null reference to a half-built object.

```java
// ✅ volatile forbids that reordering
private static volatile Singleton instance;

// ✅✅ Better — the holder idiom. Class init is lazy and thread-safe by spec.
class Singleton {
    private Singleton() {}
    private static class Holder { static final Singleton INSTANCE = new Singleton(); }
    static Singleton get() { return Holder.INSTANCE; }
}

// ✅✅✅ Best when serialization/reflection safety matters
enum Singleton { INSTANCE; }
```

---

## 10. Streams and Functional Java

```java
record Employee(String name, String dept, double salary, int age) {}

// Grouping and downstream collectors
Map<String, List<String>> namesByDept = employees.stream()
    .filter(e -> e.salary() > 50_000)
    .collect(groupingBy(Employee::dept,
             mapping(Employee::name, toList())));

Map<String, Double> avgByDept = employees.stream()
    .collect(groupingBy(Employee::dept, averagingDouble(Employee::salary)));

Map<Boolean, List<Employee>> split = employees.stream()
    .collect(partitioningBy(e -> e.age() > 40));

Optional<Employee> top = employees.stream()
    .max(comparingDouble(Employee::salary));

// Multi-key sorting
employees.stream().sorted(
    comparing(Employee::dept)
        .thenComparing(Employee::salary, reverseOrder())
        .thenComparing(Employee::name));

// flatMap
List<String> allSkills = employees.stream()
    .flatMap(e -> e.skills().stream())
    .distinct().sorted().toList();

// Custom collector
var stats = employees.stream().collect(
    teeing(counting(), summingDouble(Employee::salary), Summary::new));
```

### 10.1 Stream Rules

| Rule | Detail |
|---|---|
| Lazy | Nothing runs until a terminal operation |
| Single use | A stream is consumed once — reuse throws |
| No side effects | `forEach` mutating shared state breaks under parallel |
| Order matters | `filter` before `map` does less work |
| Short-circuits | `findFirst`, `anyMatch`, `limit` stop early |

```
   source ──▶ filter ──▶ map ──▶ sorted ──▶ collect
              ├─ stateless ─┤    stateful    terminal
                                 (buffers all elements)
```

⚠️ **Parallel streams:** `parallelStream()` uses the shared common ForkJoinPool. One slow or blocking task starves every other parallel stream in the JVM. Only use it for CPU-bound work over large (>10k elements), easily-splittable sources (arrays, `ArrayList`, ranges) — and measure. `LinkedList` and IO-bound work parallelize terribly.

---

## 11. Exceptions

```
                    Throwable
                   ┌────┴────┐
                Error      Exception
             (don't catch)   ┌──┴───────────────┐
             OOM, Stack   RuntimeException   (checked)
             Overflow     (unchecked)        IOException
                          NPE, IAE, ISE      SQLException
```

| | Checked | Unchecked |
|---|---|---|
| Must declare/handle | ✅ | ❌ |
| Means | Recoverable, expected condition | Programming bug |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

```java
// try-with-resources — auto-closes in reverse order, suppresses correctly
try (var conn = pool.getConnection();
     var stmt = conn.prepareStatement(SQL)) {
    return stmt.executeQuery();
} catch (SQLException e) {
    throw new DataAccessException("Query failed for id=" + id, e);   // ALWAYS chain
}

// Multi-catch
catch (IOException | SQLException e) { ... }
```

**Rules that matter:**
- Never swallow: `catch (Exception e) {}` is how bugs become invisible.
- Never catch `Throwable` or `Error` — you can't recover from `OutOfMemoryError`.
- Always preserve the cause when wrapping.
- Don't use exceptions for control flow — they're expensive (stack trace capture).
- Restore the interrupt flag: `catch (InterruptedException e) { Thread.currentThread().interrupt(); }`

---

## 12. Modern Java

```java
// ── Records (16) — immutable data carriers ──────────────
public record Point(int x, int y) {
    public Point {                                  // compact constructor
        if (x < 0) throw new IllegalArgumentException("x must be >= 0");
    }
    public double distance() { return Math.hypot(x, y); }
    static Point origin() { return new Point(0, 0); }
}
// free: constructor, accessors x()/y(), equals, hashCode, toString

// ── Sealed types (17) — closed hierarchies ──────────────
public sealed interface Shape permits Circle, Square, Rectangle {}
record Circle(double r) implements Shape {}
record Square(double side) implements Shape {}
record Rectangle(double w, double h) implements Shape {}

// ── Pattern matching for switch (21) — exhaustive, no default needed ──
double area(Shape s) {
    return switch (s) {
        case Circle c      -> Math.PI * c.r() * c.r();
        case Square q      -> q.side() * q.side();
        case Rectangle r   -> r.w() * r.h();
    };
}

// ── Record patterns (21) — destructuring ────────────────
String describe(Object o) {
    return switch (o) {
        case Point(int x, int y) when x == y -> "diagonal at " + x;
        case Point(int x, int y)             -> "point " + x + "," + y;
        case String s                        -> "string of length " + s.length();
        case null                            -> "null";
        default                              -> "unknown";
    };
}

// ── Text blocks (15) ────────────────────────────────────
String json = """
    { "name": "%s", "age": %d }
    """.formatted(name, age);

// ── var (10) ────────────────────────────────────────────
var map = new HashMap<String, List<Integer>>();

// ── Optional — use as a RETURN type only ────────────────
Optional<User> findUser(String id);
findUser(id).map(User::email).filter(e -> e.contains("@"))
            .ifPresentOrElse(this::send, this::logMissing);
// ❌ never as a field or parameter; never call .get() without isPresent()

// ── Useful APIs ─────────────────────────────────────────
"a,b,c".split(",");  String.join("-", list);  " x ".strip();  "ab".repeat(3);
List.of(1,2,3);  Map.of("a",1);           // immutable factories
list.stream().toList();                    // 16+
Files.readString(path);  Files.lines(path);
HttpClient.newHttpClient().send(request, BodyHandlers.ofString());
```

---

## 13. Performance

### 13.1 Common Wins

| Issue | Fix |
|---|---|
| String concat in a loop | `StringBuilder` (concat is O(n²)) |
| Boxing in hot loops | Primitive arrays, `IntStream`, Trove/fastutil |
| Unsized collections | `new ArrayList<>(expectedSize)` — avoids repeated growth |
| N+1 database queries | Batch/join fetch |
| Excessive logging | Guard expensive calls; use parameterized logging |
| Wrong collection | `ArrayDeque` over `LinkedList`; `EnumMap` over `HashMap` for enums |
| Default GC for latency-critical work | ZGC or tuned G1 |

```java
// ❌ O(n²) — a new String per iteration
String s = ""; for (String p : parts) s += p;

// ✅ O(n)
var sb = new StringBuilder(); for (String p : parts) sb.append(p);
// or String.join(",", parts)
```

### 13.2 Tooling

```bash
# JFR — production profiling at ~1% overhead
java -XX:StartFlightRecording=duration=60s,filename=rec.jfr -jar app.jar
jfr summary rec.jfr

# async-profiler — best flamegraphs, no safepoint bias
./profiler.sh -d 30 -f flame.html <pid>

# JMH — the only correct way to microbenchmark
@Benchmark @BenchmarkMode(Mode.AverageTime)
public void measure(Blackhole bh) { bh.consume(work()); }
```

### 13.3 Essential JVM Flags

```bash
-Xms4g -Xmx4g                       # equal min/max avoids resize pauses
-XX:+UseG1GC -XX:MaxGCPauseMillis=200
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
-Xlog:gc*:file=gc.log:time,uptime
-XX:+UseContainerSupport            # respect cgroup limits (default on)
-XX:MaxRAMPercentage=75             # in containers, use this instead of -Xmx
```

---

## 14. Interview Section

<details>
<summary><b>Q1. Is Java pass-by-value or pass-by-reference?</b></summary>

Always pass-by-value. For objects, the *value being copied is the reference*. So the method can mutate the object the reference points to (visible to the caller), but reassigning the parameter only changes the method's local copy (invisible to the caller).

The confusion comes from calling this "pass by reference," which would mean the callee could reassign the caller's variable — Java can't do that.
</details>

<details>
<summary><b>Q2. `==` vs `equals()`.</b></summary>

`==` compares the bits in the variable: for primitives that's the value, for references that's identity. `equals()` is a method whose default implementation is `==`, but classes override it for value semantics.

The classic trap is `Integer`: values in -128..127 come from a cache, so `==` accidentally works, then fails above 127. Always `equals` for objects, always `==` for primitives.
</details>

<details>
<summary><b>Q3. Why must you override `hashCode` whenever you override `equals`?</b></summary>

Hash-based collections locate an object by hash first, then confirm with `equals`. If two equal objects hash differently they land in different buckets, so a `HashMap` will happily contain both and `get` will miss.

The contract is one-directional: equal objects must have equal hashes; unequal objects may collide (that's just a performance cost).
</details>

<details>
<summary><b>Q4. How does HashMap work internally?</b></summary>

An array of buckets whose length is always a power of two. On `put`, the key's `hashCode` is spread (`h ^ h>>>16`) to mix high bits into low, then masked with `(n-1)` to get the index. Collisions form a linked list in that bucket; if a bucket reaches 8 entries and the table is at least 64 long, it converts to a red-black tree so worst case is O(log n).

When size exceeds capacity × 0.75, the table doubles and entries are redistributed — and because capacity is a power of two, each entry either stays put or moves exactly `oldCapacity` positions.

It's not thread-safe. Concurrent use loses updates; in Java 7 it could also create a cycle and hang a CPU permanently.
</details>

<details>
<summary><b>Q5. What is type erasure and what does it cost you?</b></summary>

Generic type information is checked at compile time then removed; the bytecode sees raw types with compiler-inserted casts. This was done so generic code could interoperate with pre-Java-5 libraries.

The costs: you can't do `new T[]`, can't use `instanceof List<String>`, can't overload on `List<String>` vs `List<Integer>`, can't have static fields of a type parameter, and can't catch a generic exception type. It also means an unchecked cast can put the wrong type into a collection and the failure surfaces later, somewhere else — "heap pollution."
</details>

<details>
<summary><b>Q6. Explain `volatile`. When is it enough and when is it not?</b></summary>

`volatile` guarantees visibility — a write is immediately visible to subsequent reads on other threads — and prevents reordering across the access. It's enough for a simple flag: one thread writes, others read.

It's not enough for compound actions. `count++` is read-modify-write, which is three operations, so two threads can interleave and lose an update. For that you need `AtomicInteger` (CAS) or a lock.

The other classic use is fixing double-checked locking, where `volatile` prevents publishing a partially constructed object.
</details>

<details>
<summary><b>Q7. What's happens-before?</b></summary>

The JMM's ordering relation. If A happens-before B, then everything A did is visible to B. Without such a relation, the compiler, JIT, and CPU may reorder freely and one thread may never observe another's writes.

The edges you get for free: program order within a thread, unlock→lock on the same monitor, volatile write→read, `Thread.start()`, `Thread.join()`, and final-field publication. It's transitive, which is how you build larger guarantees from these.
</details>

<details>
<summary><b>Q8. `synchronized` vs `ReentrantLock`.</b></summary>

`synchronized` is a language construct: automatic release even on exception, JIT can optimize it (lock elision, coarsening), and it's simpler to get right.

`ReentrantLock` is a library class offering things the keyword can't: `tryLock` with a timeout (deadlock avoidance), interruptible acquisition, optional fairness, and multiple `Condition` wait sets.

Default to `synchronized`; reach for `ReentrantLock` when you need one of those features. On virtual threads, prefer `ReentrantLock` since `synchronized` can pin the virtual thread to its carrier.
</details>

<details>
<summary><b>Q9. How does GC work, and what's a memory leak in a GC'd language?</b></summary>

Generational copying for the young space — collect Eden and survivors frequently, which is cheap because it's proportional to *live* objects, not heap size. Objects surviving enough collections are promoted to the old generation, collected rarely with mark-sweep-compact or, in G1, region-by-region prioritizing the emptiest.

A "leak" in Java is an unintentional strong reference: static collections that only grow, listeners never unregistered, `ThreadLocal`s in a pooled thread, and classloader leaks in app servers. The GC is correct — you're still holding on. Diagnose by taking a heap dump and reading the dominator tree in Eclipse MAT.
</details>

<details>
<summary><b>Q10. What are virtual threads and what changes because of them?</b></summary>

Lightweight threads scheduled by the JVM onto a small pool of carrier OS threads. They cost kilobytes instead of a megabyte, so millions are viable. When one blocks on I/O it unmounts from its carrier, freeing the OS thread.

What changes: the thread-per-request model becomes viable again at high concurrency, so much of the complexity of reactive/async code becomes unnecessary. Thread pools of virtual threads are pointless — create one per task. Watch for pinning inside `synchronized` blocks and for unbounded concurrency now that threads are cheap; you still need semaphores to limit pressure on downstream systems.
</details>

<details>
<summary><b>Q11. `String` immutability — why?</b></summary>

Security (a path or URL can't be mutated after validation), safe sharing across threads with no synchronization, hash caching (so `String` keys in maps are fast), and the string pool's interning, which would be impossible if strings could change.

The cost is that concatenation in a loop is O(n²) — every `+` allocates a new string. `StringBuilder` fixes that.
</details>

<details>
<summary><b>Q12. Abstract class vs interface — modern Java.</b></summary>

Since default methods (Java 8), interfaces can carry behavior, so the distinction narrowed. What remains: a class can implement many interfaces but extend one abstract class; abstract classes can hold mutable state and non-public members; interfaces can only have `public static final` fields.

Design-wise: an interface defines a capability ("can be compared"), an abstract class defines a partial identity in a hierarchy. Prefer interfaces for API contracts, and since Java 17, `sealed` interfaces plus records give you closed hierarchies with exhaustive pattern matching — which is often better than either.
</details>

### 🎤 Output prediction

<details>
<summary><b>P1</b></summary>

```java
Integer a = 127, b = 127, c = 128, d = 128;
System.out.println((a == b) + " " + (c == d));
```
**`true false`** — `Integer.valueOf` caches -128..127; above that, new objects.
</details>

<details>
<summary><b>P2</b></summary>

```java
System.out.println(0.1 + 0.2 == 0.3);
System.out.println(0.1 + 0.2);
```
**`false`**, **`0.30000000000000004`** — IEEE-754. Use `BigDecimal` (with the String constructor) for money.
</details>

<details>
<summary><b>P3</b></summary>

```java
String a = "hello", b = "hello";
String c = new String("hello");
System.out.println((a == b) + " " + (a == c) + " " + (a == c.intern()));
```
**`true false true`** — literals share the pool; `new` forces a distinct object; `intern()` returns the pooled one.
</details>

<details>
<summary><b>P4</b></summary>

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.remove(1);
System.out.println(list);
```
**`[1, 3]`** — `remove(int)` removes by *index*, not value. `remove(Integer.valueOf(1))` removes the value.
</details>

<details>
<summary><b>P5</b></summary>

```java
try { return 1; } finally { return 2; }
```
**`2`** — `finally` overrides the return. (And a compiler warning: never return from `finally`.)
</details>

<details>
<summary><b>P6</b></summary>

```java
int i = 0;
i = i++;
System.out.println(i);
```
**`0`** — `i++` evaluates to the old value, which is then assigned back over the incremented one.
</details>

### 🎤 Implementation drills

1. Thread-safe LRU cache (`LinkedHashMap` with `removeEldestEntry`, then from scratch)
2. A bounded blocking queue using `wait`/`notify`, then using `ReentrantLock`/`Condition`
3. Producer–consumer with graceful shutdown
4. A connection pool with timeout
5. `CompletableFuture` pipeline with timeout, retry, and fallback
6. A custom `Collector`
7. Immutable class with a builder
8. A rate limiter (token bucket) that's thread-safe
9. `deepCopy` via serialization vs manual
10. A generic `Result<T, E>` type

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                          JAVA — ONE PAGE                             ║
╠══════════════════════════════════════════════════════════════════════╣
║ Always PASS BY VALUE (the reference is what's copied)                ║
║ == is identity for objects; equals() is value                        ║
║ Override equals ⟹ MUST override hashCode (and vice versa)            ║
╠══════════════════════════════════════════════════════════════════════╣
║ MEMORY:  Stack(per thread, frames) | Heap(young+old) | Metaspace     ║
║   young: Eden + 2 Survivors, minor GC copies live objects            ║
║   G1 default; ZGC for sub-ms pauses; always set HeapDumpOnOOM        ║
╠══════════════════════════════════════════════════════════════════════╣
║ COLLECTIONS                                                          ║
║   ArrayList O(1) get, O(n) mid-insert  | ArrayDeque > LinkedList     ║
║   HashMap: power-of-2 buckets, treeify at 8, resize at 0.75          ║
║   ConcurrentHashMap for threads; CopyOnWriteArrayList for read-heavy ║
╠══════════════════════════════════════════════════════════════════════╣
║ GENERICS: erased at runtime. PECS —                                  ║
║   ? extends T = producer (read)   ? super T = consumer (write)       ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONCURRENCY                                                          ║
║   volatile = visibility only, NOT atomicity                          ║
║   happens-before: unlock→lock, volatile w→r, start(), join()         ║
║   DCL needs volatile; prefer holder idiom or enum singleton          ║
║   ALWAYS bounded queues in thread pools + rejection policy           ║
║   Java 21: virtual threads → thread-per-task, avoid synchronized     ║
╠══════════════════════════════════════════════════════════════════════╣
║ MODERN: record · sealed · pattern-match switch · record patterns     ║
║         text blocks · var · Optional (return type only) · streams    ║
╠══════════════════════════════════════════════════════════════════════╣
║ TOOLS: jcmd Thread.print · JFR · async-profiler · Eclipse MAT · JMH  ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Spring Boot](../03-backend/spring-boot.md) · [DSA Patterns](../04-dsa/00-patterns.md) · [System Design](../05-system-design/00-fundamentals.md)
