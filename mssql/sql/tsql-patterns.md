# T-SQL Patterns — Common Query Patterns and Techniques for SQL Server

## Overview

T-SQL (Transact-SQL) is Microsoft's extension of the SQL standard, providing procedural programming constructs, error handling, and SQL Server-specific features on top of standard SQL. Mastering common T-SQL patterns is essential for writing efficient, maintainable queries against SQL Server databases.

This reference covers the most frequently used query patterns in production SQL Server environments: pagination with OFFSET/FETCH, upsert operations with MERGE, window functions for analytics, Common Table Expressions (CTEs) for query organization, CROSS APPLY/OUTER APPLY for correlated subqueries, PIVOT/UNPIVOT for data reshaping, and modern string functions like STRING_AGG.

These patterns apply across SQL Server 2016, 2019, and 2022, with version-specific alternatives noted where older versions lack certain features. Understanding when to use each pattern and its performance characteristics is the difference between a query that scans millions of rows and one that completes in milliseconds.

## Key Concepts

- **Window functions** — Perform calculations across a set of rows related to the current row without collapsing the result set. Use OVER() clause with PARTITION BY and ORDER BY.
- **CTE (Common Table Expression)** — A named, temporary result set defined with WITH that exists for the scope of a single statement. Improves readability and enables recursion.
- **APPLY operator** — Evaluates a table-valued expression for each row from the outer table. CROSS APPLY (inner join semantics) and OUTER APPLY (left join semantics).
- **MERGE** — Single statement that performs INSERT, UPDATE, and/or DELETE based on whether rows match between source and target tables.
- **Sargability** — A WHERE clause predicate that allows the query optimizer to use an index seek. Functions on indexed columns generally break sargability.

## Pagination with OFFSET/FETCH

```sql
-- Standard pagination (SQL Server 2012+)
SELECT EmployeeId, Name, Department, Salary
FROM Employees
ORDER BY Name  -- ORDER BY is required for OFFSET/FETCH
OFFSET 20 ROWS          -- skip first 20 rows
FETCH NEXT 10 ROWS ONLY; -- return 10 rows

-- Parameterized pagination for application use
DECLARE @PageNumber INT = 3;
DECLARE @PageSize INT = 25;

SELECT EmployeeId, Name, Department, Salary
FROM Employees
ORDER BY EmployeeId
OFFSET (@PageNumber - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;

-- Pagination with total count (two queries — more efficient than COUNT in same query)
-- Query 1: Get total count
SELECT COUNT(*) AS TotalRows FROM Employees WHERE Department = 'Engineering';

-- Query 2: Get page
SELECT EmployeeId, Name, Salary
FROM Employees
WHERE Department = 'Engineering'
ORDER BY EmployeeId
OFFSET 0 ROWS FETCH NEXT 25 ROWS ONLY;

-- Keyset pagination (faster for large offsets)
-- Instead of OFFSET, filter by the last seen key
SELECT TOP 25 EmployeeId, Name, Salary
FROM Employees
WHERE EmployeeId > @LastSeenId  -- last EmployeeId from previous page
ORDER BY EmployeeId;

-- Legacy pagination with ROW_NUMBER (pre-2012)
;WITH Numbered AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY Name) AS RowNum
    FROM Employees
)
SELECT EmployeeId, Name, Department, Salary
FROM Numbered
WHERE RowNum BETWEEN 21 AND 30;
```

## Upsert with MERGE

