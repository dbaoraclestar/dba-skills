# SQL Server User Management — Authentication, Authorization, and Access Control

## Overview

SQL Server user management encompasses the full lifecycle of identity and access control: creating logins at the server level, mapping them to database users, assigning permissions through roles and schemas, and auditing access patterns. A solid grasp of the two-tier security model (server-level logins vs. database-level users) is essential for every DBA.

This skill covers SQL Server authentication (mixed mode), Windows/Active Directory authentication, Azure AD/Entra ID integration, contained database users, role-based access control, schema ownership, cross-database access patterns, and orphaned user remediation. Use this guide whenever you need to provision access, audit permissions, troubleshoot login failures, or migrate users between instances.

Whether you are managing a single standalone instance or a multi-replica Always On environment, the principles here apply. The key difference in AG environments is that logins must exist on every replica with matching SIDs, and contained database users avoid that problem entirely.

## Key Concepts

- **Login**: A server-level principal that grants the ability to connect to the SQL Server instance. Logins can be SQL Server authenticated (username/password) or Windows/Entra ID authenticated.
- **User**: A database-level principal mapped to a login. Users exist within a single database and hold permissions for objects in that database.
- **Server Role**: A fixed or user-defined collection of server-level permissions (e.g., sysadmin, securityadmin).
- **Database Role**: A fixed or user-defined collection of database-level permissions (e.g., db_owner, db_datareader).
- **Schema**: A namespace for database objects that also serves as a security boundary. Users can own schemas and inherit permissions on all objects within them.
- **SID (Security Identifier)**: The internal binary identifier that links a login to a user. SID mismatches cause orphaned users.
- **Contained Database User**: A user authenticated at the database level without requiring a server-level login. Portable across instances.
- **Principal**: Any entity that can request SQL Server resources (logins, users, roles, application roles).

## Creating Logins

### SQL Server Authentication Login

```sql
-- Basic login with password policy enforcement
CREATE LOGIN AppServiceAccount
WITH PASSWORD = 'C0mpl3x!P@ssw0rd2024',
     DEFAULT_DATABASE = [ApplicationDB],
     CHECK_EXPIRATION = OFF,
     CHECK_POLICY = ON;

-- Login with a specific SID (useful for AG replicas or migration)
CREATE LOGIN AppServiceAccount
WITH PASSWORD = 'C0mpl3x!P@ssw0rd2024',
     SID = 0x1234567890ABCDEF1234567890ABCDEF,
     DEFAULT_DATABASE = [ApplicationDB],
     CHECK_EXPIRATION = OFF,
     CHECK_POLICY = ON;

-- Disable a login without dropping it
ALTER LOGIN AppServiceAccount DISABLE;

-- Change password (requires sysadmin or the login itself)
ALTER LOGIN AppServiceAccount
WITH PASSWORD = 'N3wC0mpl3x!P@ss2024'
     OLD_PASSWORD = 'C0mpl3x!P@ssw0rd2024';

-- Unlock a locked-out login
ALTER LOGIN AppServiceAccount
WITH PASSWORD = 'C0mpl3x!P@ssw0rd2024' UNLOCK;
```

### Windows Authentication Login

```sql
-- Domain user login
CREATE LOGIN [DOMAIN\jsmith] FROM WINDOWS
WITH DEFAULT_DATABASE = [ApplicationDB];

-- Domain group login (preferred for manageability)
CREATE LOGIN [DOMAIN\DBA_Team] FROM WINDOWS
WITH DEFAULT_DATABASE = [master];

-- Grant a Windows group the sysadmin role
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\DBA_Team];
```

### Azure AD / Entra ID Authentication

```sql
-- Requires Azure AD admin configured on the SQL Server / Managed Instance
-- Create login from Azure AD user
CREATE LOGIN [user@company.com] FROM EXTERNAL PROVIDER;

-- Create login from Azure AD group
CREATE LOGIN [DBA_EntraGroup] FROM EXTERNAL PROVIDER;

-- Create a contained database user from Azure AD (no login needed)
USE [ApplicationDB];
CREATE USER [user@company.com] FROM EXTERNAL PROVIDER;
```

