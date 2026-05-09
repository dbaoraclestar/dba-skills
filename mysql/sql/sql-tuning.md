# MySQL SQL Tuning — Query Rewriting, Index Hints, Optimizer Switches, and Sort Optimization

## Overview

SQL tuning in MySQL focuses on helping the InnoDB query optimizer choose the most efficient execution plan. The optimizer uses cost-based decisions informed by index statistics, table cardinality estimates, and buffer pool state. When it makes suboptimal choices, you can intervene through query rewriting, index hints, optimizer switch flags, and covering indexes.

Unlike Oracle or PostgreSQL, MySQL's optimizer has historically been weaker at certain transformations (e.g., subquery decorrelation, hash joins for complex queries). Understanding these weaknesses allows you to write SQL that the MySQL optimizer handles well from the start, rather than relying on it to transform inefficient SQL into efficient plans.

This skill covers practical query rewriting techniques, index hint syntax, key optimizer_switch flags, covering indexes, implicit type conversion pitfalls, and ORDER BY / GROUP BY optimization.

## Key Concepts

- **Cost-based optimizer**: MySQL estimates the cost of different execution plans and picks the cheapest one. Cost is based on disk I/O, CPU, and memory estimates.
- **Index statistics**: InnoDB maintains cardinality estimates for each index. Stale statistics cause bad plans. Run `ANALYZE TABLE` to refresh.
- **Covering index**: An index that contains all columns needed by a query. The query is answered entirely from the index without touching the base table (no "table lookup" or "bookmark lookup").
- **Filesort**: MySQL's generic sorting algorithm. Used when no index can provide the required order. Not necessarily file-based; often in memory.
- **Implicit type conversion**: When MySQL compares a column of one type to a value of another type, it converts one side. This can prevent index usage.

## Query Rewriting Techniques

### Subquery to JOIN

```sql
-- SLOW: correlated subquery executed once per row
SELECT u.id, u.name,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u
WHERE u.status = 'active';

-- FAST: JOIN with GROUP BY (single pass)
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.name;
```

### Correlated subquery to derived table

```sql
-- SLOW: correlated subquery in WHERE
SELECT * FROM products p
WHERE p.price > (
    SELECT AVG(price) FROM products
    WHERE category_id = p.category_id
);

-- FAST: derived table (materialized once)
SELECT p.*
FROM products p
INNER JOIN (
    SELECT category_id, AVG(price) AS avg_price
    FROM products
    GROUP BY category_id
) AS cat_avg ON p.category_id = cat_avg.category_id
WHERE p.price > cat_avg.avg_price;
```

### IN subquery to EXISTS or JOIN

```sql
-- POTENTIALLY SLOW: IN with subquery (depends on optimizer version)
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- OFTEN FASTER: EXISTS (short-circuits on first match)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 1000
);

-- ALTERNATIVE: semi-join (explicit)
SELECT DISTINCT u.*
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.total > 1000;
```

### UNION ALL instead of OR on different indexes

```sql
-- SLOW: OR across columns with separate indexes
-- Optimizer may not use index merge efficiently
SELECT * FROM events
WHERE user_id = 42 OR session_id = 'abc123';

-- FASTER: UNION ALL with separate indexed queries
SELECT * FROM events WHERE user_id = 42
UNION ALL
SELECT * FROM events WHERE session_id = 'abc123' AND user_id != 42;
-- Each branch uses its own index
```

### Avoiding SELECT *

```sql
-- SLOW: retrieves all columns, cannot use covering index
SELECT * FROM orders WHERE status = 'pending';

-- FAST: select only needed columns, enabling covering index
SELECT id, user_id, total FROM orders WHERE status = 'pending';
-- With index on (status, id, user_id, total), no table lookup needed
```

## Index Hints

### USE INDEX / FORCE INDEX / IGNORE INDEX

