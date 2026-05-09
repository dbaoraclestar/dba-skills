# Table Partitioning — Horizontal Data Distribution for Manageability and Performance

## Overview

Table partitioning in SQL Server divides a table's rows into multiple physical storage units (partitions) while maintaining a single logical table. Queries, DML, and DDL operate against the table as a single object, but the storage engine can perform operations on individual partitions independently — enabling partition elimination in queries, fast data loading via partition switching, and per-partition maintenance.

Partitioning is most valuable for large tables (typically 100M+ rows) where data has a natural time-based or range-based lifecycle. The classic pattern is a fact table partitioned by date, where old partitions can be archived or dropped instantly via SWITCH operations instead of expensive DELETE statements. Partitioning also enables parallel query execution at the partition level and allows different compression strategies per partition.

It is important to understand that partitioning is primarily a manageability feature, not a query performance feature. A well-indexed non-partitioned table often outperforms a poorly designed partitioned table. Partition elimination only helps when the partition key appears in the WHERE clause. The decision to partition should be driven by data lifecycle requirements (load, archive, purge) rather than raw query speed.

## Key Concepts

- **Partition function** — Defines how rows map to partitions based on a single column's value. Specifies boundary values and whether each boundary belongs to the LEFT or RIGHT partition.
- **Partition scheme** — Maps the partitions defined by a function to filegroups. Each partition can reside on a different filegroup.
- **Partition elimination** — The query optimizer's ability to skip partitions that cannot contain matching rows, based on the WHERE clause filtering on the partition key.
- **Partition switching** — An instant metadata-only operation that moves a partition in or out of a table. The source and target must be schema-aligned.
- **Sliding window** — A pattern combining SWITCH, SPLIT, and MERGE to age out old data and add new partitions on a rolling schedule.
- **$PARTITION function** — A system function that returns the partition number for a given value, useful for diagnostics and partition-aware queries.

## Partition Functions

### RANGE LEFT vs RANGE RIGHT

```sql
-- RANGE LEFT: boundary value belongs to the LEFT (lower) partition
-- Partition 1: <= '2026-01-01'
-- Partition 2: > '2026-01-01' AND <= '2026-02-01'
-- Partition 3: > '2026-02-01' AND <= '2026-03-01'
-- Partition 4: > '2026-03-01'

CREATE PARTITION FUNCTION pf_Monthly_Left (DATE)
AS RANGE LEFT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01',
    '2026-04-01', '2026-05-01', '2026-06-01'
);

-- RANGE RIGHT: boundary value belongs to the RIGHT (upper) partition
-- Partition 1: < '2026-01-01'
-- Partition 2: >= '2026-01-01' AND < '2026-02-01'
-- Partition 3: >= '2026-02-01' AND < '2026-03-01'
-- Partition 4: >= '2026-03-01'

CREATE PARTITION FUNCTION pf_Monthly_Right (DATE)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01',
    '2026-04-01', '2026-05-01', '2026-06-01'
);

-- RANGE RIGHT is typically preferred for date partitioning
-- because each boundary is the START of its partition,
-- which aligns naturally with "this month's data" logic.
```

## Partition Schemes

```sql
-- Map all partitions to a single filegroup
CREATE PARTITION SCHEME ps_Monthly
AS PARTITION pf_Monthly_Right
ALL TO ([PRIMARY]);

-- Map partitions to different filegroups
-- (requires filegroups to exist first)
ALTER DATABASE MyDB ADD FILEGROUP FG_2026_01;
ALTER DATABASE MyDB ADD FILEGROUP FG_2026_02;
ALTER DATABASE MyDB ADD FILEGROUP FG_2026_03;
ALTER DATABASE MyDB ADD FILEGROUP FG_Archive;
ALTER DATABASE MyDB ADD FILEGROUP FG_Current;

-- Add files to filegroups
ALTER DATABASE MyDB ADD FILE
    (NAME = 'Data_2026_01', FILENAME = 'D:\Data\Data_2026_01.ndf', SIZE = 1GB)
    TO FILEGROUP FG_2026_01;

CREATE PARTITION SCHEME ps_Monthly_FG
AS PARTITION pf_Monthly_Right
TO (FG_Archive, FG_2026_01, FG_2026_02, FG_2026_03,
    FG_Current, FG_Current, FG_Current);
-- Note: N boundary values create N+1 partitions, so N+1 filegroups needed
```

## Creating Partitioned Tables

