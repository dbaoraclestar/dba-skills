# MySQL sys Schema — Human-Readable Performance Diagnostics

## Overview

The `sys` schema is a collection of views, functions, and stored procedures that
make Performance Schema data accessible and human-readable. Instead of dealing with
raw picosecond timer values and cryptic instrument names, the sys schema presents
formatted output with sensible column names, proper units (bytes, milliseconds),
and pre-built diagnostic queries.

The sys schema ships with MySQL 5.7.7+ and is installed by default. It requires
Performance Schema to be enabled. Every view in sys has two variants: a formatted
version (e.g., `statement_analysis`) that shows human-readable values, and a raw
version with an `x$` prefix (e.g., `x$statement_analysis`) that returns unformatted
numeric values suitable for programmatic consumption.

Use the sys schema for day-to-day performance triage, identifying slow queries,
finding unused indexes, checking lock contention, reviewing I/O hot spots, and
understanding memory allocation. It is the fastest path from "something is slow"
to "here is why."

## Key Concepts

- **Formatted Views**: Views without `x$` prefix return human-readable values
  (e.g., `4.52 GiB`, `125.30 ms`). Use for interactive analysis.
- **Raw Views**: Views with `x$` prefix return raw numeric values. Use for scripts,
  monitoring tools, and programmatic access.
- **sys_config Table**: Configuration key-value store that controls sys schema
  behavior (e.g., `statement_truncate_len` for max SQL text length in views).
- **Format Functions**: Helper functions like `format_bytes()`, `format_time()`,
  and `format_statement()` used internally and available for ad-hoc use.

## Statement Analysis

### statement_analysis / x$statement_analysis

Top queries ranked by total latency.

```sql
-- Top 10 queries by total execution time
SELECT query, db,
       exec_count,
       total_latency,
       avg_latency,
       rows_examined_avg,
       rows_sent_avg
FROM sys.statement_analysis
LIMIT 10;

-- Queries with the worst ratio of rows examined to rows sent
SELECT query, db,
       exec_count,
       rows_examined_avg,
       rows_sent_avg,
       ROUND(rows_examined_avg / GREATEST(rows_sent_avg, 1), 1) AS examine_to_send_ratio
FROM sys.x$statement_analysis
WHERE rows_examined_avg > 100
ORDER BY rows_examined_avg / GREATEST(rows_sent_avg, 1) DESC
LIMIT 20;
```

### statements_with_full_table_scans

```sql
-- Find queries doing full table scans
SELECT query, db,
       exec_count,
       total_latency,
       no_index_used_count,
       no_good_index_used_count,
       rows_examined_avg
FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC
LIMIT 20;
```

### statements_with_temp_tables

```sql
-- Queries creating temporary tables (potential optimization targets)
SELECT query, db,
       exec_count,
       total_latency,
       disk_tmp_tables,
       memory_tmp_tables
FROM sys.statements_with_temp_tables
WHERE disk_tmp_tables > 0
ORDER BY disk_tmp_tables DESC
LIMIT 15;
```

### statements_with_sorting

```sql
-- Queries with heavy sorting
SELECT query, db,
       exec_count,
       total_latency,
       sort_merge_passes,
       sort_rows_sorted,
       sort_using_filesort
FROM sys.statements_with_sorting
ORDER BY sort_merge_passes DESC
LIMIT 15;
```

## Host and User Summary

### host_summary

```sql
-- Activity summary grouped by client host
SELECT host,
       statements,
       statement_latency,
       table_scans,
       file_ios,
       file_io_latency,
       current_connections,
       total_connections
FROM sys.host_summary;

-- Host I/O details
SELECT host,
       file_ios, file_io_latency,
       unique_hosts
FROM sys.host_summary_by_file_io;
```

### user_summary

```sql
-- Activity summary grouped by user
SELECT user,
       statements,
       statement_latency,
       table_scans,
       file_ios,
       file_io_latency,
       current_connections
FROM sys.user_summary;

-- Statement latency breakdown by user
SELECT user,
       total AS statement_count,
       total_latency,
       avg_latency,
       max_latency
FROM sys.user_summary_by_statement_latency;
```

## I/O Analysis

### io_global_by_file_by_bytes

```sql
-- Files consuming the most I/O (by bytes)
SELECT file, total,
       total_read, total_written,
       read_pct,
       write_pct
FROM sys.io_global_by_file_by_bytes
LIMIT 20;
```

### io_global_by_file_by_latency

```sql
-- Files with the highest I/O latency
SELECT file,
       total AS io_count,
       total_latency,
       read_latency,
       write_latency
FROM sys.io_global_by_file_by_latency
LIMIT 20;
```

### io_global_by_wait_by_bytes / io_global_by_wait_by_latency

