# SQL Server Agent — Job Scheduling, Automation, and Monitoring

## Overview

SQL Server Agent is the built-in job scheduling and automation service for SQL Server. It runs as a Windows service (`SQLSERVERAGENT` or `SQLAgent$<instance>`) and provides a complete framework for executing T-SQL, PowerShell, SSIS packages, and operating system commands on schedules, in response to alerts, or as part of multi-server administration.

Agent jobs are the backbone of database maintenance automation — index rebuilds, backup routines, log shipping, data purges, ETL orchestration, and custom monitoring. Each job consists of one or more steps that execute sequentially or branch based on success/failure outcomes. Jobs can notify operators via email (Database Mail), write to the Windows Event Log, or page on-call staff.

Understanding Agent architecture is critical for SQL Server administration. A misconfigured Agent can lead to missed backups, failed maintenance, runaway jobs, and security gaps. Agent metadata lives in the `msdb` database, so `msdb` backup is essential for preserving job definitions, schedules, and history.

## Key Concepts

- **Job** — A named unit of work containing one or more steps. Stored in `msdb.dbo.sysjobs`.
- **Job Step** — A single action within a job. Types include T-SQL, CmdExec, PowerShell, SSIS, and Replication. Stored in `msdb.dbo.sysjobsteps`.
- **Schedule** — A time-based trigger for job execution. One schedule can drive multiple jobs; one job can have multiple schedules. Stored in `msdb.dbo.sysschedules`.
- **Operator** — A notification recipient defined by name, email, and/or pager address. Stored in `msdb.dbo.sysoperators`.
- **Alert** — A reactive trigger that fires on a SQL Server error number, severity level, or performance condition. Can execute a job or notify an operator.
- **Proxy** — A security context that allows non-sysadmin users to execute job steps under a specific credential.
- **Category** — An organizational label for grouping related jobs.
- **MSX/TSX** — Multi-server administration model. The Master Server (MSX) pushes job definitions to Target Servers (TSX).

## Creating Jobs

### Using System Stored Procedures

```sql
-- Step 1: Create the job
EXEC msdb.dbo.sp_add_job
    @job_name = N'Daily Index Maintenance',
    @description = N'Rebuild fragmented indexes nightly',
    @category_name = N'Database Maintenance',
    @owner_login_name = N'sa',
    @enabled = 1,
    @notify_level_eventlog = 2,      -- on failure
    @notify_level_email = 2,         -- on failure
    @notify_email_operator_name = N'DBA_Team';

-- Step 2: Add job step(s)
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Daily Index Maintenance',
    @step_name = N'Rebuild Indexes',
    @step_id = 1,
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_RebuildFragmentedIndexes @FragThreshold = 30;',
    @database_name = N'AdventureWorks',
    @on_success_action = 1,    -- quit with success
    @on_fail_action = 2,       -- quit with failure
    @retry_attempts = 2,
    @retry_interval = 5;       -- minutes between retries

-- Step 3: Add schedule
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Nightly_2AM',
    @freq_type = 4,            -- daily
    @freq_interval = 1,        -- every 1 day
    @active_start_time = 020000;  -- 02:00:00

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'Daily Index Maintenance',
    @schedule_name = N'Nightly_2AM';

-- Step 4: Assign to local server
EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'Daily Index Maintenance',
    @server_name = N'(LOCAL)';
```

### Multi-Step Job with Branching

```sql
EXEC msdb.dbo.sp_add_job
    @job_name = N'ETL Pipeline',
    @description = N'Extract, transform, load with error handling';

-- Step 1: Extract
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'ETL Pipeline',
    @step_name = N'Extract Data',
    @step_id = 1,
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_ExtractData;',
    @database_name = N'StagingDB',
    @on_success_action = 3,    -- go to next step
    @on_fail_action = 4,       -- go to step (error handler)
    @on_fail_step_id = 4;

-- Step 2: Transform
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'ETL Pipeline',
    @step_name = N'Transform Data',
    @step_id = 2,
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_TransformData;',
    @database_name = N'StagingDB',
    @on_success_action = 3,
    @on_fail_action = 4,
    @on_fail_step_id = 4;

-- Step 3: Load
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'ETL Pipeline',
    @step_name = N'Load Data',
    @step_id = 3,
    @subsystem = N'TSQL',
    @command = N'EXEC dbo.usp_LoadData;',
    @database_name = N'WarehouseDB',
    @on_success_action = 1,    -- quit with success
    @on_fail_action = 4,
    @on_fail_step_id = 4;

-- Step 4: Error handler
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'ETL Pipeline',
    @step_name = N'Error Handler',
    @step_id = 4,
    @subsystem = N'TSQL',
    @command = N'
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = N''DBMailProfile'',
            @recipients = N''dba@company.com'',
            @subject = N''ETL Pipeline Failed'',
            @body = N''Check job history for details.'';
    ',
    @database_name = N'msdb',
    @on_success_action = 2;    -- quit with failure
```

