# Query Store -- Tracking, Analyzing, and Fixing Query Performance Regressions

## Overview

Query Store is SQL Server's built-in flight recorder for query performance. Introduced in SQL Server 2016, it captures a history of query text, execution plans, and runtime statistics directly inside each database. This gives DBAs and developers the ability to see how query performance has changed over time, identify plan regressions, and force known-good plans without modifying application code.

Before Query Store, diagnosing plan regressions required painstaking manual work: capturing plan cache snapshots, correlating DMV data, setting up Extended Events sessions, and hoping the evidence was still available when you needed it. Query Store eliminates this by continuously and persistently recording execution data that survives server restarts, failovers, and plan cache flushes.

Query Store is particularly valuable during SQL Server upgrades and compatibility level changes, where the new cardinality estimator may generate different (sometimes worse) plans. By enabling Query Store before the upgrade, you have a complete baseline to compare against and can force stable plans while you investigate regressions.

## Key Concepts

**Query**: A unique SQL statement identified by its query_hash. Query Store deduplicates identical text.

**Plan**: A compiled execution plan for a query. A single query can have multiple plans over time (due to recompilation, statistics changes, parameter sniffing).

**Runtime Statistics**: Per-plan execution metrics aggregated into configurable time intervals: duration, CPU time, logical reads, logical writes, execution count, memory grants, degree of parallelism.

**Plan Regression**: When the optimizer switches to a new plan that performs worse than a previous plan for the same query. Query Store detects this automatically.

**Forced Plan**: A plan that you manually lock to a query, overriding the optimizer's choice. Query Store enforces this until you unforce it.

**Capture Mode**: Controls which queries are recorded. ALL captures everything; AUTO filters out infrequent and trivial queries; CUSTOM (2022) gives fine-grained control.

**Stale Query Threshold**: How many days of data to retain. Older data is automatically purged.

## Enabling and Configuring Query Store

```sql
-- Enable Query Store on a database
ALTER DATABASE AdventureWorks SET QUERY_STORE = ON;

-- Recommended production configuration
ALTER DATABASE AdventureWorks SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 900,       -- flush to disk every 15 min
    INTERVAL_LENGTH_MINUTES = 60,            -- aggregate stats hourly
    MAX_STORAGE_SIZE_MB = 1024,              -- 1 GB storage cap
    QUERY_CAPTURE_MODE = AUTO,               -- skip trivial queries
    SIZE_BASED_CLEANUP_MODE = AUTO,          -- auto-purge when near limit
    MAX_PLANS_PER_QUERY = 200,               -- max plans tracked per query
    WAIT_STATS_CAPTURE_MODE = ON             -- capture wait categories (2017+)
);

-- Check current Query Store configuration
SELECT
    desired_state_desc,
    actual_state_desc,
    current_storage_size_mb,
    max_storage_size_mb,
    CAST(current_storage_size_mb * 100.0 / max_storage_size_mb AS DECIMAL(5,2))
        AS pct_used,
    readonly_reason,
    interval_length_minutes,
    stale_query_threshold_days,
    query_capture_mode_desc,
    size_based_cleanup_mode_desc,
    wait_stats_capture_mode_desc
FROM sys.database_query_store_options;

-- Check if Query Store has gone read-only (common problem)
-- readonly_reason: 0=none, 1=db in read-only, 2=db in single-user,
--                  4=db in emergency, 8=secondary replica,
--                  65536=size limit reached, 131072=retries exceeded,
--                  262144=internal error
SELECT actual_state_desc, readonly_reason
FROM sys.database_query_store_options;

-- Fix read-only Query Store (usually caused by hitting size limit)
ALTER DATABASE AdventureWorks SET QUERY_STORE (
    MAX_STORAGE_SIZE_MB = 2048,
    OPERATION_MODE = READ_WRITE
);
```

## Top Resource-Consuming Queries

