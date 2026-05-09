# MySQL Upgrade and Migration — Version Upgrade Paths, Pre-checks, and Post-Validation

## Overview

MySQL version upgrades are necessary to stay on supported releases, gain performance improvements,
access new features, and receive security patches. The MySQL release model has evolved: 5.7 was
the last "traditional" GA release, 8.0 introduced the continuous release model, 8.4 is a
Long-Term Support (LTS) release, and 9.x follows the Innovation release track with shorter
support windows.

Upgrade planning requires understanding the supported upgrade paths, changes in SQL behavior,
authentication plugin defaults, removed features, and reserved word additions. The two main
upgrade strategies are in-place upgrade (upgrade binaries, start the server, let it auto-upgrade)
and logical dump/reload (dump data with mysqldump, install the new version, reload). Each has
trade-offs in downtime, risk, and complexity.

This document covers the complete upgrade lifecycle: pre-upgrade checks, execution, post-upgrade
validation, and rollback planning. It applies to standalone instances, replicas, and InnoDB
Cluster/Group Replication topologies.

## Key Concepts

- **In-Place Upgrade**: Replace MySQL binaries, start the new version against existing data files. The server performs an automatic data dictionary upgrade.
- **Logical Dump/Reload**: Export all data with mysqldump (old version), install new version on clean datadir, import the dump. Safest but requires more downtime for large databases.
- **mysql_upgrade**: Tool that checks and upgrades system tables. Deprecated in 8.0.16+ (server now auto-upgrades at startup).
- **upgrade_checker**: MySQL Shell utility (`util.checkForServerUpgrade()`) that reports incompatibilities before upgrading.
- **LTS Release**: Long-Term Support release (8.4) with 8+ years of support; recommended for production.
- **Innovation Release**: Short-lived release (9.0, 9.1, etc.) with new features and shorter support cycles.

## Supported Upgrade Paths

### Official MySQL Upgrade Matrix

| From | To | Method | Notes |
|------|----|--------|-------|
| 5.7 | 8.0 | In-place or logical | Only supported path from 5.7. Cannot skip to 8.4 directly |
| 8.0.x | 8.0.y (y > x) | In-place | Minor version upgrades within 8.0 |
| 8.0 | 8.4 (LTS) | In-place or logical | Recommended production path |
| 8.0 | 9.x | In-place or logical | Innovation track; shorter support |
| 8.4 | 9.x | In-place or logical | LTS to Innovation |
| 9.x | 9.y (y > x) | In-place | Within Innovation track |

> **Critical Rule**: MySQL only supports upgrading from one GA release to the next GA release.
> You cannot skip major versions (e.g., 5.7 directly to 8.4 is NOT supported). Upgrade
> 5.7 -> 8.0 -> 8.4 in sequence.

### Downgrade Limitations

MySQL does NOT support in-place downgrades. If an upgrade fails:
- Restore from pre-upgrade backup (preferred).
- Use logical dump taken before upgrade and reload on old version.
- For replication topologies, promote a not-yet-upgraded replica.

## Pre-Upgrade Checks

### MySQL Shell Upgrade Checker (Recommended)

```bash
# Install MySQL Shell if not present
# On RHEL/CentOS:
yum install mysql-shell

# On Debian/Ubuntu:
apt install mysql-shell

# Run the upgrade checker (5.7 -> 8.0)
mysqlsh -- util check-for-server-upgrade \
  --target-version=8.0.40 \
  root@localhost:3306

# From within MySQL Shell:
mysqlsh root@localhost:3306
```

```javascript
// Inside MySQL Shell (JavaScript mode)
util.checkForServerUpgrade('root@localhost:3306', {
  targetVersion: '8.0.40',
  outputFormat: 'TEXT'
});

// Check for 8.4 upgrade
util.checkForServerUpgrade('root@localhost:3306', {
  targetVersion: '8.4.0',
  outputFormat: 'TEXT'
});

// Save report to file
util.checkForServerUpgrade('root@localhost:3306', {
  targetVersion: '8.0.40',
  outputFormat: 'JSON',
  output: '/tmp/upgrade_report.json'
});
```

### What the Upgrade Checker Validates

- Usage of removed or deprecated SQL syntax and functions.
- Tables using storage engines other than InnoDB (MyISAM, etc.).
- Use of deprecated `utf8mb3` charset and recommendation to migrate to `utf8mb4`.
- Column names that are new reserved words in the target version.
- Incompatible `sql_mode` settings.
- `mysql_native_password` plugin usage (deprecated/removed in 8.4).
- Orphaned `.frm` files without corresponding InnoDB tables.
- Partitioned tables using non-native partitioning.
- Foreign key constraint name length issues.
- Circular directory references for tablespace files.
- `ZEROFILL` and `display width` usage for integer types (deprecated in 8.0).

