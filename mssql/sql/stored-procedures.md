# Stored Procedures in SQL Server — Design, Error Handling, and Performance

## Overview

Stored procedures are precompiled T-SQL programs stored in the database that encapsulate business logic, enforce security boundaries, and provide a stable API between applications and database schema. They are the primary mechanism for server-side logic in SQL Server and offer execution plan caching, reduced network round-trips, and granular permission control.

Well-designed stored procedures follow consistent patterns for error handling (TRY/CATCH), parameter validation, transaction management, and output. They use SET NOCOUNT ON to suppress row-count messages, parameterized dynamic SQL via sp_executesql when needed, and table-valued parameters (TVPs) for passing sets of data from applications.

Understanding parameter sniffing, the differences between temp tables and table variables, and proper security patterns (EXECUTE AS, certificate signing) is essential for writing stored procedures that perform well and maintain security in production environments.

## Key Concepts

- **Execution plan caching** — SQL Server compiles a stored procedure on first execution and caches the plan for reuse. This reduces compilation overhead but can cause parameter sniffing issues.
- **Parameter sniffing** — The optimizer uses the parameter values from the first execution to build the cached plan. If those values are atypical, subsequent calls with different values may use a suboptimal plan.
- **SET NOCOUNT ON** — Suppresses "N rows affected" messages sent to the client after each statement. Reduces network traffic and prevents ADO.NET `RecordsAffected` confusion.
- **TRY/CATCH** — Structured error handling that catches most runtime errors (not compile-time or severity 20+ errors).
- **THROW** — Re-raises the caught error preserving original error number, severity, and state (SQL Server 2012+).
- **Table-valued parameter (TVP)** — A user-defined table type that allows passing tabular data to a stored procedure as a single parameter.
- **RETURN** — Exits the procedure and returns an integer status code (0 = success by convention).

## Basic Stored Procedure Structure

```sql
CREATE OR ALTER PROCEDURE dbo.usp_GetEmployeesByDepartment
    @Department NVARCHAR(50),
    @MinSalary DECIMAL(12,2) = 0,           -- optional with default
    @RowCount INT = NULL OUTPUT              -- output parameter
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT EmployeeId, Name, Department, Salary, HireDate
    FROM dbo.Employees
    WHERE Department = @Department
      AND Salary >= @MinSalary
    ORDER BY Name;
    
    SET @RowCount = @@ROWCOUNT;
    
    RETURN 0;  -- success
END;
GO

-- Calling the procedure
DECLARE @cnt INT;
EXEC dbo.usp_GetEmployeesByDepartment 
    @Department = N'Engineering',
    @MinSalary = 80000,
    @RowCount = @cnt OUTPUT;
PRINT CONCAT('Found ', @cnt, ' employees');
```

## Error Handling with TRY/CATCH

```sql
CREATE OR ALTER PROCEDURE dbo.usp_CreateOrder
    @CustomerId INT,
    @ProductId INT,
    @Quantity INT,
    @OrderId INT = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;  -- auto-rollback on any error
    
    -- Parameter validation
    IF @CustomerId IS NULL OR @ProductId IS NULL OR @Quantity IS NULL
    BEGIN
        THROW 50001, 'CustomerId, ProductId, and Quantity are required.', 1;
    END
    
    IF @Quantity <= 0
    BEGIN
        THROW 50002, 'Quantity must be greater than zero.', 1;
    END
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Check inventory
        DECLARE @CurrentStock INT;
        SELECT @CurrentStock = Stock
        FROM dbo.Products WITH (UPDLOCK)
        WHERE ProductId = @ProductId;
        
        IF @CurrentStock IS NULL
            THROW 50003, 'Product not found.', 1;
        
        IF @CurrentStock < @Quantity
            THROW 50004, 'Insufficient inventory.', 1;
        
        -- Create order
        INSERT INTO dbo.Orders (CustomerId, ProductId, Quantity, OrderDate, Status)
        VALUES (@CustomerId, @ProductId, @Quantity, GETUTCDATE(), 'Pending');
        
        SET @OrderId = SCOPE_IDENTITY();
        
        -- Decrement inventory
        UPDATE dbo.Products
        SET Stock = Stock - @Quantity
        WHERE ProductId = @ProductId;
        
        COMMIT TRANSACTION;
        RETURN 0;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Log the error
        INSERT INTO dbo.ErrorLog (
            ErrorNumber, ErrorSeverity, ErrorState,
            ErrorProcedure, ErrorLine, ErrorMessage, ErrorTime
        )
        VALUES (
            ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE(),
            ERROR_PROCEDURE(), ERROR_LINE(), ERROR_MESSAGE(), GETUTCDATE()
        );
        
        -- Re-throw the original error
        THROW;
    END CATCH
END;
GO
```

