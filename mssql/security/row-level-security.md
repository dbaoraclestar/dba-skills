# Row-Level Security --- RLS Predicates, Dynamic Data Masking, and Access Control

## Overview

**Row-Level Security (RLS)** enables fine-grained access control over rows in a database table based on the characteristics of the user executing the query. Instead of managing access through views or application logic, RLS uses **security predicates** --- inline table-valued functions attached to tables via **security policies** --- to transparently filter or block rows. The database engine injects these predicates into every query plan, ensuring enforcement regardless of how the data is accessed.

**Dynamic Data Masking (DDM)** is a complementary feature that obfuscates column values in query results without changing the stored data. DDM is a presentation-layer protection: users with UNMASK permission see full values, while others see masked output. DDM does not prevent direct data access via inference or ad-hoc queries by privileged users, so it is best suited for casual exposure prevention rather than strict security boundaries.

Together, RLS and DDM provide defense-in-depth for multi-tenant applications, reporting systems, and regulatory environments where different users must see different subsets and representations of the same underlying data.

## Key Concepts

- **Filter Predicate**: Silently filters rows from SELECT and certain DML results. The user never knows filtered rows exist.
- **Block Predicate**: Prevents DML operations (INSERT, UPDATE, DELETE) that would violate the predicate. Raises an error instead of silently filtering.
- **Security Policy**: Binds one or more predicate functions to one or more tables, controlling which predicates apply and in what mode.
- **Predicate Function**: An inline table-valued function that returns 1 (allow) or 0 (deny) for each row. Must be in a separate schema from the protected table.
- **SESSION_CONTEXT**: A key-value session store that allows applications to pass user context (tenant ID, department, role) without embedding it in the query.
- **Dynamic Data Masking**: Column-level masking functions (default, email, random, partial) applied to column definitions. Does not encrypt data.

## Row-Level Security: Filter Predicates

```sql
-- Scenario: Multi-tenant SaaS application where each tenant
-- should only see their own data

-- Step 1: Create a predicate function
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_TenantFilter(@TenantID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT)
        OR IS_MEMBER('db_owner') = 1;  -- DBAs see all rows
GO

-- Step 2: Create the security policy
CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.Orders,
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.Customers,
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.Invoices
WITH (STATE = ON, SCHEMABINDING = ON);

-- Step 3: Set tenant context in the application
EXEC sp_set_session_context @key = N'TenantID', @value = 42;

-- Now all queries against Orders, Customers, Invoices are filtered
SELECT * FROM dbo.Orders;  -- Only sees TenantID = 42 rows
```

## Row-Level Security: Block Predicates

```sql
-- Block predicates prevent unauthorized DML operations
-- Four block predicate operations:
--   AFTER INSERT  -- block inserts that would not be visible after
--   AFTER UPDATE  -- block updates that move rows out of visibility
--   BEFORE UPDATE -- block updates on rows currently not visible
--   BEFORE DELETE -- block deletes on rows currently not visible

CREATE FUNCTION Security.fn_TenantBlock(@TenantID INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT);
GO

-- Combined filter + block policy
CREATE SECURITY POLICY Security.TenantFullPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.Orders,
ADD BLOCK PREDICATE Security.fn_TenantBlock(TenantID)
    ON dbo.Orders AFTER INSERT,
ADD BLOCK PREDICATE Security.fn_TenantBlock(TenantID)
    ON dbo.Orders AFTER UPDATE,
ADD BLOCK PREDICATE Security.fn_TenantBlock(TenantID)
    ON dbo.Orders BEFORE UPDATE,
ADD BLOCK PREDICATE Security.fn_TenantBlock(TenantID)
    ON dbo.Orders BEFORE DELETE
WITH (STATE = ON, SCHEMABINDING = ON);

-- Test: try to insert a row for a different tenant
EXEC sp_set_session_context @key = N'TenantID', @value = 42;

INSERT INTO dbo.Orders (OrderID, TenantID, Amount)
VALUES (1001, 99, 500.00);
-- Error: The attempted operation failed because the target object
-- 'dbo.Orders' has a block predicate that conflicts with this operation.
```

## SESSION_CONTEXT for User Context