```sql
-- Find the top 20 queries by total CPU time
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    SUM(rs.count_executions) AS total_executions,
    SUM(rs.avg_duration * rs.count_executions) / 1000 AS total_duration_ms,
    SUM(rs.avg_cpu_time * rs.count_executions) / 1000 AS total_cpu_ms,
    SUM(rs.avg_logical_io_reads * rs.count_executions) AS total_logical_reads,
    SUM(rs.avg_logical_io_writes * rs.count_executions) AS total_logical_writes,
    COUNT(DISTINCT p.plan_id) AS plan_count
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(DAY, -7, GETUTCDATE())
GROUP BY q.query_id, qt.query_sql_text
ORDER BY total_cpu_ms DESC;

-- Find queries with the highest average duration
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    p.plan_id,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.avg_cpu_time / 1000 AS avg_cpu_ms,
    rs.avg_logical_io_reads,
    rs.avg_physical_io_reads,
    rs.avg_query_max_used_memory * 8 AS avg_memory_grant_kb,
    rs.count_executions,
    p.is_forced_plan,
    p.last_execution_time
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(HOUR, -24, GETUTCDATE())
    AND rs.count_executions > 10
ORDER BY rs.avg_duration DESC;
```

## Plan Regression Detection

```sql
-- Find queries with plan regressions: new plan is worse than old plan
-- Compare average duration across plans for the same query
WITH PlanPerformance AS (
    SELECT
        q.query_id,
        p.plan_id,
        p.last_execution_time,
        AVG(rs.avg_duration) AS avg_duration,
        AVG(rs.avg_cpu_time) AS avg_cpu,
        AVG(rs.avg_logical_io_reads) AS avg_reads,
        SUM(rs.count_executions) AS total_executions,
        ROW_NUMBER() OVER (PARTITION BY q.query_id ORDER BY p.last_execution_time DESC) AS rn_newest,
        ROW_NUMBER() OVER (PARTITION BY q.query_id ORDER BY AVG(rs.avg_duration) ASC) AS rn_best
    FROM sys.query_store_query AS q
    JOIN sys.query_store_plan AS p ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats AS rs ON p.plan_id = rs.plan_id
    JOIN sys.query_store_runtime_stats_interval AS rsi
        ON rs.runtime_stats_interval_id = rsi.runtime_stats_interval_id
    WHERE rsi.start_time >= DATEADD(DAY, -7, GETUTCDATE())
    GROUP BY q.query_id, p.plan_id, p.last_execution_time
)
SELECT
    current_plan.query_id,
    qt.query_sql_text,
    current_plan.plan_id AS current_plan_id,
    current_plan.avg_duration / 1000 AS current_avg_ms,
    best_plan.plan_id AS best_plan_id,
    best_plan.avg_duration / 1000 AS best_avg_ms,
    CAST((current_plan.avg_duration - best_plan.avg_duration) * 100.0
        / NULLIF(best_plan.avg_duration, 0) AS DECIMAL(10,2)) AS regression_pct
FROM PlanPerformance AS current_plan
JOIN PlanPerformance AS best_plan
    ON current_plan.query_id = best_plan.query_id
    AND best_plan.rn_best = 1
JOIN sys.query_store_query AS q ON current_plan.query_id = q.query_id
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
WHERE current_plan.rn_newest = 1
    AND current_plan.plan_id <> best_plan.plan_id
    AND current_plan.avg_duration > best_plan.avg_duration * 1.5  -- 50%+ regression
ORDER BY regression_pct DESC;

-- Use built-in regression detection (2017+)
-- Automatic tuning: automatically force the last known good plan
ALTER DATABASE AdventureWorks SET AUTOMATIC_TUNING (
    FORCE_LAST_GOOD_PLAN = ON
);

-- Check automatic tuning recommendations
SELECT
    reason,
    score,
    JSON_VALUE(details, '$.implementationDetails.script') AS script,
    JSON_VALUE(state, '$.currentValue') AS current_state,
    JSON_VALUE(state, '$.reason') AS state_reason
FROM sys.dm_db_tuning_recommendations;
```

## Forcing and Unforcing Plans

