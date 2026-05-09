# MySQL Error Log — Configuration, Filtering, and Monitoring

## Overview

The MySQL error log is the primary diagnostic log for the MySQL server. It records
server startup and shutdown events, critical errors, warnings, and notes that occur
during operation. Every DBA should know how to locate, configure, and monitor this
log because it is the first place to look when something goes wrong.

Starting with MySQL 8.0, the error log was redesigned around a component-based
architecture (`log_error_services`). This replaced the simpler `log_error` +
`log_error_verbosity` model from 5.7, adding JSON output, filtering components,
and the ability to chain multiple log sinks. Understanding both models is essential
for mixed-version environments.

Use this skill whenever you need to troubleshoot server crashes, replication failures,
InnoDB recovery issues, authentication errors, or any unexpected behavior. The error
log is also critical for proactive monitoring — parsing it regularly for warnings
can prevent outages.

## Key Concepts

- **Error Log Location**: Controlled by `log_error` system variable. Defaults vary
  by platform and installation method.
- **Verbosity**: `log_error_verbosity` controls which severity levels are logged
  (1=errors, 2=errors+warnings, 3=errors+warnings+notes).
- **Log Components (8.0+)**: Modular filter and sink components that form a
  processing pipeline for error events.
- **Log Sink**: A destination for error events (file, JSON file, syslog, table).
- **Log Filter**: A component that suppresses or modifies events before they reach
  a sink (e.g., `log_filter_internal`, `log_filter_dragnet`).
- **Error Event Priority**: System > Error > Warning > Note.

## Error Log Location

### Finding the Current Error Log

```sql
-- Show current error log file path
SHOW VARIABLES LIKE 'log_error';

-- On Linux, typical locations:
--   /var/log/mysqld.log          (RPM installs)
--   /var/log/mysql/error.log     (Debian/Ubuntu)
--   <datadir>/<hostname>.err     (generic/tarball installs)

-- Check datadir for default location fallback
SHOW VARIABLES LIKE 'datadir';
```

### Setting the Error Log Location

In `my.cnf` / `my.ini`:

```ini
[mysqld]
# Absolute path — logs to this specific file
log_error = /var/log/mysql/mysqld.log

# Relative path — relative to datadir
log_error = error.log

# Empty or omit — logs to stderr (console)
# log_error =
```

### Logging to stderr (Foreground/Container Deployments)

```ini
[mysqld]
# Omit log_error or set to stderr for container/foreground use
log_error_services = 'log_filter_internal; log_sink_internal'
# The server writes to stderr when no file is configured
```

## Log Verbosity (5.7 and 8.0+)

```sql
-- Check current verbosity
SHOW VARIABLES LIKE 'log_error_verbosity';

-- Level 1: Errors only
SET GLOBAL log_error_verbosity = 1;

-- Level 2: Errors and warnings (default)
SET GLOBAL log_error_verbosity = 2;

-- Level 3: Errors, warnings, and informational notes
SET GLOBAL log_error_verbosity = 3;
```

Persistent configuration:

```ini
[mysqld]
log_error_verbosity = 3
```

## Component-Based Error Logging (8.0+)

### Architecture

MySQL 8.0 introduced a pipeline model: error events flow through filter components
and then to sink components. The pipeline is defined by `log_error_services`.

```sql
-- Show current pipeline
SHOW VARIABLES LIKE 'log_error_services';

-- Default pipeline (built-in filter + built-in file sink)
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal';
```

### Available Components

| Component              | Type   | Description                                  |
|------------------------|--------|----------------------------------------------|
| `log_filter_internal`  | Filter | Built-in filter using `log_error_verbosity`  |
| `log_filter_dragnet`   | Filter | Rule-based filtering with custom expressions |
| `log_sink_internal`    | Sink   | Traditional text format error log file       |
| `log_sink_json`        | Sink   | JSON-formatted error log file                |
| `log_sink_syseventlog` | Sink   | System event log (syslog on Linux)           |
| `log_sink_test`        | Sink   | For testing only                             |

### Installing and Using Components

```sql
-- Install the JSON sink component
INSTALL COMPONENT 'file://component_log_sink_json';

-- Enable JSON logging alongside traditional logging
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_internal; log_sink_json';
-- Creates <log_error_basename>.NN.json alongside the .log file

-- Install syslog sink
INSTALL COMPONENT 'file://component_log_sink_syseventlog';
SET GLOBAL log_error_services = 'log_filter_internal; log_sink_syseventlog';

-- Install dragnet (rule-based) filter
INSTALL COMPONENT 'file://component_log_filter_dragnet';
SET GLOBAL log_error_services = 'log_filter_dragnet; log_sink_internal';
```

