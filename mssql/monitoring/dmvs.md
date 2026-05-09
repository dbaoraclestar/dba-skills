# Dynamic Management Views — Diagnostic Queries for SQL Server Performance

## Overview

Dynamic Management Views (DMVs) and Dynamic Management Functions (DMFs) are server-scoped or database-scoped views that return internal state information about the SQL Server instance. They are the primary tool for real-time performance diagnostics, replacing many older system tables and DBCC commands.

DMVs live in the `sys` schema and follow the naming convention `sys.dm_<category>_<detail>`. They require VIEW SERVER STATE permission (server-scoped) or VIEW DATABASE STATE permission (database-scoped). Unlike querying production tables, DMV queries read from in-memory structures and are generally lightweight, though some — like `sys.dm_db_index_physical_stats` — can be expensive on large databases.

Use DMVs when you need to answer questions like: What is running right now? What is consuming the most CPU? Which indexes are unused? Where are the I/O bottlenecks? They are the foundation of every SQL Server monitoring and troubleshooting workflow.

## Key Concepts

- **Server-scoped DMVs** — Return instance-wide data. Require VIEW SERVER STATE. Examples: `sys.dm_os_wait_stats`, `sys.dm_exec_requests`.
- **Database-scoped DMVs** — Return data for the current database context. Require VIEW DATABASE STATE. Examples: `sys.dm_db_index_usage_stats`.
- **Cumulative counters** — Many DMVs accumulate since the last SQL Server restart. Always record baselines and compute deltas.
- **Plan cache** — Execution statistics in `sys.dm_exec_query_stats` are tied to cached plans. Plan eviction resets those counters.
- **CROSS APPLY with DMFs** — Functions like `sys.dm_exec_sql_text()` and `sys.dm_exec_query_plan()` require CROSS APPLY to join with other DMVs.

## Execution DMVs

### Currently Running Queries (sys.dm_exec_requests)

```sql
SELECT
    r.session_id,
    r.status,
    r.command,
    r.wait_type,
    r.wait_time,
    r.blocking_session_id,
    r.cpu_time,
    r.total_elapsed_time,
    r.reads,
    r.writes,
    r.logical_reads,
    DB_NAME(r.database_id) AS database_name,
    t.text AS sql_text,
    SUBSTRING(t.text,
        (r.statement_start_offset / 2) + 1,
        (CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(t.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset) / 2 + 1
    ) AS current_statement,
    p.query_plan
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
CROSS APPLY sys.dm_exec_query_plan(r.plan_handle) p
WHERE r.session_id > 50   -- exclude system sessions
ORDER BY r.total_elapsed_time DESC;
```

### Active Sessions with Details (sys.dm_exec_sessions)

```sql
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    s.status,
    s.cpu_time,
    s.memory_usage * 8 AS memory_kb,
    s.reads,
    s.writes,
    s.logical_reads,
    s.last_request_start_time,
    s.last_request_end_time,
    c.client_net_address,
    c.auth_scheme
FROM sys.dm_exec_sessions s
LEFT JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
WHERE s.is_user_process = 1
ORDER BY s.cpu_time DESC;
```

### Top Queries by CPU (sys.dm_exec_query_stats)

```sql
SELECT TOP 20
    qs.total_worker_time / qs.execution_count AS avg_cpu_us,
    qs.total_worker_time AS total_cpu_us,
    qs.execution_count,
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_us,
    t.text AS sql_text,
    DB_NAME(t.dbid) AS database_name,
    qs.creation_time AS plan_cached_time,
    qs.last_execution_time
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) t
ORDER BY qs.total_worker_time DESC;
```

### Plan Cache Analysis (sys.dm_exec_cached_plans)

```sql
-- Plan cache composition
SELECT
    objtype AS object_type,
    cacheobjtype AS cache_type,
    COUNT(*) AS plan_count,
    SUM(CAST(size_in_bytes AS BIGINT)) / 1024 / 1024 AS size_mb,
    AVG(usecounts) AS avg_use_count
FROM sys.dm_exec_cached_plans
GROUP BY objtype, cacheobjtype
ORDER BY size_mb DESC;

-- Single-use ad hoc plans (plan cache bloat)
SELECT
    COUNT(*) AS single_use_plans,
    SUM(CAST(size_in_bytes AS BIGINT)) / 1024 / 1024 AS wasted_mb
FROM sys.dm_exec_cached_plans
WHERE usecounts = 1
  AND objtype = 'Adhoc';
```

## I/O DMVs

### File-Level I/O Statistics (sys.dm_io_virtual_file_stats)

