# MySQL INFORMATION_SCHEMA — Metadata, Transactions, and InnoDB Internals

## Overview

`INFORMATION_SCHEMA` is MySQL's standard metadata catalog. It provides read-only
access to information about all databases, tables, columns, indexes, constraints,
privileges, and server status. Unlike the `performance_schema` (which tracks runtime
performance) or `sys` (which formats performance data), INFORMATION_SCHEMA answers
structural questions: What tables exist? How big are they? What indexes and foreign
keys are defined? What queries are currently running?

INFORMATION_SCHEMA is defined by the SQL standard and exists in all MySQL versions.
It is always available regardless of server configuration. However, some InnoDB-specific
tables were moved or replaced in MySQL 8.0 — notably `INNODB_LOCKS` and
`INNODB_LOCK_WAITS` were removed in favor of Performance Schema equivalents.

Use INFORMATION_SCHEMA for metadata discovery, schema documentation, migration
planning, storage analysis, foreign key verification, and administrative queries
about running transactions and processes. For performance diagnostics, prefer
Performance Schema and sys schema instead.

## Key Concepts

- **Virtual Tables**: INFORMATION_SCHEMA tables are views that query internal server
  data structures. They do not exist on disk.
- **Optimizer Considerations**: Queries against INFORMATION_SCHEMA can be slow on
  servers with thousands of tables because they may trigger metadata locks and file
  stat operations. Use `WHERE` clauses to narrow scope.
- **Case Sensitivity**: Table and column names in INFORMATION_SCHEMA are uppercase
  by convention, but MySQL treats them case-insensitively.
- **Read-Only**: You cannot INSERT, UPDATE, or DELETE from INFORMATION_SCHEMA tables.

## TABLES — Table Metadata and Sizes

```sql
-- List all tables in a database with row counts and sizes
SELECT TABLE_NAME,
       ENGINE,
       TABLE_ROWS,
       ROUND(DATA_LENGTH / 1048576, 2) AS data_mb,
       ROUND(INDEX_LENGTH / 1048576, 2) AS index_mb,
       ROUND((DATA_LENGTH + INDEX_LENGTH) / 1048576, 2) AS total_mb,
       ROW_FORMAT,
       AUTO_INCREMENT
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY DATA_LENGTH + INDEX_LENGTH DESC;

-- Total database sizes
SELECT TABLE_SCHEMA,
       COUNT(*) AS table_count,
       ROUND(SUM(DATA_LENGTH) / 1073741824, 2) AS data_gb,
       ROUND(SUM(INDEX_LENGTH) / 1073741824, 2) AS index_gb,
       ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1073741824, 2) AS total_gb
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys')
GROUP BY TABLE_SCHEMA
ORDER BY total_gb DESC;

-- Find tables without a primary key (anti-pattern for InnoDB)
SELECT t.TABLE_SCHEMA, t.TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES t
LEFT JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
  ON t.TABLE_SCHEMA = tc.TABLE_SCHEMA
  AND t.TABLE_NAME = tc.TABLE_NAME
  AND tc.CONSTRAINT_TYPE = 'PRIMARY KEY'
WHERE t.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                              'information_schema', 'sys')
  AND t.TABLE_TYPE = 'BASE TABLE'
  AND tc.CONSTRAINT_NAME IS NULL;

-- Tables with high fragmentation (free space ratio)
SELECT TABLE_SCHEMA, TABLE_NAME,
       ENGINE,
       ROUND(DATA_LENGTH / 1048576, 2) AS data_mb,
       ROUND(DATA_FREE / 1048576, 2) AS free_mb,
       ROUND(DATA_FREE / (DATA_LENGTH + 1) * 100, 1) AS frag_pct
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys')
  AND DATA_FREE > 10485760  -- more than 10MB free
  AND ENGINE = 'InnoDB'
ORDER BY DATA_FREE DESC;
```

## COLUMNS — Column Metadata

