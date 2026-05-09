# Optimizer Hints — Controlling MySQL Query Execution Plans with Hints, Switches, and Statistics

## Overview

MySQL's query optimizer generally makes good decisions, but there are situations where the DBA or developer knows better than the optimizer. This can happen with skewed data distributions, correlated column values, stale statistics, or complex queries where the optimizer's cost model breaks down. Optimizer hints provide a mechanism to guide or override the optimizer's decisions without modifying the query structure.

MySQL supports three levels of optimizer control: (1) statement-level hints embedded in the SQL using `/*+ ... */` comment syntax, (2) session/global-level optimizer switches via the `optimizer_switch` system variable, and (3) statistical supplements like histogram statistics that give the optimizer better information to work with. The preferred approach is to improve the information available to the optimizer (statistics, histograms) before resorting to hard overrides (hints), because hints are brittle -- they encode assumptions about data and schema that may not hold as the system evolves.

Understanding optimizer hints is essential for production troubleshooting. When a critical query suddenly regresses due to a plan change, a targeted hint can provide an immediate fix while the root cause (stale statistics, data skew, missing index) is investigated and resolved properly.

## Key Concepts

**Statement-Level Hints**: Embedded in SQL using `/*+ HINT_NAME(...) */` syntax immediately after SELECT, INSERT, UPDATE, DELETE, or REPLACE. They affect only the statement they appear in and override session-level optimizer_switch settings.

**Index Hints**: The older `USE INDEX`, `FORCE INDEX`, `IGNORE INDEX` syntax placed after the table name in the FROM clause. These predate statement-level hints and are still widely used.

**optimizer_switch**: A system variable containing a comma-separated list of flags that enable/disable specific optimizer features globally or per-session.

**Histograms**: Column-level statistics that describe data distribution, allowing the optimizer to make better cardinality estimates without creating indexes. Available in MySQL 8.0+.

**Cost Model**: The optimizer assigns a cost to each possible execution plan component (page reads, row evaluations, temporary tables, etc.) and picks the plan with the lowest total cost. The cost model can be tuned via the `mysql.server_cost` and `mysql.engine_cost` tables.

## Statement-Level Hints (/*+ ... */)

### Syntax

```sql
-- Hints go immediately after the statement keyword
SELECT /*+ HINT1(...) HINT2(...) */ columns FROM tables WHERE ...;
INSERT /*+ HINT(...) */ INTO table ...;
UPDATE /*+ HINT(...) */ table SET ...;
DELETE /*+ HINT(...) */ FROM table WHERE ...;

-- Multiple hints in one comment block
SELECT /*+ NO_INDEX_MERGE(t1) JOIN_ORDER(t1, t2, t3) */ ...

-- Hints for subqueries use query block names
SELECT /*+ QB_NAME(main) */ *
FROM orders
WHERE customer_id IN (
    SELECT /*+ QB_NAME(sub) SEMIJOIN(@sub MATERIALIZATION) */ customer_id
    FROM premium_customers
);
```

## Join Strategy Hints

### Join Order Hints

```sql
-- Force exact join order
SELECT /*+ JOIN_ORDER(customers, orders, order_items) */
    c.customer_name, o.order_date, oi.product_id
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE c.region = 'US-WEST';

-- Force a table to be first (prefix)
SELECT /*+ JOIN_PREFIX(orders) */
    c.customer_name, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_date >= '2025-01-01';

-- Force a table to be last (suffix)
SELECT /*+ JOIN_SUFFIX(order_items) */
    c.customer_name, COUNT(oi.item_id)
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
GROUP BY c.customer_name;

-- Fix a table at a specific position
SELECT /*+ JOIN_FIXED_ORDER() */
    c.customer_name, o.order_date, oi.quantity
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE c.region = 'US-WEST'
  AND o.order_date >= '2025-01-01';
-- Tables are joined in the order they appear in the FROM clause
```

### Join Algorithm Hints

```sql
-- Force hash join (MySQL 8.0.18+)
SELECT /*+ HASH_JOIN(o, oi) */
    o.order_id, SUM(oi.quantity)
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_date >= '2025-01-01'
GROUP BY o.order_id;

-- Disable hash join, use nested loop
SELECT /*+ NO_HASH_JOIN(o, oi) */
    o.order_id, SUM(oi.quantity)
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_date >= '2025-01-01'
GROUP BY o.order_id;

-- BKA (Batched Key Access) — batches index lookups to allow MRR
SELECT /*+ BKA(o) */
    c.customer_name, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = 'US-WEST';

-- Disable BKA
SELECT /*+ NO_BKA(o) */
    c.customer_name, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;

-- BNL (Block Nested Loop) — MySQL 5.7 / early 8.0
-- Replaced by hash join in MySQL 8.0.18+ but hint still accepted
SELECT /*+ BNL(o) */ ...
SELECT /*+ NO_BNL(o) */ ...
```

