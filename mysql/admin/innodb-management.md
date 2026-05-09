# InnoDB Management — Architecture, Tablespaces, Redo/Undo Logs, and Crash Recovery

## Overview

InnoDB is the default and most widely used MySQL storage engine. It provides ACID-compliant
transactions, row-level locking, MVCC (Multi-Version Concurrency Control), crash recovery,
and foreign key support. Understanding InnoDB internals is essential for capacity planning,
performance tuning, and disaster recovery.

The InnoDB architecture revolves around an in-memory buffer pool that caches data and index
pages, a redo log (WAL) that ensures durability, and an undo log that enables consistent reads
and rollback. These components work together to deliver high throughput while guaranteeing data
integrity even after unexpected crashes.

This document covers the key InnoDB subsystems, tablespace management, redo and undo log sizing,
crash recovery internals, and the `FORCE_RECOVERY` escalation ladder for severe corruption
scenarios.

## Key Concepts

- **Buffer Pool**: In-memory cache for InnoDB data and index pages. The single most important InnoDB performance parameter.
- **Redo Log (WAL)**: Write-ahead log that records all modifications. Ensures durability and enables crash recovery.
- **Undo Log**: Stores pre-modification row versions for MVCC reads and transaction rollback.
- **Doublewrite Buffer**: Protection against partial page writes; pages are written to a doublewrite area before their final location.
- **Change Buffer**: Caches changes to secondary index pages when those pages are not in the buffer pool.
- **Adaptive Hash Index (AHI)**: Automatically built in-memory hash index for frequently accessed pages.
- **Tablespace**: Logical container for InnoDB data files. System tablespace, file-per-table, general, undo, and temporary tablespaces.
- **Page**: The basic I/O unit in InnoDB; default size is 16 KB.

## InnoDB Architecture

### Buffer Pool

```sql
-- Check buffer pool size (should be 70-80% of available RAM for dedicated servers)
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

-- Set buffer pool size (can be changed online in 8.0+)
SET GLOBAL innodb_buffer_pool_size = 32 * 1024 * 1024 * 1024;  -- 32 GB

-- Multiple buffer pool instances (reduces contention)
-- Set in my.cnf; requires restart
-- innodb_buffer_pool_instances = 8  (one per GB of buffer pool, max practical ~16)

-- Buffer pool hit ratio (target > 99%)
SELECT
  (1 - (
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
  )) * 100 AS buffer_pool_hit_ratio;

-- Buffer pool stats
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Buffer pool contents by table (which objects consume buffer pool)
SELECT
  table_name,
  index_name,
  COUNT(*) AS pages,
  COUNT(*) * 16 / 1024 AS size_mb
FROM information_schema.INNODB_BUFFER_PAGE
WHERE table_name IS NOT NULL
  AND table_name NOT LIKE '%`mysql`%'
GROUP BY table_name, index_name
ORDER BY pages DESC
LIMIT 20;

-- Dump and reload buffer pool on restart (warm-up)
SET GLOBAL innodb_buffer_pool_dump_at_shutdown = ON;
SET GLOBAL innodb_buffer_pool_load_at_startup = ON;

-- Trigger manual dump/load
SET GLOBAL innodb_buffer_pool_dump_now = ON;
SET GLOBAL innodb_buffer_pool_load_now = ON;

-- Check load progress
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

### Buffer Pool Configuration in my.cnf

```ini
[mysqld]
# Primary tuning parameter
innodb_buffer_pool_size         = 32G

# Multiple instances reduce mutex contention
innodb_buffer_pool_instances    = 8

# Online resize chunk size (default 128M)
innodb_buffer_pool_chunk_size   = 256M

# Save/restore buffer pool state across restarts
innodb_buffer_pool_dump_at_shutdown = ON
innodb_buffer_pool_load_at_startup  = ON

# Percentage of most-recently-used pages to dump (default 25)
innodb_buffer_pool_dump_pct     = 40
```

## Redo Log (Write-Ahead Log)

### Architecture

The redo log ensures transaction durability. Every data modification is first written to the
redo log buffer, then flushed to the redo log files on disk, and only later applied to the
actual data pages. During crash recovery, InnoDB replays the redo log to restore committed
transactions.

### Redo Log Sizing

```sql
-- Check current redo log configuration
SHOW VARIABLES LIKE 'innodb_log%';
SHOW VARIABLES LIKE 'innodb_redo%';

-- Pre-8.0.30: redo log configured as files
-- innodb_log_file_size * innodb_log_files_in_group = total redo log capacity
SHOW VARIABLES LIKE 'innodb_log_file_size';       -- e.g., 1G per file
SHOW VARIABLES LIKE 'innodb_log_files_in_group';   -- e.g., 2 files = 2G total

