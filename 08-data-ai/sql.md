# 🗃️ SQL Mastery

> SQL is declarative — you describe the result, the engine decides how. Most SQL difficulty comes from not knowing what the engine does with what you wrote.

**Prerequisite:** [Databases](../03-backend/databases.md)

---

## 📑 Contents

1. [How a Query Actually Executes](#1-how-a-query-actually-executes)
2. [Joins](#2-joins)
3. [NULL — the Thing That Breaks Logic](#3-null)
4. [Aggregation & GROUP BY](#4-aggregation--group-by)
5. [Window Functions](#5-window-functions)
6. [CTEs and Recursion](#6-ctes-and-recursion)
7. [Subqueries](#7-subqueries)
8. [Set Operations](#8-set-operations)
9. [Modifying Data](#9-modifying-data)
10. [Query Optimization](#10-query-optimization)
11. [Advanced Patterns](#11-advanced-patterns)
12. [60 Interview Problems](#12-interview-problems)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. How a Query Actually Executes

```
   ⭐ THE LOGICAL EXECUTION ORDER — not the order you write it

   You WRITE:              The engine EVALUATES:
   ─────────────           ─────────────────────
   SELECT      6           ① FROM / JOIN     build the working set
   FROM        1           ② WHERE           filter ROWS
   WHERE       2           ③ GROUP BY        form groups
   GROUP BY    3           ④ HAVING          filter GROUPS
   HAVING      4           ⑤ SELECT          compute expressions
   WINDOW      5           ⑥ WINDOW          apply window functions
   ORDER BY    7           ⑦ DISTINCT
   LIMIT       8           ⑧ ORDER BY        sort
                           ⑨ LIMIT / OFFSET
```

```
   ⭐ THIS ORDER EXPLAINS THREE THINGS THAT CONFUSE EVERYONE

   1. ⚠️ You CANNOT use a SELECT alias in WHERE.
      WHERE runs at step ②; the alias doesn't exist until ⑤.

      ❌ SELECT price * qty AS total FROM items WHERE total > 100
      ✅ SELECT price * qty AS total FROM items WHERE price*qty > 100
      ✅ Or wrap it in a subquery/CTE

   2. ⭐ You CAN use an alias in ORDER BY — it runs at step ⑧,
      after SELECT.

   3. ⚠️ WHERE filters ROWS (before grouping);
      HAVING filters GROUPS (after aggregation).
      ⭐ Aggregates like COUNT() cannot appear in WHERE at all,
        because no groups exist yet.
```

---

## 2. Joins

```
   ⭐ THE VISUAL MODEL

   A: {1,2,3}   B: {2,3,4}

   INNER JOIN          A ∩ B          → {2,3}
   ┌───┬───┐
   │ A │ B │           only matching rows on both sides
   └───┴───┘

   LEFT JOIN           A + matches    → {1,2,3}
   ┌───────┬───┐       ⭐ unmatched right side becomes NULL
   │   A   │ B │
   └───────┴───┘

   RIGHT JOIN          B + matches    → {2,3,4}

   FULL OUTER JOIN     A ∪ B          → {1,2,3,4}

   CROSS JOIN          A × B          ⚠️ every combination —
                                        3 × 3 = 9 rows
```

```sql
-- ⭐⭐ THE #1 JOIN MISTAKE: filtering a LEFT JOIN in WHERE

-- ❌ This silently becomes an INNER JOIN
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';
--    ⚠️ Users with NO orders get NULL for o.status,
--       and NULL = 'paid' is not true → those rows vanish.

-- ✅ Put the condition in the ON clause instead
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'paid';
--                                      ⭐ filters BEFORE the join

-- ✅ Or explicitly allow NULLs through
WHERE (o.status = 'paid' OR o.id IS NULL);
```

```
   ⭐ ON vs WHERE FOR OUTER JOINS — the mental model

   ON     decides WHICH ROWS MATCH. Non-matching left rows are
          still preserved with NULLs.
   WHERE  filters the RESULT of the join, and NULL fails almost
          every comparison → outer rows get eliminated.

   ⭐ For INNER joins the two are equivalent. For OUTER joins
     they are completely different, and this is where the bug is.
```

```sql
-- ⚠️ FAN-OUT: joins MULTIPLY rows, which silently corrupts sums
SELECT u.id, SUM(o.total)
FROM users u
JOIN orders o    ON o.user_id = u.id      -- 3 orders
JOIN order_items i ON i.order_id = o.id   -- ⚠️ 4 items each
GROUP BY u.id;
-- → each order's total is now counted 4 times

-- ✅ Aggregate FIRST, then join
SELECT u.id, o.total_sum
FROM users u
JOIN (SELECT user_id, SUM(total) AS total_sum
      FROM orders GROUP BY user_id) o ON o.user_id = u.id;

-- ✅ Or use a lateral/correlated aggregate
SELECT u.id,
       (SELECT SUM(total) FROM orders WHERE user_id = u.id) AS total_sum
FROM users u;
```

### Self joins and lateral joins

```sql
-- SELF JOIN — hierarchy
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- ⭐ LATERAL — the right side can REFERENCE the left side.
--   This is how you do "top N per group" cleanly.
SELECT u.name, recent.total, recent.created_at
FROM users u
CROSS JOIN LATERAL (
    SELECT total, created_at
    FROM orders o
    WHERE o.user_id = u.id           -- ⭐ references u
    ORDER BY created_at DESC
    LIMIT 3
) recent;
```

---

## 3. NULL

```
   ⚠️⚠️ NULL IS "UNKNOWN", NOT "EMPTY" OR "ZERO".
     It propagates through everything and breaks ordinary logic.

   NULL = NULL          →  ⚠️ NULL (not TRUE!)
   NULL <> NULL         →  NULL
   NULL + 5             →  NULL
   'abc' || NULL        →  NULL
   NOT NULL             →  NULL

   ⭐ ALWAYS USE:  IS NULL / IS NOT NULL / IS DISTINCT FROM
```

```
   ⭐ THREE-VALUED LOGIC

   AND    │ TRUE  │ FALSE │ NULL          OR    │ TRUE │ FALSE│ NULL
   ───────┼───────┼───────┼──────         ──────┼──────┼──────┼──────
   TRUE   │ TRUE  │ FALSE │ NULL          TRUE  │ TRUE │ TRUE │ TRUE
   FALSE  │ FALSE │ FALSE │ FALSE  ⭐      FALSE │ TRUE │ FALSE│ NULL
   NULL   │ NULL  │ FALSE │ NULL          NULL  │ TRUE │ NULL │ NULL
                     ▲                              ▲
              FALSE AND NULL = FALSE         TRUE OR NULL = TRUE
              (⭐ the only certainty)         (⭐ short-circuits)

   ⚠️ WHERE only keeps rows where the condition is TRUE.
     NULL is not TRUE, so those rows are dropped.
```

```sql
-- ⚠️⚠️ THE NOT IN TRAP — this returns ZERO ROWS if the subquery
--    contains even one NULL
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM orders);
--   If any user_id is NULL: id NOT IN (1, 2, NULL)
--   → id <> 1 AND id <> 2 AND id <> NULL
--   → ... AND NULL  →  never TRUE  →  NO ROWS ⚠️

-- ✅ Use NOT EXISTS — it is NULL-safe and usually faster
SELECT * FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

```sql
-- ⭐ NULL HANDLING TOOLKIT
COALESCE(a, b, c)          -- first non-NULL
NULLIF(a, b)               -- NULL if a = b (⭐ guards division by zero)
a IS DISTINCT FROM b       -- ⭐ NULL-safe inequality
COUNT(*)                   -- counts ALL rows
COUNT(col)                 -- ⭐ SKIPS NULLs — a very common source
                           --   of "why don't these numbers match?"
SUM/AVG/MAX                -- ignore NULLs
ORDER BY col NULLS LAST    -- explicit NULL ordering
```

---

## 4. Aggregation & GROUP BY

```sql
SELECT
    department,
    COUNT(*)                              AS headcount,
    COUNT(manager_id)                     AS has_manager,   -- ⭐ skips NULL
    COUNT(DISTINCT team_id)               AS teams,
    SUM(salary)                           AS payroll,
    AVG(salary)                           AS avg_salary,
    -- ⭐ CONDITIONAL AGGREGATION — pivoting without a PIVOT clause
    COUNT(*) FILTER (WHERE level = 'senior') AS seniors,
    SUM(CASE WHEN level = 'senior' THEN salary ELSE 0 END) AS senior_payroll,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary,
    STRING_AGG(name, ', ' ORDER BY name)  AS members,
    ARRAY_AGG(id)                         AS ids
FROM employees
WHERE active                              -- ② filters ROWS
GROUP BY department                       -- ③
HAVING COUNT(*) > 5                       -- ④ filters GROUPS
ORDER BY payroll DESC;
```

```
   ⭐ CONDITIONAL AGGREGATION IS THE MOST UNDERUSED SQL TECHNIQUE

   It turns rows into columns without a PIVOT clause:

   SELECT
       user_id,
       COUNT(*) FILTER (WHERE status = 'paid')      AS paid,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
       SUM(total) FILTER (WHERE created_at > now() - interval '30 days')
                                                    AS recent_revenue
   FROM orders GROUP BY user_id;

   ⭐ ONE table scan produces multiple metrics. The alternative —
     several separate queries joined together — is far slower.
   (FILTER is Postgres; CASE WHEN works everywhere.)
```

```sql
-- ⭐ GROUPING SETS — multiple aggregation levels in ONE pass
SELECT region, product, SUM(revenue)
FROM sales
GROUP BY GROUPING SETS ((region, product), (region), ());
--        ⭐ per region+product, per region, and a grand total

ROLLUP(a, b)   -- (a,b), (a), ()          hierarchical totals
CUBE(a, b)     -- (a,b), (a), (b), ()     every combination
```

---

## 5. Window Functions

```
   ⭐⭐ THE KEY DIFFERENCE FROM GROUP BY

   GROUP BY   COLLAPSES rows into one row per group
   WINDOW     KEEPS every row, and adds a computed value

   ┌──────────────────────────────────────────────────────────────┐
   │  GROUP BY                    WINDOW                          │
   │  dept │ avg                  name │ dept │ salary │ dept_avg │
   │  ─────┼─────                 ─────┼──────┼────────┼───────── │
   │   eng │ 100                   Ann │  eng │    120 │      100 │
   │  sales│  80                   Bob │  eng │     80 │      100 │
   │                               Cid │ sales│     80 │       80 │
   │  ⚠️ lost the rows            ⭐ every row PLUS the aggregate  │
   └──────────────────────────────────────────────────────────────┘
```

```sql
function() OVER (
    PARTITION BY col      -- ⭐ like GROUP BY, but doesn't collapse
    ORDER BY col          -- ordering within the partition
    ROWS BETWEEN ... AND ...   -- the frame
)
```

```sql
-- ⭐ RANKING — know the difference cold
SELECT name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn,    -- 1,2,3,4 always unique
    RANK()       OVER (ORDER BY salary DESC) AS rnk,   -- 1,2,2,4 ⭐ gaps
    DENSE_RANK() OVER (ORDER BY salary DESC) AS drnk,  -- 1,2,2,3 ⭐ no gaps
    NTILE(4)     OVER (ORDER BY salary DESC) AS quartile,
    PERCENT_RANK() OVER (ORDER BY salary)    AS pct
FROM employees;
```

```sql
-- ⭐ OFFSET FUNCTIONS — comparing to other rows
SELECT
    day, revenue,
    LAG(revenue, 1)  OVER (ORDER BY day) AS prev_day,
    LEAD(revenue, 1) OVER (ORDER BY day) AS next_day,
    revenue - LAG(revenue) OVER (ORDER BY day) AS day_over_day,
    -- ⭐ percentage change, guarding division by zero
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY day))
          / NULLIF(LAG(revenue) OVER (ORDER BY day), 0), 2) AS pct_change,
    FIRST_VALUE(revenue) OVER (ORDER BY day) AS first_day,
    -- ⚠️ LAST_VALUE needs an explicit frame or it returns the CURRENT row
    LAST_VALUE(revenue) OVER (
        ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_day
FROM daily_revenue;
```

```
   ⭐⭐ THE DEFAULT FRAME IS THE MOST COMMON WINDOW BUG

   With ORDER BY present, the default frame is:
     RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
   → a RUNNING TOTAL, not a partition total.

   Without ORDER BY, the default is the whole partition.

   ┌──────────────────────────────────────────────────────────────┐
   │ SUM(x) OVER (PARTITION BY g)                                 │
   │   → the TOTAL for the group (no ORDER BY)                    │
   │                                                              │
   │ SUM(x) OVER (PARTITION BY g ORDER BY d)                      │
   │   → ⚠️ a RUNNING total (frame defaults to current row)        │
   │                                                              │
   │ ⭐ SUM(x) OVER (PARTITION BY g ORDER BY d                     │
   │        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)│
   │   → the group total, WITH ordering available                 │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ ALSO: RANGE treats ties as one unit; ROWS counts physical
     rows. With duplicate ORDER BY values they give different
     answers. ⭐ Prefer ROWS unless you specifically want RANGE.
```

```sql
-- ⭐ MOVING AVERAGE — the canonical frame example
SELECT day, revenue,
    AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
        AS moving_avg_7d,
    SUM(revenue) OVER (ORDER BY day)                    -- running total
        AS cumulative
FROM daily_revenue;
```

---

## 6. CTEs and Recursion

```sql
-- ⭐ CTEs make complex queries READABLE — chain them
WITH monthly AS (
    SELECT date_trunc('month', created_at) AS month,
           user_id, SUM(total) AS revenue
    FROM orders WHERE status = 'paid'
    GROUP BY 1, 2
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY month ORDER BY revenue DESC) AS rnk
    FROM monthly
)
SELECT month, user_id, revenue FROM ranked WHERE rnk <= 10;
```

```
   ⚠️ CTE PERFORMANCE — a real gotcha
     In PostgreSQL BEFORE version 12, a CTE was an OPTIMIZATION
     FENCE — always materialized, never inlined into the outer
     query, which could be dramatically slower.

     ⭐ Postgres 12+ inlines CTEs used once. You can force either
       behaviour explicitly:
         WITH x AS MATERIALIZED (...)
         WITH x AS NOT MATERIALIZED (...)

     ⭐ MATERIALIZED is genuinely useful when a CTE is expensive
       and referenced multiple times.
```

```sql
-- ⭐ RECURSIVE CTE — hierarchies, graphs, sequences
WITH RECURSIVE subordinates AS (
    -- ① ANCHOR: the starting rows
    SELECT id, name, manager_id, 1 AS depth, ARRAY[id] AS path
    FROM employees WHERE id = 1

    UNION ALL

    -- ② RECURSIVE: joins against the CTE itself
    SELECT e.id, e.name, e.manager_id, s.depth + 1, s.path || e.id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
    WHERE s.depth < 10                 -- ⭐ ALWAYS bound the depth
      AND NOT e.id = ANY(s.path)       -- ⭐ cycle protection
)
SELECT * FROM subordinates;
```

```
   ⭐ THE TWO SAFETY RULES FOR RECURSIVE CTEs
     1. Bound the depth — otherwise a malformed hierarchy loops
        until the server runs out of memory
     2. Track the path and exclude already-visited nodes —
        ⚠️ a cycle in the data (A manages B manages A) is an
        infinite loop otherwise

   ⚠️ UNION ALL vs UNION: UNION deduplicates on every iteration,
     which is much slower. Use UNION ALL plus explicit cycle
     detection.
```

```sql
-- ⭐ GENERATING SERIES — useful for filling gaps in time series
SELECT d::date AS day, COALESCE(SUM(o.total), 0) AS revenue
FROM generate_series('2026-01-01'::date, '2026-01-31', '1 day') d
LEFT JOIN orders o ON o.created_at::date = d::date
GROUP BY d ORDER BY d;
-- ⭐ Without the generated series, days with zero orders would be
--   MISSING rather than showing zero — a classic reporting bug.
```

---

## 7. Subqueries

```sql
-- SCALAR — returns one value
SELECT name, (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS n
FROM users u;

-- IN / NOT IN — ⚠️ NOT IN breaks with NULLs (see §3)
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders);

-- ⭐ EXISTS — usually the best choice
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

```
   ⭐ EXISTS vs IN vs JOIN — how to choose

   EXISTS   ⭐ Short-circuits on the first match. NULL-safe.
            Best when you only need to know IF a match exists.
   IN       Fine for a small static list. ⚠️ NOT IN is dangerous
            with NULLs.
   JOIN     Use when you need COLUMNS from the other table.
            ⚠️ Can multiply rows if the join isn't unique.

   ⭐ Modern optimizers often rewrite these into the same plan,
     so choose for CLARITY and NULL-safety first. EXISTS is the
     safe default.
```

---

## 8. Set Operations

```sql
SELECT id FROM a UNION     SELECT id FROM b;  -- ⚠️ dedupes (sorts — slower)
SELECT id FROM a UNION ALL SELECT id FROM b;  -- ⭐ keeps duplicates, FASTER
SELECT id FROM a INTERSECT SELECT id FROM b;  -- in both
SELECT id FROM a EXCEPT    SELECT id FROM b;  -- in a, not in b

-- ⭐ Rules: same number of columns, compatible types, and the
--   column names come from the FIRST query.
```

```
   ⭐ USE UNION ALL UNLESS YOU GENUINELY NEED DEDUPLICATION.
     UNION performs a sort or hash to remove duplicates, which
     on large result sets is a substantial and usually
     unnecessary cost.

   ⭐ EXCEPT is excellent for RECONCILIATION — finding rows in
     one dataset missing from another, in both directions:
       (A EXCEPT B) UNION ALL (B EXCEPT A)
```

---

## 9. Modifying Data

```sql
-- ⭐ UPSERT — insert or update atomically
INSERT INTO users (email, name, updated_at)
VALUES ('a@b.com', 'Ada', now())
ON CONFLICT (email) DO UPDATE
SET name = EXCLUDED.name,          -- ⭐ EXCLUDED = the proposed row
    updated_at = EXCLUDED.updated_at
WHERE users.name IS DISTINCT FROM EXCLUDED.name;   -- ⭐ skip no-op writes

-- ON CONFLICT DO NOTHING — idempotent insert
INSERT INTO events (id, payload) VALUES (...) ON CONFLICT (id) DO NOTHING;

-- ⭐ UPDATE ... FROM — update using another table
UPDATE orders o
SET status = 'shipped', shipped_at = now()
FROM shipments s
WHERE s.order_id = o.id AND s.status = 'dispatched';

-- ⭐ RETURNING — get the affected rows back in ONE round trip
INSERT INTO orders (...) VALUES (...) RETURNING id, created_at;
DELETE FROM sessions WHERE expires_at < now() RETURNING id;

-- ⭐ THE JOB QUEUE PATTERN — safe with many concurrent workers
UPDATE jobs SET status = 'processing', worker_id = $1
WHERE id IN (
    SELECT id FROM jobs WHERE status = 'pending'
    ORDER BY created_at LIMIT 10
    FOR UPDATE SKIP LOCKED          -- ⭐ each worker gets DIFFERENT rows
) RETURNING *;
```

```
   ⚠️ BATCHED UPDATES — never one giant UPDATE on a large table

   A single UPDATE touching millions of rows holds locks for
   the entire duration, bloats the WAL, and blows out replica
   lag.

   ⭐ Loop in batches with a short sleep between them:
     UPDATE t SET col = ... WHERE id IN (
         SELECT id FROM t WHERE col IS NULL LIMIT 10000
     );
     -- commit, sleep, repeat until 0 rows affected
```

---

## 10. Query Optimization

```
   ⭐ THE DIAGNOSTIC ORDER

   1. Find the slow query      pg_stat_statements ordered by
                               TOTAL time (⭐ not mean — a 5ms
                               query run a million times matters
                               more than a 2s report run daily)
   2. EXPLAIN (ANALYZE, BUFFERS)
   3. Compare ESTIMATED vs ACTUAL rows — a 10×+ gap means stale
      statistics → run ANALYZE
   4. Find where the time actually goes
   5. Fix, then ⭐ RE-MEASURE
```

```
   ⭐ READING A PLAN

   • Read INSIDE OUT — the most indented node runs first
   • `rows=` estimated vs actual — a large gap is the root cause
     of most bad plans
   • `loops=N` → multiply the time; ⭐ a high loop count is your
     N+1
   • Buffers: `read` = disk I/O (bad), `hit` = cache (good)

   ⚠️ WARNING SIGNS
     Seq Scan on a large table with a selective filter → missing index
     Sort Method: external merge Disk: 250MB           → raise work_mem
     Rows Removed by Filter: 990000                    → index isn't selective
     Nested Loop with a huge outer count               → bad estimate
```

```sql
-- ⚠️ THINGS THAT PREVENT INDEX USE
WHERE LOWER(email) = 'a@b.com'        -- ❌ function on the column
   -- ✅ CREATE INDEX ON users (LOWER(email));

WHERE DATE(created_at) = '2026-01-01' -- ❌
   -- ✅ WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02'

WHERE name LIKE '%smith'              -- ❌ leading wildcard
   -- ✅ pg_trgm GIN index enables this

WHERE varchar_col = 123               -- ❌ implicit cast on the COLUMN
WHERE a = 1 OR b = 2                  -- ❌ often; ✅ UNION of two queries
```

```
   ⭐ THE LEFTMOST PREFIX RULE

   CREATE INDEX idx ON orders (customer_id, status, created_at);

   ✅ WHERE customer_id = 5
   ✅ WHERE customer_id = 5 AND status = 'paid'
   ✅ WHERE customer_id = 5 AND status = 'paid' ORDER BY created_at
   ⚠️ WHERE customer_id = 5 AND created_at > '...'   ← skips status,
                                                       partial use
   ❌ WHERE status = 'paid'                          ← no leftmost column

   ⭐ COLUMN ORDER RULE: equality columns FIRST, range column LAST.
     A range predicate stops subsequent columns from being used
     for seeking.
```

```sql
-- ⭐ COVERING INDEX → index-only scan, no table access at all
CREATE INDEX idx ON orders (customer_id, status) INCLUDE (total, created_at);

-- ⭐ PARTIAL INDEX — index only the rows you actually query
CREATE INDEX idx_pending ON orders (created_at) WHERE status = 'pending';
-- If 99% of orders are completed, this index is ~100× smaller.
```

```sql
-- ⭐ CURSOR (KEYSET) PAGINATION — O(1) instead of O(offset)
-- ❌ OFFSET 100000 scans and discards 100,000 rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;

-- ✅ Row-value comparison against the last row seen
SELECT * FROM orders
WHERE (created_at, id) < ($last_created, $last_id)   -- ⭐ tiebreaker!
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- ⚠️ The unique tiebreaker is essential — with duplicate
--   timestamps, sorting on created_at alone silently skips or
--   repeats rows at page boundaries.
```

---

## 11. Advanced Patterns

```sql
-- ⭐ GAPS AND ISLANDS — consecutive runs
-- The trick: (row_number - value) is CONSTANT within a run
WITH marked AS (
    SELECT user_id, activity_date,
           activity_date - (ROW_NUMBER() OVER (
               PARTITION BY user_id ORDER BY activity_date))::int AS grp
    FROM activity
)
SELECT user_id, MIN(activity_date) AS streak_start,
       MAX(activity_date) AS streak_end, COUNT(*) AS streak_length
FROM marked GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

```sql
-- ⭐ RUNNING/CUMULATIVE with a reset condition
SELECT *, SUM(amount) OVER (PARTITION BY account_id ORDER BY txn_date
                            ROWS UNBOUNDED PRECEDING) AS balance
FROM transactions;

-- ⭐ TOP N PER GROUP — three ways
-- 1. Window function (clearest)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn
    FROM employees
) t WHERE rn <= 3;

-- 2. LATERAL (often faster with a good index)
SELECT d.name, e.*
FROM departments d
CROSS JOIN LATERAL (
    SELECT * FROM employees WHERE dept_id = d.id
    ORDER BY salary DESC LIMIT 3
) e;

-- 3. DISTINCT ON — ⭐ Postgres-specific, ideal for top-1
SELECT DISTINCT ON (dept_id) *
FROM employees ORDER BY dept_id, salary DESC;
```

```sql
-- ⭐ PIVOT via conditional aggregation
SELECT product,
    SUM(revenue) FILTER (WHERE quarter = 'Q1') AS q1,
    SUM(revenue) FILTER (WHERE quarter = 'Q2') AS q2,
    SUM(revenue) FILTER (WHERE quarter = 'Q3') AS q3
FROM sales GROUP BY product;

-- ⭐ UNPIVOT via a lateral VALUES list
SELECT product, quarter, revenue
FROM sales_wide, LATERAL (VALUES
    ('Q1', q1), ('Q2', q2), ('Q3', q3)
) AS v(quarter, revenue);
```

```sql
-- ⭐ COHORT RETENTION — a classic analytics interview question
WITH cohorts AS (
    SELECT user_id, date_trunc('month', MIN(created_at)) AS cohort_month
    FROM orders GROUP BY user_id
),
activity AS (
    SELECT c.cohort_month, c.user_id,
           date_trunc('month', o.created_at) AS active_month
    FROM cohorts c JOIN orders o ON o.user_id = c.user_id
)
SELECT cohort_month,
       -- ⭐ months since acquisition
       (EXTRACT(YEAR FROM active_month) - EXTRACT(YEAR FROM cohort_month)) * 12
       + (EXTRACT(MONTH FROM active_month) - EXTRACT(MONTH FROM cohort_month))
           AS month_number,
       COUNT(DISTINCT user_id) AS active_users
FROM activity
GROUP BY 1, 2 ORDER BY 1, 2;
```

---

## 12. Interview Problems

### A. Fundamentals (1–12)

**1. Second highest salary** 🟢
```sql
-- ⭐ OFFSET handles the "no second value" case by returning NULL
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Or, generalized to Nth:
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC OFFSET 1 LIMIT 1;
```

**2. Nth highest salary** 🟡
```sql
SELECT DISTINCT salary FROM employees
ORDER BY salary DESC OFFSET (N-1) LIMIT 1;
```

**3. Duplicate emails** 🟢
```sql
SELECT email FROM person GROUP BY email HAVING COUNT(*) > 1;
```

**4. Delete duplicates, keep the lowest id** 🟡
```sql
DELETE FROM person p
WHERE EXISTS (SELECT 1 FROM person q
              WHERE q.email = p.email AND q.id < p.id);
```

**5. Customers who never ordered** 🟢
```sql
-- ⭐ NOT EXISTS, not NOT IN (NULL-safe)
SELECT c.name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**6. Employees earning more than their manager** 🟢
```sql
SELECT e.name FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

**7. Department highest salary** 🟡
```sql
SELECT d.name AS dept, e.name, e.salary
FROM (SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) rnk
      FROM employees) e
JOIN departments d ON d.id = e.dept_id
WHERE e.rnk = 1;   -- ⭐ RANK includes ties
```

**8. Rising temperature (compare to previous day)** 🟢
```sql
SELECT id FROM (
    SELECT id, temperature, recorded_on,
           LAG(temperature) OVER (ORDER BY recorded_on) AS prev_t,
           LAG(recorded_on)  OVER (ORDER BY recorded_on) AS prev_d
    FROM weather
) t
WHERE temperature > prev_t AND recorded_on = prev_d + 1;
--                            ⭐ must verify the days are CONSECUTIVE
```

**9. Consecutive numbers appearing 3+ times** 🟡
```sql
SELECT DISTINCT num FROM (
    SELECT num,
        LAG(num)  OVER (ORDER BY id) AS p,
        LEAD(num) OVER (ORDER BY id) AS n
    FROM logs
) t WHERE num = p AND num = n;
```

**10. Exchange seats (swap adjacent)** 🟡
```sql
SELECT
    CASE WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM seat) THEN id
         WHEN id % 2 = 1 THEN id + 1
         ELSE id - 1 END AS id,
    student
FROM seat ORDER BY id;
```

**11. Trips and users (cancellation rate)** 🔴
```sql
SELECT request_at AS day,
    ROUND(COUNT(*) FILTER (WHERE status <> 'completed')::numeric
          / NULLIF(COUNT(*), 0), 2) AS cancellation_rate
FROM trips t
JOIN users c ON c.id = t.client_id AND NOT c.banned
JOIN users d ON d.id = t.driver_id AND NOT d.banned
WHERE request_at BETWEEN '2013-10-01' AND '2013-10-03'
GROUP BY request_at;
```

**12. Human traffic of stadium (3+ consecutive days ≥100)** 🔴
```sql
-- ⭐ Gaps-and-islands: (id - row_number) is constant within a run
WITH valid AS (
    SELECT *, id - ROW_NUMBER() OVER (ORDER BY id) AS grp
    FROM stadium WHERE people >= 100
)
SELECT id, visit_date, people FROM valid
WHERE grp IN (SELECT grp FROM valid GROUP BY grp HAVING COUNT(*) >= 3)
ORDER BY visit_date;
```

### B. Joins & Relationships (13–24)

**13. Users with no activity in 30 days** 🟢
```sql
SELECT u.* FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM events e
    WHERE e.user_id = u.id AND e.created_at > now() - interval '30 days');
```

**14. Products never sold** 🟢
```sql
SELECT p.* FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
WHERE oi.id IS NULL;   -- ⭐ the anti-join idiom
```

**15. Mutual friends** 🟡
```sql
SELECT f1.user_id, f2.user_id, COUNT(*) AS mutual
FROM friends f1 JOIN friends f2 ON f1.friend_id = f2.friend_id
WHERE f1.user_id < f2.user_id     -- ⭐ avoids duplicate pairs and self-pairs
GROUP BY 1, 2;
```

**16. Managers with at least 5 direct reports** 🟢
```sql
SELECT m.name FROM employees e
JOIN employees m ON e.manager_id = m.id
GROUP BY m.id, m.name HAVING COUNT(*) >= 5;
```

**17. Orders with all items shipped** 🟡
```sql
SELECT order_id FROM order_items
GROUP BY order_id
HAVING COUNT(*) = COUNT(*) FILTER (WHERE shipped);   -- ⭐ all rows match
```

**18. Customers who bought A but not B** 🟡
```sql
SELECT DISTINCT o.customer_id
FROM orders o JOIN order_items i ON i.order_id = o.id
WHERE i.product_id = 'A'
  AND NOT EXISTS (
      SELECT 1 FROM orders o2 JOIN order_items i2 ON i2.order_id = o2.id
      WHERE o2.customer_id = o.customer_id AND i2.product_id = 'B');
```

**19. Self-referencing hierarchy depth** 🟡
```sql
WITH RECURSIVE tree AS (
    SELECT id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
    WHERE t.depth < 20
)
SELECT * FROM tree;
```

**20. Latest order per customer** 🟡
```sql
SELECT DISTINCT ON (customer_id) *      -- ⭐ Postgres
FROM orders ORDER BY customer_id, created_at DESC;
```

**21. Overlapping date ranges** 🔴
```sql
SELECT a.id, b.id FROM bookings a
JOIN bookings b ON a.room_id = b.room_id AND a.id < b.id
WHERE a.start_date < b.end_date AND b.start_date < a.end_date;
--    ⭐ the standard overlap test: a.start < b.end AND b.start < a.end
```

**22. Employees in the same department as X** 🟢
```sql
SELECT * FROM employees
WHERE dept_id = (SELECT dept_id FROM employees WHERE name = 'X')
  AND name <> 'X';
```

**23. Fan-out safe multi-table aggregation** 🔴
```sql
-- ⭐ Aggregate each side SEPARATELY to avoid row multiplication
SELECT u.id, u.name,
       COALESCE(o.order_count, 0)  AS orders,
       COALESCE(r.review_count, 0) AS reviews
FROM users u
LEFT JOIN (SELECT user_id, COUNT(*) order_count FROM orders GROUP BY 1) o
       ON o.user_id = u.id
LEFT JOIN (SELECT user_id, COUNT(*) review_count FROM reviews GROUP BY 1) r
       ON r.user_id = u.id;
```

**24. Find missing sequence numbers** 🟡
```sql
SELECT gs AS missing
FROM generate_series(1, (SELECT MAX(id) FROM t)) gs
LEFT JOIN t ON t.id = gs
WHERE t.id IS NULL;
```

### C. Window Functions (25–38)

**25. Running total** 🟢
```sql
SELECT *, SUM(amount) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS running
FROM transactions;
```

**26. Moving 7-day average** 🟡
```sql
SELECT day, revenue,
    AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
FROM daily;
```

**27. Rank within group** 🟢
```sql
SELECT *, DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) FROM emp;
```

**28. Percentage of group total** 🟡
```sql
SELECT dept, name, salary,
    ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY dept), 2) AS pct
FROM employees;
-- ⭐ No ORDER BY in the window → the whole partition total
```

**29. Month-over-month growth** 🟡
```sql
SELECT month, revenue,
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
          / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) AS mom_pct
FROM monthly_revenue;
```

**30. First and last order per customer** 🟡
```sql
SELECT DISTINCT customer_id,
    FIRST_VALUE(id) OVER w AS first_order,
    LAST_VALUE(id)  OVER w AS last_order
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY created_at
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
-- ⭐ The explicit frame is REQUIRED for LAST_VALUE
```

**31. Median per group** 🟡
```sql
SELECT dept, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median
FROM employees GROUP BY dept;
```

**32. Deduplicate keeping the newest** 🟡
```sql
DELETE FROM t WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) rn
        FROM t
    ) x WHERE rn > 1);
