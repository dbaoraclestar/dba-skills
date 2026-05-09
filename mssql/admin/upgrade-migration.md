# SQL Server Upgrade and Migration — Planning, Execution, and Validation

## Overview

Upgrading SQL Server is a critical operation that requires careful planning, thorough testing, and validated rollback procedures. Whether upgrading from SQL Server 2016 to 2019, migrating to 2022, or moving to Azure SQL, the process involves assessing compatibility, testing application behavior, executing the upgrade, and validating the result. A poorly planned upgrade can lead to performance regressions, application failures, and extended downtime.

This skill covers supported upgrade paths, in-place vs. side-by-side migration strategies, database compatibility levels, Query Store as an upgrade safety net, the Data Migration Assistant (DMA), Distributed Replay for workload testing, Azure migration tools, post-upgrade validation, and deprecated features by version. Use this guide when planning any SQL Server version upgrade, whether on-premises or to cloud platforms.

The single most important upgrade best practice is to enable Query Store before the upgrade to capture baseline query performance, then use it after the upgrade to detect and fix regressions. This approach, combined with the database compatibility level as a staged rollout mechanism, makes SQL Server upgrades significantly safer than in previous generations.

## Key Concepts

- **In-Place Upgrade**: Running setup.exe on the existing server to upgrade the SQL Server binaries. Faster but no rollback path other than restoring from backup.
- **Side-by-Side Migration**: Installing a new SQL Server instance (on same or different hardware) and migrating databases to it. Provides a rollback path by keeping the old instance available.
- **Database Compatibility Level**: A per-database setting that controls which query optimizer behaviors and T-SQL features are available. Allows databases to run on a newer engine while maintaining older optimizer behavior.
- **Query Store**: A per-database feature that captures query plans, runtime statistics, and wait statistics. Essential for detecting performance regressions after upgrades or compatibility level changes.
- **Data Migration Assistant (DMA)**: A Microsoft tool that assesses SQL Server databases for compatibility issues, breaking changes, and feature parity before upgrading or migrating to Azure.
- **Distributed Replay**: A tool for replaying captured production workloads against a test server to validate upgrade safety under realistic conditions.
- **Cardinality Estimator (CE)**: The query optimizer component that estimates row counts. New CE versions ship with new compatibility levels and can cause plan changes.
- **Deprecated Features**: Features that are scheduled for removal in future versions. Using them generates warnings and should be addressed during upgrade planning.

## Supported Upgrade Paths

### Direct Upgrade Paths

| Source Version | Direct Upgrade Targets |
|---------------|----------------------|
| SQL Server 2012 | 2016, 2017, 2019, 2022 |
| SQL Server 2014 | 2016, 2017, 2019, 2022 |
| SQL Server 2016 | 2017, 2019, 2022 |
| SQL Server 2017 | 2019, 2022 |
| SQL Server 2019 | 2022 |

```sql
-- Check current version and edition
SELECT
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Edition') AS Edition,
    SERVERPROPERTY('ProductMajorVersion') AS MajorVersion,
    SERVERPROPERTY('EngineEdition') AS EngineEdition,
    @@VERSION AS FullVersion;

-- Check database compatibility levels
SELECT
    name,
    compatibility_level,
    CASE compatibility_level
        WHEN 100 THEN 'SQL Server 2008'
        WHEN 110 THEN 'SQL Server 2012'
        WHEN 120 THEN 'SQL Server 2014'
        WHEN 130 THEN 'SQL Server 2016'
        WHEN 140 THEN 'SQL Server 2017'
        WHEN 150 THEN 'SQL Server 2019'
        WHEN 160 THEN 'SQL Server 2022'
    END AS CompatDescription
FROM sys.databases
ORDER BY name;
```

### Unsupported Paths

SQL Server 2008/2008 R2 cannot directly upgrade to SQL Server 2019 or 2022. You must first upgrade to an intermediate version (2012 or 2014), then upgrade to the target version. Alternatively, use side-by-side migration with backup/restore or detach/attach.

## In-Place Upgrade

### Pre-Upgrade Checklist

