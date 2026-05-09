# Index Strategy — Designing, Building, and Maintaining Effective MySQL Indexes

## Overview

Indexes are the single most impactful factor in MySQL query performance. A well-designed index can turn a multi-second full table scan into a millisecond lookup. A poorly designed indexing strategy wastes disk space, slows down writes, and provides no read benefit. Every DBA needs a systematic approach to index design, not ad-hoc "add an index when it's slow" reactions.

The core challenge of indexing is balancing read performance against write overhead. Every index must be maintained on every INSERT, UPDATE (of indexed columns), and DELETE. A table with 10 indexes will have significantly slower write throughput than the same table with 3 well-designed indexes. The goal is to have the minimum number of indexes that cover the maximum number of important query patterns.

Index strategy is not a one-time design activity. As query patterns evolve, data grows, and the application changes, indexes must be reviewed, added, and removed. MySQL 8.0's invisible indexes make this process safer by allowing you to test the impact of removing an index before actually dropping it.

## Key Concepts

**B-tree Index**: The default index type in InnoDB. Organizes data in a balanced tree structure allowing O(log n) lookups. Supports equality, range, prefix, and ordering operations. Every InnoDB table has a clustered index (the primary key); all other indexes are secondary and contain a pointer back to the primary key.

**Covering Index**: An index that contains all columns needed by a query, eliminating the need to access the base table (no "bookmark lookup"). Identified by "Using index" in the EXPLAIN Extra column.

**Selectivity / Cardinality**: Selectivity is the ratio of distinct values to total rows. High selectivity (close to 1.0, like a unique column) makes great index candidates. Low selectivity (like a boolean or status column with few distinct values) makes poor standalone index candidates.

**Leftmost Prefix Rule**: A composite index on (a, b, c) can satisfy queries on (a), (a, b), or (a, b, c), but NOT on (b), (c), or (b, c) alone. Column order in composite indexes is critical.

**Clustered vs Secondary Index**: In InnoDB, the primary key IS the table data (clustered index). Secondary indexes store the indexed columns plus the primary key value. A secondary index lookup that needs non-indexed columns requires a second lookup into the clustered index (bookmark lookup).

## B-tree Indexes (InnoDB Default)

### How B-tree Indexes Work

B-tree indexes organize data in sorted order within a balanced tree. Leaf nodes contain the indexed values and pointers. For InnoDB secondary indexes, the pointer is the primary key value.

```sql
-- Simple single-column index
CREATE INDEX idx_customer_email ON customers (email);

-- Composite index (multi-column)
CREATE INDEX idx_order_cust_date ON orders (customer_id, order_date);

-- Unique index (enforces uniqueness constraint)
CREATE UNIQUE INDEX idx_employee_ssn ON employees (ssn);

-- Prefix index (for long VARCHAR/TEXT columns)
CREATE INDEX idx_product_desc ON products (description(50));
```

### The Leftmost Prefix Rule

```sql
-- Given this composite index:
CREATE INDEX idx_composite ON orders (customer_id, status, order_date);

-- These queries CAN use the index:
SELECT * FROM orders WHERE customer_id = 100;                           -- uses (customer_id)
SELECT * FROM orders WHERE customer_id = 100 AND status = 'shipped';    -- uses (customer_id, status)
SELECT * FROM orders WHERE customer_id = 100 AND status = 'shipped'
  AND order_date > '2025-01-01';                                        -- uses all three columns

-- These queries CANNOT use this index:
SELECT * FROM orders WHERE status = 'shipped';                          -- skips customer_id
SELECT * FROM orders WHERE order_date > '2025-01-01';                   -- skips customer_id, status
SELECT * FROM orders WHERE status = 'shipped' AND order_date > '2025-01-01'; -- skips customer_id

-- Range condition stops further index column usage:
SELECT * FROM orders WHERE customer_id > 100 AND status = 'shipped';
-- Only uses customer_id (range); status column in index is NOT used for filtering
```

### Composite Index Column Order Guidelines

