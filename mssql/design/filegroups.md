# Filegroups — Physical Storage Organization and Partial Database Availability

## Overview

Filegroups are the logical layer between SQL Server database objects (tables, indexes) and the physical data files (.ndf) on disk. Every database has a PRIMARY filegroup containing the system catalog and any objects not explicitly placed elsewhere. User-defined filegroups allow you to control where specific tables and indexes are physically stored, enabling I/O distribution, targeted backup/restore, partial database availability, and read-only data management.

Filegroup design is a foundational architecture decision that affects backup strategy, recovery time, I/O performance, and operational flexibility. For large databases, placing historical data on read-only filegroups eliminates those files from routine backups and DBCC CHECKDB operations. For mission-critical systems, piecemeal restore lets you bring essential filegroups online first while restoring secondary data in the background.

Modern SQL Server also uses specialized filegroup types: FILESTREAM filegroups for storing large binary objects as file system files, and memory-optimized filegroups for In-Memory OLTP tables. Understanding when and how to use each filegroup type is essential for optimal SQL Server storage architecture.

## Key Concepts

- **PRIMARY filegroup** — The default filegroup. Contains system tables and any user objects not explicitly placed on another filegroup. Cannot be made read-only.
- **User-defined filegroup** — Created to hold specific tables, indexes, or LOB data. Can be made read-only. Can be backed up and restored independently.
- **Data file** — A physical .ndf file (or .mdf for PRIMARY). Each filegroup contains one or more data files. SQL Server fills files within a filegroup proportionally.
- **Proportional fill** — SQL Server distributes writes across data files within a filegroup based on their free space ratio, not in a round-robin fashion.
- **Default filegroup** — The filegroup where objects are placed when no explicit ON clause is specified. Initially PRIMARY; can be changed with ALTER DATABASE.
- **Read-only filegroup** — A filegroup marked as READONLY. Data cannot be modified. Excluded from backups after the initial read-only backup.
- **Partial database availability** — The ability to bring a database online with only some filegroups available, while others are being restored or are offline.

## Creating Filegroups and Files

### Basic Filegroup Setup

```sql
-- Create user-defined filegroups
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_Data;
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_Indexes;
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_Archive;
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_LOB;

-- Add data files to filegroups
ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_Data_01',
    FILENAME = 'D:\Data\AW_Data_01.ndf',
    SIZE = 1GB,
    MAXSIZE = 10GB,
    FILEGROWTH = 256MB
) TO FILEGROUP FG_Data;

-- Multiple files per filegroup for I/O parallelism
ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_Data_02',
    FILENAME = 'E:\Data\AW_Data_02.ndf',   -- different disk
    SIZE = 1GB,
    MAXSIZE = 10GB,
    FILEGROWTH = 256MB
) TO FILEGROUP FG_Data;

ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_Index_01',
    FILENAME = 'F:\Indexes\AW_Index_01.ndf',
    SIZE = 512MB,
    FILEGROWTH = 128MB
) TO FILEGROUP FG_Indexes;

-- Change the default filegroup (objects without explicit ON clause go here)
ALTER DATABASE AdventureWorks MODIFY FILEGROUP FG_Data DEFAULT;
```

### Listing Filegroups and Files

```sql
-- Filegroup summary
SELECT
    fg.name AS filegroup_name,
    fg.type_desc,
    fg.is_default,
    fg.is_read_only,
    COUNT(df.file_id) AS file_count,
    SUM(df.size * 8 / 1024) AS total_size_mb,
    SUM(FILEPROPERTY(df.name, 'SpaceUsed') * 8 / 1024) AS used_mb
FROM sys.filegroups fg
JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
GROUP BY fg.name, fg.type_desc, fg.is_default, fg.is_read_only
ORDER BY fg.name;

-- Detailed file information
SELECT
    fg.name AS filegroup_name,
    df.name AS logical_name,
    df.physical_name,
    df.size * 8 / 1024 AS size_mb,
    FILEPROPERTY(df.name, 'SpaceUsed') * 8 / 1024 AS used_mb,
    df.max_size,
    df.growth,
    df.is_percent_growth
FROM sys.database_files df
JOIN sys.filegroups fg ON df.data_space_id = fg.data_space_id
ORDER BY fg.name, df.file_id;
```

