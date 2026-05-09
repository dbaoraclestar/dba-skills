# SQL Server Permissions --- GRANT, DENY, REVOKE and the Securable Hierarchy

## Overview

SQL Server implements a layered permission model organized around three core concepts: **principals** (who is requesting access), **securables** (what is being accessed), and **permissions** (what action is allowed). Every object in SQL Server lives inside a hierarchy of securables --- server, database, schema, and object --- and permissions can be granted, denied, or revoked at any level.

Understanding the permission hierarchy is essential for building least-privilege security architectures. A DENY at a higher scope overrides a GRANT at a lower scope, which means careless DENY statements can lock out users in unexpected ways. Conversely, ownership chaining allows stored procedures and views to access underlying objects without the caller needing direct permissions on those objects.

This skill covers the full permission lifecycle: granting granular permissions, using EXECUTE AS for impersonation, signing modules with certificates, configuring application roles, querying effective permissions, and managing cross-database access.

## Key Concepts

- **Principal**: An entity that can request access. Server-level principals include logins, server roles, and certificates. Database-level principals include users, database roles, application roles, and certificates.
- **Securable**: An object to which access can be controlled. The hierarchy is: Server > Database > Schema > Object (table, view, procedure, function, etc.).
- **Permission**: An action allowed or denied on a securable (SELECT, INSERT, EXECUTE, ALTER, CONTROL, etc.).
- **CONTROL**: A special permission that effectively grants ownership-like capabilities on a securable and all securables beneath it.
- **Ownership Chaining**: When a stored procedure or view accesses objects owned by the same principal, SQL Server skips permission checks on the underlying objects.
- **Impersonation**: The ability to execute code under the security context of another principal via EXECUTE AS.

## Permission Hierarchy

The server-database-schema-object hierarchy determines how permissions cascade.

```sql
-- Server-level permission: grant to a login
GRANT VIEW SERVER STATE TO [AppMonitorLogin];

-- Database-level permission: grant to a database user
USE AppDatabase;
GRANT CREATE TABLE TO [DevUser];

-- Schema-level permission: grant all permissions on a schema
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::Sales TO [SalesRole];

-- Object-level permission: grant on a specific table
GRANT SELECT ON OBJECT::Sales.Orders TO [ReportUser];

-- Column-level permission: restrict to specific columns
GRANT SELECT ON OBJECT::HR.Employees (EmployeeID, FirstName, LastName) TO [LimitedUser];
DENY SELECT ON OBJECT::HR.Employees (Salary, SSN) TO [LimitedUser];
```

## GRANT, DENY, and REVOKE

```sql
-- GRANT: allows the permission
GRANT EXECUTE ON OBJECT::dbo.usp_GetCustomer TO [AppUser];

-- GRANT WITH GRANT OPTION: allows the grantee to grant to others
GRANT SELECT ON SCHEMA::Sales TO [SalesManager] WITH GRANT OPTION;

-- DENY: explicitly blocks the permission (overrides GRANT)
DENY DELETE ON OBJECT::Sales.Orders TO [JuniorDev];

-- REVOKE: removes a previously granted or denied permission
REVOKE SELECT ON SCHEMA::Sales FROM [FormerEmployee];

-- REVOKE GRANT OPTION FOR: removes ability to grant to others
REVOKE GRANT OPTION FOR SELECT ON SCHEMA::Sales FROM [SalesManager];
```

## Server-Level Permissions

```sql
-- Common server-level permissions
GRANT VIEW SERVER STATE TO [MonitorLogin];        -- DMV access
GRANT ALTER ANY DATABASE TO [DBALogin];           -- Create/alter databases
GRANT ALTER ANY LOGIN TO [SecurityAdmin];         -- Manage logins
GRANT CONNECT SQL TO [AppLogin];                  -- Basic connectivity

-- Server roles (fixed)
ALTER SERVER ROLE sysadmin ADD MEMBER [SeniorDBA];
ALTER SERVER ROLE securityadmin ADD MEMBER [SecurityTeam];
ALTER SERVER ROLE dbcreator ADD MEMBER [DevLead];

-- User-defined server roles (SQL Server 2012+)
CREATE SERVER ROLE [AppAdmins];
GRANT ALTER ANY DATABASE TO [AppAdmins];
GRANT VIEW ANY DEFINITION TO [AppAdmins];
ALTER SERVER ROLE [AppAdmins] ADD MEMBER [AppTeamLogin];
```

## Database Roles and Object Permissions

```sql
-- Create a custom database role
CREATE ROLE [SalesReadWrite];
GRANT SELECT, INSERT, UPDATE ON SCHEMA::Sales TO [SalesReadWrite];
DENY DELETE ON SCHEMA::Sales TO [SalesReadWrite];

-- Add users to the role
ALTER ROLE [SalesReadWrite] ADD MEMBER [SalesApp];
ALTER ROLE [SalesReadWrite] ADD MEMBER [JohnDoe];

-- Nested roles
CREATE ROLE [SalesAdmin];
ALTER ROLE [SalesReadWrite] ADD MEMBER [SalesAdmin];
GRANT DELETE ON SCHEMA::Sales TO [SalesAdmin];
GRANT EXECUTE ON SCHEMA::Sales TO [SalesAdmin];
```