## Index Hints (Statement-Level)

### Modern /*+ ... */ Syntax (MySQL 8.0+)

```sql
-- Force index usage
SELECT /*+ INDEX(orders idx_customer_date) */
    order_id, order_date
FROM orders
WHERE customer_id = 100 AND order_date >= '2025-01-01';

-- Prevent index usage
SELECT /*+ NO_INDEX(orders idx_old_status) */
    order_id, status
FROM orders
WHERE status = 'pending';

-- Force join index
SELECT /*+ JOIN_INDEX(o idx_customer_id) */
    c.customer_name, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;

-- Force group-by index
SELECT /*+ GROUP_INDEX(orders idx_status) */
    status, COUNT(*)
FROM orders
GROUP BY status;

-- Force order-by index
SELECT /*+ ORDER_INDEX(orders idx_order_date) */
    order_id, order_date
FROM orders
ORDER BY order_date DESC
LIMIT 100;

-- Index merge hints
SELECT /*+ INDEX_MERGE(orders idx_customer, idx_status) */
    *
FROM orders
WHERE customer_id = 100 OR status = 'urgent';

SELECT /*+ NO_INDEX_MERGE(orders) */
    *
FROM orders
WHERE customer_id = 100 OR status = 'urgent';
```

### Traditional Index Hints (USE/FORCE/IGNORE INDEX)

These are placed after the table name in the FROM clause and have been available since MySQL 5.0.

```sql
-- USE INDEX: suggest indexes to optimizer (it may still choose a table scan)
SELECT * FROM orders USE INDEX (idx_customer_date)
WHERE customer_id = 100 AND order_date >= '2025-01-01';

-- FORCE INDEX: like USE INDEX but makes table scan very expensive to the optimizer
SELECT * FROM orders FORCE INDEX (idx_customer_date)
WHERE customer_id = 100 AND order_date >= '2025-01-01';

-- IGNORE INDEX: exclude specific indexes from consideration
SELECT * FROM orders IGNORE INDEX (idx_old_unused)
WHERE customer_id = 100;

-- Scope-specific hints (FOR JOIN, FOR ORDER BY, FOR GROUP BY)
SELECT * FROM orders USE INDEX FOR ORDER BY (idx_order_date)
WHERE customer_id = 100
ORDER BY order_date DESC;

SELECT * FROM orders
FORCE INDEX FOR JOIN (idx_customer_id)
USE INDEX FOR ORDER BY (idx_order_date)
WHERE customer_id = 100
ORDER BY order_date DESC;

-- USE INDEX () with empty list forces a full table scan
SELECT * FROM orders USE INDEX ()
WHERE customer_id = 100;
```

### USE INDEX vs FORCE INDEX

```sql
-- USE INDEX: optimizer still considers table scan if it thinks it is cheaper
-- FORCE INDEX: optimizer strongly avoids table scan; only uses listed indexes
--              (it can still choose a scan if no listed index is applicable)

-- Common pattern: query was using index yesterday, now it chooses table scan
-- Quick fix (investigate root cause later):
SELECT * FROM orders FORCE INDEX (idx_customer_date)
WHERE customer_id = 100 AND order_date >= '2025-01-01';
```

## Subquery and Semijoin Hints

```sql
-- Control semijoin strategy
SELECT /*+ SEMIJOIN(@sub MATERIALIZATION) */
    c.customer_name
FROM customers c
WHERE c.customer_id IN (
    SELECT /*+ QB_NAME(sub) */ o.customer_id
    FROM orders o
    WHERE o.total_amount > 10000
);

-- Available semijoin strategies:
-- DUPSWEEDOUT, FIRSTMATCH, LOOSESCAN, MATERIALIZATION

-- Disable semijoin (force to use subquery as-is)
SELECT /*+ NO_SEMIJOIN(@sub) */
    c.customer_name
FROM customers c
WHERE c.customer_id IN (
    SELECT /*+ QB_NAME(sub) */ o.customer_id
    FROM orders o
    WHERE o.total_amount > 10000
);

-- Control subquery strategy
SELECT /*+ SUBQUERY(@sub MATERIALIZATION) */
    ...;
-- or
SELECT /*+ SUBQUERY(@sub INTOEXISTS) */
    ...;
```