```sql
-- Suggest an index (optimizer may still choose differently)
SELECT * FROM orders USE INDEX (idx_status_date)
WHERE status = 'pending' AND created_at > '2025-01-01';

-- Force an index (optimizer must use it if possible)
SELECT * FROM orders FORCE INDEX (idx_status_date)
WHERE status = 'pending' AND created_at > '2025-01-01';

-- Ignore an index (exclude it from consideration)
SELECT * FROM orders IGNORE INDEX (idx_created_at)
WHERE status = 'pending' AND created_at > '2025-01-01';

-- Scope hints to specific operations
SELECT * FROM orders USE INDEX FOR ORDER BY (idx_created_at)
ORDER BY created_at DESC LIMIT 20;

SELECT * FROM orders USE INDEX FOR JOIN (idx_user_id)
INNER JOIN users ON orders.user_id = users.id;

-- Multiple hints
SELECT * FROM orders
    USE INDEX (idx_status_date)
    IGNORE INDEX FOR ORDER BY (idx_status_date)
WHERE status = 'pending';
```

### When to use index hints

```sql
-- 1. Optimizer picks wrong index due to stale statistics
-- First try: ANALYZE TABLE orders;
-- If still wrong, use FORCE INDEX

-- 2. Testing alternative plans
EXPLAIN SELECT * FROM orders USE INDEX (idx_a) WHERE ...;
EXPLAIN SELECT * FROM orders USE INDEX (idx_b) WHERE ...;
-- Compare rows examined and costs

-- 3. Forcing a covering index scan
EXPLAIN SELECT id, status FROM orders FORCE INDEX (idx_status)
WHERE status IN ('pending', 'processing');
-- Type: index (index scan) instead of ALL (table scan)
```

## Optimizer Switch Flags

```sql
-- View current settings
SELECT @@optimizer_switch\G

-- Set for session
SET SESSION optimizer_switch = 'batched_key_access=on,mrr=on,mrr_cost_based=off';
```

### Key optimizer switches

```sql
-- batched_key_access (BKA): batches index lookups for joins
-- Off by default. Turning on can help join-heavy queries.
SET SESSION optimizer_switch = 'batched_key_access=on,mrr_cost_based=off';

EXPLAIN SELECT /*+ BKA(o) */ u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active';

-- mrr (Multi-Range Read): sorts index lookups by primary key
-- to convert random I/O to sequential I/O
-- Requires mrr_cost_based=off to always activate
SET SESSION optimizer_switch = 'mrr=on,mrr_cost_based=off';

-- semijoin: controls semi-join optimizations for IN/EXISTS subqueries
-- On by default. Strategies: FirstMatch, LooseScan, Materialization, DuplicateWeedout
SET SESSION optimizer_switch = 'semijoin=on,firstmatch=on,loosescan=on';

-- subquery_materialization: materializes subqueries into temp tables
-- On by default. Helps when subquery result is reused.
SET SESSION optimizer_switch = 'materialization=on';

-- derived_merge: merges derived tables/views into outer query
-- On by default. Sometimes turning off forces materialization
-- which can be faster for complex derived tables.
SET SESSION optimizer_switch = 'derived_merge=off';

-- index_merge: combines multiple indexes on a single table
-- On by default. Subtypes: intersection, union, sort_union
SET SESSION optimizer_switch = 'index_merge=on,index_merge_intersection=on';

-- prefer_ordering_index: optimizer prefers index that avoids filesort
-- On by default (8.0.21+). Turn off if it picks a bad ORDER BY index.
SET SESSION optimizer_switch = 'prefer_ordering_index=off';

-- use_invisible_indexes: allows using invisible indexes (for testing)
SET SESSION optimizer_switch = 'use_invisible_indexes=on';
```

### Optimizer hints (8.0+, inline in SQL)

```sql
-- Join order
SELECT /*+ JOIN_ORDER(u, o, oi) */ u.name, o.id, oi.product_id
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id;

-- Index hints via optimizer hints
SELECT /*+ INDEX(orders idx_status_date) */ *
FROM orders WHERE status = 'pending';

-- NO_INDEX (8.0+)
SELECT /*+ NO_INDEX(orders idx_created_at) */ *
FROM orders WHERE status = 'pending';

-- Semijoin strategy
SELECT /*+ SEMIJOIN(@subq1 MATERIALIZATION) */ *
FROM users
WHERE id IN (SELECT /*+ QB_NAME(subq1) */ user_id FROM orders);

-- Set a resource limit
SELECT /*+ MAX_EXECUTION_TIME(5000) */ * FROM big_table WHERE ...;
-- Timeout after 5 seconds
```