### Manual Pre-Upgrade Checks

```sql
-- 1. Check current version
SELECT VERSION();

-- 2. Check storage engines in use
SELECT ENGINE, COUNT(*) AS table_count,
       ROUND(SUM(DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS size_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
GROUP BY ENGINE;

-- 3. Find tables using MyISAM (convert to InnoDB before upgrade)
SELECT TABLE_SCHEMA, TABLE_NAME, ENGINE
FROM information_schema.TABLES
WHERE ENGINE = 'MyISAM'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 4. Check for deprecated sql_mode values
SELECT @@sql_mode;
-- Removed in 8.0: NO_AUTO_CREATE_USER, DB2, MAXDB, MSSQL, MYSQL323, MYSQL40,
-- ORACLE, POSTGRESQL, NO_FIELD_OPTIONS, NO_KEY_OPTIONS, NO_TABLE_OPTIONS

-- 5. Find columns using new reserved words (8.0 additions)
-- Key 8.0 reserved words: CUME_DIST, DENSE_RANK, EMPTY, EXCEPT, FIRST_VALUE,
-- GROUPING, GROUPS, JSON_TABLE, LAG, LAST_VALUE, LATERAL, LEAD, NTH_VALUE,
-- NTILE, OF, OVER, PERCENT_RANK, RANK, RECURSIVE, ROW_NUMBER, WINDOW
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME
FROM information_schema.COLUMNS
WHERE COLUMN_NAME IN (
  'RANK', 'GROUPS', 'EMPTY', 'OVER', 'WINDOW', 'RECURSIVE',
  'ROW_NUMBER', 'LATERAL', 'LEAD', 'LAG', 'OF', 'EXCEPT'
)
AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 6. Check authentication plugins in use
SELECT plugin, COUNT(*) AS user_count
FROM mysql.user
GROUP BY plugin;

-- 7. Check for deprecated features
SELECT TABLE_SCHEMA, TABLE_NAME
FROM information_schema.TABLES
WHERE CREATE_OPTIONS LIKE '%partitioned%';

-- 8. Check for utf8mb3 usage (implicit utf8 = utf8mb3 in 5.7, utf8mb4 in 8.0)
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME
FROM information_schema.COLUMNS
WHERE CHARACTER_SET_NAME = 'utf8'
  AND TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys');

-- 9. Verify no pending XA transactions
XA RECOVER;

-- 10. Check table integrity
mysqlcheck --all-databases --check-upgrade -u root -p
```

## In-Place Upgrade Procedure

### 5.7 to 8.0 In-Place Upgrade

```bash
# Step 1: Full backup (critical - no rollback without this)
mysqldump --all-databases --routines --triggers --events \
  --single-transaction --master-data=2 \
  -u root -p > /backup/pre_upgrade_5.7_full.sql

# Also take a physical backup
xtrabackup --backup --target-dir=/backup/pre_upgrade_physical \
  --user=root --password='P@ss!'

# Step 2: Run upgrade checker
mysqlsh -- util check-for-server-upgrade root@localhost:3306 \
  --target-version=8.0.40

# Step 3: Fix all issues reported by the checker

# Step 4: Slow shutdown (flush all dirty pages)
mysql -u root -p -e "SET GLOBAL innodb_fast_shutdown = 0;"
systemctl stop mysqld

# Step 5: Back up the data directory
cp -a /var/lib/mysql /var/lib/mysql_5.7_backup

# Step 6: Upgrade MySQL binaries
# On RHEL/CentOS:
yum remove mysql-community-server mysql-community-client
yum install mysql-community-server-8.0.40 mysql-community-client-8.0.40

# On Debian/Ubuntu:
apt remove mysql-server mysql-client
apt install mysql-server=8.0.40-1 mysql-client=8.0.40-1

# Step 7: Review my.cnf for deprecated options
# Remove: query_cache_size, query_cache_type (removed in 8.0)
# Remove: innodb_file_format (removed in 8.0)
# Remove: deprecated sql_mode values
# Update: default_authentication_plugin = caching_sha2_password (or keep mysql_native_password temporarily)

# Step 8: Start MySQL 8.0
systemctl start mysqld

# Step 9: Run mysql_upgrade (8.0 < 8.0.16)
mysql_upgrade -u root -p
# In 8.0.16+, the server auto-upgrades system tables at startup
# Check error log for "System table upgrade required" messages

# Step 10: Restart after upgrade
systemctl restart mysqld

# Step 11: Verify
mysql -u root -p -e "SELECT VERSION();"
mysqlcheck --all-databases --check-upgrade -u root -p
```

