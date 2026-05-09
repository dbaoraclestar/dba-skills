# Wait Events — Diagnosing MySQL Performance Bottlenecks with Performance Schema Waits

## Overview

Wait event analysis is the most systematic approach to diagnosing MySQL performance problems. Instead of guessing whether the issue is CPU, I/O, locking, or something else, wait events tell you exactly what MySQL threads are waiting for. Every time a thread cannot make forward progress -- because it needs to read a page from disk, acquire a mutex, wait for a lock, or synchronize with another thread -- a wait event is recorded in Performance Schema.

This is analogous to Oracle's ASH/AWR wait event methodology, adapted for MySQL's architecture. The fundamental question is: "What are the database threads spending their time on?" If 80% of wait time is on I/O waits, you have a storage problem. If 80% is on mutex waits, you have a concurrency bottleneck. If 80% is on lock waits, you have a locking/contention problem. Wait events quantify the answer.

Performance Schema's wait event instrumentation is comprehensive, covering file I/O, table I/O, table locks, InnoDB internal mutexes and read-write locks, thread synchronization, network I/O, and more. However, not all instruments are enabled by default -- many are disabled to reduce overhead. A key DBA skill is knowing which instruments to enable for specific diagnostic scenarios and how to interpret the results.

## Key Concepts

**Wait Event**: A measurable period when a thread is blocked, waiting for a resource. Each wait has an event name, duration (in picoseconds), and source location (file and line number in MySQL source code).

**Instrument**: A named probe point in MySQL source code that generates wait events. Instruments follow a hierarchical naming convention: `wait/io/file/innodb/innodb_data_file`, `wait/synch/mutex/innodb/buf_pool_mutex`, etc.

**Consumer**: A destination table where event data is stored. Consumers include `events_waits_current` (current waits), `events_waits_history` (recent waits per thread), and `events_waits_history_long` (recent waits across all threads).

**Instrument Hierarchy**: Wait instruments are organized in a tree:
- `wait/io/file/*` -- File I/O operations
- `wait/io/table/*` -- Table I/O (row-level reads/writes)
- `wait/lock/table/*` -- Table-level locks
- `wait/lock/metadata/*` -- Metadata locks (DDL coordination)
- `wait/synch/mutex/*` -- Mutual exclusion locks (internal)
- `wait/synch/rwlock/*` -- Read-write locks (internal)
- `wait/synch/cond/*` -- Condition variable waits (internal)
- `wait/synch/sxlock/*` -- Shared-exclusive locks (internal, 8.0)

## Configuring Wait Event Instrumentation

### Checking Current Setup

```sql
-- Which wait instruments are enabled?
SELECT
    SUBSTRING_INDEX(NAME, '/', 3) AS instrument_group,
    COUNT(*) AS total,
    SUM(ENABLED = 'YES') AS enabled,
    SUM(TIMED = 'YES') AS timed
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'wait/%'
GROUP BY instrument_group
ORDER BY instrument_group;
```

### Enabling Instruments and Consumers

```sql
-- Enable all wait instruments (comprehensive but higher overhead)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';

-- Enable only I/O waits (common starting point)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/io/%';

-- Enable only lock waits
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/lock/%';

-- Enable only InnoDB mutex/rwlock waits
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/synch/%/innodb/%';

-- Enable wait event consumers
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN (
    'events_waits_current',
    'events_waits_history',
    'events_waits_history_long'
);

-- Persistent configuration (my.cnf)
-- [mysqld]
-- performance-schema-instrument = 'wait/io/%=ON'
-- performance-schema-instrument = 'wait/lock/%=ON'
-- performance-schema-instrument = 'wait/synch/mutex/innodb/%=ON'
-- performance-schema-consumer-events-waits-current = ON
-- performance-schema-consumer-events-waits-history = ON
-- performance-schema-consumer-events-waits-history-long = ON
```

### Consumer Sizing