```sql
-- List all columns for a table with details
SELECT COLUMN_NAME, COLUMN_TYPE, IS_NULLABLE,
       COLUMN_DEFAULT, COLUMN_KEY, EXTRA,
       CHARACTER_SET_NAME, COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;

-- Find all columns of a specific type across all schemas
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, COLUMN_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE DATA_TYPE = 'enum'
  AND TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys')
ORDER BY TABLE_SCHEMA, TABLE_NAME;

-- Find columns with implicit defaults that may cause issues
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME,
       COLUMN_TYPE, IS_NULLABLE, COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys')
  AND IS_NULLABLE = 'NO'
  AND COLUMN_DEFAULT IS NULL
  AND EXTRA NOT LIKE '%auto_increment%'
  AND COLUMN_KEY != 'PRI';

-- Generated columns
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME,
       COLUMN_TYPE, GENERATION_EXPRESSION, EXTRA
FROM INFORMATION_SCHEMA.COLUMNS
WHERE GENERATION_EXPRESSION IS NOT NULL
  AND GENERATION_EXPRESSION != ''
  AND TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys');
```

## STATISTICS — Index Information

```sql
-- List all indexes for a table
SELECT INDEX_NAME, NON_UNIQUE, SEQ_IN_INDEX,
       COLUMN_NAME, COLLATION, CARDINALITY,
       SUB_PART, NULLABLE, INDEX_TYPE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'orders'
ORDER BY INDEX_NAME, SEQ_IN_INDEX;

-- Index column composition (multi-column indexes shown together)
SELECT TABLE_SCHEMA, TABLE_NAME, INDEX_NAME,
       GROUP_CONCAT(COLUMN_NAME ORDER BY SEQ_IN_INDEX) AS index_columns,
       NON_UNIQUE,
       INDEX_TYPE
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'mydb'
GROUP BY TABLE_SCHEMA, TABLE_NAME, INDEX_NAME, NON_UNIQUE, INDEX_TYPE
ORDER BY TABLE_NAME, INDEX_NAME;

-- Find duplicate indexes (same leading columns)
SELECT a.TABLE_SCHEMA, a.TABLE_NAME,
       a.INDEX_NAME AS idx1,
       b.INDEX_NAME AS idx2,
       a.COLUMN_NAME AS shared_leading_column
FROM INFORMATION_SCHEMA.STATISTICS a
JOIN INFORMATION_SCHEMA.STATISTICS b
  ON a.TABLE_SCHEMA = b.TABLE_SCHEMA
  AND a.TABLE_NAME = b.TABLE_NAME
  AND a.SEQ_IN_INDEX = 1
  AND b.SEQ_IN_INDEX = 1
  AND a.COLUMN_NAME = b.COLUMN_NAME
  AND a.INDEX_NAME < b.INDEX_NAME
WHERE a.TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                              'information_schema', 'sys');

-- Index cardinality (selectivity) check
SELECT TABLE_SCHEMA, TABLE_NAME, INDEX_NAME,
       COLUMN_NAME, CARDINALITY,
       (SELECT TABLE_ROWS FROM INFORMATION_SCHEMA.TABLES t
        WHERE t.TABLE_SCHEMA = s.TABLE_SCHEMA
          AND t.TABLE_NAME = s.TABLE_NAME) AS table_rows
FROM INFORMATION_SCHEMA.STATISTICS s
WHERE TABLE_SCHEMA = 'mydb'
  AND SEQ_IN_INDEX = 1
ORDER BY CARDINALITY ASC;
```

## KEY_COLUMN_USAGE and REFERENTIAL_CONSTRAINTS — Foreign Keys

