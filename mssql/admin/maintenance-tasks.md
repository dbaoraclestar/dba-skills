# SQL Server Maintenance Tasks — Index, Statistics, Integrity, and Automation

## Overview

Regular database maintenance is essential for sustaining SQL Server performance, reliability, and recoverability. Without proactive maintenance, indexes fragment, statistics become stale, corruption goes undetected, and performance degrades progressively. A well-designed maintenance strategy addresses index health, statistics accuracy, data integrity verification, and tempdb optimization.

This skill covers index maintenance (rebuild vs. reorganize), statistics updates, DBCC CHECKDB for consistency checking, database and file shrinking (and why to avoid it), tempdb best practices, instant file initialization, SQL Server Agent job configuration, and the trade-offs between maintenance plans and custom scripts. Use this guide when designing a maintenance strategy, troubleshooting performance degradation related to fragmentation or stale statistics, or setting up automated maintenance jobs.

The gold standard for SQL Server index and integrity maintenance is Ola Hallengren's Maintenance Solution, a set of free, open-source stored procedures used by thousands of organizations worldwide. This guide references Hallengren's approach alongside native T-SQL methods.

## Key Concepts

- **Index Fragmentation**: Physical disorder of index pages on disk. External fragmentation (logical vs. physical page order) degrades scan performance. Internal fragmentation (partially filled pages) wastes space.
- **Index Rebuild (ALTER INDEX REBUILD)**: Drops and recreates the index, removing all fragmentation. Can be done ONLINE (Enterprise) or OFFLINE. Generates full transaction log for the index.
- **Index Reorganize (ALTER INDEX REORGANIZE)**: An online, incremental, low-impact operation that defragments the leaf level. Always online, always resumable, minimal log generation.
- **Statistics**: Metadata about data distribution in columns, used by the query optimizer to estimate cardinalities and choose execution plans. Stale statistics lead to poor plan choices.
- **DBCC CHECKDB**: The primary tool for verifying logical and physical integrity of all objects in a database. Detects corruption before it causes data loss.
- **Auto-Shrink**: A database option that periodically releases unused file space. Causes severe fragmentation and should always be disabled.
- **Instant File Initialization (IFI)**: Allows SQL Server to skip zero-initialization when growing data files, dramatically reducing growth time. Does not apply to log files.

## Index Maintenance

### Assessing Fragmentation

```sql
-- Check fragmentation for all indexes in a database
-- Use LIMITED mode for quick assessment (reads parent pages only)
SELECT
    OBJECT_SCHEMA_NAME(ips.object_id) AS SchemaName,
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    i.type_desc AS IndexType,
    ips.partition_number,
    ips.index_depth,
    ips.index_level,
    CAST(ips.avg_fragmentation_in_percent AS DECIMAL(5,1)) AS FragPct,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.fragment_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i
    ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 1000       -- Skip tiny indexes
  AND ips.avg_fragmentation_in_percent > 5
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Detailed mode for specific index (reads all pages, slower)
SELECT *
FROM sys.dm_db_index_physical_stats(
    DB_ID(), OBJECT_ID('app.Orders'), 1, NULL, 'DETAILED');
```

### Rebuild vs. Reorganize Decision Matrix

| Fragmentation | Page Count | Action | Rationale |
|--------------|-----------|--------|-----------|
| < 5% | Any | None | Fragmentation is negligible |
| 5% - 30% | > 1000 | REORGANIZE | Low-impact, online, sufficient for moderate fragmentation |
| > 30% | > 1000 | REBUILD | Reorganize becomes inefficient at high fragmentation |
| Any | < 1000 | None | Small indexes fit in few extents; fragmentation is irrelevant |

### Index Rebuild

```sql
-- Offline rebuild (all editions, locks the table)
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders]
REBUILD WITH (
    FILLFACTOR = 90,
    SORT_IN_TEMPDB = ON,
    STATISTICS_NORECOMPUTE = OFF,
    DATA_COMPRESSION = PAGE
);

-- Online rebuild (Enterprise Edition)
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders]
REBUILD WITH (
    ONLINE = ON,
    SORT_IN_TEMPDB = ON,
    MAXDOP = 4
);

-- Resumable online rebuild (SQL 2017+)
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders]
REBUILD WITH (
    ONLINE = ON,
    RESUMABLE = ON,
    MAX_DURATION = 60    -- Pause after 60 minutes
);

-- Resume a paused rebuild
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders] RESUME;

-- Abort a paused rebuild
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders] ABORT;

-- Rebuild ALL indexes on a table
ALTER INDEX ALL ON [app].[Orders]
REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);
```

