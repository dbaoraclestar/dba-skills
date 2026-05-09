# SQL Tuning in SQL Server — Query Rewriting, Hints, and Optimization Techniques

## Overview

SQL tuning in SQL Server involves rewriting queries to leverage the optimizer's strengths, applying targeted hints when the optimizer's defaults are suboptimal, and ensuring that WHERE clauses, JOINs, and expressions are structured to allow index seeks. The goal is to reduce logical reads, eliminate unnecessary scans, and ensure stable, predictable execution plans.

SQL Server's query optimizer is cost-based: it evaluates multiple candidate plans and selects the one with the lowest estimated cost based on statistics, indexes, and cardinality estimates. When the optimizer makes a poor choice (due to stale statistics, parameter sniffing, or complex query shapes), targeted interventions like query rewrites, hints, or plan guides can correct the behavior without restructuring the application.

This reference covers the most impactful tuning techniques: making predicates sargable, eliminating implicit conversions, rewriting subqueries as JOINs or APPLY, mitigating parameter sniffing, using temp tables for plan separation, and applying the right hint for the right situation. Each technique includes the diagnostic pattern (how to identify the problem) and the fix.

## Key Concepts

- **Sargability** — A predicate is sargable (Search ARGument ABLE) if it can be evaluated using an index seek. Functions, arithmetic, or type conversions on indexed columns break sargability.
- **Implicit conversion** — When SQL Server automatically converts one data type to another in a comparison. This often prevents index seeks and is visible in execution plans as a CONVERT_IMPLICIT warning.
- **Cardinality estimation** — The optimizer's estimate of how many rows each operation will produce. Bad estimates cascade through the plan, causing wrong join types, missing parallelism, or memory grant problems.
- **Parameter sniffing** — The optimizer compiles a plan using the parameter values from the first execution. Atypical first values create plans that perform poorly for subsequent calls.
- **Plan regression** — A previously fast query becomes slow due to a plan change (statistics update, index change, or recompile).
- **Seek vs scan** — An index seek navigates the B-tree to find specific rows (O(log n)). A scan reads all leaf pages (O(n)). Seeks are generally preferred for selective queries.

## Making WHERE Clauses Sargable

```sql
-- PROBLEM: Function on indexed column prevents index seek
-- BAD: table scan — YEAR() wraps the column
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2025;

-- GOOD: range predicate enables index seek on OrderDate
SELECT * FROM Orders 
WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';

-- BAD: CONVERT on indexed column
SELECT * FROM Orders WHERE CONVERT(DATE, OrderDate) = '2025-06-15';

-- GOOD: range on the original column
SELECT * FROM Orders 
WHERE OrderDate >= '2025-06-15' AND OrderDate < '2025-06-16';

-- BAD: arithmetic on indexed column
SELECT * FROM Products WHERE Price * 1.1 > 100;

-- GOOD: arithmetic on the constant side
SELECT * FROM Products WHERE Price > 100 / 1.1;

-- BAD: ISNULL on indexed column (prevents seek)
SELECT * FROM Orders WHERE ISNULL(ShipDate, '1900-01-01') > '2025-01-01';

-- GOOD: explicit IS NOT NULL + comparison
SELECT * FROM Orders WHERE ShipDate IS NOT NULL AND ShipDate > '2025-01-01';

-- BAD: LIKE with leading wildcard (scan)
SELECT * FROM Customers WHERE Email LIKE '%@gmail.com';

-- GOOD: computed column with reverse + index (for suffix search)
ALTER TABLE Customers ADD EmailReversed AS REVERSE(Email) PERSISTED;
CREATE INDEX IX_EmailReversed ON Customers(EmailReversed);
SELECT * FROM Customers WHERE REVERSE(Email) LIKE REVERSE('%@gmail.com');
-- Or use full-text search for arbitrary substring matching

-- BAD: LTRIM/RTRIM on indexed column
SELECT * FROM Employees WHERE LTRIM(RTRIM(Name)) = 'Alice';

-- GOOD: fix the data quality or use computed column
SELECT * FROM Employees WHERE Name = 'Alice';
```

## Eliminating Implicit Conversions

