# Index Strategy -- Designing, Maintaining, and Optimizing SQL Server Indexes

## Overview

Indexes are the primary mechanism for fast data retrieval in SQL Server. A well-designed indexing strategy can turn a query that scans millions of rows into a near-instant seek operation. Conversely, missing indexes cause full table scans, while excessive or overlapping indexes waste storage, slow down writes, and increase maintenance overhead.

Effective index strategy requires understanding your workload. OLTP systems favor narrow, selective indexes on frequently queried columns with minimal overhead on INSERT/UPDATE/DELETE operations. Analytical and reporting workloads benefit from wider covering indexes and columnstore indexes that compress and batch-process large datasets. Most real-world systems are a mix of both, requiring careful balancing.

SQL Server provides several index types: traditional B-tree rowstore indexes (clustered and nonclustered), columnstore indexes for analytical workloads, filtered indexes for subset queries, and memory-optimized indexes for In-Memory OLTP tables. The missing index DMVs and Database Engine Tuning Advisor help identify gaps, but human judgment is essential to consolidate suggestions and avoid over-indexing.

## Key Concepts

**Clustered Index**: Defines the physical order of data in the table. Each table can have only one clustered index. The leaf level of a clustered index IS the data. Choosing the right clustering key (narrow, unique, ever-increasing, static) is one of the most impactful design decisions.

**Nonclustered Index**: A separate B-tree structure containing the index key columns plus a pointer (row locator) back to the base table. A table can have up to 999 nonclustered indexes.

**Covering Index**: A nonclustered index that contains all columns needed by a query, eliminating the need for Key Lookups to the base table.

**Included Columns**: Non-key columns stored only at the leaf level of a nonclustered index. Used to create covering indexes without widening the B-tree key.

**Filtered Index**: A nonclustered index with a WHERE clause that indexes only a subset of rows. Smaller, more efficient, lower maintenance cost.

**Columnstore Index**: Stores data in column-oriented compressed segments rather than row-oriented pages. Provides massive compression and batch-mode processing for analytical queries.

**Fill Factor**: The percentage of each index page that is filled during index creation or rebuild. Lower fill factor leaves space for future inserts, reducing page splits at the cost of more storage.

## Clustered Index Design

```sql
-- The ideal clustered index key is:
-- 1. Narrow (4-8 bytes) -- it's included in every nonclustered index
-- 2. Unique -- avoids the hidden 4-byte uniquifier
-- 3. Ever-increasing -- minimizes page splits (IDENTITY, SEQUENCE)
-- 4. Static -- never updated (avoids row movement)

-- Best practice: Use an IDENTITY or SEQUENCE column
CREATE TABLE Sales.Orders (
    OrderID INT IDENTITY(1,1) NOT NULL,
    OrderDate DATE NOT NULL,
    CustomerID INT NOT NULL,
    TotalAmount DECIMAL(18,2) NOT NULL,
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (OrderID)
);

-- Alternative: Natural clustering for range-scan workloads
-- Good when most queries filter by date range
CREATE CLUSTERED INDEX IX_Orders_OrderDate
ON Sales.Orders (OrderDate, OrderID);

-- GUID clustered keys cause fragmentation and wide nonclustered indexes
-- If you must use GUIDs, use NEWSEQUENTIALID() instead of NEWID()
CREATE TABLE Sales.WebOrders (
    WebOrderID UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID() NOT NULL,
    -- ...
    CONSTRAINT PK_WebOrders PRIMARY KEY CLUSTERED (WebOrderID)
);
```

## Nonclustered Index Design

```sql
-- Basic nonclustered index for equality lookups
CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
ON Sales.Orders (CustomerID);

-- Composite index for multi-column predicates
-- Column order matters: most selective / equality columns first
CREATE NONCLUSTERED INDEX IX_Orders_CustDate
ON Sales.Orders (CustomerID, OrderDate);

-- Covering index with INCLUDE columns
-- Eliminates Key Lookups for queries selecting these columns
CREATE NONCLUSTERED INDEX IX_Orders_CustDate_Cover
ON Sales.Orders (CustomerID, OrderDate)
INCLUDE (TotalAmount, ShipAddress, Status);

-- Example: This query is fully covered by the index above
SELECT OrderDate, TotalAmount, Status
FROM Sales.Orders
WHERE CustomerID = 1001
    AND OrderDate >= '2024-01-01';
```