### Index Reorganize

```sql
-- Reorganize is always online and interruptible
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders] REORGANIZE;

-- Reorganize with LOB compaction
ALTER INDEX [IX_Orders_CustomerDate] ON [app].[Orders]
REORGANIZE WITH (LOB_COMPACTION = ON);

-- Reorganize all indexes on a table
ALTER INDEX ALL ON [app].[Orders] REORGANIZE;
```

### Ola Hallengren's IndexOptimize

```sql
-- Install Ola Hallengren's maintenance solution
-- Download from: https://ola.hallengren.com/

-- Index maintenance for all user databases
EXECUTE dbo.IndexOptimize
    @Databases = 'USER_DATABASES',
    @FragmentationLow = NULL,                   -- Skip < 5%
    @FragmentationMedium = 'INDEX_REORGANIZE',  -- Reorganize 5-30%
    @FragmentationHigh = 'INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE',
    @FragmentationLevel1 = 5,
    @FragmentationLevel2 = 30,
    @MinNumberOfPages = 1000,
    @UpdateStatistics = 'ALL',
    @OnlyModifiedStatistics = 'Y',
    @MaxDOP = 4,
    @SortInTempdb = 'Y',
    @TimeLimit = 14400,     -- 4 hour time limit
    @LogToTable = 'Y';

-- Check execution history
SELECT *
FROM dbo.CommandLog
WHERE CommandType IN ('ALTER_INDEX', 'UPDATE_STATISTICS')
ORDER BY StartTime DESC;
```

### Automated Index Maintenance Script

```sql
-- Custom script: rebuild or reorganize based on fragmentation
DECLARE @SchemaName NVARCHAR(128), @TableName NVARCHAR(128),
        @IndexName NVARCHAR(128), @FragPct DECIMAL(5,1),
        @PageCount BIGINT, @SQL NVARCHAR(MAX);

DECLARE idx_cursor CURSOR FOR
SELECT
    OBJECT_SCHEMA_NAME(ips.object_id),
    OBJECT_NAME(ips.object_id),
    i.name,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 1000
  AND ips.avg_fragmentation_in_percent > 5
  AND i.name IS NOT NULL
ORDER BY ips.avg_fragmentation_in_percent DESC;

OPEN idx_cursor;
FETCH NEXT FROM idx_cursor INTO @SchemaName, @TableName, @IndexName, @FragPct, @PageCount;

WHILE @@FETCH_STATUS = 0
BEGIN
    IF @FragPct > 30
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REBUILD WITH (ONLINE = ON, SORT_IN_TEMPDB = ON, MAXDOP = 4);';
    ELSE
        SET @SQL = 'ALTER INDEX [' + @IndexName + '] ON [' + @SchemaName + '].[' + @TableName + '] REORGANIZE;';

    PRINT @SQL;
    EXEC sp_executesql @SQL;

    FETCH NEXT FROM idx_cursor INTO @SchemaName, @TableName, @IndexName, @FragPct, @PageCount;
END;

CLOSE idx_cursor;
DEALLOCATE idx_cursor;
```

## Statistics Maintenance

