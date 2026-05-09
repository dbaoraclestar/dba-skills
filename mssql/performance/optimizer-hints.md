# Optimizer Hints -- Overriding SQL Server Query Optimizer Behavior

## Overview

SQL Server's query optimizer is a cost-based optimizer that generally produces good execution plans. However, there are situations where the optimizer makes suboptimal choices due to stale statistics, parameter sniffing, cardinality estimation errors, or edge cases that the cost model does not handle well. Optimizer hints give you the ability to override the optimizer's decisions and direct it toward a specific plan shape, join strategy, parallelism level, or index choice.

Hints should be treated as a last resort after exhausting other approaches: updating statistics, creating appropriate indexes, rewriting the query, or using Query Store plan forcing. Hints create a maintenance burden because they override the optimizer's ability to adapt to changing data distributions, schema changes, and SQL Server version improvements. A hint that helps today may hurt tomorrow when the data profile changes.

That said, hints are sometimes necessary and appropriate. Parameter sniffing fixes, emergency production hotfixes, vendor application queries you cannot modify, and known optimizer limitations all justify hint usage. SQL Server 2022's Query Store hints provide a clean way to apply hints without modifying application code, making them more practical for production use.

## Key Concepts

**Query Hints**: Applied at the statement level using the OPTION clause. Affect the entire query's compilation and execution behavior (MAXDOP, RECOMPILE, join algorithm preference, etc.).

**Table Hints**: Applied to individual table references using the WITH clause. Control index selection, locking behavior, and scan direction for that specific table access.

**Join Hints**: Applied between two tables in a JOIN clause. Force a specific join algorithm (LOOP, HASH, MERGE) for that particular join.

**Plan Guides**: Server-side objects that attach hints to queries without modifying application code. Useful for vendor applications where you cannot change the SQL.

**Query Store Hints (2022)**: Associate hints with a Query Store query_id. Cleaner and more manageable than plan guides, with built-in monitoring for hint failures.

**USE HINT**: Modern syntax for applying session-level hints as query hints. Replaces many trace flag-based workarounds.

## Query Hints (OPTION Clause)

### RECOMPILE

```sql
-- Forces fresh compilation every execution
-- Eliminates parameter sniffing; optimizer sees actual parameter values
-- Use for: queries with wildly varying parameters that get bad cached plans
SELECT CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE OrderDate BETWEEN @StartDate AND @EndDate
    AND Status = @Status
OPTION (RECOMPILE);

-- Inside stored procedures -- per-statement RECOMPILE
CREATE OR ALTER PROCEDURE GetOrdersByDateRange
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Only this statement recompiles; others use cached plan
    SELECT CustomerID, OrderDate, TotalAmount
    FROM Sales.Orders
    WHERE OrderDate BETWEEN @StartDate AND @EndDate
    OPTION (RECOMPILE);
END;

-- RECOMPILE on the procedure definition (entire procedure recompiles)
CREATE OR ALTER PROCEDURE GetOrdersByDateRange
    @StartDate DATE,
    @EndDate DATE
WITH RECOMPILE
AS
BEGIN
    SELECT CustomerID, OrderDate, TotalAmount
    FROM Sales.Orders
    WHERE OrderDate BETWEEN @StartDate AND @EndDate;
END;
```

### OPTIMIZE FOR

```sql
-- Compile the plan optimized for specific parameter values
-- Useful when you know the "typical" parameter profile
SELECT CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustID
OPTION (OPTIMIZE FOR (@CustID = 1001));

-- OPTIMIZE FOR UNKNOWN: Use average density instead of sniffed values
-- Good when there is no single "typical" value
SELECT CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustID
    AND OrderDate > @StartDate
OPTION (OPTIMIZE FOR (@CustID UNKNOWN, @StartDate UNKNOWN));

-- Combine with specific and unknown
SELECT CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustID
    AND Status = @Status
OPTION (OPTIMIZE FOR (@CustID UNKNOWN, @Status = 'Active'));
```

### MAXDOP

```sql
-- Control the degree of parallelism for a specific query
-- Override the server-wide MAXDOP setting

-- Force serial execution (no parallelism)
SELECT CustomerID, COUNT(*), SUM(TotalAmount)
FROM Sales.Orders
GROUP BY CustomerID
OPTION (MAXDOP 1);

-- Limit parallelism to 4 threads
SELECT o.OrderID, c.CustomerName, p.ProductName
FROM Sales.Orders o
JOIN Sales.Customers c ON o.CustomerID = c.CustomerID
JOIN Sales.OrderDetails od ON o.OrderID = od.OrderID
JOIN Sales.Products p ON od.ProductID = p.ProductID
WHERE o.OrderDate > '2024-01-01'
OPTION (MAXDOP 4);
```