```sql
-- I/O wait types ranked by bytes
SELECT event_name, total,
       total_latency,
       avg_latency,
       total_requested
FROM sys.io_global_by_wait_by_bytes
LIMIT 15;
```

## Schema and Table Statistics

### schema_table_statistics

```sql
-- Table access statistics (reads, writes, latency)
SELECT table_schema, table_name,
       rows_fetched, fetch_latency,
       rows_inserted, insert_latency,
       rows_updated, update_latency,
       rows_deleted, delete_latency,
       io_read, io_write
FROM sys.schema_table_statistics
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY fetch_latency DESC
LIMIT 20;

-- Tables with the most I/O
SELECT table_schema, table_name,
       io_read_requests, io_read,
       io_write_requests, io_write
FROM sys.schema_table_statistics_with_buffer
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY io_read_requests + io_write_requests DESC
LIMIT 15;
```

### schema_auto_increment_columns

```sql
-- Auto-increment columns approaching their maximum
SELECT table_schema, table_name, column_name,
       data_type, column_type,
       auto_increment,
       max_value,
       ROUND(auto_increment / max_value * 100, 2) AS pct_used
FROM sys.schema_auto_increment_columns
WHERE auto_increment / max_value > 0.5
ORDER BY pct_used DESC;
```

## Index Analysis

### schema_unused_indexes

```sql
-- Indexes that have never been used (candidates for removal)
SELECT object_schema, object_name, index_name
FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY object_schema, object_name;
```

### schema_redundant_indexes

```sql
-- Redundant indexes (one index is a left-prefix of another)
SELECT table_schema, table_name,
       redundant_index_name,
       redundant_index_columns,
       dominant_index_name,
       dominant_index_columns
FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql', 'performance_schema', 'sys');

-- IMPORTANT: Verify before dropping. An index might be "redundant" in structure
-- but needed for a covering query that the dominant index does not cover.
```

## Lock Analysis

### innodb_lock_waits

```sql
-- Current InnoDB lock waits (who is blocking whom)
SELECT waiting_trx_id,
       waiting_query,
       waiting_lock_type,
       waiting_lock_mode,
       blocking_trx_id,
       blocking_query,
       blocking_lock_type,
       blocking_lock_mode,
       locked_table,
       locked_index
FROM sys.innodb_lock_waits;

-- The sql_kill_blocking_connection column provides a ready-made KILL command
SELECT waiting_query,
       blocking_query,
       wait_age,
       sql_kill_blocking_connection
FROM sys.innodb_lock_waits
ORDER BY wait_age DESC;
```

## Wait Analysis

### wait_classes_global_by_avg_latency

```sql
-- Wait event classes by average latency
SELECT event_class,
       total, total_latency,
       avg_latency,
       min_latency, max_latency
FROM sys.wait_classes_global_by_avg_latency
WHERE total > 0;
```

### waits_global_by_latency

```sql
-- Top individual wait events
SELECT events, total,
       total_latency, avg_latency,
       max_latency
FROM sys.waits_global_by_latency
LIMIT 20;
```

## Stored Procedures

### ps_setup_enable_consumer / ps_setup_disable_consumer

```sql
-- Enable all statement consumers
CALL sys.ps_setup_enable_consumer('events_statements');

-- Enable all wait consumers
CALL sys.ps_setup_enable_consumer('events_waits');

-- Disable wait history consumers
CALL sys.ps_setup_disable_consumer('events_waits_history');
```

### ps_setup_enable_instrument / ps_setup_disable_instrument

```sql
-- Enable all stage instruments
CALL sys.ps_setup_enable_instrument('stage');

-- Enable memory instruments
CALL sys.ps_setup_enable_instrument('memory');
```

### diagnostics()

```sql
-- Comprehensive diagnostic report (produces large output)
-- Captures two snapshots separated by a sleep interval
CALL sys.diagnostics(60, 120, 'current');
-- Args: sleep between snapshots (sec), max target runtime (sec), output format

-- WARNING: This produces extensive output. Best piped to a file:
-- mysql -e "CALL sys.diagnostics(30, 60, 'current')" > diag.txt
```

### ps_trace_thread()

```sql
-- Trace all activity for a specific thread for 10 seconds
-- Creates a DOT file for graphviz visualization
CALL sys.ps_trace_thread(42, CONCAT('/tmp/thread_trace_', NOW()+0, '.dot'), 10, 0, TRUE, TRUE, TRUE);
```

### ps_truncate_all_tables()

```sql
-- Reset all Performance Schema summary tables
CALL sys.ps_truncate_all_tables(FALSE);
-- Pass TRUE for verbose output showing which tables were truncated
```

