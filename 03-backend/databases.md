# 🗄️ Databases

> The database is where correctness lives. This book covers how storage engines actually work, why indexes are shaped the way they are, what transactions really guarantee, and how to model data you won't regret.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [Storage Engines: B-Tree vs LSM](#2-storage-engines)
3. [Indexes](#3-indexes)
4. [Query Execution and Planning](#4-query-execution)
5. [Transactions and ACID](#5-transactions-and-acid)
6. [Isolation Levels and Anomalies](#6-isolation-levels)
7. [Concurrency Control](#7-concurrency-control)
8. [Locking and Deadlocks](#8-locking)
9. [Data Modeling](#9-data-modeling)
10. [Normalization and Denormalization](#10-normalization)
11. [NoSQL Landscape](#11-nosql)
12. [Replication](#12-replication)
13. [Sharding and Partitioning](#13-sharding)
14. [Performance Tuning](#14-performance-tuning)
15. [Migrations](#15-migrations)
16. [Operational Concerns](#16-operations)
17. [Interview Section](#17-interview-section)
18. [Cheat Sheet](#18-cheat-sheet)

---

## 1. Mental Model

```
   A database is four layers. Every performance question
   is "which layer is the bottleneck?"

   ┌─────────────────────────────────────────────────────────┐
   │  QUERY LAYER      parse → rewrite → plan → optimize     │
   │                   (cost-based, uses statistics)          │
   ├─────────────────────────────────────────────────────────┤
   │  TRANSACTION      MVCC, locks, WAL, isolation           │
   │  LAYER            (what makes concurrency correct)       │
   ├─────────────────────────────────────────────────────────┤
   │  ACCESS METHODS   B-tree, LSM, hash, heap, bitmap       │
   │                   (how rows are found)                   │
   ├─────────────────────────────────────────────────────────┤
   │  STORAGE          pages, buffer pool, disk I/O          │
   │                   (the actual bytes)                     │
   └─────────────────────────────────────────────────────────┘
```

**The one number that explains everything:**

```
   L1 cache        ~1 ns          ┐
   RAM             ~100 ns        │  100× slower than cache
   NVMe SSD        ~100 µs        │  1,000× slower than RAM
   Network (same DC) ~500 µs      │
   HDD seek        ~10 ms         ┘  100,000× slower than RAM

   → Every database design is about MINIMIZING DISK SEEKS.
     That is why B-trees are wide and shallow.
     That is why the buffer pool exists.
     That is why sequential scans can beat random index lookups.
```

---

## 2. Storage Engines

### 2.1 B-Tree (B+Tree) — Postgres, MySQL/InnoDB, Oracle

```
                    ┌─────────────────┐
                    │   50  │   100   │        ROOT (internal node:
                    └──┬────┴────┬────┘         keys + child pointers only)
             ┌─────────┘         └──────────┐
        ┌────▼──────┐              ┌────────▼───┐
        │ 20 │ 35   │              │ 70  │  85  │   INTERNAL
        └─┬──┴──┬───┘              └──┬──┴───┬──┘
     ┌────┘     └────┐          ┌─────┘      └────┐
   ┌─▼───┐        ┌──▼──┐    ┌──▼──┐          ┌───▼─┐
   │10,15│◀──────▶│20,30│◀──▶│50,60│◀────────▶│70,80│   LEAVES
   └─────┘        └─────┘    └─────┘          └─────┘
      ▲ leaves are LINKED → range scans are sequential
      ▲ ALL data lives in leaves (that's the "+" in B+tree)

   Height for 100M rows with ~200 keys per node: ~4 levels
   → any row is 4 page reads away, and the top 2-3 levels
     are always cached in RAM. So it's effectively 1 disk read.
```

**Why "B"-tree shape:** node size matches the disk page size (typically 8 KB in Postgres, 16 KB in InnoDB). One page read gets you ~200 keys, so the tree stays shallow. A binary tree over 100M rows would be 27 levels deep = 27 disk seeks.

### 2.2 LSM Tree — Cassandra, RocksDB, LevelDB, ScyllaDB

```
   WRITE PATH                              READ PATH
   ──────────                              ─────────
   1. Append to WAL (sequential, fast)     1. Check memtable
   2. Insert into MEMTABLE (in-memory      2. Check bloom filters per SSTable
      sorted structure, e.g. skiplist)        (probabilistic: "definitely not
                                               here" or "maybe here")
   3. When full → flush to an immutable    3. Read matching SSTables newest→oldest
      SSTable on disk (sequential write)   4. Merge results

   ┌─────────────┐
   │  MEMTABLE   │  in RAM, sorted
   └──────┬──────┘
          │ flush when full
          ▼
   L0 │ SST │ SST │ SST │      ← may have overlapping key ranges
          │ compaction (merge + drop deleted/overwritten keys)
          ▼
   L1 │ SST │ SST │ SST │ SST │  ← non-overlapping, 10× larger
          ▼
   L2 │ ...............................│  ← 10× larger again
```

### 2.3 The Fundamental Tradeoff

| | B-Tree | LSM Tree |
|---|---|---|
| Write | In-place update → **random I/O**, read-modify-write | Append-only → **sequential I/O** |
| Write amplification | Lower (~2-3×) | Higher (~10-30× from compaction) |
| Read | Predictable, ~1 lookup | May check several SSTables |
| Read amplification | Low | Higher (mitigated by bloom filters) |
| Space amplification | Fragmentation (~30% waste) | Old versions until compacted |
| Range scans | Excellent (linked leaves) | Good (merge iterators) |
| Compression | Page-level, moderate | Block-level, excellent |
| Best for | **Read-heavy, OLTP, ad-hoc queries** | **Write-heavy, time-series, logs** |

```
   RUM Conjecture: you can optimize for two of
   Read, Update, Memory — never all three.

              Read
               /\
              /  \
        B-tree    \
            /      \
           /________\
      Memory      Update(write)
                    LSM
```

---

## 3. Indexes

### 3.1 What an index actually is

An index is a **redundant, sorted copy** of some columns plus a pointer back to the row. It trades write speed and disk space for read speed.

```
   Table (heap, unordered)          Index on email (B-tree, sorted)
   ┌──────────────────────┐         ┌──────────────────────────┐
   │ ctid  id  email  name│         │ email          → ctid    │
   │ (0,1)  3  c@x   Cy   │◀────────│ a@x            → (0,3)   │
   │ (0,2)  1  a@y   Al   │         │ a@y            → (0,2)   │
   │ (0,3)  7  a@x   Ad   │         │ b@z            → (1,1)   │
   │ (1,1)  9  b@z   Bo   │         │ c@x            → (0,1)   │
   └──────────────────────┘         └──────────────────────────┘
```

### 3.2 Index types

| Type | Structure | Good for | DB |
|---|---|---|---|
| **B-tree** | Balanced tree | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, prefix `LIKE 'a%'` | Everywhere |
| **Hash** | Hash table | `=` only. No ranges, no sorting | PG, MySQL(memory) |
| **GIN** | Inverted index | Arrays, JSONB containment, full-text | Postgres |
| **GiST** | Generalized search tree | Geometric, ranges, nearest-neighbor | Postgres |
| **BRIN** | Min/max per block range | **Huge** tables with natural physical ordering (time-series) | Postgres |
| **Bitmap** | Bit vectors | Low-cardinality columns in analytics | Oracle, PG (transient) |
| **Full-text** | Inverted | Text search with ranking | PG, MySQL, ES |
| **Vector (HNSW/IVFFlat)** | Graph/clustered | Similarity search on embeddings | pgvector, dedicated |

BRIN deserves attention: an index on a 1 TB time-series table might be 50 GB as a B-tree and **50 MB** as BRIN, because it only stores the min/max timestamp per 128-page range. It only works when the physical row order correlates with the indexed column — which is exactly the case for append-only time-series.

### 3.3 Composite indexes and the leftmost prefix rule

```sql
CREATE INDEX idx ON orders (customer_id, status, created_at);
```

```
   The index is sorted by customer_id, then status, then created_at.
   Think of it like a phone book sorted by (last, first, middle).

   ✅ USES the index:
      WHERE customer_id = 5
      WHERE customer_id = 5 AND status = 'paid'
      WHERE customer_id = 5 AND status = 'paid' AND created_at > '2026-01-01'
      WHERE customer_id = 5 ORDER BY status, created_at
      WHERE customer_id = 5 AND status = 'paid' ORDER BY created_at   ← no sort needed!

   ⚠️ PARTIAL use (only the customer_id part):
      WHERE customer_id = 5 AND created_at > '...'   -- skips status, can't seek on it

   ❌ CANNOT use it:
      WHERE status = 'paid'                          -- no leftmost column
      WHERE created_at > '...'
```

**Column order rules:**
1. Equality columns first, range columns last.
2. Among equality columns, most selective first *(this matters less than people think — put the ones always present first).*
3. A column used with `>`/`<`/`BETWEEN` stops further columns from being used for seeking.

```sql
-- ❌ range before equality: can't seek on status
CREATE INDEX bad ON orders (created_at, status);
-- ✅
CREATE INDEX good ON orders (status, created_at);
```

### 3.4 Covering indexes

```sql
-- Query needs id, status, total
SELECT id, total FROM orders WHERE customer_id = 5 AND status = 'paid';

-- Index-only scan: everything the query needs is IN the index,
-- so the table is never touched
CREATE INDEX idx ON orders (customer_id, status) INCLUDE (id, total);  -- PG 11+
CREATE INDEX idx ON orders (customer_id, status, id, total);           -- any DB
```

```
   Normal index scan:              Index-only scan:
   index → find ctid → HEAP READ   index → done ✅
        (random I/O!)               (no table access)
```

This is often a 10× improvement. In Postgres it requires the visibility map to be current, which means keeping autovacuum healthy.

### 3.5 Partial and expression indexes

```sql
-- Partial: index only the rows you query. Smaller, faster, cheaper to maintain.
CREATE INDEX idx_active ON users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_pending ON orders (created_at) WHERE status = 'pending';
-- If 99% of orders are completed, this index is 100× smaller.

-- Expression: index the computed value
CREATE INDEX idx_lower_email ON users (LOWER(email));
-- Now this uses the index:
SELECT * FROM users WHERE LOWER(email) = 'a@b.com';
```

### 3.6 Why indexes get ignored

```sql
-- ❌ Function on the column kills index usage
WHERE LOWER(email) = 'a@b.com'          -- unless you have an expression index
WHERE DATE(created_at) = '2026-01-01'   -- ✅ use: created_at >= '...' AND < '...'
WHERE id + 0 = 5

-- ❌ Leading wildcard
WHERE name LIKE '%smith'                -- ✅ trigram index (pg_trgm) fixes this

-- ❌ Type mismatch → implicit cast
WHERE varchar_col = 123                 -- casts the COLUMN, not the literal

-- ❌ OR across different columns
WHERE a = 1 OR b = 2                    -- ✅ UNION of two indexed queries

-- ❌ Low selectivity: if 40% of rows match, a seq scan is genuinely faster
--    because random I/O for 40% of rows costs more than reading it all
--    sequentially. The planner is right; don't fight it.

-- ❌ Stale statistics → the planner has a wrong row estimate
ANALYZE orders;
```

### 3.7 The cost of indexes

```
   Every INSERT/UPDATE/DELETE must update EVERY index on the table.

   5 indexes → 6× the write work (1 heap + 5 index writes)
   → Also 6× WAL volume, 6× replication traffic

   Find unused indexes (Postgres):
   SELECT schemaname, relname, indexrelname, idx_scan,
          pg_size_pretty(pg_relation_size(indexrelid))
   FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY 5 DESC;
```

---

## 4. Query Execution

### 4.1 Reading an EXPLAIN plan

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id, u.name;
```

```
 HashAggregate  (cost=1250.00..1300.00 rows=5000 width=40)
                (actual time=45.2..47.1 rows=4823 loops=1)
   Group Key: u.id
   ->  Hash Join  (cost=200.00..1100.00 rows=20000 width=36)
                  (actual time=5.1..38.4 rows=19540 loops=1)
         Hash Cond: (o.user_id = u.id)
         ->  Seq Scan on orders o  (actual time=0.01..12.3 rows=100000 loops=1)
               Buffers: shared hit=1250 read=3200        ← read = DISK
         ->  Hash  (actual time=5.0..5.0 rows=5000 loops=1)
               ->  Index Scan using idx_users_created on users u
                     Index Cond: (created_at > '2026-01-01')
 Planning Time: 0.3 ms
 Execution Time: 48.9 ms
```

**How to read it:**

```
   1. Read INSIDE OUT — the most indented node runs first
   2. Compare `rows=` estimated  vs  `rows=` actual
      → a 10×+ gap means bad statistics → run ANALYZE
   3. `loops=N` means the node ran N times; multiply the time
   4. Buffers: `read` = disk I/O (bad), `hit` = cache (good)
   5. Find the node consuming the most actual time
```

### 4.2 Scan types

| Node | Meaning | When it's right |
|---|---|---|
| `Seq Scan` | Read the whole table | Small table, or >~10-20% of rows match |
| `Index Scan` | Walk index, fetch each row | High selectivity, few rows |
| `Index Only Scan` | Index has everything ⭐ | Covering index |
| `Bitmap Heap Scan` | Collect ctids, sort, read pages in order | Medium selectivity — turns random I/O into sequential |
| `Nested Loop` | For each outer row, probe inner | Small outer × indexed inner |
| `Hash Join` | Build hash of one side, probe with the other | Large, unsorted, equality join |
| `Merge Join` | Both sides sorted, walk together | Both already sorted, or huge inputs |

```
   JOIN ALGORITHM COST
   
   Nested Loop:  O(n × m)     but O(n × log m) with an index on the inner
                 ✅ best when n is tiny
   Hash Join:    O(n + m)     needs memory for the hash table
                 ✅ best for large equality joins
   Merge Join:   O(n log n + m log m)  including sorts
                 ✅ best when inputs are pre-sorted
```

### 4.3 Warning signs in a plan

```
   ⚠️ rows=1 estimated but rows=500000 actual   → stale stats, run ANALYZE
   ⚠️ Seq Scan on a large table with a filter    → missing index
   ⚠️ Sort Method: external merge Disk: 250MB    → increase work_mem
   ⚠️ Nested Loop with a huge outer row count    → planner misjudged; check stats
   ⚠️ loops=10000 on an inner node               → this is your N+1
   ⚠️ Rows Removed by Filter: 990000             → index isn't selective enough
```

---

## 5. Transactions and ACID

```
   A — ATOMICITY    All or nothing. Implemented by the WAL + undo.
   C — CONSISTENCY  Constraints hold before and after.
                    ⚠️ This is YOUR job (the app + constraints), not the DB's magic.
   I — ISOLATION    Concurrent transactions don't corrupt each other.
                    This is the hard one — see isolation levels.
   D — DURABILITY   Committed = survives a crash. WAL flushed to disk (fsync).
```

### 5.1 Write-Ahead Logging

```
   THE RULE: write the log record to durable storage BEFORE
             modifying the data page.

   1. Change happens in the buffer pool (memory) — page marked dirty
   2. WAL record appended to the log buffer
   3. COMMIT → fsync the WAL to disk  ← the ONLY synchronous disk write
   4. Client gets "committed"
   5. Later, a background writer flushes dirty pages (checkpoint)

   Crash recovery:
     REDO  — replay WAL records after the last checkpoint
     UNDO  — roll back transactions that were in-flight
```

Why this is fast: the WAL is a **sequential append**, and only that one write must be synchronous. Data pages are written lazily, in batches, in whatever order is efficient.

```
   fsync is the expensive part:
     ~0.1 ms on NVMe with a battery-backed cache
     ~5-10 ms on a spinning disk
   → Group commit batches many transactions into one fsync.
   → synchronous_commit=off trades durability (a few hundred ms
     of committed transactions) for a big throughput win. Sometimes correct.
```

---

## 6. Isolation Levels

### 6.1 The anomalies

```
   DIRTY READ
   T1: UPDATE balance = 500        (not committed)
   T2:                  SELECT balance → 500   ← reads uncommitted data
   T1: ROLLBACK                                ← that 500 never existed

   NON-REPEATABLE READ
   T1: SELECT balance → 100
   T2: UPDATE balance = 500; COMMIT
   T1: SELECT balance → 500        ← same query, different answer

   PHANTOM READ
   T1: SELECT COUNT(*) WHERE age > 30 → 10
   T2: INSERT a row with age = 35; COMMIT
   T1: SELECT COUNT(*) WHERE age > 30 → 11   ← new ROWS appeared

   LOST UPDATE
   T1: read 100 ─────────────▶ write 110
   T2:      read 100 ──────────────────▶ write 120   ← T1's +10 is lost

   WRITE SKEW  (the subtle one)
   Constraint: at least one doctor must be on call.
   T1: sees 2 on call, sets doctor A to off-duty
   T2: sees 2 on call, sets doctor B to off-duty
   Both commit. Now ZERO doctors on call — no row was written twice,
   but the invariant is broken.
```

### 6.2 The levels

| Level | Dirty read | Non-repeatable | Phantom | Lost update | Write skew |
|---|---|---|---|---|---|
| Read Uncommitted | ✅ possible | ✅ | ✅ | ✅ | ✅ |
| **Read Committed** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Repeatable Read** | ❌ | ❌ | ❌* | ❌* | ✅ |
| **Serializable** | ❌ | ❌ | ❌ | ❌ | ❌ |

\* In Postgres, Repeatable Read is snapshot isolation and *does* prevent phantoms and lost updates (it aborts with a serialization failure). In MySQL InnoDB, Repeatable Read uses gap locks. The SQL standard is weaker than both.

**Defaults you must know:**

```
   PostgreSQL   → READ COMMITTED
   MySQL/InnoDB → REPEATABLE READ
   Oracle       → READ COMMITTED
   SQL Server   → READ COMMITTED (with locking, not MVCC, by default)
```

⚠️ This difference bites people porting applications. Postgres READ COMMITTED takes a **new snapshot for every statement**; MySQL REPEATABLE READ takes one snapshot for the whole transaction.

### 6.3 Choosing a level

```sql
-- Read Committed (default): fine for most CRUD
-- Each statement sees a fresh snapshot.

-- Repeatable Read: reports/exports needing a consistent view
BEGIN ISOLATION LEVEL REPEATABLE READ;
  SELECT SUM(amount) FROM accounts;      -- consistent snapshot
  SELECT COUNT(*) FROM accounts;         -- same snapshot
COMMIT;

-- Serializable: when correctness depends on invariants across rows
BEGIN ISOLATION LEVEL SERIALIZABLE;
  -- Postgres uses SSI (Serializable Snapshot Isolation): optimistic,
  -- detects dangerous read-write dependency cycles and ABORTS one.
  -- ⚠️ Your app MUST handle serialization_failure (SQLSTATE 40001) and retry.
COMMIT;
```

```python
# Retry loop — mandatory with SERIALIZABLE
for attempt in range(5):
    try:
        with db.transaction(isolation="serializable"):
            do_work()
        break
    except SerializationFailure:
        time.sleep(0.05 * 2**attempt * random.random())
else:
    raise
```

---

## 7. Concurrency Control

### 7.1 MVCC

Multi-Version Concurrency Control: writers create new versions instead of overwriting, so **readers never block writers and writers never block readers.**

```
   Postgres row versions (each row has xmin/xmax)

   ┌────────────────────────────────────────────────┐
   │ id=1 balance=100  xmin=100  xmax=105           │ ← old version
   │ id=1 balance=150  xmin=105  xmax=null          │ ← current
   └────────────────────────────────────────────────┘

   A transaction with snapshot xid=103 sees the FIRST version
   A transaction with snapshot xid=110 sees the SECOND version

   Visibility rule: a row is visible if
     xmin is committed AND xmin < my snapshot
     AND (xmax is null OR xmax > my snapshot OR xmax aborted)
```

**The cost: dead tuples.** Old versions accumulate and must be reclaimed.

```
   Postgres:  VACUUM reclaims dead tuples; autovacuum does it automatically.
              ⚠️ A long-running transaction blocks vacuum from cleaning ANY
                 tuple newer than its snapshot → table bloat → slow everything.
              ⚠️ Transaction ID wraparound: if vacuum never runs, at ~2 billion
                 transactions the DB shuts down to prevent data loss.

   MySQL:     Old versions go to the UNDO log (rollback segments), which
              also grows under long transactions.

   Oracle:    Undo tablespace; "snapshot too old" when it's exhausted.
```

🏭 **The single most important operational rule: do not hold transactions open.** No user input, no HTTP calls, no `sleep` inside a transaction.

### 7.2 Optimistic vs Pessimistic

```sql
-- PESSIMISTIC: lock first, then work. Blocks others.
BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;   -- exclusive row lock
  -- ... nobody else can touch this row ...
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

SELECT ... FOR UPDATE NOWAIT;         -- fail immediately instead of waiting
SELECT ... FOR UPDATE SKIP LOCKED;    -- ⭐ skip locked rows — the queue pattern
SELECT ... FOR SHARE;                 -- shared lock: others can read, not write

-- OPTIMISTIC: assume no conflict, detect at write time
UPDATE documents SET content = $1, version = version + 1
WHERE id = $2 AND version = $3;       -- 0 rows affected = someone beat you → 409
```

| | Pessimistic | Optimistic |
|---|---|---|
| Conflict rate | High | Low |
| Cost | Blocking, deadlock risk | Retry / user-visible conflict |
| Use | Inventory, seat booking, balances | Document editing, profile updates |

**`SKIP LOCKED` is the job queue pattern** — multiple workers can pull distinct jobs from one table without blocking each other:

```sql
UPDATE jobs SET status = 'processing', worker_id = $1
WHERE id IN (
  SELECT id FROM jobs WHERE status = 'pending'
  ORDER BY created_at LIMIT 10
  FOR UPDATE SKIP LOCKED           -- ⭐ each worker gets different rows
)
RETURNING *;
```

---

## 8. Locking

```
   LOCK COMPATIBILITY (Postgres row/table level, simplified)

              │ ACCESS  │  ROW    │  ROW    │  SHARE  │ EXCLUSIVE │ ACCESS
              │ SHARE   │  SHARE  │  EXCL   │         │           │ EXCL
   ───────────┼─────────┼─────────┼─────────┼─────────┼───────────┼────────
   ACCESS SH  │   ✅    │   ✅    │   ✅    │   ✅    │    ✅     │   ❌
   ROW SHARE  │   ✅    │   ✅    │   ✅    │   ✅    │    ❌     │   ❌
   ROW EXCL   │   ✅    │   ✅    │   ✅    │   ❌    │    ❌     │   ❌
   SHARE      │   ✅    │   ✅    │   ❌    │   ✅    │    ❌     │   ❌
   EXCLUSIVE  │   ✅    │   ❌    │   ❌    │   ❌    │    ❌     │   ❌
   ACCESS EXC │   ❌    │   ❌    │   ❌    │   ❌    │    ❌     │   ❌

   SELECT takes ACCESS SHARE. ALTER TABLE takes ACCESS EXCLUSIVE.
   → an ALTER TABLE waits for every running query, and every new query
     queues behind it. This is how a "quick migration" takes down a site.
```

### 8.1 Deadlocks

```
   T1: LOCK row A ───────▶ wants row B ──┐
                                          │  CYCLE → deadlock
   T2: LOCK row B ───────▶ wants row A ──┘

   The DB detects the cycle and kills one transaction (the "victim").
```

**Prevention:**

```sql
-- ✅ Always acquire locks in a consistent global order
UPDATE accounts SET ... WHERE id IN (1, 2) ORDER BY id;
-- (or in application code: sort the IDs before locking)

-- ✅ Keep transactions short
-- ✅ Use a lower isolation level where correct
-- ✅ Set a lock timeout so you fail fast instead of hanging
SET lock_timeout = '3s';
SET statement_timeout = '30s';
SET idle_in_transaction_session_timeout = '60s';   -- ⭐ kill zombie transactions
```

### 8.2 Safe migrations

```sql
-- ❌ Locks the table for the entire rebuild
CREATE INDEX idx ON big_table (col);
-- ✅ No write lock; slower, can fail and leave an invalid index
CREATE INDEX CONCURRENTLY idx ON big_table (col);

-- ❌ Rewrites the whole table (older Postgres) / locks it
ALTER TABLE t ADD COLUMN c int NOT NULL DEFAULT 0;
-- ✅ Multi-step
ALTER TABLE t ADD COLUMN c int;                          -- instant, nullable
-- backfill in batches
ALTER TABLE t ADD CONSTRAINT c_nn CHECK (c IS NOT NULL) NOT VALID;
ALTER TABLE t VALIDATE CONSTRAINT c_nn;                  -- takes a weaker lock
```

⚠️ **Always set a `lock_timeout` before a migration.** Without it, `ALTER TABLE` waits behind a long query while every new query queues behind *it* — a full outage from a "safe" DDL.

---

## 9. Data Modeling

### 9.1 Modeling process

```
   1. Identify ENTITIES        (nouns: user, order, product)
   2. Identify RELATIONSHIPS   (1:1, 1:N, M:N)
   3. Identify ATTRIBUTES      (with types AND constraints)
   4. Identify ACCESS PATTERNS ⭐ what queries will run, how often
   5. Normalize to 3NF
   6. Selectively denormalize where step 4 demands it
```

Step 4 is the one people skip. In NoSQL it comes *first* — you design tables around queries. In SQL it should still inform your indexes and any denormalization.

### 9.2 Relationships

```sql
-- 1:N — foreign key on the "many" side
CREATE TABLE orders (
  id         bigserial PRIMARY KEY,
  user_id    bigint NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
  total_cents bigint NOT NULL CHECK (total_cents >= 0),
  status     text NOT NULL CHECK (status IN ('pending','paid','shipped','cancelled')),
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON orders (user_id, created_at DESC);

-- M:N — junction table
CREATE TABLE order_items (
  order_id   bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity   int NOT NULL CHECK (quantity > 0),
  unit_price_cents bigint NOT NULL,    -- ⭐ snapshot the price at purchase time
  PRIMARY KEY (order_id, product_id)
);

-- 1:1 — shared PK, or a unique FK
CREATE TABLE user_profiles (
  user_id bigint PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  bio     text
);
```

⚠️ **Snapshot values that must not change historically.** An order line must store the price at purchase time, not join to the current product price — otherwise last year's invoices change when you run a sale.

### 9.3 Key choices

| Key type | Pro | Con |
|---|---|---|
| `bigserial` / auto-increment | Small, sequential (good B-tree locality) | Leaks volume; hotspot on the rightmost page in distributed setups |
| UUID v4 | Client-generatable, no coordination | Random → terrible index locality, page splits, 16 bytes |
| **UUID v7 / ULID** ⭐ | Time-ordered *and* globally unique | 16 bytes |
| Natural key (email, SKU) | Meaningful | Business values change; wide FKs |
| Composite | Models the domain | Awkward FKs |

🏭 **Recommendation:** `bigserial` for internal PKs, plus a separate public-facing prefixed ULID/UUIDv7 for API exposure. You get index locality internally and non-enumerable IDs externally.

### 9.4 Modeling patterns

```sql
-- Soft delete (partial index keeps it fast)
deleted_at timestamptz;
CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL;

-- Audit trail
CREATE TABLE order_events (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL,
  event_type text NOT NULL,
  payload jsonb NOT NULL,
  actor_id bigint,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON order_events (order_id, created_at);

-- Temporal / slowly changing dimension
CREATE TABLE price_history (
  product_id bigint NOT NULL,
  price_cents bigint NOT NULL,
  valid_from timestamptz NOT NULL,
  valid_to   timestamptz,                     -- null = current
  EXCLUDE USING gist (product_id WITH =,      -- ⭐ no overlapping periods
    tstzrange(valid_from, valid_to) WITH &&)
);

-- Multi-tenancy: row-level
tenant_id bigint NOT NULL;                    -- on EVERY table
CREATE INDEX ON orders (tenant_id, created_at);   -- tenant_id FIRST in every index
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- Hierarchies
-- 1. Adjacency list + recursive CTE  (simple, flexible)
parent_id bigint REFERENCES categories(id);
WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 1 AS depth FROM categories WHERE id = $1
  UNION ALL
  SELECT c.id, c.name, c.parent_id, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
) SELECT * FROM tree;
-- 2. Materialized path (fast subtree reads): path ltree / text '1.4.9'
-- 3. Closure table (fast both ways, more writes)
-- 4. Nested set (fastest reads, painful writes)
```

### 9.5 JSONB — use with discipline

```sql
data jsonb NOT NULL DEFAULT '{}';
CREATE INDEX ON events USING GIN (data);              -- containment queries
CREATE INDEX ON events USING GIN (data jsonb_path_ops);  -- smaller, @> only
CREATE INDEX ON events ((data->>'user_id'));          -- specific key, B-tree

SELECT * FROM events WHERE data @> '{"type": "click"}';
```

| Use JSONB for | Don't use JSONB for |
|---|---|
| Genuinely variable schemas (webhook payloads, event data) | Fields you filter or join on regularly |
| User-defined custom fields | Anything needing a constraint or FK |
| Sparse attributes across many types | Data you'll aggregate frequently |
| Caching a denormalized read model | Avoiding the work of schema design |

⚠️ JSONB has no referential integrity, no type checking, no column statistics (so the planner guesses), and every update rewrites the entire document. It's a tool, not a schema-design escape hatch.

---

## 10. Normalization

```
   1NF  Atomic values. No repeating groups, no arrays-as-CSV-strings.
   2NF  1NF + no partial dependency on part of a composite key.
   3NF  2NF + no transitive dependency (non-key → non-key).
   BCNF Stricter 3NF: every determinant is a candidate key.
   4NF  No multi-valued dependencies.
   5NF  No join dependencies.

   → In practice, aim for 3NF/BCNF, then denormalize deliberately.
```

**Example of the 3NF violation people actually make:**

```sql
-- ❌ city and state depend on zip, not on the order id
CREATE TABLE orders (id, customer_id, zip, city, state, total);
-- Update anomaly: change a zip's city and you must update every order row.

-- ✅
CREATE TABLE orders (id, customer_id, zip, total);
CREATE TABLE zip_codes (zip PRIMARY KEY, city, state);
```

### Denormalize when — and only when

```
   ✅ A read query is measurably too slow and the join is the cause
   ✅ Read:write ratio is very high
   ✅ You have a plan to keep the copy consistent
      (trigger, materialized view, application logic, CDC)
   ✅ Staleness is acceptable and documented

   ❌ "Joins are slow" as a general belief — they usually aren't
   ❌ Before measuring
```

```sql
-- Counter cache: avoid COUNT(*) on a large child table
ALTER TABLE posts ADD COLUMN comment_count int NOT NULL DEFAULT 0;
-- Maintain with a trigger, and reconcile periodically —
-- triggers can drift after bulk operations.

-- Materialized view for expensive aggregates
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT date_trunc('day', created_at) AS day, SUM(total_cents) AS revenue
FROM orders WHERE status = 'paid' GROUP BY 1;
CREATE UNIQUE INDEX ON daily_revenue (day);          -- required for CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;  -- no read lock
```

---

## 11. NoSQL

```
                        CAP THEOREM
   During a network PARTITION, choose Consistency or Availability.
   (When there's no partition, you get both — the "P" is not optional.)

              Consistency
                  /\
                 /  \
      CP        /    \        CA (single node only —
   MongoDB     /      \        not achievable in a
   HBase      /        \       real distributed system)
   Zookeeper /__________\
   etcd    Partition  Availability
           tolerance      AP
                       Cassandra
                       DynamoDB (tunable)
                       Riak

   PACELC extends it: else (no partition), choose Latency or Consistency.
   DynamoDB = PA/EL, Postgres = PC/EC
```

| Type | Examples | Model | Best for |
|---|---|---|---|
| **Key-Value** | Redis, DynamoDB, Memcached | `key → blob` | Caching, sessions, simple lookups |
| **Document** | MongoDB, Couchbase, Firestore | Nested JSON docs | Varied schemas, aggregates read together |
| **Wide-Column** | Cassandra, ScyllaDB, HBase | Row key → dynamic columns | Massive writes, time-series |
| **Graph** | Neo4j, Neptune | Nodes + edges | Traversals, recommendations, fraud rings |
| **Search** | Elasticsearch, OpenSearch | Inverted index | Full-text, faceted search, logs |
| **Time-Series** | InfluxDB, TimescaleDB | Timestamped metrics | Monitoring, IoT |
| **Vector** | pgvector, Pinecone, Milvus | Embeddings + ANN | Semantic search, RAG |

### 11.1 Cassandra data modeling

The rule: **one table per query.** Denormalization is the design, not a compromise.

```sql
PRIMARY KEY ((partition_key), clustering_col1, clustering_col2)
             └─ determines the NODE      └─ sort order WITHIN the partition

-- Query: "recent messages in a room"
CREATE TABLE messages_by_room (
  room_id uuid,
  bucket text,                     -- ⭐ e.g. '2026-08' — bounds partition size
  created_at timeuuid,
  message_id uuid,
  body text,
  PRIMARY KEY ((room_id, bucket), created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

⚠️ Cassandra's failure mode is the **unbounded partition**. If `room_id` alone is the partition key, a busy room's partition grows forever until reads time out and compaction chokes. Bucketing by time is mandatory. Target partitions under ~100 MB.

### 11.2 DynamoDB single-table design

```
   PK              SK                    Attributes
   ─────────────── ───────────────────── ─────────────────────
   USER#123        PROFILE               name, email
   USER#123        ORDER#2026-08-01#456  total, status
   USER#123        ORDER#2026-08-05#789  total, status
   ORDER#456       METADATA              user_id, total

   One query: PK = USER#123 AND SK begins_with ORDER#
   → all of a user's orders, sorted, in ONE request.

   GSI (Global Secondary Index) inverts the access:
   GSI1PK = ORDER#456 → look up an order without knowing the user
```

The tradeoff is stark: DynamoDB gives you predictable single-digit-millisecond latency at any scale, but you must know your access patterns before you create the table, and adding a new one often means a migration or a new GSI.

---

## 12. Replication

```
   ┌──────────┐   WAL / binlog stream    ┌──────────┐
   │ PRIMARY  │─────────────────────────▶│ REPLICA  │  read-only
   │ (writes) │─────────────────────────▶│ REPLICA  │
   └──────────┘                          └──────────┘

   ASYNCHRONOUS   commit locally, ship later
                  ✅ fast writes  ❌ data loss window on failover
                  ❌ read-your-writes broken on replicas

   SYNCHRONOUS    wait for N replicas to confirm
                  ✅ no data loss  ❌ write latency = slowest replica
                                    ❌ availability drops (a stalled replica
                                       blocks all writes unless configured
                                       to degrade)

   SEMI-SYNC      wait for the replica to RECEIVE (not apply) — a middle ground
```

### Replication lag

```
   The single most common production surprise:

   1. POST /orders → writes to PRIMARY
   2. 302 redirect → GET /orders/123 → reads from a REPLICA
   3. Replica is 200 ms behind → 404 Not Found
   4. User: "my order disappeared"
```

**Fixes:**

```
   • Read-your-writes: route a user's reads to the primary for N seconds
     after they write (sticky session, or a cookie with a timestamp)
   • Track the write LSN/GTID in the session, and wait for the replica
     to catch past it before reading
   • Read from the primary for anything in a read-after-write flow
   • Monitor lag and pull replicas out of rotation above a threshold
```

### Topologies

| Topology | Writes | Use |
|---|---|---|
| Single primary | One node | Default — simple, consistent |
| Multi-primary | Any node | ⚠️ Conflict resolution required; usually a mistake |
| Cascading | Primary → replica → replica | Reduces primary load, adds lag |
| Quorum (Dynamo-style) | W + R > N | Tunable consistency |

```
   Quorum: N=3 replicas
     W=2, R=2  →  W+R > N  →  strong consistency, tolerates 1 failure
     W=1, R=1  →  fast, eventually consistent
     W=3, R=1  →  fast reads, slow writes, no write availability on any failure
```

---

## 13. Sharding

```
   Vertical:   split by TABLE      users DB | orders DB | analytics DB
               ✅ easy first step  ❌ no cross-DB joins/transactions

   Horizontal: split by ROW        shard 1: users 1-1M | shard 2: 1M-2M
               ✅ scales infinitely ❌ genuinely hard
```

### Sharding strategies

```
   RANGE-BASED     users 1-1M → shard A, 1M-2M → shard B
   ✅ range queries work     ❌ hotspots (newest shard gets all writes)

   HASH-BASED      hash(user_id) % N
   ✅ even distribution      ❌ resharding moves ~everything, no range queries

   CONSISTENT HASH  ring with virtual nodes
   ✅ adding a node moves only 1/N of keys      ← the standard answer
   
   DIRECTORY       lookup table: key → shard
   ✅ total flexibility, easy rebalance   ❌ the directory is a SPOF + extra hop

   GEO             shard by region
   ✅ data residency compliance, low latency   ❌ cross-region queries painful
```

```
   Consistent hashing with virtual nodes

        ┌───── 0 ─────┐
      A₃              B₁
    ┌─┘                └─┐
   C₂                    A₁      Each physical node owns many
   │      ring            │      points on the ring, so removing
   B₃                    C₁      one redistributes its keys across
    └─┐                ┌─┘       ALL remaining nodes evenly.
      A₂              B₂
        └──── 180 ────┘

   key → hash → walk clockwise → first node owns it
```

### What sharding costs you

```
   ❌ Cross-shard JOINs                 → denormalize, or join in the app
   ❌ Cross-shard TRANSACTIONS          → sagas, 2PC (slow, fragile), or avoid
   ❌ Global unique constraints         → central sequence service, or UUIDs
   ❌ Global secondary indexes          → a separate index shard / search cluster
   ❌ Resharding                        → the hardest operation you'll ever run
   ❌ Aggregations across all shards    → scatter-gather, or a separate OLAP store
```

🏭 **Do everything else first.** Read replicas, caching, better indexes, connection pooling, archiving cold data, vertical scaling, and partitioning within one database will take you very far. A single well-tuned Postgres instance handles tens of thousands of transactions per second and several terabytes.

**Choosing a shard key** is the decision you cannot undo:

```
   ✅ High cardinality
   ✅ Even distribution (no celebrity users)
   ✅ Present in most queries — otherwise every query is scatter-gather
   ✅ Aligns with your transaction boundary (keep a transaction on one shard)

   Common good key: tenant_id in B2B SaaS — the natural isolation unit
```

### Partitioning (within one database)

```sql
-- Declarative partitioning: one logical table, many physical ones
CREATE TABLE events (
  id bigserial, created_at timestamptz NOT NULL, payload jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

-- Benefits:
--  • Partition pruning: queries with a date filter scan only relevant partitions
--  • DROP TABLE events_2026_01  → instant deletion of old data (vs a slow DELETE)
--  • Smaller indexes per partition, faster VACUUM
```

This is usually the right answer before true sharding.

---

## 14. Performance Tuning

### 14.1 Diagnostic order

```
   1. Which queries are slow?        pg_stat_statements / slow query log
   2. Why is THAT query slow?        EXPLAIN (ANALYZE, BUFFERS)
   3. Is it I/O, CPU, or locks?      pg_stat_activity, wait events
   4. Fix the biggest one            index / rewrite / schema / config
   5. Re-measure                     ← people skip this
```

```sql
-- Postgres: top queries by total time
SELECT substring(query, 1, 80) AS q, calls,
       round(total_exec_time::numeric, 1) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows / GREATEST(calls, 1) AS avg_rows
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;

-- What is running right now / what is blocking
SELECT pid, now() - query_start AS duration, state, wait_event_type, wait_event,
       substring(query, 1, 60)
FROM pg_stat_activity
WHERE state != 'idle' ORDER BY duration DESC;

-- Blocking chains
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked.query AS blocked_query, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- Cache hit ratio (want > 0.99)
SELECT sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit + heap_blks_read), 0)
FROM pg_statio_user_tables;

-- Table bloat / vacuum health
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS pct_dead,
       last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;
```

### 14.2 The N+1 problem

```python
# ❌ 1 + 100 queries
orders = db.query("SELECT * FROM orders LIMIT 100")
for o in orders:
    o.user = db.query("SELECT * FROM users WHERE id = ?", o.user_id)

# ✅ 2 queries
orders = db.query("SELECT * FROM orders LIMIT 100")
user_ids = {o.user_id for o in orders}
users = {u.id: u for u in db.query("SELECT * FROM users WHERE id = ANY(?)", user_ids)}
for o in orders: o.user = users[o.user_id]

# ✅ 1 query
db.query("SELECT o.*, u.* FROM orders o JOIN users u ON u.id = o.user_id LIMIT 100")
```

ORM equivalents: `selectinload`/`joinedload` (SQLAlchemy), `include` (Prisma), `@EntityGraph`/`JOIN FETCH` (JPA), `select_related`/`prefetch_related` (Django).

🏭 Add an ORM query-count assertion to your integration tests. N+1 problems are invisible with 10 rows in dev and fatal with 10,000 in production.

### 14.3 Key configuration (Postgres)

```ini
shared_buffers = 25% of RAM              # DB's own cache
effective_cache_size = 50-75% of RAM     # planner hint about OS cache
work_mem = 16-64MB                       # PER SORT/HASH NODE — multiply by
                                         # concurrency × nodes per query!
maintenance_work_mem = 1GB               # VACUUM, CREATE INDEX
max_connections = 100-200                # use a pooler instead of raising this
random_page_cost = 1.1                   # SSD (default 4.0 assumes spinning disk)
effective_io_concurrency = 200           # SSD
wal_compression = on
checkpoint_completion_target = 0.9
autovacuum_vacuum_scale_factor = 0.05    # more aggressive than the 0.2 default
```

⚠️ `work_mem` is the classic OOM cause: it's allocated **per sort/hash operation per query**, so `work_mem=256MB` with 100 connections running 3-node queries can request 75 GB.

### 14.4 Connection pooling

```
   Each Postgres connection is a separate OS PROCESS (~5-10 MB).
   500 connections = 5 GB of RAM doing nothing but context switching.

   App ──┐
   App ──┼──▶ PgBouncer ──▶ 20 real DB connections
   App ──┘    (1000 client
              connections)

   Pool modes:
     session     — connection held for the whole client session (safe, least pooling)
     transaction — released at COMMIT ⭐ best ratio; breaks session state
                   (no prepared statements without care, no advisory locks,
                    no SET, no LISTEN/NOTIFY)
     statement   — released per statement; no transactions at all
```

Rule of thumb for pool size: `connections ≈ (core_count × 2) + effective_spindle_count`. More connections than that reduces throughput — the database becomes context-switch bound.

---

## 15. Migrations

### The expand-contract pattern

Zero-downtime schema change requires the old and new code to both work during rollout.

```
   ┌─ EXPAND ────────────────────────────────────────────────┐
   │ Add the new thing. Both old and new code work.          │
   │ • Add a nullable column                                 │
   │ • Add a new table                                       │
   │ • CREATE INDEX CONCURRENTLY                             │
   └────────────────────┬────────────────────────────────────┘
   ┌─ MIGRATE ──────────▼────────────────────────────────────┐
   │ • Deploy code that writes to BOTH old and new           │
   │ • Backfill existing rows in batches (throttled)         │
   │ • Deploy code that READS from new                       │
   └────────────────────┬────────────────────────────────────┘
   ┌─ CONTRACT ─────────▼────────────────────────────────────┐
   │ • Stop writing to old                                   │
   │ • Verify no reads (log/monitor first!)                  │
   │ • Drop the old column/table                             │
   └─────────────────────────────────────────────────────────┘
```

```sql
-- Backfill in batches — never one giant UPDATE
DO $$
DECLARE batch int := 10000; updated int;
BEGIN
  LOOP
    UPDATE users SET new_col = transform(old_col)
    WHERE id IN (SELECT id FROM users WHERE new_col IS NULL LIMIT batch);
    GET DIAGNOSTICS updated = ROW_COUNT;
    EXIT WHEN updated = 0;
    COMMIT;
    PERFORM pg_sleep(0.1);        -- let replicas catch up, reduce lock pressure
  END LOOP;
END $$;
```

**Migration safety checklist:**

```
   □ lock_timeout set (fail fast rather than queue behind a long query)
   □ statement_timeout set
   □ CONCURRENTLY for index creation
   □ No table rewrites on large tables
   □ Backfills batched with sleeps
   □ Reversible, or a documented forward-fix
   □ Tested against a production-sized dataset
   □ Replica lag monitored during the run
```

---

## 16. Operations

### Backups

```
   3-2-1 rule: 3 copies, 2 different media, 1 offsite.

   • Full backup: pg_basebackup / pg_dump
   • Continuous WAL archiving → Point-In-Time Recovery
   • ⭐ TEST YOUR RESTORES. A backup you've never restored is a hypothesis.
   • Measure and document RTO (how long to recover) and
     RPO (how much data you can lose)
```

### Monitoring

| Metric | Alert when |
|---|---|
| Replication lag | > 10 s (or your read-after-write budget) |
| Connection count | > 80% of max |
| Cache hit ratio | < 0.98 |
| Longest transaction | > 5 min (blocks vacuum!) |
| Dead tuple ratio | > 20% |
| Disk usage | > 75% |
| Transaction ID age | > 1 billion (wraparound risk) |
| Deadlocks/sec | any sustained rate |
| p99 query latency | above your SLO |

---

## 17. Interview Section

<details>
<summary><b>Q1. Explain how a B-tree index works and why databases use it.</b></summary>

A B+tree is a balanced tree where internal nodes hold only keys and child pointers, and all actual data lives in the leaves, which are linked together in sorted order.

The node size matches the disk page size, so one I/O retrieves hundreds of keys. That makes the tree extremely shallow — around four levels for 100 million rows. Since the top levels are always cached, looking up any row is effectively one disk read.

The linked leaves are why range scans and `ORDER BY` are cheap: once you find the start, you walk sequentially rather than re-traversing.

The alternative shapes fail for specific reasons. A binary tree over 100M rows is 27 levels deep, so 27 seeks. A hash index gives O(1) equality but can't do ranges or ordering at all. B-trees give you equality, ranges, ordering, and prefix matching from one structure, which is why they're the default everywhere.
</details>

<details>
<summary><b>Q2. B-tree vs LSM tree.</b></summary>

B-trees update in place, so writes are random I/O with a read-modify-write cycle. LSM trees buffer writes in memory and flush sorted immutable files sequentially, then merge them in the background through compaction.

That makes LSM dramatically better for write-heavy workloads — sequential writes are orders of magnitude faster — at the cost of read amplification, since a read may need to check several files. Bloom filters mitigate that by cheaply ruling out files that definitely don't contain the key.

LSM also compresses better because blocks are immutable and sorted, but it has high write amplification from compaction, and compaction competes with foreground traffic for I/O.

So: B-tree for OLTP and read-heavy work with ad-hoc queries — Postgres, InnoDB. LSM for write-heavy ingest, time-series, and logs — Cassandra, RocksDB. The RUM conjecture frames it generally: you optimize for two of read, update, and memory, never all three.
</details>

<details>
<summary><b>Q3. Walk me through ACID.</b></summary>

Atomicity means all-or-nothing, implemented via the write-ahead log and undo information so a crash mid-transaction rolls back cleanly.

Consistency means the database moves from one valid state to another — but this one is mostly *your* responsibility. The database enforces the constraints you declare; it doesn't know your business invariants unless you express them.

Isolation is the hard one: concurrent transactions shouldn't see each other's intermediate states. It's a spectrum, not a boolean, which is what isolation levels are.

Durability means once committed, it survives a crash. That's the WAL being fsynced before acknowledging the commit.

The mechanism underneath most of it is the WAL: log the intent before modifying the data page. Since the log is a sequential append, only one synchronous write is needed at commit, and the actual data pages can be written lazily in the background.
</details>

<details>
<summary><b>Q4. Explain isolation levels and their anomalies.</b></summary>

Read Uncommitted allows dirty reads — seeing data that may be rolled back. Almost nobody uses it.

Read Committed prevents dirty reads. Each statement sees a snapshot taken at statement start, so you can still get non-repeatable reads and phantoms within a transaction. It's Postgres's default and fine for most CRUD.

Repeatable Read takes one snapshot for the whole transaction. In Postgres this is full snapshot isolation and also prevents phantoms and lost updates, aborting with a serialization error instead. It's MySQL's default, where gap locks provide the phantom protection.

Serializable guarantees the result is equivalent to some serial execution. Postgres implements this optimistically with SSI, detecting dangerous dependency cycles and aborting a transaction — so your application must handle serialization failures and retry.

The anomaly worth knowing beyond the textbook three is write skew: two transactions each read an overlapping set, each writes a disjoint row, both commit, and a cross-row invariant breaks. Snapshot isolation permits it; only Serializable prevents it.
</details>

<details>
<summary><b>Q5. What is MVCC and what does it cost?</b></summary>

Multi-Version Concurrency Control: instead of overwriting a row, writers create a new version stamped with transaction IDs. Each transaction sees the versions that were committed as of its snapshot. So readers never block writers and writers never block readers, which is a massive concurrency win over lock-based approaches.

The cost is garbage. Old versions accumulate and must be reclaimed — VACUUM in Postgres, the undo log in MySQL and Oracle.

The failure mode people hit in production: a long-running transaction pins a snapshot, so vacuum cannot reclaim anything newer than it. Tables bloat, indexes bloat, and query performance degrades across the whole database — from one idle-in-transaction connection. Postgres also has transaction ID wraparound, which forces a shutdown if vacuum falls far enough behind.

The operational rule that follows: never hold a transaction open across user input, network calls, or anything you don't control. And set `idle_in_transaction_session_timeout`.
</details>

<details>
<summary><b>Q6. When would you denormalize?</b></summary>

When a specific read path is measurably too slow because of joins or aggregation, the read-to-write ratio is high, and I have a concrete plan to keep the duplicate consistent.

The most common legitimate cases are counter caches — storing `comment_count` rather than running `COUNT(*)` on a large child table — and materialized views for expensive aggregates.

What I'd push back on is denormalizing preemptively because "joins are slow." A well-indexed join on a modern database is usually fast; the actual problem is more often a missing index, an N+1 pattern, or a bad query plan from stale statistics.

Whenever I denormalize, I write down how the copy stays in sync — trigger, application logic, CDC, or scheduled refresh — and add a reconciliation job, because every denormalized value eventually drifts.
</details>

<details>
<summary><b>Q7. How would you find and fix a slow query?</b></summary>

Start by finding which query actually matters — `pg_stat_statements` sorted by total execution time, not mean, because a 5ms query called a million times matters more than a 2-second report run daily.

Then `EXPLAIN (ANALYZE, BUFFERS)` on that query. I'm looking at three things: the gap between estimated and actual row counts, which reveals stale statistics; where the actual time is concentrated; and the buffer numbers, since `read` means disk I/O while `hit` means cache.

Common findings and fixes: a sequential scan with a selective filter means a missing index. A huge `loops=` count on an inner node is an N+1. `Sort Method: external merge` means `work_mem` is too small. A wildly wrong row estimate means `ANALYZE` or extended statistics for correlated columns.

Then I fix one thing and re-measure — the step people skip. And I'd check whether the query is even necessary, since the fastest query is the one you cached or eliminated.
</details>

<details>
<summary><b>Q8. Explain sharding and its costs.</b></summary>

Sharding splits rows across independent databases so writes scale horizontally. The strategies are range-based, hash-based, consistent hashing, or a directory service. Consistent hashing with virtual nodes is usually the answer because adding a node moves only 1/N of the keys instead of nearly everything.

The costs are what matter in an interview. You lose cross-shard joins, cross-shard transactions, global unique constraints, and global secondary indexes. Aggregations become scatter-gather. Resharding is the hardest operation you'll ever run.

So my strong default is to exhaust everything else first: read replicas, caching, index tuning, connection pooling, archiving cold data, table partitioning within one database, and vertical scaling. A single tuned Postgres instance handles tens of thousands of TPS and multiple terabytes.

When sharding is genuinely needed, the shard key choice is irreversible and dominates everything. It needs high cardinality, even distribution, presence in most queries, and alignment with transaction boundaries. In B2B SaaS, `tenant_id` is usually the natural answer because it's already the isolation unit.
</details>

<details>
<summary><b>Q9. What is replication lag and how do you handle it?</b></summary>

With asynchronous replication, the primary commits and acknowledges before replicas have applied the change. Replicas trail by milliseconds normally, seconds or minutes under load or long-running queries.

The classic bug: a user submits a form, gets redirected to a read that hits a replica, and sees a 404 or stale data — "my order disappeared."

Handling it depends on the guarantee you need. For read-your-writes, either route that user's reads to the primary for a short window after a write, or capture the write's LSN in the session and have the replica wait until it's caught past that point. For anything in a read-after-write flow, just read from the primary.

Operationally, monitor lag and remove replicas from the load balancer above a threshold, so a stalled replica doesn't serve badly stale data. And know that synchronous replication removes the lag but couples your write latency and availability to the replica's health.
</details>

<details>
<summary><b>Q10. How do you do a zero-downtime schema migration?</b></summary>

Expand-contract, in three phases, with a deploy between each.

Expand: add the new structure in a backward-compatible way — a nullable column, a new table, `CREATE INDEX CONCURRENTLY`. Old code keeps working.

Migrate: deploy code that writes to both old and new, backfill existing rows in throttled batches with sleeps so replicas keep up and locks stay short, then deploy code that reads from the new structure.

Contract: stop writing to the old structure, verify nothing reads it — with monitoring, not assumption — then drop it.

The operational details that prevent outages: always set `lock_timeout` before DDL, because `ALTER TABLE` needs an ACCESS EXCLUSIVE lock and will queue behind a long-running query while every new query queues behind it. That's how a "quick migration" becomes a full outage. Never do a single giant `UPDATE`. And test against production-sized data, since a migration that takes 200ms on a dev database can take 40 minutes on 500 million rows.
</details>

<details>
<summary><b>Q11. SQL vs NoSQL — how do you decide?</b></summary>

I start from access patterns and consistency requirements rather than data volume, which is what people usually cite.

Relational is the default. You get ACID transactions, referential integrity, a declarative query language that handles queries you didn't anticipate, and mature tooling. Modern Postgres also does JSON, full-text search, geospatial, and vectors — so "we need flexible schema" often isn't a reason to leave.

I'd reach for NoSQL when there's a specific fit: key-value for caching and sessions; wide-column when write throughput genuinely exceeds what a single primary can take and the access patterns are known and narrow; document when aggregates are always read as a unit; graph when the core queries are multi-hop traversals, which are painful in SQL; search when you need relevance ranking and faceting.

The honest framing is that NoSQL trades query flexibility and transactional guarantees for scale and predictable latency. If you don't need that scale, you're paying the cost without the benefit. And most systems end up polyglot anyway — Postgres for the source of truth, Redis for cache, Elasticsearch for search.
</details>

<details>
<summary><b>Q12. Explain deadlocks and how to prevent them.</b></summary>

A deadlock is a cycle in the wait-for graph: T1 holds A and wants B, T2 holds B and wants A. Neither can proceed, so the database detects the cycle and kills one as the victim.

The primary prevention is acquiring locks in a consistent global order — sort IDs before locking, so two transactions touching the same rows always take them in the same sequence. That makes the cycle impossible.

Beyond that: keep transactions short so the window for overlap is small, use the lowest correct isolation level, and set `lock_timeout` so you fail fast rather than hanging.

Since deadlocks can't be eliminated entirely under concurrency, application code that does multi-row updates should be prepared to retry on deadlock with backoff. And I'd log the deadlock details — Postgres reports both queries involved — because the fix is usually obvious once you see which two code paths take locks in opposite order.
</details>

---

## 18. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        DATABASES — ONE PAGE                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ B-TREE: shallow+wide, page-sized nodes, linked leaves → ranges       ║
║ LSM: memtable→SSTable, sequential writes, compaction, bloom filters  ║
║   B-tree=read-heavy OLTP · LSM=write-heavy/time-series               ║
╠══════════════════════════════════════════════════════════════════════╣
║ INDEX: leftmost-prefix rule. Equality cols FIRST, range col LAST.    ║
║   covering/INCLUDE → index-only scan (no heap access) ⭐              ║
║   partial WHERE · expression LOWER(x) · BRIN for huge time-series    ║
║   killed by: function on column · leading % · type mismatch · OR     ║
║   every index = another write per INSERT/UPDATE                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ EXPLAIN: read inside-out · est vs actual rows (10× gap = ANALYZE)    ║
║   Buffers read=disk · loops=N means N+1 · external merge = work_mem  ║
╠══════════════════════════════════════════════════════════════════════╣
║ ISOLATION  dirty | non-repeatable | phantom | lost upd | write skew  ║
║   ReadCommitted  ❌         ✅         ✅        ✅         ✅        ║
║   RepeatableRead ❌         ❌         ❌*       ❌*        ✅        ║
║   Serializable   ❌         ❌         ❌        ❌         ❌        ║
║   PG default=RC · MySQL default=RR · SERIALIZABLE needs RETRY logic  ║
╠══════════════════════════════════════════════════════════════════════╣
║ MVCC: versions not overwrites. Readers never block writers.          ║
║   ⚠️ LONG TRANSACTIONS BLOCK VACUUM → bloat → everything slows        ║
║   set idle_in_transaction_session_timeout                            ║
║ FOR UPDATE SKIP LOCKED = the job-queue pattern                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ MIGRATIONS: expand → migrate(dual write + batched backfill) →        ║
║   contract. CREATE INDEX CONCURRENTLY. ALWAYS set lock_timeout.      ║
╠══════════════════════════════════════════════════════════════════════╣
║ SCALING ORDER: index → cache → read replicas → pooling →             ║
║   partition → archive → vertical → THEN shard                        ║
║   shard key = high cardinality + even + in most queries (irreversible)║
╠══════════════════════════════════════════════════════════════════════╣
║ ALWAYS: parameterized queries · N+1 checks in tests · test restores  ║
║   monitor: repl lag, longest txn, dead tuples, cache hit, conn count ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [SQL Mastery](../08-data-ai/sql.md) · [Caching](caching.md) · [API Design](api-design.md) · [System Design](../05-system-design/00-fundamentals.md)
