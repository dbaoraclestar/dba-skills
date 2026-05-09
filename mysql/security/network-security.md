# Network Security — SSL/TLS, Firewall, Connection Controls, and Network Hardening

## Overview

Network security for MySQL encompasses all measures that protect the database server at the
network layer: encrypting connections with SSL/TLS, restricting which hosts can connect,
limiting connection resources, firewalling SQL statements, and isolating administrative
traffic. These controls form the first line of defense against unauthorized access and
network-based attacks.

A properly hardened MySQL network configuration prevents eavesdropping on database traffic,
blocks connections from unauthorized sources, limits the blast radius of compromised
credentials, and provides DBA-only administrative channels that are isolated from application
traffic. These measures are especially critical in cloud and shared-network environments
where database servers may be reachable from broader network segments.

MySQL 8.0 introduced several network security enhancements including a dedicated
administrative port, connection compression controls, and the MySQL Enterprise Firewall.
Combined with OS-level firewalls and network segmentation, these features provide
defense-in-depth for production database infrastructure.

## Key Concepts

- **SSL/TLS**: Encryption protocol for securing client-server communication. MySQL supports TLS 1.2 and TLS 1.3.
- **require_secure_transport**: Server variable that rejects any unencrypted connection attempt.
- **bind-address**: Controls which network interfaces MySQL listens on for incoming connections.
- **admin_address**: Dedicated network address for administrative connections (8.0+), separate from application traffic.
- **MySQL Enterprise Firewall**: SQL-level firewall that learns and enforces allowed query patterns per user.
- **Connection Limits**: Server and per-user limits on concurrent connections to prevent resource exhaustion.
- **skip-networking**: Disables TCP/IP networking entirely, allowing only local Unix socket connections.

## SSL/TLS Setup

### Server Certificate Configuration

```ini
# my.cnf - Production SSL/TLS configuration
[mysqld]
# Certificate paths
ssl-ca   = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key  = /etc/mysql/ssl/server-key.pem

# Enforce minimum TLS version
tls_version = TLSv1.2,TLSv1.3

# TLS 1.2 cipher suites (restrict to strong ciphers)
ssl-cipher = ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256

# TLS 1.3 cipher suites
tls_ciphersuites = TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256

# Require encrypted connections for all users
require_secure_transport = ON
```

### Generating Production Certificates

```bash
# Generate a proper CA and server/client certificates

# 1. Create CA
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -nodes -days 3650 \
  -key ca-key.pem -out ca.pem \
  -subj "/C=US/O=Company/CN=MySQL Internal CA"

# 2. Create server certificate with SAN (Subject Alternative Name)
cat > server-ext.cnf << 'EXTEOF'
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = mysql-primary.company.com
DNS.2 = mysql-replica1.company.com
IP.1  = 10.0.1.10
IP.2  = 10.0.1.11
EXTEOF

openssl req -newkey rsa:4096 -days 1095 -nodes \
  -keyout server-key.pem -out server-req.pem \
  -subj "/C=US/O=Company/CN=mysql-primary.company.com"

openssl x509 -req -in server-req.pem -days 1095 \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem -extfile server-ext.cnf -extensions v3_req

# 3. Create client certificate
openssl req -newkey rsa:4096 -days 1095 -nodes \
  -keyout client-key.pem -out client-req.pem \
  -subj "/C=US/O=Company/CN=mysql-client"

openssl x509 -req -in client-req.pem -days 1095 \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out client-cert.pem

# 4. Verify the chain
openssl verify -CAfile ca.pem server-cert.pem
openssl verify -CAfile ca.pem client-cert.pem

# 5. Check certificate details
openssl x509 -in server-cert.pem -noout -text | grep -A2 "Subject Alternative Name"
openssl x509 -in server-cert.pem -noout -dates

# 6. Set permissions
chmod 600 *-key.pem
chown mysql:mysql /etc/mysql/ssl/*.pem
```

### Verifying SSL Status