## Placing Tables and Indexes on Filegroups

```sql
-- Place a table on a specific filegroup
CREATE TABLE dbo.SalesData (
    sale_id      INT IDENTITY PRIMARY KEY NONCLUSTERED ON FG_Indexes,
    sale_date    DATE NOT NULL,
    amount       DECIMAL(19,4) NOT NULL,
    description  NVARCHAR(MAX) NULL
) ON FG_Data                      -- clustered index / heap on FG_Data
TEXTIMAGE_ON FG_LOB;              -- LOB data (NVARCHAR(MAX)) on FG_LOB

-- Place nonclustered indexes on a separate filegroup
CREATE NONCLUSTERED INDEX IX_SalesData_Date
ON dbo.SalesData (sale_date)
INCLUDE (amount)
ON FG_Indexes;

-- Move an existing table to a different filegroup
-- (requires rebuilding the clustered index)
CREATE CLUSTERED INDEX CIX_SalesData_Date
ON dbo.SalesData (sale_date)
WITH (DROP_EXISTING = ON)
ON FG_Data;

-- Move a heap to a different filegroup (SQL Server 2014+)
ALTER TABLE dbo.HeapTable REBUILD WITH (DATA_COMPRESSION = NONE)
ON FG_Data;
```

## Read-Only Filegroups

```sql
-- Mark a filegroup as read-only
ALTER DATABASE AdventureWorks
    MODIFY FILEGROUP FG_Archive READONLY;

-- Verify read-only status
SELECT name, is_read_only
FROM sys.filegroups
WHERE name = 'FG_Archive';

-- Attempting to modify data on a read-only filegroup raises an error
-- Msg 652: The filegroup "FG_Archive" is readonly.

-- Revert to read-write (for maintenance)
ALTER DATABASE AdventureWorks
    MODIFY FILEGROUP FG_Archive READWRITE;

-- Benefits of read-only filegroups:
-- 1. No checkpoint writes — reduced I/O
-- 2. Excluded from subsequent full/differential backups
-- 3. DBCC CHECKDB can skip read-only filegroups (WITH EXTENDED_LOGICAL_CHECKS)
-- 4. Data corruption protection (no accidental modifications)
```

### Read-Only Archival Pattern

```sql
-- Pattern: move aged data to a read-only filegroup

-- Step 1: Create the archive filegroup and file
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_2025_Archive;
ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_Archive_2025',
    FILENAME = 'G:\Archive\AW_Archive_2025.ndf',
    SIZE = 2GB
) TO FILEGROUP FG_2025_Archive;

-- Step 2: Create archive table on the filegroup
CREATE TABLE dbo.Sales_2025 (
    sale_id   INT NOT NULL,
    sale_date DATE NOT NULL,
    amount    DECIMAL(19,4) NOT NULL,
    CONSTRAINT CK_Sales2025_Date
        CHECK (sale_date >= '2025-01-01' AND sale_date < '2026-01-01')
) ON FG_2025_Archive;

-- Step 3: Load data (partition switch, INSERT, or BCP)
INSERT dbo.Sales_2025
SELECT sale_id, sale_date, amount
FROM dbo.SalesData
WHERE sale_date >= '2025-01-01' AND sale_date < '2026-01-01';

-- Step 4: Delete from source
DELETE FROM dbo.SalesData
WHERE sale_date >= '2025-01-01' AND sale_date < '2026-01-01';

-- Step 5: Mark as read-only
ALTER DATABASE AdventureWorks
    MODIFY FILEGROUP FG_2025_Archive READONLY;

-- Step 6: Take one final backup of the read-only filegroup
BACKUP DATABASE AdventureWorks
    FILEGROUP = 'FG_2025_Archive'
    TO DISK = 'H:\Backups\AW_FG_2025_Archive.bak'
    WITH COMPRESSION;
-- This filegroup never needs to be backed up again
```

## Partial Database Availability

### Piecemeal Restore

