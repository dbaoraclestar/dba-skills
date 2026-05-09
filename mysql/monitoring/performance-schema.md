# MySQL Performance Schema — Instrumentation, Consumers, and Diagnostic Queries

## Overview

The Performance Schema is MySQL's built-in instrumentation framework for monitoring
server execution at a low level. It captures timing and count data for virtually
every internal operation — statement execution, wait events, I/O, memory allocation,
locking, and more. It is the foundation that the `sys` schema builds upon.

Unlike the `INFORMATION_SCHEMA`, which provides metadata about database objects,
Performance Schema provides runtime performance data. It uses in-memory tables
(no disk I/O for collection) and is designed to have minimal overhead when properly
configured. In MySQL 8.0+, most useful instruments are enabled by default.

Use Performance Schema when you need to diagnose slow queries, identify wait
bottlenecks, profile memory usage, track mutex contention, or understand I/O
patterns. It is the most powerful diagnostic tool built into MySQL and should be
the first stop for any performance investigation.

## Key Concepts

- **Instrument**: A probe point in the MySQL source code that can generate events
  (e.g., `statement/sql/select`, `wait/io/file/innodb/innodb_data_file`).
- **Consumer**: A destination table that stores collected events. Consumers must be
  enabled for events to be recorded.
- **Actor**: A combination of `HOST` and `USER` that determines which connections
  are instrumented.
- **Object**: A database object filter (`OBJECT_SCHEMA`, `OBJECT_NAME`,
  `OBJECT_TYPE`) controlling which objects are instrumented.
- **Event**: A single occurrence captured by an instrument (a statement, wait, stage,
  transaction, or memory allocation).
- **Digest**: A normalized form of a SQL statement with literals replaced by `?`,
  used for aggregation.

## Enabling and Disabling Instruments

### Check What Is Enabled

```sql
-- Count instruments by status
SELECT ENABLED, TIMED, COUNT(*)
FROM performance_schema.setup_instruments
GROUP BY ENABLED, TIMED;

-- Find all statement instruments
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%';

-- Find all wait instruments (I/O, synch, etc.)
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'wait/%'
ORDER BY NAME;
```

### Enable/Disable Instruments at Runtime

```sql
-- Enable all wait instruments
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';

-- Enable memory instrumentation (disabled by default in some versions)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES'
WHERE NAME LIKE 'memory/%';

-- Disable a specific instrument
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME = 'wait/synch/mutex/innodb/buf_pool_mutex';
```

### Enable at Startup (my.cnf)

```ini
[mysqld]
performance_schema = ON

# Enable specific instruments at startup
performance-schema-instrument = 'wait/%=ON'
performance-schema-instrument = 'memory/%=ON'
performance-schema-instrument = 'stage/%=ON'

# Enable specific consumers at startup
performance-schema-consumer-events-waits-current = ON
performance-schema-consumer-events-waits-history = ON
performance-schema-consumer-events-stages-current = ON
performance-schema-consumer-events-stages-history = ON
```

## Setup Tables

### setup_instruments

Controls what is instrumented.

```sql
-- Show instrument tree hierarchy
SELECT SUBSTRING_INDEX(NAME, '/', 2) AS instrument_class,
       COUNT(*) AS total,
       SUM(ENABLED = 'YES') AS enabled
FROM performance_schema.setup_instruments
GROUP BY instrument_class
ORDER BY instrument_class;
```

### setup_consumers

Controls which destination tables receive events.

```sql
-- Show all consumers and their status
SELECT * FROM performance_schema.setup_consumers;

-- Consumer hierarchy (parent must be enabled for child to work):
--   global_instrumentation
--     thread_instrumentation
--       events_waits_current
--         events_waits_history
--         events_waits_history_long
--       events_stages_current
--         events_stages_history
--         events_stages_history_long
--       events_statements_current
--         events_statements_history
--         events_statements_history_long
--       events_transactions_current
--         events_transactions_history
--         events_transactions_history_long
--     statements_digest

-- Enable statement history
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN ('events_statements_history', 'events_statements_history_long');

-- Enable wait event tracking
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE 'events_waits%';
```

### setup_actors

Controls which users/hosts are instrumented.

```sql
-- Show actor filtering
SELECT * FROM performance_schema.setup_actors;

-- Default: all users, all hosts are instrumented
-- Restrict to specific user
DELETE FROM performance_schema.setup_actors;
INSERT INTO performance_schema.setup_actors (HOST, USER, ENABLED, HISTORY)
VALUES ('%', 'app_user', 'YES', 'YES');

-- Monitor all users but exclude monitoring tools
INSERT INTO performance_schema.setup_actors VALUES ('%', '%', 'YES', 'YES');
DELETE FROM performance_schema.setup_actors WHERE USER = 'datadog';
```

### setup_objects

Controls which database objects are instrumented.

```sql
-- Show object filtering
SELECT * FROM performance_schema.setup_objects;

-- Disable instrumentation for a specific schema
INSERT INTO performance_schema.setup_objects
  (OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME, ENABLED, TIMED)
VALUES ('TABLE', 'temp_schema', '%', 'NO', 'NO');
```

