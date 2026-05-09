# Auditing — Audit Plugins, Log Filtering, and Compliance

## Overview

Database auditing records who did what, when, and from where within a MySQL instance.
Audit logs are essential for security investigations, compliance requirements, and
operational accountability. They provide a tamper-evident trail of connections, queries,
and administrative actions that can be reviewed during incidents or regulatory audits.

MySQL offers auditing through the MySQL Enterprise Audit plugin (commercial) and through
community alternatives like the Percona Audit Log plugin and the MariaDB Audit Plugin.
Each solution captures similar events but differs in configuration, filtering capabilities,
and output formats.

Proper audit configuration balances security coverage with performance impact. Logging
every query on a high-throughput OLTP system can degrade performance and produce
unmanageable log volumes. Effective auditing uses filtering to capture security-relevant
events while minimizing overhead on normal operations.

## Key Concepts

- **Audit Plugin**: A server plugin that intercepts server events (connections, queries, errors) and writes them to an audit log file.
- **Audit Log Filter**: Rules that control which events are recorded and which are excluded. Reduces log volume and performance impact.
- **Audit Log Format**: The structure of audit log entries. Options include old-style XML, new-style XML, JSON (8.0+), and CSV.
- **Connection Events**: Records of user login and logout, including source IP, authentication method, and SSL status.
- **Query Events**: Records of SQL statements executed, including the statement text, affected schema, and execution status.
- **Table Access Events**: Records of which tables were read from or written to during query execution.
- **Compliance Framework**: Regulatory requirements (PCI-DSS, HIPAA, SOX, GDPR) that mandate specific audit capabilities.

## MySQL Enterprise Audit Plugin

### Installation and Setup

```sql
-- Install the Enterprise Audit plugin
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- Verify installation
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME = 'audit_log';

-- Check audit log status
SHOW VARIABLES LIKE 'audit_log%';
SHOW STATUS LIKE 'Audit_log%';
```

```ini
# my.cnf configuration
[mysqld]
plugin-load-add = audit_log.so

# Log format: OLD (XML), NEW (XML), JSON, CSV
audit_log_format = JSON

# Log file location
audit_log_file = /var/log/mysql/audit.log

# Logging policy: ALL, LOGINS, QUERIES, NONE
audit_log_policy = ALL

# Buffer size for async writes
audit_log_buffer_size = 16777216    # 16MB

# Rotation: size-based
audit_log_rotate_on_size = 536870912    # 512MB

# Maximum retained log files (8.0.24+)
audit_log_max_size = 5368709120     # 5GB total

# Compression (8.0.17+)
audit_log_compression = GZIP

# Encryption (8.0.17+)
audit_log_encryption = AES
```

### Audit Log Filtering (JSON-based, 8.0+)

```sql
-- Create a filter that logs all connection events and DDL statements
SELECT audit_log_filter_set_filter('security_filter', '{
  "filter": {
    "class": [
      {
        "name": "connection",
        "event": {
          "name": ["connect", "disconnect", "change_user"],
          "log": true
        }
      },
      {
        "name": "general",
        "event": {
          "name": ["status"],
          "log": {
            "field": {
              "name": "general_command.str",
              "value": ["Query", "Execute"]
            }
          }
        }
      }
    ]
  }
}');

-- Assign filter to a user (all activity by this user is filtered through it)
SELECT audit_log_filter_set_user('%', 'security_filter');

-- Create a filter for specific users only
SELECT audit_log_filter_set_filter('dba_filter', '{
  "filter": {
    "class": {
      "name": "general",
      "event": {
        "name": "status",
        "log": true
      }
    }
  }
}');
SELECT audit_log_filter_set_user('dba_admin@%', 'dba_filter');

-- Remove a filter assignment
SELECT audit_log_filter_remove_user('%');

-- Remove a filter definition
SELECT audit_log_filter_remove_filter('security_filter');

-- List current filters
SELECT * FROM mysql.audit_log_filter;
SELECT * FROM mysql.audit_log_user;
```

### Audit Log Rotation and Management

```sql
-- Manual log rotation
SET GLOBAL audit_log_rotate_on_size = 0;
SELECT audit_log_rotate();

-- Read audit log entries (JSON format, 8.0.22+)
SELECT audit_log_read(audit_log_read_bookmark());

-- Read specific entries with filtering
SELECT audit_log_read('{"start": {"timestamp": "2026-05-01 00:00:00"}}');

-- Prune old audit logs (8.0.24+)
SELECT audit_log_prune('2026-04-01 00:00:00');
```

### Example Audit Log Entries (JSON Format)

```json
{
  "timestamp": "2026-05-08 14:23:01",
  "id": 1,
  "class": "connection",
  "event": "connect",
  "connection_id": 42,
  "account": { "user": "app_user", "host": "10.0.1.50" },
  "login": { "user": "app_user", "os": "", "ip": "10.0.1.50", "proxy": "" },
  "connection_data": {
    "connection_type": "SSL/TLS",
    "status": 0,
    "db": "app_production"
  }
}

{
  "timestamp": "2026-05-08 14:23:05",
  "id": 2,
  "class": "general",
  "event": "status",
  "connection_id": 42,
  "account": { "user": "app_user", "host": "10.0.1.50" },
  "general_data": {
    "command": "Query",
    "sql_command": "select",
    "query": "SELECT * FROM customers WHERE id = 12345",
    "status": 0
  }
}
```

