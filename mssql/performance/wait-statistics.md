# Wait Statistics -- Diagnosing SQL Server Bottlenecks Through Wait Analysis

## Overview

Wait statistics are the cornerstone of SQL Server performance troubleshooting methodology. Every thread in SQL Server is either running on a CPU scheduler, waiting for a resource, or waiting in a queue for CPU time. When a thread cannot proceed because it needs a resource (a lock, a disk page, a memory grant, a network buffer), it records the type of wait and the duration. These accumulated wait statistics reveal exactly where your SQL Server instance is spending time waiting rather than doing useful work.

The wait statistics methodology, popularized by Paul Randal and the SQLskills team, inverts the traditional approach to performance tuning. Instead of guessing at bottlenecks and applying random fixes, you start by asking: "What is SQL Server waiting on most?" The answer directs your investigation to the actual bottleneck -- whether it is disk I/O, CPU pressure, lock contention, memory pressure, or client application delays.

SQL Server exposes wait statistics at multiple granularities: instance-wide cumulative waits (sys.dm_os_wait_stats), per-session waits (sys.dm_exec_session_wait_stats), and per-query waits (via Query Store or Extended Events). Combining these views gives you both the big picture and the ability to drill down to specific problem queries.

## Key Concepts

**Wait Type**: A named category describing what resource a thread is waiting for (e.g., PAGEIOLATCH_SH, LCK_M_X, CXPACKET).

**Wait Time**: Total elapsed time (in milliseconds) that threads have spent waiting for a particular resource. This is the sum of resource wait time and signal wait time.

**Resource Wait Time**: Time spent waiting for the actual resource to become available (e.g., waiting for a disk read to complete).

**Signal Wait Time**: Time spent waiting for a CPU scheduler after the resource became available. High signal waits indicate CPU pressure -- threads get their resource but must queue for CPU time.

**Waiter List**: The internal queue where waiting threads are parked. Each resource type has its own waiter list management.

**Benign Waits**: Background system waits that occur even on idle servers (e.g., WAITFOR, LAZYWRITER_SLEEP, BROKER_EVENTHANDLER). These should be filtered out of analysis.

## Instance-Wide Wait Statistics

### Capturing and Analyzing Cumulative Waits