```sql
SELECT
    DB_NAME(vfs.database_id) AS database_name,
    mf.name AS logical_file_name,
    mf.physical_name,
    mf.type_desc,
    vfs.num_of_reads,
    vfs.num_of_writes,
    vfs.num_of_bytes_read / 1024 / 1024 AS read_mb,
    vfs.num_of_bytes_written / 1024 / 1024 AS written_mb,
    vfs.io_stall_read_ms,
    vfs.io_stall_write_ms,
    CASE WHEN vfs.num_of_reads > 0
         THEN vfs.io_stall_read_ms / vfs.num_of_reads
         ELSE 0 END AS avg_read_latency_ms,
    CASE WHEN vfs.num_of_writes > 0
         THEN vfs.io_stall_write_ms / vfs.num_of_writes
         ELSE 0 END AS avg_write_latency_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) vfs
JOIN sys.master_files mf
    ON vfs.database_id = mf.database_id
   AND vfs.file_id = mf.file_id
ORDER BY vfs.io_stall DESC;
```

### Pending I/O Requests (sys.dm_io_pending_io_requests)

```sql
SELECT
    pio.io_type,
    pio.io_pending_ms_ticks,
    DB_NAME(vfs.database_id) AS database_name,
    mf.physical_name,
    mf.type_desc
FROM sys.dm_io_pending_io_requests pio
JOIN sys.dm_io_virtual_file_stats(NULL, NULL) vfs
    ON pio.io_handle = vfs.file_handle
JOIN sys.master_files mf
    ON vfs.database_id = mf.database_id
   AND vfs.file_id = mf.file_id
ORDER BY pio.io_pending_ms_ticks DESC;
```

## Index DMVs

### Index Usage Statistics (sys.dm_db_index_usage_stats)

```sql
-- Unused indexes (candidates for removal)
SELECT
    OBJECT_SCHEMA_NAME(i.object_id) AS schema_name,
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id
   AND i.index_id = ius.index_id
   AND ius.database_id = DB_ID()
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
  AND i.type_desc = 'NONCLUSTERED'
  AND (ius.user_seeks + ius.user_scans + ius.user_lookups) = 0
ORDER BY ius.user_updates DESC;
```

### Index Fragmentation (sys.dm_db_index_physical_stats)

```sql
-- WARNING: can be expensive on large databases; use LIMITED mode
SELECT
    OBJECT_SCHEMA_NAME(ps.object_id) AS schema_name,
    OBJECT_NAME(ps.object_id) AS table_name,
    i.name AS index_name,
    ps.index_type_desc,
    ps.avg_fragmentation_in_percent,
    ps.page_count,
    ps.avg_page_space_used_in_percent,
    ps.record_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ps
JOIN sys.indexes i
    ON ps.object_id = i.object_id
   AND ps.index_id = i.index_id
WHERE ps.avg_fragmentation_in_percent > 10
  AND ps.page_count > 1000
ORDER BY ps.avg_fragmentation_in_percent DESC;
```

### Index Operational Statistics (sys.dm_db_index_operational_stats)

```sql
SELECT
    OBJECT_NAME(os.object_id) AS table_name,
    i.name AS index_name,
    os.leaf_insert_count,
    os.leaf_update_count,
    os.leaf_delete_count,
    os.row_lock_count,
    os.row_lock_wait_count,
    os.row_lock_wait_in_ms,
    os.page_lock_count,
    os.page_lock_wait_count,
    os.page_latch_wait_count,
    os.page_latch_wait_in_ms
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) os
JOIN sys.indexes i
    ON os.object_id = i.object_id
   AND os.index_id = i.index_id
WHERE os.row_lock_wait_count > 0
ORDER BY os.row_lock_wait_in_ms DESC;
```

## OS DMVs

### Wait Statistics (sys.dm_os_wait_stats)

```sql
-- Top waits excluding benign idle waits
SELECT TOP 20
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    max_wait_time_ms,
    signal_wait_time_ms,
    wait_time_ms - signal_wait_time_ms AS resource_wait_time_ms,
    CAST(100.0 * wait_time_ms / SUM(wait_time_ms) OVER () AS DECIMAL(5,2)) AS pct
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK', 'BROKER_TO_FLUSH', 'SQLTRACE_BUFFER_FLUSH',
    'CLR_AUTO_EVENT', 'CLR_MANUAL_EVENT', 'LAZYWRITER_SLEEP',
    'CHECKPOINT_QUEUE', 'WAITFOR', 'XE_TIMER_EVENT',
    'XE_DISPATCHER_WAIT', 'FT_IFTS_SCHEDULER_IDLE_WAIT',
    'LOGMGR_QUEUE', 'DIRTY_PAGE_POLL', 'HADR_FILESTREAM_IOMGR_IOCOMPLETION',
    'SP_SERVER_DIAGNOSTICS_SLEEP', 'BROKER_EVENTHANDLER',
    'REQUEST_FOR_DEADLOCK_SEARCH', 'BROKER_TASK_STOP',
    'BROKER_RECEIVE_WAITFOR', 'CXCONSUMER'
)
AND waiting_tasks_count > 0
ORDER BY wait_time_ms DESC;
```