## Creating Database Users

```sql
USE [ApplicationDB];

-- Map a user to an existing login
CREATE USER [AppUser] FOR LOGIN [AppServiceAccount]
WITH DEFAULT_SCHEMA = [app];

-- User without login (for cross-database chaining or certificate-based access)
CREATE USER [CertBasedUser] WITHOUT LOGIN;

-- Map a Windows login
CREATE USER [DOMAIN\jsmith] FOR LOGIN [DOMAIN\jsmith]
WITH DEFAULT_SCHEMA = [dbo];

-- Create a user mapped to a Windows group
CREATE USER [DOMAIN\DBA_Team] FOR LOGIN [DOMAIN\DBA_Team];
```

## Contained Database Users

```sql
-- First, enable contained database authentication at instance level
EXEC sp_configure 'contained database authentication', 1;
RECONFIGURE;

-- Set database to partial containment
ALTER DATABASE [ApplicationDB] SET CONTAINMENT = PARTIAL;

-- Create a contained user with password (no login required)
USE [ApplicationDB];
CREATE USER [ContainedAppUser]
WITH PASSWORD = 'C0mpl3x!P@ssw0rd2024',
     DEFAULT_SCHEMA = [app];

-- Contained Windows user
CREATE USER [DOMAIN\jsmith] WITH DEFAULT_SCHEMA = [dbo];

-- Connect using contained user
-- Connection string must specify database name:
-- Server=myserver;Database=ApplicationDB;User Id=ContainedAppUser;Password=...
```

## Server Roles

### Fixed Server Roles

```sql
-- Add a login to a fixed server role
ALTER SERVER ROLE sysadmin ADD MEMBER [AppServiceAccount];
ALTER SERVER ROLE securityadmin ADD MEMBER [DOMAIN\SecurityTeam];
ALTER SERVER ROLE dbcreator ADD MEMBER [DOMAIN\DevLead];
ALTER SERVER ROLE serveradmin ADD MEMBER [DOMAIN\DBA_Junior];

-- Remove from server role
ALTER SERVER ROLE sysadmin DROP MEMBER [AppServiceAccount];

-- List all members of a server role
SELECT
    r.name AS RoleName,
    m.name AS MemberName,
    m.type_desc AS MemberType
FROM sys.server_role_members srm
JOIN sys.server_principals r ON srm.role_principal_id = r.principal_id
JOIN sys.server_principals m ON srm.member_principal_id = m.principal_id
ORDER BY r.name, m.name;
```

**Fixed Server Roles Reference:**

| Role | Purpose |
|------|---------|
| sysadmin | Full control over the instance |
| securityadmin | Manage logins and permissions |
| serveradmin | Configure server-wide settings |
| setupadmin | Manage linked servers |
| processadmin | Kill processes |
| diskadmin | Manage disk files |
| dbcreator | Create and alter databases |
| bulkadmin | Run BULK INSERT |
| public | Default role for every login |

### User-Defined Server Roles (SQL Server 2012+)

```sql
-- Create a custom server role for monitoring
CREATE SERVER ROLE MonitoringRole;

-- Grant specific server-level permissions
GRANT VIEW SERVER STATE TO MonitoringRole;
GRANT VIEW ANY DATABASE TO MonitoringRole;
GRANT VIEW ANY DEFINITION TO MonitoringRole;
GRANT ALTER TRACE TO MonitoringRole;

-- Add members
ALTER SERVER ROLE MonitoringRole ADD MEMBER [DOMAIN\MonitoringService];

-- Audit: list custom server roles and their permissions
SELECT
    pr.name AS RoleName,
    pe.permission_name,
    pe.state_desc
FROM sys.server_principals pr
JOIN sys.server_permissions pe ON pr.principal_id = pe.grantee_principal_id
WHERE pr.type = 'R' AND pr.is_fixed_role = 0;
```

## Database Roles

### Fixed Database Roles