```sql
-- Check server SSL configuration
SHOW VARIABLES LIKE 'have_ssl';              -- YES if SSL is available
SHOW VARIABLES LIKE 'require_secure_transport';
SHOW VARIABLES LIKE 'tls_version';
SHOW VARIABLES LIKE 'ssl_cipher';

-- Verify current connection is encrypted
SHOW STATUS LIKE 'Ssl_cipher';               -- Non-empty = encrypted
SHOW STATUS LIKE 'Ssl_version';              -- TLSv1.2 or TLSv1.3

-- Detailed SSL session info
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.session_status
WHERE VARIABLE_NAME IN (
  'Ssl_cipher', 'Ssl_version', 'Ssl_server_not_after',
  'Ssl_server_not_before', 'Ssl_sessions_reused'
);

-- Check which users require SSL
SELECT user, host, ssl_type, ssl_cipher, x509_issuer, x509_subject
FROM mysql.user
WHERE ssl_type != ''
ORDER BY user;

-- Identify unencrypted connections
SELECT pps.THREAD_ID, pps.PROCESSLIST_USER, pps.PROCESSLIST_HOST,
       pps.CONNECTION_TYPE,
       sslv.VARIABLE_VALUE AS ssl_version,
       sslc.VARIABLE_VALUE AS ssl_cipher
FROM performance_schema.threads pps
LEFT JOIN performance_schema.status_by_thread sslv
  ON pps.THREAD_ID = sslv.THREAD_ID AND sslv.VARIABLE_NAME = 'Ssl_version'
LEFT JOIN performance_schema.status_by_thread sslc
  ON pps.THREAD_ID = sslc.THREAD_ID AND sslc.VARIABLE_NAME = 'Ssl_cipher'
WHERE pps.TYPE = 'FOREGROUND'
  AND pps.PROCESSLIST_USER IS NOT NULL;
```

### Requiring SSL Per User

```sql
-- Require any SSL connection
ALTER USER 'app_user'@'%' REQUIRE SSL;

-- Require X.509 client certificate
ALTER USER 'admin_user'@'%' REQUIRE X509;

-- Require specific certificate attributes
ALTER USER 'strict_user'@'%'
  REQUIRE ISSUER '/C=US/O=Company/CN=MySQL Internal CA'
  AND SUBJECT '/C=US/O=Company/CN=mysql-client';

-- Require specific cipher
ALTER USER 'cipher_user'@'%'
  REQUIRE CIPHER 'ECDHE-RSA-AES256-GCM-SHA384';

-- Remove SSL requirement
ALTER USER 'user1'@'%' REQUIRE NONE;
```

## Connection Limits

### Server-Level Limits

```ini
# my.cnf - Connection limits
[mysqld]
# Maximum total connections
max_connections = 500

# Per-user connection limit (0 = use max_connections)
max_user_connections = 50

# Connection timeout for initial handshake (seconds)
connect_timeout = 10

# Timeout for idle connections (seconds)
wait_timeout = 28800              # non-interactive (8 hours)
interactive_timeout = 28800       # interactive clients

# Maximum connection errors before host is blocked
max_connect_errors = 100

# Thread cache for connection reuse
thread_cache_size = 50
```

### Per-User Connection Limits

```sql
-- Set connection limits per user account
ALTER USER 'app_user'@'%'
  WITH MAX_CONNECTIONS_PER_HOUR 1000
       MAX_QUERIES_PER_HOUR 100000
       MAX_UPDATES_PER_HOUR 10000
       MAX_USER_CONNECTIONS 20;

-- View current limits
SELECT user, host,
       max_connections, max_questions, max_updates, max_user_connections
FROM mysql.user
WHERE user = 'app_user';

-- Reset resource counters (without waiting for the hour to roll over)
FLUSH USER_RESOURCES;

-- Monitor current connection counts per user
SELECT USER, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS
FROM performance_schema.accounts
WHERE USER IS NOT NULL
ORDER BY CURRENT_CONNECTIONS DESC;

-- Check for hosts blocked due to max_connect_errors
SELECT IP, SUM_CONNECT_ERRORS, FIRST_ERROR, LAST_ERROR
FROM performance_schema.host_cache
WHERE SUM_CONNECT_ERRORS > 0
ORDER BY SUM_CONNECT_ERRORS DESC;

-- Unblock hosts
TRUNCATE TABLE performance_schema.host_cache;
-- or
FLUSH HOSTS;
```

## Bind Address and Network Interface Control

```ini
# my.cnf - Network binding options

# Listen on all interfaces (default, least secure)
bind-address = 0.0.0.0

# Listen only on localhost (most secure for local-only access)
bind-address = 127.0.0.1

# Listen on a specific interface
bind-address = 10.0.1.10

# Listen on multiple addresses (8.0.13+)
bind-address = 10.0.1.10,127.0.0.1

# Listen on IPv6
bind-address = ::

# Disable TCP networking entirely (Unix socket only)
skip-networking = ON

# Custom port
port = 3307

# Unix socket path
socket = /var/run/mysqld/mysqld.sock
```

## Administrative Port (MySQL 8.0+)

The admin port provides a separate network endpoint for DBA connections that is not
affected by `max_connections` limits on the main port.

```ini
# my.cnf - Administrative port configuration
[mysqld]
# Enable admin port on a separate address/port
admin_address = 127.0.0.1
admin_port = 33062

# Separate SSL for admin connections (optional)
admin_ssl_ca   = /etc/mysql/ssl/admin-ca.pem
admin_ssl_cert = /etc/mysql/ssl/admin-cert.pem
admin_ssl_key  = /etc/mysql/ssl/admin-key.pem

# Require admin SSL
admin_tls_version = TLSv1.3

# Limit admin connections
create_admin_listener_thread = ON
```