```sql
-- Force a specific plan for a query
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 87;

-- Unforce a plan
EXEC sp_query_store_unforce_plan @query_id = 42, @plan_id = 87;

-- View all currently forced plans
SELECT
    q.query_id,
    qt.query_sql_text,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.count_executions,
    p.last_execution_time
FROM sys.query_store_plan AS p
JOIN sys.query_store_query AS q ON p.query_id = q.query_id
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
OUTER APPLY (
    SELECT TOP 1 avg_duration, count_executions
    FROM sys.query_store_runtime_stats
    WHERE plan_id = p.plan_id
    ORDER BY last_execution_time DESC
) AS rs
WHERE p.is_forced_plan = 1;

-- Check for force failures (plan can't be forced due to schema changes)
SELECT
    q.query_id,
    p.plan_id,
    p.force_failure_count,
    p.last_force_failure_reason_desc
FROM sys.query_store_plan AS p
JOIN sys.query_store_query AS q ON p.query_id = q.query_id
WHERE p.is_forced_plan = 1
    AND p.force_failure_count > 0;
```

## A/B Plan Comparison

```sql
-- Compare performance of two plans for the same query
DECLARE @query_id INT = 42;
DECLARE @plan_a INT = 87;
DECLARE @plan_b INT = 91;

SELECT
    p.plan_id,
    AVG(rs.avg_duration) / 1000 AS avg_duration_ms,
    AVG(rs.avg_cpu_time) / 1000 AS avg_cpu_ms,
    AVG(rs.avg_logical_io_reads) AS avg_reads,
    AVG(rs.avg_logical_io_writes) AS avg_writes,
    AVG(rs.avg_physical_io_reads) AS avg_physical_reads,
    AVG(rs.avg_query_max_used_memory) * 8 AS avg_memory_grant_kb,
    AVG(rs.avg_dop) AS avg_dop,
    SUM(rs.count_executions) AS total_executions,
    MIN(rs.min_duration) / 1000 AS min_duration_ms,
    MAX(rs.max_duration) / 1000 AS max_duration_ms,
    STDEV(rs.avg_duration) / 1000 AS stdev_duration_ms
FROM sys.query_store_plan AS p
JOIN sys.query_store_runtime_stats AS rs ON p.plan_id = rs.plan_id
WHERE p.query_id = @query_id
    AND p.plan_id IN (@plan_a, @plan_b)
GROUP BY p.plan_id;

-- View the actual plan XML for comparison
SELECT
    p.plan_id,
    TRY_CAST(p.query_plan AS XML) AS query_plan_xml
FROM sys.query_store_plan AS p
WHERE p.query_id = @query_id
    AND p.plan_id IN (@plan_a, @plan_b);
```

## Wait Statistics in Query Store (2017+)

```sql
-- Per-query wait category breakdown
SELECT TOP 20
    q.query_id,
    qt.query_sql_text,
    ws.wait_category_desc,
    SUM(ws.avg_query_wait_time_ms * ws.total_query_wait_time_ms)
        AS total_wait_impact,
    AVG(ws.avg_query_wait_time_ms) AS avg_wait_ms,
    SUM(ws.total_query_wait_time_ms) AS total_wait_ms
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p ON q.query_id = p.query_id
JOIN sys.query_store_wait_stats AS ws ON p.plan_id = ws.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi
    ON ws.runtime_stats_interval_id = rsi.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(DAY, -1, GETUTCDATE())
GROUP BY q.query_id, qt.query_sql_text, ws.wait_category_desc
ORDER BY total_wait_ms DESC;

-- Wait categories: CPU, Lock, Latch, BufferLatch, BufferIO,
-- Compilation, SqlClr, Mirroring, Transaction, Idle,
-- Preemptive, ServiceBroker, TranLogIO, NetworkIO,
-- Parallelism, Memory, UserWait, Tracing, FullTextSearch,
-- OtherDiskIO, Replication, LogRateGovernor (Azure)
```

## Query Store Hints (SQL Server 2022)

```sql
-- Apply query-level hints without modifying application code
-- Equivalent to adding OPTION() hints but managed externally

-- Force MAXDOP for a specific query
EXEC sys.sp_query_store_set_hints
    @query_id = 42,
    @query_hints = N'OPTION (MAXDOP 4)';

-- Force RECOMPILE to avoid parameter sniffing
EXEC sys.sp_query_store_set_hints
    @query_id = 55,
    @query_hints = N'OPTION (RECOMPILE)';

-- Force a specific join order
EXEC sys.sp_query_store_set_hints
    @query_id = 78,
    @query_hints = N'OPTION (FORCE ORDER)';

-- Combine multiple hints
EXEC sys.sp_query_store_set_hints
    @query_id = 42,
    @query_hints = N'OPTION (MAXDOP 2, RECOMPILE)';

-- View active hints
SELECT
    query_hint_id,
    query_id,
    query_hint_text,
    source_desc,     -- 'User' or 'CE_feedback' etc.
    last_query_hint_failure_reason_desc
FROM sys.query_store_query_hints;

-- Remove a hint
EXEC sys.sp_query_store_clear_hints @query_id = 42;
```