```sql
-- Check consumer table sizes
SHOW VARIABLES LIKE 'performance_schema_events_waits_history_size';       -- per thread (default 10)
SHOW VARIABLES LIKE 'performance_schema_events_waits_history_long_size';  -- global (default 10000)

-- Increase if you need more history (requires restart)
-- [mysqld]
-- performance_schema_events_waits_history_size = 20
-- performance_schema_events_waits_history_long_size = 50000
```

## Top Wait Events — The Starting Point

### Global Top Waits

```sql
-- Top 20 wait events by total time
SELECT
    EVENT_NAME,
    COUNT_STAR AS total_waits,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_wait_ms,
    ROUND(MAX_TIMER_WAIT / 1e9, 3) AS max_wait_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle'
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### Top Waits by Category

```sql
-- Top wait categories (I/O vs locks vs synch)
SELECT
    SUBSTRING_INDEX(EVENT_NAME, '/', 3) AS wait_category,
    COUNT(*) AS distinct_events,
    SUM(COUNT_STAR) AS total_waits,
    ROUND(SUM(SUM_TIMER_WAIT) / 1e12, 3) AS total_wait_sec,
    ROUND(SUM(SUM_TIMER_WAIT) /
        (SELECT SUM(SUM_TIMER_WAIT) FROM performance_schema.events_waits_summary_global_by_event_name
         WHERE EVENT_NAME != 'idle') * 100, 2) AS pct_of_total
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle'
  AND COUNT_STAR > 0
GROUP BY wait_category
ORDER BY total_wait_sec DESC;
```

### Using sys Schema for Top Waits

```sql
-- sys schema wraps Performance Schema with human-readable formatting
SELECT * FROM sys.waits_global_by_latency
WHERE events != 'idle'
LIMIT 20;

-- Per-thread waits (identify which connections are waiting most)
SELECT
    thread_id,
    event_name,
    total,
    total_latency,
    avg_latency,
    max_latency
FROM sys.waits_by_host_by_latency
WHERE host IS NOT NULL
LIMIT 20;
```

## File I/O Waits (wait/io/file/*)

File I/O waits are among the most common performance bottlenecks. They indicate disk reads or writes.

### Top Files by I/O Wait Time

```sql
-- Which files are causing the most I/O wait time?
SELECT
    FILE_NAME,
    EVENT_NAME,
    COUNT_STAR AS io_ops,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_wait_ms,
    ROUND(SUM_NUMBER_OF_BYTES_READ / 1024 / 1024, 2) AS read_mb,
    ROUND(SUM_NUMBER_OF_BYTES_WRITE / 1024 / 1024, 2) AS write_mb,
    COUNT_READ,
    COUNT_WRITE
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### I/O by Event Type

```sql
-- Breakdown of file I/O wait events
SELECT
    EVENT_NAME,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_ms,
    ROUND(MAX_TIMER_WAIT / 1e9, 3) AS max_ms,
    ROUND(SUM_NUMBER_OF_BYTES_READ / 1024 / 1024, 2) AS read_mb,
    ROUND(SUM_NUMBER_OF_BYTES_WRITE / 1024 / 1024, 2) AS write_mb
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/io/file/%'
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### Key File I/O Wait Events

```
wait/io/file/innodb/innodb_data_file     -- InnoDB tablespace data files (.ibd)
wait/io/file/innodb/innodb_log_file      -- InnoDB redo log writes
wait/io/file/innodb/innodb_temp_file     -- InnoDB temporary tablespace
wait/io/file/sql/binlog                  -- Binary log writes
wait/io/file/sql/relay_log               -- Relay log I/O (replicas)
wait/io/file/sql/io_cache                -- Query cache / sort files
wait/io/file/sql/FRM                     -- Table definition files (.frm, 5.7)
wait/io/file/innodb/innodb_dblwr_file    -- Doublewrite buffer I/O
```

### Using sys Schema for I/O Analysis

```sql
-- I/O latency by file (human-readable)
SELECT * FROM sys.io_global_by_file_by_latency LIMIT 20;

-- I/O by bytes (which files transfer the most data)
SELECT * FROM sys.io_global_by_file_by_bytes LIMIT 20;

