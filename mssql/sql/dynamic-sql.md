# Dynamic SQL in SQL Server — sp_executesql, Safety, and Practical Patterns

## Overview

Dynamic SQL is T-SQL code that is constructed as a string at runtime and then executed. It is necessary when table names, column names, sort orders, or query structures cannot be determined at compile time. Common use cases include dynamic search screens, generic reporting procedures, dynamic PIVOT queries, and administrative scripts that operate across multiple databases or tables.

SQL Server provides two mechanisms for executing dynamic SQL: `sp_executesql` (parameterized, preferred) and `EXEC()` (non-parameterized, legacy). sp_executesql enables plan caching through parameterization and protects against SQL injection for parameter values. EXEC() concatenates raw strings and should be avoided for any query that includes user-provided values.

The key to safe, performant dynamic SQL is separating structure (table names, column names, sort order) from values (filter criteria, parameter values). Structure must be validated and injected via QUOTENAME; values must be passed as parameters through sp_executesql. This dual approach prevents SQL injection while maintaining execution plan reuse.

## Key Concepts

- **sp_executesql** — System stored procedure that executes a parameterized Unicode string. Supports input/output parameters. Plans are cached and reused when the SQL template matches.
- **EXEC() / EXECUTE()** — Executes a raw string. No parameterization, no plan caching by template. Avoid for anything with user input.
- **QUOTENAME()** — Wraps an identifier in square brackets and escapes embedded `]` characters. Essential for safely injecting table/column names into dynamic SQL.
- **SQL injection** — An attack where malicious SQL is concatenated into a query string. sp_executesql with parameters prevents value-based injection; QUOTENAME prevents identifier-based injection.
- **Plan cache reuse** — sp_executesql caches plans by the SQL template string. Different parameter values reuse the same plan, improving performance.
- **Catch-all query** — A search pattern where multiple optional parameters filter results, with NULL meaning "no filter." Dynamic SQL is one of the best solutions for this pattern.

## sp_executesql Fundamentals

```sql
-- Basic parameterized query
DECLARE @sql NVARCHAR(MAX);
DECLARE @params NVARCHAR(MAX);

SET @sql = N'SELECT EmployeeId, Name, Salary 
             FROM dbo.Employees 
             WHERE Department = @dept AND Salary >= @minSal';

SET @params = N'@dept NVARCHAR(50), @minSal DECIMAL(12,2)';

EXEC sp_executesql @sql, @params,
    @dept = N'Engineering',
    @minSal = 80000.00;

-- With output parameter
DECLARE @sql NVARCHAR(MAX);
DECLARE @params NVARCHAR(MAX);
DECLARE @count INT;

SET @sql = N'SELECT @cnt = COUNT(*) FROM dbo.Employees WHERE Department = @dept';
SET @params = N'@dept NVARCHAR(50), @cnt INT OUTPUT';

EXEC sp_executesql @sql, @params,
    @dept = N'Engineering',
    @cnt = @count OUTPUT;

PRINT CONCAT('Count: ', @count);

-- Multiple statements in single sp_executesql call
SET @sql = N'
    UPDATE dbo.Products SET Price = @newPrice WHERE ProductId = @pid;
    SELECT @affected = @@ROWCOUNT;
';
SET @params = N'@newPrice DECIMAL(12,2), @pid INT, @affected INT OUTPUT';

DECLARE @rowsAffected INT;
EXEC sp_executesql @sql, @params,
    @newPrice = 29.99,
    @pid = 42,
    @affected = @rowsAffected OUTPUT;
```

## EXEC(@sql) — When and Why to Avoid

```sql
-- EXEC() is non-parameterized — each unique string gets its own plan
-- Only use when sp_executesql cannot work (rare)

-- EXEC with simple string
DECLARE @sql NVARCHAR(MAX) = N'SELECT GETDATE()';
EXEC(@sql);

-- EXEC across databases (one valid use case)
DECLARE @dbName NVARCHAR(128) = QUOTENAME(N'OtherDatabase');
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM ' + @dbName + N'.dbo.SomeTable';
EXEC(@sql);

-- NEVER do this — SQL injection vulnerability
DECLARE @userInput NVARCHAR(100) = N'Engineering';
-- BAD: user could pass "'; DROP TABLE Employees; --"
EXEC(N'SELECT * FROM Employees WHERE Department = ''' + @userInput + N'''');

-- CORRECT approach with sp_executesql
EXEC sp_executesql 
    N'SELECT * FROM Employees WHERE Department = @dept',
    N'@dept NVARCHAR(100)',
    @dept = @userInput;
```