### 8.0 to 8.4 In-Place Upgrade

```bash
# Step 1: Backup
mysqldump --all-databases --routines --triggers --events \
  --single-transaction --source-data=2 \
  -u root -p > /backup/pre_upgrade_8.0_full.sql

# Step 2: Run upgrade checker targeting 8.4
mysqlsh -- util check-for-server-upgrade root@localhost:3306 \
  --target-version=8.4.0

# Step 3: Address findings
# Key 8.4 changes:
# - mysql_native_password removed by default
# - expire_logs_days removed (use binlog_expire_logs_seconds)
# - SHOW SLAVE STATUS removed (use SHOW REPLICA STATUS)
# - CHANGE MASTER TO removed (use CHANGE REPLICATION SOURCE TO)

# Step 4: Migrate users from mysql_native_password BEFORE upgrade
mysql -u root -p -e "
  SELECT user, host, plugin FROM mysql.user
  WHERE plugin = 'mysql_native_password';"
# For each user:
mysql -u root -p -e "
  ALTER USER 'appuser'@'%'
  IDENTIFIED WITH caching_sha2_password BY 'NewP@ss!';"

# Step 5: Slow shutdown
mysql -u root -p -e "SET GLOBAL innodb_fast_shutdown = 0;"
systemctl stop mysqld

# Step 6: Back up datadir
cp -a /var/lib/mysql /var/lib/mysql_8.0_backup

# Step 7: Upgrade binaries to 8.4
yum install mysql-community-server-8.4.0 mysql-community-client-8.4.0

# Step 8: Update my.cnf
# Remove: expire_logs_days (use binlog_expire_logs_seconds)
# Remove: innodb_log_file_size, innodb_log_files_in_group (use innodb_redo_log_capacity)
# Remove: slave_* variables (use replica_* equivalents)
# Remove: master_info_repository, relay_log_info_repository (TABLE is the only option)

# Step 9: Start MySQL 8.4
systemctl start mysqld

# Step 10: Verify
mysql -u root -p -e "SELECT VERSION();"
```

## Logical Dump/Reload Upgrade

```bash
# Safest approach, especially for major version jumps
# Downside: more downtime (proportional to database size)

# Step 1: Dump from old version
mysqldump --all-databases --routines --triggers --events \
  --single-transaction --set-gtid-purged=OFF \
  -u root -p > /backup/logical_upgrade_dump.sql

# Step 2: Install new MySQL version on fresh datadir
# Stop old MySQL
systemctl stop mysqld

# Move old datadir
mv /var/lib/mysql /var/lib/mysql_old

# Install new version
yum install mysql-community-server-8.4.0

# Initialize fresh datadir
mysqld --initialize-insecure --user=mysql
# or with random password:
# mysqld --initialize --user=mysql  (check error log for temp password)

# Start new version
systemctl start mysqld

# Step 3: Reload the dump
mysql -u root -p < /backup/logical_upgrade_dump.sql

# Step 4: Verify
mysql -u root -p -e "SELECT VERSION();"
mysqlcheck --all-databases --check --extended -u root -p
```

### Using MySQL Shell for Faster Logical Migration

```bash
# Dump from source (parallel, compressed, chunked)
mysqlsh root@old_host:3306 -- util dump-instance /backup/shell_dump \
  --threads=8 --compression=zstd --ocimds=false

# Load on target (parallel, much faster than mysql < dump.sql)
mysqlsh root@new_host:3306 -- util load-dump /backup/shell_dump \
  --threads=8 --resetProgress --ignoreVersion
```

## Replication-Based Rolling Upgrade

For minimal downtime, upgrade replicas first, then failover.

```bash
# Step 1: Upgrade replicas one at a time
# On replica:
systemctl stop mysqld
# Upgrade binaries to new version
# Update my.cnf
systemctl start mysqld
# Verify replication resumes and catches up
mysql -e "SHOW REPLICA STATUS\G" | grep -E "Running|Behind"

# Step 2: After all replicas are upgraded, promote one
# Application failover to upgraded replica (now new primary)

# Step 3: Upgrade the old primary (now a replica)
# Stop old primary, upgrade binaries, start, re-attach as replica

# Step 4: Verify entire topology
```

