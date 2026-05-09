# dbatools -- PowerShell Module for SQL Server Administration

## Overview

dbatools is an open-source PowerShell module with 500+ commands for SQL Server administration, covering backups, migrations, diagnostics, security, best practices validation, and instance management. It wraps the SQL Server Management Objects (SMO) library and provides a consistent, pipeline-friendly interface that turns complex multi-step administrative tasks into one-liners.

dbatools is the right tool when you need to automate SQL Server operations across multiple instances, perform migrations between servers, validate backups, collect diagnostics, or enforce configuration standards. It replaces hundreds of manual SSMS clicks with scriptable, repeatable commands that work across SQL Server 2008 through 2022 and Azure SQL.

Use this skill when automating SQL Server administration with PowerShell, performing instance migrations, validating backups, collecting performance diagnostics, or applying best practice configurations.

## Key Concepts

- **Pipeline-friendly**: Most commands accept pipeline input and produce objects that can be piped to other commands. `Get-DbaDatabase | Backup-DbaDatabase` backs up every database in one pipeline.
- **Multi-instance**: Most commands accept `-SqlInstance` as an array, operating across many servers in parallel.
- **SMO-based**: Under the hood, dbatools uses SQL Server Management Objects, giving it access to the full SQL Server API.
- **Credential handling**: Supports SQL authentication via `-SqlCredential`, Windows authentication (default), and Azure AD tokens.
- **Non-destructive by default**: Dangerous operations require `-Force` or `-Confirm:$false`. Many commands support `-WhatIf` to preview actions.

## Installation and Setup

```powershell
# Install from PowerShell Gallery
Install-Module dbatools -Scope CurrentUser

# Update to latest version
Update-Module dbatools

# Import module (auto-imported on first use in PS 5.1+)
Import-Module dbatools

# Verify installation
Get-Module dbatools -ListAvailable

# Set default SQL instances for the session
$servers = "PROD-SQL-01", "PROD-SQL-02", "STG-SQL-01"
```

## Connection Testing

```powershell
# Test connectivity to SQL Server instances
Test-DbaConnection -SqlInstance "PROD-SQL-01", "PROD-SQL-02", "STG-SQL-01"

# Output shows: ComputerName, InstanceName, SqlConnected, WmiConnected,
#               PingSucceeded, AuthType, ConnectingAsUser

# Connect with SQL authentication
$cred = Get-Credential -UserName "sa"
Test-DbaConnection -SqlInstance "PROD-SQL-01" -SqlCredential $cred

# Find SQL Server instances on the network
Find-DbaInstance -ComputerName "10.0.1.0/24" | Format-Table
```

## Database Operations

```powershell
# List all databases on an instance
Get-DbaDatabase -SqlInstance "PROD-SQL-01" |
    Select-Object Name, Status, RecoveryModel, SizeMB, LastFullBackup

# Get database details for specific databases
Get-DbaDatabase -SqlInstance "PROD-SQL-01" -Database "MyApp", "Reporting" |
    Select-Object Name, Collation, CompatibilityLevel, PageVerify, Owner

# Get database size information
Get-DbaDbSpace -SqlInstance "PROD-SQL-01" -Database "MyApp" |
    Format-Table Database, FileName, FileType, UsedMB, AvailableMB, PercentUsed

# Check database integrity (DBCC CHECKDB)
Invoke-DbaDbDbccCheckDb -SqlInstance "PROD-SQL-01" -Database "MyApp"

# Copy a database to another instance (backup/restore method)
Copy-DbaDatabase -Source "PROD-SQL-01" -Destination "STG-SQL-01" `
    -Database "MyApp" -BackupRestore -SharedPath "\\fileserver\backups"

# Set database options
Set-DbaDbRecoveryModel -SqlInstance "PROD-SQL-01" -Database "MyApp" -RecoveryModel Full -Confirm:$false
```

## Backup and Restore

```powershell
# Full backup
Backup-DbaDatabase -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Path "\\backupserver\sqlbackups" -CompressBackup -Verify -CopyOnly