```

**33. Sessionize events (30-min gap)** 🔴
```sql
-- ⭐ Start a new session when the gap exceeds the threshold,
--   then cumulative-sum the flags into session IDs
WITH gaps AS (
    SELECT *, CASE WHEN created_at - LAG(created_at) OVER w > interval '30 min'
                        OR LAG(created_at) OVER w IS NULL
                   THEN 1 ELSE 0 END AS is_new
    FROM events WINDOW w AS (PARTITION BY user_id ORDER BY created_at)
)
SELECT *, SUM(is_new) OVER (PARTITION BY user_id ORDER BY created_at) AS session_id
FROM gaps;
```

**34. Longest streak of consecutive days** 🔴
```sql
WITH marked AS (
    SELECT user_id, day,
           day - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY day))::int AS grp
    FROM (SELECT DISTINCT user_id, activity_date::date AS day FROM activity) d
)
SELECT user_id, MAX(streak) FROM (
    SELECT user_id, grp, COUNT(*) AS streak FROM marked GROUP BY 1, 2
) s GROUP BY user_id;
```

**35. Top 3 per category** 🟡
```sql
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) rn
    FROM products
) t WHERE rn <= 3;
```

**36. Cumulative distinct count** 🔴
```sql
-- ⭐ COUNT(DISTINCT) isn't allowed as a window function —
--   flag first-occurrences, then cumulative-sum
WITH firsts AS (
    SELECT day, user_id,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY day) AS rn
    FROM events
)
SELECT day, SUM(COUNT(*) FILTER (WHERE rn = 1)) OVER (ORDER BY day) AS cum_users
FROM firsts GROUP BY day ORDER BY day;
```

**37. Compare each row to the group average** 🟡
```sql
SELECT *, salary - AVG(salary) OVER (PARTITION BY dept) AS diff_from_avg
FROM employees;
```

**38. Islands of consecutive identical values** 🔴
```sql
SELECT MIN(day) AS start_day, MAX(day) AS end_day, status, COUNT(*) AS len
FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY day)
            - ROW_NUMBER() OVER (PARTITION BY status ORDER BY day) AS grp
    FROM daily_status
) t GROUP BY grp, status ORDER BY start_day;
```

### D. Analytics & Business (39–52)

**39. DAU / MAU ratio (stickiness)** 🟡
```sql
SELECT d.day,
       d.dau,
       m.mau,
       ROUND(100.0 * d.dau / NULLIF(m.mau, 0), 2) AS stickiness_pct