```sql
-- MERGE: insert if not exists, update if exists
MERGE INTO Products AS target
USING (VALUES (@ProductId, @Name, @Price, @Stock)) 
    AS source (ProductId, Name, Price, Stock)
ON target.ProductId = source.ProductId
WHEN MATCHED THEN
    UPDATE SET 
        Name = source.Name,
        Price = source.Price,
        Stock = source.Stock,
        ModifiedDate = GETUTCDATE()
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ProductId, Name, Price, Stock, CreatedDate)
    VALUES (source.ProductId, source.Name, source.Price, source.Stock, GETUTCDATE())
OUTPUT $action, INSERTED.ProductId, INSERTED.Name;
-- Always end MERGE with semicolon (required)

-- Bulk upsert from staging table
MERGE INTO Products AS target
USING StagingProducts AS source
ON target.ProductId = source.ProductId
WHEN MATCHED AND (
    target.Price <> source.Price OR 
    target.Stock <> source.Stock
) THEN
    UPDATE SET 
        Price = source.Price,
        Stock = source.Stock,
        ModifiedDate = GETUTCDATE()
WHEN NOT MATCHED BY TARGET THEN
    INSERT (ProductId, Name, Price, Stock, CreatedDate)
    VALUES (source.ProductId, source.Name, source.Price, source.Stock, GETUTCDATE())
WHEN NOT MATCHED BY SOURCE THEN
    DELETE
OUTPUT $action, COALESCE(INSERTED.ProductId, DELETED.ProductId) AS ProductId;

-- Simpler upsert alternative (no MERGE, avoids MERGE bugs)
BEGIN TRANSACTION;
    UPDATE Products SET Price = @Price, Stock = @Stock WHERE ProductId = @ProductId;
    IF @@ROWCOUNT = 0
        INSERT INTO Products (ProductId, Name, Price, Stock) VALUES (@ProductId, @Name, @Price, @Stock);
COMMIT;
```

## Window Functions

```sql
-- ROW_NUMBER: unique sequential number per partition
SELECT 
    EmployeeId,
    Name,
    Department,
    Salary,
    ROW_NUMBER() OVER (PARTITION BY Department ORDER BY Salary DESC) AS SalaryRank
FROM Employees;

-- RANK and DENSE_RANK: handle ties differently
SELECT 
    Name,
    Department,
    Salary,
    RANK()       OVER (ORDER BY Salary DESC) AS Rank,       -- gaps after ties (1,2,2,4)
    DENSE_RANK() OVER (ORDER BY Salary DESC) AS DenseRank   -- no gaps (1,2,2,3)
FROM Employees;

-- Top N per group (e.g., top 3 earners per department)
;WITH Ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY Department ORDER BY Salary DESC) AS rn
    FROM Employees
)
SELECT * FROM Ranked WHERE rn <= 3;

-- LAG and LEAD: access previous/next row values
SELECT 
    OrderDate,
    Amount,
    LAG(Amount, 1, 0)  OVER (ORDER BY OrderDate) AS PreviousAmount,
    LEAD(Amount, 1, 0) OVER (ORDER BY OrderDate) AS NextAmount,
    Amount - LAG(Amount, 1) OVER (ORDER BY OrderDate) AS ChangeFromPrevious
FROM DailySales;

-- Running total
SELECT 
    OrderDate,
    Amount,
    SUM(Amount) OVER (ORDER BY OrderDate ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS RunningTotal
FROM DailySales;

-- Moving average (last 7 rows)
SELECT 
    OrderDate,
    Amount,
    AVG(Amount) OVER (ORDER BY OrderDate ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS MovingAvg7Day
FROM DailySales;

-- FIRST_VALUE and LAST_VALUE
SELECT 
    Department,
    Name,
    Salary,
    FIRST_VALUE(Name) OVER (PARTITION BY Department ORDER BY Salary DESC) AS HighestPaid,
    LAST_VALUE(Name)  OVER (PARTITION BY Department ORDER BY Salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS LowestPaid
FROM Employees;

-- Conditional aggregation with window functions
SELECT 
    Department,
    COUNT(*) OVER (PARTITION BY Department) AS DeptCount,
    SUM(CASE WHEN Salary > 100000 THEN 1 ELSE 0 END) OVER (PARTITION BY Department) AS HighEarners,
    AVG(Salary) OVER (PARTITION BY Department) AS AvgSalary
FROM Employees;
```

## CTEs and Recursive CTEs

