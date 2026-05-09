# MySQL Backup and Recovery — Logical Dumps, Physical Backups, and Point-in-Time Recovery

## Overview

A reliable backup and recovery strategy is the most critical responsibility of any MySQL DBA.
MySQL offers multiple backup approaches: logical backups with mysqldump and mysqlpump, physical
(hot) backups with Percona XtraBackup or MySQL Enterprise Backup, and the clone plugin for
full-instance copies. Each method has trade-offs in speed, lock behavior, storage, and
granularity.

The choice of backup method depends on database size, RPO/RTO requirements, storage engine
(InnoDB vs. others), and whether the instance serves OLTP traffic during the backup window.
For databases under 50-100 GB, logical backups with mysqldump are often sufficient. For larger
databases, physical backup tools are essential to keep backup windows manageable.

Point-in-time recovery (PITR) combines a base backup with binary log replay to restore data
to any specific moment. This is the standard method for recovering from accidental data
deletion or corruption. A backup without tested restore procedures is not a backup at all.

## Key Concepts

- **Logical Backup**: SQL dump files containing CREATE and INSERT statements. Portable across versions and platforms but slower to restore on large datasets.
- **Physical Backup**: Raw copies of InnoDB data files, redo logs, and system tablespace. Fast backup and restore but version- and platform-specific.
- **Hot Backup**: Taken while the database is online and serving traffic, with no (or minimal) locking.
- **Consistent Backup**: All data in the backup reflects a single point in time (MVCC snapshot for InnoDB).
- **PITR (Point-in-Time Recovery)**: Restoring a base backup then replaying binary logs up to a target timestamp or position.
- **Clone Plugin** (8.0.17+): Built-in mechanism to create a physical copy of a local or remote MySQL instance.

## mysqldump — Logical Backup

### Full Database Backup

```bash
# Full backup of all databases (InnoDB-safe, consistent snapshot)
mysqldump --single-transaction --routines --triggers --events \
  --set-gtid-purged=OFF --all-databases \
  -u backup_user -p > /backup/full_backup_$(date +%Y%m%d_%H%M%S).sql

# Full backup with compression
mysqldump --single-transaction --routines --triggers --events \
  --all-databases -u backup_user -p | gzip > /backup/full_$(date +%F).sql.gz

# Record binary log position for PITR capability
mysqldump --single-transaction --master-data=2 --routines --triggers --events \
  --all-databases -u backup_user -p > /backup/full_with_binlog_pos.sql
```

### Partial and Database-Level Backups

```bash
# Single database backup
mysqldump --single-transaction --routines --triggers \
  myapp_db -u backup_user -p > /backup/myapp_db.sql

# Multiple databases
mysqldump --single-transaction --databases myapp_db reporting_db \
  -u backup_user -p > /backup/selected_dbs.sql

# Single table backup
mysqldump --single-transaction myapp_db orders customers \
  -u backup_user -p > /backup/myapp_orders_customers.sql

# Schema only (no data)
mysqldump --no-data --routines --triggers --events \
  myapp_db -u backup_user -p > /backup/myapp_schema.sql

# Data only (no CREATE statements)
mysqldump --no-create-info --single-transaction \
  myapp_db -u backup_user -p > /backup/myapp_data.sql

# WHERE clause filtering
mysqldump --single-transaction myapp_db orders \
  --where="created_at >= '2025-01-01'" \
  -u backup_user -p > /backup/orders_2025.sql
```

### Key mysqldump Options

| Option | Purpose |
|--------|---------|
| `--single-transaction` | Consistent InnoDB snapshot without locking (uses REPEATABLE READ) |
| `--master-data=2` | Writes CHANGE MASTER comment with binlog position |
| `--source-data=2` | Same as above (8.0.26+ terminology) |
| `--set-gtid-purged=OFF` | Omit GTID info (use for non-replication restores) |
| `--routines` | Include stored procedures and functions |
| `--triggers` | Include triggers (default ON) |
| `--events` | Include scheduled events |
| `--quick` | Fetch rows one at a time (default; reduces memory for large tables) |
| `--max-allowed-packet=256M` | Handle large BLOBs/TEXT fields |
| `--hex-blob` | Dump binary columns as hex (safe for transport) |
| `--tab=/dir` | Dump separate .sql (schema) and .txt (data) per table |