## Covering Indexes

```sql
-- A covering index includes all columns referenced in a query
-- The optimizer shows "Using index" in Extra column of EXPLAIN

-- Query:
SELECT user_id, status, created_at FROM orders WHERE status = 'pending';

-- Covering index:
CREATE INDEX idx_covering ON orders (status, user_id, created_at);

-- EXPLAIN shows:
-- type: ref
-- key: idx_covering
-- Extra: Using index  <-- no table lookup

-- Common covering index patterns:

-- Pattern 1: WHERE + SELECT columns
-- Query: SELECT name, email FROM users WHERE status = 'active'
CREATE INDEX idx_status_name_email ON users (status, name, email);

-- Pattern 2: WHERE + ORDER BY + SELECT
-- Query: SELECT id, total FROM orders WHERE user_id = ? ORDER BY created_at DESC
CREATE INDEX idx_uid_created_id_total ON orders (user_id, created_at DESC, id, total);

-- Pattern 3: JOIN covering
-- Query: SELECT o.id FROM orders o JOIN users u ON o.user_id = u.id WHERE u.status = 'active'
-- Index on users (status, id) covers the users side
-- Index on orders (user_id, id) covers the orders side
```

## Avoiding Implicit Type Conversions

```sql
-- PROBLEM: column is VARCHAR, but compared with an integer
-- MySQL converts the column to INT, preventing index use
EXPLAIN SELECT * FROM users WHERE phone = 5551234567;
-- type: ALL (full table scan)
-- phone is VARCHAR(20), but compared to number

-- FIX: use matching types
EXPLAIN SELECT * FROM users WHERE phone = '5551234567';
-- type: ref (index lookup)

-- PROBLEM: column is INT, compared with string containing a number
-- MySQL converts the string to INT -- index IS used, but behavior is fragile
SELECT * FROM users WHERE id = '42';  -- works, but bad practice

-- PROBLEM: charset/collation mismatch in JOINs
-- If tables use different charsets, MySQL converts one side, preventing index use
SELECT * FROM t1 JOIN t2 ON t1.name = t2.name;
-- If t1.name is utf8mb4 and t2.name is latin1, full scan on one side

-- FIX: ensure matching charsets
ALTER TABLE t2 MODIFY name VARCHAR(100) CHARACTER SET utf8mb4;

-- PROBLEM: function on indexed column
SELECT * FROM orders WHERE DATE(created_at) = '2025-01-15';
-- Cannot use index on created_at

-- FIX: range query instead of function
SELECT * FROM orders
WHERE created_at >= '2025-01-15 00:00:00'
  AND created_at < '2025-01-16 00:00:00';
```

## ORDER BY Optimization

### Using index for ORDER BY (no filesort)

```sql
-- Index on (status, created_at)
-- This uses the index for both filtering AND sorting:
SELECT id, status, created_at FROM orders
WHERE status = 'pending'
ORDER BY created_at;
-- EXPLAIN: Using index condition (no filesort)

-- DESC index (8.0+)
CREATE INDEX idx_status_created_desc ON orders (status, created_at DESC);
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 20;

-- Mixed ASC/DESC requires matching index direction (8.0+)
CREATE INDEX idx_mixed ON orders (user_id ASC, created_at DESC);
SELECT * FROM orders WHERE user_id = 42 ORDER BY user_id ASC, created_at DESC;
```

### When filesort is unavoidable

```sql
-- ORDER BY column not in the index used for WHERE
SELECT * FROM orders WHERE status = 'pending' ORDER BY total;
-- Index on (status) helps WHERE but not ORDER BY

-- ORDER BY expression
SELECT * FROM orders ORDER BY total * quantity;

-- ORDER BY with LIMIT optimization
-- MySQL optimizes ORDER BY ... LIMIT n by maintaining a priority queue
-- of n rows instead of sorting the entire result set
SELECT * FROM large_table ORDER BY score DESC LIMIT 10;
-- Only keeps top 10 in memory during scan
```

### Sort buffer tuning