```sql
-- Core wait statistics query with benign waits filtered out
-- This is the starting point for any performance investigation
WITH WaitStats AS (
    SELECT
        wait_type,
        waiting_tasks_count,
        wait_time_ms,
        max_wait_time_ms,
        signal_wait_time_ms,
        wait_time_ms - signal_wait_time_ms AS resource_wait_time_ms,
        100.0 * wait_time_ms / SUM(wait_time_ms) OVER() AS pct_total_wait,
        ROW_NUMBER() OVER (ORDER BY wait_time_ms DESC) AS rn
    FROM sys.dm_os_wait_stats
    WHERE wait_type NOT IN (
        -- Filter benign/background waits
        N'BROKER_EVENTHANDLER', N'BROKER_RECEIVE_WAITFOR',
        N'BROKER_TASK_STOP', N'BROKER_TO_FLUSH',
        N'BROKER_TRANSMITTER', N'CHECKPOINT_QUEUE',
        N'CHKPT', N'CLR_AUTO_EVENT', N'CLR_MANUAL_EVENT',
        N'CLR_SEMAPHORE', N'DBMIRROR_DBM_EVENT',
        N'DBMIRROR_EVENTS_QUEUE', N'DBMIRROR_WORKER_QUEUE',
        N'DBMIRRORING_CMD', N'DIRTY_PAGE_POLL',
        N'DISPATCHER_QUEUE_SEMAPHORE', N'EXECSYNC',
        N'FSAGENT', N'FT_IFTS_SCHEDULER_IDLE_WAIT',
        N'FT_IFTSHC_MUTEX', N'HADR_CLUSAPI_CALL',
        N'HADR_FILESTREAM_IOMGR_IOCOMPLETION',
        N'HADR_LOGCAPTURE_WAIT', N'HADR_NOTIFICATION_DEQUEUE',
        N'HADR_TIMER_TASK', N'HADR_WORK_QUEUE',
        N'KSOURCE_WAKEUP', N'LAZYWRITER_SLEEP',
        N'LOGMGR_QUEUE', N'MEMORY_ALLOCATION_EXT',
        N'ONDEMAND_TASK_QUEUE', N'PARALLEL_REDO_DRAIN_WORKER',
        N'PARALLEL_REDO_LOG_CACHE', N'PARALLEL_REDO_TRAN_LIST',
        N'PARALLEL_REDO_WORKER_SYNC',
        N'PARALLEL_REDO_WORKER_WAIT_WORK',
        N'PREEMPTIVE_OS_FLUSHFILEBUFFERS',
        N'PREEMPTIVE_XE_GETTARGETSTATE',
        N'PVS_PREALLOCATE',
        N'PWAIT_ALL_COMPONENTS_INITIALIZED',
        N'PWAIT_DIRECTLOGCONSUMER_GETNEXT',
        N'QDS_PERSIST_TASK_MAIN_LOOP_SLEEP',
        N'QDS_ASYNC_QUEUE',
        N'QDS_CLEANUP_STALE_QUERIES_TASK_MAIN_LOOP_SLEEP',
        N'QDS_SHUTDOWN_QUEUE',
        N'REDO_THREAD_PENDING_WORK',
        N'REQUEST_FOR_DEADLOCK_SEARCH',
        N'RESOURCE_QUEUE', N'SERVER_IDLE_CHECK',
        N'SLEEP_BPOOL_FLUSH', N'SLEEP_DBSTARTUP',
        N'SLEEP_DCOMSTARTUP', N'SLEEP_MASTERDBREADY',
        N'SLEEP_MASTERMDREADY', N'SLEEP_MASTERUPGRADED',
        N'SLEEP_MSDBSTARTUP', N'SLEEP_SYSTEMTASK',
        N'SLEEP_TASK', N'SLEEP_TEMPDBSTARTUP',
        N'SNI_HTTP_ACCEPT', N'SOS_WORK_DISPATCHER',
        N'SP_SERVER_DIAGNOSTICS_SLEEP',
        N'SQLTRACE_BUFFER_FLUSH',
        N'SQLTRACE_INCREMENTAL_FLUSH_SLEEP',
        N'SQLTRACE_WAIT_ENTRIES',
        N'WAIT_FOR_RESULTS', N'WAITFOR',
        N'WAITFOR_TASKSHUTDOWN',
        N'WAIT_XTP_CKPT_CLOSE',
        N'WAIT_XTP_HOST_WAIT',
        N'WAIT_XTP_OFFLINE_CKPT_NEW_LOG',
        N'WAIT_XTP_RECOVERY',
        N'XE_BUFFERMGR_ALLPROCESSED_EVENT',
        N'XE_DISPATCHER_JOIN', N'XE_DISPATCHER_WAIT',
        N'XE_TIMER_EVENT'
    )
    AND waiting_tasks_count > 0
)
SELECT
    wait_type,
    waiting_tasks_count,
    CAST(wait_time_ms / 1000.0 AS DECIMAL(18,2)) AS wait_time_sec,
    CAST(resource_wait_time_ms / 1000.0 AS DECIMAL(18,2)) AS resource_wait_sec,
    CAST(signal_wait_time_ms / 1000.0 AS DECIMAL(18,2)) AS signal_wait_sec,
    CAST(pct_total_wait AS DECIMAL(5,2)) AS pct_total,
    CAST(SUM(pct_total_wait) OVER (ORDER BY wait_time_ms DESC) AS DECIMAL(5,2)) AS running_pct,
    CAST(wait_time_ms / waiting_tasks_count AS DECIMAL(18,2)) AS avg_wait_ms
FROM WaitStats
WHERE rn <= 20
ORDER BY wait_time_ms DESC;
```

