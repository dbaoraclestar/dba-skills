# MySQL Router — Connection Routing for InnoDB Cluster and High Availability

## Overview

MySQL Router is a lightweight middleware process that sits between application clients and MySQL server backends, providing transparent connection routing, load balancing, and automatic failover. It is the standard routing component in the InnoDB Cluster architecture, working alongside Group Replication and MySQL Shell to deliver high availability without application-level changes.

Router reads cluster metadata to discover the current topology (which instance is primary, which are secondaries), routes read-write connections to the primary, and distributes read-only connections across secondaries. When a failover occurs, Router detects the topology change through its metadata cache and automatically redirects new connections to the new primary within seconds.

This skill covers Router bootstrap and configuration, routing modes, read-write splitting, metadata cache behavior, monitoring, deployment patterns, and troubleshooting for InnoDB Cluster environments.

## Key Concepts

- **MySQL Router**: A proxy process that routes MySQL client connections to appropriate backend servers.
- **Bootstrap**: The process of configuring Router by connecting to an InnoDB Cluster and auto-generating its configuration file.
- **Metadata cache**: An in-memory cache of cluster topology that Router refreshes periodically to detect failovers.
- **Routing strategy**: The algorithm Router uses to select a backend: `first-available`, `next-available`, `round-robin`, `round-robin-with-fallback`.
- **Read-write splitting**: Routing write connections (port 6446) to the primary and read connections (port 6447) to secondaries.
- **X Protocol routing**: Router can also proxy X Protocol connections (ports 6448/6449) for MySQL Shell and X DevAPI clients.

## Installation

```bash
# Debian/Ubuntu
sudo apt-get install mysql-router

# RHEL/CentOS
sudo yum install mysql-router-community

# Or install from MySQL APT/YUM repository
# (Included with MySQL Server packages)

# Verify installation
mysqlrouter --version
```

## Bootstrap Configuration

### Basic bootstrap

```bash
# Bootstrap against an InnoDB Cluster
# Router connects to the cluster, reads metadata, and generates config
mysqlrouter --bootstrap ic_admin@db1:3306 \
  --directory=/etc/mysqlrouter \
  --account=router_user \
  --account-create=always \
  --user=mysqlrouter \
  --force

# Bootstrap with specific cluster name
mysqlrouter --bootstrap ic_admin@db1:3306 \
  --directory=/opt/mysqlrouter \
  --name=router-app01 \
  --account=router_user \
  --account-create=always

# Bootstrap creates these files:
# /etc/mysqlrouter/mysqlrouter.conf   — main configuration
# /etc/mysqlrouter/data/              — runtime data
# /etc/mysqlrouter/log/               — log files
# /etc/mysqlrouter/run/               — PID file
# /etc/mysqlrouter/start.sh           — startup script
# /etc/mysqlrouter/stop.sh            — shutdown script
```

### Generated configuration file

```ini
# /etc/mysqlrouter/mysqlrouter.conf (auto-generated, can be customized)

[DEFAULT]
logging_folder=/etc/mysqlrouter/log
runtime_folder=/etc/mysqlrouter/run
data_folder=/etc/mysqlrouter/data
keyring_path=/etc/mysqlrouter/data/keyring
master_key_path=/etc/mysqlrouter/mysqlrouter.key
connect_timeout=5
read_timeout=30
dynamic_state=/etc/mysqlrouter/data/state.json

[logger]
level=INFO
filename=mysqlrouter.log
timestamp_precision=second

[metadata_cache:myCluster]
cluster_type=gr
router_id=1
user=router_user
metadata_cluster=myCluster
ttl=0.5                              # Metadata refresh interval (seconds)
auth_cache_ttl=-1
auth_cache_refresh_interval=2
use_gr_notifications=0

[routing:myCluster_rw]
bind_address=0.0.0.0
bind_port=6446
destinations=metadata-cache://myCluster/?role=PRIMARY
routing_strategy=first-available
protocol=classic
connection_sharing=1
client_ssl_mode=PREFERRED
server_ssl_mode=AS_CLIENT
max_connect_errors=100
max_connections=1024

[routing:myCluster_ro]
bind_address=0.0.0.0
bind_port=6447
destinations=metadata-cache://myCluster/?role=SECONDARY
routing_strategy=round-robin-with-fallback
protocol=classic
connection_sharing=1
client_ssl_mode=PREFERRED
server_ssl_mode=AS_CLIENT
max_connections=2048

[routing:myCluster_x_rw]
bind_address=0.0.0.0
bind_port=6448
destinations=metadata-cache://myCluster/?role=PRIMARY
routing_strategy=first-available
protocol=x

[routing:myCluster_x_ro]
bind_address=0.0.0.0
bind_port=6449
destinations=metadata-cache://myCluster/?role=SECONDARY
routing_strategy=round-robin-with-fallback
protocol=x
```

## Routing Modes and Strategies

### Port assignments (convention)

| Port | Protocol | Role | Strategy |
|------|----------|------|----------|
| 6446 | Classic | Read-Write (PRIMARY) | first-available |
| 6447 | Classic | Read-Only (SECONDARY) | round-robin-with-fallback |
| 6448 | X Protocol | Read-Write (PRIMARY) | first-available |
| 6449 | X Protocol | Read-Only (SECONDARY) | round-robin-with-fallback |