## Key Tables: Statements

```sql
-- Top 10 queries by total execution time (from digest summary)
SELECT DIGEST_TEXT,
       COUNT_STAR AS exec_count,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
       ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_sec,
       ROUND(SUM_ROWS_EXAMINED / COUNT_STAR) AS avg_rows_examined,
       ROUND(SUM_ROWS_SENT / COUNT_STAR) AS avg_rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;

-- Currently running statements
SELECT THREAD_ID, EVENT_ID,
       TRUNCATE(TIMER_WAIT / 1e12, 2) AS elapsed_sec,
       SQL_TEXT,
       ROWS_EXAMINED, ROWS_SENT
FROM performance_schema.events_statements_current
WHERE SQL_TEXT IS NOT NULL
  AND END_EVENT_ID IS NULL
ORDER BY TIMER_WAIT DESC;

-- Recent statement history for a specific thread
SELECT EVENT_ID,
       TRUNCATE(TIMER_WAIT / 1e12, 4) AS duration_sec,
       SQL_TEXT,
       ROWS_EXAMINED,
       ERRORS, WARNINGS
FROM performance_schema.events_statements_history
WHERE THREAD_ID = <thread_id>
ORDER BY EVENT_ID DESC;

-- Queries with full table scans (no index used)
SELECT DIGEST_TEXT,
       COUNT_STAR,
       SUM_NO_INDEX_USED,
       SUM_NO_GOOD_INDEX_USED,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
ORDER BY SUM_NO_INDEX_USED DESC
LIMIT 20;
```

## Key Tables: Waits

```sql
-- Top wait events by total time
SELECT EVENT_NAME,
       COUNT_STAR,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
       ROUND(AVG_TIMER_WAIT / 1e12, 6) AS avg_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Current waits for all threads (what is MySQL waiting on right now)
SELECT t.PROCESSLIST_ID, t.PROCESSLIST_USER, t.PROCESSLIST_HOST,
       w.EVENT_NAME,
       TRUNCATE(w.TIMER_WAIT / 1e12, 4) AS wait_sec
FROM performance_schema.events_waits_current w
JOIN performance_schema.threads t ON w.THREAD_ID = t.THREAD_ID
WHERE w.EVENT_NAME != 'idle'
ORDER BY w.TIMER_WAIT DESC;
```

## Key Tables: Stages

```sql
-- Enable stage instruments first
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'stage/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE 'events_stages%';

-- See stages for currently executing statements
SELECT t.PROCESSLIST_ID,
       s.EVENT_NAME AS stage,
       TRUNCATE(s.TIMER_WAIT / 1e12, 4) AS stage_sec,
       s.WORK_COMPLETED, s.WORK_ESTIMATED
FROM performance_schema.events_stages_current s
JOIN performance_schema.threads t ON s.THREAD_ID = t.THREAD_ID
WHERE t.PROCESSLIST_ID IS NOT NULL;

-- Stage breakdown for ALTER TABLE progress monitoring
SELECT EVENT_NAME, WORK_COMPLETED, WORK_ESTIMATED,
       ROUND(WORK_COMPLETED / WORK_ESTIMATED * 100, 1) AS pct_complete
FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%';
```

## Key Tables: File I/O

```sql
-- File I/O summary by file type
SELECT EVENT_NAME,
       COUNT_READ, ROUND(SUM_TIMER_READ / 1e12, 2) AS read_sec,
       COUNT_WRITE, ROUND(SUM_TIMER_WRITE / 1e12, 2) AS write_sec,
       COUNT_MISC, ROUND(SUM_TIMER_MISC / 1e12, 2) AS misc_sec
FROM performance_schema.file_summary_by_event_name
WHERE COUNT_READ + COUNT_WRITE > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 15;

-- File I/O summary by individual file (hottest files)
SELECT FILE_NAME,
       COUNT_READ, ROUND(SUM_TIMER_READ / 1e12, 2) AS read_sec,
       COUNT_WRITE, ROUND(SUM_TIMER_WRITE / 1e12, 2) AS write_sec,
       ROUND(SUM_NUMBER_OF_BYTES_READ / 1048576, 1) AS read_mb,
       ROUND(SUM_NUMBER_OF_BYTES_WRITE / 1048576, 1) AS write_mb
FROM performance_schema.file_summary_by_instance
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

## Key Tables: Table I/O

```sql
-- Table I/O waits summary (which tables are accessed most)
SELECT OBJECT_SCHEMA, OBJECT_NAME,
       COUNT_READ, COUNT_WRITE, COUNT_FETCH,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
       ROUND(SUM_TIMER_READ / 1e12, 2) AS read_sec,
       ROUND(SUM_TIMER_WRITE / 1e12, 2) AS write_sec
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('performance_schema', 'mysql', 'sys')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- Index usage by table (which indexes are actually used)
SELECT OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME,
       COUNT_READ, COUNT_WRITE,
       ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA NOT IN ('performance_schema', 'mysql', 'sys')
  AND INDEX_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