# Backup all databases
Backup-DbaDatabase -SqlInstance "PROD-SQL-01" -Path "\\backupserver\sqlbackups" `
    -CompressBackup -Type Full

# Differential backup
Backup-DbaDatabase -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Path "\\backupserver\sqlbackups" -Type Differential

# Transaction log backup
Backup-DbaDatabase -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Path "\\backupserver\sqlbackups" -Type Log

# Restore database
Restore-DbaDatabase -SqlInstance "STG-SQL-01" -Path "\\backupserver\sqlbackups\MyApp" `
    -DatabaseName "MyApp_Restored" -WithReplace

# Point-in-time restore
Restore-DbaDatabase -SqlInstance "STG-SQL-01" `
    -Path "\\backupserver\sqlbackups\MyApp" `
    -DatabaseName "MyApp_PITR" `
    -RestoreTime (Get-Date "2025-03-15 14:30:00") `
    -WithReplace

# Test last backup (restore to temp, verify, drop)
Test-DbaLastBackup -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Destination "STG-SQL-01" -MaxMB 10240

# Get backup history
Get-DbaDbBackupHistory -SqlInstance "PROD-SQL-01" -Database "MyApp" -Last |
    Select-Object Database, Type, Start, End, TotalSize, CompressedBackupSize
```

## Instance Migration

```powershell
# Full instance migration (databases, logins, Agent jobs, linked servers, etc.)
Start-DbaMigration -Source "OLD-SQL-01" -Destination "NEW-SQL-01" `
    -BackupRestore -SharedPath "\\fileserver\migration" `
    -Exclude "SpConfigure"  # Exclude specific migration steps

# Migrate specific components
Copy-DbaLogin -Source "OLD-SQL-01" -Destination "NEW-SQL-01"
Copy-DbaAgentJob -Source "OLD-SQL-01" -Destination "NEW-SQL-01"
Copy-DbaLinkedServer -Source "OLD-SQL-01" -Destination "NEW-SQL-01"
Copy-DbaDbMail -Source "OLD-SQL-01" -Destination "NEW-SQL-01"

# Compare configurations between instances
Compare-DbaDbModule -Source "OLD-SQL-01" -Destination "NEW-SQL-01" |
    Where-Object Status -eq "Different"

# Export logins to T-SQL script (for manual migration)
Export-DbaLogin -SqlInstance "PROD-SQL-01" -Path "C:\migration\logins.sql"
```

## Performance Diagnostics

```powershell
# Top wait statistics
Get-DbaWaitStatistic -SqlInstance "PROD-SQL-01" -Threshold 95 |
    Select-Object WaitType, WaitSeconds, Percentage, AvgWaitSec |
    Format-Table -AutoSize

# Top resource-consuming queries
Get-DbaTopResourceUsage -SqlInstance "PROD-SQL-01" -Type Duration -Database "MyApp" |
    Select-Object Database, ObjectName, TotalElapsedTime, ExecutionCount,
                  AvgElapsedTime, QueryText -First 10

# Currently running queries
Get-DbaRunningQuery -SqlInstance "PROD-SQL-01" |
    Where-Object { $_.Status -ne "sleeping" } |
    Select-Object Spid, Database, Login, Status, CpuTime, Command, SqlText

# Blocking sessions
Get-DbaProcess -SqlInstance "PROD-SQL-01" |
    Where-Object { $_.BlockedBy -ne 0 } |
    Select-Object Spid, BlockedBy, Database, Command, LastBatch

# Index usage analysis
Get-DbaHelpIndex -SqlInstance "PROD-SQL-01" -Database "MyApp" |
    Where-Object { $_.UserSeeks -eq 0 -and $_.UserScans -eq 0 } |
    Select-Object TableName, IndexName, IndexType, SizeMB

