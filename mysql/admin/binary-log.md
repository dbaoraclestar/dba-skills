# MySQL Binary Log — Formats, Configuration, mysqlbinlog Tool, PITR, and Encryption

## Overview

The MySQL binary log (binlog) records all data-modifying statements and row changes made to the
database. It serves three critical purposes: point-in-time recovery (PITR), replication (source
ships binlog events to replicas), and auditing (forensic analysis of what changed and when).
Every production MySQL instance should have binary logging enabled.

Binary logs are written sequentially as the server processes transactions. Each binlog file
contains a series of events describing individual changes. The server rotates to a new file when
the current file reaches `max_binlog_size` (default 1 GB) or when a `FLUSH BINARY LOGS`
command is issued. Old files are automatically purged based on retention settings.

Understanding binlog formats (ROW, STATEMENT, MIXED), the `mysqlbinlog` extraction tool, and
the mechanics of binary log-based recovery is essential for any MySQL DBA. MySQL 8.0 added
binlog compression and encryption capabilities for storage efficiency and security.

## Key Concepts

- **Binary Log File**: Sequential file (e.g., `binlog.000001`) containing a stream of binlog events.
- **Binary Log Index**: File (e.g., `binlog.index`) listing all active binlog files.
- **Binlog Event**: Atomic unit in the binary log (Query_event, Write_rows, Update_rows, Delete_rows, GTID_event, etc.).
- **Binlog Format**: Determines how changes are recorded: ROW (row images), STATEMENT (SQL text), or MIXED (hybrid).
- **Position**: Byte offset within a binlog file; used for position-based replication and PITR.
- **GTID**: Global Transaction Identifier; uniquely identifies each transaction across all servers in a replication topology.

## Binlog Formats

### ROW Format (Recommended)

Records the actual row changes (before/after images). Deterministic and safe for all operations.

```sql
SET GLOBAL binlog_format = 'ROW';
-- or in my.cnf:
-- binlog_format = ROW
```

**Advantages**: Deterministic; handles non-deterministic functions (UUID(), RAND(), NOW()) correctly; safer for replication.
**Disadvantages**: Larger binlog size for bulk operations; harder to read raw events.

### STATEMENT Format

Records the SQL statement as executed. Compact but non-deterministic in some cases.

```sql
SET GLOBAL binlog_format = 'STATEMENT';
```

**Advantages**: Compact; human-readable in binlog.
**Disadvantages**: Non-deterministic functions produce different results on replica; some statements are unsafe.

### MIXED Format

Uses STATEMENT by default, automatically switches to ROW for statements that are not safe for statement-based replication.

```sql
SET GLOBAL binlog_format = 'MIXED';
```

### Row Image Control

```sql
-- Controls how much row data is logged in ROW format
SHOW VARIABLES LIKE 'binlog_row_image';

-- FULL (default): log all columns in before/after images
SET GLOBAL binlog_row_image = 'FULL';

-- MINIMAL: log only PK + changed columns (smaller binlog, but less info for forensics)
SET GLOBAL binlog_row_image = 'MINIMAL';

-- NOBLOB: like FULL but excludes unchanged BLOB/TEXT columns
SET GLOBAL binlog_row_image = 'NOBLOB';
```

## Enabling and Configuring Binary Logs

### Basic Configuration

```ini
[mysqld]
# Enable binary logging (on by default in 8.0)
log-bin                   = /var/lib/mysql/binlog

# Format (ROW recommended)
binlog-format             = ROW
binlog-row-image          = FULL

# Sync binlog to disk on every commit (safest; slight perf cost)
sync-binlog               = 1

# Maximum size per binlog file (default 1GB)
max-binlog-size           = 1G

# Retention: auto-purge binlogs older than 7 days
# 8.0+ (preferred):
binlog_expire_logs_seconds = 604800
# 5.7 (deprecated in 8.0):
expire_logs_days           = 7

# Server ID (required for binlog, even without replication)
server-id                 = 1
```