```sql
-- Implicit conversion: comparing NVARCHAR parameter to VARCHAR column
-- The column gets converted (widened), preventing index seek

-- PROBLEM: @name is NVARCHAR, column is VARCHAR
DECLARE @name NVARCHAR(100) = N'Alice';
SELECT * FROM Employees WHERE Name = @name;  -- Name is VARCHAR(100)
-- Execution plan shows CONVERT_IMPLICIT(NVARCHAR, Name) — scans index

-- FIX 1: Match parameter type to column type
DECLARE @name VARCHAR(100) = 'Alice';
SELECT * FROM Employees WHERE Name = @name;  -- index seek

-- FIX 2: In application code, use correct SqlDbType
-- C#: command.Parameters.Add("@name", SqlDbType.VarChar, 100).Value = "Alice";
-- Not: command.Parameters.AddWithValue("@name", "Alice");  // sends as NVARCHAR

-- Common implicit conversion issues:
-- INT vs BIGINT: smaller type is converted up (usually fine)
-- VARCHAR vs NVARCHAR: VARCHAR column converted to NVARCHAR (breaks seeks)
-- INT vs VARCHAR: VARCHAR column converted to INT (breaks seeks, can error)
-- DATE vs DATETIME: usually fine, DATE widened to DATETIME

-- Detect implicit conversions in execution plans (XML plan)
SELECT 
    qs.query_hash,
    st.text,
    qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
WHERE qp.query_plan.exist('//Warnings/PlanAffectingConvert') = 1;
```

## Subquery to JOIN Rewrites

```sql
-- Correlated subquery in SELECT (scalar subquery) — executes once per row
-- BAD: repeated subquery execution
SELECT 
    o.OrderId,
    o.CustomerId,
    o.Amount,
    (SELECT c.Name FROM Customers c WHERE c.CustomerId = o.CustomerId) AS CustomerName,
    (SELECT SUM(oi.Quantity) FROM OrderItems oi WHERE oi.OrderId = o.OrderId) AS TotalItems
FROM Orders o;

-- GOOD: JOIN (single pass)
SELECT 
    o.OrderId,
    o.CustomerId,
    o.Amount,
    c.Name AS CustomerName,
    items.TotalItems
FROM Orders o
JOIN Customers c ON o.CustomerId = c.CustomerId
LEFT JOIN (
    SELECT OrderId, SUM(Quantity) AS TotalItems
    FROM OrderItems
    GROUP BY OrderId
) items ON o.OrderId = items.OrderId;

-- Correlated subquery in WHERE — rewrite to JOIN or EXISTS
-- BAD: correlated subquery
SELECT * FROM Employees e
WHERE e.Salary > (SELECT AVG(Salary) FROM Employees WHERE Department = e.Department);

-- GOOD: JOIN with pre-computed aggregation
SELECT e.*
FROM Employees e
JOIN (
    SELECT Department, AVG(Salary) AS AvgSalary
    FROM Employees
    GROUP BY Department
) da ON e.Department = da.Department
WHERE e.Salary > da.AvgSalary;

-- NOT IN subquery with NULLs — dangerous
-- BAD: returns 0 rows if ANY CustomerId in Orders is NULL
SELECT * FROM Customers
WHERE CustomerId NOT IN (SELECT CustomerId FROM Orders);

-- GOOD: NOT EXISTS handles NULLs correctly
SELECT * FROM Customers c
WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId);
```

## Correlated Subquery to APPLY

```sql
-- CROSS APPLY is ideal for top-N-per-group (replaces correlated subquery)

-- BAD: correlated subquery with TOP (may use nested loops with poor estimates)
SELECT 
    c.CustomerId,
    c.Name,
    (SELECT TOP 1 o.OrderDate FROM Orders o 
     WHERE o.CustomerId = c.CustomerId ORDER BY o.OrderDate DESC) AS LastOrderDate
FROM Customers c;

-- GOOD: CROSS APPLY — optimizer can choose optimal join strategy
SELECT 
    c.CustomerId,
    c.Name,
    lastOrder.OrderDate AS LastOrderDate,
    lastOrder.Amount AS LastOrderAmount
FROM Customers c
CROSS APPLY (
    SELECT TOP 1 OrderDate, Amount
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) lastOrder;

-- OUTER APPLY for LEFT JOIN semantics (include customers with no orders)
SELECT 
    c.CustomerId,
    c.Name,
    lastOrder.OrderDate
FROM Customers c
OUTER APPLY (
    SELECT TOP 1 OrderDate
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    ORDER BY o.OrderDate DESC
) lastOrder;
```

