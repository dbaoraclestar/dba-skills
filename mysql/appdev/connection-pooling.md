# MySQL Connection Pooling — Application Pools, ProxySQL, MySQL Router, and Thread Pool

## Overview

Connection pooling is the practice of maintaining a cache of reusable database connections to eliminate the overhead of creating and destroying connections for each client request. Without pooling, every request pays the cost of TCP handshake, TLS negotiation, MySQL authentication, and session initialization -- typically 5-50ms per connection.

Pooling exists at multiple layers: application-level (within the app process), middleware-level (ProxySQL, MySQL Router), and server-level (the thread_pool plugin). The right combination depends on your architecture. A typical production setup uses application-level pooling (HikariCP, SQLAlchemy pool, or mysql2 pool) combined with ProxySQL for connection multiplexing and query routing.

This skill covers each pooling layer, configuration patterns, `max_connections` tuning, and diagnosing connection exhaustion.

## Key Concepts

- **Connection pool**: A fixed set of pre-established connections shared among application threads/requests. Connections are checked out, used, and returned.
- **Connection multiplexing**: A proxy (like ProxySQL) maps many frontend (client) connections to fewer backend (MySQL) connections. Reduces server load when clients are mostly idle.
- **Pool size**: The maximum number of connections in the pool. Undersized pools cause request queuing; oversized pools waste server memory and cause context switching.
- **Connection lifecycle**: Connections age out. Pools evict idle connections and replace broken ones. `maxLifetime` (pool-side) should be less than `wait_timeout` (server-side).
- **max_connections**: MySQL server limit on total simultaneous connections. Default is 151. Each connection consumes ~10MB or more of server memory.

## Application-Level Pooling

### HikariCP (Java)

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/mydb?rewriteBatchedStatements=true");
config.setUsername("app_user");
config.setPassword("secret");

// Pool sizing: connections = ((core_count * 2) + disk_spindles)
// For a 4-core server with SSD: ~10 connections is often optimal
config.setMaximumPoolSize(10);
config.setMinimumIdle(5);

// Lifecycle
config.setIdleTimeout(600_000);       // 10 min idle before eviction
config.setMaxLifetime(1_800_000);     // 30 min max lifetime
config.setConnectionTimeout(10_000);  // 10 sec to get connection from pool
config.setKeepaliveTime(300_000);     // 5 min keepalive ping

// Validation
config.setConnectionTestQuery("SELECT 1");

// Performance
config.addDataSourceProperty("cachePrepStmts", "true");
config.addDataSourceProperty("prepStmtCacheSize", "250");
config.addDataSourceProperty("useServerPrepStmts", "true");

HikariDataSource ds = new HikariDataSource(config);
```

### SQLAlchemy pool (Python)

```python
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://app_user:secret@127.0.0.1:3306/mydb?charset=utf8mb4",
    pool_size=10,          # maintained connections
    max_overflow=20,       # temporary extra connections under load
    pool_timeout=10,       # seconds to wait for connection
    pool_recycle=1800,     # seconds before recycling (< wait_timeout)
    pool_pre_ping=True,    # test connection before use
    echo_pool="debug",     # log pool events (disable in prod)
)

# Pool status
print(engine.pool.status())
# Pool size: 10  Connections in pool: 8  Current Overflow: 2
```

### mysql2 pool (Node.js)

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: '127.0.0.1',
  user: 'app_user',
  password: 'secret',
  database: 'mydb',
  connectionLimit: 20,      // max pool size
  maxIdle: 10,              // max idle connections
  idleTimeout: 60000,       // ms before idle eviction
  waitForConnections: true, // queue when pool exhausted
  queueLimit: 0,            // unlimited queue (set limit in prod)
  enableKeepAlive: true,
  keepAliveInitialDelay: 30000,
});
```

## ProxySQL

ProxySQL is a high-performance MySQL proxy that provides connection multiplexing, query routing, query caching, read/write splitting, and connection pooling at the middleware layer.

### Architecture

```
App (1000 connections) --> ProxySQL (port 6033) --> MySQL (50 connections)
```

### Installation and basic configuration

