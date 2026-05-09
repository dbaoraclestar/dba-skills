# Statistics Management -- Maintaining Accurate Statistics for Optimal Query Plans

## Overview

Statistics are metadata objects that describe the distribution of values in columns and indexes. The SQL Server query optimizer relies on statistics to estimate cardinality -- the number of rows each operator in an execution plan will produce. Accurate cardinality estimates lead to optimal plans; stale or missing statistics lead to poor plans, wrong join strategies, incorrect memory grants, and degraded performance.

Every index automatically has associated statistics, and SQL Server can auto-create statistics on non-indexed columns referenced in query predicates. However, automatic maintenance has limitations. The default auto-update threshold can leave statistics stale on large tables, sampling rates may be insufficient for skewed distributions, and filtered or partitioned workloads may need manual statistics strategies.

Understanding how statistics work, when they become stale, and how to manage them is a foundational performance tuning skill. Many "mysterious" performance regressions trace back to outdated statistics causing the optimizer to choose plans based on wildly inaccurate row estimates.

## Key Concepts

**Statistics Object**: A metadata structure containing a histogram (up to 200 steps showing value distribution), density vector (average uniqueness of column combinations), and header information (rows sampled, last update time).

**Histogram**: A 200-step approximation of value distribution for the first column of a statistics object. Each step defines a range boundary (RANGE_HI_KEY), the number of rows equal to that boundary (EQ_ROWS), the number of rows within the range (RANGE_ROWS), and the number of distinct values in the range (DISTINCT_RANGE_ROWS).

**Density Vector**: Contains the average density (1 / distinct_values) for each column prefix combination. Used for GROUP BY and multi-column equality estimates.

**Auto-Create Statistics**: When enabled (default), SQL Server automatically creates single-column statistics on columns used in predicates that lack a useful index.

**Auto-Update Statistics**: When enabled (default), SQL Server automatically updates statistics when approximately 20% of rows have changed (or using a dynamic threshold with trace flag 2371 / 2016+ behavior).

**Sampling**: Statistics can be built by scanning the full table (FULLSCAN) or sampling a percentage of rows. Higher sampling yields more accurate histograms at the cost of longer update time.

**Cardinality Estimator (CE)**: The optimizer component that uses statistics to predict row counts. SQL Server has two versions: the legacy CE (pre-2014) and the new CE (2014+), each with different assumptions and estimation models.

## Auto-Create and Auto-Update Configuration

```sql
-- Check auto statistics settings for a database
SELECT
    name,
    is_auto_create_stats_on,
    is_auto_update_stats_on,
    is_auto_update_stats_async_on,
    is_auto_create_stats_incremental_on  -- 2014+
FROM sys.databases
WHERE name = DB_NAME();

-- Enable auto-create and auto-update (should be ON for most databases)
ALTER DATABASE AdventureWorks SET AUTO_CREATE_STATISTICS ON;
ALTER DATABASE AdventureWorks SET AUTO_UPDATE_STATISTICS ON;

-- Enable asynchronous auto-update
-- Query uses stale stats immediately; update happens in background
-- Prevents stats update from blocking query compilation
ALTER DATABASE AdventureWorks SET AUTO_UPDATE_STATISTICS_ASYNC ON;

-- Auto-update threshold behavior:
-- Default (pre-2016 without TF 2371):
--   Table rows < 500: update after 500 modifications
--   Table rows >= 500: update after 20% of rows modified
--
-- Dynamic threshold (2016+ default, or TF 2371 on older versions):
--   SQRT(1000 * table_rows) modifications trigger update
--   For a 1M row table: ~31,623 changes vs 200,000 with the 20% rule
--   Much more responsive on large tables

-- Enable dynamic threshold on pre-2016 instances
DBCC TRACEON(2371, -1);

-- Check if TF 2371 is active
DBCC TRACESTATUS(2371);
```

## Manual Statistics Updates

