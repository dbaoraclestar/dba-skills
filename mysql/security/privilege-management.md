# Privilege Management — GRANT, REVOKE, and Least-Privilege Access Control

## Overview

MySQL privilege management is the foundation of database security. It controls which users
can perform which operations on which objects. The privilege system uses a hierarchical model
where privileges can be granted at global, database, table, column, and routine levels, with
higher-level grants cascading down to lower levels.

Understanding privilege management is critical for any DBA maintaining production MySQL
instances. Overly permissive grants are one of the most common security vulnerabilities in
database deployments, and the principle of least privilege should guide every access decision.

MySQL 8.0 introduced a major overhaul with dynamic privileges and roles, replacing the
monolithic SUPER privilege with fine-grained alternatives. This shift requires DBAs to
rethink legacy grant patterns and adopt more precise access controls.

## Key Concepts

- **Static Privileges**: Traditional privileges defined at server compile time (SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, etc.). Available in all MySQL versions.
- **Dynamic Privileges**: Privileges registered at runtime by server components or plugins (8.0+). Examples include BACKUP_ADMIN, ROLE_ADMIN, CONNECTION_ADMIN.
- **Privilege Levels**: Global (`*.*`), database (`db.*`), table (`db.table`), column (`db.table(col)`), routine (`PROCEDURE/FUNCTION`).
- **Roles**: Named collections of privileges that can be granted to users (8.0+). Simplify privilege management for groups of users with similar access needs.
- **Privilege Inheritance**: Grants at a higher level automatically apply at lower levels. A global SELECT grant implies SELECT on every database, table, and column.
- **Grant Tables**: System tables in the `mysql` database that store privilege information: `user`, `db`, `tables_priv`, `columns_priv`, `procs_priv`, `global_grants`.

## Privilege Levels in Detail

### Global Privileges

Global privileges apply to all databases on the server. They are stored in `mysql.user`
and `mysql.global_grants`.

```sql
-- Grant global read access
GRANT SELECT ON *.* TO 'readonly_user'@'10.0.0.%';

-- Grant administrative privileges
GRANT PROCESS, RELOAD, SHOW DATABASES ON *.* TO 'admin_user'@'localhost';

-- View global grants stored in mysql.user
SELECT user, host, Select_priv, Insert_priv, Update_priv, Delete_priv,
       Create_priv, Drop_priv, Reload_priv, Process_priv, Super_priv
FROM mysql.user
WHERE user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema')
ORDER BY user, host;
```

### Database-Level Privileges

```sql
-- Grant full access to a specific database
GRANT ALL PRIVILEGES ON app_production.* TO 'app_user'@'10.0.0.%';

-- Grant read-only access to a reporting database
GRANT SELECT, SHOW VIEW ON reporting.* TO 'report_user'@'%';

-- Grant DDL privileges for schema management
GRANT CREATE, ALTER, DROP, INDEX, CREATE VIEW ON staging.* TO 'devops'@'10.0.0.%';

-- View database-level grants
SELECT user, host, db, Select_priv, Insert_priv, Update_priv, Delete_priv
FROM mysql.db
WHERE user = 'app_user'
ORDER BY db;
```

### Table-Level Privileges

```sql
-- Grant access to specific tables
GRANT SELECT, INSERT, UPDATE ON app_production.orders TO 'order_svc'@'10.0.0.%';
GRANT SELECT ON app_production.products TO 'order_svc'@'10.0.0.%';

-- Grant DELETE only on the sessions table (for cleanup jobs)
GRANT SELECT, DELETE ON app_production.sessions TO 'cleanup_job'@'localhost';
```

### Column-Level Privileges

```sql
-- Grant access to specific columns (useful for PII protection)
GRANT SELECT (order_id, product_id, quantity, status)
ON app_production.orders TO 'analytics_user'@'10.0.0.%';

-- Hide sensitive columns from certain users
GRANT SELECT (user_id, username, created_at)
ON app_production.users TO 'support_user'@'10.0.0.%';
-- Note: email, phone, ssn columns are NOT granted
```

### Routine-Level Privileges

```sql
-- Grant EXECUTE on specific stored procedures
GRANT EXECUTE ON PROCEDURE app_production.process_refund TO 'cs_agent'@'10.0.0.%';
GRANT EXECUTE ON FUNCTION app_production.calculate_tax TO 'app_user'@'10.0.0.%';
```

## GRANT and REVOKE Syntax