```bash
# Install (Debian/Ubuntu)
apt-get install proxysql2

# Admin interface
mysql -h 127.0.0.1 -P 6032 -u admin -padmin --prompt='ProxySQLAdmin> '
```

```sql
-- Add backend MySQL servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES (10, '10.0.1.100', 3306, 100);  -- writer

INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES (20, '10.0.1.101', 3306, 100),  -- reader 1
       (20, '10.0.1.102', 3306, 100);  -- reader 2

-- Configure connection multiplexing
UPDATE global_variables SET variable_value = 200
WHERE variable_name = 'mysql-max_connections';

UPDATE global_variables SET variable_value = 1
WHERE variable_name = 'mysql-multiplexing';

-- Read/write splitting rules
INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup, apply)
VALUES (1, '^SELECT .* FOR UPDATE', 10, 1),   -- writes to hostgroup 10
       (2, '^SELECT',               20, 1);    -- reads to hostgroup 20

-- Query caching (cache SELECTs for 5 seconds)
INSERT INTO mysql_query_rules (rule_id, match_pattern, cache_ttl, apply)
VALUES (3, '^SELECT .* FROM config', 5000, 1);

-- Apply changes
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;
```

### Monitoring ProxySQL

```sql
-- Connection pool stats
SELECT hostgroup, srv_host, srv_port, status,
       ConnUsed, ConnFree, ConnOK, ConnERR, MaxConnUsed
FROM stats_mysql_connection_pool;

-- Query digest (top queries by time)
SELECT digest_text, count_star, sum_time,
       ROUND(sum_time/count_star) AS avg_time_us
FROM stats_mysql_query_digest
ORDER BY sum_time DESC LIMIT 20;

-- Connection multiplexing efficiency
SELECT * FROM stats_mysql_global
WHERE variable_name LIKE '%multiplex%';
```

## MySQL Router

MySQL Router is Oracle's official middleware for InnoDB Cluster and Group Replication. It provides transparent failover and load balancing.

```bash
# Bootstrap from InnoDB Cluster
mysqlrouter --bootstrap root@primary:3306 --directory /opt/mysqlrouter --user mysqlrouter

# Ports after bootstrap:
# 6446 = read-write (routes to primary)
# 6447 = read-only (round-robin across secondaries)
```

```python
# Application connects to Router, not directly to MySQL
engine = create_engine(
    "mysql+pymysql://app_user:secret@127.0.0.1:6446/mydb",  # R/W port
    pool_pre_ping=True
)

read_engine = create_engine(
    "mysql+pymysql://app_user:secret@127.0.0.1:6447/mydb",  # RO port
    pool_pre_ping=True
)
```

## Thread Pool Plugin (Enterprise)

The thread_pool plugin is a server-side feature (MySQL Enterprise Edition) that limits the number of concurrently executing threads, preventing the server from being overwhelmed by too many simultaneous queries.

```sql
-- Enable thread pool (my.cnf)
-- [mysqld]
-- plugin-load-add = thread_pool.so
-- thread_handling = pool-of-threads
-- thread_pool_size = 16            -- typically = CPU cores
-- thread_pool_max_transactions_limit = 100

-- Monitor thread pool
SELECT * FROM performance_schema.tp_thread_group_stats;
SELECT * FROM performance_schema.tp_thread_state;
```

## max_connections Tuning

```sql
-- Check current setting
SHOW VARIABLES LIKE 'max_connections';

-- Check current usage
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Threads_running';

-- Peak usage ratio
SELECT
    @@max_connections AS max_connections,
    variable_value AS max_used,
    ROUND(variable_value / @@max_connections * 100, 1) AS pct_used
FROM performance_schema.global_status
WHERE variable_name = 'Max_used_connections';
```

### Sizing formula

```
max_connections = (app_pool_size * num_app_instances) + admin_reserve + replication_threads

Example:
  App pool size:     20
  App instances:      5
  Admin reserve:     10
  Replication:        2
  Total:            112  --> set max_connections = 150 (with headroom)
```

### Memory impact

```sql
-- Per-connection memory (approximate)
-- Base: ~10KB (thread stack, net buffer)
-- Active query: read_buffer_size + sort_buffer_size + join_buffer_size + tmp_table_size
-- Worst case per connection: ~10-20MB

-- Estimate total memory at max connections
SELECT @@max_connections *
    (@@read_buffer_size + @@sort_buffer_size + @@join_buffer_size) / 1024 / 1024
AS estimated_per_connection_mb;
```

