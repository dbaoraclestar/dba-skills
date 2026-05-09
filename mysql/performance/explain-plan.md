# EXPLAIN Plan Analysis — Reading and Optimizing MySQL Query Execution Plans

## Overview

The `EXPLAIN` statement is the primary tool for understanding how MySQL executes a query. It reveals the query execution plan chosen by the optimizer, including which indexes are used, the join order, estimated row counts, and the access methods applied to each table. Every DBA and developer working with MySQL should be fluent in reading EXPLAIN output.

Use EXPLAIN whenever a query runs slower than expected, when validating that a new index is actually being used, or when reviewing SQL before deploying to production. It is non-destructive and does not execute the underlying query (except for `EXPLAIN ANALYZE`, which does execute it). Starting with MySQL 8.0.18, `EXPLAIN ANALYZE` provides actual execution statistics alongside estimates, making it the gold standard for plan analysis.

EXPLAIN works with SELECT, INSERT, UPDATE, DELETE, and REPLACE statements. For DML statements, it shows what the optimizer would do without actually modifying data. Understanding every column of EXPLAIN output is foundational to MySQL performance tuning.

## Key Concepts

**Execution Plan**: The sequence of operations MySQL will perform to satisfy a query. The optimizer evaluates multiple strategies and picks the one with the lowest estimated cost.

**Access Type (type column)**: Describes how MySQL accesses rows in a table, ranging from the worst (`ALL` -- full table scan) to the best (`system`/`const` -- single row lookup).

**Key Length (key_len)**: The number of bytes of the index MySQL actually uses. Critical for composite indexes to determine how many columns of a multi-column index are being utilized.

**Filtered**: The estimated percentage of rows that will survive the WHERE clause after the access method fetches them. A low filtered value means MySQL fetches many rows but discards most of them.

**Rows**: The optimizer's estimate of how many rows it will examine. This is an estimate, not exact. Large discrepancies between estimated and actual rows indicate stale statistics.

## Access Types (type column) — Best to Worst

The `type` column is the single most important field in EXPLAIN output. Here are the access types ranked from best to worst:

```
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > 
unique_subquery > index_subquery > range > index > ALL
```

### system and const

```sql
-- const: primary key or unique index lookup with a constant value
EXPLAIN SELECT * FROM orders WHERE order_id = 100042;
-- +----+-------+------+---------------+---------+---------+-------+------+-------+
-- | id | type  | table| possible_keys | key     | key_len | ref   | rows | Extra |
-- +----+-------+------+---------------+---------+---------+-------+------+-------+
-- |  1 | const | orders| PRIMARY      | PRIMARY | 8       | const |    1 |       |
-- +----+-------+------+---------------+---------+---------+-------+------+-------+
```

### eq_ref

One row read from this table for each combination of rows from the previous tables. Used when the join uses all parts of a PRIMARY KEY or UNIQUE NOT NULL index.

```sql
EXPLAIN SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.order_date >= '2025-01-01';
-- orders: type=range (on order_date index)
-- customers: type=eq_ref (primary key lookup per order row)
```

### ref

All rows with matching index values are read. Used for non-unique indexes or leftmost prefix of a composite index.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 5023;
-- type=ref, key=idx_customer_id, ref=const
```

### range

Index range scan. Occurs with `>`, `<`, `>=`, `<=`, `BETWEEN`, `IN()`, and `LIKE 'prefix%'`.

```sql
EXPLAIN SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-03-31';
-- type=range, key=idx_order_date
```

### index

Full index scan. MySQL reads every entry in the index (not the table). Better than ALL because the index is typically smaller than the full table, but still reads every row in the index.

```sql
EXPLAIN SELECT customer_id FROM orders;
-- type=index (scanning the idx_customer_id index to get all values)
```

### ALL

Full table scan. The worst access type. MySQL reads every row in the table.

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(order_date) = 2025;
-- type=ALL (function on column prevents index usage)
```