### Join Algorithm Hints

```sql
-- Force HASH JOIN for all joins in the query
SELECT o.OrderID, c.CustomerName
FROM Sales.Orders o
JOIN Sales.Customers c ON o.CustomerID = c.CustomerID
OPTION (HASH JOIN);

-- Force MERGE JOIN for all joins
SELECT o.OrderID, c.CustomerName
FROM Sales.Orders o
JOIN Sales.Customers c ON o.CustomerID = c.CustomerID
OPTION (MERGE JOIN);

-- Force LOOP JOIN for all joins
SELECT o.OrderID, c.CustomerName
FROM Sales.Orders o
JOIN Sales.Customers c ON o.CustomerID = c.CustomerID
OPTION (LOOP JOIN);

-- WARNING: Query-level join hints force ALL joins in the query
-- to use that algorithm. Prefer per-join hints (see Join Hints section)
-- for multi-join queries where only one join needs overriding.
```

### FORCE ORDER

```sql
-- Force the optimizer to use tables in the FROM clause order
-- Use when the optimizer's join order is suboptimal
SELECT o.OrderID, c.CustomerName, p.ProductName
FROM Sales.Customers c          -- processed first
JOIN Sales.Orders o             -- joined second
    ON c.CustomerID = o.CustomerID
JOIN Sales.OrderDetails od      -- joined third
    ON o.OrderID = od.OrderID
JOIN Sales.Products p           -- joined last
    ON od.ProductID = p.ProductID
WHERE c.Country = 'US'
OPTION (FORCE ORDER);
```

### USE PLAN

```sql
-- Force a complete execution plan specified as XML
-- Captured from a previous good execution
DECLARE @PlanXml NVARCHAR(MAX);
-- Capture the plan XML from a known-good execution
-- then paste it as the USE PLAN parameter

SELECT CustomerID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustID
OPTION (USE PLAN N'<ShowPlanXML xmlns="http://schemas.microsoft.com/sqlserver/2004/07/showplan">
  <!-- paste full plan XML here -->
</ShowPlanXML>');

-- This is fragile: any schema change breaks the plan
-- Prefer Query Store plan forcing over USE PLAN
```

### Other Query Hints

```sql
-- FAST n: Optimize for fast return of first n rows
-- Useful for paginated queries where you want quick initial results
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders
ORDER BY OrderDate DESC
OPTION (FAST 100);

-- MIN_GRANT_PERCENT / MAX_GRANT_PERCENT: Control memory grant size
-- Prevent excessive grants that starve other queries
SELECT CustomerID, COUNT(*), SUM(TotalAmount)
FROM Sales.Orders
GROUP BY CustomerID
ORDER BY SUM(TotalAmount) DESC
OPTION (MIN_GRANT_PERCENT = 1, MAX_GRANT_PERCENT = 10);

-- NO_PERFORMANCE_SPOOL: Disable eager/lazy spools
-- Spools are temp storage for subquery results; sometimes counterproductive
SELECT *
FROM Sales.Orders o
WHERE EXISTS (
    SELECT 1 FROM Sales.OrderDetails od WHERE od.OrderID = o.OrderID
)
OPTION (NO_PERFORMANCE_SPOOL);

-- QUERYTRACEON: Enable a trace flag for a specific query
-- Requires sysadmin or plan guide; not usable by regular users directly
SELECT CustomerID, COUNT(*)
FROM Sales.Orders
GROUP BY CustomerID
OPTION (QUERYTRACEON 9481);  -- Force legacy CE for this query

-- QUERYTRACEON with plan guides (non-sysadmin workaround)
EXEC sp_create_plan_guide
    @name = N'Guide_LegacyCE',
    @stmt = N'SELECT CustomerID, COUNT(*) FROM Sales.Orders GROUP BY CustomerID',
    @type = N'SQL',
    @module_or_batch = NULL,
    @params = NULL,
    @hints = N'OPTION (QUERYTRACEON 9481)';

-- DISABLE_OPTIMIZED_PLAN_FORCING (2022): Opt out of optimized plan forcing
-- for a specific query if it causes issues
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (DISABLE_OPTIMIZED_PLAN_FORCING);
```