```sql
-- 1. Document current configuration
SELECT * FROM sys.configurations ORDER BY name;

-- Save to file
EXEC sp_configure;

-- 2. Record linked servers
SELECT * FROM sys.servers WHERE is_linked = 1;

-- 3. Record SQL Agent jobs
SELECT
    j.name AS JobName,
    j.enabled,
    j.description,
    s.name AS ScheduleName,
    s.freq_type,
    s.active_start_time
FROM msdb.dbo.sysjobs j
LEFT JOIN msdb.dbo.sysjobschedules js ON j.job_id = js.job_id
LEFT JOIN msdb.dbo.sysschedules s ON js.schedule_id = s.schedule_id
ORDER BY j.name;

-- 4. Record database mail profiles
SELECT * FROM msdb.dbo.sysmail_profile;
SELECT * FROM msdb.dbo.sysmail_account;

-- 5. Script all logins with SIDs and passwords
SELECT
    'CREATE LOGIN [' + name + '] WITH PASSWORD = '
    + CONVERT(VARCHAR(MAX), LOGINPROPERTY(name, 'PasswordHash'), 1)
    + ' HASHED, SID = ' + CONVERT(VARCHAR(MAX), sid, 1)
    + ', DEFAULT_DATABASE = [' + default_database_name + ']'
    + ', CHECK_POLICY = ' + CASE is_policy_checked WHEN 1 THEN 'ON' ELSE 'OFF' END
    + ', CHECK_EXPIRATION = ' + CASE is_expiration_checked WHEN 1 THEN 'ON' ELSE 'OFF' END
    + ';'
FROM sys.sql_logins
WHERE name NOT LIKE '##%';

-- 6. Full backups of ALL databases including system databases
BACKUP DATABASE [master] TO DISK = N'E:\PreUpgrade\master.bak' WITH COMPRESSION, CHECKSUM;
BACKUP DATABASE [msdb] TO DISK = N'E:\PreUpgrade\msdb.bak' WITH COMPRESSION, CHECKSUM;
BACKUP DATABASE [model] TO DISK = N'E:\PreUpgrade\model.bak' WITH COMPRESSION, CHECKSUM;
-- Plus all user databases

-- 7. Enable Query Store on all user databases (critical for regression detection)
-- Run this BEFORE the upgrade to capture baseline performance
EXEC sp_MSforeachdb '
    IF ''?'' NOT IN (''master'', ''tempdb'', ''model'', ''msdb'')
    BEGIN
        ALTER DATABASE [?] SET QUERY_STORE = ON (
            OPERATION_MODE = READ_WRITE,
            DATA_FLUSH_INTERVAL_SECONDS = 900,
            INTERVAL_LENGTH_MINUTES = 30,
            MAX_STORAGE_SIZE_MB = 1024,
            QUERY_CAPTURE_MODE = ALL,
            SIZE_BASED_CLEANUP_MODE = AUTO
        );
    END';

-- 8. Check for deprecated features in use
SELECT
    instance_name AS Feature,
    cntr_value AS UsageCount
FROM sys.dm_os_performance_counters
WHERE object_name LIKE '%Deprecated Features%'
  AND cntr_value > 0
ORDER BY cntr_value DESC;
```

### In-Place Upgrade Execution

```powershell
# Run SQL Server setup with upgrade action
# From SQL Server installation media:
.\Setup.exe /ACTION=Upgrade `
    /INSTANCENAME=MSSQLSERVER `
    /IACCEPTSQLSERVERLICENSETERMS `
    /UpdateEnabled=True `
    /UpdateSource="C:\SQLUpdates"

# Silent upgrade with configuration file
.\Setup.exe /ConfigurationFile="C:\Upgrade\ConfigurationFile.ini"

# Check upgrade readiness
.\Setup.exe /ACTION=RunDiscovery
```

## Side-by-Side Migration

### Migration Steps