```sql
-- List all foreign keys in a database
SELECT kcu.TABLE_NAME,
       kcu.COLUMN_NAME,
       kcu.CONSTRAINT_NAME,
       kcu.REFERENCED_TABLE_NAME,
       kcu.REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
WHERE kcu.TABLE_SCHEMA = 'mydb'
  AND kcu.REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY kcu.TABLE_NAME, kcu.CONSTRAINT_NAME, kcu.ORDINAL_POSITION;

-- Foreign key details with ON DELETE / ON UPDATE actions
SELECT rc.CONSTRAINT_NAME,
       rc.TABLE_NAME AS child_table,
       rc.REFERENCED_TABLE_NAME AS parent_table,
       rc.UPDATE_RULE,
       rc.DELETE_RULE,
       rc.MATCH_OPTION
FROM INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS rc
WHERE rc.CONSTRAINT_SCHEMA = 'mydb'
ORDER BY rc.TABLE_NAME;

-- Find tables referencing a specific parent table
SELECT kcu.TABLE_NAME AS child_table,
       kcu.COLUMN_NAME AS child_column,
       kcu.CONSTRAINT_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
WHERE kcu.TABLE_SCHEMA = 'mydb'
  AND kcu.REFERENCED_TABLE_NAME = 'customers';

-- All constraints by type
SELECT CONSTRAINT_NAME, TABLE_NAME, CONSTRAINT_TYPE
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY TABLE_NAME, CONSTRAINT_TYPE;
```

## PROCESSLIST — Running Queries

```sql
-- Show all active connections (better than SHOW PROCESSLIST for filtering)
SELECT ID, USER, HOST, DB,
       COMMAND, TIME, STATE, INFO
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;

-- Find long-running queries (over 60 seconds)
SELECT ID, USER, HOST, DB,
       TIME AS seconds_running,
       STATE,
       LEFT(INFO, 200) AS query_text
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE COMMAND != 'Sleep'
  AND TIME > 60
ORDER BY TIME DESC;

-- Connection count by user and host
SELECT USER, HOST, COUNT(*) AS connections,
       SUM(COMMAND = 'Sleep') AS sleeping,
       SUM(COMMAND != 'Sleep') AS active
FROM INFORMATION_SCHEMA.PROCESSLIST
GROUP BY USER, HOST
ORDER BY connections DESC;

-- Find queries waiting on locks
SELECT ID, USER, HOST, DB,
       TIME, STATE, INFO
FROM INFORMATION_SCHEMA.PROCESSLIST
WHERE STATE LIKE '%lock%'
   OR STATE LIKE '%waiting%'
ORDER BY TIME DESC;
```

## InnoDB Transaction Tables

### INNODB_TRX — Active Transactions

```sql
-- All active InnoDB transactions
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       trx_rows_locked, trx_rows_modified,
       trx_isolation_level,
       LEFT(trx_query, 200) AS current_query,
       trx_mysql_thread_id
FROM INFORMATION_SCHEMA.INNODB_TRX
ORDER BY trx_started ASC;

-- Long-running transactions (over 5 minutes)
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
       trx_rows_locked, trx_rows_modified,
       trx_mysql_thread_id
FROM INFORMATION_SCHEMA.INNODB_TRX
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 300
ORDER BY trx_started ASC;
```

### INNODB_LOCKS and INNODB_LOCK_WAITS (5.7 Only)

```sql
-- Lock waits: who is blocking whom (MySQL 5.7)
SELECT w.requesting_trx_id AS waiting_trx,
       w.blocking_trx_id AS blocking_trx,
       r.trx_mysql_thread_id AS waiting_thread,
       b.trx_mysql_thread_id AS blocking_thread,
       LEFT(r.trx_query, 100) AS waiting_query,
       LEFT(b.trx_query, 100) AS blocking_query,
       l.lock_table, l.lock_index, l.lock_type, l.lock_mode
FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS w
JOIN INFORMATION_SCHEMA.INNODB_TRX r ON w.requesting_trx_id = r.trx_id
JOIN INFORMATION_SCHEMA.INNODB_TRX b ON w.blocking_trx_id = b.trx_id
JOIN INFORMATION_SCHEMA.INNODB_LOCKS l ON w.requested_lock_id = l.lock_id;
```

### data_locks and data_lock_waits (8.0+ — Performance Schema)