## Scalar UDF Elimination

```sql
-- Scalar UDFs prevent parallelism and execute row-by-row (pre-2019)
-- BAD: scalar UDF in SELECT
CREATE FUNCTION dbo.fn_GetFullName(@EmpId INT) RETURNS NVARCHAR(200)
AS
BEGIN
    DECLARE @result NVARCHAR(200);
    SELECT @result = FirstName + ' ' + LastName FROM Employees WHERE EmployeeId = @EmpId;
    RETURN @result;
END;
GO

SELECT OrderId, dbo.fn_GetFullName(EmployeeId) AS EmpName FROM Orders;
-- Executes fn_GetFullName once per row, forces serial plan

-- GOOD: inline the logic as a JOIN
SELECT o.OrderId, e.FirstName + ' ' + e.LastName AS EmpName
FROM Orders o
JOIN Employees e ON o.EmployeeId = e.EmployeeId;

-- Or use an inline table-valued function (optimizer can inline these)
CREATE FUNCTION dbo.fn_GetFullNameInline(@EmpId INT)
RETURNS TABLE
AS
RETURN (
    SELECT FirstName + ' ' + LastName AS FullName
    FROM Employees
    WHERE EmployeeId = @EmpId
);
GO

SELECT o.OrderId, fn.FullName
FROM Orders o
CROSS APPLY dbo.fn_GetFullNameInline(o.EmployeeId) fn;

-- SQL Server 2019+: scalar UDF inlining (automatic for eligible functions)
-- Check eligibility:
SELECT o.name, m.is_inlineable
FROM sys.sql_modules m
JOIN sys.objects o ON m.object_id = o.object_id
WHERE o.type = 'FN';
```

## Parameter Sniffing Mitigation

```sql
-- Detect parameter sniffing: same query, vastly different performance

-- Strategy 1: OPTION(RECOMPILE) — best for infrequent queries
SELECT * FROM Orders WHERE Status = @Status
OPTION (RECOMPILE);

-- Strategy 2: OPTIMIZE FOR UNKNOWN — uses average statistics
SELECT * FROM Orders WHERE Status = @Status
OPTION (OPTIMIZE FOR (@Status UNKNOWN));

-- Strategy 3: OPTIMIZE FOR specific value — when you know the typical value
SELECT * FROM Orders WHERE Status = @Status
OPTION (OPTIMIZE FOR (@Status = 'Active'));

-- Strategy 4: Local variable trick (hides sniffed value from optimizer)
CREATE OR ALTER PROCEDURE dbo.usp_OrdersByStatus @Status NVARCHAR(20)
AS
BEGIN
    DECLARE @s NVARCHAR(20) = @Status;
    SELECT * FROM Orders WHERE Status = @s;
END;

-- Strategy 5: Plan guides (non-invasive, no code change required)
EXEC sp_create_plan_guide
    @name = N'PG_OrdersByStatus',
    @stmt = N'SELECT * FROM Orders WHERE Status = @Status',
    @type = N'SQL',
    @module_or_batch = NULL,
    @hints = N'OPTION(OPTIMIZE FOR (@Status UNKNOWN))';

-- SQL Server 2022: Parameter Sensitive Plan optimization (PSP)
-- Optimizer automatically creates multiple plans for different parameter ranges
-- Check if PSP is active:
SELECT * FROM sys.query_store_plan_forcing_locations;
```

## Temp Table Materialization for Plan Separation