### Dragnet Filter Rules (Advanced Filtering)

```sql
-- Suppress specific error codes
SET GLOBAL dragnet.log_error_filter_rules =
  'IF err_code == 1287 THEN drop.';

-- Suppress notes (similar to verbosity=2 but with more control)
SET GLOBAL dragnet.log_error_filter_rules =
  'IF prio >= NOTE THEN drop.';

-- Only keep errors from InnoDB subsystem
SET GLOBAL dragnet.log_error_filter_rules =
  'IF NOT subsystem == "InnoDB" THEN drop.';
```

### JSON Error Log Format

When `log_sink_json` is enabled, each event is a JSON object:

```json
{
  "prio": 2,
  "err_code": 11321,
  "source_line": 1205,
  "source_file": "sql_parse.cc",
  "subsystem": "Server",
  "ts": 1620000000000000,
  "thread": 42,
  "err_symbol": "ER_PARSER_TRACE",
  "SQL_state": "HY000",
  "label": "Warning",
  "msg": "Example warning message"
}
```

## Error Log Table (mysql.general_log Alternative)

```sql
-- The error_log table was added in MySQL 8.0.22
-- It stores recent error log entries in the Performance Schema
SELECT * FROM performance_schema.error_log
ORDER BY LOGGED DESC
LIMIT 20;

-- Filter by priority
SELECT LOGGED, PRIO, ERROR_CODE, SUBSYSTEM, DATA
FROM performance_schema.error_log
WHERE PRIO = 'Error'
ORDER BY LOGGED DESC
LIMIT 50;

-- Filter by subsystem
SELECT LOGGED, PRIO, ERROR_CODE, DATA
FROM performance_schema.error_log
WHERE SUBSYSTEM = 'InnoDB'
ORDER BY LOGGED DESC;

-- Search for specific messages
SELECT LOGGED, PRIO, DATA
FROM performance_schema.error_log
WHERE DATA LIKE '%crash%' OR DATA LIKE '%recovery%'
ORDER BY LOGGED DESC;
```

## Common Error Messages and What They Mean

### Startup / Shutdown

| Message Pattern                                        | Meaning                                   |
|--------------------------------------------------------|-------------------------------------------|
| `ready for connections`                                | Server started successfully               |
| `Shutdown complete`                                    | Clean shutdown finished                   |
| `InnoDB: Database was not shutdown normally!`          | Crash recovery will run                   |
| `InnoDB: Starting crash recovery`                     | InnoDB is replaying redo logs             |
| `Too many connections`                                 | `max_connections` limit reached            |

### Replication

| Message Pattern                                        | Meaning                                   |
|--------------------------------------------------------|-------------------------------------------|
| `Slave SQL thread retried transaction N time(s)`       | Transient replication error with retry     |
| `Error 'Duplicate entry' on query`                     | Replication conflict — duplicate key       |
| `The slave coordinator and worker threads are stopped` | Replication has halted; investigate        |
| `Got fatal error 1236 from master`                     | Binary log position lost; re-sync needed  |

### InnoDB

| Message Pattern                                        | Meaning                                   |
|--------------------------------------------------------|-------------------------------------------|
| `InnoDB: page cleaner took Nms`                        | I/O subsystem too slow; check disk perf   |
| `InnoDB: Long semaphore wait`                          | Contention; potential hang if > 600s       |
| `InnoDB: Cannot allocate memory`                       | Buffer pool or OS memory exhausted         |
| `InnoDB: Deadlock found`                               | Transaction deadlock detected and rolled back |

## Log Rotation

### Linux logrotate (Recommended for File-Based Logs)

```bash
# /etc/logrotate.d/mysql
/var/log/mysql/mysqld.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 640 mysql mysql
    postrotate
        # Signal MySQL to reopen log file
        if test -x /usr/bin/mysqladmin; then
            /usr/bin/mysqladmin --defaults-file=/root/.my.cnf flush-error-log
        fi
    endscript
}
```

### Manual Flush

```sql
-- Flush and reopen the error log (after rotating the file externally)
FLUSH ERROR LOGS;

-- In 8.0+ this closes and reopens the file specified by log_error
-- In 5.7 this renames the old file with a -old suffix and creates a new one
```

### Rotation with performance_schema.error_log Table

```sql
-- The error_log table is ring-buffer based, controlled by:
SHOW VARIABLES LIKE 'log_error_services';  -- must include log_sink_internal
-- The table retains a limited number of rows automatically
-- No explicit rotation needed for the table
```

