# SQL Server Error Log — Reading, Managing, and Interpreting Diagnostic Logs

## Overview

The SQL Server error log is the primary diagnostic log for the database engine, recording startup information, configuration changes, errors, warnings, and informational messages. Every SQL Server instance maintains a set of numbered error log files that cycle on restart or on demand.

Understanding the error log is essential for troubleshooting startup failures, login issues, memory pressure, I/O errors, Availability Group state changes, and many other operational problems. SQL Server also writes a subset of events to the Windows Application Event Log, and SQL Server Agent maintains its own separate error log.

The error log works alongside other diagnostic sources — Extended Events, the default trace, ring buffers, and the Windows Event Log — but it remains the first place to look when something goes wrong. Configuring proper log retention, cycling, and monitoring is a baseline operational requirement for every SQL Server instance.

## Key Concepts

- **Error log location** — Determined by the `-e` startup parameter. Default: `<install_path>\MSSQL\Log\ERRORLOG`. Viewable via `SERVERPROPERTY('ErrorLogFileName')`.
- **Log cycling** — A new error log is created on each SQL Server restart and when `sp_cycle_errorlog` is executed. The current log is `ERRORLOG`; previous logs are numbered `ERRORLOG.1`, `ERRORLOG.2`, etc.
- **Retention count** — By default SQL Server retains 6 archived logs. Configurable from 6 to 99 via SSMS or registry.
- **Severity levels** — Messages range from informational (severity 1-9) to fatal (severity 20-25). Severities 19-25 also write to the Windows Application Event Log.
- **SQL Server Agent log** — A separate log at `<install_path>\MSSQL\Log\SQLAGENT.OUT` recording agent job activity, alerts, and errors.

## Reading the Error Log

### sp_readerrorlog

```sql
-- Read current error log (log number 0)
EXEC sp_readerrorlog;

-- Read a specific archived log (1 = previous, 2 = two back, etc.)
EXEC sp_readerrorlog 1;

-- Read SQL Server Agent log (log type 2)
EXEC sp_readerrorlog 0, 2;

-- Search for a specific string
EXEC sp_readerrorlog 0, 1, N'error';

-- Search with two strings (AND logic)
EXEC sp_readerrorlog 0, 1, N'Login failed', N'sa';

-- Search within a time range
EXEC sp_readerrorlog 0, 1, NULL, NULL,
    '2026-05-01 00:00:00',
    '2026-05-09 23:59:59';
```

### xp_readerrorlog (Extended Procedure)

```sql
-- xp_readerrorlog provides more parameters than sp_readerrorlog
-- Parameters: log_number, log_type (1=SQL, 2=Agent),
--             search_string1, search_string2,
--             start_time, end_time, sort_order (N'asc' or N'desc')

EXEC xp_readerrorlog 0, 1,
    N'I/O',            -- search string 1
    NULL,              -- search string 2
    NULL,              -- start time
    NULL,              -- end time
    N'desc';           -- sort newest first

-- Find all login failures in the last 24 hours
EXEC xp_readerrorlog 0, 1,
    N'Login failed',
    NULL,
    DATEADD(HOUR, -24, GETDATE()),
    GETDATE(),
    N'desc';
```

### Finding the Error Log Location

```sql
-- Method 1: Server property
SELECT SERVERPROPERTY('ErrorLogFileName') AS error_log_path;

-- Method 2: From startup parameters in the registry
EXEC xp_instance_regread
    N'HKEY_LOCAL_MACHINE',
    N'SOFTWARE\Microsoft\MSSQLServer\MSSQLServer\Parameters',
    N'SQLArg1';

-- Method 3: From sys.dm_os_server_diagnostics_log_configurations
-- (SQL Server 2012+)
SELECT [path], max_size, max_files
FROM sys.dm_os_server_diagnostics_log_configurations;
```

## Error Log Cycling and Retention

### Cycling the Error Log

```sql
-- Manually cycle the error log (creates a new ERRORLOG,
-- renames current to ERRORLOG.1, etc.)
EXEC sp_cycle_errorlog;
```

It is a best practice to cycle the error log on a schedule (e.g., daily or weekly via a SQL Agent job) to keep individual log files manageable in size.

```sql
-- SQL Agent job step to cycle the log weekly
-- Step type: T-SQL
EXEC sp_cycle_errorlog;
```

### Configuring Retention Count

```sql
-- Check current number of error logs retained
-- (SSMS: Management > SQL Server Logs > right-click > Configure)

-- Via registry (requires restart to take effect)
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'SOFTWARE\Microsoft\MSSQLServer\MSSQLServer',
    N'NumErrorLogs',
    N'REG_DWORD',
    30;    -- retain 30 archived logs

-- Verify the setting
EXEC xp_instance_regread
    N'HKEY_LOCAL_MACHINE',
    N'SOFTWARE\Microsoft\MSSQLServer\MSSQLServer',
    N'NumErrorLogs';
```

## Common Error Messages and Interpretation