```sql
-- Check statistics age and modification count
SELECT
    OBJECT_SCHEMA_NAME(s.object_id) AS SchemaName,
    OBJECT_NAME(s.object_id) AS TableName,
    s.name AS StatisticName,
    s.auto_created,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.modification_counter,
    CAST(sp.rows_sampled * 100.0 / NULLIF(sp.rows, 0) AS DECIMAL(5,1)) AS SamplePct
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE OBJECT_SCHEMA_NAME(s.object_id) NOT IN ('sys')
  AND sp.modification_counter > 0
ORDER BY sp.modification_counter DESC;

-- Update statistics for a specific table
UPDATE STATISTICS [app].[Orders] WITH FULLSCAN;

-- Update with sampling (faster for large tables)
UPDATE STATISTICS [app].[Orders] WITH SAMPLE 30 PERCENT;

-- Update a specific statistic
UPDATE STATISTICS [app].[Orders] [IX_Orders_CustomerDate] WITH FULLSCAN;

-- Update all statistics in the database
EXEC sp_updatestats;

-- Update only statistics that have been modified
-- (SQL Server 2016+ with trace flag 2371 or auto-update threshold)
EXEC sp_updatestats 'resample';

-- Ola Hallengren's statistics update
EXECUTE dbo.IndexOptimize
    @Databases = 'USER_DATABASES',
    @FragmentationLow = NULL,
    @FragmentationMedium = NULL,
    @FragmentationHigh = NULL,
    @UpdateStatistics = 'ALL',
    @OnlyModifiedStatistics = 'Y',
    @StatisticsSample = 100,    -- FULLSCAN
    @LogToTable = 'Y';
```

### Auto-Update Statistics Behavior

```sql
-- Check if auto-update statistics is enabled
SELECT
    name,
    is_auto_update_stats_on,
    is_auto_update_stats_async_on,
    is_auto_create_stats_on,
    is_auto_create_stats_incremental_on
FROM sys.databases
WHERE name = 'ApplicationDB';

-- Enable async statistics update (reduces query compilation stalls)
ALTER DATABASE [ApplicationDB] SET AUTO_UPDATE_STATISTICS_ASYNC ON;

-- Enable incremental statistics (partitioned tables, SQL 2014+)
ALTER DATABASE [ApplicationDB] SET AUTO_CREATE_STATISTICS ON
    (INCREMENTAL = ON);

-- Create incremental statistics on a partitioned table
CREATE STATISTICS [ST_Orders_OrderDate]
ON [app].[Orders] ([OrderDate])
WITH FULLSCAN, INCREMENTAL = ON;
```

## DBCC CHECKDB — Integrity Checks

```sql
-- Full consistency check (run weekly at minimum)
DBCC CHECKDB ('ApplicationDB')
WITH NO_INFOMSGS, ALL_ERRORMSGS, DATA_PURITY;

-- Physical-only check (faster, for daily runs)
DBCC CHECKDB ('ApplicationDB')
WITH PHYSICAL_ONLY, NO_INFOMSGS;

-- Extended logical checks (includes indexed views, XML indexes)
DBCC CHECKDB ('ApplicationDB')
WITH EXTENDED_LOGICAL_CHECKS, NO_INFOMSGS;

-- Check a specific table
DBCC CHECKTABLE ('app.Orders')
WITH NO_INFOMSGS, ALL_ERRORMSGS;

-- Check the allocation structures only (fastest)
DBCC CHECKALLOC ('ApplicationDB')
WITH NO_INFOMSGS;

-- Check catalog integrity
DBCC CHECKCATALOG ('ApplicationDB')
WITH NO_INFOMSGS;

-- Ola Hallengren's integrity check
EXECUTE dbo.DatabaseIntegrityCheck
    @Databases = 'USER_DATABASES',
    @CheckCommands = 'CHECKDB',
    @PhysicalOnly = 'N',
    @ExtendedLogicalChecks = 'N',
    @TimeLimit = 7200,      -- 2 hour limit
    @LogToTable = 'Y';

-- For VLDB: spread CHECKDB across the week using CHECKTABLE
-- Monday: tables A-F, Tuesday: tables G-L, etc.
```

### Handling Corruption

```sql
-- If CHECKDB finds corruption:
-- Step 1: ALWAYS try to restore from a clean backup first

-- Step 2: If no backup, attempt repair (LAST RESORT - may lose data)
-- Put database in single-user mode
ALTER DATABASE [ApplicationDB] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;

-- Attempt repair
DBCC CHECKDB ('ApplicationDB', REPAIR_ALLOW_DATA_LOSS);

-- Return to multi-user
ALTER DATABASE [ApplicationDB] SET MULTI_USER;

-- Step 3: Run CHECKDB again to verify repair
DBCC CHECKDB ('ApplicationDB')
WITH NO_INFOMSGS, ALL_ERRORMSGS;
```