## SET_VAR Hint

Temporarily set a system variable for the duration of a single statement. Extremely powerful for per-query tuning.

```sql
-- Increase join buffer for one large join
SELECT /*+ SET_VAR(join_buffer_size = 16777216) */
    c.customer_name, COUNT(o.order_id)
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_name;

-- Increase sort buffer for one large sort
SELECT /*+ SET_VAR(sort_buffer_size = 8388608) */
    *
FROM orders
ORDER BY total_amount DESC
LIMIT 1000;

-- Increase tmp_table_size for one query with large GROUP BY
SELECT /*+ SET_VAR(tmp_table_size = 134217728) SET_VAR(max_heap_table_size = 134217728) */
    product_category, COUNT(*) AS cnt, AVG(price) AS avg_price
FROM products
GROUP BY product_category;

-- Use a different optimizer switch for one query
SELECT /*+ SET_VAR(optimizer_switch = 'index_merge=off') */
    *
FROM orders
WHERE customer_id = 100 OR status = 'urgent';

-- Adjust max_execution_time (query timeout)
SELECT /*+ MAX_EXECUTION_TIME(5000) */
    *
FROM large_table
WHERE complex_condition;
-- Kills the query if it exceeds 5 seconds
```

## optimizer_switch System Variable

The `optimizer_switch` controls which optimizer features are enabled. Each flag can be `on` or `off`.

### Viewing and Setting Flags

```sql
-- View all optimizer_switch flags
SELECT @@optimizer_switch\G
-- Or formatted:
SELECT REPLACE(@@optimizer_switch, ',', '\n') AS optimizer_flags;
```

### Key Flags

```sql
-- Index merge optimizations
SET SESSION optimizer_switch = 'index_merge=off';
SET SESSION optimizer_switch = 'index_merge_union=off';
SET SESSION optimizer_switch = 'index_merge_sort_union=off';
SET SESSION optimizer_switch = 'index_merge_intersection=off';

-- Semijoin strategies
SET SESSION optimizer_switch = 'semijoin=on';
SET SESSION optimizer_switch = 'materialization=on';
SET SESSION optimizer_switch = 'firstmatch=on';
SET SESSION optimizer_switch = 'loosescan=on';
SET SESSION optimizer_switch = 'duplicateweedout=on';

-- Derived table (subquery in FROM) optimizations
SET SESSION optimizer_switch = 'derived_merge=on';   -- Merge derived table into outer query
SET SESSION optimizer_switch = 'derived_condition_pushdown=on';  -- Push WHERE into derived table

-- Batched Key Access
SET SESSION optimizer_switch = 'batched_key_access=off';  -- Default off; enable for MRR benefit
SET SESSION optimizer_switch = 'mrr=on';                   -- Multi-Range Read
SET SESSION optimizer_switch = 'mrr_cost_based=on';        -- Let optimizer decide MRR usage

-- ICP (Index Condition Pushdown)
SET SESSION optimizer_switch = 'index_condition_pushdown=on';  -- Default on

-- Hash join (8.0.18+)
SET SESSION optimizer_switch = 'hash_join=on';  -- Default on in 8.0.18+

-- Invisible indexes
SET SESSION optimizer_switch = 'use_invisible_indexes=on';  -- Test invisible indexes

-- Condition fanout filter (8.0)
SET SESSION optimizer_switch = 'condition_fanout_filter=on';

-- Prefer ordering index (8.0.21+)
SET SESSION optimizer_switch = 'prefer_ordering_index=on';

-- Hypergraph optimizer (8.0.31+, experimental)
-- SET SESSION optimizer_switch = 'hypergraph_optimizer=on';
```

### Setting Multiple Flags

```sql
-- Set multiple flags at once (flags not mentioned keep their current value)
SET SESSION optimizer_switch =
    'index_merge=off,index_merge_union=off,mrr=on,mrr_cost_based=off';

-- Reset to defaults
SET SESSION optimizer_switch = DEFAULT;
```

## Histogram Statistics (MySQL 8.0+)

Histograms help the optimizer estimate how many rows match a condition on columns without indexes. They are especially valuable for columns with skewed distributions.