## Percona Audit Log Plugin (Community)

The Percona Audit Log plugin provides audit capabilities for Percona Server and is
compatible with standard MySQL community edition.

```ini
# my.cnf configuration for Percona Audit Log
[mysqld]
plugin-load-add = audit_log.so

# Output format: OLD, NEW, JSON, CSV
audit_log_format = JSON

# Logging policy: ALL, LOGINS, QUERIES, NONE
audit_log_policy = ALL

# Output handler: FILE, SYSLOG
audit_log_handler = FILE
audit_log_file = /var/log/mysql/percona_audit.log

# Size-based rotation
audit_log_rotate_on_size = 536870912

# Maximum number of rotated files to keep
audit_log_rotations = 10

# Include/exclude users
audit_log_include_accounts = 'app_user@%,admin_user@%'
# OR
audit_log_exclude_accounts = 'monitor_user@localhost,repl_user@%'

# Include/exclude databases
audit_log_include_databases = 'app_production,financial_data'
# OR
audit_log_exclude_databases = 'information_schema,performance_schema,sys,mysql'

# Include/exclude commands
audit_log_include_commands = 'insert,update,delete,create_table,drop_table,alter_table,grant'
# OR
audit_log_exclude_commands = 'select,show_status,show_variables,ping'
```

```sql
-- Install Percona Audit Log
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- Verify
SHOW VARIABLES LIKE 'audit_log%';

-- Dynamic filtering changes (no restart needed)
SET GLOBAL audit_log_include_accounts = 'admin@%,root@localhost';
SET GLOBAL audit_log_exclude_commands = 'select,show_databases';

-- Manual rotation
SET GLOBAL audit_log_rotate_on_size = 0;
SET GLOBAL audit_log_flush = ON;
```

## MariaDB Audit Plugin

The MariaDB Audit Plugin can be used with standard MySQL (up to MySQL 5.7) for basic
audit capabilities without Enterprise licensing.

```ini
# my.cnf for MariaDB Audit Plugin on MySQL
[mysqld]
plugin-load-add = server_audit.so

# Events to log: CONNECT, QUERY, TABLE, QUERY_DDL, QUERY_DML, QUERY_DCL
server_audit_events = CONNECT,QUERY_DDL,QUERY_DCL

# Output type: file, syslog
server_audit_output_type = file
server_audit_file_path = /var/log/mysql/mariadb_audit.log

# Rotation
server_audit_file_rotate_size = 536870912
server_audit_file_rotations = 10

# User filtering
server_audit_incl_users = admin_user,dba_user
# OR
server_audit_excl_users = monitoring_user,repl_user

# Logging control
server_audit_logging = ON
```

```sql
-- Install MariaDB Audit Plugin
INSTALL PLUGIN server_audit SONAME 'server_audit.so';

-- Enable logging
SET GLOBAL server_audit_logging = ON;

-- Filter to DDL and DCL only (schema changes and privilege changes)
SET GLOBAL server_audit_events = 'QUERY_DDL,QUERY_DCL,CONNECT';
```

## Compliance Considerations

### PCI-DSS Requirements

```sql
-- PCI-DSS 10.2: Log the following events
-- 10.2.1: All individual user access to cardholder data
-- 10.2.2: All actions taken by any individual with root or admin privileges
-- 10.2.4: Invalid logical access attempts
-- 10.2.5: Use of and changes to identification and authentication mechanisms
-- 10.2.6: Initialization, stopping, or pausing of audit logs

-- Filter for PCI-DSS compliance (Enterprise Audit)
SELECT audit_log_filter_set_filter('pci_filter', '{
  "filter": {
    "class": [
      {
        "name": "connection",
        "log": true
      },
      {
        "name": "general",
        "event": {
          "name": "status",
          "log": true
        }
      },
      {
        "name": "table_access",
        "log": true
      }
    ]
  }
}');

-- Monitor for stopped/paused audit logging
-- Set up external monitoring to alert if audit_log plugin is unloaded
-- Check periodically:
SELECT PLUGIN_STATUS FROM information_schema.PLUGINS
WHERE PLUGIN_NAME = 'audit_log';
```

### HIPAA Audit Requirements

```sql
-- HIPAA requires audit controls for ePHI access
-- Log all access to databases containing protected health information

-- Percona approach: include only PHI databases
SET GLOBAL audit_log_include_databases = 'patient_records,medical_claims,pharmacy';
SET GLOBAL audit_log_policy = 'ALL';

-- Enterprise approach: table-access filtering for PHI tables
SELECT audit_log_filter_set_filter('hipaa_filter', '{
  "filter": {
    "class": [
      { "name": "connection", "log": true },
      {
        "name": "table_access",
        "log": true
      }
    ]
  }
}');
```