## THROW vs RAISERROR

```sql
-- THROW (SQL Server 2012+): preferred for new code
-- Raises error, honors SET XACT_ABORT, preserves original error info when re-throwing
BEGIN CATCH
    THROW;  -- re-throws caught error with original number/severity/state
END CATCH

-- Custom error with THROW
THROW 50001, 'Custom error message', 1;
-- Error number must be >= 50000 for user-defined errors

-- RAISERROR: legacy, still used for specific scenarios
-- Can raise messages from sys.messages catalog
-- Supports printf-style formatting
RAISERROR(N'Order %d not found for customer %s', 16, 1, @OrderId, @CustomerName);

-- RAISERROR with severity levels:
-- 1-10: informational (no error, not caught by CATCH)
-- 11-16: user errors (caught by CATCH)
-- 17-19: resource/software errors (caught by CATCH)
-- 20-25: fatal errors (kills connection, not caught by CATCH)

-- RAISERROR WITH NOWAIT: immediate flush to client (useful for progress reporting)
RAISERROR('Step 1 complete', 0, 1) WITH NOWAIT;
```

## Table-Valued Parameters (TVPs)

```sql
-- Step 1: Create the table type
CREATE TYPE dbo.OrderItemType AS TABLE (
    ProductId INT NOT NULL,
    Quantity INT NOT NULL,
    UnitPrice DECIMAL(12,2) NOT NULL
);
GO

-- Step 2: Use in stored procedure
CREATE OR ALTER PROCEDURE dbo.usp_CreateOrderWithItems
    @CustomerId INT,
    @Items dbo.OrderItemType READONLY,  -- TVPs are always READONLY
    @OrderId INT = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Create order header
        INSERT INTO dbo.Orders (CustomerId, OrderDate, Status)
        VALUES (@CustomerId, GETUTCDATE(), 'Pending');
        SET @OrderId = SCOPE_IDENTITY();
        
        -- Insert all items from TVP
        INSERT INTO dbo.OrderItems (OrderId, ProductId, Quantity, UnitPrice)
        SELECT @OrderId, ProductId, Quantity, UnitPrice
        FROM @Items;
        
        -- Update order total
        UPDATE dbo.Orders
        SET TotalAmount = (
            SELECT SUM(Quantity * UnitPrice) FROM dbo.OrderItems WHERE OrderId = @OrderId
        )
        WHERE OrderId = @OrderId;
        
        COMMIT TRANSACTION;
        RETURN 0;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO

-- Calling from T-SQL
DECLARE @items dbo.OrderItemType;
INSERT INTO @items VALUES (101, 2, 29.99), (205, 1, 49.99), (310, 5, 9.99);
DECLARE @newOrderId INT;
EXEC dbo.usp_CreateOrderWithItems @CustomerId = 42, @Items = @items, @OrderId = @newOrderId OUTPUT;
```

## Temp Tables vs Table Variables

```sql
-- #temp tables: statistics, indexes, parallel plans, spill to disk
-- Better for large result sets (>100 rows), joins, complex queries
CREATE OR ALTER PROCEDURE dbo.usp_ComplexReport @StartDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    CREATE TABLE #OrderSummary (
        CustomerId INT,
        OrderCount INT,
        TotalAmount DECIMAL(18,2),
        INDEX IX_Customer CLUSTERED (CustomerId)
    );
    
    INSERT INTO #OrderSummary
    SELECT CustomerId, COUNT(*), SUM(Amount)
    FROM Orders
    WHERE OrderDate >= @StartDate
    GROUP BY CustomerId;
    
    -- SQL Server creates statistics on #temp, enabling good join plans
    SELECT c.Name, s.OrderCount, s.TotalAmount
    FROM Customers c
    JOIN #OrderSummary s ON c.CustomerId = s.CustomerId
    ORDER BY s.TotalAmount DESC;
END;
GO

-- @table variables: no statistics, no parallel plans (pre-2019), in-memory
-- Better for small result sets (<100 rows), simple lookups
CREATE OR ALTER PROCEDURE dbo.usp_GetManagerChain @EmployeeId INT
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @chain TABLE (
        Level INT,
        EmployeeId INT,
        Name NVARCHAR(100),
        PRIMARY KEY (Level)
    );
    
    -- Small result set, table variable is fine
    ;WITH Managers AS (
        SELECT 0 AS Level, EmployeeId, Name, ManagerId
        FROM Employees WHERE EmployeeId = @EmployeeId
        UNION ALL
        SELECT m.Level + 1, e.EmployeeId, e.Name, e.ManagerId
        FROM Employees e JOIN Managers m ON e.EmployeeId = m.ManagerId
    )
    INSERT INTO @chain SELECT Level, EmployeeId, Name FROM Managers;
    
    SELECT * FROM @chain ORDER BY Level;
END;
GO

-- SQL Server 2019+: table variable deferred compilation
-- Optimizer can now see actual row count for @table variables
-- Reduces the need to switch to #temp tables for better estimates
```

