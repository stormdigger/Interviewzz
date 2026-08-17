# ⚡ Caching

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton
>
> This book is mostly about the first one.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The Cache Hierarchy](#2-the-cache-hierarchy)
3. [Caching Patterns](#3-caching-patterns)
4. [Eviction Policies](#4-eviction-policies)
5. [Invalidation Strategies](#5-invalidation)
6. [The Three Catastrophic Failures](#6-catastrophic-failures)
7. [Redis Deep Dive](#7-redis)
8. [Distributed Locking](#8-distributed-locking)
9. [Consistency Models](#9-consistency)
10. [HTTP and CDN Caching](#10-http-cdn)
11. [Sizing and Metrics](#11-sizing-metrics)
12. [Production Checklist](#12-production-checklist)
13. [Interview Section](#13-interview-section)
14. [Cheat Sheet](#14-cheat-sheet)

---

## 1. Mental Model

🧠 **A cache trades correctness for speed.** Every cache introduces a window where your data is wrong. The entire discipline is choosing how wide that window is and what happens at its edges.

```
   Three questions before adding any cache:

   ┌────────────────────────────────────────────────────────┐
   │ 1. What is the read:write ratio?                       │
   │    < 10:1  → a cache may cost more than it saves       │
   │    > 100:1 → caching is almost certainly worth it      │
   ├────────────────────────────────────────────────────────┤
   │ 2. How stale can this data be?                         │
   │    Answer in SECONDS. "It must be fresh" is not an     │
   │    answer — it's an unexamined requirement.            │
   ├────────────────────────────────────────────────────────┤
   │ 3. What happens when the cache is empty or wrong?      │
   │    If the answer is "the database dies," you have      │
   │    built a fragile system, not a fast one.             │
   └────────────────────────────────────────────────────────┘
```

### The math that justifies caching

```
   Without cache:  every request → DB (10 ms)
   With cache:     hit → Redis (0.5 ms) | miss → Redis + DB (10.5 ms)

   Average latency = (hit_rate × 0.5) + ((1 - hit_rate) × 10.5)

   hit rate  95%  →  1.0 ms   (10× improvement)
   hit rate  90%  →  1.5 ms
   hit rate  80%  →  2.5 ms
   hit rate  50%  →  5.5 ms
   hit rate  20%  →  8.5 ms   ← barely worth the complexity

   ⭐ Cache value is HIGHLY nonlinear in hit rate.
      Going 90% → 99% halves your DB load again.
```

---

## 2. The Cache Hierarchy

```
   ┌──────────────────────────────────────────────────────────────┐
   │ BROWSER CACHE            ~0 ms        per user               │
   │   Cache-Control, ETag, service worker                        │
   ├──────────────────────────────────────────────────────────────┤
   │ CDN / EDGE               ~10-30 ms    global, shared         │
   │   Cloudflare, CloudFront, Fastly                             │
   ├──────────────────────────────────────────────────────────────┤
   │ REVERSE PROXY            ~1-5 ms      per datacenter         │
   │   nginx, Varnish                                             │
   ├──────────────────────────────────────────────────────────────┤
   │ APPLICATION / IN-PROCESS ~0.001 ms    per instance ⚠️ not     │
   │   Caffeine, LRU map, @lru_cache          shared → coherence  │
   ├──────────────────────────────────────────────────────────────┤
   │ DISTRIBUTED CACHE        ~0.5-2 ms    shared across app tier │
   │   Redis, Memcached                                           │
   ├──────────────────────────────────────────────────────────────┤
   │ DATABASE BUFFER POOL     ~0.1 ms      inside the DB          │
   │   shared_buffers, InnoDB buffer pool                         │
   ├──────────────────────────────────────────────────────────────┤
   │ DISK                     ~100 µs - 10 ms                     │
   └──────────────────────────────────────────────────────────────┘

   Rule: cache as close to the user as the freshness requirement allows.
```

### Multi-level (L1 + L2)

```java
// L1: in-process, microseconds, but NOT coherent across instances
// L2: Redis, shared, sub-millisecond

public Product get(String id) {
    Product p = l1.getIfPresent(id);              // Caffeine, ~50ns
    if (p != null) return p;

    p = redis.get("product:" + id);               // ~500µs
    if (p != null) { l1.put(id, p); return p; }

    p = db.findById(id);                          // ~10ms
    redis.setex("product:" + id, 300, p);
    l1.put(id, p);
    return p;
}
```

⚠️ **The L1 coherence problem:** with 20 app instances, an invalidation must reach all 20 L1 caches. Options:

```
   1. Short L1 TTL (5-30 s) — accept bounded staleness. Simplest, usually right.
   2. Pub/sub invalidation — publish the key to a Redis channel; every
      instance subscribes and evicts. Adds a race window but works well.
   3. Redis client-side caching (RESP3 tracking) — Redis pushes invalidation
      messages to clients that cached a key. Elegant, less widely deployed.
   4. Don't use L1 for anything that must be fresh.
```

---

## 3. Caching Patterns

### 3.1 Cache-Aside (Lazy Loading) ⭐ the default

```
   READ                              WRITE
   ────                              ─────
   1. Check cache                    1. Write to DB
   2. HIT  → return                  2. INVALIDATE cache key
   3. MISS → read DB                    (delete, don't update)
   4. Populate cache
   5. Return
```

```python
async def get_user(user_id: str) -> User:
    key = f"user:{user_id}"
    if cached := await redis.get(key):
        return User.model_validate_json(cached)

    user = await db.fetch_user(user_id)
    if user is None:
        await redis.setex(key, 60, NULL_SENTINEL)     # cache the miss (see §6.2)
        return None

    await redis.setex(key, 300, user.model_dump_json())
    return user

async def update_user(user_id: str, data: dict) -> User:
    user = await db.update_user(user_id, data)
    await redis.delete(f"user:{user_id}")             # ⭐ DELETE, not SET
    return user
```

⚠️ **Delete on write, don't update.** Updating the cache creates a race:

```
   T1: read DB → value A
   T2: write DB → value B, SET cache = B
   T1: SET cache = A            ← stale value now cached indefinitely

   Deleting is safe: the next reader repopulates from the current DB state.
```

| Pro | Con |
|---|---|
| Only requested data is cached | Every miss pays full latency |
| Cache failure ≠ system failure | Cold start hits the DB hard |
| Simple, works with any store | Brief staleness window on write |

### 3.2 Read-Through

The cache library owns the DB fetch, so application code just calls `get`.

```java
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .refreshAfterWrite(Duration.ofMinutes(1))    // ⭐ async refresh, serve stale
    .recordStats()
    .build(id -> db.findProduct(id));            // the loader

Product p = cache.get(id);                       // populates automatically
```

`refreshAfterWrite` is underrated: after 1 minute the next request triggers a background reload while immediately returning the slightly stale value. Users never wait for a refresh.

### 3.3 Write-Through

```
   Write → cache → DB (synchronously) → ack
```

Cache is never stale, but every write pays both latencies, and you cache data that may never be read.

### 3.4 Write-Behind (Write-Back)

```
   Write → cache → ack immediately
                 → async batch flush to DB
```

Fastest writes and it coalesces repeated updates to the same key. **But you will lose data if the cache dies before flushing.** Only acceptable for genuinely low-value, high-volume data — view counters, activity heartbeats, analytics events.

### 3.5 Refresh-Ahead

Proactively refresh entries before they expire, based on access patterns. Eliminates miss latency for hot keys at the cost of refreshing things nobody asked for.

### Comparison

| Pattern | Read latency | Write latency | Consistency | Data loss risk |
|---|---|---|---|---|
| Cache-aside | Miss = slow | Fast | Eventual | None |
| Read-through | Miss = slow | — | Eventual | None |
| Write-through | Always fast | Slow | Strong-ish | None |
| Write-behind | Always fast | Fastest | Weak | **Yes** |
| Refresh-ahead | Always fast | — | Eventual | None |

---

## 4. Eviction Policies

```
   LRU   Least Recently Used
         ✅ good general default, matches temporal locality
         ❌ one full scan flushes the whole cache

   LFU   Least Frequently Used
         ✅ resists scan pollution, keeps genuinely hot keys
         ❌ new items are disadvantaged; needs aging/decay

   FIFO  First In First Out
         ✅ trivial   ❌ ignores access patterns

   TTL   Time based
         ✅ predictable staleness bound   ❌ doesn't manage memory

   Random
         ✅ O(1), surprisingly competitive, no metadata overhead

   ARC / W-TinyLFU
         ✅ adaptive — balances recency and frequency
         Used by Caffeine (W-TinyLFU); typically 5-15% better hit rate than LRU
```

### Redis `maxmemory-policy`

```
   noeviction        → writes return an error when full  ⚠️ default!
   allkeys-lru       → evict any key, LRU               ⭐ pure cache
   allkeys-lfu       → evict any key, LFU               ⭐ skewed access
   allkeys-random
   volatile-lru      → only keys WITH a TTL             ⭐ mixed cache + data
   volatile-lfu
   volatile-ttl      → shortest remaining TTL first
   volatile-random
```

⚠️ The default `noeviction` means a full Redis starts **rejecting writes** rather than evicting. If you're using Redis purely as a cache, set `allkeys-lru`. If Redis holds both cache and durable data (sessions, queues), use `volatile-*` and make sure cache keys have TTLs while durable keys don't.

### O(1) LRU implementation

```python
class LRUCache:
    """Doubly linked list (order) + hash map (lookup) = O(1) get and put."""
    class Node:
        __slots__ = ('key', 'val', 'prev', 'next')
        def __init__(self, key=None, val=None):
            self.key, self.val = key, val
            self.prev = self.next = None

    def __init__(self, capacity: int):
        self.cap = capacity
        self.map: dict = {}
        self.head = self.Node()          # sentinel: most recent side
        self.tail = self.Node()          # sentinel: least recent side
        self.head.next, self.tail.prev = self.tail, self.head

    def _remove(self, node):
        node.prev.next, node.next.prev = node.next, node.prev

    def _add_front(self, node):
        node.next, node.prev = self.head.next, self.head
        self.head.next.prev = self.head.next = node

    def get(self, key):
        node = self.map.get(key)
        if node is None: return -1
        self._remove(node); self._add_front(node)      # mark as recently used
        return node.val

    def put(self, key, val):
        if (node := self.map.get(key)) is not None:
            node.val = val
            self._remove(node); self._add_front(node)
            return
        if len(self.map) >= self.cap:
            lru = self.tail.prev                        # evict from the tail
            self._remove(lru); del self.map[lru.key]
        node = self.Node(key, val)
        self.map[key] = node
        self._add_front(node)
```

---

## 5. Invalidation

### 5.1 Strategies

```
   TTL (time to live)              simplest, bounded staleness
     ✅ self-healing, no coordination
     ❌ stale until expiry; synchronized expiry causes stampedes

   EXPLICIT INVALIDATION           delete the key on write
     ✅ fresh immediately
     ❌ must find EVERY key affected — easy to miss one

   VERSIONED KEYS                  user:123:v7
     ✅ atomic "invalidation" by bumping the version; old keys expire naturally
     ❌ wastes memory until the old entries age out

   TAG-BASED                       invalidate everything tagged "product:5"
     ✅ handles fan-out (a product appears in 20 cached lists)
     ❌ needs bookkeeping (a Redis SET of keys per tag)

   EVENT-DRIVEN (CDC)              database changelog → invalidation events
     ✅ catches writes from ANY source, including manual SQL
     ❌ infrastructure complexity (Debezium, Kafka)
```

### 5.2 Versioned keys

```python
# Instead of tracking every derived key, bump a version
async def invalidate_user(user_id: str):
    await redis.incr(f"user:{user_id}:version")

async def get_user_profile(user_id: str):
    v = await redis.get(f"user:{user_id}:version") or 0
    key = f"user:{user_id}:profile:v{v}"
    ...
```

One `INCR` invalidates every derived cache entry for that user simultaneously. Old keys become unreachable and expire on their own.

### 5.3 Tag-based

```python
async def cache_with_tags(key: str, value: str, tags: list[str], ttl: int):
    pipe = redis.pipeline()
    pipe.setex(key, ttl, value)
    for tag in tags:
        pipe.sadd(f"tag:{tag}", key)
        pipe.expire(f"tag:{tag}", ttl + 60)
    await pipe.execute()

async def invalidate_tag(tag: str):
    keys = await redis.smembers(f"tag:{tag}")
    if keys:
        await redis.delete(*keys, f"tag:{tag}")
```

Use when one entity appears in many cached views — a product in category lists, search results, recommendations, and the homepage.

### 5.4 The write-invalidate race

```
   Even "delete on write" has a window:

   T1 (read):  cache MISS, reads DB → value A
   T2 (write): writes B to DB, DELETEs cache key
   T1:         SETs cache = A                      ← stale A cached for the full TTL

   Mitigations:
   • Short TTL bounds the damage (usually sufficient)
   • Delayed double delete: delete, write, sleep ~500ms, delete again
   • Set with a version check (Lua CAS)
   • Accept it — this race requires precise timing and is rare
```

🏭 In practice, a short TTL plus delete-on-write is correct enough for the overwhelming majority of systems. Don't build a distributed consensus protocol to close a 50-millisecond window on cached product descriptions.

---

## 6. Catastrophic Failures

These three are the classic interview questions and the classic outages.

### 6.1 Cache Stampede (Thundering Herd / Dogpile)

```
   A hot key expires. 1,000 concurrent requests all miss simultaneously.
   All 1,000 hit the database with the same expensive query.

   Cache:  ████████████│ EXPIRED
                       │
   Requests:           ├─▶ DB ┐
                       ├─▶ DB ├─ 1000 identical queries
                       ├─▶ DB ├─ database falls over
                       └─▶ DB ┘
```

**Fix 1 — Probabilistic early expiration (XFetch).** Elegant and lock-free:

```python
import random, math, time

async def get_with_early_expiry(key: str, ttl: int, beta: float = 1.0):
    packed = await redis.get(key)
    if packed:
        value, delta, expiry = unpack(packed)     # delta = last recompute duration
        # As expiry approaches, probability of early recompute rises
        if time.time() - delta * beta * math.log(random.random()) < expiry:
            return value                          # still fresh enough

    value, delta = await timed(recompute)         # ONE request usually wins the race
    await redis.setex(key, ttl, pack(value, delta, time.time() + ttl))
    return value
```

Each request independently decides to refresh early, with probability rising as expiry nears. Typically exactly one request recomputes, with no locking.

**Fix 2 — Distributed lock + stale serving:**

```python
async def get_with_lock(key: str, ttl: int):
    if v := await redis.get(key): return v

    lock = f"lock:{key}"
    if await redis.set(lock, "1", nx=True, ex=10):    # I won — I recompute
        try:
            value = await expensive_query()
            await redis.setex(key, ttl, value)
            await redis.setex(f"stale:{key}", ttl * 10, value)   # long-lived backup
            return value
        finally:
            await redis.delete(lock)
    else:                                             # someone else is computing
        if stale := await redis.get(f"stale:{key}"):
            return stale                              # ⭐ serve stale, don't queue
        await asyncio.sleep(0.05)
        return await get_with_lock(key, ttl)
```

**Fix 3 — Jittered TTL.** Prevents *synchronized* expiry across many keys:

```python
ttl = base_ttl + random.randint(0, base_ttl // 4)     # ±25% jitter
```

Without jitter, 10,000 keys warmed at deploy time all expire in the same second.

**Fix 4 — Request coalescing in-process** (Go's `singleflight`, or a per-process map of in-flight futures) collapses concurrent identical requests within one instance.

### 6.2 Cache Penetration

```
   Requests for keys that DON'T EXIST bypass the cache entirely
   and hit the DB every time. An attacker requests
   /users/999999999, /users/999999998, ... — free DB load amplification.
```

**Fix 1 — Cache the negative result:**

```python
if user is None:
    await redis.setex(key, 60, NULL_SENTINEL)    # short TTL for negatives
    return None
```

Short TTL on negatives so a genuinely-created entity appears quickly.

**Fix 2 — Bloom filter** — probabilistic set membership in a tiny amount of memory:

```python
# "definitely not present" or "maybe present" — never a false negative
if not bloom.might_contain(user_id):
    return None                                   # skip cache AND database
```

```
   Bloom filter: m bits, k hash functions

   add("abc"):  set bits at h1("abc"), h2("abc"), h3("abc")
   check("xyz"): ALL k bits set? → maybe present : definitely absent

   1% false positive rate ≈ 9.6 bits per element
   → 100 million IDs in ~120 MB, vs gigabytes for a real set
```

**Fix 3 — Validate before querying.** If IDs are ULIDs, reject anything that isn't a valid ULID before touching any storage.

### 6.3 Cache Avalanche

```
   MANY keys expire at once, OR the cache layer goes down entirely.
   Full traffic hits the database, which cannot take it.

   Common triggers:
     • Deploy warms 100k keys with identical TTL → all expire together
     • Redis restarts / fails over
     • A `FLUSHALL` (accidental or during a bad deploy)
```

**Defenses:**

```
   1. TTL jitter                     (prevents synchronized expiry)
   2. Multi-level cache              (L1 absorbs the L2 outage)
   3. Circuit breaker on the DB      (shed load rather than collapse)
   4. Rate limit DB queries per key  (semaphore)
   5. Serve stale on any error       ⭐ the single most valuable one
   6. Redis HA — replicas + Sentinel/Cluster, persistence enabled
   7. Warm the cache before shifting traffic to a new deployment
```

```python
STALE_TTL = 3600     # keep a copy far beyond the fresh TTL

async def resilient_get(key: str):
    try:
        if v := await redis.get(key): return v
    except RedisError:
        logger.warning("cache down, degrading to DB")

    try:
        v = await db.fetch(key)
        await safe_set(key, v)
        return v
    except DatabaseError:
        if stale := await safe_get(f"stale:{key}"):
            metrics.incr("cache.served_stale")
            return stale                          # ⭐ stale beats an error page
        raise
```

🏭 **"Serve stale on error" is the highest-leverage resilience pattern in caching.** A 10-minute-old product price is vastly better than a 500 error.

---

## 7. Redis

### 7.1 Data structures and what they're for

```redis
# STRING — counters, JSON blobs, flags
SET user:1 '{"name":"Ada"}' EX 300 NX
INCR page:views
SETRANGE / GETRANGE / APPEND

# HASH — objects with independently updatable fields
HSET user:1 name Ada email a@b.com
HINCRBY user:1 login_count 1
HGETALL user:1              # ⚠️ O(n) — avoid on large hashes

# LIST — queues, stacks, recent-items
LPUSH queue:jobs '{"id":1}'
BRPOP queue:jobs 5          # blocking pop — worker pattern
LTRIM recent:user:1 0 99    # keep only the newest 100

# SET — uniqueness, tags, relationships
SADD tags:post:1 redis cache
SINTER tags:post:1 tags:post:2
SPOP / SRANDMEMBER

# SORTED SET — leaderboards, priority queues, time-series, rate limits ⭐
ZADD leaderboard 1500 player:1
ZREVRANGE leaderboard 0 9 WITHSCORES       # top 10
ZRANGEBYSCORE events 1723640000 1723643600 # time window
ZREMRANGEBYSCORE ratelimit:u1 0 <cutoff>   # sliding window rate limiter

# BITMAP — 1 bit per user: daily actives in ~12 MB for 100M users
SETBIT active:2026-08-14 12345 1
BITCOUNT active:2026-08-14
BITOP AND result active:day1 active:day2   # retention analysis

# HYPERLOGLOG — cardinality in 12 KB, ~0.81% error
PFADD visitors:today user1 user2
PFCOUNT visitors:today
PFMERGE visitors:week visitors:mon visitors:tue

# STREAM — append-only log with consumer groups (Kafka-lite)
XADD events * type click user 1
XREADGROUP GROUP workers w1 COUNT 10 BLOCK 5000 STREAMS events >
XACK events workers <id>

# GEO — proximity search (built on sorted sets)
GEOADD drivers -122.4 37.7 driver:1
GEOSEARCH drivers FROMLONLAT -122.4 37.7 BYRADIUS 5 km ASC
```

### 7.2 Persistence

| | RDB (snapshot) | AOF (append-only file) |
|---|---|---|
| What | Point-in-time binary dump | Every write command logged |
| Restart speed | Fast | Slower (replays the log) |
| Data loss | Up to the snapshot interval | ≤1 s with `everysec` |
| File size | Compact | Larger (rewritten periodically) |
| Fork cost | Copy-on-write spike ⚠️ | Lower |

🏭 Enable **both** for durable data: RDB for fast restarts and backups, AOF with `appendfsync everysec` to bound loss to one second. For a pure cache, disable both — you don't need to persist data you can recompute, and it removes the fork pause.

### 7.3 Deployment topologies

```
   STANDALONE       single node
                    ❌ SPOF, limited to one machine's memory

   REPLICATION      primary + replicas, async
                    ✅ read scaling, backup  ❌ manual failover

   SENTINEL         replication + automatic failover
                    ✅ HA  ❌ still one shard's worth of memory
                    ⚠️ clients must be Sentinel-aware

   CLUSTER          16384 hash slots across N primaries, each with replicas
                    ✅ horizontal scale + HA
                    ❌ multi-key ops must share a hash slot
```

```redis
# Cluster: force related keys onto the same slot with hash tags
MSET {user:1}:profile "..." {user:1}:settings "..."
#     ^^^^^^^^ only the braces are hashed → same slot → multi-key ops work
```

### 7.4 The commands that cause outages

```
   ❌ KEYS *              O(n), BLOCKS the single-threaded server
      ✅ SCAN 0 MATCH prefix:* COUNT 100    (cursor-based, non-blocking)

   ❌ FLUSHALL / FLUSHDB  instant total data loss
      ✅ rename-command FLUSHALL "" in redis.conf

   ❌ Huge values / big collections    one 500 MB key stalls everything
   ❌ HGETALL on a 1M-field hash        O(n) on the main thread
   ❌ Unbounded LPUSH without LTRIM     memory exhaustion
   ❌ SAVE (synchronous)                blocks until the dump completes
```

⚠️ **Redis command execution is single-threaded.** One O(n) command on a large collection blocks *every other client*. This is the single most common cause of mysterious Redis latency spikes.

```redis
# Find them
SLOWLOG GET 10
CONFIG SET slowlog-log-slower-than 10000    # log anything over 10ms
LATENCY DOCTOR
MEMORY DOCTOR
--bigkeys                                    # redis-cli scan for large keys
```

### 7.5 Redis vs Memcached

| | Redis | Memcached |
|---|---|---|
| Data structures | Rich (10+ types) | Strings only |
| Persistence | RDB + AOF | None |
| Replication | Built-in | None |
| Threading | Single-threaded core (I/O threads available) | Multi-threaded |
| Memory efficiency | Higher overhead | Slightly better for plain KV |
| Clustering | Built-in | Client-side sharding |
| Pub/sub, streams, Lua | ✅ | ❌ |

Memcached is still marginally faster for the narrow case of pure string caching with many cores. Redis wins essentially everywhere else, and its data structures often let you replace application logic entirely.

---

## 8. Distributed Locking

```python
# ── Simple lock: correct only if you accept its limitations ──
async def acquire(key: str, token: str, ttl_ms: int) -> bool:
    return await redis.set(f"lock:{key}", token, nx=True, px=ttl_ms)

# Release MUST be atomic and check ownership, or you release someone else's lock
RELEASE = """
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
else
  return 0
end
"""
await redis.eval(RELEASE, 1, f"lock:{key}", token)
```

⚠️ **The correctness problem no lock library solves:**

```
   1. Client A acquires the lock (TTL 10 s)
   2. Client A stalls — GC pause, network hiccup, VM migration
   3. Lock expires; Client B acquires it
   4. Client A wakes up, believes it still holds the lock
   5. BOTH mutate the resource

   This is unavoidable in an asynchronous system. Redlock does not fix it.
```

**The only real fix is a fencing token:**

```
   Lock service returns a monotonically increasing token with each grant.
   The protected RESOURCE rejects any write with a token lower than
   the highest it has seen.

   A gets token 33 → stalls
   B gets token 34 → writes with 34 → storage records 34
   A wakes, writes with 33 → REJECTED (33 < 34) ✅
```

🏭 **Practical guidance:** if the lock is an *optimization* (avoid duplicate work, reduce load), a simple Redis lock is fine. If the lock is required for *correctness* (financial transactions, inventory), use the database's own transactional guarantees — `SELECT ... FOR UPDATE`, a unique constraint, or an atomic conditional update — not a distributed lock.

---

## 9. Consistency

```
   STRONG        every read sees the latest write
                 → don't cache, or write-through with synchronous invalidation
                 → costs latency and availability

   READ-YOUR-WRITES   a user always sees their own writes
                      → after a write, route that user's reads to the source,
                        or invalidate their cache synchronously

   MONOTONIC READS    you never see time go backwards
                      → sticky sessions to one cache node/replica

   EVENTUAL      converges within the TTL window
                 → the default for cached data; document the window
```

### Deciding what to cache

```
   ┌──────────────────────┬───────────┬─────────────────────────────┐
   │ Data                 │    TTL    │ Notes                       │
   ├──────────────────────┼───────────┼─────────────────────────────┤
   │ Account balance      │  DON'T    │ Correctness-critical        │
   │ Inventory count      │  DON'T    │ Or cache + reserve atomically│
   │ Auth permissions     │  1-5 min  │ ⚠️ revocation must be fast   │
   │ User profile         │  5-15 min │ Invalidate on write         │
   │ Product catalog      │  1-6 hr   │ Invalidate on update        │
   │ Search results       │  1-5 min  │ Include filters in the key  │
   │ Homepage/feed        │  30-60 s  │ High traffic, tolerant      │
   │ Config / feature flag│  30-60 s  │ Needs fast propagation      │
   │ Static assets        │  1 year   │ Content-hashed filenames    │
   │ Aggregates/reports   │  1-24 hr  │ Expensive to compute        │
   │ Session data         │  session  │ Not really a cache          │
   └──────────────────────┴───────────┴─────────────────────────────┘
```

⚠️ **Never cache a permission check longer than your incident response tolerance.** If you fire someone, a 1-hour permission cache means an hour of continued access.

---

## 10. HTTP and CDN

```http
Cache-Control: public, max-age=31536000, immutable      # hashed static assets
Cache-Control: private, max-age=0, must-revalidate      # user-specific HTML
Cache-Control: public, s-maxage=600, max-age=60         # CDN 10min, browser 1min
Cache-Control: public, max-age=60, stale-while-revalidate=600
Cache-Control: no-store                                 # never cache (sensitive)
Vary: Accept-Encoding, Accept-Language                  # ⚠️ each value = separate entry
ETag: "a1b2c3"
```

⚠️ **`Vary: Cookie` destroys cache efficiency** — every distinct cookie value creates a separate cache entry, so your hit rate approaches zero. Never vary on Cookie at the CDN.

### `stale-while-revalidate` — the best HTTP caching directive

```
   max-age=60, stale-while-revalidate=600

   t=0-60s     serve fresh from cache
   t=60-660s   serve STALE immediately + revalidate in the background
   t>660s      block and fetch fresh

   → users never wait for revalidation, and the origin gets
     exactly one request per key per window
```

### CDN cache key design

```
   Default key: host + path + query string
   
   ⚠️ Any unique query param creates a new entry:
      /page?utm_source=twitter  ≠  /page
      → strip tracking params at the edge, or set an explicit
        cache key that ignores them

   ⭐ Purge strategies:
      • Path purge:     /products/123
      • Wildcard purge: /products/*   (often slow/rate-limited)
      • Tag purge:      surrogate-key header → purge by tag ⭐ best
      • Versioned URLs: never purge; change the URL
```

```http
# Fastly/Cloudflare surrogate keys — purge many URLs by tag
Surrogate-Key: product-123 category-shoes
Surrogate-Control: max-age=3600
```

---

## 11. Sizing and Metrics

### Working set

```
   How much memory do I need?

   1. Identify the working set: the data actually accessed
      in a typical window (not the total dataset)
   2. Measure or estimate the average serialized entry size
   3. Add per-key overhead (~50-100 bytes in Redis)
   4. Multiply by 1.5-2× headroom (fragmentation, replication buffers)

   Example: 1M active users × 2 KB profile × 1.5 = ~3 GB

   Access is usually Zipfian: the top 20% of keys serve ~80% of requests.
   → Caching 20% of your data often yields an 80% hit rate.
     This is why small caches work surprisingly well.
```

### Metrics that matter

| Metric | Target | Meaning |
|---|---|---|
| Hit rate | > 90% | Below 80%, question the cache's value |
| p99 latency | < 5 ms | Spikes = big keys or blocking commands |
| Eviction rate | ~0 | Sustained evictions = undersized |
| Memory fragmentation ratio | 1.0-1.5 | > 1.5 = fragmentation; > 1 with swap = disaster |
| Connected clients | Stable | Growth = connection leak |
| Blocked clients | ~0 | Blocking ops piling up |
| Keyspace misses/sec | — | Spike = stampede or penetration |

```redis
INFO stats            # keyspace_hits, keyspace_misses, evicted_keys
INFO memory           # used_memory, mem_fragmentation_ratio
INFO replication      # master_repl_offset vs slave offset = lag
```

```
   hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses)
```

---

## 12. Production Checklist

```
   DESIGN
   □ Read:write ratio justifies the cache
   □ Acceptable staleness documented IN SECONDS
   □ Cache key naming convention (app:entity:id:version)
   □ TTL chosen per data type, with jitter
   □ Behavior when the cache is unavailable is defined and tested

   STAMPEDE / PENETRATION / AVALANCHE
   □ TTL jitter on every set
   □ Stampede protection on expensive keys (early expiry or lock)
   □ Negative caching or a bloom filter for nonexistent keys
   □ Serve-stale-on-error path implemented
   □ Cache warming before shifting traffic to a new deployment

   OPERATIONS
   □ maxmemory + eviction policy set (NOT the noeviction default)
   □ KEYS / FLUSHALL renamed or disabled
   □ Persistence configured to match the role (cache vs store)
   □ HA: replicas + Sentinel or Cluster
   □ Slowlog enabled and monitored
   □ Big-key scan in CI or on a schedule
   □ Connection pooling with sane limits and timeouts
   □ Client-side timeouts SHORTER than the DB timeout ⭐

   SECURITY
   □ AUTH / ACLs enabled, not on a public interface
   □ TLS in transit
   □ No sensitive data cached without encryption
   □ Cache keys include the tenant ID in multi-tenant systems ⭐
```

⚠️ **The multi-tenant cache key bug** is one of the worst you can ship: cache `product:5` without the tenant ID and tenant B sees tenant A's data. Always namespace by tenant.

---

## 13. Interview Section

<details>
<summary><b>Q1. Explain cache-aside vs write-through vs write-behind.</b></summary>

Cache-aside puts the application in control: check the cache, on a miss read the database and populate. On writes, update the database and delete the cache key. It's the default because a cache failure degrades to slow rather than broken, and you only cache what's actually requested.

Write-through writes to cache and database synchronously, so the cache is never stale — but every write pays both latencies and you cache data that may never be read.

Write-behind writes to the cache and acknowledges immediately, flushing to the database asynchronously in batches. It gives the fastest writes and coalesces repeated updates, but you will lose data if the cache dies before flushing. That's acceptable for view counters, not for orders.

The detail worth mentioning in cache-aside: on write you *delete* rather than update, because updating creates a race where a slow reader can write back a stale value it read before your update.
</details>

<details>
<summary><b>Q2. What is a cache stampede and how do you prevent it?</b></summary>

A hot key expires and every concurrent request misses simultaneously, so they all execute the same expensive query against the database at once. A key served a thousand requests per second becomes a thousand identical database queries in the same instant.

Four mitigations, roughly in order of elegance. Probabilistic early expiration — each request independently decides to recompute with a probability that rises as expiry approaches, so typically exactly one refreshes and no locking is needed. A distributed lock where one request recomputes while others are served a long-lived stale copy instead of queueing. TTL jitter, which prevents many keys warmed at the same time from expiring together. And in-process request coalescing, collapsing concurrent identical requests within one instance.

I'd combine jitter everywhere as a baseline with early expiration or lock-plus-stale on specifically expensive keys.
</details>

<details>
<summary><b>Q3. Cache penetration — what and how do you defend?</b></summary>

Requests for keys that don't exist. The cache never has them, so every request falls through to the database. It's naturally occurring with deleted content, but it's also a cheap attack — request sequential nonexistent IDs and you amplify load with no cache benefit at all.

Three defenses. Cache the negative result with a short TTL, so a repeated miss is served from cache — short so a genuinely-created entity appears promptly. A bloom filter in front, which answers "definitely absent" or "maybe present" using about 10 bits per element, so 100 million IDs fit in roughly 120 MB and you skip both cache and database for known-absent keys. And input validation — if IDs are ULIDs, reject malformed ones before touching storage.

Negative caching alone handles most real cases; bloom filters matter when the key space is enormous.
</details>

<details>
<summary><b>Q4. How do you handle cache invalidation across many derived keys?</b></summary>

Three approaches depending on the shape of the problem.

Versioned keys: keep a version counter per entity and include it in every derived key. One `INCR` atomically invalidates every derived entry, and the orphaned keys expire naturally. Simple and race-free, at the cost of some wasted memory.

Tag-based: maintain a Redis set of keys per tag, and on invalidation delete every key in the set. This handles fan-out where one product appears in twenty cached lists. It needs bookkeeping but is precise.

Event-driven via CDC: stream the database changelog through something like Debezium and emit invalidation events. The advantage is it catches writes from *any* source, including manual SQL and batch jobs that bypass your application code. The cost is real infrastructure.

For most systems I'd use short TTLs plus explicit deletes on the obvious paths, and only add versioning or tags where the fan-out is genuinely painful.
</details>

<details>
<summary><b>Q5. Why is Redis fast, and what makes it slow?</b></summary>

Fast because everything is in memory, the data structures are chosen for O(1) or O(log n) operations, and the core command loop is single-threaded — which sounds like a limitation but eliminates lock contention and context switching entirely. Plus an efficient event loop over non-blocking I/O, and a compact protocol.

Slow for exactly one reason usually: that same single thread. Any O(n) command on a large collection blocks every other client. `KEYS *` on a million keys, `HGETALL` on a huge hash, `SMEMBERS` on a large set, or deleting a multi-gigabyte key will produce latency spikes that look mysterious until you check the slowlog.

Other causes: memory pressure triggering eviction or swap, the copy-on-write fork for RDB snapshots causing a pause on write-heavy workloads, and network round trips when you issue a thousand sequential commands instead of pipelining them.

The diagnostic path is SLOWLOG, LATENCY DOCTOR, and `redis-cli --bigkeys`.
</details>

<details>
<summary><b>Q6. Is a Redis distributed lock safe?</b></summary>

Safe enough as an optimization, not safe enough for correctness.

The mechanics are straightforward: `SET key token NX PX ttl` to acquire, and a Lua script that checks ownership before deleting to release — you must check ownership, or you'll release a lock that timed out and was acquired by someone else.

The unfixable problem is that a client can stall — a GC pause, network partition, VM migration — long enough for its lock to expire, then wake up believing it still holds it. Now two clients think they own it. Redlock doesn't solve this; the argument between Martin Kleppmann and Salvatore Sanfilippo about it is worth reading.

The only real fix is fencing tokens: the lock service issues a monotonically increasing number, and the protected resource rejects writes carrying a token lower than the highest it's seen. That requires the storage layer to participate.

So in practice: if the lock prevents duplicate work, Redis is fine. If the lock protects money or inventory, use the database's own transactional guarantees — `SELECT FOR UPDATE`, a unique constraint, or a conditional update.
</details>

<details>
<summary><b>Q7. What should you never cache?</b></summary>

Anything where a stale read causes an incorrect action rather than a slightly outdated display.

Account balances and inventory counts — a stale read means overdrafting or overselling. Handle those with the database's transactional guarantees, or cache the value but perform the actual decrement atomically at the source.

Permissions and auth state need care. You can cache them, but only for as long as you can tolerate a revoked user retaining access. If you fire someone, a one-hour permission cache means an hour of continued access. I'd cache for a minute or two and invalidate explicitly on revocation.

Anything sensitive that would sit unencrypted in a cache that's often less hardened than the primary database.

And in multi-tenant systems, any key that doesn't include the tenant ID — that's not "shouldn't cache" so much as a data leak waiting to happen.
</details>

<details>
<summary><b>Q8. How would you size a cache?</b></summary>

Start from the working set — the data actually accessed in a typical window — not the total dataset. Access patterns are usually Zipfian, so the top 20% of keys serve around 80% of requests. That's why relatively small caches achieve high hit rates.

Then: average serialized entry size, plus per-key overhead of roughly 50-100 bytes in Redis, times the number of entries you want resident, times 1.5-2× for fragmentation and replication buffers.

But the empirical approach beats the estimate: deploy with generous memory, measure the hit rate and eviction rate, then tune down. If evictions are near zero and hit rate is high, you have headroom. Sustained evictions with a falling hit rate means you're undersized.

The number I'd actually watch is the marginal return — going from 90% to 95% hit rate halves database load again, so it's often worth more memory than a naive cost calculation suggests.
</details>

<details>
<summary><b>Q9. Your cache goes down at peak traffic. What happens?</b></summary>

If nothing was designed for it, the database receives full production traffic instantly, connection pools exhaust, query latency climbs, timeouts cascade, and you have a full outage. The cache going down took the site down — which means the cache was a load-bearing dependency, not an optimization.

What should happen: the application detects cache errors quickly via a short client timeout, logs it, and falls through to the database with a circuit breaker limiting concurrent queries so the database degrades rather than collapses. Requests that can't get through are served stale from a longer-lived backup copy, or get a degraded response.

The design principles behind that: client timeouts to the cache must be shorter than database timeouts so a slow cache doesn't consume your request budget; keep an L1 in-process cache so each instance absorbs some of the outage; implement serve-stale-on-error, which is the single highest-leverage resilience pattern here; and run Redis with replicas and automatic failover so a single node loss isn't a total loss.

And I'd want this tested — deliberately killing the cache in a staging environment under load, because everyone believes their fallback works until they try it.
</details>

<details>
<summary><b>Q10. Explain `stale-while-revalidate`.</b></summary>

An HTTP cache directive that separates "how long is this fresh" from "how long may it be served while refreshing."

With `max-age=60, stale-while-revalidate=600`, the cache serves fresh content for the first minute. Between one and eleven minutes it serves the stale copy *immediately* while fetching a fresh one in the background. Past that it blocks and fetches.

Two benefits. Users never wait for revalidation, so p99 latency stays flat rather than spiking whenever an entry expires. And the origin receives exactly one request per key per window instead of a burst — it's built-in stampede protection at the HTTP layer.

The same concept appears throughout caching: Caffeine's `refreshAfterWrite`, Next.js ISR, and TanStack Query's stale-while-revalidate behavior are all this idea. It's usually the right default for anything where slightly stale is acceptable, which is most content.
</details>

---

## 14. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                          CACHING — ONE PAGE                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ BEFORE CACHING ASK: read:write ratio? staleness budget IN SECONDS?   ║
║   what happens when it's empty or down?                              ║
║ Hit rate value is NONLINEAR: 90%→99% halves DB load again            ║
╠══════════════════════════════════════════════════════════════════════╣
║ PATTERNS                                                             ║
║   cache-aside ⭐ default — read: check→miss→DB→populate               ║
║                            write: DB then DELETE key (not SET!)      ║
║   read-through · write-through · write-behind(data loss risk)        ║
╠══════════════════════════════════════════════════════════════════════╣
║ THE THREE FAILURES                                                   ║
║   STAMPEDE  hot key expires, N requests hit DB                       ║
║     → jitter TTL + early probabilistic expiry OR lock+serve stale    ║
║   PENETRATION  nonexistent keys bypass cache                         ║
║     → cache negatives (short TTL) OR bloom filter                    ║
║   AVALANCHE  mass expiry / cache down                                ║
║     → jitter + L1 + circuit breaker + SERVE STALE ON ERROR ⭐         ║
╠══════════════════════════════════════════════════════════════════════╣
║ INVALIDATION: TTL · explicit delete · versioned keys (INCR) ·        ║
║   tags (SET of keys) · CDC events                                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ REDIS: single-threaded core → NEVER run O(n) on big collections      ║
║   KEYS→SCAN · HGETALL on huge hash · big key delete = stall          ║
║   set maxmemory-policy (default noeviction REJECTS WRITES!)          ║
║   structures: string hash list set zset bitmap hll stream geo        ║
║   zset = leaderboard + priority queue + sliding-window rate limit    ║
╠══════════════════════════════════════════════════════════════════════╣
║ LOCKS: SET NX PX + Lua ownership check on release                    ║
║   ⚠️ NOT safe for correctness (stall→expiry→double holder)            ║
║   need fencing tokens, or use DB transactions instead                ║
╠══════════════════════════════════════════════════════════════════════╣
║ NEVER CACHE: balances · inventory · anything where stale = wrong     ║
║ ALWAYS: tenant_id in the key · client timeout < DB timeout           ║
║ HTTP: stale-while-revalidate is the best directive available         ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Databases](databases.md) · [API Design](api-design.md) · [System Design](../05-system-design/00-fundamentals.md) · [Browser Performance](../02-frontend/browser-performance.md)