### Snapshot-Based Wait Analysis

```sql
-- Take before/after snapshots to measure waits over a specific interval
-- Step 1: Capture baseline
IF OBJECT_ID('tempdb..#wait_baseline') IS NOT NULL DROP TABLE #wait_baseline;
SELECT wait_type, waiting_tasks_count, wait_time_ms, signal_wait_time_ms
INTO #wait_baseline
FROM sys.dm_os_wait_stats
WHERE waiting_tasks_count > 0;

-- Step 2: Wait for your interval (or run the problem workload)
WAITFOR DELAY '00:05:00';  -- 5-minute sample

-- Step 3: Calculate delta
SELECT
    w.wait_type,
    w.waiting_tasks_count - ISNULL(b.waiting_tasks_count, 0) AS delta_tasks,
    w.wait_time_ms - ISNULL(b.wait_time_ms, 0) AS delta_wait_ms,
    w.signal_wait_time_ms - ISNULL(b.signal_wait_time_ms, 0) AS delta_signal_ms,
    (w.wait_time_ms - ISNULL(b.wait_time_ms, 0)) -
    (w.signal_wait_time_ms - ISNULL(b.signal_wait_time_ms, 0)) AS delta_resource_ms
FROM sys.dm_os_wait_stats AS w
LEFT JOIN #wait_baseline AS b ON w.wait_type = b.wait_type
WHERE (w.wait_time_ms - ISNULL(b.wait_time_ms, 0)) > 0
ORDER BY delta_wait_ms DESC;

-- Reset wait statistics (useful before a targeted test)
DBCC SQLPERF('sys.dm_os_wait_stats', CLEAR);
```

## Per-Session and Per-Query Waits

```sql
-- Per-session waits (SQL Server 2016+)
-- Shows waits accumulated by a specific session since it connected
SELECT
    session_id,
    wait_type,
    waiting_tasks_count,
    wait_time_ms,
    signal_wait_time_ms,
    wait_time_ms - signal_wait_time_ms AS resource_wait_ms
FROM sys.dm_exec_session_wait_stats
WHERE session_id = @@SPID   -- or a specific session_id
ORDER BY wait_time_ms DESC;

-- Correlate waits with active queries
SELECT
    r.session_id,
    r.wait_type,
    r.wait_time,
    r.wait_resource,
    r.status,
    r.command,
    r.cpu_time,
    r.total_elapsed_time,
    r.logical_reads,
    SUBSTRING(st.text, (r.statement_start_offset / 2) + 1,
        ((CASE r.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE r.statement_end_offset
        END - r.statement_start_offset) / 2) + 1) AS current_statement,
    r.blocking_session_id
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st
WHERE r.session_id > 50  -- exclude system sessions
    AND r.status = 'suspended'  -- actively waiting
ORDER BY r.wait_time DESC;
```

## Common Wait Types Deep Dive

### CXPACKET and CXCONSUMER (Parallelism)

```sql
-- CXPACKET: A parallel worker thread is waiting for other threads to finish
-- CXCONSUMER (2016 SP2+): The consumer (gathering) thread waits for producers
-- These are NORMAL in parallel plans. High waits indicate skewed parallelism.

-- Check current MAXDOP setting
SELECT name, value_in_use
FROM sys.configurations
WHERE name = 'max degree of parallelism';

-- Check cost threshold for parallelism
SELECT name, value_in_use
FROM sys.configurations
WHERE name = 'cost threshold for parallelism';

-- Recommended: Set MAXDOP per Microsoft guidelines
-- NUMA node-aware; typically 8 or fewer for OLTP
EXEC sp_configure 'max degree of parallelism', 8;
RECONFIGURE;

-- Raise cost threshold from default 5 to reduce trivial parallelism
EXEC sp_configure 'cost threshold for parallelism', 50;
RECONFIGURE;
```

### PAGEIOLATCH_SH and PAGEIOLATCH_EX (Disk I/O)