## Table Hints (WITH Clause)

### Index Hints

```sql
-- Force use of a specific index
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (INDEX(IX_Orders_CustomerID))
WHERE CustomerID = 1001;

-- Force use of the clustered index (table scan / clustered index scan)
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (INDEX(0))   -- 0 = heap scan
WHERE CustomerID = 1001;

SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (INDEX(1))   -- 1 = clustered index scan
WHERE CustomerID = 1001;

-- Force a specific index by name
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (INDEX(PK_Orders))
WHERE OrderDate > '2024-01-01';

-- FORCESEEK: Force an index seek operation (no scans allowed)
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (FORCESEEK)
WHERE CustomerID = 1001;

-- FORCESEEK with specific index and seek columns
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (FORCESEEK(IX_Orders_CustDate(CustomerID, OrderDate)))
WHERE CustomerID = 1001 AND OrderDate > '2024-01-01';

-- FORCESCAN: Force a scan (prevent seeks)
-- Rarely useful; sometimes helps when optimizer overestimates seek efficiency
SELECT OrderID, OrderDate, CustomerID
FROM Sales.Orders WITH (FORCESCAN)
WHERE OrderDate BETWEEN '2024-01-01' AND '2024-12-31';
```

### Locking Hints

```sql
-- NOLOCK (READ UNCOMMITTED): Avoid shared locks; allows dirty reads
-- Use with caution: can read uncommitted, rolled-back, or partial data
SELECT OrderID, OrderDate, TotalAmount
FROM Sales.Orders WITH (NOLOCK)
WHERE OrderDate > '2024-01-01';

-- READPAST: Skip locked rows instead of waiting
-- Useful for queue-processing patterns
SELECT TOP 10 OrderID, Status
FROM Sales.OrderQueue WITH (READPAST, UPDLOCK, ROWLOCK)
WHERE Status = 'Pending';

-- HOLDLOCK (SERIALIZABLE): Hold shared locks until transaction ends
-- Prevents phantom reads
SELECT OrderID, TotalAmount
FROM Sales.Orders WITH (HOLDLOCK)
WHERE CustomerID = 1001;

-- UPDLOCK: Acquire update locks instead of shared locks
-- Prevents deadlocks in read-then-update patterns
BEGIN TRAN;
SELECT OrderID, Status
FROM Sales.Orders WITH (UPDLOCK, ROWLOCK)
WHERE OrderID = 42518;

UPDATE Sales.Orders SET Status = 'Processing'
WHERE OrderID = 42518;
COMMIT;

-- TABLOCK: Acquire table-level lock (bulk operations)
INSERT INTO Sales.OrderArchive WITH (TABLOCK)
SELECT * FROM Sales.Orders WHERE OrderDate < '2023-01-01';

-- TABLOCKX: Acquire exclusive table-level lock
TRUNCATE TABLE Sales.TempStaging;
INSERT INTO Sales.TempStaging WITH (TABLOCKX)
SELECT * FROM ExternalSource;
```

### Other Table Hints

```sql
-- KEEPIDENTITY: Preserve identity values during bulk insert
INSERT INTO Sales.Orders WITH (KEEPIDENTITY)
SELECT * FROM StagingOrders;

-- KEEPDEFAULTS: Use column defaults for missing columns
INSERT INTO Sales.Orders WITH (KEEPDEFAULTS)
(CustomerID, OrderDate)
SELECT CustomerID, OrderDate FROM StagingOrders;

-- NOWAIT: Fail immediately if lock cannot be acquired
SELECT OrderID, Status
FROM Sales.Orders WITH (NOWAIT)
WHERE OrderID = 42518;
-- Raises error 1222 instead of waiting for the lock

-- SPATIAL_WINDOW_MAX_CELLS: Hint for spatial index queries
SELECT *
FROM GeoData WITH (SPATIAL_WINDOW_MAX_CELLS = 256)
WHERE Location.STIntersects(@SearchArea) = 1;
```

## Join Hints (Per-Join)