```sql
-- Step 1: Install new SQL Server instance on target (same or different server)
-- Step 2: Configure new instance to match source (memory, parallelism, etc.)

-- Step 3: Backup and restore databases
-- On source:
BACKUP DATABASE [ApplicationDB]
TO DISK = N'\\FileShare\Migration\ApplicationDB_Full.bak'
WITH COMPRESSION, CHECKSUM, COPY_ONLY;

-- On target:
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'\\FileShare\Migration\ApplicationDB_Full.bak'
WITH MOVE N'ApplicationDB' TO N'D:\Data\ApplicationDB.mdf',
     MOVE N'ApplicationDB_log' TO N'L:\Logs\ApplicationDB_log.ldf',
     RECOVERY, STATS = 10;

-- Step 4: Migrate logins (use sp_help_revlogin or scripted approach)
-- Step 5: Migrate Agent jobs (script from source, run on target)
-- Step 6: Migrate linked servers, database mail, etc.
-- Step 7: Update application connection strings
-- Step 8: Validate and decommission old instance after stabilization

-- Alternative: Use detach/attach for local migration
-- On source:
EXEC sp_detach_db @dbname = 'ApplicationDB', @skipchecks = 'true';
-- Move files to new instance paths
-- On target:
CREATE DATABASE [ApplicationDB] ON
    (FILENAME = N'D:\Data\ApplicationDB.mdf'),
    (FILENAME = N'L:\Logs\ApplicationDB_log.ldf')
FOR ATTACH;
```

### Using Log Shipping for Minimal Downtime Migration

```sql
-- Set up log shipping from source to target for near-zero downtime migration

-- On source: configure log shipping backup job
EXEC master.dbo.sp_add_log_shipping_primary_database
    @database = N'ApplicationDB',
    @backup_directory = N'\\FileShare\LogShip\',
    @backup_share = N'\\FileShare\LogShip\',
    @backup_job_name = N'LSBackup_ApplicationDB',
    @backup_retention_period = 1440,
    @backup_compression = 1;

-- On target: configure log shipping restore job
EXEC master.dbo.sp_add_log_shipping_secondary_primary
    @primary_server = N'SourceServer',
    @primary_database = N'ApplicationDB',
    @backup_source_directory = N'\\FileShare\LogShip\',
    @copy_job_name = N'LSCopy_ApplicationDB',
    @restore_job_name = N'LSRestore_ApplicationDB';

-- Cutover procedure:
-- 1. Stop application traffic to source
-- 2. Take final tail-log backup on source
-- 3. Restore final log on target WITH RECOVERY
-- 4. Point applications to new server
```

## Database Compatibility Level

```sql
-- Compatibility level is the KEY to safe upgrades
-- You can run databases on SQL 2022 engine with SQL 2016 optimizer behavior

-- Check current level
SELECT name, compatibility_level FROM sys.databases;

-- Keep old compatibility level after upgrade (safe start)
-- Databases retain their compatibility level through upgrades
ALTER DATABASE [ApplicationDB] SET COMPATIBILITY_LEVEL = 130;  -- SQL 2016 behavior

-- Staged upgrade: raise compatibility level after testing
ALTER DATABASE [ApplicationDB] SET COMPATIBILITY_LEVEL = 140;  -- SQL 2017
ALTER DATABASE [ApplicationDB] SET COMPATIBILITY_LEVEL = 150;  -- SQL 2019
ALTER DATABASE [ApplicationDB] SET COMPATIBILITY_LEVEL = 160;  -- SQL 2022

-- What changes at each compatibility level:
-- 130 (2016): New cardinality estimator v2, batch mode on rowstore (limited)
-- 140 (2017): Adaptive joins, interleaved execution, batch mode memory grants
-- 150 (2019): Batch mode on rowstore, table variable deferred compilation,
--             scalar UDF inlining, approximate query processing
-- 160 (2022): Parameter Sensitive Plan optimization, optimized plan forcing,
--             CE feedback, DOP feedback, cardinality estimation enhancements
```

### Compatibility Level + Query Store Strategy

