# SQL Server Backup and Recovery — Protecting Data with Full, Differential, and Log Backups

## Overview

Backup and recovery is the single most critical responsibility of a SQL Server DBA. A sound backup strategy protects against hardware failure, data corruption, accidental data deletion, ransomware, and disaster scenarios. SQL Server provides a flexible backup architecture supporting full, differential, and transaction log backups that can be combined to achieve any desired Recovery Point Objective (RPO) and Recovery Time Objective (RTO).

This skill covers backup types and their mechanics, recovery models, BACKUP and RESTORE T-SQL syntax, backup compression, backup to Azure Blob Storage, point-in-time recovery, tail-log backups, restore sequences, backup verification, and maintenance plan configuration. Use this guide whenever you need to design a backup strategy, restore a database, troubleshoot backup failures, or validate your recovery capability.

Understanding the interplay between recovery models and backup types is fundamental. The FULL recovery model enables point-in-time recovery via transaction log backups but requires active log management. The SIMPLE model auto-truncates the log at checkpoints, sacrificing granular recovery for lower maintenance overhead.

## Key Concepts

- **Full Backup**: A complete copy of the database including all data and enough transaction log to restore to a consistent state. Serves as the base for differential and log restore chains.
- **Differential Backup**: Contains all data pages modified since the last full backup. Each differential is cumulative (not incremental), so you only need the last differential in a restore sequence.
- **Transaction Log Backup**: Backs up the active portion of the transaction log and truncates it. Required for point-in-time recovery. Only available in FULL or BULK_LOGGED recovery models.
- **Tail-Log Backup**: A special log backup taken just before a restore operation to capture any log records not yet backed up. Prevents data loss during disaster recovery.
- **Recovery Model**: Controls how the transaction log is managed. FULL (log backups required, PITR supported), SIMPLE (auto-truncate, no PITR), BULK_LOGGED (minimal logging for bulk ops, limited PITR).
- **Backup Set**: The output of a single BACKUP operation. Multiple backup sets can exist in a single media file.
- **Media Set / Backup Device**: The physical destination (disk file, tape, URL) where backup sets are written.
- **LSN (Log Sequence Number)**: Internal identifier that chains backups together in the correct restore sequence.
- **Copy-Only Backup**: A backup that does not affect the differential base or log chain. Used for ad-hoc copies without disrupting scheduled backups.

## Full Backups

```sql
-- Basic full backup to disk
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Full_20240115.bak'
WITH COMPRESSION,
     CHECKSUM,
     STATS = 10,
     NAME = N'ApplicationDB-Full Database Backup',
     DESCRIPTION = N'Nightly full backup';

-- Full backup with multiple stripe files (parallel I/O)
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Full_1.bak',
   DISK = N'F:\Backups\ApplicationDB_Full_2.bak',
   DISK = N'G:\Backups\ApplicationDB_Full_3.bak',
   DISK = N'H:\Backups\ApplicationDB_Full_4.bak'
WITH COMPRESSION,
     CHECKSUM,
     STATS = 5;

-- Copy-only backup (does not affect backup chain)
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_CopyOnly.bak'
WITH COPY_ONLY,
     COMPRESSION,
     CHECKSUM;

-- Backup with expiration and retention
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH COMPRESSION,
     CHECKSUM,
     RETAINDAYS = 30,
     INIT;  -- Overwrite existing backup set in file
```

## Differential Backups

```sql
-- Differential backup (captures changes since last full)
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Diff_20240115_1200.bak'
WITH DIFFERENTIAL,
     COMPRESSION,
     CHECKSUM,
     STATS = 10;

-- Monitor differential backup size growth
SELECT
    database_name,
    backup_start_date,
    backup_finish_date,
    CAST(backup_size / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS BackupSizeMB,
    CAST(compressed_backup_size / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS CompressedMB,
    type AS BackupType  -- D=Full, I=Differential, L=Log
FROM msdb.dbo.backupset
WHERE database_name = 'ApplicationDB'
  AND type = 'I'
ORDER BY backup_start_date DESC;
```

## Transaction Log Backups

```sql
-- Transaction log backup
BACKUP LOG [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Log_20240115_1200.trn'
WITH COMPRESSION,
     CHECKSUM,
     STATS = 10;

-- Tail-log backup before restore (captures remaining log)
BACKUP LOG [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_TailLog.trn'
WITH NORECOVERY,  -- Leaves database in restoring state
     COMPRESSION,
     CHECKSUM;

-- Tail-log when database is damaged (force backup of available log)
BACKUP LOG [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_TailLog.trn'
WITH CONTINUE_AFTER_ERROR,
     NORECOVERY;
```