```sql
-- Update statistics for a specific table (all stats on the table)
UPDATE STATISTICS Sales.Orders;

-- Update a specific statistics object
UPDATE STATISTICS Sales.Orders IX_Orders_CustomerID;

-- Update with FULLSCAN (most accurate, slowest)
UPDATE STATISTICS Sales.Orders WITH FULLSCAN;

-- Update with a specific sample rate
UPDATE STATISTICS Sales.Orders WITH SAMPLE 50 PERCENT;

-- Update with row count sample
UPDATE STATISTICS Sales.Orders WITH SAMPLE 100000 ROWS;

-- Update all statistics in the database
EXEC sp_updatestats;

-- Update only statistics that need it (based on modification counter)
-- sp_updatestats uses default sampling
EXEC sp_updatestats 'resample';  -- uses previous sample rate

-- Update statistics with NO RECOMPUTE
-- Prevents auto-update from overriding your manual update
UPDATE STATISTICS Sales.Orders IX_Orders_CustomerID WITH FULLSCAN, NORECOMPUTE;

-- Re-enable auto-update after NORECOMPUTE was set
UPDATE STATISTICS Sales.Orders IX_Orders_CustomerID WITH RESAMPLE;
```

## Viewing Statistics Properties

```sql
-- List all statistics objects on a table
SELECT
    s.name AS stats_name,
    s.auto_created,
    s.user_created,
    s.no_recompute,
    s.is_temporary,
    s.is_incremental,
    s.has_filter,
    s.filter_definition,
    STATS_DATE(s.object_id, s.stats_id) AS last_updated,
    sp.rows AS table_rows,
    sp.rows_sampled,
    CAST(sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(5,2)) AS sample_pct,
    sp.modification_counter,
    sp.persisted_sample_percent  -- 2016 SP1+
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID('Sales.Orders')
ORDER BY sp.modification_counter DESC;

-- Find stale statistics across the database
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stats_name,
    sp.rows AS table_rows,
    sp.rows_sampled,
    sp.modification_counter,
    CAST(sp.modification_counter * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(10,2))
        AS pct_modified,
    STATS_DATE(s.object_id, s.stats_id) AS last_updated,
    DATEDIFF(DAY, STATS_DATE(s.object_id, s.stats_id), GETDATE()) AS days_since_update
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE sp.modification_counter > 0
    AND OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
ORDER BY sp.modification_counter DESC;
```

## DBCC SHOW_STATISTICS Output

```sql
-- Full SHOW_STATISTICS output (header + density vector + histogram)
DBCC SHOW_STATISTICS('Sales.Orders', 'IX_Orders_CustomerID');

-- Header only: metadata about the statistics object
DBCC SHOW_STATISTICS('Sales.Orders', 'IX_Orders_CustomerID') WITH STAT_HEADER;
-- Key fields:
-- Updated:        When stats were last updated
-- Rows:           Total rows in table at update time
-- Rows Sampled:   Rows actually read to build histogram
-- Steps:          Number of histogram steps (max 200)
-- Average key length: Helps estimate index size
-- String Index:   YES if histogram has string summary stats

-- Density vector: selectivity for column combinations
DBCC SHOW_STATISTICS('Sales.Orders', 'IX_Orders_CustomerID') WITH DENSITY_VECTOR;
-- Density = 1 / distinct_values
-- Used for GROUP BY and equality on non-leading columns
-- Lower density = more selective = better for seeks

-- Histogram: value distribution for the leading column
DBCC SHOW_STATISTICS('Sales.Orders', 'IX_Orders_CustomerID') WITH HISTOGRAM;
-- Columns:
-- RANGE_HI_KEY:      Upper bound of the histogram step
-- RANGE_ROWS:        Rows between previous and current RANGE_HI_KEY
-- EQ_ROWS:           Rows exactly equal to RANGE_HI_KEY
-- DISTINCT_RANGE_ROWS: Distinct values in the range
-- AVG_RANGE_ROWS:    Average rows per distinct value in range

-- Example: Interpreting histogram for query estimation
-- For: WHERE CustomerID = 1001
-- 1. Find the step containing 1001
-- 2. If 1001 IS the RANGE_HI_KEY: estimated rows = EQ_ROWS
-- 3. If 1001 is WITHIN a range: estimated rows = AVG_RANGE_ROWS
-- 4. If 1001 is BEYOND the histogram: use density * total_rows
```

## Filtered Statistics