## Parameter Sniffing and OPTION(RECOMPILE)

```sql
-- Problem: first call with atypical value creates a bad cached plan
CREATE OR ALTER PROCEDURE dbo.usp_GetOrdersByStatus
    @Status NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    -- If first call is @Status = 'Archived' (1M rows), plan uses table scan
    -- Subsequent call with @Status = 'Pending' (10 rows) reuses scan plan
    SELECT OrderId, CustomerId, Amount, OrderDate
    FROM Orders
    WHERE Status = @Status;
END;
GO

-- Fix 1: OPTION(RECOMPILE) — recompile every time (best for infrequent calls)
SELECT OrderId, CustomerId, Amount, OrderDate
FROM Orders
WHERE Status = @Status
OPTION (RECOMPILE);

-- Fix 2: Local variable (hides parameter value from optimizer)
CREATE OR ALTER PROCEDURE dbo.usp_GetOrdersByStatus
    @Status NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @LocalStatus NVARCHAR(20) = @Status;
    SELECT * FROM Orders WHERE Status = @LocalStatus;
    -- Uses average density estimate instead of sniffed value
END;
GO

-- Fix 3: OPTIMIZE FOR hint
SELECT * FROM Orders WHERE Status = @Status
OPTION (OPTIMIZE FOR (@Status = 'Pending'));  -- optimize for typical value

-- Fix 4: OPTIMIZE FOR UNKNOWN
SELECT * FROM Orders WHERE Status = @Status
OPTION (OPTIMIZE FOR (@Status UNKNOWN));  -- uses average statistics
```

## Dynamic SQL with sp_executesql

```sql
-- sp_executesql: parameterized dynamic SQL (safe, cacheable)
CREATE OR ALTER PROCEDURE dbo.usp_SearchEmployees
    @Name NVARCHAR(100) = NULL,
    @Department NVARCHAR(50) = NULL,
    @MinSalary DECIMAL(12,2) = NULL,
    @MaxSalary DECIMAL(12,2) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @sql NVARCHAR(MAX) = N'
        SELECT EmployeeId, Name, Department, Salary
        FROM dbo.Employees
        WHERE 1 = 1';
    
    DECLARE @params NVARCHAR(MAX) = N'
        @pName NVARCHAR(100),
        @pDept NVARCHAR(50),
        @pMinSalary DECIMAL(12,2),
        @pMaxSalary DECIMAL(12,2)';
    
    IF @Name IS NOT NULL
        SET @sql += N' AND Name LIKE @pName + N''%''';
    
    IF @Department IS NOT NULL
        SET @sql += N' AND Department = @pDept';
    
    IF @MinSalary IS NOT NULL
        SET @sql += N' AND Salary >= @pMinSalary';
    
    IF @MaxSalary IS NOT NULL
        SET @sql += N' AND Salary <= @pMaxSalary';
    
    SET @sql += N' ORDER BY Name';
    
    EXEC sp_executesql @sql, @params,
        @pName = @Name,
        @pDept = @Department,
        @pMinSalary = @MinSalary,
        @pMaxSalary = @MaxSalary;
END;
GO
```

## Security Patterns

```sql
-- EXECUTE AS: impersonate another user/login
CREATE OR ALTER PROCEDURE dbo.usp_GetSensitiveData
WITH EXECUTE AS 'DataReaderUser'  -- runs as this user regardless of caller
AS
BEGIN
    SET NOCOUNT ON;
    SELECT * FROM dbo.SensitiveTable;
END;
GO

-- Grant EXECUTE without underlying table permissions (ownership chaining)
-- If proc and tables are in same schema with same owner, no direct table permission needed
GRANT EXECUTE ON dbo.usp_GetSensitiveData TO AppRole;

-- Certificate signing: grant elevated permissions without EXECUTE AS
-- Step 1: Create certificate
CREATE CERTIFICATE ProcCert WITH SUBJECT = 'For signing procedures';
-- Step 2: Create user from certificate
CREATE USER ProcCertUser FROM CERTIFICATE ProcCert;
-- Step 3: Grant permissions to cert user
GRANT SELECT ON dbo.SensitiveTable TO ProcCertUser;
-- Step 4: Sign the procedure
ADD SIGNATURE TO dbo.usp_GetSensitiveData BY CERTIFICATE ProcCert;
-- Now any user with EXECUTE on the proc gets the cert user's permissions
```

