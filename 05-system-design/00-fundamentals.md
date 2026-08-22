# 🏛️ System Design Fundamentals

> Every large system is built from the same handful of ideas. This book is those ideas, explained from first principles — so that when you see Uber's or Netflix's architecture later, you recognize the parts.

---

## 📑 Table of Contents

1. [The Fundamental Constraint](#1-the-fundamental-constraint)
2. [Numbers Every Engineer Must Know](#2-numbers-every-engineer-must-know)
3. [Scaling: Vertical vs Horizontal](#3-scaling)
4. [Load Balancing](#4-load-balancing)
5. [Statelessness](#5-statelessness)
6. [Replication](#6-replication)
7. [Partitioning / Sharding](#7-partitioning--sharding)
8. [CAP and PACELC](#8-cap-and-pacelc)
9. [Consistency Models](#9-consistency-models)
10. [Caching Strategy](#10-caching-strategy)
11. [Asynchronous Processing](#11-asynchronous-processing)
12. [Failure and Resilience](#12-failure-and-resilience)
13. [Back-of-the-Envelope Estimation](#13-estimation)
14. [The Evolution of an Architecture](#14-evolution)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. The Fundamental Constraint

#### 💬 Why distributed systems are hard

Everything difficult about system design traces back to one physical fact: **information cannot travel faster than light, and machines fail independently.**

If you could have one infinitely fast, infinitely large, never-failing computer, system design would not exist. You'd just write a program. Every technique in this book is a workaround for the fact that you can't.

The moment you add a second machine, you inherit an entire category of problems. **So the first rule of system design is: don't distribute until you must.** A single well-tuned PostgreSQL instance handles tens of thousands of transactions per second and terabytes of data. That covers the vast majority of real products.

```mermaid
flowchart TD
    Root["🌐 Two physical facts<br/>1) light has a speed limit<br/>2) machines fail independently"] --> Q{"Do you actually<br/>need a second machine?"}
    Q -->|"No — one box handles it"| One["✅ ONE MACHINE<br/><b>No distributed-systems problems</b>"]
    Q -->|"Yes — load or durability<br/>demands it"| Many["⚠️ MANY MACHINES<br/><b>Partial failure becomes normal</b>"]

    One --> O1["One source of truth"]
    One --> O2["Instant 'reads see writes'"]
    One --> O3["Real transactions"]
    One --> O4["It's up, or it's down —<br/>no ambiguous middle state"]
    One --> O5["🔴 Limited by one box's capacity<br/>🔴 One failure = total outage"]

    Many --> M1["❌ Which copy is correct?"]
    Many --> M2["❌ Replicas lag behind"]
    Many --> M3["❌ Cross-machine transactions<br/>are painful and slow"]
    Many --> M4["❌ Partial failure: half up,<br/>half down, and they disagree<br/>about what happened"]

    style Root fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style Q fill:#e1f5fe,stroke:#0277bd,color:#000
    style One fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style Many fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style O1 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style O2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style O3 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style O4 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style O5 fill:#ffcdd2,stroke:#c62828,color:#000
    style M1 fill:#ffcdd2,stroke:#c62828,color:#000
    style M2 fill:#ffcdd2,stroke:#c62828,color:#000
    style M3 fill:#ffcdd2,stroke:#c62828,color:#000
    style M4 fill:#ffcdd2,stroke:#c62828,color:#000
```

---

## 2. Numbers Every Engineer Must Know

You cannot design a system without a feel for magnitudes. Memorize the shape of this table, not the exact digits.

```
   LATENCY (Jeff Dean's numbers, updated)

   L1 cache reference                        0.5 ns    ┐
   Branch mispredict                           5 ns    │ CPU
   L2 cache reference                          7 ns    │
   Mutex lock/unlock                          25 ns    ┘
   Main memory reference                     100 ns    ← RAM
   Compress 1 KB with Zippy                3,000 ns
   Send 1 KB over 1 Gbps network          10,000 ns  = 10 µs
   Read 4 KB randomly from SSD           150,000 ns  = 150 µs   ← SSD
   Read 1 MB sequentially from memory    250,000 ns  = 250 µs
   Round trip within the same datacenter 500,000 ns  = 500 µs   ← LAN
   Read 1 MB sequentially from SSD     1,000,000 ns  = 1 ms
   Disk seek (spinning)               10,000,000 ns  = 10 ms    ← HDD
   Read 1 MB sequentially from disk   20,000,000 ns  = 20 ms
   Send packet CA → Netherlands → CA 150,000,000 ns  = 150 ms   ← WAN
```

#### 📊 Scaled to human time

This is the version that actually builds intuition. Imagine 1 nanosecond = 1 second:

```mermaid
flowchart LR
    L1["🔥 L1 cache<br/><b>0.5 ns</b><br/>grab it off your desk"] --> RAM["🟢 RAM<br/><b>100 ns</b><br/>~2 min walk to<br/>the filing cabinet"]
    RAM --> SSD["🟡 SSD read<br/><b>150 µs</b><br/>~2 days, order &amp; wait"]
    SSD --> DC["🟡 Datacenter round trip<br/><b>500 µs</b><br/>~6 days, ship across town"]
    DC --> Disk["🔴 Disk seek (HDD)<br/><b>10 ms</b><br/>~4 months, wait a season"]
    Disk --> WAN["🔴 CA → Netherlands<br/><b>150 ms</b><br/>~5 years, a college degree"]

    style L1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style RAM fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style SSD fill:#fff9c4,stroke:#f9a825,color:#000
    style DC fill:#fff9c4,stroke:#f9a825,color:#000
    style Disk fill:#ffcdd2,stroke:#c62828,color:#000
    style WAN fill:#ffcdd2,stroke:#c62828,stroke-width:3px,color:#000
```

**What this tells you to do:**
- Keep hot data in memory. RAM is ~1,500× faster than SSD.
- Batch network calls. One round trip costs the same as ~5,000 memory reads.
- Sequential beats random, everywhere, by a lot.
- Cross-region calls are catastrophic in a request path. Replicate data instead.

### Capacity numbers worth knowing

```
   ONE MODERN SERVER (roughly)
   ─────────────────────────────
   Requests/sec (simple API)       ~10,000 - 50,000
   Requests/sec (heavy work)       ~500 - 2,000
   PostgreSQL writes/sec           ~5,000 - 20,000 (with good hardware)
   Redis operations/sec            ~100,000+
   Kafka messages/sec per broker   ~100,000 - 1,000,000
   Concurrent connections          ~10,000 - 100,000 (with async I/O)

   TIME
   ─────────────────────────────
   1 day       = 86,400 sec ≈ 10⁵
   1 month     = 2.6M sec   ≈ 2.5 × 10⁶
   1 year      = 31.5M sec  ≈ 3 × 10⁷

   ⭐ Handy shortcut: 1 million requests/day ≈ 12 requests/sec
```

### Data size reference

```
   char / bool          1 byte
   int / float          4 bytes
   long / double        8 bytes
   UUID                 16 bytes
   Timestamp            8 bytes
   Short text (name)    ~20-50 bytes
   URL                  ~100 bytes
   Tweet                ~300 bytes
   JSON API response    ~1-10 KB
   Web page (HTML)      ~50-100 KB
   Photo (compressed)   ~200 KB - 2 MB
   Song (MP3)           ~3-5 MB
   Movie (1080p)        ~1-3 GB
```

---

## 3. Scaling

```mermaid
flowchart LR
    subgraph Vertical["⬆️ VERTICAL — scale up"]
        direction TB
        Big["🖥️ ONE big box<br/>more CPU, RAM, disk"]
        VPro["✅ Simple — no code changes<br/>✅ No distributed-systems problems<br/>✅ Real transactions still work"]
        VCon["❌ Hard ceiling on machine size<br/>❌ Cost grows super-linearly<br/>❌ Single point of failure<br/>❌ Downtime to upgrade"]
        Big --- VPro
        Big --- VCon
    end
    subgraph Horizontal["➡️ HORIZONTAL — scale out"]
        direction TB
        Many["🖥️🖥️🖥️🖥️ MANY small boxes"]
        HPro["✅ Nearly unlimited ceiling<br/>✅ Fault tolerant — lose one, survive<br/>✅ Cheaper per unit of capacity<br/>✅ Can scale incrementally"]
        HCon["❌ Complexity explodes<br/>❌ Consistency becomes hard<br/>❌ Needs statelessness + coordination"]
        Many --- HPro
        Many --- HCon
    end

    style Vertical fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style Horizontal fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style Big fill:#fff,stroke:#f9a825,stroke-width:2px,color:#000
    style Many fill:#fff,stroke:#2e7d32,stroke-width:2px,color:#000
    style VPro fill:#fff9c4,stroke:#f9a825,color:#000
    style VCon fill:#ffcdd2,stroke:#c62828,color:#000
    style HPro fill:#c8e6c9,stroke:#2e7d32,color:#000
    style HCon fill:#ffcdd2,stroke:#c62828,color:#000
```

🏭 **The practical order:** vertical scaling first (it's cheap and buys years), then read replicas, then caching, then horizontal scaling of stateless services, and only finally sharding the database. People reverse this order and pay dearly.

```mermaid
flowchart TD
    A["1️⃣ Vertical scaling<br/><b>Bigger box</b><br/>cheap, buys years, no code change"] --> B["2️⃣ Read replicas<br/><b>Scale reads</b>"]
    B --> C["3️⃣ Caching<br/><b>Redis in front of the DB</b>"]
    C --> D["4️⃣ Horizontal scaling<br/><b>of stateless app servers</b>"]
    D --> E["5️⃣ Shard the database<br/><b>LAST resort — irreversible</b>"]

    style A fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style B fill:#c8e6c9,stroke:#2e7d32,color:#000
    style C fill:#c8e6c9,stroke:#2e7d32,color:#000
    style D fill:#fff9c4,stroke:#f9a825,color:#000
    style E fill:#ffcdd2,stroke:#c62828,stroke-width:3px,color:#000
```

---

## 4. Load Balancing

#### 💬 What it actually does
A load balancer sits in front of your servers and decides where each request goes. It also does health checking — if a server stops responding, it's removed from rotation automatically.

```mermaid
flowchart TD
    Client(["Client request"]) --> LB{"⚖️ Load Balancer<br/>health checks all backends"}
    LB -->|"routes"| A1["App1 ✅ healthy"]
    LB -->|"routes"| A2["App2 ✅ healthy"]
    LB -.->|"❌ removed<br/>from rotation"| A3["App3 — failing<br/>health check"]

    style Client fill:#e1f5fe,stroke:#0277bd,color:#000
    style LB fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style A1 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style A2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style A3 fill:#ffcdd2,stroke:#c62828,color:#000
```

### Layer 4 vs Layer 7

```mermaid
flowchart LR
    Req(["Incoming request"]) --> Choice{"Which layer?"}
    Choice -->|"Just need speed,<br/>no content awareness"| L4["🟡 L4 — Transport<br/>routes on IP + port<br/><b>very fast, low overhead</b><br/>no path-based routing<br/>TLS passes through untouched"]
    Choice -->|"Need to route on<br/>URL / headers / cookies"| L7["🟢 L7 — Application<br/>understands HTTP<br/><b>TLS termination, WAF,<br/>path routing</b><br/>slower — must parse request"]
    L4 --> L4ex["Examples:<br/>AWS NLB, IPVS"]
    L7 --> L7ex["Examples:<br/>nginx, ALB, Envoy, HAProxy"]

    style Req fill:#e1f5fe,stroke:#0277bd,color:#000
    style Choice fill:#e1f5fe,stroke:#0277bd,color:#000
    style L4 fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style L7 fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style L4ex fill:#fff,stroke:#f9a825,color:#000
    style L7ex fill:#fff,stroke:#2e7d32,color:#000
```

### Algorithms

| Algorithm | How it picks | Best for |
|---|---|---|
| **Round robin** | Next server in rotation | Uniform servers, uniform requests |
| **Weighted round robin** | Proportional to capacity | Mixed server sizes |
| **Least connections** | Fewest active connections | Long-lived / variable-duration requests ⭐ |
| **Least response time** | Fastest recent responses | Latency-sensitive |
| **IP hash** | `hash(client IP)` | Sticky sessions without cookies |
| **Consistent hashing** | Hash ring | Cache affinity — see below ⭐ |

#### 💬 Why consistent hashing matters

Suppose you route cache requests with `hash(key) % N`. Add one server and `N` changes — so **almost every key** now maps somewhere new, and your entire cache misses at once. That's a stampede that can take down your database.

Consistent hashing fixes this by placing servers on a ring: each key hashes to a point, then walks **clockwise** to the first node it finds. Adding `nodeD` between `nodeA` and `nodeB` only moves the keys that fall between them — about 1/N of all keys, not all of them.

```mermaid
flowchart TD
    Naive["🔴 hash(key) % N<br/>Add/remove ONE server<br/>→ almost EVERY key remaps<br/>→ cache stampede hits the DB"] -->|"the problem"| Ring["🟡 Hash ring<br/>servers placed at points<br/>on a circle; key walks<br/>clockwise to first node"]
    Ring -->|"add nodeD"| Better["🟢 Consistent hashing<br/>only keys between the<br/>neighbors move — ~1/N<br/>of all keys, not all of them"]
    Better -->|"uneven load risk"| Virtual["🟢 + Virtual nodes<br/>~150 points per physical server<br/><b>evens distribution, spreads<br/>a leaving node's keys across<br/>ALL survivors</b>"]

    style Naive fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style Ring fill:#fff9c4,stroke:#f9a825,color:#000
    style Better fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style Virtual fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
```

---

## 5. Statelessness

#### 💬 Why this is the single most important architectural property

A **stateless** server stores nothing about a client between requests. Any server can handle any request. That one property is what makes horizontal scaling possible.

```mermaid
flowchart TD
    subgraph Stateful["🔴 STATEFUL — sticky sessions"]
        direction LR
        C1["Client"] -->|"MUST come back<br/>to App2"| S2["App2<br/>(session in memory)"]
        S1b["App1"]
        S3b["App3"]
        SC1["Consequences:<br/>App2 dies → users logged out<br/>Uneven load<br/>Can't deploy without disruption<br/>Autoscaling is unsafe"]
    end
    subgraph Stateless["🟢 STATELESS — any server, any request"]
        direction LR
        C2["Client"] --> S1["App1"]
        C2 --> S2b["App2"]
        C2 --> S3["App3"]
        S1 --> Redis[("Redis<br/>session store")]
        S2b --> Redis
        S3 --> Redis
        SC2["Consequences:<br/>Any server, any request<br/>Kill any box freely<br/>Autoscale without thought"]
    end

    style Stateful fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style Stateless fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style C1 fill:#fff,stroke:#c62828,color:#000
    style S2 fill:#fff,stroke:#c62828,color:#000
    style S1b fill:#fff,stroke:#c62828,color:#000
    style S3b fill:#fff,stroke:#c62828,color:#000
    style SC1 fill:#ffcdd2,stroke:#c62828,color:#000
    style C2 fill:#fff,stroke:#2e7d32,color:#000
    style S1 fill:#fff,stroke:#2e7d32,color:#000
    style S2b fill:#fff,stroke:#2e7d32,color:#000
    style S3 fill:#fff,stroke:#2e7d32,color:#000
    style SC2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Redis fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
```

**Where state should actually live:**

```mermaid
flowchart LR
    Session["Session data"] -->|"lives in"| Redis2["Redis, or a signed<br/>JWT the client carries"]
    UserData["User data"] -->|"lives in"| DB2[("Database")]
    Files["Uploaded files"] -->|"lives in"| S3["Object storage (S3)<br/><b>never local disk</b>"]
    Computation["Cached computation"] -->|"lives in"| Redis3["Redis / Memcached"]
    InFlight["In-flight work"] -->|"lives in"| Queue["Message queue"]

    style Session fill:#e1f5fe,stroke:#0277bd,color:#000
    style UserData fill:#e1f5fe,stroke:#0277bd,color:#000
    style Files fill:#e1f5fe,stroke:#0277bd,color:#000
    style Computation fill:#e1f5fe,stroke:#0277bd,color:#000
    style InFlight fill:#e1f5fe,stroke:#0277bd,color:#000
    style Redis2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style DB2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style S3 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Redis3 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Queue fill:#c8e6c9,stroke:#2e7d32,color:#000
```

---

## 6. Replication

#### 💬 Two reasons to replicate
1. **Availability** — a copy survives when the primary dies.
2. **Read scaling** — spread reads across many machines.

```mermaid
flowchart LR
    P[("🔵 PRIMARY<br/>handles all writes")] -->|"replication stream"| R1[("REPLICA<br/>reads only")]
    P -->|"replication stream"| R2[("REPLICA<br/>reads only")]
    Clients(["Write traffic"]) --> P
    Readers(["Read traffic"]) --> R1
    Readers --> R2

    style P fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style R1 fill:#e1f5fe,stroke:#0277bd,color:#000
    style R2 fill:#e1f5fe,stroke:#0277bd,color:#000
    style Clients fill:#fff9c4,stroke:#f9a825,color:#000
    style Readers fill:#fff9c4,stroke:#f9a825,color:#000
```

### Synchronous vs asynchronous

```mermaid
flowchart LR
    Choice{"Can you tolerate<br/>losing recent writes<br/>on failover?"}
    Choice -->|"Yes, need speed"| Async["🟡 ASYNCHRONOUS<br/>ack client immediately<br/>ship to replicas later<br/><b>fast, but a lag window<br/>can lose data</b>"]
    Choice -->|"No, zero data loss<br/>is required"| Sync["🟢 SYNCHRONOUS<br/>wait for N replicas<br/>to confirm, THEN ack<br/><b>safe, but write latency<br/>= slowest replica</b>"]

    style Choice fill:#e1f5fe,stroke:#0277bd,color:#000
    style Async fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style Sync fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
```

#### ⚠️ The replication lag bug you will absolutely hit

**The fixes:**
- **Read-your-writes** — route that user's reads to the primary for N seconds after they write
- Track the write position (LSN/GTID) in the session and make the replica wait until it has caught up past it
- Just read from the primary in read-after-write flows
- Monitor lag; pull replicas out of rotation above a threshold

```mermaid
sequenceDiagram
    participant U as User
    participant P as Primary
    participant R as Replica (200ms behind)

    U->>P: 1. Submit form (write)
    P-->>U: ack
    U->>R: 2. Redirect, read detail page
    R-->>U: 3. 404 Not Found ❌ (hasn't replicated yet)
    Note over U,R: 😱 User: "my order disappeared!"
    Note over U,R: FIX 1: route this user's reads to the PRIMARY for N seconds after a write<br/>FIX 2: track write position (LSN/GTID), make replica wait until caught up<br/>FIX 3: monitor lag, pull replicas out of rotation above a threshold
```

---

## 7. Partitioning / Sharding

#### 💬 What sharding is
Replication copies *the same data* to many machines. Sharding splits *different data* across machines. Replication scales reads; **sharding is the only thing that scales writes.**

```mermaid
flowchart TD
    Data[("Full dataset")] --> Rep["🔵 REPLICATION<br/>same data copied<br/>to every node<br/><b>scales READS</b>"]
    Data --> Shard["🔵 SHARDING<br/>different data<br/>split across nodes<br/><b>scales WRITES + storage</b>"]

    style Data fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style Rep fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style Shard fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
```

### Strategies

```mermaid
flowchart TD
    Range["🔴 RANGE-BASED<br/>users 1-1M → A, 1M-2M → B<br/><b>hotspots: newest shard<br/>takes all new writes</b>"]
    Hash["🟡 HASH-BASED<br/>hash(user_id) % N<br/><b>even, but range queries<br/>impossible; resharding<br/>moves ~everything</b>"]
    Consistent["🟢 CONSISTENT HASH<br/>ring + virtual nodes<br/><b>adding a node moves<br/>only 1/N of keys</b><br/>← the standard answer"]
    Directory["🟡 DIRECTORY<br/>lookup service maps<br/>key → shard<br/><b>flexible, but the<br/>directory is a SPOF</b>"]
    Geo["🟡 GEOGRAPHIC<br/>shard by region<br/><b>compliance + latency win,<br/>cross-region queries hurt</b>"]

    style Range fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style Hash fill:#fff9c4,stroke:#f9a825,color:#000
    style Consistent fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style Directory fill:#fff9c4,stroke:#f9a825,color:#000
    style Geo fill:#fff9c4,stroke:#f9a825,color:#000
```

### ⚠️ What sharding costs you

```mermaid
flowchart LR
    J["❌ Cross-shard JOINs"] -->|"fix"| Jf["Denormalize, or<br/>join in the app"]
    T["❌ Cross-shard TRANSACTIONS"] -->|"fix"| Tf["Sagas, or<br/>avoid entirely"]
    U["❌ Global unique constraints"] -->|"fix"| Uf["Central ID service,<br/>or UUIDs"]
    I["❌ Global secondary indexes"] -->|"fix"| If["A separate<br/>search cluster"]
    A["❌ Aggregations over all data"] -->|"fix"| Af["Scatter-gather, or a<br/>separate OLAP store"]
    R["❌ RESHARDING"] -->|"fix"| Rf["🔴 The hardest operation<br/>you will ever run —<br/>no shortcut"]

    style J fill:#ffcdd2,stroke:#c62828,color:#000
    style T fill:#ffcdd2,stroke:#c62828,color:#000
    style U fill:#ffcdd2,stroke:#c62828,color:#000
    style I fill:#ffcdd2,stroke:#c62828,color:#000
    style A fill:#ffcdd2,stroke:#c62828,color:#000
    style R fill:#ffcdd2,stroke:#c62828,stroke-width:3px,color:#000
    style Jf fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Tf fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Uf fill:#c8e6c9,stroke:#2e7d32,color:#000
    style If fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Af fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Rf fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
```

🏭 **Choosing the shard key is irreversible.** It needs high cardinality, even distribution (watch out for celebrity users), presence in most queries (or every query becomes scatter-gather), and alignment with your transaction boundaries. In B2B SaaS, `tenant_id` is usually the natural answer.

---

## 8. CAP and PACELC

#### 💬 CAP stated correctly

CAP is widely misquoted as "pick two of three." That's wrong. Partitions are not something you choose — they *happen*, because networks fail. The real statement is:

> **When a network partition occurs, you must choose between consistency and availability.**

```mermaid
flowchart TD
    Split["✂️ Network partition splits the cluster<br/>Node A and Node B can no longer<br/>exchange messages"] --> P["A write arrives at Node A.<br/>Node A cannot reach Node B."]
    P --> Choice{"What do you do?<br/>ONLY two options exist."}
    Choice -->|"Refuse it, return<br/>an error"| CP["🟢 CP — CONSISTENCY<br/>Refuse the write (or read)<br/>'I'd rather be down<br/>than wrong.'<br/><b>Banking, inventory, locks<br/>etcd, ZooKeeper, HBase</b>"]
    Choice -->|"Accept locally,<br/>reconcile later"| AP["🟡 AP — AVAILABILITY<br/>Accept the write locally<br/>'I'd rather be wrong for<br/>a moment than unavailable.'<br/><b>Social feeds, carts, DNS<br/>Cassandra, DynamoDB</b>"]

    style Split fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style P fill:#e1f5fe,stroke:#0277bd,color:#000
    style Choice fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style CP fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style AP fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
```

#### PACELC — the more useful version

`IF (Partition) → choose Availability or Consistency` · `ELSE → choose Latency or Consistency`

⭐ The "else" branch is what you actually live with day to day. Partitions are rare. The latency-vs-consistency tradeoff is present in **every request**.

```mermaid
flowchart TD
    Start{"Is the network<br/>partitioned right now?"}
    Start -->|"Yes (rare)"| PC{"choose"}
    PC -->|"Availability"| PA["PA — stay up,<br/>may be inconsistent"]
    PC -->|"Consistency"| PCn["PC — refuse rather<br/>than be wrong"]
    Start -->|"No (the normal case,<br/>EVERY request)"| EL{"choose"}
    EL -->|"Latency"| ELn["🟢 EL — respond fast,<br/>maybe stale"]
    EL -->|"Consistency"| ECn["🟡 EC — always consistent,<br/>accept the latency"]

    ELn --> DynamoDB["DynamoDB = PA/EL<br/>available under partition,<br/>low latency otherwise"]
    ELn --> Cassandra["Cassandra = PA/EL<br/>(tunable per query)"]
    ECn --> Postgres["PostgreSQL = PC/EC<br/>consistent always,<br/>accepts the latency"]
    ECn --> Mongo["MongoDB = PC/EC<br/>(tunable)"]

    style Start fill:#e1f5fe,stroke:#0277bd,color:#000
    style PC fill:#e1f5fe,stroke:#0277bd,color:#000
    style EL fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style PA fill:#fff9c4,stroke:#f9a825,color:#000
    style PCn fill:#fff9c4,stroke:#f9a825,color:#000
    style ELn fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style ECn fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style DynamoDB fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Cassandra fill:#c8e6c9,stroke:#2e7d32,color:#000
    style Postgres fill:#fff9c4,stroke:#f9a825,color:#000
    style Mongo fill:#fff9c4,stroke:#f9a825,color:#000
```

---

## 9. Consistency Models

#### 💬 A spectrum, not a switch

```
**Strongest (slow, expensive) → weakest (fast, cheap):**

```mermaid
flowchart TD
    Lin["🔴 LINEARIZABLE<br/>every read sees the most<br/>recent write, globally, instantly<br/>system behaves as if there's<br/>ONE copy<br/><b>cost: coordination on every op</b><br/>locks, leader election, ledgers"]
    Seq["Sequential<br/>all nodes see operations in<br/>the SAME order, but that order<br/>may not match real time"]
    Causal["Causal<br/>causally-related ops seen in<br/>order by everyone; concurrent<br/>ops may be seen in any order<br/>comment threads — a reply never<br/>appears before its post"]
    RYW["Read-your-writes<br/>you always see your own writes;<br/>others may lag<br/>profile edits, posting —<br/>the minimum users expect"]
    Mono["Monotonic reads<br/>time never goes backwards<br/>for a single user"]
    Event["🟢 EVENTUAL<br/>if writes stop, all replicas<br/>eventually converge<br/><b>no guarantee about when</b><br/>view counts, recs, DNS"]

    Lin -->|"relax:<br/>drop real-time order"| Seq
    Seq -->|"relax:<br/>only order causal ops"| Causal
    Causal -->|"relax:<br/>only guarantee own writes"| RYW
    RYW -->|"relax:<br/>drop write visibility"| Mono
    Mono -->|"relax:<br/>drop time-ordering too"| Event

    style Lin fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style Seq fill:#ffe0b2,stroke:#ef6c00,color:#000
    style Causal fill:#fff9c4,stroke:#f9a825,color:#000
    style RYW fill:#fff9c4,stroke:#f9a825,color:#000
    style Mono fill:#e1f5fe,stroke:#0277bd,color:#000
    style Event fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
```

🎤 **The interview move:** never say "we'll use eventual consistency" without saying *what the user-visible consequence is*. Say instead: *"Likes can be eventually consistent — a user seeing 41 instead of 42 for a few seconds is fine. But the account balance must be strongly consistent, because showing a stale balance could let someone overdraw."*

---

## 10. Caching Strategy

Rule: cache as close to the user as the freshness budget allows.

```mermaid
flowchart LR
    Browser["Browser<br/>~0 ms<br/>per user"] --> CDN["CDN / Edge<br/>~10-30 ms<br/>global, shared"]
    CDN --> LBc["Load balancer<br/>~1-5 ms<br/>per datacenter"]
    LBc --> AppMem["App in-memory<br/>~0.001 ms<br/>⚠️ NOT shared"]
    AppMem --> Redis["Redis<br/>~0.5-2 ms<br/>shared across app tier"]
    Redis --> DBBuf["DB buffer pool<br/>~0.1 ms<br/>inside the database"]
    DBBuf --> Disk["Disk<br/>~100 µs - 10 ms"]

    style Browser fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style CDN fill:#c8e6c9,stroke:#2e7d32,color:#000
    style LBc fill:#fff9c4,stroke:#f9a825,color:#000
    style AppMem fill:#ffcdd2,stroke:#c62828,color:#000
    style Redis fill:#fff9c4,stroke:#f9a825,color:#000
    style DBBuf fill:#fff9c4,stroke:#f9a825,color:#000
    style Disk fill:#ffcdd2,stroke:#c62828,color:#000
```

The three failure modes you must be able to name — stampede, penetration, avalanche — plus their fixes, are covered in depth in **[Caching](../03-backend/caching.md)**.

**The one-line version:**
- **Stampede** — a hot key expires and 1,000 requests hit the DB at once. Fix: TTL jitter + probabilistic early refresh, or a lock with stale-serving.
- **Penetration** — requests for keys that don't exist bypass the cache entirely. Fix: cache negatives, or a bloom filter.
- **Avalanche** — mass expiry or the cache dying takes out the DB. Fix: jitter, multi-level cache, circuit breaker, and **serve stale on error**.

```mermaid
stateDiagram-v2
    [*] --> CacheCheck: request arrives
    CacheCheck --> Hit: key found
    CacheCheck --> Miss: key absent
    Hit --> [*]: return cached value
    Miss --> DBRead: read from DB
    DBRead --> Populate: write result to cache
    Populate --> [*]: return value

    note right of Miss
        STAMPEDE risk: many requests
        miss the same hot key at once
        FIX: TTL jitter, lock + stale-serve
    end note
    note right of DBRead
        PENETRATION risk: key never
        exists in DB either
        FIX: cache negatives / bloom filter
    end note
```

---

## 11. Asynchronous Processing

#### 💬 The core question
For every operation ask: **does the user need this finished before I respond?**

```mermaid
flowchart LR
    Q{"Does the user need<br/>this finished before<br/>I respond?"}
    Q -->|"Yes"| Sync["🔴 SYNCHRONOUS<br/>user waits"]
    Q -->|"No"| Async["🟢 ASYNCHRONOUS<br/>user doesn't wait"]

    Sync --> S1["Payment authorization"]
    Sync --> S2["Login"]
    Sync --> S3["Reading their own data"]
    Sync --> S4["Anything they'll<br/>see immediately"]

    Async --> A1["Sending confirmation email"]
    Async --> A2["Generating a thumbnail"]
    Async --> A3["Updating analytics"]
    Async --> A4["Notifying the warehouse"]
    Async --> A5["Recomputing recommendations"]
    Async --> A6["Indexing for search"]

    style Q fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style Sync fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
    style Async fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style S1 fill:#fff,stroke:#c62828,color:#000
    style S2 fill:#fff,stroke:#c62828,color:#000
    style S3 fill:#fff,stroke:#c62828,color:#000
    style S4 fill:#fff,stroke:#c62828,color:#000
    style A1 fill:#fff,stroke:#2e7d32,color:#000
    style A2 fill:#fff,stroke:#2e7d32,color:#000
    style A3 fill:#fff,stroke:#2e7d32,color:#000
    style A4 fill:#fff,stroke:#2e7d32,color:#000
    style A5 fill:#fff,stroke:#2e7d32,color:#000
    style A6 fill:#fff,stroke:#2e7d32,color:#000
```

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant Q as Queue
    participant W as Worker

    C->>A: request
    A->>Q: enqueue job
    A-->>C: 200 OK (~50 ms)
    Note over C,A: user doesn't wait
    Q->>W: deliver job
    W->>W: process (~5 sec)
    Note over W: traffic spikes are BUFFERED,<br/>not shed; worker downtime<br/>delays work, doesn't lose it
    Note over C,W: ✅ BUYS YOU: fast API responses · independent scaling<br/>of API/workers · add consumers without touching the producer
    Note over C,W: ❌ COSTS: eventual consistency · duplicates (at-least-once<br/>is the practical default) · debugging spans processes/time ·<br/>backpressure must be designed, not assumed
```

⭐ **The outbox pattern** is the single most important thing to know here: you cannot atomically write to your database *and* publish to a queue. Write both to the database in one transaction, and have a relay publish from the outbox table. Full detail in [Queues & Streaming](../03-backend/queues-streaming.md#10-outbox-pattern).

---

## 12. Failure and Resilience

#### 💬 The mindset shift
At small scale, failure is an exception. At scale, **failure is constant**. With 10,000 servers and a 3-year MTBF, something is failing roughly every 2.6 hours. Design for it as the normal case.

### The resilience toolkit

```mermaid
flowchart TD
    Toolkit["🧰 RESILIENCE TOOLKIT"]
    Toolkit --> TO["⏱️ TIMEOUT<br/>never wait forever<br/>a missing timeout is how ONE slow<br/>dependency exhausts your entire<br/>thread pool<br/><b>⭐ callers must time out FASTER<br/>than the callee's own limit</b>"]
    Toolkit --> RE["🔁 RETRY<br/>exponential backoff + JITTER<br/>without jitter, clients retry in<br/>lockstep and hammer the<br/>recovering service ('thundering herd')<br/><b>⚠️ only retry IDEMPOTENT ops</b>"]
    Toolkit --> CB["⚡ CIRCUIT BREAKER<br/>stop calling a service that's<br/>clearly down<br/><b>turns a slow cascading failure<br/>into a FAST, contained one</b><br/>(state machine diagrammed below)"]
    Toolkit --> BH["🚧 BULKHEAD<br/>isolate resource pools<br/>separate thread/connection pools<br/>per dependency, so one saturated<br/>downstream can't consume everything"]
    Toolkit --> RL["🚦 RATE LIMIT<br/>protect yourself from<br/>clients AND yourself"]
    Toolkit --> GD["🩹 GRACEFUL DEGRADATION<br/>serve something rather than nothing<br/>recs down → static list<br/>cache down → serve stale<br/>search down → browse categories"]

    style Toolkit fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
    style TO fill:#c8e6c9,stroke:#2e7d32,color:#000
    style RE fill:#c8e6c9,stroke:#2e7d32,color:#000
    style CB fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style BH fill:#c8e6c9,stroke:#2e7d32,color:#000
    style RL fill:#c8e6c9,stroke:#2e7d32,color:#000
    style GD fill:#c8e6c9,stroke:#2e7d32,color:#000
```

```mermaid
stateDiagram-v2
    [*] --> CLOSED
    CLOSED --> OPEN: failures exceed threshold
    OPEN --> HALF_OPEN: after a cooldown
    HALF_OPEN --> CLOSED: probe succeeds
    HALF_OPEN --> OPEN: probe fails

    note right of CLOSED
        🟢 normal — requests pass through
    end note
    note right of OPEN
        🔴 fail fast — no calls reach
        the downstream service
    end note
    note right of HALF_OPEN
        🟡 let ONE probe request through
        to test recovery
    end note
```

#### ⚠️ Cascading failure — how outages actually happen

A slow dependency (bad deploy, slow query) doesn't stay contained — it propagates upstream, one exhausted thread pool at a time, until the whole system is down.

```mermaid
flowchart LR
    C["Service C<br/>slows down 🐌"] -->|"calls pile up,<br/>threads block"| B["Service B<br/>thread pool exhausts"]
    B -->|"calls pile up"| A["Service A<br/>exhausts too"]
    A --> Down["🔴 Whole system down —<br/>caused by ONE slow dependency"]

    TO["✅ Timeout"] -.->|"breaks link 1"| C
    CB["✅ Circuit breaker"] -.->|"breaks link 2"| B
    BH["✅ Bulkhead"] -.->|"breaks link 3"| A
    LS["✅ Load shedding"] -.->|"rejects early"| Down

    style C fill:#ffcdd2,stroke:#c62828,color:#000
    style B fill:#ffcdd2,stroke:#c62828,color:#000
    style A fill:#ffcdd2,stroke:#c62828,color:#000
    style Down fill:#ffcdd2,stroke:#c62828,stroke-width:3px,color:#000
    style TO fill:#c8e6c9,stroke:#2e7d32,color:#000
    style CB fill:#c8e6c9,stroke:#2e7d32,color:#000
    style BH fill:#c8e6c9,stroke:#2e7d32,color:#000
    style LS fill:#c8e6c9,stroke:#2e7d32,color:#000
```

### Single points of failure

```
   Walk your architecture diagram and ask of EVERY box:
   "if this dies right now, what happens?"

   Common SPOFs people miss:
   □ The load balancer itself
   □ The primary database (do you have automatic failover? TESTED?)
   □ The cache (does the DB survive full traffic?)
   □ DNS
   □ The service discovery registry
   □ The CI/CD system (can you deploy a fix during an outage?)
   □ A single availability zone
   □ Your certificate expiring
```

---

## 13. Estimation

#### 💬 Why interviewers ask for this
Not because the numbers matter. Because it shows whether you can reason about magnitudes — and whether your design is even in the right ballpark. A design that needs 500 servers and a design that needs 5 look completely different.

### The template

```
   ┌─ 1. USERS ────────────────────────────────────────────────┐
   │  DAU (daily active users)                                 │
   │  Actions per user per day                                 │
   ├─ 2. QPS ──────────────────────────────────────────────────┤
   │  Average QPS = (DAU × actions) / 86,400                    │
   │  Peak QPS    = average × 2 to 10   ⭐ always state this     │
   ├─ 3. STORAGE ──────────────────────────────────────────────┤
   │  Bytes per item × items per day × retention               │
   │  × replication factor (usually 3)                          │
   ├─ 4. BANDWIDTH ────────────────────────────────────────────┤
   │  QPS × payload size                                        │
   ├─ 5. MEMORY (cache) ───────────────────────────────────────┤
   │  Apply the 80/20 rule: cache the hot 20%                   │
   └───────────────────────────────────────────────────────────┘
```

```mermaid
flowchart TD
    Users["1️⃣ USERS<br/>DAU × actions/day"] --> QPS["2️⃣ QPS<br/>avg = DAU×actions / 86,400<br/>peak = avg × 2-10"]
    QPS --> Storage["3️⃣ STORAGE<br/>bytes/item × items/day<br/>× retention × replication"]
    Storage --> BW["4️⃣ BANDWIDTH<br/>QPS × payload size"]
    BW --> Mem["5️⃣ MEMORY (cache)<br/>80/20 rule — cache the hot 20%"]

    style Users fill:#e1f5fe,stroke:#0277bd,color:#000
    style QPS fill:#e1f5fe,stroke:#0277bd,color:#000
    style Storage fill:#e1f5fe,stroke:#0277bd,color:#000
    style BW fill:#e1f5fe,stroke:#0277bd,color:#000
    style Mem fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#000
```

### 📊 Worked example — a Twitter-like service

```
   ASSUMPTIONS (state them out loud, and round aggressively)
   • 300M DAU
   • Each user reads 100 tweets/day, posts 0.1 tweets/day
   • Tweet = 300 bytes text + metadata ≈ 1 KB stored
   • 10% of tweets include an image (~200 KB)

   ─── QPS ───────────────────────────────────────────────
   Reads:  300M × 100 / 86,400  ≈  350,000 QPS  average
                                ≈  1M QPS       peak (×3)

   Writes: 300M × 0.1 / 86,400  ≈  350 QPS      average
                                ≈  1,000 QPS    peak

   ⭐ READ:WRITE ≈ 1000:1  → this is a READ-HEAVY system.
     That single ratio drives the entire design:
     aggressive caching, read replicas, fan-out on write,
     precomputed timelines.

   ─── STORAGE ───────────────────────────────────────────
   Tweets/day  = 300M × 0.1 = 30M tweets
   Text/day    = 30M × 1 KB = 30 GB/day
   Images/day  = 3M × 200 KB = 600 GB/day
   Total       ≈ 630 GB/day  ≈ 230 TB/year
   With 3× replication ≈ 700 TB/year

   ─── BANDWIDTH ─────────────────────────────────────────
   Read egress = 350K QPS × 1 KB ≈ 350 MB/s ≈ 2.8 Gbps
   (images go through a CDN, so they don't hit origin)

   ─── CACHE ─────────────────────────────────────────────
   Cache the 20% of tweets that get 80% of reads:
   Daily active tweets ≈ 30M × 0.2 = 6M × 1 KB = 6 GB
   Plus timelines: 300M users × 200 tweet IDs × 8 bytes = 480 GB
   → sizeable but very feasible on a Redis cluster
```

```mermaid
flowchart LR
    DAU["300M DAU"] --> Reads["Reads: 350K QPS avg<br/>~1M QPS peak"]
    DAU --> Writes["Writes: 350 QPS avg<br/>~1,000 QPS peak"]
    Reads --> Ratio["⭐ READ:WRITE ≈ 1000:1<br/><b>READ-HEAVY system</b>"]
    Writes --> Ratio
    Ratio --> Design["Drives the design:<br/>aggressive caching,<br/>read replicas,<br/>fan-out on write,<br/>precomputed timelines"]

    style DAU fill:#e1f5fe,stroke:#0277bd,color:#000
    style Reads fill:#fff9c4,stroke:#f9a825,color:#000
    style Writes fill:#fff9c4,stroke:#f9a825,color:#000
    style Ratio fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    style Design fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
```

🎤 **How to present this:** round to powers of ten, say your assumptions out loud, and — most importantly — **state what the numbers imply**. "1000:1 read-to-write means I'll precompute timelines on write rather than compute on read" is the sentence that earns the points, not the arithmetic.

---

## 14. Evolution

#### 💬 How real architectures actually grow
Nobody starts with microservices and Kafka. Systems evolve under pressure, and each stage is a response to a specific pain.

```
   ── STAGE 1: 0 → 1,000 users ─────────────────────────────────
   ┌────────────────────────┐
   │  One box: app + DB     │      Monolith. SQLite or Postgres.
   └────────────────────────┘      Deploy = git push.
   Pain: none. This is correct. Don't over-engineer.

   ── STAGE 2: 1K → 10K ────────────────────────────────────────
   ┌────────┐      ┌────────┐
   │  App   │─────▶│   DB   │      Split app and DB.
   └────────┘      └────────┘      Add a CDN for static assets.
   Pain: the DB and app compete for CPU/RAM.

   ── STAGE 3: 10K → 100K ──────────────────────────────────────
              ┌────┐  ┌────┐
   LB ───────▶│App │  │App │       Multiple stateless app servers.
              └──┬─┘  └─┬──┘       Sessions → Redis.
                 └──┬───┘          Add a cache layer.
              ┌─────▼─────┐        Add read replicas.
              │ Redis     │
              ├───────────┤
              │ DB primary│──▶ replica ──▶ replica
              └───────────┘
   Pain: one app server can't take the traffic; DB reads saturate.

   ── STAGE 4: 100K → 1M ───────────────────────────────────────
   + Message queue for async work
   + Object storage for uploads
   + Search cluster (Elasticsearch)
   + Multiple availability zones
   + Real monitoring and alerting
   Pain: slow endpoints block fast ones; deploys are risky.

   ── STAGE 5: 1M → 10M+ ───────────────────────────────────────
   + Split the monolith where team boundaries demand it
   + Shard the database (LAST resort)
   + Multi-region for latency and DR
   + Data warehouse for analytics (separate from OLTP)
   + Service mesh, distributed tracing
   Pain: organizational as much as technical — teams block each other.
```

```mermaid
flowchart LR
    S1["Stage 1<br/>0→1K users<br/>🟢 monolith<br/>app + DB, one box"] --> S2["Stage 2<br/>1K→10K<br/>🟢 split app / DB<br/>+ CDN"]
    S2 --> S3["Stage 3<br/>10K→100K<br/>🟡 LB + stateless app<br/>+ Redis + read replicas"]
    S3 --> S4["Stage 4<br/>100K→1M<br/>🟡 + queue, object storage,<br/>search, multi-AZ"]
    S4 --> S5["Stage 5<br/>1M→10M+<br/>🔴 shard DB, multi-region,<br/>service mesh — LAST resort"]

    style S1 fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px,color:#000
    style S2 fill:#c8e6c9,stroke:#2e7d32,color:#000
    style S3 fill:#fff9c4,stroke:#f9a825,color:#000
    style S4 fill:#fff9c4,stroke:#f9a825,stroke-width:2px,color:#000
    style S5 fill:#ffcdd2,stroke:#c62828,stroke-width:2px,color:#000
```

⚠️ **Every stage adds complexity that costs you forever.** Only advance when a specific, measured pain forces you to. "We might need to scale someday" is not a reason.

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                 SYSTEM DESIGN FUNDAMENTALS — ONE PAGE                ║
╠══════════════════════════════════════════════════════════════════════╣
║ FIRST RULE: don't distribute until you must. One good Postgres box   ║
║   handles most products. Every machine you add adds a failure mode.  ║
╠══════════════════════════════════════════════════════════════════════╣
║ LATENCY (memorize the SHAPE)                                         ║
║   RAM 100ns · SSD 150µs · datacenter RTT 500µs · disk seek 10ms      ║
║   cross-continent 150ms                                              ║
║   → cache in memory · batch network calls · never cross-region       ║
║     in a request path                                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ SCALE ORDER: vertical → read replicas → cache → horizontal(stateless)║
║   → partition → SHARD (last, irreversible)                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ STATELESSNESS is what makes horizontal scaling possible.             ║
║   sessions→Redis · files→S3 · nothing on local disk                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ REPLICATION scales READS · SHARDING scales WRITES                    ║
║   ⚠️ replication lag → read-your-writes bug. Route to primary after   ║
║      a write, or wait for the LSN.                                   ║
║ CONSISTENT HASHING: adding a node moves 1/N of keys, not all of them ║
║   (use ~150 virtual nodes per physical node)                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ CAP: partitions HAPPEN. When one does, choose C or A.                ║
║   PACELC is more useful: Else, choose Latency or Consistency —       ║
║   that's the tradeoff present in EVERY request.                      ║
║ Name the USER-VISIBLE consequence of your consistency choice.        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ASYNC: "does the user need this before I respond?"                   ║
║   ⭐ OUTBOX PATTERN — you cannot atomically write DB + queue          ║
╠══════════════════════════════════════════════════════════════════════╣
║ RESILIENCE: timeout · retry+backoff+JITTER · circuit breaker ·       ║
║   bulkhead · rate limit · graceful degradation · SERVE STALE         ║
║   Cascading failure = slow dependency + no timeout + no breaker      ║
╠══════════════════════════════════════════════════════════════════════╣
║ ESTIMATION: DAU → QPS (peak = avg × 2-10) → storage → bandwidth      ║
║   1M requests/day ≈ 12 QPS   ·   1 day ≈ 10⁵ sec                     ║
║   ⭐ The read:write RATIO drives the whole design                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Building Blocks →](01-building-blocks.md) · **Related:** [Databases](../03-backend/databases.md) · [Caching](../03-backend/caching.md) · [Queues](../03-backend/queues-streaming.md)