-- I/O latency breakdown for wait types
SELECT * FROM sys.io_global_by_wait_by_latency LIMIT 20;
```

## Table I/O Waits (wait/io/table/*)

Table I/O waits measure time spent reading from or writing to tables at the row level (after passing through the buffer pool).

```sql
-- Top tables by I/O wait time
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_STAR AS total_ops,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    COUNT_FETCH AS fetches,
    COUNT_INSERT AS inserts,
    COUNT_UPDATE AS updates,
    COUNT_DELETE AS deletes,
    ROUND(SUM_TIMER_FETCH / 1e12, 3) AS fetch_wait_sec,
    ROUND(SUM_TIMER_INSERT / 1e12, 3) AS insert_wait_sec,
    ROUND(SUM_TIMER_UPDATE / 1e12, 3) AS update_wait_sec,
    ROUND(SUM_TIMER_DELETE / 1e12, 3) AS delete_wait_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys', 'information_schema')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

### Table I/O by Index

```sql
-- Which indexes are causing the most I/O?
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_STAR AS total_ops,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND INDEX_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Tables with full scans (INDEX_NAME = NULL means no index used)
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_FETCH AS full_scan_fetches,
    ROUND(SUM_TIMER_FETCH / 1e12, 3) AS full_scan_wait_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NULL
  AND OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND COUNT_FETCH > 0
ORDER BY SUM_TIMER_FETCH DESC
LIMIT 20;
```

## Lock Waits

### InnoDB Row Lock Waits

```sql
-- Current InnoDB lock waits (who is blocking whom)
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    b.trx_started AS blocking_trx_started,
    TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_age_sec
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.INNODB_TRX r ON r.trx_id = w.requesting_trx_id\G
```

### Using sys Schema for Lock Analysis (MySQL 8.0)

```sql
-- InnoDB lock waits (sys schema view)
SELECT * FROM sys.innodb_lock_waits\G

-- Output includes:
-- wait_started, wait_age, waiting_query, waiting_lock_mode
-- blocking_query, blocking_lock_mode, blocking_trx_started, blocking_trx_age
-- locked_table, locked_index, locked_type
-- sql_kill_blocking_query (ready-to-run KILL command)
-- sql_kill_blocking_connection
```

### Performance Schema Lock Wait Events

```sql
-- Lock wait events summary (MySQL 8.0+)
SELECT
    EVENT_NAME,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_wait_ms,
    ROUND(MAX_TIMER_WAIT / 1e9, 3) AS max_wait_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/lock/%'
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC;
```

### Table Lock Waits

```sql
-- Table lock wait summary
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_STAR AS total_lock_waits,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    COUNT_READ_NORMAL,
    COUNT_READ_WITH_SHARED_LOCKS,
    COUNT_READ_NO_INSERT,
    COUNT_WRITE_ALLOW_WRITE,
    COUNT_WRITE_CONCURRENT_INSERT,
    COUNT_WRITE_LOW_PRIORITY,
    COUNT_WRITE_NORMAL
FROM performance_schema.table_lock_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
  AND SUM_TIMER_WAIT > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

## Metadata Lock Waits

Metadata locks (MDL) coordinate access between DDL and DML operations. A common production issue is an ALTER TABLE waiting because a long-running query holds a metadata lock on the table.

```sql
-- Current metadata lock waits (MySQL 8.0+)
SELECT
    pt.THREAD_ID,
    pt.PROCESSLIST_ID,
    pt.PROCESSLIST_USER,
    pt.PROCESSLIST_HOST,
    pt.PROCESSLIST_DB,
    pt.PROCESSLIST_COMMAND,
    pt.PROCESSLIST_TIME,
    pt.PROCESSLIST_STATE,
    pt.PROCESSLIST_INFO AS query,
    ml.OBJECT_TYPE,
    ml.OBJECT_SCHEMA,
    ml.OBJECT_NAME,
    ml.LOCK_TYPE,
    ml.LOCK_DURATION,
    ml.LOCK_STATUS
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads pt ON pt.THREAD_ID = ml.OWNER_THREAD_ID
WHERE ml.LOCK_STATUS = 'PENDING'
ORDER BY pt.PROCESSLIST_TIME DESC;