1. **Equality columns first**: Columns compared with `=` should come before range columns
2. **Higher selectivity first** (among equality columns): Put the most selective column first
3. **Range column last**: The column used with `>`, `<`, `BETWEEN`, `LIKE 'prefix%'` goes at the end
4. **ORDER BY alignment**: If possible, the ORDER BY columns should follow WHERE columns in index order

```sql
-- Query pattern: WHERE region = ? AND status = ? AND created_at > ? ORDER BY created_at
-- Best index:
CREATE INDEX idx_region_status_created ON orders (region, status, created_at);
-- region (equality) -> status (equality) -> created_at (range + order)
```

## Covering Indexes

A covering index contains all columns the query needs, eliminating table lookups entirely.

```sql
-- Query:
SELECT customer_id, order_date, total_amount
FROM orders
WHERE customer_id = 100 AND order_date >= '2025-01-01';

-- Covering index for this query:
CREATE INDEX idx_covering ON orders (customer_id, order_date, total_amount);

-- EXPLAIN will show "Using index" in Extra column (no table access needed)
```

```sql
-- Covering index for a COUNT query
-- Query: SELECT COUNT(*) FROM orders WHERE status = 'pending'
-- Any index starting with status will be covering for COUNT(*):
CREATE INDEX idx_status ON orders (status);
-- The index alone can satisfy the count without touching the table
```

## Hash Indexes (MEMORY Engine)

Hash indexes provide O(1) lookups for exact equality but do not support range queries, ordering, or partial key matching.

```sql
-- Only available on MEMORY (HEAP) engine tables
CREATE TABLE session_cache (
    session_id VARCHAR(64) NOT NULL,
    user_id INT NOT NULL,
    data TEXT,
    INDEX USING HASH (session_id)
) ENGINE = MEMORY;

-- Hash indexes ONLY support = and <=> operators
-- They do NOT support: >, <, BETWEEN, ORDER BY, LIKE
```

Note: InnoDB's Adaptive Hash Index (AHI) is an internal optimization that automatically builds hash indexes on frequently accessed B-tree index pages. It is not user-configurable at the index level.

## Fulltext Indexes

Fulltext indexes support natural language text search on CHAR, VARCHAR, and TEXT columns.

```sql
-- Create fulltext index
CREATE FULLTEXT INDEX idx_ft_product ON products (product_name, description);

-- Natural language search
SELECT product_id, product_name,
       MATCH(product_name, description) AGAINST ('wireless bluetooth headphones') AS relevance
FROM products
WHERE MATCH(product_name, description) AGAINST ('wireless bluetooth headphones')
ORDER BY relevance DESC
LIMIT 20;

-- Boolean mode search (more control)
SELECT product_id, product_name
FROM products
WHERE MATCH(product_name, description)
      AGAINST ('+wireless +bluetooth -wired' IN BOOLEAN MODE);

-- Minimum word length: ft_min_word_len (MyISAM) or innodb_ft_min_token_size (InnoDB, default 3)
-- Stopwords: innodb_ft_server_stopword_table or innodb_ft_user_stopword_table
```

## Spatial Indexes

For geometry data types using R-tree indexes.

```sql
CREATE TABLE stores (
    store_id INT PRIMARY KEY,
    name VARCHAR(100),
    location POINT NOT NULL SRID 4326,
    SPATIAL INDEX idx_location (location)
) ENGINE=InnoDB;

-- Find stores within 10 km of a point
SELECT store_id, name,
       ST_Distance_Sphere(location, ST_GeomFromText('POINT(-73.985 40.748)', 4326)) AS distance_m
FROM stores
WHERE ST_Contains(
    ST_Buffer(ST_GeomFromText('POINT(-73.985 40.748)', 4326), 0.09),
    location
)
ORDER BY distance_m
LIMIT 10;
```

## Index Merge

When no single index covers a query's WHERE clause, MySQL may merge results from multiple indexes.

