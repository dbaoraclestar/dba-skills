# SQL Server Auditing --- Server Audit, Audit Specifications, and Compliance

## Overview

SQL Server Audit provides a comprehensive framework for tracking and logging database engine events. Built on Extended Events infrastructure, the audit system operates at two levels: **server audit specifications** capture instance-level events (login attempts, server role changes, backup operations), and **database audit specifications** capture database-level events (SELECT, INSERT, UPDATE, DELETE on specific objects, schema changes, permission modifications).

Audit data can be written to files on disk, the Windows Security Event Log, the Windows Application Event Log, or Azure Blob Storage. The file-based target is the most common for on-premises deployments because it offers the best performance, supports large volumes, and allows automated processing via the `fn_get_audit_file` function.

A well-designed audit strategy addresses regulatory compliance requirements (PCI-DSS, HIPAA, SOX, GDPR) while minimizing performance overhead. This means auditing security-critical actions at the server level, selectively auditing data access at the database level, and filtering events to capture only what compliance and security teams need.

## Key Concepts

- **Server Audit**: The container object that defines where audit data is written (file, event log, blob). Multiple specifications can share one audit.
- **Server Audit Specification**: Defines which server-level action groups to capture (logins, configuration changes, etc.).
- **Database Audit Specification**: Defines which database-level actions to capture on specific securables (tables, schemas, etc.).
- **Action Group**: A predefined collection of related events. Examples: `BATCH_COMPLETED_GROUP`, `DATABASE_OBJECT_ACCESS_GROUP`, `LOGIN_CHANGE_PASSWORD_GROUP`.
- **Audit Action**: A specific DML or DDL operation on a named securable (e.g., SELECT on a specific table by a specific principal).
- **C2 Audit Mode**: A legacy compliance feature that logs all SQL statements; replaced by SQL Server Audit in modern versions.
- **Common Criteria Compliance**: An ISO/IEC 15408 certification level that SQL Server can be configured to meet.

## Creating a Server Audit

```sql
-- Create audit writing to file
CREATE SERVER AUDIT SecurityAudit
TO FILE (
    FILEPATH = 'C:\SQLAudit\',
    MAXSIZE = 512 MB,
    MAX_ROLLOVER_FILES = 20,
    RESERVE_DISK_SPACE = ON
)
WITH (
    QUEUE_DELAY = 1000,          -- ms before flushing to file
    ON_FAILURE = CONTINUE,       -- CONTINUE or SHUTDOWN
    AUDIT_GUID = NEWID()
);

-- Create audit writing to Windows Security Event Log
CREATE SERVER AUDIT SecurityEventLogAudit
TO SECURITY_LOG
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE
);

-- Create audit writing to Windows Application Event Log
CREATE SERVER AUDIT AppEventLogAudit
TO APPLICATION_LOG
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE
);

-- Azure SQL: audit to Azure Blob Storage
CREATE SERVER AUDIT AzureBlobAudit
TO URL (
    PATH = 'https://mystorageaccount.blob.core.windows.net/sqlaudit',
    RETENTION_DAYS = 90
);

-- Enable the server audit
ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);

-- View audit status
SELECT
    name,
    status_desc,
    audit_file_path,
    type_desc AS TargetType,
    queue_delay,
    on_failure_desc,
    is_state_enabled
FROM sys.server_audits;
```

## Server Audit Specification