## Diagnosing Connection Exhaustion

```sql
-- Current connections by user and host
SELECT user, host, db, command, time, state, COUNT(*) AS cnt
FROM information_schema.processlist
GROUP BY user, host, db, command, state
ORDER BY cnt DESC;

-- Connections in Sleep state (potential leaks)
SELECT user, host, db, time AS idle_seconds
FROM information_schema.processlist
WHERE command = 'Sleep'
ORDER BY time DESC
LIMIT 20;

-- Check if connections are being refused
SHOW STATUS LIKE 'Aborted_connects';
SHOW STATUS LIKE 'Connection_errors%';

-- Kill long-idle connections (emergency)
SELECT CONCAT('KILL ', id, ';') AS kill_cmd
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 3600;
```

### Detecting pool leaks in application

```python
# SQLAlchemy: monitor pool events
from sqlalchemy import event

@event.listens_for(engine, "checkout")
def on_checkout(dbapi_conn, connection_record, connection_proxy):
    logging.debug("Connection checked out from pool")

@event.listens_for(engine, "checkin")
def on_checkin(dbapi_conn, connection_record):
    logging.debug("Connection returned to pool")

# Log pool status periodically
logging.info(f"Pool status: {engine.pool.status()}")
```

## Best Practices

- Size application pools conservatively. For most workloads, 10-20 connections per app instance is sufficient. More connections cause context switching overhead on the server.
- Set pool `maxLifetime` (or `pool_recycle`) to less than MySQL's `wait_timeout` (default 28800s) to prevent stale connections.
- Use `pool_pre_ping` (SQLAlchemy) or `connectionTestQuery` (HikariCP) to detect broken connections before use.
- Deploy ProxySQL when you need connection multiplexing (many clients, few active queries), read/write splitting, or query-level routing.
- Reserve 10-20 connections in `max_connections` for admin/monitoring access so DBAs can connect during emergencies.
- Monitor `Max_used_connections` to understand peak demand. Set `max_connections` to 1.5x the observed peak.
- Always close/release connections in `finally` blocks or use context managers. Leaked connections are the most common cause of pool exhaustion.
- Use MySQL Router for InnoDB Cluster environments; use ProxySQL for more advanced routing and multiplexing needs.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Setting pool size equal to `max_connections` | No room for admin, replication, or other apps | Pool size should be a fraction of `max_connections` |
| `pool_recycle` longer than `wait_timeout` | Stale connections cause "MySQL server has gone away" | Set `pool_recycle` < `wait_timeout` |
| Not using `pool_pre_ping` | First query on stale connection fails | Enable pre-ping in pool config |
| Creating a new pool per request (e.g., per Lambda invocation) | Pool defeats its own purpose, connections spike | Create pool once at startup or use an external pooler |
| Setting `max_connections = 10000` as a "fix" | Server runs out of memory, crashes | Tune properly with pooling; reduce rather than increase |
| ProxySQL without monitoring | Query rules may route incorrectly, silent failures | Monitor `stats_mysql_query_digest` and connection pool stats |

## MySQL Version Notes

- **5.7**: `max_connections` default is 151. No thread_pool in Community. ProxySQL and MySQL Router both work.
- **8.0**: Same defaults. `performance_schema.global_status` replaces `SHOW GLOBAL STATUS` in some contexts. MySQL Router supports InnoDB Cluster natively. `caching_sha2_password` may require ProxySQL 2.0.12+ for compatibility.
- **8.4 / 9.x**: Thread pool available in Community Edition (8.4+). Connection handling improvements reduce per-connection memory. ProxySQL 2.6+ required for full compatibility.

## Sources

- [MySQL max_connections](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_connections)
- [ProxySQL Documentation](https://proxysql.com/documentation/)
- [MySQL Router Documentation](https://dev.mysql.com/doc/mysql-router/8.0/en/)
- [HikariCP Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [MySQL Thread Pool](https://dev.mysql.com/doc/refman/8.0/en/thread-pool.html)
