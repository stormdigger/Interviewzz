# 🏗️ Case Studies Vol. 2 — Google Docs · YouTube · Dropbox · Airbnb · Stripe

> Five systems teaching correctness under concurrency, massive media pipelines, sync protocols, search/inventory, and money.

**Prerequisite:** [Framework](02-framework.md) · [Vol. 1](03-case-studies-1.md)

| System | The defining problem it teaches |
|---|---|
| [Google Docs](#-google-docs) | Concurrent editing — OT vs CRDT |
| [YouTube](#-youtube) | Video pipeline at planetary scale |
| [Dropbox](#-dropbox) | File sync, deduplication, conflict resolution |
| [Airbnb](#-airbnb) | Search, availability, and double-booking |
| [Stripe](#-stripe) | Money — exactly-once, idempotency, audit |

---

# 📝 Google Docs

> **Teaches:** the hardest correctness problem in consumer software — many people editing the same document simultaneously, with no lost work and no divergence.

## 1. Requirements

```
   FUNCTIONAL
   • Multiple users edit the same document simultaneously
   • Changes appear in ~real time (<200ms perceived)
   • See other users' cursors and selections
   • Full revision history, with restore
   • Offline editing, syncing on reconnect
   • Comments, suggestions

   NON-FUNCTIONAL
   SCALE          ~1B users; typical doc has 1-10 concurrent editors,
                  but must survive 100+
   LATENCY        ⭐ Local edits must feel INSTANT (0ms perceived)
   CONSISTENCY    ⭐ STRONG EVENTUAL CONSISTENCY — all replicas must
                  converge to an identical document
   DURABILITY     ⭐ Never lose a keystroke
   OFFLINE        Full editing offline, merge on reconnect
```

## 2. The Core Problem

#### 💬 Why concurrent editing is genuinely hard

Two people edit the same sentence at the same moment. Both apply their change locally (because waiting for the server would make typing feel awful). Now the two copies differ. When the edits are exchanged, they must converge to the *same* result — and that result must be sensible.

```
   Document: "HELLO"

   User A at position 1 inserts "X"    →  "HXELLO"
   User B at position 3 inserts "Y"    →  "HELYLO"

   Naive exchange:
     A applies B's op (insert Y at 3) to "HXELLO"  →  "HXEYLLO"
     B applies A's op (insert X at 1) to "HELYLO"  →  "HXELYLO"

                        ❌ DIVERGED

   ⭐ THE PROBLEM: B's operation was computed against the ORIGINAL
     document. After A's insert, position 3 no longer means what
     B meant. Positions shift.
```

Two families of solutions exist.

## 3. Solution A — Operational Transformation (OT)

#### 💬 The idea
Before applying a remote operation, **transform** it against every concurrent operation you've already applied, so its intent is preserved in the new context.

```
   TRANSFORM FUNCTION

   T(opB, opA) → opB'

   "Given that opA was applied first, what is the correct
    version of opB to apply now?"

   In our example:
     A inserted at position 1 (before B's position 3)
     → B's position must shift right by 1
     → T(insert Y at 3, insert X at 1) = insert Y at 4

   A applies:  "HXELLO" + insert Y at 4  →  "HXELYLO"
   B applies:  "HELYLO" + insert X at 1  →  "HXELYLO"

                        ✅ CONVERGED
```

```
   THE TRANSFORMATION RULES (insert/delete, simplified)

   T(ins@i, ins@j)  →  if i > j: ins@(i+1)   else: ins@i
                        (tie at i == j needs a deterministic
                         tiebreak — usually by site ID)

   T(ins@i, del@j)  →  if i > j: ins@(i-1)   else: ins@i

   T(del@i, ins@j)  →  if i >= j: del@(i+1)  else: del@i

   T(del@i, del@j)  →  if i > j: del@(i-1)
                        if i == j: NO-OP (already deleted)
                        else: del@i
```

```
   ARCHITECTURE — OT REQUIRES A CENTRAL SERVER

   ┌────────┐          ┌──────────────────┐          ┌────────┐
   │ Client │─────────▶│  SERVER          │◀─────────│ Client │
   │   A    │          │  • assigns a     │          │   B    │
   │        │◀─────────│    GLOBAL ORDER  │─────────▶│        │
   └────────┘          │  • transforms    │          └────────┘
                       │    against       │
                       │    pending ops   │
                       │  • broadcasts    │
                       └──────────────────┘

   ⭐ The server is the single source of truth for ORDER.
     Every client's op is transformed against ops the server
     has already accepted, then broadcast.

   ✅ Compact operations (just position + content)
   ✅ Preserves user intent well for text
   ✅ Efficient — no metadata bloat
   ❌ ⭐ Transformation functions are notoriously hard to get right.
      Google's implementation took years to stabilize; several
      published OT algorithms were later shown to be incorrect.
   ❌ Requires a central server — no true peer-to-peer
   ❌ Complexity grows badly with richer data types
```

## 4. Solution B — CRDTs

#### 💬 The idea
Design the data structure so that **concurrent operations commute** — applying them in any order gives the same result. Then no transformation is needed at all.

```
   FOR TEXT: give every character a UNIQUE, IMMUTABLE, ORDERABLE ID
   instead of a positional index.

   "HELLO" becomes:
     [H:(1,A)] [E:(2,A)] [L:(3,A)] [L:(4,A)] [O:(5,A)]
                  ▲
                  └─ (logical position, site ID)

   A inserts X between H and E:
     → new ID (1.5, A) — an identifier BETWEEN the neighbours
   B inserts Y between L and L:
     → new ID (3.5, B)

   Merging = union the sets, then SORT by ID.
   ⭐ Order of arrival is irrelevant. Convergence is guaranteed
     by the structure itself, not by a protocol.
```

```
   THE CRDT FAMILIES

   ┌──────────────────────────────────────────────────────────────┐
   │ G-Counter     grow-only counter (merge = max per replica)    │
   │ PN-Counter    increment/decrement (two G-Counters)           │
   │ G-Set         grow-only set (merge = union)                  │
   │ 2P-Set        add + remove once (can't re-add)               │
   │ LWW-Register  last-write-wins by timestamp                   │
   │ OR-Set        observed-remove set (add wins over concurrent  │
   │               remove — the practical choice)                 │
   │ RGA / LSEQ /  sequence CRDTs for text                        │
   │   Logoot / YATA (Yjs uses YATA)                              │
   └──────────────────────────────────────────────────────────────┘

   ✅ ⭐ NO CENTRAL SERVER NEEDED — true peer-to-peer possible
   ✅ Offline-first works naturally
   ✅ Convergence is mathematically guaranteed, not protocol-dependent
   ✅ Much easier to reason about and test
   ❌ Metadata overhead — every character carries an ID
   ❌ Tombstones: deleted characters must be retained (garbage
      collection is the hard part)
   ❌ Intent preservation can be subtly worse than OT in edge cases
```

```
   ⭐ WHICH ONE?

   Google Docs uses OT (built before CRDTs matured, and the
   central-server model fits their architecture).

   Figma, Linear, Notion, and most NEW collaborative products
   use CRDTs — Yjs and Automerge made them practical.

   THE INTERVIEW ANSWER: know both, and state the tradeoff.
   "OT is more compact but requires a central server and the
    transformation logic is famously error-prone. CRDTs carry
    metadata overhead but guarantee convergence structurally
    and work offline and peer-to-peer. For a new system I'd
    choose a CRDT — the correctness guarantee is worth the bytes."
```

## 5. Architecture

```
   ┌──────────┐                                   ┌──────────┐
   │ Client A │                                   │ Client B │
   │ ┌──────┐ │                                   │ ┌──────┐ │
   │ │local │ │  ⭐ edits apply LOCALLY FIRST      │ │local │ │
   │ │ doc  │ │     → typing feels instant        │ │ doc  │ │
   │ └──────┘ │                                   │ └──────┘ │
   └────┬─────┘                                   └────▲─────┘
        │ WebSocket (ops)                              │
        ▼                                              │
   ┌────────────────────────────────────────────────────────────┐
   │              COLLABORATION SERVICE                         │
   │  • one session per document (sticky routing)               │
   │  • assigns the canonical operation order                   │
   │  • transforms (OT) or merges (CRDT)                        │
   │  • broadcasts to all connected clients                     │
   │  • tracks presence (cursors, selections)                   │
   └───┬───────────────────────────────────┬────────────────────┘
       ▼                                   ▼
   ┌─────────────────┐            ┌──────────────────────┐
   │  OPERATION LOG  │            │  PRESENCE (Redis)    │
   │  append-only    │            │  ephemeral, TTL'd    │
   │  = full history │            │  never persisted     │
   └────────┬────────┘            └──────────────────────┘
            │ periodic
            ▼
   ┌─────────────────┐
   │  SNAPSHOTS      │  ⭐ so loading a doc doesn't require
   │  (materialized) │     replaying 500,000 operations
   └─────────────────┘
```

```
   ⭐ THE OPERATION LOG IS THE SOURCE OF TRUTH

   The document is a DERIVED value — a fold over the op log.
   This gives you, for free:
     • complete revision history
     • undo/redo
     • "who changed this line" attribution
     • time-travel to any point
     • recovery from any corruption

   Snapshots are purely an optimization: replay from the last
   snapshot rather than from operation zero.
```

## 6. Deep Dive — Presence

```
   ⚠️ PRESENCE IS A DIFFERENT PROBLEM FROM DOCUMENT STATE

   ┌────────────────────┬────────────────────────────────────┐
   │ DOCUMENT OPS       │ PRESENCE (cursors, selections)     │
   ├────────────────────┼────────────────────────────────────┤
   │ Must never be lost │ ⭐ Loss is FINE — it refreshes      │
   │ Must be ordered    │ Order doesn't matter               │
   │ Persisted forever  │ Ephemeral, TTL'd, never persisted  │
   │ Every op matters   │ Throttle to ~10-20/sec             │
   └────────────────────┴────────────────────────────────────┘

   → Send presence over a SEPARATE channel with different
     guarantees. Mixing them means either over-engineering
     presence or under-engineering document ops.

   ⭐ Cursor positions must be transformed too! If someone
     inserts text above my cursor, my displayed cursor position
     must shift — otherwise remote cursors drift visibly.
```

## 7. Deep Dive — Offline

```
   OFFLINE EDITING FLOW

   1. Local edits queue up, applied immediately to the local doc
   2. Reconnect
   3. Send the queued ops with the last-known server version
   4. Server transforms them against everything that happened
      while offline
   5. Server sends back the ops the client missed
   6. Client applies them, transformed against its local queue

   ⭐ WITH CRDTs THIS IS ALMOST TRIVIAL: just merge the two
     states. Convergence is structural. This is the single
     strongest argument for CRDTs in an offline-first product.

   ⚠️ EDGE CASE: offline for a month, meanwhile the document
     was heavily edited. Transformation cost grows with the
     number of concurrent ops. At some point you fall back to
     "your version is saved as a copy" rather than attempting
     a merge that would produce nonsense.
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"OT vs CRDT — which would you choose and why?"</b></summary>

Both solve the same problem: making concurrent edits converge. They differ in where the complexity lives.

OT keeps operations compact — just a position and content — and transforms each incoming operation against concurrent ones so its intent survives the position shifts. The cost is that the transformation functions are famously hard to get right; several published OT algorithms were later proven incorrect, and Google's took years to stabilize. It also requires a central server to establish canonical ordering.

CRDTs move the complexity into the data structure. Every character gets a unique orderable ID instead of a positional index, so concurrent operations commute by construction — merging is just a union and sort. Convergence is mathematically guaranteed rather than protocol-dependent. The cost is metadata overhead per character and tombstones for deleted content, which need garbage collection.

For a new system I'd choose a CRDT. The correctness guarantee is worth the bytes, offline-first works naturally, and libraries like Yjs have made them production-ready. Google Docs uses OT largely because it predates mature CRDTs and their central-server architecture already fits.
</details>

<details>
<summary><b>"How do you make typing feel instant?"</b></summary>

Apply every local edit to the local document immediately, before any network round trip. The user sees their keystroke with zero latency because nothing is waiting on the server.

The operation is then sent asynchronously. When remote operations arrive, they're transformed against the local pending queue and applied. The local document is always ahead of the server, and reconciliation happens in the background.

This is optimistic local application, and it's why both OT and CRDT exist — if you could afford to wait for the server to arbitrate every keystroke, neither would be necessary.

The visible consequence is that your own text may briefly shift position as remote edits arrive above it. That's the correct behaviour, and users tolerate it far better than input lag.
</details>

<details>
<summary><b>"How do you store the document?"</b></summary>

The operation log is the source of truth — an append-only sequence of every edit. The document itself is a derived value, a fold over that log.

That gives revision history, undo/redo, per-line attribution, time travel, and recovery from corruption, all for free rather than as separate features.

Loading a document by replaying half a million operations would be slow, so periodic snapshots materialize the document state, and loading replays only from the most recent snapshot forward.

For a document with heavy history I'd also compact the log — old operations beyond the retention window collapse into the snapshot, keeping storage bounded while preserving the history users actually access.
</details>

---

# ▶️ YouTube

> **Teaches:** the full video pipeline — ingestion, transcoding, storage tiering, delivery, and recommendation at planetary scale.

## 1. Requirements

```
   FUNCTIONAL
   • Upload video (any format, any size)
   • Watch with adaptive quality
   • Search
   • Recommendations
   • Comments, likes, subscriptions
   • Live streaming
   • Creator analytics

   NON-FUNCTIONAL
   SCALE          ~2.5B users · ~500 hours uploaded per MINUTE ·
                  ~1B hours watched per DAY
   AVAILABILITY   99.9%+
   LATENCY        Playback start < 2s
   DURABILITY     ⭐ Never lose an upload
```

## 2. Estimation

```
   UPLOADS   500 hours/min = 30,000 hours/hour = 720,000 hours/day
             At ~1 GB/hour raw for HD (much more for 4K):
             → ~720 TB/day of source material  ⭐

   TRANSCODING OUTPUT
   Each video → ~10 resolutions × multiple codecs
             → output is roughly 2-5× the source size
             → several PB/day of encoded output

   WATCHING  1B hours/day
             ÷ 86,400 → ~11.5M concurrent viewers average
             Peak maybe 25M+
             × 3 Mbps average → ⭐ ~75 Tbps peak egress

   ⭐ SAME CONCLUSION AS NETFLIX: this cannot be served from
     datacenters. Google's global edge network and peering
     arrangements are the entire delivery story.

   TRANSCODING COMPUTE
   720,000 hours/day of source, each transcoded into ~10 variants.
   Even at faster-than-realtime encoding, this is one of the
   largest compute workloads on earth. It runs on preemptible
   capacity and is scheduled around datacenter load.
```

## 3. The Upload & Transcode Pipeline

```
   ┌──────────┐
   │ Creator  │
   └────┬─────┘
        │ ⭐ RESUMABLE upload (chunked, byte-range)
        │    a 4-hour 4K upload MUST survive a dropped connection
        ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  UPLOAD SERVICE                                             │
   │  • issues an upload session ID                              │
   │  • accepts chunks, tracks which are received                │
   │  • client can query "which chunks do you have?" and resume  │
   └────────────────────────┬────────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  RAW STORAGE (durable, cheap)                               │
   │  ⭐ Persisted BEFORE any processing — durability first       │
   └────────────────────────┬────────────────────────────────────┘
                            │ emits an event
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  TRANSCODING PIPELINE (massively parallel)                  │
   │                                                             │
   │  ① SPLIT the video into chunks (~5-10 sec, at GOP           │
   │     boundaries so each chunk is independently decodable)    │
   │                                                             │
   │  ② FAN OUT — each chunk × each target variant is an         │
   │     independent job on a worker fleet                       │
   │     ⭐ THIS is what makes it tractable: a 2-hour video       │
   │       becomes ~1,440 chunks, transcoded in PARALLEL.        │
   │       Wall-clock time is bounded by ONE chunk, not the      │
   │       whole video.                                          │
   │                                                             │
   │  ③ MERGE chunks back into per-variant files                 │
   │  ④ PACKAGE into DASH/HLS with manifests                     │
   │  ⑤ VALIDATE — quality checks, A/V sync                      │
   │  ⑥ ML PASSES — Content ID matching, thumbnails,             │
   │     auto-captions, moderation classifiers                   │
   └────────────────────────┬────────────────────────────────────┘
                            ▼
   ┌─────────────────────────────────────────────────────────────┐
   │  DISTRIBUTE TO EDGE / CDN                                   │
   │  ⭐ TIERED: low resolutions first, so the video is           │
   │     watchable within minutes while 4K is still encoding     │
   └─────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE CHUNK-PARALLEL INSIGHT

   Transcoding a 2-hour video serially might take 4 hours.
   Split into 1,440 chunks processed in parallel across a
   large worker fleet, and wall-clock time drops to minutes.

   THE REQUIREMENT: chunks must split at GOP (Group of Pictures)
   boundaries — points where a keyframe starts — so each chunk
   can be decoded independently without needing the previous one.
```

## 4. Deep Dive — Storage Tiering by Popularity

```
   ⭐ VIDEO POPULARITY IS EXTREMELY LONG-TAILED

   ~1% of videos       → ~80% of all views
   ~50% of videos      → almost never watched after week one

   → Store them completely differently.

   ┌──────────────────────────────────────────────────────────────┐
   │ VIRAL / POPULAR                                              │
   │   • all resolutions pre-encoded and kept hot                 │
   │   • pushed to edge caches globally                           │
   │   • replicated heavily                                       │
   ├──────────────────────────────────────────────────────────────┤
   │ MODERATE                                                     │
   │   • common resolutions kept, exotic ones dropped             │
   │   • regional edge caching only                               │
   ├──────────────────────────────────────────────────────────────┤
   │ LONG TAIL                                                    │
   │   • ⭐ store ONE master; transcode ON DEMAND if requested     │
   │   • cold storage                                             │
   │   • a slower first play for a video nobody watches is        │
   │     an excellent trade                                       │
   └──────────────────────────────────────────────────────────────┘

   ⭐ This is the same insight as Instagram's tiering, but the
     dimension is POPULARITY rather than AGE — and the savings
     are even larger because encoded variants are expensive
     to both store and produce.
```

## 5. Deep Dive — Recommendations

```
   THE SAME TWO-STAGE FUNNEL AS INSTAGRAM, at larger scale

   ┌──────────────────────────────────────────────────────────────┐
   │ STAGE 1: CANDIDATE GENERATION                                │
   │   Billions of videos → a few hundred candidates              │
   │                                                              │
   │   Multiple parallel sources:                                 │
   │     • collaborative filtering (people like you watched...)   │
   │     • content similarity via embeddings (ANN search)         │
   │     • the creator's other videos / your subscriptions        │
   │     • trending in your region and language                   │
   │     • ⭐ each source has its own recall characteristics —     │
   │       diversity of SOURCES creates diversity of RESULTS      │
   ├──────────────────────────────────────────────────────────────┤
   │ STAGE 2: RANKING                                             │
   │   Hundreds → ~20, ordered                                    │
   │                                                              │
   │   ⭐ Optimizes for WATCH TIME, not click-through.             │
   │     This was a deliberate, consequential change — CTR         │
   │     optimization rewards clickbait; watch time rewards       │
   │     content people actually stay for.                        │
   │                                                              │
   │   Signals: predicted watch time · your history · video age · │
   │            channel affinity · session context ·              │
   │            negative feedback (weighted heavily)              │
   ├──────────────────────────────────────────────────────────────┤
   │ STAGE 3: RE-RANKING                                          │
   │   diversity · freshness · integrity filters · policy         │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE COLD START PROBLEM

   A newly uploaded video has no engagement data. How do you
   know if it's good?

   • Content-based features (title, description, thumbnail,
     visual/audio embeddings) → find similar known videos
   • Creator reputation and past performance
   • ⭐ EXPLORATION: deliberately show new content to a small
     random sample of users to gather signal
     — this is a multi-armed bandit problem: balance
     exploiting known-good content against exploring
     potentially-better unknown content
```

## 6. Deep Dive — Live Streaming

```
   ⚠️ LIVE IS A COMPLETELY DIFFERENT SYSTEM FROM VOD

   ┌────────────────────┬──────────────────────────────────────┐
   │ VOD                │ LIVE                                 │
   ├────────────────────┼──────────────────────────────────────┤
   │ Transcode offline, │ ⭐ Transcode in REAL TIME — no second │
   │ any time you like  │    chances, no re-runs               │
   │ Full CDN pre-warm  │ Content doesn't exist until it does  │
   │ Seek anywhere      │ Latency budget: 2-30 seconds         │
   │ Perfect quality    │ Must degrade gracefully under load   │
   └────────────────────┴──────────────────────────────────────┘

   PIPELINE
   Encoder (creator) ──RTMP/SRT──▶ Ingest ──▶ Real-time transcode
                                                     │
                                          ┌──────────┴──────────┐
                                          ▼                     ▼
                                   Segment + package      Recording
                                   (HLS/DASH, 2-6s)       (becomes VOD)
                                          │
                                          ▼
                                    CDN ──▶ Viewers

   ⭐ THE LATENCY / RELIABILITY TRADEOFF
     Shorter segments = lower latency but more requests and
     less buffer resilience. Low-latency HLS uses partial
     segments to get to ~2-5 seconds; standard HLS is ~15-30.
     Interactive use cases (auctions, betting) need sub-second
     and require WebRTC instead.
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you transcode 500 hours of video uploaded every minute?"</b></summary>

The key move is splitting each video into chunks at GOP boundaries — points where a keyframe starts, so each chunk decodes independently — then treating every chunk-and-variant pair as an independent job on a large worker fleet.

That turns a serial problem into an embarrassingly parallel one. A two-hour video becomes roughly 1,440 chunks; transcoded in parallel, wall-clock time is bounded by a single chunk rather than the whole video. Chunks are then merged back into per-variant files and packaged into DASH or HLS.

Two operational details matter. Transcoding runs tiered — low resolutions first, so the video is watchable within minutes while 4K is still processing. And because it's batch work that tolerates interruption, it runs on preemptible capacity scheduled around datacenter load, which makes an otherwise enormous compute bill manageable.

Crucially, the raw upload is persisted durably before any processing starts. Losing a creator's upload is unacceptable; losing a transcode job just means re-running it.
</details>

<details>
<summary><b>"How do you store this much video affordably?"</b></summary>

Tiering by popularity rather than age, because video views follow an extreme power law — roughly 1% of videos get 80% of views, and half of all uploads are essentially never watched after their first week.

Popular videos get every resolution pre-encoded, replicated heavily, and pushed to edge caches globally. Moderate videos keep common resolutions and get regional caching. For the long tail, store one master and transcode on demand if someone actually requests it.

That last decision is the big one. Pre-encoding ten variants of a video nobody watches is pure waste in both storage and compute. Accepting a slower first play for an unwatched video is an excellent trade — and if it does get requested repeatedly, it promotes into a warmer tier automatically.

Beyond tiering: modern codecs like AV1 cut bitrate substantially for popular content where the encoding cost amortizes over millions of views, and per-title encoding tunes the bitrate ladder to each video's actual complexity.
</details>

<details>
<summary><b>"How do recommendations work, and what's the cold start problem?"</b></summary>

Two stages. Candidate generation cuts billions of videos to a few hundred using several parallel cheap retrieval sources — collaborative filtering, embedding similarity via approximate nearest neighbour search, your subscriptions, regional trending. Having diverse *sources* is what creates diverse *results*.

Ranking then applies an expensive model to that shortlist, predicting watch time rather than click-through. That distinction matters enormously: optimizing click-through rewards clickbait thumbnails, while optimizing watch time rewards content people actually stay for. It was a deliberate and consequential change.

Cold start is the problem that a new upload has no engagement data. It's handled with content-based features — title, thumbnail, visual and audio embeddings — to find similar known videos, plus creator reputation as a prior.

The most important mechanism is deliberate exploration: showing new content to a small random sample of users to gather signal. That's a multi-armed bandit tradeoff, balancing exploiting known-good content against exploring potentially-better unknowns. Without it, new creators could never break through, and the system would ossify around existing winners.
</details>

---

# 📦 Dropbox

> **Teaches:** file synchronization, content-addressed deduplication, delta sync, and conflict resolution.

## 1. Requirements

```
   FUNCTIONAL
   • Sync files across devices
   • Share files/folders with others
   • Version history
   • Selective sync (don't download everything)
   • Offline access
   • Conflict handling

   NON-FUNCTIONAL
   SCALE          ~700M users, exabytes stored
   ⭐ EFFICIENCY   Bandwidth and storage are the dominant costs
   RELIABILITY    ⭐ Never lose or corrupt a file
   LATENCY        Sync should feel near-instant for small changes
```

## 2. The Core Insight — Chunking

#### 💬 Why you never sync whole files

```
   ⚠️ NAIVE: a 100 MB file changes by one byte → upload 100 MB

   ⭐ CHUNKED: split every file into blocks (Dropbox uses 4 MB).
     A one-byte change affects ONE block → upload 4 MB.
     With variable-size chunking, often far less.

   File "report.pdf" (10 MB)
   ┌────────┬────────┬────────┐
   │ block1 │ block2 │ block3 │
   │ hash:  │ hash:  │ hash:  │
   │ a3f... │ 7b2... │ c91... │
   └────────┴────────┴────────┘
              ▲
              └─ user edits page 5 → only block2's hash changes
                 → upload ONLY block2
```

### ⭐ Content-addressed storage + deduplication

```
   Blocks are stored and referenced BY THEIR HASH, not by name.

   ┌─────────────────────────────────────────────────────────────┐
   │  BLOCK STORE:   hash → bytes                                │
   │                                                             │
   │  a3f2... → [4 MB of data]                                   │
   │  7b21... → [4 MB of data]                                   │
   │  c914... → [4 MB of data]                                   │
   ├─────────────────────────────────────────────────────────────┤
   │  METADATA:  file → ordered list of block hashes             │
   │                                                             │
   │  alice/report.pdf  → [a3f2, 7b21, c914]                     │
   │  bob/report_copy.pdf → [a3f2, 7b21, c914]  ⭐ SAME BLOCKS    │
   └─────────────────────────────────────────────────────────────┘

   ⭐ GLOBAL DEDUPLICATION
   If a million users have the same PDF, it's stored ONCE.
   The upload protocol exploits this:

   Client: "I have blocks a3f2, 7b21, c914"
   Server: "I already have a3f2 and 7b21. Send only c914."

   → Uploading a file that already exists somewhere in the
     system takes seconds regardless of size.
```

```
   ⚠️ THE SECURITY PROBLEM WITH CROSS-USER DEDUPLICATION

   If deduplication works across users, an attacker can probe:
   "Does anyone have a file with this hash?" — by observing
   whether the upload is fast (dedup hit) or slow.

   That leaks information about what files exist in the system.

   MITIGATIONS:
   • Deduplicate only WITHIN a user/team boundary
   • Require proof-of-possession (prove you have the full block,
     not just the hash) before granting a dedup shortcut
   • Add randomized delays to mask timing signals
```

### Content-defined chunking

```
   ⚠️ FIXED-SIZE CHUNKING HAS A FATAL WEAKNESS

   Insert one byte at the START of a file, and EVERY subsequent
   block boundary shifts. Every hash changes. You re-upload
   the entire file.

   ┌────────┬────────┬────────┐
   │ block1 │ block2 │ block3 │   original
   └────────┴────────┴────────┘
    ↓ insert 1 byte at position 0
   ┌────────┬────────┬────────┬─┐
   │ block1'│ block2'│ block3'│…│  ⭐ EVERY block changed
   └────────┴────────┴────────┴─┘

   ✅ CONTENT-DEFINED CHUNKING (Rabin fingerprinting)

   Boundaries are chosen based on the CONTENT, not on offset:
   slide a rolling hash over the data and cut wherever the
   hash matches a pattern (e.g. low 13 bits are zero).

   Now inserting a byte shifts only the ONE block containing
   the insertion. Subsequent boundaries realign naturally
   because they depend on content, not position.
```

## 3. Architecture

```
   ┌──────────────┐                          ┌──────────────┐
   │  Client A    │                          │  Client B    │
   │  ┌────────┐  │                          │  ┌────────┐  │
   │  │ Watcher│  │ filesystem events        │  │ Watcher│  │
   │  │ Indexer│  │                          │  │ Indexer│  │
   │  │ Chunker│  │                          │  │ Chunker│  │
   │  └────────┘  │                          │  └────────┘  │
   └──────┬───────┘                          └──────▲───────┘
          │                                         │
          │ ① metadata sync (small, frequent)       │ notify
          ▼                                         │
   ┌─────────────────────────────────────────────────────────┐
   │  METADATA SERVICE                                       │
   │  • file tree, versions, permissions, block lists         │
   │  • ⭐ this is the SOURCE OF TRUTH for structure           │
   │  • sharded by user/namespace                             │
   │  • must be strongly consistent                           │
   └───────────────────────┬─────────────────────────────────┘
                           │
   ┌───────────────────────┴─────────────────────────────────┐
   │  NOTIFICATION SERVICE (long-poll / WebSocket)           │
   │  "your namespace changed, come sync"                     │
   └─────────────────────────────────────────────────────────┘

          │ ② block transfer (large, only what's missing)
          ▼
   ┌─────────────────────────────────────────────────────────┐
   │  BLOCK STORAGE  (content-addressed, hash → bytes)       │
   │  • immutable — a block never changes                    │
   │  • therefore trivially cacheable and replicable         │
   └─────────────────────────────────────────────────────────┘
```

```
   ⭐ THE METADATA / BLOCK SPLIT IS THE KEY ARCHITECTURAL DECISION

   METADATA          small, frequent, needs strong consistency,
                     needs transactions → relational database
   BLOCKS            large, immutable, content-addressed →
                     object storage, trivially replicated

   These have completely different requirements. Separating
   them lets each be optimized independently — and it's why
   the metadata service can be strongly consistent while
   block storage is eventually consistent.
```

## 4. Deep Dive — Sync Protocol

```
   ① DETECT CHANGE
      Filesystem watcher (inotify/FSEvents/ReadDirectoryChangesW)
      ⚠️ Watchers are unreliable at scale — they miss events and
        have per-directory limits. So ALSO run a periodic full
        scan as a safety net.

   ② HASH
      Chunk the file, compute block hashes
      ⭐ Compare against the local index first — if hashes are
        unchanged, do nothing. Most "changes" are touch events
        with identical content.

   ③ COMMIT METADATA
      Send the new block list + parent version to the server.
      ⭐ This is an OPTIMISTIC CONCURRENCY check:
        "I'm updating from version 7 to 8"
        → server accepts only if the current version is 7
        → otherwise: CONFLICT

   ④ UPLOAD MISSING BLOCKS
      Server responds with which hashes it doesn't have.
      Client uploads only those.

   ⑤ NOTIFY OTHER DEVICES
      Long-poll/WebSocket notification: "namespace changed"

   ⑥ OTHER DEVICES PULL
      Fetch new metadata, diff against local, download only
      the blocks they're missing, reassemble.
```

## 5. Deep Dive — Conflict Resolution

#### 💬 Why Dropbox doesn't merge

```
   ⭐ CRITICAL DESIGN DECISION: Dropbox does NOT attempt to
     merge conflicting file edits.

   WHY? Because it can't. A .docx, .psd, or .zip has internal
   structure that a byte-level merge would corrupt. Silently
   producing a corrupt file is far worse than producing two files.

   INSTEAD: keep both.
     report.pdf
     report (Alice's conflicted copy 2026-08-14).pdf

   ✅ No data is EVER lost
   ✅ The user — who understands the content — decides
   ❌ Slightly annoying UX

   ⭐ CONTRAST WITH GOOGLE DOCS: Docs CAN merge because it
     controls the document format and understands its structure.
     Dropbox handles arbitrary opaque bytes. This is the entire
     reason the two systems make opposite choices.
```

```
   THE CONFLICT DETECTION MECHANISM

   Server holds:  report.pdf @ version 7

   Alice (offline) edits from v7 → submits v8
   Bob (offline)   edits from v7 → submits v8

   Alice commits first:  server is now at v8  ✅
   Bob commits:          "update from v7" but the server is at v8
                         → ⚠️ CONFLICT DETECTED
                         → create a conflicted copy for Bob

   ⭐ This is optimistic concurrency control — exactly the same
     mechanism as an ETag/If-Match check in HTTP, or a version
     column in a database.
```

## 6. Deep Dive — Bandwidth Optimization

```
   THE STACK OF OPTIMIZATIONS, in order of impact

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. DEDUPLICATION       don't send blocks that already exist  │
   │                        ⭐ biggest win by far                  │
   ├──────────────────────────────────────────────────────────────┤
   │ 2. DELTA SYNC          send only changed blocks              │
   ├──────────────────────────────────────────────────────────────┤
   │ 3. COMPRESSION         compress blocks before transfer       │
   │                        (skip already-compressed formats)     │
   ├──────────────────────────────────────────────────────────────┤
   │ 4. LAN SYNC            ⭐ if another device on the same LAN   │
   │                        has the block, get it locally         │
   │                        — never touches the internet          │
   ├──────────────────────────────────────────────────────────────┤
   │ 5. BATCHING            many small file changes → one request │
   ├──────────────────────────────────────────────────────────────┤
   │ 6. THROTTLING          respect the user's bandwidth; back    │
   │                        off when they're on a metered link    │
   └──────────────────────────────────────────────────────────────┘
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you sync a 1 GB file efficiently when one byte changes?"</b></summary>

Content-defined chunking. The file is split into blocks whose boundaries are determined by a rolling hash over the content rather than by fixed offsets, then each block is identified by its hash.

A one-byte change alters exactly one block's hash, so only that block is uploaded — typically a few megabytes rather than a gigabyte.

Content-defined boundaries matter specifically because fixed-size chunking breaks catastrophically on insertion. Insert a byte at the start of a file and every subsequent fixed boundary shifts, so every hash changes and you re-upload everything. With content-defined boundaries, subsequent cut points realign naturally because they depend on content, not position.

On top of that, the upload protocol asks the server which block hashes it already has, so any block that exists anywhere in the system — from any user's earlier upload — is never transferred at all.
</details>

<details>
<summary><b>"How does deduplication work, and what's the security risk?"</b></summary>

Blocks are stored content-addressed, meaning the storage key *is* the hash of the content. File metadata is just an ordered list of block hashes. If a million users have the same PDF, the blocks exist once and a million metadata entries point at them.

The upload protocol exploits this: the client sends the list of hashes, the server replies with which ones it's missing, and only those are transferred. Uploading a file that already exists takes seconds regardless of size.

The security risk is that cross-user deduplication creates a side channel. An attacker can construct a file, attempt to upload it, and infer from the speed whether someone else already has it — leaking information about what content exists in the system.

The mitigations are deduplicating only within a user or team boundary, requiring proof-of-possession so a client must demonstrate it holds the full block rather than just the hash, and adding randomized delays to mask the timing signal. Most consumer services now scope deduplication to reduce this exposure.
</details>

<details>
<summary><b>"Why doesn't Dropbox merge conflicting edits?"</b></summary>

Because it can't do so safely. Dropbox handles arbitrary opaque bytes — a .docx, a .psd, a database file, a zip archive. These have internal structure, checksums, and offset tables. A byte-level merge would produce a file that looks plausible and is actually corrupt.

Silently corrupting a file is far worse than producing two files, so Dropbox keeps both: the original plus a conflicted copy attributed to the user and timestamped. No data is ever lost, and the person who understands the content decides.

The contrast with Google Docs is instructive. Docs *can* merge because it controls the document format and understands its structure — it knows what a character insertion means. Dropbox has no such knowledge. The two systems make opposite choices for exactly this reason, and recognizing that distinction is the real answer to the question.

Detection itself is straightforward optimistic concurrency: each update declares the version it's based on, and the server rejects it if that's no longer current — the same mechanism as an HTTP If-Match header or a version column in a database.
</details>

---

# 🏠 Airbnb

> **Teaches:** geospatial + faceted search, availability calendars, and preventing double-booking.

## 1. Requirements

```
   FUNCTIONAL
   • Search listings by location, dates, guests, filters
   • View listing details, photos, reviews
   • Book with instant confirmation or host approval
   • Host manages calendar and pricing
   • Messaging between guest and host
   • Payments (with a hold until check-in)

   NON-FUNCTIONAL
   SCALE          ~7M listings, ~150M users, ~2M nights booked/day
   ⭐ CONSISTENCY  A listing must NEVER be double-booked
   LATENCY        Search < 500ms
   AVAILABILITY   99.99% — search and booking are revenue-critical
```

## 2. The Core Problems

```
   ① SEARCH is multi-dimensional and expensive
      geography + dates + guests + price + amenities + ratings
      ⭐ Availability is a RANGE query over a calendar, not a
        simple equality filter — that's what makes it hard

   ② BOOKING must be strongly consistent
      Two guests booking the same dates is a real-world failure
      that costs money and trust
```

## 3. Deep Dive — Search

```
   ┌─────────────────────────────────────────────────────────────┐
   │  THE PIPELINE                                               │
   │                                                             │
   │  ① GEO FILTER                                               │
   │     Map viewport / city → bounding box or geohash prefixes  │
   │     → narrows millions of listings to thousands             │
   │                                                             │
   │  ② AVAILABILITY FILTER  ⭐ the expensive one                 │
   │     "available for ALL nights in [check-in, check-out)"     │
   │                                                             │
   │  ③ ATTRIBUTE FILTERS                                        │
   │     guests ≥ N · price range · amenities · instant book     │
   │                                                             │
   │  ④ RANK                                                     │
   │     quality · price competitiveness · booking likelihood ·  │
   │     personalization · host responsiveness                   │
   │                                                             │
   │  ⑤ PAGINATE + return                                        │
   └─────────────────────────────────────────────────────────────┘
```

### ⭐ The availability data structure

```
   ⚠️ THE NAIVE APPROACH FAILS

   A row per listing per date:
     (listing_id, date, available)
   7M listings × 365 days = 2.5 BILLION rows.
   And a search asking "available all of Aug 10-17" needs to
   verify 7 consecutive rows per listing across thousands of
   listings. Far too slow.

   ✅ BITMAP APPROACH

   One bitmap per listing, one bit per day, covering ~2 years:
     730 bits ≈ 92 bytes per listing
     7M listings × 92 bytes ≈ 650 MB total  ⭐ fits in memory

   Availability check for a date range becomes a BITWISE AND
   against a mask for those dates:

     listing bitmap  1 1 1 0 0 1 1 1 1 1 1 0 ...
     query mask      0 0 0 0 0 1 1 1 0 0 0 0    (Aug 10-12)
     AND             0 0 0 0 0 1 1 1 0 0 0 0
                     └─ equals the mask → AVAILABLE ✅

   ⭐ This is a handful of CPU instructions per listing instead
     of a database query. Millions of listings can be filtered
     in milliseconds.
```

```
   ARCHITECTURE

   ┌──────────────┐
   │   SEARCH     │
   │   SERVICE    │
   └──────┬───────┘
          ├──────────────────┬─────────────────────┐
          ▼                  ▼                     ▼
   ┌─────────────┐    ┌─────────────┐    ┌──────────────────┐
   │ Elasticsearch│    │ Availability│    │  Pricing Service │
   │ (attributes, │    │ bitmap index│    │  (dynamic rates) │
   │  geo, text)  │    │ (in-memory) │    └──────────────────┘
   └─────────────┘    └─────────────┘
          ▲                  ▲
          │ CDC              │ CDC
   ┌──────┴──────────────────┴───────────────────────────────┐
   │  SOURCE OF TRUTH: listings + bookings (Postgres)        │
   └─────────────────────────────────────────────────────────┘

   ⭐ Search indexes are DERIVED and eventually consistent.
     A listing appearing in search 30 seconds late is fine.
     A double-booking is not. Different guarantees for
     different systems.
```

## 4. Deep Dive — Preventing Double-Booking

#### 💬 The core correctness requirement

```
   ⚠️ THE RACE

   Guest A and Guest B both search, both see Aug 10-15 available,
   both click "Book" within the same second.

   Without protection, both bookings succeed. One guest arrives
   to find someone else in the property. This is a catastrophic
   product failure.
```

```
   ⭐ SOLUTION: DATABASE-ENFORCED EXCLUSION

   The database — not application logic — must guarantee this.

   POSTGRES EXCLUSION CONSTRAINT (the cleanest form):

   CREATE TABLE bookings (
     id           bigserial PRIMARY KEY,
     listing_id   bigint NOT NULL,
     guest_id     bigint NOT NULL,
     stay         daterange NOT NULL,
     status       text NOT NULL,
     EXCLUDE USING gist (
       listing_id WITH =,
       stay       WITH &&        -- ⭐ && means "overlaps"
     ) WHERE (status IN ('confirmed', 'pending'))
   );

   → The DATABASE physically cannot store two overlapping
     confirmed bookings for the same listing.
   → The second concurrent insert FAILS with a constraint
     violation, which the application turns into a clean
     "no longer available" message.
   → No application-level locking required.
   → Correct even under arbitrary concurrency and retries.
```

```
   ⚠️ IF YOUR DATABASE LACKS RANGE EXCLUSION CONSTRAINTS

   Fall back to explicit locking:

   BEGIN;
     SELECT * FROM listings WHERE id = :id FOR UPDATE;  -- lock

     SELECT 1 FROM bookings
     WHERE listing_id = :id
       AND status IN ('confirmed','pending')
       AND daterange(check_in, check_out) && :requested;
     -- if any row → abort, unavailable

     INSERT INTO bookings (...);
   COMMIT;

   ⭐ The row lock on the LISTING serializes all booking attempts
     for that listing. Contention is per-listing, which is
     acceptable because a single listing gets few simultaneous
     booking attempts.
```

### The booking state machine

```
   ┌──────────┐  guest submits   ┌──────────┐
   │ INITIATED│─────────────────▶│ PENDING  │  ⭐ dates are HELD
   └──────────┘                  └────┬─────┘     during this window
                                      │
              ┌───────────────────────┼──────────────────┐
        host declines /         host accepts /        payment
        timeout expires         instant book          fails
              ▼                       ▼                  ▼
        ┌──────────┐            ┌──────────┐      ┌──────────┐
        │ DECLINED │            │CONFIRMED │      │  FAILED  │
        └──────────┘            └────┬─────┘      └──────────┘
                                     │
                          ┌──────────┼──────────┐
                     cancelled    checked in
                          ▼          ▼
                   ┌──────────┐ ┌──────────┐
                   │CANCELLED │ │ COMPLETED│
                   └──────────┘ └──────────┘

   ⭐ THE PENDING HOLD IS ESSENTIAL
     While a request awaits host approval or payment
     authorization, the dates must be blocked — otherwise
     another guest books them and you have to unwind an
     approved reservation.

     But the hold must EXPIRE. A pending request that never
     resolves would block the calendar forever. A background
     job releases expired holds.
```

## 5. Deep Dive — Pricing & Ranking

```
   DYNAMIC PRICING (host-facing "Smart Pricing")

   Inputs:
     • local demand (searches vs available supply for those dates)
     • seasonality and day-of-week patterns
     • local events (concerts, conferences — big multipliers)
     • comparable listings' prices and occupancy
     • lead time (last-minute discounting)
     • the host's own historical acceptance of suggestions

   ⭐ Constraint: hosts set floors and ceilings. The system
     SUGGESTS; it doesn't unilaterally control the host's price.
     Trust matters more than optimization here.
```

```
   SEARCH RANKING — optimizing for BOOKINGS, not clicks

   ⭐ The objective is booking probability × value, not
     click-through. A listing that gets clicked but never
     booked is worse than useless.

   Signals:
     • quality: reviews, ratings, photo quality, completeness
     • host: response rate, cancellation history, superhost status
     • price competitiveness vs similar listings
     • personalization: your past bookings and searches
     • ⭐ FAIRNESS: new listings need exposure to accumulate
       reviews, or the marketplace ossifies around incumbents
       → deliberate exploration budget, same bandit tradeoff
         as YouTube's cold start
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you prevent double-booking?"</b></summary>

The database must enforce it, not application logic — because application-level checks always have a race window between "check availability" and "insert booking."

In Postgres the cleanest form is an exclusion constraint: a GiST index on listing ID with equality and the stay daterange with the overlap operator, filtered to active statuses. The database physically cannot store two overlapping confirmed bookings for the same listing. The second concurrent insert fails with a constraint violation, which the application translates into a clean "no longer available" message.

Without range exclusion support, the fallback is `SELECT ... FOR UPDATE` on the listing row to serialize booking attempts for that listing, then check and insert inside the same transaction. Contention is per-listing, which is fine — a single property doesn't get thousands of simultaneous booking attempts.

The other essential piece is the pending hold. While a request awaits host approval or payment authorization, the dates must be blocked, or another guest books them and you have to unwind an approved reservation. But the hold needs an expiry, with a background job releasing stale ones — otherwise an abandoned request blocks a calendar indefinitely.
</details>

<details>
<summary><b>"How do you make availability search fast across millions of listings?"</b></summary>

A row per listing per date is around 2.5 billion rows for seven million listings over a year, and checking "available for all seven nights" means verifying consecutive rows across thousands of candidates. Far too slow for a 500ms search budget.

Instead, represent each listing's availability as a bitmap — one bit per day over roughly two years, which is about 92 bytes per listing. Seven million listings is around 650 megabytes total, comfortably in memory.

An availability check then becomes a bitwise AND between the listing's bitmap and a mask for the requested dates. If the result equals the mask, every requested night is free. That's a handful of CPU instructions rather than a database query, so millions of listings can be filtered in milliseconds.

The bitmap index is derived from the booking source of truth via change data capture, and it's eventually consistent — which is fine, because a listing appearing in search slightly late is acceptable while a double-booking is not. The correctness guarantee lives in the database constraint, not the search index.
</details>

<details>
<summary><b>"How would you rank search results?"</b></summary>

The objective is booking probability times value, not click-through. A listing that gets clicked but never booked wastes the guest's time and the host's response capacity.

The signals split into a few groups. Quality signals — review count and rating, photo quality, listing completeness. Host reliability — response rate, cancellation history, superhost status, since a cancelled booking is a severe negative experience. Price competitiveness relative to comparable listings for those dates. And personalization from the guest's past bookings and current search behaviour.

The part people miss is fairness. New listings have no reviews and no booking history, so a pure engagement-optimizing ranker would never surface them, and the marketplace would ossify around incumbents. That requires a deliberate exploration budget — showing new listings to some searches to let them accumulate signal.

That's the same multi-armed bandit tradeoff as YouTube's cold start, and it's a genuine business necessity in a two-sided marketplace: without new supply gaining traction, the supply side eventually stops growing.
</details>

---

# 💳 Stripe

> **Teaches:** money. Exactly-once semantics, idempotency, ledgers, and the highest correctness bar in software.

## 1. Requirements

```
   FUNCTIONAL
   • Accept card payments
   • Handle authorization, capture, refund
   • Recurring subscriptions
   • Payouts to merchants
   • Webhooks for asynchronous events
   • Dispute/chargeback handling

   NON-FUNCTIONAL
   ⭐ CORRECTNESS  Absolute. A double charge is unacceptable.
                   Money must never be created or destroyed.
   AVAILABILITY    99.999% — downtime is directly lost revenue
                   for every merchant
   LATENCY         < 1s for authorization
   AUDITABILITY    ⭐ Every state change permanently traceable
   COMPLIANCE      PCI-DSS, SOX, regional financial regulation
```

## 2. The Core Problem — Exactly-Once in an Unreliable World

#### 💬 The fundamental issue

```
   ⚠️ THE NETWORK CANNOT TELL YOU WHAT HAPPENED

   Client                 Network              Stripe
     │  POST /charges ──────────────────────────▶│
     │                                           │  ✅ card charged
     │              ✗ response lost               │
     │  (timeout)                                │
     │                                           │
     │  Did it work? THE CLIENT CANNOT KNOW.     │

   If the client retries: double charge.  ❌
   If the client doesn't retry: possibly no charge, and
     the customer's order is lost.  ❌

   ⭐ NEITHER OPTION IS ACCEPTABLE. This is why idempotency
     keys exist, and why every serious payment API has them.
```

## 3. Deep Dive — Idempotency

```
   THE PROTOCOL

   POST /v1/charges
   Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
   { "amount": 5000, "currency": "usd", "source": "tok_visa" }

   ⭐ The key is CLIENT-GENERATED. Only the client knows that
     two requests represent the same intent.
```

```python
async def create_charge(body: ChargeIn, idempotency_key: str):
    fingerprint = sha256(canonical_json(body))

    # ① ATOMIC CLAIM — the unique index IS the concurrency control
    try:
        async with db.transaction():
            await db.execute(
                "INSERT INTO idempotency_keys (key, fingerprint, status) "
                "VALUES ($1, $2, 'in_progress')",
                idempotency_key, fingerprint)
    except UniqueViolation:
        rec = await db.fetchone(
            "SELECT * FROM idempotency_keys WHERE key = $1", idempotency_key)

        # ② SAME KEY, DIFFERENT PAYLOAD → client bug, surface it
        if rec.fingerprint != fingerprint:
            raise Conflict("Idempotency-Key reused with a different payload")

        # ③ STILL RUNNING → don't execute concurrently
        if rec.status == 'in_progress':
            raise Conflict("Request in progress; retry shortly")

        # ④ ⭐ REPLAY THE STORED RESPONSE — not just "already done"
        return json.loads(rec.response_body)

    # ⑤ FIRST TIME — do the real work, store the response atomically
    async with db.transaction():
        charge = await process_payment(body)
        await db.execute(
            "UPDATE idempotency_keys SET status='completed', "
            "response_body=$2, response_status=201 WHERE key=$1",
            idempotency_key, json.dumps(charge))
    return charge
```

```
   ⭐ THE FOUR DETAILS THAT MATTER

   1. STORE AND REPLAY THE FULL RESPONSE
      Not a "seen" flag. The retry must receive the identical
      body and status code, or the client can't proceed.

   2. FINGERPRINT THE PAYLOAD
      Same key with different content is a client bug. Return
      409 rather than silently returning the wrong result.

   3. HANDLE THE IN-FLIGHT CASE
      Two simultaneous retries must not both execute. The
      'in_progress' state plus the unique index handles this.

   4. EXPIRE KEYS (~24h)
      Bounded storage; long enough to cover any realistic retry.
```

## 4. Deep Dive — The Ledger

#### 💬 Why payments use double-entry bookkeeping

```
   ⭐ MONEY IS NEVER "UPDATED". IT IS MOVED.

   ❌ NEVER DO THIS
      UPDATE accounts SET balance = balance - 100 WHERE id = 1;
      UPDATE accounts SET balance = balance + 100 WHERE id = 2;

      • no record of WHY
      • no way to audit
      • a partial failure loses or creates money
      • you can't answer "what was the balance last Tuesday?"

   ✅ DOUBLE-ENTRY LEDGER — append-only, immutable

   ┌──────────────────────────────────────────────────────────────┐
   │ ledger_entries (append-only, NEVER updated or deleted)       │
   │                                                              │
   │  id │ txn_id │ account       │ direction │ amount │ currency │
   │ ────┼────────┼───────────────┼───────────┼────────┼───────── │
   │  1  │ txn_A  │ customer_card │  DEBIT    │  5000  │  usd     │
   │  2  │ txn_A  │ merchant_bal  │  CREDIT   │  4855  │  usd     │
   │  3  │ txn_A  │ stripe_fees   │  CREDIT   │   145  │  usd     │
   │                                                              │
   │  ⭐ INVARIANT: for every txn_id,                              │
   │       SUM(debits) == SUM(credits)                            │
   │     Enforced by a constraint or a checked transaction.       │
   │     Money is CONSERVED. It cannot be created or destroyed.   │
   └──────────────────────────────────────────────────────────────┘

   BALANCE = SUM of entries for that account.
   Cached/materialized for speed, but always RECONSTRUCTIBLE
   from the entries. The cache can be wrong; the log cannot.
```

```
   ⭐ WHY APPEND-ONLY IS NON-NEGOTIABLE

   • Complete audit trail — regulators require this
   • Time travel: "what was this balance on any past date?"
   • Corrections are REVERSING ENTRIES, never edits
     (an error stays visible, with a compensating entry —
      that's how accounting has worked for 500 years)
   • Bugs can be diagnosed from history
   • Reconciliation against bank statements is possible
```

## 5. Deep Dive — The Payment Flow

```
   ① AUTHORIZATION (hold funds, don't move them)
      Merchant → Stripe → Card network → Issuing bank
      Bank checks funds/fraud → approves → funds are HELD
      ⭐ Typically valid ~7 days

   ② CAPTURE (actually take the money)
      Often immediate; separated for use cases like
      "charge when the item ships"

   ③ SETTLEMENT (batch, T+1 or T+2)
      Networks settle in batches. ⭐ The money doesn't actually
      move in real time — authorization is a promise.

   ④ PAYOUT (Stripe → merchant's bank)
      On the merchant's schedule, minus fees and reserves

   ⑤ DISPUTES / CHARGEBACKS (up to 120 days later!)
      ⭐ A transaction is not truly final for MONTHS.
        The data model must handle a reversal of something
        that settled long ago.
```

```
   ⚠️ WHY THIS MATTERS FOR THE DESIGN

   • A payment has a LONG lifecycle with many possible reversals
   • State must be modeled explicitly, never inferred
   • Every state transition must be logged with a reason
   • ⭐ You cannot delete or archive payment data aggressively —
     a chargeback four months later needs the full history
```

## 6. Deep Dive — Webhooks

```
   ⭐ WEBHOOKS ARE AN API YOU PROVIDE TO SOMEONE ELSE'S SERVER

   That inverts every assumption: you don't control their
   availability, their latency, or their correctness.

   ┌──────────────────────────────────────────────────────────────┐
   │ DELIVERY: at-least-once, with exponential backoff            │
   │   retries over hours to days (e.g. 1m, 5m, 30m, 2h, 12h)     │
   │   ⭐ consumers MUST dedupe on event_id                        │
   ├──────────────────────────────────────────────────────────────┤
   │ SECURITY: HMAC-SHA256 over timestamp + RAW body              │
   │   Stripe-Signature: t=1723640400,v1=5257a8...                │
   │   ⭐ Sign the RAW BYTES before parsing — re-serializing JSON  │
   │     changes whitespace and key order and breaks the signature│
   │   ⭐ Timestamp inside the signed payload (±5 min) prevents    │
   │     replay attacks                                           │
   ├──────────────────────────────────────────────────────────────┤
   │ ORDERING: ⭐ NOT GUARANTEED                                   │
   │   Events can arrive out of order. Each carries the full      │
   │   object state, so consumers apply the latest version        │
   │   rather than reconstructing from a sequence of deltas.      │
   ├──────────────────────────────────────────────────────────────┤
   │ OPERATIONS                                                   │
   │   • dashboard showing failed deliveries + the reason         │
   │   • manual replay                                            │
   │   • auto-disable endpoints failing continuously              │
   │   • the sender is a QUEUE CONSUMER, never inline with the    │
   │     triggering request — a slow customer endpoint must not   │
   │     slow down Stripe's own API                               │
   └──────────────────────────────────────────────────────────────┘
```

```python
def verify_signature(raw_body: bytes, header: str, secret: str) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    ts, sig = parts["t"], parts["v1"]
    if abs(time.time() - int(ts)) > 300:                    # replay window
        return False
    expected = hmac.new(secret.encode(),
                        f"{ts}.".encode() + raw_body,       # ⭐ RAW bytes
                        hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, sig)               # ⭐ constant-time
```

## 7. Deep Dive — Availability at Five Nines

```
   ⭐ EVERY MINUTE OF DOWNTIME IS LOST REVENUE FOR EVERY MERCHANT

   • Multi-region active-active for the API layer
   • ⭐ The ledger is the hard part — it needs strong consistency,
     so it can't be trivially multi-master. Typically:
       - a single writer region per account/shard
       - synchronous replication within a region
       - carefully orchestrated regional failover
   • Graceful degradation: if fraud scoring is unavailable, fall
     back to conservative rules rather than rejecting everything
   • ⭐ Card network dependencies are outside your control —
     circuit breakers and fallback routing between acquirers
   • Extensive canary and progressive rollout — a bad deploy to
     a payments system is a financial incident, not just an outage
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you guarantee a customer is never double-charged?"</b></summary>

Client-generated idempotency keys, because only the client knows that two requests represent the same intent — the network can't tell you whether a timed-out request succeeded.

Server side, the key is inserted into a table with a unique constraint inside a transaction. That unique index *is* the concurrency control, so two simultaneous retries can't both proceed.

Four details make it actually work. You store and replay the full response, not just a "seen" flag — the retry must get the identical body and status or the client can't proceed. You fingerprint the request payload and return 409 if the same key arrives with different content, because that's a client bug worth surfacing rather than silently returning the wrong result. You handle the in-flight case with an 'in_progress' state so a retry arriving mid-execution gets told to wait. And keys expire after about a day to bound storage.

Underneath, the ledger provides a second layer of protection: it's append-only double-entry, so even if something went wrong, the money movement is fully traceable and reversible via compensating entries rather than being silently incorrect.
</details>

<details>
<summary><b>"Why double-entry bookkeeping instead of a balance column?"</b></summary>

A balance column tells you the current number and nothing else. You can't audit it, you can't explain how it got there, you can't answer what it was last Tuesday, and a partial failure between two updates either creates or destroys money.

Double-entry makes every money movement an immutable pair of entries — a debit and a credit — sharing a transaction ID, with the invariant that debits equal credits for every transaction. Money is conserved by construction; it can't be created or destroyed by a bug.

The balance becomes a derived value: the sum of entries for that account. You materialize it for speed, but it's always reconstructible from the log. The cache can be wrong; the log cannot.

The properties that follow are exactly what a financial system needs. Complete audit trail, which regulators require. Time travel to any historical balance. Corrections as reversing entries rather than edits, so an error stays visible with a compensating entry — which is how accounting has worked for five hundred years. And reconciliation against bank statements becomes possible because you have the full movement history.
</details>

<details>
<summary><b>"Design the webhook system."</b></summary>

The framing that matters is that webhooks are an API you provide to someone else's server, which inverts your assumptions — you control neither their availability nor their correctness.

Delivery is at-least-once with exponential backoff over a long window, something like a minute to twelve hours across several attempts. Every event carries a unique ID and consumers are documented as needing to dedupe on it, because at-least-once guarantees duplicates will happen.

Security is HMAC-SHA256 over a timestamp concatenated with the raw request body. Signing the raw bytes before any parsing is essential — re-serializing JSON changes whitespace and key ordering and breaks verification. The timestamp inside the signed payload with a five-minute tolerance prevents replay.

Ordering is explicitly not guaranteed, so each event carries the full object state rather than a delta. Consumers apply the latest version they've seen rather than trying to reconstruct from a sequence.

Operationally you need a dashboard of failed deliveries with reasons, manual replay, and auto-disabling of endpoints that fail continuously. And the sender must be a queue consumer rather than inline with the triggering request — a slow customer endpoint must never slow down your own API.
</details>

<details>
<summary><b>"A payment succeeded at the bank but your database write failed. Now what?"</b></summary>

This is the dual-write problem in its most consequential form, and it's why payment systems are built around reconciliation rather than assuming writes succeed.

First, ordering matters: record the *intent* durably before calling the card network. So there's always a local record saying "we attempted this charge with this idempotency key," even if the outcome is unknown.

Second, the outcome is then reconciled rather than assumed. If the database write of the result fails, the payment sits in an indeterminate state, and a reconciliation job queries the card network for the authoritative status of that transaction and repairs the local record. Payment processors provide exactly this lookup for exactly this reason.

Third, retries are safe because the idempotency key propagates down to the network call, so re-attempting can't double-charge.

The general principle is that in payments you never trust a single write. Every day, settlement files from the networks are reconciled against your ledger, and discrepancies are investigated. The system is designed on the assumption that individual operations will fail in ambiguous ways, and correctness comes from reconciliation loops rather than from any single operation being atomic across systems.
</details>

---

## 📋 Volume Summary

```
╔══════════════════════════════════════════════════════════════════════╗
║                   CASE STUDIES VOL. 2 — RECALL                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ GOOGLE DOCS  concurrent edit convergence                             ║
║   OT: transform ops against concurrent ones · compact but            ║
║       error-prone · needs a central server                           ║
║   CRDT: unique IDs per char → ops COMMUTE · converges structurally · ║
║       works offline/P2P · metadata + tombstone cost                  ║
║   ⭐ apply local edits FIRST (instant typing), reconcile after        ║
║   ⭐ op log is the source of truth; document is a fold over it        ║
║   presence = separate channel, lossy, ephemeral, throttled           ║
╠══════════════════════════════════════════════════════════════════════╣
║ YOUTUBE  ⭐ chunk-parallel transcoding at GOP boundaries              ║
║   → wall-clock bounded by ONE chunk, not the whole video             ║
║   persist raw BEFORE processing · tiered encode (low res first)      ║
║   ⭐ tier storage by POPULARITY: long tail = 1 master, transcode      ║
║     on demand                                                        ║
║   ranking optimizes WATCH TIME not CTR · cold start = exploration    ║
║   live is a different system: real-time transcode, latency budget    ║
╠══════════════════════════════════════════════════════════════════════╣
║ DROPBOX  ⭐ content-defined chunking (Rabin) — fixed-size breaks on   ║
║     insertion                                                        ║
║   content-addressed blocks → global dedup ("which hashes do you      ║
║     have?") ⚠️ dedup leaks existence → scope it, proof-of-possession  ║
║   ⭐ METADATA (small/consistent/relational) vs BLOCKS (large/         ║
║     immutable/object store) — separate them                          ║
║   ⭐ NEVER merges files — arbitrary bytes can't be safely merged;     ║
║     conflicted copy instead. Contrast with Docs, which owns the format║
║   optimistic concurrency on version = conflict detection             ║
╠══════════════════════════════════════════════════════════════════════╣
║ AIRBNB  ⭐ availability BITMAP (1 bit/day, ~92B/listing, 650MB total) ║
║     → bitwise AND instead of a DB query, millions filtered in ms     ║
║   ⭐ double-booking prevented by a DB EXCLUSION CONSTRAINT            ║
║     (gist: listing = AND daterange &&) — not application logic       ║
║   pending hold blocks dates + MUST expire                            ║
║   search index derived + eventually consistent; booking strongly so  ║
║   ranking optimizes BOOKINGS not clicks · exploration for new supply ║
╠══════════════════════════════════════════════════════════════════════╣
║ STRIPE  ⭐ idempotency key: client-generated, unique index = the lock,║
║     STORE AND REPLAY the response, fingerprint payload → 409,        ║
║     handle in-flight, expire ~24h                                    ║
║   ⭐ DOUBLE-ENTRY APPEND-ONLY LEDGER — debits == credits per txn      ║
║     balance is DERIVED · corrections are REVERSING ENTRIES           ║
║   lifecycle: authorize → capture → settle (T+1/2) → dispute (120d!)  ║
║   webhooks: HMAC over timestamp + RAW bytes · at-least-once ·        ║
║     no ordering → send full state · queue-driven, never inline       ║
║   ⭐ correctness comes from RECONCILIATION LOOPS, not atomic writes   ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Case Studies Vol. 3 →](05-case-studies-3.md) · **Related:** [Vol. 1](03-case-studies-1.md) · [Framework](02-framework.md)