```sql
-- PAGEIOLATCH_SH: Waiting for a data page to be read from disk (shared latch)
-- PAGEIOLATCH_EX: Waiting for a page read for modification (exclusive latch)
-- Root cause: Insufficient memory (buffer pool too small) or slow storage

-- Check buffer pool hit ratio
SELECT
    CAST((a.cntr_value * 1.0 / b.cntr_value) * 100.0 AS DECIMAL(5,2))
        AS buffer_cache_hit_ratio
FROM sys.dm_os_performance_counters a
JOIN sys.dm_os_performance_counters b
    ON a.object_name = b.object_name
WHERE a.counter_name = 'Buffer cache hit ratio'
    AND b.counter_name = 'Buffer cache hit ratio base';

-- Find which databases/files have the most I/O waits
SELECT
    DB_NAME(vfs.database_id) AS database_name,
    mf.physical_name,
    vfs.io_stall_read_ms,
    vfs.num_of_reads,
    CAST(vfs.io_stall_read_ms / NULLIF(vfs.num_of_reads, 0) AS DECIMAL(10,2))
        AS avg_read_latency_ms,
    vfs.io_stall_write_ms,
    vfs.num_of_writes,
    CAST(vfs.io_stall_write_ms / NULLIF(vfs.num_of_writes, 0) AS DECIMAL(10,2))
        AS avg_write_latency_ms
FROM sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN sys.master_files AS mf
    ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id
ORDER BY vfs.io_stall_read_ms + vfs.io_stall_write_ms DESC;
```

### LCK_M_* (Lock Contention)

```sql
-- LCK_M_S:  Waiting for shared lock (SELECT blocked by exclusive)
-- LCK_M_X:  Waiting for exclusive lock (UPDATE/DELETE blocked)
-- LCK_M_U:  Waiting for update lock
-- LCK_M_IX: Waiting for intent exclusive lock
-- LCK_M_SCH_M: Waiting for schema modification lock (DDL blocked)

-- Find current blocking chains
SELECT
    r.session_id AS blocked_spid,
    r.blocking_session_id AS blocker_spid,
    r.wait_type,
    r.wait_time / 1000 AS wait_sec,
    r.wait_resource,
    DB_NAME(r.database_id) AS database_name,
    blocked_text.text AS blocked_query,
    blocker_text.text AS blocker_query
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS blocked_text
LEFT JOIN sys.dm_exec_requests AS br ON r.blocking_session_id = br.session_id
OUTER APPLY sys.dm_exec_sql_text(br.sql_handle) AS blocker_text
WHERE r.blocking_session_id > 0
ORDER BY r.wait_time DESC;

-- Check lock escalation settings
SELECT
    OBJECT_NAME(object_id) AS table_name,
    lock_escalation_desc
FROM sys.tables
WHERE lock_escalation_desc <> 'TABLE';
```

### ASYNC_NETWORK_IO (Client Delays)

```sql
-- SQL Server has results ready but the client is not consuming them fast enough
-- Common causes: Client-side row-by-row processing, network latency, SSMS rendering

-- Find sessions with ASYNC_NETWORK_IO waits
SELECT
    r.session_id,
    r.wait_type,
    r.wait_time,
    s.host_name,
    s.program_name,
    s.login_name,
    st.text AS query_text
FROM sys.dm_exec_requests AS r
JOIN sys.dm_exec_sessions AS s ON r.session_id = s.session_id
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS st
WHERE r.wait_type = 'ASYNC_NETWORK_IO';
```

### SOS_SCHEDULER_YIELD (CPU Pressure)