### Startup and Recovery

```sql
-- Search for startup messages
EXEC sp_readerrorlog 0, 1, N'Recovery is complete';
EXEC sp_readerrorlog 0, 1, N'Starting up database';

-- Database recovery progress
EXEC sp_readerrorlog 0, 1, N'Recovery of database';
```

**Key startup messages:**
- `Recovery is complete. This is an informational message only.` — Normal startup completed.
- `Starting up database '<name>'.` — Each database recovery begins.
- `Recovery of database '<name>' (N) is M% complete` — Long recovery progress.
- `Server is listening on` — Network endpoints are ready.

### Memory and Resource Pressure

```sql
-- Memory-related errors
EXEC sp_readerrorlog 0, 1, N'memory';
EXEC sp_readerrorlog 0, 1, N'buffer pool';
EXEC sp_readerrorlog 0, 1, N'AppDomain';
```

**Key memory messages:**
- `A significant part of sql server process memory has been paged out` — OS memory pressure forcing SQL Server pages to disk.
- `Buffer Pool scan took N seconds` — Large buffer pool scan indicates memory pressure.
- `Failed to allocate` — Out-of-memory condition.

### I/O Errors

```sql
-- I/O warnings and errors
EXEC sp_readerrorlog 0, 1, N'I/O requests taking longer than';
EXEC sp_readerrorlog 0, 1, N'FlushCache';
EXEC sp_readerrorlog 0, 1, N'Operating system error';
```

**Key I/O messages:**
- `SQL Server has encountered N occurrence(s) of I/O requests taking longer than 15 seconds` — Storage latency warning (severity: high).
- `FlushCache: cleaned up N bufs` — Lazy writer activity; monitor for frequency.
- `Operating system error 21(The device is not ready.)` — Disk/SAN connectivity issue.

### Login and Security

```sql
-- Login failures
EXEC sp_readerrorlog 0, 1, N'Login failed';

-- Successful logins (if login auditing is enabled)
EXEC sp_readerrorlog 0, 1, N'Login succeeded';
```

**Login failure reasons:**
- `Reason: Password did not match` — Wrong password.
- `Reason: Could not find a login matching the name provided` — Unknown login.
- `Reason: Token-based server access validation failed` — Windows auth issue.
- `Reason: The account is disabled` — Disabled SQL login.

### Configuring Login Auditing

```sql
-- Set login auditing level via registry
-- 0 = None, 1 = Successful logins only,
-- 2 = Failed logins only, 3 = Both
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'SOFTWARE\Microsoft\MSSQLServer\MSSQLServer',
    N'AuditLevel',
    N'REG_DWORD',
    2;    -- Failed logins only (recommended baseline)
-- Requires SQL Server restart to take effect
```

## Availability Group Error Patterns

```sql
-- AG state changes
EXEC sp_readerrorlog 0, 1, N'availability';
EXEC sp_readerrorlog 0, 1, N'AlwaysOn';
EXEC sp_readerrorlog 0, 1, N'HADR';

-- Replica role transitions
EXEC sp_readerrorlog 0, 1, N'changed from';

-- Lease timeout / health check
EXEC sp_readerrorlog 0, 1, N'lease';
```

**Key AG messages:**
- `The state of the local availability replica in availability group 'AG_Name' has changed from 'PRIMARY_NORMAL' to 'RESOLVING_NORMAL'` — Failover occurred.
- `Always On Availability Groups connection with secondary database terminated` — Replica disconnect.
- `The lease between availability group 'AG_Name' and the Windows Server Failover Cluster has expired` — Lease timeout triggered failover.
- `HADR: Thread pool` — HADR worker thread issues, possible resource exhaustion.

## Severity Levels

| Severity Range | Category | Action |
|---------------|----------|--------|
| 0-9 | Informational | No action needed; status messages |
| 10 | Informational (user errors) | Application-level errors returned to client |
| 11-16 | User-correctable errors | Fix application logic, permissions, or syntax |
| 17 | Insufficient resources | Monitor resource availability (memory, disk, locks) |
| 18 | Nonfatal internal error | Report to DBA; may self-resolve |
| 19 | Resource limit error | Contact DBA; may require config changes |
| 20 | Fatal error in current statement | Session may be terminated; investigate immediately |
| 21 | Fatal error in database process | Affects all sessions in the database |
| 22 | Fatal error: table integrity suspect | Run DBCC CHECKDB immediately |
| 23 | Fatal error: database integrity suspect | Database may be damaged; restore from backup |
| 24 | Fatal hardware error | Investigate storage/hardware immediately |
| 25 | Fatal system error | Systemwide failure; SQL Server may terminate |

## Windows Event Log Integration