```sql
-- Piecemeal restore: bring PRIMARY online first, then secondary filegroups

-- Step 1: Restore PRIMARY filegroup (database comes online in partial mode)
RESTORE DATABASE AdventureWorks
    FILEGROUP = 'PRIMARY'
    FROM DISK = 'H:\Backups\AW_Full.bak'
    WITH PARTIAL, NORECOVERY;

RESTORE LOG AdventureWorks
    FROM DISK = 'H:\Backups\AW_Log.trn'
    WITH RECOVERY;

-- Database is now ONLINE with PRIMARY filegroup available
-- Other filegroups show status RECOVERY_PENDING or OFFLINE

-- Step 2: Restore secondary filegroups (while database is online)
RESTORE DATABASE AdventureWorks
    FILEGROUP = 'FG_Data'
    FROM DISK = 'H:\Backups\AW_Full.bak'
    WITH NORECOVERY;

RESTORE LOG AdventureWorks
    FROM DISK = 'H:\Backups\AW_Log.trn'
    WITH RECOVERY;

-- Step 3: Read-only filegroups can be restored from the last
-- pre-read-only backup (no log restore needed if unchanged)
RESTORE DATABASE AdventureWorks
    FILEGROUP = 'FG_2025_Archive'
    FROM DISK = 'H:\Backups\AW_FG_2025_Archive.bak'
    WITH RECOVERY;
```

### Checking Filegroup Online Status

```sql
-- Check which filegroups are online
SELECT
    fg.name AS filegroup_name,
    fg.is_read_only,
    df.state_desc AS file_state
FROM sys.filegroups fg
JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
ORDER BY fg.name;

-- Possible states: ONLINE, OFFLINE, RESTORING, RECOVERY_PENDING, SUSPECT
```

## Filegroup Backup and Restore

```sql
-- Full filegroup backup
BACKUP DATABASE AdventureWorks
    FILEGROUP = 'FG_Data'
    TO DISK = 'H:\Backups\AW_FG_Data.bak'
    WITH COMPRESSION, CHECKSUM;

-- Multiple filegroups in one backup
BACKUP DATABASE AdventureWorks
    FILEGROUP = 'FG_Data',
    FILEGROUP = 'FG_Indexes'
    TO DISK = 'H:\Backups\AW_FG_DataIdx.bak'
    WITH COMPRESSION;

-- Restore a single filegroup (online restore, Enterprise only)
RESTORE DATABASE AdventureWorks
    FILEGROUP = 'FG_Data'
    FROM DISK = 'H:\Backups\AW_FG_Data.bak'
    WITH NORECOVERY;

-- Apply transaction logs to bring filegroup current
RESTORE LOG AdventureWorks
    FROM DISK = 'H:\Backups\AW_Log_1.trn'
    WITH NORECOVERY;
RESTORE LOG AdventureWorks
    FROM DISK = 'H:\Backups\AW_Log_2.trn'
    WITH RECOVERY;

-- List filegroups in a backup set
RESTORE FILELISTONLY FROM DISK = 'H:\Backups\AW_Full.bak';
```

## FILESTREAM Filegroups

```sql
-- Enable FILESTREAM at the instance level first
-- (via SQL Server Configuration Manager or sp_configure)
EXEC sp_configure 'filestream access level', 2;  -- 0=off, 1=T-SQL, 2=T-SQL+Win32
RECONFIGURE;

-- Create a FILESTREAM filegroup
ALTER DATABASE AdventureWorks ADD FILEGROUP FG_FileStream CONTAINS FILESTREAM;

-- Add a FILESTREAM data container (directory, not a file)
ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_FileStream',
    FILENAME = 'D:\FileStream\AW_FS'     -- directory path, not file
) TO FILEGROUP FG_FileStream;

-- Create a table with FILESTREAM column
CREATE TABLE dbo.Documents (
    doc_id       UNIQUEIDENTIFIER ROWGUIDCOL NOT NULL UNIQUE DEFAULT NEWID(),
    doc_name     NVARCHAR(256) NOT NULL,
    doc_content  VARBINARY(MAX) FILESTREAM NULL,
    created_at   DATETIME2(3) NOT NULL DEFAULT SYSDATETIME()
) ON FG_Data
FILESTREAM_ON FG_FileStream;

-- FILESTREAM requirements:
-- 1. Table must have a UNIQUEIDENTIFIER column with ROWGUIDCOL
-- 2. FILESTREAM column must be VARBINARY(MAX)
-- 3. Database must have a FILESTREAM filegroup
-- 4. Instance must have FILESTREAM enabled
```

