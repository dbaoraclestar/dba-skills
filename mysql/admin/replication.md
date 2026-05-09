# MySQL Replication — Async, Semi-Sync, GTID, Group Replication, and InnoDB Cluster

## Overview

MySQL replication copies data from a source (master) server to one or more replica (slave)
servers by shipping binary log events. It is the foundation for read scaling, high availability,
disaster recovery, and offline analytics. MySQL supports multiple replication topologies: simple
source-replica, cascading, multi-source, and circular.

The replication model has evolved significantly across versions. Traditional position-based
replication uses binary log file names and byte offsets. GTID-based replication (5.6+) assigns
a globally unique identifier to every transaction, simplifying failover and re-pointing replicas.
Group Replication (5.7.17+) provides built-in consensus-based multi-primary or single-primary
clustering, and InnoDB Cluster wraps Group Replication with MySQL Shell and MySQL Router for
a complete HA solution.

Regardless of topology, replication lag monitoring, conflict resolution, and failure recovery
are essential operational skills. This document covers setup, monitoring, and troubleshooting
for all major replication modes.

## Key Concepts

- **Binary Log (binlog)**: Transaction log on the source; the stream of change events shipped to replicas.
- **Relay Log**: Local copy of binlog events on the replica, consumed by the SQL applier thread.
- **IO Thread**: Replica thread that connects to the source and fetches binlog events.
- **SQL Thread (Applier)**: Replica thread that reads relay logs and applies events.
- **GTID (Global Transaction Identifier)**: `server_uuid:transaction_id` pair that uniquely identifies every committed transaction.
- **Semi-Synchronous Replication**: Source waits for at least one replica to acknowledge receipt of events before committing returns to the client.
- **Group Replication**: Built-in Paxos-based consensus protocol for automatic failover and conflict detection.

## Traditional Asynchronous Replication Setup

### Source Configuration

```ini
# my.cnf on source
[mysqld]
server-id              = 1
log-bin                = /var/lib/mysql/binlog
binlog-format          = ROW
binlog-row-image       = FULL
sync-binlog            = 1
innodb-flush-log-at-trx-commit = 1
```

```sql
-- Create replication user on source
CREATE USER 'repl_user'@'10.0.0.%'
  IDENTIFIED WITH caching_sha2_password BY 'ReplP@ss2024!'
  REQUIRE SSL;

GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'10.0.0.%';
FLUSH PRIVILEGES;

-- Check binary log position
SHOW MASTER STATUS\G
-- or in 8.0.26+:
SHOW BINARY LOG STATUS\G
```

### Replica Configuration

```ini
# my.cnf on replica
[mysqld]
server-id              = 2
relay-log              = /var/lib/mysql/relay-log
read-only              = ON
super-read-only        = ON
log-bin                = /var/lib/mysql/binlog
log-replica-updates    = ON
```

```sql
-- Configure and start replication (traditional position-based)
-- MySQL 8.0.23+ syntax:
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.0.1',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplP@ss2024!',
  SOURCE_LOG_FILE = 'binlog.000042',
  SOURCE_LOG_POS = 154,
  SOURCE_SSL = 1,
  GET_SOURCE_PUBLIC_KEY = 1;

START REPLICA;
SHOW REPLICA STATUS\G

-- Legacy syntax (5.7 / pre-8.0.23):
CHANGE MASTER TO
  MASTER_HOST = '10.0.0.1',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl_user',
  MASTER_PASSWORD = 'ReplP@ss2024!',
  MASTER_LOG_FILE = 'binlog.000042',
  MASTER_LOG_POS = 154;

START SLAVE;
SHOW SLAVE STATUS\G
```

## GTID-Based Replication

### Enabling GTIDs

```ini
# my.cnf (both source and replica)
[mysqld]
gtid-mode              = ON
enforce-gtid-consistency = ON
log-bin                = /var/lib/mysql/binlog
log-replica-updates    = ON
binlog-format          = ROW
```

### Setting Up GTID Replication

```sql
-- On replica: configure using GTID auto-positioning
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.0.1',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplP@ss2024!',
  SOURCE_AUTO_POSITION = 1,
  SOURCE_SSL = 1,
  GET_SOURCE_PUBLIC_KEY = 1;

START REPLICA;

-- Verify GTID status
SELECT @@gtid_mode, @@enforce_gtid_consistency;
SHOW REPLICA STATUS\G
-- Key fields: Retrieved_Gtid_Set, Executed_Gtid_Set
```

### GTID Advantage: Failover

```sql
-- When promoting a replica to source, other replicas just point to the new source
-- No need to find binlog file/position; GTIDs handle it automatically
STOP REPLICA;
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.0.2',  -- new source (former replica)
  SOURCE_PORT = 3306,
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplP@ss2024!',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;
```