```sql
-- Create a partitioned table
CREATE TABLE dbo.SalesHistory (
    sale_id      BIGINT IDENTITY(1,1) NOT NULL,
    sale_date    DATE NOT NULL,
    customer_id  INT NOT NULL,
    product_id   INT NOT NULL,
    quantity     INT NOT NULL,
    amount       DECIMAL(19,4) NOT NULL,
    region       VARCHAR(50) NOT NULL,
    CONSTRAINT PK_SalesHistory PRIMARY KEY (sale_id, sale_date)
        -- partition key MUST be part of the primary key
) ON ps_Monthly(sale_date);   -- place on partition scheme

-- Create nonclustered indexes on the partition scheme (aligned)
CREATE NONCLUSTERED INDEX IX_SalesHistory_Customer
ON dbo.SalesHistory (customer_id, sale_date)
ON ps_Monthly(sale_date);     -- partition-aligned index

-- Non-aligned index (on a specific filegroup, not partitioned)
CREATE NONCLUSTERED INDEX IX_SalesHistory_Product
ON dbo.SalesHistory (product_id)
ON [PRIMARY];                 -- not partition-aligned
```

## Partition Elimination in Queries

```sql
-- Partition elimination occurs when the WHERE clause filters on
-- the partition key column. The optimizer skips non-matching partitions.

-- GOOD: partition elimination happens (filters on sale_date)
SELECT SUM(amount)
FROM dbo.SalesHistory
WHERE sale_date >= '2026-03-01' AND sale_date < '2026-04-01';

-- Check the actual execution plan for:
-- "Actual Partition Count: 1" (only 1 partition scanned)

-- BAD: no partition elimination (no filter on partition key)
SELECT SUM(amount)
FROM dbo.SalesHistory
WHERE customer_id = 12345;
-- Scans ALL partitions

-- Verify partition elimination with SET STATISTICS IO
SET STATISTICS IO ON;
-- Look for which partitions were accessed
```

## Sliding Window Pattern

### The Complete Lifecycle

```sql
-- Step 1: Create staging table for switch-out (same schema, same filegroup)
CREATE TABLE dbo.SalesHistory_Archive (
    sale_id      BIGINT NOT NULL,
    sale_date    DATE NOT NULL,
    customer_id  INT NOT NULL,
    product_id   INT NOT NULL,
    quantity     INT NOT NULL,
    amount       DECIMAL(19,4) NOT NULL,
    region       VARCHAR(50) NOT NULL,
    CONSTRAINT CK_Archive_Date CHECK (sale_date < '2026-01-01')
) ON FG_Archive;

-- Step 2: Switch oldest partition OUT (instant metadata operation)
ALTER TABLE dbo.SalesHistory
    SWITCH PARTITION 1 TO dbo.SalesHistory_Archive;
-- Partition 1 is now empty; Archive table has the data

-- Step 3: Merge the now-empty boundary to clean up
ALTER PARTITION FUNCTION pf_Monthly_Right()
    MERGE RANGE ('2026-01-01');
-- Removes the boundary; reduces partition count by 1

-- Step 4: Split to add a new future partition
-- First, ensure the NEXT USED filegroup is set
ALTER PARTITION SCHEME ps_Monthly
    NEXT USED FG_Current;

ALTER PARTITION FUNCTION pf_Monthly_Right()
    SPLIT RANGE ('2026-07-01');
-- Adds a new boundary; increases partition count by 1

-- Step 5: Create staging table for switch-in (bulk load new data)
CREATE TABLE dbo.SalesHistory_Staging (
    sale_id      BIGINT NOT NULL,
    sale_date    DATE NOT NULL,
    customer_id  INT NOT NULL,
    product_id   INT NOT NULL,
    quantity     INT NOT NULL,
    amount       DECIMAL(19,4) NOT NULL,
    region       VARCHAR(50) NOT NULL,
    CONSTRAINT CK_Staging_Date CHECK (
        sale_date >= '2026-07-01' AND sale_date < '2026-08-01'
    )
) ON FG_Current;

-- Bulk load data into staging
-- ... INSERT or BULK INSERT ...

-- Step 6: Switch staging IN (instant)
ALTER TABLE dbo.SalesHistory_Staging
    SWITCH TO dbo.SalesHistory PARTITION $PARTITION.pf_Monthly_Right('2026-07-01');
```

### Important Switch Requirements