## Memory-Optimized Filegroup (In-Memory OLTP)

```sql
-- Add a memory-optimized filegroup (only one per database)
ALTER DATABASE AdventureWorks
    ADD FILEGROUP FG_InMemory CONTAINS MEMORY_OPTIMIZED_DATA;

-- Add a data container for checkpoint file pairs
ALTER DATABASE AdventureWorks ADD FILE (
    NAME = 'AW_InMemory',
    FILENAME = 'D:\InMemory\AW_IMOLTP'    -- directory path
) TO FILEGROUP FG_InMemory;

-- Create a memory-optimized table
CREATE TABLE dbo.SessionCache (
    session_key   NVARCHAR(100) NOT NULL PRIMARY KEY NONCLUSTERED
        HASH WITH (BUCKET_COUNT = 131072),
    session_data  NVARCHAR(MAX) NOT NULL,
    expiry_time   DATETIME2 NOT NULL,
    INDEX IX_Expiry NONCLUSTERED (expiry_time)
) WITH (
    MEMORY_OPTIMIZED = ON,
    DURABILITY = SCHEMA_AND_DATA     -- survives restart
    -- DURABILITY = SCHEMA_ONLY      -- data lost on restart (tempdb-like)
);

-- Memory-optimized filegroup characteristics:
-- 1. Only one per database
-- 2. Cannot be removed once created
-- 3. Stores checkpoint file pairs (data + delta files)
-- 4. Data is primarily in memory; files are for durability/recovery
-- 5. Cannot be made read-only
```

## TempDB Filegroup Configuration

```sql
-- TempDB best practices: multiple data files for contention reduction
-- Rule of thumb: 1 file per CPU core up to 8, then add in groups of 4

-- Check current tempdb file configuration
SELECT
    name,
    physical_name,
    size * 8 / 1024 AS size_mb,
    growth,
    is_percent_growth
FROM tempdb.sys.database_files;

-- Add tempdb data files (uniform size is critical)
ALTER DATABASE tempdb ADD FILE (
    NAME = 'tempdev2',
    FILENAME = 'T:\TempDB\tempdev2.ndf',
    SIZE = 4GB,
    FILEGROWTH = 512MB
);

ALTER DATABASE tempdb ADD FILE (
    NAME = 'tempdev3',
    FILENAME = 'T:\TempDB\tempdev3.ndf',
    SIZE = 4GB,
    FILEGROWTH = 512MB
);

ALTER DATABASE tempdb ADD FILE (
    NAME = 'tempdev4',
    FILENAME = 'T:\TempDB\tempdev4.ndf',
    SIZE = 4GB,
    FILEGROWTH = 512MB
);

-- Ensure all tempdb files are the same size
-- (proportional fill works best with equal-sized files)

-- Monitor tempdb allocation contention
SELECT
    session_id,
    wait_type,
    wait_duration_ms,
    resource_description
FROM sys.dm_os_waiting_tasks
WHERE wait_type LIKE 'PAGE%LATCH%'
  AND resource_description LIKE '2:%'   -- database_id 2 = tempdb
ORDER BY wait_duration_ms DESC;

-- Check for GAM/SGAM/PFS contention in tempdb
SELECT
    wait_type,
    waiting_tasks_count,
    wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type IN ('PAGELATCH_UP', 'PAGELATCH_EX', 'PAGELATCH_SH')
ORDER BY wait_time_ms DESC;
```

## Monitoring Filegroup Space Usage

```sql
-- Space usage by filegroup
SELECT
    fg.name AS filegroup_name,
    SUM(df.size * 8 / 1024) AS total_mb,
    SUM(FILEPROPERTY(df.name, 'SpaceUsed') * 8 / 1024) AS used_mb,
    SUM(df.size * 8 / 1024) - SUM(FILEPROPERTY(df.name, 'SpaceUsed') * 8 / 1024) AS free_mb,
    CAST(100.0 * SUM(FILEPROPERTY(df.name, 'SpaceUsed')) / SUM(df.size) AS DECIMAL(5,2)) AS pct_used
FROM sys.filegroups fg
JOIN sys.database_files df ON fg.data_space_id = df.data_space_id
GROUP BY fg.name
ORDER BY fg.name;

-- Objects on each filegroup
SELECT
    fg.name AS filegroup_name,
    OBJECT_SCHEMA_NAME(p.object_id) + '.' + OBJECT_NAME(p.object_id) AS object_name,
    i.name AS index_name,
    i.type_desc,
    SUM(au.total_pages * 8 / 1024) AS total_mb,
    SUM(p.rows) AS row_count
FROM sys.allocation_units au
JOIN sys.partitions p ON au.container_id = p.hobt_id
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.filegroups fg ON au.data_space_id = fg.data_space_id
WHERE OBJECTPROPERTY(p.object_id, 'IsUserTable') = 1
GROUP BY fg.name, p.object_id, i.name, i.type_desc
ORDER BY fg.name, total_mb DESC;
```