FROM (SELECT day, COUNT(DISTINCT user_id) dau FROM events GROUP BY 1) d
JOIN LATERAL (
    SELECT COUNT(DISTINCT user_id) mau FROM events
    WHERE day > d.day - 30 AND day <= d.day
) m ON true;
```

**40. Cohort retention** 🔴 — see [§11](#11-advanced-patterns)

**41. Funnel conversion** 🔴
```sql
SELECT
    COUNT(DISTINCT user_id) FILTER (WHERE step = 'view')     AS viewed,
    COUNT(DISTINCT user_id) FILTER (WHERE step = 'cart')     AS carted,
    COUNT(DISTINCT user_id) FILTER (WHERE step = 'checkout') AS checked_out,
    COUNT(DISTINCT user_id) FILTER (WHERE step = 'purchase') AS purchased,
    ROUND(100.0 * COUNT(DISTINCT user_id) FILTER (WHERE step = 'purchase')
          / NULLIF(COUNT(DISTINCT user_id) FILTER (WHERE step = 'view'), 0), 2)
        AS conversion_pct
FROM funnel_events;
```

**42. Revenue by month with growth** 🟡
```sql
SELECT month, revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
    SUM(revenue) OVER (ORDER BY month) AS cumulative
FROM (SELECT date_trunc('month', created_at) month, SUM(total) revenue
      FROM orders WHERE status = 'paid' GROUP BY 1) m;
