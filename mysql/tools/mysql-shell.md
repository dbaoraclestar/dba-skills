# MySQL Shell — Interactive Client, Admin API, and Utility Functions

## Overview

MySQL Shell (mysqlsh) is MySQL's advanced client and administration tool, replacing the traditional `mysql` command-line client for modern workflows. It supports three language modes (SQL, JavaScript, Python), provides the Admin API for InnoDB Cluster management, and includes high-performance utility functions for data import/export, parallel loading, and server upgrade checks.

MySQL Shell connects via both the classic MySQL protocol (port 3306) and the X Protocol (port 33060). The X Protocol enables document store operations and is required for some Admin API functions. For DBA operations, MySQL Shell is the primary tool for deploying and managing InnoDB Cluster, InnoDB ClusterSet, and InnoDB ReplicaSet topologies.

This skill covers the three shell modes, Admin API for cluster management, utility functions for data operations, parallel loading, JSON output, and upgrade checking workflows.

## Key Concepts

- **MySQL Shell (mysqlsh)**: The multi-mode interactive client supporting SQL, JavaScript, and Python.
- **X Protocol**: MySQL's modern protocol (port 33060) supporting document operations and Admin API.
- **Admin API**: JavaScript/Python API accessed via the `dba` global object for cluster lifecycle management.
- **InnoDB Cluster**: A high-availability solution combining Group Replication, MySQL Router, and MySQL Shell.
- **Utility functions**: The `util` global object providing data import/export, dump/load, and upgrade check tools.
- **Session**: A connection object in MySQL Shell that provides query execution and schema browsing.

## Shell Modes

### Connecting and switching modes

```bash
# Connect via classic protocol
mysqlsh root@localhost:3306

# Connect via X Protocol
mysqlsh root@localhost:33060

# Connect with specific mode
mysqlsh --sql root@localhost:3306
mysqlsh --js root@localhost:3306
mysqlsh --py root@localhost:3306

# Connect with URI and password prompt
mysqlsh mysql://root@db-primary:3306/mydb --password

# SSL connection
mysqlsh root@db-primary:3306 --ssl-mode=REQUIRED --ssl-ca=/path/to/ca.pem
```

### SQL mode

```sql
-- Switch to SQL mode from JS/Python
\sql

-- Standard SQL queries
SELECT @@version, @@hostname;

SHOW DATABASES;

SELECT table_name, engine, table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;

-- Use system commands
\system ls -la /var/lib/mysql/

-- Source a SQL file
\source /path/to/script.sql
```

### JavaScript mode

```javascript
// Switch to JavaScript mode
\js

// Connect to a server
var session = mysql.getSession('root@localhost:3306');

// Run SQL from JavaScript
var result = session.runSql('SELECT * FROM mydb.users LIMIT 5');
var row;
while (row = result.fetchOne()) {
    print(row.name + ': ' + row.email);
}

// Get schema object
var db = session.getSchema('mydb');
var users = db.getTable('users');

// Table operations via X DevAPI
var result = users.select(['name', 'email'])
    .where('status = :status')
    .bind('status', 'active')
    .limit(10)
    .execute();

print(result.fetchAll());
```

### Python mode

```python
# Switch to Python mode
\py

# Run SQL from Python
result = session.run_sql('SHOW PROCESSLIST')
columns = result.get_columns()
for col in columns:
    print(f"{col.get_column_label():>20}", end="")
print()
for row in result.fetch_all():
    for i in range(len(columns)):
        print(f"{str(row[i]):>20}", end="")
    print()

# Schema browsing
db = session.get_schema('mydb')
tables = db.get_tables()
for t in tables:
    print(t.get_name())
```

## Admin API — InnoDB Cluster Management

### Configure instance for Group Replication

```javascript
\js

// Check instance readiness
dba.checkInstanceConfiguration('root@db1:3306');

// Configure instance (sets required GR variables)
dba.configureInstance('root@db1:3306', {
    clusterAdmin: 'ic_admin',
    clusterAdminPassword: 'StrongPass123!',
    interactive: false,
});

// Configure all instances in the cluster
dba.configureInstance('root@db2:3306', { clusterAdmin: 'ic_admin', clusterAdminPassword: 'StrongPass123!' });
dba.configureInstance('root@db3:3306', { clusterAdmin: 'ic_admin', clusterAdminPassword: 'StrongPass123!' });
```

### Create and manage InnoDB Cluster