## Best Practices

- Separate the PRIMARY filegroup from user data. Keep only system catalog objects on PRIMARY; set a user-defined filegroup as DEFAULT.
- Use read-only filegroups for historical data to eliminate backup overhead and prevent accidental modifications.
- Make all tempdb data files the same size with the same growth settings. Proportional fill depends on equal file sizes.
- Place transaction log files on their own dedicated volume, never in a filegroup (log files are not in any filegroup).
- Use multiple data files per filegroup on high-throughput workloads to reduce allocation contention and enable parallel I/O.
- Plan filegroup layout before creating large databases. Moving objects between filegroups requires index rebuilds.
- Back up read-only filegroups once after marking them read-only. They are excluded from subsequent full and differential backups.
- Use piecemeal restore for large databases to reduce RTO. Bring PRIMARY and critical filegroups online first.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Putting everything on PRIMARY filegroup | Cannot leverage partial backup/restore, no I/O isolation | Create user filegroups; change default filegroup away from PRIMARY |
| Unequal tempdb file sizes | Proportional fill skews writes to the largest file; contention persists | Make all tempdb data files identical in size and growth |
| Not setting NEXT USED before partition SPLIT | SPLIT fails if no filegroup is designated for the new partition | Always call ALTER PARTITION SCHEME ... NEXT USED before SPLIT |
| Forgetting FILESTREAM requires UNIQUEIDENTIFIER ROWGUIDCOL | CREATE TABLE fails with a confusing error | Add a UNIQUEIDENTIFIER column with ROWGUIDCOL to FILESTREAM tables |
| Never backing up read-only filegroups after making them read-only | No backup exists for the read-only data; irrecoverable on disk failure | Take a filegroup backup immediately after marking READONLY |
| Adding memory-optimized filegroup without planning | Cannot remove it once created; permanent database metadata change | Plan In-Memory OLTP adoption carefully before adding the filegroup |

## SQL Server Version Notes

- **SQL Server 2016** — Multiple memory-optimized filegroup containers supported (previously limited to one). Improved FILESTREAM scalability. TempDB configuration during setup (number of files auto-detected by CPU count).
- **SQL Server 2017** — Indirect checkpoint default for new databases (affects filegroup I/O patterns). FILESTREAM supported on Linux (limited). Clusterless Always On readable secondaries with filegroup placement.
- **SQL Server 2019** — Memory-optimized tempdb metadata (reduces tempdb allocation contention without adding files). Hybrid buffer pool for persistent memory (PMEM) on filegroup data files. Accelerated Database Recovery per filegroup.
- **SQL Server 2022** — XML compression on filegroup data. Improved buffer pool scan for large filegroups. S3-compatible object storage for backup (indirect filegroup impact). Contained Availability Groups with per-AG filegroup isolation.

## Sources

- [Database Files and Filegroups](https://learn.microsoft.com/en-us/sql/relational-databases/databases/database-files-and-filegroups)
- [Piecemeal Restore](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/piecemeal-restores-sql-server)
- [FILESTREAM](https://learn.microsoft.com/en-us/sql/relational-databases/blob/filestream-sql-server)
- [Memory-Optimized Filegroup](https://learn.microsoft.com/en-us/sql/relational-databases/in-memory-oltp/the-memory-optimized-filegroup)
- [tempdb Database](https://learn.microsoft.com/en-us/sql/relational-databases/databases/tempdb-database)
- [Read-Only Filegroups and Backup](https://learn.microsoft.com/en-us/sql/relational-databases/backup-restore/back-up-files-and-filegroups-sql-server)
