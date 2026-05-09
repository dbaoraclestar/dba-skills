# Percona Toolkit — Essential DBA Tools for MySQL Operations

## Overview

Percona Toolkit is a collection of command-line tools for MySQL database administration, developed by Percona. These tools address common operational tasks that MySQL does not handle natively: analyzing slow query logs, performing online schema changes on live tables, verifying replication consistency, collecting diagnostic data during incidents, and archiving old data without blocking production queries.

The toolkit is the industry standard for MySQL operations at scale. Tools like `pt-query-digest` and `pt-online-schema-change` are used daily by DBAs managing everything from single-server deployments to large replication topologies. Each tool is a standalone Perl script with extensive options, safety checks, and dry-run modes.

This skill covers the most critical tools: pt-query-digest, pt-online-schema-change, pt-table-checksum, pt-table-sync, pt-stalk, pt-archiver, pt-heartbeat, and pt-show-grants, with practical examples and production-ready configurations.

## Key Concepts

- **Percona Toolkit**: A suite of ~30 Perl scripts for MySQL administration, freely available under GPLv2.
- **DSN (Data Source Name)**: Connection string format used by toolkit tools: `h=host,P=port,u=user,p=pass,D=db,t=table`.
- **Chunk**: A range of rows processed as a unit during table operations (pt-osc, pt-archiver, pt-table-checksum).
- **Throttling**: Automatic pause/resume based on replication lag or server load thresholds.
- **Sentinel file**: A file whose presence signals a tool to stop (e.g., `/tmp/pt-osc-sentinel`).

## Installation

```bash
# Debian/Ubuntu
sudo apt-get install percona-toolkit

# RHEL/CentOS
sudo yum install percona-toolkit

# From tarball
wget https://downloads.percona.com/downloads/percona-toolkit/LATEST/binary/tarball/percona-toolkit-3.5.7_x86_64.tar.gz
tar xzf percona-toolkit-*.tar.gz
cd percona-toolkit-*/bin/
sudo cp pt-* /usr/local/bin/

# Verify installation
pt-query-digest --version
```

## pt-query-digest — Slow Query Log Analysis

### Analyzing the slow query log

```bash
# Basic analysis
pt-query-digest /var/log/mysql/slow.log

# Filter by time range
pt-query-digest /var/log/mysql/slow.log \
  --since '2024-11-01 00:00:00' \
  --until '2024-11-02 00:00:00'

# Top 20 queries by total time
pt-query-digest /var/log/mysql/slow.log \
  --limit=20 \
  --order-by=Query_time:sum

# Filter specific database
pt-query-digest /var/log/mysql/slow.log \
  --filter '$event->{db} eq "mydb"'

# Filter queries taking > 5 seconds
pt-query-digest /var/log/mysql/slow.log \
  --filter '$event->{Query_time} > 5'

# Output to file
pt-query-digest /var/log/mysql/slow.log \
  --output=report \
  --limit=50 \
  > /tmp/slow_query_report.txt
```

### Analyzing from tcpdump

```bash
# Capture MySQL traffic
sudo tcpdump -i eth0 port 3306 -s 65535 -x -nn -q -tttt -c 10000 \
  > /tmp/mysql_traffic.txt

# Analyze captured traffic
pt-query-digest --type=tcpdump /tmp/mysql_traffic.txt
```

### Analyzing from PERFORMANCE_SCHEMA

```bash
pt-query-digest \
  --type=slowlog \
  --processlist \
  h=localhost,u=root,p=secret \
  --run-time=60 \
  --interval=1
```

### Review output

```
# Query 1: 2.45k QPS, 12.34x concurrency, ID 0xA1B2C3D4E5F6
# Scores: V/M = 0.03
# Time range: 2024-11-01T00:00:00 to 2024-11-01T23:59:59
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         45  211.8k
# Exec time     62   4320s    10ms      5s    20ms    50ms    30ms    15ms
# Lock time     12     45s     0       1s    200us   500us   800us   100us
# Rows sent     23   1.2M       1     500      10      20      15       5
# Rows examine  45  50.0M       1   50000     250     500     800     100
#
# SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC LIMIT ?\G
```

## pt-online-schema-change — Online ALTER TABLE