## mysqlpump — Parallel Logical Backup

```bash
# Parallel dump with 4 threads
mysqlpump --default-parallelism=4 --single-transaction \
  --all-databases -u backup_user -p > /backup/pump_full.sql

# Exclude specific databases
mysqlpump --default-parallelism=4 --single-transaction \
  --exclude-databases=mysql,sys,performance_schema,information_schema \
  -u backup_user -p > /backup/pump_app_dbs.sql

# Parallel with compression
mysqlpump --default-parallelism=4 --single-transaction --compress-output=zlib \
  --all-databases -u backup_user -p > /backup/pump_full.zlib

# Specific databases with per-database parallelism
mysqlpump --parallel-schemas=4:myapp_db --parallel-schemas=2:reporting_db \
  --single-transaction -u backup_user -p > /backup/pump_selected.sql
```

> **Note**: `mysqlpump` is deprecated in MySQL 8.0.34 and removed in 8.4. Use `mysqldump`
> or `mysql-shell`'s `util.dumpInstance()` / `util.dumpSchemas()` for new deployments.

## Percona XtraBackup — Physical Hot Backup

### Full Backup

```bash
# Full backup
xtrabackup --backup --target-dir=/backup/full_$(date +%F) \
  --user=backup_user --password='BackupP@ss!'

# Full backup with compression
xtrabackup --backup --compress --compress-threads=4 \
  --target-dir=/backup/full_compressed_$(date +%F) \
  --user=backup_user --password='BackupP@ss!'

# Full backup streamed to remote host
xtrabackup --backup --stream=xbstream --user=backup_user --password='BackupP@ss!' | \
  ssh backup_server "cat > /backup/full_$(date +%F).xbstream"
```

### Incremental Backup

```bash
# Sunday: full backup
xtrabackup --backup --target-dir=/backup/base \
  --user=backup_user --password='BackupP@ss!'

# Monday: incremental based on full
xtrabackup --backup --target-dir=/backup/inc1 \
  --incremental-basedir=/backup/base \
  --user=backup_user --password='BackupP@ss!'

# Tuesday: incremental based on Monday
xtrabackup --backup --target-dir=/backup/inc2 \
  --incremental-basedir=/backup/inc1 \
  --user=backup_user --password='BackupP@ss!'
```

### Preparing and Restoring XtraBackup

```bash
# Prepare full backup (apply redo log)
xtrabackup --prepare --target-dir=/backup/base

# For incremental: apply incrementals to base
xtrabackup --prepare --apply-log-only --target-dir=/backup/base
xtrabackup --prepare --apply-log-only --target-dir=/backup/base \
  --incremental-dir=/backup/inc1
xtrabackup --prepare --target-dir=/backup/base \
  --incremental-dir=/backup/inc2

# Stop MySQL, clear datadir, restore
systemctl stop mysqld
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/base
chown -R mysql:mysql /var/lib/mysql
systemctl start mysqld
```

### XtraBackup Version Compatibility

| XtraBackup Version | MySQL Versions Supported |
|--------------------|--------------------------|
| 2.4.x              | MySQL 5.6, 5.7           |
| 8.0.x              | MySQL 8.0                |
| 8.4.x              | MySQL 8.4                |

## MySQL Enterprise Backup (MEB)

```bash
# Full backup
mysqlbackup --user=backup_user --password='BackupP@ss!' \
  --backup-dir=/backup/meb_full backup-and-apply-log

# Incremental backup
mysqlbackup --user=backup_user --password='BackupP@ss!' \
  --incremental --incremental-base=history:last_backup \
  --backup-dir=/backup/meb_inc1 backup

# Compressed backup
mysqlbackup --user=backup_user --password='BackupP@ss!' \
  --compress --backup-dir=/backup/meb_compressed backup-and-apply-log

# Restore
mysqlbackup --backup-dir=/backup/meb_full copy-back-and-apply-log
```

## Clone Plugin (MySQL 8.0.17+)

```sql
-- Install clone plugin
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

-- Local clone (creates a complete copy of the instance)
CLONE LOCAL DATA DIRECTORY = '/backup/clone_$(date +%F)';

-- Remote clone (copies data from a donor instance)
-- On recipient:
SET GLOBAL clone_valid_donor_list = 'donor_host:3306';
CLONE INSTANCE FROM 'clone_user'@'donor_host':3306
  IDENTIFIED BY 'CloneP@ss!';

-- Monitor clone progress
SELECT * FROM performance_schema.clone_status\G
SELECT * FROM performance_schema.clone_progress;
```

