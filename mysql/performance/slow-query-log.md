# Slow Query Log — Capturing, Analyzing, and Acting on Slow SQL in MySQL

## Overview

The MySQL slow query log is the primary mechanism for identifying queries that exceed a defined execution time threshold. It records the full SQL text, execution time, rows examined, rows sent, and other metadata for every query that takes longer than `long_query_time` seconds. This log is indispensable for performance triage, capacity planning, and ongoing query optimization.

Every production MySQL instance should have the slow query log enabled. The overhead is minimal (a sequential file append per slow query), and the diagnostic value is enormous. Without it, you are flying blind -- relying on application-level metrics or user complaints to discover performance problems. The slow query log lets you proactively find and fix issues before they escalate.

For high-throughput environments or when you need aggregated analysis without filesystem access, the Performance Schema `events_statements_summary_by_digest` table provides a complementary approach, offering cumulative statistics grouped by normalized query pattern (digest). Both approaches have their place: the slow query log gives you individual query instances with full SQL text, while Performance Schema gives you aggregated statistics.

## Key Concepts

**long_query_time**: The threshold in seconds (supports microsecond precision) above which a query is logged. Default is 10 seconds, which is far too high for most production workloads. A value of 1 or even 0.5 is common.

**Query Digest**: A normalized form of a query where literal values are replaced with placeholders. This allows grouping structurally identical queries that differ only in parameter values. Example: `SELECT * FROM orders WHERE id = ?`.

**Rows Examined vs Rows Sent**: Rows examined is the number of rows MySQL read internally. Rows sent is the number of rows returned to the client. A large ratio of examined-to-sent indicates inefficient queries that read many rows but discard most of them.

**Lock Time**: The time the query spent waiting for table locks (not InnoDB row locks). High lock time on MyISAM tables indicates contention; on InnoDB it is usually negligible unless metadata locks are involved.

## Enabling and Configuring the Slow Query Log

### Runtime Configuration (No Restart Required)

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';

-- Set threshold to 1 second
SET GLOBAL long_query_time = 1;

-- Log queries not using indexes (useful for dev/staging, noisy in prod)
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- Throttle the "not using indexes" logging to avoid flooding
SET GLOBAL log_throttle_queries_not_using_indexes = 60;

-- Log slow admin statements (ALTER TABLE, ANALYZE TABLE, etc.)
SET GLOBAL log_slow_admin_statements = 'ON';

-- Set log file location
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow-query.log';

-- Verify settings
SHOW GLOBAL VARIABLES LIKE 'slow_query%';
SHOW GLOBAL VARIABLES LIKE 'long_query_time';
SHOW GLOBAL VARIABLES LIKE 'log_queries_not_using_indexes';
SHOW GLOBAL VARIABLES LIKE 'log_slow_admin_statements';
```

### Persistent Configuration (my.cnf / my.ini)

```ini
[mysqld]
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/slow-query.log
long_query_time         = 1
log_queries_not_using_indexes = 0
log_throttle_queries_not_using_indexes = 60
log_slow_admin_statements = 1

# MySQL 8.0+ additional options
log_slow_extra          = 1
# Logs additional fields: Thread_id, Errno, Killed, Bytes_received,
# Bytes_sent, Read_first, Read_last, Read_key, Read_next, Read_prev,
# Read_rnd, Read_rnd_next, Sort_merge_passes, Sort_range_count,
# Sort_rows, Sort_scan_count, Created_tmp_disk_tables, Created_tmp_tables,
# Start, End
```

### Per-Session Control

```sql
-- Temporarily capture all queries (threshold = 0) for a specific session
SET SESSION long_query_time = 0;

-- Run the suspicious workload...

-- Restore normal threshold
SET SESSION long_query_time = 1;
```

## Reading the Slow Query Log

### Log Entry Format

```
# Time: 2025-06-15T14:23:45.123456Z
# User@Host: app_user[app_user] @ app-server-01 [10.20.30.40]  Id: 42981
# Query_time: 3.245102  Lock_time: 0.000234  Rows_sent: 50  Rows_examined: 1248930
SET timestamp=1718457825;
SELECT o.order_id, o.order_date, c.customer_name, SUM(oi.quantity * oi.unit_price) AS total
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
WHERE o.order_date BETWEEN '2025-01-01' AND '2025-06-15'
  AND c.region = 'US-EAST'
