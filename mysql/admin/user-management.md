# MySQL User Management — Authentication, Privileges, Roles, and Password Policies

## Overview

MySQL user management encompasses creating, modifying, and removing database accounts along with
their authentication methods, privileges, and resource limits. Starting with MySQL 8.0, the
privilege system was significantly enhanced with native role support, partial revokes, and the
`caching_sha2_password` default authentication plugin.

Proper user management is critical for database security. Every MySQL deployment should follow
the principle of least privilege: grant only the permissions each application or administrator
actually needs, audit accounts regularly, enforce password policies, and remove stale accounts
promptly.

This document covers the full lifecycle of MySQL user accounts from creation through auditing
and removal, including version-specific features across 5.7, 8.0, and 8.4/9.x.

## Key Concepts

- **Account**: Identified by `'user'@'host'` pair. `'app'@'10.0.0.%'` and `'app'@'localhost'` are distinct accounts.
- **Authentication Plugin**: Controls how credentials are verified (`mysql_native_password`, `caching_sha2_password`, `auth_socket`, LDAP, etc.).
- **Privilege**: A specific permission (SELECT, INSERT, CREATE, SUPER, etc.) granted at global, database, table, column, or routine level.
- **Role** (8.0+): A named collection of privileges that can be granted to users. Simplifies privilege management at scale.
- **Partial Revoke** (8.0+): The ability to exclude specific schemas from a global privilege grant.
- **Password Policy**: Server-enforced rules for password complexity, expiration, reuse history, and failed-login lockout.

## Creating Users

### Basic User Creation

```sql
-- Create user with password
CREATE USER 'appuser'@'10.0.0.%' IDENTIFIED BY 'SecureP@ss2024!';

-- Create user with specific authentication plugin
CREATE USER 'appuser'@'localhost'
  IDENTIFIED WITH caching_sha2_password BY 'SecureP@ss2024!';

-- Create user with resource limits
CREATE USER 'batch_user'@'%'
  IDENTIFIED BY 'BatchP@ss!'
  WITH MAX_QUERIES_PER_HOUR 1000
       MAX_UPDATES_PER_HOUR 500
       MAX_CONNECTIONS_PER_HOUR 100
       MAX_USER_CONNECTIONS 10;

-- Create user with SSL requirement
CREATE USER 'secure_user'@'%'
  IDENTIFIED BY 'SSLonly!'
  REQUIRE SSL;

-- Create user requiring specific X.509 attributes
CREATE USER 'cert_user'@'%'
  IDENTIFIED BY 'CertP@ss!'
  REQUIRE SUBJECT '/CN=client1'
  AND ISSUER '/CN=MyCA';
```

### Password Expiration and Account Locking

```sql
-- Create user with password that expires in 90 days
CREATE USER 'contractor'@'%'
  IDENTIFIED BY 'TempP@ss!'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- Create user with password expired immediately (must change on first login)
CREATE USER 'newuser'@'%'
  IDENTIFIED BY 'ChangeMe!'
  PASSWORD EXPIRE;

-- Create user with account initially locked
CREATE USER 'future_user'@'%'
  IDENTIFIED BY 'FutureP@ss!'
  ACCOUNT LOCK;

-- Create user with failed-login tracking (8.0.19+)
CREATE USER 'appuser2'@'%'
  IDENTIFIED BY 'AppP@ss!'
  FAILED_LOGIN_ATTEMPTS 3
  PASSWORD_LOCK_TIME 1;  -- locked for 1 day after 3 failures
```

## Altering Users

```sql
-- Change password
ALTER USER 'appuser'@'10.0.0.%' IDENTIFIED BY 'NewSecureP@ss2025!';

-- Change authentication plugin
ALTER USER 'appuser'@'localhost'
  IDENTIFIED WITH mysql_native_password BY 'LegacyP@ss!';

-- Unlock an account
ALTER USER 'future_user'@'%' ACCOUNT UNLOCK;

-- Set password expiration policy
ALTER USER 'appuser'@'10.0.0.%' PASSWORD EXPIRE INTERVAL 180 DAY;

-- Never expire password (service accounts)
ALTER USER 'svc_account'@'%' PASSWORD EXPIRE NEVER;

-- Modify resource limits
ALTER USER 'batch_user'@'%'
  WITH MAX_QUERIES_PER_HOUR 2000
       MAX_CONNECTIONS_PER_HOUR 200;

-- Change current user's own password
ALTER USER USER() IDENTIFIED BY 'MyNewPassword!';

-- Set password history and reuse restrictions (8.0+)
ALTER USER 'appuser'@'%'
  PASSWORD HISTORY 5
  PASSWORD REUSE INTERVAL 365 DAY;
```