```bash
# Add a column (dry run first)
pt-online-schema-change \
  --alter "ADD COLUMN phone VARCHAR(20) DEFAULT NULL" \
  --dry-run \
  D=mydb,t=users,h=db1,u=root,p=secret

# Execute the change
pt-online-schema-change \
  --alter "ADD COLUMN phone VARCHAR(20) DEFAULT NULL" \
  --host=db1 \
  --user=root \
  --ask-pass \
  --chunk-size=1000 \
  --chunk-time=0.5 \
  --max-lag=1s \
  --check-interval=1 \
  --max-load="Threads_running:25" \
  --critical-load="Threads_running:100" \
  --set-vars="wait_timeout=10000,lock_wait_timeout=1" \
  --recurse=1 \
  --check-replication-filters \
  --progress=time,30 \
  --execute \
  D=mydb,t=users

# Add an index
pt-online-schema-change \
  --alter "ADD INDEX idx_email (email)" \
  --max-lag=2s \
  --max-load="Threads_running:30" \
  --execute \
  D=mydb,t=users,h=db1,u=root,p=secret

# Change column type
pt-online-schema-change \
  --alter "MODIFY COLUMN description MEDIUMTEXT" \
  --chunk-size=500 \
  --execute \
  D=mydb,t=products,h=db1,u=root,p=secret

# Handle foreign keys
pt-online-schema-change \
  --alter "ADD COLUMN region_id INT" \
  --alter-foreign-keys-method=rebuild_constraints \
  --execute \
  D=mydb,t=orders,h=db1,u=root,p=secret
```

## pt-table-checksum — Replication Consistency Verification

```bash
# Check all tables in a database
pt-table-checksum \
  --host=primary \
  --user=ptcheck \
  --password=secret \
  --databases=mydb \
  --check-replication-filters \
  --replicate=percona.checksums \
  --no-check-binlog-format \
  --chunk-size=1000 \
  --max-lag=2s

# Check specific tables
pt-table-checksum \
  --host=primary \
  --user=ptcheck \
  --password=secret \
  --databases=mydb \
  --tables=users,orders,products \
  --replicate=percona.checksums

# Only report differences (after checksum has run)
pt-table-checksum \
  --host=primary \
  --user=ptcheck \
  --password=secret \
  --replicate=percona.checksums \
  --replicate-check-only

# Output example:
#             TS ERRORS  DIFFS     ROWS  DIFF_ROWS  CHUNKS SKIPPED    TIME TABLE
# 11-01T10:00:00      0      0    50000          0      50       0   2.500 mydb.users
# 11-01T10:00:03      0      1   200000       1234     200       0   8.200 mydb.orders
```

## pt-table-sync — Fix Replication Drift

```bash
# Preview what would be synced (dry run)
pt-table-sync \
  --print \
  --replicate=percona.checksums \
  h=primary,u=root,p=secret

# Sync differences found by pt-table-checksum
pt-table-sync \
  --execute \
  --replicate=percona.checksums \
  --sync-to-master \
  h=replica1,u=root,p=secret

# Sync a specific table
pt-table-sync \
  --execute \
  --no-check-triggers \
  h=primary,D=mydb,t=orders,u=root,p=secret \
  h=replica1
```

## pt-stalk — Collect Diagnostics on Trigger Conditions

```bash
# Trigger when Threads_running > 30
pt-stalk \
  --function=status \
  --variable=Threads_running \
  --threshold=30 \
  --cycles=5 \
  --interval=1 \
  --iterations=3 \
  --run-time=60 \
  --dest=/tmp/pt-stalk-data \
  --log=/tmp/pt-stalk.log \
  -- --user=root --password=secret --host=db1

# Trigger on custom query
pt-stalk \
  --function=processlist \
  --variable=State \
  --match="Waiting for table metadata lock" \
  --threshold=5 \
  --dest=/tmp/pt-stalk-mdl \
  -- --user=root --password=secret --host=db1

# Collected data includes:
# - SHOW PROCESSLIST
# - SHOW ENGINE INNODB STATUS
# - SHOW GLOBAL STATUS
# - SHOW GLOBAL VARIABLES
# - vmstat, iostat, mpstat snapshots
# - /proc/diskstats, /proc/meminfo
```

## pt-archiver — Archive and Purge Rows

```bash
# Archive old orders to archive table
pt-archiver \
  --source h=db1,D=mydb,t=orders,u=root,p=secret \
  --dest h=archive-db,D=archive,t=orders_archive \
  --where "created_at < '2023-01-01'" \
  --limit=1000 \
  --commit-each \
  --progress=10000 \
  --statistics

# Purge (delete without archiving)
pt-archiver \
  --source h=db1,D=mydb,t=session_log,u=root,p=secret \
  --purge \
  --where "created_at < NOW() - INTERVAL 90 DAY" \
  --limit=500 \
  --max-lag=1s \
  --check-interval=1 \
  --progress=5000

# Archive with throttling
pt-archiver \
  --source h=db1,D=mydb,t=events,u=root,p=secret \
  --dest h=archive-db,D=archive,t=events_archive \
  --where "event_date < '2024-01-01'" \
  --limit=1000 \
  --max-lag=2s \
  --bulk-delete \
  --bulk-insert \
  --commit-each \
  --sleep=0.1
```

## pt-heartbeat — Measure Replication Lag