```

**43. Customer lifetime value** 🟡
```sql
SELECT customer_id,
    COUNT(*) AS order_count,
    SUM(total) AS lifetime_value,
    AVG(total) AS avg_order_value,
    MIN(created_at) AS first_order,
    MAX(created_at) AS last_order,
    MAX(created_at) - MIN(created_at) AS customer_lifespan
FROM orders WHERE status = 'paid' GROUP BY customer_id;
```

**44. RFM segmentation** 🔴
```sql
WITH rfm AS (
    SELECT customer_id,
        NTILE(5) OVER (ORDER BY MAX(created_at))  AS recency,
        NTILE(5) OVER (ORDER BY COUNT(*))         AS frequency,
        NTILE(5) OVER (ORDER BY SUM(total))       AS monetary
    FROM orders GROUP BY customer_id
)
SELECT *, CASE WHEN recency >= 4 AND frequency >= 4 THEN 'champion'
               WHEN recency >= 4 THEN 'recent'
               WHEN frequency >= 4 THEN 'loyal'
               WHEN recency <= 2 AND frequency <= 2 THEN 'at_risk'
               ELSE 'other' END AS segment
FROM rfm;
```

**45. Year-over-year comparison** 🟡
```sql
SELECT month_of_year, revenue_2026, revenue_2025,
       ROUND(100.0*(revenue_2026-revenue_2025)/NULLIF(revenue_2025,0),2) AS yoy_pct