### Checking Binlog Status

```sql
-- Verify binlog is enabled
SHOW VARIABLES LIKE 'log_bin';

-- Current binlog file and position
SHOW BINARY LOG STATUS\G
-- or pre-8.0.22:
SHOW MASTER STATUS\G

-- List all binlog files
SHOW BINARY LOGS;

-- Binlog format in use
SHOW VARIABLES LIKE 'binlog_format';

-- Check retention settings
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'expire_logs_days';

-- Space used by binlogs
SELECT
  ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2) AS binlog_size_gb,
  COUNT(*) AS num_files
FROM performance_schema.binary_log_status;

-- Or from the filesystem:
-- du -sh /var/lib/mysql/binlog.*
```

### Rotating and Purging Binary Logs

```sql
-- Force rotation to a new binlog file
FLUSH BINARY LOGS;

-- Purge binlogs older than a specific date
PURGE BINARY LOGS BEFORE '2025-06-01 00:00:00';

-- Purge all binlogs before a specific file
PURGE BINARY LOGS TO 'binlog.000050';

-- Change retention at runtime
SET GLOBAL binlog_expire_logs_seconds = 259200;  -- 3 days

-- Check that replicas have consumed events before purging
SHOW REPLICAS;
-- Verify Replica_IO_State and position for each replica
```

> **Warning**: Never purge binary logs that have not yet been consumed by all replicas.
> Check `SHOW REPLICA STATUS` on all replicas before purging.

## mysqlbinlog Tool

### Basic Usage

```bash
# Display events from a binlog file
mysqlbinlog /var/lib/mysql/binlog.000042

# Display events in a human-readable format (decode ROW events)
mysqlbinlog --verbose /var/lib/mysql/binlog.000042

# Double verbose: show column types and metadata
mysqlbinlog -vv /var/lib/mysql/binlog.000042

# Filter by database
mysqlbinlog --database=myapp_db /var/lib/mysql/binlog.000042

# Filter by position range
mysqlbinlog --start-position=154 --stop-position=52638 \
  /var/lib/mysql/binlog.000042

# Filter by time range
mysqlbinlog --start-datetime="2025-06-15 10:00:00" \
  --stop-datetime="2025-06-15 14:30:00" \
  /var/lib/mysql/binlog.000042

# Read remote binlog from a running server
mysqlbinlog --read-from-remote-server --host=10.0.0.1 --port=3306 \
  --user=repl_user --password='ReplP@ss!' binlog.000042

# Output to file for later replay
mysqlbinlog --verbose /var/lib/mysql/binlog.000042 > /tmp/binlog_events.sql
```

### Analyzing Specific Events

```bash
# Find a DROP TABLE event
mysqlbinlog --verbose /var/lib/mysql/binlog.000042 | grep -B5 -A5 "DROP TABLE"

# Find large transactions (many row events)
mysqlbinlog --verbose /var/lib/mysql/binlog.000042 | \
  grep -c "### INSERT INTO\|### UPDATE\|### DELETE FROM"

# Find events affecting a specific table
mysqlbinlog --verbose /var/lib/mysql/binlog.000042 | \
  grep -A10 "### INSERT INTO \`myapp_db\`.\`orders\`"

# Show GTID events
mysqlbinlog /var/lib/mysql/binlog.000042 | grep "SET @@SESSION.GTID_NEXT"

# Extract specific GTID transactions
mysqlbinlog --include-gtids="3cee38b3-abcd-1234-5678-aabbccddeeff:100-150" \
  /var/lib/mysql/binlog.000042

# Exclude specific GTID transactions (e.g., skip a bad transaction)
mysqlbinlog --exclude-gtids="3cee38b3-abcd-1234-5678-aabbccddeeff:142" \
  /var/lib/mysql/binlog.000042
```

### Replaying Binary Logs

