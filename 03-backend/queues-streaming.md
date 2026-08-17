# 📨 Message Queues & Event Streaming

> Queues decouple systems in time. The hard part is not sending messages — it's what happens when they arrive twice, out of order, or not at all.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [Queue vs Log](#2-queue-vs-log)
3. [Delivery Semantics](#3-delivery-semantics)
4. [Idempotency and Deduplication](#4-idempotency)
5. [Ordering](#5-ordering)
6. [Apache Kafka](#6-kafka)
7. [RabbitMQ](#7-rabbitmq)
8. [Cloud Queues](#8-cloud-queues)
9. [Failure Handling](#9-failure-handling)
10. [The Outbox Pattern](#10-outbox-pattern)
11. [Event-Driven Architecture](#11-event-driven-architecture)
12. [Sagas and Distributed Transactions](#12-sagas)
13. [Stream Processing](#13-stream-processing)
14. [Operations](#14-operations)
15. [Interview Section](#15-interview-section)
16. [Cheat Sheet](#16-cheat-sheet)

---

## 1. Mental Model

```
   SYNCHRONOUS (RPC)              ASYNCHRONOUS (queue)
   ─────────────────              ────────────────────
   A ──────request──────▶ B       A ──▶ [ QUEUE ] ──▶ B
     ◀─────response──────
                                  A doesn't wait.
   A is coupled to:               A doesn't know B exists.
     • B being up                 B can be down for an hour.
     • B's latency                B can scale independently.
     • B's capacity               Load is buffered, not shed.
   A fails when B fails.
```

**What a queue actually buys you:**

| Benefit | Mechanism |
|---|---|
| **Decoupling** | Producer doesn't know the consumer |
| **Buffering** | Absorbs traffic spikes; consumers work at their own rate |
| **Resilience** | Consumer downtime delays work, doesn't lose it |
| **Load leveling** | 10k/s burst → steady 1k/s processing |
| **Fan-out** | One event, many independent consumers |
| **Retry** | Built-in redelivery on failure |

**What it costs you:**

```
   ❌ Eventual consistency — the work isn't done when you respond
   ❌ Duplicates — at-least-once is the practical default
   ❌ Ordering is not free
   ❌ Debugging spans process boundaries and time
   ❌ Another system to operate, monitor, and secure
   ❌ Backpressure must be designed, not assumed
```

🏭 **Rule:** if the caller needs the result to respond, use RPC. If the work can happen later, use a queue. The mistake is using a queue and then polling for completion — that's RPC with extra steps.

---

## 2. Queue vs Log

This is the fundamental architectural split.

```
   MESSAGE QUEUE (RabbitMQ, SQS)
   ─────────────────────────────
   ┌───┬───┬───┬───┬───┐
   │ 5 │ 4 │ 3 │ 2 │ 1 │ ──▶ consumer ──▶ ACK ──▶ message DELETED
   └───┴───┴───┴───┴───┘
   • Messages are removed once acknowledged
   • Competing consumers: each message goes to ONE consumer
   • The broker tracks per-message state
   • Natural work distribution


   EVENT LOG (Kafka, Pulsar, Kinesis)
   ──────────────────────────────────
   offset:  0   1   2   3   4   5   6   7   8
          ┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
          │ e │ e │ e │ e │ e │ e │ e │ e │ e │   append-only, immutable
          └───┴───┴───┴───┴───┴───┴───┴───┴───┘
                    ▲               ▲
              consumer A       consumer B
              (offset 3)       (offset 7)
   • Messages are RETAINED (by time or size), not deleted on read
   • Each consumer group tracks its OWN offset
   • Replay is trivial: reset the offset
   • Multiple independent consumers read the same data
```

| | Queue | Log |
|---|---|---|
| After consumption | Deleted | Retained |
| Multiple consumers | Compete for messages | Each gets everything |
| Replay | ❌ (gone) | ✅ (rewind offset) |
| Ordering | Per queue (fragile with concurrency) | Per partition (strong) |
| Throughput | High | **Very high** (sequential disk I/O) |
| Per-message operations | ✅ delay, priority, selective ack | ❌ offsets only |
| Consumer state | Broker holds it | Consumer holds it |
| Best for | Task distribution, RPC-ish work | Event sourcing, analytics, fan-out |

**Choosing:**

```
   "Do this job"          → QUEUE   (task, one worker, may need priority/delay)
   "This happened"        → LOG     (event, many interested parties, replayable)
```

---

## 3. Delivery Semantics

```
   AT-MOST-ONCE     fire and forget; ack before processing
                    ✅ fastest, simplest
                    ❌ messages lost on crash
                    Use: metrics, logs, telemetry where loss is acceptable

   AT-LEAST-ONCE    ack after processing; retry on failure     ⭐ the default
                    ✅ no loss
                    ❌ DUPLICATES — your consumer MUST be idempotent
                    Use: essentially everything

   EXACTLY-ONCE     ⚠️ impossible in general across system boundaries
                    Achievable as at-least-once DELIVERY plus
                    idempotent PROCESSING, or within a single
                    transactional system (Kafka transactions)
```

### Why exactly-once is a lie (across systems)

```
   Consumer                        External System (DB, API)
     │  process message              │
     ├──────────────────────────────▶│  side effect applied ✅
     │                               │
     │  ✗ CRASH before ack           │
     │                               │
   (restart)                         │
     │  message redelivered          │
     ├──────────────────────────────▶│  side effect applied AGAIN ❌

   There is no way to make "apply the effect" and "record the ack"
   atomic when they live in different systems. The Two Generals
   Problem applies.

   → The answer is not stronger delivery. It's IDEMPOTENT PROCESSING.
```

Kafka's "exactly-once semantics" is real but bounded: it works when the input, the processing, and the output are all within Kafka (read → transform → write, with the offset commit in the same transaction). The moment you write to an external database or call an external API, you're back to at-least-once plus idempotency.

---

## 4. Idempotency

The single most important consumer property. Design for it from the start.

### 4.1 Natural idempotency

```python
# ✅ Idempotent by nature — same result no matter how many times it runs
await db.execute("UPDATE users SET status = 'active' WHERE id = $1", uid)   # SET
await redis.sadd("processed", event_id)                                     # SET add
await s3.put_object(Key=key, Body=body)                                     # overwrite

# ❌ NOT idempotent
await db.execute("UPDATE accounts SET balance = balance - 100 WHERE id=$1") # delta
await send_email(to, subject, body)                                         # side effect
await payment_api.charge(card, 5000)                                        # side effect
```

### 4.2 Dedup table

```python
async def handle(event: Event):
    async with db.transaction():
        try:
            await db.execute(
                "INSERT INTO processed_events (event_id, processed_at) VALUES ($1, now())",
                event.id)
        except UniqueViolation:
            logger.info("duplicate, skipping", event_id=event.id)
            return                                   # already done

        # ⭐ Side effect and dedup record commit TOGETHER — atomic
        await apply_business_logic(event)
```

The key insight: put the dedup insert and the side effect **in the same transaction**. If they're separate, a crash between them either double-processes or silently drops.

```sql
CREATE TABLE processed_events (
  event_id     text PRIMARY KEY,
  processed_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON processed_events (processed_at);       -- for cleanup
-- Purge older than your maximum redelivery window
DELETE FROM processed_events WHERE processed_at < now() - interval '7 days';
```

### 4.3 Conditional update (optimistic)

```sql
-- Only apply if we haven't seen this version — no separate dedup table
UPDATE orders
SET status = $2, last_event_id = $3, version = version + 1
WHERE id = $1 AND version = $4;
-- 0 rows affected = already applied or out of order
```

### 4.4 Effect-scoped idempotency keys

For external calls, pass through an idempotency key derived from the event:

```python
await stripe.charge(
    amount=event.amount,
    idempotency_key=f"order-{event.order_id}-charge",   # deterministic
)
```

The downstream system does the deduplication. This is why every good payment API supports idempotency keys (see [API Design §9](api-design.md#9-idempotency)).

---

## 5. Ordering

```
   Ordering is only guaranteed within a PARTITION (Kafka) or a
   single-consumer QUEUE (RabbitMQ).

   ┌─────────────────────────────────────────────────────────┐
   │ Producer sends: A1, A2, A3 (all for user A)             │
   │                                                          │
   │ 3 partitions, round-robin:                              │
   │   P0: A1        P1: A2        P2: A3                    │
   │   3 consumers process in parallel → ORDER LOST          │
   │                                                          │
   │ Partition by user_id:                                   │
   │   P1: A1, A2, A3  → one consumer, ORDER PRESERVED ✅     │
   └─────────────────────────────────────────────────────────┘
```

**The universal technique: partition by entity key.**

```java
// Kafka: same key → same partition → ordered
producer.send(new ProducerRecord<>("orders", order.getUserId(), event));
```

```
   TRADEOFF
   Ordering requires serialization within a key.
   
   ✅ Order preserved per user
   ❌ Parallelism limited to the number of partitions
   ❌ A "celebrity" key (one very active user) creates a hot partition
   ❌ Adding partitions changes the key→partition mapping and
      breaks ordering during the transition
```

**Multiple consumers with ordering** — RabbitMQ's consistent-hash exchange or Kafka's partition assignment both solve this by routing a key consistently to one consumer.

### Handling out-of-order messages

```python
# Version/sequence check — reject stale events
if event.version <= current.version:
    return                          # old news, ignore

# Or use event timestamps with a buffering window (watermarks)
# Or make handlers commutative so order doesn't matter
```

---

## 6. Kafka

### 6.1 Architecture

```
   TOPIC "orders" — 3 partitions, replication factor 3

   Partition 0  [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ]  leader: broker1  ISR: {1,2,3}
   Partition 1  [ 0 ][ 1 ][ 2 ]            leader: broker2  ISR: {2,3,1}
   Partition 2  [ 0 ][ 1 ][ 2 ][ 3 ]       leader: broker3  ISR: {3,1,2}

   ┌──────────────────────────────────────────────────────────────┐
   │  PRODUCERS                                                   │
   │     key → hash → partition (or round-robin if no key)        │
   └────────────────────────┬─────────────────────────────────────┘
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  BROKERS                                                     │
   │   • Append to a segment file — SEQUENTIAL disk write         │
   │   • OS page cache serves reads (zero-copy sendfile)          │
   │   • Replicate to followers; ISR = in-sync replicas           │
   └────────────────────────┬─────────────────────────────────────┘
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  CONSUMER GROUP "billing"                                    │
   │    consumer-1 → partition 0                                  │
   │    consumer-2 → partition 1, 2                               │
   │    ⚠️ More consumers than partitions = idle consumers         │
   │                                                              │
   │  CONSUMER GROUP "analytics"  ← independent offsets,          │
   │    reads the SAME data                                       │
   └──────────────────────────────────────────────────────────────┘
```

**Why Kafka is fast:**

```
   1. Sequential disk I/O — appending to a log is as fast as
      sequential RAM access on many workloads, and far faster
      than random disk I/O
   2. Zero-copy (sendfile) — data goes disk → NIC without
      passing through user space
   3. Batching + compression at the producer
   4. No per-message broker state — the consumer owns its offset
   5. Partitioning = linear horizontal scaling
```

### 6.2 Producer configuration

```java
props.put("acks", "all");              // wait for all ISR — no data loss
props.put("enable.idempotence", true); // dedupe producer retries (seq numbers)
props.put("max.in.flight.requests.per.connection", 5);  // ≤5 with idempotence
props.put("retries", Integer.MAX_VALUE);
props.put("compression.type", "zstd"); // or lz4 for lower CPU
props.put("linger.ms", 10);            // ⭐ batch for 10ms — huge throughput win
props.put("batch.size", 65536);
props.put("delivery.timeout.ms", 120000);
```

```
   acks=0    fire and forget           fastest, data loss on any failure
   acks=1    leader only               loses data if the leader dies before replication
   acks=all  all in-sync replicas ⭐   no loss (with min.insync.replicas ≥ 2)
```

⚠️ **`acks=all` alone is not enough.** With `min.insync.replicas=1`, "all ISR" can mean one replica. Set `min.insync.replicas=2` with `replication.factor=3` — that tolerates one broker loss while still guaranteeing durability.

### 6.3 Consumer configuration

```java
props.put("enable.auto.commit", false);      // ⭐ commit manually, AFTER processing
props.put("auto.offset.reset", "earliest");  // or "latest" for new groups
props.put("max.poll.records", 500);
props.put("max.poll.interval.ms", 300000);   // must exceed your processing time!
props.put("session.timeout.ms", 45000);
props.put("isolation.level", "read_committed");   // skip aborted transactions
```

```java
while (running) {
    var records = consumer.poll(Duration.ofMillis(1000));
    for (var record : records) {
        process(record);                     // must be idempotent
    }
    consumer.commitSync();                   // ⭐ AFTER processing → at-least-once
}
```

⚠️ **`enable.auto.commit=true` gives you at-most-once with data loss.** Offsets commit on a timer regardless of whether processing succeeded, so a crash loses everything since the last commit. Always commit manually after processing.

⚠️ **`max.poll.interval.ms`** is the most common cause of infinite rebalance loops: if processing a batch takes longer than this, the broker assumes the consumer died, rebalances, and the reassigned consumer hits the same problem.

### 6.4 Rebalancing

```
   Triggered by: a consumer joining/leaving, a consumer timing out,
                 or partition count changing.

   EAGER (default, old)      stop-the-world: ALL consumers stop, all
                             partitions revoked, then reassigned
   COOPERATIVE STICKY ⭐      incremental: only the moving partitions
                             are revoked; others keep processing

   props.put("partition.assignment.strategy",
             "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
   
   STATIC MEMBERSHIP — group.instance.id survives restarts without
                       triggering a rebalance. Essential for rolling deploys.
```

### 6.5 Retention and compaction

```
   DELETE (default)     retention.ms=604800000     drop segments after 7 days
   COMPACT              keep only the LATEST value per key, forever

   Compaction turns a topic into a durable changelog / snapshot:

   before:  (k1,v1) (k2,v2) (k1,v3) (k3,v4) (k2,v5)
   after:                   (k1,v3) (k3,v4) (k2,v5)

   → Used for: CDC streams, config topics, Kafka Streams state stores
   → A null value is a "tombstone" — marks the key for deletion
```

---

## 7. RabbitMQ

### 7.1 The exchange model

```
   Producer ──▶ EXCHANGE ──(binding + routing key)──▶ QUEUE ──▶ Consumer

   ┌──────────────────────────────────────────────────────────────┐
   │ DIRECT     routing_key == binding_key exactly                │
   │            "order.created" → queue bound to "order.created"  │
   ├──────────────────────────────────────────────────────────────┤
   │ TOPIC ⭐    pattern matching                                  │
   │            *  = exactly one word    #  = zero or more        │
   │            "order.*.eu"  matches  order.created.eu           │
   │            "order.#"     matches  order.created.eu.paid      │
   ├──────────────────────────────────────────────────────────────┤
   │ FANOUT     ignore routing key, broadcast to ALL bound queues │
   ├──────────────────────────────────────────────────────────────┤
   │ HEADERS    match on header attributes rather than routing key│
   └──────────────────────────────────────────────────────────────┘
```

### 7.2 Reliability configuration

```python
# Producer
channel.confirm_delivery()                        # publisher confirms
channel.basic_publish(
    exchange='orders', routing_key='order.created',
    body=payload,
    properties=pika.BasicProperties(
        delivery_mode=2,                          # ⭐ persistent message
        message_id=str(event_id),
        content_type='application/json',
    ))

# Queue must ALSO be durable, or it vanishes on broker restart
channel.queue_declare(queue='orders', durable=True, arguments={
    'x-message-ttl': 86_400_000,
    'x-max-length': 1_000_000,
    'x-dead-letter-exchange': 'dlx',
    'x-queue-type': 'quorum',                     # ⭐ replicated, Raft-based
})

# Consumer
channel.basic_qos(prefetch_count=10)              # ⭐ don't hoard messages
channel.basic_consume(queue='orders', on_message_callback=handle, auto_ack=False)

def handle(ch, method, props, body):
    try:
        process(body)
        ch.basic_ack(method.delivery_tag)
    except TransientError:
        ch.basic_nack(method.delivery_tag, requeue=True)     # retry
    except PermanentError:
        ch.basic_nack(method.delivery_tag, requeue=False)    # → DLX
```

⚠️ **Durability requires all three:** a durable queue, persistent messages (`delivery_mode=2`), and publisher confirms. Missing any one means silent message loss on broker restart.

⚠️ **`prefetch_count` matters enormously.** The default (unlimited) means one consumer grabs the entire queue into memory while others starve. Set it to roughly the number of messages a consumer can process concurrently.

### 7.3 Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Distributed log | Broker with routing |
| Throughput | Millions/s | ~50k/s per queue |
| Latency | ~5-10 ms | **~1 ms** |
| Retention | Days/forever | Until consumed |
| Replay | ✅ | ❌ |
| Routing | Topic + partition only | **Rich** (topic/header/direct) |
| Per-message features | ❌ | Priority, TTL, delay, selective ack |
| Ordering | Per partition | Per queue |
| Operational complexity | Higher | Lower |
| Best for | Event streaming, analytics, CDC, fan-out | Task queues, RPC, complex routing |

---

## 8. Cloud Queues

### AWS SQS

```
   STANDARD                          FIFO
   ────────                          ────
   Unlimited throughput              300 TPS (3000 with batching)
   At-least-once                     Exactly-once processing (5-min dedup window)
   Best-effort ordering              Strict ordering per MessageGroupId
   Cheaper                           More expensive
```

```python
sqs.send_message(
    QueueUrl=url, MessageBody=json.dumps(payload),
    MessageGroupId=str(user_id),                 # FIFO: ordering scope
    MessageDeduplicationId=event_id,             # FIFO: 5-minute dedup
)

resp = sqs.receive_message(
    QueueUrl=url, MaxNumberOfMessages=10,
    WaitTimeSeconds=20,          # ⭐ LONG POLLING — cuts cost and latency
    VisibilityTimeout=60,        # must exceed processing time
)
# Delete = ack. Not deleting = redelivery after VisibilityTimeout.
sqs.delete_message(QueueUrl=url, ReceiptHandle=msg['ReceiptHandle'])

# Extend visibility for long-running work
sqs.change_message_visibility(QueueUrl=url, ReceiptHandle=rh, VisibilityTimeout=300)
```

⚠️ **`VisibilityTimeout` shorter than processing time** causes the message to be redelivered while you're still working on it — the same job runs twice concurrently. This is the #1 SQS bug.

| Service | Model | Use |
|---|---|---|
| **SQS** | Queue | Task distribution, decoupling |
| **SNS** | Pub/sub fan-out | One event → many SQS queues, Lambda, HTTP |
| **EventBridge** | Event bus with rules/filtering | SaaS integrations, schema-based routing |
| **Kinesis** | Log (Kafka-like) | Streaming, replay, ordered shards |
| **MSK** | Managed Kafka | You need actual Kafka |

**The standard AWS fan-out pattern:**

```
                     ┌──▶ SQS(billing)   ──▶ Lambda
   Event ──▶ SNS ────┼──▶ SQS(email)     ──▶ ECS worker
                     └──▶ SQS(analytics) ──▶ Firehose → S3

   SNS gives fan-out; SQS gives each consumer its own buffer,
   retry, and DLQ. This combination is better than SNS→Lambda
   directly because the queue absorbs consumer downtime.
```

---

## 9. Failure Handling

### 9.1 Retry with backoff

```python
DELAYS = [1, 5, 30, 120, 600]        # seconds — exponential-ish

async def handle_with_retry(msg):
    attempt = int(msg.headers.get("x-attempt", 0))
    try:
        await process(msg)
    except TransientError as e:
        if attempt >= len(DELAYS):
            await send_to_dlq(msg, reason=str(e))
            return
        delay = DELAYS[attempt] * (0.5 + random.random())   # ⭐ jitter
        await schedule_retry(msg, delay, attempt + 1)
    except PermanentError as e:
        await send_to_dlq(msg, reason=str(e))               # don't retry
```

**Classify errors — this is what separates good consumers from bad ones:**

```
   TRANSIENT → retry
     network timeout, 503, deadlock, rate limited, temporary unavailability

   PERMANENT → dead-letter immediately
     malformed payload, validation failure, 400, referenced entity deleted,
     business rule violation

   Retrying a permanent error wastes capacity and delays real work.
   Dead-lettering a transient error loses work unnecessarily.
```

### 9.2 Dead Letter Queues

```
   main queue ──▶ consumer ──✗ fail 5× ──▶ DLQ

   A DLQ is not a graveyard. It requires:
   □ An alert when depth > 0 ⭐
   □ A dashboard showing the failure reason per message
   □ A documented replay mechanism
   □ A runbook: who looks at it, how often, what they do
```

⚠️ **An unmonitored DLQ is worse than no DLQ** — it silently absorbs failures and creates the illusion of a healthy system while data quietly disappears.

### 9.3 Poison messages

```python
# A message that ALWAYS fails will consume retries forever and can
# block an ordered partition permanently.

MAX_ATTEMPTS = 5
if attempt >= MAX_ATTEMPTS:
    await dlq.send(msg, error=last_error, attempts=attempt)
    await commit_offset()      # ⭐ MOVE PAST IT — don't block the partition
    return
```

In Kafka, a poison message in a partition blocks everything behind it, because offsets advance in order. You must either skip it (commit past it) or route it to a DLQ topic — the partition cannot wait.

### 9.4 Backpressure

```
   Producer 10k/s ──▶ [QUEUE growing] ──▶ Consumer 1k/s
                       💥 unbounded growth → memory/disk exhaustion

   Strategies:
   1. Bounded queue + block the producer     ✅ propagates pressure upstream
   2. Bounded queue + reject (429)           ✅ shed load explicitly at the edge
   3. Drop oldest / drop newest              ✅ for metrics, telemetry
   4. Autoscale consumers on queue depth ⭐   ✅ the usual answer
   5. Rate-limit the producer

   ❌ Unbounded queue with "we'll catch up later" — you won't
```

```yaml
# KEDA — scale Kubernetes consumers on queue depth
triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.../orders
      queueLength: "50"        # target ~50 messages per replica
```

---

## 10. Outbox Pattern

**The dual-write problem** — the single most important pattern in this book:

```
   ❌ BROKEN
   async def create_order(data):
       order = await db.insert(data)          # ✅ committed
       await kafka.send("order.created", order)  # ✗ CRASH HERE
       # Order exists in the DB, but no event was ever published.
       # Downstream systems never learn about it. Silent inconsistency.

   Swapping the order doesn't help — then you publish an event
   for an order that doesn't exist.

   You cannot atomically write to two systems.
```

### The solution: write both to the database, in one transaction

```
   ┌──────────────────────────────────────────────────────┐
   │  ONE DATABASE TRANSACTION                            │
   │    INSERT INTO orders  (...)                         │
   │    INSERT INTO outbox  (event payload)               │
   │  COMMIT  ← atomic: both or neither ✅                 │
   └───────────────────────┬──────────────────────────────┘
                           ▼
   ┌──────────────────────────────────────────────────────┐
   │  RELAY (separate process)                            │
   │    poll outbox WHERE published_at IS NULL            │
   │    → publish to Kafka                                │
   │    → mark published                                  │
   │  (or: Debezium reads the WAL directly — no polling)  │
   └───────────────────────┬──────────────────────────────┘
                           ▼
                     Kafka / RabbitMQ
```

```sql
CREATE TABLE outbox (
  id            bigserial PRIMARY KEY,
  aggregate_type text NOT NULL,          -- 'order'
  aggregate_id   text NOT NULL,          -- '123'
  event_type     text NOT NULL,          -- 'order.created'
  payload        jsonb NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  published_at   timestamptz
);
CREATE INDEX ON outbox (created_at) WHERE published_at IS NULL;   -- partial index
```

```python
# Write side
async def create_order(data):
    async with db.transaction():
        order = await db.insert_order(data)
        await db.insert_outbox(
            aggregate_type="order", aggregate_id=order.id,
            event_type="order.created", payload=order.to_dict())
    return order          # both committed atomically

# Relay side (or use Debezium CDC and skip this entirely)
async def relay():
    while True:
        rows = await db.fetch("""
            SELECT * FROM outbox WHERE published_at IS NULL
            ORDER BY id LIMIT 100
            FOR UPDATE SKIP LOCKED""")           # ⭐ safe with many relay instances
        for row in rows:
            await kafka.send(row.event_type, key=row.aggregate_id, value=row.payload)
            await db.execute("UPDATE outbox SET published_at=now() WHERE id=$1", row.id)
        if not rows:
            await asyncio.sleep(0.5)
```

This gives **at-least-once publication** — a crash between publishing and marking published republishes the event. Which is fine, because consumers are idempotent.

🏭 **Debezium** reads the database WAL directly and publishes changes to Kafka with no polling and no relay process. It's the production-grade version of this pattern, and it also captures writes made outside your application.

---

## 11. Event-Driven Architecture

### 11.1 Event types

```
   EVENT NOTIFICATION       thin — "order 123 changed"
     ✅ small, low coupling
     ❌ consumer must call back for details (chatty, couples to the API)

   EVENT-CARRIED STATE ⭐    fat — full order data in the event
     ✅ consumer is autonomous, no callback needed
     ❌ larger messages, data duplication, versioning matters

   EVENT SOURCING           the event log IS the source of truth
     ✅ full audit history, time travel, rebuild any projection
     ❌ significant complexity, schema evolution is hard, queries need projections
```

### 11.2 Event schema design

```json
{
  "event_id": "evt_01HXYZ",                    // for deduplication
  "event_type": "order.created",
  "event_version": "1.2",                      // ⭐ schema version
  "occurred_at": "2026-08-14T10:00:00Z",       // when it HAPPENED
  "published_at": "2026-08-14T10:00:01Z",      // when it was SENT
  "aggregate_id": "ord_123",
  "aggregate_type": "order",
  "correlation_id": "req_abc",                 // ⭐ traces one user action
  "causation_id": "evt_01HXYW",                // which event caused this one
  "actor": { "type": "user", "id": "usr_9" },
  "data": { "order_id": "ord_123", "total_cents": 4500, "currency": "USD" }
}
```

**Schema evolution rules** (same discipline as [protobuf](api-design.md#13-grpc)):

```
   ✅ Add optional fields
   ✅ Add new event types
   ❌ Remove or rename fields
   ❌ Change field types
   ❌ Change the meaning of an existing field

   Use a schema registry (Confluent, AWS Glue) to ENFORCE compatibility
   at publish time rather than discovering breakage in production.
```

### 11.3 Naming

```
   ✅ order.created        past tense — it already happened
   ✅ payment.failed
   ✅ user.email_changed

   ❌ create_order         imperative = a COMMAND, not an event
   ❌ order.creating       events are facts, not progress reports

   Commands go to ONE handler and may be rejected.
   Events go to MANY subscribers and cannot be rejected — they're history.
```

### 11.4 Choreography vs Orchestration

```
   CHOREOGRAPHY — services react to events independently

   OrderService ──event──▶ PaymentService ──event──▶ ShippingService
                                                  ──▶ EmailService
   ✅ loose coupling, easy to add consumers
   ❌ no single place shows the whole flow
   ❌ debugging requires tracing across services
   ❌ cyclic dependencies creep in unnoticed


   ORCHESTRATION — a coordinator drives the flow

              ┌─────────────────┐
              │  OrderSaga      │
              └────┬───┬───┬────┘
          ┌────────┘   │   └────────┐
          ▼            ▼            ▼
      Payment      Shipping      Email
   ✅ the flow is explicit and testable
   ✅ centralized error handling and compensation
   ❌ the orchestrator becomes a coupling point
```

🏭 Use choreography for simple fan-out ("order created → notify these five systems"). Use orchestration for multi-step business processes with compensation ("checkout: reserve, charge, ship, or undo everything").

---

## 12. Sagas

Distributed transactions without 2PC. Each step has a compensating action.

```
   HAPPY PATH
   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Reserve │─▶│ Charge  │─▶│ Create  │─▶│ Notify  │
   │inventory│  │ payment │  │ shipment│  │  user   │
   └─────────┘  └─────────┘  └─────────┘  └─────────┘

   FAILURE AT STEP 3 → compensate in REVERSE order
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Release │◀─│ Refund  │◀─│  FAIL   │
   │inventory│  │ payment │  │         │
   └─────────┘  └─────────┘  └─────────┘
```

```python
class CheckoutSaga:
    steps = [
        (reserve_inventory, release_inventory),
        (charge_payment,    refund_payment),
        (create_shipment,   cancel_shipment),
    ]

    async def execute(self, ctx):
        completed = []
        try:
            for action, compensation in self.steps:
                await action(ctx)                    # each must be idempotent
                completed.append(compensation)
                await self.persist_state(ctx, completed)   # ⭐ survive a crash
        except Exception as e:
            for compensation in reversed(completed):
                try:
                    await compensation(ctx)          # must ALSO be idempotent
                except Exception:
                    await alert_manual_intervention(ctx)  # compensation failed
            raise
```

**What sagas cost you:**

```
   ❌ No isolation — other transactions see intermediate states
      (an order exists as "pending" while payment is in flight)
   ❌ Compensations aren't true rollbacks — you can't un-send an email;
      you send an apology
   ❌ Saga state must be durable, or a crash leaves it half-done forever
   ❌ Compensations can themselves fail → manual intervention queue
   ❌ Reasoning about all interleavings is genuinely hard
```

**Why not two-phase commit?** 2PC gives real atomicity but blocks: if the coordinator dies after prepare, every participant holds locks indefinitely. It also requires all participants to support it, and it doesn't survive partitions. In practice, sagas plus idempotency win for microservices, and 2PC survives mainly inside single-vendor database clusters.

---

## 13. Stream Processing

```
   STATELESS      map, filter, enrich per record
   STATEFUL       aggregations, joins, windows — needs a state store

   WINDOWS
     Tumbling   |──5m──|──5m──|──5m──|          fixed, non-overlapping
     Hopping    |──5m──|                        overlapping
                   |──5m──|
     Sliding    continuous, per-event windows
     Session    |─active─|  gap  |─active─|     grouped by inactivity gap
```

### Event time vs processing time

```
   EVENT TIME:      when it actually happened  ⭐ correct
   PROCESSING TIME: when your system saw it    (simpler, wrong under delay)

   A mobile client is offline for 2 hours, then uploads.
   Processing time puts those events in today's bucket.
   Event time puts them where they belong.

   WATERMARK = "I believe I've seen all events up to time T"
   → lets a window close. Late arrivals after the watermark are
     either dropped, sent to a side output, or trigger a window update.
```

```java
// Kafka Streams
StreamsBuilder builder = new StreamsBuilder();
builder.stream("orders", Consumed.with(Serdes.String(), orderSerde))
    .filter((k, v) -> v.getTotal() > 100)
    .groupBy((k, v) -> v.getCustomerId())
    .windowedBy(TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1)))
    .aggregate(Stats::new, (k, v, agg) -> agg.add(v), Materialized.with(...))
    .toStream()
    .to("order-stats");
```

| Framework | Model | Strength |
|---|---|---|
| **Kafka Streams** | Library in your app | No cluster needed; Kafka-native |
| **Flink** | Distributed runtime | Best event-time/watermark semantics, true streaming |
| **Spark Structured Streaming** | Micro-batch | Unified batch + stream, big ecosystem |
| **ksqlDB** | SQL over Kafka | Fastest to write, limited expressiveness |

---

## 14. Operations

### Metrics that matter

| Metric | Alert on | Why |
|---|---|---|
| **Consumer lag** | > threshold, or growing | ⭐ The single most important queue metric |
| Queue depth | Sustained growth | Consumers can't keep up |
| Oldest message age | > SLA | Latency of the *worst* message |
| DLQ depth | **> 0** | Something is failing |
| Processing time p99 | > timeout budget | Rebalance risk |
| Rebalance frequency | Any sustained rate | Config problem |
| Redelivery rate | Rising | Idempotency or timeout issue |

```
   Consumer lag = latest offset − committed offset

   Lag alone is not enough — lag of 1M on a 100k/s consumer is 10 seconds;
   lag of 1000 on a 10/s consumer is 100 seconds. Alert on
   ESTIMATED TIME TO DRAIN, not raw lag.
```

### Testing

```
   □ Send the same message twice — does the result stay correct? (idempotency)
   □ Kill the consumer mid-processing — is the message redelivered?
   □ Send a malformed message — does it dead-letter without blocking?
   □ Send 10× normal volume — does backpressure work or does memory explode?
   □ Stop the consumer for an hour — does it catch up, and how fast?
   □ Deliver messages out of order — is the final state still correct?
   □ Kill the broker leader — do producers and consumers recover?
```

---

## 15. Interview Section

<details>
<summary><b>Q1. When do you use a message queue instead of a direct call?</b></summary>

When the caller doesn't need the result to respond. If a user submits an order, the payment must be confirmed synchronously, but sending the confirmation email, updating analytics, and notifying the warehouse can all happen later.

The concrete benefits: the API stays fast because it doesn't wait for slow downstream work; a downstream outage delays work instead of failing the user's request; traffic spikes get buffered rather than shed; and you can add new consumers to an event without touching the producer.

The costs are real though — eventual consistency, duplicate delivery, harder debugging across process boundaries, and another system to operate. So I wouldn't queue something just because it's fashionable.

The anti-pattern I'd flag is using a queue and then polling for completion. That's RPC with extra latency and complexity.
</details>

<details>
<summary><b>Q2. Explain delivery semantics. Is exactly-once real?</b></summary>

At-most-once acknowledges before processing, so a crash loses the message. Fine for metrics, not for orders.

At-least-once acknowledges after processing, so a crash causes redelivery. This is the practical default, and it means duplicates are guaranteed to happen eventually.

Exactly-once across system boundaries is not achievable. The reason is the Two Generals Problem: applying a side effect in an external system and recording the acknowledgment are two operations in two systems, and you can't make them atomic. A crash between them either double-processes or drops.

Kafka's exactly-once semantics is real but bounded — it works when input, processing, and output are all within Kafka, because the offset commit joins the same transaction. The moment you write to an external database or call an external API, you're back to at-least-once.

The correct framing is: at-least-once delivery plus idempotent processing gives you effectively-once *outcomes*. That's what you actually build.
</details>

<details>
<summary><b>Q3. How do you make a consumer idempotent?</b></summary>

Four approaches, in order of preference.

Natural idempotency — design the operation so repeating it is harmless. `SET status = 'active'` is idempotent; `balance = balance - 100` is not. Where possible, express changes as absolute values rather than deltas.

A dedup table with the event ID as the primary key. The critical detail is putting the dedup insert and the side effect in the *same database transaction*, so they commit atomically. If they're separate, a crash between them reintroduces the exact bug you're solving.

Conditional updates using a version or sequence number — apply only if the stored version matches what you expect. Zero rows affected means it was already applied.

For external calls, pass a deterministic idempotency key derived from the event and let the downstream system deduplicate. This is why every serious payment API supports idempotency keys.

I'd also add a retention policy on the dedup table — keys only need to live as long as your maximum redelivery window.
</details>

<details>
<summary><b>Q4. Kafka vs RabbitMQ.</b></summary>

They're different data structures wearing similar clothes. Kafka is a distributed append-only log; RabbitMQ is a broker with routing and per-message state.

Kafka retains messages after consumption, so multiple independent consumer groups read the same data at their own pace and you can replay by resetting an offset. Its throughput is very high because it's sequential disk I/O with zero-copy transfer, and ordering is guaranteed per partition. That makes it right for event streaming, analytics pipelines, CDC, and fan-out where consumers are added over time.

RabbitMQ deletes messages on acknowledgment and gives you rich routing — topic patterns, headers, priorities, per-message TTL, delayed delivery, selective acknowledgment. Latency is lower. That makes it right for task distribution and workflows where individual messages need individual treatment.

The heuristic I use: "this happened, and several parties care" is a log; "do this job" is a queue.
</details>

<details>
<summary><b>Q5. Explain the outbox pattern and the problem it solves.</b></summary>

It solves the dual-write problem: you need to save to your database *and* publish an event, and you cannot make writes to two systems atomic. If you write the database then crash before publishing, downstream systems never learn about the change. Swapping the order just moves the bug.

The outbox pattern makes both writes go to the same database in one transaction — the business record and an outbox row containing the event. That's atomic, so either both exist or neither does.

Then a separate relay process polls the outbox for unpublished rows and publishes them to the broker, marking them published. Using `FOR UPDATE SKIP LOCKED` lets you run multiple relay instances safely.

This gives at-least-once publication, since a crash between publishing and marking republishes — which is fine because consumers are idempotent.

The production-grade version replaces the poller with Debezium reading the database write-ahead log directly. That removes polling latency and also captures writes made outside your application, like manual SQL.
</details>

<details>
<summary><b>Q6. How do you guarantee ordering?</b></summary>

Ordering is only guaranteed within a partition in Kafka or a single-consumer queue in RabbitMQ. So the technique is to partition by the entity key — all events for a given user or order go to the same partition, which is consumed by one consumer, in order.

The tradeoff is that ordering means serialization. Your parallelism is capped at the partition count, a very active key creates a hot partition, and adding partitions changes the key-to-partition mapping, which breaks ordering during the transition.

The better question is usually whether you need ordering at all. Often you can make handlers commutative so order doesn't matter, or include a version number and reject stale events. That's more robust than depending on infrastructure ordering guarantees, because it survives redelivery, replay, and reconfiguration.

Global ordering across all keys requires a single partition, which caps throughput at one consumer — almost never the right answer.
</details>

<details>
<summary><b>Q7. What is a dead letter queue and how do you operate one?</b></summary>

A separate queue receiving messages that failed repeatedly, so they stop consuming retry capacity and stop blocking the main queue.

The important part is classification. Transient errors — timeouts, 503s, deadlocks, rate limits — deserve retries with exponential backoff and jitter. Permanent errors — malformed payloads, validation failures, a referenced entity that no longer exists — should dead-letter immediately, because retrying them wastes capacity and delays real work.

Operationally, a DLQ needs an alert on depth greater than zero, a dashboard showing why each message failed, a documented replay mechanism, and a named owner who checks it. An unmonitored DLQ is worse than none, because it silently absorbs failures while the system looks healthy.

The related issue is poison messages. In Kafka, a message that always fails blocks its entire partition, since offsets advance in order. You have to commit past it after N attempts and route it to a DLQ topic — the partition can't wait.
</details>

<details>
<summary><b>Q8. Explain the saga pattern.</b></summary>

A way to get a multi-service business transaction without distributed locking. You break it into local transactions, each with a compensating action, and if a later step fails you run the compensations in reverse.

Checkout is the canonical example: reserve inventory, charge payment, create shipment. If shipment creation fails, you refund the payment and release the inventory.

Two flavors. Choreography has each service react to events with no coordinator — loosely coupled, but no single place shows the whole flow, so debugging is painful. Orchestration has an explicit coordinator driving each step — the flow is visible and testable, at the cost of centralizing coupling. I'd use choreography for simple fan-out and orchestration for real business processes with compensation.

What sagas genuinely cost: no isolation, so other transactions observe intermediate states. Compensations aren't rollbacks — you can't unsend an email, only send an apology. Saga state has to be persisted or a crash leaves it half-done. And compensations can themselves fail, so you need a manual intervention queue.

Versus two-phase commit: 2PC gives true atomicity but blocks all participants if the coordinator dies, and doesn't survive partitions. For microservices, sagas plus idempotency win.
</details>

<details>
<summary><b>Q9. Your consumer lag is growing. Walk me through diagnosis.</b></summary>

First, quantify it as time to drain rather than raw message count, because a million messages behind a fast consumer is seconds while a thousand behind a slow one is minutes.

Then determine whether it's a producer spike or a consumer slowdown, by comparing production rate to consumption rate over time. Those have completely different fixes.

If it's the consumer: check processing time per message — has p99 grown? Usually the cause is a downstream dependency getting slower, a database query degrading, or a new code path. Check for rebalances, since frequent rebalancing means consumers spend their time reassigning rather than processing, often because processing exceeds `max.poll.interval.ms`. Check for a poison message blocking a partition. And check whether all partitions are lagging or just one — a single lagging partition means a hot key.

Immediate mitigations: scale out consumers if partition count allows, increase batch size, or temporarily shed non-critical work. If consumers already equal partition count, adding more does nothing — you need more partitions, which is a plan rather than a quick fix.

Longer term I'd want autoscaling on lag, an alert on time-to-drain, and enough partition headroom that scaling out is always available.
</details>

<details>
<summary><b>Q10. How do you handle backpressure?</b></summary>

The core principle is that unbounded queues turn a throughput problem into an out-of-memory crash. Bound everything.

The options in order: autoscale consumers on queue depth, which is the usual answer with something like KEDA. Bound the queue and block the producer, which propagates pressure upstream to where it can be handled. Bound the queue and reject at the edge with a 429, which sheds load explicitly and honestly. Or drop messages by policy, which is fine for metrics and telemetry where recency beats completeness.

What I'd avoid is an unbounded queue with the assumption you'll catch up later. If the producer is sustainably faster than the consumer, you never catch up — you just delay the failure and make it worse when it arrives.

There's also a subtlety about where the queue lives. Buffering in the broker is fine; buffering unboundedly inside a consumer process — pulling a large batch into memory and processing slowly — is how you OOM. RabbitMQ's `prefetch_count` and Kafka's `max.poll.records` exist exactly to bound that.
</details>

---

## 16. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                  QUEUES & STREAMING — ONE PAGE                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ QUEUE (RabbitMQ/SQS): deleted on ack, competing consumers, routing   ║
║ LOG (Kafka/Kinesis): retained, per-group offsets, REPLAYABLE         ║
║   "do this job" → queue  |  "this happened" → log                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ DELIVERY: at-most-once(loss) · AT-LEAST-ONCE(dupes) ⭐ · exactly-once ║
║   is IMPOSSIBLE across systems → at-least-once + IDEMPOTENT consumer ║
╠══════════════════════════════════════════════════════════════════════╣
║ IDEMPOTENCY: natural (SET not delta) · dedup table IN THE SAME TXN   ║
║   as the side effect · version check · downstream idempotency key    ║
╠══════════════════════════════════════════════════════════════════════╣
║ OUTBOX — you CANNOT atomically write to DB + broker                  ║
║   INSERT business row + INSERT outbox row in ONE transaction         ║
║   relay polls (FOR UPDATE SKIP LOCKED) or Debezium reads the WAL     ║
╠══════════════════════════════════════════════════════════════════════╣
║ ORDERING only within a partition/queue → partition by ENTITY KEY     ║
║   costs parallelism; hot keys; better to make handlers commutative   ║
╠══════════════════════════════════════════════════════════════════════╣
║ KAFKA: acks=all + min.insync.replicas=2 + enable.idempotence         ║
║   MANUAL offset commit AFTER processing (auto-commit = data loss)    ║
║   max.poll.interval.ms > processing time or infinite rebalances      ║
║   consumers ≤ partitions · compaction = keep latest per key          ║
╠══════════════════════════════════════════════════════════════════════╣
║ RABBIT: durable queue + delivery_mode=2 + publisher confirms (ALL 3) ║
║   set prefetch_count or one consumer hoards the queue                ║
║ SQS: VisibilityTimeout MUST exceed processing time · long polling    ║
╠══════════════════════════════════════════════════════════════════════╣
║ FAILURE: classify transient(retry+backoff+jitter) vs permanent(DLQ)  ║
║   ALERT ON DLQ DEPTH > 0 · poison msg blocks a Kafka partition       ║
║ BACKPRESSURE: bound everything · autoscale on lag · never "catch up" ║
╠══════════════════════════════════════════════════════════════════════╣
║ SAGA: local txns + compensations in reverse. No isolation.           ║
║   Compensations must be idempotent AND can themselves fail.          ║
║ #1 METRIC: consumer lag as TIME TO DRAIN, not message count          ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [API Design](api-design.md) · [Databases](databases.md) · [System Design](../05-system-design/00-fundamentals.md) · [Distributed Theory](../05-system-design/06-distributed-theory.md)