```sql
-- Complex queries sometimes get bad plans because the optimizer
-- makes poor cardinality estimates through multiple joins

-- PROBLEM: optimizer estimates 1 row from CTE, reality is 50,000
;WITH HeavyFilter AS (
    SELECT CustomerId, SUM(Amount) AS TotalAmount
    FROM Orders
    WHERE OrderDate >= '2024-01-01'
    GROUP BY CustomerId
    HAVING SUM(Amount) > 1000
)
SELECT c.Name, hf.TotalAmount, od.ProductName
FROM HeavyFilter hf
JOIN Customers c ON hf.CustomerId = c.CustomerId
JOIN OrderDetails od ON hf.CustomerId = od.CustomerId;

-- FIX: materialize intermediate result in #temp
-- SQL Server creates statistics on #temp tables, giving accurate estimates
CREATE TABLE #HighValueCustomers (
    CustomerId INT PRIMARY KEY,
    TotalAmount DECIMAL(18,2)
);

INSERT INTO #HighValueCustomers
SELECT CustomerId, SUM(Amount)
FROM Orders
WHERE OrderDate >= '2024-01-01'
GROUP BY CustomerId
HAVING SUM(Amount) > 1000;

-- Now the optimizer knows the exact row count for the join
SELECT c.Name, hv.TotalAmount, od.ProductName
FROM #HighValueCustomers hv
JOIN Customers c ON hv.CustomerId = c.CustomerId
JOIN OrderDetails od ON hv.CustomerId = od.CustomerId;

DROP TABLE #HighValueCustomers;
```

## Table and Query Hints

```sql
-- NOLOCK (READ UNCOMMITTED): dirty reads, skipped rows possible
-- Use ONLY for rough estimates, monitoring, never for business logic
SELECT COUNT(*) FROM Orders WITH (NOLOCK);  -- approximate count is OK
-- Never use for financial queries, status checks, or decision-making

-- FORCESEEK: force index seek (when optimizer wrongly chooses scan)
SELECT * FROM Orders WITH (FORCESEEK)
WHERE CustomerId = @custId;

-- FORCESEEK on specific index
SELECT * FROM Orders WITH (FORCESEEK, INDEX(IX_Orders_CustomerId))
WHERE CustomerId = @custId;

-- FORCESCAN: force scan (rare, when seek + lookup is more expensive than scan)
SELECT * FROM Orders WITH (FORCESCAN)
WHERE Status IN ('A','B','C','D','E','F','G');

-- INDEX hint: force use of specific index
SELECT * FROM Orders WITH (INDEX(IX_Orders_OrderDate))
WHERE OrderDate >= '2025-01-01';

-- QUERY hints
-- Force parallelism or serial execution
SELECT * FROM LargeTable WHERE col = @val
OPTION (MAXDOP 8);  -- up to 8 threads

SELECT * FROM SmallLookup WHERE col = @val
OPTION (MAXDOP 1);  -- force serial (avoid parallel overhead)

-- Memory grant hint (fix spills to tempdb)
SELECT * FROM Orders ORDER BY Amount DESC
OPTION (MIN_GRANT_PERCENT = 5, MAX_GRANT_PERCENT = 25);

-- HASH/MERGE/LOOP join hints
SELECT o.*, c.Name
FROM Orders o
INNER HASH JOIN Customers c ON o.CustomerId = c.CustomerId;

-- USE HINT for named optimizer behaviors
SELECT * FROM Orders WHERE Status = @s
OPTION (USE HINT('ENABLE_PARALLEL_PLAN_PREFERENCE'));

SELECT * FROM Orders
OPTION (USE HINT('FORCE_LEGACY_CARDINALITY_ESTIMATION'));  -- pre-2014 CE
```

## Computed Columns for Indexing

```sql
-- When queries filter on expressions, create a computed column + index

-- Scenario: frequent queries filter on UPPER(Email)
ALTER TABLE Customers
ADD EmailUpper AS UPPER(Email) PERSISTED;

CREATE INDEX IX_Customers_EmailUpper ON Customers(EmailUpper);

-- Now this query seeks on the index:
SELECT * FROM Customers WHERE UPPER(Email) = 'ALICE@EXAMPLE.COM';

-- Scenario: filter on JSON property
ALTER TABLE Products
ADD Color AS CAST(JSON_VALUE(Attributes, '$.color') AS NVARCHAR(50)) PERSISTED;
CREATE INDEX IX_Products_Color ON Products(Color);

-- Scenario: filter on computed date part
ALTER TABLE Orders
ADD OrderYear AS YEAR(OrderDate) PERSISTED;
CREATE INDEX IX_Orders_Year ON Orders(OrderYear);
-- But prefer the range predicate approach instead when possible:
-- WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'
```

## Diagnosing Query Performance