```bash
# Replay all events from a binlog file
mysqlbinlog /var/lib/mysql/binlog.000042 | mysql -u root -p

# Replay a range of positions
mysqlbinlog --start-position=154 --stop-position=52638 \
  /var/lib/mysql/binlog.000042 | mysql -u root -p

# Replay multiple binlog files in sequence
mysqlbinlog /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 \
  /var/lib/mysql/binlog.000044 | mysql -u root -p

# Replay with specific database filter
mysqlbinlog --database=myapp_db /var/lib/mysql/binlog.000042 | mysql -u root -p

# Replay to a specific timestamp (PITR)
mysqlbinlog --stop-datetime="2025-06-15 14:30:00" \
  /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 | mysql -u root -p
```

## Binlog Event Types

| Event Type | Description |
|------------|-------------|
| `FORMAT_DESCRIPTION_EVENT` | Binlog file header; describes the binlog format version |
| `QUERY_EVENT` | SQL statement (DDL, or DML in STATEMENT format) |
| `TABLE_MAP_EVENT` | Maps table ID to database.table name (precedes row events) |
| `WRITE_ROWS_EVENT` | INSERT in ROW format |
| `UPDATE_ROWS_EVENT` | UPDATE in ROW format (before + after images) |
| `DELETE_ROWS_EVENT` | DELETE in ROW format |
| `XID_EVENT` | Transaction commit marker (XA transaction ID) |
| `GTID_LOG_EVENT` | GTID assignment for the following transaction |
| `PREVIOUS_GTIDS_LOG_EVENT` | Set of all GTIDs in previous binlog files |
| `ROTATE_EVENT` | Points to the next binlog file |
| `ANONYMOUS_GTID_LOG_EVENT` | Transaction without GTID (GTID mode OFF) |
| `TRANSACTION_PAYLOAD_EVENT` | Compressed transaction payload (8.0.20+) |

## Point-in-Time Recovery Using Binlog

### Step-by-Step PITR Procedure

```bash
# Scenario: Accidental DROP TABLE at 14:32:15 on June 15, 2025

# Step 1: Restore the most recent full backup
mysql -u root -p < /backup/full_backup_20250615_020000.sql

# Step 2: Identify the backup's binlog position
head -50 /backup/full_backup_20250615_020000.sql | grep "CHANGE MASTER\|SOURCE_LOG"
# -- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000042', MASTER_LOG_POS=154;

# Step 3: Find the exact position of the DROP TABLE
mysqlbinlog --start-position=154 --verbose \
  /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 | \
  grep -B20 "DROP TABLE"
# Note the position BEFORE the DROP TABLE event, e.g., at-position 487293

# Step 4: Replay binlog up to just before the DROP TABLE
mysqlbinlog --start-position=154 --stop-position=487293 \
  /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 | \
  mysql -u root -p

# Step 5 (optional): Skip the DROP TABLE and continue replaying
# Find the position AFTER the DROP TABLE event, e.g., 487412
mysqlbinlog --start-position=487412 \
  /var/lib/mysql/binlog.000043 /var/lib/mysql/binlog.000044 | \
  mysql -u root -p
```

### PITR with GTIDs

```bash
# Find the GTID of the bad transaction
mysqlbinlog --verbose /var/lib/mysql/binlog.000043 | \
  grep -B5 "DROP TABLE" | grep "GTID_NEXT"
# SET @@SESSION.GTID_NEXT= '3cee38b3-...:1543'

# Replay all events EXCEPT the bad GTID
mysqlbinlog --exclude-gtids="3cee38b3-...:1543" \
  /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 \
  /var/lib/mysql/binlog.000044 | mysql -u root -p
```

## Binlog Compression (8.0.20+)

```sql
-- Enable binlog transaction compression
SET GLOBAL binlog_transaction_compression = ON;

-- Set compression algorithm (only zstd supported currently)
SET GLOBAL binlog_transaction_compression_level_zstd = 3;  -- 1-22, default 3

-- Check compression status
SHOW STATUS LIKE 'Binlog_transaction_compression%';

-- Compression stats
SELECT * FROM performance_schema.binary_log_transaction_compression_stats\G
```