```javascript
// Connect to the primary instance
shell.connect('ic_admin@db1:3306');

// Create cluster
var cluster = dba.createCluster('myCluster', {
    multiPrimary: false,        // Single-primary mode (recommended)
    memberWeight: 50,
    exitStateAction: 'READ_ONLY',
    autoRejoinTries: 3,
    expelTimeout: 5,
});

// Add secondary instances
cluster.addInstance('ic_admin@db2:3306', { recoveryMethod: 'clone' });
cluster.addInstance('ic_admin@db3:3306', { recoveryMethod: 'clone' });

// Check cluster status
cluster.status();
cluster.status({ extended: 1 });  // Detailed status

// Describe cluster topology
cluster.describe();

// Get cluster handle (reconnect scenario)
var cluster = dba.getCluster();
```

### Cluster operations

```javascript
// Switch primary
cluster.setPrimaryInstance('ic_admin@db2:3306');

// Remove an instance
cluster.removeInstance('ic_admin@db3:3306');

// Rejoin an instance after it went offline
cluster.rejoinInstance('ic_admin@db3:3306');

// Dissolve cluster (careful!)
cluster.dissolve({ force: false });

// Set cluster-wide options
cluster.setOption('exitStateAction', 'ABORT_SERVER');
cluster.setOption('autoRejoinTries', 5);

// Set instance-specific options
cluster.setInstanceOption('ic_admin@db2:3306', 'memberWeight', 30);

// Rescan cluster (detect topology changes)
cluster.rescan();

// Force quorum when majority is lost
cluster.forceQuorumUsingPartitionOf('ic_admin@db1:3306');

// Reboot cluster from complete outage
var cluster = dba.rebootClusterFromCompleteOutage('myCluster');
```

### InnoDB ReplicaSet (async replication)

```javascript
// Create ReplicaSet (async replication, simpler than Cluster)
shell.connect('admin@primary:3306');
var rs = dba.createReplicaSet('myReplicaSet');

// Add replica
rs.addInstance('admin@replica1:3306', { recoveryMethod: 'clone' });

// Check status
rs.status({ extended: 1 });

// Failover
rs.setPrimaryInstance('admin@replica1:3306');

// Force failover when primary is unreachable
rs.forcePrimaryInstance('admin@replica1:3306');
```

## Utility Functions

### Dump and load (parallel, compressed)

```javascript
\js

// Dump entire instance (all databases)
util.dumpInstance('/tmp/full_dump', {
    threads: 8,
    compression: 'zstd',
    consistent: true,
    excludeSchemas: ['mysql', 'information_schema', 'performance_schema', 'sys'],
});

// Dump specific schemas
util.dumpSchemas(['mydb', 'analytics'], '/tmp/schema_dump', {
    threads: 8,
    compression: 'zstd',
});

// Dump specific tables
util.dumpTables('mydb', ['users', 'orders'], '/tmp/table_dump', {
    threads: 4,
    where: { 'orders': "created_at >= '2024-01-01'" },
});

// Load dump (parallel)
util.loadDump('/tmp/full_dump', {
    threads: 16,
    progressFile: '/tmp/load_progress.json',
    resetProgress: false,        // Resume from last checkpoint
    deferTableIndexes: 'all',    // Create indexes after data load
    loadUsers: true,
    ignoreExistingObjects: false,
    showProgress: true,
});
```

### Export and import tables

```javascript
// Export table to CSV/TSV
util.exportTable('mydb.users', '/tmp/users.csv', {
    dialect: 'csv',
    fieldsTerminatedBy: ',',
    fieldsEnclosedBy: '"',
    linesTerminatedBy: '\n',
    threads: 4,
});

// Export to JSON
util.exportTable('mydb.users', '/tmp/users.json', {
    dialect: 'json',
});

// Import table from CSV (parallel)
util.importTable('/tmp/users.csv', {
    schema: 'mydb',
    table: 'users',
    dialect: 'csv',
    threads: 8,
    bytesPerChunk: '50M',
    columns: ['name', 'email', 'created_at'],
    decodeColumns: { 'created_at': '@1' },
    showProgress: true,
});

// Import from multiple files
util.importTable(['/tmp/data_part1.csv', '/tmp/data_part2.csv'], {
    schema: 'mydb',
    table: 'events',
    threads: 8,
});
```

### Server upgrade check