FROM (
    SELECT EXTRACT(MONTH FROM created_at) AS month_of_year,
        SUM(total) FILTER (WHERE EXTRACT(YEAR FROM created_at)=2026) AS revenue_2026,
        SUM(total) FILTER (WHERE EXTRACT(YEAR FROM created_at)=2025) AS revenue_2025
    FROM orders GROUP BY 1
) t ORDER BY month_of_year;
```

**46. Churn rate by month** 🔴
```sql
WITH monthly_active AS (
    SELECT DISTINCT user_id, date_trunc('month', created_at) AS month
    FROM events
)
SELECT a.month,
    COUNT(*) AS active_last_month,
    COUNT(*) FILTER (WHERE b.user_id IS NULL) AS churned,
    ROUND(100.0*COUNT(*) FILTER (WHERE b.user_id IS NULL)/COUNT(*), 2) AS churn_pct
FROM monthly_active a
LEFT JOIN monthly_active b
       ON b.user_id = a.user_id AND b.month = a.month + interval '1 month'
GROUP BY a.month ORDER BY a.month;
```

**47. Median order value per month** 🟡
```sql
SELECT date_trunc('month', created_at) AS month,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total) AS median_order
FROM orders GROUP BY 1 ORDER BY 1;
```

**48. Products frequently bought together** 🔴
```sql
SELECT a.product_id AS product_a, b.product_id AS product_b, COUNT(*) AS times
FROM order_items a
JOIN order_items b ON a.order_id = b.order_id AND a.product_id < b.product_id
GROUP BY 1, 2 HAVING COUNT(*) >= 10 ORDER BY times DESC;
```

**49. Fill gaps in a time series** 🟡
```sql
SELECT d::date AS day, COALESCE(SUM(o.total), 0) AS revenue
FROM generate_series('2026-01-01'::date, '2026-12-31', '1 day') d
LEFT JOIN orders o ON o.created_at::date = d::date
GROUP BY d ORDER BY d;
```

**50. Rolling 28-day active users** 🔴
```sql
SELECT day,
    (SELECT COUNT(DISTINCT user_id) FROM events e
     WHERE e.day > d.day - 28 AND e.day <= d.day) AS rolling_28d_users