## The Extra Column

The `Extra` column contains critical details about how MySQL processes each step.

| Extra Value | Meaning | Good/Bad |
|---|---|---|
| Using index | Covering index; no table data access needed | Good |
| Using where | WHERE clause filters rows after fetching | Neutral |
| Using temporary | Temporary table created (GROUP BY, DISTINCT, UNION) | Investigate |
| Using filesort | Extra sort pass needed, not served by index order | Investigate |
| Using index condition | Index Condition Pushdown (ICP) active | Good |
| Using join buffer | No index on join column; block nested loop used | Bad |
| Using MRR | Multi-Range Read optimization | Good |
| Impossible WHERE | WHERE clause always evaluates to false | Info |
| Select tables optimized away | Resolved via index without reading tables (MIN/MAX) | Good |
| Backward index scan | Scanning index in reverse order (8.0+) | Neutral |

## EXPLAIN FORMAT=JSON

JSON format provides far more detail than traditional tabular output, including cost estimates and detailed descriptions.

```sql
EXPLAIN FORMAT=JSON
SELECT c.customer_name, COUNT(o.order_id) AS order_count
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_date >= '2025-01-01'
GROUP BY c.customer_name
HAVING order_count > 5
ORDER BY order_count DESC\G
```

Key JSON fields to examine:

```json
{
  "query_cost": "1245.60",
  "table": {
    "table_name": "orders",
    "access_type": "range",
    "rows_examined_per_scan": 8432,
    "rows_produced_per_join": 8432,
    "filtered": "100.00",
    "cost_info": {
      "read_cost": "832.10",
      "eval_cost": "843.20",
      "prefix_cost": "1675.30",
      "data_read_per_join": "1M"
    },
    "used_columns": ["order_id", "customer_id", "order_date"],
    "attached_condition": "(`db`.`o`.`order_date` >= '2025-01-01')"
  }
}
```

## EXPLAIN FORMAT=TREE

Available since MySQL 8.0.16. Displays the execution plan as a tree, showing the actual iterator-based execution structure.

```sql
EXPLAIN FORMAT=TREE
SELECT c.customer_name, SUM(o.total_amount)
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = 'US-WEST'
GROUP BY c.customer_name\G

-- -> Aggregate: sum(o.total_amount)
--     -> Nested loop inner join  (cost=2345.10 rows=4520)
--         -> Index lookup on c using idx_region (region='US-WEST')  (cost=45.20 rows=120)
--         -> Index lookup on o using idx_customer_id (customer_id=c.customer_id)  (cost=15.80 rows=38)
```

## EXPLAIN ANALYZE (MySQL 8.0.18+)

EXPLAIN ANALYZE actually executes the query and shows both estimated and actual row counts and timing. This is the most powerful diagnostic tool.

```sql
EXPLAIN ANALYZE
SELECT p.product_name, SUM(oi.quantity) AS total_sold
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.order_date >= '2025-01-01'
  AND p.category = 'Electronics'
GROUP BY p.product_name
ORDER BY total_sold DESC
LIMIT 20\G

-- -> Limit: 20 row(s)  (actual time=45.230..45.235 rows=20 loops=1)
--     -> Sort: total_sold DESC, limit input to 20 row(s) per chunk
--        (actual time=45.228..45.232 rows=20 loops=1)
--         -> Table scan on <temporary>  (actual time=44.105..44.156 rows=187 loops=1)
--             -> Aggregate using temporary table
--                (actual time=44.102..44.102 rows=187 loops=1)
--                 -> Nested loop inner join  (cost=12450.30 rows=8930)
--                    (actual time=0.125..38.442 rows=9215 loops=1)
--                     -> Nested loop inner join  (cost=5680.20 rows=8930)
--                        (actual time=0.098..22.315 rows=9215 loops=1)
--                         -> Index lookup on p using idx_category (category='Electronics')
--                            (cost=32.50 rows=245) (actual time=0.042..0.385 rows=187 loops=1)
--                         -> Index lookup on oi using idx_product (product_id=p.product_id)
--                            (cost=18.20 rows=36) (actual time=0.015..0.112 rows=49 loops=187)
--                     -> Single-row index lookup on o using PRIMARY (order_id=oi.order_id)
--                        (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=9215)
```

