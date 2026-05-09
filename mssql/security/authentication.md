# SQL Server Authentication --- Logins, Identity Providers, and Access Control

## Overview

SQL Server supports multiple authentication mechanisms that control how users prove their identity before gaining access to the database engine. The two foundational modes are **Windows Authentication** (integrated with Active Directory) and **SQL Server Authentication** (username/password stored in SQL Server). Modern deployments add **Microsoft Entra ID** (formerly Azure AD) for cloud-integrated identity.

Choosing the right authentication mode depends on the deployment model, compliance requirements, and application architecture. Windows Authentication is preferred for on-premises environments because it leverages Kerberos and Group Policy. SQL Server Authentication is necessary when applications run on non-Windows platforms or need self-contained credentials. Entra ID authentication bridges on-premises and cloud identity for hybrid scenarios.

This skill covers authentication configuration, contained database users, Kerberos SPN setup, certificate-based authentication, password policies, login auditing, and Azure managed identity patterns.

## Key Concepts

- **Login**: A server-level principal that authenticates to the SQL Server instance. Logins map to database users.
- **Authentication Mode**: SQL Server can run in Windows Authentication Only mode or Mixed Mode (Windows + SQL Server authentication).
- **Contained Database User**: A database-level user that authenticates directly to the database without a server-level login.
- **SPN (Service Principal Name)**: A unique identifier for a Kerberos service, required for delegation and double-hop scenarios.
- **Managed Identity**: An Azure identity automatically managed by the platform, eliminating the need for credentials in code.
- **CHECK_POLICY**: Enforces Windows password policies (complexity, lockout) on SQL Server logins.

## Authentication Modes

```sql
-- Check current authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS WindowsAuthOnly;
-- 1 = Windows only, 0 = Mixed mode

-- Change authentication mode (requires restart)
-- Via T-SQL (registry update):
USE master;
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'LoginMode', REG_DWORD, 2;
-- 1 = Windows only, 2 = Mixed mode

-- Verify logins and their authentication type
SELECT
    name,
    type_desc,
    is_disabled,
    create_date,
    modify_date,
    LOGINPROPERTY(name, 'PasswordLastSetTime') AS PasswordLastSet
FROM sys.server_principals
WHERE type IN ('S', 'U', 'G', 'E', 'X')  -- SQL, Windows user, Windows group, Entra
ORDER BY type_desc, name;
```

## Windows Authentication

```sql
-- Create a login from a Windows domain account
CREATE LOGIN [DOMAIN\ServiceAccount] FROM WINDOWS
    WITH DEFAULT_DATABASE = AppDB;

-- Create a login from a Windows group
CREATE LOGIN [DOMAIN\DBAdmins] FROM WINDOWS
    WITH DEFAULT_DATABASE = master;

-- Map to a database user
USE AppDB;
CREATE USER [DOMAIN\ServiceAccount] FOR LOGIN [DOMAIN\ServiceAccount];
ALTER ROLE db_datareader ADD MEMBER [DOMAIN\ServiceAccount];

-- Grant server role to a Windows group (all group members inherit)
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\DBAdmins];
```

## SQL Server Authentication

```sql
-- Create a SQL Server login with password policy
CREATE LOGIN AppLogin
    WITH PASSWORD = 'Str0ng!P@ssw0rd#2024',
    DEFAULT_DATABASE = AppDB,
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = ON;

-- Map to a database user
USE AppDB;
CREATE USER AppUser FOR LOGIN AppLogin;
ALTER ROLE db_datareader ADD MEMBER AppUser;
ALTER ROLE db_datawriter ADD MEMBER AppUser;

-- Change password
ALTER LOGIN AppLogin WITH PASSWORD = 'N3w$ecure!Pass#2024';

-- Force password change on next login
ALTER LOGIN AppLogin WITH PASSWORD = 'T3mp!Pass#2024' MUST_CHANGE;

-- Unlock a locked account
ALTER LOGIN AppLogin WITH PASSWORD = 'T3mp!Pass#2024' UNLOCK;

-- Disable/enable a login
ALTER LOGIN AppLogin DISABLE;
ALTER LOGIN AppLogin ENABLE;
```

## Password Policies

```sql
-- CHECK_POLICY: enforces Windows password complexity and lockout
-- CHECK_EXPIRATION: enforces password age policies from Windows
CREATE LOGIN StrictLogin
    WITH PASSWORD = 'C0mpl3x!P@ss#2024',
    CHECK_POLICY = ON,
    CHECK_EXPIRATION = ON;

-- Query password policy status for all SQL logins
SELECT
    name,
    is_policy_checked,
    is_expiration_checked,
    is_disabled,
    LOGINPROPERTY(name, 'IsLocked') AS IsLocked,
    LOGINPROPERTY(name, 'IsExpired') AS IsExpired,
    LOGINPROPERTY(name, 'IsMustChange') AS MustChange,
    LOGINPROPERTY(name, 'LockoutTime') AS LockoutTime,
    LOGINPROPERTY(name, 'PasswordLastSetTime') AS PasswordLastSet,
    LOGINPROPERTY(name, 'BadPasswordCount') AS BadPasswordCount,
    LOGINPROPERTY(name, 'BadPasswordTime') AS LastBadPassword,
    LOGINPROPERTY(name, 'DaysUntilExpiration') AS DaysUntilExpiration
FROM sys.sql_logins
ORDER BY name;

-- Find logins without password policy enforcement
SELECT name, is_policy_checked, is_expiration_checked
FROM sys.sql_logins
WHERE is_policy_checked = 0 OR is_expiration_checked = 0;
```