### Creating Users and Granting Privileges

```sql
-- Create user and grant in separate statements (preferred in 8.0+)
CREATE USER 'app_user'@'10.0.0.%'
  IDENTIFIED BY 'strong_password_here'
  PASSWORD EXPIRE INTERVAL 90 DAY
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 1;

GRANT SELECT, INSERT, UPDATE, DELETE ON app_production.* TO 'app_user'@'10.0.0.%';

-- Grant with GRANT OPTION (allows user to grant their privileges to others)
GRANT SELECT ON reporting.* TO 'report_admin'@'%' WITH GRANT OPTION;
```

### Revoking Privileges

```sql
-- Revoke specific privileges
REVOKE INSERT, UPDATE, DELETE ON app_production.* FROM 'readonly_user'@'10.0.0.%';

-- Revoke all privileges
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'former_employee'@'%';

-- Revoke GRANT OPTION without revoking the privilege itself
REVOKE GRANT OPTION ON reporting.* FROM 'report_admin'@'%';
```

### Viewing Current Grants

```sql
-- Show grants for a specific user
SHOW GRANTS FOR 'app_user'@'10.0.0.%';

-- Show grants for the current session user
SHOW GRANTS FOR CURRENT_USER();

-- Query grant tables directly for comprehensive audit
SELECT CONCAT(user, '@', host) AS account,
       IF(Select_priv='Y','SELECT,',''),
       IF(Insert_priv='Y','INSERT,',''),
       IF(Update_priv='Y','UPDATE,',''),
       IF(Delete_priv='Y','DELETE,',''),
       IF(Create_priv='Y','CREATE,',''),
       IF(Drop_priv='Y','DROP,',''),
       IF(Super_priv='Y','SUPER,',''),
       IF(Grant_priv='Y','GRANT,','')
FROM mysql.user
WHERE user NOT LIKE 'mysql.%'
ORDER BY user;
```

## Dynamic Privileges (MySQL 8.0+)

MySQL 8.0 decomposed the SUPER privilege into fine-grained dynamic privileges.

```sql
-- Instead of SUPER, grant only what is needed
GRANT BACKUP_ADMIN ON *.* TO 'backup_user'@'localhost';
GRANT REPLICATION_SLAVE_ADMIN ON *.* TO 'repl_user'@'10.0.0.%';
GRANT CONNECTION_ADMIN ON *.* TO 'admin_user'@'localhost';
GRANT SYSTEM_VARIABLES_ADMIN ON *.* TO 'tuning_user'@'localhost';

-- List all available dynamic privileges
SELECT * FROM information_schema.USER_PRIVILEGES
WHERE PRIVILEGE_TYPE NOT IN ('SELECT','INSERT','UPDATE','DELETE',
  'CREATE','DROP','RELOAD','SHUTDOWN','PROCESS','FILE','REFERENCES',
  'INDEX','ALTER','SHOW DATABASES','SUPER','CREATE TEMPORARY TABLES',
  'LOCK TABLES','EXECUTE','REPLICATION SLAVE','REPLICATION CLIENT',
  'CREATE VIEW','SHOW VIEW','CREATE ROUTINE','ALTER ROUTINE',
  'CREATE USER','EVENT','TRIGGER','CREATE TABLESPACE','CREATE ROLE',
  'DROP ROLE');

-- View dynamic grants
SELECT * FROM mysql.global_grants WHERE USER = 'admin_user';
```

## Roles (MySQL 8.0+)

```sql
-- Create roles
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Grant privileges to roles
GRANT SELECT ON app_production.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON app_production.* TO 'app_write';
GRANT ALL PRIVILEGES ON app_production.* TO 'app_admin';

-- Grant roles to users
GRANT 'app_read' TO 'analyst'@'10.0.0.%';
GRANT 'app_read', 'app_write' TO 'app_user'@'10.0.0.%';
GRANT 'app_admin' TO 'dba_user'@'localhost';

-- Set default roles (activated automatically on login)
SET DEFAULT ROLE 'app_read' TO 'analyst'@'10.0.0.%';
SET DEFAULT ROLE ALL TO 'app_user'@'10.0.0.%';

-- Activate roles in the current session
SET ROLE 'app_read';
SET ROLE ALL;                  -- activate all granted roles
SET ROLE ALL EXCEPT 'app_admin';

-- Show which roles are active
SELECT CURRENT_ROLE();

-- Mandatory roles (activated for every user automatically)
SET GLOBAL mandatory_roles = 'app_read';
```