-- 8.0.30+: unified redo log capacity parameter
SHOW VARIABLES LIKE 'innodb_redo_log_capacity';    -- e.g., 2G total

-- Change redo log capacity online (8.0.30+)
SET GLOBAL innodb_redo_log_capacity = 4 * 1024 * 1024 * 1024;  -- 4 GB

-- Monitor redo log usage
SHOW STATUS LIKE 'Innodb_redo_log%';

-- Check redo log checkpoint age (how much redo log is in use)
SELECT
  NAME, SUBSYSTEM, STATUS
FROM performance_schema.innodb_metrics
WHERE NAME LIKE 'log_%'
ORDER BY NAME;
```

### Redo Log Sizing Guidelines

| Workload | Recommended Redo Log Size |
|----------|---------------------------|
| Light OLTP (< 100 TPS) | 512 MB - 1 GB |
| Medium OLTP (100-1000 TPS) | 2 GB - 4 GB |
| Heavy OLTP (> 1000 TPS) | 4 GB - 16 GB |
| Bulk loading / ETL | 8 GB - 32 GB |

```sql
-- Estimate optimal redo log size:
-- Check redo log write rate over a peak period
SHOW STATUS LIKE 'Innodb_os_log_written';
-- Wait 60 seconds, then check again
-- (value2 - value1) * 60 = bytes per hour at peak
-- Target: redo log should hold at least 1-2 hours of peak writes
```

### Redo Log Configuration (my.cnf)

```ini
[mysqld]
# Pre-8.0.30:
innodb_log_file_size    = 2G
innodb_log_files_in_group = 2
# Total redo log = 4G

# 8.0.30+:
innodb_redo_log_capacity = 4G

# Flush method
innodb_flush_log_at_trx_commit = 1   # 1 = full ACID (safest)
                                      # 2 = flush per second (faster, less safe)
                                      # 0 = no flush (fastest, risk data loss)

# Log buffer size (increase for large transactions)
innodb_log_buffer_size = 64M
```

## Undo Log and Tablespace Management

### Undo Architecture

The undo log stores old versions of rows to support MVCC and rollback. Long-running transactions
can cause undo log growth ("history list length" bloat), which increases disk usage and slows
purging.

```sql
-- Check undo tablespace status
SELECT TABLESPACE_NAME, FILE_NAME, FILE_TYPE, STATUS, ENGINE
FROM information_schema.FILES
WHERE FILE_TYPE = 'UNDO LOG';

-- Check history list length (target: < 10,000 in normal operation)
SHOW STATUS LIKE 'Innodb_history_list_length';

-- Check purge lag
SELECT NAME, COUNT
FROM information_schema.INNODB_METRICS
WHERE NAME = 'trx_rseg_history_len';

-- Identify long-running transactions holding undo
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_seconds,
       trx_rows_modified, trx_rows_locked
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;
```

### Managing Undo Tablespaces (8.0+)

```sql
-- List undo tablespaces
SELECT * FROM information_schema.INNODB_TABLESPACES
WHERE SPACE_TYPE = 'Undo';

-- Create additional undo tablespace
CREATE UNDO TABLESPACE undo_003 ADD DATAFILE 'undo_003.ibu';

-- Mark an undo tablespace inactive (for truncation)
ALTER UNDO TABLESPACE undo_001 SET INACTIVE;

-- Check truncation progress
SELECT NAME, STATE FROM information_schema.INNODB_TABLESPACES
WHERE SPACE_TYPE = 'Undo';

-- Re-activate after truncation
ALTER UNDO TABLESPACE undo_001 SET ACTIVE;

-- Drop an empty undo tablespace (must be inactive and empty)
DROP UNDO TABLESPACE undo_003;
```

### Undo Configuration

```ini
[mysqld]
# Enable automatic undo tablespace truncation
innodb_undo_log_truncate = ON

# Truncation frequency (purge loops between truncation attempts)
innodb_purge_rseg_truncate_frequency = 128

# Maximum undo tablespace size before truncation triggers
innodb_max_undo_log_size = 1G

# Number of undo tablespaces (minimum 2 for truncation)
# Set at initialization; cannot be changed later (pre-8.0.14)
innodb_undo_tablespaces = 2
```

## Tablespace Management

### System Tablespace

```sql
-- Check system tablespace (ibdata1)
SELECT FILE_NAME, TABLESPACE_NAME, INITIAL_SIZE, TOTAL_EXTENTS, FREE_EXTENTS
FROM information_schema.FILES
WHERE TABLESPACE_NAME = 'innodb_system';