### SOX Compliance

```sql
-- SOX requires audit trail for financial data changes
-- Focus on DML and DDL on financial databases

-- Percona approach
SET GLOBAL audit_log_include_databases = 'general_ledger,accounts_payable,accounts_receivable';
SET GLOBAL audit_log_include_commands = 'insert,update,delete,create_table,alter_table,drop_table,grant,revoke';
```

## Analyzing Audit Logs

```bash
# Parse JSON audit log for failed logins
jq 'select(.class == "connection" and .event == "connect" 
    and .connection_data.status != 0)' /var/log/mysql/audit.log

# Find all DDL statements
jq 'select(.general_data.sql_command 
    | test("create|alter|drop|truncate"))' /var/log/mysql/audit.log

# Find queries by specific user
jq 'select(.account.user == "admin_user" 
    and .class == "general")' /var/log/mysql/audit.log

# Count events per user
jq -r '.account.user' /var/log/mysql/audit.log | sort | uniq -c | sort -rn

# Find privilege changes (GRANT/REVOKE)
jq 'select(.general_data.sql_command 
    | test("grant|revoke"))' /var/log/mysql/audit.log

# Extract connection events with timestamps
jq -r 'select(.class == "connection") 
    | [.timestamp, .event, .account.user, .account.host, 
       .connection_data.status] | @csv' /var/log/mysql/audit.log
```

## Performance Impact Monitoring

```sql
-- Monitor audit log performance overhead
SHOW STATUS LIKE 'Audit_log%';

-- Key metrics:
-- Audit_log_current_size      - Current log file size
-- Audit_log_events            - Total events logged
-- Audit_log_events_filtered   - Events excluded by filters
-- Audit_log_events_lost       - Events dropped (buffer overflow)
-- Audit_log_events_written    - Events written to log
-- Audit_log_total_size        - Total size of all log files
-- Audit_log_write_waits       - Times a write had to wait for buffer space

-- If Audit_log_events_lost > 0, increase buffer size:
SET GLOBAL audit_log_buffer_size = 33554432;   -- 32MB
```

## Best Practices

- Start with connection events and DDL/DCL logging; add DML logging selectively for sensitive tables.
- Use JSON format for audit logs -- it is easier to parse programmatically than XML.
- Implement log rotation and retention policies before enabling auditing in production.
- Ship audit logs to a centralized SIEM (Splunk, ELK, Datadog) for tamper-proof storage and alerting.
- Exclude monitoring and replication accounts from audit logging to reduce noise.
- Never store audit logs on the same filesystem as MySQL data files.
- Monitor `Audit_log_events_lost` to detect buffer overflow conditions.
- Test audit configuration on a staging instance first to measure performance impact.
- Document which events are logged and why, mapping to specific compliance requirements.
- Set up alerts for audit plugin being unloaded or audit logging being disabled.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Logging ALL events including SELECT on high-throughput OLTP | Severe performance degradation and massive log files | Use filters to log only connections, DDL, DCL, and DML on sensitive tables |
| Storing audit logs on the same disk as MySQL data | Audit log growth can fill the data disk, crashing MySQL | Use a separate filesystem or ship logs to remote storage |
| Not monitoring Audit_log_events_lost | Events silently dropped without alerting | Monitor this counter and increase audit_log_buffer_size if non-zero |
| Using the MariaDB Audit Plugin on MySQL 8.0 | Compatibility issues and potential crashes | Use Enterprise Audit or Percona Audit Log for MySQL 8.0+ |
| No log rotation or retention policy | Audit logs grow unbounded until disk is full | Configure audit_log_rotate_on_size and audit_log_max_size |
| Excluding too many events for "performance" | Missing critical security events during investigation | Start broad, then selectively exclude only high-volume, low-value events |

## MySQL Version Notes

- **5.7**: Enterprise Audit plugin available (XML format). MariaDB Audit Plugin compatible. Percona Audit Log plugin available for Percona Server 5.7. No JSON format. No rule-based filtering (only policy-level: ALL, LOGINS, QUERIES, NONE). No audit log encryption or compression.
- **8.0**: JSON log format added. Rule-based filtering with `audit_log_filter_set_filter()`. Audit log compression (GZIP, 8.0.17). Audit log encryption (AES, 8.0.17). `audit_log_read()` for programmatic access (8.0.22). `audit_log_max_size` for retention (8.0.24). `audit_log_prune()` for cleanup. MariaDB Audit Plugin NOT recommended for MySQL 8.0.
- **8.4/9.x**: Audit log component replaces plugin architecture. Enhanced JSON filtering syntax. Improved performance for high-throughput auditing. Better integration with keyring components for log encryption.

## Sources

- https://dev.mysql.com/doc/refman/8.0/en/audit-log.html
- https://dev.mysql.com/doc/refman/8.0/en/audit-log-filtering.html
- https://dev.mysql.com/doc/refman/8.0/en/audit-log-file-formats.html
- https://docs.percona.com/percona-server/8.0/audit-log-plugin.html
- https://dev.mysql.com/doc/refman/8.0/en/audit-log-reference.html
