# Authentication — Plugins, Multi-Factor Auth, and Connection Security

## Overview

MySQL authentication determines how users prove their identity when connecting to the
server. The authentication system is plugin-based, allowing different users to authenticate
via different mechanisms: native password hashing, SHA-256 challenge-response, Unix sockets,
LDAP directories, or external PAM modules.

MySQL 8.0 introduced a significant change by making `caching_sha2_password` the default
authentication plugin, replacing the long-standing `mysql_native_password`. This change
improves security but can cause connectivity issues with older clients and drivers that
do not support the new plugin. Understanding the authentication landscape is essential for
smooth upgrades and secure deployments.

MySQL 8.0.27 added multi-factor authentication (MFA), enabling up to three authentication
factors per account. This brings MySQL in line with modern security requirements for
sensitive environments where passwords alone are insufficient.

## Key Concepts

- **Authentication Plugin**: A server-side component that validates credentials during connection. Each user account is associated with one (or more, with MFA) plugin.
- **caching_sha2_password**: Default in MySQL 8.0+. Uses SHA-256 hashing with server-side caching for performance. Requires SSL/TLS or RSA key exchange on first connection.
- **mysql_native_password**: Legacy default (pre-8.0). Uses SHA-1 hashing. Widely compatible but cryptographically weaker.
- **auth_socket / auth_pam**: OS-level authentication plugins that validate based on the operating system user or PAM configuration.
- **Multi-Factor Authentication (MFA)**: Requires multiple authentication methods to succeed during login (8.0.27+).
- **default_authentication_plugin**: Server variable controlling which plugin is used for new CREATE USER statements when no plugin is specified.

## Authentication Plugins

### mysql_native_password

The traditional MySQL authentication method using a double SHA-1 hash.

```sql
-- Create user with mysql_native_password (explicit)
CREATE USER 'legacy_app'@'10.0.0.%'
  IDENTIFIED WITH mysql_native_password BY 'password_here';

-- Check which plugin a user is using
SELECT user, host, plugin, password_expired
FROM mysql.user
WHERE user = 'legacy_app';

-- Server variable (5.7 default)
-- In my.cnf:
-- [mysqld]
-- default_authentication_plugin = mysql_native_password
```

### caching_sha2_password (MySQL 8.0+ Default)

Uses SHA-256 with a server-side cache. Faster than sha256_password after the first
successful authentication because credentials are cached in memory.

```sql
-- Create user with caching_sha2_password (default in 8.0)
CREATE USER 'modern_app'@'10.0.0.%'
  IDENTIFIED WITH caching_sha2_password BY 'strong_password';

-- Force cache invalidation (useful after password change)
FLUSH PRIVILEGES;

-- Check RSA public key (needed for non-SSL connections)
SHOW STATUS LIKE 'Caching_sha2_password_rsa_public_key';

-- Server configuration for RSA keys
-- [mysqld]
-- caching_sha2_password_public_key_path = /var/lib/mysql/public_key.pem
-- caching_sha2_password_private_key_path = /var/lib/mysql/private_key.pem
-- caching_sha2_password_auto_generate_rsa_keys = ON
```

### sha256_password

Pure SHA-256 authentication without caching. Requires SSL or RSA for every connection.

```sql
-- Create user with sha256_password
CREATE USER 'secure_user'@'10.0.0.%'
  IDENTIFIED WITH sha256_password BY 'password_here';

-- Client connection with RSA public key
-- mysql --user=secure_user --password --server-public-key-path=/path/to/public_key.pem
```

### auth_socket (Unix Socket Authentication)

Authenticates based on the OS username of the connecting process. No password needed.

```sql
-- Create user authenticated via Unix socket
CREATE USER 'root'@'localhost' IDENTIFIED WITH auth_socket;

-- Install the plugin if not loaded
INSTALL PLUGIN auth_socket SONAME 'auth_socket.so';

-- Connection works only from local socket as the matching OS user
-- sudo -u mysql mysql   (connects as 'mysql'@'localhost')
```

### LDAP Authentication (Enterprise)