## Semi-Synchronous Replication

### Setup

```sql
-- Install plugins (source and replica)
-- Source:
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
-- or for 5.7 / pre-8.0.26:
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

-- Replica:
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';

-- Enable on source
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_timeout = 5000;  -- 5 seconds; falls back to async

-- Enable on replica
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
STOP REPLICA IO_THREAD;
START REPLICA IO_THREAD;

-- Verify
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

### Wait Point Options

```sql
-- AFTER_SYNC (default in 8.0): source waits after writing to binlog, before commit
-- Ensures replica has the event before any client sees the commit
SET GLOBAL rpl_semi_sync_source_wait_point = 'AFTER_SYNC';

-- AFTER_COMMIT: source waits after commit (phantom reads possible on failover)
SET GLOBAL rpl_semi_sync_source_wait_point = 'AFTER_COMMIT';

-- Require at least N replicas to acknowledge
SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;
```

## Group Replication / InnoDB Cluster

### Group Replication Prerequisites

```ini
# my.cnf for Group Replication
[mysqld]
server-id                       = 1
gtid-mode                       = ON
enforce-gtid-consistency        = ON
log-bin                         = /var/lib/mysql/binlog
log-replica-updates             = ON
binlog-format                   = ROW
binlog-checksum                 = NONE
relay-log-info-repository       = TABLE
transaction-write-set-extraction = XXHASH64

# Group Replication settings
plugin_load_add                 = 'group_replication.so'
group_replication_group_name    = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot = OFF
group_replication_local_address = "10.0.0.1:33061"
group_replication_group_seeds   = "10.0.0.1:33061,10.0.0.2:33061,10.0.0.3:33061"
group_replication_single_primary_mode = ON
```

### Bootstrap and Join

```sql
-- On first node: bootstrap the group
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;

-- On additional nodes: join the group
START GROUP_REPLICATION;

-- Check group membership
SELECT * FROM performance_schema.replication_group_members;

-- Check who is primary
SELECT MEMBER_HOST, MEMBER_PORT, MEMBER_ROLE
FROM performance_schema.replication_group_members;

-- Switch to multi-primary mode
SELECT group_replication_switch_to_multi_primary_mode();
-- Switch back to single-primary
SELECT group_replication_switch_to_single_primary_mode('server_uuid_here');
```

### InnoDB Cluster with MySQL Shell

```javascript
// Create cluster from MySQL Shell
dba.configureInstance('root@10.0.0.1:3306');
dba.configureInstance('root@10.0.0.2:3306');
dba.configureInstance('root@10.0.0.3:3306');

var cluster = dba.createCluster('myCluster');
cluster.addInstance('root@10.0.0.2:3306');
cluster.addInstance('root@10.0.0.3:3306');

// Check status
cluster.status();

// Rejoin a node after failure
cluster.rejoinInstance('root@10.0.0.2:3306');

// Set up MySQL Router
cluster.setupRouterAccount('router_user@%');
// Then on the router host:
// mysqlrouter --bootstrap root@10.0.0.1:3306 --user=mysqlrouter
```

## Multi-Source Replication

```sql
-- Configure two replication channels
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.0.1',
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplP@ss!',
  SOURCE_AUTO_POSITION = 1
  FOR CHANNEL 'source_dc1';

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '10.0.1.1',
  SOURCE_USER = 'repl_user',
  SOURCE_PASSWORD = 'ReplP@ss!',
  SOURCE_AUTO_POSITION = 1
  FOR CHANNEL 'source_dc2';

-- Start all channels
START REPLICA FOR CHANNEL 'source_dc1';
START REPLICA FOR CHANNEL 'source_dc2';

-- Check status per channel
SHOW REPLICA STATUS FOR CHANNEL 'source_dc1'\G
SHOW REPLICA STATUS FOR CHANNEL 'source_dc2'\G

-- Stop a specific channel
STOP REPLICA FOR CHANNEL 'source_dc2';
```

## Replication Filters

```sql
-- Database-level filters (set in my.cnf or dynamically in 8.0)
-- Source side (less common):
-- binlog-do-db = myapp_db
-- binlog-ignore-db = test_db

-- Replica side (preferred):
CHANGE REPLICATION FILTER
  REPLICATE_DO_DB = (myapp_db, reporting_db),
  REPLICATE_IGNORE_DB = (test_db);

-- Table-level filters
CHANGE REPLICATION FILTER
  REPLICATE_DO_TABLE = (myapp_db.orders, myapp_db.customers),
  REPLICATE_IGNORE_TABLE = (myapp_db.temp_staging);

-- Wildcard table filters
CHANGE REPLICATION FILTER
  REPLICATE_WILD_DO_TABLE = ('myapp_db.%'),
  REPLICATE_WILD_IGNORE_TABLE = ('myapp_db.tmp_%');