GROUP BY o.order_id, o.order_date, c.customer_name
ORDER BY total DESC
LIMIT 50;
```

### Key Fields to Examine

| Field | What It Tells You |
|---|---|
| Query_time | Total execution time in seconds (with microsecond precision) |
| Lock_time | Time waiting for table-level locks |
| Rows_sent | Rows returned to the client |
| Rows_examined | Rows MySQL had to read internally |
| Rows_examined / Rows_sent ratio | Efficiency indicator; >100:1 warrants investigation |

### Quick Command-Line Analysis

```bash
# Find the top 10 slowest queries by execution time
grep -A 2 "Query_time:" /var/log/mysql/slow-query.log | \
  grep "Query_time" | sort -t: -k2 -rn | head -10

# Count slow queries per hour
grep "^# Time:" /var/log/mysql/slow-query.log | \
  cut -d'T' -f2 | cut -d':' -f1 | sort | uniq -c | sort -rn

# Find queries examining > 1 million rows
grep "Rows_examined:" /var/log/mysql/slow-query.log | \
  awk -F'Rows_examined: ' '{print $2}' | awk '{if ($1 > 1000000) print $0}'
```

## mysqldumpslow — Built-in Log Analyzer

`mysqldumpslow` ships with MySQL and provides basic aggregation.

```bash
# Top 10 queries by total time
mysqldumpslow -s t -t 10 /var/log/mysql/slow-query.log

# Top 10 queries by count (most frequent slow queries)
mysqldumpslow -s c -t 10 /var/log/mysql/slow-query.log

# Top 10 queries by average time
mysqldumpslow -s at -t 10 /var/log/mysql/slow-query.log

# Top 10 queries by rows examined
mysqldumpslow -s r -t 10 /var/log/mysql/slow-query.log

# Filter to a specific database
mysqldumpslow -s t -t 10 -g "FROM orders" /var/log/mysql/slow-query.log
```

### Sort Options
- `-s t` -- sort by total query time
- `-s c` -- sort by count
- `-s at` -- sort by average query time
- `-s r` -- sort by rows sent
- `-s ar` -- sort by average rows sent

## pt-query-digest — Professional Analysis (Percona Toolkit)

`pt-query-digest` is the industry-standard tool for slow query log analysis. It normalizes queries, computes statistics, and produces actionable reports.

### Installation

```bash
# Percona Toolkit installation
# Debian/Ubuntu
apt-get install percona-toolkit

# RHEL/CentOS
yum install percona-toolkit

# macOS
brew install percona-toolkit
```

### Basic Usage

```bash
# Analyze the entire slow query log
pt-query-digest /var/log/mysql/slow-query.log

# Analyze only queries from the last 24 hours
pt-query-digest --since '24h' /var/log/mysql/slow-query.log

# Analyze a specific time range
pt-query-digest --since '2025-06-14' --until '2025-06-15' /var/log/mysql/slow-query.log

# Filter to specific database
pt-query-digest --filter '$event->{db} eq "production"' /var/log/mysql/slow-query.log

# Filter to queries examining > 100k rows
pt-query-digest --filter '$event->{Rows_examined} > 100000' /var/log/mysql/slow-query.log

# Save results to a review table for tracking over time
pt-query-digest --review h=localhost,D=slow_query_review,t=query_review \
  /var/log/mysql/slow-query.log