```sql
-- SESSION_CONTEXT stores key-value pairs for the connection lifetime
-- Applications set context after authenticating the user

-- Set context (typically done in connection middleware)
EXEC sp_set_session_context @key = N'TenantID', @value = 42;
EXEC sp_set_session_context @key = N'UserRole', @value = 'Manager';
EXEC sp_set_session_context @key = N'Department', @value = 'Sales';

-- Read-only context (cannot be changed once set)
EXEC sp_set_session_context @key = N'TenantID', @value = 42,
    @read_only = 1;
-- Subsequent attempts to change TenantID will fail

-- Read context values
SELECT
    SESSION_CONTEXT(N'TenantID') AS TenantID,
    SESSION_CONTEXT(N'UserRole') AS UserRole,
    SESSION_CONTEXT(N'Department') AS Department;

-- Use in predicate function for role-based filtering
CREATE FUNCTION Security.fn_DepartmentFilter(@Department NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @Department = CAST(SESSION_CONTEXT(N'Department') AS NVARCHAR(50))
        OR CAST(SESSION_CONTEXT(N'UserRole') AS NVARCHAR(50)) = 'Admin';
GO

-- Alternative: use USER_NAME() or SUSER_SNAME() for identity-based filtering
CREATE FUNCTION Security.fn_UserFilter(@OwnerUser NVARCHAR(128))
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @OwnerUser = USER_NAME()
        OR IS_MEMBER('Managers') = 1;
GO
```

## Advanced RLS Patterns

```sql
-- Pattern: Hierarchical access (manager sees their team's data)
CREATE TABLE Security.UserHierarchy (
    UserName NVARCHAR(128) PRIMARY KEY,
    ManagerName NVARCHAR(128),
    Department NVARCHAR(50)
);

CREATE FUNCTION Security.fn_HierarchyFilter(@OwnerUser NVARCHAR(128))
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @OwnerUser = USER_NAME()
        OR EXISTS (
            SELECT 1 FROM Security.UserHierarchy
            WHERE UserName = @OwnerUser
            AND ManagerName = USER_NAME()
        )
        OR IS_MEMBER('db_owner') = 1;
GO

-- Pattern: Time-based access (data visible only during business hours)
CREATE FUNCTION Security.fn_TimeFilter(@Sensitive BIT)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @Sensitive = 0  -- non-sensitive always visible
        OR (DATEPART(HOUR, GETDATE()) BETWEEN 8 AND 17
            AND DATEPART(WEEKDAY, GETDATE()) BETWEEN 2 AND 6);
GO

-- Pattern: Shared + private rows
CREATE FUNCTION Security.fn_SharedPrivate(
    @OwnerUser NVARCHAR(128),
    @IsPublic BIT
)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_securitypredicate_result
    WHERE @IsPublic = 1
        OR @OwnerUser = USER_NAME()
        OR IS_MEMBER('db_owner') = 1;
GO

CREATE SECURITY POLICY Security.SharedPolicy
ADD FILTER PREDICATE Security.fn_SharedPrivate(CreatedBy, IsPublic)
    ON dbo.Documents
WITH (STATE = ON, SCHEMABINDING = ON);
```

## Managing Security Policies

```sql
-- View all security policies
SELECT
    p.name AS PolicyName,
    p.is_enabled,
    p.is_schema_bound,
    pr.target_schema_name,
    OBJECT_NAME(pr.target_object_id) AS TargetTable,
    pr.predicate_type_desc,
    pr.predicate_definition,
    pr.operation_desc
FROM sys.security_policies p
JOIN sys.security_predicates pr ON p.object_id = pr.object_id
ORDER BY p.name;

-- Disable a policy temporarily (maintenance, bulk load)
ALTER SECURITY POLICY Security.TenantPolicy WITH (STATE = OFF);

-- Perform bulk operations without RLS
BULK INSERT dbo.Orders FROM 'C:\Data\orders.csv'
WITH (FIELDTERMINATOR = ',', ROWTERMINATOR = '\n');

-- Re-enable the policy
ALTER SECURITY POLICY Security.TenantPolicy WITH (STATE = ON);

-- Add a predicate to an existing policy
ALTER SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.Returns;

-- Remove a predicate from a policy
ALTER SECURITY POLICY Security.TenantPolicy
DROP FILTER PREDICATE ON dbo.Returns;

-- Drop a security policy
DROP SECURITY POLICY Security.TenantPolicy;
```

