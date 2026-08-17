# 🏗️ Case Studies Vol. 1 — Uber · Netflix · Twitter · WhatsApp · Instagram

> Five systems, each teaching a different core problem. Read for the *mechanism*, not the diagram — the goal is to recognize these patterns when a novel problem wears them as a costume.

**Prerequisite:** [Framework](02-framework.md)

| System | The defining problem it teaches |
|---|---|
| [Uber](#-uber) | Real-time geospatial matching |
| [Netflix](#-netflix) | Massive-scale content delivery |
| [Twitter](#-twitter) | Fan-out and the celebrity problem |
| [WhatsApp](#-whatsapp) | Persistent connections at scale |
| [Instagram](#-instagram) | Feed ranking + media pipeline |

---

# 🚗 Uber

> **Teaches:** geospatial indexing, real-time matching, state machines under unreliable networks.

## 1. Requirements

### Functional
```
   RIDER                              DRIVER
   • Request a ride (pickup, dest)    • Go online/offline
   • See nearby drivers on a map      • Receive ride offers
   • Track the driver in real time    • Accept/decline
   • Get fare estimate                • Navigate, update status
   • Pay, rate                        • Get paid
```

### Out of scope (state this)
Uber Eats, pooling, scheduled rides, driver onboarding, pricing algorithm internals.

### Non-functional
```
   SCALE          ~130M monthly users, ~6M drivers, ~25M trips/day
   LATENCY        Matching < 5 seconds. Location updates < 1 sec.
   AVAILABILITY   99.99% — a rider stranded is a real-world failure
   CONSISTENCY    ⭐ CRITICAL: a driver must NEVER be assigned two
                  rides simultaneously. This is a strong-consistency
                  requirement in an otherwise eventually-consistent
                  system.
   GEO            Global, but ⭐ every trip is LOCAL — a rider in
                  Delhi never needs data from São Paulo.
                  → this is the key architectural insight
```

## 2. Estimation

```
   DRIVERS ONLINE          ~1M concurrent at peak
   LOCATION UPDATES        every 4 seconds
     → 1M / 4 = 250,000 writes/sec  ⭐ this is the dominant load

   TRIPS                   25M/day = ~290/sec average
                                   = ~1,000/sec peak
   MATCHING SEARCHES       ~10× trips (many searches, fewer bookings)
                           = ~10,000/sec peak

   LOCATION DATA           250K/sec × 50 bytes = 12.5 MB/sec
                           = ~1 TB/day raw
   ⭐ IMPLICATION: location writes dwarf everything else by 250×.
     The location system must be a separate, specialized path —
     it cannot go through the same database as trips.
```

## 3. The Core Problem — Geospatial Matching

#### 💬 Why "find nearby drivers" is hard

The naive approach is a SQL query:

```sql
SELECT * FROM drivers
WHERE lat BETWEEN ? AND ? AND lng BETWEEN ? AND ?;
```

This fails for three reasons. First, a B-tree index on `(lat, lng)` can only efficiently filter on `lat` — the second column can't be range-scanned independently, so you scan a whole latitude band. Second, at 250K writes/sec the index is constantly being rebuilt. Third, a lat/lng box isn't a circle, and Earth isn't flat, so the math is wrong near the poles and at the antimeridian.

The solution is to **convert 2D space into 1D keys** so that nearby points have nearby keys.

### Approach A — Geohash

```
   Recursively divide the world in half, alternating longitude
   and latitude, and record each choice as a bit.

   ┌───────────────┬───────────────┐
   │               │               │      Level 1: 2 cells
   │       0       │       1       │      (split longitude)
   │               │               │
   └───────────────┴───────────────┘

   ┌───────┬───────┬───────┬───────┐
   │  00   │  01   │  10   │  11   │      Level 2: 4 cells
   ├───────┼───────┼───────┼───────┤      (now split latitude)
   │  00   │  01   │  10   │  11   │
   └───────┴───────┴───────┴───────┘

   Interleave the bits, encode in base32 → "9q8yyk8ytpxr"

   ⭐ THE MAGIC: a shared PREFIX means physical proximity.
     "9q8yy" covers ~1 km²
     "9q8yyk" covers ~150 m²

   So "find everything near me" becomes a PREFIX SCAN — which
   a B-tree does perfectly.
```

```
   PRECISION TABLE
   ┌────────┬──────────────────┐
   │ chars  │ cell size        │
   ├────────┼──────────────────┤
   │   4    │ ~20 km           │
   │   5    │ ~2.4 km          │
   │   6    │ ~600 m           │  ← typical for driver matching
   │   7    │ ~76 m            │
   │   8    │ ~19 m            │
   └────────┴──────────────────┘
```

⚠️ **The geohash boundary problem:**

```
   Two points 10 meters apart can have COMPLETELY different
   geohashes if they straddle a cell boundary:

   ┌─────────────┬─────────────┐
   │             │             │
   │   "9q8yy"   │  "9q8yz"    │
   │          ●  │  ●          │   ← 10m apart, no shared prefix!
   │             │             │
   └─────────────┴─────────────┘

   FIX: always query the target cell PLUS its 8 neighbours.
   Most geohash libraries provide a `neighbors()` function.
```

### Approach B — S2 (Google) and H3 (Uber's actual choice)

```
   S2 (Google)                      H3 (Uber) ⭐
   Projects Earth onto a cube,      HEXAGONAL grid
   then uses a Hilbert curve

   ✅ Better locality than geohash   ✅ ⭐ All 6 neighbours are
   ✅ Handles poles correctly           EQUIDISTANT — squares have
   ✅ Cells are near-uniform area       corner-neighbours that are
                                        1.41× farther than edge ones
                                     ✅ Better for movement modeling,
                                        flow, and coverage
                                     ✅ 16 resolution levels
```

```
   WHY HEXAGONS FOR RIDESHARING

   SQUARE GRID                      HEX GRID
   ┌───┬───┬───┐                      ⬡   ⬡
   │ ▲ │ ▲ │ ▲ │                    ⬡  ●  ⬡      all 6 neighbours
   ├───┼───┼───┤                      ⬡   ⬡      are exactly the
   │ ▲ │ ● │ ▲ │                                 same distance
   ├───┼───┼───┤
   │ ▲ │ ▲ │ ▲ │   diagonal neighbours are
   └───┴───┴───┘   41% farther than edge ones

   For "how far is the nearest driver" and for modeling
   supply/demand flow across a city, uniform neighbour
   distance matters a lot.
```

## 4. Architecture

```
   ┌──────────┐                              ┌──────────┐
   │  RIDER   │                              │  DRIVER  │
   │   app    │                              │   app    │
   └────┬─────┘                              └────┬─────┘
        │ HTTPS + WebSocket                       │ WebSocket
        ▼                                         ▼
   ┌─────────────────────────────────────────────────────────┐
   │                    API GATEWAY                          │
   │              auth · rate limit · routing                │
   └───┬─────────────────┬──────────────────┬────────────────┘
       ▼                 ▼                  ▼
  ┌─────────┐    ┌──────────────┐    ┌──────────────┐
  │  TRIP   │    │  DISPATCH    │    │  LOCATION    │
  │ SERVICE │◀──▶│   SERVICE    │◀──▶│   SERVICE    │
  │         │    │  (matching)  │    │ 250K wr/sec  │
  └────┬────┘    └──────┬───────┘    └──────┬───────┘
       │                │                   │
       ▼                ▼                   ▼
  ┌─────────┐    ┌──────────────┐    ┌──────────────┐
  │ Trips DB│    │ Supply/Demand│    │ Redis Geo    │
  │(Postgres│    │   (Redis)    │    │ + in-memory  │
  │ sharded │    └──────────────┘    │ H3 index     │
  │ by city)│                        └──────────────┘
  └─────────┘
       │
       ▼ (async, via Kafka)
  ┌──────────────────────────────────────────────────────┐
  │  PAYMENTS · PRICING · ANALYTICS · FRAUD · NOTIFY     │
  └──────────────────────────────────────────────────────┘
```

### ⭐ The insight that makes it tractable: geographic sharding

```
   Every trip is LOCAL. A rider in Delhi never needs data about
   drivers in São Paulo.

   → Shard EVERYTHING by city/region.
   → Each region is an independent, self-contained system.
   → Cross-region queries essentially never happen.
   → A region can fail without affecting others.
   → You can deploy region by region.

   This turns a "global scale" problem into ~1,000 independent
   "city scale" problems, each of which fits comfortably on
   modest infrastructure.
```

## 5. Deep Dive — The Matching Flow

```
   ① RIDER REQUESTS
      POST /rides  {pickup: {lat,lng}, dest, product: "uberX"}
      → returns immediately with a request_id (202 Accepted)
      → rider's app opens/uses a WebSocket for updates

   ② FIND CANDIDATES
      • Compute the H3 cell for the pickup point
      • Fetch drivers in that cell + rings 1 and 2 (k-ring)
      • Filter: online, not on a trip, correct vehicle class,
        acceptance rate above threshold
      • Typically yields 10-50 candidates

   ③ RANK
      Not just nearest — the score blends:
        • ETA to pickup (⭐ via road network, NOT straight-line)
        • Driver rating and acceptance rate
        • Whether the driver is heading toward the destination
        • Fairness / earnings balancing
        • Predicted demand at the destination

   ④ OFFER (sequentially, or in small batches)
      Send to the top candidate, wait ~15 seconds
      → declined or timeout → next candidate
      ⭐ This is the CRITICAL SECTION — see below

   ⑤ ON ACCEPT
      • Atomically transition driver → ON_TRIP
      • Create the trip record
      • Notify both parties over WebSocket
      • Start location streaming to the rider

   ⑥ DURING THE TRIP
      Driver location → Location Service → pushed to the rider
      Trip state machine advances on driver actions

   ⑦ ON COMPLETE
      • Trip → COMPLETED, driver → AVAILABLE
      • Emit a trip.completed event to Kafka
      • Payments, receipts, ratings all happen ASYNCHRONOUSLY
```

### ⭐ Deep dive — preventing double assignment

#### 💬 The problem
Two riders request simultaneously. Both matching processes see the same driver as available. Both offer. The driver accepts one — but if the other process has already marked them assigned, you have a driver committed to two trips. In the physical world, that's a stranded customer.

```
   THIS IS A STRONG CONSISTENCY REQUIREMENT inside an otherwise
   eventually-consistent system. It must be enforced atomically.
```

**Solution: a conditional atomic state transition.**

```sql
-- The driver's state lives in ONE authoritative place.
-- The WHERE clause is the lock.
UPDATE drivers
SET    status = 'ASSIGNED',
       trip_id = :trip_id,
       assigned_at = now()
WHERE  driver_id = :driver_id
  AND  status = 'AVAILABLE';     -- ⭐ fails if someone beat us

-- 0 rows affected → someone else got them → try the next candidate
```

```
   Or in Redis, with a Lua script for atomicity:

   if redis.call('GET', KEYS[1]) == 'AVAILABLE' then
       redis.call('SET', KEYS[1], 'ASSIGNED', 'EX', 30)
       return 1
   else
       return 0
   end

   ⭐ The 30-second expiry is important: if the assigning process
     crashes after claiming the driver but before completing the
     trip creation, the driver auto-releases rather than being
     stuck forever.
```

```
   ⚠️ WHY A DISTRIBUTED LOCK IS THE WRONG TOOL HERE

   A Redis lock can't guarantee mutual exclusion if a process
   stalls past its lease (GC pause, network partition). For
   correctness-critical state you want the STATE ITSELF to be
   the concurrency control — a conditional update on the
   authoritative record. See [Caching §8](../03-backend/caching.md#8-distributed-locking).
```

### ⭐ Deep dive — the location update firehose

#### 💬 The problem
250,000 writes per second, of data that is worthless 4 seconds later. Sending this through a normal database would be absurd.

```
   ┌──────────────────────────────────────────────────────────────┐
   │ KEY INSIGHT: location data is EPHEMERAL and LOSSY-TOLERABLE.  │
   │ Missing one update out of 100 changes nothing. This frees    │
   │ you from durability requirements entirely on the hot path.   │
   └──────────────────────────────────────────────────────────────┘

   DESIGN

   Driver app
      │  batched: send every 4s, or on significant movement
      │  ⭐ adaptive: slower updates when stationary or on a
      │     highway with predictable position
      ▼
   WebSocket gateway (sticky by driver, geographically routed)
      │
      ├──▶ IN-MEMORY H3 index (the hot path)
      │      • ~1M drivers × ~100 bytes = 100 MB — fits in RAM
      │      • updated in place, no durability needed
      │      • THIS is what matching queries hit
      │
      └──▶ Kafka (the cold path, async)
             • trip replay, analytics, fraud detection
             • surge pricing input
             • driver payment verification
             • NOT in the request path
```

```
   ⭐ THE TWO-PATH SPLIT IS THE WHOLE TRICK

   Hot path:  in-memory, no durability, optimized for read latency
   Cold path: durable log, optimized for throughput and replay

   Conflating them — trying to make one system do both — is what
   makes this problem look impossible.
```

### ⭐ Deep dive — the trip state machine

```
   ┌───────────┐  rider requests  ┌───────────┐
   │ REQUESTED │─────────────────▶│ MATCHING  │
   └───────────┘                  └─────┬─────┘
                          ┌─────────────┼─────────────┐
                    no drivers      accepted      cancelled
                          ▼             ▼             ▼
                   ┌───────────┐ ┌───────────┐ ┌───────────┐
                   │  FAILED   │ │ ACCEPTED  │ │ CANCELLED │
                   └───────────┘ └─────┬─────┘ └───────────┘
                                       ▼
                                 ┌───────────┐
                                 │  ARRIVED  │  driver at pickup
                                 └─────┬─────┘
                                       ▼
                                 ┌───────────┐
                                 │IN_PROGRESS│
                                 └─────┬─────┘
                                       ▼
                                 ┌───────────┐
                                 │ COMPLETED │
                                 └───────────┘

   ⭐ WHY AN EXPLICIT STATE MACHINE MATTERS

   Mobile networks are terrible. Messages arrive late, twice,
   or out of order. An explicit state machine with allowed
   transitions means:

   • Duplicate "arrived" events are idempotent (already ARRIVED)
   • An out-of-order "completed" before "in_progress" is REJECTED
   • Every transition is logged → full auditability
   • Recovery after a crash is unambiguous
```

## 6. Surge Pricing

```
   ┌──────────────────────────────────────────────────────────┐
   │ Per H3 cell, on a rolling window:                        │
   │                                                          │
   │   demand = open ride requests                            │
   │   supply = available drivers                             │
   │   ratio  = demand / supply                               │
   │                                                          │
   │   ratio > threshold  →  multiplier                       │
   │                                                          │
   │   Effects:  ↑ price → some riders defer (demand ↓)       │
   │             ↑ earnings → drivers move in (supply ↑)      │
   │             → the market rebalances                      │
   └──────────────────────────────────────────────────────────┘

   ⚠️ Constraints that must be designed in:
   • SMOOTHING — no violent multiplier swings between refreshes
   • CAPPING — legal limits, and caps during emergencies
   • LOCKED at quote time — a rider must not be charged more
     than they were shown
   • Neighbouring cells shouldn't differ wildly (spatial smoothing)
```

## 7. Failure Modes

```
   ⚠️ NO DRIVERS AVAILABLE
      → expand the search radius progressively
      → offer a different product tier
      → show honest wait estimates, offer to notify

   ⚠️ DRIVER GOES OFFLINE MID-TRIP (tunnel, dead battery)
      → last known location + heading is retained
      → rider sees "reconnecting", not an error
      → if prolonged, support intervention is triggered
      → ⭐ the trip does NOT auto-cancel — the physical trip
        is still happening

   ⚠️ THE MATCHING SERVICE IS DOWN IN ONE REGION
      → because of geographic sharding, ONLY that region
        is affected — this is the payoff for that design

   ⚠️ PAYMENT FAILS AFTER THE TRIP
      → ⭐ trip completion is NEVER blocked on payment
      → the ride happened; that's a fact to record
      → payment retries asynchronously, and unpaid balance
        blocks the NEXT ride, not this one

   ⚠️ GPS DRIFT / SPOOFING
      → sanity-check against road network map-matching
      → reject physically impossible jumps
      → cross-check with accelerometer data
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you handle a driver being offered two rides at once?"</b></summary>

The driver's status is held in one authoritative place, and assignment is a conditional atomic update: set status to ASSIGNED **only where the current status is AVAILABLE**. If zero rows are affected, someone else won the race and we move to the next candidate.

I'd deliberately avoid a distributed lock here. A Redis lock can't guarantee exclusion if the holder stalls past its lease — a GC pause or network partition means two processes can both believe they hold it. For correctness-critical state, the state record itself should be the concurrency control.

I'd also add a short expiry on the assignment claim, so if the assigning process crashes between claiming the driver and creating the trip, the driver auto-releases instead of being stuck offline.
</details>

<details>
<summary><b>"250K location writes/sec — how?"</b></summary>

The key realization is that location data is ephemeral and loss-tolerant. Missing one update out of a hundred changes nothing for the user. That frees the hot path from durability requirements entirely.

So I'd split it into two paths. The hot path writes into an in-memory H3-indexed structure — a million drivers at ~100 bytes each is only 100 MB, which fits comfortably in RAM. That's what matching queries read, and it needs no persistence because it's rebuilt from the stream within seconds.

The cold path publishes to Kafka asynchronously for analytics, trip replay, fraud detection, and surge input. That path optimizes for throughput and durability, and it's never in a user request path.

On the client side I'd also reduce the volume at source: batch updates, and adapt the frequency — a stationary driver doesn't need 4-second updates, and a driver on a highway has highly predictable position.
</details>

<details>
<summary><b>"Why hexagons instead of a square grid?"</b></summary>

Two reasons. In a square grid, the four diagonal neighbours are 41% farther away than the four edge neighbours, so "the ring of cells around me" isn't a consistent distance. With hexagons all six neighbours are equidistant, which makes radius expansion during matching uniform and makes distance approximations more accurate.

Second, hexagons tile without the ambiguity squares have at corners, which matters when modeling flow — supply and demand moving across a city. Uber built H3 specifically for this and open-sourced it.

Geohash is simpler and works fine for basic proximity, but it has the boundary problem where two points ten metres apart can have completely different prefixes if they straddle a cell edge, so you always have to query the eight neighbours too.
</details>

<details>
<summary><b>"How would you scale this globally?"</b></summary>

The critical insight is that every trip is local — a rider in Delhi never needs data about drivers in São Paulo. So I'd shard everything geographically by city or region.

That turns one global-scale problem into roughly a thousand independent city-scale problems. Each region is a self-contained deployment: its own matching service, its own location index, its own trip database shard. Cross-region queries essentially never happen.

The benefits compound. A region failure is contained. You can deploy region by region as a natural canary. Data residency requirements are satisfied structurally rather than through policy. And each region's infrastructure is sized to actual local demand rather than global peak.

The parts that stay global are user accounts, payment methods, and analytics — all low-volume and tolerant of higher latency.
</details>

<details>
<summary><b>"What if the matching service crashes mid-assignment?"</b></summary>

Several layers. The assignment claim has a short TTL, so a crashed process releases the driver automatically rather than leaving them stuck offline.

The trip record is written with an explicit state, so recovery reads the state and knows exactly where it stopped. Because transitions are validated against an explicit state machine, replaying or resuming can't produce an invalid state.

The rider's request is a durable record, not just an in-flight computation — so another matching worker can pick it up. And the rider app is polling or on a WebSocket, so if matching takes longer than expected they see "still finding a driver" rather than an error.

Finally, I'd add a timeout on the whole request: if matching hasn't succeeded within a bounded window, fail explicitly with a clear message and no charge, rather than leaving the rider waiting indefinitely.
</details>

---

# 🎬 Netflix

> **Teaches:** CDN economics, adaptive streaming, chaos engineering, and the "precompute everything" philosophy.

## 1. Requirements

### Functional
```
   • Browse and search a catalogue
   • Personalized homepage rows
   • Stream video with adaptive quality
   • Resume across devices
   • Downloads for offline viewing
   • Multiple profiles per account
```

### Non-functional
```
   SCALE          ~270M subscribers, ~15% of global internet
                  downstream traffic at peak
   AVAILABILITY   99.99% for playback — ⭐ playback availability
                  matters far more than browse
   LATENCY        Start playback < 2 seconds
                  ⭐ Rebuffer ratio is the metric that matters most
   QUALITY        Adapt to bandwidth from 500 kbps to 25 Mbps
   GEO            190+ countries
```

## 2. Estimation

```
   CONCURRENT STREAMS (peak)    ~20M
   AVERAGE BITRATE              ~5 Mbps
   PEAK BANDWIDTH               20M × 5 Mbps = 100 Tbps  ⭐

   ⭐ THIS NUMBER DRIVES EVERYTHING.

   100 Tbps is more than most countries' total internet capacity.
   You cannot serve this from datacenters. Physics and economics
   both forbid it.

   → The ENTIRE architecture is a consequence of this one number.
   → Netflix must push content to the EDGE, inside ISP networks.

   STORAGE
   Catalogue ~20,000 titles
   Each encoded into ~1,200 files
     (multiple codecs × resolutions × bitrates × audio × subtitles)
   → several petabytes of encoded content
```

## 3. Architecture — The Three Parts

```
   ┌──────────────────────────────────────────────────────────────┐
   │ ① CONTROL PLANE (AWS)                                        │
   │    Everything EXCEPT the video bytes:                        │
   │    signup, auth, browse, search, recommendations, billing,   │
   │    metadata, A/B testing, playback authorization             │
   │    → hundreds of microservices                               │
   ├──────────────────────────────────────────────────────────────┤
   │ ② DATA PLANE — OPEN CONNECT (Netflix's own CDN)              │
   │    ⭐ Purpose-built appliances placed INSIDE ISP datacenters  │
   │    and at internet exchanges. Serves ~100% of video bytes.   │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ ENCODING PIPELINE (AWS, offline)                           │
   │    Ingest a master → transcode into ~1,200 variants →        │
   │    validate → distribute to Open Connect appliances          │
   └──────────────────────────────────────────────────────────────┘
```

### ⭐ Why Netflix built its own CDN

```
   Commercial CDNs charge per GB. At Netflix's volume that's
   billions of dollars per year — and no CDN had the capacity.

   OPEN CONNECT'S DEAL WITH ISPs:

   Netflix gives the ISP a free server appliance.
   The ISP racks it and provides power + a network port.

   ✅ ISP wins: Netflix traffic no longer crosses their expensive
     transit links. It's served locally. Massive cost saving.
   ✅ Netflix wins: near-zero marginal bandwidth cost, and the
     content sits one hop from the viewer.
   ✅ User wins: lower latency, fewer rebuffers, higher quality.

   ⭐ THE PREDICTIVE FILL — the clever part

   Netflix knows what people will watch tomorrow, per region,
   with high accuracy. So during off-peak hours (usually the
   early morning), it PUSHES content to each appliance.

   → At peak, essentially every request is a local cache HIT.
   → No origin fetches during peak. No cache misses.
   → This is the opposite of a normal CDN, which fills reactively
     on a miss.
```

## 4. Deep Dive — Adaptive Bitrate Streaming

#### 💬 The problem
Network bandwidth fluctuates constantly. A fixed bitrate either buffers on a bad connection or wastes quality on a good one.

```
   ENCODING — every title becomes a LADDER of variants

   ┌────────────┬──────────┬─────────────────────────────────┐
   │ Resolution │ Bitrate  │ Use                             │
   ├────────────┼──────────┼─────────────────────────────────┤
   │ 4K HDR     │ 15 Mbps  │ Fast fixed line, big screen     │
   │ 1080p      │ 5 Mbps   │ Typical broadband               │
   │ 720p       │ 3 Mbps   │ Good mobile / weak broadband    │
   │ 480p       │ 1.5 Mbps │ Congested mobile                │
   │ 360p       │ 750 kbps │ Poor connection                 │
   │ 240p       │ 300 kbps │ Emergency — audio must survive  │
   └────────────┴──────────┴─────────────────────────────────┘

   Each variant is cut into 2-10 second SEGMENTS.
```

```
   PLAYBACK — the client decides, segment by segment

   Time →
   ├──seg1──┤├──seg2──┤├──seg3──┤├──seg4──┤├──seg5──┤
     1080p     1080p      480p      720p     1080p
                          ▲
                    bandwidth dropped → client switched DOWN
                    mid-stream, seamlessly

   THE CLIENT'S ALGORITHM
   1. Measure throughput of recent segment downloads
   2. Check buffer level (how many seconds are ready to play?)
   3. Pick the highest bitrate that:
        • recent throughput can sustain, AND
        • keeps the buffer above a safety threshold
   4. ⭐ Be conservative going UP, aggressive going DOWN
      — a rebuffer is far worse for the user than lower quality

   ⭐ WHY THE CLIENT DECIDES, NOT THE SERVER
   Only the client knows its true conditions — CPU load, screen
   size, actual measured throughput, battery state. A server
   can only guess. This also makes the server stateless: it just
   serves segment files.
```

### Per-title encoding — the optimization that saved petabytes

```
   ❌ OLD: one fixed bitrate ladder for every title

   A talking-heads drama and a fast-action superhero film got
   the SAME 5 Mbps 1080p encode. But the drama looks perfect
   at 2 Mbps, and the action film needs 8 Mbps.

   ✅ NEW: analyze each title's complexity, generate a CUSTOM ladder

   → simple content: ~50% bandwidth reduction at equal quality
   → complex content: higher bitrate where it's actually needed
   → aggregate: massive bandwidth and storage savings

   ⭐ EVEN FURTHER: per-SHOT encoding. Complexity varies within
     a title too — a static dialogue scene and an explosion have
     wildly different needs. Netflix encodes shot by shot.
```

## 5. Deep Dive — Personalization

```
   ⭐ EVERY PIXEL OF THE HOMEPAGE IS PERSONALIZED

   • WHICH rows appear, and in what order
   • WHICH titles are in each row, and in what order
   • WHICH artwork is shown for each title  ← often overlooked
   • The row TITLES themselves

   ARTWORK PERSONALIZATION IS REAL:
   The same film might show a romantic still to one user and an
   action still to another, based on their viewing history.
   This measurably increases engagement.
```

```
   ARCHITECTURE — precompute, don't compute on request

   ┌────────────────────────────────────────────────────────┐
   │ OFFLINE (batch, hourly/daily)                          │
   │   • Train ranking models on viewing history            │
   │   • Precompute candidate sets per user cohort          │
   │   • Write results to a fast key-value store            │
   ├────────────────────────────────────────────────────────┤
   │ NEAR-REAL-TIME                                         │
   │   • Adjust for very recent activity                    │
   │     ("you just finished season 1 → show season 2")     │
   ├────────────────────────────────────────────────────────┤
   │ REQUEST TIME (must be <100ms)                          │
   │   • Fetch the precomputed rows                         │
   │   • Apply availability filters (region/licensing)      │
   │   • Apply "continue watching" and freshness rules      │
   │   • Render                                             │
   └────────────────────────────────────────────────────────┘

   ⭐ The request path does almost NO computation. This is the
     recurring pattern in read-heavy systems: move work from
     read time to write/batch time.
```

## 6. Deep Dive — Resilience

Netflix essentially invented modern chaos engineering because their scale made failure constant.

```
   ┌──────────────────────────────────────────────────────────────┐
   │ CHAOS MONKEY — randomly kills instances IN PRODUCTION        │
   │   Forces every service to survive instance death, always.    │
   │   You can't have an untested failover if it fires weekly.    │
   ├──────────────────────────────────────────────────────────────┤
   │ CHAOS KONG — takes out an entire AWS REGION                  │
   │   Validates that traffic can be evacuated to other regions.  │
   ├──────────────────────────────────────────────────────────────┤
   │ HYSTRIX (now resilience4j) — circuit breakers everywhere     │
   │   Every cross-service call is wrapped, with a defined        │
   │   FALLBACK. No call can hang indefinitely.                   │
   ├──────────────────────────────────────────────────────────────┤
   │ GRACEFUL DEGRADATION — the design philosophy ⭐               │
   │                                                              │
   │   Personalization down → show a generic popular list         │
   │   Search down         → show browse categories               │
   │   Artwork service down→ show default artwork                 │
   │   Ratings down        → hide ratings, keep playing           │
   │                                                              │
   │   ⭐ PLAYBACK NEVER STOPS. Everything else is optional.       │
   │     The product hierarchy is encoded into the architecture.  │
   └──────────────────────────────────────────────────────────────┘
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"Why did Netflix build its own CDN?"</b></summary>

Economics and capacity. At peak Netflix pushes on the order of 100 terabits per second — more than most countries' total internet capacity. No commercial CDN had that capacity, and at per-GB pricing the bill would have been billions annually.

Open Connect inverts the model. Netflix gives ISPs a free appliance; the ISP provides rack space, power, and a port. The ISP saves enormously on transit costs because Netflix traffic no longer crosses their expensive upstream links, and Netflix gets near-zero marginal bandwidth cost with content sitting one hop from the viewer.

The genuinely clever part is predictive fill. Netflix can predict regional viewing demand accurately, so it pushes content to appliances during off-peak hours. At peak, essentially every request is a local cache hit — no origin fetches at all. That's the opposite of a normal CDN, which fills reactively on cache misses.
</details>

<details>
<summary><b>"How does adaptive bitrate work, and why does the client decide?"</b></summary>

Each title is encoded into a ladder of quality variants, and each variant is cut into short segments — typically 2 to 10 seconds. The client requests segments one at a time and picks the quality level for each.

The client's algorithm combines measured throughput from recent downloads with the current buffer level, then selects the highest bitrate that recent throughput can sustain while keeping the buffer above a safety threshold. It's deliberately conservative about stepping up and aggressive about stepping down, because a rebuffer hurts the user far more than a temporary quality drop.

The client decides rather than the server because only the client knows its actual conditions — real measured throughput, CPU load, screen size, battery state. A server can only guess. It also keeps the server completely stateless: it just serves segment files, which is exactly what a CDN is good at.
</details>

<details>
<summary><b>"What is per-title encoding and why does it matter?"</b></summary>

Originally Netflix used one fixed bitrate ladder for every title. That's wasteful in both directions — a static talking-heads drama looks perfect at 2 Mbps, while a fast-action film needs 8 Mbps to avoid artifacts. Using 5 Mbps for both wastes bandwidth on one and degrades the other.

Per-title encoding analyzes each title's visual complexity and generates a custom ladder. For simple content that's roughly a 50% bandwidth reduction at equivalent perceived quality; for complex content it allocates more where it's actually needed.

Netflix pushed this further to per-shot encoding, since complexity varies within a title too — a dialogue scene and an explosion have completely different requirements. At Netflix's scale, a few percent of bandwidth is worth enormous money, so this optimization pays for very substantial encoding compute.
</details>

<details>
<summary><b>"How do you keep playback available when services fail?"</b></summary>

By encoding the product hierarchy into the architecture. Playback is sacred; everything else is optional and degrades independently.

Concretely, every cross-service call is wrapped in a circuit breaker with a defined fallback. If personalization fails, show a generic popular list. If search fails, show browse categories. If the artwork service fails, use default images. If ratings fail, hide ratings. In every case the user can still find something and press play.

This is validated continuously rather than assumed. Chaos Monkey randomly terminates production instances, so no service can have an untested failover path. Chaos Kong evacuates entire AWS regions to prove cross-region failover actually works. The philosophy is that if you don't test failure constantly, you're just hoping — and hope fails at scale.
</details>

---

# 🐦 Twitter

> **Teaches:** fan-out strategies, the celebrity problem, and read-heavy system design.

## 1. Requirements

```
   FUNCTIONAL
   • Post a tweet (280 chars, optional media)
   • Follow / unfollow users
   • Home timeline (tweets from followees, reverse chronological)
   • User timeline (one user's tweets)
   • Like, retweet, reply

   OUT OF SCOPE: DMs, search, trending, ads, spaces, lists

   NON-FUNCTIONAL
   SCALE          ~250M DAU, ~500M tweets/day
   ⭐ READ:WRITE   ~1000:1 — this single number drives everything
   LATENCY        Timeline load < 200ms p99
   AVAILABILITY   99.99%
   CONSISTENCY    Eventual is fine — a tweet appearing 2 seconds
                  late is acceptable. This is a huge freedom.
```

## 2. Estimation

```
   WRITES   500M tweets/day / 86,400 ≈ 6,000 tweets/sec
                                     ≈ 18,000/sec peak

   READS    250M users × ~50 timeline loads/day
            = 12.5B / 86,400 ≈ 145,000 QPS
                             ≈ 500,000 QPS peak

   ⭐ RATIO ≈ 1000:1 READ HEAVY
     → precompute on WRITE, never compute on READ
     → cache aggressively
     → accept write amplification to buy read speed

   STORAGE  500M × 300 bytes = 150 GB/day text
            + media (via object storage + CDN)
            ≈ 55 TB/year text, × 3 replication ≈ 165 TB/year

   TIMELINE CACHE
   250M users × 800 tweet IDs × 8 bytes ≈ 1.6 TB
   → but only ~10% of users are active daily
   → ~160 GB of hot timeline data. Very feasible in Redis.
```

## 3. The Core Problem — Timeline Generation

#### 💬 Why this is the whole interview

A home timeline is *"all tweets from everyone I follow, merged, newest first."* Computing that on demand means querying N followees and merging — at 500K QPS with an average of 200 followees, that's 100 million queries per second. Impossible.

```
   ─── OPTION A: FAN-OUT ON READ (pull) ─────────────────────

   Store each tweet once. On timeline request, query all followees
   and merge.

   ┌──────┐                      ┌──────────┐
   │Tweet │───▶ tweets table ───▶│ On read: │
   │ post │      (one write)     │ query N  │───▶ merge ───▶ timeline
   └──────┘                      │ followees│
                                 └──────────┘

   ✅ Write is O(1) — cheap and instant
   ✅ No storage duplication
   ✅ No wasted work for users who never log in
   ❌ Read is O(N followees) with a merge — far too slow
   ❌ Impossible to cache effectively (every user's timeline differs)

   ─── OPTION B: FAN-OUT ON WRITE (push) ────────────────────

   When a user tweets, push the tweet ID into every follower's
   precomputed timeline list.

   ┌──────┐     ┌──────────────┐     ┌─────────────────────┐
   │Tweet │────▶│ Fan-out      │────▶│ follower1: [ids...] │
   │ post │     │ worker       │────▶│ follower2: [ids...] │
   └──────┘     │ (async)      │────▶│ follower3: [ids...] │
                └──────────────┘     └─────────────────────┘
                                          (Redis lists)

   ✅ Read is O(1) — just fetch a list. Blazing fast.
   ✅ Trivially cacheable
   ❌ Write amplification: 1 tweet → N writes
   ❌ ⚠️ A user with 100M followers = 100M writes for ONE tweet
   ❌ Wasted work for inactive followers

   ─── ⭐ OPTION C: HYBRID — what actually works ─────────────
```

```
   ┌──────────────────────────────────────────────────────────────┐
   │                    THE HYBRID DESIGN                         │
   │                                                              │
   │  ON WRITE:                                                   │
   │    if follower_count < THRESHOLD (~10,000):                  │
   │        → FAN OUT: push the tweet ID to each follower's       │
   │          timeline list (async, via a queue)                  │
   │    else (celebrity):                                         │
   │        → DO NOT fan out. Store once. Full stop.              │
   │                                                              │
   │  ON READ:                                                    │
   │    1. Fetch the precomputed timeline list (fast)             │
   │    2. Fetch recent tweets from the FEW celebrities this      │
   │       user follows (small N, heavily cached — every one      │
   │       of a celebrity's followers reads the same data)        │
   │    3. Merge the two by timestamp, return                     │
   └──────────────────────────────────────────────────────────────┘

   ✅ Write amplification is BOUNDED (celebrities excluded)
   ✅ Read is still fast: one list fetch + a small merge
   ✅ Celebrity tweets are cached once and read by millions —
      the most cache-efficient possible arrangement
   ✅ Matches the actual power-law distribution of follower counts
```

### ⭐ Refinements that show depth

```
   1. FAN OUT ONLY TO ACTIVE USERS
      Only push to users who logged in within ~30 days.
      Inactive users get their timeline built lazily on return.
      → cuts fan-out volume dramatically (most accounts are dormant)

   2. CAP THE TIMELINE LENGTH
      Store ~800 tweet IDs per user. Nobody scrolls further.
      Deeper pagination falls back to the pull path.
      → bounds memory: 800 × 8 bytes = 6.4 KB per user

   3. FAN-OUT IS ASYNCHRONOUS
      POST /tweets writes the tweet and enqueues a fan-out job,
      then returns immediately. A user with 50,000 followers
      doesn't wait for 50,000 writes.

   4. STORE IDs, NOT TWEET CONTENT
      Timeline lists hold tweet IDs. Content is fetched from a
      separate cache by ID.
      → one copy of each tweet's content, not N copies
      → edits and deletes update one place

   5. THE THRESHOLD IS TUNABLE AND SHOULD BE MEASURED
      10,000 is illustrative. The right number balances fan-out
      cost against read-merge cost, and depends on your infra.
```

## 4. Architecture

```
   ┌────────┐
   │ Client │
   └───┬────┘
       ▼
   ┌─────────────────────────────────────────────────────────┐
   │                    API GATEWAY                          │
   └───┬─────────────────────────────────┬───────────────────┘
       ▼                                 ▼
   ┌─────────────┐                 ┌─────────────┐
   │   WRITE     │                 │    READ     │
   │   PATH      │                 │    PATH     │
   └──────┬──────┘                 └──────┬──────┘
          │                               │
          ▼                               ▼
   ┌─────────────┐                 ┌─────────────────┐
   │Tweet Service│                 │Timeline Service │
   └──────┬──────┘                 └────────┬────────┘
          │                                 │
          ├──▶ Tweets DB (sharded by tweet_id)
          │                                 │
          └──▶ Kafka ──▶ Fan-out workers    │
                              │             │
                              ▼             ▼
                    ┌──────────────────────────────┐
                    │  REDIS TIMELINE CLUSTER      │
                    │  user_id → [tweet_id, ...]   │
                    │  (capped at ~800)            │
                    └──────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────────────────┐
                    │  TWEET CONTENT CACHE         │
                    │  tweet_id → {text, author…}  │
                    └──────────────────────────────┘

   Social graph (follows) → its own service + store,
   heavily cached, since fan-out reads it constantly
```

## 5. Deep Dive — The Social Graph

```
   THE TWO QUERIES, AND THEIR VERY DIFFERENT PROFILES

   "Who does user X follow?"      → bounded (thousands at most)
                                    read on every timeline build

   "Who follows user X?"     ⚠️   → UNBOUNDED (up to 100M+)
                                    read on every fan-out

   STORAGE
   follows (follower_id, followee_id, created_at)
     index (follower_id)   → "who do I follow"
     index (followee_id)   → "who follows me"

   ⭐ Shard by follower_id so "who do I follow" is a single-shard
     query — that's the one on the hot read path.
     "Who follows me" becomes scatter-gather, but it's only used
     by asynchronous fan-out where latency doesn't matter.
```

## 6. Deep Dive — The Celebrity Problem

#### 💬 Beyond fan-out

The celebrity problem shows up in several places, not just fan-out:

```
   ⚠️ HOT KEY ON READ
      A celebrity's tweet is fetched by millions simultaneously.
      One cache key, one Redis node → that node melts.

      FIX: replicate hot keys across multiple cache nodes and
      read from a random replica. Or add a small local (L1) cache
      in each app server — with a short TTL, since the same content
      is being requested by everyone anyway.

   ⚠️ ENGAGEMENT COUNTER CONTENTION
      Millions of likes on one tweet → contention on one row.

      FIX: sharded counters. Split the counter across N keys,
      increment a random one, sum on read.
      counter:tweet123:0 ... counter:tweet123:15
      → 16× the write throughput, and reads sum 16 small values.

   ⚠️ THE THUNDERING HERD ON POST
      A celebrity tweets; millions refresh at once.

      FIX: the content is identical for everyone, so it caches
      perfectly. Ensure the cache is warmed on write rather than
      populated by the first million readers.
```

## 7. Failure Modes

```
   ⚠️ FAN-OUT QUEUE BACKS UP
      → timelines go stale (tweets appear late)
      → NOT an outage — degraded freshness
      → monitor consumer lag as TIME-TO-DRAIN, not message count
      → autoscale workers on that metric

   ⚠️ REDIS TIMELINE CLUSTER DOWN
      → fall back to compute-on-read at degraded latency
      → ⭐ this fallback path must EXIST and be tested, or
        a cache outage becomes a total outage

   ⚠️ DELETES AND EDITS
      A tweet is already fanned out to a million timelines.
      → do NOT try to remove it from every list (too expensive)
      → keep a tombstone set; FILTER at read time
      → the ID stays in the list but resolves to nothing
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"Walk me through fan-out on write vs read."</b></summary>

Fan-out on read stores each tweet once and builds the timeline at request time by querying every followee and merging. Writes are cheap, but reads are O(number of followees) with a merge — at Twitter's read volume that's on the order of a hundred million queries per second. Not viable.

Fan-out on write pushes the tweet ID into every follower's precomputed list at post time. Reads become a single list fetch, which is exactly what you want at a thousand-to-one read ratio. The cost is write amplification — and for an account with a hundred million followers, one tweet becomes a hundred million writes.

So the real answer is hybrid. Fan out for normal accounts, don't fan out for accounts above a follower threshold. On read, merge the precomputed list with recent tweets from the handful of celebrities the user follows. Celebrity tweets cache beautifully because millions of people read the identical data.

Then the refinements matter: only fan out to recently-active users, cap the stored timeline at a few hundred IDs, do the fan-out asynchronously via a queue so posting returns immediately, and store IDs rather than content so there's one copy of each tweet.
</details>

<details>
<summary><b>"How do you handle a tweet being deleted after fan-out?"</b></summary>

You don't remove it from a million timeline lists — that's as expensive as the original fan-out and it would happen on a synchronous user action.

Instead, maintain a tombstone set of deleted tweet IDs. The timeline read path already fetches tweet content by ID from a separate cache, so a deleted tweet simply resolves to nothing and gets filtered out. The stale ID sits harmlessly in the list until it ages out of the capped window.

The same mechanism handles blocks and mutes — those are also read-time filters rather than write-time list surgery. That's the general principle: for anything that's cheap to check at read time and expensive to apply at write time, filter on read.
</details>

<details>
<summary><b>"How would you add search?"</b></summary>

Search is a separate, derived system — never the source of truth. Tweets flow from the write path into Kafka, and a consumer indexes them into Elasticsearch or a similar inverted index.

That separation matters for three reasons. Indexing is eventually consistent and shouldn't slow down the write path. Search has completely different scaling characteristics from timeline serving. And if the search cluster is lost, you rebuild it from the tweet store rather than losing data.

For real-time search — finding a tweet seconds after it's posted — you'd want a hot recent index separate from the historical one, since the recency requirements and the volume are very different. Ranking blends text relevance with engagement signals and recency, which is why searching Twitter doesn't return purely chronological results.
</details>

<details>
<summary><b>"What's the hot key problem here and how do you fix it?"</b></summary>

When a celebrity tweets, millions of people request the identical cache key simultaneously. That's a single Redis key on a single node, and that node saturates while the rest of the cluster is idle.

Three mitigations. Replicate hot keys across multiple cache nodes with a suffix, and have clients read from a random replica — spreading one logical key across N physical nodes. Add a small in-process L1 cache in each app server with a very short TTL; since everyone wants the same data, even a one-second TTL absorbs enormous load. And warm the cache proactively on write rather than letting the first million readers race to populate it.

The related contention problem is engagement counters. Millions of likes on one tweet contend on a single row or key. The fix is sharded counters — split it across sixteen keys, increment a random one, and sum on read. That's sixteen times the write throughput at the cost of a slightly more expensive read.
</details>

---

# 💬 WhatsApp

> **Teaches:** persistent connections, message delivery guarantees, end-to-end encryption, and doing more with far fewer servers.

## 1. Requirements

```
   FUNCTIONAL
   • 1:1 and group messaging
   • Delivery receipts (sent ✓, delivered ✓✓, read ✓✓ blue)
   • Online/last-seen presence
   • Media sharing
   • End-to-end encryption
   • Multi-device

   NON-FUNCTIONAL
   SCALE          ~2B users, ~100B messages/day
   LATENCY        < 100ms delivery when both parties online
   RELIABILITY    ⭐ Messages must NEVER be lost
   ORDERING       Messages within a conversation must arrive in order
   E2E ENCRYPTION ⭐ The server must not be able to read messages —
                  this constrains the entire architecture
```

## 2. Estimation

```
   MESSAGES       100B/day / 86,400 ≈ 1.2M messages/sec
                                    ≈ 3M/sec peak

   CONCURRENT CONNECTIONS   ~500M+ simultaneously online

   ⭐ THE DEFINING CONSTRAINT: connections, not throughput.

   500M persistent TCP connections. At 1M connections per server
   (achievable with tuned Erlang/BEAM), that's 500 servers just
   for connection handling.

   WhatsApp famously ran ~450M users with ~50 engineers, on a few
   hundred servers, using Erlang — which is built for exactly this.

   STORAGE
   ⭐ Messages are NOT stored long-term on the server.
     Delivered = deleted. This is a deliberate design choice that
     makes the storage problem almost disappear.
     Only undelivered messages are queued.
```

## 3. Architecture

```
   ┌──────────┐                                    ┌──────────┐
   │ Client A │                                    │ Client B │
   └────┬─────┘                                    └────▲─────┘
        │ persistent connection                          │
        │ (WebSocket / custom XMPP-derived protocol)      │
        ▼                                                │
   ┌─────────────────────────────────────────────────────┴───────┐
   │              CONNECTION SERVERS (Erlang/BEAM)               │
   │   • ~1M concurrent connections per server                   │
   │   • Maintains: user_id → which server holds their connection│
   └───┬─────────────────────────────────────────────────────────┘
       ▼
   ┌───────────────────────┐        ┌──────────────────────────┐
   │  SESSION REGISTRY     │        │   MESSAGE QUEUE          │
   │  user → server        │        │   (per recipient, for    │
   │  (Redis / distributed)│        │    OFFLINE users only)   │
   └───────────────────────┘        └──────────────────────────┘
                                              │
                                              ▼
                                    ┌──────────────────────────┐
                                    │  PUSH NOTIFICATION       │
                                    │  (APNs / FCM)            │
                                    │  wakes the app, which    │
                                    │  then reconnects         │
                                    └──────────────────────────┘

   MEDIA: uploaded separately to blob storage; the MESSAGE
          contains only a URL + decryption key
```

## 4. Deep Dive — Message Delivery

```
   THE FLOW

   ① A sends to server over the persistent connection
   ② Server assigns a message ID + server timestamp
   ③ Server ACKs to A immediately        → single grey ✓ (SENT)
   ④ Server looks up B in the session registry
        ├─ B ONLINE  → push over B's connection
        └─ B OFFLINE → enqueue + send a push notification
   ⑤ B's device ACKs receipt              → double grey ✓✓ (DELIVERED)
   ⑥ B opens the chat, sends a read receipt → blue ✓✓ (READ)
   ⑦ ⭐ Server DELETES the message once delivered
```

```
   ⭐ THE THREE-TICK SYSTEM IS THE DELIVERY GUARANTEE MADE VISIBLE

   ✓      server received it       (durability boundary crossed)
   ✓✓     device received it       (end-to-end delivery confirmed)
   ✓✓ blue user read it            (application-level acknowledgment)

   Each tick is a distinct acknowledgment hop. This is unusually
   good UX for a distributed systems problem — the user can see
   exactly where their message is.
```

### ⭐ Ordering and idempotency

```
   PROBLEM: mobile networks retry. Duplicates are guaranteed.

   SOLUTION
   • The CLIENT generates a message ID (UUID) before sending
   • The server deduplicates on that ID
   • Retries are therefore idempotent and safe

   ORDERING
   • Per-conversation sequence numbers
   • The receiving client buffers out-of-order messages briefly
     and reorders before display
   • ⭐ Ordering is only guaranteed WITHIN a conversation, which
     is the only place users actually perceive it
```

## 5. Deep Dive — End-to-End Encryption

#### 💬 Why E2E changes the architecture

If the server cannot read messages, then the server cannot: search them, generate previews, run spam classifiers on content, or do server-side backup. Every feature must be reconsidered. This constraint is architectural, not a feature toggle.

```
   THE SIGNAL PROTOCOL — the essential mechanism

   ① KEY EXCHANGE (X3DH — Extended Triple Diffie-Hellman)
      Each user publishes to the server:
        • identity key      (long-term)
        • signed prekey     (medium-term, rotated)
        • one-time prekeys  (batch, consumed on use)

      To start a conversation, A fetches B's key bundle and
      derives a shared secret — WITHOUT B being online.
      ⭐ The server stores public keys only. It never sees a
        private key and cannot derive the shared secret.

   ② DOUBLE RATCHET
      Every message advances a key chain, so each message uses
      a DIFFERENT key.

      ┌──────────────────────────────────────────────────────┐
      │ FORWARD SECRECY                                      │
      │   Compromising today's key does NOT decrypt           │
      │   yesterday's messages.                               │
      ├──────────────────────────────────────────────────────┤
      │ POST-COMPROMISE SECURITY (break-in recovery)          │
      │   A new DH exchange periodically injects fresh        │
      │   entropy, so an attacker who stole a key LOSES       │
      │   access after the next ratchet.                      │
      └──────────────────────────────────────────────────────┘

   ③ GROUP MESSAGING — sender keys
      Naive: encrypt separately for each of N members → O(N) work
      per message, terrible for large groups.

      Sender keys: each member distributes a symmetric sender key
      once (pairwise-encrypted). Subsequent messages are encrypted
      once with that key and fanned out.
      → O(1) encryption per message, O(N) only on membership change
```

```
   ⚠️ WHAT E2E COSTS YOU
   • No server-side search (search must be local, on-device)
   • No server-side spam/abuse detection on content
     (must rely on metadata and behavioural signals)
   • Multi-device is genuinely hard — each device needs its own
     key material and session state
   • Backup requires a separate user-held key, or it isn't E2E
```

## 6. Deep Dive — Connection Management

```
   ⭐ WHY ERLANG/BEAM WAS THE RIGHT CHOICE

   • Lightweight processes (~2 KB each, not OS threads)
     → millions of concurrent connections per machine
   • Preemptive scheduling — one slow process can't block others
   • Supervision trees — crashed processes restart automatically
     without taking down neighbours
   • Hot code reloading — deploy without dropping connections ⭐
   • Built for exactly this: telecom switches with the same shape
     of problem (millions of long-lived connections, soft real-time)
```

```
   KEEPING CONNECTIONS ALIVE ON MOBILE

   ⚠️ Mobile networks aggressively kill idle TCP connections and
     NAT mappings expire (often at 5-30 minutes, and it varies
     wildly by carrier).

   • Heartbeats tuned per network type — too frequent drains
     battery, too rare drops the connection
   • Exponential backoff with jitter on reconnect
     ⭐ Without jitter, a server restart causes 1M clients to
       reconnect simultaneously and knock it over again
   • Fall back to push notifications when the connection can't
     be maintained (backgrounded app, doze mode)
   • Resume sessions rather than full re-handshake where possible
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"How do you handle 500M concurrent connections?"</b></summary>

Connections, not throughput, are the defining constraint. With around a million concurrent connections per tuned server, that's a few hundred connection servers — which is why WhatsApp famously ran at enormous scale with a very small fleet and team.

The technology choice matters here. Erlang's BEAM gives you lightweight processes at around 2 KB each rather than OS threads at megabytes, preemptive scheduling so one slow connection can't block others, supervision trees for automatic recovery, and hot code reloading so you can deploy without dropping connections. It was built for telecom switches, which is structurally the same problem.

The other half is a session registry mapping each user to whichever server holds their connection, so a message for user B can be routed to the right box. That registry is the coordination point and needs to be fast and highly available.

On the client side, mobile networks kill idle connections aggressively and NAT mappings expire unpredictably, so you need heartbeats tuned per network type, and reconnection with exponential backoff plus jitter — without jitter, a server restart triggers a million simultaneous reconnects that knock it over again.
</details>

<details>
<summary><b>"How do you guarantee messages are never lost?"</b></summary>

Acknowledgments at every hop, plus client-generated IDs for idempotency.

The client generates a UUID before sending, so retries are naturally deduplicated by the server. The server persists the message and acknowledges — that's the first tick, and it marks the durability boundary. If the recipient is online the message is pushed and their device acknowledges — second tick. If they're offline it's queued and a push notification wakes the app.

The message is only deleted once delivery is confirmed. So a failure at any point results in a retry rather than a loss, and the visible tick state tells the user exactly how far it got.

The interesting design consequence is that because delivered messages are deleted server-side, WhatsApp's storage problem largely disappears — they only store what hasn't been delivered yet. That's a deliberate choice that trades server-side message history for enormous operational simplicity.
</details>

<details>
<summary><b>"Explain end-to-end encryption and what it costs you."</b></summary>

The Signal Protocol has two parts. X3DH handles key exchange: each user publishes an identity key, a signed prekey, and a batch of one-time prekeys to the server. A sender can fetch that bundle and derive a shared secret without the recipient being online. The server only ever holds public keys.

The Double Ratchet then advances a key chain with every message, so each message uses a different key. That gives forward secrecy — compromising today's key doesn't decrypt yesterday's messages — and post-compromise security, where periodic fresh Diffie-Hellman exchanges mean an attacker who steals a key loses access after the next ratchet.

For groups, encrypting separately for each member is O(N) per message. Sender keys fix that: each member distributes a symmetric sender key once, then messages are encrypted once and fanned out, so it's O(1) per message and O(N) only on membership changes.

The costs are real and architectural. No server-side search, so search must be on-device. No content-based spam detection, so abuse handling relies on metadata and behaviour. Multi-device is genuinely hard because each device needs its own key material. And backup isn't end-to-end unless the user holds a separate key.
</details>

---

# 📸 Instagram

> **Teaches:** media pipelines, feed ranking, and the read-heavy pattern applied to rich content.

## 1. Requirements

```
   FUNCTIONAL
   • Upload photos/videos with captions
   • Follow users
   • Feed (ranked, not purely chronological)
   • Stories (24-hour expiry)
   • Like, comment
   • Explore / discovery

   NON-FUNCTIONAL
   SCALE          ~2B MAU, ~100M photos/day
   ⭐ READ:WRITE   ~100:1
   LATENCY        Feed load < 200ms
   AVAILABILITY   99.9%
   CONSISTENCY    Eventual acceptable for feed; likes/counts
                  can lag noticeably without harm
```

## 2. Estimation

```
   UPLOADS     100M photos/day / 86,400 ≈ 1,200/sec
                                        ≈ 4,000/sec peak

   FEED READS  500M DAU × 20 loads = 10B/day ≈ 115,000 QPS
                                             ≈ 350,000 QPS peak

   STORAGE     Original photo ~3 MB
               + ~5 resized variants ≈ 2 MB
               ≈ 5 MB per upload

               100M × 5 MB = 500 TB/day  ⭐
               × 365 ≈ 180 PB/year

   ⭐ IMPLICATION: storage is the dominant cost, not compute.
     This drives: aggressive compression, tiered storage,
     and serving everything through a CDN so origin egress
     is near zero.

   BANDWIDTH   Feed images served from CDN.
               Origin only sees cache misses — a few percent.
```

## 3. Architecture

```
   ┌────────┐
   │ Client │
   └───┬────┘
       │ ⭐ direct upload via presigned URL — bytes never
       │    touch the application servers
       ▼
   ┌─────────────────┐         ┌──────────────────────────────┐
   │  OBJECT STORAGE │────────▶│  MEDIA PROCESSING PIPELINE   │
   │   (originals)   │  event  │  (async, via queue)          │
   └─────────────────┘         │  • resize into ~5 variants   │
                               │  • re-encode (WebP/AVIF)     │
                               │  • strip EXIF (⭐ privacy)    │
                               │  • generate blurhash         │
                               │  • ML: content moderation,   │
                               │    object/scene tagging      │
                               └───────────┬──────────────────┘
                                           ▼
                               ┌──────────────────────────────┐
                               │  PROCESSED MEDIA → CDN       │
                               └──────────────────────────────┘

   ┌─────────────────────────────────────────────────────────┐
   │  METADATA PATH                                          │
   │  API ──▶ Post Service ──▶ Postgres (sharded by user_id) │
   │                       └──▶ Kafka ──▶ Feed fan-out       │
   │                                  ──▶ Search indexing    │
   │                                  ──▶ Analytics          │
   └─────────────────────────────────────────────────────────┘
```

### ⭐ The upload flow — why presigned URLs matter

```
   ❌ NAIVE                        ✅ PRESIGNED URL
   Client → App server → S3        Client → App: "I want to upload"
                                   App → Client: signed URL
   App bandwidth: 2× the file      Client → S3: PUT (direct)
   App servers tied up for the     S3 → event → processing pipeline
     entire upload duration
   Timeouts on large videos        App bandwidth: ~0
   Can't scale uploads             App servers: free immediately
     independently                 Scales with S3, not with your fleet
```

## 4. Deep Dive — Feed Ranking

#### 💬 Why Instagram isn't chronological

A chronological feed shows you what's newest. A ranked feed shows you what you're most likely to engage with. The shift happened because as follow counts grew, chronological feeds buried the content people actually wanted.

```
   TWO-STAGE ARCHITECTURE — the standard pattern for all ranking

   ┌──────────────────────────────────────────────────────────────┐
   │ STAGE 1: CANDIDATE GENERATION  (thousands → hundreds)        │
   │   Cheap, high-recall retrieval from several sources:         │
   │     • recent posts from people you follow                    │
   │     • posts from accounts similar to ones you engage with    │
   │     • trending in your interest clusters                     │
   │   ⭐ Optimized for RECALL — don't miss anything good          │
   ├──────────────────────────────────────────────────────────────┤
   │ STAGE 2: RANKING  (hundreds → ~50 ordered)                   │
   │   Expensive ML model scoring each candidate on               │
   │   predicted engagement:                                      │
   │     P(like) · P(comment) · P(share) · P(dwell time) ·        │
   │     P(profile visit) · P(negative feedback — weighted DOWN)  │
   │   ⭐ Optimized for PRECISION                                  │
   ├──────────────────────────────────────────────────────────────┤
   │ STAGE 3: RE-RANKING / BUSINESS RULES                         │
   │     • diversity — don't show 5 posts from one account        │
   │     • ad insertion at set positions                          │
   │     • freshness boost                                        │
   │     • integrity filters (policy violations, borderline)      │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ SIGNALS THAT FEED THE RANKER

   ABOUT THE POST      recency · media type · engagement velocity
                       (likes per minute since posting — a strong
                       early-quality signal)

   ABOUT THE AUTHOR    your history with them · how often you view
                       their profile · whether you DM them
                       (⭐ closeness signals are extremely predictive)

   ABOUT YOU           session length · time of day · what you
                       engaged with recently · scroll velocity

   NEGATIVE            "not interested" · reports · fast scroll-past
                       (⭐ these are weighted heavily — avoiding bad
                       content matters as much as finding good)
```

### Why two stages instead of one

```
   Running the expensive model on every possible post is
   computationally impossible — millions of candidates × a deep
   model × 350K QPS.

   Stage 1 uses cheap heuristics and precomputed embeddings to
   cut millions down to hundreds. Stage 2 spends real compute only
   on that shortlist.

   ⭐ This funnel pattern — cheap recall then expensive precision —
     is universal in recommendation, search, and ads.
```

## 5. Deep Dive — Stories

#### 💬 Why stories are architecturally different

```
   ┌────────────────────┬────────────────────────────────────────┐
   │ POSTS              │ STORIES                                │
   ├────────────────────┼────────────────────────────────────────┤
   │ Permanent          │ ⭐ Expire in 24 hours                   │
   │ Ranked feed        │ Chronological within an author         │
   │ Public engagement  │ Private view list                      │
   │ Grows forever      │ ⭐ Bounded working set                  │
   └────────────────────┴────────────────────────────────────────┘

   THE 24-HOUR EXPIRY IS A GIFT ARCHITECTURALLY:

   • The active dataset is bounded — only ~1 day of stories exist
   • TTL-based storage means no deletion job needed
     (Redis TTL, or a Cassandra table with TTL set per row)
   • The "who has an unseen story" ring at the top of the app
     is a small, cacheable per-user computation
   • Storage cost is bounded rather than growing forever

   VIEW TRACKING
   Each story keeps a set of viewer IDs.
   ⭐ Contention risk: a celebrity story gets millions of views.
     → use a probabilistic structure (HyperLogLog) for the COUNT
       and a capped/sampled list for the actual viewer names,
       or shard the viewer set.
```

## 6. Deep Dive — Media Storage Tiering

```
   ⭐ ACCESS FOLLOWS A STEEP DECAY CURVE

   Day 0-1:    ~80% of all views of a photo
   Day 2-7:    ~15%
   Day 8-30:   ~4%
   After:      ~1%, and falling

   → TIER THE STORAGE ACCORDINGLY

   ┌──────────────────────────────────────────────────────────┐
   │ HOT      CDN edge + hot object storage    (0-7 days)     │
   │ WARM     standard object storage          (7-90 days)    │
   │ COLD     infrequent-access tier           (90-365 days)  │
   │ ARCHIVE  glacier-class                    (1 year+)      │
   └──────────────────────────────────────────────────────────┘

   Lifecycle policies move objects automatically.
   ⭐ This routinely cuts storage cost by 60-80% with no
     user-visible impact, because almost nobody requests
     a 3-year-old photo — and when they do, a slightly
     slower first load is acceptable.

   ⚠️ Keep the ORIGINAL for re-processing (new formats, new
     resolutions, ML re-analysis), but keep it in cold storage.
     Serve only the derived variants.
```

## 🎤 Interview Follow-Ups

<details>
<summary><b>"Walk me through the upload path."</b></summary>

The client asks the API for an upload URL. The server authenticates, validates quota, and returns a presigned URL scoped to a specific key with a short expiry. The client uploads directly to object storage — the bytes never touch my application servers.

That's the critical decision. Proxying uploads means paying for twice the bandwidth, tying up application servers for the duration of each upload, and dealing with timeouts on large videos. With presigned URLs, upload capacity scales with object storage rather than with my fleet.

The object storage write emits an event that triggers the processing pipeline asynchronously: generate resized variants, re-encode to modern formats, strip EXIF for privacy, generate a blurhash placeholder, and run content moderation and tagging models.

Meanwhile the metadata — caption, author, timestamp — is written to the database immediately, so the post exists right away with a placeholder while processing completes. The client can show the local copy optimistically.
</details>

<details>
<summary><b>"How does feed ranking work?"</b></summary>

Two stages, because running an expensive model over every candidate post is computationally impossible at hundreds of thousands of QPS.

Stage one is candidate generation: cheap, high-recall retrieval pulling from several sources — recent posts from people you follow, posts from accounts similar to ones you engage with, trending content in your interest clusters. This cuts millions of possibilities to a few hundred.

Stage two runs a real model over that shortlist, predicting engagement probabilities — likelihood of a like, comment, share, dwell time, profile visit — and importantly, likelihood of negative feedback, which is weighted heavily downward.

Stage three applies business rules: diversity so you don't see five posts from one account, ad insertion, freshness boosts, and integrity filters.

The signals that matter most are closeness signals — how often you view someone's profile, whether you message them — and engagement velocity, meaning likes per minute since posting, which is a strong early quality indicator.

That cheap-recall-then-expensive-precision funnel is the universal pattern across recommendations, search, and ads.
</details>

<details>
<summary><b>"How do you store 180 PB per year affordably?"</b></summary>

Tiering, driven by the access decay curve. Roughly 80% of a photo's lifetime views happen in the first day, and it drops off steeply after that.

So: hot object storage plus CDN for the first week, standard storage to about ninety days, infrequent-access tier to a year, and archive-class storage beyond that. Lifecycle policies move objects automatically with no application code involved.

That's typically a 60 to 80% cost reduction with no user-visible impact, because almost nobody requests a three-year-old photo — and when they do, a slightly slower first load is acceptable.

Beyond tiering: aggressive modern codecs like AVIF cut sizes substantially, serving everything through a CDN means origin egress is near zero, and generating only the variants that are actually requested rather than all of them upfront saves both storage and processing.

I'd keep originals for reprocessing — new formats and new model versions come along — but in cold storage, serving only derived variants.
</details>

---

## 📋 Volume Summary

```
╔══════════════════════════════════════════════════════════════════════╗
║                   CASE STUDIES VOL. 1 — RECALL                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ UBER      geospatial: geohash/S2/H3 · hexagons = uniform neighbours  ║
║   ⭐ GEOGRAPHIC SHARDING turns global scale into N city-scale problems║
║   ⭐ conditional atomic update (not a lock) prevents double-assignment ║
║   ⭐ split hot path (in-memory, lossy) from cold path (Kafka, durable)║
║   explicit state machine survives unreliable mobile networks         ║
╠══════════════════════════════════════════════════════════════════════╣
║ NETFLIX   100 Tbps peak → you CANNOT serve from datacenters          ║
║   ⭐ own CDN inside ISPs + PREDICTIVE FILL during off-peak            ║
║   ABR: client decides per segment (only it knows real conditions)    ║
║   per-title/per-shot encoding — huge bandwidth savings               ║
║   ⭐ precompute personalization offline; request path does no work    ║
║   graceful degradation: PLAYBACK NEVER STOPS, everything else optional║
╠══════════════════════════════════════════════════════════════════════╣
║ TWITTER   1000:1 read ratio → precompute on WRITE                    ║
║   ⭐ HYBRID FAN-OUT: push for normal users, pull for celebrities      ║
║   refinements: active users only · cap at ~800 · async · store IDs   ║
║   deletes → tombstone + filter on READ (never un-fan-out)            ║
║   hot key → replicate across nodes + L1 cache; sharded counters      ║
╠══════════════════════════════════════════════════════════════════════╣
║ WHATSAPP  CONNECTIONS are the constraint, not throughput             ║
║   Erlang/BEAM: ~1M connections/server, hot reload, supervision       ║
║   ⭐ delivered = DELETED — storage problem mostly disappears          ║
║   client-generated message ID → retries are idempotent               ║
║   Signal: X3DH + Double Ratchet (forward secrecy + break-in recovery)║
║   sender keys make group E2E O(1) per message                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ INSTAGRAM ⭐ presigned URLs — bytes never touch your app servers      ║
║   two-stage ranking: cheap RECALL → expensive PRECISION              ║
║   stories: 24h TTL makes the working set BOUNDED                     ║
║   ⭐ storage tiering on the access decay curve = 60-80% cost cut      ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Case Studies Vol. 2 →](04-case-studies-2.md) · **Related:** [Framework](02-framework.md) · [Building Blocks](01-building-blocks.md)