```javascript
// Check MySQL server readiness for upgrade (e.g., 5.7 -> 8.0)
util.checkForServerUpgrade('root@localhost:3306', {
    targetVersion: '8.0.36',
    outputFormat: 'TEXT',         // or 'JSON'
});

// Save upgrade check report
util.checkForServerUpgrade('root@localhost:3306', {
    targetVersion: '8.4.0',
    outputFormat: 'JSON',
});
```

## JSON Output Mode

```bash
# Start shell with JSON output
mysqlsh --json=pretty root@localhost:3306

# Or set during session
\option outputFormat=json/pretty
```

```javascript
// JSON output is useful for scripting
mysqlsh --json root@localhost:3306 -e "SELECT @@version AS version"
// Output: {"version": "8.0.36"}

// Tabbed output for human consumption
\option outputFormat=tabbed
```

## Scripting and Automation

```bash
# Execute SQL file
mysqlsh root@localhost:3306 --sql -f /path/to/script.sql

# Execute JavaScript file
mysqlsh root@localhost:3306 --js -f /path/to/admin.js

# Execute inline command
mysqlsh root@localhost:3306 --sql -e "SELECT COUNT(*) FROM mydb.users"

# Execute with JSON output for parsing
mysqlsh root@localhost:3306 --json --sql -e "SHOW STATUS LIKE 'Threads%'" | jq '.'

# Non-interactive cluster status check
mysqlsh ic_admin@db1:3306 --js -e "print(dba.getCluster().status())"
```

## Best Practices

- Use MySQL Shell instead of the legacy `mysql` client for all administrative operations.
- Always use `recoveryMethod: 'clone'` when adding instances to a cluster with existing data.
- Use `util.dumpInstance` / `util.loadDump` instead of `mysqldump` for large databases (10-100x faster with parallel threads).
- Set `deferTableIndexes: 'all'` during `loadDump` to speed up data loading by building indexes after all data is loaded.
- Use `util.checkForServerUpgrade` before any major version upgrade to identify incompatibilities.
- Use `compression: 'zstd'` for dumps to reduce disk space and network transfer time.
- Save `progressFile` during `loadDump` so interrupted loads can resume from the last checkpoint.
- Use X Protocol (port 33060) for Admin API operations; classic protocol for SQL-only workflows.
- Test cluster failover scenarios in staging before relying on them in production.
- Use `cluster.status({ extended: 1 })` for detailed diagnostics including transaction gaps and apply queue size.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `recoveryMethod: 'incremental'` on a new instance with no data | Recovery fails because binary logs may be purged | Use `recoveryMethod: 'clone'` for instances without existing data |
| Not running `dba.configureInstance` before `addInstance` | Instance join fails due to missing Group Replication configuration | Always run `dba.configureInstance` on every instance before cluster operations |
| Using `mysqldump` instead of `util.dumpInstance` for large databases | Dump takes hours instead of minutes; no parallelism, no compression | Switch to `util.dumpInstance` with `threads: 8` and `compression: 'zstd'` |
| Forgetting `--password` flag in automated scripts | Connection fails with authentication error; password prompt hangs in non-interactive mode | Use `--password=<pass>` or credential store; avoid storing passwords in scripts |
| Running `cluster.dissolve()` on a production cluster accidentally | All instances leave the cluster; application loses HA | Use `force: false` (default) and always confirm interactively; restrict access to Admin API credentials |

## MySQL Version Notes

- **5.7**: MySQL Shell available as separate download. Limited Admin API support (no InnoDB Cluster in early 5.7). `util.checkForServerUpgrade` is the primary use case for 5.7-to-8.0 upgrades.
- **8.0**: Full Admin API for InnoDB Cluster, ReplicaSet, and ClusterSet. `util.dumpInstance` and `util.loadDump` available from 8.0.21+. X Protocol enabled by default. Clone plugin for fast provisioning.
- **8.4/9.x**: Enhanced `util.dumpInstance` with parallel DDL. Improved Admin API with read replica support in InnoDB Cluster. `util.checkForServerUpgrade` updated for 9.x compatibility checks. MySQL Shell 8.4 includes InnoDB ClusterSet improvements for disaster recovery.

## Sources

- [MySQL Shell User Guide](https://dev.mysql.com/doc/mysql-shell/8.0/en/)
- [MySQL Admin API Reference](https://dev.mysql.com/doc/dev/mysqlsh-api-javascript/8.0/group__AdminAPI.html)
- [MySQL Shell Utilities](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities.html)
- [InnoDB Cluster Guide](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster.html)
- [MySQL Shell Dump and Load](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-dump-instance-schema.html)