```sql
-- Create filtered statistics for a subset of data
-- Useful when value distribution varies significantly across segments
CREATE STATISTICS STAT_Orders_ActiveCustomers
ON Sales.Orders (OrderDate, TotalAmount)
WHERE Status = 'Active';

-- Create filtered statistics for skewed data
-- E.g., 95% of rows have Country = 'US' but queries often target other countries
CREATE STATISTICS STAT_Orders_NonUS
ON Sales.Orders (CustomerID, OrderDate)
WHERE Country <> 'US';

-- Filtered statistics are auto-updated independently
-- View filter definition
SELECT
    name,
    filter_definition,
    STATS_DATE(object_id, stats_id) AS last_updated
FROM sys.stats
WHERE object_id = OBJECT_ID('Sales.Orders')
    AND has_filter = 1;

-- Limitations:
-- 1. Cannot reference computed columns
-- 2. Cannot use complex expressions (only simple comparisons)
-- 3. Filtered stats may not be used with parameterized queries
--    unless the predicate matches exactly
```

## Incremental Statistics on Partitioned Tables

```sql
-- Incremental statistics: maintain per-partition stats independently
-- Only available on partitioned tables, introduced in SQL Server 2014
-- Avoids full table scan when only one partition has changed

-- Enable incremental statistics at the database level
ALTER DATABASE AdventureWorks
SET AUTO_CREATE_STATISTICS ON (INCREMENTAL = ON);

-- Create incremental statistics on a partitioned table
CREATE STATISTICS STAT_FactSales_OrderDate
ON dw.FactSales (OrderDate)
WITH INCREMENTAL = ON;

-- Update only the modified partitions
UPDATE STATISTICS dw.FactSales STAT_FactSales_OrderDate
WITH RESAMPLE ON PARTITIONS (5, 6);
-- Only partitions 5 and 6 are rescanned; others retain existing stats

-- Check incremental statistics details
SELECT
    s.name,
    s.is_incremental,
    isp.partition_number,
    isp.rows,
    isp.rows_sampled,
    isp.modification_counter,
    isp.last_updated
FROM sys.stats AS s
CROSS APPLY sys.dm_db_incremental_stats_properties(s.object_id, s.stats_id) AS isp
WHERE s.object_id = OBJECT_ID('dw.FactSales')
    AND s.is_incremental = 1
ORDER BY isp.partition_number;
```

## Statistics Sampling Strategies

```sql
-- FULLSCAN: Reads every row. Most accurate. Use for:
-- - Critical tables driving complex queries
-- - Columns with highly skewed distribution
-- - After bulk loads or data migrations
UPDATE STATISTICS Sales.Orders IX_Orders_CustomerID WITH FULLSCAN;

-- SAMPLE n PERCENT: Reads a random sample. Use for:
-- - Very large tables where FULLSCAN takes too long
-- - Routine maintenance where approximate stats are acceptable
UPDATE STATISTICS Sales.Orders IX_Orders_CustomerID WITH SAMPLE 25 PERCENT;

-- Persisted sample percent (2016 SP1+):
-- Lock in a sample rate so auto-update uses it instead of default
-- Useful when you know a specific table needs higher than default sampling
UPDATE STATISTICS Sales.Orders WITH FULLSCAN, PERSIST_SAMPLE_PERCENT = ON;
-- Future auto-updates will use 100% (FULLSCAN) for this table

-- Check persisted sample settings
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stats_name,
    sp.persisted_sample_percent
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE s.object_id = OBJECT_ID('Sales.Orders');

-- Parallel statistics update (2022+):
-- Large table stats updates can use parallelism
-- Controlled by MAXDOP option
UPDATE STATISTICS Sales.Orders WITH FULLSCAN, MAXDOP = 8;
```

## Cardinality Estimator Versions