## Database Shrink — Why to Avoid It

```sql
-- DBCC SHRINKDATABASE: shrinks all files in the database
-- DO NOT USE THIS ROUTINELY
DBCC SHRINKDATABASE ('ApplicationDB', 10);  -- Target 10% free space

-- DBCC SHRINKFILE: shrink a specific file
-- Only use for one-time space reclamation after a known event
DBCC SHRINKFILE (N'ApplicationDB', 10240);  -- Shrink to 10 GB

-- Why shrinking is harmful:
-- 1. Causes massive index fragmentation (pages are moved to fill gaps)
-- 2. Generates enormous I/O and log activity
-- 3. The database will grow back to its natural size, wasting all the effort
-- 4. Fragmentation after shrink degrades query performance

-- Disable auto-shrink (should ALWAYS be OFF)
ALTER DATABASE [ApplicationDB] SET AUTO_SHRINK OFF;

-- Check for databases with auto-shrink enabled
SELECT name, is_auto_shrink_on
FROM sys.databases
WHERE is_auto_shrink_on = 1;
```

## TempDB Configuration

```sql
-- Check current tempdb configuration
SELECT
    mf.name AS LogicalName,
    mf.physical_name,
    mf.type_desc,
    CAST(mf.size * 8.0 / 1024.0 AS DECIMAL(10,2)) AS SizeMB,
    CASE mf.is_percent_growth
        WHEN 1 THEN CAST(mf.growth AS VARCHAR) + '%'
        ELSE CAST(mf.growth * 8.0 / 1024.0 AS DECIMAL(10,2)) + ' MB'
    END AS AutoGrowth
FROM sys.master_files mf
WHERE mf.database_id = 2  -- tempdb
ORDER BY mf.type, mf.file_id;

-- Best practice: multiple equally-sized data files
-- General rule: 1 file per logical CPU core, up to 8 files
-- Add more only if PFS/GAM/SGAM contention is observed

-- Add tempdb data files (requires restart to take full effect)
ALTER DATABASE [tempdb]
ADD FILE (
    NAME = N'tempdev2',
    FILENAME = N'T:\TempDB\tempdev2.ndf',
    SIZE = 8192MB,
    FILEGROWTH = 1024MB
);

ALTER DATABASE [tempdb]
ADD FILE (
    NAME = N'tempdev3',
    FILENAME = N'T:\TempDB\tempdev3.ndf',
    SIZE = 8192MB,
    FILEGROWTH = 1024MB
);

ALTER DATABASE [tempdb]
ADD FILE (
    NAME = N'tempdev4',
    FILENAME = N'T:\TempDB\tempdev4.ndf',
    SIZE = 8192MB,
    FILEGROWTH = 1024MB
);

-- Ensure all tempdb files are the same size
-- SQL Server uses proportional fill; unequal files cause hotspots

-- Trace flag 1118 (pre-SQL 2016): use uniform extent allocation
-- In SQL 2016+, this is the default behavior and the flag is unnecessary
DBCC TRACEON (1118, -1);  -- Only needed for SQL 2014 and earlier

-- Monitor tempdb contention (PFS/GAM/SGAM waits)
SELECT
    session_id,
    wait_type,
    wait_duration_ms,
    resource_description
FROM sys.dm_os_waiting_tasks
WHERE wait_type LIKE 'PAGE%LATCH%'
  AND resource_description LIKE '2:%'  -- database_id 2 = tempdb
ORDER BY wait_duration_ms DESC;
```

## Instant File Initialization

```sql
-- Check if IFI is enabled
-- Method 1: Query (SQL 2016 SP1+)
SELECT
    servicename,
    instant_file_initialization_enabled
FROM sys.dm_server_services
WHERE servicename LIKE 'SQL Server%';
```

```powershell
# Method 2: Check via PowerShell (verify SE_MANAGE_VOLUME_NAME privilege)
$sqlAccount = (Get-WmiObject Win32_Service -Filter "Name='MSSQLSERVER'").StartName
# Then verify this account has "Perform Volume Maintenance Tasks" in Local Security Policy

# Grant IFI via PowerShell (requires restart)
$sid = (New-Object System.Security.Principal.NTAccount($sqlAccount)).Translate(
    [System.Security.Principal.SecurityIdentifier]).Value
# Add to SE_MANAGE_VOLUME_NAME privilege via secpol.msc or group policy
```