```sql
-- Per-session sort buffer
SHOW VARIABLES LIKE 'sort_buffer_size';  -- default: 256KB

-- Increase for sessions with large sorts
SET SESSION sort_buffer_size = 4 * 1024 * 1024;  -- 4MB

-- Monitor sort operations
SHOW STATUS LIKE 'Sort%';
-- Sort_merge_passes: sorts that spilled to disk (increase sort_buffer_size if high)
-- Sort_rows: total rows sorted
-- Sort_scan: sorts from full table scans
-- Sort_range: sorts from range scans
```

## GROUP BY Optimization

```sql
-- Loose index scan (best case)
-- Index on (department_id, salary)
SELECT department_id, MAX(salary) FROM employees GROUP BY department_id;
-- EXPLAIN Extra: Using index for group-by

-- Tight index scan
-- Index on (department_id, status, salary)
SELECT department_id, status, AVG(salary)
FROM employees
WHERE status = 'active'
GROUP BY department_id, status;

-- Implicit sorting: MySQL historically sorted GROUP BY results
-- In 8.0+, GROUP BY does NOT guarantee order
-- If you need ordered results, add explicit ORDER BY
SELECT department_id, COUNT(*)
FROM employees
GROUP BY department_id
ORDER BY department_id;  -- explicit ORDER BY required in 8.0+

-- ROLLUP for subtotals
SELECT department_id, status, COUNT(*), SUM(salary)
FROM employees
GROUP BY department_id, status WITH ROLLUP;
```

## Best Practices

- Always start with `EXPLAIN` (or `EXPLAIN ANALYZE` in 8.0.18+) before tuning. Understand the current plan before changing anything.
- Run `ANALYZE TABLE` before investigating performance issues. Stale index statistics cause bad optimizer decisions.
- Design indexes for the query, not the table. A covering index for your critical queries eliminates table lookups entirely.
- Avoid functions on indexed columns in WHERE clauses. Rewrite `WHERE YEAR(date_col) = 2025` as a range.
- Use `FORCE INDEX` sparingly and as a last resort. Prefer `ANALYZE TABLE` and query rewriting first. Forced indexes become stale as data changes.
- Profile with `performance_schema` statement digests to find the queries that consume the most time, not just the slowest individual executions.
- Keep `optimizer_switch` changes session-scoped. Never change global optimizer switches in production without thorough testing.
- Use `EXPLAIN FORMAT=JSON` or `EXPLAIN ANALYZE` for detailed cost estimates and actual row counts.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Function on indexed column in WHERE | Index cannot be used, full table scan | Rewrite to avoid the function (use range instead) |
| Implicit type conversion (VARCHAR vs INT) | Index bypassed, full scan | Match data types in comparisons |
| Relying on GROUP BY ordering (8.0+) | Results come in unpredictable order | Add explicit ORDER BY |
| Using `SELECT *` when a covering index exists | Table lookup for every row | Select only needed columns |
| Global `optimizer_switch` changes | Affects all sessions, may regress other queries | Use session-level or hint-level overrides |
| Not running ANALYZE TABLE after bulk loads | Stale statistics, bad plans | Run `ANALYZE TABLE` after significant data changes |
| Using FORCE INDEX permanently | Works today, may be wrong after data growth/distribution changes | Revisit forced indexes periodically |

## MySQL Version Notes

- **5.7**: No descending indexes (all indexes are ascending). No optimizer hints in SQL comments. Limited subquery optimization. `GROUP BY` implies `ORDER BY`. No hash joins.
- **8.0**: Descending indexes (8.0.1+). Hash joins for equi-joins without indexes (8.0.18+). `EXPLAIN ANALYZE` (8.0.18+). `GROUP BY` no longer implies `ORDER BY`. Optimizer hints (`/*+ ... */`) supported. Invisible indexes for testing. `prefer_ordering_index` switch (8.0.21+).
- **8.4 / 9.x**: Improved hash join performance. Better cost model for covering indexes. Parallel query execution for some operations.

## Sources

- [MySQL Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [MySQL EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [MySQL Index Hints](https://dev.mysql.com/doc/refman/8.0/en/index-hints.html)
- [MySQL Optimizer Hints](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)
- [MySQL optimizer_switch](https://dev.mysql.com/doc/refman/8.0/en/switchable-optimizations.html)