```sql
-- Simple LDAP authentication (Enterprise only)
INSTALL PLUGIN authentication_ldap_simple SONAME 'authentication_ldap_simple.so';

CREATE USER 'ldap_user'@'%'
  IDENTIFIED WITH authentication_ldap_simple
  BY 'cn=users,dc=company,dc=com';

-- SASL LDAP authentication (supports SCRAM-SHA-256)
INSTALL PLUGIN authentication_ldap_sasl SONAME 'authentication_ldap_sasl.so';

-- Configure LDAP server connection
SET GLOBAL authentication_ldap_simple_server_host = 'ldap.company.com';
SET GLOBAL authentication_ldap_simple_server_port = 389;
SET GLOBAL authentication_ldap_simple_bind_base_dn = 'dc=company,dc=com';

-- For LDAPS (LDAP over SSL)
SET GLOBAL authentication_ldap_simple_tls = ON;
SET GLOBAL authentication_ldap_simple_ca_path = '/etc/ssl/certs/';
```

### PAM Authentication (Enterprise)

```sql
-- Install PAM authentication plugin
INSTALL PLUGIN authentication_pam SONAME 'authentication_pam.so';

-- Create user with PAM authentication
CREATE USER 'pam_user'@'%'
  IDENTIFIED WITH authentication_pam AS 'mysql_pam_service';

-- /etc/pam.d/mysql_pam_service example:
-- auth     required pam_unix.so
-- account  required pam_unix.so
```

## Multi-Factor Authentication (MySQL 8.0.27+)

MFA allows requiring up to three authentication factors per user account.

```sql
-- Create user with 2-factor authentication
CREATE USER 'secure_admin'@'localhost'
  IDENTIFIED WITH caching_sha2_password BY 'primary_password'
  AND IDENTIFIED WITH authentication_ldap_sasl AS 'uid=admin,ou=people,dc=corp';

-- Create user with 3-factor authentication
CREATE USER 'high_sec_user'@'localhost'
  IDENTIFIED WITH caching_sha2_password BY 'password1'
  AND IDENTIFIED WITH authentication_ldap_simple BY 'ldap_config'
  AND IDENTIFIED WITH authentication_fido;    -- hardware key

-- Modify existing user to add second factor
ALTER USER 'existing_user'@'localhost'
  ADD 2 FACTOR IDENTIFIED WITH authentication_ldap_simple AS 'ldap_config';

-- Remove a factor
ALTER USER 'existing_user'@'localhost' DROP 2 FACTOR;

-- Client connection with MFA
-- mysql --user=secure_admin --password1='primary' --password2='secondary'

-- Check MFA status
SELECT user, host, plugin, User_attributes
FROM mysql.user
WHERE user = 'secure_admin'\G
```

## Password Management

```sql
-- Password expiration policies
CREATE USER 'employee'@'%'
  IDENTIFIED BY 'initial_password'
  PASSWORD EXPIRE INTERVAL 90 DAY;

-- Global password expiration default
SET GLOBAL default_password_lifetime = 90;   -- days, 0 = never

-- Password reuse restrictions (8.0+)
CREATE USER 'secure_user'@'%'
  IDENTIFIED BY 'password'
  PASSWORD HISTORY 5                    -- cannot reuse last 5 passwords
  PASSWORD REUSE INTERVAL 365 DAY;     -- cannot reuse within 365 days

-- Failed login lockout (8.0+)
CREATE USER 'user1'@'%'
  IDENTIFIED BY 'password'
  FAILED_LOGIN_ATTEMPTS 5
  PASSWORD_LOCK_TIME 2;                -- lock for 2 days; 0 = until manual unlock

-- Password validation (validate_password component)
INSTALL COMPONENT 'file://component_validate_password';

SET GLOBAL validate_password.policy = STRONG;        -- LOW, MEDIUM, STRONG
SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;

-- Dual passwords for rotation without downtime (8.0.14+)
ALTER USER 'app_user'@'%'
  IDENTIFIED BY 'new_password'
  RETAIN CURRENT PASSWORD;

-- After all app instances updated, discard old password
ALTER USER 'app_user'@'%' DISCARD OLD PASSWORD;

-- Random password generation (8.0.18+)
CREATE USER 'new_user'@'%' IDENTIFIED BY RANDOM PASSWORD;
ALTER USER 'existing_user'@'%' IDENTIFIED BY RANDOM PASSWORD;
SET PASSWORD FOR 'user1'@'%' TO RANDOM;
```

## Connection Troubleshooting