# Output as JSON for programmatic consumption
pt-query-digest --output json /var/log/mysql/slow-query.log
```

### Reading pt-query-digest Output

The report has three sections:

**1. Overall Summary**
```
# Overall: 4.82k total, 127 unique, 3.35 QPS, 1.23x concurrency
# Time range: 2025-06-14T00:00:00 to 2025-06-15T00:00:00
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time          1774s   100ms    45s      368ms   1s    892ms   245ms
# Lock time            12s       0   200ms     2ms     8ms    15ms    1ms
# Rows sent         1.24M       0  50.00k   257.42  487.00  1.82k   97.36
# Rows examine     82.45M       0   5.23M  17.10k  98.15k 142.56k  2.87k
```

**2. Profile (Top Queries Ranked by Total Time)**
```
# Rank Query ID                      Response time  Calls  R/Call  V/M
# ==== ============================  ============== ====== ======= =====
#    1 0xA1B2C3D4E5F6A7B8           450.2300 25.4%    342  1.3163  0.82
#    2 0xB2C3D4E5F6A7B8C9           312.8900 17.6%     89  3.5157  2.14
#    3 0xC3D4E5F6A7B8C9D0           198.4500 11.2%   1205  0.1647  0.03
```

**3. Query Detail**
```
# Query 1: 0.24 QPS, 0.31x concurrency, ID 0xA1B2C3D4E5F6A7B8 at byte 45023
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          7     342
# Exec time     25    450s   200ms    12s      1s      3s      2s    800ms
# Lock time      2   240ms    0       50ms    1ms     2ms     5ms    0
# Rows sent      4  51.30k       1     150   150.00  150.00    0.00  150.00
# Rows examine  38  31.24M  10.50k 248.50k  91.35k 201.74k  62.54k  78.23k
# EXPLAIN
SELECT o.order_id, o.order_date, c.customer_name ...
```

### Comparing Reports Over Time

```bash
# Generate daily digests and compare
pt-query-digest --since '2025-06-14' --until '2025-06-15' slow-query.log > digest_0614.txt
pt-query-digest --since '2025-06-15' --until '2025-06-16' slow-query.log > digest_0615.txt
diff digest_0614.txt digest_0615.txt
```

## Log Rotation

The slow query log can grow very large on busy systems. Proper rotation is essential.

### Using FLUSH SLOW LOGS

```bash
#!/bin/bash
# rotate_slow_log.sh — safe slow query log rotation
LOG_DIR="/var/log/mysql"
LOG_FILE="slow-query.log"
RETENTION_DAYS=14

cd "$LOG_DIR" || exit 1

# Rename current log
mv "$LOG_FILE" "${LOG_FILE}.$(date +%Y%m%d%H%M%S)"

# Tell MySQL to open a new log file
mysql -u root -e "FLUSH SLOW LOGS;"

# Compress old logs
find "$LOG_DIR" -name "slow-query.log.*" -mmin +5 -exec gzip {} \;

# Remove logs older than retention period
find "$LOG_DIR" -name "slow-query.log.*.gz" -mtime +${RETENTION_DAYS} -delete
```

### Using logrotate (Linux)

```
# /etc/logrotate.d/mysql-slow-log
/var/log/mysql/slow-query.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 640 mysql mysql
    postrotate
        /usr/bin/mysql -u root -e "FLUSH SLOW LOGS;" 2>/dev/null || true
    endscript
}
```

## Performance Schema Alternative

For environments where filesystem-based log analysis is impractical (e.g., managed databases like RDS/Aurora), use Performance Schema.

### events_statements_summary_by_digest

```sql
-- Top 20 queries by total execution time
SELECT
    SCHEMA_NAME,
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_time_sec,
    ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_time_sec,
    ROUND(MAX_TIMER_WAIT / 1e12, 2) AS max_time_sec,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 1) AS examine_to_sent_ratio,
    FIRST_SEEN,
    LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
  AND DIGEST_TEXT NOT LIKE 'EXPLAIN%'
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20\G
```

### Queries with High Rows Examined to Rows Sent Ratio

```sql
-- Queries that examine many rows but return few (candidates for index optimization)
SELECT
    SCHEMA_NAME,
    LEFT(DIGEST_TEXT, 120) AS query_preview,
    COUNT_STAR AS exec_count,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT,
    ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 0) AS examine_per_sent,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
  AND SUM_ROWS_EXAMINED / SUM_ROWS_SENT > 1000
  AND COUNT_STAR > 10
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 20;
```

### Full Scan Queries

```sql
-- Queries doing full table scans
SELECT
    SCHEMA_NAME,
    LEFT(DIGEST_TEXT, 120) AS query_preview,
    COUNT_STAR AS exec_count,
    SUM_NO_INDEX_USED AS full_scans,
    SUM_NO_GOOD_INDEX_USED AS no_good_index,
    ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE (SUM_NO_INDEX_USED > 0 OR SUM_NO_GOOD_INDEX_USED > 0)
  AND COUNT_STAR > 100
ORDER BY SUM_NO_INDEX_USED DESC
LIMIT 20;
```

### Using sys Schema for Statement Analysis

```sql
-- sys schema provides pre-built views (MySQL 5.7+)
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 20;

-- Statements with full table scans
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC LIMIT 20;

-- Statements with temp tables
SELECT * FROM sys.statements_with_temp_tables
ORDER BY disk_tmp_tables DESC LIMIT 20;

-- Statements with sorting
SELECT * FROM sys.statements_with_sorting
ORDER BY total_latency DESC LIMIT 20;
```

### Resetting Performance Schema Statistics

```sql
-- Reset statement statistics (useful before a test window)
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;