```sql
-- Per-join hints: Apply to a specific join, not all joins in the query
-- Syntax: table1 {LOOP | HASH | MERGE} JOIN table2

-- Force HASH JOIN for the Orders-Customers join only
SELECT o.OrderID, c.CustomerName, p.ProductName
FROM Sales.Orders o
INNER HASH JOIN Sales.Customers c ON o.CustomerID = c.CustomerID
INNER JOIN Sales.Products p ON o.ProductID = p.ProductID;
-- The Products join uses whatever the optimizer chooses

-- Force LOOP JOIN for the detail lookup
SELECT o.OrderID, od.ProductID, od.Quantity
FROM Sales.Orders o
INNER JOIN Sales.OrderDetails od ON o.OrderID = od.OrderID
INNER LOOP JOIN Sales.Products p ON od.ProductID = p.ProductID;

-- Force MERGE JOIN (requires sorted inputs)
SELECT o.OrderID, c.CustomerName
FROM Sales.Orders o
INNER MERGE JOIN Sales.Customers c ON o.CustomerID = c.CustomerID;

-- REMOTE join hint: Force one side to be evaluated at the remote server
-- Only applicable for linked server queries
SELECT o.OrderID, r.RemoteData
FROM Sales.Orders o
INNER REMOTE JOIN RemoteServer.RemoteDB.dbo.RemoteTable r
    ON o.OrderID = r.OrderID;

-- NOTE: Per-join hints implicitly force join order (like FORCE ORDER)
-- for the hinted joins. The optimizer cannot reorder them.
```

## Plan Guides

```sql
-- Plan guides attach hints to queries without modifying application code
-- Three types: SQL, OBJECT, TEMPLATE

-- SQL plan guide: Match a specific SQL statement
EXEC sp_create_plan_guide
    @name = N'Guide_OrderLookup',
    @stmt = N'SELECT * FROM Sales.Orders WHERE CustomerID = @p0',
    @type = N'SQL',
    @module_or_batch = NULL,
    @params = N'@p0 int',
    @hints = N'OPTION (RECOMPILE)';

-- OBJECT plan guide: Target a statement inside a stored procedure
EXEC sp_create_plan_guide
    @name = N'Guide_GetOrders_Select',
    @stmt = N'SELECT OrderID, OrderDate, TotalAmount
FROM Sales.Orders
WHERE CustomerID = @CustID AND OrderDate > @StartDate',
    @type = N'OBJECT',
    @module_or_batch = N'dbo.GetOrdersByCustomer',
    @params = NULL,
    @hints = N'OPTION (OPTIMIZE FOR (@CustID UNKNOWN))';

-- TEMPLATE plan guide: Force parameterization for ad hoc queries
DECLARE @stmt NVARCHAR(MAX);
DECLARE @params NVARCHAR(MAX);
EXEC sp_get_query_template
    N'SELECT * FROM Sales.Orders WHERE OrderID = 42518',
    @stmt OUTPUT,
    @params OUTPUT;

EXEC sp_create_plan_guide
    @name = N'Template_OrderLookup',
    @stmt = @stmt,
    @type = N'TEMPLATE',
    @module_or_batch = NULL,
    @params = @params,
    @hints = N'OPTION (PARAMETERIZATION FORCED)';

-- List all plan guides
SELECT
    name,
    scope_type_desc,
    is_disabled,
    query_text,
    hints,
    create_date
FROM sys.plan_guides
ORDER BY create_date DESC;

-- Validate a plan guide
SELECT * FROM sys.fn_validate_plan_guide(plan_guide_id)
FROM sys.plan_guides
WHERE name = 'Guide_OrderLookup';

-- Disable / Enable / Drop plan guides
EXEC sp_control_plan_guide N'DISABLE', N'Guide_OrderLookup';
EXEC sp_control_plan_guide N'ENABLE', N'Guide_OrderLookup';
EXEC sp_control_plan_guide N'DROP', N'Guide_OrderLookup';

-- Drop all plan guides in the database
EXEC sp_control_plan_guide N'DROP ALL';
```

## Query Store Hints (SQL Server 2022)

