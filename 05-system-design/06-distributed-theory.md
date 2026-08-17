# 🔬 Distributed Systems Theory

> The impossibility results and formal models behind everything in the previous books. You don't need this to pass most interviews — but it's what separates "I've read about Kafka" from "I understand why Kafka is shaped that way."

**Prerequisite:** [Fundamentals](00-fundamentals.md)

---

## 📑 Contents

1. [The Two Hard Facts](#1-the-two-hard-facts)
2. [Failure Models](#2-failure-models)
3. [The FLP Impossibility Result](#3-the-flp-impossibility-result)
4. [Time, Clocks, and Ordering](#4-time-clocks-and-ordering)
5. [Consensus: Paxos and Raft](#5-consensus-paxos-and-raft)
6. [Quorums](#6-quorums)
7. [Consistency Models, Formally](#7-consistency-models-formally)
8. [CRDTs](#8-crdts)
9. [Distributed Transactions](#9-distributed-transactions)
10. [Gossip Protocols](#10-gossip-protocols)
11. [Probabilistic Data Structures](#11-probabilistic-data-structures)
12. [Cheat Sheet](#12-cheat-sheet)

---

## 1. The Two Hard Facts

#### 💬 Everything reduces to these

```
   ┌──────────────────────────────────────────────────────────────┐
   │ FACT 1: THE NETWORK IS UNRELIABLE                            │
   │                                                              │
   │   Messages can be:  lost · delayed arbitrarily · duplicated  │
   │                     · reordered · delivered after you gave up│
   │                                                              │
   │   ⭐ CRITICAL COROLLARY: you cannot distinguish               │
   │      "the node is dead" from "the node is slow."             │
   │      A timeout is a GUESS, never a fact.                     │
   ├──────────────────────────────────────────────────────────────┤
   │ FACT 2: NODES FAIL INDEPENDENTLY                             │
   │                                                              │
   │   Half your system can be up while the other half is down,   │
   │   and the two halves can DISAGREE about what happened.       │
   │   This is "partial failure" and it has no analogue in        │
   │   single-machine programming.                                │
   └──────────────────────────────────────────────────────────────┘
```

### The Two Generals Problem

```
   Two armies must attack simultaneously to win.
   They communicate only by messengers crossing enemy territory.

   General A ──"attack at dawn"──▶ General B
             ◀──────"ack"────────

   But A doesn't know the ack arrived...
   so B sends an ack of the ack...
   but B doesn't know THAT arrived...

   ⭐ PROVEN IMPOSSIBLE: no finite number of messages can
     guarantee both generals know they agree.

   PRACTICAL CONSEQUENCE:
   You can never achieve exactly-once delivery across a network.
   → at-least-once delivery + IDEMPOTENT processing
   → this is why every payment API has idempotency keys
     ([Stripe](04-case-studies-2.md#3-deep-dive--idempotency))
```

### The Eight Fallacies of Distributed Computing

```
   Every one of these is FALSE, and assuming any of them
   produces a specific class of production incident.

   1. The network is reliable       → need retries, timeouts, idempotency
   2. Latency is zero               → batch calls, never chatty protocols
   3. Bandwidth is infinite         → compress, paginate, stream
   4. The network is secure         → TLS everywhere, zero trust
   5. Topology doesn't change       → service discovery, health checks
   6. There is one administrator    → coordination, versioning, contracts
   7. Transport cost is zero        → serialization and egress cost real money
   8. The network is homogeneous    → protocol negotiation, varied MTUs
```

---

## 2. Failure Models

```
   Ordered from easiest to hardest to tolerate:

   ┌──────────────────────────────────────────────────────────────┐
   │ CRASH-STOP        A node fails and never returns.            │
   │                   Easiest. Detect and replace.               │
   ├──────────────────────────────────────────────────────────────┤
   │ CRASH-RECOVERY    A node fails and later comes back, possibly│
   │                   with stale state. ⭐ This is the realistic  │
   │                   model for most systems — and it's why      │
   │                   write-ahead logs and fencing tokens exist. │
   ├──────────────────────────────────────────────────────────────┤
   │ OMISSION          A node drops some messages but not others. │
   │                   Partial connectivity — nastier than it     │
   │                   sounds, because health checks may pass     │
   │                   while real traffic fails.                  │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ BYZANTINE       A node behaves ARBITRARILY: lies, sends    │
   │                   contradictory messages to different peers, │
   │                   corrupts data.                             │
   │                   Requires 3f+1 nodes to tolerate f failures │
   │                   (vs 2f+1 for crash faults).                │
   │                   ⭐ Assume crash faults inside your own      │
   │                     datacenter. Byzantine tolerance is for   │
   │                     blockchains and adversarial environments.│
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ THE GRAY FAILURE — the one that causes real outages

   Not "up" or "down" but DEGRADED:
     • a node responds to health checks but times out on real work
     • a disk is 100× slower than normal but not failed
     • a network link drops 5% of packets
     • a node accepts connections but its thread pool is exhausted

   ⭐ Gray failures are worse than crashes because failover
     doesn't trigger. The node stays in rotation, poisoning
     a fraction of all requests.

   MITIGATIONS:
     • health checks that exercise the REAL path, not just /ping
     • latency-based ejection (outlier detection in a service mesh)
     • ⭐ "slow is the new down" — treat p99 latency spikes as
       a failure signal, not just errors
```

---

## 3. The FLP Impossibility Result

#### 💬 The most important theorem you'll never directly use

```
   ⭐ FLP (Fischer, Lynch, Paterson, 1985)

   "In an asynchronous system where even ONE node may fail by
    crashing, there is NO deterministic algorithm that
    guarantees consensus."

   WHY: in a fully asynchronous model there's no bound on message
   delay, so you cannot distinguish a crashed node from a slow one.
   Any algorithm that waits forever might hang; any algorithm that
   gives up might be wrong.
```

```
   ⚠️ BUT CONSENSUS SYSTEMS OBVIOUSLY EXIST. How?

   FLP rules out an algorithm that is BOTH always-safe AND
   always-live in a fully asynchronous model. Real systems
   escape by weakening one assumption:

   ┌──────────────────────────────────────────────────────────────┐
   │ ① PARTIAL SYNCHRONY  ⭐ what Raft and Paxos actually assume   │
   │    "The network is eventually well-behaved for long enough." │
   │    → use TIMEOUTS as failure detectors.                      │
   │    → SAFETY is always preserved; LIVENESS is only            │
   │      guaranteed during periods of synchrony.                 │
   │    ⭐ This is the crucial framing: a Raft cluster under       │
   │      constant partition never makes progress — but it        │
   │      never becomes INCORRECT either.                         │
   ├──────────────────────────────────────────────────────────────┤
   │ ② RANDOMIZATION                                              │
   │    Terminate with probability 1 rather than certainty.       │
   │    (Ben-Or's algorithm; used in some BFT protocols.)         │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ FAILURE DETECTORS                                          │
   │    Assume an oracle that eventually identifies crashed       │
   │    nodes. Formally this is what timeouts approximate.        │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE TAKEAWAY FOR PRACTICE:
     Never design a system that must reach consensus to serve
     a request on the critical path. Consensus is for small,
     critical metadata — leader election, configuration,
     membership — not for high-volume data operations.
```

---

## 4. Time, Clocks, and Ordering

#### 💬 Why you can't just use timestamps

```
   ⚠️ WALL-CLOCK TIME IS NOT A VALID ORDERING MECHANISM

   • Clocks drift (typically 10-100 ppm — seconds per day)
   • NTP corrections can move a clock BACKWARDS
   • Leap seconds have caused real, famous outages
   • VM migrations and suspend/resume jump the clock
   • ⭐ Two events 1ms apart on different machines cannot be
     reliably ordered by their timestamps

   ⚠️ THE CLASSIC BUG: last-write-wins by wall clock.
     A node with a fast clock silently wins every conflict,
     and writes from correct nodes are permanently lost.
```

### Lamport Clocks — logical time

```
   RULES (a simple counter per node):
   1. Increment your counter before every local event
   2. Send your counter with every message
   3. On receive:  counter = max(local, received) + 1

   ┌─────────────────────────────────────────────────────────────┐
   │  Node A   1 ──── 2 ────────────▶ 3                          │
   │                  │                                          │
   │                  │ msg(2)                                   │
   │                  ▼                                          │
   │  Node B   1 ──── 3 ──── 4                                   │
   │                  ▲                                          │
   │                  └─ max(1, 2) + 1 = 3                       │
   └─────────────────────────────────────────────────────────────┘

   ⭐ GUARANTEE:  a → b  implies  L(a) < L(b)
   ⚠️ THE CONVERSE IS FALSE: L(a) < L(b) does NOT imply
     a happened before b. They may be concurrent.

   → Lamport clocks give a TOTAL ORDER, but that order is
     partly arbitrary. Useful for tie-breaking; useless for
     detecting concurrency.
```

### Vector Clocks — detecting concurrency

```
   Each node keeps a VECTOR of counters, one per node.

   ┌─────────────────────────────────────────────────────────────┐
   │  Node A  [1,0,0] ── [2,0,0] ──────────▶ [3,0,0]             │
   │                        │                                    │
   │                        │ msg([2,0,0])                       │
   │                        ▼                                    │
   │  Node B  [0,1,0] ── [2,2,0] ── [2,3,0]                      │
   │                        ▲                                    │
   │                        └─ element-wise max, then increment  │
   │                           own position                      │
   └─────────────────────────────────────────────────────────────┘

   COMPARING TWO VECTORS:
     V1 < V2   (V1 happened before V2)   if every element of V1
                                          is ≤ V2's AND at least
                                          one is strictly less
     V1 > V2   (V2 happened before V1)   symmetric
     ⭐ NEITHER                           → CONCURRENT → a real
                                            conflict requiring
                                            application resolution

   ⭐ This is exactly what [Amazon's cart](05-case-studies-3.md#3-deep-dive--the-cart)
     uses to detect that two versions genuinely conflict,
     at which point the union merge rule applies.

   ⚠️ COST: the vector grows with the number of nodes.
     Dotted version vectors and version vectors with pruning
     address this in practice.
```

### Hybrid Logical Clocks (HLC)

```
   ⭐ THE PRACTICAL SYNTHESIS — used by CockroachDB, MongoDB

   Combine physical time (so timestamps are human-meaningful and
   roughly correct) with a logical counter (so causality is
   never violated).

   HLC = (physical_time, logical_counter)

   • Close to wall-clock time, so it's useful for TTLs,
     debugging, and human interpretation
   • ⭐ Never goes backwards, and always respects causality
   • Bounded divergence from real time

   → You get most of the benefit of both without vector-clock
     size growth.
```

### TrueTime (Google Spanner)

```
   ⭐ THE MOST INTERESTING ENGINEERING ANSWER TO THIS PROBLEM

   Instead of pretending clocks are exact, Spanner makes
   UNCERTAINTY EXPLICIT.

   TrueTime returns an INTERVAL, not a point:
       TT.now() → [earliest, latest]

   Backed by GPS receivers and atomic clocks in every datacenter,
   the interval width ε is typically ~1-7 milliseconds.

   ⭐ THE COMMIT-WAIT TRICK

   Before committing a transaction at timestamp T, Spanner simply
   WAITS until TT.now().earliest > T.

   That guarantees T is definitely in the past everywhere, so any
   later transaction gets a strictly greater timestamp.

   → EXTERNAL CONSISTENCY (linearizability) across a GLOBAL
     database, which was widely believed impractical.

   COST: every commit waits ~2ε (a few milliseconds), and you
   need atomic clocks in your datacenters.

   ⭐ THE LESSON: sometimes the answer to "we can't measure this
     precisely" is "then measure the error bound and design
     around it" rather than "pretend the error is zero."
```

---

## 5. Consensus: Paxos and Raft

#### 💬 What consensus actually means

```
   N nodes must agree on ONE value, with these properties:

   ┌──────────────────────────────────────────────────────────────┐
   │ AGREEMENT    No two correct nodes decide different values    │
   │ VALIDITY     The decided value was proposed by some node     │
   │ TERMINATION  Every correct node eventually decides           │
   │              ⭐ FLP says you can't always guarantee this      │
   └──────────────────────────────────────────────────────────────┘

   USES: leader election · replicated state machines ·
         distributed locks · configuration · membership
```

### Raft — designed to be understandable

```
   ⭐ RAFT DECOMPOSES CONSENSUS INTO THREE INDEPENDENT PROBLEMS

   ① LEADER ELECTION
   ② LOG REPLICATION
   ③ SAFETY

   That decomposition is the entire contribution — Paxos is
   equally correct but notoriously hard to reason about and
   implement correctly.
```

```
   ── ① LEADER ELECTION ────────────────────────────────────────

   Every node is in one of three states:

        ┌──────────┐  times out,          ┌───────────┐
        │ FOLLOWER │  starts election ───▶│ CANDIDATE │
        └──────────┘                      └─────┬─────┘
             ▲                                  │ wins majority
             │ discovers a higher term          ▼
             │ or a current leader        ┌──────────┐
             └────────────────────────────│  LEADER  │
                                          └──────────┘

   • Time is divided into TERMS (monotonically increasing).
     ⭐ A term is a logical clock — it's how stale leaders are
       detected and rejected.
   • A follower that receives no heartbeat within its election
     timeout becomes a candidate and requests votes.
   • ⭐ RANDOMIZED election timeouts (e.g. 150-300ms) prevent
     split votes — without randomization, all followers time
     out simultaneously and repeatedly split the vote.
   • A candidate winning a MAJORITY becomes leader.
```

```
   ── ② LOG REPLICATION ────────────────────────────────────────

   All client writes go through the leader.

   LEADER      [1:x=1][2:y=2][3:x=5]
                  │      │      │
                  ▼      ▼      ▼   AppendEntries RPCs
   FOLLOWER 1  [1:x=1][2:y=2][3:x=5]   ✅ committed
   FOLLOWER 2  [1:x=1][2:y=2][3:x=5]   ✅ (majority reached)
   FOLLOWER 3  [1:x=1][2:y=2]          ⏳ catching up

   ⭐ An entry is COMMITTED once a MAJORITY has stored it.
     Only then is it applied to the state machine and
     acknowledged to the client.

   ⭐ THE LOG MATCHING PROPERTY
     If two logs have an entry with the same index AND term,
     then the logs are IDENTICAL in all preceding entries.

     This is enforced by a consistency check in every
     AppendEntries: the leader includes the index and term of
     the PRECEDING entry, and a follower rejects the append if
     it doesn't match. Rejection causes the leader to walk
     backwards until it finds the point of agreement.
```

```
   ── ③ SAFETY: the election restriction ───────────────────────

   ⭐ A candidate cannot win an election unless its log is at
     least as up to date as a majority of the cluster.

   WHY THIS MATTERS: it guarantees that any newly elected leader
   already contains every committed entry. Without it, a node
   with a stale log could be elected and silently erase
   committed data.

   "Up to date" is defined as: higher last-entry term wins;
   if terms are equal, longer log wins.
```

```
   ⭐ WHY MAJORITY (QUORUM) IS THE MAGIC

   Any two majorities of the same set MUST overlap in at least
   one node.

     5 nodes → majority is 3
     {A,B,C} and {C,D,E} overlap at C

   → The overlapping node carries knowledge forward from the
     old majority to the new one. That's how a new leader is
     guaranteed to know about anything the previous majority
     committed.

   → Tolerates ⌊(N-1)/2⌋ failures:
     3 nodes → 1 failure    5 nodes → 2 failures
     ⚠️ EVEN cluster sizes gain nothing: 4 nodes still only
       tolerate 1 failure, so always use an odd number.
```

### Paxos in one paragraph

```
   Multi-Paxos is functionally equivalent to Raft but structured
   differently: a two-phase protocol (prepare/promise, then
   accept/accepted) with proposal numbers, and no distinguished
   leader in the basic formulation (though practical Multi-Paxos
   elects a stable leader to skip phase 1).

   ⭐ INTERVIEW ANSWER: "Paxos and Raft solve the same problem
     with equivalent guarantees. Raft decomposes it into leader
     election, log replication, and safety, which makes it far
     easier to implement correctly — which is why etcd, Consul,
     TiKV, and CockroachDB all use Raft."
```

---

## 6. Quorums

```
   ⭐ THE QUORUM INEQUALITY

        W + R > N   →   guaranteed read-your-writes

   N = replicas · W = replicas that must ack a write
   R = replicas that must respond to a read

   WHY: if W + R > N, the write set and read set MUST overlap
        in at least one replica, which holds the latest value.

   ┌────────────────────────────────────────────────────────────┐
   │  N=3, W=2, R=2   ⭐ the standard configuration              │
   │                                                            │
   │  write ──▶ [A]✅ [B]✅ [C]                                  │
   │  read  ──▶      [B]✅ [C]✅                                 │
   │                  ▲                                         │
   │                  └─ overlap → the read sees the write ✅    │
   │                                                            │
   │  Tolerates 1 node down for both reads and writes.          │
   └────────────────────────────────────────────────────────────┘
```

```
   TUNING THE TRADEOFF

   ┌──────────────────┬──────────────────────────────────────────┐
   │ W=1, R=1  (N=3)  │ Fastest both ways. NO consistency        │
   │                  │ guarantee. Use for: caches, metrics.     │
   ├──────────────────┼──────────────────────────────────────────┤
   │ W=N, R=1         │ Fast reads, slow writes. A single node   │
   │                  │ down blocks ALL writes. ⚠️ poor          │
   │                  │ availability.                            │
   ├──────────────────┼──────────────────────────────────────────┤
   │ W=1, R=N         │ Fast writes, slow reads. Same problem    │
   │                  │ mirrored.                                │
   ├──────────────────┼──────────────────────────────────────────┤
   │ W=2, R=2  (N=3)  │ ⭐ BALANCED. Survives one failure on      │
   │                  │ both paths. The usual choice.            │
   └──────────────────┴──────────────────────────────────────────┘
```

```
   ⚠️ SLOPPY QUORUMS + HINTED HANDOFF  (Dynamo-style)

   If the "home" replicas for a key are unreachable, write to
   ANY N healthy nodes instead, with a hint about the intended
   destination. When the home node recovers, hand the data off.

   ✅ Massively improves write availability during partitions
   ❌ ⭐ W + R > N no longer guarantees overlap — you can read
      stale data even with a "quorum," because the write went
      to different nodes than the read is querying.

   → This is a deliberate AP choice. Cassandra and DynamoDB
     offer it; know that enabling it weakens your consistency
     guarantee, not just your durability.
```

```
   ANTI-ENTROPY — how replicas converge

   ① READ REPAIR       on a read, detect stale replicas and fix
                       them inline. Cheap, but only repairs data
                       that's actually read.
   ② HINTED HANDOFF    deliver writes that were buffered during
                       a node's downtime
   ③ MERKLE TREES      ⭐ efficiently find WHICH keys differ
                       between two replicas

   ┌──────────────────────────────────────────────────────────┐
   │            MERKLE TREE COMPARISON                        │
   │                                                          │
   │        root hash          root hash                      │
   │          ≠ ────────────────── ≠                          │
   │         /  \                /  \                         │
   │       H1    H2            H1    H2'                      │
   │       =      ≠            =      ≠                       │
   │            /  \                /  \                      │
   │          H3    H4            H3    H4'                   │
   │          =      ≠            =      ≠                    │
   │                                                          │
   │  ⭐ Compare root hashes: if equal, replicas are identical │
   │    and you're done in O(1).                              │
   │  ⭐ If different, descend only into mismatched subtrees.  │
   │    → find the differing keys in O(log n) comparisons     │
   │      instead of transferring the entire dataset.         │
   └──────────────────────────────────────────────────────────┘
```

---

## 7. Consistency Models, Formally

```
   STRONGEST ────────────────────────────────────────▶ WEAKEST

   ┌──────────────────────────────────────────────────────────────┐
   │ LINEARIZABILITY (atomic consistency)                         │
   │   Every operation appears to take effect INSTANTANEOUSLY at  │
   │   some point between its invocation and its response, and    │
   │   that order matches REAL TIME.                              │
   │   ⭐ The system is indistinguishable from a single copy.      │
   │   Cost: coordination on every operation.                     │
   │   Examples: etcd, ZooKeeper, Spanner, a single-node DB       │
   ├──────────────────────────────────────────────────────────────┤
   │ SEQUENTIAL CONSISTENCY                                       │
   │   All nodes see operations in the SAME order, and each       │
   │   process's own operations appear in its program order.      │
   │   ⚠️ But that order need not match real time — an operation   │
   │     that finished before another started may appear after.   │
   ├──────────────────────────────────────────────────────────────┤
   │ CAUSAL CONSISTENCY                                           │
   │   Causally-related operations are seen in order by everyone. │
   │   Concurrent operations may be seen in any order.            │
   │   ⭐ The strongest model achievable in an always-available    │
   │     system (proven). This is the practical ceiling for AP.   │
   │   Example: a reply never appears before the post it answers. │
   ├──────────────────────────────────────────────────────────────┤
   │ SESSION GUARANTEES (per-client, composable)                  │
   │   • Read-your-writes    — you see your own writes            │
   │   • Monotonic reads     — you never see time go backwards    │
   │   • Monotonic writes    — your writes apply in order         │
   │   • Writes-follow-reads — writes are ordered after the reads │
   │                           that caused them                   │
   │   ⭐ These are what users actually NOTICE. A system with all  │
   │     four feels correct even if it's only eventually          │
   │     consistent globally.                                     │
   ├──────────────────────────────────────────────────────────────┤
   │ EVENTUAL CONSISTENCY                                         │
   │   If writes stop, replicas eventually converge.              │
   │   ⚠️ Says NOTHING about when, or what you see meanwhile.      │
   │   Technically satisfied by a system that returns garbage     │
   │   for a year then converges.                                 │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ LINEARIZABILITY vs SERIALIZABILITY — the classic confusion

   ┌────────────────────────┬─────────────────────────────────────┐
   │ LINEARIZABILITY        │ SERIALIZABILITY                     │
   ├────────────────────────┼─────────────────────────────────────┤
   │ About SINGLE objects   │ About TRANSACTIONS (multi-object)   │
   │ A recency guarantee    │ An isolation guarantee              │
   │ Respects REAL TIME     │ ⚠️ Any serial order will do — it may │
   │                        │   reorder transactions arbitrarily  │
   │ From distributed       │ From database theory                │
   │   systems theory       │                                     │
   └────────────────────────┴─────────────────────────────────────┘

   ⭐ STRICT SERIALIZABILITY = both.
     Transactions are serializable AND that order respects
     real time. This is what Spanner provides, and what most
     people actually mean when they say "strong consistency."
```

---

## 8. CRDTs

#### 💬 The formal foundation

```
   ⭐ THE INSIGHT: if your merge operation forms a JOIN
     SEMILATTICE, convergence is GUARANTEED regardless of
     message order, duplication, or delay.

   REQUIRED PROPERTIES OF THE MERGE FUNCTION:

   ┌──────────────────────────────────────────────────────────────┐
   │ ASSOCIATIVE   (a ⊔ b) ⊔ c  =  a ⊔ (b ⊔ c)                    │
   │               → grouping doesn't matter                      │
   ├──────────────────────────────────────────────────────────────┤
   │ COMMUTATIVE   a ⊔ b  =  b ⊔ a                                │
   │               → ORDER doesn't matter (no coordination!)      │
   ├──────────────────────────────────────────────────────────────┤
   │ IDEMPOTENT    a ⊔ a  =  a                                    │
   │               → DUPLICATES don't matter (at-least-once       │
   │                 delivery is fine)                            │
   └──────────────────────────────────────────────────────────────┘

   ⭐ Together these mean: deliver updates however you like —
     out of order, duplicated, delayed — and every replica
     still converges to the identical state. No consensus,
     no leader, no coordination.
```

### The two families

```
   ┌──────────────────────────────────────────────────────────────┐
   │ STATE-BASED (CvRDT — convergent)                             │
   │   Ship the whole state; merge with the join operation.       │
   │   ✅ Simple, tolerant of any network behaviour                │
   │   ❌ Bandwidth-heavy (mitigated by delta-CRDTs)               │
   ├──────────────────────────────────────────────────────────────┤
   │ OPERATION-BASED (CmRDT — commutative)                        │
   │   Ship the operations; they must commute.                    │
   │   ✅ Compact messages                                         │
   │   ❌ Requires exactly-once, causally-ordered delivery         │
   └──────────────────────────────────────────────────────────────┘
```

### Worked examples

```
   ── G-COUNTER (grow-only counter) ────────────────────────────

   Each replica keeps a per-node count; merge = element-wise max.

   Node A: {A:3, B:0, C:0}     value = 3
   Node B: {A:0, B:5, C:0}     value = 5
   ──────────────────────────────────── merge (element-wise max)
   Merged: {A:3, B:5, C:0}     value = sum = 8  ✅

   ⭐ Why max and not sum? Because merging twice must be
     idempotent. Summing would double-count on redelivery;
     max is naturally idempotent.

   ── PN-COUNTER (increment and decrement) ─────────────────────
   Two G-Counters: one for increments, one for decrements.
   value = sum(P) − sum(N)

   ── LWW-REGISTER ─────────────────────────────────────────────
   Value + timestamp; merge keeps the higher timestamp.
   ⚠️ Requires a tiebreaker (node ID) for equal timestamps, and
     it silently DISCARDS the losing write. Simple but lossy.

   ── OR-SET (observed-remove set) ⭐ the practical set ─────────

   Problem: a naive set can't distinguish "never added" from
   "added then removed," so concurrent add/remove is ambiguous.

   Solution: tag every add with a unique ID. A remove only
   removes the tags it has OBSERVED.

   A: add(x) with tag t1     →  {x: {t1}}
   B: (concurrently) removes x, but has only observed t1
                             →  removes t1
   A: add(x) again with t2   →  {x: {t1, t2}}
   ───────────────────────────────────────── merge
   Result: {x: {t2}}  →  x IS present  ⭐ ADD WINS

   ⭐ "Add wins" on concurrent add/remove is the standard and
     usually correct choice — same asymmetric-cost reasoning as
     [Amazon's cart](05-case-studies-3.md#3-deep-dive--the-cart).
```

```
   ⚠️ THE REAL COSTS OF CRDTs

   • METADATA OVERHEAD — tags, IDs, and version vectors can
     exceed the size of the actual data
   • TOMBSTONES — removed elements must be retained to prevent
     resurrection. Garbage collecting them safely requires
     knowing every replica has seen the removal, which is
     itself a coordination problem.
   • ⭐ CONVERGENCE ≠ CORRECTNESS. CRDTs guarantee all replicas
     reach the SAME state — not that the state is what a user
     would consider right. Concurrent edits to the same word
     can converge to interleaved gibberish.
```

---

## 9. Distributed Transactions

### Two-Phase Commit (2PC)

```
   ┌─ PHASE 1: PREPARE ──────────────────────────────────────────┐
   │  Coordinator ──"can you commit?"──▶ Participant A → YES     │
   │              ──"can you commit?"──▶ Participant B → YES     │
   │  ⭐ Participants that vote YES must be able to commit even   │
   │    after a crash — so they durably log their vote and HOLD  │
   │    LOCKS until told what to do.                             │
   ├─ PHASE 2: COMMIT ───────────────────────────────────────────┤
   │  Coordinator ──"COMMIT"──▶ A ✅                              │
   │              ──"COMMIT"──▶ B ✅                              │
   └─────────────────────────────────────────────────────────────┘

   ⚠️⚠️ THE FATAL FLAW: BLOCKING

   If the coordinator crashes AFTER participants voted YES but
   BEFORE sending the decision:

   • Participants cannot commit (maybe someone voted NO)
   • Participants cannot abort (maybe everyone voted YES)
   • ⭐ They must WAIT, HOLDING LOCKS, indefinitely.

   → One coordinator failure can freeze every participant.
   → 3PC adds a phase to reduce blocking, but it fails under
     network partitions, so it's rarely used in practice.
```

```
   ⭐ WHEN 2PC IS ACTUALLY FINE
     • Within a single database cluster (the coordinator is
       highly available and colocated)
     • Short transactions with low contention
     • XA transactions across a small number of trusted resources

   ⚠️ WHEN IT ISN'T
     • Across microservices over a WAN
     • Long-running business processes
     • Anywhere availability matters more than strict atomicity
     → use SAGAS instead
```

### Sagas

```
   Break the transaction into local transactions, each with a
   COMPENSATING action.

   FORWARD:   reserve inventory → charge card → create shipment
   FAILURE AT STEP 3:
   COMPENSATE: ← release inventory ← refund card

   ┌──────────────────────────────────────────────────────────────┐
   │ ⭐ WHAT SAGAS GIVE UP — say this explicitly                   │
   │                                                              │
   │  ❌ NO ISOLATION. Other transactions observe intermediate     │
   │     states (an order exists as "pending" mid-flight).        │
   │  ❌ Compensations are NOT rollbacks. You can't un-ship a      │
   │     package or un-send an email — you issue a return or a    │
   │     correction. Each compensation is a NEW business event.   │
   │  ❌ Compensations can themselves FAIL → you need a manual     │
   │     intervention queue.                                      │
   │  ❌ Saga state must be durable, or a crash leaves it          │
   │     half-completed forever.                                  │
   │  ❌ Every step AND every compensation must be IDEMPOTENT.     │
   └──────────────────────────────────────────────────────────────┘

   ORCHESTRATION vs CHOREOGRAPHY:
     Orchestration — a coordinator drives the steps.
       ✅ the flow is visible, testable, and resumable
       ❌ the orchestrator is a coupling point
     Choreography — services react to each other's events.
       ✅ loose coupling
       ❌ no single place shows the whole flow → very hard to debug

   ⭐ Use orchestration for real business processes with
     compensation ([Amazon orders](05-case-studies-3.md#5-deep-dive--order-orchestration)),
     choreography for simple fan-out.
```

### Percolator / deterministic transactions

```
   ⭐ MODERN ALTERNATIVES WORTH NAMING

   PERCOLATOR (Google, used by TiDB)
     Optimistic MVCC + 2PC where the LOCKS live in the data
     store itself rather than in a coordinator's memory.
     → a coordinator crash doesn't block: another client can
       resolve the abandoned lock by reading its state.
     ⭐ This directly fixes 2PC's blocking problem.

   CALVIN / DETERMINISTIC DATABASES (FaunaDB)
     Agree on the transaction ORDER first, then execute
     deterministically on every replica.
     → no commit protocol needed at all, because every replica
       independently reaches the same result.
     ❌ requires knowing the read/write set in advance.
```

---

## 10. Gossip Protocols

#### 💬 Scalable membership and dissemination

```
   THE PROBLEM: how do 1,000 nodes agree on who's alive without
   an O(N²) heartbeat mesh or a central registry?

   ⭐ GOSSIP: each node periodically picks a few RANDOM peers
     and exchanges state.

   ┌──────────────────────────────────────────────────────────────┐
   │  Round 0:   1 node knows                                     │
   │  Round 1:   2 nodes                                          │
   │  Round 2:   4 nodes                                          │
   │  Round 3:   8 nodes                                          │
   │             ...                                              │
   │  ⭐ Spreads EXPONENTIALLY → O(log N) rounds to reach          │
   │    everyone, with high probability.                          │
   └──────────────────────────────────────────────────────────────┘

   ✅ No single point of failure
   ✅ Scales to thousands of nodes
   ✅ Robust to message loss (redundancy is inherent)
   ✅ Constant per-node bandwidth regardless of cluster size
   ❌ Eventually consistent — no instant global view
   ❌ Redundant messages (the same news arrives many times)
```

```
   ⭐ SWIM — the protocol most systems actually use
     (Consul, Serf, and similar)

   ① DIRECT PROBE       node A pings node B
   ② INDIRECT PROBE     if no response, A asks k other nodes
                        to ping B on its behalf
                        ⭐ this distinguishes "B is down" from
                          "the A→B link is broken" — a genuinely
                          important distinction
   ③ SUSPICION          B is marked SUSPECT, not immediately
                        dead; the suspicion is gossiped
   ④ CONFIRMATION       B can refute the suspicion by responding
   ⑤ DEATH              after a timeout, B is declared dead
                        and that fact is gossiped

   ⭐ The suspicion mechanism dramatically reduces false
     positives from transient network blips — which otherwise
     cause flapping membership and needless rebalancing.
```

---

## 11. Probabilistic Data Structures

#### 💬 Trading exactness for enormous space savings

```
   ⭐ THE GENERAL PRINCIPLE: for many questions at scale, an
     approximate answer in kilobytes beats an exact answer in
     gigabytes — especially when the exact answer would require
     memory you don't have.
```

### Bloom Filter — "definitely not present" or "maybe present"

```
   m bits, k hash functions

   ADD("abc"):    set bits at h1("abc"), h2("abc"), h3("abc")
   QUERY("xyz"):  are ALL k bits set?
                    no  → DEFINITELY NOT PRESENT  ✅ certain
                    yes → MAYBE present (could be a false positive)

   ┌──────────────────────────────────────────────────────────────┐
   │  bits:  0 1 0 1 1 0 0 1 0 1 0 0                              │
   │           ▲   ▲ ▲                                            │
   │           └───┴─┴── "abc" set these three                    │
   │                                                              │
   │  ⭐ NO FALSE NEGATIVES — that's the useful guarantee          │
   │  ⚠️ FALSE POSITIVES possible — and you cannot DELETE          │
   │    (clearing a bit might unset it for another element)       │
   │    → counting Bloom filters allow deletion at 4× the space   │
   └──────────────────────────────────────────────────────────────┘

   SIZING:  ~9.6 bits per element for a 1% false positive rate
            ~14.4 bits per element for 0.1%

   → 100 MILLION elements at 1% FP ≈ 120 MB
     (versus many gigabytes for an exact set)

   USES: cache penetration defense ([Caching §6.2](../03-backend/caching.md#62-cache-penetration)) ·
         LSM-tree SSTable skipping · "have I seen this URL?" ·
         avoiding expensive disk lookups
```

### HyperLogLog — cardinality in constant space

```
   ⭐ COUNT DISTINCT ELEMENTS IN ~12 KB, WITH ~0.81% ERROR,
     REGARDLESS OF WHETHER THERE ARE 1,000 OR 1 BILLION.

   THE INTUITION:
   Hash each element and look at the position of the leftmost
   1-bit. In a stream of random hashes, seeing a hash that
   starts with 10 zeros suggests you've probably seen around
   2¹⁰ distinct values — because that's how rare such a hash is.

   Averaging that estimate across many independent "buckets"
   (using a harmonic mean, which suppresses outliers) makes it
   statistically robust.

   ✅ Mergeable! HLL(A) ⊔ HLL(B) = HLL(A ∪ B)
      ⭐ This is what makes it work across shards and time
        windows — count uniques per hour, merge for the day.

   USES: unique visitors · distinct search terms · cardinality
         estimation for the query planner
```

### Count-Min Sketch — frequency estimation

```
   A 2D array of counters with d hash functions.

   INCREMENT(x):  for each row i, increment counter[i][hi(x)]
   ESTIMATE(x):   return MIN over all rows of counter[i][hi(x)]

   ⭐ Taking the MINIMUM is the trick — collisions can only
     INFLATE a counter, never deflate it, so the smallest
     estimate is the closest to truth.

   ✅ Never underestimates
   ⚠️ May overestimate (bounded by the error parameter)

   USES: heavy hitters · trending topics · per-key rate limiting
         at scale · finding the top-K in a stream
```

### Other structures worth naming

```
   • CUCKOO FILTER      like a Bloom filter but supports DELETE,
                        with better lookup performance
   • T-DIGEST           accurate percentile estimation over a
                        stream (⭐ how p99 latency is computed
                        without storing every sample)
   • MINHASH            estimate Jaccard similarity between sets
                        → near-duplicate detection
   • SIMHASH            locality-sensitive hashing for documents
```

---

## 12. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║              DISTRIBUTED SYSTEMS THEORY — ONE PAGE                   ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ TWO HARD FACTS                                                     ║
║   1. You CANNOT distinguish "dead" from "slow." A timeout is a guess.║
║   2. Partial failure: half the system up, and the halves DISAGREE.   ║
║   Two Generals → exactly-once is impossible → at-least-once +        ║
║   IDEMPOTENCY is the only correct answer                             ║
╠══════════════════════════════════════════════════════════════════════╣
║ FLP: no deterministic consensus in a fully async system with 1 fault ║
║   → real systems assume PARTIAL SYNCHRONY: safety ALWAYS,            ║
║     liveness only during synchronous periods                         ║
║   ⭐ never put consensus on a high-volume request path                ║
╠══════════════════════════════════════════════════════════════════════╣
║ TIME: wall clocks CANNOT order distributed events                    ║
║   Lamport  → total order, but can't detect concurrency               ║
║   Vector   → DETECTS concurrency (Amazon cart); grows with N         ║
║   HLC      → physical + logical, the practical choice                ║
║   TrueTime → make uncertainty EXPLICIT, then commit-wait past it     ║
║   ⚠️ LWW by wall clock silently loses writes from slow-clock nodes    ║
╠══════════════════════════════════════════════════════════════════════╣
║ RAFT = leader election + log replication + safety                    ║
║   terms are a logical clock · RANDOMIZED timeouts prevent split votes║
║   committed = majority stored · log matching property                ║
║   election restriction: only an up-to-date log can win               ║
║   ⭐ quorums work because any two majorities OVERLAP                  ║
║   tolerate ⌊(N-1)/2⌋ → always use an ODD number of nodes             ║
╠══════════════════════════════════════════════════════════════════════╣
║ QUORUM: W + R > N → read-your-writes.  N=3,W=2,R=2 is the default   ║
║   ⚠️ sloppy quorums BREAK the overlap guarantee (deliberate AP choice)║
║   anti-entropy: read repair · hinted handoff · MERKLE TREES          ║
╠══════════════════════════════════════════════════════════════════════╣
║ CONSISTENCY  linearizable > sequential > causal > session > eventual ║
║   ⭐ LINEARIZABILITY = single object + real time (recency)            ║
║     SERIALIZABILITY = transactions + any serial order (isolation)    ║
║     STRICT SERIALIZABILITY = both (Spanner)                          ║
║   ⭐ causal is the STRONGEST model possible while staying available   ║
╠══════════════════════════════════════════════════════════════════════╣
║ CRDT: merge must be ASSOCIATIVE + COMMUTATIVE + IDEMPOTENT           ║
║   → converges under any order, duplication, or delay. No coordination║
║   G-Counter(max not sum) · OR-Set(tags, ADD WINS) · LWW(lossy)       ║
║   ⚠️ convergence ≠ correctness · tombstone GC is the hard part        ║
╠══════════════════════════════════════════════════════════════════════╣
║ 2PC ⚠️ BLOCKS if the coordinator dies after PREPARE — participants    ║
║   hold locks forever. Fine inside one cluster, wrong across services.║
║ SAGA: local txns + compensations. NO isolation. Compensations are    ║
║   new business events, not rollbacks, and can THEMSELVES fail.       ║
║ Percolator puts locks IN THE DATA → fixes 2PC blocking               ║
╠══════════════════════════════════════════════════════════════════════╣
║ GOSSIP: random peers, O(log N) rounds, constant per-node bandwidth   ║
║   SWIM adds INDIRECT PROBES (distinguishes node-down from link-down) ║
║   + SUSPICION (prevents flapping on transient blips)                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ PROBABILISTIC                                                        ║
║   BLOOM       no false negatives · ~9.6 bits/elem @ 1% FP · no delete║
║   HYPERLOGLOG count distinct in ~12 KB @ 0.81% error · MERGEABLE     ║
║   COUNT-MIN   frequency, takes MIN → never underestimates            ║
║   T-DIGEST    percentiles over a stream (how p99 is actually computed)║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Fundamentals](00-fundamentals.md) · [Building Blocks](01-building-blocks.md) · [Databases](../03-backend/databases.md) · [Queues & Streaming](../03-backend/queues-streaming.md)

**Further reading:** *Designing Data-Intensive Applications* (Kleppmann) · the Raft paper (Ongaro & Ousterhout) · the Dynamo paper (DeCandia et al.) · the Spanner paper (Corbett et al.) · Jepsen analyses (Kyle Kingsbury)
