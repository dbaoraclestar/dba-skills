# Encryption — TDE, SSL/TLS, Keyring Plugins, and Data Protection

## Overview

MySQL encryption protects data in two complementary layers: data-at-rest encryption
(Transparent Data Encryption, or TDE) secures files on disk so that physical access to
storage does not expose plaintext data, while data-in-transit encryption (SSL/TLS) protects
data flowing between clients and the server over the network.

TDE in MySQL uses tablespace-level encryption managed through keyring plugins. When enabled,
InnoDB encrypts tablespace files, redo logs, undo logs, and binary logs using AES-256. The
encryption is transparent to applications -- no SQL changes are required, and the server
handles encryption and decryption automatically during I/O operations.

SSL/TLS encryption is critical for preventing eavesdropping and man-in-the-middle attacks,
especially when database connections traverse untrusted networks. MySQL can generate its own
certificates or use externally provided CA-signed certificates for production use.

## Key Concepts

- **TDE (Transparent Data Encryption)**: Encrypts data files on disk without application changes. Uses a two-tier key architecture: tablespace keys encrypted by a master key stored in a keyring.
- **Keyring Plugin**: Server plugin that manages encryption keys. Options include file-based, encrypted file, HashiCorp Vault, Oracle Key Vault, and AWS KMS.
- **Master Encryption Key**: The top-level key that encrypts individual tablespace keys. Stored in and managed by the keyring plugin. Can be rotated without re-encrypting data.
- **Tablespace Key**: Per-tablespace symmetric key (AES-256) used to encrypt/decrypt data pages. Encrypted by the master key.
- **SSL/TLS**: Encrypts the network connection between MySQL client and server. Uses X.509 certificates.
- **require_secure_transport**: Server variable that rejects all unencrypted connections.

## Tablespace Encryption (TDE)

### Setting Up the Keyring

```ini
# my.cnf - keyring_file plugin (simplest, for non-production or single-server)
[mysqld]
early-plugin-load = keyring_file.so
keyring_file_data = /var/lib/mysql-keyring/keyring

# my.cnf - keyring_encrypted_file (password-protected keyring file)
[mysqld]
early-plugin-load = keyring_encrypted_file.so
keyring_encrypted_file_data = /var/lib/mysql-keyring/keyring-encrypted
keyring_encrypted_file_password = 'keyring_file_passphrase'

# my.cnf - keyring_hashicorp (HashiCorp Vault integration, Enterprise)
[mysqld]
early-plugin-load = keyring_hashicorp.so
keyring_hashicorp_server_url = https://vault.company.com:8200
keyring_hashicorp_role_id = <role-id>
keyring_hashicorp_secret_id = <secret-id>
keyring_hashicorp_store_path = /v1/kv/mysql
keyring_hashicorp_auth_path = /v1/auth/approle/login
keyring_hashicorp_ca_path = /etc/ssl/certs/vault-ca.pem
```

```sql
-- Verify keyring plugin is loaded
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME LIKE 'keyring%';

-- 8.0.24+ component-based keyring (replaces plugins)
INSTALL COMPONENT 'file://component_keyring_file';
-- Configured via component manifest file, not server variables
```

### Encrypting InnoDB Tablespaces

```sql
-- Encrypt a specific table's tablespace
ALTER TABLE app_production.credit_cards ENCRYPTION = 'Y';

-- Encrypt an existing general tablespace
ALTER TABLESPACE ts_sensitive ENCRYPTION = 'Y';

-- Create an encrypted general tablespace
CREATE TABLESPACE ts_encrypted
  ADD DATAFILE 'ts_encrypted.ibd'
  ENCRYPTION = 'Y';

-- Create table in encrypted tablespace
CREATE TABLE sensitive_data (
    id BIGINT PRIMARY KEY,
    ssn VARCHAR(11),
    account_number VARCHAR(20)
) TABLESPACE ts_encrypted;

-- Set default encryption for new tablespaces
SET GLOBAL default_table_encryption = ON;

-- Per-schema default encryption
CREATE DATABASE secure_db DEFAULT ENCRYPTION = 'Y';
ALTER DATABASE secure_db DEFAULT ENCRYPTION = 'Y';

-- Verify encryption status
SELECT TABLE_SCHEMA, TABLE_NAME, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE CREATE_OPTIONS LIKE '%ENCRYPTION%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;

-- More detailed encryption info (8.0.16+)
SELECT SPACE, NAME, FLAG, ENCRYPTION
FROM information_schema.INNODB_TABLESPACES
WHERE ENCRYPTION = 'Y';
```

### Master Key Rotation

```sql
-- Rotate the master encryption key (does NOT re-encrypt data files)
-- Generates a new master key, re-wraps all tablespace keys with the new master key
ALTER INSTANCE ROTATE INNODB MASTER KEY;

-- This should be done periodically (e.g., quarterly) or after
-- suspected key compromise. It is an online operation.
```