## Filtered Indexes

```sql
-- Filtered index: Only indexes rows matching the WHERE clause
-- Much smaller than a full index; ideal for sparse predicates

-- Index only active orders (ignore the 99% that are completed)
CREATE NONCLUSTERED INDEX IX_Orders_Active
ON Sales.Orders (CustomerID, OrderDate)
INCLUDE (TotalAmount)
WHERE Status = 'Active';

-- Index only non-NULL values for a sparse column
CREATE NONCLUSTERED INDEX IX_Orders_PromoCode
ON Sales.Orders (PromoCode)
WHERE PromoCode IS NOT NULL;

-- Limitations:
-- 1. Parameterized queries may not match the filter
-- 2. The query predicate must be a superset of the filter
-- 3. Cannot use filtered indexes with certain features (MERGE, parameterized plans)
```

## Columnstore Indexes

```sql
-- Clustered Columnstore Index (CCI):
-- Replaces the entire table storage with column-oriented compressed segments
-- Best for large fact tables in data warehousing / analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON dw.FactSales;

-- Insert performance: Use bulk inserts of 102,400+ rows to avoid
-- the delta store (rowgroup quality depends on batch size)

-- Nonclustered Columnstore Index (NCCI):
-- Adds columnstore alongside existing rowstore (real-time operational analytics)
-- Available on OLTP tables without changing the base storage
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Analytics
ON Sales.Orders (OrderDate, CustomerID, TotalAmount, ProductCategory);

-- Filtered NCCI for "hot" data (2016+):
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_Orders_Recent
ON Sales.Orders (OrderDate, CustomerID, TotalAmount)
WHERE OrderDate >= '2024-01-01';

-- Monitor columnstore rowgroup quality
SELECT
    OBJECT_NAME(object_id) AS table_name,
    index_id,
    row_group_id,
    state_desc,
    total_rows,
    deleted_rows,
    size_in_bytes / 1024 AS size_kb,
    trim_reason_desc
FROM sys.dm_db_column_store_row_group_physical_stats
WHERE OBJECT_NAME(object_id) = 'FactSales'
ORDER BY row_group_id;

-- Rebuild columnstore to merge small/deleted rowgroups
ALTER INDEX CCI_FactSales ON dw.FactSales
REORGANIZE WITH (COMPRESS_ALL_ROW_GROUPS = ON);
```

## Missing Index DMVs

```sql
-- Find the most impactful missing indexes
-- The improvement_measure formula estimates total query cost savings
SELECT TOP 25
    CONVERT(DECIMAL(18,2),
        migs.avg_total_user_cost * migs.avg_user_impact *
        (migs.user_seeks + migs.user_scans)
    ) AS improvement_measure,
    DB_NAME(mid.database_id) AS database_name,
    mid.statement AS table_name,
    mid.equality_columns,
    mid.inequality_columns,
    mid.included_columns,
    migs.user_seeks,
    migs.user_scans,
    migs.last_user_seek,
    CAST(migs.avg_total_user_cost AS DECIMAL(10,2)) AS avg_cost,
    CAST(migs.avg_user_impact AS DECIMAL(5,2)) AS avg_impact_pct
FROM sys.dm_db_missing_index_groups AS mig
INNER JOIN sys.dm_db_missing_index_group_stats AS migs
    ON mig.index_group_handle = migs.group_handle
INNER JOIN sys.dm_db_missing_index_details AS mid
    ON mig.index_handle = mid.index_handle
WHERE mid.database_id = DB_ID()
ORDER BY improvement_measure DESC;

-- Generate CREATE INDEX statements from DMV suggestions
SELECT
    'CREATE NONCLUSTERED INDEX [IX_' +
    OBJECT_NAME(mid.object_id) + '_' +
    REPLACE(REPLACE(REPLACE(ISNULL(mid.equality_columns,''), '[',''), ']',''), ', ','_') +
    '] ON ' + mid.statement +
    ' (' + ISNULL(mid.equality_columns, '') +
    CASE WHEN mid.inequality_columns IS NOT NULL
        THEN CASE WHEN mid.equality_columns IS NOT NULL
            THEN ', ' ELSE '' END + mid.inequality_columns
        ELSE '' END +
    ')' +
    CASE WHEN mid.included_columns IS NOT NULL
        THEN ' INCLUDE (' + mid.included_columns + ')'
        ELSE '' END + ';' AS create_index_statement
FROM sys.dm_db_missing_index_details AS mid
WHERE mid.database_id = DB_ID();
```