```sql
-- Basic CTE for readability
;WITH DepartmentStats AS (
    SELECT 
        Department,
        COUNT(*) AS EmpCount,
        AVG(Salary) AS AvgSalary,
        MAX(Salary) AS MaxSalary
    FROM Employees
    GROUP BY Department
)
SELECT 
    e.Name,
    e.Salary,
    ds.AvgSalary AS DeptAvgSalary,
    e.Salary - ds.AvgSalary AS AboveAvg
FROM Employees e
JOIN DepartmentStats ds ON e.Department = ds.Department
WHERE e.Salary > ds.AvgSalary;

-- Multiple CTEs
;WITH 
ActiveOrders AS (
    SELECT CustomerId, COUNT(*) AS OrderCount, SUM(Amount) AS TotalAmount
    FROM Orders
    WHERE Status = 'Active'
    GROUP BY CustomerId
),
CustomerTier AS (
    SELECT CustomerId, OrderCount, TotalAmount,
        CASE 
            WHEN TotalAmount > 10000 THEN 'Gold'
            WHEN TotalAmount > 5000 THEN 'Silver'
            ELSE 'Bronze'
        END AS Tier
    FROM ActiveOrders
)
SELECT c.Name, ct.Tier, ct.TotalAmount
FROM Customers c
JOIN CustomerTier ct ON c.CustomerId = ct.CustomerId;

-- Recursive CTE: org chart / hierarchy traversal
;WITH OrgChart AS (
    -- Anchor: top-level managers (no manager)
    SELECT EmployeeId, Name, ManagerId, 0 AS Level,
           CAST(Name AS NVARCHAR(MAX)) AS HierarchyPath
    FROM Employees
    WHERE ManagerId IS NULL
    
    UNION ALL
    
    -- Recursive: each employee under their manager
    SELECT e.EmployeeId, e.Name, e.ManagerId, oc.Level + 1,
           CAST(oc.HierarchyPath + ' > ' + e.Name AS NVARCHAR(MAX))
    FROM Employees e
    JOIN OrgChart oc ON e.ManagerId = oc.EmployeeId
)
SELECT * FROM OrgChart
ORDER BY HierarchyPath
OPTION (MAXRECURSION 100);  -- default is 100, max is 32767, 0 = unlimited

-- Recursive CTE: generate date series
;WITH DateSeries AS (
    SELECT CAST('2025-01-01' AS DATE) AS dt
    UNION ALL
    SELECT DATEADD(DAY, 1, dt) FROM DateSeries WHERE dt < '2025-12-31'
)
SELECT dt FROM DateSeries
OPTION (MAXRECURSION 0);
```

## CROSS APPLY and OUTER APPLY

```sql
-- CROSS APPLY: like INNER JOIN with a correlated subquery / table-valued function
-- Get the 3 most recent orders for each customer
SELECT c.CustomerId, c.Name, o.OrderId, o.OrderDate, o.Amount
FROM Customers c
CROSS APPLY (
    SELECT TOP 3 OrderId, OrderDate, Amount
    FROM Orders
    WHERE CustomerId = c.CustomerId
    ORDER BY OrderDate DESC
) o;

-- OUTER APPLY: like LEFT JOIN — includes customers with no matching orders
SELECT c.CustomerId, c.Name, o.OrderId, o.OrderDate, o.Amount
FROM Customers c
OUTER APPLY (
    SELECT TOP 3 OrderId, OrderDate, Amount
    FROM Orders
    WHERE CustomerId = c.CustomerId
    ORDER BY OrderDate DESC
) o;

-- CROSS APPLY with table-valued function
SELECT c.CustomerId, c.Name, s.TotalAmount, s.OrderCount
FROM Customers c
CROSS APPLY dbo.fn_CustomerSummary(c.CustomerId) s;

-- CROSS APPLY to split/unnest (e.g., comma-separated values)
SELECT e.EmployeeId, e.Name, skill.value AS Skill
FROM Employees e
CROSS APPLY STRING_SPLIT(e.Skills, ',') AS skill;

-- CROSS APPLY for top-1 subquery (more efficient than correlated subquery in SELECT)
SELECT 
    d.DepartmentName,
    topEmp.Name AS HighestPaid,
    topEmp.Salary
FROM Departments d
CROSS APPLY (
    SELECT TOP 1 Name, Salary
    FROM Employees e
    WHERE e.DepartmentId = d.DepartmentId
    ORDER BY Salary DESC
) topEmp;
```

## PIVOT and UNPIVOT