```sql
-- Check execution statistics for a query
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

SELECT * FROM Orders WHERE CustomerId = 42;
-- Look for: logical reads, physical reads, CPU time, elapsed time

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;

-- Find most expensive cached queries by logical reads
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_logical_reads,
    qs.execution_count,
    qs.total_logical_reads,
    qs.total_worker_time / 1000 AS total_cpu_ms,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset WHEN -1 THEN DATALENGTH(st.text)
          ELSE qs.statement_end_offset END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_logical_reads DESC;

-- Check for missing indexes suggested by the optimizer
SELECT 
    d.statement AS table_name,
    d.equality_columns,
    d.inequality_columns,
    d.included_columns,
    s.avg_user_impact AS pct_improvement,
    s.user_seeks + s.user_scans AS potential_uses,
    ROUND(s.avg_total_user_cost * s.avg_user_impact * (s.user_seeks + s.user_scans) / 100.0, 2) AS improvement_score
FROM sys.dm_db_missing_index_details d
JOIN sys.dm_db_missing_index_groups g ON d.index_handle = g.index_handle
JOIN sys.dm_db_missing_index_group_stats s ON g.index_group_handle = s.group_handle
WHERE d.database_id = DB_ID()
ORDER BY improvement_score DESC;
```

## Best Practices

- Always check sargability first: ensure WHERE clause predicates do not wrap indexed columns in functions or expressions.
- Fix implicit conversions by matching parameter types to column types in both T-SQL and application code.
- Prefer `NOT EXISTS` over `NOT IN` with subqueries to avoid NULL-related correctness bugs and improve plan quality.
- Use CROSS APPLY instead of correlated subqueries for top-N-per-group patterns.
- Replace scalar UDFs in SELECT/WHERE with inline logic, JOINs, or inline table-valued functions.
- Address parameter sniffing proactively: OPTION(RECOMPILE) for variable queries, OPTIMIZE FOR for predictable distributions.
- Materialize complex intermediate results in #temp tables to give the optimizer accurate statistics.
- Use hints as a last resort after rewriting the query and ensuring statistics/indexes are current.
- Turn on SET STATISTICS IO/TIME to measure logical reads, not just elapsed time.
- Monitor missing index DMVs regularly, but evaluate each suggestion critically — not all suggestions should be implemented.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Function on indexed column in WHERE | Prevents index seek, causes full scan | Move function to the constant side or use computed column |
| AddWithValue in C# with string parameter | Sends NVARCHAR, implicit conversion on VARCHAR column | Use `Add()` with explicit SqlDbType matching column type |
| Using NOLOCK for business-critical reads | Dirty reads, skipped rows, duplicate rows | Use RCSI (READ_COMMITTED_SNAPSHOT) for non-blocking reads |
| Over-hinting (every query has INDEX/FORCESEEK) | Plans break when data distribution changes or indexes are modified | Use hints only as a last resort, document why |
| Ignoring parameter sniffing | Intermittent slow queries after plan cache flush | Add OPTION(RECOMPILE) or OPTIMIZE FOR UNKNOWN |
| CTEs as "materialized views" | CTEs re-execute each reference, no statistics | Use #temp tables for multi-reference intermediate results |
| Correlated scalar subqueries in SELECT list | Row-by-row execution, prevents parallelism | Rewrite as JOIN or CROSS APPLY |

## SQL Server Version Notes

- **SQL Server 2016** — Query Store (track plan regressions), live query statistics, new cardinality estimator improvements. Batch mode for columnstore indexes.
- **SQL Server 2019** — Scalar UDF inlining, table variable deferred compilation, batch mode on rowstore, adaptive joins, memory grant feedback. Intelligent Query Processing (IQP) suite.
- **SQL Server 2022** — Parameter Sensitive Plan (PSP) optimization, cardinality estimation feedback, degree-of-parallelism feedback, optimized plan forcing, memory grant feedback percentile-based. IQP v2.
- **Azure SQL Database** — Same IQP features as 2022. Automatic tuning (auto-create/drop indexes, force plans). Automatic plan correction.
- **Compatibility level matters** — IQP features require specific compatibility levels (150 for 2019, 160 for 2022). Check: `SELECT compatibility_level FROM sys.databases WHERE name = DB_NAME()`.

## Sources

- https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing
- https://learn.microsoft.com/en-us/sql/relational-databases/query-processing-architecture-guide
- https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-query
- https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-exec-query-stats-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitivity-plan-optimization
- https://learn.microsoft.com/en-us/sql/relational-databases/indexes/indexes