```sql
-- IFI only affects data files (.mdf, .ndf), NOT log files (.ldf)
-- Benefits: CREATE DATABASE, ALTER DATABASE (file growth), restore operations
-- complete much faster because pages are not zero-initialized

-- Example impact: growing a 100 GB data file
-- Without IFI: several minutes (must write zeros to every page)
-- With IFI: sub-second (OS allocates space without zeroing)
```

## SQL Server Agent Jobs

```sql
-- Create a maintenance job with multiple steps
USE [msdb];

-- Create the job
EXEC dbo.sp_add_job
    @job_name = N'DB Maintenance - ApplicationDB',
    @enabled = 1,
    @description = N'Weekly index rebuild, statistics update, and integrity check',
    @category_name = N'Database Maintenance',
    @owner_login_name = N'sa',
    @notify_level_email = 2;  -- Notify on failure

-- Step 1: Integrity Check
EXEC dbo.sp_add_jobstep
    @job_name = N'DB Maintenance - ApplicationDB',
    @step_name = N'CHECKDB',
    @step_id = 1,
    @subsystem = N'TSQL',
    @command = N'DBCC CHECKDB (''ApplicationDB'') WITH NO_INFOMSGS, ALL_ERRORMSGS, DATA_PURITY;',
    @database_name = N'ApplicationDB',
    @on_success_action = 3,   -- Go to next step
    @on_fail_action = 2,      -- Quit with failure
    @retry_attempts = 0;

-- Step 2: Index Maintenance
EXEC dbo.sp_add_jobstep
    @job_name = N'DB Maintenance - ApplicationDB',
    @step_name = N'Index Optimize',
    @step_id = 2,
    @subsystem = N'TSQL',
    @command = N'EXECUTE dbo.IndexOptimize
        @Databases = ''ApplicationDB'',
        @FragmentationLow = NULL,
        @FragmentationMedium = ''INDEX_REORGANIZE'',
        @FragmentationHigh = ''INDEX_REBUILD_ONLINE,INDEX_REBUILD_OFFLINE'',
        @FragmentationLevel1 = 5,
        @FragmentationLevel2 = 30,
        @UpdateStatistics = ''ALL'',
        @OnlyModifiedStatistics = ''Y'',
        @LogToTable = ''Y'';',
    @database_name = N'ApplicationDB',
    @on_success_action = 1,   -- Quit with success
    @on_fail_action = 2;      -- Quit with failure

-- Schedule: Every Sunday at 2:00 AM
EXEC dbo.sp_add_jobschedule
    @job_name = N'DB Maintenance - ApplicationDB',
    @name = N'Weekly Sunday 2AM',
    @freq_type = 8,            -- Weekly
    @freq_interval = 1,        -- Sunday
    @freq_recurrence_factor = 1,
    @active_start_time = 020000;  -- 2:00 AM

-- Assign to local server
EXEC dbo.sp_add_jobserver
    @job_name = N'DB Maintenance - ApplicationDB',
    @server_name = N'(LOCAL)';
```

## Maintenance Plans vs. Custom Scripts

| Aspect | Maintenance Plans | Custom Scripts / Ola Hallengren |
|--------|------------------|-------------------------------|
| Ease of setup | GUI-based, simple | Requires T-SQL knowledge |
| Flexibility | Limited options | Full control over every parameter |
| Index maintenance | Rebuild all or reorganize all (no mixed mode) | Smart: reorganize low frag, rebuild high frag |
| Logging | Basic SSIS logging | Detailed CommandLog table |
| Performance | Often rebuilds everything (wasteful) | Targets only fragmented indexes |
| Community support | Microsoft docs only | Extensive community, widely adopted |
| Time limits | No time-boxing | Built-in @TimeLimit parameter |
| Enterprise readiness | Adequate for small environments | Production-grade for any scale |

**Recommendation**: Use Ola Hallengren's Maintenance Solution for all production environments. Reserve maintenance plans for small, non-critical instances where simplicity is paramount.