### Scheduler Health (sys.dm_os_schedulers)

```sql
SELECT
    scheduler_id,
    cpu_id,
    status,
    current_tasks_count,
    runnable_tasks_count,   -- > 0 indicates CPU pressure
    active_workers_count,
    work_queue_count,
    pending_disk_io_count
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE';
```

### Memory Clerks (sys.dm_os_memory_clerks)

```sql
SELECT TOP 15
    type AS clerk_type,
    SUM(pages_kb) / 1024 AS allocated_mb
FROM sys.dm_os_memory_clerks
GROUP BY type
ORDER BY SUM(pages_kb) DESC;
```

### Performance Counters (sys.dm_os_performance_counters)

```sql
-- Key counters at a glance
SELECT
    object_name,
    counter_name,
    instance_name,
    cntr_value
FROM sys.dm_os_performance_counters
WHERE counter_name IN (
    'Page life expectancy',
    'Buffer cache hit ratio',
    'Batch Requests/sec',
    'SQL Compilations/sec',
    'SQL Re-Compilations/sec',
    'User Connections',
    'Lock Waits/sec',
    'Deadlocks/sec',
    'Transactions/sec'
)
ORDER BY object_name, counter_name;
```

## Best Practices

- Always filter out system sessions (`session_id > 50`) when querying execution DMVs to avoid noise.
- Use `CROSS APPLY` (not `JOIN`) with DMFs like `sys.dm_exec_sql_text()` and `sys.dm_exec_query_plan()`.
- Remember that most DMVs reset on SQL Server restart; establish baselines and compute deltas for trending.
- Use `LIMITED` mode for `sys.dm_db_index_physical_stats` on production to reduce I/O overhead.
- Exclude well-known benign wait types from `sys.dm_os_wait_stats` analysis to focus on actionable waits.
- Check `runnable_tasks_count` in `sys.dm_os_schedulers` as the primary CPU pressure indicator.
- Query `sys.dm_exec_query_stats` for historical patterns; query `sys.dm_exec_requests` for what is happening right now.
- Combine multiple DMVs to build a complete picture (e.g., requests + sessions + connections).

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running `sys.dm_db_index_physical_stats` in DETAILED mode on large tables | Heavy I/O, long runtime, potential blocking | Use LIMITED or SAMPLED mode; schedule off-peak |
| Ignoring plan cache bloat from ad hoc queries | Excessive memory consumption for single-use plans | Enable `optimize for ad hoc workloads` server option |
| Not excluding idle wait types from wait stats analysis | Misleading top waits (e.g., SLEEP_TASK dominates) | Maintain an exclusion list of benign wait types |
| Treating DMV counters as point-in-time snapshots | Cumulative values since restart give no rate information | Capture two snapshots and compute the delta |
| Using `JOIN` instead of `CROSS APPLY` with DMFs | Query errors or incorrect results | Always use `CROSS APPLY` with `sys.dm_exec_sql_text()`, `sys.dm_exec_query_plan()` |
| Querying `sys.dm_db_index_usage_stats` right after a restart | All counters are zero; indexes appear unused | Wait for a representative workload cycle before drawing conclusions |

## SQL Server Version Notes

- **SQL Server 2016** — Introduced `sys.dm_exec_function_stats` for scalar UDF tracking. Query Store DMVs (`sys.query_store_*`) added as catalog views.
- **SQL Server 2017** — Added `sys.dm_db_log_info` replacing DBCC LOGINFO. `sys.dm_db_log_stats` for transaction log summary. Automatic tuning DMVs added.
- **SQL Server 2019** — Introduced `sys.dm_exec_query_plan_stats` for last actual execution plan (requires LAST_QUERY_PLAN_STATS database-scoped config). Intelligent Query Processing DMVs for adaptive joins and batch mode on rowstore. `sys.dm_db_page_info` replaces undocumented DBCC PAGE for page header inspection.
- **SQL Server 2022** — Added `sys.dm_exec_query_statistics_xml` enhancements, ledger-related DMVs, and Parameter Sensitive Plan optimization DMVs. `sys.dm_os_out_of_memory_events` tracks OOM conditions.

## Sources

- [sys.dm_exec_requests](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-requests-transact-sql)
- [sys.dm_os_wait_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
- [sys.dm_db_index_physical_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-physical-stats-transact-sql)
- [sys.dm_io_virtual_file_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-io-virtual-file-stats-transact-sql)
- [DMV Reference Guide](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/system-dynamic-management-views)