```sql
-- The recommended upgrade approach:
-- Phase 1: Upgrade SQL Server engine, keep old compatibility level
-- Phase 2: Enable Query Store (if not already on)
-- Phase 3: Collect baseline in Query Store (1-2 weeks)
-- Phase 4: Raise compatibility level
-- Phase 5: Monitor Query Store for regressions
-- Phase 6: Force old plans for regressed queries

-- Identify regressed queries after changing compatibility level
-- SQL 2016+: Use Query Store Regressed Queries report in SSMS
-- Or query programmatically:
SELECT
    q.query_id,
    qt.query_sql_text,
    rs_old.avg_duration AS OldAvgDurationUs,
    rs_new.avg_duration AS NewAvgDurationUs,
    CAST(rs_new.avg_duration / NULLIF(rs_old.avg_duration, 0) AS DECIMAL(10,2))
        AS DurationRatio,
    p_old.plan_id AS OldPlanId,
    p_new.plan_id AS NewPlanId
FROM sys.query_store_query q
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p_old ON q.query_id = p_old.query_id
JOIN sys.query_store_plan p_new ON q.query_id = p_new.query_id
JOIN sys.query_store_runtime_stats rs_old ON p_old.plan_id = rs_old.plan_id
JOIN sys.query_store_runtime_stats rs_new ON p_new.plan_id = rs_new.plan_id
WHERE rs_new.avg_duration > rs_old.avg_duration * 2  -- 2x regression threshold
  AND p_old.plan_id <> p_new.plan_id
  AND rs_old.last_execution_time < '2024-01-15'    -- Before upgrade
  AND rs_new.first_execution_time > '2024-01-15'   -- After upgrade
ORDER BY DurationRatio DESC;

-- Force a known-good plan for a regressed query
EXEC sp_query_store_force_plan @query_id = 42, @plan_id = 17;

-- Unforce a plan
EXEC sp_query_store_unforce_plan @query_id = 42, @plan_id = 17;
```

## Data Migration Assistant (DMA)

```powershell
# DMA is a standalone Windows application
# Download from: https://www.microsoft.com/en-us/download/details.aspx?id=53595

# DMA Command Line Assessment
DmaCmd.exe /AssessmentName="PreUpgradeAssessment" `
    /AssessmentDatabases="Server=SQLPROD01;Initial Catalog=ApplicationDB;Integrated Security=true" `
    /AssessmentTargetPlatform="SqlServer2022" `
    /AssessmentEvaluateCompatibilityIssues `
    /AssessmentEvaluateFeatureParity `
    /AssessmentOverwriteResult `
    /AssessmentResultJson="C:\Assessment\Results.json"

# DMA reports:
# - Breaking changes (must fix before upgrade)
# - Behavior changes (may need attention)
# - Deprecated features (plan to remove usage)
# - Feature parity issues (for Azure SQL migration)
```

### Common DMA Findings

```sql
-- Breaking change: discontinuing features
-- Example: RAISERROR with format string (discontinued in 2016)
-- Old: RAISERROR 50001 'Error message'
-- New: RAISERROR('Error message', 16, 1);

-- Behavior change: implicit conversion changes
-- Example: String to datetime conversion behavior

-- Deprecated: non-ANSI joins
-- Old: SELECT * FROM A, B WHERE A.id *= B.id  (left outer join)
-- New: SELECT * FROM A LEFT JOIN B ON A.id = B.id

-- Find deprecated syntax in your codebase
SELECT
    object_name(object_id) AS ProcedureName,
    definition
FROM sys.sql_modules
WHERE definition LIKE '%*=%' OR definition LIKE '%=*%';
```

## Distributed Replay

```powershell
# Distributed Replay captures production workload and replays against test server
# Requires: Controller, Client(s), and Admin tools

# Step 1: Capture trace on production
# Use SQL Server Profiler or Extended Events to capture workload

# Step 2: Preprocess the trace
dreplay.exe preprocess `
    -m "ControllerServer" `
    -i "C:\Traces\ProductionWorkload.trc" `
    -d "C:\Traces\Preprocessed"

# Step 3: Replay against test server (upgraded version)
dreplay.exe replay `
    -m "ControllerServer" `
    -d "C:\Traces\Preprocessed" `
    -s "TestServer2022" `
    -w "C:\Traces\Results"

# Compare results between old and new versions
```

## Azure Migration

```powershell
# Azure Migrate: for discovering and assessing SQL Server estate
# Azure Database Migration Service (DMS): for online migration to Azure SQL

# Assessment using Azure Migrate CLI
az sql assessment create `
    --name "SQLAssessment" `
    --resource-group "MigrationRG" `
    --source-platform "SqlOnPrem" `
    --target-platform "AzureSqlDatabase"

