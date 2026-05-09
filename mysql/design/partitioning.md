# MySQL Partitioning — Strategies, Pruning, and Maintenance

## Overview

Partitioning divides a single logical table into multiple physical pieces (partitions)
that are stored and managed independently. The MySQL server handles routing queries
to the correct partitions transparently. When a query's WHERE clause matches the
partition key, the optimizer eliminates non-matching partitions entirely — this is
called partition pruning and can dramatically reduce the amount of data scanned.

Partitioning is most beneficial for very large tables (hundreds of millions of rows
or more) where queries consistently filter on the partition key, and where partition-level
maintenance operations (DROP PARTITION, TRUNCATE PARTITION) can replace expensive
DELETE statements. It is not a substitute for proper indexing and should not be used
on small tables.

Use this reference when designing time-series data models, archival strategies,
large lookup tables, or any scenario where you need to efficiently manage and query
tables with billions of rows.

## Key Concepts

- **Partition Key**: The column(s) or expression used to determine which partition
  a row belongs to. Must be part of every unique key (including the primary key).
- **Partition Pruning**: The optimizer's ability to skip partitions that cannot
  contain matching rows based on the WHERE clause.
- **Partition-Level Operations**: DDL operations that act on individual partitions
  (ADD, DROP, TRUNCATE, REORGANIZE, EXCHANGE, ANALYZE, REPAIR, OPTIMIZE).
- **Subpartitioning**: A second level of partitioning within each partition
  (composite partitioning). Only HASH and KEY subpartitioning are supported.

## RANGE Partitioning

Rows are assigned to partitions based on a column value falling within a defined
range. Most commonly used for time-series data.

```sql
-- Partition by year using RANGE
CREATE TABLE sales (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    sale_date   DATE NOT NULL,
    customer_id INT UNSIGNED NOT NULL,
    amount      DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (id, sale_date)
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);

-- Partition by month using RANGE COLUMNS (no function needed)
CREATE TABLE audit_log (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    log_date   DATE NOT NULL,
    action     VARCHAR(50) NOT NULL,
    details    TEXT,
    PRIMARY KEY (id, log_date)
) PARTITION BY RANGE COLUMNS (log_date) (
    PARTITION p202501 VALUES LESS THAN ('2025-02-01'),
    PARTITION p202502 VALUES LESS THAN ('2025-03-01'),
    PARTITION p202503 VALUES LESS THAN ('2025-04-01'),
    PARTITION p202504 VALUES LESS THAN ('2025-05-01'),
    PARTITION p202505 VALUES LESS THAN ('2025-06-01'),
    PARTITION p202506 VALUES LESS THAN ('2025-07-01'),
    PARTITION pmax    VALUES LESS THAN (MAXVALUE)
);

-- RANGE on integer column
CREATE TABLE log_entries (
    id      BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id INT UNSIGNED NOT NULL,
    message VARCHAR(500) NOT NULL,
    PRIMARY KEY (id, user_id)
) PARTITION BY RANGE (user_id) (
    PARTITION p0    VALUES LESS THAN (100000),
    PARTITION p1    VALUES LESS THAN (200000),
    PARTITION p2    VALUES LESS THAN (300000),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

## LIST Partitioning

Rows are assigned based on column value matching a discrete set.

```sql
-- Partition by region code
CREATE TABLE orders (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    region_code TINYINT UNSIGNED NOT NULL,
    order_date  DATE NOT NULL,
    total       DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (id, region_code)
) PARTITION BY LIST (region_code) (
    PARTITION p_americas VALUES IN (1, 2, 3),
    PARTITION p_europe   VALUES IN (4, 5, 6),
    PARTITION p_asia     VALUES IN (7, 8, 9),
    PARTITION p_other    VALUES IN (10, 11, 12)
);

-- LIST COLUMNS for string-based partitioning
CREATE TABLE tickets (
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    status   VARCHAR(20) NOT NULL,
    subject  VARCHAR(200) NOT NULL,
    PRIMARY KEY (id, status)
) PARTITION BY LIST COLUMNS (status) (
    PARTITION p_open    VALUES IN ('new', 'open', 'reopened'),
    PARTITION p_active  VALUES IN ('in_progress', 'pending', 'on_hold'),
    PARTITION p_closed  VALUES IN ('resolved', 'closed', 'cancelled')
);
```

## HASH Partitioning

Rows are distributed across partitions using a modulus function. Provides even
distribution but does not support partition pruning on range queries.

```sql
-- HASH partitioning for even data distribution
CREATE TABLE sessions (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id     INT UNSIGNED NOT NULL,
    session_data JSON,
    created_at   DATETIME NOT NULL,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id)
PARTITIONS 16;

-- LINEAR HASH: uses a power-of-two algorithm
-- Faster to add/drop partitions but less even distribution
CREATE TABLE cache_entries (
    id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    cache_key INT UNSIGNED NOT NULL,
    value    BLOB,
    PRIMARY KEY (id, cache_key)
) PARTITION BY LINEAR HASH (cache_key)
PARTITIONS 8;
```

## KEY Partitioning

Similar to HASH but MySQL chooses the hash function. Can partition on non-integer
columns and on the primary key by default.

```sql
-- KEY partitioning on primary key
CREATE TABLE messages (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    sender_id  INT UNSIGNED NOT NULL,
    body       TEXT NOT NULL,
    sent_at    DATETIME NOT NULL,
    PRIMARY KEY (id)
) PARTITION BY KEY ()
PARTITIONS 8;
-- Empty KEY() uses the primary key