## Best Practices

- Run DBCC CHECKDB weekly on every production database. For VLDBs, use PHYSICAL_ONLY daily and full CHECKDB weekly.
- Use Ola Hallengren's IndexOptimize with smart fragmentation thresholds rather than rebuilding all indexes blindly.
- Update statistics after index rebuilds are skipped (reorganize does not update statistics). Use @UpdateStatistics = 'ALL' with @OnlyModifiedStatistics = 'Y'.
- Enable instant file initialization for the SQL Server service account to avoid long waits during data file growth.
- Configure tempdb with multiple equally-sized data files (one per logical core, up to 8) on fast, dedicated storage.
- Disable auto-shrink on every database. It causes fragmentation, wastes I/O, and the files grow back.
- Set autogrowth to fixed MB values (not percentages) for both data and log files.
- Schedule maintenance jobs during low-activity windows with @TimeLimit to prevent them from running into business hours.
- Monitor job history and the Ola Hallengren CommandLog table for failures, increasing duration, or missed maintenance windows.
- Pre-size databases and files to their expected steady-state size. Rely on autogrowth only as a safety net.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Rebuilding all indexes regardless of fragmentation | Wasted I/O, unnecessary log generation, extended maintenance windows | Check fragmentation first; skip indexes below 5% and under 1000 pages |
| Not running DBCC CHECKDB regularly | Corruption spreads undetected; backups contain corrupted data | Schedule weekly CHECKDB; verify backup integrity with RESTORE VERIFYONLY |
| Leaving auto-shrink enabled | Constant shrink/grow cycles fragment data, waste I/O, degrade performance | ALTER DATABASE SET AUTO_SHRINK OFF on every database |
| Using 1 tempdb data file on multi-core server | PFS/GAM/SGAM latch contention causing waits | Add files (1 per core, up to 8); ensure equal sizing |
| Skipping statistics updates after reorganize | Stale statistics cause poor query plans; performance regression | Include UPDATE STATISTICS in the maintenance routine |
| Running maintenance during peak hours | User-facing queries contend for resources; timeouts | Schedule during low-activity windows; use @TimeLimit |
| Using maintenance plans for large environments | No smart fragmentation handling; rebuilds everything; no time limits | Switch to Ola Hallengren's solution or custom scripts |
| Not monitoring maintenance job failures | Failed CHECKDB or index jobs go unnoticed; issues accumulate | Configure Agent email alerts on job failure; review CommandLog regularly |

## SQL Server Version Notes

- **SQL Server 2014**: Incremental statistics for partitioned tables. Buffer pool extension (SSD as L2 cache). Improved cardinality estimator impacts statistics usage.
- **SQL Server 2016**: DBCC CHECKDB improvements for tempdb, sys.dm_db_log_info replaces DBCC LOGINFO, online non-clustered columnstore index rebuild, TF 1118 default behavior (uniform extent allocation), query store for regression detection after maintenance.
- **SQL Server 2017**: Resumable online index rebuilds. Automatic tuning (force last known good plan). Improved tempdb metadata contention handling.
- **SQL Server 2019**: Resumable online index creation (not just rebuild). Concurrent index operations improvements. Improved tempdb metadata optimization (in-memory optimized tempdb metadata). Accelerated Database Recovery changes log and undo management.
- **SQL Server 2022**: Parameter Sensitive Plan optimization reduces need for manual statistics/index intervention. Improved resumable operations. XML compression for XML indexes. Ordered clustered columnstore index (affects rebuild behavior). sys.dm_db_index_operational_stats improvements.

## Sources

- [Ola Hallengren's SQL Server Maintenance Solution](https://ola.hallengren.com/)
- [ALTER INDEX (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-index-transact-sql)
- [UPDATE STATISTICS (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/update-statistics-transact-sql)
- [DBCC CHECKDB (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql)
- [Reorganize and Rebuild Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes)
- [Instant File Initialization](https://learn.microsoft.com/en-us/sql/relational-databases/databases/database-instant-file-initialization)
- [TempDB Configuration](https://learn.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database)
- [SQL Server Agent](https://learn.microsoft.com/en-us/sql/ssms/agent/sql-server-agent)