```sql
-- Lock waits in MySQL 8.0+ (replaces INNODB_LOCKS/INNODB_LOCK_WAITS)
SELECT dl.ENGINE, dl.ENGINE_LOCK_ID,
       dl.ENGINE_TRANSACTION_ID, dl.THREAD_ID,
       dl.LOCK_TYPE, dl.LOCK_MODE, dl.LOCK_STATUS,
       dl.OBJECT_SCHEMA, dl.OBJECT_NAME, dl.INDEX_NAME,
       dl.LOCK_DATA
FROM performance_schema.data_locks dl
WHERE dl.LOCK_STATUS = 'WAITING';

-- Blocking relationships in 8.0+
SELECT dlw.REQUESTING_ENGINE_TRANSACTION_ID AS waiting_trx,
       dlw.BLOCKING_ENGINE_TRANSACTION_ID AS blocking_trx,
       rl.OBJECT_SCHEMA, rl.OBJECT_NAME,
       rl.LOCK_TYPE, rl.LOCK_MODE
FROM performance_schema.data_lock_waits dlw
JOIN performance_schema.data_locks rl
  ON dlw.REQUESTING_ENGINE_LOCK_ID = rl.ENGINE_LOCK_ID;
```

## INNODB_METRICS — InnoDB Counters

```sql
-- All enabled InnoDB metrics
SELECT NAME, SUBSYSTEM, COUNT, TYPE, COMMENT
FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE STATUS = 'enabled'
ORDER BY SUBSYSTEM, NAME;

-- Buffer pool hit ratio
SELECT
  (SELECT COUNT FROM INFORMATION_SCHEMA.INNODB_METRICS
   WHERE NAME = 'buffer_pool_read_requests') AS logical_reads,
  (SELECT COUNT FROM INFORMATION_SCHEMA.INNODB_METRICS
   WHERE NAME = 'buffer_pool_reads') AS disk_reads,
  ROUND(
    (1 - (SELECT COUNT FROM INFORMATION_SCHEMA.INNODB_METRICS
          WHERE NAME = 'buffer_pool_reads') /
         GREATEST((SELECT COUNT FROM INFORMATION_SCHEMA.INNODB_METRICS
                   WHERE NAME = 'buffer_pool_read_requests'), 1)
    ) * 100, 2
  ) AS hit_ratio_pct;

-- Enable additional metrics by subsystem
SET GLOBAL innodb_monitor_enable = 'dml_%';
SET GLOBAL innodb_monitor_enable = 'lock_%';

-- Undo log metrics
SELECT NAME, COUNT, COMMENT
FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE SUBSYSTEM = 'transaction'
ORDER BY NAME;

-- Adaptive hash index metrics
SELECT NAME, COUNT, COMMENT
FROM INFORMATION_SCHEMA.INNODB_METRICS
WHERE SUBSYSTEM = 'adaptive_hash_index';
```

## Additional Useful Tables

### ROUTINES — Stored Procedures and Functions

```sql
SELECT ROUTINE_SCHEMA, ROUTINE_NAME, ROUTINE_TYPE,
       DATA_TYPE, CREATED, LAST_ALTERED
FROM INFORMATION_SCHEMA.ROUTINES
WHERE ROUTINE_SCHEMA = 'mydb';
```

### TRIGGERS

```sql
SELECT TRIGGER_SCHEMA, TRIGGER_NAME, EVENT_MANIPULATION,
       EVENT_OBJECT_TABLE, ACTION_TIMING, ACTION_STATEMENT
FROM INFORMATION_SCHEMA.TRIGGERS
WHERE TRIGGER_SCHEMA = 'mydb';
```

### VIEWS

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, VIEW_DEFINITION,
       IS_UPDATABLE, CHECK_OPTION
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'mydb';
```

### PARTITIONS

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, PARTITION_NAME,
       SUBPARTITION_NAME, PARTITION_METHOD,
       PARTITION_EXPRESSION, PARTITION_DESCRIPTION,
       TABLE_ROWS,
       ROUND(DATA_LENGTH / 1048576, 2) AS data_mb
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND PARTITION_NAME IS NOT NULL
ORDER BY TABLE_NAME, PARTITION_ORDINAL_POSITION;
```

## Best Practices

- Always include a `WHERE TABLE_SCHEMA = ...` clause when querying
  INFORMATION_SCHEMA to avoid scanning all databases and triggering unnecessary
  metadata locks.
