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

### Reading Deadlocks from system_health (No Setup Required)

The default `system_health` session captures deadlocks automatically. No custom XE session needed for basic deadlock analysis:

```sql
-- Extract deadlock XML from system_health ring buffer
SELECT XEvent.query('(event/data/value/deadlock)[1]') AS deadlock_graph
FROM (
    SELECT CAST(target_data AS XML) AS TargetData
    FROM sys.dm_xe_session_targets st
    JOIN sys.dm_xe_sessions s ON s.address = st.event_session_address
    WHERE s.name = 'system_health'
        AND st.target_name = 'ring_buffer'
) AS Data
CROSS APPLY TargetData.nodes('RingBufferTarget/event[@name="xml_deadlock_report"]') AS XEventData(XEvent);

-- Extract deadlocks from system_health file target (persisted across restarts)
SELECT
    event_data.value('(event/@timestamp)[1]', 'datetime2') AS deadlock_time,
    event_data.query('(event/data/value/deadlock)[1]') AS deadlock_graph
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('system_health*.xel', NULL, NULL, NULL)
    WHERE object_name = 'xml_deadlock_report'
) x
ORDER BY deadlock_time DESC;
```

In SSMS: navigate to **Management > Extended Events > Sessions > system_health > package0.event_file**, right-click, "View Target Data", then filter by name = `xml_deadlock_report`. Click the **Deadlock** tab for visual graph.

### Locking and Blocking Monitor

```sql
-- Real-time blocking chain detection
CREATE EVENT SESSION [BlockingMonitor]
ON SERVER
ADD EVENT sqlserver.blocked_process_report
(
    ACTION (
        sqlserver.database_name,
        sqlserver.session_id,
        sqlserver.sql_text,
        sqlserver.client_hostname
    )
)
ADD TARGET package0.event_file
(
    SET filename = N'C:\XE\Blocking.xel',
        max_file_size = 50,
        max_rollover_files = 10
)
WITH (
    MAX_MEMORY = 4096 KB,
    STARTUP_STATE = ON
);

-- Must set blocked process threshold (seconds) for events to fire
EXEC sp_configure 'blocked process threshold (s)', 5;
RECONFIGURE;
```

### SQL Server Lock Modes Quick Reference

| Mode | Name | Compatible With | Use |
|------|------|----------------|-----|
| S | Shared | S, IS, U | SELECT reads |
| U | Update | S, IS | Scan for update target |
| X | Exclusive | None | INSERT, UPDATE, DELETE |
| IS | Intent Shared | S, IS, IX, IU, SIX | Higher-level intent |
| IX | Intent Exclusive | IS, IU | Higher-level intent |
| Sch-S | Schema Stability | All except Sch-M | Query compilation |
| Sch-M | Schema Modification | None | ALTER TABLE, DROP |
| BU | Bulk Update | BU, Sch-S | BULK INSERT |

Lock escalation: row → page → table. Disable per-table if needed:

```sql
ALTER TABLE Orders SET (LOCK_ESCALATION = DISABLE);  -- or ROW_OR_PAGE (2008+)
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
- [Monitor Deadlocks with system_health Extended Events](https://www.mssqltips.com/sqlservertip/6430/monitor-deadlocks-in-sql-server-with-systemhealth-extended-events/)
- [Capturing Deadlock Information in XML Format](https://www.mssqltips.com/sqlservertip/1234/capturing-sql-server-deadlock-information-in-xml-format/)
- [Report SQL Server Deadlock Occurrences](https://www.sqlshack.com/report-sql-server-deadlock-occurrences/)
- [Save Deadlock Graphs with SQL Server Profiler](https://learn.microsoft.com/en-us/sql/relational-databases/performance/save-deadlock-graphs-sql-server-profiler)
- [Transaction Locking and Row Versioning Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide)
- [Modes of Transactions in SQL Server](https://www.sqlshack.com/modes-of-transactions-in-sql-server/)
- [KILL SPID Command in SQL Server](https://www.sqlshack.com/kill-spid-command-in-sql-server/)
- [Monitor AG with Extended Events](https://www.sqlshack.com/monitor-sql-server-always-on-availability-groups-using-extended-events/)