```sql
USE [ApplicationDB];

-- Add user to database roles
ALTER ROLE db_datareader ADD MEMBER [AppUser];
ALTER ROLE db_datawriter ADD MEMBER [AppUser];

-- Read-only access
ALTER ROLE db_datareader ADD MEMBER [ReportUser];

-- Full database control (use sparingly)
ALTER ROLE db_owner ADD MEMBER [AppAdmin];

-- DDL-only access
ALTER ROLE db_ddladmin ADD MEMBER [DOMAIN\DevLead];

-- Remove from role
ALTER ROLE db_datawriter DROP MEMBER [AppUser];
```

**Fixed Database Roles Reference:**

| Role | Purpose |
|------|---------|
| db_owner | Full control within the database |
| db_securityadmin | Manage roles and permissions |
| db_accessadmin | Manage user access |
| db_ddladmin | Run DDL commands |
| db_datareader | SELECT on all tables/views |
| db_datawriter | INSERT/UPDATE/DELETE on all tables |
| db_backupoperator | Run BACKUP commands |
| db_denydatareader | Cannot SELECT (overrides grants) |
| db_denydatawriter | Cannot modify data (overrides grants) |

### Custom Database Roles

```sql
USE [ApplicationDB];

-- Create a custom role for the application tier
CREATE ROLE AppServiceRole;

-- Grant granular permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::app TO AppServiceRole;
GRANT EXECUTE ON SCHEMA::app TO AppServiceRole;
DENY DELETE ON [app].[AuditLog] TO AppServiceRole;

-- Add users to the custom role
ALTER ROLE AppServiceRole ADD MEMBER [AppUser];

-- Create a read-only role for specific schemas
CREATE ROLE ReportingRole;
GRANT SELECT ON SCHEMA::reporting TO ReportingRole;
GRANT SELECT ON SCHEMA::lookup TO ReportingRole;
```

## Schema Management

```sql
USE [ApplicationDB];

-- Create a schema owned by a role
CREATE SCHEMA [app] AUTHORIZATION [AppServiceRole];

-- Transfer object ownership to a schema
ALTER SCHEMA [app] TRANSFER [dbo].[Orders];

-- Change schema owner
ALTER AUTHORIZATION ON SCHEMA::[app] TO [dbo];

-- List all schemas and their owners
SELECT
    s.name AS SchemaName,
    p.name AS OwnerName,
    p.type_desc AS OwnerType
FROM sys.schemas s
JOIN sys.database_principals p ON s.principal_id = p.principal_id
ORDER BY s.name;
```

## Orphaned Users

```sql
-- Detect orphaned users (users without matching logins)
SELECT
    dp.name AS UserName,
    dp.sid AS UserSID,
    dp.type_desc
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type IN ('S', 'U')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.sid IS NULL;

-- Alternative: use the built-in report
EXEC sp_change_users_login 'Report';  -- Deprecated but still works

-- Fix orphaned user by remapping to existing login
ALTER USER [AppUser] WITH LOGIN = [AppServiceAccount];

-- Fix orphaned user and create login if needed
-- Step 1: Get the SID from the orphaned user
SELECT name, sid FROM sys.database_principals WHERE name = 'AppUser';

-- Step 2: Create login with matching SID
CREATE LOGIN [AppServiceAccount]
WITH PASSWORD = 'C0mpl3x!P@ssw0rd2024',
     SID = 0x<sid_from_step_1>;

-- Bulk fix: remap all orphaned users (generate script)
SELECT
    'ALTER USER [' + dp.name + '] WITH LOGIN = [' + dp.name + '];'
FROM sys.database_principals dp
LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
WHERE dp.type = 'S'
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
  AND sp.sid IS NULL;
```

## EXECUTE AS and Impersonation

```sql
-- Impersonate a login at server level
EXECUTE AS LOGIN = 'AppServiceAccount';
-- ... do work ...
REVERT;

-- Impersonate a user at database level
EXECUTE AS USER = 'AppUser';
SELECT USER_NAME(), SUSER_SNAME();
-- ... test permissions ...
REVERT;

-- Grant impersonation rights
GRANT IMPERSONATE ON LOGIN::[AppServiceAccount] TO [DOMAIN\DBA_Admin];
GRANT IMPERSONATE ON USER::[AppUser] TO [TestUser];

-- Stored procedure with EXECUTE AS for elevated access
CREATE PROCEDURE [app].[GetSensitiveData]
WITH EXECUTE AS 'ElevatedUser'
AS
BEGIN
    SELECT * FROM [secure].[SensitiveTable];
END;
```