## Safe Dynamic Table and Column Names

```sql
-- QUOTENAME wraps identifiers in [] and escapes embedded ] characters
-- It is the ONLY safe way to inject identifiers into dynamic SQL

-- Dynamic table name
CREATE OR ALTER PROCEDURE dbo.usp_GetRowCount
    @SchemaName NVARCHAR(128),
    @TableName NVARCHAR(128),
    @RowCount INT = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Validate: table must exist
    IF NOT EXISTS (
        SELECT 1 FROM sys.tables t
        JOIN sys.schemas s ON t.schema_id = s.schema_id
        WHERE s.name = @SchemaName AND t.name = @TableName
    )
    BEGIN
        THROW 50001, 'Table does not exist.', 1;
    END
    
    DECLARE @sql NVARCHAR(MAX) = N'SELECT @cnt = COUNT(*) FROM '
        + QUOTENAME(@SchemaName) + N'.' + QUOTENAME(@TableName);
    
    EXEC sp_executesql @sql, N'@cnt INT OUTPUT', @cnt = @RowCount OUTPUT;
END;
GO

-- Dynamic column name with QUOTENAME
CREATE OR ALTER PROCEDURE dbo.usp_GetTopByColumn
    @TableName NVARCHAR(128),
    @ColumnName NVARCHAR(128),
    @TopN INT = 10
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Validate column exists in the table
    IF NOT EXISTS (
        SELECT 1 FROM sys.columns c
        JOIN sys.tables t ON c.object_id = t.object_id
        WHERE t.name = @TableName AND c.name = @ColumnName
    )
    BEGIN
        THROW 50001, 'Column does not exist in the specified table.', 1;
    END
    
    DECLARE @sql NVARCHAR(MAX) = N'SELECT TOP(@n) * FROM '
        + QUOTENAME(@TableName) + N' ORDER BY ' + QUOTENAME(@ColumnName) + N' DESC';
    
    EXEC sp_executesql @sql, N'@n INT', @n = @TopN;
END;
GO
```

## Dynamic Search / Catch-All Query Pattern

```sql
-- The "kitchen sink" or "catch-all" query: multiple optional filters
-- Dynamic SQL approach (best performance)
CREATE OR ALTER PROCEDURE dbo.usp_SearchOrders
    @OrderId INT = NULL,
    @CustomerId INT = NULL,
    @Status NVARCHAR(20) = NULL,
    @MinAmount DECIMAL(12,2) = NULL,
    @MaxAmount DECIMAL(12,2) = NULL,
    @StartDate DATE = NULL,
    @EndDate DATE = NULL,
    @SortColumn NVARCHAR(50) = N'OrderDate',
    @SortDir NVARCHAR(4) = N'DESC'
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Validate sort column against whitelist
    IF @SortColumn NOT IN (N'OrderId', N'OrderDate', N'Amount', N'Status', N'CustomerId')
        SET @SortColumn = N'OrderDate';
    
    IF @SortDir NOT IN (N'ASC', N'DESC')
        SET @SortDir = N'DESC';
    
    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT OrderId, CustomerId, Amount, OrderDate, Status
        FROM dbo.Orders
        WHERE 1 = 1';
    
    DECLARE @params NVARCHAR(MAX) = N'
        @pOrderId INT,
        @pCustomerId INT,
        @pStatus NVARCHAR(20),
        @pMinAmount DECIMAL(12,2),
        @pMaxAmount DECIMAL(12,2),
        @pStartDate DATE,
        @pEndDate DATE';
    
    IF @OrderId IS NOT NULL
        SET @sql += N' AND OrderId = @pOrderId';
    
    IF @CustomerId IS NOT NULL
        SET @sql += N' AND CustomerId = @pCustomerId';
    
    IF @Status IS NOT NULL
        SET @sql += N' AND Status = @pStatus';
    
    IF @MinAmount IS NOT NULL
        SET @sql += N' AND Amount >= @pMinAmount';
    
    IF @MaxAmount IS NOT NULL
        SET @sql += N' AND Amount <= @pMaxAmount';
    
    IF @StartDate IS NOT NULL
        SET @sql += N' AND OrderDate >= @pStartDate';
    
    IF @EndDate IS NOT NULL
        SET @sql += N' AND OrderDate <= @pEndDate';
    
    -- Safe sort injection: validated against whitelist, injected with QUOTENAME
    SET @sql += N' ORDER BY ' + QUOTENAME(@SortColumn) + N' ' + @SortDir;
    
    EXEC sp_executesql @sql, @params,
        @pOrderId = @OrderId,
        @pCustomerId = @CustomerId,
        @pStatus = @Status,
        @pMinAmount = @MinAmount,
        @pMaxAmount = @MaxAmount,
        @pStartDate = @StartDate,
        @pEndDate = @EndDate;
END;
GO

-- Alternative: static SQL with OPTION(RECOMPILE)
-- Simpler but recompiles every execution
CREATE OR ALTER PROCEDURE dbo.usp_SearchOrders_Static
    @OrderId INT = NULL,
    @CustomerId INT = NULL,
    @Status NVARCHAR(20) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT OrderId, CustomerId, Amount, OrderDate, Status
    FROM dbo.Orders
    WHERE (@OrderId IS NULL OR OrderId = @OrderId)
      AND (@CustomerId IS NULL OR CustomerId = @CustomerId)
      AND (@Status IS NULL OR Status = @Status)
    OPTION (RECOMPILE);  -- essential for good plan with nullable params
END;
GO
```