# DMS for online migration (minimal downtime)
az dms project task create `
    --name "MigrateAppDB" `
    --project-name "SQLMigration" `
    --resource-group "MigrationRG" `
    --service-name "MigrationService" `
    --task-type "MigrateSqlServerSqlDb" `
    --source-connection-json @source.json `
    --target-connection-json @target.json `
    --database-options-json @dboptions.json
```

```sql
-- Azure SQL Database: compatibility level mapping
-- Azure SQL DB always runs the latest engine
-- Use compatibility level to control optimizer behavior
ALTER DATABASE [ApplicationDB] SET COMPATIBILITY_LEVEL = 150;  -- SQL 2019 behavior on Azure

-- Azure SQL Managed Instance: most compatible with on-premises
-- Supports: Agent jobs, cross-database queries, linked servers, CLR, Service Broker
-- Missing: FILESTREAM, buffer pool extension, some trace flags
```

## Post-Upgrade Validation

```sql
-- 1. Verify SQL Server version and patch level
SELECT @@VERSION;
SELECT SERVERPROPERTY('ProductVersion'), SERVERPROPERTY('ProductLevel');

-- 2. Check all databases are online
SELECT name, state_desc, recovery_model_desc, compatibility_level
FROM sys.databases;

-- 3. Run DBCC CHECKDB on all databases
EXEC sp_MSforeachdb '
    IF ''?'' NOT IN (''tempdb'')
    BEGIN
        PRINT ''Checking: ?'';
        DBCC CHECKDB (''?'') WITH NO_INFOMSGS, ALL_ERRORMSGS;
    END';

-- 4. Verify Query Store is capturing data
SELECT
    name,
    is_query_store_on,
    actual_state_desc,
    current_storage_size_mb,
    max_storage_size_mb
FROM sys.databases d
CROSS APPLY sys.dm_db_query_store_internal_state(d.database_id) qs
WHERE d.database_id > 4;

-- Alternative for compatibility:
SELECT
    name,
    is_query_store_on
FROM sys.databases
WHERE database_id > 4;

-- 5. Check for performance regressions via Query Store
-- (See Compatibility Level + Query Store section above)

-- 6. Verify Agent jobs are running
SELECT
    j.name,
    j.enabled,
    jh.run_status,
    jh.run_date,
    jh.run_time,
    jh.message
FROM msdb.dbo.sysjobs j
JOIN msdb.dbo.sysjobhistory jh ON j.job_id = jh.job_id
WHERE jh.step_id = 0  -- Job outcome row
ORDER BY jh.run_date DESC, jh.run_time DESC;

-- 7. Verify linked servers
SELECT name, data_source, provider FROM sys.servers WHERE is_linked = 1;
EXEC sp_testlinkedserver 'LinkedServerName';

-- 8. Check error log for post-upgrade warnings
EXEC xp_readerrorlog 0, 1, N'error';
EXEC xp_readerrorlog 0, 1, N'warning';
EXEC xp_readerrorlog 0, 1, N'deprecated';

-- 9. Validate database mail
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'Default',
    @recipients = 'dba@company.com',
    @subject = 'Post-Upgrade Validation - SQL Server Upgrade Complete',
    @body = 'SQL Server upgrade completed successfully. All databases online.';

-- 10. Update compatibility level when ready (after monitoring period)
-- Wait at least 1-2 weeks with Query Store monitoring before raising
```

## Deprecated Features by Version

### Deprecated in SQL Server 2016

```sql
-- Features deprecated (will be removed in future versions):
-- - Database mirroring (replaced by AG)
-- - Non-ANSI outer joins (*=, =*)
-- - DATABASEPROPERTY (use DATABASEPROPERTYEX)
-- - sp_addtype (use CREATE TYPE)
-- - SET ROWCOUNT for INSERT/UPDATE/DELETE
-- - WRITETEXT/UPDATETEXT/READTEXT (use MAX data types)

-- Check usage of deprecated features
SELECT
    instance_name AS Feature,
    cntr_value AS UsageCount