## Contained Database Users

```sql
-- Enable contained databases at the instance level
EXEC sp_configure 'contained database authentication', 1;
RECONFIGURE;

-- Set a database to partial containment
ALTER DATABASE AppDB SET CONTAINMENT = PARTIAL;

-- Create a contained user with password (no server login needed)
USE AppDB;
CREATE USER ContainedAppUser WITH PASSWORD = 'C0nta1ned!P@ss#2024';
ALTER ROLE db_datareader ADD MEMBER ContainedAppUser;

-- Create a contained user from Windows authentication
CREATE USER [DOMAIN\AppUser] WITH DEFAULT_SCHEMA = dbo;

-- Migrate an existing login-based user to contained
EXEC sp_migrate_user_to_contained
    @username = 'AppUser',
    @rename = N'AppUser_contained',
    @disablelogin = N'disable';

-- List contained users
SELECT
    name,
    type_desc,
    authentication_type_desc,
    default_schema_name
FROM sys.database_principals
WHERE authentication_type > 0;
```

## Microsoft Entra ID (Azure AD) Authentication

```sql
-- Create a login from Entra ID (SQL Server 2022+ on-premises, or Azure SQL)
CREATE LOGIN [user@company.com] FROM EXTERNAL PROVIDER;

-- Create a login from an Entra ID group
CREATE LOGIN [EntraDBAdmins] FROM EXTERNAL PROVIDER;

-- Map to a database user
USE AppDB;
CREATE USER [user@company.com] FOR LOGIN [user@company.com];
ALTER ROLE db_datareader ADD MEMBER [user@company.com];

-- Azure SQL Database: create user directly (no login needed)
CREATE USER [user@company.com] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [user@company.com];

-- Set Entra ID admin (Azure SQL)
-- Done via Azure Portal, CLI, or PowerShell:
-- az sql server ad-admin create --resource-group myRG \
--     --server-name myServer --display-name "DBA Team" \
--     --object-id <entra-group-object-id>

-- Verify Entra ID configuration
SELECT
    name,
    type_desc,
    authentication_type_desc
FROM sys.database_principals
WHERE type IN ('E', 'X');  -- E = Entra user, X = Entra group
```

## Kerberos and SPN Configuration

```sql
-- Check if the current connection uses Kerberos
SELECT
    auth_scheme,
    net_transport,
    client_net_address,
    local_net_address
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;

-- Register SPNs (run from command line as domain admin)
-- setspn -S MSSQLSvc/sqlserver.domain.com:1433 DOMAIN\SQLServiceAccount
-- setspn -S MSSQLSvc/sqlserver.domain.com DOMAIN\SQLServiceAccount

-- Verify registered SPNs
-- setspn -L DOMAIN\SQLServiceAccount

-- Check for Kerberos issues in the error log
EXEC xp_readerrorlog 0, 1, N'SSPI';
EXEC xp_readerrorlog 0, 1, N'Kerberos';

-- Enable constrained delegation for linked servers (double-hop)
-- Configured in Active Directory: set the SQL service account for
-- "Trust this user for delegation to specified services only"
-- Then in SQL Server:
EXEC sp_addlinkedserver @server = 'RemoteServer',
    @srvproduct = '', @provider = 'SQLNCLI',
    @datasrc = 'remote.domain.com';

EXEC sp_addlinkedsrvlogin @rmtsrvname = 'RemoteServer',
    @useself = 'TRUE';  -- Uses Kerberos delegation
```

## Certificate-Based Authentication

```sql
-- Create a certificate for authentication
USE master;
CREATE CERTIFICATE AuthCert
    WITH SUBJECT = 'Login Authentication Certificate',
    START_DATE = '2024-01-01',
    EXPIRY_DATE = '2030-12-31';

-- Create a login mapped to the certificate
CREATE LOGIN CertLogin FROM CERTIFICATE AuthCert;
GRANT CONNECT SQL TO CertLogin;

-- Create a database user from the certificate
USE AppDB;
CREATE USER CertUser FROM CERTIFICATE AuthCert;
ALTER ROLE db_datareader ADD MEMBER CertUser;

-- Export certificate for use by applications
BACKUP CERTIFICATE AuthCert
    TO FILE = 'C:\Certs\AuthCert.cer'
    WITH PRIVATE KEY (
        FILE = 'C:\Certs\AuthCert.pvk',
        ENCRYPTION BY PASSWORD = 'C3rt!Backup#2024'
    );
```