```sql
-- Errors with severity >= 19 are automatically written to
-- the Windows Application Event Log.
-- Additionally, you can force messages to the event log:

RAISERROR('Custom monitoring message', 10, 1) WITH LOG;

-- SQL Server Agent alerts can be configured to fire on
-- specific error numbers or severity levels in the event log.
-- Example: alert on severity 17+ errors
EXEC msdb.dbo.sp_add_alert
    @name = N'Severity 17 Errors',
    @message_id = 0,
    @severity = 17,
    @notification_message = N'A severity 17 error has occurred.',
    @job_id = NULL;
```

## SQL Server Agent Error Log

```sql
-- Read the Agent error log
EXEC sp_readerrorlog 0, 2;   -- 2 = Agent log

-- Search Agent log for job failures
EXEC sp_readerrorlog 0, 2, N'failed';

-- Agent log location
-- Default: <install>\MSSQL\Log\SQLAGENT.OUT
-- Also cycles with Agent restart

-- Cycle the Agent error log
EXEC msdb.dbo.sp_cycle_agent_errorlog;
```

## Automated Error Log Monitoring

```sql
-- Create a table to track last-checked position
CREATE TABLE dbo.ErrorLogMonitor (
    last_check_time DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);
INSERT dbo.ErrorLogMonitor VALUES (SYSDATETIME());
GO

-- Procedure to check for new critical errors
CREATE OR ALTER PROCEDURE dbo.usp_CheckErrorLog
AS
BEGIN
    DECLARE @last_check DATETIME2;
    SELECT @last_check = last_check_time FROM dbo.ErrorLogMonitor;

    CREATE TABLE #errors (LogDate DATETIME, ProcessInfo VARCHAR(50), Text NVARCHAR(MAX));

    INSERT #errors
    EXEC xp_readerrorlog 0, 1, NULL, NULL, @last_check, NULL;

    -- Flag critical messages
    SELECT LogDate, ProcessInfo, Text
    FROM #errors
    WHERE Text LIKE '%Error:%'
       OR Text LIKE '%I/O requests taking longer%'
       OR Text LIKE '%Login failed%'
       OR Text LIKE '%Stack Dump%'
       OR Text LIKE '%memory%paged out%'
    ORDER BY LogDate DESC;

    UPDATE dbo.ErrorLogMonitor SET last_check_time = SYSDATETIME();
END;
GO
```

## Best Practices

- Configure error log retention to at least 30 files so that diagnostic history survives multiple restarts.
- Schedule `sp_cycle_errorlog` weekly or daily via a SQL Agent job to keep individual log files small and searchable.
- Set login auditing to "Failed logins only" as a minimum security baseline.
- Monitor for severity 17+ errors proactively using SQL Agent alerts or external monitoring tools.
- Check the error log after every SQL Server restart to verify all databases recovered cleanly.
- Review I/O warning messages (`I/O requests taking longer than 15 seconds`) as early indicators of storage problems.
- For Availability Groups, monitor for lease timeout and role transition messages to detect unexpected failovers.
- Archive old error logs before they are overwritten if you need long-term retention for compliance.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Leaving default retention of 6 error logs | Diagnostic history lost after 6 restarts or cycles | Increase to 30+ via SSMS or registry setting |
| Never cycling the error log | Single file grows to hundreds of MB; slow to search | Schedule weekly `sp_cycle_errorlog` via Agent job |
| Ignoring I/O latency warnings in the log | Storage degradation goes unnoticed until outage | Set up alerting on "I/O requests taking longer" messages |
| Not checking error log after patching or restart | Failed database recovery or configuration issues missed | Standard operating procedure: review log post-restart |
| Confusing sp_readerrorlog parameters | Reading wrong log (SQL vs Agent) or wrong archive number | Remember: param 1 = log number, param 2 = log type (1=SQL, 2=Agent) |
| Relying solely on Windows Event Log | Only severity 19+ errors appear there; many issues missed | Always check the SQL Server error log directly |

## SQL Server Version Notes

- **SQL Server 2016** — Added support for error log messages related to Query Store operations. TempDB configuration messages at startup improved.
- **SQL Server 2017** — Linux deployments write to `/var/opt/mssql/log/errorlog`. No Windows Event Log integration on Linux; use `journalctl` instead. Added automatic tuning messages.
- **SQL Server 2019** — Accelerated Database Recovery (ADR) messages appear during recovery. Verbose tempdb metadata contention logging added. Improved memory broker error messages.
- **SQL Server 2022** — Ledger verification messages. Buffer pool parallel scan logging. Improved Availability Group diagnostics with detailed failover reason codes. Contained AG error patterns differ from traditional AG.

## Sources

- [sp_readerrorlog](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-readerrorlog-transact-sql)
- [Viewing the SQL Server Error Log](https://learn.microsoft.com/en-us/sql/tools/configuration-manager/viewing-the-sql-server-error-log)
- [sp_cycle_errorlog](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-cycle-errorlog-transact-sql)
- [Database Engine Error Severities](https://learn.microsoft.com/en-us/sql/relational-databases/errors-events/database-engine-error-severities)
- [Configure Login Auditing](https://learn.microsoft.com/en-us/sql/ssms/configure-login-auditing-sql-server-management-studio)