# Missing index suggestions
Find-DbaDbUnusedIndex -SqlInstance "PROD-SQL-01" -Database "MyApp"
Get-DbaDbMissingIndex -SqlInstance "PROD-SQL-01" -Database "MyApp" |
    Sort-Object ImprovementMeasure -Descending |
    Select-Object Database, TableName, EqualityColumns, InequalityColumns, IncludeColumns -First 10
```

## Query Execution

```powershell
# Execute a query
Invoke-DbaQuery -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Query "SELECT TOP 10 * FROM dbo.Customers ORDER BY CreatedAt DESC"

# Execute a script file
Invoke-DbaQuery -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -File "C:\scripts\maintenance.sql"

# Execute across multiple instances
Invoke-DbaQuery -SqlInstance "PROD-SQL-01", "PROD-SQL-02" `
    -Query "SELECT @@SERVERNAME AS ServerName, DB_NAME() AS DatabaseName, COUNT(*) AS TableCount FROM sys.tables" `
    -Database "MyApp"

# Execute with parameters
Invoke-DbaQuery -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Query "SELECT * FROM dbo.Customers WHERE CustomerID = @ID" `
    -SqlParameters @{ ID = 42 }

# Export results to CSV
Invoke-DbaQuery -SqlInstance "PROD-SQL-01" -Database "MyApp" `
    -Query "SELECT * FROM dbo.Customers" |
    Export-Csv -Path "C:\exports\customers.csv" -NoTypeInformation
```

## Memory and Configuration

```powershell
# Check current max memory setting
Get-DbaMaxMemory -SqlInstance "PROD-SQL-01"

# Set max memory based on OS memory (recommended)
Set-DbaMaxMemory -SqlInstance "PROD-SQL-01" -Max 32768  # 32 GB

# Get recommended max memory
Test-DbaMaxMemory -SqlInstance "PROD-SQL-01"

# View all sp_configure settings
Get-DbaSpConfigure -SqlInstance "PROD-SQL-01" |
    Where-Object { $_.RunningValue -ne $_.DefaultValue } |
    Select-Object Name, RunningValue, DefaultValue, Description

# Set sp_configure options
Set-DbaSpConfigure -SqlInstance "PROD-SQL-01" -Name "cost threshold for parallelism" -Value 50
Set-DbaSpConfigure -SqlInstance "PROD-SQL-01" -Name "max degree of parallelism" -Value 4
```

## Ola Hallengren Maintenance Solution

```powershell
# Install Ola Hallengren's maintenance solution (industry standard)
Install-DbaMaintenanceSolution -SqlInstance "PROD-SQL-01" `
    -Database "master" `
    -BackupLocation "\\backupserver\sqlbackups" `
    -CleanupTime 168  # 7 days retention
    -InstallJobs      # Create SQL Agent jobs

# The installation creates these Agent jobs:
# - DatabaseBackup - SYSTEM_DATABASES - FULL
# - DatabaseBackup - USER_DATABASES - FULL
# - DatabaseBackup - USER_DATABASES - DIFF
# - DatabaseBackup - USER_DATABASES - LOG
# - DatabaseIntegrityCheck - SYSTEM_DATABASES
# - DatabaseIntegrityCheck - USER_DATABASES
# - IndexOptimize - USER_DATABASES
# - CommandLog Cleanup
# - Output File Cleanup
```

## Best Practices Validation

```powershell
# Run comprehensive best practices check
$results = Invoke-DbaDiagnosticQuery -SqlInstance "PROD-SQL-01"

# Check specific best practice areas
Test-DbaDbCompression -SqlInstance "PROD-SQL-01" -Database "MyApp"
Test-DbaOptimizeForAdHoc -SqlInstance "PROD-SQL-01"
Test-DbaDbOwner -SqlInstance "PROD-SQL-01"
Test-DbaDbRecoveryModel -SqlInstance "PROD-SQL-01"

# Power Plan check (should be High Performance)
Test-DbaPowerPlan -ComputerName "PROD-SQL-01"

# Disk allocation unit size (should be 64KB for SQL data files)
Test-DbaDbVirtualLogFile -SqlInstance "PROD-SQL-01"
```