## Dynamic PIVOT

```sql
-- When PIVOT column values are not known at compile time
CREATE OR ALTER PROCEDURE dbo.usp_SalesByRegionPivot
    @Year INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Step 1: Get distinct column values
    DECLARE @columns NVARCHAR(MAX);
    
    SELECT @columns = STRING_AGG(QUOTENAME(Region), N',')
    FROM (SELECT DISTINCT Region FROM Sales WHERE YEAR(SaleDate) = @Year) AS r;
    
    IF @columns IS NULL
    BEGIN
        THROW 50001, 'No sales data found for the specified year.', 1;
    END
    
    -- Step 2: Build PIVOT query
    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT ProductName, ' + @columns + N'
        FROM (
            SELECT p.ProductName, s.Region, s.Amount
            FROM Sales s
            JOIN Products p ON s.ProductId = p.ProductId
            WHERE YEAR(s.SaleDate) = @pYear
        ) AS src
        PIVOT (
            SUM(Amount) FOR Region IN (' + @columns + N')
        ) AS pvt
        ORDER BY ProductName';
    
    EXEC sp_executesql @sql, N'@pYear INT', @pYear = @Year;
END;
GO
```

## Debugging Dynamic SQL

```sql
-- PRINT the generated SQL before executing
CREATE OR ALTER PROCEDURE dbo.usp_DebugExample
    @Department NVARCHAR(50),
    @Debug BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT EmployeeId, Name, Salary
        FROM dbo.Employees
        WHERE Department = @dept
        ORDER BY Salary DESC';
    
    -- Debug mode: print SQL and parameters without executing
    IF @Debug = 1
    BEGIN
        PRINT N'--- Generated SQL ---';
        PRINT @sql;
        PRINT N'--- Parameters ---';
        PRINT CONCAT(N'@dept = ', QUOTENAME(@Department, ''''));
        RETURN;
    END
    
    EXEC sp_executesql @sql, N'@dept NVARCHAR(50)', @dept = @Department;
END;
GO

-- For SQL > 4000 chars, PRINT truncates. Use this helper:
DECLARE @sql NVARCHAR(MAX) = N'... very long SQL ...';
DECLARE @pos INT = 1;
WHILE @pos <= LEN(@sql)
BEGIN
    PRINT SUBSTRING(@sql, @pos, 4000);
    SET @pos += 4000;
END

-- Or use the XML trick for full output in SSMS Messages tab:
SELECT @sql AS [processing-instruction(sql)] FOR XML PATH('');
```

## Dynamic SQL in Administrative Scripts

```sql
-- Execute a statement in every user database
DECLARE @sql NVARCHAR(MAX) = N'';

SELECT @sql += N'
USE ' + QUOTENAME(name) + N';
SELECT DB_NAME() AS DatabaseName, 
       SUM(size) * 8 / 1024 AS SizeMB
FROM sys.database_files;
'
FROM sys.databases
WHERE database_id > 4  -- skip system databases
  AND state = 0;       -- online only

EXEC sp_executesql @sql;

-- Alternative: sp_MSforeachdb (undocumented but widely used)
EXEC sp_MSforeachdb N'
    USE [?];
    IF DB_ID() > 4
    SELECT DB_NAME() AS db, COUNT(*) AS table_count 
    FROM sys.tables;
';

-- Generate index rebuild statements dynamically
SELECT 
    N'ALTER INDEX ' + QUOTENAME(i.name) + N' ON ' 
    + QUOTENAME(s.name) + N'.' + QUOTENAME(t.name) 
    + N' REBUILD WITH (ONLINE = ON);' AS RebuildCommand
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
JOIN sys.tables t ON i.object_id = t.object_id
JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE ips.avg_fragmentation_in_percent > 30
  AND ips.page_count > 1000
  AND i.name IS NOT NULL;
```

