# Extended Events — Lightweight Event Tracing for SQL Server

## Overview

Extended Events (XE) is SQL Server's modern, lightweight event-tracing framework that replaces the deprecated SQL Trace and Profiler. Built on an asynchronous architecture with minimal overhead, XE is the recommended approach for all diagnostic event capture starting with SQL Server 2012.

The framework is built on four core components: events (observable points in code execution), targets (consumers that store or display event data), predicates (filters that control which event firings are captured), and actions (additional data collected when an event fires). Sessions combine these components into a running trace configuration.

Unlike SQL Trace, Extended Events can capture events across the entire engine — not just query execution but also storage, memory, locking, and operating system interactions. The system_health session runs by default and captures critical diagnostic data without any setup. For production monitoring, XE sessions with file targets and appropriate predicates provide the best balance of detail and performance.

## Key Concepts

- **Event** — A defined point of interest in the SQL Server engine. Each event carries a payload of columns with contextual data.
- **Target** — A consumer of events. Common targets: `event_file` (async file), `ring_buffer` (in-memory circular), `histogram` (bucketed aggregation), `event_counter` (count only).
- **Predicate** — A filter expression evaluated before the event fires. Reduces overhead by discarding unneeded events at the source.
- **Action** — Additional data collected when an event passes the predicate. Examples: `sqlserver.sql_text`, `sqlserver.query_hash`, `sqlserver.session_id`.
- **Package** — A container that groups related events, targets, actions, and predicates. Core packages: `sqlserver`, `sqlos`, `package0`.
- **Session** — A named configuration that binds events + predicates + actions to targets. Sessions can be started, stopped, altered, and dropped.
- **Causality tracking** — The `package0.event_sequence` and `activity_id` actions enable correlating events across a session into a causal chain.

## Creating XE Sessions

### Basic Session Structure

```sql
CREATE EVENT SESSION [session_name]
ON SERVER    -- or ON DATABASE for Azure SQL Database
ADD EVENT package.event_name
(
    ACTION (sqlserver.sql_text, sqlserver.session_id)
    WHERE (predicate_expression)
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\session_name.xel',
        max_file_size = 100,    -- MB per file
        max_rollover_files = 5  -- retain 5 files
)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 30 SECONDS,
    STARTUP_STATE = ON    -- auto-start on SQL Server restart
);

ALTER EVENT SESSION [session_name] ON SERVER STATE = START;
```

### Predicate Syntax

```sql
-- Numeric comparison
WHERE (sqlserver.database_id = 5)

-- String comparison
WHERE (sqlserver.database_name = N'AdventureWorks')

-- Duration filter (microseconds)
WHERE (duration > 1000000)  -- > 1 second

-- Combining predicates
WHERE (sqlserver.database_id = 5 AND duration > 500000)

-- NOT IN equivalent
WHERE (sqlserver.session_id > 50
   AND sqlserver.is_system = 0)
```

## Common XE Sessions

### Query Performance Tracking

```sql
CREATE EVENT SESSION [QueryPerformance]
ON SERVER
ADD EVENT sqlserver.sql_statement_completed
(
    ACTION (
        sqlserver.sql_text,
        sqlserver.query_hash,
        sqlserver.query_plan_hash,
        sqlserver.session_id,
        sqlserver.database_name,
        sqlserver.username
    )
    WHERE (duration > 1000000          -- > 1 second
       AND sqlserver.is_system = 0)
),
ADD EVENT sqlserver.rpc_completed
(
    ACTION (
        sqlserver.sql_text,
        sqlserver.query_hash,
        sqlserver.session_id,
        sqlserver.database_name
    )
    WHERE (duration > 1000000
       AND sqlserver.is_system = 0)
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\QueryPerformance.xel',
        max_file_size = 200,
        max_rollover_files = 10
)
WITH (
    MAX_MEMORY = 8192 KB,
    MAX_DISPATCH_LATENCY = 15 SECONDS,
    STARTUP_STATE = ON
);
```

### Deadlock Monitoring

```sql
CREATE EVENT SESSION [DeadlockMonitor]
ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
(
    ACTION (
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.sql_text
    )
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\Deadlocks.xel',
        max_file_size = 50,
        max_rollover_files = 20
)
WITH (
    MAX_MEMORY = 4096 KB,
    STARTUP_STATE = ON
);
```

### Wait Statistics Per Query