> **Important**: MySQL supports replication from older source to newer replica (within one major
> version). Replication from newer source to older replica is NOT supported.

## Post-Upgrade Validation

```sql
-- 1. Verify version
SELECT VERSION();

-- 2. Check all databases are accessible
SHOW DATABASES;

-- 3. Check for upgrade errors in error log
-- grep -i "error\|warning\|upgrade" /var/log/mysqld.log | tail -50

-- 4. Run table checks
mysqlcheck --all-databases --check-upgrade --auto-repair -u root -p

-- 5. Verify user accounts and authentication
SELECT user, host, plugin, account_locked, password_expired
FROM mysql.user
ORDER BY user;

-- 6. Check replication status (if applicable)
SHOW REPLICA STATUS\G

-- 7. Verify stored routines
SELECT ROUTINE_SCHEMA, ROUTINE_NAME, ROUTINE_TYPE
FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA NOT IN ('mysql', 'sys');

-- 8. Verify events and triggers
SELECT EVENT_SCHEMA, EVENT_NAME, STATUS FROM information_schema.EVENTS;
SELECT TRIGGER_SCHEMA, TRIGGER_NAME FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA NOT IN ('mysql', 'sys', 'information_schema');

-- 9. Verify InnoDB status
SHOW ENGINE INNODB STATUS\G

-- 10. Run application smoke tests
-- Execute critical application queries and verify results

-- 11. Check performance baseline
-- Compare query response times and throughput against pre-upgrade baseline
```

## Common Upgrade Issues and Solutions

### Authentication Plugin Changes (5.7 -> 8.0)

```sql
-- Problem: Applications using mysql_native_password fail with caching_sha2_password
-- Solution A: Migrate users to new plugin (preferred)
ALTER USER 'appuser'@'%'
  IDENTIFIED WITH caching_sha2_password BY 'NewP@ss!';

-- Solution B: Keep mysql_native_password temporarily
-- In my.cnf:
-- default_authentication_plugin = mysql_native_password

-- Solution C: Upgrade client/connector libraries to support caching_sha2_password
-- MySQL Connector/J 8.0+, PyMySQL 0.10+, mysqlclient 2.0+, etc.
```

### sql_mode Changes

```sql
-- Check current sql_mode
SELECT @@sql_mode;

-- 8.0 default includes: ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES,
-- NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_ENGINE_SUBSTITUTION

-- Common issue: queries that worked in 5.7 fail with ONLY_FULL_GROUP_BY
-- Fix: update queries to include all non-aggregated columns in GROUP BY
-- Temporary workaround (not recommended long-term):
SET GLOBAL sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
```

### Reserved Word Conflicts

```sql
-- Problem: Column or table named RANK, GROUPS, OVER, etc. fails in 8.0
-- Solution: Quote identifiers with backticks
SELECT `rank`, `groups` FROM my_table WHERE `over` > 100;

-- Better solution: rename the columns
ALTER TABLE my_table CHANGE `rank` ranking INT;
ALTER TABLE my_table CHANGE `groups` group_name VARCHAR(100);
```

### Query Cache Removal (8.0)

```sql
-- Problem: my.cnf has query_cache_size or query_cache_type; MySQL 8.0 fails to start
-- Solution: Remove all query_cache_* variables from my.cnf
-- grep -n "query_cache" /etc/my.cnf  (find and remove these lines)
```

### Integer Display Width Deprecation (8.0)

```sql
-- Problem: INT(11) display width generates deprecation warnings in 8.0
-- Solution: Remove display width specifications
ALTER TABLE my_table MODIFY col1 INT;  -- instead of INT(11)
-- Note: TINYINT(1) is still recognized as boolean by some connectors
```

## Rollback Plan

```bash
# Option 1: Restore from backup (cleanest)
systemctl stop mysqld
rm -rf /var/lib/mysql
# Reinstall old MySQL version
# Restore from pre-upgrade backup
mysql -u root -p < /backup/pre_upgrade_full.sql

# Option 2: Restore datadir backup (fastest for in-place upgrade)
systemctl stop mysqld
rm -rf /var/lib/mysql
mv /var/lib/mysql_old_backup /var/lib/mysql
# Reinstall old MySQL version binaries
systemctl start mysqld

# Option 3: Promote a not-yet-upgraded replica
# Only works if you upgraded using rolling strategy and have an old-version replica
```

## Best Practices