```sql
-- PIVOT: rows to columns
SELECT *
FROM (
    SELECT Department, YEAR(HireDate) AS HireYear, EmployeeId
    FROM Employees
) AS src
PIVOT (
    COUNT(EmployeeId)
    FOR HireYear IN ([2022], [2023], [2024], [2025])
) AS pvt;
-- Result: Department | 2022 | 2023 | 2024 | 2025

-- Dynamic PIVOT (when column values aren't known at compile time)
DECLARE @columns NVARCHAR(MAX), @sql NVARCHAR(MAX);

SELECT @columns = STRING_AGG(QUOTENAME(Department), ',')
FROM (SELECT DISTINCT Department FROM Employees) AS d;

SET @sql = N'
SELECT HireYear, ' + @columns + N'
FROM (
    SELECT Department, YEAR(HireDate) AS HireYear, EmployeeId
    FROM Employees
) AS src
PIVOT (COUNT(EmployeeId) FOR Department IN (' + @columns + N')) AS pvt
ORDER BY HireYear';

EXEC sp_executesql @sql;

-- UNPIVOT: columns to rows
SELECT ProductId, Attribute, Value
FROM (
    SELECT ProductId, 
           CAST(Height AS NVARCHAR(50)) AS Height,
           CAST(Width AS NVARCHAR(50)) AS Width,
           CAST(Weight AS NVARCHAR(50)) AS Weight
    FROM Products
) AS src
UNPIVOT (
    Value FOR Attribute IN (Height, Width, Weight)
) AS unpvt;
```

## String Functions and Aggregation

```sql
-- STRING_AGG (SQL Server 2017+): concatenate values with separator
SELECT 
    Department,
    STRING_AGG(Name, ', ') WITHIN GROUP (ORDER BY Name) AS EmployeeList
FROM Employees
GROUP BY Department;

-- STRING_SPLIT (SQL Server 2016+): split delimited string into rows
SELECT value AS Tag
FROM STRING_SPLIT('sql,server,performance,tuning', ',');

-- STRING_SPLIT with ordinal (SQL Server 2022+)
SELECT value, ordinal
FROM STRING_SPLIT('a,b,c,d', ',', 1)  -- enable_ordinal = 1
ORDER BY ordinal;

-- CONCAT and CONCAT_WS (with separator)
SELECT CONCAT(FirstName, ' ', LastName) AS FullName FROM Employees;
SELECT CONCAT_WS(', ', City, State, ZipCode) AS Address FROM Addresses;
-- CONCAT_WS skips NULLs: CONCAT_WS(', ', 'Seattle', NULL, '98101') = 'Seattle, 98101'

-- Pre-2017 string aggregation with FOR XML PATH
SELECT 
    Department,
    STUFF((
        SELECT ', ' + Name
        FROM Employees e2
        WHERE e2.Department = e1.Department
        ORDER BY Name
        FOR XML PATH(''), TYPE
    ).value('.', 'NVARCHAR(MAX)'), 1, 2, '') AS EmployeeList
FROM Employees e1
GROUP BY Department;
```

## EXISTS vs IN Performance

```sql
-- EXISTS: stops at first match (short-circuits)
-- Generally preferred for correlated subqueries
SELECT c.CustomerId, c.Name
FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerId = c.CustomerId
    AND o.OrderDate >= '2025-01-01'
);

-- IN: evaluates the full subquery result set
-- Better for small, static lists
SELECT * FROM Employees
WHERE Department IN ('Engineering', 'Marketing', 'Sales');

-- IN with subquery (optimizer usually converts to semi-join, same as EXISTS)
SELECT * FROM Employees
WHERE DepartmentId IN (SELECT DepartmentId FROM Departments WHERE Active = 1);

-- NOT EXISTS vs NOT IN: EXISTS handles NULLs correctly
-- NOT IN returns no rows if subquery contains ANY NULL value
-- Always prefer NOT EXISTS over NOT IN with subqueries
SELECT c.CustomerId, c.Name
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId
);

-- TOP WITH TIES: include all rows with same value as last row
SELECT TOP 5 WITH TIES Name, Department, Salary
FROM Employees
ORDER BY Salary DESC;
-- Returns all employees tied at 5th highest salary (may return > 5 rows)
```