- Use `TABLE_ROWS` from the `TABLES` table as an estimate only — InnoDB uses
  sampling and the value can be off by 40-50% on large tables. Use `SELECT COUNT(*)`
  for exact counts.
- For lock analysis in MySQL 8.0+, use `performance_schema.data_locks` and
  `data_lock_waits` instead of the removed INNODB_LOCKS/INNODB_LOCK_WAITS.
- Prefer `performance_schema.processlist` over `INFORMATION_SCHEMA.PROCESSLIST`
  in MySQL 8.0+ — the Performance Schema version does not require a mutex and
  performs better under high concurrency.
- Enable additional `INNODB_METRICS` counters selectively for specific
  investigations; some counters have overhead when enabled.
- Use INFORMATION_SCHEMA for metadata discovery and schema documentation, but use
  Performance Schema and sys schema for performance diagnostics.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Querying INFORMATION_SCHEMA without schema filter | Full metadata scan across all databases; may hold metadata locks for minutes | Always add `WHERE TABLE_SCHEMA = 'specific_db'` |
| Treating `TABLE_ROWS` as exact | InnoDB estimates can be 40-50% off; leads to incorrect capacity planning | Use `SELECT COUNT(*)` when precision matters; `ANALYZE TABLE` to refresh estimates |
| Using `INNODB_LOCKS`/`INNODB_LOCK_WAITS` in MySQL 8.0 | These tables were removed; queries fail with "table not found" | Use `performance_schema.data_locks` and `data_lock_waits` instead |
| Using INFORMATION_SCHEMA.PROCESSLIST for monitoring | Requires global mutex in 5.7; degrades under high connection counts | In 8.0+, use `performance_schema.processlist` or `SHOW PROCESSLIST` |
| Ignoring `DATA_FREE` in TABLES output | High `DATA_FREE` indicates fragmentation wasting disk space | Monitor fragmentation; run `OPTIMIZE TABLE` or `ALTER TABLE ... ENGINE=InnoDB` |
| Not checking for tables without primary keys | InnoDB uses a hidden clustered index, degrading performance and replication | Query TABLE_CONSTRAINTS regularly; add explicit PKs |

## MySQL Version Notes

- **5.7**: `INNODB_LOCKS`, `INNODB_LOCK_WAITS`, and `INNODB_SYS_*` tables are
  available in INFORMATION_SCHEMA. `PROCESSLIST` table requires mutex. Partitions
  info available in INFORMATION_SCHEMA.PARTITIONS.
- **8.0**: `INNODB_LOCKS` and `INNODB_LOCK_WAITS` removed — replaced by
  `performance_schema.data_locks` and `data_lock_waits`. `INNODB_SYS_*` tables
  renamed to `INNODB_*` (e.g., `INNODB_SYS_TABLES` became `INNODB_TABLES`).
  `PROCESSLIST` table still works but `performance_schema.processlist` is preferred.
  `INNODB_METRICS` unchanged. New `COLUMN_STATISTICS` table for histogram data.
  New `ST_GEOMETRY_COLUMNS`, `ST_SPATIAL_REFERENCE_SYSTEMS` for spatial metadata.
- **8.4 / 9.x**: Continued evolution of INFORMATION_SCHEMA. Check release notes for
  any deprecated or newly added tables. The trend is to move runtime data to
  Performance Schema and keep INFORMATION_SCHEMA for structural metadata only.

## Sources

- [MySQL 8.0 Reference Manual: INFORMATION_SCHEMA](https://dev.mysql.com/doc/refman/8.0/en/information-schema.html)
- [MySQL 8.0: INFORMATION_SCHEMA TABLES Table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-tables-table.html)
- [MySQL 8.0: INFORMATION_SCHEMA INNODB_TRX Table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-trx-table.html)
- [MySQL 8.0: INFORMATION_SCHEMA INNODB_METRICS Table](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-metrics-table.html)
- [MySQL 8.0: Performance Schema data_locks Table](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html)
- [MySQL 5.7: INFORMATION_SCHEMA INNODB_LOCKS Table](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-locks-table.html)