## Unused Index Detection

```sql
-- Find indexes with zero seeks/scans but high write overhead
-- Candidates for removal (verify with full workload cycle first)
SELECT
    OBJECT_NAME(i.object_id) AS table_name,
    i.name AS index_name,
    i.type_desc,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,  -- write overhead from this index
    ius.last_user_seek,
    ius.last_user_scan,
    ps.row_count,
    CAST(ps.reserved_page_count * 8.0 / 1024 AS DECIMAL(10,2)) AS index_size_mb
FROM sys.indexes AS i
LEFT JOIN sys.dm_db_index_usage_stats AS ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
    AND ius.database_id = DB_ID()
JOIN sys.dm_db_partition_stats AS ps
    ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
    AND i.type_desc = 'NONCLUSTERED'
    AND i.is_primary_key = 0
    AND i.is_unique_constraint = 0
    AND ISNULL(ius.user_seeks, 0) = 0
    AND ISNULL(ius.user_scans, 0) = 0
    AND ISNULL(ius.user_lookups, 0) = 0
ORDER BY ius.user_updates DESC;

-- Find duplicate/overlapping indexes
-- Two indexes where one is a left-prefix of the other
SELECT
    OBJECT_NAME(i1.object_id) AS table_name,
    i1.name AS index_1,
    i2.name AS index_2,
    COL_NAME(i1.object_id, ic1.column_id) AS shared_leading_column
FROM sys.indexes i1
JOIN sys.index_columns ic1
    ON i1.object_id = ic1.object_id AND i1.index_id = ic1.index_id
JOIN sys.indexes i2
    ON i1.object_id = i2.object_id AND i1.index_id < i2.index_id
JOIN sys.index_columns ic2
    ON i2.object_id = ic2.object_id AND i2.index_id = ic2.index_id
WHERE ic1.key_ordinal = 1 AND ic2.key_ordinal = 1
    AND ic1.column_id = ic2.column_id
    AND i1.type_desc = 'NONCLUSTERED'
    AND i2.type_desc = 'NONCLUSTERED'
ORDER BY table_name, index_1;
```

## Index Fragmentation and Maintenance

```sql
-- Check index fragmentation levels
SELECT
    OBJECT_NAME(ips.object_id) AS table_name,
    i.name AS index_name,
    ips.index_type_desc,
    ips.avg_fragmentation_in_percent,
    ips.page_count,
    ips.avg_page_space_used_in_percent,
    ips.record_count,
    ips.fragment_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') AS ips
JOIN sys.indexes AS i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.page_count > 1000  -- skip tiny indexes
    AND ips.avg_fragmentation_in_percent > 5
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Maintenance strategy:
-- < 5% fragmentation: Do nothing
-- 5-30% fragmentation: REORGANIZE (online, minimal locking)
-- > 30% fragmentation: REBUILD (more thorough, can be done online in Enterprise)

-- Reorganize (always online, log-friendly)
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders REORGANIZE;

-- Rebuild offline (locks table, faster)
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders REBUILD;

-- Rebuild online (Enterprise Edition only, minimal blocking)
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders
REBUILD WITH (ONLINE = ON);

-- Rebuild all indexes on a table
ALTER INDEX ALL ON Sales.Orders REBUILD WITH (ONLINE = ON);

-- Resumable index rebuild (2017+): Can pause and resume
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders
REBUILD WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 60);

-- Check and resume a paused rebuild
SELECT * FROM sys.index_resumable_operations;
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders RESUME;

-- Set fill factor during rebuild
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders
REBUILD WITH (FILLFACTOR = 90, ONLINE = ON);
```

## Online and Resumable Index Operations