## Format Functions

```sql
-- format_bytes: Convert byte count to human-readable
SELECT sys.format_bytes(1073741824);
-- Result: '1.00 GiB'

-- format_time: Convert picosecond timer to human-readable
SELECT sys.format_time(123456789012345);
-- Result: '2.06 min'

-- format_statement: Truncate long SQL for display
SELECT sys.format_statement('SELECT very_long_column_list FROM some_table WHERE complex_conditions...');
-- Truncates to statement_truncate_len from sys_config (default 64 chars)

-- format_path: Shorten file paths using @@datadir and @@tmpdir
SELECT sys.format_path('/var/lib/mysql/mydb/mytable.ibd');
-- Result: '@@datadir/mydb/mytable.ibd'

-- ps_thread_id: Get Performance Schema thread ID from connection ID
SELECT sys.ps_thread_id(CONNECTION_ID());
```

## sys_config Table

```sql
-- View current configuration
SELECT * FROM sys.sys_config;

-- Change SQL truncation length in views
UPDATE sys.sys_config
SET value = 128
WHERE variable = 'statement_truncate_len';

-- Change the default Performance Schema enabled setting
UPDATE sys.sys_config
SET value = 'YES'
WHERE variable = 'ps_thread_trx_info';
```

## Best Practices

- Use the formatted views (`statement_analysis`) for interactive troubleshooting
  and the raw views (`x$statement_analysis`) for scripts and monitoring tools.
- Start with `statement_analysis` for slow-query triage, then drill into
  `statements_with_full_table_scans` and `statements_with_temp_tables`.
- Regularly check `schema_unused_indexes` and `schema_redundant_indexes` to identify
  indexes that waste space and slow down writes.
- Use `innodb_lock_waits` as the first tool for diagnosing blocking — it shows
  waiting and blocking queries together with a ready-made KILL command.
- Reset Performance Schema statistics (`ps_truncate_all_tables`) before running
  a specific workload analysis to get clean data for that window.
- Use `diagnostics()` to capture a comprehensive snapshot when filing support
  tickets or investigating intermittent issues.
- Do not drop indexes identified by `schema_unused_indexes` without first checking
  if they might be used by infrequent but critical queries (batch jobs, month-end).

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using formatted views in monitoring scripts | String values like `4.52 GiB` cannot be compared or aggregated numerically | Use `x$` raw views for any programmatic consumption |
| Dropping "unused" indexes without checking batch workloads | Indexes may be critical for nightly ETL, monthly reports, or infrequent queries | Verify usage over a full business cycle before dropping |
| Running `diagnostics()` interactively without output redirect | Massive output floods the terminal and may time out | Pipe to a file: `mysql -e "CALL sys.diagnostics(30,60,'current')" > diag.txt` |
| Ignoring `schema_redundant_indexes` results | Redundant indexes waste disk, buffer pool memory, and slow every INSERT/UPDATE | Review and drop confirmed redundant indexes |
| Relying on sys schema when Performance Schema is OFF | All sys views return empty results without Performance Schema | Ensure `performance_schema = ON` in my.cnf |

## MySQL Version Notes

- **5.7**: sys schema available from 5.7.7. Must be installed explicitly on upgrades
  from 5.6 (`mysql_upgrade` handles this). Some views differ slightly from 8.0.
  No `x$innodb_buffer_stats_by_table` in earlier 5.7 releases.
- **8.0**: sys schema is always present and updated automatically. New views and
  functions added (e.g., `ps_is_thread_instrumented()`). `innodb_lock_waits` view
  updated to use `performance_schema.data_locks`/`data_lock_waits` instead of the
  removed `INFORMATION_SCHEMA.INNODB_LOCKS`/`INNODB_LOCK_WAITS` tables.
  `format_bytes()` and `format_time()` available as built-in functions (not just
  sys schema functions).
- **8.4 / 9.x**: Continued evolution. Check `SELECT * FROM sys.version` for the
  installed sys schema version. New diagnostic views may be added; check release
  notes.

## Sources

- [MySQL 8.0 Reference Manual: sys Schema](https://dev.mysql.com/doc/refman/8.0/en/sys-schema.html)
- [MySQL 8.0: sys Schema Views](https://dev.mysql.com/doc/refman/8.0/en/sys-schema-views.html)
- [MySQL 8.0: sys Schema Stored Procedures](https://dev.mysql.com/doc/refman/8.0/en/sys-schema-procedures.html)
- [MySQL 8.0: sys Schema Functions](https://dev.mysql.com/doc/refman/8.0/en/sys-schema-functions.html)
- [MySQL 5.7: sys Schema Reference](https://dev.mysql.com/doc/refman/5.7/en/sys-schema.html)