```sql
-- Thread voluntarily yields the CPU scheduler after its quantum expires
-- High SOS_SCHEDULER_YIELD = CPU-bound workload

-- Check CPU utilization history from ring buffer
SELECT TOP 30
    record_id,
    dateadd(ms, -1 * (ts_now - [timestamp]), GETDATE()) AS event_time,
    SQLProcessUtilization AS sql_cpu_pct,
    SystemIdle AS idle_pct,
    100 - SystemIdle - SQLProcessUtilization AS other_process_cpu_pct
FROM (
    SELECT
        record.value('(./Record/@id)[1]', 'int') AS record_id,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS SystemIdle,
        record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS SQLProcessUtilization,
        [timestamp],
        ts_now
    FROM (
        SELECT [timestamp],
            CONVERT(XML, record) AS record,
            cpu_ticks / (cpu_ticks / ms_ticks) AS ts_now
        FROM sys.dm_os_ring_buffers
        CROSS JOIN sys.dm_os_sys_info
        WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
    ) AS x
) AS y
ORDER BY record_id DESC;

-- Check scheduler health
SELECT
    scheduler_id,
    cpu_id,
    current_tasks_count,
    runnable_tasks_count,  -- > 0 indicates CPU pressure
    active_workers_count,
    work_queue_count
FROM sys.dm_os_schedulers
WHERE status = 'VISIBLE ONLINE';
```

### WRITELOG (Transaction Log Writes)

```sql
-- Waiting for transaction log to flush to disk
-- Every COMMIT must wait for log hardening (write-ahead logging)

-- Check transaction log I/O performance
SELECT
    DB_NAME(database_id) AS db_name,
    io_stall_write_ms / NULLIF(num_of_writes, 0) AS avg_log_write_latency_ms,
    num_of_writes,
    num_of_bytes_written / 1048576 AS log_mb_written
FROM sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN sys.master_files AS mf
    ON vfs.database_id = mf.database_id AND vfs.file_id = mf.file_id
WHERE mf.type_desc = 'LOG'
ORDER BY avg_log_write_latency_ms DESC;
```

### RESOURCE_SEMAPHORE (Memory Grants)

```sql
-- Queries waiting for memory grants before they can execute
-- Typically from large sorts, hash joins, or hash aggregates

-- Check pending memory grants
SELECT
    session_id,
    request_time,
    grant_time,
    requested_memory_kb,
    granted_memory_kb,
    required_memory_kb,
    used_memory_kb,
    max_used_memory_kb,
    query_cost,
    dop,
    wait_order,
    is_next_candidate
FROM sys.dm_exec_query_memory_grants
ORDER BY requested_memory_kb DESC;

-- Check memory grant resource semaphore status
SELECT
    pool_id,
    resource_semaphore_id,
    target_memory_kb / 1024 AS target_mb,
    max_target_memory_kb / 1024 AS max_target_mb,
    total_memory_kb / 1024 AS total_mb,
    available_memory_kb / 1024 AS available_mb,
    granted_memory_kb / 1024 AS granted_mb,
    used_memory_kb / 1024 AS used_mb,
    grantee_count,
    waiter_count
FROM sys.dm_exec_query_resource_semaphores;
```

## Signal Waits Analysis

```sql
-- Signal wait percentage indicates CPU pressure
-- Signal wait > 15-20% of total wait = significant CPU queuing
SELECT
    CAST(SUM(signal_wait_time_ms) AS DECIMAL(18,2)) AS total_signal_wait_ms,
    CAST(SUM(wait_time_ms) AS DECIMAL(18,2)) AS total_wait_ms,
    CAST(100.0 * SUM(signal_wait_time_ms) / SUM(wait_time_ms) AS DECIMAL(5,2))
        AS signal_wait_pct
FROM sys.dm_os_wait_stats
WHERE wait_time_ms > 0;
```

## Extended Events for Wait Capture