## Backup to Azure Blob Storage

```sql
-- Step 1: Create a credential using SAS token
CREATE CREDENTIAL [https://mystorageaccount.blob.core.windows.net/sqlbackups]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = 'sv=2022-11-02&ss=b&srt=o&sp=rwdlacitfx&se=2025-12-31...';

-- Step 2: Backup to URL
BACKUP DATABASE [ApplicationDB]
TO URL = N'https://mystorageaccount.blob.core.windows.net/sqlbackups/ApplicationDB_Full.bak'
WITH COMPRESSION,
     CHECKSUM,
     STATS = 10,
     MAXTRANSFERSIZE = 4194304,
     BLOCKSIZE = 65536;

-- Striped backup to Azure (multiple blobs)
BACKUP DATABASE [ApplicationDB]
TO URL = N'https://mystorageaccount.blob.core.windows.net/sqlbackups/AppDB_1.bak',
   URL = N'https://mystorageaccount.blob.core.windows.net/sqlbackups/AppDB_2.bak',
   URL = N'https://mystorageaccount.blob.core.windows.net/sqlbackups/AppDB_3.bak'
WITH COMPRESSION,
     CHECKSUM,
     MAXTRANSFERSIZE = 4194304,
     BLOCKSIZE = 65536;

-- Managed backup to Azure (automated, SQL 2016+)
EXEC msdb.managed_backup.sp_backup_config_basic
    @enable_backup = 1,
    @database_name = 'ApplicationDB',
    @container_url = 'https://mystorageaccount.blob.core.windows.net/sqlbackups',
    @retention_days = 30;
```

## Restore Operations

### Full Restore

```sql
-- Simple full restore
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH RECOVERY,
     STATS = 10;

-- Restore with NORECOVERY (to apply subsequent backups)
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH NORECOVERY,
     STATS = 10;

-- Restore with file relocation (MOVE)
RESTORE DATABASE [ApplicationDB_Restored]
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH MOVE N'ApplicationDB' TO N'D:\Data\ApplicationDB_Restored.mdf',
     MOVE N'ApplicationDB_log' TO N'L:\Logs\ApplicationDB_Restored_log.ldf',
     RECOVERY,
     REPLACE,
     STATS = 10;

-- Restore from striped backup
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Full_1.bak',
     DISK = N'F:\Backups\ApplicationDB_Full_2.bak',
     DISK = N'G:\Backups\ApplicationDB_Full_3.bak',
     DISK = N'H:\Backups\ApplicationDB_Full_4.bak'
WITH RECOVERY, STATS = 5;
```

### Full + Differential + Log Restore Sequence

```sql
-- Step 1: Tail-log backup (if database is accessible)
BACKUP LOG [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_TailLog.trn'
WITH NORECOVERY;

-- Step 2: Restore full backup with NORECOVERY
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH NORECOVERY, REPLACE, STATS = 10;

-- Step 3: Restore differential with NORECOVERY
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Diff.bak'
WITH NORECOVERY, STATS = 10;

-- Step 4: Restore log backups in order with NORECOVERY
RESTORE LOG [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Log_001.trn'
WITH NORECOVERY;

RESTORE LOG [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Log_002.trn'
WITH NORECOVERY;

-- Step 5: Restore tail-log and bring online
RESTORE LOG [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_TailLog.trn'
WITH RECOVERY;
```

### Point-in-Time Recovery

```sql
-- Restore to a specific point in time using STOPAT
-- Step 1: Full with NORECOVERY
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH NORECOVERY, REPLACE;

-- Step 2: Differential with NORECOVERY (if applicable)
RESTORE DATABASE [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Diff.bak'
WITH NORECOVERY;

-- Step 3: Log backup with STOPAT
RESTORE LOG [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Log.trn'
WITH STOPAT = '2024-01-15T14:30:00',
     RECOVERY;

-- Restore to a marked transaction
RESTORE LOG [ApplicationDB]
FROM DISK = N'E:\Backups\ApplicationDB_Log.trn'
WITH STOPATMARK = 'BeforeBatchDelete',
     RECOVERY;
```

## Recovery Models

```sql
-- Check current recovery model
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'ApplicationDB';

-- Change recovery model
ALTER DATABASE [ApplicationDB] SET RECOVERY FULL;
ALTER DATABASE [ApplicationDB] SET RECOVERY SIMPLE;
ALTER DATABASE [ApplicationDB] SET RECOVERY BULK_LOGGED;

-- After switching from SIMPLE to FULL, take a full backup immediately
-- (no log backups are possible until the first full backup in FULL mode)
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB_Full_AfterModelChange.bak'
WITH COMPRESSION, CHECKSUM;
```