FROM (SELECT DISTINCT day FROM events) d ORDER BY day;
```

**51. Attribution — first vs last touch** 🔴
```sql
SELECT
    FIRST_VALUE(channel) OVER w AS first_touch,
    LAST_VALUE(channel)  OVER w AS last_touch,
    conversion_value
FROM touchpoints
WINDOW w AS (PARTITION BY user_id ORDER BY touched_at
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
```

**52. A/B test results with significance inputs** 🔴
```sql
SELECT variant,
    COUNT(*) AS users,
    COUNT(*) FILTER (WHERE converted) AS conversions,
    ROUND(100.0*COUNT(*) FILTER (WHERE converted)/COUNT(*), 3) AS conv_rate,
    -- ⭐ standard error, for a two-proportion z-test
    SQRT((COUNT(*) FILTER (WHERE converted)::numeric/COUNT(*))
         * (1 - COUNT(*) FILTER (WHERE converted)::numeric/COUNT(*))
         / COUNT(*)) AS std_error
FROM experiment_assignments GROUP BY variant;
```

### E. Optimization & Edge Cases (53–60)

**53. Rewrite a slow OFFSET pagination** 🟡 — see [§10](#10-query-optimization)

**54. Find the missing index for a query** 🟡
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 5 AND status = 'paid'
ORDER BY created_at DESC LIMIT 20;
-- ⭐ Seq Scan + high "Rows Removed by Filter" →
CREATE INDEX ON orders (customer_id, status, created_at DESC);
--                       ⭐ equality, equality, then the ORDER BY column
```

**55. Detect and fix an N+1 pattern** 🟡
```sql
-- ⚠️ Symptom in the plan: loops=1000 on an inner node
-- ✅ Replace per-row lookups with a single join or an IN with an array
SELECT * FROM users WHERE id = ANY($1::bigint[]);
```

**56. Safe large UPDATE** 🟡 — batched loop, see [§9](#9-modifying-data)

**57. Deduplicate a huge table efficiently** 🔴
```sql
-- ⭐ For very large tables, rebuilding beats deleting
CREATE TABLE t_new AS
SELECT DISTINCT ON (email) * FROM t ORDER BY email, created_at DESC;
-- then swap the tables in a transaction
```

**58. Count distinct approximately at scale** 🟡
```sql
-- ⭐ HyperLogLog — ~0.8% error in a few KB, and MERGEABLE
SELECT hll_cardinality(hll_add_agg(hll_hash_text(user_id))) FROM events;
```

**59. Query with correct NULL semantics** 🟡
```sql
-- Find rows where a value CHANGED, treating NULL correctly
SELECT * FROM audit
WHERE old_value IS DISTINCT FROM new_value;   -- ⭐ NULL-safe <>
```

**60. Prevent double-booking with a constraint** 🔴
```sql
-- ⭐ The DATABASE enforces it — not application logic
ALTER TABLE bookings ADD CONSTRAINT no_overlap
EXCLUDE USING gist (
    room_id WITH =,
    tstzrange(start_at, end_at) WITH &&    -- ⭐ && means "overlaps"
) WHERE (status IN ('confirmed', 'pending'));
```

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                        SQL — ONE PAGE                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ EXECUTION ORDER: FROM→WHERE→GROUP BY→HAVING→SELECT→WINDOW→        ║
║   DISTINCT→ORDER BY→LIMIT                                            ║
║   → ⚠️ no SELECT alias in WHERE · ✅ alias IS allowed in ORDER BY      ║
║   → WHERE filters ROWS, HAVING filters GROUPS                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ LEFT JOIN + condition in WHERE = silently an INNER JOIN           ║
║   → put the condition in ON, or allow `OR x.id IS NULL`              ║
║ ⚠️ FAN-OUT: joining two 1:N tables MULTIPLIES rows and corrupts sums  ║
║   → aggregate each side separately, THEN join                        ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️⚠️ NULL = unknown. NULL = NULL is NULL, not TRUE.                   ║
║   ⭐ NOT IN with any NULL returns ZERO ROWS → always use NOT EXISTS   ║
║   COUNT(*) counts all · COUNT(col) SKIPS NULLs                       ║
║   IS DISTINCT FROM = NULL-safe <>  ·  NULLIF guards division by zero ║
╠══════════════════════════════════════════════════════════════════════╣
║ WINDOW keeps rows; GROUP BY collapses them                           ║
║   ROW_NUMBER(1,2,3,4) · RANK(1,2,2,4 gaps) · DENSE_RANK(1,2,2,3)     ║
║   ⭐⭐ DEFAULT FRAME with ORDER BY = RUNNING total, not group total    ║
║     → add ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING   ║
║   ⚠️ LAST_VALUE needs an explicit frame · prefer ROWS over RANGE      ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ CONDITIONAL AGGREGATION (FILTER / CASE) = pivot in one scan        ║
║ ⭐ GAPS AND ISLANDS: (value − ROW_NUMBER()) is constant within a run  ║
║ RECURSIVE CTE: ⭐ ALWAYS bound depth + track path (cycle protection)  ║
║ UNION ALL unless you truly need dedup (UNION sorts — slower)         ║
╠══════════════════════════════════════════════════════════════════════╣
║ INDEXES: leftmost prefix · ⭐ equality cols FIRST, range col LAST     ║
║   killed by: function on column · leading % · type mismatch · OR     ║
║   covering INCLUDE → index-only scan · partial WHERE → tiny index    ║
║ ⭐ CURSOR pagination not OFFSET — and ALWAYS include a unique         ║
║   tiebreaker in the sort key                                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ EXPLAIN: read inside-out · est vs actual rows (10× gap → ANALYZE) ·  ║
║   loops=N means N+1 · Buffers read = disk                            ║
║ ⭐ FOR UPDATE SKIP LOCKED = the job-queue pattern                     ║
║ ⭐ EXCLUDE USING gist (... WITH &&) = DB-enforced no-overlap          ║
║ ⚠️ Batch large UPDATEs — never one giant statement                    ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Databases](../03-backend/databases.md) · [Data Engineering](data-engineering.md)