FROM sys.dm_os_performance_counters
WHERE object_name LIKE '%Deprecated Features%'
  AND cntr_value > 0
ORDER BY cntr_value DESC;
```

### Deprecated in SQL Server 2019

```sql
-- Features deprecated:
-- - Database compatibility level 100 (SQL 2008)
-- - DBCC SHOWCONTIG (use sys.dm_db_index_physical_stats)
-- - sp_dbcmptlevel (use ALTER DATABASE SET COMPATIBILITY_LEVEL)
-- - STRING_ESCAPE with type 'html' (only 'json' supported)
-- - DISABLE_DEFERRED_CONSTRAINT_CHECKS
-- - sp_addserver for local server (use sp_serveroption)
```

### Deprecated in SQL Server 2022

```sql
-- Features deprecated:
-- - Database compatibility level 110 (SQL 2012)
-- - Machine Learning Services with R/Python (moving to extensibility framework)
-- - Stretch Database
-- - Replication to Azure SQL Database (use Azure-native replication)
-- - SQL Server Reporting Services (SSRS) will continue but with reduced investment

-- Discontinued features (removed entirely):
-- SQL Server 2019 discontinued:
-- - DBCC DBREINDEX (use ALTER INDEX REBUILD)
-- - DBCC INDEXDEFRAG (use ALTER INDEX REORGANIZE)
-- - sp_indexoption (use ALTER INDEX)
-- - FASTFIRSTROW hint (use OPTION (FAST n))
```

## Upgrade Rollback Strategy

```sql
-- In-place upgrade rollback options:
-- 1. Restore from pre-upgrade full backups (system + user databases)
-- 2. Uninstall new version, reinstall old version, restore databases
-- 3. VM snapshot rollback (if running on a VM)

-- Side-by-side migration rollback:
-- Simply point applications back to the old server (still running)

-- Rollback checklist:
-- - Full backups of ALL databases before upgrade
-- - Scripted logins, Agent jobs, linked servers, and server configuration
-- - Document connection strings and DNS entries
-- - VM snapshot (if applicable)
-- - Tested rollback procedure in a non-production environment
```

## PowerShell Upgrade Automation

```powershell
# Pre-upgrade assessment script
$sourceInstance = "SQLPROD01"
$targetVersion = "SQL Server 2022"
$outputPath = "C:\UpgradeAssessment"

# Collect server configuration
$config = Invoke-Sqlcmd -ServerInstance $sourceInstance -Query "
    SELECT name, value_in_use FROM sys.configurations ORDER BY name
" | Export-Csv "$outputPath\server_config.csv" -NoTypeInformation

# Collect database info
$databases = Invoke-Sqlcmd -ServerInstance $sourceInstance -Query "
    SELECT name, compatibility_level, recovery_model_desc,
           state_desc, is_query_store_on
    FROM sys.databases
" | Export-Csv "$outputPath\databases.csv" -NoTypeInformation

# Enable Query Store on all user databases
$userDbs = Invoke-Sqlcmd -ServerInstance $sourceInstance -Query "
    SELECT name FROM sys.databases
    WHERE database_id > 4 AND state = 0 AND is_query_store_on = 0
"
foreach ($db in $userDbs) {
    $sql = "ALTER DATABASE [$($db.name)] SET QUERY_STORE = ON (
        OPERATION_MODE = READ_WRITE,
        MAX_STORAGE_SIZE_MB = 1024
    );"
    Invoke-Sqlcmd -ServerInstance $sourceInstance -Query $sql
    Write-Host "Enabled Query Store on: $($db.name)"
}

# Script all logins
$logins = Invoke-Sqlcmd -ServerInstance $sourceInstance -Query "
    SELECT name FROM sys.server_principals
    WHERE type IN ('S', 'U', 'G')
    AND name NOT LIKE '##%' AND name NOT IN ('sa', 'public')