### Redo Log and Undo Log Encryption

```sql
-- Encrypt InnoDB redo logs (8.0.13+)
SET GLOBAL innodb_redo_log_encrypt = ON;

-- Encrypt InnoDB undo logs (8.0.13+)
SET GLOBAL innodb_undo_log_encrypt = ON;

-- Verify in my.cnf for persistence
-- [mysqld]
-- innodb_redo_log_encrypt = ON
-- innodb_undo_log_encrypt = ON
```

### Binary Log and Relay Log Encryption

```sql
-- Encrypt binary logs and relay logs (8.0.14+)
SET GLOBAL binlog_encryption = ON;

-- Rotate the binary log encryption key
ALTER INSTANCE ROTATE BINLOG MASTER KEY;

-- Verify encryption status
SHOW VARIABLES LIKE 'binlog_encryption';

-- Note: existing unencrypted binlog files remain unencrypted.
-- Only new binlog files written after enabling are encrypted.
```

## SSL/TLS for Connections (Data-in-Transit)

### Server-Side SSL Configuration

```ini
# my.cnf - SSL/TLS server configuration
[mysqld]
ssl-ca   = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key  = /etc/mysql/ssl/server-key.pem

# Minimum TLS version
tls_version = TLSv1.2,TLSv1.3

# Restrict cipher suites (TLS 1.2)
ssl-cipher = ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384

# Restrict cipher suites (TLS 1.3)
tls_ciphersuites = TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256

# Require all connections to use SSL
require_secure_transport = ON

# Admin connection separate SSL (8.0+)
admin_ssl_ca   = /etc/mysql/ssl/admin-ca.pem
admin_ssl_cert = /etc/mysql/ssl/admin-cert.pem
admin_ssl_key  = /etc/mysql/ssl/admin-key.pem
```

### Generating Self-Signed Certificates

```bash
# MySQL auto-generates certificates on first start if not present
# Manual generation with mysql_ssl_rsa_setup:
mysql_ssl_rsa_setup --datadir=/var/lib/mysql

# Manual generation with OpenSSL:
# 1. Generate CA key and certificate
openssl genrsa 4096 > ca-key.pem
openssl req -new -x509 -nodes -days 3650 -key ca-key.pem \
  -out ca.pem -subj "/CN=MySQL CA"

# 2. Generate server key and certificate
openssl req -newkey rsa:4096 -days 1095 -nodes \
  -keyout server-key.pem -out server-req.pem \
  -subj "/CN=mysql-server.company.com"
openssl rsa -in server-key.pem -out server-key.pem
openssl x509 -req -in server-req.pem -days 1095 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out server-cert.pem

# 3. Generate client key and certificate
openssl req -newkey rsa:4096 -days 1095 -nodes \
  -keyout client-key.pem -out client-req.pem \
  -subj "/CN=mysql-client"
openssl rsa -in client-key.pem -out client-key.pem
openssl x509 -req -in client-req.pem -days 1095 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 02 \
  -out client-cert.pem

# 4. Verify certificates
openssl verify -CAfile ca.pem server-cert.pem client-cert.pem

# 5. Set permissions
chmod 600 server-key.pem client-key.pem ca-key.pem
chown mysql:mysql /etc/mysql/ssl/*.pem
```

### Client-Side SSL Connection

```bash
# Connect with SSL verification
mysql -u app_user -p \
  --ssl-ca=/etc/mysql/ssl/ca.pem \
  --ssl-cert=/etc/mysql/ssl/client-cert.pem \
  --ssl-key=/etc/mysql/ssl/client-key.pem \
  --ssl-mode=VERIFY_IDENTITY \
  -h mysql-server.company.com

# SSL modes:
#   DISABLED       - no SSL
#   PREFERRED      - try SSL, fall back to unencrypted (default)
#   REQUIRED       - SSL required, no certificate verification
#   VERIFY_CA      - SSL required, verify server CA
#   VERIFY_IDENTITY - SSL required, verify CA + hostname match
```

### Requiring SSL for Specific Users

```sql
-- Require SSL for a user
CREATE USER 'secure_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE SSL;

-- Require specific X.509 attributes
CREATE USER 'cert_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE X509;

-- Require specific issuer and subject
CREATE USER 'strict_user'@'%'
  IDENTIFIED BY 'password'
  REQUIRE ISSUER '/CN=MySQL CA'
  AND SUBJECT '/CN=mysql-client'
  AND CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';

-- Verify SSL status of current connection
SHOW STATUS LIKE 'Ssl_cipher';
SHOW STATUS LIKE 'Ssl_version';
SELECT * FROM performance_schema.session_status
WHERE VARIABLE_NAME LIKE 'Ssl%';
```

### Checking SSL Status