-- Per-channel filters (8.0+)
CHANGE REPLICATION FILTER
  REPLICATE_DO_DB = (myapp_db)
  FOR CHANNEL 'source_dc1';
```

## Monitoring Replication

### Key Status Checks

```sql
-- Comprehensive replica status
SHOW REPLICA STATUS\G

-- Key fields to monitor:
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0
-- Last_IO_Error: (should be empty)
-- Last_SQL_Error: (should be empty)
-- Retrieved_Gtid_Set vs Executed_Gtid_Set

-- Quick lag check
SELECT
  CHANNEL_NAME,
  SERVICE_STATE AS io_state,
  LAST_ERROR_NUMBER AS io_err
FROM performance_schema.replication_connection_status;

SELECT
  CHANNEL_NAME,
  SERVICE_STATE AS sql_state,
  LAST_ERROR_NUMBER AS sql_err,
  LAST_APPLIED_TRANSACTION_END_APPLY_TIMESTAMP,
  APPLYING_TRANSACTION
FROM performance_schema.replication_applier_status_by_worker;
```

### Heartbeat Table for Accurate Lag Measurement

```sql
-- On source: create heartbeat table and update it every second
CREATE DATABASE IF NOT EXISTS heartbeat;
CREATE TABLE heartbeat.repl_heartbeat (
  id INT PRIMARY KEY,
  ts TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6)
    ON UPDATE CURRENT_TIMESTAMP(6)
);
INSERT INTO heartbeat.repl_heartbeat VALUES (1, NOW(6));

-- Cron or event to update every second on source:
CREATE EVENT heartbeat.update_heartbeat
  ON SCHEDULE EVERY 1 SECOND
  DO UPDATE heartbeat.repl_heartbeat SET ts = NOW(6) WHERE id = 1;

-- On replica: measure actual lag
SELECT
  TIMESTAMPDIFF(SECOND, ts, NOW(6)) AS replication_lag_seconds
FROM heartbeat.repl_heartbeat
WHERE id = 1;
```

### Parallel Replication (Multi-Threaded Applier)

```sql
-- Enable parallel applier (8.0+)
STOP REPLICA;
SET GLOBAL replica_parallel_workers = 8;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL replica_preserve_commit_order = ON;
START REPLICA;

-- Monitor parallel worker status
SELECT WORKER_ID, SERVICE_STATE, LAST_ERROR_NUMBER, LAST_ERROR_MESSAGE
FROM performance_schema.replication_applier_status_by_worker;

-- 5.7 syntax:
SET GLOBAL slave_parallel_workers = 8;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
```

## Troubleshooting Common Issues

### Duplicate Key Error (Error 1062)

```sql
-- Option 1: Skip the error (lose the conflicting row from source)
STOP REPLICA;
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA;

-- Option 2: Skip by GTID (preferred with GTID replication)
STOP REPLICA;
SET GTID_NEXT = 'source_uuid:transaction_number';
BEGIN; COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;

-- Option 3: Fix the data on replica to match source, then restart
STOP REPLICA;
-- DELETE or UPDATE the conflicting row on replica
START REPLICA;
```

### Relay Log Corruption

```bash
# Symptoms: relay log read failure, applier stops
# Fix: reset relay logs and re-fetch from source

# On replica:
mysql -e "STOP REPLICA; RESET REPLICA; START REPLICA;"

# If GTID is enabled, auto-positioning handles re-fetch automatically
# If position-based, you need to specify the correct binlog position
```

### Replication Lag

```sql
-- Check for long-running queries on replica
SELECT * FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep' AND TIME > 30
ORDER BY TIME DESC;

-- Check if parallel applier is utilized
SHOW STATUS LIKE 'Replica_parallel%';

-- Increase parallel workers
STOP REPLICA;
SET GLOBAL replica_parallel_workers = 16;
START REPLICA;

-- Check for large transactions on source
-- (single large transaction serializes applier)
SELECT * FROM performance_schema.events_statements_history_long
WHERE ROWS_AFFECTED > 100000
ORDER BY TIMER_END DESC LIMIT 10;
```

### Errant Transactions

```sql
-- Find errant transactions (GTIDs on replica not on source)
-- On source:
SELECT @@global.gtid_executed;

-- On replica:
SELECT @@global.gtid_executed;

-- Compare using GTID_SUBTRACT
SELECT GTID_SUBTRACT(
  (SELECT @@global.gtid_executed),  -- replica's GTIDs
  'source_uuid:1-N'                 -- source's GTIDs
) AS errant_transactions;