```sql
-- Query Store hints: The modern replacement for plan guides
-- Cleaner, more manageable, with built-in failure tracking
-- Requires Query Store to be enabled

-- Step 1: Find the query_id in Query Store
SELECT
    q.query_id,
    qt.query_sql_text,
    rs.avg_duration / 1000 AS avg_duration_ms,
    rs.count_executions
FROM sys.query_store_query AS q
JOIN sys.query_store_query_text AS qt ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan AS p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE '%Sales.Orders%'
ORDER BY rs.avg_duration DESC;

-- Step 2: Apply a hint to the query
EXEC sys.sp_query_store_set_hints
    @query_id = 42,
    @query_hints = N'OPTION (RECOMPILE)';

-- More examples
-- Force MAXDOP
EXEC sys.sp_query_store_set_hints @query_id = 42,
    @query_hints = N'OPTION (MAXDOP 4)';

-- Force a table hint via query hint syntax
EXEC sys.sp_query_store_set_hints @query_id = 42,
    @query_hints = N'OPTION (TABLE HINT (Sales.Orders, FORCESEEK))';

-- Combine multiple hints
EXEC sys.sp_query_store_set_hints @query_id = 42,
    @query_hints = N'OPTION (MAXDOP 2, RECOMPILE, FORCE ORDER)';

-- Force legacy cardinality estimator
EXEC sys.sp_query_store_set_hints @query_id = 42,
    @query_hints = N'OPTION (USE HINT (''FORCE_LEGACY_CARDINALITY_ESTIMATION''))';

-- View active Query Store hints
SELECT
    query_hint_id,
    query_id,
    query_hint_text,
    last_query_hint_failure_reason_desc,
    query_hint_failure_count,
    source_desc
FROM sys.query_store_query_hints;

-- Clear a Query Store hint
EXEC sys.sp_query_store_clear_hints @query_id = 42;
```

## USE HINT for Session-Level Hints

```sql
-- USE HINT provides named hints that replace many trace flag workarounds
-- Available from SQL Server 2016 SP1+

-- Force legacy cardinality estimator
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (USE HINT ('FORCE_LEGACY_CARDINALITY_ESTIMATION'));

-- Force default cardinality estimator (new CE)
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (USE HINT ('FORCE_DEFAULT_CARDINALITY_ESTIMATION'));

-- Disable parameter sniffing for this query
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (USE HINT ('DISABLE_PARAMETER_SNIFFING'));

-- Enable optimized nested loop (batch sort before key lookup)
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (USE HINT ('ENABLE_QUERY_OPTIMIZER_HOTFIXES'));

-- Disable batch mode processing (force row mode)
SELECT CustomerID, SUM(TotalAmount)
FROM Sales.Orders
GROUP BY CustomerID
OPTION (USE HINT ('DISALLOW_BATCH_MODE'));

-- Assume minimum selectivity for LIKE predicates
SELECT * FROM Sales.Orders
WHERE CustomerName LIKE '%smith%'
OPTION (USE HINT ('ASSUME_MIN_SELECTIVITY_FOR_FILTER_ESTIMATES'));

-- Available USE HINT names (partial list):
-- FORCE_LEGACY_CARDINALITY_ESTIMATION
-- FORCE_DEFAULT_CARDINALITY_ESTIMATION
-- DISABLE_PARAMETER_SNIFFING
-- ENABLE_QUERY_OPTIMIZER_HOTFIXES
-- ASSUME_MIN_SELECTIVITY_FOR_FILTER_ESTIMATES
-- ASSUME_FULL_INDEPENDENCE_FOR_FILTER_ESTIMATES
-- ASSUME_PARTIAL_CORRELATION_FOR_FILTER_ESTIMATES
-- DISABLE_OPTIMIZER_ROWGOAL
-- ENABLE_HIST_AMENDMENT_FOR_ASC_KEYS
-- DISALLOW_BATCH_MODE
-- FORCE_LEGACY_CARDINALITY_ESTIMATION
-- QUERY_OPTIMIZER_COMPATIBILITY_LEVEL_n (110, 120, 130, 140, 150, 160)

-- Combine multiple USE HINT values
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (USE HINT ('DISABLE_PARAMETER_SNIFFING', 'ENABLE_QUERY_OPTIMIZER_HOTFIXES'));
```

## QUERYTRACEON for Per-Query Trace Flags

```sql
-- Apply trace flags to a single query (requires sysadmin or plan guide)
-- Common performance-related trace flags:

-- TF 4199: Enable all optimizer hotfixes
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (QUERYTRACEON 4199);

-- TF 9481: Force legacy cardinality estimator (pre-2014)
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (QUERYTRACEON 9481);

-- TF 2312: Force new cardinality estimator (2014+)
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (QUERYTRACEON 2312);

-- TF 8649: Force parallel plan (even for low-cost queries)
SELECT COUNT(*) FROM Sales.Orders
OPTION (QUERYTRACEON 8649);

-- TF 8690: Disable batch mode processing
SELECT CustomerID, COUNT(*)
FROM Sales.Orders
GROUP BY CustomerID
OPTION (QUERYTRACEON 8690);

-- Combine multiple trace flags
SELECT * FROM Sales.Orders WHERE CustomerID = @cid
OPTION (QUERYTRACEON 4199, QUERYTRACEON 9481);

-- NOTE: USE HINT is preferred over QUERYTRACEON when available (2016 SP1+)
-- USE HINT does not require sysadmin and is more readable
```