-- System tablespace configuration
SHOW VARIABLES LIKE 'innodb_data_file_path';
-- Default: ibdata1:12M:autoextend
```

```ini
[mysqld]
# System tablespace configuration
innodb_data_file_path = ibdata1:1G:autoextend
innodb_data_home_dir  = /var/lib/mysql

# Autoextend increment (in MB)
innodb_autoextend_increment = 64
```

### File-Per-Table Tablespaces

```sql
-- Check setting (default ON since 5.6.6)
SHOW VARIABLES LIKE 'innodb_file_per_table';

-- Each table gets its own .ibd file
-- Benefits: individual table reclaim, transportable tablespaces, per-table compression

-- Reclaim space after large DELETE
ALTER TABLE myapp_db.large_table ENGINE=InnoDB;
-- or
OPTIMIZE TABLE myapp_db.large_table;

-- Check table size in tablespace
SELECT
  table_schema,
  table_name,
  ROUND(data_length / 1024 / 1024, 2) AS data_mb,
  ROUND(index_length / 1024 / 1024, 2) AS index_mb,
  ROUND(data_free / 1024 / 1024, 2) AS free_mb
FROM information_schema.TABLES
WHERE table_schema = 'myapp_db'
ORDER BY data_length + index_length DESC
LIMIT 20;
```

### General Tablespaces (5.7.6+)

```sql
-- Create a general tablespace
CREATE TABLESPACE ts_app
  ADD DATAFILE 'ts_app.ibd'
  ENGINE = InnoDB;

-- Create table in specific tablespace
CREATE TABLE myapp_db.orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  customer_id INT NOT NULL,
  total DECIMAL(10,2)
) TABLESPACE ts_app;

-- Move existing table to general tablespace
ALTER TABLE myapp_db.orders TABLESPACE ts_app;

-- Move table back to file-per-table
ALTER TABLE myapp_db.orders TABLESPACE innodb_file_per_table;

-- Drop an empty general tablespace
DROP TABLESPACE ts_app;

-- List all tablespaces
SELECT SPACE, NAME, SPACE_TYPE, STATE, FILE_SIZE, ALLOCATED_SIZE
FROM information_schema.INNODB_TABLESPACES
ORDER BY FILE_SIZE DESC;
```

### Temporary Tablespace

```sql
-- Check temporary tablespace configuration
SHOW VARIABLES LIKE 'innodb_temp_data_file_path';
-- Default: ibtmp1:12M:autoextend

-- Session temporary tablespace pool (8.0+)
SHOW VARIABLES LIKE 'innodb_temp_tablespaces_dir';

-- Monitor temp tablespace usage
SELECT * FROM information_schema.INNODB_SESSION_TEMP_TABLESPACES;

-- Check global temp tablespace size
SELECT FILE_NAME, TABLESPACE_NAME, TOTAL_EXTENTS * EXTENT_SIZE / 1024 / 1024 AS size_mb
FROM information_schema.FILES
WHERE TABLESPACE_NAME = 'innodb_temporary';
```

## Doublewrite Buffer

```sql
-- Check doublewrite status
SHOW VARIABLES LIKE 'innodb_doublewrite';

-- 8.0.20+: doublewrite files are separate from system tablespace
SHOW VARIABLES LIKE 'innodb_doublewrite_dir';
SHOW VARIABLES LIKE 'innodb_doublewrite_files';
SHOW VARIABLES LIKE 'innodb_doublewrite_pages';
SHOW VARIABLES LIKE 'innodb_doublewrite_batch_size';

-- Monitor doublewrite activity
SHOW STATUS LIKE 'Innodb_dblwr%';
```

```ini
[mysqld]
# Doublewrite configuration (8.0.20+)
innodb_doublewrite       = ON          # Never disable unless using atomic writes (ZFS, FusionIO)
innodb_doublewrite_dir   = /var/lib/mysql  # Separate disk for performance
innodb_doublewrite_files = 2
innodb_doublewrite_pages = 128
```

## Crash Recovery

### How Crash Recovery Works

1. **Redo Log Scan**: InnoDB reads the redo log from the last checkpoint to the end.
2. **Redo Application**: All logged modifications are re-applied to data pages (roll forward).
3. **Undo Pass**: Uncommitted transactions are identified and rolled back using undo logs.
4. **Purge**: Old undo records are cleaned up after recovery completes.

```sql
-- Monitor crash recovery progress (in error log)
-- Look for messages like:
-- [InnoDB] Applying a batch of N redo log records...
-- [InnoDB] Starting crash recovery from checkpoint...