## Cross-Database Access

```sql
-- Enable cross-database ownership chaining (instance-level)
EXEC sp_configure 'cross db ownership chaining', 1;
RECONFIGURE;

-- Or enable per-database
ALTER DATABASE [ApplicationDB] SET DB_CHAINING ON;

-- Grant CONNECT to another database
USE [OtherDB];
CREATE USER [AppUser] FOR LOGIN [AppServiceAccount];
GRANT SELECT ON SCHEMA::shared TO [AppUser];

-- Using certificates for cross-database access (more secure)
-- Step 1: Create certificate in source database
USE [ApplicationDB];
CREATE CERTIFICATE CrossDBCert
WITH SUBJECT = 'Certificate for cross-database access';

-- Step 2: Create user from certificate in target database
BACKUP CERTIFICATE CrossDBCert TO FILE = 'C:\temp\CrossDBCert.cer';

USE [OtherDB];
CREATE CERTIFICATE CrossDBCert FROM FILE = 'C:\temp\CrossDBCert.cer';
CREATE USER CertUser FOR CERTIFICATE CrossDBCert;
GRANT SELECT ON SCHEMA::shared TO CertUser;

-- Step 3: Sign the stored procedure
USE [ApplicationDB];
ADD SIGNATURE TO [app].[CrossDBProc] BY CERTIFICATE CrossDBCert;
```

## Permission Auditing

```sql
-- All effective permissions for the current user
SELECT * FROM fn_my_permissions(NULL, 'SERVER');
SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
SELECT * FROM fn_my_permissions('dbo.Orders', 'OBJECT');

-- All permissions granted in a database
SELECT
    pr.name AS PrincipalName,
    pr.type_desc AS PrincipalType,
    pe.permission_name,
    pe.state_desc,
    COALESCE(o.name, pe.class_desc) AS ObjectName,
    s.name AS SchemaName
FROM sys.database_permissions pe
JOIN sys.database_principals pr ON pe.grantee_principal_id = pr.principal_id
LEFT JOIN sys.objects o ON pe.major_id = o.object_id AND pe.class = 1
LEFT JOIN sys.schemas s ON o.schema_id = s.schema_id
WHERE pr.name NOT IN ('public', 'guest')
ORDER BY pr.name, pe.permission_name;

-- Role membership hierarchy
WITH RoleHierarchy AS (
    SELECT
        r.name AS RoleName,
        m.name AS MemberName,
        m.type_desc AS MemberType,
        1 AS Level
    FROM sys.database_role_members drm
    JOIN sys.database_principals r ON drm.role_principal_id = r.principal_id
    JOIN sys.database_principals m ON drm.member_principal_id = m.principal_id
)
SELECT * FROM RoleHierarchy
ORDER BY RoleName, MemberName;

-- Who has sysadmin?
SELECT name, type_desc, is_disabled, create_date, modify_date
FROM sys.server_principals
WHERE IS_SRVROLEMEMBER('sysadmin', name) = 1
ORDER BY name;
```

## PowerShell User Management

```powershell
# List all logins on an instance
Import-Module SqlServer
$server = Get-SqlInstance -ServerInstance "SQLPROD01"
$server.Logins | Select-Object Name, LoginType, IsDisabled, CreateDate |
    Format-Table -AutoSize

# Create a login
$securePassword = ConvertTo-SecureString "C0mpl3x!P@ss" -AsPlainText -Force
Add-SqlLogin -ServerInstance "SQLPROD01" `
    -LoginName "NewAppLogin" `
    -LoginType SqlLogin `
    -DefaultDatabase "ApplicationDB" `
    -EnforcePasswordPolicy `
    -SecurePassword $securePassword

# Sync logins across AG replicas
# Export logins with SIDs from primary
$logins = Invoke-Sqlcmd -ServerInstance "PRIMARY01" -Query "
    SELECT name, sid, LOGINPROPERTY(name, 'PasswordHash') AS PasswordHash
    FROM sys.sql_logins
    WHERE name NOT LIKE '##%' AND name NOT IN ('sa')