## Custom Capture Policies (SQL Server 2022)

```sql
-- CUSTOM capture mode for fine-grained control
ALTER DATABASE AdventureWorks SET QUERY_STORE (
    QUERY_CAPTURE_MODE = CUSTOM,
    QUERY_CAPTURE_POLICY = (
        STALE_CAPTURE_POLICY_THRESHOLD = 24 HOURS,
        EXECUTION_COUNT = 30,                -- min executions before capture
        TOTAL_COMPILE_CPU_TIME_MS = 1000,    -- min compile CPU
        TOTAL_EXECUTION_CPU_TIME_MS = 100    -- min execution CPU
    )
);
-- This captures only queries that either:
-- Execute 30+ times, OR
-- Use 1000ms+ compile CPU, OR
-- Use 100ms+ execution CPU
-- within the stale threshold window
```

## Data Management and Purging

```sql
-- Manually purge all Query Store data
ALTER DATABASE AdventureWorks SET QUERY_STORE CLEAR;

-- Remove data for a specific query
EXEC sp_query_store_remove_query @query_id = 42;

-- Remove a specific plan
EXEC sp_query_store_remove_plan @plan_id = 87;

-- Check Query Store space usage by component
SELECT
    'Query Text' AS component,
    COUNT(*) AS item_count,
    SUM(DATALENGTH(query_sql_text)) / 1024 / 1024 AS size_mb
FROM sys.query_store_query_text
UNION ALL
SELECT
    'Plans',
    COUNT(*),
    SUM(DATALENGTH(TRY_CAST(query_plan AS NVARCHAR(MAX)))) / 1024 / 1024
FROM sys.query_store_plan
UNION ALL
SELECT
    'Runtime Stats',
    COUNT(*),
    NULL  -- size not directly available
FROM sys.query_store_runtime_stats;

-- Flush in-memory Query Store data to disk immediately
EXEC sp_query_store_flush_db;
```

## Query Store for Upgrade Safety

```sql
-- BEFORE upgrading compatibility level: establish baseline
ALTER DATABASE AdventureWorks SET QUERY_STORE = ON;
ALTER DATABASE AdventureWorks SET QUERY_STORE (
    QUERY_CAPTURE_MODE = ALL,  -- capture everything during baseline
    INTERVAL_LENGTH_MINUTES = 30
);

-- Run representative workload for several days to build baseline

-- Upgrade compatibility level
ALTER DATABASE AdventureWorks SET COMPATIBILITY_LEVEL = 160;  -- SQL 2022

-- Monitor for regressions after upgrade
SELECT
    q.query_id,
    qt.query_sql_text,
    p_old.plan_id AS old_plan_id,
    p_new.plan_id AS new_plan_id,
    rs_old.avg_duration / 1000 AS old_avg_ms,
    rs_new.avg_duration / 1000 AS new_avg_ms,
    CAST((rs_new.avg_duration - rs_old.avg_duration) * 100.0
        / NULLIF(rs_old.avg_duration, 0) AS DECIMAL(10,2)) AS regression_pct
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p_old ON q.query_id = p_old.query_id
JOIN sys.query_store_plan AS p_new ON q.query_id = p_new.query_id
JOIN sys.query_store_runtime_stats AS rs_old ON p_old.plan_id = rs_old.plan_id
JOIN sys.query_store_runtime_stats AS rs_new ON p_new.plan_id = rs_new.plan_id
WHERE p_old.compatibility_level < 160
    AND p_new.compatibility_level = 160
    AND rs_new.avg_duration > rs_old.avg_duration * 1.2
ORDER BY regression_pct DESC;

-- Force old plan if regression is confirmed
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 87;
```

## Best Practices