### Reading EXPLAIN ANALYZE Output

- **actual time=X..Y**: X is time to return the first row, Y is time to return all rows (in milliseconds)
- **rows=N**: Actual number of rows produced
- **loops=N**: Number of times this step was executed
- **Total work** = actual time * loops

### Spotting Problems: Estimated vs Actual Row Discrepancies

```sql
-- If you see something like:
--   (cost=100.50 rows=10) (actual time=5.230..892.100 rows=48500 loops=1)
-- The optimizer estimated 10 rows but got 48,500. This means:
-- 1. Table statistics are stale -> run ANALYZE TABLE
-- 2. Data distribution is skewed -> consider histograms (8.0+)
-- 3. Correlated column values confuse the optimizer
```

## key_len Interpretation

The `key_len` value tells you how many bytes of a composite index are being used. Calculate expected key_len from column types:

| Data Type | Bytes | If NULLABLE add |
|---|---|---|
| INT | 4 | +1 |
| BIGINT | 8 | +1 |
| DATE | 3 | +1 |
| DATETIME | 5 (5.6+) | +1 |
| TIMESTAMP | 4 | +1 |
| CHAR(N) | N * charset_maxlen | +1 |
| VARCHAR(N) | N * charset_maxlen + 2 | +1 |

```sql
-- Given index: idx_composite (customer_id INT NOT NULL, status VARCHAR(20), order_date DATE)
-- charset = utf8mb4 (4 bytes per char)
-- Full index usage: key_len = 4 + (20*4+2) + 3 = 89
-- First two columns: key_len = 4 + (20*4+2) = 86
-- First column only: key_len = 4

EXPLAIN SELECT * FROM orders
WHERE customer_id = 100 AND status = 'shipped';
-- key_len=86 -> using first two columns of composite index
```

## Common Bad Plan Symptoms and Fixes

### Symptom: Full Table Scan on Large Table

```sql
-- Problem: function applied to indexed column
EXPLAIN SELECT * FROM orders WHERE DATE(created_at) = '2025-06-15';
-- type=ALL

-- Fix: rewrite to use range scan
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2025-06-15' AND created_at < '2025-06-16';
-- type=range
```

### Symptom: Using filesort on Large Result Set

```sql
-- Problem: ORDER BY column not matching index order
EXPLAIN SELECT * FROM orders WHERE customer_id = 100 ORDER BY order_date;

-- Fix: create composite index matching both WHERE and ORDER BY
ALTER TABLE orders ADD INDEX idx_cust_date (customer_id, order_date);
```

### Symptom: Using temporary + Using filesort Together

```sql
-- Problem: GROUP BY and ORDER BY on different columns
EXPLAIN SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id
ORDER BY COUNT(*) DESC;

-- Fix: if possible, add index on GROUP BY column; accept filesort for ORDER BY on aggregate
ALTER TABLE orders ADD INDEX idx_customer_id (customer_id);
```

### Symptom: Using join buffer (Block Nested Loop)

```sql
-- Problem: missing index on join column
EXPLAIN SELECT * FROM orders o JOIN shipments s ON s.tracking_number = o.tracking_number;
-- Extra: Using join buffer (Block Nested Loop)

-- Fix: add index on the join column
ALTER TABLE shipments ADD INDEX idx_tracking (tracking_number);
```

## Comparing Plans Before and After Index Changes