"

# Recreate on secondary with matching SID
foreach ($login in $logins) {
    $sid = "0x" + [BitConverter]::ToString($login.sid).Replace("-","")
    $hash = "0x" + [BitConverter]::ToString($login.PasswordHash).Replace("-","")
    $sql = "IF NOT EXISTS (SELECT 1 FROM sys.server_principals WHERE name = '$($login.name)')
            CREATE LOGIN [$($login.name)]
            WITH PASSWORD = $hash HASHED, SID = $sid,
            CHECK_POLICY = OFF;"
    Invoke-Sqlcmd -ServerInstance "SECONDARY01" -Query $sql
}
```

## Best Practices

- Use Windows Authentication or Entra ID over SQL Authentication whenever possible for centralized credential management and Kerberos support.
- Assign permissions to roles, not individual users. This simplifies onboarding, offboarding, and auditing.
- Use contained database users in Always On AG environments to eliminate orphaned-user issues during failover.
- Never grant sysadmin to application service accounts. Create custom server roles with least-privilege permissions.
- Use schemas as security boundaries and grant permissions at the schema level rather than object-by-object.
- Implement a regular login audit process: identify disabled logins, orphaned users, and overprivileged accounts quarterly.
- When migrating databases, always script logins with their SIDs to prevent orphaned users on the destination.
- Avoid the `sa` login for applications. Rename it and disable it if not needed, or use a strong password and audit its use.
- Use DENY sparingly and document every DENY permission, since DENY overrides GRANT and can cause subtle access failures.
- Document your role hierarchy and permission model. Use a permission matrix spreadsheet for complex environments.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Granting sysadmin to application accounts | Full instance compromise if app is breached | Create custom server role with minimal required permissions |
| Ignoring orphaned users after database restore/migration | Users cannot connect; application failures | Run orphaned user detection query after every restore and remap with ALTER USER |
| Using `sa` account for application connections | No audit trail; full privilege escalation | Create dedicated SQL login with least privilege; disable or rename `sa` |
| Not syncing logins across AG replicas | Login failures after failover | Script logins with SIDs and deploy to all replicas; or use contained users |
| Granting db_owner instead of specific permissions | Users can drop tables, alter schema, change permissions | Create custom database role with only SELECT/INSERT/UPDATE/DELETE/EXECUTE as needed |
| Using CHECK_EXPIRATION = ON for service accounts | Service account locks out; application outage | Set CHECK_EXPIRATION = OFF for non-interactive service accounts |
| Forgetting to revoke access when employees leave | Stale accounts become attack vectors | Integrate with AD group-based access; automate deprovisioning |
| Mixing contained and non-contained users | Confusion about authentication path; duplicate users | Choose one strategy per database and document it |

## SQL Server Version Notes

- **SQL Server 2012**: Introduced contained database users and user-defined server roles.
- **SQL Server 2016**: Added Always Encrypted (column-level encryption impacting user access patterns), Row-Level Security, and Dynamic Data Masking.
- **SQL Server 2017**: Linux support introduced; Windows Authentication via AD on Linux requires additional Kerberos configuration.
- **SQL Server 2019**: Added certificate-based authentication improvements, Always Encrypted with Secure Enclaves for richer queries on encrypted data, and improved contained AG support.
- **SQL Server 2022**: Native Azure AD (Entra ID) authentication for on-premises SQL Server, Microsoft Purview integration for data governance, ledger tables for tamper-evident auditing, and contained AG improvements eliminating the need for login sync in many scenarios.

## Sources

- [CREATE LOGIN (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)
- [CREATE USER (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-user-transact-sql)
- [Server-Level Roles](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/server-level-roles)
- [Database-Level Roles](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/database-level-roles)
- [Contained Database Users](https://learn.microsoft.com/en-us/sql/relational-databases/security/contained-database-users-making-your-database-portable)
- [Troubleshoot Orphaned Users](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/troubleshoot-orphaned-users-sql-server)
- [Azure AD Authentication for SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview)
- [EXECUTE AS (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/execute-as-transact-sql)