```sql
-- Check which authentication plugin a user is configured with
SELECT user, host, plugin, authentication_string,
       password_expired, account_locked
FROM mysql.user
WHERE user NOT LIKE 'mysql.%'
ORDER BY user;

-- View connection errors
SELECT * FROM performance_schema.host_cache
WHERE SUM_CONNECT_ERRORS > 0
ORDER BY SUM_CONNECT_ERRORS DESC;

-- Reset host cache (clear blocked hosts)
TRUNCATE TABLE performance_schema.host_cache;
-- or
FLUSH HOSTS;

-- Test SSL status from client
-- mysql -u user -p -h host --ssl-mode=REQUIRED -e "SHOW STATUS LIKE 'Ssl%'"

-- Common error: "Authentication plugin 'caching_sha2_password' cannot be loaded"
-- Fix: upgrade client driver, or switch user to mysql_native_password:
ALTER USER 'legacy_app'@'%'
  IDENTIFIED WITH mysql_native_password BY 'password';

-- Common error: "Access denied using password: YES"
-- Diagnostic queries:
SELECT user, host, plugin FROM mysql.user WHERE user = 'problem_user';
SHOW VARIABLES LIKE 'default_authentication_plugin';

-- Check max_connect_errors threshold
SHOW VARIABLES LIKE 'max_connect_errors';
```

## Default Authentication Plugin Configuration

```ini
# my.cnf configuration for different scenarios

# MySQL 5.7 default (explicit)
[mysqld]
default_authentication_plugin = mysql_native_password

# MySQL 8.0 default (explicit)
[mysqld]
default_authentication_plugin = caching_sha2_password

# MySQL 8.0 with legacy compatibility
[mysqld]
default_authentication_plugin = mysql_native_password

# MySQL 8.4+ (deprecated variable, use authentication_policy instead)
[mysqld]
authentication_policy = caching_sha2_password
```

## Best Practices

- Use `caching_sha2_password` for all new accounts on MySQL 8.0+.
- Always require SSL/TLS when using SHA-256 based authentication over the network.
- Implement password expiration policies for all human (non-service) accounts.
- Use dual passwords during credential rotation to avoid application downtime.
- Configure `validate_password` component with STRONG policy in production.
- Prefer `auth_socket` for local administrative access (root@localhost).
- Enable multi-factor authentication for DBA and privileged accounts.
- Audit authentication plugins in use regularly with queries against `mysql.user`.
- Set `FAILED_LOGIN_ATTEMPTS` and `PASSWORD_LOCK_TIME` to mitigate brute-force attacks.
- When upgrading from 5.7 to 8.0, test all application connections against `caching_sha2_password` before cutover.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Upgrading to 8.0 without testing client compatibility with caching_sha2_password | Applications fail to connect after upgrade | Test all clients pre-upgrade; use mysql_native_password temporarily if needed |
| Using mysql_native_password on 8.4+ without realizing it is disabled by default | CREATE USER fails or connections rejected | Explicitly re-enable via authentication_policy or migrate to caching_sha2_password |
| Not configuring RSA keys for caching_sha2_password on non-SSL connections | First-time connections fail with "public key retrieval is not allowed" | Enable SSL, or set --get-server-public-key on the client, or configure RSA key paths |
| Setting PASSWORD_LOCK_TIME to 0 (manual unlock only) on service accounts | Locked service account causes full outage until DBA intervenes | Use a numeric lock time (e.g., 1 day) for service accounts; reserve 0 for human accounts |
| Granting LDAP-authenticated users without testing LDAP server connectivity | Users locked out if LDAP server is unreachable | Test LDAP connectivity and configure failover before creating LDAP-authenticated accounts |
| Not using dual passwords during rotation | Brief window where old credentials are invalid but still in use by running apps | Use RETAIN CURRENT PASSWORD, update apps, then DISCARD OLD PASSWORD |

## MySQL Version Notes

- **5.7**: Default plugin is `mysql_native_password`. No MFA support. Password expiration available but no reuse restrictions. `validate_password` is a plugin (not component). GRANT can implicitly create users unless `NO_AUTO_CREATE_USER` sql_mode is set.
- **8.0**: Default changed to `caching_sha2_password`. MFA added in 8.0.27. Dual passwords in 8.0.14. Random password generation in 8.0.18. `validate_password` migrated to a component. `default_authentication_plugin` controls new user defaults.
- **8.4/9.x**: `mysql_native_password` disabled by default (can be re-enabled). `default_authentication_plugin` deprecated in favor of `authentication_policy` (supports multi-factor defaults). FIDO/WebAuthn authentication improvements. Stricter password reuse enforcement.

## Sources

- https://dev.mysql.com/doc/refman/8.0/en/authentication-plugins.html
- https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html
- https://dev.mysql.com/doc/refman/8.0/en/multifactor-authentication.html
- https://dev.mysql.com/doc/refman/8.0/en/password-management.html
- https://dev.mysql.com/doc/refman/8.4/en/authentication-policy.html
- https://dev.mysql.com/doc/refman/8.0/en/validate-password.html