```sql
-- For SWITCH to succeed, ALL of the following must be true:
-- 1. Source and target have identical column definitions
-- 2. Source and target have identical indexes
-- 3. Source has a CHECK constraint matching the target partition boundaries
-- 4. Source is on the same filegroup as the target partition
-- 5. Target partition must be empty (for switch-in)
-- 6. Source table must be empty after switch (for switch-out, automatic)
-- 7. Both tables must have the same compression settings
-- 8. Foreign key constraints: source cannot have FKs referencing it

-- Verify switch compatibility
ALTER TABLE dbo.SalesHistory_Staging
    SWITCH TO dbo.SalesHistory PARTITION 7
    WITH (WAIT_AT_LOW_PRIORITY (MAX_DURATION = 1 MINUTES, ABORT_AFTER_WAIT = SELF));
```

## $PARTITION Diagnostic Function

```sql
-- Find which partition a value falls into
SELECT $PARTITION.pf_Monthly_Right('2026-03-15') AS partition_number;

-- Count rows per partition
SELECT
    p.partition_number,
    p.rows,
    prv_left.value AS lower_boundary,
    prv_right.value AS upper_boundary,
    fg.name AS filegroup_name
FROM sys.partitions p
JOIN sys.destination_data_spaces dds
    ON p.partition_number = dds.destination_id
JOIN sys.partition_schemes ps
    ON dds.partition_scheme_id = ps.data_space_id
JOIN sys.filegroups fg
    ON dds.data_space_id = fg.data_space_id
LEFT JOIN sys.partition_range_values prv_left
    ON ps.function_id = prv_left.function_id
   AND p.partition_number - 1 = prv_left.boundary_id
LEFT JOIN sys.partition_range_values prv_right
    ON ps.function_id = prv_right.function_id
   AND p.partition_number = prv_right.boundary_id
WHERE p.object_id = OBJECT_ID('dbo.SalesHistory')
  AND p.index_id IN (0, 1)   -- heap or clustered index
ORDER BY p.partition_number;
```

## Partition-Aligned Indexes

```sql
-- Aligned index: partitioned on the same scheme as the table
-- Required for partition switching
-- Partition key must be in the index key or INCLUDE list

-- Aligned clustered index
CREATE CLUSTERED INDEX CIX_Sales
ON dbo.SalesHistory (sale_date, sale_id)
ON ps_Monthly(sale_date);

-- Aligned nonclustered index
CREATE NONCLUSTERED INDEX IX_Sales_Region
ON dbo.SalesHistory (region, sale_date)
INCLUDE (amount)
ON ps_Monthly(sale_date);

-- Unique indexes must include the partition key
-- This FAILS:
-- CREATE UNIQUE INDEX IX_Sales_ID ON dbo.SalesHistory(sale_id) ON ps_Monthly(sale_date);
-- This WORKS:
CREATE UNIQUE INDEX IX_Sales_ID
ON dbo.SalesHistory(sale_id, sale_date)
ON ps_Monthly(sale_date);
```

## Data Compression Per Partition

```sql
-- Different compression per partition
-- Hot partitions: no compression (faster writes)
-- Warm partitions: row compression
-- Cold partitions: page compression

ALTER TABLE dbo.SalesHistory
    REBUILD PARTITION = 1 WITH (DATA_COMPRESSION = PAGE);

ALTER TABLE dbo.SalesHistory
    REBUILD PARTITION = 5 WITH (DATA_COMPRESSION = ROW);

ALTER TABLE dbo.SalesHistory
    REBUILD PARTITION = 7 WITH (DATA_COMPRESSION = NONE);

-- Estimate compression savings
EXEC sp_estimate_data_compression_savings
    @schema_name = 'dbo',
    @object_name = 'SalesHistory',
    @index_id = NULL,
    @partition_number = 3,
    @data_compression = 'PAGE';
```

## Partitioning and Columnstore

```sql
-- Columnstore indexes on partitioned tables (SQL Server 2016+)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_SalesHistory
ON dbo.SalesHistory
ON ps_Monthly(sale_date);

-- Partition switching works with columnstore
-- Staging table must also have a columnstore index
CREATE TABLE dbo.SalesHistory_CCI_Staging (
    sale_id      BIGINT NOT NULL,
    sale_date    DATE NOT NULL,
    customer_id  INT NOT NULL,
    product_id   INT NOT NULL,
    quantity     INT NOT NULL,
    amount       DECIMAL(19,4) NOT NULL,
    region       VARCHAR(50) NOT NULL,
    CONSTRAINT CK_CCI_Stage CHECK (
        sale_date >= '2026-07-01' AND sale_date < '2026-08-01'
    ),
    INDEX CCI_Staging CLUSTERED COLUMNSTORE
) ON FG_Current;

-- Archival compression for columnstore on cold partitions
ALTER TABLE dbo.SalesHistory
    REBUILD PARTITION = 1
    WITH (DATA_COMPRESSION = COLUMNSTORE_ARCHIVE);
```