## Schedule Types

```sql
-- Daily at specific time
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Daily_3AM',
    @freq_type = 4,               -- 4 = daily
    @freq_interval = 1,
    @active_start_time = 030000;

-- Weekly on specific days (Mon, Wed, Fri)
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'MWF_6PM',
    @freq_type = 8,               -- 8 = weekly
    @freq_interval = 42,          -- Mon(2) + Wed(8) + Fri(32) = 42
    @freq_recurrence_factor = 1,  -- every 1 week
    @active_start_time = 180000;

-- Monthly on day 1
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Monthly_First',
    @freq_type = 16,              -- 16 = monthly
    @freq_interval = 1,           -- day 1
    @freq_recurrence_factor = 1,
    @active_start_time = 010000;

-- Every 15 minutes during business hours
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Min_BizHours',
    @freq_type = 4,               -- daily
    @freq_interval = 1,
    @freq_subday_type = 4,        -- 4 = minutes
    @freq_subday_interval = 15,
    @active_start_time = 080000,  -- 8 AM
    @active_end_time = 180000;    -- 6 PM

-- One-time execution
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'RunOnce',
    @freq_type = 1,               -- 1 = one-time
    @active_start_date = 20260515,
    @active_start_time = 220000;
```

## Operators and Notifications

```sql
-- Create an operator
EXEC msdb.dbo.sp_add_operator
    @name = N'DBA_Team',
    @enabled = 1,
    @email_address = N'dba-team@company.com';

-- Add notification to existing job
EXEC msdb.dbo.sp_update_job
    @job_name = N'Daily Index Maintenance',
    @notify_level_email = 2,         -- 0=never, 1=success, 2=failure, 3=always
    @notify_email_operator_name = N'DBA_Team';

-- Notify on specific job completion
EXEC msdb.dbo.sp_add_notification
    @alert_name = N'Severity 17 Errors',
    @operator_name = N'DBA_Team',
    @notification_method = 1;       -- 1=email, 2=pager, 4=net send
```

## Proxies for Non-sysadmin Execution

```sql
-- Step 1: Create a credential mapping to a Windows account
CREATE CREDENTIAL [ETLCredential]
WITH IDENTITY = N'DOMAIN\etl_service_account',
     SECRET = N'P@ssw0rd!';

-- Step 2: Create a proxy using the credential
EXEC msdb.dbo.sp_add_proxy
    @proxy_name = N'ETL_Proxy',
    @credential_name = N'ETLCredential',
    @description = N'Proxy for ETL SSIS package execution';

-- Step 3: Grant the proxy access to specific subsystems
EXEC msdb.dbo.sp_grant_proxy_to_subsystem
    @proxy_name = N'ETL_Proxy',
    @subsystem_id = 11;    -- SSIS subsystem

EXEC msdb.dbo.sp_grant_proxy_to_subsystem
    @proxy_name = N'ETL_Proxy',
    @subsystem_id = 3;     -- CmdExec subsystem

-- Step 4: Grant login access to use the proxy
EXEC msdb.dbo.sp_grant_login_to_proxy
    @login_name = N'DOMAIN\etl_user',
    @proxy_name = N'ETL_Proxy';

-- Step 5: Use proxy in a job step
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'ETL Pipeline',
    @step_name = N'Run SSIS Package',
    @subsystem = N'SSIS',
    @command = N'/ISSERVER "\SSISDB\ETL\Project\Package.dtsx" /SERVER "." /Par "$ServerOption::LOGGING_LEVEL(Int16)";1',
    @proxy_name = N'ETL_Proxy';
```

## SSIS Package Execution

```sql
-- Execute SSIS package from SSISDB catalog
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'SSIS ETL Job',
    @step_name = N'Execute SSIS Package',
    @subsystem = N'SSIS',
    @command = N'/ISSERVER "\SSISDB\CatalogFolder\ProjectName\PackageName.dtsx"
                 /SERVER "."
                 /Par "$ServerOption::LOGGING_LEVEL(Int16)";1
                 /Par "$Project::ConnectionString";"Data Source=PROD;Initial Catalog=SourceDB;"
                 /CALLERINFO SQLAGENT',
    @database_name = N'master',
    @proxy_name = N'ETL_Proxy';
```

## PowerShell Job Steps