```sql
-- Online index creation (Enterprise Edition)
-- Does not block concurrent DML
CREATE NONCLUSTERED INDEX IX_Orders_Status
ON Sales.Orders (Status, OrderDate)
INCLUDE (CustomerID, TotalAmount)
WITH (ONLINE = ON, SORT_IN_TEMPDB = ON);

-- Resumable online index creation (2019+)
-- Can be paused and resumed across maintenance windows
CREATE NONCLUSTERED INDEX IX_Orders_Region
ON Sales.Orders (Region, OrderDate)
WITH (ONLINE = ON, RESUMABLE = ON, MAX_DURATION = 120);

-- Pause a running resumable operation
ALTER INDEX IX_Orders_Region ON Sales.Orders PAUSE;

-- Resume later
ALTER INDEX IX_Orders_Region ON Sales.Orders RESUME
WITH (MAX_DURATION = 60);

-- Abort if no longer needed
ALTER INDEX IX_Orders_Region ON Sales.Orders ABORT;

-- WAIT_AT_LOW_PRIORITY: Reduce blocking during online operations
ALTER INDEX IX_Orders_CustomerID ON Sales.Orders
REBUILD WITH (
    ONLINE = ON,
    WAIT_AT_LOW_PRIORITY (
        MAX_DURATION = 10 MINUTES,
        ABORT_AFTER_WAIT = SELF  -- abort rebuild if can't get lock
    )
);
```

## Index Size and Storage Analysis

```sql
-- Index sizes for all tables in the database
SELECT
    s.name AS schema_name,
    t.name AS table_name,
    i.name AS index_name,
    i.type_desc,
    CAST(SUM(ps.reserved_page_count) * 8.0 / 1024 AS DECIMAL(10,2)) AS index_size_mb,
    SUM(ps.row_count) AS row_count,
    i.fill_factor
FROM sys.tables AS t
JOIN sys.schemas AS s ON t.schema_id = s.schema_id
JOIN sys.indexes AS i ON t.object_id = i.object_id
JOIN sys.dm_db_partition_stats AS ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
GROUP BY s.name, t.name, i.name, i.type_desc, i.fill_factor
ORDER BY SUM(ps.reserved_page_count) DESC;

-- Total index vs data ratio per table
SELECT
    OBJECT_NAME(i.object_id) AS table_name,
    SUM(CASE WHEN i.index_id <= 1 THEN ps.reserved_page_count ELSE 0 END) * 8 / 1024 AS data_mb,
    SUM(CASE WHEN i.index_id > 1 THEN ps.reserved_page_count ELSE 0 END) * 8 / 1024 AS index_mb,
    COUNT(DISTINCT CASE WHEN i.index_id > 1 THEN i.index_id END) AS nc_index_count
FROM sys.indexes AS i
JOIN sys.dm_db_partition_stats AS ps ON i.object_id = ps.object_id AND i.index_id = ps.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id
HAVING SUM(ps.reserved_page_count) > 128  -- > 1MB
ORDER BY index_mb DESC;
```

## Non-Clustered Index Design Principles

### Column Ordering in Composite Indexes

Place equality predicate columns before range predicate columns in the index key:

```sql
-- Query pattern: WHERE Status = 'Active' AND OrderDate > '2024-01-01'
-- GOOD: equality column first, range column second
CREATE NONCLUSTERED INDEX IX_Orders_Status_Date
ON Orders (Status, OrderDate)
INCLUDE (CustomerID, TotalAmount);

-- BAD: range column first — cannot seek past first column
CREATE NONCLUSTERED INDEX IX_Orders_Date_Status
ON Orders (OrderDate, Status);
```

### INCLUDE vs Key Columns

Use INCLUDE for columns needed only in the SELECT list or JOIN output — they are stored at leaf level only, keeping the B-tree narrow for seeks:

```sql
-- Covering index: eliminates Key Lookups for this query pattern
CREATE NONCLUSTERED INDEX IX_Orders_Customer
ON Orders (CustomerID)
INCLUDE (OrderDate, TotalAmount, ShipAddress)
WITH (ONLINE = ON, FILLFACTOR = 90);
```

INCLUDE supports up to 1023 non-key columns and allows data types prohibited in key columns (VARCHAR(MAX), XML, etc.).

### Filtered Indexes for Subset Queries

```sql
-- Index only active orders — smaller, faster, less maintenance
CREATE NONCLUSTERED INDEX IX_Orders_ActiveOnly
ON Orders (CustomerID, OrderDate)
WHERE Status = 'Active';

-- Filtered index for NULL-handling queries
CREATE NONCLUSTERED INDEX IX_Orders_Unshipped
ON Orders (OrderDate)
WHERE ShipDate IS NULL;
```

### When to Create vs Skip

Create when: table has high read-to-write ratio, queries consistently filter on same columns, execution plans show scans or Key Lookups on hot queries.