-- Find the blocker for metadata lock waits
SELECT
    blocker.PROCESSLIST_ID AS blocker_pid,
    blocker.PROCESSLIST_USER AS blocker_user,
    blocker.PROCESSLIST_TIME AS blocker_duration,
    blocker.PROCESSLIST_INFO AS blocker_query,
    blocked.PROCESSLIST_ID AS blocked_pid,
    blocked.PROCESSLIST_INFO AS blocked_query,
    ml_blocker.LOCK_TYPE AS blocker_lock_type,
    ml_blocked.LOCK_TYPE AS blocked_lock_type,
    ml_blocked.OBJECT_NAME AS locked_table
FROM performance_schema.metadata_locks ml_blocked
JOIN performance_schema.metadata_locks ml_blocker
    ON ml_blocked.OBJECT_SCHEMA = ml_blocker.OBJECT_SCHEMA
    AND ml_blocked.OBJECT_NAME = ml_blocker.OBJECT_NAME
    AND ml_blocked.LOCK_STATUS = 'PENDING'
    AND ml_blocker.LOCK_STATUS = 'GRANTED'
JOIN performance_schema.threads blocked ON blocked.THREAD_ID = ml_blocked.OWNER_THREAD_ID
JOIN performance_schema.threads blocker ON blocker.THREAD_ID = ml_blocker.OWNER_THREAD_ID
WHERE ml_blocked.OBJECT_TYPE = 'TABLE';
```

### Enabling Metadata Lock Instrumentation

```sql
-- Metadata lock instruments (may need to be enabled)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME = 'wait/lock/metadata/sql/mdl';

-- Persistent config
-- [mysqld]
-- performance-schema-instrument = 'wait/lock/metadata/sql/mdl=ON'
```

## InnoDB Internal Waits (Mutex/RWLock)

### Common InnoDB Mutex Waits

```sql
-- Top InnoDB mutex waits
SELECT
    EVENT_NAME,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_wait_sec,
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_wait_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/synch/mutex/innodb/%'
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 15;
```

### Key InnoDB Mutex/RWLock Events and Their Meaning

```
wait/synch/mutex/innodb/buf_pool_mutex
  -> Buffer pool LRU list contention
  -> Fix: increase innodb_buffer_pool_instances

wait/synch/mutex/innodb/log_sys_mutex
  -> Redo log write contention
  -> Fix: increase innodb_log_buffer_size; check storage speed

wait/synch/mutex/innodb/trx_mutex
  -> Transaction management contention
  -> Fix: reduce long transactions; check for excessive row-level locking

wait/synch/mutex/innodb/lock_mutex
  -> InnoDB lock system contention
  -> Fix: reduce lock contention; optimize transaction scope

wait/synch/mutex/innodb/dict_sys_mutex
  -> Data dictionary contention (table open/close)
  -> Fix: increase table_open_cache; reduce DDL frequency

wait/synch/mutex/innodb/fil_system_mutex
  -> Tablespace management contention
  -> Fix: reduce file-per-table operations; batch DDL

wait/synch/rwlock/innodb/btr_search_latch
  -> Adaptive Hash Index contention
  -> Fix: increase innodb_adaptive_hash_index_parts or disable AHI

wait/synch/rwlock/innodb/index_tree_rw_lock
  -> B-tree structure modification contention
  -> Fix: reduce hot-spot inserts (sequential PKs with high concurrency)

wait/synch/sxlock/innodb/hash_table_locks
  -> AHI partition locks (MySQL 8.0)
  -> Fix: increase innodb_adaptive_hash_index_parts or disable AHI