```sql
-- PowerShell job step
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Server Health Check',
    @step_name = N'Check Disk Space',
    @subsystem = N'PowerShell',
    @command = N'
        $threshold = 10GB
        $drives = Get-WmiObject Win32_LogicalDisk -Filter "DriveType=3"
        foreach ($drive in $drives) {
            if ($drive.FreeSpace -lt $threshold) {
                $msg = "Low disk: $($drive.DeviceID) - $([math]::Round($drive.FreeSpace/1GB,2)) GB free"
                Write-Warning $msg
                throw $msg
            }
        }
    ',
    @on_fail_action = 2;
```

## Monitoring Job History

### Querying Job History Tables

```sql
-- Recent job execution history
SELECT
    j.name AS job_name,
    h.step_id,
    h.step_name,
    CASE h.run_status
        WHEN 0 THEN 'Failed'
        WHEN 1 THEN 'Succeeded'
        WHEN 2 THEN 'Retry'
        WHEN 3 THEN 'Canceled'
        WHEN 4 THEN 'In Progress'
    END AS run_status,
    msdb.dbo.agent_datetime(h.run_date, h.run_time) AS run_datetime,
    STUFF(STUFF(RIGHT('000000' + CAST(h.run_duration AS VARCHAR(6)), 6),
        3, 0, ':'), 6, 0, ':') AS duration_hhmmss,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE h.step_id = 0   -- job outcome step
ORDER BY h.instance_id DESC;

-- Failed jobs in the last 24 hours
SELECT
    j.name AS job_name,
    h.step_name,
    msdb.dbo.agent_datetime(h.run_date, h.run_time) AS run_datetime,
    h.message
FROM msdb.dbo.sysjobhistory h
JOIN msdb.dbo.sysjobs j ON h.job_id = j.job_id
WHERE h.run_status = 0
  AND msdb.dbo.agent_datetime(h.run_date, h.run_time) > DATEADD(HOUR, -24, GETDATE())
ORDER BY h.instance_id DESC;
```

### Currently Running Jobs

```sql
SELECT
    j.name AS job_name,
    ja.start_execution_date,
    DATEDIFF(MINUTE, ja.start_execution_date, GETDATE()) AS running_minutes,
    ja.last_executed_step_id,
    js.step_name AS current_step
FROM msdb.dbo.sysjobactivity ja
JOIN msdb.dbo.sysjobs j ON ja.job_id = j.job_id
LEFT JOIN msdb.dbo.sysjobsteps js
    ON ja.job_id = js.job_id
   AND ja.last_executed_step_id + 1 = js.step_id
WHERE ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.syssessions)
  AND ja.start_execution_date IS NOT NULL
  AND ja.stop_execution_date IS NULL;
```

### Job Schedule Overview

```sql
SELECT
    j.name AS job_name,
    j.enabled AS job_enabled,
    s.name AS schedule_name,
    s.enabled AS schedule_enabled,
    CASE s.freq_type
        WHEN 1 THEN 'One-time'
        WHEN 4 THEN 'Daily'
        WHEN 8 THEN 'Weekly'
        WHEN 16 THEN 'Monthly'
        WHEN 32 THEN 'Monthly relative'
        WHEN 64 THEN 'On Agent start'
        WHEN 128 THEN 'When idle'
    END AS frequency,
    ja.next_scheduled_run_date
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
JOIN msdb.dbo.sysschedules s ON js.schedule_id = s.schedule_id
LEFT JOIN msdb.dbo.sysjobactivity ja ON j.job_id = ja.job_id
    AND ja.session_id = (SELECT MAX(session_id) FROM msdb.dbo.syssessions)
ORDER BY ja.next_scheduled_run_date;
```

## Alerts on Performance Conditions

```sql
-- Alert when tempdb data file exceeds 80% full
EXEC msdb.dbo.sp_add_alert
    @name = N'TempDB Space Alert',
    @message_id = 0,
    @severity = 0,
    @enabled = 1,
    @delay_between_responses = 900,  -- 15 minutes
    @performance_condition = N'SQLServer:Databases|Percent Log Used|tempdb|>|80',
    @notification_message = N'TempDB log usage exceeds 80%';

EXEC msdb.dbo.sp_add_notification
    @alert_name = N'TempDB Space Alert',
    @operator_name = N'DBA_Team',
    @notification_method = 1;

-- Alert on severity 19+ errors
EXEC msdb.dbo.sp_add_alert
    @name = N'Severity 19+ Errors',
    @severity = 19,
    @enabled = 1,
    @delay_between_responses = 60,
    @include_event_description_in = 1;

-- Alert on specific error number
EXEC msdb.dbo.sp_add_alert
    @name = N'Deadlock Alert',
    @message_id = 1205,
    @enabled = 1,
    @delay_between_responses = 60,
    @job_name = N'Capture Deadlock Details';
```

## Token Replacement