### Routing strategies explained

```ini
# first-available: Connect to the first available destination
# Used for RW routing to PRIMARY (only one primary in single-primary mode)
[routing:rw]
routing_strategy=first-available
destinations=metadata-cache://myCluster/?role=PRIMARY

# round-robin: Distribute connections evenly across all destinations
[routing:ro_balanced]
routing_strategy=round-robin
destinations=metadata-cache://myCluster/?role=SECONDARY

# round-robin-with-fallback: Round-robin across secondaries,
# fall back to primary if no secondaries available
[routing:ro_fallback]
routing_strategy=round-robin-with-fallback
destinations=metadata-cache://myCluster/?role=SECONDARY

# next-available: Use the next server if the current one fails
# (no load balancing, failover only)
[routing:ha]
routing_strategy=next-available
destinations=metadata-cache://myCluster/?role=PRIMARY
```

### Static routing (without InnoDB Cluster)

```ini
# Static routing to specific hosts (no metadata cache)
[routing:static_rw]
bind_port=6446
routing_strategy=first-available
destinations=db-primary:3306
protocol=classic

[routing:static_ro]
bind_port=6447
routing_strategy=round-robin
destinations=db-replica1:3306,db-replica2:3306,db-replica3:3306
protocol=classic
```

## Application Connection

### Connecting through Router

```bash
# Read-write connection (goes to PRIMARY)
mysql -h router-host -P 6446 -u app_user -p mydb

# Read-only connection (goes to SECONDARY, round-robin)
mysql -h router-host -P 6447 -u app_user -p mydb
```

### Application connection strings

```python
# Python (PyMySQL)
# Read-write
rw_conn = pymysql.connect(host='router-host', port=6446, user='app', password='pass', db='mydb')
# Read-only
ro_conn = pymysql.connect(host='router-host', port=6447, user='app', password='pass', db='mydb')
```

```java
// Java (JDBC)
// Read-write
String rwUrl = "jdbc:mysql://router-host:6446/mydb?useSSL=true";
// Read-only
String roUrl = "jdbc:mysql://router-host:6447/mydb?useSSL=true";
```

```javascript
// Node.js (mysql2)
// Read-write
const rwPool = mysql.createPool({ host: 'router-host', port: 6446, user: 'app', database: 'mydb' });
// Read-only
const roPool = mysql.createPool({ host: 'router-host', port: 6447, user: 'app', database: 'mydb' });
```

## Monitoring Router Status

### REST API (MySQL Router 8.0.17+)

```ini
# Enable REST API in mysqlrouter.conf
[rest_api]
require_realm=default_auth_realm

[rest_router]
require_realm=default_auth_realm

[rest_routing]
require_realm=default_auth_realm

[rest_metadata_cache]
require_realm=default_auth_realm

[http_server]
port=8443
ssl=1
ssl_cert=/etc/mysqlrouter/data/router-cert.pem
ssl_key=/etc/mysqlrouter/data/router-key.pem

[http_auth_realm:default_auth_realm]
backend=default_auth_backend
method=basic
name=default_realm

[http_auth_backend:default_auth_backend]
backend=metadata_cache
```

### REST API endpoints

```bash
# Router status
curl -k -u router_user:pass https://router-host:8443/api/20190715/router/status

# List all routes
curl -k -u router_user:pass https://router-host:8443/api/20190715/routes

# Specific route status (active connections, destinations)
curl -k -u router_user:pass https://router-host:8443/api/20190715/routes/myCluster_rw/status

# Route health
curl -k -u router_user:pass https://router-host:8443/api/20190715/routes/myCluster_rw/health

# Metadata cache status
curl -k -u router_user:pass https://router-host:8443/api/20190715/metadata/myCluster/status

# Route connections
curl -k -u router_user:pass https://router-host:8443/api/20190715/routes/myCluster_rw/connections
```

### Log monitoring

```bash
# Watch Router logs
tail -f /etc/mysqlrouter/log/mysqlrouter.log

# Key log messages to watch for:
# "metadata changes detected" — topology change
# "Potential topology change detected" — failover in progress
# "Member role changed" — primary/secondary switch
# "Connected to server" — new backend connection
# "Lost connection to server" — backend failure detected
```

## Metadata Cache Configuration

```ini
[metadata_cache:myCluster]
# How often to refresh metadata (seconds)
ttl=0.5

# Use GR notifications for faster failover detection (MySQL 8.0.17+)
# 0 = polling only, 1 = use X Protocol notifications
use_gr_notifications=1

# Authentication cache settings
auth_cache_ttl=-1                     # -1 = never expire
auth_cache_refresh_interval=2         # Refresh interval in seconds

# Connection timeout to metadata server
connect_timeout=5
read_timeout=30
```

### Metadata cache behavior during failover