-- KEY partitioning on specific column
CREATE TABLE user_data (
    user_id  INT UNSIGNED NOT NULL,
    data_key VARCHAR(50) NOT NULL,
    value    TEXT,
    PRIMARY KEY (user_id, data_key)
) PARTITION BY KEY (user_id)
PARTITIONS 16;
```

## Subpartitioning (Composite Partitioning)

```sql
-- RANGE-HASH composite: partition by date range, subpartition by hash
CREATE TABLE metrics (
    id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    metric_date DATE NOT NULL,
    host_id     INT UNSIGNED NOT NULL,
    value       DOUBLE NOT NULL,
    PRIMARY KEY (id, metric_date, host_id)
) PARTITION BY RANGE (YEAR(metric_date))
  SUBPARTITION BY HASH (host_id)
  SUBPARTITIONS 4 (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
-- Creates 4 partitions x 4 subpartitions = 16 physical partitions
```

## Partition Pruning

### How Pruning Works

```sql
-- This query prunes to only partition p2025
EXPLAIN SELECT * FROM sales WHERE sale_date = '2025-06-15';
-- EXPLAIN output shows: partitions: p2025

-- Range query prunes to p2024 and p2025
EXPLAIN SELECT * FROM sales
WHERE sale_date BETWEEN '2024-06-01' AND '2025-03-31';
-- partitions: p2024,p2025

-- This query CANNOT prune (function on partition column)
EXPLAIN SELECT * FROM sales WHERE MONTH(sale_date) = 6;
-- partitions: all partitions scanned (no pruning)
```

### EXPLAIN and Partitions

```sql
-- Always check partitions column in EXPLAIN
EXPLAIN SELECT * FROM audit_log
WHERE log_date >= '2025-03-01' AND log_date < '2025-04-01';
-- partitions: p202503

-- Extended EXPLAIN with partition details
EXPLAIN FORMAT=JSON SELECT * FROM sales
WHERE sale_date = '2025-01-15' AND customer_id = 42\G

-- Verify pruning is working as expected
EXPLAIN SELECT COUNT(*) FROM sales WHERE YEAR(sale_date) = 2025;
-- If function wraps partition column, pruning may be lost
-- Fix: use range condition instead
EXPLAIN SELECT COUNT(*) FROM sales
WHERE sale_date >= '2025-01-01' AND sale_date < '2026-01-01';
```

## Partition Maintenance Operations

### ADD PARTITION

```sql
-- Add a new partition (only works if no MAXVALUE partition exists,
-- or use REORGANIZE to split MAXVALUE)
ALTER TABLE sales ADD PARTITION (
    PARTITION p2027 VALUES LESS THAN (2028)
);

-- If MAXVALUE partition exists, reorganize it
ALTER TABLE sales REORGANIZE PARTITION pmax INTO (
    PARTITION p2027 VALUES LESS THAN (2028),
    PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

### DROP PARTITION (Instant Data Removal)

```sql
-- Drop an entire partition — instant, no row-by-row delete
ALTER TABLE sales DROP PARTITION p2022;
-- WARNING: This permanently removes all data in the partition

-- Compare with DELETE (much slower for large partitions)
-- DELETE FROM sales WHERE sale_date < '2023-01-01';  -- row-by-row, generates undo
```

### TRUNCATE PARTITION

```sql
-- Empty a partition without dropping its definition
ALTER TABLE sales TRUNCATE PARTITION p2023;
```

### REORGANIZE PARTITION

```sql
-- Split a partition into two
ALTER TABLE sales REORGANIZE PARTITION p2025 INTO (
    PARTITION p2025h1 VALUES LESS THAN (2025.5),  -- this doesn't work with YEAR()
    PARTITION p2025h2 VALUES LESS THAN (2026)
);

-- Merge multiple partitions into one
ALTER TABLE sales REORGANIZE PARTITION p2022, p2023 INTO (
    PARTITION p_archive VALUES LESS THAN (2024)
);
```

### EXCHANGE PARTITION

```sql
-- Swap a partition with a standalone table (instant, no data copy)
-- The standalone table must have the same structure, no partitioning
CREATE TABLE sales_archive_2022 LIKE sales;
ALTER TABLE sales_archive_2022 REMOVE PARTITIONING;

ALTER TABLE sales EXCHANGE PARTITION p2022 WITH TABLE sales_archive_2022;
-- p2022 is now empty; sales_archive_2022 has the data
```

### ANALYZE / REPAIR / OPTIMIZE PARTITION

```sql
-- Update statistics for specific partitions
ALTER TABLE sales ANALYZE PARTITION p2025, p2026;

-- Repair a specific partition
ALTER TABLE sales REPAIR PARTITION p2025;

-- Optimize (defragment) a specific partition
ALTER TABLE sales OPTIMIZE PARTITION p2025;
```

## Limitations and Constraints

### Unique Key Rule

Every unique key (including the primary key) must include ALL columns used in the
partition expression.

```sql
-- This FAILS: PK does not include partition column
CREATE TABLE bad_example (
    id        INT NOT NULL AUTO_INCREMENT PRIMARY KEY,  -- ERROR
    sale_date DATE NOT NULL,
    amount    DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2025 VALUES LESS THAN (2026)
);

-- This WORKS: PK includes partition column
CREATE TABLE good_example (
    id        INT NOT NULL AUTO_INCREMENT,
    sale_date DATE NOT NULL,
    amount    DECIMAL(10,2),
    PRIMARY KEY (id, sale_date)
) PARTITION BY RANGE (YEAR(sale_date)) (
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

### Other Limitations

```sql
-- Maximum 8192 partitions per table (including subpartitions)
-- Foreign keys are not supported on partitioned tables
-- FULLTEXT indexes are not supported on partitioned tables (before 8.0)
-- Spatial indexes are not supported on partitioned tables
-- Temporary tables cannot be partitioned
```

## When to Partition vs When Not To

### Good Candidates for Partitioning

- Time-series data with queries always filtering on date/time
- Tables over 100M rows where archival (DROP PARTITION) replaces DELETE
- Data with clear access patterns matching the partition scheme
- Tables where partition-wise operations (TRUNCATE, EXCHANGE) simplify admin

### Poor Candidates for Partitioning

- Tables under 10M rows (indexing alone is sufficient)
- Tables where queries rarely filter on the partition key
- Tables requiring foreign keys
- OLTP tables with point lookups on the primary key (partitioning adds overhead)
- Tables where the partition key is not part of common query patterns

## Best Practices

- Always include a MAXVALUE (RANGE) or DEFAULT (LIST in 8.0.28+) catch-all
  partition to prevent insert failures when data exceeds defined ranges.
- Verify partition pruning with EXPLAIN before going to production — if EXPLAIN
  shows all partitions, the query is not benefiting from partitioning.
- Use RANGE COLUMNS instead of RANGE with functions when possible — it supports
  direct comparison without function wrapping, enabling better pruning.
- Plan ahead for partition maintenance: automate ADD PARTITION for future periods
  and DROP PARTITION for archival.
- Keep the number of partitions reasonable (tens to low hundreds). Thousands of
  partitions increase metadata overhead and slow DDL.
- Use EXCHANGE PARTITION for fast archival — swap data to a non-partitioned archive
  table without copying rows.
- Never apply functions to the partition column in WHERE clauses — this defeats
  pruning. Use range comparisons instead.
- Test query plans with realistic data volumes; pruning behavior may differ between
  empty tables and populated tables.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Partitioning small tables (under 10M rows) | Adds overhead with no benefit; indexes handle these sizes efficiently | Only partition large tables with clear access patterns |
| Omitting partition column from primary key | CREATE TABLE fails with error 1503 | Include partition columns in all unique keys |
| Using functions on partition column in WHERE | Partition pruning is defeated; all partitions are scanned | Use range comparisons: `col >= X AND col < Y` |
| No MAXVALUE/catch-all partition | Inserts fail when data exceeds the highest defined range | Always add a MAXVALUE partition |
| Creating thousands of partitions | Excessive file handles, slow DDL, metadata overhead | Keep partitions to hundreds or fewer; use coarser granularity |
| Expecting partitioning to fix slow point lookups | Partitioning helps range scans and admin, not random PK lookups | Use proper indexing for point lookups; partition for range/admin benefits |

## MySQL Version Notes

- **5.7**: Maximum 8192 partitions. RANGE, LIST, HASH, KEY, and subpartitioning
  all supported. No DEFAULT partition for LIST. No native partition management
  events. Partitioned tables do not support foreign keys or spatial indexes.
  FULLTEXT indexes on partitioned tables require InnoDB.
- **8.0**: Same partition types supported. Native partitioning only (plugin-based
  partitioning removed). `ALTER TABLE ... EXCHANGE PARTITION` supports WITH/WITHOUT
  VALIDATION. Partitioned tables using InnoDB support FULLTEXT indexes.
  `INFORMATION_SCHEMA.PARTITIONS` and `SHOW CREATE TABLE` improvements.
  DEFAULT partition for LIST partitioning added in 8.0.28.
- **8.4 / 9.x**: Continued refinement. Check release notes for partition limit
  changes, new partition types, or DDL improvements.

## Sources

- [MySQL 8.0 Reference Manual: Partitioning](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)
- [MySQL 8.0: Partition Pruning](https://dev.mysql.com/doc/refman/8.0/en/partitioning-pruning.html)
- [MySQL 8.0: Partition Management](https://dev.mysql.com/doc/refman/8.0/en/partitioning-management.html)
- [MySQL 8.0: Partition Limitations](https://dev.mysql.com/doc/refman/8.0/en/partitioning-limitations.html)
- [MySQL 8.0: EXCHANGE PARTITION](https://dev.mysql.com/doc/refman/8.0/en/partitioning-management-exchange.html)