```sql
-- Check which CE version a database uses
-- Compatibility level determines the default CE:
-- 110 (2012) and below: Legacy CE
-- 120 (2014) and above: New CE
SELECT
    name,
    compatibility_level,
    CASE
        WHEN compatibility_level < 120 THEN 'Legacy CE'
        ELSE 'New CE (2014+)'
    END AS default_ce_version
FROM sys.databases
WHERE name = DB_NAME();

-- Force legacy CE for a specific query (when new CE gives bad estimates)
SELECT CustomerID, COUNT(*)
FROM Sales.Orders
WHERE OrderDate > '2024-01-01'
GROUP BY CustomerID
OPTION (USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION'));

-- Force legacy CE database-wide (via scoped configuration)
ALTER DATABASE SCOPED CONFIGURATION SET LEGACY_CARDINALITY_ESTIMATION = ON;

-- Key differences between Legacy CE and New CE:
-- 1. Multi-predicate correlation assumption:
--    Legacy: predicates are independent (multiply selectivities)
--    New: assumes some correlation (exponential backoff formula)
--
-- 2. Join estimation:
--    Legacy: uses histogram alignment
--    New: uses simple containment assumption
--
-- 3. Ascending key problem:
--    Legacy: estimates 1 row for values beyond histogram max
--    New: uses density to estimate rows beyond histogram max

-- Use Query Store to compare plans under different CE versions
-- The query_id is the same; plan_id changes with CE version

-- CE feedback (2022): Automatically adjusts CE model per query
-- when initial estimates are significantly wrong
SELECT
    q.query_id,
    f.feature_desc,       -- 'CE_feedback'
    f.feedback_data,      -- JSON with CE adjustments
    f.state_desc          -- 'Active', 'Expired', etc.
FROM sys.query_store_query AS q
JOIN sys.query_store_plan_feedback AS f ON q.query_id = f.query_id
WHERE f.feature_desc = 'CE_feedback';
```

## Async Statistics Update

```sql
-- Normal auto-update (synchronous): Query compilation blocks until
-- statistics are updated. Can cause compilation delays.

-- Async auto-update: Query compiles with stale stats immediately.
-- Stats update happens in background. Next compilation uses fresh stats.
-- Tradeoff: One bad plan vs compilation delay.

ALTER DATABASE AdventureWorks SET AUTO_UPDATE_STATISTICS_ASYNC ON;

-- When to use async updates:
-- 1. OLTP systems where compilation latency matters
-- 2. Tables that change rapidly but queries can tolerate one stale plan
-- 3. When you see WAIT_ON_SYNC_STATISTICS_REFRESH waits (2019+)

-- Monitor async stats update activity
SELECT
    counter_name,
    cntr_value
FROM sys.dm_os_performance_counters
WHERE counter_name IN (
    'Auto-Update Stats Async',
    'Auto-Update Stats'
);

-- Check for pending async stats updates
SELECT
    OBJECT_NAME(s.object_id) AS table_name,
    s.name AS stats_name,
    sp.modification_counter,
    sp.rows,
    STATS_DATE(s.object_id, s.stats_id) AS last_updated
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
WHERE sp.modification_counter >
    CASE WHEN sp.rows < 500 THEN 500
         ELSE SQRT(1000 * sp.rows)  -- dynamic threshold approximation
    END
ORDER BY sp.modification_counter DESC;
```

## Automated Statistics Maintenance Script

```sql
-- Targeted statistics update for stale stats only
-- Uses dynamic threshold logic to find what truly needs updating
DECLARE @sql NVARCHAR(MAX) = '';

SELECT @sql = @sql +
    'UPDATE STATISTICS ' +
    QUOTENAME(SCHEMA_NAME(o.schema_id)) + '.' + QUOTENAME(OBJECT_NAME(s.object_id)) +
    ' ' + QUOTENAME(s.name) +
    CASE
        WHEN sp.rows < 100000 THEN ' WITH FULLSCAN;' + CHAR(13)
        WHEN sp.rows < 1000000 THEN ' WITH SAMPLE 50 PERCENT;' + CHAR(13)
        ELSE ' WITH SAMPLE 25 PERCENT;' + CHAR(13)
    END
FROM sys.stats AS s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) AS sp
JOIN sys.objects AS o ON s.object_id = o.object_id
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND sp.modification_counter > SQRT(1000 * sp.rows)
    AND sp.rows > 0
ORDER BY sp.modification_counter DESC;

PRINT @sql;
-- EXEC sp_executesql @sql;  -- Uncomment to execute
```