```
1. Primary goes down
2. Group Replication elects new primary (typically 5-30 seconds)
3. Router detects topology change via:
   a. GR notification (if use_gr_notifications=1) — near-instant
   b. Metadata cache TTL polling (default 0.5s) — up to TTL delay
4. Router updates internal routing table
5. New connections route to new primary
6. Existing connections to old primary get error; application reconnects
   → Reconnect goes through Router → lands on new primary
```

## Starting and Managing Router

```bash
# Start Router
mysqlrouter --config=/etc/mysqlrouter/mysqlrouter.conf &

# Or use the generated scripts
/etc/mysqlrouter/start.sh

# Stop Router
/etc/mysqlrouter/stop.sh
# Or: kill $(cat /etc/mysqlrouter/run/mysqlrouter.pid)

# Systemd service
sudo systemctl start mysqlrouter
sudo systemctl enable mysqlrouter
sudo systemctl status mysqlrouter

# Systemd service file (if not auto-created)
# /etc/systemd/system/mysqlrouter.service
[Unit]
Description=MySQL Router
After=network.target

[Service]
Type=simple
User=mysqlrouter
ExecStart=/usr/bin/mysqlrouter --config=/etc/mysqlrouter/mysqlrouter.conf
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Deployment Patterns

### Co-located Router (recommended for most cases)

```
Application Server 1 ─── Router (localhost:6446/6447) ───┐
Application Server 2 ─── Router (localhost:6446/6447) ───┤── InnoDB Cluster
Application Server 3 ─── Router (localhost:6446/6447) ───┘   (db1, db2, db3)
```

Benefits: No single point of failure for Router. Each app instance has its own Router.

### Centralized Router with load balancer

```
                                    ┌── Router A ──┐
Application ── Load Balancer (VIP) ─┤              ├── InnoDB Cluster
                                    └── Router B ──┘
```

Use when: Applications cannot co-locate Router (e.g., serverless, legacy apps).

### Connection tuning for deployment

```ini
# High-traffic deployment
[routing:myCluster_rw]
max_connections=2048
max_connect_errors=500
connect_timeout=3
client_connect_timeout=9
net_buffer_length=16384
thread_stack_size=65536
connection_sharing=1

[routing:myCluster_ro]
max_connections=4096
max_connect_errors=500
connection_sharing=1
```

## Best Practices

- Deploy Router co-located with application instances (one Router per app server) for zero SPOF.
- Use `round-robin-with-fallback` for read-only routing so reads fall back to primary if all secondaries are down.
- Set `use_gr_notifications=1` for faster failover detection (sub-second vs TTL polling).
- Keep `ttl=0.5` (default) for metadata cache; lower values increase metadata query load on the cluster.
- Set `max_connections` on Router based on expected application connection pool size.
- Enable the REST API for monitoring Router health and integrating with load balancers.
- Use connection sharing (MySQL Router 8.2+) to reduce backend connections.
- Bootstrap Router with `--account-create=always` to ensure the Router metadata user exists.
- Monitor Router logs for "metadata changes detected" messages to track failover events.
- Include Router in your HA testing: kill the primary and verify connections automatically reroute.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running a single centralized Router without redundancy | Router becomes a single point of failure; all apps lose connectivity if Router dies | Deploy co-located Routers on each app server, or use multiple Routers behind a load balancer |
| Not bootstrapping Router after cluster topology changes | Router uses stale metadata; new instances not included in routing | Re-run `mysqlrouter --bootstrap` after adding/removing cluster members |
| Setting `ttl` too low (e.g., 0.1s) | Excessive metadata queries overload the cluster; each Router polls every 100ms | Keep `ttl=0.5` (default); use `use_gr_notifications=1` for faster detection instead |
| Connecting applications directly to MySQL ports (3306) instead of Router ports (6446/6447) | Applications bypass HA routing; no automatic failover | Update all connection strings to use Router ports (6446 for RW, 6447 for RO) |
| Not setting `connection_sharing` on Router 8.2+ | Each application connection holds a dedicated backend connection; wastes MySQL connections | Enable `connection_sharing=1` to multiplex idle client connections |

## MySQL Version Notes

- **5.7**: MySQL Router 2.x available but lacks metadata cache for InnoDB Cluster (Cluster requires 5.7.17+). Static routing only for traditional replication setups.
- **8.0**: Full InnoDB Cluster integration with metadata cache. REST API available from 8.0.17+. GR notifications from 8.0.17+. X Protocol routing supported. Connection sharing available from 8.2+.
- **8.4/9.x**: Enhanced connection sharing and routing performance. Improved metadata cache efficiency. Router 8.4 supports InnoDB ClusterSet routing for multi-datacenter setups. Automatic Router version compatibility checks with cluster metadata.

## Sources

- [MySQL Router Documentation](https://dev.mysql.com/doc/mysql-router/8.0/en/)
- [MySQL Router Configuration](https://dev.mysql.com/doc/mysql-router/8.0/en/mysql-router-configuration.html)
- [Deploying MySQL Router](https://dev.mysql.com/doc/mysql-router/8.0/en/mysql-router-deploying.html)
- [MySQL Router REST API](https://dev.mysql.com/doc/mysql-router/8.0/en/mysql-router-rest-api.html)
- [InnoDB Cluster with Router](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-router.html)