### Creating Histograms

```sql
-- Create a histogram with 100 buckets (default)
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;

-- Create histograms on multiple columns
ANALYZE TABLE orders UPDATE HISTOGRAM ON status, region, payment_method WITH 100 BUCKETS;

-- For columns with few distinct values, fewer buckets suffice
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;

-- For continuous distributions, more buckets give better precision
ANALYZE TABLE orders UPDATE HISTOGRAM ON total_amount WITH 256 BUCKETS;
```

### Viewing Histogram Data

```sql
-- View histogram metadata
SELECT
    SCHEMA_NAME,
    TABLE_NAME,
    COLUMN_NAME,
    JSON_EXTRACT(HISTOGRAM, '$."histogram-type"') AS hist_type,
    JSON_LENGTH(JSON_EXTRACT(HISTOGRAM, '$.buckets')) AS num_buckets,
    JSON_EXTRACT(HISTOGRAM, '$."number-of-buckets-specified"') AS buckets_specified,
    JSON_EXTRACT(HISTOGRAM, '$."sampling-rate"') AS sampling_rate,
    JSON_EXTRACT(HISTOGRAM, '$."last-updated"') AS last_updated
FROM information_schema.COLUMN_STATISTICS
WHERE TABLE_NAME = 'orders'\G

-- Singleton histogram (for low-cardinality columns)
-- Each bucket: [value, cumulative_frequency]
-- Example: status column with 5 distinct values

-- Equi-height histogram (for high-cardinality columns)  
-- Each bucket: [lower_bound, upper_bound, cumulative_frequency, num_distinct]
-- Example: total_amount column with continuous distribution
```

### When Histograms Help

```sql
-- Scenario: highly skewed status column (90% 'completed', 5% 'shipped', 5% other)
-- Without histogram: optimizer assumes uniform distribution
-- With histogram: optimizer knows 'pending' matches only 2% of rows

-- Before histogram:
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- rows: 50000 (assumes uniform ~20% for each of 5 statuses)

-- After histogram:
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 10 BUCKETS;
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- rows: 5000 (knows 'pending' is ~2% of rows)
-- Optimizer may now choose an index it previously rejected
```

### Dropping Histograms

```sql
-- Drop histogram for a specific column
ANALYZE TABLE orders DROP HISTOGRAM ON status;

-- Drop histograms for multiple columns
ANALYZE TABLE orders DROP HISTOGRAM ON status, region;
```

## Optimizer Cost Model

MySQL's cost model can be tuned via two system tables. This is an advanced technique for environments where the default costs do not reflect the actual hardware characteristics.

```sql
-- View current server-level cost constants
SELECT * FROM mysql.server_cost;
-- +------------------------------+------------+---------------------+---------+
-- | cost_name                    | cost_value | last_update         | comment |
-- +------------------------------+------------+---------------------+---------+
-- | disk_temptable_create_cost   |       NULL |                     | NULL    |
-- | disk_temptable_row_cost      |       NULL |                     | NULL    |
-- | key_compare_cost             |       NULL |                     | NULL    |
-- | memory_temptable_create_cost |       NULL |                     | NULL    |
-- | memory_temptable_row_cost    |       NULL |                     | NULL    |
-- | row_evaluate_cost            |       NULL |                     | NULL    |
-- +------------------------------+------------+---------------------+---------+
-- NULL means using the compiled-in default

-- View engine-level cost constants
SELECT * FROM mysql.engine_cost;
-- device_type 0 = default for all devices
-- Costs: io_block_read_cost, memory_block_read_cost

-- Example: adjust for SSD (make disk I/O cheaper relative to memory)
UPDATE mysql.engine_cost
SET cost_value = 1.0
WHERE cost_name = 'io_block_read_cost';
-- Default is 1.0; reduce to 0.25-0.5 for fast SSDs to make the optimizer
-- more willing to use indexes (since random I/O is cheaper on SSDs)

-- After changing costs, flush to take effect:
FLUSH OPTIMIZER_COSTS;
```

## Combining Hints for Complex Queries

```sql
-- Real-world example: force a specific plan for a known problematic query
SELECT /*+
    JOIN_ORDER(c, o, oi)
    INDEX(o idx_customer_date)
    INDEX(oi idx_order_product)
    NO_INDEX_MERGE(o)
    SET_VAR(join_buffer_size = 4194304)
    MAX_EXECUTION_TIME(10000)
*/
    c.customer_name,
    o.order_date,
    p.product_name,
    oi.quantity,
    oi.unit_price
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE c.region = 'US-WEST'
  AND o.order_date >= '2025-01-01'
  AND o.status = 'completed'
ORDER BY o.order_date DESC
LIMIT 100;
```