## Best Practices

- Keep AUTO_CREATE_STATISTICS and AUTO_UPDATE_STATISTICS enabled on all production databases -- they are essential for the optimizer
- Use FULLSCAN on critical tables with skewed distributions where default sampling produces inaccurate histograms
- Enable trace flag 2371 on pre-2016 instances (or verify dynamic threshold is active on 2016+) to get timely updates on large tables
- Consider AUTO_UPDATE_STATISTICS_ASYNC on OLTP systems to prevent stats updates from blocking query compilation
- Monitor modification_counter via sys.dm_db_stats_properties to find statistics approaching the update threshold
- Use filtered statistics for columns with highly non-uniform distribution across logical partitions of data
- Use incremental statistics on partitioned tables to avoid full table scans during statistics maintenance
- Update statistics after large data loads, ETL operations, or partition switches -- do not wait for auto-update
- Persist sample percent (2016 SP1+) on tables that need higher-than-default sampling for accurate estimates
- Test both legacy CE and new CE when upgrading -- the new CE is generally better but can regress on specific query patterns

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Disabling AUTO_UPDATE_STATISTICS globally | Statistics become stale; optimizer uses increasingly wrong estimates | Re-enable auto-update; use NORECOMPUTE only on specific stats with manual maintenance |
| Using only default sampling on skewed data | Histogram misses rare values; severe cardinality estimation errors | Use FULLSCAN or higher sample rates; create filtered statistics |
| Ignoring modification_counter on large tables | 20% threshold (pre-2016) allows millions of changes before update | Enable TF 2371 or upgrade to 2016+; monitor modification_counter proactively |
| Running UPDATE STATISTICS with NORECOMPUTE and forgetting | Statistics freeze permanently; no auto-update; slow degradation over time | Audit for no_recompute=1 regularly; remove NORECOMPUTE or add manual job |
| Updating statistics immediately before every query | Excessive compilation overhead; stats update becomes the bottleneck | Trust auto-update for routine changes; manual update after bulk operations only |
| Not updating stats after bulk load / partition switch | Optimizer has no knowledge of new data; wildly wrong cardinality estimates | Always UPDATE STATISTICS after ETL, bulk insert, or partition switch operations |
| Using the wrong cardinality estimator version | Regressions when upgrading compat level; unexplained plan changes | Test with both CE versions; use database scoped configuration or query hints to override |

## SQL Server Version Notes

**SQL Server 2014**: New Cardinality Estimator introduced (compatibility level 120). Trace flag 2312 forces new CE; TF 9481 forces legacy CE. Incremental statistics for partitioned tables.

**SQL Server 2016**: Dynamic auto-update threshold enabled by default (equivalent to TF 2371). Persisted sample percent (SP1). sys.dm_db_stats_properties improvements. WAIT_ON_SYNC_STATISTICS_REFRESH visible in wait stats. Parallel CREATE STATISTICS.

**SQL Server 2019**: Lightweight statistics profiling on by default. Improved auto-update behavior for temporal tables. Approximate statistics (APPROX_PERCENTILE_CONT/DISC). sys.dm_db_stats_histogram DMF for querying histograms without DBCC.

**SQL Server 2022**: Cardinality Estimation feedback (automatic CE model adjustments per query). Parallel UPDATE STATISTICS with MAXDOP option. Improved statistics on ascending keys. sys.query_store_plan_feedback for CE feedback tracking. Statistics auto-update uses a more intelligent sampling strategy.

## Sources

- [Statistics Overview (Microsoft Docs)](https://learn.microsoft.com/en-us/sql/relational-databases/statistics/statistics)
- [UPDATE STATISTICS (T-SQL Reference)](https://learn.microsoft.com/en-us/sql/t-sql/statements/update-statistics-transact-sql)
- [DBCC SHOW_STATISTICS](https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-show-statistics-transact-sql)
- [Cardinality Estimation (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/cardinality-estimation-sql-server)
- [sys.dm_db_stats_properties](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-stats-properties-transact-sql)
- [Incremental Statistics](https://learn.microsoft.com/en-us/sql/relational-databases/statistics/statistics#incremental-statistics)