```sql
CREATE EVENT SESSION [WaitStats]
ON SERVER
ADD EVENT sqlos.wait_completed
(
    ACTION (
        sqlserver.sql_text,
        sqlserver.session_id,
        sqlserver.query_hash
    )
    WHERE (duration > 10000             -- > 10 ms
       AND sqlserver.session_id > 50
       AND opcode = 1)                  -- completed waits only
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\WaitStats.xel',
        max_file_size = 100,
        max_rollover_files = 5
)
WITH (
    MAX_MEMORY = 8192 KB,
    MAX_DISPATCH_LATENCY = 30 SECONDS
);
```

### Long-Running Queries with Execution Plans

```sql
CREATE EVENT SESSION [LongRunningQueries]
ON SERVER
ADD EVENT sqlserver.sql_statement_completed
(
    ACTION (
        sqlserver.sql_text,
        sqlserver.query_plan_hash,
        sqlserver.session_id,
        sqlserver.database_name,
        sqlserver.client_hostname,
        sqlserver.client_app_name
    )
    WHERE (duration > 30000000)   -- > 30 seconds
),
ADD EVENT sqlserver.query_post_execution_showplan
(
    WHERE (duration > 30000000)
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\LongRunning.xel',
        max_file_size = 100,
        max_rollover_files = 5
)
WITH (
    MAX_MEMORY = 8192 KB,
    MAX_DISPATCH_LATENCY = 10 SECONDS
);
```

## The system_health Session

The `system_health` session runs by default since SQL Server 2008 and captures critical diagnostic data automatically.

```sql
-- View system_health events
SELECT
    xed.value('(@name)', 'VARCHAR(100)') AS event_name,
    xed.value('(@timestamp)', 'DATETIME2') AS event_time,
    xed.query('.') AS event_data
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets st
    JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
    WHERE s.name = 'system_health'
      AND st.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('RingBufferTarget/event') AS xdr(xed)
ORDER BY event_time DESC;

-- Deadlocks from system_health
SELECT
    xed.value('(@timestamp)', 'DATETIME2') AS deadlock_time,
    xed.query('.') AS deadlock_graph
FROM (
    SELECT CAST(target_data AS XML) AS target_data
    FROM sys.dm_xe_session_targets st
    JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
    WHERE s.name = 'system_health'
      AND st.target_name = 'ring_buffer'
) AS data
CROSS APPLY target_data.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdr(xed)
ORDER BY deadlock_time DESC;
```

## Reading XE File Targets

### Using sys.fn_xe_file_target_read_file

```sql
-- Read all events from file target
SELECT
    object_name AS event_name,
    CAST(event_data AS XML) AS event_xml,
    file_name,
    file_offset
FROM sys.fn_xe_file_target_read_file(
    N'C:\XE\QueryPerformance*.xel',   -- wildcards supported
    NULL,    -- deprecated metadata file parameter
    NULL,    -- starting file (NULL = all)
    NULL     -- starting offset
);

-- Parse specific fields from event XML
SELECT
    event_xml.value('(event/@timestamp)', 'DATETIME2') AS event_time,
    event_xml.value('(event/data[@name="duration"]/value)[1]', 'BIGINT') / 1000 AS duration_ms,
    event_xml.value('(event/data[@name="cpu_time"]/value)[1]', 'BIGINT') / 1000 AS cpu_ms,
    event_xml.value('(event/data[@name="logical_reads"]/value)[1]', 'BIGINT') AS logical_reads,
    event_xml.value('(event/action[@name="sql_text"]/value)[1]', 'NVARCHAR(MAX)') AS sql_text,
    event_xml.value('(event/action[@name="database_name"]/value)[1]', 'NVARCHAR(128)') AS db_name
FROM (
    SELECT CAST(event_data AS XML) AS event_xml
    FROM sys.fn_xe_file_target_read_file(
        N'C:\XE\QueryPerformance*.xel', NULL, NULL, NULL)
) AS xe
ORDER BY event_time DESC;
```

## Ring Buffer vs File Target

```sql
-- Ring buffer: in-memory, limited size, good for short-term diagnosis
ADD TARGET package0.ring_buffer
(
    SET max_memory = 4096   -- KB, wraps when full
)

-- Event file: async disk writes, scalable, production-safe
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\trace.xel',
        max_file_size = 100,       -- MB per file
        max_rollover_files = 5     -- total files retained
)

-- Histogram: bucketed aggregation, great for counting patterns
ADD TARGET package0.histogram
(
    SET source_type = 0,          -- event data field
        source = N'database_id',  -- field to bucket by
        slots = 256
)
```