## Azure Managed Identity

```sql
-- Use system-assigned managed identity (Azure SQL, Managed Instance)
-- No credentials in code; Azure handles token acquisition

-- Grant access to a managed identity in Azure SQL
CREATE USER [my-app-service] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [my-app-service];
ALTER ROLE db_datawriter ADD MEMBER [my-app-service];

-- Use user-assigned managed identity
CREATE USER [my-managed-identity-name] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [my-managed-identity-name];

-- Connection string for managed identity (application side)
-- Server=myserver.database.windows.net;Database=mydb;
-- Authentication=Active Directory Managed Identity;
```

## Login Auditing

```sql
-- Configure login auditing level via T-SQL
USE master;
EXEC xp_instance_regwrite
    N'HKEY_LOCAL_MACHINE',
    N'Software\Microsoft\MSSQLServer\MSSQLServer',
    N'AuditLevel', REG_DWORD, 3;
-- 0 = None, 1 = Successful logins, 2 = Failed logins, 3 = Both

-- Read login audit from error log
EXEC xp_readerrorlog 0, 1, N'Login failed';
EXEC xp_readerrorlog 0, 1, N'Login succeeded';

-- Query failed logins from the default trace
SELECT
    TextData,
    LoginName,
    HostName,
    ApplicationName,
    StartTime,
    ServerName
FROM fn_trace_gettable(
    (SELECT REVERSE(SUBSTRING(REVERSE(path),
        CHARINDEX('\', REVERSE(path)), 260)) + 'log.trc'
     FROM sys.traces WHERE is_default = 1), DEFAULT)
WHERE EventClass = 20  -- Login Failed
ORDER BY StartTime DESC;

-- Monitor active sessions and authentication method
SELECT
    s.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    c.auth_scheme,
    s.login_time,
    s.status
FROM sys.dm_exec_sessions s
JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
WHERE s.is_user_process = 1
ORDER BY s.login_time DESC;
```

## Best Practices

- Use Windows Authentication whenever possible; it provides Kerberos-based security and centralized password management.
- Enable CHECK_POLICY and CHECK_EXPIRATION on all SQL Server logins to enforce password complexity and rotation.
- Use contained database users for databases that may be moved or restored to different servers.
- Configure SPNs correctly to enable Kerberos; NTLM fallback is weaker and does not support delegation.
- Use managed identities in Azure to eliminate stored credentials entirely.
- Disable the sa login or rename it and assign a strong password.
- Audit failed login attempts and set up alerts for brute-force patterns.
- Use Entra ID groups rather than individual user accounts for easier management.
- Separate service account logins from interactive user logins.
- Rotate SQL Server login passwords on a regular schedule aligned with organizational policy.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Leaving sa enabled with a weak password | Complete server compromise via brute-force | Disable sa or use a 20+ character password with CHECK_POLICY ON |
| Not registering SPNs for the SQL service account | Kerberos fails silently, falls back to NTLM | Run setspn -S with the correct service account and verify with setspn -L |
| Using SQL Authentication without CHECK_POLICY | Passwords never expire, no complexity enforcement | Enable CHECK_POLICY and CHECK_EXPIRATION on all SQL logins |
| Hardcoding credentials in application config files | Credential exposure if config files are compromised | Use Windows Auth, managed identity, or Azure Key Vault references |
| Not disabling logins for departed employees | Stale accounts can be exploited | Implement a process to disable logins tied to AD account deprovisioning |
| Using TRUSTWORTHY instead of proper Kerberos delegation | Escalation path from any db_owner to sysadmin | Configure constrained Kerberos delegation and certificate signing |

## SQL Server Version Notes

- **SQL Server 2016**: No native Azure AD support for on-premises. Contained database users support Windows and SQL authentication. Always Encrypted introduces certificate-based column encryption that affects authentication workflows.
- **SQL Server 2019**: Added support for Azure AD authentication on Linux. Improved Always Encrypted with secure enclaves. Certificate-based authentication enhancements. Better integration with Azure Key Vault for credential management.
- **SQL Server 2022**: Full Microsoft Entra ID authentication support for on-premises SQL Server via Azure Arc. Introduced Entra ID login types (E, X) in sys.server_principals. Supports Azure managed identity for on-premises backups to Azure. Granular TLS 1.3 support for connections.

## Sources

- [Authentication Mode](https://learn.microsoft.com/en-us/sql/relational-databases/security/choose-an-authentication-mode)
- [CREATE LOGIN (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-login-transact-sql)
- [Contained Database Users](https://learn.microsoft.com/en-us/sql/relational-databases/security/contained-database-users-making-your-database-portable)
- [Microsoft Entra Authentication](https://learn.microsoft.com/en-us/sql/relational-databases/security/authentication-access/azure-ad-authentication-sql-server-overview)
- [Register an SPN for Kerberos](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections)
- [Password Policy](https://learn.microsoft.com/en-us/sql/relational-databases/security/password-policy)