-- After recovery, verify data integrity
CHECK TABLE myapp_db.orders EXTENDED;
mysqlcheck --all-databases --check --extended -u root -p
```

### InnoDB Force Recovery

When InnoDB cannot start normally due to corruption, use `innodb_force_recovery` to start
in a degraded mode and salvage data. This is a **last resort**.

```ini
[mysqld]
# Add to my.cnf, start with lowest level that works, escalate if needed
innodb_force_recovery = 1
```

| Level | Name | Behavior |
|-------|------|----------|
| 0 | Normal | Default; full crash recovery |
| 1 | SRV_FORCE_IGNORE_CORRUPT | Skip corrupt pages; continue startup |
| 2 | SRV_FORCE_NO_BACKGROUND | Do not run background operations (purge, insert buffer merge, etc.) |
| 3 | SRV_FORCE_NO_TRX_UNDO | Skip transaction rollback after crash recovery |
| 4 | SRV_FORCE_NO_IBUF_MERGE | Do not merge insert buffer entries; skip them |
| 5 | SRV_FORCE_NO_UNDO_LOG_SCAN | Do not look at undo logs; treat incomplete transactions as committed |
| 6 | SRV_FORCE_NO_LOG_REDO | Do not replay redo log; start with whatever is on disk |

```sql
-- Recovery procedure:
-- 1. Set innodb_force_recovery to the lowest effective level
-- 2. Start MySQL
-- 3. Dump all data with mysqldump
-- 4. Stop MySQL
-- 5. Remove innodb_force_recovery from my.cnf
-- 6. Remove all InnoDB data files (ibdata1, *.ibd, ib_logfile*, #ib_redo*)
-- 7. Start MySQL fresh
-- 8. Reload the dump

-- Dump data while in force recovery mode
mysqldump --all-databases --routines --triggers --events \
  --single-transaction -u root -p > /backup/emergency_dump.sql
```

> **Critical**: At `innodb_force_recovery >= 4`, InnoDB is read-only. You cannot run
> INSERT, UPDATE, DELETE. At level 6, the redo log is not replayed, so some committed
> transactions may be lost.

## InnoDB Data Dictionary (8.0+)

```sql
-- MySQL 8.0 moved the data dictionary from .frm files to InnoDB system tables
-- This eliminates metadata inconsistencies between .frm and InnoDB

-- Query the data dictionary
SELECT * FROM information_schema.INNODB_TABLES
WHERE NAME LIKE 'myapp_db/%';

SELECT * FROM information_schema.INNODB_COLUMNS
WHERE TABLE_ID = (
  SELECT TABLE_ID FROM information_schema.INNODB_TABLES
  WHERE NAME = 'myapp_db/orders'
);

-- Serialized Dictionary Information (SDI)
-- Each .ibd file contains SDI (JSON metadata for the table)
-- Extract SDI from a .ibd file:
-- ibd2sdi /var/lib/mysql/myapp_db/orders.ibd
```

## InnoDB Monitoring

```sql
-- Comprehensive InnoDB status report
SHOW ENGINE INNODB STATUS\G

-- Key sections to examine:
-- SEMAPHORES: mutex/rw-lock waits (contention indicators)
-- TRANSACTIONS: active transaction list, lock waits
-- FILE I/O: pending I/O operations
-- BUFFER POOL AND MEMORY: hit ratio, dirty pages, free pages
-- ROW OPERATIONS: insert/update/delete/read rates
-- LOG: redo log sequence numbers, checkpoint age

-- Enable InnoDB monitors for detailed output
SET GLOBAL innodb_status_output = ON;        -- standard monitor to error log
SET GLOBAL innodb_status_output_locks = ON;  -- include lock details

-- InnoDB metrics (detailed counters)
SELECT NAME, SUBSYSTEM, COUNT, AVG_COUNT, STATUS
FROM information_schema.INNODB_METRICS
WHERE STATUS = 'enabled'
ORDER BY SUBSYSTEM, NAME;

-- Enable specific metrics
SET GLOBAL innodb_monitor_enable = 'buffer_pool_hit_rate';
SET GLOBAL innodb_monitor_enable = 'log_lsn_checkpoint_age';
```

## Best Practices

- Size `innodb_buffer_pool_size` to 70-80% of available RAM on dedicated database servers.
- Use multiple buffer pool instances (1 per GB, up to ~16) to reduce mutex contention.
- Enable buffer pool dump/load at shutdown/startup to avoid cold-cache performance drops.
- Set `innodb_flush_log_at_trx_commit = 1` for ACID compliance; only use `= 2` when you accept a 1-second data loss window.
- Size redo logs to hold 1-2 hours of peak write traffic to avoid frequent checkpointing.
- Use `innodb_redo_log_capacity` (8.0.30+) instead of the old `innodb_log_file_size` pair.
- Enable `innodb_undo_log_truncate = ON` with at least 2 undo tablespaces to prevent unbounded undo growth.
- Monitor `Innodb_history_list_length`; values above 10,000 indicate purge lag or long-running transactions.
- Keep `innodb_file_per_table = ON` (default) for space reclaim and tablespace portability.
- Never disable the doublewrite buffer unless the filesystem guarantees atomic 16 KB writes (ZFS, DirectFS).
- Reclaim space from large deletes with `ALTER TABLE ... ENGINE=InnoDB` or `OPTIMIZE TABLE`.
- Practice `innodb_force_recovery` procedures in a lab so you know the steps before a real crisis.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Buffer pool too small | Excessive disk I/O; poor query performance; hit ratio below 95% | Set to 70-80% of server RAM; monitor hit ratio |
| `innodb_flush_log_at_trx_commit = 0` in production | Up to 1 second of committed transactions lost on crash | Use `= 1` for ACID; use `= 2` only with informed risk acceptance |
| Redo log undersized | Frequent aggressive checkpointing; write stalls; "async" and "sync" flush warnings in error log | Size redo log for 1-2 hours of peak writes; monitor checkpoint age |
| Long-running transactions | Undo log grows unbounded; purge cannot free old versions; increased disk usage and slower reads | Set `wait_timeout` / `interactive_timeout`; monitor `INNODB_TRX`; kill stale transactions |
| Disabling doublewrite buffer | Partial page writes after crash corrupt data permanently; unrecoverable without backup | Keep `innodb_doublewrite = ON` unless filesystem guarantees atomic writes |
| Using `innodb_force_recovery` without dumping data | Running in force recovery mode permanently; data inconsistencies accumulate | Dump data, rebuild instance from scratch, reload |
| Single undo tablespace | Cannot truncate undo tablespace online; undo space grows without reclaim | Ensure at least 2 undo tablespaces with `innodb_undo_log_truncate = ON` |
| Never monitoring `SHOW ENGINE INNODB STATUS` | Miss critical contention, deadlocks, and performance bottlenecks | Review InnoDB status during performance issues; parse SEMAPHORES section |

## MySQL Version Notes

### MySQL 5.7
- Redo log configured via `innodb_log_file_size` and `innodb_log_files_in_group`; resizing requires shutdown.
- General tablespaces introduced (5.7.6).
- Online buffer pool resizing available.
- Native partitioning for InnoDB (5.7.6+).
- Data dictionary still uses `.frm` files alongside InnoDB internal dictionary.
- Undo tablespaces can be separate but truncation is not automatic by default.

### MySQL 8.0
- Transactional data dictionary replaces `.frm` files (atomic DDL).
- `innodb_redo_log_capacity` (8.0.30+) replaces `innodb_log_file_size` + `innodb_log_files_in_group`; online resizable.
- Dedicated redo log files in `#innodb_redo/` subdirectory (8.0.30+).
- Undo tablespace management: `CREATE/ALTER/DROP UNDO TABLESPACE` DDL.
- Doublewrite buffer in separate files (8.0.20+).
- Session temporary tablespace pool.
- `innodb_dedicated_server` auto-tunes buffer pool, redo log, and flush method based on system memory.
- Parallel read threads for `COUNT(*)` and `CHECK TABLE`.

### MySQL 8.4 / 9.x
- `innodb_log_file_size` and `innodb_log_files_in_group` removed; only `innodb_redo_log_capacity` supported.
- Improved redo log archiving for backup integration.
- Enhanced parallel purge performance.
- Adaptive page flushing improvements.
- Better NUMA-aware buffer pool allocation.
- `innodb_buffer_pool_in_core_file` controls whether buffer pool is included in core dumps.

## Sources

- [MySQL 8.0 Reference: InnoDB Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- [MySQL 8.0 Reference: InnoDB Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)
- [MySQL 8.0 Reference: InnoDB Redo Log](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)
- [MySQL 8.0 Reference: InnoDB Undo Tablespaces](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-tablespaces.html)
- [MySQL 8.0 Reference: InnoDB Recovery](https://dev.mysql.com/doc/refman/8.0/en/innodb-recovery.html)
- [MySQL 8.0 Reference: Forcing InnoDB Recovery](https://dev.mysql.com/doc/refman/8.0/en/forcing-innodb-recovery.html)
- [MySQL 8.4 Reference: InnoDB Storage Engine](https://dev.mysql.com/doc/refman/8.4/en/innodb-storage-engine.html)