## Dropping Users

```sql
-- Drop a single user
DROP USER 'olduser'@'%';

-- Drop if exists (avoids error if account is missing)
DROP USER IF EXISTS 'olduser'@'%';

-- Drop multiple users at once
DROP USER 'temp1'@'%', 'temp2'@'%', 'temp3'@'localhost';
```

## Privilege Management

### Granting Privileges

```sql
-- Grant all privileges on a specific database
GRANT ALL PRIVILEGES ON myapp.* TO 'appuser'@'10.0.0.%';

-- Grant specific privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp.* TO 'appuser'@'10.0.0.%';

-- Grant read-only access
GRANT SELECT ON reporting.* TO 'analyst'@'%';

-- Grant table-level privileges
GRANT SELECT, UPDATE (salary, bonus) ON hr.employees TO 'hr_user'@'%';

-- Grant with ability to re-grant (use sparingly)
GRANT SELECT ON myapp.* TO 'lead_dev'@'%' WITH GRANT OPTION;

-- Grant global administrative privileges
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'monitor'@'localhost';

-- Grant EXECUTE on stored procedures
GRANT EXECUTE ON PROCEDURE myapp.process_order TO 'appuser'@'%';

-- Apply privilege changes
FLUSH PRIVILEGES;
```

### Revoking Privileges

```sql
-- Revoke specific privileges
REVOKE INSERT, UPDATE, DELETE ON myapp.* FROM 'analyst'@'%';

-- Revoke all privileges
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'olduser'@'%';
```

### Partial Revokes (8.0+)

```sql
-- Enable partial revokes
SET PERSIST partial_revokes = ON;

-- Grant global SELECT, then exclude sensitive schemas
GRANT SELECT ON *.* TO 'analyst'@'%';
REVOKE SELECT ON mysql.* FROM 'analyst'@'%';
REVOKE SELECT ON performance_schema.* FROM 'analyst'@'%';
REVOKE SELECT ON sys.* FROM 'analyst'@'%';
```

## Roles (MySQL 8.0+)

### Creating and Assigning Roles

```sql
-- Create roles
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Grant privileges to roles
GRANT SELECT ON myapp.* TO 'app_read';
GRANT INSERT, UPDATE, DELETE ON myapp.* TO 'app_write';
GRANT ALL PRIVILEGES ON myapp.* TO 'app_admin';

-- Assign roles to users
GRANT 'app_read' TO 'analyst'@'%';
GRANT 'app_read', 'app_write' TO 'appuser'@'10.0.0.%';
GRANT 'app_admin' TO 'dba'@'localhost';

-- Set default roles (activated automatically on login)
SET DEFAULT ROLE ALL TO 'appuser'@'10.0.0.%';
SET DEFAULT ROLE 'app_read' TO 'analyst'@'%';

-- Activate roles in current session
SET ROLE 'app_admin';
SET ROLE ALL;
SET ROLE NONE;

-- Check current active roles
SELECT CURRENT_ROLE();
```

### Managing Roles

```sql
-- Show privileges of a role
SHOW GRANTS FOR 'app_read';

-- Revoke role from user
REVOKE 'app_write' FROM 'appuser'@'10.0.0.%';

-- Drop a role
DROP ROLE 'app_admin';

-- Set mandatory roles for all users (server-wide)
SET PERSIST mandatory_roles = 'app_read';
```

## Password Validation Plugin

### Configuration

```sql
-- Check if validate_password is installed
SHOW PLUGINS LIKE 'validate_password%';

-- Install (if not loaded)
-- 5.7: validate_password plugin
INSTALL PLUGIN validate_password SONAME 'validate_password.so';
-- 8.0+: validate_password component
INSTALL COMPONENT 'file://component_validate_password';

-- View current settings
SHOW VARIABLES LIKE 'validate_password%';

-- Configure password policy
SET PERSIST validate_password.policy = STRONG;        -- LOW, MEDIUM, STRONG
SET PERSIST validate_password.length = 12;
SET PERSIST validate_password.mixed_case_count = 1;
SET PERSIST validate_password.number_count = 1;
SET PERSIST validate_password.special_char_count = 1;

-- Check password strength (returns 0-100)
SELECT VALIDATE_PASSWORD_STRENGTH('MyP@ssw0rd!');
```