## Conditional Aggregation

```sql
-- Count/sum by category in a single query (avoid multiple queries)
SELECT 
    Department,
    COUNT(*) AS TotalEmployees,
    COUNT(CASE WHEN Salary >= 100000 THEN 1 END) AS HighEarners,
    COUNT(CASE WHEN Salary < 50000 THEN 1 END) AS LowEarners,
    SUM(CASE WHEN Status = 'Active' THEN 1 ELSE 0 END) AS ActiveCount,
    AVG(CASE WHEN Gender = 'F' THEN Salary END) AS AvgFemaleSalary,
    AVG(CASE WHEN Gender = 'M' THEN Salary END) AS AvgMaleSalary
FROM Employees
GROUP BY Department;

-- IIF shorthand (SQL Server 2012+)
SELECT 
    Department,
    SUM(IIF(Status = 'Active', 1, 0)) AS ActiveCount,
    SUM(IIF(Status = 'Inactive', 1, 0)) AS InactiveCount
FROM Employees
GROUP BY Department;
```

## Best Practices

- Use OFFSET/FETCH for standard pagination, but switch to keyset pagination (WHERE Id > @lastId) for deep pages (offset > 10K rows).
- Prefer `NOT EXISTS` over `NOT IN` with subqueries to avoid NULL-related bugs where NOT IN returns zero rows.
- Use CTEs to break complex queries into readable named steps, but note that CTEs are not materialized (they execute each time referenced).
- CROSS APPLY is more efficient than correlated subqueries in SELECT for top-N-per-group patterns.
- End every MERGE statement with a semicolon and consider the simpler UPDATE/INSERT alternative to avoid known MERGE bugs.
- Use STRING_AGG (2017+) instead of the FOR XML PATH string concatenation hack.
- Use window functions instead of self-joins for LAG/LEAD, running totals, and ranking.
- Prefer EXISTS over COUNT(*) > 0 for existence checks — it short-circuits.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using NOT IN with a subquery that can return NULL | Returns zero rows unexpectedly | Use NOT EXISTS instead |
| OFFSET/FETCH without ORDER BY | Syntax error (ORDER BY is required) | Always specify ORDER BY |
| Referencing a CTE multiple times expecting it to cache | CTE re-executes each time, performance hit | Materialize into #temp table if referenced multiple times |
| MERGE without terminal semicolon | Syntax error or ambiguous batch parsing | Always end MERGE with `;` |
| LAST_VALUE without ROWS UNBOUNDED FOLLOWING | Returns current row value (default frame) | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Using ROW_NUMBER for pagination on non-unique ORDER BY | Non-deterministic page contents | Add a unique tiebreaker column to ORDER BY |
| Dynamic PIVOT without QUOTENAME | SQL injection vulnerability | Always wrap dynamic column names with QUOTENAME() |

## SQL Server Version Notes

- **SQL Server 2012** — OFFSET/FETCH, LAG/LEAD, FIRST_VALUE/LAST_VALUE, IIF, TRY_CONVERT, THROW.
- **SQL Server 2016** — STRING_SPLIT, AT TIME ZONE, DROP IF EXISTS, JSON support.
- **SQL Server 2017** — STRING_AGG, TRIM, TRANSLATE, CONCAT_WS. Graph tables (NODE/EDGE).
- **SQL Server 2019** — APPROX_COUNT_DISTINCT, UTF-8 collations, scalar UDF inlining, table variable deferred compilation.
- **SQL Server 2022** — GENERATE_SERIES, GREATEST/LEAST, DATETRUNC, STRING_SPLIT with ordinal, WINDOW clause, IS [NOT] DISTINCT FROM, LTRIM/RTRIM with characters.

## Sources

- https://learn.microsoft.com/en-us/sql/t-sql/queries/select-over-clause-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/queries/select-order-by-clause-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/queries/from-transact-sql#using-apply
- https://learn.microsoft.com/en-us/sql/t-sql/queries/from-using-pivot-and-unpivot
- https://learn.microsoft.com/en-us/sql/t-sql/functions/string-agg-transact-sql