-- Alternatively, reset all Performance Schema statistics
CALL sys.ps_truncate_all_tables(FALSE);
```

## Workflow: From Slow Log to Fix

```sql
-- Step 1: Identify the top offender from pt-query-digest or Performance Schema
-- (using Performance Schema example)
SELECT DIGEST, DIGEST_TEXT, COUNT_STAR, 
       ROUND(SUM_TIMER_WAIT/1e12, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 1\G

-- Step 2: Get the full query text (if DIGEST_TEXT is truncated)
SELECT DIGEST_TEXT
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST = 'abc123...'\G

-- Step 3: EXPLAIN the query
EXPLAIN FORMAT=TREE <paste the query here>\G

-- Step 4: Check for missing indexes, full scans, or suboptimal access types
-- Step 5: Create index or rewrite query
-- Step 6: Verify improvement with EXPLAIN ANALYZE
-- Step 7: Monitor Performance Schema to confirm improvement in production
```

## Best Practices

- Set `long_query_time` to 1 second or less in production; 10 seconds is too high to catch problems early
- Enable `log_slow_extra` (MySQL 8.0+) for additional handler and sort statistics
- Use `pt-query-digest` for regular analysis, not just firefighting -- run it daily or weekly
- Enable `log_queries_not_using_indexes` only in dev/staging or with throttling enabled to avoid log floods
- Rotate logs daily and retain at least 14 days for trend analysis
- Combine slow query log analysis with Performance Schema for a complete picture
- Track slow query trends over time using pt-query-digest's `--review` feature
- Use Performance Schema on managed databases (RDS, Aurora, Cloud SQL) where filesystem access is limited
- Reset Performance Schema digests before performance testing to get clean measurements

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Leaving `long_query_time` at the default 10 seconds | Misses queries in the 1-9 second range that cumulatively cause major performance issues | Set to 1 second or lower; adjust based on workload |
| Enabling `log_queries_not_using_indexes` without throttling | Log grows to gigabytes per hour on busy systems, filling disk | Set `log_throttle_queries_not_using_indexes = 60` to cap at 60/minute |
| Never rotating the slow query log | Disk fills up; log analysis takes forever | Set up daily log rotation with FLUSH SLOW LOGS |
| Analyzing only the slowest individual queries | Misses high-frequency moderate queries (1s x 10,000/day = 2.7 hours of DB time) | Sort by total time (`-s t` in mysqldumpslow, rank by total_time in pt-query-digest) |
| Using mysqldumpslow instead of pt-query-digest | Misses statistical distributions, percentiles, and trend analysis | Install Percona Toolkit; use pt-query-digest for production analysis |
| Ignoring Rows_examined / Rows_sent ratio | Missing queries that read millions of rows to return a handful | Flag queries where examined/sent > 1000 for index review |
| Not capturing the baseline before changes | Cannot prove whether a change actually improved performance | Always capture a digest report before and after schema or query changes |

## MySQL Version Notes

**MySQL 5.7**:
- Slow query log available with all standard fields
- `log_slow_extra` not available (added in 8.0.14)
- Performance Schema events_statements_summary_by_digest available
- sys schema included by default
- `log_throttle_queries_not_using_indexes` available

**MySQL 8.0**:
- `log_slow_extra = ON` adds handler counts, sort stats, tmp table stats (8.0.14+)
- `log_slow_replica_statements` replaces `log_slow_slave_statements` (deprecated terminology)
- Improved Performance Schema digest storage (longer DIGEST_TEXT, configurable `performance_schema_max_digest_length`)
- sys schema enhancements with additional statement analysis views
- Component-based slow query log filter plugins possible

**MySQL 8.4 / 9.x**:
- Further Performance Schema enhancements for statement telemetry
- Improved digest normalization for prepared statements
- Better integration with observability platforms via component infrastructure
- Deprecated `log_slow_slave_statements` removed; use `log_slow_replica_statements`

## Sources

- [MySQL Slow Query Log](https://dev.mysql.com/doc/refman/8.0/en/slow-query-log.html)
- [MySQL Performance Schema Statement Digests](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-statement-digests.html)
- [pt-query-digest Documentation](https://docs.percona.com/percona-toolkit/pt-query-digest.html)
- [MySQL sys Schema](https://dev.mysql.com/doc/refman/8.0/en/sys-schema.html)
- [MySQL Server System Variables (log_slow_extra)](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_log_slow_extra)