```sql
-- Token replacement in job step commands
-- Tokens are evaluated at runtime
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'Backup Job',
    @step_name = N'Full Backup with Tokens',
    @subsystem = N'TSQL',
    @command = N'
        BACKUP DATABASE [MyDB]
        TO DISK = N''C:\Backups\MyDB_$(ESCAPE_SQUOTE(JOBID))_$(ESCAPE_SQUOTE(STRTDT))_$(ESCAPE_SQUOTE(STRTTM)).bak''
        WITH COMPRESSION, CHECKSUM;
    ';

-- Common tokens:
-- $(ESCAPE_SQUOTE(JOBID))  - Job ID (GUID)
-- $(ESCAPE_SQUOTE(STRTDT)) - Job start date (YYYYMMDD)
-- $(ESCAPE_SQUOTE(STRTTM)) - Job start time (HHMMSS)
-- $(ESCAPE_SQUOTE(MACH))   - Machine name
-- $(ESCAPE_SQUOTE(INST))   - Instance name
-- $(ESCAPE_SQUOTE(SQLDIR)) - SQL Server install directory
-- $(ESCAPE_SQUOTE(SRVR))   - Server name
```

## Multi-Server Administration (MSX/TSX)

```sql
-- On the Master Server (MSX): enlist a target server
EXEC msdb.dbo.sp_msx_enlist
    @server_name = N'TargetServer01',
    @location = N'DataCenter1';

-- Create a multi-server job (runs on all TSX servers)
EXEC msdb.dbo.sp_add_job
    @job_name = N'Enterprise Backup Check',
    @description = N'Verify last backup across all servers';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'Enterprise Backup Check',
    @server_name = N'TargetServer01';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'Enterprise Backup Check',
    @server_name = N'TargetServer02';

-- On a Target Server: defect from MSX
EXEC msdb.dbo.sp_msx_defect;
```

## Best Practices

- Always define operators and configure notifications for job failures — silent failures are the most dangerous.
- Back up the `msdb` database regularly; all job definitions, schedules, and history live there.
- Use proxies instead of running all job steps as `sa` or the Agent service account — follow least-privilege principles.
- Set meaningful `@retry_attempts` and `@retry_interval` for steps that may encounter transient failures (e.g., network access, linked servers).
- Configure job history retention to keep enough rows for trend analysis (default max of 1000 total rows is often too low).
- Use job categories to organize jobs by function (Maintenance, ETL, Monitoring, Reporting).
- Avoid overlapping schedules for resource-intensive jobs; use the `@on_success_action = 4` (go to step) pattern to chain dependent steps.
- Always use `ESCAPE_SQUOTE()` around tokens to prevent SQL injection in token replacement.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Default job history retention (1000 rows total, 100 per job) | History purged too quickly; cannot troubleshoot past failures | Increase via `sp_set_sqlagent_properties` or SSMS Agent properties |
| Running CmdExec/PowerShell steps under the Agent service account | Overprivileged execution; security risk | Create dedicated proxies with minimal permissions |
| No notification on job failure | Failed jobs go unnoticed for hours or days | Configure operator email notifications for all production jobs |
| Not backing up msdb | Lose all job definitions, schedules, operator configs on disaster | Include msdb in your backup strategy |
| Using `@on_fail_action = 3` (go to next step) blindly | Job reports success even when intermediate steps fail | Use `@on_fail_action = 4` to branch to error-handling step |
| Overlapping schedule for long-running jobs | Multiple instances of the same job run concurrently, causing contention | Add logic to check for already-running instance or stagger schedules |

## SQL Server Version Notes

- **SQL Server 2016** — Agent supports SSIS Scale Out execution. Database Mail replaces SQL Mail as the sole notification method.
- **SQL Server 2017** — SQL Server Agent available on Linux (limited functionality: T-SQL and CmdExec steps only; no SSIS, PowerShell, or proxy support on Linux). Agent is packaged separately (`mssql-server-agent`).
- **SQL Server 2019** — Improved Agent job step execution on Linux. Better integration with PolyBase external data jobs. Agent on Linux supports additional job step types.
- **SQL Server 2022** — Agent improvements for contained Availability Groups. Elastic job agent integration for Azure SQL (hybrid scenarios). Improved error handling and logging in Agent subsystems.

## Sources

- [SQL Server Agent Overview](https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent)
- [sp_add_job](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-add-job-transact-sql)
- [sp_add_jobstep](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-add-jobstep-transact-sql)
- [Create a SQL Server Agent Proxy](https://learn.microsoft.com/en-us/sql/ssms/agent/create-a-sql-server-agent-proxy)
- [Use Tokens in Job Steps](https://learn.microsoft.com/en-us/sql/ssms/agent/use-tokens-in-job-steps)
- [Multi-Server Administration](https://learn.microsoft.com/en-us/sql/ssms/agent/automated-administration-across-an-enterprise)