## Best Practices

- Treat hints as a last resort after updating statistics, creating indexes, and analyzing the root cause of poor plan choices
- Document every hint with a comment explaining WHY it was added, WHEN it was added, and WHAT ticket or investigation led to it
- Prefer OPTION (RECOMPILE) or OPTIMIZE FOR UNKNOWN over hard-coded OPTIMIZE FOR values -- they adapt better to data changes
- Use per-join hints (INNER HASH JOIN) rather than query-level join hints (OPTION (HASH JOIN)) when only one join needs overriding
- Prefer Query Store hints (2022) or plan guides over embedding hints in application code -- they can be managed without code deployments
- Use USE HINT named hints instead of QUERYTRACEON trace flags whenever possible -- they are more readable and do not require sysadmin
- Periodically review all hints and plan guides to verify they are still beneficial -- data distribution changes can make hints counterproductive
- Test hint effectiveness by comparing actual execution plans with and without the hint
- Avoid NOLOCK (READ UNCOMMITTED) as a blanket performance fix -- it can return incorrect data and cause errors on schema changes
- Use READPAST instead of NOLOCK for queue-processing patterns where skipping locked rows is acceptable

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Applying NOLOCK everywhere as a "performance boost" | Dirty reads, phantom reads, incorrect aggregations, scan errors on page splits | Remove NOLOCK; use proper isolation levels (RCSI/SNAPSHOT) for non-blocking reads |
| Using query-level HASH JOIN when only one join needs it | Forces ALL joins to hash, even when nested loops or merge would be better | Use per-join hints: INNER HASH JOIN on the specific join |
| Adding INDEX hints and forgetting when the index is renamed/dropped | Query fails with error or silently uses a worse plan | Use FORCESEEK instead of specific index names; monitor for hint failures |
| Setting MAXDOP 1 to "fix" CXPACKET waits | Eliminates parallelism for queries that genuinely benefit from it | Investigate root cause; tune cost threshold; use MAXDOP 1 only for specific queries |
| Using FORCE ORDER without understanding join order implications | Locks in a suboptimal join order; optimizer cannot adapt to data changes | Only use when you have verified the join order is consistently better |
| Leaving OPTIMIZE FOR values that no longer represent typical data | Plan optimized for a stale parameter value; bad for current data distribution | Use OPTIMIZE FOR UNKNOWN instead; review specific values periodically |
| Applying hints without documenting the reason | Future DBAs cannot evaluate if the hint is still needed; tech debt accumulates | Always add comments with ticket numbers, dates, and rationale |

## SQL Server Version Notes

**SQL Server 2016**: USE HINT syntax introduced (SP1). DISABLE_PARAMETER_SNIFFING, FORCE_LEGACY_CARDINALITY_ESTIMATION available as named hints. MIN_GRANT_PERCENT and MAX_GRANT_PERCENT query hints added.

**SQL Server 2017**: Additional USE HINT values. QUERY_OPTIMIZER_COMPATIBILITY_LEVEL_n hint for per-query compatibility level override. Adaptive join hints interact with batch mode adaptive joins.

**SQL Server 2019**: DISABLE_BATCH_MODE_ADAPTIVE_JOINS hint. DISABLE_INTERLEAVED_EXECUTION_TVF hint. DISABLE_TSQL_SCALAR_UDF_INLINING hint. More granular control over Intelligent Query Processing features.

**SQL Server 2022**: Query Store hints (sp_query_store_set_hints) -- apply hints via Query Store without code changes. TABLE HINT syntax in Query Store hints. DISABLE_OPTIMIZED_PLAN_FORCING hint. Parameter Sensitive Plan (PSP) optimization reduces the need for RECOMPILE hints. DOP feedback reduces need for MAXDOP hints.

## Sources

- [Query Hints (T-SQL Reference)](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query)
- [Table Hints (T-SQL Reference)](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table)
- [Join Hints (T-SQL Reference)](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-join)
- [Plan Guides](https://learn.microsoft.com/en-us/sql/relational-databases/performance/plan-guides)
- [Query Store Hints (2022)](https://learn.microsoft.com/en-us/sql/relational-databases/performance/query-store-hints)
- [USE HINT Names](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query#use_hint)