```sql
-- Server SSL configuration
SHOW VARIABLES LIKE '%ssl%';
SHOW VARIABLES LIKE 'tls_version';
SHOW VARIABLES LIKE 'require_secure_transport';

-- Current connection SSL details
\s
-- or
STATUS;
-- Look for "SSL: Cipher in use is ..."

-- All connections and their SSL status (performance_schema)
SELECT THREAD_ID, PROCESSLIST_USER, PROCESSLIST_HOST,
       CONNECTION_TYPE
FROM performance_schema.threads
WHERE TYPE = 'FOREGROUND'
  AND PROCESSLIST_USER IS NOT NULL;

-- Check SSL-related status counters
SHOW STATUS LIKE 'Ssl_accepts';
SHOW STATUS LIKE 'Ssl_finished_accepts';
SHOW STATUS LIKE 'Ssl_client_connects';
```

## Keyring Plugin Comparison

| Feature | keyring_file | keyring_encrypted_file | keyring_hashicorp | keyring_aws |
|---------|-------------|----------------------|-------------------|-------------|
| Encryption at rest | No | Yes (AES-256) | Yes (Vault) | Yes (KMS) |
| Key backup | File copy | File copy | Vault HA | AWS managed |
| Access control | OS file perms | OS file perms + passphrase | Vault policies | IAM policies |
| Multi-server | No (local file) | No (local file) | Yes (centralized) | Yes (centralized) |
| Enterprise only | No | No | Yes | Yes |
| Recommended for | Dev/test | Single prod server | Enterprise prod | AWS deployments |

## Best Practices

- Use `keyring_hashicorp` or `keyring_aws` in production for centralized key management and auditing.
- Enable `require_secure_transport = ON` to ensure no unencrypted connections are accepted.
- Set `tls_version = TLSv1.2,TLSv1.3` and disable older TLS versions.
- Encrypt redo logs, undo logs, and binary logs in addition to tablespace files for complete at-rest protection.
- Rotate the InnoDB master key on a regular schedule (quarterly is common).
- Use `VERIFY_IDENTITY` SSL mode on clients to prevent man-in-the-middle attacks.
- Store keyring files outside the MySQL data directory on a separate, access-controlled filesystem.
- Back up keyring files separately from database backups; without the keyring, encrypted backups are unrecoverable.
- Use CA-signed certificates in production; self-signed certificates are acceptable only for internal/dev environments.
- Monitor certificate expiration dates and rotate before expiry.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using keyring_file in multi-server setups without syncing | Different servers have different master keys; replicas cannot read encrypted data | Use centralized keyring (HashiCorp Vault, AWS KMS) |
| Not backing up the keyring file | Data permanently lost if keyring is deleted or corrupted | Back up keyring separately from data; test restore procedure |
| Setting ssl-mode=PREFERRED (client default) and assuming connections are encrypted | Falls back to unencrypted silently if SSL handshake fails | Use ssl-mode=REQUIRED or VERIFY_IDENTITY |
| Enabling binlog_encryption without keyring | Server fails to start or cannot write binary logs | Install and configure keyring plugin before enabling binlog encryption |
| Encrypting tables but not redo/undo logs | Sensitive data visible in redo/undo log files on disk | Enable innodb_redo_log_encrypt and innodb_undo_log_encrypt |
| Using expired or self-signed certificates in production without VERIFY_CA | Vulnerable to man-in-the-middle attacks | Use CA-signed certs and ssl-mode=VERIFY_IDENTITY |

## MySQL Version Notes

- **5.7**: TDE available for file-per-table tablespaces only. Keyring plugins: `keyring_file`, `keyring_okv` (Enterprise). No redo/undo/binlog encryption. SSL auto-generated certificates introduced. `mysql_ssl_rsa_setup` utility added.
- **8.0**: General tablespace encryption added. Redo log encryption (8.0.13), undo log encryption (8.0.13), binary log encryption (8.0.14). `default_table_encryption` variable. Schema-level `DEFAULT ENCRYPTION`. Component-based keyring (8.0.24+) alongside plugin-based. `keyring_hashicorp` and `keyring_aws` plugins. TLS 1.3 support. `admin_ssl_*` variables for admin port.
- **8.4/9.x**: Keyring plugins deprecated in favor of keyring components. `keyring_file` plugin removed; use `component_keyring_file` instead. Improved TLS 1.3 cipher suite controls. Doublewrite file encryption added. Performance improvements for encrypted tablespace I/O.

## Sources

- https://dev.mysql.com/doc/refman/8.0/en/innodb-data-encryption.html
- https://dev.mysql.com/doc/refman/8.0/en/keyring.html
- https://dev.mysql.com/doc/refman/8.0/en/using-encrypted-connections.html
- https://dev.mysql.com/doc/refman/8.0/en/binary-log-encryption.html
- https://dev.mysql.com/doc/refman/8.0/en/creating-ssl-rsa-files.html
- https://dev.mysql.com/doc/refman/8.4/en/keyring-component-installation.html