## Monitoring with sys.dm_db_partition_stats

```sql
-- Partition stats overview
SELECT
    OBJECT_SCHEMA_NAME(ps.object_id) AS schema_name,
    OBJECT_NAME(ps.object_id) AS table_name,
    i.name AS index_name,
    ps.partition_number,
    ps.row_count,
    ps.used_page_count * 8 / 1024 AS used_mb,
    ps.reserved_page_count * 8 / 1024 AS reserved_mb,
    ps.in_row_data_page_count,
    ps.lob_used_page_count,
    p.data_compression_desc
FROM sys.dm_db_partition_stats ps
JOIN sys.partitions p
    ON ps.partition_id = p.partition_id
LEFT JOIN sys.indexes i
    ON ps.object_id = i.object_id
   AND ps.index_id = i.index_id
WHERE ps.object_id = OBJECT_ID('dbo.SalesHistory')
ORDER BY ps.index_id, ps.partition_number;

-- Identify skewed partitions (uneven data distribution)
SELECT
    partition_number,
    row_count,
    CAST(100.0 * row_count / SUM(row_count) OVER () AS DECIMAL(5,2)) AS pct_of_total
FROM sys.dm_db_partition_stats
WHERE object_id = OBJECT_ID('dbo.SalesHistory')
  AND index_id IN (0, 1)
ORDER BY partition_number;
```

## Best Practices

- Use RANGE RIGHT for date-based partitioning so each boundary represents the start of its partition period.
- Always split on an empty partition boundary to avoid expensive data movement. SPLIT a future boundary before data arrives.
- Always merge an empty partition boundary after switching data out. MERGE on an empty boundary is instant.
- Set NEXT USED on the partition scheme before every SPLIT operation to avoid errors.
- Include the partition key in all unique indexes and primary keys — this is required by SQL Server.
- Use partition-aligned indexes to enable partition switching; non-aligned indexes block SWITCH.
- Apply data compression tiering: NONE for hot/write-heavy partitions, ROW for warm, PAGE for cold, COLUMNSTORE_ARCHIVE for archive.
- Maintain an empty "future" partition at the right end to catch data that arrives ahead of the next SPLIT.
- Automate the sliding window process in a SQL Agent job to ensure consistent partition maintenance.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Splitting a non-empty partition | Expensive data movement, long-running operation, blocking | Always split boundaries in the empty future partition |
| Forgetting NEXT USED before SPLIT | SPLIT fails with an error about no next filegroup | Always call `ALTER PARTITION SCHEME ... NEXT USED` before SPLIT |
| Missing partition key in unique indexes | CREATE INDEX fails with error | Include partition key in every unique index and PK |
| Non-aligned indexes blocking SWITCH | SWITCH fails because all indexes must be aligned | Ensure all indexes use the partition scheme or drop non-aligned indexes |
| No CHECK constraint on staging table for switch-in | SWITCH fails because SQL Server cannot verify data fits the target partition | Add a CHECK constraint matching the exact partition boundaries |
| Partitioning small tables | Overhead of partition management with no benefit | Only partition tables with 100M+ rows or clear lifecycle requirements |

## SQL Server Version Notes

- **SQL Server 2016** — Partition-level online index rebuild. Truncate individual partitions (`TRUNCATE TABLE ... WITH (PARTITIONS (1, 3))`). Up to 15,000 partitions per table (was 1,000 before 2012).
- **SQL Server 2017** — Resumable online index rebuild for partitioned indexes. Improved partition elimination with filtered statistics.
- **SQL Server 2019** — `WAIT_AT_LOW_PRIORITY` for partition switch operations. Improved intelligent query processing with partitioned columnstore.
- **SQL Server 2022** — Partition elimination improvements for parameterized queries. Ordered clustered columnstore index on partitioned tables. XML compression on partitioned data. Ledger tables support partitioning.

## Sources

- [Partitioned Tables and Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/partitions/partitioned-tables-and-indexes)
- [CREATE PARTITION FUNCTION](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-partition-function-transact-sql)
- [CREATE PARTITION SCHEME](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-partition-scheme-transact-sql)
- [Switch a Partition](https://learn.microsoft.com/en-us/sql/t-sql/statements/alter-table-transact-sql#switch-table)
- [sys.dm_db_partition_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-partition-stats-transact-sql)
- [$PARTITION Function](https://learn.microsoft.com/en-us/sql/t-sql/functions/partition-transact-sql)