## Security Considerations

```sql
-- SQL injection attack examples and prevention

-- Vulnerable: string concatenation
DECLARE @userInput NVARCHAR(100) = N'''; DROP TABLE Employees; --';
EXEC(N'SELECT * FROM Employees WHERE Name = ''' + @userInput + N'''');
-- Executes: SELECT * FROM Employees WHERE Name = ''; DROP TABLE Employees; --'

-- Safe: parameterized with sp_executesql
EXEC sp_executesql 
    N'SELECT * FROM Employees WHERE Name = @name',
    N'@name NVARCHAR(100)',
    @name = @userInput;
-- The @name parameter is treated as a literal value, not executable SQL

-- Safe: QUOTENAME for identifiers
DECLARE @tableName NVARCHAR(128) = N'Employees; DROP TABLE Users';
-- QUOTENAME returns [Employees; DROP TABLE Users] — treated as a single identifier name
-- This will error with "Invalid object name" rather than executing DROP TABLE

-- Whitelist validation for sort columns, operators, etc.
IF @SortColumn NOT IN (N'Name', N'Date', N'Amount')
    THROW 50001, 'Invalid sort column.', 1;

-- QUOTENAME length limit: 128 characters (sysname max)
-- If input > 128 chars, QUOTENAME returns NULL — always check for NULL result
IF QUOTENAME(@identifier) IS NULL
    THROW 50001, 'Invalid identifier.', 1;
```

## Best Practices

- Always use `sp_executesql` with parameters for dynamic SQL — never concatenate user-provided values into SQL strings.
- Use `QUOTENAME()` for any dynamic table name, column name, or schema name. Always validate that the identifier exists before using it.
- Maintain a whitelist for sort columns, sort directions, and operators — never allow arbitrary user input for these structural elements.
- Add a `@Debug BIT = 0` parameter to procedures with dynamic SQL to enable printing the generated SQL without executing.
- Keep dynamic SQL as short as possible — extract only the truly dynamic parts and keep the rest in static SQL.
- Use `NVARCHAR(MAX)` for all dynamic SQL variables (not NVARCHAR(4000)) to avoid silent truncation.
- Validate input parameters before building dynamic SQL — reject invalid values early.
- Test with adversarial inputs: single quotes, semicolons, comment markers (`--`), and very long strings.
- Document why dynamic SQL is needed in a comment — future maintainers should understand the requirement.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using EXEC() with concatenated user input | SQL injection vulnerability | Use sp_executesql with parameters |
| Using NVARCHAR(4000) for @sql variable | Silent truncation of SQL > 4000 chars | Use NVARCHAR(MAX) |
| Not using QUOTENAME for dynamic identifiers | SQL injection via identifier names | Wrap all dynamic identifiers with QUOTENAME() |
| Forgetting N prefix on Unicode string literals | Non-Unicode string, potential data loss | Always use N'...' for NVARCHAR strings |
| Not validating dynamic sort columns | Injection via ORDER BY clause | Whitelist valid column names |
| Checking QUOTENAME result for NULL | QUOTENAME returns NULL for >128 char input | Validate length before QUOTENAME, check for NULL after |
| Nesting dynamic SQL (EXEC inside EXEC) | Extremely hard to debug, security nightmare | Restructure to single-level dynamic SQL |

## SQL Server Version Notes

- **SQL Server 2016** — All dynamic SQL patterns work. sp_executesql supports up to 2100 parameters. STRING_SPLIT available for building IN lists.
- **SQL Server 2017** — STRING_AGG simplifies building comma-separated column lists for PIVOT.
- **SQL Server 2019** — Table variable deferred compilation improves perf when dynamic SQL populates table variables. Scalar UDF inlining may affect dynamic SQL that calls scalar functions.
- **SQL Server 2022** — GENERATE_SERIES useful in dynamic admin scripts. STRING_SPLIT with ordinal helps ordered list processing. No fundamental changes to dynamic SQL mechanics.
- **Azure SQL Database** — Same sp_executesql behavior. Query Store tracks dynamic SQL plans effectively. Automatic tuning can force plans for parameterized dynamic SQL.

## Sources

- https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-executesql-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/functions/quotename-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/security/sql-injection
- https://learn.microsoft.com/en-us/sql/odbc/reference/develop-app/constructing-dynamic-sql-statements
- https://www.sommarskog.se/dynamic_sql.html