```sql
-- Create a server audit specification for security events
CREATE SERVER AUDIT SPECIFICATION ServerSecuritySpec
FOR SERVER AUDIT SecurityAudit
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (LOGIN_CHANGE_PASSWORD_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_CHANGE_GROUP),
ADD (SERVER_STATE_CHANGE_GROUP),
ADD (SERVER_OPERATION_GROUP),
ADD (BACKUP_RESTORE_GROUP),
ADD (AUDIT_CHANGE_GROUP),
ADD (SERVER_PRINCIPAL_CHANGE_GROUP),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP),
ADD (TRACE_CHANGE_GROUP)
WITH (STATE = ON);

-- Common action groups for compliance
-- FAILED_LOGIN_GROUP             -- Failed login attempts
-- SUCCESSFUL_LOGIN_GROUP         -- Successful logins
-- BATCH_COMPLETED_GROUP          -- Every batch executed (high volume)
-- DATABASE_OBJECT_ACCESS_GROUP   -- DML on audited objects
-- SCHEMA_OBJECT_ACCESS_GROUP     -- Schema-level access
-- LOGIN_CHANGE_PASSWORD_GROUP    -- Password changes
-- SERVER_ROLE_MEMBER_CHANGE_GROUP-- Server role membership changes
-- DATABASE_ROLE_MEMBER_CHANGE_GROUP -- Database role membership
-- SERVER_PERMISSION_CHANGE_GROUP -- GRANT/DENY/REVOKE at server level
-- DATABASE_PERMISSION_CHANGE_GROUP  -- GRANT/DENY/REVOKE at DB level
-- BACKUP_RESTORE_GROUP           -- Backup and restore operations
-- DATABASE_CHANGE_GROUP          -- CREATE/ALTER/DROP DATABASE
-- DATABASE_OBJECT_CHANGE_GROUP   -- DDL on objects
-- SERVER_STATE_CHANGE_GROUP      -- SHUTDOWN, RESTART
-- AUDIT_CHANGE_GROUP             -- Changes to audit configuration

-- View server audit specification details
SELECT
    a.name AS AuditName,
    s.name AS SpecName,
    d.audit_action_name,
    d.class_desc,
    s.is_state_enabled
FROM sys.server_audit_specifications s
JOIN sys.server_audits a ON s.audit_guid = a.audit_guid
JOIN sys.server_audit_specification_details d ON s.server_specification_id = d.server_specification_id;
```

## Database Audit Specification

```sql
-- Audit all DML on sensitive tables
USE AppDB;
CREATE DATABASE AUDIT SPECIFICATION SensitiveDataAccess
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE
    ON OBJECT::HR.Employees BY public),
ADD (SELECT, INSERT, UPDATE, DELETE
    ON OBJECT::Finance.Transactions BY public),
ADD (SELECT ON OBJECT::HR.Salaries BY public),
ADD (EXECUTE ON SCHEMA::HR BY public)
WITH (STATE = ON);

-- Audit schema changes in a database
CREATE DATABASE AUDIT SPECIFICATION SchemaChangeAudit
FOR SERVER AUDIT SecurityAudit
ADD (DATABASE_OBJECT_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_CHANGE_GROUP),
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_PRINCIPAL_CHANGE_GROUP)
WITH (STATE = ON);

-- Audit specific user actions on specific objects
CREATE DATABASE AUDIT SPECIFICATION UserActionAudit
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE
    ON OBJECT::Sales.CustomerPII BY [ExternalAppUser]),
ADD (SELECT ON OBJECT::Sales.CreditCards BY [ReportUser])
WITH (STATE = ON);

-- View database audit specification details
SELECT
    a.name AS AuditName,
    s.name AS SpecName,
    d.audit_action_name,
    d.class_desc,
    OBJECT_NAME(d.major_id) AS ObjectName,
    dp.name AS AuditedPrincipal,
    s.is_state_enabled
FROM sys.database_audit_specifications s
JOIN sys.server_audits a ON s.audit_guid = a.audit_guid
JOIN sys.database_audit_specification_details d
    ON s.database_specification_id = d.database_specification_id
LEFT JOIN sys.database_principals dp ON d.audited_principal_id = dp.principal_id;
```

## Filtering Audit Events