```sql
-- Grant SERVICE_CONNECTION_ADMIN for admin port access (required)
GRANT SERVICE_CONNECTION_ADMIN ON *.* TO 'dba_user'@'localhost';

-- Connect via admin port
-- mysql -u dba_user -p -h 127.0.0.1 -P 33062

-- Verify admin port is listening
SHOW VARIABLES LIKE 'admin_address';
SHOW VARIABLES LIKE 'admin_port';
```

## MySQL Enterprise Firewall

The Enterprise Firewall learns allowed SQL patterns for each user and blocks queries
that do not match the learned whitelist.

```sql
-- Install the firewall plugin
INSTALL PLUGIN mysql_firewall SONAME 'firewall.so';
INSTALL PLUGIN mysql_firewall_users SONAME 'firewall.so';
INSTALL PLUGIN mysql_firewall_whitelist SONAME 'firewall.so';

-- Enable the firewall
SET GLOBAL mysql_firewall_mode = ON;

-- Step 1: Set user to RECORDING mode (learning phase)
CALL mysql.sp_set_firewall_mode('app_user@10.0.0.%', 'RECORDING');

-- Step 2: Run representative application workload
-- (All queries executed by this user are recorded as allowed patterns)

-- Step 3: Switch to PROTECTING mode (enforcement)
CALL mysql.sp_set_firewall_mode('app_user@10.0.0.%', 'PROTECTING');

-- Now any query that doesn't match a recorded pattern is BLOCKED

-- Step 4: Check firewall status
SELECT * FROM information_schema.MYSQL_FIREWALL_USERS;
SELECT * FROM information_schema.MYSQL_FIREWALL_WHITELIST
WHERE USERHOST = 'app_user@10.0.0.%';

-- View blocked queries in error log
-- Check for: "[Note] Plugin MYSQL_FIREWALL reported: 'ACCESS DENIED...'"

-- DETECTING mode: log but don't block (useful for testing)
CALL mysql.sp_set_firewall_mode('app_user@10.0.0.%', 'DETECTING');

-- Disable firewall for a user
CALL mysql.sp_set_firewall_mode('app_user@10.0.0.%', 'OFF');

-- Group profiles (8.0.23+): apply firewall rules to groups of users
CALL mysql.sp_set_firewall_group_mode('app_group', 'RECORDING');
CALL mysql.sp_firewall_group_enlist('app_group', 'app_user@10.0.0.%');
-- After recording:
CALL mysql.sp_set_firewall_group_mode('app_group', 'PROTECTING');
```

## Connection Compression

```ini
# my.cnf - Connection compression
[mysqld]
# Compression algorithms (8.0.18+)
protocol_compression_algorithms = zlib,zstd,uncompressed

# Legacy compression (5.7 compatible)
# Clients request with --compress flag
```

```sql
-- Check compression status of current connection
SHOW STATUS LIKE 'Compression';
SHOW STATUS LIKE 'Compression_algorithm';
SHOW STATUS LIKE 'Compression_level';

-- View server-supported algorithms
SHOW VARIABLES LIKE 'protocol_compression_algorithms';

-- Per-connection from client:
-- mysql -u user -p -h host --compression-algorithms=zstd --zstd-compression-level=3
```

## Proxy Protocol Support

Proxy protocol preserves the original client IP address when connections pass through
a load balancer or proxy.

```ini
# my.cnf - Proxy protocol (8.0.14+)
[mysqld]
# Enable proxy protocol on the main port
# Only listed subnets are expected to send proxy protocol headers
proxy_protocol_networks = "10.0.0.0/8,192.168.1.0/24"

# Note: connections from listed networks MUST send proxy protocol headers
# connections from other networks connect normally
```

```sql
-- Verify proxy protocol configuration
SHOW VARIABLES LIKE 'proxy_protocol_networks';

-- With proxy protocol, PROCESSLIST_HOST shows the real client IP
-- (not the load balancer IP)
SELECT PROCESSLIST_USER, PROCESSLIST_HOST
FROM performance_schema.threads
WHERE TYPE = 'FOREGROUND';
```

## OS-Level Network Hardening

```bash
# iptables rules for MySQL server

# Allow MySQL from application subnet only
iptables -A INPUT -p tcp --dport 3306 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -j DROP

# Allow admin port from DBA jump host only
iptables -A INPUT -p tcp --dport 33062 -s 10.0.0.5/32 -j ACCEPT
iptables -A INPUT -p tcp --dport 33062 -j DROP

# Allow replication from replica servers
iptables -A INPUT -p tcp --dport 3306 -s 10.0.2.10/32 -j ACCEPT
iptables -A INPUT -p tcp --dport 3306 -s 10.0.2.11/32 -j ACCEPT

# firewalld alternative
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.1.0/24" port protocol="tcp" port="3306" accept'
firewall-cmd --reload

# Verify listening ports
ss -tlnp | grep mysqld
# Expected: *:3306 and 127.0.0.1:33062
```