-- Empty result = no errant transactions (good)
-- Non-empty = errant transactions exist (must be resolved before failover)
```

## Best Practices

- Use GTID-based replication for all new deployments; it vastly simplifies failover and re-pointing.
- Set `binlog_format = ROW` for deterministic replication; avoid STATEMENT format in production.
- Enable `super_read_only` on replicas to prevent accidental writes.
- Use `sync_binlog = 1` and `innodb_flush_log_at_trx_commit = 1` on the source for crash-safe replication.
- Deploy heartbeat tables (e.g., via `pt-heartbeat`) for accurate lag measurement instead of relying solely on `Seconds_Behind_Source`.
- Enable multi-threaded applier (`replica_parallel_workers`) with `LOGICAL_CLOCK` type for better throughput.
- Monitor `Replica_IO_Running`, `Replica_SQL_Running`, and error fields in automated alerts.
- Test failover procedures regularly in non-production; document the exact switchover steps.
- Avoid large transactions (> 1 GB) on the source; they serialize the applier and cause lag spikes.
- Use replication filters only when necessary; prefer separate instances over complex filtering.
- Enable SSL for replication channels, especially across data centers.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Setting `binlog_format = STATEMENT` in production | Non-deterministic functions (UUID(), NOW()) cause data drift | Use `binlog_format = ROW`; it is the default since 8.0 |
| Not enabling `log_replica_updates` on replicas | Cascading replicas or failover promotion loses events | Always set `log_replica_updates = ON` |
| Relying solely on `Seconds_Behind_Source` for lag | Value is inaccurate when IO thread is behind or SQL thread is idle between events | Use heartbeat tables with sub-second precision |
| Writing directly to a replica | Creates errant transactions; breaks GTID consistency; causes replication divergence | Enable `super_read_only = ON` on all replicas |
| Skipping errors without investigation | Data divergence accumulates silently; production data becomes inconsistent | Investigate every replication error; use `pt-table-checksum` to verify consistency |
| Using `server-id = 0` or duplicate server IDs | Replication silently fails or causes circular loops | Assign unique `server-id` to every instance in the topology |
| Not monitoring replication at all | Replication breaks unnoticed; replica falls hours behind | Set up automated monitoring with alerts on lag > threshold and thread state changes |
| Filtering replication on the source side (`binlog-do-db`) | Cross-database statements may be missed due to `USE` context dependency | Use replica-side filters (`REPLICATE_DO_DB`) or no filters at all |

## MySQL Version Notes

### MySQL 5.7
- GTID replication available and stable; requires `enforce_gtid_consistency = ON`.
- Semi-sync via `rpl_semi_sync_master`/`rpl_semi_sync_slave` plugin names.
- Group Replication plugin (5.7.17+); generally prefer 8.0 for production GR.
- Multi-threaded slave applier with `slave_parallel_workers` and `slave_parallel_type`.
- `CHANGE MASTER TO` / `SHOW SLAVE STATUS` terminology.
- `expire_logs_days` for binlog expiration.

### MySQL 8.0
- Terminology changes (8.0.23+): `REPLICA` replaces `SLAVE`, `SOURCE` replaces `MASTER`.
- `caching_sha2_password` default requires `GET_SOURCE_PUBLIC_KEY = 1` for replication users.
- Group Replication significantly improved; InnoDB Cluster and ClusterSet for multi-DC.
- `replica_parallel_type = LOGICAL_CLOCK` is the default in 8.0.27+.
- `binlog_transaction_dependency_tracking` controls parallelism detection.
- `CHANGE REPLICATION FILTER ... FOR CHANNEL` for per-channel filters.
- Clone plugin enables fast replica provisioning.
- Performance Schema tables for replication monitoring (preferred over `SHOW REPLICA STATUS`).

### MySQL 8.4 / 9.x
- `SHOW SLAVE STATUS` and `CHANGE MASTER TO` fully removed; use new syntax.
- `replica_parallel_workers` defaults to 4 (was 0/1 in earlier versions).
- `binlog_transaction_dependency_tracking` deprecated; writeset-based parallelism is automatic.
- InnoDB ClusterSet GA for disaster recovery across regions.
- Improved automatic member recovery in Group Replication.
- `group_replication_paxos_single_leader` for better single-primary performance.

## Sources

- [MySQL 8.0 Reference: Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [MySQL 8.0 Reference: GTID Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids.html)
- [MySQL 8.0 Reference: Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html)
- [MySQL 8.0 Reference: Semi-Sync Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-semisync.html)
- [MySQL 8.0 Reference: InnoDB Cluster](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-innodb-cluster.html)
- [MySQL 8.4 Reference: Replication](https://dev.mysql.com/doc/refman/8.4/en/replication.html)
- [Percona pt-table-checksum](https://docs.percona.com/percona-toolkit/pt-table-checksum.html)