## Point-in-Time Recovery (PITR)

### Using Binary Logs

```bash
# Step 1: Identify the binlog position from the backup
head -50 /backup/full_with_binlog_pos.sql | grep "CHANGE MASTER"
# -- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000042', MASTER_LOG_POS=154;

# Step 2: Restore the base backup
mysql -u root -p < /backup/full_with_binlog_pos.sql

# Step 3: Identify the stop point (e.g., just before the bad DROP TABLE)
mysqlbinlog --start-position=154 /var/lib/mysql/binlog.000042 | \
  grep -B5 "DROP TABLE" | head -20

# Step 4: Replay binlog up to the target position
mysqlbinlog --start-position=154 --stop-position=52638 \
  /var/lib/mysql/binlog.000042 | mysql -u root -p

# PITR using timestamps
mysqlbinlog --start-datetime="2025-06-15 00:00:00" \
  --stop-datetime="2025-06-15 14:30:00" \
  /var/lib/mysql/binlog.000042 /var/lib/mysql/binlog.000043 | \
  mysql -u root -p

# PITR with GTID-based binlogs
mysqlbinlog --include-gtids="3cee38b3-...:1-1542" \
  --exclude-gtids="3cee38b3-...:1543" \
  /var/lib/mysql/binlog.000042 | mysql -u root -p
```

### PITR Workflow

1. Restore the most recent full backup.
2. Identify the exact position or timestamp of the unwanted event.
3. Replay binary logs from the backup's binlog position to just before the event.
4. Optionally skip the bad event and continue replaying the remaining logs.

## MySQL Shell Dump/Load Utilities (8.0+)

```bash
# Full instance dump (parallel, compressed)
mysqlsh root@localhost -- util dump-instance /backup/shell_dump \
  --threads=8 --compression=zstd

# Schema-level dump
mysqlsh root@localhost -- util dump-schemas myapp_db,reporting_db \
  --outputUrl=/backup/schema_dump --threads=4

# Table-level dump
mysqlsh root@localhost -- util dump-tables myapp_db orders,customers \
  --outputUrl=/backup/table_dump

# Load dump (parallel)
mysqlsh root@localhost -- util load-dump /backup/shell_dump \
  --threads=8 --resetProgress
```

## Backup Verification

```bash
# Verify logical backup by restoring to a test instance
mysql -u root -p -h test_host test_restore < /backup/full_backup.sql

# Check row counts after restore
mysql -u root -p -h test_host -e "
SELECT table_schema, table_name, table_rows
FROM information_schema.TABLES
WHERE table_schema = 'myapp_db'
ORDER BY table_name;"

# Verify XtraBackup with checksum
xtrabackup --prepare --target-dir=/backup/base 2>&1 | tail -5
# Look for: "completed OK!"

# Check backup file integrity
gunzip -t /backup/full_backup.sql.gz && echo "OK" || echo "CORRUPTED"

# Automated backup validation script
mysqlcheck -u root -p --all-databases --check --extended
```

### Backup Monitoring Query

```sql
-- Check last backup status (if using backup_history table)
SELECT backup_id, start_time, end_time,
       TIMEDIFF(end_time, start_time) AS duration,
       backup_type, status, binlog_file, binlog_pos
FROM mysql.backup_history
ORDER BY start_time DESC
LIMIT 5;
```

## Backup User Setup

```sql
-- Create dedicated backup user with minimum required privileges
CREATE USER 'backup_user'@'localhost'
  IDENTIFIED BY 'BackupSecureP@ss!'
  PASSWORD EXPIRE NEVER;

-- For mysqldump
GRANT SELECT, SHOW VIEW, TRIGGER, EVENT, RELOAD,
      LOCK TABLES, REPLICATION CLIENT, PROCESS
ON *.* TO 'backup_user'@'localhost';

-- Additional for XtraBackup
GRANT BACKUP_ADMIN, REPLICATION CLIENT, PROCESS,
      SELECT, RELOAD, LOCK TABLES
ON *.* TO 'backup_user'@'localhost';

-- Additional for clone plugin
GRANT BACKUP_ADMIN, CLONE_ADMIN ON *.* TO 'backup_user'@'localhost';

FLUSH PRIVILEGES;
```