```bash
# Start heartbeat writer on primary (run as daemon)
pt-heartbeat \
  --host=primary \
  --user=heartbeat \
  --password=secret \
  --database=percona \
  --create-table \
  --update \
  --daemonize \
  --interval=0.5 \
  --pid=/var/run/pt-heartbeat.pid

# Monitor lag on replica
pt-heartbeat \
  --host=replica1 \
  --user=heartbeat \
  --password=secret \
  --database=percona \
  --monitor \
  --interval=1

# Check lag once
pt-heartbeat \
  --host=replica1 \
  --user=heartbeat \
  --password=secret \
  --database=percona \
  --check

# Output: 0.50s (current lag in seconds)

# Stop the daemon
pt-heartbeat --stop --pid=/var/run/pt-heartbeat.pid
```

## pt-show-grants — Dump User Grants

```bash
# Dump all grants
pt-show-grants --host=db1 --user=root --password=secret

# Dump grants for specific users
pt-show-grants \
  --host=db1 \
  --user=root \
  --password=secret \
  --only=app_user,readonly_user

# Output as revoke statements (for cleanup)
pt-show-grants \
  --host=db1 \
  --user=root \
  --password=secret \
  --revoke

# Diff grants between servers
diff <(pt-show-grants --host=primary --user=root --password=secret --sort) \
     <(pt-show-grants --host=replica1 --user=root --password=secret --sort)
```

## Additional Useful Tools

### pt-kill — Kill queries matching criteria

```bash
# Kill queries running longer than 60 seconds
pt-kill \
  --host=db1 --user=root --password=secret \
  --busy-time=60 \
  --kill \
  --print \
  --interval=5 \
  --daemonize \
  --log=/tmp/pt-kill.log

# Kill queries matching a pattern (dry run)
pt-kill \
  --host=db1 --user=root --password=secret \
  --match-command=Query \
  --match-info "SELECT.*FROM huge_table" \
  --busy-time=30 \
  --print
```

### pt-summary / pt-mysql-summary

```bash
# System summary
pt-summary

# MySQL server summary
pt-mysql-summary --host=db1 --user=root --password=secret
```

## Best Practices

- Always run `--dry-run` or `--print` before `--execute` on any destructive operation.
- Use `--max-lag` and `--max-load` on all tools that modify data to prevent overloading replicas.
- Set `--critical-load` as a circuit breaker: the tool aborts if load exceeds this threshold.
- Run `pt-table-checksum` weekly in production to detect silent replication drift early.
- Use `pt-heartbeat` instead of `SHOW SLAVE STATUS` for accurate replication lag measurement.
- Store `pt-stalk` output in a dedicated directory with timestamps for post-incident analysis.
- Use `--bulk-delete` and `--bulk-insert` with `pt-archiver` for significantly faster archival operations.
- Schedule `pt-show-grants` dumps as part of your backup process to preserve user permissions.
- Monitor `pt-online-schema-change` progress via the `--progress` flag and sentinel file for emergency stops.
- Test all toolkit operations on staging replicas before running on production primaries.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running pt-osc without `--check-replication-filters` | Changes replicate to filtered replicas causing errors | Always include `--check-replication-filters` |
| Using pt-table-checksum with `binlog_format=STATEMENT` on replicas | Checksums are non-deterministic; false positive diffs | Set `--no-check-binlog-format` or use ROW format |
| Running pt-archiver without `--limit` on a huge table | Single massive DELETE locks the table for minutes | Set `--limit=1000` to delete in manageable chunks |
| Not creating the heartbeat table before monitoring | pt-heartbeat --monitor fails with "table doesn't exist" | Run `--create-table --update` first on the primary |
| Forgetting to stop pt-heartbeat daemon before server maintenance | Heartbeat writes continue, potentially causing issues during upgrades | Use `--stop` with the PID file before maintenance windows |

## MySQL Version Notes

- **5.7**: Full toolkit compatibility. `pt-online-schema-change` requires single triggers per table event (5.7.2+ relaxes this). `pt-table-checksum` works with both STATEMENT and ROW binlog format.
- **8.0**: Toolkit 3.3+ required for MySQL 8.0 compatibility. `caching_sha2_password` may require `--no-version-check` or password via DSN. `pt-online-schema-change` compatible with atomic DDL. Some tools need `--no-check-binlog-format` due to default ROW format.
- **8.4/9.x**: Requires Percona Toolkit 3.5+. `mysql_native_password` removal requires updating DSN authentication. Instant DDL support means some operations no longer need pt-osc. Check Percona's release notes for version-specific compatibility.

## Sources

- [Percona Toolkit Documentation](https://docs.percona.com/percona-toolkit/)
- [pt-query-digest](https://docs.percona.com/percona-toolkit/pt-query-digest.html)
- [pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [pt-table-checksum](https://docs.percona.com/percona-toolkit/pt-table-checksum.html)
- [pt-stalk](https://docs.percona.com/percona-toolkit/pt-stalk.html)
- [pt-archiver](https://docs.percona.com/percona-toolkit/pt-archiver.html)