## Auditing Privilege Usage

```sql
-- Find users with excessive privileges
SELECT user, host,
       Super_priv, Grant_priv, File_priv, Process_priv,
       Create_user_priv, Reload_priv, Shutdown_priv
FROM mysql.user
WHERE (Super_priv = 'Y' OR Grant_priv = 'Y' OR File_priv = 'Y'
       OR Shutdown_priv = 'Y')
  AND user NOT IN ('root', 'mysql.sys')
ORDER BY user;

-- Find accounts with wildcard host patterns
SELECT user, host FROM mysql.user
WHERE host = '%' AND user NOT LIKE 'mysql.%'
ORDER BY user;

-- Find users who can grant privileges to others
SELECT DISTINCT GRANTEE
FROM information_schema.USER_PRIVILEGES
WHERE IS_GRANTABLE = 'YES';

-- Identify unused accounts (no recent connections) using performance_schema
SELECT u.user, u.host, u.account_locked,
       s.CURRENT_CONNECTIONS, s.TOTAL_CONNECTIONS
FROM mysql.user u
LEFT JOIN performance_schema.accounts s
  ON u.user = s.USER AND u.host = s.HOST
WHERE u.user NOT LIKE 'mysql.%'
  AND (s.TOTAL_CONNECTIONS IS NULL OR s.TOTAL_CONNECTIONS = 0)
ORDER BY u.user;
```

## Best Practices

- Always follow the principle of least privilege: grant only the minimum permissions needed for the task.
- Use roles (8.0+) to group privileges instead of granting directly to users.
- Replace SUPER with specific dynamic privileges when upgrading to 8.0.
- Never use `GRANT ALL ON *.*` for application accounts.
- Restrict host patterns as tightly as possible; avoid `'%'` when a subnet will do.
- Regularly audit privileges using SHOW GRANTS and queries against grant tables.
- Use `WITH GRANT OPTION` sparingly and only for delegated administration.
- Lock or drop accounts immediately when personnel leave the organization.
- Document all privilege grants with a ticket or change-management reference.
- Set `PASSWORD EXPIRE INTERVAL` and `FAILED_LOGIN_ATTEMPTS` on all human accounts.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Granting ALL PRIVILEGES ON *.* to application accounts | Full server access including DROP, SHUTDOWN | Grant only needed privileges on specific databases |
| Using `'user'@'%'` instead of restricting hosts | Account accessible from any network | Use `'user'@'10.0.0.%'` or specific IP/subnet |
| Forgetting FLUSH PRIVILEGES after direct grant-table edits | Changes not applied until server restart | Use GRANT/REVOKE statements instead of direct table edits; if editing tables, run FLUSH PRIVILEGES |
| Granting SUPER in MySQL 8.0 instead of dynamic privileges | Overly broad administrative access | Use BACKUP_ADMIN, SYSTEM_VARIABLES_ADMIN, etc. |
| Not revoking GRANT OPTION when removing privileges | User can still delegate revoked privileges | Explicitly REVOKE GRANT OPTION as well |
| Creating users without password expiry policies | Stale passwords remain valid indefinitely | Set PASSWORD EXPIRE INTERVAL 90 DAY |

## MySQL Version Notes

- **5.7**: Static privileges only. SUPER is the catch-all admin privilege. No roles. Password validation via validate_password plugin. `GRANT` implicitly creates users if `NO_AUTO_CREATE_USER` is not in sql_mode (removed default in 5.7.11+).
- **8.0**: Dynamic privileges replace SUPER. Roles introduced. `CREATE USER` and `GRANT` are separate statements (GRANT no longer creates users). `caching_sha2_password` is the default authentication plugin. `mysql.global_grants` table added for dynamic privileges. Partial revokes supported with `partial_revokes=ON`.
- **8.4/9.x**: `mysql_native_password` plugin deprecated and disabled by default. Multi-factor authentication support expanded. Password reuse policies enforced more strictly. SUPER privilege officially deprecated; existing grants still work but trigger warnings.

## Sources

- https://dev.mysql.com/doc/refman/8.0/en/privilege-system.html
- https://dev.mysql.com/doc/refman/8.0/en/grant.html
- https://dev.mysql.com/doc/refman/8.0/en/roles.html
- https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html
- https://dev.mysql.com/doc/refman/8.4/en/privilege-changes.html