## Best Practices

- Automate daily backups with cron; run full backups weekly and incrementals daily.
- Always use `--single-transaction` for InnoDB-based mysqldump backups to avoid table locks.
- Include `--routines`, `--triggers`, and `--events` in logical backups; they are easy to forget.
- Record the binary log position with every backup (`--master-data=2` or `--source-data=2`).
- Compress backups to save storage; use `zstd` (MySQL Shell) or `gzip`/`lz4` (XtraBackup) based on CPU/space trade-offs.
- Test restores regularly on a non-production instance; an untested backup is not a backup.
- Keep binary logs for at least 7 days (`binlog_expire_logs_seconds = 604800`) to allow PITR.
- Store backups off-host; use a dedicated backup server, NFS, or cloud storage (S3, GCS).
- Encrypt backup files at rest, especially when stored off-site.
- Monitor backup job exit codes and file sizes; alert on failures or anomalous sizes.
- Document your restore procedure and practice it at least quarterly.
- For databases > 100 GB, prefer XtraBackup or MySQL Shell dump/load over mysqldump.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not using `--single-transaction` with mysqldump | Global read lock acquired; blocks all writes during backup | Always add `--single-transaction` for InnoDB tables |
| Forgetting `--routines` and `--events` | Stored procedures, functions, and scheduled events missing from backup | Include `--routines --triggers --events` in every logical backup |
| Not testing restores | Discover backup is corrupt or incomplete only during a real outage | Schedule monthly restore tests to a staging instance |
| Mixing XtraBackup and MySQL major versions | XtraBackup 2.4 cannot back up MySQL 8.0; version mismatch causes silent corruption | Match XtraBackup version to MySQL version (2.4 for 5.7, 8.0.x for 8.0) |
| Purging binary logs before backup is safe | Lose ability to do point-in-time recovery between backups | Set `binlog_expire_logs_seconds` appropriately; never manually purge without checking replica lag |
| Running mysqldump with MyISAM tables without `--lock-all-tables` | Inconsistent backup of MyISAM tables (they do not support transactions) | Use `--lock-all-tables` when MyISAM tables exist, or convert to InnoDB |
| Storing backups only on the same host | Disk failure or host loss destroys both database and backups | Replicate backups to remote storage immediately after completion |
| Ignoring backup size trends | Gradual backup bloat indicates data growth; sudden size drop may mean missing data | Monitor and alert on backup file sizes; compare against expected ranges |

## MySQL Version Notes

### MySQL 5.7
- mysqldump and mysqlpump are the primary built-in tools.
- XtraBackup 2.4 for physical backups.
- `--master-data` option for binlog position recording.
- No clone plugin.
- `expire_logs_days` (integer days) for binlog expiration.

### MySQL 8.0
- Clone plugin available (8.0.17+); useful for provisioning replicas and fast restores.
- MySQL Shell `util.dumpInstance()` and `util.loadDump()` for parallel, chunked, compressed dumps.
- `--source-data` replaces `--master-data` (8.0.26+).
- `binlog_expire_logs_seconds` replaces `expire_logs_days` (both work; prefer seconds).
- `mysqlpump` deprecated in 8.0.34.
- XtraBackup 8.0.x required (incompatible with 2.4).

### MySQL 8.4 / 9.x
- `mysqlpump` removed; use MySQL Shell dump utilities or mysqldump.
- Clone plugin improvements for faster provisioning.
- XtraBackup 8.4 required for MySQL 8.4 compatibility.
- Enhanced backup progress monitoring in performance_schema.
- `--master-data` fully deprecated; use `--source-data`.

## Sources

- [MySQL 8.0 Reference: mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)
- [MySQL 8.0 Reference: Clone Plugin](https://dev.mysql.com/doc/refman/8.0/en/clone-plugin.html)
- [MySQL 8.0 Reference: PITR](https://dev.mysql.com/doc/refman/8.0/en/point-in-time-recovery.html)
- [Percona XtraBackup Documentation](https://docs.percona.com/percona-xtrabackup/8.0/)
- [MySQL Shell Dump Utilities](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-dump-instance-schema.html)
- [MySQL 8.4 Reference: Backup and Recovery](https://dev.mysql.com/doc/refman/8.4/en/backup-and-recovery.html)