**Recovery Model Comparison:**

| Feature | FULL | SIMPLE | BULK_LOGGED |
|---------|------|--------|-------------|
| Point-in-time recovery | Yes | No | Limited (not through bulk ops) |
| Log backups required | Yes | No (auto-truncate) | Yes |
| Minimal logging for bulk ops | No | Yes | Yes |
| Transaction log growth risk | High if logs not backed up | Low | Moderate |
| Typical use case | Production OLTP | Dev/Test, read-only | Data warehouse loading |

## Backup Verification

```sql
-- Verify backup without restoring (checks readability and checksums)
RESTORE VERIFYONLY
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak'
WITH CHECKSUM;

-- Read backup header information
RESTORE HEADERONLY
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak';

-- List files in backup set
RESTORE FILELISTONLY
FROM DISK = N'E:\Backups\ApplicationDB_Full.bak';

-- DBCC CHECKDB: full consistency check (schedule regularly)
DBCC CHECKDB ('ApplicationDB')
WITH NO_INFOMSGS, ALL_ERRORMSGS, DATA_PURITY;

-- Physical-only check (faster, less thorough)
DBCC CHECKDB ('ApplicationDB')
WITH PHYSICAL_ONLY, NO_INFOMSGS;

-- Check backup history for a database
SELECT
    bs.database_name,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(MINUTE, bs.backup_start_date, bs.backup_finish_date) AS DurationMin,
    CASE bs.type
        WHEN 'D' THEN 'Full'
        WHEN 'I' THEN 'Differential'
        WHEN 'L' THEN 'Log'
    END AS BackupType,
    CAST(bs.backup_size / 1024.0 / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS BackupSizeGB,
    CAST(bs.compressed_backup_size / 1024.0 / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS CompressedGB,
    bmf.physical_device_name
FROM msdb.dbo.backupset bs
JOIN msdb.dbo.backupmediafamily bmf ON bs.media_set_id = bmf.media_set_id
WHERE bs.database_name = 'ApplicationDB'
ORDER BY bs.backup_start_date DESC;
```

## Backup Compression

```sql
-- Enable backup compression as server default
EXEC sp_configure 'backup compression default', 1;
RECONFIGURE;

-- Override default per backup
BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB.bak'
WITH COMPRESSION;        -- Force compression on

BACKUP DATABASE [ApplicationDB]
TO DISK = N'E:\Backups\ApplicationDB.bak'
WITH NO_COMPRESSION;     -- Force compression off

-- Monitor compression ratio
SELECT
    database_name,
    backup_start_date,
    CAST(backup_size / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS OriginalMB,
    CAST(compressed_backup_size / 1024.0 / 1024.0 AS DECIMAL(10,2)) AS CompressedMB,
    CAST(100.0 - (compressed_backup_size * 100.0 / backup_size) AS DECIMAL(5,1)) AS CompressionPct
FROM msdb.dbo.backupset
WHERE database_name = 'ApplicationDB'
  AND compressed_backup_size IS NOT NULL
ORDER BY backup_start_date DESC;
```

## PowerShell Backup and Restore

```powershell
# Full backup with PowerShell
Backup-SqlDatabase -ServerInstance "SQLPROD01" `
    -Database "ApplicationDB" `
    -BackupFile "E:\Backups\ApplicationDB_Full.bak" `
    -CompressionOption On `
    -Checksum `
    -CopyOnly

# Transaction log backup
Backup-SqlDatabase -ServerInstance "SQLPROD01" `
    -Database "ApplicationDB" `
    -BackupFile "E:\Backups\ApplicationDB_Log.trn" `
    -BackupAction Log `
    -CompressionOption On