### Policy Levels

| Level  | Checks Performed                                        |
|--------|---------------------------------------------------------|
| LOW    | Password length only                                    |
| MEDIUM | Length + numeric, lowercase, uppercase, special chars   |
| STRONG | Medium + dictionary file check                          |

## User and Privilege Auditing

### Useful Audit Queries

```sql
-- List all users with authentication plugin and account status
SELECT user, host, plugin, account_locked, password_expired,
       password_lifetime, password_last_changed
FROM mysql.user
ORDER BY user, host;

-- Find users with global ALL PRIVILEGES
SELECT grantee, privilege_type
FROM information_schema.USER_PRIVILEGES
WHERE privilege_type = 'SUPER'
   OR privilege_type = 'ALL PRIVILEGES'
ORDER BY grantee;

-- Show all grants for a specific user
SHOW GRANTS FOR 'appuser'@'10.0.0.%';

-- Find users with GRANT OPTION
SELECT DISTINCT grantee
FROM information_schema.USER_PRIVILEGES
WHERE is_grantable = 'YES';

-- Find users with no password set
SELECT user, host FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL;

-- Find accounts with wildcard host patterns
SELECT user, host FROM mysql.user
WHERE host LIKE '%\%%' OR host = '';

-- List users with SUPER privilege (deprecated in 8.0)
SELECT user, host FROM mysql.user WHERE Super_priv = 'Y';

-- Show database-level grants
SELECT * FROM mysql.db WHERE user = 'appuser';

-- Find users who have not connected recently (8.0+ performance_schema)
SELECT u.user, u.host, u.password_last_changed,
       s.last_seen
FROM mysql.user u
LEFT JOIN (
  SELECT user, host, MAX(last_seen) as last_seen
  FROM performance_schema.accounts
  WHERE user IS NOT NULL
  GROUP BY user, host
) s ON u.user = s.user AND u.host = s.host
ORDER BY s.last_seen ASC;

-- List all roles and their members (8.0+)
SELECT FROM_USER AS role_name, FROM_HOST,
       TO_USER AS granted_to, TO_HOST
FROM mysql.role_edges
ORDER BY FROM_USER, TO_USER;

-- Check default roles (8.0+)
SELECT * FROM mysql.default_roles;

-- Privilege summary by user
SELECT grantee,
       GROUP_CONCAT(DISTINCT privilege_type ORDER BY privilege_type SEPARATOR ', ') AS privileges
FROM information_schema.USER_PRIVILEGES
GROUP BY grantee
ORDER BY grantee;
```

### Audit Script for Privilege Review

```sql
-- Comprehensive privilege report
SELECT
  u.user,
  u.host,
  u.plugin AS auth_plugin,
  u.account_locked,
  u.password_expired,
  u.password_last_changed,
  COALESCE(
    GROUP_CONCAT(DISTINCT
      CONCAT(db.Db, ': ',
        CASE WHEN db.Select_priv='Y' THEN 'S' ELSE '' END,
        CASE WHEN db.Insert_priv='Y' THEN 'I' ELSE '' END,
        CASE WHEN db.Update_priv='Y' THEN 'U' ELSE '' END,
        CASE WHEN db.Delete_priv='Y' THEN 'D' ELSE '' END
      )
    SEPARATOR ' | '),
    'NO DB PRIVS'
  ) AS db_privileges
FROM mysql.user u
LEFT JOIN mysql.db db ON u.user = db.user AND u.host = db.host
WHERE u.user NOT IN ('mysql.sys', 'mysql.session', 'mysql.infoschema')
GROUP BY u.user, u.host, u.plugin, u.account_locked,
         u.password_expired, u.password_last_changed
ORDER BY u.user, u.host;
```

## Best Practices