## Ownership Chaining

```sql
-- When the proc and table share the same owner, callers only need
-- EXECUTE on the proc; they do NOT need SELECT on the table.
CREATE PROCEDURE dbo.usp_GetActiveOrders
AS
BEGIN
    SELECT OrderID, CustomerID, OrderDate, TotalAmount
    FROM dbo.Orders
    WHERE Status = 'Active';
END;
GO

GRANT EXECUTE ON dbo.usp_GetActiveOrders TO [AppUser];
-- AppUser can call the proc even without SELECT on dbo.Orders

-- Cross-schema ownership chaining (disabled by default)
-- Enable for a specific database:
ALTER DATABASE AppDB SET DB_CHAINING ON;

-- Cross-database ownership chaining (use with caution)
EXEC sp_configure 'cross db ownership chaining', 1;
RECONFIGURE;
```

## EXECUTE AS (Impersonation)

```sql
-- Module-level impersonation: proc runs as a specific user
CREATE PROCEDURE dbo.usp_AdminTask
WITH EXECUTE AS 'AdminUser'
AS
BEGIN
    -- This code runs with AdminUser's permissions
    TRUNCATE TABLE dbo.StagingData;
END;
GO

-- Session-level impersonation
EXECUTE AS LOGIN = 'AppLogin';
SELECT SUSER_NAME() AS CurrentLogin;
REVERT;  -- Return to original context

-- EXECUTE AS OWNER: runs as the object owner
CREATE PROCEDURE dbo.usp_OwnerTask
WITH EXECUTE AS OWNER
AS
BEGIN
    ALTER INDEX ALL ON dbo.LargeTable REBUILD;
END;
GO

-- Check impersonation permission
SELECT HAS_PERMS_BY_NAME('AppLogin', 'LOGIN', 'IMPERSONATE');
```

## Certificate-Signed Modules

```sql
-- Create a certificate in master for server-level operations
USE master;
CREATE CERTIFICATE ServerTaskCert
    WITH SUBJECT = 'Certificate for server admin tasks';

-- Create a login mapped to the certificate
CREATE LOGIN ServerTaskLogin FROM CERTIFICATE ServerTaskCert;
GRANT VIEW SERVER STATE TO ServerTaskLogin;

-- Sign the stored procedure with the certificate
USE AppDB;
CREATE CERTIFICATE ServerTaskCert FROM CERTIFICATE master.dbo.ServerTaskCert;
CREATE USER ServerTaskUser FROM CERTIFICATE ServerTaskCert;

ADD SIGNATURE TO dbo.usp_ServerMonitor BY CERTIFICATE ServerTaskCert;

-- Now usp_ServerMonitor has VIEW SERVER STATE without the caller
-- needing that permission directly
```

## Application Roles

```sql
-- Create an application role with a password
CREATE APPLICATION ROLE [SalesApp]
    WITH PASSWORD = 'C0mpl3x!P@ssw0rd';

-- Grant permissions to the application role
GRANT SELECT, INSERT, UPDATE ON SCHEMA::Sales TO [SalesApp];
GRANT EXECUTE ON SCHEMA::Sales TO [SalesApp];

-- Activate in the application (connection-level)
DECLARE @cookie VARBINARY(8000);
EXEC sp_setapprole 'SalesApp', 'C0mpl3x!P@ssw0rd',
    @fCreateCookie = true, @cookie = @cookie OUTPUT;

-- Perform operations under the application role context
SELECT * FROM Sales.Customers;

-- Deactivate the application role
EXEC sp_unsetapprole @cookie;
```

## Cross-Database Permissions

```sql
-- Option 1: Certificate-based cross-database access
USE TargetDB;
CREATE CERTIFICATE CrossDBCert FROM CERTIFICATE SourceDB.dbo.CrossDBCert;
CREATE USER CrossDBUser FROM CERTIFICATE CrossDBCert;
GRANT SELECT ON SCHEMA::dbo TO CrossDBUser;

-- Option 2: Module signing for cross-database proc
USE SourceDB;
CREATE PROCEDURE dbo.usp_ReadFromTargetDB
AS
BEGIN
    SELECT * FROM TargetDB.dbo.ReferenceData;
END;
GO
ADD SIGNATURE TO dbo.usp_ReadFromTargetDB BY CERTIFICATE CrossDBCert;

-- Option 3: EXECUTE AS with TRUSTWORTHY (less secure)
ALTER DATABASE SourceDB SET TRUSTWORTHY ON;
CREATE PROCEDURE dbo.usp_CrossDB
WITH EXECUTE AS 'dbo'
AS
BEGIN
    SELECT * FROM TargetDB.dbo.ReferenceData;
END;
GO
```