```sql
-- Filter audit events using WHERE clause (SQL Server 2012+)
CREATE SERVER AUDIT FilteredAudit
TO FILE (
    FILEPATH = 'C:\SQLAudit\Filtered\',
    MAXSIZE = 256 MB,
    MAX_ROLLOVER_FILES = 10
)
WITH (
    QUEUE_DELAY = 1000,
    ON_FAILURE = CONTINUE
)
WHERE (
    -- Only capture events from specific databases
    database_name = 'AppDB'
    -- Or exclude system databases
    OR (database_name NOT IN ('master', 'msdb', 'tempdb', 'model'))
);

-- Filter by specific schema or object
CREATE SERVER AUDIT SensitiveDataAudit
TO FILE (
    FILEPATH = 'C:\SQLAudit\Sensitive\',
    MAXSIZE = 256 MB
)
WHERE (
    schema_name = 'HR'
    AND object_name IN ('Employees', 'Salaries', 'SSN_Data')
);

-- Filter to capture only non-service-account activity
CREATE SERVER AUDIT UserActivityAudit
TO FILE (
    FILEPATH = 'C:\SQLAudit\UserActivity\',
    MAXSIZE = 256 MB
)
WHERE (
    server_principal_name NOT LIKE 'svc_%'
    AND server_principal_name NOT IN ('sa', 'SQLAgent')
);
```

## Reading Audit Files

```sql
-- Read all audit records from a specific file
SELECT *
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT);

-- Read with filtering and useful columns
SELECT
    event_time,
    action_id,
    succeeded,
    server_principal_name,
    database_name,
    schema_name,
    object_name,
    statement,
    additional_information,
    session_id,
    host_name,
    application_name,
    client_ip
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE event_time >= DATEADD(HOUR, -24, GETUTCDATE())
ORDER BY event_time DESC;

-- Failed login analysis
SELECT
    event_time,
    server_principal_name AS AttemptedLogin,
    client_ip AS SourceIP,
    application_name,
    additional_information,
    COUNT(*) OVER (PARTITION BY server_principal_name, client_ip) AS AttemptCount
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id = 'LGIF'  -- Login Failed
    AND event_time >= DATEADD(HOUR, -1, GETUTCDATE())
ORDER BY event_time DESC;

-- Schema change report
SELECT
    event_time,
    server_principal_name AS ChangedBy,
    database_name,
    schema_name,
    object_name,
    action_id,
    statement
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE action_id IN ('CR', 'AL', 'DR')  -- Create, Alter, Drop
    AND event_time >= DATEADD(DAY, -7, GETUTCDATE())
ORDER BY event_time DESC;

-- Sensitive data access report
SELECT
    event_time,
    server_principal_name,
    database_name,
    schema_name,
    object_name,
    action_id,
    CASE action_id
        WHEN 'SL' THEN 'SELECT'
        WHEN 'IN' THEN 'INSERT'
        WHEN 'UP' THEN 'UPDATE'
        WHEN 'DL' THEN 'DELETE'
        WHEN 'EX' THEN 'EXECUTE'
    END AS ActionName,
    client_ip,
    application_name
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
WHERE schema_name = 'HR'
    AND object_name IN ('Employees', 'Salaries')
    AND event_time >= DATEADD(DAY, -30, GETUTCDATE())
ORDER BY event_time DESC;
```

## Transaction Log as Audit Trail

The transaction log (`fn_dblog`) provides a last-resort audit capability for DML changes when no formal audit was configured. Useful for forensic investigation after data changes.

```sql
-- Read the active transaction log for recent operations
SELECT
    [Current LSN], Operation, Context,
    [Transaction ID], [Begin Time],
    OBJECT_NAME([Lock Information]) AS ObjectName,
    [RowLog Contents 0], [RowLog Contents 1]
FROM fn_dblog(NULL, NULL)
WHERE Operation IN ('LOP_INSERT_ROWS', 'LOP_DELETE_ROWS', 'LOP_MODIFY_ROW')
ORDER BY [Current LSN] DESC;

-- Read backed-up transaction log for point-in-time investigation
SELECT *
FROM fn_dump_dblog(NULL, NULL, N'DISK', 1,
    N'E:\Backups\AppDB_log.trn',
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT,
    DEFAULT, DEFAULT)
WHERE Operation IN ('LOP_INSERT_ROWS', 'LOP_DELETE_ROWS', 'LOP_MODIFY_ROW');
```