"
Write-Host "Found $($logins.Count) logins to migrate"
```

## Best Practices

- Always enable Query Store before upgrading and let it capture baseline performance for at least 1-2 weeks before changing the compatibility level.
- Use side-by-side migration for critical production systems. It provides a clean rollback path and allows validation before cutover.
- Keep the old compatibility level after upgrading the engine. Raise it only after validating performance with Query Store data.
- Run DMA assessment before any upgrade to identify breaking changes, deprecated features, and compatibility issues.
- Take verified full backups of all databases (including system databases) immediately before the upgrade. Test the restore process.
- Test the upgrade in a non-production environment with realistic workloads before touching production. Use Distributed Replay or workload-generating tools.
- Document everything: current configuration, logins, Agent jobs, linked servers, connection strings, and DNS entries. Script everything for both migration and rollback.
- Plan for rollback explicitly. Know how to revert to the previous version and test the rollback procedure.
- Upgrade during a maintenance window with adequate time for validation and potential rollback. Communicate the plan to stakeholders.
- After upgrading, monitor the SQL error log, Query Store, and application metrics closely for at least 2 weeks before considering the upgrade complete.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Upgrading without enabling Query Store first | No baseline for detecting performance regressions | Enable Query Store 2+ weeks before upgrade; capture baseline |
| Immediately raising compatibility level after upgrade | Sudden cardinality estimator changes cause widespread plan regressions | Keep old compat level; raise gradually after validation |
| Not testing the upgrade in a non-production environment | Unexpected failures, performance issues, and extended downtime | Build a test environment mirroring production; replay workloads |
| Forgetting to migrate logins with matching SIDs | Orphaned users; applications cannot connect | Script logins with SIDs from source and create on target before migration |
| In-place upgrade without verified backups | No rollback path if upgrade fails or causes corruption | Take and verify full backups; test restore before starting upgrade |
| Ignoring deprecated feature warnings | Code breaks when features are removed in the target version | Run DMA assessment; fix deprecated usage before upgrading |
| Skipping post-upgrade DBCC CHECKDB | Corruption introduced during upgrade goes undetected | Run CHECKDB on all databases immediately after upgrade |
| Upgrading the OS and SQL Server simultaneously | Cannot isolate the cause of issues | Upgrade one at a time; stabilize between each change |

## SQL Server Version Notes

- **SQL Server 2016**: Query Store introduced (critical for upgrade safety). Temporal tables, Row-Level Security, Dynamic Data Masking, Always Encrypted, Stretch Database, JSON support, PolyBase, R Services.
- **SQL Server 2017**: First version on Linux. Adaptive query processing (adaptive joins, interleaved execution), graph database, automatic plan correction, Python in ML Services, resumable online index rebuild.
- **SQL Server 2019**: Intelligent Query Processing (batch mode on rowstore, table variable deferred compilation, scalar UDF inlining, approximate query processing), Big Data Clusters (deprecated in 2022), Accelerated Database Recovery, UTF-8 support, Java extensibility, secure enclaves.
- **SQL Server 2022**: Ledger tables for tamper-evident auditing, Parameter Sensitive Plan optimization, Query Store hints, DOP and CE feedback, contained availability groups, link feature for Azure SQL Managed Instance, S3-compatible object storage backup, Azure AD/Entra ID authentication, Intel QAT hardware compression, Synapse Link integration.

## Sources

- [Upgrade SQL Server](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/upgrade-sql-server)
- [Supported Version and Edition Upgrades](https://learn.microsoft.com/en-us/sql/database-engine/install-windows/supported-version-and-edition-upgrades-2022)
- [Database Compatibility Level](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-database-transact-sql-compatibility-level)
- [Query Store Best Practices for Upgrading](https://learn.microsoft.com/en-us/sql/relational-databases/performance/best-practice-with-the-query-store)
- [Data Migration Assistant](https://learn.microsoft.com/en-us/sql/dma/dma-overview)
- [Distributed Replay](https://learn.microsoft.com/en-us/sql/tools/distributed-replay/distributed-replay)
- [Azure SQL Migration](https://learn.microsoft.com/en-us/azure/azure-sql/migration-guides/database/sql-server-to-sql-database-overview)
- [Deprecated Database Engine Features](https://learn.microsoft.com/en-us/sql/database-engine/deprecated-database-engine-features-in-sql-server-2022)
- [What's New in SQL Server 2022](https://learn.microsoft.com/en-us/sql/sql-server/what-s-new-in-sql-server-2022)
