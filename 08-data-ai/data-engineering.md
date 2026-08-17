# 🏭 Data Engineering

> Data engineering is building systems that move and reshape data reliably. The hard parts are almost never the transformation logic — they're correctness under failure, schema change, and scale.

**Prerequisite:** [SQL](sql.md) · [Databases](../03-backend/databases.md) · [Queues & Streaming](../03-backend/queues-streaming.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [OLTP vs OLAP](#2-oltp-vs-olap)
3. [Warehouse, Lake, Lakehouse](#3-warehouse-lake-lakehouse)
4. [File Formats](#4-file-formats)
5. [ETL vs ELT](#5-etl-vs-elt)
6. [Ingestion](#6-ingestion)
7. [Change Data Capture](#7-change-data-capture)
8. [Dimensional Modeling](#8-dimensional-modeling)
9. [Batch Processing & Spark](#9-batch-processing--spark)
10. [Stream Processing](#10-stream-processing)
11. [Orchestration](#11-orchestration)
12. [Data Quality](#12-data-quality)
13. [Governance & Cost](#13-governance--cost)
14. [Interview Section](#14-interview-section)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. The Mental Model

```
   ⭐ EVERY DATA PLATFORM IS THE SAME FIVE STAGES

   ┌──────────────────────────────────────────────────────────────┐
   │  SOURCES      app DBs · events · APIs · files · SaaS         │
   │      ▼                                                       │
   │  INGESTION    batch · streaming · CDC                        │
   │      ▼                                                       │
   │  STORAGE      lake (raw) · warehouse (modeled)               │
   │      ▼                                                       │
   │  TRANSFORM    clean · join · aggregate · model               │
   │      ▼                                                       │
   │  SERVE        BI · APIs · ML features · reverse ETL          │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE THREE QUESTIONS FOR ANY PIPELINE
     1. What's the LATENCY requirement? (drives batch vs stream)
     2. What are the CORRECTNESS guarantees? (drives idempotency
        and reconciliation design)
     3. What happens when it FAILS mid-run? (this is the one
        people design last and should design first)
```

```
   ⭐ THE MEDALLION ARCHITECTURE — the standard layering

   ┌──────────────────────────────────────────────────────────────┐
   │ BRONZE   RAW, immutable, exactly as received                 │
   │          ⭐ Never transform on write. Keeping raw data means  │
   │            you can REPROCESS when logic changes or a bug is  │
   │            found — which will happen.                        │
   ├──────────────────────────────────────────────────────────────┤
   │ SILVER   Cleaned, deduplicated, typed, conformed             │
   │          One row per real-world entity, quality-checked      │
   ├──────────────────────────────────────────────────────────────┤
   │ GOLD     Business-level aggregates and metrics               │
   │          What analysts and dashboards actually query         │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY BRONZE MATTERS MORE THAN IT LOOKS
     Transformation logic is always wrong eventually. If you
     only kept the transformed output, a bug discovered in
     month six is unrecoverable. With immutable raw data you
     just reprocess.
```

---

## 2. OLTP vs OLAP

```
   ┌──────────────────────┬───────────────────────────────────────┐
   │ OLTP                 │ OLAP                                  │
   ├──────────────────────┼───────────────────────────────────────┤
   │ Many small txns      │ Few huge scans                        │
   │ Read/write a FEW ROWS│ ⭐ Read MANY rows, FEW COLUMNS         │
   │ Normalized (3NF)     │ Denormalized (star schema)            │
   │ ⭐ ROW storage        │ ⭐ COLUMNAR storage                    │
   │ Indexes for point    │ Partitioning + clustering for scans   │
   │   lookups            │                                       │
   │ ms latency, high     │ seconds latency, huge throughput      │
   │   concurrency        │                                       │
   │ Postgres, MySQL      │ Snowflake, BigQuery, Redshift,        │
   │                      │ ClickHouse, DuckDB                    │
   └──────────────────────┴───────────────────────────────────────┘
```

```
   ⭐⭐ WHY COLUMNAR STORAGE CHANGES EVERYTHING

   ROW STORAGE — a query reads whole rows off disk
   ┌────────────────────────────────────────────────┐
   │ [id|name|email|age|city|created] [id|name|...] │
   └────────────────────────────────────────────────┘
     SELECT AVG(age) → ⚠️ reads EVERY column of EVERY row

   COLUMNAR — each column stored contiguously
   ┌──────────┬──────────┬──────────┬──────────────┐
   │ id: 1,2… │ name: …  │ age: …   │ city: …      │
   └──────────┴──────────┴──────────┴──────────────┘
     SELECT AVG(age) → ⭐ reads ONLY the age column

   ⭐ THREE COMPOUNDING WINS
     1. Read only the columns you need — often 10-50× less I/O
     2. ⭐ COMPRESSION is dramatically better, because adjacent
        values in a column are similar (run-length encoding,
        dictionary encoding). 5-20× is typical.
     3. Vectorized execution — process a batch of column values
        per CPU instruction

   ⚠️ THE TRADEOFF: writing or updating a single row means
     touching every column file. Columnar stores are built for
     append-and-scan, not for point updates.
```

---

## 3. Warehouse, Lake, Lakehouse

```
   ┌──────────────────────────────────────────────────────────────┐
   │ DATA WAREHOUSE                                               │
   │   Structured, schema-on-WRITE, SQL-first                     │
   │   ✅ Fast queries · governed · mature tooling                 │
   │   ⚠️ Expensive · structured data only · vendor lock-in        │
   ├──────────────────────────────────────────────────────────────┤
   │ DATA LAKE                                                    │
   │   Raw files in object storage, schema-on-READ                │
   │   ✅ Cheap · any format · ML-friendly                          │
   │   ⚠️⚠️ Becomes a DATA SWAMP without governance — no schema     │
   │     enforcement, no transactions, unknown lineage             │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ LAKEHOUSE (Delta Lake, Iceberg, Hudi)                      │
   │   ⭐ A TRANSACTION LOG on top of files in object storage       │
   │   ✅ ACID transactions on a data lake                          │
   │   ✅ Schema enforcement and evolution                          │
   │   ✅ ⭐ TIME TRAVEL — query the table as of any past version    │
   │   ✅ Efficient upserts and deletes (⭐ GDPR deletion!)          │
   │   ⭐ This is where the industry has converged.                 │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ HOW A TABLE FORMAT ADDS ACID TO OBJECT STORAGE

   The insight: object storage has no transactions, but you can
   build them with a METADATA LOG.

   my_table/
     data/
       part-0001.parquet     ⭐ files are IMMUTABLE
       part-0002.parquet
     _delta_log/  (or metadata/)
       00000.json   → "version 0 consists of files A, B"
       00001.json   → "version 1 adds C, removes A"
       00002.json   → "version 2 adds D"

   ⭐ A "commit" is an atomic write of a new log entry.
     Readers read the log first to learn which files constitute
     the current version.

   ⭐ THIS GIVES YOU FOR FREE:
     • Atomicity — readers never see a half-written update
     • ⭐ Time travel — just read an older log entry
     • Concurrent writers with optimistic conflict detection
     • Efficient deletes — mark files removed, rewrite only
       the affected ones
```

---

## 4. File Formats

```
   ┌──────────┬─────────┬────────────────────────────────────────┐
   │ Format   │ Layout  │ Notes                                  │
   ├──────────┼─────────┼────────────────────────────────────────┤
   │ CSV      │ row     │ ⚠️ No types, no compression, ambiguous  │
   │          │         │ quoting/escaping. Fine for exchange,   │
   │          │         │ terrible for storage.                  │
   │ JSON     │ row     │ Flexible, ⚠️ verbose, slow to parse     │
   │ JSONL    │ row     │ ⭐ One JSON per line — SPLITTABLE, so    │
   │          │         │ it parallelizes. Good landing format.  │
   │ Avro     │ row     │ ⭐ Schema embedded, excellent schema     │
   │          │         │ EVOLUTION. Best for streaming/Kafka.   │
   │ ⭐Parquet │ COLUMNAR│ ⭐ THE analytics standard. Column        │
   │          │         │ pruning, predicate pushdown, great     │
   │          │         │ compression, statistics per row group. │
   │ ORC      │ columnar│ Similar; Hive ecosystem                │
   └──────────┴─────────┴────────────────────────────────────────┘

   ⭐ THE RULE OF THUMB
     Streaming and row-by-row writes  → Avro
     Analytics storage and querying   → Parquet
     Interchange with humans/tools    → CSV or JSONL
```

```
   ⭐ WHY PARQUET IS SO MUCH FASTER — three mechanisms

   1. COLUMN PRUNING
      Reading 3 of 50 columns reads ~6% of the bytes.

   2. ⭐ PREDICATE PUSHDOWN via row-group statistics
      Each row group stores min/max per column. A query with
      `WHERE date = '2026-08-14'` SKIPS ENTIRE ROW GROUPS whose
      max date is earlier — without reading them at all.

   3. ENCODING + COMPRESSION
      Dictionary encoding for low-cardinality columns,
      run-length encoding for sorted data, then Snappy or Zstd.
      ⭐ 5-20× smaller than raw JSON is typical.

   ⭐ THE PRACTICAL CONSEQUENCE: SORT your data by the columns
     you filter on most. It makes min/max statistics tight,
     which makes pushdown effective. Unsorted data defeats it.
```

```
   ⚠️⚠️ THE SMALL FILES PROBLEM — the most common lake mistake

   Streaming writes create thousands of tiny files. Each file
   has metadata overhead, requires a separate request, and the
   query planner spends more time listing files than reading data.

   ⭐ TARGET: 128 MB to 1 GB per file.

   FIXES
     • Compaction jobs that merge small files (Delta OPTIMIZE,
       Iceberg rewrite_data_files)
     • Buffer longer before writing in streaming jobs
     • Repartition before writing in Spark
     • ⭐ Partition on a column with the RIGHT cardinality —
       partitioning by user_id creates millions of directories
       and is far worse than not partitioning at all
```

---

## 5. ETL vs ELT

```
   ETL — transform BEFORE loading
   Extract → ⭐ Transform (separate compute) → Load

   ✅ Only clean data lands in the warehouse
   ✅ Less warehouse storage
   ⚠️ Transformation logic lives outside SQL, harder to iterate
   ⚠️ ⭐ Reprocessing requires re-extracting from source

   ⭐ ELT — transform AFTER loading  (the modern default)
   Extract → Load raw → ⭐ Transform IN the warehouse

   ✅ ⭐ Raw data is preserved → reprocess anytime logic changes
   ✅ Transformations are SQL — version controlled, testable,
     accessible to analysts
   ✅ Warehouse compute is elastic and cheap
   ✅ Load is simple and fast, so ingestion rarely breaks
   ⚠️ More storage (⭐ but storage is the cheapest thing you buy)
   ⚠️ ⚠️ Raw sensitive data lands in the warehouse — needs
     access control and sometimes tokenization at ingest
```

```
   ⭐ WHY ELT WON

   Storage got cheap and warehouse compute got elastic. Once
   those two things were true, keeping raw data and
   transforming it repeatedly became cheaper than getting the
   transformation right the first time — which is impossible
   anyway.

   The decisive advantage is REPROCESSING. When you discover
   a bug in month six, ELT means re-running SQL. ETL means
   hoping the source system still has the data.
```

```sql
-- ⭐ dbt — the standard ELT transformation tool
-- models/marts/fct_orders.sql
{{ config(materialized='incremental', unique_key='order_id',
          partition_by={'field': 'order_date', 'data_type': 'date'}) }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.total_cents,
    c.segment
FROM {{ ref('stg_orders') }} o          -- ⭐ ref() builds the DAG
LEFT JOIN {{ ref('dim_customers') }} c USING (customer_id)

{% if is_incremental() %}
  -- ⭐ Only process new data on subsequent runs
  WHERE o.updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

```
   ⭐ WHAT dbt ACTUALLY GIVES YOU
     • A DAG inferred from ref() — no manual dependency wiring
     • ⭐ Tests as first-class citizens (not null, unique,
       accepted values, referential integrity)
     • Documentation and lineage generated from the code
     • Environments, so dev doesn't touch production tables
     • ⭐ Version-controlled, code-reviewed transformations —
       which is the actual cultural shift
```

---

## 6. Ingestion

```
   ┌──────────────────────────────────────────────────────────────┐
   │ FULL SNAPSHOT      Copy the entire table every run           │
   │   ✅ Simple, self-correcting                                  │
   │   ⚠️ Doesn't scale; heavy load on the source                  │
   ├──────────────────────────────────────────────────────────────┤
   │ INCREMENTAL        WHERE updated_at > last_watermark         │
   │   ✅ Efficient                                                │
   │   ⚠️⚠️ MISSES HARD DELETES · ⚠️ needs a reliable updated_at    │
   │     that is ALWAYS maintained · ⚠️ clock skew and             │
   │     long-running transactions can cause missed rows          │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ CDC              Read the database's replication log       │
   │   ✅ Catches EVERY change including DELETES                   │
   │   ✅ Near real-time, minimal source load                      │
   │   ⚠️ Operationally more complex                               │
   ├──────────────────────────────────────────────────────────────┤
   │ EVENT STREAM       The application emits events directly     │
   │   ✅ Captures INTENT, not just final state                    │
   │   ⚠️ Requires application changes and discipline              │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⚠️ THE WATERMARK PROBLEM — why incremental loads silently
     lose data

   Filtering on `updated_at > last_run` looks correct but:

   • ⭐ A transaction that STARTED before your watermark but
     COMMITTED after it has an updated_at in the past — you
     will never see it
   • Clock skew between servers shifts boundaries
   • Hard deletes leave no row to detect
   • A bulk update that forgets to touch updated_at is invisible

   ⭐ MITIGATIONS
     • Overlap the window (re-read the last N minutes) and
       deduplicate on the primary key
     • Use a monotonic sequence rather than a timestamp where
       available
     • ⭐ Periodic full reconciliation to catch drift
     • Or just use CDC, which has none of these problems
```

---

## 7. Change Data Capture

```
   ⭐ CDC READS THE DATABASE'S OWN REPLICATION LOG — the WAL in
     Postgres, the binlog in MySQL.

   ┌──────────────────────────────────────────────────────────────┐
   │  Source DB                                                   │
   │     │ writes to its WAL / binlog anyway                      │
   │     ▼                                                        │
   │  ⭐ CDC connector (Debezium) reads the log as a replica       │
   │     │                                                        │
   │     ▼                                                        │
   │  Kafka topic per table                                       │
   │     │  { before: {...}, after: {...}, op: "u", ts: ... }     │
   │     ▼                                                        │
   │  Sink → data lake / warehouse                                │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY IT'S SUPERIOR TO POLLING
     • Captures EVERY change, including deletes and rapid
       successive updates to the same row
     • ⭐ Near-zero load on the source — it's reading a log the
       database already writes
     • Near real-time
     • Ordered and complete by construction
     • ⭐ Captures the BEFORE image, which polling cannot
```

```
   ⚠️ CDC OPERATIONAL REALITIES

   • ⭐ INITIAL SNAPSHOT: you need the existing table contents
     plus the ongoing stream, stitched together consistently.
     Debezium handles this but it's the fragile part.
   • ⭐ SCHEMA CHANGES flow through and downstream consumers
     must tolerate them — use a schema registry with
     compatibility enforcement.
   • ⚠️ Replication slots RETAIN WAL until consumed. A stalled
     CDC connector fills the source database's disk and can
     take production down. ⭐ Monitor replication lag as a
     production alert, not a data alert.
   • Some DDL isn't captured; some column types serialize
     awkwardly.
```

```
   ⭐ CDC ALSO SOLVES THE DUAL-WRITE PROBLEM

   You cannot atomically write to a database AND publish to a
   queue. CDC reads the database's own log, so the event is
   derived from the committed transaction — there's nothing to
   get out of sync.

   ⭐ This is the production-grade version of the outbox pattern.
     See [Queues §10](../03-backend/queues-streaming.md#10-outbox-pattern).
```

---

## 8. Dimensional Modeling

```
   ⭐ STAR SCHEMA — the standard analytics model

                    ┌──────────────┐
                    │  dim_date    │
                    └──────┬───────┘
                           │
   ┌──────────────┐   ┌────▼─────────────┐   ┌──────────────┐
   │dim_customer  │───│   FACT_ORDERS    │───│  dim_product │
   └──────────────┘   │  ⭐ measures +    │   └──────────────┘
                      │  foreign keys    │
                      └────┬─────────────┘
                           │
                    ┌──────▼───────┐
                    │  dim_store   │
                    └──────────────┘

   FACTS       measurements/events. Many rows, numeric,
               append-mostly. (order_total, quantity)
   DIMENSIONS  ⭐ the context you filter and group BY.
               Fewer rows, descriptive, changes slowly.
               (customer name, product category, date attributes)
```

```
   ⭐ GRAIN IS THE FIRST DECISION, AND THE MOST IMPORTANT

   "What does ONE ROW of this fact table represent?"

     one order?  one order LINE?  one shipment?

   ⚠️ Getting this wrong makes every downstream aggregate
     subtly incorrect, and it's expensive to change later.
   ⭐ Declare it explicitly, write it in the model
     documentation, and test it with a uniqueness assertion.
```

```
   ⭐ SLOWLY CHANGING DIMENSIONS — how to handle attributes that
     change over time

   ┌──────────────────────────────────────────────────────────────┐
   │ TYPE 0   Never changes (a birth date)                        │
   │ TYPE 1   OVERWRITE. ⚠️ History is LOST — last year's reports  │
   │          silently change when someone moves house.           │
   │ ⭐TYPE 2  ADD A NEW ROW with validity dates.                  │
   │          ⭐ Preserves history — this is usually what you want │
   │          for anything used in historical reporting.          │
   │ TYPE 3   Add a "previous value" column (limited history)     │
   │ TYPE 4   Separate rapidly-changing attributes into a         │
   │          "mini-dimension"                                    │
   └──────────────────────────────────────────────────────────────┘
```

```sql
-- ⭐ TYPE 2 SCD structure
customer_key | customer_id | name | city      | valid_from | valid_to   | is_current
-------------|-------------|------|-----------|------------|------------|----------
     1       |    1001     | Ada  | London    | 2020-01-01 | 2024-06-01 | false
     2       |    1001     | Ada  | Berlin    | 2024-06-01 | 9999-12-31 | true
             ⭐ surrogate key differs; business key is the same

-- ⭐ Facts reference the SURROGATE key, so a historical order
--   correctly joins to the customer's city AT THAT TIME.
--   That's the entire point of Type 2.
```

```
   ⭐ FACT TABLE TYPES
     TRANSACTION    one row per event (most common)
     PERIODIC SNAPSHOT  one row per entity per period
                    (daily account balance)
     ACCUMULATING SNAPSHOT  one row per process instance,
                    updated as it progresses (order lifecycle
                    with milestone timestamps)
     FACTLESS       records that something HAPPENED with no
                    measure (student attended class)
```

---

## 9. Batch Processing & Spark

```
   ⭐ SPARK'S EXECUTION MODEL

   ┌──────────────────────────────────────────────────────────────┐
   │  DRIVER — builds the plan, schedules tasks                   │
   │     │                                                        │
   │     ├──▶ EXECUTOR (JVM)  ── task ── task ── task            │
   │     ├──▶ EXECUTOR        ── task ── task                     │
   │     └──▶ EXECUTOR        ── task ── task ── task            │
   └──────────────────────────────────────────────────────────────┘

   ⭐ TRANSFORMATIONS ARE LAZY. Nothing runs until an ACTION
     (count, collect, write) triggers execution — which lets
     the optimizer see the whole pipeline and rewrite it.

   NARROW  map, filter — each output partition depends on ONE
           input partition. ⭐ No data movement.
   ⭐ WIDE  groupBy, join, distinct — output depends on MANY
           input partitions → ⚠️ SHUFFLE (data crosses the
           network). This is where jobs get slow.
```

```
   ⭐⭐ THE SHUFFLE IS ALMOST ALWAYS THE BOTTLENECK

   A shuffle writes intermediate data to disk, transfers it
   across the network, and re-reads it. It's orders of
   magnitude more expensive than any local computation.

   ⭐ MINIMIZE SHUFFLES
     • Filter EARLY — before joins and aggregations
     • ⭐ BROADCAST JOIN: if one side fits in memory, send it to
       every executor and join locally — NO SHUFFLE AT ALL.
       broadcast(small_df) — the single biggest Spark win.
     • Pre-partition data by the join key so it's already
       co-located
     • Use reduceByKey rather than groupByKey (⭐ it aggregates
       locally before shuffling)
     • Cache results reused across multiple actions
```

```
   ⚠️⚠️ DATA SKEW — the classic Spark production failure

   One partition holds far more data than the others. 199 tasks
   finish in seconds; one runs for an hour or dies with OOM.

   ⭐ CAUSES: a null-heavy join key, a "celebrity" value, or a
     default value like 'UNKNOWN' that dominates.

   ⭐ FIXES
     • SALTING: append a random suffix to the hot key, join in
       two stages, then aggregate the results
     • Filter nulls out and handle them separately
     • Broadcast the small side to avoid the shuffle entirely
     • ⭐ Adaptive Query Execution (Spark 3+) detects and splits
       skewed partitions automatically — enable it
```

```python
# ⭐ Practical Spark essentials
df = spark.read.parquet("s3://bucket/events/")

df.filter(F.col("date") >= "2026-01-01")     # ⭐ filter FIRST
  .join(F.broadcast(small_df), "key")        # ⭐ broadcast the small side
  .groupBy("category").agg(F.sum("amount").alias("total"))
  .write.mode("overwrite")
  .partitionBy("date")                       # ⚠️ choose cardinality carefully
  .parquet("s3://bucket/output/")

# ⭐ Partition count rule of thumb: 2-4× the number of cores,
#   targeting ~128 MB per partition
df.repartition(200, "join_key")   # full shuffle, even distribution
df.coalesce(10)                   # ⭐ reduce partitions WITHOUT a shuffle
                                  #   (use before writing to avoid tiny files)
```

---

## 10. Stream Processing

```
   ⭐ THE HARD PART OF STREAMING IS TIME.

   EVENT TIME       when it actually HAPPENED  ⭐ correct
   PROCESSING TIME  when your system SAW it    (simpler, wrong
                                                under delay)

   ⚠️ A mobile client offline for two hours then syncing will
     put those events in the wrong bucket under processing time.
```

```
   ⭐ WATERMARKS — "I believe I have seen all events up to time T"

   ┌──────────────────────────────────────────────────────────────┐
   │  events arriving:  ●  ●   ●  ● ●    ●        ● (late!)       │
   │  event time:      10:00 10:02 10:01 10:05  09:58             │
   │                                                              │
   │  watermark = max_event_time − allowed_lateness               │
   │                                                              │
   │  ⭐ A window CLOSES when the watermark passes its end.        │
   │    Events arriving after that are LATE — drop them, send     │
   │    them to a side output, or update the emitted result.      │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE FUNDAMENTAL TRADEOFF
     Larger allowed lateness  → more correct, higher latency,
                                more state retained
     Smaller allowed lateness → faster results, more dropped
                                events

     There is no right answer — it's a product decision about
     how much correctness you'll trade for freshness.
```

```
   WINDOW TYPES
     TUMBLING   |──5m──|──5m──|──5m──|      fixed, non-overlapping
     HOPPING    |──5m──|                    overlapping
                   |──5m──|
     SLIDING    continuous, per-event
     ⭐ SESSION  |─active─| gap |─active─|   grouped by inactivity
```

```
   ┌──────────────────┬───────────────────────────────────────────┐
   │ Kafka Streams    │ A LIBRARY in your app — no cluster needed │
   │ ⭐ Flink          │ Best event-time and watermark semantics.  │
   │                  │ True streaming, strong state management.  │
   │ Spark Structured │ Micro-batch. ⭐ Unified batch + stream API │
   │ ksqlDB           │ SQL over Kafka — fastest to write         │
   └──────────────────┴───────────────────────────────────────────┘
```

```
   ⭐ THE LAMBDA / KAPPA DEBATE

   LAMBDA    Run a batch layer AND a speed layer, then merge.
             ⚠️ You maintain the SAME LOGIC TWICE, in two
               different systems, and they drift.

   ⭐ KAPPA   ONE streaming pipeline. To reprocess, replay the
             log from the beginning.
             ✅ One codebase, one set of semantics.
             ⭐ This is why replayable logs (Kafka) matter — they
               make batch reprocessing a special case of
               streaming rather than a separate system.
```

---

## 11. Orchestration

```
   ⭐ WHAT AN ORCHESTRATOR ACTUALLY PROVIDES
     • Dependency management (a DAG)
     • Scheduling, including ⭐ BACKFILL of historical periods
     • Retries with backoff
     • Alerting on failure
     • Observability of run history
     • ⭐ Idempotent, parameterized reruns
```

```python
# Airflow — the standard, though heavier than modern alternatives
@dag(schedule="0 2 * * *", start_date=datetime(2026, 1, 1),
     catchup=True,                      # ⭐ backfill missed intervals
     max_active_runs=1,                 # ⭐ prevents overlapping runs
     default_args={"retries": 3, "retry_delay": timedelta(minutes=5)})
def daily_pipeline():

    @task
    def extract(logical_date=None):
        # ⭐ Use the LOGICAL date, never datetime.now() —
        #   otherwise backfills process the wrong data
        return fetch_data(date=logical_date)

    @task
    def transform(raw): ...

    @task
    def load(clean): ...

    load(transform(extract()))
```

```
   ⭐⭐ IDEMPOTENCY IS THE CORE REQUIREMENT

   Every task must produce the same result when re-run for the
   same logical date. Without it, retries and backfills corrupt
   data instead of fixing it.

   ⭐ THE PATTERNS
     • ⭐ Partition by the logical date, and OVERWRITE that
       partition rather than appending
     • MERGE/upsert on a natural key rather than INSERT
     • Delete-then-insert for the target period, in one
       transaction
     • ⚠️ NEVER use now() or CURRENT_DATE inside a task —
       always the injected logical date. This single mistake
       makes backfills silently wrong.
```

```
   MODERN ALTERNATIVES
     Dagster      ⭐ Asset-oriented — you declare the DATA ASSETS
                  and their dependencies rather than tasks.
                  Better lineage and testing story.
     Prefect      Pythonic, dynamic DAGs
     Temporal     Durable execution for long-running workflows
     dbt Cloud    Transformation-only, but excellent at it
```

---

## 12. Data Quality

```
   ⭐ THE DIMENSIONS
     COMPLETENESS   are values missing?
     UNIQUENESS     duplicates on the key?
     VALIDITY       correct format, within range?
     ACCURACY       does it match reality?
     CONSISTENCY    do related datasets agree?
     TIMELINESS     is it fresh enough?
```

```yaml
# ⭐ dbt tests — quality as code, run on every build
models:
  - name: fct_orders
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns: [order_id]     # ⭐ asserts the GRAIN
    columns:
      - name: order_id
        tests: [unique, not_null]
      - name: customer_id
        tests:
          - not_null
          - relationships:                        # ⭐ referential integrity
              to: ref('dim_customers')
              field: customer_id
      - name: status
        tests:
          - accepted_values:
              values: [pending, paid, shipped, cancelled]
      - name: total_cents
        tests:
          - dbt_utils.accepted_range: { min_value: 0 }
```

```
   ⭐ FRESHNESS AND VOLUME CHECKS CATCH WHAT SCHEMA TESTS MISS

   • ⭐ FRESHNESS: has the table been updated recently?
     A silently-stopped pipeline passes every schema test while
     serving stale data — which is worse than failing loudly.
   • ⭐ VOLUME ANOMALY: is today's row count wildly different
     from the trailing average? A 90% drop means an upstream
     problem, not a business event.
   • DISTRIBUTION DRIFT: has a column's distribution shifted?
   • ⭐ RECONCILIATION: does the warehouse total match the source
     system total? This is the check that catches the bugs
     everything else misses.
```

```
   ⭐ FAIL LOUDLY, AND DECIDE WHETHER TO FAIL OPEN OR CLOSED

   ⚠️ The worst outcome is a pipeline that silently produces
     wrong data. Downstream consumers trust it, decisions get
     made, and the error is discovered weeks later.

   ⭐ For critical tables, BLOCK publication on test failure —
     serving yesterday's correct data beats serving today's
     wrong data.
```

---

## 13. Governance & Cost

```
   ⭐ GOVERNANCE ESSENTIALS

   □ ⭐ LINEAGE — which upstream tables feed this? Essential for
     impact analysis before a change, and for debugging.
   □ CATALOG — searchable metadata so people can find data
     without asking
   □ OWNERSHIP — every dataset has a named owner
   □ ⭐ ACCESS CONTROL — column-level and row-level for PII
   □ PII handling — classify, mask, tokenize; ⭐ know where
     personal data lives so GDPR deletion is possible
   □ Retention policies enforced automatically
   □ ⭐ CONTRACTS between producers and consumers, so a
     schema change doesn't silently break downstream
```

```
   ⭐ COST CONTROL — where warehouse money actually goes

   ┌──────────────────────────────────────────────────────────────┐
   │ ⭐ FULL TABLE SCANS                                           │
   │   → partition + cluster on the columns you filter by;        │
   │     the biggest single lever                                 │
   ├──────────────────────────────────────────────────────────────┤
   │ SELECT *                                                     │
   │   → ⭐ columnar storage charges by COLUMN scanned; selecting  │
   │     3 of 50 columns costs ~6% as much                        │
   ├──────────────────────────────────────────────────────────────┤
   │ REPEATED IDENTICAL QUERIES                                   │
   │   → materialize the result; result caching                   │
   ├──────────────────────────────────────────────────────────────┤
   │ FULL REFRESH OF LARGE TABLES                                 │
   │   → incremental models                                       │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ IDLE WAREHOUSES                                            │
   │   → auto-suspend aggressively                                │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ SMALL FILES                                                │
   │   → compaction; the planner overhead exceeds the read cost   │
   └──────────────────────────────────────────────────────────────┘

   ⭐ Attribute cost by team and by model, and make it visible.
     Unattributed cost is nobody's problem and grows forever.
```

---

## 14. Interview Section

<details>
<summary><b>Q1. ETL vs ELT — which and why?</b></summary>

ELT for anything modern. Extract, load the raw data, then transform inside the warehouse.

The decisive advantage is reprocessing. Transformation logic is always wrong eventually — you discover a bug, or a definition changes. With ELT you re-run SQL against data you still have. With ETL you're hoping the source system still holds the history, which for operational databases it usually doesn't.

The economics enabled it. Storage became very cheap and warehouse compute became elastic, so keeping raw data and transforming repeatedly costs less than getting it right the first time — which isn't possible anyway.

There are practical benefits too. Transformations are SQL, so they're version controlled, testable, code reviewed, and accessible to analysts rather than locked in a separate framework. And ingestion becomes simple and rarely breaks, because it does nothing but move bytes.

The real caveat is governance. Raw data lands in the warehouse including anything sensitive, so you need column-level access control and sometimes tokenization at ingest. That's a solvable problem, but it needs to be solved deliberately rather than discovered during an audit.
</details>

<details>
<summary><b>Q2. Why is columnar storage faster for analytics?</b></summary>

Three compounding effects.

Column pruning: analytical queries typically touch a few columns out of many. Row storage forces you to read entire rows off disk; columnar reads only the columns referenced, often a tenth or less of the bytes.

Compression: adjacent values within a column are similar, so dictionary encoding and run-length encoding work extremely well. Five to twenty times smaller than raw JSON is typical, and less data means less I/O.

Vectorized execution: a batch of column values can be processed per CPU instruction, rather than jumping around a row structure.

Parquet adds a fourth mechanism that's underappreciated — predicate pushdown. Each row group stores min and max per column, so a query filtering on a date can skip entire row groups without reading them at all.

The practical consequence is that you should sort data by the columns you filter on most, because that makes those statistics tight. Unsorted data defeats pushdown entirely.

The tradeoff is that writing or updating a single row means touching every column file, which is why columnar stores are for append-and-scan, not point updates. That's the OLTP/OLAP split in one sentence.
</details>

<details>
<summary><b>Q3. Explain CDC and when you'd use it.</b></summary>

Change data capture reads the database's own replication log — the write-ahead log in Postgres, the binlog in MySQL — and turns it into a stream of change events.

It's superior to polling for several reasons. It captures every change including deletes, which incremental extraction based on `updated_at` structurally cannot see. It catches rapid successive updates to the same row rather than just the final state. It puts near-zero load on the source, because it's reading a log the database writes anyway. And it includes the before-image, which polling can't provide.

The watermark problems with polling are worth naming: a transaction that started before your watermark but committed after it has a past timestamp and will never be seen. Clock skew shifts boundaries. A bulk update that forgets to touch `updated_at` is invisible.

CDC also solves the dual-write problem. You can't atomically write to a database and publish to a queue, but CDC derives events from the committed transaction, so there's nothing to get out of sync. It's the production-grade version of the outbox pattern.

The operational caveat that matters most: replication slots retain WAL until consumed, so a stalled CDC connector fills the source database's disk and can take production down. Replication lag needs to be a production alert, not a data-team alert.
</details>

<details>
<summary><b>Q4. What is a slowly changing dimension?</b></summary>

A dimension attribute that changes over time, where you have to decide whether history matters.

Type 1 overwrites the value. Simple, but history is lost — and the consequence is that last year's reports silently change when a customer moves house, because the historical order now joins to the new city.

Type 2 adds a new row with validity dates and an is-current flag. The business key stays the same but each version gets a distinct surrogate key, and facts reference the surrogate key. That means a historical order correctly joins to the customer's attributes as they were at the time. That's the entire point.

Type 3 keeps a previous-value column for limited history. Type 4 splits rapidly-changing attributes into a mini-dimension.

Type 2 is usually what you want for anything used in historical reporting, and Type 1 is fine for corrections — fixing a typo in a name shouldn't create a new version.

The related decision I'd raise is grain: what does one row of the fact table represent? Getting that wrong makes every downstream aggregate subtly incorrect and is expensive to change later, so I'd declare it explicitly and enforce it with a uniqueness test.
</details>

<details>
<summary><b>Q5. Your Spark job is slow. How do you debug it?</b></summary>

Start with the Spark UI and look at stage-level timing to find where the time goes.

The first question is whether there's a shuffle. Wide transformations — joins, groupBy, distinct — write intermediate data to disk, move it across the network, and re-read it. That's orders of magnitude more expensive than any local computation, so shuffles dominate almost every slow job.

The single biggest win is usually a broadcast join. If one side fits in memory, broadcasting it to every executor eliminates the shuffle entirely.

The second thing I'd check is skew — look at task duration distribution within a stage. The signature is 199 tasks finishing in seconds and one running for an hour or dying with an out-of-memory error. Causes are null-heavy join keys, celebrity values, or a default like 'UNKNOWN' that dominates. Fixes are salting the hot key, handling nulls separately, or broadcasting. Adaptive Query Execution in Spark 3 detects and splits skewed partitions automatically and should just be enabled.

Then partition sizing — targeting around 128 MB per partition, with roughly two to four times the core count. Too few partitions means poor parallelism; too many means scheduling overhead dominates.

And structural improvements: filter before joining, use `reduceByKey` over `groupByKey` since it aggregates locally before shuffling, cache anything reused across multiple actions, and coalesce before writing to avoid producing thousands of tiny files.
</details>

<details>
<summary><b>Q6. How do you ensure data quality?</b></summary>

Tests as code, running on every build, with failures blocking publication for critical tables.

The schema-level tests are the baseline: uniqueness on the key, which also asserts the grain; not-null on required columns; accepted values on enums; referential integrity to dimensions; and range checks on numerics. dbt makes these declarative and they run automatically.

But the checks that catch real incidents are different. Freshness — has this table been updated recently? A silently stopped pipeline passes every schema test while serving increasingly stale data, which is worse than failing loudly. Volume anomaly detection — is today's row count wildly different from the trailing average? A ninety percent drop is an upstream problem, not a business event. And reconciliation against the source system, which catches the bugs everything else misses.

On failure handling, I'd block publication for critical tables. Serving yesterday's correct data is better than serving today's wrong data, because downstream consumers trust it and decisions get made on it.

And I'd add data contracts between producers and consumers, so an upstream schema change fails at the boundary rather than silently corrupting a dashboard three layers downstream.
</details>

<details>
<summary><b>Q7. Batch or streaming — how do you decide?</b></summary>

Start from the latency requirement, and be sceptical of the stated one. "Real-time" frequently means "faster than the current daily batch," and hourly would satisfy the actual business need at a fraction of the complexity.

Streaming genuinely earns its cost when decisions are made on the data within seconds — fraud detection, personalization, operational alerting, dynamic pricing.

Batch is simpler in every way that matters operationally: easier to test, easier to reason about, trivially reprocessable, and failures are much less urgent. If a batch job fails at 2am you fix it at 9am and re-run.

The costs of streaming that people underestimate are correctness under out-of-order data — you need event-time processing with watermarks, and then you're making a product decision about how long to wait for late events versus how fresh results should be — plus state management, exactly-once semantics, and the fact that debugging a streaming job is substantially harder.

If I did need streaming, I'd argue for a Kappa-style architecture: one streaming pipeline, with reprocessing done by replaying the log from the beginning. The Lambda alternative means maintaining the same logic twice in two different systems, and they drift.
</details>

<details>
<summary><b>Q8. What makes a pipeline idempotent, and why does it matter?</b></summary>

Idempotent means re-running a task for the same logical period produces the same result. It matters because without it, retries and backfills corrupt data rather than fixing it — and retries are guaranteed to happen.

The patterns: partition output by the logical date and overwrite that partition rather than appending. Or merge on a natural key rather than inserting. Or delete-then-insert for the target period within one transaction.

The mistake that breaks it most often is using `now()` or `CURRENT_DATE` inside a task instead of the orchestrator's injected logical date. That makes the task's behaviour depend on when it runs rather than what period it's processing, so a backfill silently processes the wrong data — and it looks like it succeeded.

The related design point is keeping raw data immutable in a bronze layer. When transformation logic turns out to be wrong — which it will — you reprocess from raw rather than needing the source system to still have the history.

Together those two things mean a bug is a re-run rather than an incident.
</details>

---

## 15. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                   DATA ENGINEERING — ONE PAGE                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ SOURCES → INGEST → STORE → TRANSFORM → SERVE                         ║
║ ⭐ ASK: latency requirement? correctness guarantees? failure mode?    ║
║ MEDALLION: bronze(RAW, immutable) → silver(clean) → gold(business)   ║
║   ⭐ keep raw so you can REPROCESS — your logic WILL be wrong         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ COLUMNAR wins by: column pruning + far better compression +        ║
║   vectorization + ⭐ predicate pushdown via row-group min/max         ║
║   → SORT by your filter columns or pushdown does nothing             ║
║ Parquet for analytics · Avro for streaming/schema evolution          ║
║ ⚠️⚠️ SMALL FILES kill lakes → compact to 128MB-1GB; ⚠️ don't partition ║
║   on a high-cardinality column                                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ELT > ETL: raw preserved → REPROCESS anytime · SQL transforms are  ║
║   version-controlled and testable · storage is cheap                 ║
║ LAKEHOUSE = a transaction LOG over object storage → ACID +           ║
║   ⭐ time travel + efficient deletes (GDPR)                           ║
╠══════════════════════════════════════════════════════════════════════╣
║ INGEST: ⚠️ incremental on updated_at MISSES DELETES and rows from     ║
║   long transactions → ⭐ CDC reads the WAL/binlog: every change,      ║
║   deletes included, near-zero source load, solves dual-write         ║
║   ⚠️ CDC: a stalled connector RETAINS WAL and can fill the source disk║
╠══════════════════════════════════════════════════════════════════════╣
║ STAR SCHEMA: ⭐ DECLARE THE GRAIN FIRST ("one row = one what?")       ║
║   ⭐ SCD TYPE 2 = new row + validity dates → facts join to the        ║
║     attributes AS THEY WERE. Type 1 overwrites and LOSES history.    ║
╠══════════════════════════════════════════════════════════════════════╣
║ SPARK: lazy until an action · ⭐ THE SHUFFLE IS THE BOTTLENECK        ║
║   ⭐ broadcast join = no shuffle (biggest single win)                 ║
║   ⚠️ SKEW: 199 tasks fast, 1 forever → salt the key / enable AQE      ║
║   filter early · coalesce before writing · ~128MB per partition      ║
╠══════════════════════════════════════════════════════════════════════╣
║ STREAMING: ⭐ EVENT time not processing time · watermarks close       ║
║   windows · lateness = correctness vs freshness tradeoff             ║
║   ⭐ KAPPA (one pipeline, replay to reprocess) > Lambda (same logic   ║
║     maintained twice in two systems)                                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ IDEMPOTENCY: overwrite by partition · merge on key ·             ║
║   ⚠️ NEVER now() inside a task — use the LOGICAL date, or backfills   ║
║   silently process the wrong data                                    ║
║ QUALITY: schema tests + ⭐ FRESHNESS + ⭐ VOLUME ANOMALY +             ║
║   ⭐ RECONCILIATION vs source. Block publication on failure —         ║
║   stale-but-correct beats fresh-but-wrong.                           ║
║ COST: partition/cluster · never SELECT * · incremental models ·      ║
║   auto-suspend · ⭐ attribute cost by team or it grows forever        ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [ML & LLM Systems →](ml-llm-systems.md) · **Related:** [SQL](sql.md) · [Queues & Streaming](../03-backend/queues-streaming.md)