Note: `fn_dblog` is undocumented and unsupported. For production audit requirements, always use SQL Server Audit or temporal tables.

## C2 Audit Mode and Common Criteria

```sql
-- C2 Audit Mode (legacy, use SQL Server Audit instead)
-- Logs all access to all statements; high overhead
EXEC sp_configure 'c2 audit mode', 1;
RECONFIGURE;

-- Check if C2 is enabled
SELECT name, value_in_use
FROM sys.configurations
WHERE name = 'c2 audit mode';

-- Common Criteria Compliance (supplements SQL Server Audit)
EXEC sp_configure 'common criteria compliance enabled', 1;
RECONFIGURE;
-- Enables:
--   Residual Information Protection (memory clearing)
--   Login auditing for column-level GRANT/DENY
--   View Server State mapped to VIEW SERVER SECURITY STATE

-- Check Common Criteria status
SELECT name, value_in_use
FROM sys.configurations
WHERE name = 'common criteria compliance enabled';
```

## Compliance Mappings

```sql
-- PCI-DSS: Requirements 7, 8, 10
-- Req 10.1: Audit trails for all access to cardholder data
CREATE DATABASE AUDIT SPECIFICATION PCI_DataAccess
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE
    ON SCHEMA::Payment BY public)
WITH (STATE = ON);

-- Req 10.2: Audit individual user actions, root/admin actions,
-- access to audit trails, failed access attempts
CREATE SERVER AUDIT SPECIFICATION PCI_ServerAudit
FOR SERVER AUDIT SecurityAudit
ADD (FAILED_LOGIN_GROUP),
ADD (SUCCESSFUL_LOGIN_GROUP),
ADD (SERVER_ROLE_MEMBER_CHANGE_GROUP),
ADD (AUDIT_CHANGE_GROUP),
ADD (SERVER_PERMISSION_CHANGE_GROUP),
ADD (BACKUP_RESTORE_GROUP)
WITH (STATE = ON);

-- HIPAA: Audit access to PHI (Protected Health Information)
CREATE DATABASE AUDIT SPECIFICATION HIPAA_PHI_Access
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE
    ON OBJECT::Clinical.PatientRecords BY public),
ADD (SELECT, INSERT, UPDATE, DELETE
    ON OBJECT::Clinical.Diagnoses BY public),
ADD (SELECT ON OBJECT::Clinical.MedicalImages BY public),
ADD (DATABASE_PERMISSION_CHANGE_GROUP),
ADD (DATABASE_ROLE_MEMBER_CHANGE_GROUP)
WITH (STATE = ON);

-- SOX: Audit financial data and system changes
CREATE DATABASE AUDIT SPECIFICATION SOX_FinancialAudit
FOR SERVER AUDIT SecurityAudit
ADD (SELECT, INSERT, UPDATE, DELETE
    ON SCHEMA::Finance BY public),
ADD (DATABASE_OBJECT_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_CHANGE_GROUP)
WITH (STATE = ON);
```

## Audit Maintenance

```sql
-- Disable an audit specification before modification
ALTER DATABASE AUDIT SPECIFICATION SensitiveDataAccess
WITH (STATE = OFF);

-- Add new actions to an existing specification
ALTER DATABASE AUDIT SPECIFICATION SensitiveDataAccess
ADD (SELECT ON OBJECT::HR.Benefits BY public);

ALTER DATABASE AUDIT SPECIFICATION SensitiveDataAccess
WITH (STATE = ON);

-- Resize or change audit file settings
ALTER SERVER AUDIT SecurityAudit WITH (STATE = OFF);
ALTER SERVER AUDIT SecurityAudit
TO FILE (
    FILEPATH = 'D:\SQLAudit\',
    MAXSIZE = 1 GB,
    MAX_ROLLOVER_FILES = 50
);
ALTER SERVER AUDIT SecurityAudit WITH (STATE = ON);

-- Clean up old audit files (via SQL Agent job or script)
-- fn_get_audit_file reads all matching files; archive or delete
-- old .sqlaudit files per retention policy

-- Count audit records per day for capacity planning
SELECT
    CAST(event_time AS DATE) AS AuditDate,
    COUNT(*) AS RecordCount
FROM fn_get_audit_file('C:\SQLAudit\SecurityAudit*.sqlaudit', DEFAULT, DEFAULT)
GROUP BY CAST(event_time AS DATE)
ORDER BY AuditDate DESC;
```