## Memory Instrumentation

```sql
-- Enable memory instruments (if not already)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES'
WHERE NAME LIKE 'memory/%';

-- Total memory by component
SELECT SUBSTRING_INDEX(EVENT_NAME, '/', 3) AS component,
       ROUND(SUM(CURRENT_NUMBER_OF_BYTES_USED) / 1048576, 1) AS current_mb,
       ROUND(SUM(HIGH_NUMBER_OF_BYTES_USED) / 1048576, 1) AS high_water_mb
FROM performance_schema.memory_summary_global_by_event_name
GROUP BY component
ORDER BY SUM(CURRENT_NUMBER_OF_BYTES_USED) DESC
LIMIT 20;

-- Memory by user
SELECT USER,
       ROUND(SUM(CURRENT_NUMBER_OF_BYTES_USED) / 1048576, 1) AS current_mb
FROM performance_schema.memory_summary_by_user_by_event_name
GROUP BY USER
ORDER BY SUM(CURRENT_NUMBER_OF_BYTES_USED) DESC;

-- Detailed InnoDB memory usage
SELECT EVENT_NAME,
       CURRENT_COUNT_USED AS alloc_count,
       ROUND(CURRENT_NUMBER_OF_BYTES_USED / 1048576, 2) AS current_mb,
       ROUND(HIGH_NUMBER_OF_BYTES_USED / 1048576, 2) AS high_water_mb
FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/innodb/%'
  AND CURRENT_NUMBER_OF_BYTES_USED > 0
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC;
```

## Resetting Counters

```sql
-- Reset all statement digest statistics
TRUNCATE performance_schema.events_statements_summary_by_digest;

-- Reset file I/O statistics
TRUNCATE performance_schema.file_summary_by_event_name;
TRUNCATE performance_schema.file_summary_by_instance;

-- Reset table I/O statistics
TRUNCATE performance_schema.table_io_waits_summary_by_table;

-- Reset all events (current, history, history_long)
CALL sys.ps_truncate_all_tables(FALSE);
```

## Best Practices

- Always keep `performance_schema = ON` in production. The default instrumentation
  in MySQL 8.0 has very low overhead (typically less than 5%).
- Enable `events_statements_history` and `events_waits_current` consumers for
  ad-hoc troubleshooting readiness.
- Use `events_statements_summary_by_digest` for ongoing query performance monitoring
  rather than enabling the slow query log for everything.
- Enable memory instruments when investigating memory leaks or unexpected growth,
  but be aware they add some overhead.
- Periodically `TRUNCATE` summary tables to get fresh statistics for a specific
  time window analysis.
- Use `setup_actors` to limit instrumentation scope in high-connection environments
  where full instrumentation has measurable overhead.
- Pair Performance Schema with the `sys` schema for human-readable output of the
  same underlying data.
- Size the history tables via `performance_schema_events_*_history_long_size`
  variables at startup if defaults are too small.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Disabling Performance Schema entirely for "performance" | Lose all runtime diagnostics; flying blind | Leave it ON; disable only specific expensive instruments |
| Enabling consumers without enabling parent consumers | Child consumers silently collect nothing | Enable the full consumer hierarchy (global > thread > specific) |
| Forgetting to enable instruments before consumers | Consumer tables remain empty despite being enabled | Enable both the instrument AND the consumer |
| Not resetting summary tables before benchmarking | Results reflect cumulative data since last reset, skewing analysis | TRUNCATE relevant summary tables before starting measurement |
| Reading raw timer values without converting | Picosecond values are unreadable; mistakes in calculation | Divide by 1e12 for seconds or use sys schema views |
| Enabling all instruments at startup on high-throughput systems | Some instruments (especially mutex/rwlock) add measurable overhead | Enable selectively; test overhead before enabling broadly |

## MySQL Version Notes

- **5.7**: Performance Schema is ON by default. Memory instruments exist but are
  mostly disabled by default. Transaction event tracking is available. Statement
  digests use `DIGEST_TEXT` (limited to `performance_schema_max_digest_length`).
- **8.0**: Significantly more instruments enabled by default. Error log table
  (`error_log`) added in 8.0.22. Data lock instrumentation
  (`data_locks`, `data_lock_waits`) replaces `INNODB_LOCKS`/`INNODB_LOCK_WAITS`.
  New `variables_info` table shows when/how each variable was last set.
  `setup_threads` table added for thread-level control.
- **8.4 / 9.x**: Continued refinement of instruments. Telemetry trace integration
  for OpenTelemetry. Check release notes for new instrument classes and any
  changes to default consumer/instrument settings.

## Sources

- [MySQL 8.0 Reference Manual: Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)
- [MySQL 8.0: Performance Schema Quick Start](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-quick-start.html)
- [MySQL 8.0: Performance Schema Setup Tables](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-setup-tables.html)
- [MySQL 8.0: Performance Schema Statement Digests](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-statement-digests.html)
- [MySQL 8.0: Performance Schema Memory Summary Tables](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-memory-summary-tables.html)