```

### InnoDB Mutex Monitor

```sql
-- SHOW ENGINE INNODB MUTEX (shows mutex with waits)
-- Note: requires innodb_monitor_enable or specific setup
SHOW ENGINE INNODB MUTEX;
-- Shows: Type, Name, Status (os_waits=N)
-- Focus on mutexes with high os_waits counts
```

## Real-Time Wait Analysis for Active Sessions

```sql
-- What are currently running threads waiting on right now?
SELECT
    t.PROCESSLIST_ID AS pid,
    t.PROCESSLIST_USER AS user,
    t.PROCESSLIST_DB AS db,
    t.PROCESSLIST_COMMAND AS command,
    t.PROCESSLIST_TIME AS time_sec,
    LEFT(t.PROCESSLIST_INFO, 80) AS query,
    ewc.EVENT_NAME AS current_wait,
    ROUND(ewc.TIMER_WAIT / 1e9, 3) AS wait_ms,
    ewc.OBJECT_SCHEMA,
    ewc.OBJECT_NAME,
    ewc.INDEX_NAME
FROM performance_schema.threads t
LEFT JOIN performance_schema.events_waits_current ewc
    ON ewc.THREAD_ID = t.THREAD_ID
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
  AND t.PROCESSLIST_ID IS NOT NULL
  AND t.TYPE = 'FOREGROUND'
ORDER BY t.PROCESSLIST_TIME DESC;
```

### Wait Event Timeline for a Specific Thread

```sql
-- Historical waits for a specific thread (for diagnosis)
SELECT
    EVENT_NAME,
    TIMER_START,
    ROUND(TIMER_WAIT / 1e9, 3) AS wait_ms,
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    OPERATION,
    SOURCE
FROM performance_schema.events_waits_history
WHERE THREAD_ID = (
    SELECT THREAD_ID FROM performance_schema.threads WHERE PROCESSLIST_ID = 42
)
ORDER BY TIMER_START;
```

## Resetting Wait Statistics

```sql
-- Reset all wait event statistics (useful before a diagnostic window)
TRUNCATE TABLE performance_schema.events_waits_summary_global_by_event_name;
TRUNCATE TABLE performance_schema.events_waits_summary_by_thread_by_event_name;
TRUNCATE TABLE performance_schema.events_waits_history;
TRUNCATE TABLE performance_schema.events_waits_history_long;

-- Reset file I/O statistics
TRUNCATE TABLE performance_schema.file_summary_by_instance;
TRUNCATE TABLE performance_schema.file_summary_by_event_name;

-- Reset table I/O statistics
TRUNCATE TABLE performance_schema.table_io_waits_summary_by_table;
TRUNCATE TABLE performance_schema.table_io_waits_summary_by_index_usage;

-- Reset everything via sys schema
CALL sys.ps_truncate_all_tables(FALSE);
```

## Diagnostic Workflow: Wait Event Triage

```sql
-- Step 1: Identify the top wait category
SELECT
    CASE
        WHEN EVENT_NAME LIKE 'wait/io/file/%' THEN 'File I/O'
        WHEN EVENT_NAME LIKE 'wait/io/table/%' THEN 'Table I/O'
        WHEN EVENT_NAME LIKE 'wait/lock/%' THEN 'Locks'
        WHEN EVENT_NAME LIKE 'wait/synch/mutex/%' THEN 'Mutex'
        WHEN EVENT_NAME LIKE 'wait/synch/rwlock/%' THEN 'RWLock'
        WHEN EVENT_NAME LIKE 'wait/synch/sxlock/%' THEN 'SXLock'
        WHEN EVENT_NAME LIKE 'wait/synch/cond/%' THEN 'Condition'
        ELSE 'Other'
    END AS category,
    SUM(COUNT_STAR) AS total_events,
    ROUND(SUM(SUM_TIMER_WAIT) / 1e12, 3) AS total_wait_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle' AND COUNT_STAR > 0
GROUP BY category
ORDER BY total_wait_sec DESC;

-- Step 2: Drill into the top category
-- (example: File I/O is the top category)
SELECT EVENT_NAME, COUNT_STAR,
       ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_sec,
       ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'wait/io/file/%' AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- Step 3: Identify specific files/tables causing the waits
-- (depends on the wait type found in Step 2)

-- Step 4: Correlate with queries
-- Which queries are generating the most waits?
SELECT
    LEFT(ess.DIGEST_TEXT, 100) AS query,
    ess.COUNT_STAR AS exec_count,
    ROUND(ess.SUM_TIMER_WAIT / 1e12, 3) AS total_exec_sec,
    ess.SUM_ROWS_EXAMINED,
    ess.SUM_CREATED_TMP_DISK_TABLES,
    ess.SUM_NO_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest ess