## Multi-Instance Reporting

```powershell
# Daily health check across all instances
$servers = "PROD-SQL-01", "PROD-SQL-02", "STG-SQL-01", "DEV-SQL-01"

# Disk space check
Get-DbaDbSpace -SqlInstance $servers |
    Where-Object { $_.PercentUsed -gt 80 } |
    Select-Object SqlInstance, Database, FileName, UsedMB, AvailableMB, PercentUsed |
    Sort-Object PercentUsed -Descending

# Backup freshness check
Get-DbaDbBackupHistory -SqlInstance $servers -Last |
    Where-Object { $_.End -lt (Get-Date).AddHours(-24) -and $_.Type -eq "Full" } |
    Select-Object SqlInstance, Database, Type, End |
    Sort-Object End

# Agent job failures (last 24 hours)
Get-DbaAgentJobHistory -SqlInstance $servers -StartDate (Get-Date).AddHours(-24) |
    Where-Object { $_.Status -eq "Failed" } |
    Select-Object SqlInstance, Job, StepName, RunDate, Message
```

## Best Practices

- Use `Test-DbaConnection` before batch operations to verify all instances are reachable
- Always use `-WhatIf` before destructive operations like `Remove-DbaDatabase` or `Start-DbaMigration`
- Pipe results to `Export-Csv` or `ConvertTo-Json` for reporting and documentation
- Use `-SqlCredential` with `Get-Credential` instead of hardcoding passwords
- Run `Test-DbaLastBackup` regularly to verify backup integrity (not just backup completion)
- Install Ola Hallengren's maintenance solution via `Install-DbaMaintenanceSolution` as a baseline
- Use `Compare-DbaInstance` when validating migration completeness
- Store common instance lists in variables or CSV files for consistent multi-server operations
- Schedule daily health checks using PowerShell scheduled tasks or SQL Server Agent

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running `Start-DbaMigration` without testing connectivity first | Migration fails partway through, leaving incomplete state | Run `Test-DbaConnection` on both source and destination first |
| Using `-Force` on `Copy-DbaDatabase` without checking for existing database | Overwrites target database with same name without warning | Use `Get-DbaDatabase` to check target first; use `-WithReplace` explicitly |
| Not using `-CompressBackup` for large databases | Backup files 3-5x larger, slower transfer, more storage cost | Always use `-CompressBackup` (or enable server-level backup compression default) |
| Forgetting `-Confirm:$false` in automated scripts | Script hangs waiting for interactive confirmation | Add `-Confirm:$false` for unattended execution (after thorough testing) |
| Running resource-intensive diagnostics during peak hours | Performance impact on production workload | Schedule `Invoke-DbaDiagnosticQuery` and `Get-DbaTopResourceUsage` during off-hours |

## SQL Server Version Notes

- **SQL Server 2016**: Full dbatools support. Query Store commands available (`Get-DbaDbQueryStoreOption`, `Set-DbaDbQueryStoreOption`). Always Encrypted management via `Get-DbaDbEncryptionKey`. Stretch Database support.
- **SQL Server 2019**: dbatools supports Accelerated Database Recovery diagnostics. Big Data Cluster commands available (limited). `Get-DbaDbFeatureUsage` identifies features in use for upgrade planning. Extended Events improvements accessible via `Get-DbaXESession`.
- **SQL Server 2022**: dbatools supports Ledger table verification, contained availability group management, and Query Store hints. `Test-DbaDbCompatibility` helps identify databases needing compatibility level updates. Backup to S3-compatible storage supported via `Backup-DbaDatabase -AzureBaseUrl`.

## Sources

- https://dbatools.io
- https://docs.dbatools.io
- https://github.com/dataplat/dbatools
- https://dbatools.io/commands/
- https://ola.hallengren.com (Maintenance Solution)
- https://learn.microsoft.com/en-us/powershell/module/sqlserver/