## Naming and Design Conventions

```sql
-- Naming conventions
-- usp_ prefix for user stored procedures (avoid sp_ which searches master first)
-- Include action verb: usp_Get, usp_Create, usp_Update, usp_Delete, usp_Search
-- Examples: usp_GetEmployeeById, usp_CreateOrder, usp_SearchProducts

-- Standard template
CREATE OR ALTER PROCEDURE dbo.usp_ActionEntity
    @RequiredParam DATATYPE,
    @OptionalParam DATATYPE = DefaultValue,
    @OutputParam DATATYPE = NULL OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    
    -- Validation
    -- Transaction (if needed)
    -- Core logic
    -- Output assignment
    
    RETURN 0;
END;
GO
```

## Best Practices

- Always start with `SET NOCOUNT ON` to eliminate unnecessary row-count messages.
- Use `SET XACT_ABORT ON` in any procedure with explicit transactions to ensure automatic rollback on errors.
- Prefer `THROW` over `RAISERROR` for error raising in SQL Server 2012+. Use `THROW` without parameters in CATCH to re-raise.
- Check `@@TRANCOUNT > 0` before ROLLBACK in CATCH blocks to avoid "no corresponding BEGIN TRANSACTION" errors.
- Name procedures with `usp_` prefix, never `sp_` (which triggers a search in the master database first).
- Use TVPs instead of comma-delimited string parameters for passing sets of data.
- Use `#temp` tables for intermediate result sets over ~100 rows; use `@table` variables for small lookup sets.
- Address parameter sniffing proactively: use `OPTION(RECOMPILE)` for infrequent, variable-data procedures; use `OPTIMIZE FOR` hints for known data distributions.
- Always use `sp_executesql` for dynamic SQL — never `EXEC(@sql)` with concatenated values.
- Use `SCOPE_IDENTITY()` to get the last inserted identity value, not `@@IDENTITY` (which crosses scopes/triggers).

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `sp_` prefix for user procedures | Extra lookup in master database on every call | Use `usp_` prefix |
| Missing `SET NOCOUNT ON` | Extra network round-trips, ADO.NET RecordsAffected confusion | Add as first statement |
| Using `@@IDENTITY` instead of `SCOPE_IDENTITY()` | Returns wrong ID if trigger inserts into identity table | Use `SCOPE_IDENTITY()` |
| Not handling parameter sniffing | Slow queries after plan cache with atypical values | Use `OPTION(RECOMPILE)` or `OPTIMIZE FOR` |
| CATCH block without checking `@@TRANCOUNT` | Error on ROLLBACK when no transaction active | Always check `IF @@TRANCOUNT > 0` |
| Using `EXEC(@sql)` with string concatenation | SQL injection, no plan caching | Use `sp_executesql` with parameters |
| Temp table in frequently called proc without cleanup | Recompilation on schema changes to cached #temp | Use table variables for tiny sets or accept recompile cost |

## SQL Server Version Notes

- **SQL Server 2016** — CREATE OR ALTER supported. DROP IF EXISTS. Session-scoped global temp tables (##). Natively compiled procs enhanced.
- **SQL Server 2019** — Table variable deferred compilation (better cardinality for @table variables). Scalar UDF inlining (some scalar functions converted to inline for better perf). Batch mode on rowstore.
- **SQL Server 2022** — Parameter-sensitive plan optimization (PSP, addresses some parameter sniffing automatically). GENERATE_SERIES, GREATEST, LEAST available in procedures. Optimized plan forcing.
- **Azure SQL Database** — Same T-SQL surface as 2022. Automatic tuning can force/unforce plans. Query Store enabled by default.

## Sources

- https://learn.microsoft.com/en-us/sql/relational-databases/stored-procedures/stored-procedures-database-engine
- https://learn.microsoft.com/en-us/sql/t-sql/statements/create-procedure-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/language-elements/try-catch-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/language-elements/throw-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/tables/use-table-valued-parameters-database-engine
- https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitivity-plan-optimization