```ini
[mysqld]
# Enable in my.cnf
binlog_transaction_compression = ON
binlog_transaction_compression_level_zstd = 3
```

**Benefits**: 50-70% reduction in binlog size for typical workloads; reduces disk I/O, network
bandwidth for replication, and storage costs.

**Notes**: Compression happens at the transaction level. Compressed events use the
`TRANSACTION_PAYLOAD_EVENT` type. `mysqlbinlog` transparently decompresses when reading.

## Binlog Encryption (8.0.14+)

```sql
-- Prerequisites: keyring plugin must be installed and configured
-- Install keyring_file plugin (for testing; use keyring_encrypted_file or KMS in production)
-- In my.cnf: early-plugin-load=keyring_file.so

-- Enable binlog encryption
SET GLOBAL binlog_encryption = ON;

-- Verify encryption status
SHOW VARIABLES LIKE 'binlog_encryption';
SHOW BINARY LOG STATUS\G
-- Encrypted: Yes

-- Rotate the encryption key
ALTER INSTANCE ROTATE BINLOG MASTER KEY;

-- Note: mysqlbinlog requires --read-from-remote-server to read encrypted binlogs
-- Local file reading will not work with encrypted files
mysqlbinlog --read-from-remote-server --host=localhost \
  --user=admin --password='AdminP@ss!' binlog.000042
```

```ini
[mysqld]
# Binlog encryption configuration
early-plugin-load  = keyring_file.so
keyring_file_data  = /var/lib/mysql-keyring/keyring
binlog_encryption  = ON
```

## Binlog Monitoring and Diagnostics

```sql
-- Binlog cache usage (should have near-zero disk use)
SHOW STATUS LIKE 'Binlog_cache%';
-- Binlog_cache_use: total transactions using the binlog cache
-- Binlog_cache_disk_use: transactions that exceeded cache and wrote to disk

-- If disk_use is high, increase binlog_cache_size
SHOW VARIABLES LIKE 'binlog_cache_size';
SET GLOBAL binlog_cache_size = 1048576;  -- 1 MB

-- Statement cache for non-transactional statements
SHOW STATUS LIKE 'Binlog_stmt_cache%';

-- Binlog space usage
SHOW BINARY LOGS;

-- Count events per type in a binlog file (from shell)
-- mysqlbinlog --verbose binlog.000042 | grep "^#" | awk '{print $6}' | sort | uniq -c | sort -rn

-- Monitor binlog writes per second
SELECT VARIABLE_VALUE FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Com_insert';

-- Check binlog group commit effectiveness
SHOW STATUS LIKE 'Binlog_group_commit%';
```

### Binlog-Related System Variables

```sql
-- Important binlog variables
SHOW VARIABLES WHERE Variable_name IN (
  'log_bin',
  'binlog_format',
  'binlog_row_image',
  'sync_binlog',
  'max_binlog_size',
  'binlog_expire_logs_seconds',
  'binlog_cache_size',
  'binlog_stmt_cache_size',
  'binlog_transaction_compression',
  'binlog_encryption',
  'binlog_checksum',
  'binlog_order_commits',
  'binlog_group_commit_sync_delay',
  'binlog_group_commit_sync_no_delay_count'
);
```

## Best Practices