```sql
-- Given separate indexes on customer_id and status:
CREATE INDEX idx_customer ON orders (customer_id);
CREATE INDEX idx_status ON orders (status);

EXPLAIN SELECT * FROM orders WHERE customer_id = 100 OR status = 'urgent';
-- type: index_merge
-- Extra: Using union(idx_customer, idx_status); Using where

-- Index merge types:
-- Union: OR conditions across different indexes
-- Intersection: AND conditions across different indexes
-- Sort-Union: OR conditions requiring sorting
```

Index merge is usually a sign that a composite index would be more efficient. If you see index_merge frequently for the same query pattern, create a proper composite index.

## Invisible Indexes (MySQL 8.0+)

Invisible indexes exist and are maintained but are ignored by the optimizer (unless `use_invisible_indexes` is enabled in the session). They allow safe testing of index removal.

```sql
-- Make an existing index invisible (test impact of removing it)
ALTER TABLE orders ALTER INDEX idx_old_status INVISIBLE;

-- Monitor for slow queries or plan regressions...
-- If no problems after sufficient observation:
DROP INDEX idx_old_status ON orders;

-- If problems appear, make it visible again:
ALTER TABLE orders ALTER INDEX idx_old_status VISIBLE;

-- Create a new index as invisible for testing
ALTER TABLE orders ADD INDEX idx_test_new (customer_id, region) INVISIBLE;

-- Test in your session only
SET SESSION optimizer_switch = 'use_invisible_indexes=on';
EXPLAIN SELECT * FROM orders WHERE customer_id = 100 AND region = 'US-EAST';
-- If the plan is good, make it visible globally
ALTER TABLE orders ALTER INDEX idx_test_new VISIBLE;
```

## Descending Indexes (MySQL 8.0+)

Before MySQL 8.0, index order specifications were parsed but ignored (all indexes were ascending). MySQL 8.0 supports actual descending indexes.

```sql
-- Mixed-order composite index for pagination patterns
CREATE INDEX idx_cust_date_desc ON orders (customer_id ASC, order_date DESC);

-- Efficient for: "most recent orders per customer"
SELECT * FROM orders
WHERE customer_id = 100
ORDER BY order_date DESC
LIMIT 20;
-- No "Backward index scan" needed; the index is natively in the right order

-- Before 8.0, this query would show "Backward index scan" or "Using filesort"
```

## Functional Indexes (MySQL 8.0.13+)

Index on an expression rather than a column value. Solves the classic "function on column prevents index usage" problem.

```sql
-- Problem: searching by lowercase email
SELECT * FROM customers WHERE LOWER(email) = 'john@example.com';
-- Cannot use a regular index on email column

-- Solution: functional index
CREATE INDEX idx_lower_email ON customers ((LOWER(email)));

-- Now the query uses the index:
EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'john@example.com';
-- type=ref, key=idx_lower_email

-- Index on JSON extracted value
CREATE INDEX idx_json_color ON products ((CAST(attributes->>'$.color' AS CHAR(30))));

SELECT * FROM products
WHERE CAST(attributes->>'$.color' AS CHAR(30)) = 'blue';

-- Index on computed date part
CREATE INDEX idx_order_year ON orders ((YEAR(order_date)));
SELECT * FROM orders WHERE YEAR(order_date) = 2025;
```

## Index Condition Pushdown (ICP)

ICP pushes part of the WHERE condition evaluation down to the storage engine layer, reducing the number of full row reads.

```sql
-- Given index: (customer_id, zipcode)
-- Query: WHERE customer_id > 100 AND zipcode = '90210'

-- Without ICP: InnoDB reads all rows with customer_id > 100, sends them to MySQL
--              server layer, which then filters by zipcode
-- With ICP:    InnoDB evaluates zipcode = '90210' during the index scan itself,
--              only reading full rows that match both conditions

-- ICP is shown as "Using index condition" in EXPLAIN Extra
-- Enabled by default. To disable (for testing):
SET optimizer_switch = 'index_condition_pushdown=off';
```

## When NOT to Index

Not every column benefits from an index. Avoid indexing when:

```sql
-- 1. Very low cardinality columns (unless part of a composite)
-- Boolean/flag columns: gender ENUM('M','F'), is_active TINYINT(1)
-- Standalone index on these is rarely useful (50% selectivity)

-- 2. Tables with very few rows (< 1000)
-- Full table scan is faster than index lookup for tiny tables

-- 3. Columns that are frequently updated
-- Each update to an indexed column requires index maintenance
-- Example: last_login_time, view_count, running_balance

-- 4. Wide columns (long VARCHAR/TEXT) without prefix indexes
-- Large index entries = fewer entries per page = deeper tree = more I/O

-- 5. Write-heavy tables where read patterns are unpredictable
-- Every index slows down INSERT/UPDATE/DELETE
```

## Analyzing Index Usage and Waste

### Finding Unused Indexes

```sql
-- Indexes that have never been used since last server restart
-- (Performance Schema must be enabled)
SELECT
    s.TABLE_SCHEMA,
    s.TABLE_NAME,
    s.INDEX_NAME,
    s.COLUMN_NAME,
    s.SEQ_IN_INDEX,
    t.TABLE_ROWS
FROM information_schema.STATISTICS s
JOIN information_schema.TABLES t
    ON s.TABLE_SCHEMA = t.TABLE_SCHEMA AND s.TABLE_NAME = t.TABLE_NAME
LEFT JOIN performance_schema.table_io_waits_summary_by_index_usage w
    ON s.TABLE_SCHEMA = w.OBJECT_SCHEMA
    AND s.TABLE_NAME = w.OBJECT_NAME
    AND s.INDEX_NAME = w.INDEX_NAME
WHERE w.INDEX_NAME IS NULL
  AND s.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema', 'sys')
  AND s.INDEX_NAME != 'PRIMARY'
ORDER BY t.TABLE_ROWS DESC;

-- Using sys schema (simpler)
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'sys');
```

### Finding Duplicate Indexes

```sql
-- Using sys schema
SELECT * FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'sys');

-- Example: idx_a(customer_id) is redundant if idx_ab(customer_id, order_date) exists
-- because idx_ab satisfies all queries that idx_a would serve
```

### Index Size Analysis

```sql
-- Index sizes per table
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_ROWS,
    ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_size_mb,
    ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_size_mb,
    ROUND(INDEX_LENGTH / NULLIF(DATA_LENGTH, 0) * 100, 1) AS index_to_data_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'performance_schema', 'information_schema', 'sys')
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY INDEX_LENGTH DESC
LIMIT 20;
```

## Cardinality and Selectivity Analysis

```sql
-- Check index cardinality (estimated distinct values)
SHOW INDEX FROM orders;

-- Detailed selectivity analysis
SELECT
    COLUMN_NAME,
    COUNT(DISTINCT customer_id) AS distinct_values,
    COUNT(*) AS total_rows,
    ROUND(COUNT(DISTINCT customer_id) / COUNT(*), 4) AS selectivity
FROM orders
GROUP BY 1;

-- Refresh cardinality statistics
ANALYZE TABLE orders;
```

## Histogram Statistics (MySQL 8.0+)

Histograms provide the optimizer with data distribution information without creating an index. Useful for columns with skewed distributions that appear in WHERE clauses but are not worth indexing.

```sql
-- Create histogram with 100 buckets (default)
ANALYZE TABLE orders UPDATE HISTOGRAM ON status, region WITH 100 BUCKETS;

-- View histogram data
SELECT
    SCHEMA_NAME,
    TABLE_NAME,
    COLUMN_NAME,
    HISTOGRAM->>'$."histogram-type"' AS histogram_type,
    JSON_LENGTH(HISTOGRAM, '$.buckets') AS num_buckets,
    HISTOGRAM->>'$."number-of-buckets-specified"' AS buckets_specified
FROM information_schema.COLUMN_STATISTICS;

-- Drop histogram
ANALYZE TABLE orders DROP HISTOGRAM ON status;
```

## Best Practices