## XE vs SQL Trace/Profiler

| Feature | Extended Events | SQL Trace / Profiler |
|---------|----------------|---------------------|
| Status | Current, actively developed | Deprecated since SQL Server 2012 |
| Overhead | Minimal with predicates | Significant, especially server-side trace |
| Scope | Full engine coverage | Query execution only |
| Azure support | Yes (ON DATABASE) | No |
| Filtering | Predicate pushdown at source | Filter after capture |
| Correlation | Causality tracking, activity_id | Limited |
| Live viewing | SSMS Live Data Viewer | Profiler GUI |
| Automation | T-SQL DDL, PowerShell | T-SQL (sp_trace_*) |

## Managing Sessions

```sql
-- List running sessions
SELECT name, create_time, total_buffer_size
FROM sys.dm_xe_sessions;

-- Stop a session
ALTER EVENT SESSION [QueryPerformance] ON SERVER STATE = STOP;

-- Drop a session
DROP EVENT SESSION [QueryPerformance] ON SERVER;

-- List available events
SELECT p.name AS package, o.name AS event_name, o.description
FROM sys.dm_xe_objects o
JOIN sys.dm_xe_packages p ON o.package_guid = p.guid
WHERE o.object_type = 'event'
ORDER BY p.name, o.name;

-- List available actions
SELECT p.name AS package, o.name AS action_name
FROM sys.dm_xe_objects o
JOIN sys.dm_xe_packages p ON o.package_guid = p.guid
WHERE o.object_type = 'action'
ORDER BY p.name, o.name;
```

## Best Practices

- Always use predicates to filter events at the source rather than capturing everything and filtering later.
- Prefer event_file targets over ring_buffer for production sessions — they handle high-volume events without memory pressure.
- Set `STARTUP_STATE = ON` for sessions that must survive SQL Server restarts.
- Use `MAX_DISPATCH_LATENCY` of 10-30 seconds for production; lower values increase overhead.
- Choose `ALLOW_SINGLE_EVENT_LOSS` retention mode for production to prevent blocking if the target cannot keep up.
- Store XE files on a dedicated drive, not on the same volume as data or log files.
- Use `query_hash` and `query_plan_hash` actions to aggregate similar queries across parameterized variations.
- Monitor session memory usage through `sys.dm_xe_sessions` to ensure sessions are not consuming excessive memory.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Capturing events without predicates | Massive data volume, performance degradation | Add WHERE clauses to filter by duration, database, or session_id |
| Using ring_buffer for high-volume long-running traces | Buffer wraps; early events lost silently | Use event_file target with rollover for anything beyond short diagnosis |
| Leaving `query_post_execution_showplan` on without tight filters | Showplan collection is expensive; severe CPU overhead | Add strict duration predicates (> 30s) or disable after investigation |
| Forgetting to set STARTUP_STATE = ON | Session disappears after SQL Server restart | Include `STARTUP_STATE = ON` in the WITH clause |
| Parsing XE XML with incorrect XPath | NULL results, missing data | Test XPath expressions against sample event_data; use correct namespace prefixes |
| Still using SQL Trace / Profiler in production | Deprecated, higher overhead, no Azure support | Migrate all traces to Extended Events sessions |

## SQL Server Version Notes

- **SQL Server 2016** — Query Store integration events added. Live Query Statistics events. `query_thread_profile` event for per-operator thread stats.
- **SQL Server 2017** — Added `query_store_*` events for plan forcing and regression detection. Linux support for XE with file targets.
- **SQL Server 2019** — `query_post_execution_plan_profile` (lightweight profiling v3) replaces older showplan events with significantly reduced overhead. New events for Intelligent Query Processing (adaptive joins, batch mode on rowstore).
- **SQL Server 2022** — Parameter Sensitive Plan events. Ledger verification events. Enhanced Query Store hints events. `degree_of_parallelism` event improvements.

## Sources

- [Extended Events Overview](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/extended-events)
- [Quick Start: Extended Events](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/quick-start-extended-events-in-sql-server)
- [sys.fn_xe_file_target_read_file](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-xe-file-target-read-file-transact-sql)
- [Use the system_health Session](https://learn.microsoft.com/en-us/sql/relational-databases/extended-events/use-the-system-health-session)
- [SQL Trace Deprecation](https://learn.microsoft.com/en-us/sql/relational-databases/event-classes/sql-trace-event-classes)