- Always enable binary logging in production (`log-bin`); it is essential for PITR and replication.
- Use `binlog_format = ROW` for deterministic, safe replication and reliable PITR.
- Set `sync_binlog = 1` for crash-safe binary logging (writes are flushed to disk on each commit).
- Set `binlog_expire_logs_seconds` appropriately (7 days minimum); balance between PITR window and disk space.
- Never manually delete binlog files from the filesystem; always use `PURGE BINARY LOGS`.
- Verify all replicas have consumed events before purging; check `SHOW REPLICA STATUS` on each replica.
- Enable binlog compression (8.0.20+) to reduce storage and replication bandwidth.
- Enable binlog encryption in environments with compliance requirements (PCI, HIPAA).
- Monitor `Binlog_cache_disk_use`; if it grows, increase `binlog_cache_size`.
- Use `binlog_group_commit_sync_delay` to batch commits and improve throughput on high-TPS systems.
- Keep `mysqlbinlog` tool version matched to the server version for compatibility.
- Test PITR procedures regularly; document the exact steps for your environment.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not enabling binary logging | No point-in-time recovery possible; no replication | Set `log-bin` in `my.cnf`; enabled by default in 8.0 |
| Using `binlog_format = STATEMENT` in production | Non-deterministic functions cause data drift on replicas; unsafe for PITR | Switch to `binlog_format = ROW` |
| Setting `sync_binlog = 0` | Binary log events can be lost on crash; committed transactions may be unreplayable | Use `sync_binlog = 1` for durability |
| Manually deleting binlog files with `rm` | Binlog index becomes inconsistent; replication and `SHOW BINARY LOGS` break | Use `PURGE BINARY LOGS` SQL command |
| Purging binlogs without checking replicas | Replica IO thread cannot find the next event; replication breaks | Always verify replica positions before purging |
| Setting retention too short | Cannot do PITR beyond the retention window | Set `binlog_expire_logs_seconds` to at least 7 days (604800) |
| Not backing up binlogs | If the database host fails, binlogs are lost and PITR gap exists | Ship binlogs to a backup server in near-real-time |
| Ignoring `Binlog_cache_disk_use` | Large transactions spill to disk, causing I/O spikes | Monitor and increase `binlog_cache_size` if disk use is frequent |

## MySQL Version Notes

### MySQL 5.7
- Binary logging is OFF by default; must explicitly set `log-bin`.
- `expire_logs_days` for retention (integer days only).
- No binlog compression.
- No binlog encryption.
- `SHOW MASTER STATUS` / `SHOW SLAVE STATUS` terminology.
- `binlog_format` default is `STATEMENT` unless explicitly set.

### MySQL 8.0
- Binary logging is ON by default.
- `binlog_format` default is `ROW`.
- `binlog_expire_logs_seconds` replaces `expire_logs_days` (both work; prefer seconds).
- Binlog transaction compression (8.0.20+): `binlog_transaction_compression = ON`.
- Binlog encryption (8.0.14+): `binlog_encryption = ON` with keyring plugin.
- `SHOW BINARY LOG STATUS` replaces `SHOW MASTER STATUS` (8.0.22+).
- `binlog_transaction_dependency_tracking` for parallel applier optimization.
- New event type `TRANSACTION_PAYLOAD_EVENT` for compressed transactions.
- `mysqlbinlog --read-from-remote-server` required for encrypted binlogs.

### MySQL 8.4 / 9.x
- `expire_logs_days` removed; only `binlog_expire_logs_seconds` supported.
- `SHOW MASTER STATUS` removed; use `SHOW BINARY LOG STATUS`.
- Enhanced binlog compression with better `zstd` integration.
- Component-based keyring replaces plugin-based keyring for encryption.
- `binlog_format` variable deprecated (ROW is the only supported format going forward).
- Improved `mysqlbinlog` output formatting for ROW events.

## Sources

- [MySQL 8.0 Reference: Binary Log](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)
- [MySQL 8.0 Reference: mysqlbinlog](https://dev.mysql.com/doc/refman/8.0/en/mysqlbinlog.html)
- [MySQL 8.0 Reference: Binlog Transaction Compression](https://dev.mysql.com/doc/refman/8.0/en/binary-log-transaction-compression.html)
- [MySQL 8.0 Reference: Encrypting Binary Log Files](https://dev.mysql.com/doc/refman/8.0/en/replication-binlog-encryption.html)
- [MySQL 8.0 Reference: PITR Using Binlog](https://dev.mysql.com/doc/refman/8.0/en/point-in-time-recovery-binlog.html)
- [MySQL 8.4 Reference: Binary Log](https://dev.mysql.com/doc/refman/8.4/en/binary-log.html)