## Dynamic Data Masking

```sql
-- Default mask: full masking (0 for numeric, xxxx for string, 1900-01-01 for date)
CREATE TABLE HR.EmployeeDirectory (
    EmployeeID INT PRIMARY KEY,
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Email NVARCHAR(100) MASKED WITH (FUNCTION = 'email()'),
    Phone VARCHAR(15) MASKED WITH (FUNCTION = 'default()'),
    SSN CHAR(11) MASKED WITH (FUNCTION = 'partial(0, "XXX-XX-", 4)'),
    Salary MONEY MASKED WITH (FUNCTION = 'random(10000, 99999)'),
    BirthDate DATE MASKED WITH (FUNCTION = 'default()')
);

-- Mask functions:
-- default()           -- full masking based on data type
-- email()             -- aXXX@XXXX.com
-- random(start, end)  -- random number in range (numeric types)
-- partial(prefix, padding, suffix)
--   partial(2, "XXXX", 2) on "1234567890" → "12XXXX90"

-- Add mask to existing column
ALTER TABLE HR.EmployeeDirectory
ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(0, "XXX-XXX-", 4)');

-- Remove mask from a column
ALTER TABLE HR.EmployeeDirectory
ALTER COLUMN Phone DROP MASKED;

-- Grant UNMASK permission to a user
GRANT UNMASK TO [HRManager];

-- Grant UNMASK on a specific schema (SQL Server 2022+)
GRANT UNMASK ON SCHEMA::HR TO [HRTeam];

-- Grant UNMASK on a specific object (SQL Server 2022+)
GRANT UNMASK ON OBJECT::HR.EmployeeDirectory TO [PayrollApp];

-- Revoke UNMASK
REVOKE UNMASK FROM [FormerHRManager];

-- Test masking as a specific user
EXECUTE AS USER = 'LimitedUser';
SELECT * FROM HR.EmployeeDirectory;
-- Email: aXXX@XXXX.com, SSN: XXX-XX-6789, Salary: random value
REVERT;

-- View masked columns
SELECT
    t.name AS TableName,
    c.name AS ColumnName,
    c.is_masked,
    c.masking_function
FROM sys.masked_columns c
JOIN sys.tables t ON c.object_id = t.object_id
WHERE c.is_masked = 1;
```

## Combining RLS with DDM and Other Features

```sql
-- Layered security: RLS filters rows, DDM masks columns,
-- object permissions control table access

-- Layer 1: RLS ensures tenants only see their own rows
CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE Security.fn_TenantFilter(TenantID)
    ON dbo.CustomerData
WITH (STATE = ON, SCHEMABINDING = ON);

-- Layer 2: DDM masks sensitive columns for non-privileged users
ALTER TABLE dbo.CustomerData
ALTER COLUMN CreditCardNumber ADD MASKED WITH (FUNCTION = 'partial(0, "XXXX-XXXX-XXXX-", 4)');

ALTER TABLE dbo.CustomerData
ALTER COLUMN EmailAddress ADD MASKED WITH (FUNCTION = 'email()');

-- Layer 3: Object permissions restrict who can access the table
GRANT SELECT ON dbo.CustomerData TO [AppRole];
DENY SELECT ON dbo.CustomerData (CreditCardNumber) TO [BasicReportRole];

-- Layer 4: Always Encrypted for the most sensitive columns
-- (SSN, full credit card number stored encrypted at rest)

-- Verify the combined effect
EXEC sp_set_session_context @key = N'TenantID', @value = 42;
EXECUTE AS USER = 'BasicReportUser';
SELECT * FROM dbo.CustomerData;
-- Sees: only tenant 42 rows, masked credit card, masked email
REVERT;
```

## Performance Considerations