Skip when: table is write-heavy (every index adds INSERT/UPDATE/DELETE overhead), the column has very low cardinality (Gender, Boolean), or an existing index can be extended with INCLUDE columns instead.

## Best Practices

- Choose a narrow, unique, ever-increasing, static clustered index key (IDENTITY or SEQUENCE columns are ideal)
- Create covering indexes with INCLUDE columns instead of widening the index key -- included columns are stored only at the leaf level
- Consolidate missing index DMV suggestions that share the same leading columns rather than creating each one individually
- Use filtered indexes for queries that frequently target a known subset of data (e.g., WHERE Status = 'Active')
- Verify unused index candidates against a full business cycle (monthly/quarterly reports) before dropping them
- Use online operations (Enterprise Edition) for production index builds and rebuilds to minimize blocking
- Use resumable index operations (2017+) for large indexes that cannot complete in a single maintenance window
- Set fill factor lower (80-90%) only on indexes with random inserts causing page splits; leave sequential indexes at 100%
- Add columnstore indexes to large tables that serve both OLTP and analytical queries (real-time operational analytics pattern)
- Monitor sys.dm_db_index_operational_stats for lock waits, page splits, and latch contention at the index level

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using GUID (NEWID()) as clustered index key | Massive fragmentation, wide nonclustered indexes, random I/O | Switch to IDENTITY/SEQUENCE or use NEWSEQUENTIALID() |
| Creating every missing index suggestion verbatim | Index bloat, slow DML, overlapping indexes, wasted storage | Consolidate suggestions; test with representative workload |
| Never checking for unused indexes | Wasted storage and write overhead on every INSERT/UPDATE/DELETE | Regularly audit with dm_db_index_usage_stats; drop truly unused indexes |
| Rebuilding indexes with < 5% fragmentation | Unnecessary I/O and log generation during maintenance | Only reorganize at 5-30%, rebuild above 30%; skip tiny indexes |
| Dropping production indexes without validation | Query performance regression if index was used by monthly/quarterly jobs | Disable (not drop) first; monitor for one full business cycle |
| Missing INCLUDE columns on frequently used indexes | Excessive Key Lookups; fast seek followed by slow random I/O to base table | Add INCLUDE columns for SELECT-list and join columns |
| Ignoring columnstore for analytical queries on large tables | Full rowstore scans; orders of magnitude slower than columnstore batch mode | Add nonclustered columnstore index for analytical access patterns |

## SQL Server Version Notes

**SQL Server 2016**: Nonclustered columnstore indexes updatable and filterable. Online index rebuild for nonclustered columnstore. Operational analytics pattern (NCCI on OLTP tables). In-Memory OLTP index improvements.

**SQL Server 2017**: Resumable online index rebuild. Automatic tuning for index recommendations. Columnstore supports LOB columns (varchar(max), etc.). Clustered columnstore online rebuild.

**SQL Server 2019**: Resumable online index creation (not just rebuild). Hybrid buffer pool for persistent memory. Columnstore with ordered clustered columnstore indexes (preview). Hash index enhancements for In-Memory OLTP.

**SQL Server 2022**: XML compression for XML indexes. Ordered clustered columnstore indexes (GA). Resumable add table constraints (UNIQUE, PRIMARY KEY). Improved columnstore segment elimination. Shrink database with wait at low priority.

## Sources

- [Indexes Overview (Microsoft Docs)](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/indexes)
- [Clustered and Nonclustered Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- [Columnstore Indexes Guide](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/columnstore-indexes-overview)
- [Missing Index DMVs](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-missing-index-details-transact-sql)
- [Resumable Index Operations](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/guidelines-for-online-index-operations)
- [Index Architecture and Design Guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)
- [Overview of Clustered Indexes](https://www.sqlshack.com/overview-of-sql-server-clustered-index/)
- [Overview of Non-Clustered Indexes](https://www.sqlshack.com/overview-of-non-clustered-indexes-in-sql-server/)
- [Designing Effective Clustered Indexes](https://www.sqlshack.com/designing-effective-sql-server-clustered-indexes/)
- [Designing Effective Non-Clustered Indexes](https://www.sqlshack.com/designing-effective-sql-server-non-clustered-indexes/)
- [Non-Clustered Indexes with Included Columns](https://www.sqlshack.com/sql-server-non-clustered-indexes-with-included-columns/)
- [SQL Server Indexes Series](https://www.sqlshack.com/sql-server-indexes-series-intro/)