## Monitoring for Critical Errors

### Shell Script: Tail and Alert on Errors

```bash
#!/bin/bash
# Monitor MySQL error log for critical messages
LOG_FILE=$(mysql -N -e "SHOW VARIABLES LIKE 'log_error'" | awk '{print $2}')

tail -F "$LOG_FILE" | while read -r line; do
    if echo "$line" | grep -qiE '(fatal|crash|corruption|assert|cannot allocate|long semaphore)'; then
        echo "CRITICAL: $line" | mail -s "MySQL Alert: $(hostname)" dba-team@example.com
    fi
done
```

### SQL-Based Monitoring (8.0.22+)

```sql
-- Check for errors in the last hour
SELECT LOGGED, PRIO, ERROR_CODE, SUBSYSTEM, DATA
FROM performance_schema.error_log
WHERE PRIO IN ('Error', 'Warning')
  AND LOGGED > NOW() - INTERVAL 1 HOUR
ORDER BY LOGGED DESC;

-- Count errors by subsystem in the last 24 hours
SELECT SUBSYSTEM, PRIO, COUNT(*) AS cnt
FROM performance_schema.error_log
WHERE LOGGED > NOW() - INTERVAL 24 HOUR
GROUP BY SUBSYSTEM, PRIO
ORDER BY cnt DESC;
```

## Best Practices

- Always set `log_error_verbosity = 3` in non-production environments for maximum
  diagnostic detail.
- Use `log_error_verbosity = 2` in production (errors + warnings) as the minimum.
- Configure log rotation to prevent the error log from consuming disk space.
- In MySQL 8.0+, enable `log_sink_json` alongside `log_sink_internal` for easier
  automated parsing while keeping human-readable logs.
- Monitor the error log continuously with a log shipper (Filebeat, Fluentd, Promtail)
  or a tail-based alerting script.
- After any crash or unexpected restart, review the error log before doing anything
  else — it will show crash recovery progress and any corruption detected.
- In containerized deployments, log to stderr and let the container runtime handle
  log collection.
- Use `performance_schema.error_log` for quick lookback queries instead of grep on
  large log files.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Leaving `log_error_verbosity = 1` in production | Warnings about deprecations, replication lag, and InnoDB pressure are silently lost | Set to at least 2; use 3 for troubleshooting |
| Not configuring log rotation | Error log grows until disk fills, causing a server crash | Set up logrotate with `FLUSH ERROR LOGS` in postrotate |
| Changing `log_error_services` without installing components | Server falls back to default logging or fails to start | Run `INSTALL COMPONENT` before referencing new components |
| Ignoring `InnoDB: page cleaner` warnings | Indicates I/O bottleneck that degrades performance over time | Investigate disk latency; increase `innodb_io_capacity` |
| Parsing the traditional log with JSON tools | Text format is not machine-parseable; fragile regex patterns break | Enable `log_sink_json` and parse the `.json` log file instead |
| Not reviewing error log after version upgrade | Deprecation warnings and behavior changes go unnoticed | Read the full error log after every upgrade |

## MySQL Version Notes

- **5.7**: Simple model — `log_error` sets the file path, `log_error_verbosity`
  controls what is logged. No component architecture. `log_syslog` variable
  controls syslog output. No JSON sink. No `performance_schema.error_log` table.
- **8.0**: Component-based architecture via `log_error_services`. JSON sink
  (`log_sink_json`), syslog sink (`log_sink_syseventlog`), and dragnet filter
  (`log_filter_dragnet`) available as loadable components. `log_syslog` removed
  in favor of `log_sink_syseventlog`. `performance_schema.error_log` table added
  in 8.0.22.
- **8.4 / 9.x**: Same component architecture as 8.0. Some error codes and subsystem
  names may change. `log_error_suppression_list` (added 8.0.13) continues to work
  for suppressing specific error codes without needing dragnet. Check release notes
  for any new default component changes.

## Sources

- [MySQL 8.0 Reference Manual: The Error Log](https://dev.mysql.com/doc/refman/8.0/en/error-log.html)
- [MySQL 8.0: Error Log Configuration](https://dev.mysql.com/doc/refman/8.0/en/error-log-configuration.html)
- [MySQL 8.0: Error Log Components](https://dev.mysql.com/doc/refman/8.0/en/error-log-component-configuration.html)
- [MySQL 8.0: log_error_services](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_log_error_services)
- [MySQL 8.0: performance_schema.error_log](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-error-log-table.html)
- [MySQL 5.7 Reference Manual: The Error Log](https://dev.mysql.com/doc/refman/5.7/en/error-log.html)