- Always run `util.checkForServerUpgrade()` before any major version upgrade.
- Take and verify a full backup before starting the upgrade; test the restore on a separate host.
- Read the full release notes for the target version; pay special attention to "Incompatible Changes."
- Upgrade a non-production instance first (staging/dev) and run application test suites against it.
- Use the rolling upgrade strategy for replication topologies to minimize downtime.
- Set `innodb_fast_shutdown = 0` before stopping the old version to ensure a clean shutdown.
- Plan for rollback; keep the old datadir backup and old version binaries available.
- Migrate authentication plugins proactively; do not wait until the upgrade forces it.
- Update application connectors/drivers to versions compatible with the target MySQL release.
- Keep my.cnf clean; remove all deprecated and removed variables before starting the new version.
- Monitor query performance closely for the first 24-48 hours after upgrade; optimizer changes can alter execution plans.
- Document the upgrade procedure and timeline; share with the operations team.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Skipping major versions (5.7 -> 8.4 directly) | Unsupported path; may cause data corruption or startup failure | Follow the supported path: 5.7 -> 8.0 -> 8.4 |
| Not running the upgrade checker | Discover breaking changes only after the upgrade; extended downtime | Always run `util.checkForServerUpgrade()` beforehand |
| Not backing up before upgrade | No rollback path if upgrade fails or causes data issues | Take full backup (logical + physical) and verify it |
| Leaving deprecated my.cnf options | MySQL refuses to start or logs excessive warnings | Remove `query_cache_*`, `innodb_file_format`, deprecated `sql_mode` values, etc. |
| Ignoring authentication plugin changes | All application connections fail after 8.0 upgrade | Migrate to `caching_sha2_password` or set `default_authentication_plugin` before upgrade |
| Upgrading source before replicas | Newer source -> older replica replication is not supported; replication breaks | Always upgrade replicas first, then promote one, then upgrade old source |
| Not testing with real application workload | SQL syntax, optimizer behavior, or connector changes break production | Run full application test suite against upgraded staging instance |
| Using `innodb_fast_shutdown = 2` before upgrade | Crash recovery required by new version with different redo log format; startup fails | Use `innodb_fast_shutdown = 0` for a clean shutdown |

## MySQL Version Notes

### MySQL 5.7 -> 8.0 Key Changes
- Default auth: `mysql_native_password` -> `caching_sha2_password`.
- Query cache removed entirely.
- Data dictionary: `.frm` files -> transactional InnoDB data dictionary.
- Default `sql_mode` includes `ONLY_FULL_GROUP_BY`.
- `utf8` charset defaults to `utf8mb4` in 8.0.
- Integer display width deprecated (`INT(11)` -> `INT`).
- 30+ new reserved words (RANK, GROUPS, OVER, WINDOW, etc.).
- `mysql_upgrade` deprecated in 8.0.16+ (auto-upgrade at startup).
- Dynamic privileges replace `SUPER` (SYSTEM_VARIABLES_ADMIN, etc.).
- JSON enhancements, window functions, CTEs, lateral derived tables.

### MySQL 8.0 -> 8.4 Key Changes
- `mysql_native_password` plugin removed by default.
- `expire_logs_days` removed; use `binlog_expire_logs_seconds`.
- `SHOW SLAVE STATUS` / `CHANGE MASTER TO` removed.
- `innodb_log_file_size` / `innodb_log_files_in_group` removed; use `innodb_redo_log_capacity`.
- `master_info_repository` / `relay_log_info_repository` removed (TABLE is the only option).
- `mysqlpump` removed; use `mysqldump` or MySQL Shell utilities.
- `binlog_format` deprecated (ROW is the only format going forward).
- TLSv1 and TLSv1.1 removed.

### MySQL 8.4 -> 9.x Key Changes
- Innovation releases with shorter support cycles.
- Further deprecated feature removal.
- New features available earlier than next LTS.
- Upgrade from 8.4 to 9.x is supported.
- Organizations prioritizing stability should stay on 8.4 LTS.

## Sources

- [MySQL 8.0 Reference: Upgrading MySQL](https://dev.mysql.com/doc/refman/8.0/en/upgrading.html)
- [MySQL 8.0 Reference: Changes in MySQL 8.0](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html)
- [MySQL 8.4 Reference: Upgrading to MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/upgrading.html)
- [MySQL Shell: Upgrade Checker Utility](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-upgrade.html)
- [MySQL 8.0 Reference: Removed Features](https://dev.mysql.com/doc/refman/8.0/en/mysql-nutshell.html#mysql-nutshell-removals)
- [MySQL 8.4 Release Notes](https://dev.mysql.com/doc/relnotes/mysql/8.4/en/)