# Restore database
Restore-SqlDatabase -ServerInstance "SQLPROD01" `
    -Database "ApplicationDB_Restored" `
    -BackupFile "E:\Backups\ApplicationDB_Full.bak" `
    -RelocateFile @(
        New-Object Microsoft.SqlServer.Management.Smo.RelocateFile(
            "ApplicationDB", "D:\Data\ApplicationDB_Restored.mdf"),
        New-Object Microsoft.SqlServer.Management.Smo.RelocateFile(
            "ApplicationDB_log", "L:\Logs\ApplicationDB_Restored_log.ldf")
    )

# Automated backup script with retention
$backupDir = "E:\Backups"
$retentionDays = 14
$databases = Get-SqlDatabase -ServerInstance "SQLPROD01" |
    Where-Object { $_.Name -notin @("tempdb") -and $_.Status -eq "Normal" }

foreach ($db in $databases) {
    $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
    $backupFile = Join-Path $backupDir "$($db.Name)_Full_$timestamp.bak"
    try {
        Backup-SqlDatabase -ServerInstance "SQLPROD01" `
            -Database $db.Name `
            -BackupFile $backupFile `
            -CompressionOption On `
            -Checksum
        Write-Host "SUCCESS: $($db.Name) backed up to $backupFile"
    } catch {
        Write-Error "FAILED: $($db.Name) - $_"
    }
}

# Clean up old backups
Get-ChildItem $backupDir -Filter "*.bak" |
    Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$retentionDays) } |
    Remove-Item -Force
```

## Best Practices

- Always use CHECKSUM with every BACKUP and periodically run RESTORE VERIFYONLY to ensure backup integrity.
- Take a full backup immediately after changing the recovery model from SIMPLE to FULL; log backups cannot begin until the first full backup.
- Use backup compression by default. It reduces I/O, storage, and network transfer time with minimal CPU overhead on modern hardware.
- Implement the 3-2-1 rule: 3 copies of data, on 2 different media types, with 1 copy offsite (Azure Blob, tape, or replicated storage).
- Test your restore process regularly. An untested backup is not a backup. Restore to a non-production server quarterly at minimum.
- Use COPY_ONLY for ad-hoc backups to avoid disrupting differential and log backup chains.
- Schedule transaction log backups every 5-15 minutes for production OLTP databases in FULL recovery model to minimize data loss and control log growth.
- Always take a tail-log backup before restoring a database, unless the database is already inaccessible.
- Monitor backup duration and size trends over time to detect anomalies (sudden size growth may indicate data issues).
- Stripe large database backups across multiple files to reduce backup duration through parallel I/O.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not taking log backups in FULL recovery model | Transaction log grows until disk fills; database goes offline | Schedule log backups every 5-15 minutes; monitor log file size |
| Forgetting tail-log backup before restore | Loss of all transactions since last log backup | Always run BACKUP LOG ... WITH NORECOVERY before RESTORE |
| Using RECOVERY on intermediate restore steps | Database comes online prematurely; remaining backups cannot be applied | Use NORECOVERY for all steps except the final one |
| Changing recovery model to SIMPLE in production | Breaks log chain; point-in-time recovery is impossible for that gap | Keep FULL recovery model for production; use BULK_LOGGED temporarily during large loads if needed |
| Not testing restores | Discover backup is corrupt or incomplete during a real disaster | Schedule monthly restore tests to a non-production server |
| Backing up to the same disk as data files | Disk failure loses both data and backups simultaneously | Backup to separate storage, network share, or Azure Blob |
| Ignoring CHECKSUM verification | Corrupted backups discovered only during disaster recovery | Always use WITH CHECKSUM and periodically run RESTORE VERIFYONLY |
| Shrinking the log file after every log backup | Constant file growth/shrink cycles cause fragmentation and I/O overhead | Size the log file appropriately and let it stay at steady-state |

## SQL Server Version Notes

- **SQL Server 2016**: Backup to URL supports Azure Blob block blobs. Managed backup to Azure introduced. Backup compression available in Standard Edition (previously Enterprise only).
- **SQL Server 2017**: Backup to Linux-hosted SQL Server. Snapshot backups for Azure VMs using Azure Backup integration.
- **SQL Server 2019**: Accelerated database recovery (ADR) reduces recovery time by using a persistent version store, changing how the transaction log is managed. Backup encryption with certificates improved.
- **SQL Server 2022**: Backup and restore to/from S3-compatible object storage (AWS S3, MinIO, etc.) via BACKUP TO URL with S3 connector. Improved T-SQL snapshot backup support. Intel QuickAssist (QAT) hardware-assisted backup compression. Contained AG backup considerations.

## Sources

- [BACKUP (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/backup-transact-sql)
- [RESTORE (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/restore-statements-transact-sql)
- [Recovery Models (SQL Server)](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/recovery-models-sql-server)
- [SQL Server Backup to URL](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/sql-server-backup-to-url)
- [Tail-Log Backups](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/tail-log-backups-sql-server)
- [DBCC CHECKDB](https://learn.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql)
- [Backup Compression](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-compression-sql-server)
- [Point-in-Time Recovery](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/restore-a-sql-server-database-to-a-point-in-time-full-recovery-model)