## Best Practices

- Improve statistics (ANALYZE TABLE, histograms) before resorting to optimizer hints
- Use hints as targeted fixes for specific queries, not as a general practice
- Document why each hint is needed in a code comment alongside the query
- Re-evaluate hints periodically -- they may become unnecessary or counterproductive after data changes
- Prefer session-level optimizer_switch changes over global changes
- Use SET_VAR hints for per-query buffer sizing instead of bloating global buffer sizes
- Test hint effectiveness with EXPLAIN before and after applying the hint
- Use FORCE INDEX sparingly; it prevents the optimizer from adapting to data growth
- Create histograms on columns that appear in WHERE clauses with skewed distributions
- Use invisible indexes and SET optimizer_switch='use_invisible_indexes=on' to test new indexes safely

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Using FORCE INDEX on every slow query | Hints become stale as data changes; optimizer cannot adapt | Investigate root cause (statistics, index design); use FORCE INDEX only as temporary fix |
| Setting optimizer_switch globally instead of per-session | Affects all connections; may regress other queries | Use SET SESSION or per-query SET_VAR hint |
| Creating histograms on indexed columns | Redundant; optimizer already uses index statistics for indexed columns | Create histograms on non-indexed columns with skewed distributions |
| Not refreshing histograms after major data changes | Stale histograms lead to wrong cardinality estimates | Re-run ANALYZE TABLE ... UPDATE HISTOGRAM after bulk loads |
| Applying join order hints without understanding the data | Forced join order may be far worse than optimizer's choice | Always EXPLAIN before and after; verify with EXPLAIN ANALYZE |
| Using USE INDEX when FORCE INDEX is needed | Optimizer may still choose table scan if it thinks it is cheaper | Use FORCE INDEX if you truly need to prevent table scans |
| Modifying mysql.server_cost without testing | Wrong cost values degrade plans across all queries | Test cost changes on replicas first; benchmark before production deployment |

## MySQL Version Notes

**MySQL 5.7**:
- Traditional index hints (USE/FORCE/IGNORE INDEX) available
- optimizer_switch with most flags (no hash_join, no hypergraph)
- No statement-level `/*+ ... */` hint syntax for indexes
- No SET_VAR hint
- No histogram statistics
- BNL (Block Nested Loop) for joins without indexes
- Query block naming with `QB_NAME()` available

**MySQL 8.0**:
- Statement-level `/*+ ... */` hints for indexes, joins, subqueries (8.0.0+)
- SET_VAR hint (8.0.0+)
- MAX_EXECUTION_TIME hint (moved from 5.7 optimizer hint syntax)
- Histogram statistics with ANALYZE TABLE ... UPDATE HISTOGRAM (8.0.0)
- Hash join replaces BNL for equi-joins (8.0.18+)
- HASH_JOIN / NO_HASH_JOIN hints (8.0.18+)
- INDEX / NO_INDEX / JOIN_INDEX / GROUP_INDEX / ORDER_INDEX hints (8.0.20+)
- `prefer_ordering_index` optimizer_switch flag (8.0.21+)
- `derived_condition_pushdown` optimizer_switch flag (8.0.22+)
- Hypergraph optimizer (experimental, 8.0.31+)

**MySQL 8.4 / 9.x**:
- Improved hypergraph optimizer moving toward general availability
- Enhanced hint syntax for new join algorithms
- Better cost model defaults for modern hardware (NVMe, large RAM)
- Improved histogram accuracy with adaptive sampling
- New optimizer_switch flags for advanced join strategies

## Sources

- [MySQL Optimizer Hints](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)
- [MySQL Index Hints](https://dev.mysql.com/doc/refman/8.0/en/index-hints.html)
- [MySQL optimizer_switch](https://dev.mysql.com/doc/refman/8.0/en/switchable-optimizations.html)
- [MySQL Histogram Statistics](https://dev.mysql.com/doc/refman/8.0/en/optimizer-statistics.html)
- [MySQL Optimizer Cost Model](https://dev.mysql.com/doc/refman/8.0/en/cost-model.html)
- [MySQL SET_VAR Hint](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-set-var)