- Design composite indexes based on actual query patterns, not individual columns
- Follow the equality-range-order rule for composite index column ordering
- Use covering indexes for the most critical queries to avoid table lookups
- Run ANALYZE TABLE after significant data changes to refresh cardinality estimates
- Use invisible indexes (8.0+) to safely test before adding or dropping indexes
- Audit indexes regularly: drop unused indexes and merge duplicate/redundant indexes
- Prefer composite indexes over relying on index merge optimization
- Use prefix indexes for long VARCHAR/TEXT columns, but test that the prefix length provides sufficient selectivity
- Keep primary keys short (INT/BIGINT) because every secondary index stores the PK value
- Consider the write cost: each additional index slows down INSERT/UPDATE/DELETE operations
- Use functional indexes (8.0+) instead of generated columns when you only need the index

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Creating single-column indexes for every column | Excessive write overhead, wasted space, optimizer confusion | Design composite indexes based on query patterns |
| Wrong column order in composite index | Index is useless for intended queries (violates leftmost prefix rule) | Put equality columns first, range columns last, match ORDER BY |
| Indexing low-cardinality columns standalone (e.g., `status`, `type`) | Index scan reads nearly as many rows as full table scan | Include in composite index with a selective leading column, or use histograms |
| Never dropping unused indexes | Write overhead on every DML, extra buffer pool memory consumption | Audit with `sys.schema_unused_indexes` quarterly; drop or make invisible |
| Prefix index too short on VARCHAR columns | Low selectivity, many false positives requiring table lookups | Test prefix lengths: `SELECT COUNT(DISTINCT LEFT(col,N))/COUNT(*)` for various N |
| Using UUID as primary key without ordering | Random inserts cause massive page splits, fragmentation, poor cache utilization | Use ordered UUIDs (UUID_TO_BIN with swap_flag), or use BIGINT AUTO_INCREMENT |
| Ignoring index size relative to buffer pool | Indexes larger than buffer pool cause constant I/O | Monitor index sizes; ensure working set fits in `innodb_buffer_pool_size` |

## MySQL Version Notes

**MySQL 5.7**:
- B-tree indexes with leftmost prefix rule
- Fulltext indexes on InnoDB (added in 5.6)
- No invisible indexes, no descending indexes, no functional indexes
- Index hints (USE/FORCE/IGNORE INDEX) available
- Optimizer can use index merge (union, intersection, sort-union)
- No histogram statistics

**MySQL 8.0**:
- Invisible indexes (8.0.0) for safe testing
- Descending indexes (8.0.0) for efficient reverse-order scans
- Functional indexes (8.0.13) for expression-based indexing
- Histogram statistics (8.0.0) for better cardinality estimates
- Hash joins for equi-joins without indexes (8.0.18+)
- `EXPLAIN ANALYZE` for actual vs estimated comparison (8.0.18+)
- `sys.schema_unused_indexes` and `sys.schema_redundant_indexes` views
- Multi-valued indexes for JSON arrays (8.0.17+)

**MySQL 8.4 / 9.x**:
- Improved optimizer cost model for index selection
- Better statistics for functional indexes
- Enhanced invisible index management
- Performance improvements for multi-valued JSON indexes
- Improved handling of index hints with new join types

## Sources

- [MySQL CREATE INDEX Statement](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)
- [MySQL InnoDB Index Types](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
- [MySQL Invisible Indexes](https://dev.mysql.com/doc/refman/8.0/en/invisible-indexes.html)
- [MySQL Descending Indexes](https://dev.mysql.com/doc/refman/8.0/en/descending-indexes.html)
- [MySQL Functional Key Parts](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-functional-key-parts)
- [MySQL Index Condition Pushdown](https://dev.mysql.com/doc/refman/8.0/en/index-condition-pushdown-optimization.html)
- [MySQL Histogram Statistics](https://dev.mysql.com/doc/refman/8.0/en/optimizer-statistics.html)
- [MySQL Multi-Valued Indexes](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-multi-valued)