```sql
-- Create an Extended Events session to capture waits per query
CREATE EVENT SESSION [QueryWaits] ON SERVER
ADD EVENT sqlos.wait_completed (
    SET collect_wait_resource = (1)
    ACTION (
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.query_hash,
        sqlserver.database_name
    )
    WHERE ([duration] > 10000)  -- waits longer than 10ms
        AND ([opcode] = (1))    -- end of wait only
        AND sqlserver.session_id > 50
)
ADD TARGET package0.event_file (
    SET filename = N'QueryWaits', max_file_size = 100
)
WITH (MAX_MEMORY = 8192 KB, MAX_DISPATCH_LATENCY = 5 SECONDS);

ALTER EVENT SESSION [QueryWaits] ON SERVER STATE = START;

-- To query the captured data:
SELECT
    event_data.value('(event/@name)[1]', 'varchar(50)') AS event_name,
    event_data.value('(event/data[@name="wait_type"]/text)[1]', 'varchar(100)') AS wait_type,
    event_data.value('(event/data[@name="duration"]/value)[1]', 'bigint') AS duration_ms,
    event_data.value('(event/action[@name="sql_text"]/value)[1]', 'varchar(max)') AS sql_text
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('QueryWaits*.xel', NULL, NULL, NULL)
) AS x;
```

## Best Practices

- Always filter out benign/background waits before analyzing -- they dominate raw output and obscure real problems
- Use snapshot-based analysis (delta between two points) rather than cumulative totals for meaningful measurement
- Focus on the top 3-5 wait types -- they typically account for 80%+ of total wait time
- Correlate wait statistics with other DMVs (I/O stats, CPU stats, memory counters) to confirm the diagnosis
- Check signal wait percentage as a quick CPU pressure indicator before diving into scheduler details
- Use sys.dm_exec_session_wait_stats (2016+) to isolate waits for specific sessions during troubleshooting
- Reset wait stats (DBCC SQLPERF CLEAR) before running a targeted test to get clean measurements
- Do not treat all waits as problems -- some waits (like CXPACKET) are normal parts of healthy parallel processing
- Establish a baseline of normal waits so you can detect deviations during incidents

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Analyzing cumulative waits since last restart | Data skewed by old workloads; months of irrelevant history | Use delta snapshots over a representative time window |
| Treating all CXPACKET waits as a problem | Setting MAXDOP=1 cripples analytical and batch workloads | Tune cost threshold for parallelism; only reduce MAXDOP if skew is evident |
| Ignoring signal waits | Missing CPU pressure that compounds other wait types | Monitor signal wait percentage; add CPU or reduce workload when > 20% |
| Setting MAXDOP=1 globally to eliminate parallelism waits | Massive regression for queries that benefit from parallelism | Set MAXDOP per workload; use Resource Governor for mixed environments |
| Chasing ASYNC_NETWORK_IO on the server side | The problem is the client, not the server | Fix client-side consumption; use pagination; avoid SELECT * |
| Not filtering benign waits | Background waits dominate output and hide real issues | Maintain and use a comprehensive exclusion list |
| Resetting wait stats on a production server during peak hours | Lose baseline data needed for comparison | Reset only during maintenance windows or on test systems |

## SQL Server Version Notes

**SQL Server 2016**: sys.dm_exec_session_wait_stats introduced for per-session waits. CXCONSUMER wait type separated from CXPACKET (SP2/CU2). Query Store provides per-query wait categories.

**SQL Server 2017**: Wait statistics categories in Query Store (CPU, I/O, Memory, Lock, Latch, etc.). Automatic tuning can use wait data to detect plan regressions.

**SQL Server 2019**: Lightweight Query Profiling on by default (no trace flag needed). New wait types for Accelerated Database Recovery (ADR). WAIT_ON_SYNC_STATISTICS_REFRESH wait type for tracking statistics update impact.

**SQL Server 2022**: New wait types for Parameter Sensitive Plan optimization (PSP). Enhanced wait categorization in Query Store. Improved parallel redo wait tracking for Always On readable secondaries.

## Sources

- [Wait Types Reference (Microsoft Docs)](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-os-wait-stats-transact-sql)
- [SQL Server Wait Statistics (Paul Randal / SQLskills)](https://www.sqlskills.com/help/waits/)
- [Troubleshoot Slow Queries with Wait Statistics](https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store#use-wait-statistics)
- [sys.dm_exec_session_wait_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-session-wait-stats-transact-sql)
- [Extended Events Overview](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/extended-events)