## Best Practices

- Write audit data to a dedicated disk volume separate from database data and log files.
- Set ON_FAILURE = SHUTDOWN for the most sensitive compliance scenarios; use CONTINUE for monitoring-only audits.
- Filter audit events at the audit or specification level to reduce volume and storage costs.
- Audit all failed login attempts and all changes to security principals and permissions at the server level.
- Use database audit specifications to target specific sensitive tables rather than auditing all objects.
- Archive audit files regularly and protect them from tampering (read-only share, WORM storage).
- Monitor audit file size and rollover frequency; set alerts when files approach MAXSIZE.
- Test audit restores: confirm that fn_get_audit_file can read archived files successfully.
- Keep QUEUE_DELAY at 1000ms or higher to minimize performance impact; lower values increase I/O.
- Document the mapping between compliance requirements and audit specifications.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using BATCH_COMPLETED_GROUP without filtering | Massive audit file growth, significant performance degradation | Use database audit specifications on specific objects instead |
| Setting ON_FAILURE = SHUTDOWN for non-critical audits | Server stops if audit target is unavailable (disk full, etc.) | Use CONTINUE for non-compliance audits; reserve SHUTDOWN for PCI/HIPAA |
| Not monitoring audit file disk space | Audit files fill disk, causing audit failure or server shutdown | Set up disk space alerts and automate archival with retention policies |
| Auditing public on every table | Extreme volume capturing system queries and health checks | Audit specific schemas or tables, exclude service accounts via filter |
| Forgetting to back up audit files before server migration | Loss of audit trail, compliance gap | Include audit file archival in migration runbook |
| Modifying specifications without disabling first | ALTER fails with error | Always SET STATE = OFF before modifying, then SET STATE = ON |

## SQL Server Version Notes

- **SQL Server 2016**: SQL Server Audit available in all editions (previously Enterprise only). Audit filtering with WHERE clause on server audits. Database audit specifications support ACTION groups. Temporal tables provide built-in audit trail for data changes.
- **SQL Server 2019**: Improved audit performance with asynchronous writing. Data classification integration (sys.sensitivity_classifications) works with audit to tag columns. Audit for Big Data Clusters. Enhanced audit metadata in fn_get_audit_file output.
- **SQL Server 2022**: Azure-integrated auditing for Arc-enabled instances. Audit to Azure Blob Storage for hybrid deployments. Ledger database provides tamper-evident audit via blockchain-inspired verification. Improved audit throughput for high-volume OLTP workloads. Built-in data classification with CLASSIFY sensitivity labels.

## Sources

- [SQL Server Audit](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-database-engine)
- [CREATE SERVER AUDIT](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-server-audit-transact-sql)
- [SQL Server Audit Action Groups](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-action-groups-and-actions)
- [fn_get_audit_file](https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/sys-fn-get-audit-file-transact-sql)
- [Common Criteria Compliance](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/common-criteria-compliance-enabled-server-configuration-option)
- [SQL Server Audit Records](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/sql-server-audit-records)
- [View a SQL Server Audit Log](https://learn.microsoft.com/en-us/sql/relational-databases/security/auditing/view-a-sql-server-audit-log)
- [C2 Audit Mode Server Configuration Option](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/c2-audit-mode-server-configuration-option)
- [SQL Server Database Auditing Techniques](https://solutioncenter.apexsql.com/sql-server-database-auditing-techniques/)
- [Read a SQL Server Transaction Log](https://solutioncenter.apexsql.com/read-a-sql-server-transaction-log/)