```sql
-- Step 1: Check current plan
EXPLAIN FORMAT=JSON SELECT * FROM payments
WHERE merchant_id = 'M-90210' AND payment_date >= '2025-01-01'
ORDER BY payment_date DESC LIMIT 50\G

-- Step 2: Try an invisible index (MySQL 8.0+) to test without affecting other queries
ALTER TABLE payments ADD INDEX idx_merchant_date (merchant_id, payment_date DESC) INVISIBLE;

-- Step 3: Make it visible to your session only
SET optimizer_switch = 'use_invisible_indexes=on';

-- Step 4: Check new plan
EXPLAIN FORMAT=JSON SELECT * FROM payments
WHERE merchant_id = 'M-90210' AND payment_date >= '2025-01-01'
ORDER BY payment_date DESC LIMIT 50\G

-- Step 5: If plan is better, make index visible globally
ALTER TABLE payments ALTER INDEX idx_merchant_date VISIBLE;
```

## Best Practices

- Always run EXPLAIN before deploying new queries or schema changes to production
- Use EXPLAIN ANALYZE for accurate timing data, but be aware it actually executes the query
- Check key_len to verify composite indexes are fully utilized
- Compare estimated rows vs actual rows (via EXPLAIN ANALYZE) to detect stale statistics
- Run ANALYZE TABLE periodically to keep optimizer statistics fresh
- Use EXPLAIN FORMAT=JSON for cost analysis and FORMAT=TREE for iterator visualization
- Pay attention to filtered percentage -- low values indicate wasted row reads
- Check for "Using temporary" and "Using filesort" in Extra and evaluate if indexes can eliminate them
- On MySQL 8.0+, use invisible indexes to test index additions safely before making them visible

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Applying functions to indexed columns in WHERE (e.g., `YEAR(col)`) | Full table scan, index ignored | Rewrite as range condition on the raw column |
| Ignoring key_len on composite indexes | Only leftmost columns used, rest wasted | Verify key_len matches expected byte count for desired columns |
| Trusting EXPLAIN rows estimate blindly | Poor capacity planning, wrong index choice | Use EXPLAIN ANALYZE for actual row counts |
| Not running ANALYZE TABLE after bulk loads | Stale statistics lead to bad plans | Schedule ANALYZE TABLE after major data changes |
| Using SELECT * instead of needed columns | Prevents covering index optimization | Select only required columns |
| Ignoring the join buffer warning | Nested loop without index, O(n*m) cost | Add index on the join predicate column |
| Creating too many single-column indexes instead of composites | Index merge is slower than a proper composite | Design composite indexes for frequent query patterns |

## MySQL Version Notes

**MySQL 5.7**:
- EXPLAIN FORMAT=JSON available
- No EXPLAIN ANALYZE
- No EXPLAIN FORMAT=TREE
- No invisible indexes for safe testing
- Uses Block Nested Loop (BNL) for joins without indexes

**MySQL 8.0**:
- EXPLAIN FORMAT=TREE added (8.0.16)
- EXPLAIN ANALYZE added (8.0.18)
- Hash join replaces BNL for equi-joins without indexes (8.0.18+)
- Invisible indexes for safe testing (8.0)
- Descending indexes (8.0) - actual DESC index scans, not backward scans
- Functional indexes (8.0.13+)
- Histogram statistics (8.0)
- EXPLAIN shows actual hash join usage

**MySQL 8.4 / 9.x**:
- EXPLAIN ANALYZE improvements with more granular iterator timing
- Improved cost model accuracy for hash joins
- Better handling of derived table optimization visibility in plans
- Enhanced JSON format output with additional fields

## Sources

- [MySQL EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [MySQL EXPLAIN ANALYZE](https://dev.mysql.com/doc/refman/8.0/en/explain.html#explain-analyze)
- [MySQL Extended EXPLAIN Output](https://dev.mysql.com/doc/refman/8.0/en/explain-extended.html)
- [MySQL Optimizing Queries with EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html)
- [MySQL EXPLAIN FORMAT=TREE](https://dev.mysql.com/doc/refman/8.0/en/explain.html#explain-format-tree)