## Network Security Audit Queries

```sql
-- Find accounts accessible from any host
SELECT user, host FROM mysql.user
WHERE host = '%'
  AND user NOT LIKE 'mysql.%'
ORDER BY user;

-- Find accounts without SSL requirement
SELECT user, host, ssl_type
FROM mysql.user
WHERE ssl_type = ''
  AND user NOT LIKE 'mysql.%'
  AND host != 'localhost'
ORDER BY user;

-- Check for connections from unexpected sources
SELECT HOST, CURRENT_CONNECTIONS, TOTAL_CONNECTIONS
FROM performance_schema.hosts
WHERE HOST NOT IN ('localhost', '10.0.1.%', '10.0.2.%')
  AND HOST IS NOT NULL
  AND TOTAL_CONNECTIONS > 0
ORDER BY TOTAL_CONNECTIONS DESC;

-- Monitor connection attempts and failures
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
  'Connections', 'Aborted_clients', 'Aborted_connects',
  'Connection_errors_accept', 'Connection_errors_internal',
  'Connection_errors_max_connections',
  'Connection_errors_peer_address',
  'Connection_errors_tcpwrap'
);
```

## Best Practices

- Set `require_secure_transport = ON` in production to enforce SSL/TLS on all connections.
- Use `bind-address` to listen only on interfaces that need database access; never bind to 0.0.0.0 if only internal access is needed.
- Configure the admin port (`admin_address`, `admin_port`) for DBA connections separate from application traffic.
- Restrict host patterns in user accounts to specific subnets rather than using `'%'`.
- Set `max_user_connections` per application account to prevent a single application from consuming all connections.
- Use OS-level firewalls (iptables/firewalld) as an additional layer; do not rely solely on MySQL's host-based access control.
- Enable proxy protocol when using load balancers to preserve real client IP addresses for auditing and access control.
- Monitor `Aborted_connects` and `Connection_errors_max_connections` to detect connection storms or attacks.
- Disable `skip-name-resolve` to prevent DNS-based spoofing attacks and improve connection performance.
- Review `performance_schema.host_cache` regularly for blocked hosts that may indicate brute-force attempts.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Binding to 0.0.0.0 in production without firewall rules | MySQL accessible from any network that can route to the server | Use specific bind-address or implement OS-level firewall rules |
| Setting max_connections too high without per-user limits | Single compromised account can exhaust all server resources | Set MAX_USER_CONNECTIONS per account proportional to expected load |
| Not configuring admin_address | DBA locked out when max_connections is reached during connection storms | Configure admin_address on localhost with SERVICE_CONNECTION_ADMIN grant |
| Using skip-networking when replication or remote monitoring is needed | Replicas and monitoring tools cannot connect | Use bind-address restrictions and SSL instead of disabling networking entirely |
| Proxy protocol enabled but load balancer not configured to send headers | All connections from the load balancer subnet are rejected | Coordinate proxy protocol configuration between MySQL and the load balancer |
| Relying only on MySQL host restrictions without OS-level firewall | Port scanning and connection attempts still reach MySQL process | Layer OS firewall rules with MySQL access control for defense-in-depth |

## MySQL Version Notes

- **5.7**: SSL auto-setup on first start. `mysql_ssl_rsa_setup` utility. No admin port. No proxy protocol. Connection compression via `--compress` flag only (zlib). Enterprise Firewall available (user-level profiles only). `bind-address` supports single address only.
- **8.0**: Admin port (`admin_address`, `admin_port`) added. Proxy protocol support (8.0.14). Multiple bind addresses (8.0.13). `protocol_compression_algorithms` with zstd support (8.0.18). Enterprise Firewall group profiles (8.0.23). TLS 1.3 support. `require_secure_transport` available. `Connection_errors_*` status variables enhanced. `skip-name-resolve` improvements.
- **8.4/9.x**: Admin port SSL improvements. Enhanced TLS cipher suite management. Connection attribute logging improvements. Performance improvements for SSL handshake. Deprecated older TLS versions more aggressively. Improved proxy protocol IPv6 support.

## Sources

- https://dev.mysql.com/doc/refman/8.0/en/using-encrypted-connections.html
- https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_admin_address
- https://dev.mysql.com/doc/refman/8.0/en/firewall.html
- https://dev.mysql.com/doc/refman/8.0/en/connection-control.html
- https://dev.mysql.com/doc/refman/8.0/en/proxy-protocol-support.html
- https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_bind_address