- Enable Query Store on all production databases -- the overhead is minimal (typically 1-3% CPU) and the diagnostic value is enormous
- Set MAX_STORAGE_SIZE_MB appropriately (1-4 GB for most databases) and enable SIZE_BASED_CLEANUP_MODE = AUTO to prevent going read-only
- Use QUERY_CAPTURE_MODE = AUTO for production to skip trivial queries; use ALL only during short baseline captures
- Monitor actual_state_desc and readonly_reason regularly -- a read-only Query Store silently stops collecting data
- Enable AUTOMATIC_TUNING (FORCE_LAST_GOOD_PLAN = ON) on SQL Server 2017+ for automatic plan regression mitigation
- Enable Query Store before any compatibility level upgrade, migration, or major schema change as a safety net
- Use INTERVAL_LENGTH_MINUTES = 60 for production (less granular but less overhead); 15-30 for targeted troubleshooting
- Flush Query Store data before planned failovers to avoid data loss: EXEC sp_query_store_flush_db
- Use Query Store hints (2022) instead of modifying application SQL to add OPTION hints
- Review forced plans periodically -- they may become suboptimal as data distribution changes

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not monitoring Query Store actual_state | Store silently goes read-only; stops collecting data; missed regressions | Alert on readonly_reason != 0; auto-remediate with size increase |
| Setting MAX_STORAGE_SIZE_MB too low | Query Store hits limit, goes read-only, purges valuable history | Set 1-4 GB for typical databases; monitor pct_used |
| Using QUERY_CAPTURE_MODE = ALL in production permanently | Excessive overhead from capturing trivial queries; bloated storage | Use AUTO for ongoing production; ALL only for short baseline captures |
| Forcing plans without monitoring force_failure_count | Schema changes invalidate forced plans; queries silently use bad plans | Monitor is_forced_plan + force_failure_count; re-evaluate after schema changes |
| Clearing Query Store data during an active investigation | Destroys the historical evidence needed for regression analysis | Export data before clearing; use remove_query for surgical cleanup |
| Not enabling Query Store before a version upgrade | No baseline for plan regression detection; flying blind | Enable 2+ weeks before any upgrade or compatibility level change |
| Setting INTERVAL_LENGTH_MINUTES too small (1 minute) | Excessive storage consumption and overhead from high-frequency aggregation | Use 60 minutes for production; 15-30 minutes for short investigations |

## SQL Server Version Notes

**SQL Server 2016**: Query Store introduced. Basic capture modes (ALL, AUTO, NONE). Plan forcing. Available in Standard Edition with SP1. Not enabled by default.

**SQL Server 2017**: Wait statistics capture in Query Store. Automatic tuning (FORCE_LAST_GOOD_PLAN). sys.dm_db_tuning_recommendations DMV. Interleaved execution and adaptive joins tracked.

**SQL Server 2019**: Query Store enabled by default for new databases (in Azure SQL Database). Optimized plan forcing. Custom capture policies (preview). Memory grant feedback persistence in Query Store.

**SQL Server 2022**: Query Store enabled by default for all new databases. Query Store hints (sp_query_store_set_hints). CUSTOM capture mode (GA). Optimized plan forcing improvements. Query Store for read replicas. CE feedback, DOP feedback, and memory grant feedback integrated with Query Store. Query Store link for hybrid scenarios.

## Sources

- [Query Store Overview (Microsoft Docs)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Best Practices for Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store)
- [Automatic Tuning](https://learn.microsoft.com/en-us/sql/relational-databases/automatic-tuning/automatic-tuning)
- [Query Store Hints (2022)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-hints)
- [Upgrading Databases with Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-usage-scenarios#CEUpgrade)
- [SQL Server Query Store Overview](https://www.sqlshack.com/sql-server-query-store-overview/)
- [The Query Store in Action](https://www.sqlshack.com/the-sql-server-query-store-in-action/)
- [Use Cases for Query Store](https://www.sqlshack.com/use-cases-for-query-store-in-sql-server/)
- [Performance Monitoring via Query Store](https://www.sqlshack.com/performance-monitoring-via-sql-server-query-store/)
- [Query Store: Your Database's Flight Recorder](https://www.sqlshack.com/query-store-your-databases-flight-recorder/)
- [Tune Performance with Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/tune-performance-with-the-query-store)
- [Query Store Usage Scenarios](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-usage-scenarios)
- [Query Store Hints Best Practices](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-hints-best-practices)