```sql
-- Check if RLS predicates appear in query plans
SET STATISTICS IO ON;
SET STATISTICS TIME ON;

-- The predicate function is inlined into every query plan
SELECT * FROM dbo.Orders WHERE Amount > 1000;
-- Execution plan shows the security predicate as a Filter operator

-- Index recommendations for RLS
-- Index the columns used in predicate functions
CREATE INDEX IX_Orders_TenantID ON dbo.Orders (TenantID);

-- For SESSION_CONTEXT-based predicates, the optimizer cannot use
-- statistics effectively because the value is unknown at compile time
-- Consider using OPTION (RECOMPILE) for critical queries
SELECT * FROM dbo.Orders
WHERE Status = 'Pending'
OPTION (RECOMPILE);

-- Monitor predicate function overhead
SELECT
    qs.total_worker_time / qs.execution_count AS AvgCPU,
    qs.total_logical_reads / qs.execution_count AS AvgReads,
    qs.execution_count,
    SUBSTRING(qt.text, qs.statement_start_offset / 2 + 1,
        (qs.statement_end_offset - qs.statement_start_offset) / 2 + 1) AS QueryText
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%fn_TenantFilter%'
ORDER BY AvgCPU DESC;

-- Avoid these anti-patterns in predicate functions:
-- 1. User-defined functions (non-inlineable)
-- 2. Subqueries that reference large tables without indexes
-- 3. Complex joins in the predicate function
-- 4. Non-sargable predicates (functions on columns)
```

## Best Practices

- Always use SCHEMABINDING on predicate functions to prevent accidental changes to dependent objects.
- Create predicate functions in a dedicated Security schema, separate from the tables they protect.
- Use SESSION_CONTEXT with @read_only = 1 for tenant identifiers to prevent tampering within a session.
- Index the columns referenced in predicate functions to minimize the performance impact of row filtering.
- Combine filter and block predicates together: filter prevents data leaks on reads, block prevents unauthorized writes.
- Test RLS policies by impersonating different users with EXECUTE AS and verifying both visible rows and blocked operations.
- Disable security policies during bulk load operations, then re-enable; verify data integrity after load.
- Use Dynamic Data Masking as a convenience layer, not a security boundary; users can potentially infer masked values through repeated queries.
- Grant UNMASK at the most specific scope possible (object level in 2022+) rather than at the database level.
- Document all security policies and their predicate logic; include them in database change management processes.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Relying on DDM as the sole security mechanism | Users with direct query access can infer masked values through WHERE clauses | Use DDM for casual exposure prevention; combine with RLS and column permissions for true security |
| Not using SCHEMABINDING on predicate functions | Dependent objects can be altered or dropped, breaking the policy silently | Always include WITH SCHEMABINDING on predicate function definitions |
| Predicate function references a large lookup table without indexes | Every query on the protected table triggers a full scan of the lookup table | Create appropriate indexes on lookup tables used by predicate functions |
| Forgetting to set SESSION_CONTEXT as read_only | Application code or SQL injection can change the tenant context mid-session | Use @read_only = 1 when setting security-critical session context values |
| Not adding block predicates alongside filter predicates | Users can insert rows for other tenants even though they cannot read them | Add AFTER INSERT and AFTER UPDATE block predicates to complement filter predicates |
| Granting UNMASK at the database level instead of object level | All masked columns in the entire database become visible | Use granular UNMASK on specific schemas or objects (SQL Server 2022+) |

## SQL Server Version Notes

- **SQL Server 2016**: Introduced Row-Level Security (filter and block predicates, security policies) and Dynamic Data Masking (default, email, random, partial functions). RLS predicates are always inlined. DDM UNMASK permission is database-scoped only.
- **SQL Server 2019**: Performance improvements for RLS predicate evaluation. Improved query optimizer handling of security predicates with better cardinality estimates. Memory-optimized tables support for RLS. Batch mode processing compatible with RLS predicates.
- **SQL Server 2022**: Granular UNMASK permissions at the schema, table, and column level (previously database-level only). Improved DDM performance for large result sets. Enhanced RLS compatibility with ledger tables. Better integration of RLS with query store for plan analysis of predicate overhead.

## Sources

- [Row-Level Security](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security)
- [CREATE SECURITY POLICY](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-security-policy-transact-sql)
- [Dynamic Data Masking](https://learn.microsoft.com/en-us/sql/relational-databases/security/dynamic-data-masking)
- [SESSION_CONTEXT](https://learn.microsoft.com/en-us/sql/t-sql/functions/session-context-transact-sql)
- [sp_set_session_context](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-set-session-context-transact-sql)
- [Security Predicates](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security#security-predicates)