ORDER BY ess.SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Best Practices

- Start with the global top waits view to identify the bottleneck category before diving deep
- Enable only the instruments you need for the current investigation to minimize overhead
- Reset statistics (TRUNCATE) before a diagnostic window to get clean measurements
- Use sys schema views for quick analysis; switch to raw Performance Schema tables for detailed work
- Monitor metadata lock waits proactively -- they are a common cause of "stuck" ALTER TABLE operations
- Compare wait profiles across different time periods to detect regressions
- Correlate wait events with statement events to connect waits to specific queries
- Keep `events_waits_history_long` enabled in production with a reasonable size for post-mortem analysis
- Know the key InnoDB mutex names and what they mean -- this speeds up root cause analysis
- Use `innodb_lock_waits` from sys schema for quick lock contention diagnosis in production emergencies

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Not enabling wait event instruments | No wait data collected; blind to bottlenecks | Enable instruments for the category you are investigating |
| Enabling all instruments permanently in production | 5-10% overhead from comprehensive instrumentation | Enable selectively; disable after investigation |
| Ignoring the `idle` event in aggregations | `idle` dominates all wait summaries, hiding real issues | Always filter `WHERE EVENT_NAME != 'idle'` |
| Looking only at individual wait duration, not total time | A 1ms wait happening 1M times/second is worse than a 1s wait happening once | Sort by `SUM_TIMER_WAIT`, not `AVG_TIMER_WAIT` |
| Not correlating waits with queries | You know what the system waits on but not which queries cause it | Join `events_waits_*` with `events_statements_*` via `NESTING_EVENT_ID` |
| Confusing table I/O waits with file I/O waits | Different granularity; table I/O is logical, file I/O is physical | Understand the hierarchy: query -> table I/O -> buffer pool -> file I/O |
| Not monitoring metadata locks until ALTER TABLE hangs | MDL waits can block DDL for hours without visibility | Enable `wait/lock/metadata/sql/mdl` instrument; monitor proactively |

## MySQL Version Notes

**MySQL 5.7**:
- Performance Schema wait events available
- Many instruments disabled by default
- No `performance_schema.metadata_locks` table (use SHOW PROCESSLIST for "Waiting for table metadata lock")
- `events_waits_current/history/history_long` available
- sys schema included (5.7.7+) with basic wait views
- InnoDB mutex monitoring via SHOW ENGINE INNODB MUTEX

**MySQL 8.0**:
- `performance_schema.metadata_locks` table added for MDL visibility
- `performance_schema.data_locks` and `data_lock_waits` replace `INNODB_LOCKS` and `INNODB_LOCK_WAITS`
- More instruments enabled by default
- SX-lock (shared-exclusive lock) instruments added (`wait/synch/sxlock/*`)
- Improved sys schema wait views
- Better integration of wait events with statement and stage events
- `events_waits_summary_by_account_by_event_name` for per-account analysis
- Wait event instrumentation for clone plugin operations

**MySQL 8.4 / 9.x**:
- Enhanced wait event categorization
- Improved performance of wait instrumentation (lower overhead)
- Better correlation between wait events and other Performance Schema event types
- New instruments for parallel query execution waits
- Telemetry component integration for external observability

## Sources

- [MySQL Performance Schema Wait Event Tables](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-wait-tables.html)
- [MySQL Performance Schema Setup](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-startup-configuration.html)
- [MySQL Performance Schema Instrument Naming](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-instrument-naming.html)
- [MySQL InnoDB Monitors](https://dev.mysql.com/doc/refman/8.0/en/innodb-monitors.html)
- [MySQL sys Schema](https://dev.mysql.com/doc/refman/8.0/en/sys-schema.html)
- [MySQL Metadata Locking](https://dev.mysql.com/doc/refman/8.0/en/metadata-locking.html)
- [MySQL data_locks and data_lock_waits Tables](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html)