## Querying Effective Permissions

```sql
-- What permissions does the current user have on a specific object?
SELECT *
FROM fn_my_permissions('Sales.Orders', 'OBJECT')
ORDER BY subentity_name, permission_name;

-- What server-level permissions does the current user have?
SELECT *
FROM fn_my_permissions(NULL, 'SERVER')
ORDER BY permission_name;

-- What database-level permissions does the current user have?
SELECT *
FROM fn_my_permissions(NULL, 'DATABASE')
ORDER BY permission_name;

-- Check a specific permission programmatically
IF HAS_PERMS_BY_NAME('Sales.Orders', 'OBJECT', 'SELECT') = 1
    PRINT 'User has SELECT permission on Sales.Orders';

-- All explicitly granted/denied permissions for a principal
SELECT
    dp.name AS PrincipalName,
    dp.type_desc AS PrincipalType,
    p.permission_name,
    p.state_desc AS PermissionState,
    p.class_desc AS SecurableClass,
    OBJECT_NAME(p.major_id) AS SecurableName
FROM sys.database_permissions p
JOIN sys.database_principals dp ON p.grantee_principal_id = dp.principal_id
WHERE dp.name = 'AppUser'
ORDER BY p.class_desc, OBJECT_NAME(p.major_id);

-- Role membership chain for a user
WITH RoleMembership AS (
    SELECT
        m.member_principal_id,
        m.role_principal_id,
        dp_member.name AS MemberName,
        dp_role.name AS RoleName,
        1 AS Depth
    FROM sys.database_role_members m
    JOIN sys.database_principals dp_member ON m.member_principal_id = dp_member.principal_id
    JOIN sys.database_principals dp_role ON m.role_principal_id = dp_role.principal_id
    WHERE dp_member.name = 'AppUser'
    UNION ALL
    SELECT
        rm.role_principal_id,
        m.role_principal_id,
        rm.RoleName,
        dp_role.name,
        rm.Depth + 1
    FROM sys.database_role_members m
    JOIN RoleMembership rm ON m.member_principal_id = rm.role_principal_id
    JOIN sys.database_principals dp_role ON m.role_principal_id = dp_role.principal_id
)
SELECT MemberName, RoleName, Depth FROM RoleMembership ORDER BY Depth;
```

## Best Practices

- Follow the principle of least privilege: grant only the permissions needed for a specific task.
- Prefer database roles over granting permissions directly to users; roles simplify management and auditing.
- Avoid using DENY unless absolutely necessary; it overrides all GRANTs and complicates troubleshooting.
- Use certificate-signed modules instead of TRUSTWORTHY for cross-database access.
- Never grant CONTROL SERVER or sysadmin to application accounts.
- Use EXECUTE AS at the module level (stored procedures, functions) rather than session level.
- Regularly audit effective permissions using fn_my_permissions and sys.database_permissions.
- Use schema-level permissions when a role needs access to all objects within a schema.
- Create custom server roles (2012+) instead of adding logins to fixed server roles.
- Review ownership chains and document them; unintended chains can expose data.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Granting sysadmin to application logins | Full server control, bypasses all permission checks | Create a custom role with only the required permissions |
| Using DENY at database level to block one table | Blocks access to that object even through role grants | REVOKE the GRANT instead of using DENY, or use column-level DENY |
| Setting TRUSTWORTHY ON without understanding impact | Any dbo-context code can access other databases | Use certificate signing for cross-database access |
| Not revoking permissions when users change roles | Accumulated permissions exceed job requirements | Audit permissions quarterly; use roles for easy add/remove |
| Granting WITH GRANT OPTION to non-admin users | Uncontrolled permission proliferation | Reserve WITH GRANT OPTION for security administrators |
| Ignoring orphaned users after login changes | Users retain database access with no valid login | Run EXEC sp_change_users_login 'Report' or ALTER USER regularly |

## SQL Server Version Notes

- **SQL Server 2016**: Introduced Always Encrypted (impacts permission model for encrypted columns), row-level security predicates, dynamic data masking. CONTROL permission on column master key required for Always Encrypted.
- **SQL Server 2019**: Added GRANT CONNECT to availability group listener endpoints. Improved certificate management for Always Encrypted with secure enclaves. Supports managed instance link permissions.
- **SQL Server 2022**: Introduced Microsoft Entra ID (formerly Azure AD) authentication for on-premises SQL Server. Granular server-level permissions for Azure Arc-enabled instances. Ledger table permissions (ALTER LEDGER). Improved contained availability group user management.

## Sources

- [Permissions (Database Engine)](https://learn.microsoft.com/en-us/sql/relational-databases/security/permissions-database-engine)
- [GRANT (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/grant-transact-sql)
- [Ownership Chains](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/ownership-chains)
- [EXECUTE AS (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-transact-sql)
- [Application Roles](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/application-roles)
- [fn_my_permissions (Transact-SQL)](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-my-permissions-transact-sql)
- [Certificate-Signed Modules](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/module-signing)