- Always specify the host part of accounts; never rely on `'user'@'%'` when a narrower host pattern suffices.
- Use `caching_sha2_password` (MySQL 8.0+ default) for new accounts; migrate legacy `mysql_native_password` accounts.
- Enforce password validation with at least MEDIUM policy in production.
- Set `PASSWORD EXPIRE INTERVAL` on human accounts; use `PASSWORD EXPIRE NEVER` only for verified service accounts.
- Use roles (8.0+) instead of granting privileges directly to individual users.
- Set `SET DEFAULT ROLE ALL` for roles that should always be active.
- Enable `partial_revokes` to protect system schemas from global grants.
- Run privilege audits quarterly; drop accounts unused for 90+ days.
- Never grant SUPER or ALL PRIVILEGES at the global level to application accounts.
- Store passwords in a secrets manager, not in application config files or scripts.
- Use `ACCOUNT LOCK` instead of `DROP USER` when temporarily disabling access.
- Enable the `connection_control` plugin to throttle brute-force login attempts.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `'user'@'%'` for all accounts | Allows connections from any host, increasing attack surface | Restrict host to specific IPs or subnets: `'user'@'10.0.1.%'` |
| Granting `ALL PRIVILEGES ON *.*` to application users | Application compromise gives full server control | Grant only needed privileges on specific databases |
| Not setting `PASSWORD EXPIRE` on human accounts | Passwords remain valid indefinitely; stale credentials are a breach risk | Set `PASSWORD EXPIRE INTERVAL 90 DAY` on interactive accounts |
| Forgetting `FLUSH PRIVILEGES` after direct mysql table edits | Privilege changes not applied until server reload | Always use `GRANT`/`REVOKE` statements instead of direct table manipulation |
| Using `mysql_native_password` on MySQL 8.0+ | Weaker authentication, deprecated in 8.4 | Migrate to `caching_sha2_password` or `auth_socket` |
| Not auditing GRANT OPTION holders | Users with GRANT OPTION can create privilege escalation paths | Regularly query `information_schema.USER_PRIVILEGES` for `is_grantable='YES'` |
| Creating roles but not setting default roles | Users must manually `SET ROLE` each session; applications may lack expected privileges | Use `SET DEFAULT ROLE ALL TO 'user'@'host'` |
| Ignoring partial revokes for global grants | Global SELECT grant exposes `mysql`, `performance_schema`, `sys` | Enable `partial_revokes=ON` and revoke on system schemas |

## MySQL Version Notes

### MySQL 5.7
- Default auth plugin: `mysql_native_password`.
- No native role support (emulated via granting one user's privileges to another).
- Password validation via `validate_password` plugin (not component).
- No partial revokes.
- `SUPER` privilege is the main administrative privilege.

### MySQL 8.0
- Default auth plugin: `caching_sha2_password` (breaks some older clients/connectors).
- Native roles with `CREATE ROLE`, `GRANT role TO user`, `SET DEFAULT ROLE`.
- Partial revokes introduced (`partial_revokes=ON`).
- Dynamic privileges replace `SUPER` (e.g., `SYSTEM_VARIABLES_ADMIN`, `REPLICATION_SLAVE_ADMIN`).
- `validate_password` repackaged as a component.
- `FAILED_LOGIN_ATTEMPTS` and `PASSWORD_LOCK_TIME` (8.0.19+).
- Password history and reuse interval.
- `SET PERSIST` for runtime + config file changes.

### MySQL 8.4 / 9.x
- `mysql_native_password` plugin removed by default in 8.4; must be explicitly loaded if needed.
- Multi-factor authentication (MFA) support enhanced.
- `GRANT ... TO ... WITH ADMIN OPTION` refined semantics.
- `authentication_webauthn` plugin for FIDO/WebAuthn (passwordless).
- Improved audit log filtering for user events.

## Sources

- [MySQL 8.0 Reference: Account Management](https://dev.mysql.com/doc/refman/8.0/en/account-management-statements.html)
- [MySQL 8.0 Reference: Roles](https://dev.mysql.com/doc/refman/8.0/en/roles.html)
- [MySQL 8.0 Reference: Password Validation](https://dev.mysql.com/doc/refman/8.0/en/validate-password.html)
- [MySQL 8.0 Reference: Partial Revokes](https://dev.mysql.com/doc/refman/8.0/en/partial-revokes.html)
- [MySQL 8.4 Reference: Account Management](https://dev.mysql.com/doc/refman/8.4/en/account-management-statements.html)
- [MySQL 8.0 Reference: Connection Control](https://dev.mysql.com/doc/refman/8.0/en/connection-control.html)
