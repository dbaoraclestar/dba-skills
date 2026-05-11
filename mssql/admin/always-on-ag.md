# SQL Server Always On Availability Groups — High Availability and Disaster Recovery

## Overview

Always On Availability Groups (AG) is SQL Server's flagship high availability and disaster recovery solution, introduced in SQL Server 2012. An AG consists of a group of databases that fail over together as a unit, with one primary replica handling read-write workloads and up to eight secondary replicas providing redundancy, read scaling, and offsite disaster recovery.

AGs replace the older database mirroring technology and offer significant advantages: multiple secondaries, readable replicas, automatic page repair, flexible failover policies, and integration with Windows Server Failover Clustering (WSFC) or Linux Pacemaker. Understanding AG architecture is essential for designing resilient SQL Server environments.

This skill covers AG creation and configuration, synchronous vs asynchronous commit modes, failover types, read-only routing, distributed availability groups, monitoring, and troubleshooting common issues like data movement suspension and redo queue growth.

## Key Concepts

- **Availability Group**: A logical container for one or more user databases that fail over together. Each AG has one primary replica and up to eight secondary replicas.
- **Primary Replica**: The instance hosting the read-write copy of the databases. All write operations go here.
- **Secondary Replica**: An instance hosting a synchronized (or synchronizing) copy of the databases. Can be configured for read-only access.
- **Availability Group Listener**: A virtual network name (VNN) and IP address that clients use to connect. The listener automatically redirects to the current primary after failover.
- **Database Mirroring Endpoint**: The TCP endpoint used for inter-replica communication. Each instance needs exactly one, typically on port 5022.
- **Synchronous Commit**: The primary waits for the secondary to harden the log before acknowledging the commit to the client. Guarantees zero data loss but adds latency.
- **Asynchronous Commit**: The primary commits without waiting for the secondary. Provides better performance but risks data loss during failover.
- **Automatic Failover**: AG fails over automatically when the primary becomes unavailable. Requires synchronous commit and a WSFC quorum.
- **Manual Failover**: DBA-initiated planned failover, typically for maintenance. No data loss with synchronous commit.
- **Forced Failover (with data loss)**: Emergency failover to an asynchronous secondary when the primary is unavailable. Accepts potential data loss.
- **Seeding Mode**: How the initial database copy is created on secondaries. Automatic seeding (SQL Server handles it) or manual (DBA restores backups).

## AG Architecture and Setup

### Prerequisites

```sql
-- 1. Enable Always On at instance level (requires restart)
-- Via SQL Server Configuration Manager or PowerShell:
```

```powershell
Enable-SqlAlwaysOn -ServerInstance "SQLNODE01" -Force
Enable-SqlAlwaysOn -ServerInstance "SQLNODE02" -Force
# Restart SQL Server service after enabling
Restart-Service -Name "MSSQLSERVER" -Force
```

```sql
-- 2. Create database mirroring endpoint on each replica
-- On SQLNODE01:
CREATE ENDPOINT [Hadr_endpoint]
    STATE = STARTED
    AS TCP (LISTENER_PORT = 5022)
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = WINDOWS NEGOTIATE,
        ENCRYPTION = REQUIRED ALGORITHM AES
    );

-- Grant CONNECT to the service account
GRANT CONNECT ON ENDPOINT::Hadr_endpoint TO [DOMAIN\SQLServiceAccount];

-- 3. Ensure databases are in FULL recovery model
ALTER DATABASE [ApplicationDB] SET RECOVERY FULL;
ALTER DATABASE [OrdersDB] SET RECOVERY FULL;

-- Take full backups (required before adding to AG)
BACKUP DATABASE [ApplicationDB] TO DISK = N'\\Share\Backups\ApplicationDB.bak' WITH COMPRESSION;
BACKUP DATABASE [OrdersDB] TO DISK = N'\\Share\Backups\OrdersDB.bak' WITH COMPRESSION;
```

### Create Availability Group

```sql
-- Create AG with two synchronous replicas and one async (DR site)
CREATE AVAILABILITY GROUP [AG_Production]
WITH (
    AUTOMATED_BACKUP_PREFERENCE = SECONDARY,
    DB_FAILOVER = ON,           -- SQL 2016+: enhanced failover detection
    DTC_SUPPORT = NONE,
    CLUSTER_TYPE = WSFC          -- SQL 2017+: WSFC, EXTERNAL, or NONE
)
FOR DATABASE [ApplicationDB], [OrdersDB]
REPLICA ON
    N'SQLNODE01' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE01.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (
            ALLOW_CONNECTIONS = READ_ONLY,
            READ_ONLY_ROUTING_URL = N'TCP://SQLNODE01.domain.com:1433'
        ),
        PRIMARY_ROLE (
            ALLOW_CONNECTIONS = READ_WRITE,
            READ_ONLY_ROUTING_LIST = (N'SQLNODE02', N'SQLNODE03')
        )
    ),
    N'SQLNODE02' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE02.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (
            ALLOW_CONNECTIONS = READ_ONLY,
            READ_ONLY_ROUTING_URL = N'TCP://SQLNODE02.domain.com:1433'
        ),
        PRIMARY_ROLE (
            ALLOW_CONNECTIONS = READ_WRITE,
            READ_ONLY_ROUTING_LIST = (N'SQLNODE01', N'SQLNODE03')
        )
    ),
    N'SQLNODE03' WITH (
        ENDPOINT_URL = N'TCP://SQLNODE03-DR.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SECONDARY_ROLE (ALLOW_CONNECTIONS = READ_ONLY)
    );

-- Create the listener
ALTER AVAILABILITY GROUP [AG_Production]
ADD LISTENER N'AG-PROD-LISTENER' (
    WITH IP (
        (N'10.0.1.100', N'255.255.255.0'),   -- Subnet 1
        (N'10.0.2.100', N'255.255.255.0')    -- Subnet 2 (multi-subnet)
    ),
    PORT = 1433
);
```

### Join Secondaries

```sql
-- On each secondary replica:
-- Join to the availability group
ALTER AVAILABILITY GROUP [AG_Production] JOIN;

-- Grant automatic seeding permission
ALTER AVAILABILITY GROUP [AG_Production]
    GRANT CREATE ANY DATABASE;
-- SQL Server will automatically seed the databases

-- If using manual seeding instead:
-- 1. Restore full + log backup on secondary WITH NORECOVERY
-- 2. Then join the database to the AG:
ALTER DATABASE [ApplicationDB] SET HADR AVAILABILITY GROUP = [AG_Production];
```

## Failover Operations

### Planned Manual Failover (No Data Loss)

```sql
-- Run on the target secondary (will become new primary)
-- Requires synchronous commit mode
ALTER AVAILABILITY GROUP [AG_Production] FAILOVER;

-- Verify the new primary
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.operational_state_desc,
    ars.synchronization_health_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id;
```

### Forced Failover (Potential Data Loss)

```sql
-- Emergency failover to async secondary when primary is unavailable
-- Run on the target secondary:
ALTER AVAILABILITY GROUP [AG_Production] FORCE_FAILOVER_ALLOW_DATA_LOSS;

-- After primary comes back online, it joins as secondary
-- You may need to resume data movement:
ALTER DATABASE [ApplicationDB] SET HADR RESUME;
```

### Failover via PowerShell

```powershell
# Planned failover
Switch-SqlAvailabilityGroup -Path "SQLSERVER:\SQL\SQLNODE02\DEFAULT\AvailabilityGroups\AG_Production"

# Forced failover
Switch-SqlAvailabilityGroup -Path "SQLSERVER:\SQL\SQLNODE03\DEFAULT\AvailabilityGroups\AG_Production" `
    -AllowDataLoss -Force
```

## Read-Only Routing

```sql
-- Configure read-only routing URLs and lists
-- On each replica, set the read-only routing URL
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQLNODE01' WITH (
    SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQLNODE01.domain.com:1433')
);

ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQLNODE02' WITH (
    SECONDARY_ROLE (READ_ONLY_ROUTING_URL = N'TCP://SQLNODE02.domain.com:1433')
);

-- Set the routing list on the primary role (ordered preference)
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQLNODE01' WITH (
    PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = (N'SQLNODE02', N'SQLNODE03'))
);

ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQLNODE02' WITH (
    PRIMARY_ROLE (READ_ONLY_ROUTING_LIST = (N'SQLNODE01', N'SQLNODE03'))
);

-- Read-only routing load balancing (SQL 2016+, uses nested parentheses)
ALTER AVAILABILITY GROUP [AG_Production]
MODIFY REPLICA ON N'SQLNODE01' WITH (
    PRIMARY_ROLE (
        READ_ONLY_ROUTING_LIST = ((N'SQLNODE02', N'SQLNODE03'))
    )
);

-- Client connection string for read-only routing:
-- Server=AG-PROD-LISTENER;Database=ApplicationDB;
--   ApplicationIntent=ReadOnly;MultiSubnetFailover=True
```

## Distributed Availability Groups

```sql
-- Distributed AG: spans two separate WSFC clusters (cross-datacenter DR)
-- On the primary AG cluster:
CREATE AVAILABILITY GROUP [DistAG_Global]
WITH (DISTRIBUTED)
AVAILABILITY GROUP ON
    N'AG_Production' WITH (
        LISTENER_URL = N'TCP://AG-PROD-LISTENER.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    ),
    N'AG_DR' WITH (
        LISTENER_URL = N'TCP://AG-DR-LISTENER.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    );

-- On the DR cluster, join the distributed AG:
ALTER AVAILABILITY GROUP [DistAG_Global] JOIN
AVAILABILITY GROUP ON
    N'AG_Production' WITH (
        LISTENER_URL = N'TCP://AG-PROD-LISTENER.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    ),
    N'AG_DR' WITH (
        LISTENER_URL = N'TCP://AG-DR-LISTENER.domain.com:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    );
```

## Basic Availability Groups (Standard Edition)

```sql
-- SQL Server 2016 SP1+: Basic AG for Standard Edition
-- Limitations: 2 replicas only, 1 database per AG, no read-only secondary
CREATE AVAILABILITY GROUP [AG_BasicApp]
WITH (
    BASIC,
    DB_FAILOVER = ON
)
FOR DATABASE [SingleAppDB]
REPLICA ON
    N'SQLSTD01' WITH (
        ENDPOINT_URL = N'TCP://SQLSTD01.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC
    ),
    N'SQLSTD02' WITH (
        ENDPOINT_URL = N'TCP://SQLSTD02.domain.com:5022',
        AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
        FAILOVER_MODE = AUTOMATIC,
        SEEDING_MODE = AUTOMATIC
    );
```

## Monitoring AG Health

### DMV Queries

```sql
-- Overall AG health status
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    ars.role_desc,
    ars.operational_state_desc,
    ars.connected_state_desc,
    ars.synchronization_health_desc,
    ars.last_connect_error_description,
    ars.last_connect_error_timestamp
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id
ORDER BY ag.name, ar.replica_server_name;

-- Database-level synchronization status
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    db.name AS DatabaseName,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.is_suspended,
    drs.suspend_reason_desc,
    drs.log_send_queue_size AS LogSendQueueKB,
    drs.log_send_rate AS LogSendRateKBps,
    drs.redo_queue_size AS RedoQueueKB,
    drs.redo_rate AS RedoRateKBps,
    drs.last_hardened_lsn,
    drs.last_redone_time,
    DATEDIFF(SECOND, drs.last_redone_time, GETDATE()) AS SecondsSinceLastRedo
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id
JOIN sys.databases db ON drs.database_id = db.database_id
ORDER BY ag.name, ar.replica_server_name, db.name;

-- Estimated data loss and recovery time
SELECT
    ag.name AS AGName,
    ar.replica_server_name,
    db.name AS DatabaseName,
    drs.last_hardened_lsn,
    drs.last_commit_lsn,
    drs.last_commit_time,
    DATEDIFF(SECOND, drs.last_commit_time, GETDATE()) AS EstimatedDataLossSec,
    CASE
        WHEN drs.redo_rate > 0
        THEN drs.redo_queue_size / drs.redo_rate
        ELSE NULL
    END AS EstimatedRecoveryTimeSec
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id
JOIN sys.databases db ON drs.database_id = db.database_id
WHERE drs.is_local = 0  -- Remote replicas only
ORDER BY ag.name, ar.replica_server_name;

-- Listener information
SELECT
    ag.name AS AGName,
    agl.dns_name AS ListenerName,
    agl.port AS ListenerPort,
    aglip.ip_address,
    aglip.ip_subnet_mask,
    aglip.state_desc AS IPState
FROM sys.availability_group_listeners agl
JOIN sys.availability_groups ag ON agl.group_id = ag.group_id
JOIN sys.availability_group_listener_ip_addresses aglip
    ON agl.listener_id = aglip.listener_id;
```

### PowerShell Monitoring

```powershell
# Quick AG health check
Test-SqlAvailabilityGroup -Path "SQLSERVER:\SQL\SQLNODE01\DEFAULT\AvailabilityGroups\AG_Production"

# Detailed replica status
Get-SqlAvailabilityReplica -Path "SQLSERVER:\SQL\SQLNODE01\DEFAULT\AvailabilityGroups\AG_Production" |
    Select-Object Name, Role, AvailabilityMode, FailoverMode,
                  ConnectionState, RollupSynchronizationState |
    Format-Table -AutoSize

# Database synchronization status
Get-SqlAvailabilityDatabase -Path "SQLSERVER:\SQL\SQLNODE01\DEFAULT\AvailabilityGroups\AG_Production" |
    Select-Object Name, SynchronizationState, IsJoined, IsSuspended |
    Format-Table -AutoSize
```

## Troubleshooting Common Issues

### Data Movement Suspended

```sql
-- Find suspended databases and the reason
SELECT
    db.name AS DatabaseName,
    ar.replica_server_name,
    drs.synchronization_state_desc,
    drs.is_suspended,
    drs.suspend_reason_desc,
    drs.last_sent_time,
    drs.last_hardened_time,
    drs.last_redone_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.databases db ON drs.database_id = db.database_id
WHERE drs.is_suspended = 1;

-- Resume data movement
ALTER DATABASE [ApplicationDB] SET HADR RESUME;

-- If resuming fails, check the SQL Server error log
EXEC xp_readerrorlog 0, 1, N'availability';
```

### Redo Queue Growing

```sql
-- Monitor redo queue and rate over time
SELECT
    ar.replica_server_name,
    db.name AS DatabaseName,
    drs.redo_queue_size AS RedoQueueKB,
    drs.redo_rate AS RedoRateKBps,
    CASE
        WHEN drs.redo_rate > 0
        THEN CAST(drs.redo_queue_size / drs.redo_rate AS DECIMAL(10,1))
        ELSE -1
    END AS EstCatchupTimeSec,
    drs.last_redone_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
JOIN sys.databases db ON drs.database_id = db.database_id
WHERE drs.is_local = 1
  AND drs.is_primary_replica = 0
ORDER BY drs.redo_queue_size DESC;

-- Common causes of redo queue buildup:
-- 1. Long-running queries on readable secondary blocking redo
-- 2. Insufficient CPU/IO on secondary
-- 3. Heavy write workload on primary
-- 4. Large transactions (index rebuilds, bulk loads)
```

### Endpoint Connectivity Issues

```sql
-- Check endpoint status
SELECT name, state_desc, port, role_desc, connection_auth_desc
FROM sys.database_mirroring_endpoints;

-- Test endpoint connectivity from one replica to another
-- (Run on the initiating replica)
SELECT * FROM sys.dm_hadr_availability_replica_states
WHERE connected_state_desc = 'DISCONNECTED';

-- Check for blocked ports or firewall issues
-- PowerShell test from one node to another:
```

```powershell
Test-NetConnection -ComputerName "SQLNODE02" -Port 5022
```

## WSFC Quorum Modes and Voting

Understanding quorum is essential for AG and FCI reliability. The quorum determines whether the cluster has enough votes to remain online.

### Quorum Modes

| Mode | Description | Use When |
|------|-------------|----------|
| Node Majority | More than half of voting nodes must be online | Odd number of nodes |
| Node and File Share Majority | Nodes + file share witness vote | Even number of nodes |
| Node and Disk Majority | Nodes + shared disk witness vote | FCIs with shared storage |
| Cloud Witness (2016+) | Azure blob storage as witness | Cross-datacenter or hybrid |

### Quorum DMVs

```sql
-- Check cluster quorum state
SELECT
    cluster_name,
    quorum_type_desc,
    quorum_state_desc
FROM sys.dm_hadr_cluster;

-- Check node voting weights
SELECT
    member_name,
    member_type_desc,
    member_state_desc,
    number_of_quorum_votes
FROM sys.dm_hadr_cluster_members;
```

### Voting Best Practices

- Include all nodes hosting primary replicas and automatic failover targets
- Exclude DR-site nodes from voting to prevent split-brain
- Maintain an odd number of total votes (add witness if needed)
- After forced quorum, reassess NodeWeight before bringing other nodes online

### Forced Quorum Start (Disaster Recovery)

When the cluster loses quorum due to a multi-node failure, force-start from the surviving node:

```powershell
# PowerShell: force cluster start without quorum
Import-Module FailoverClusters
$node = "SQLNODE01"
Stop-ClusterNode -Name $node
Start-ClusterNode -Name $node -FixQuorum

# Set surviving node as voting member
(Get-ClusterNode $node).NodeWeight = 1

# Verify cluster state
Get-ClusterNode -Cluster $node | Format-Table NodeName, State, NodeWeight
```

```cmd
-- CMD alternative
net.exe stop clussvc
net.exe start clussvc /forcequorum
```

After forced quorum, immediately perform a forced AG failover if the primary was on a failed node:

```sql
ALTER AVAILABILITY GROUP [AG_Production] FORCE_FAILOVER_ALLOW_DATA_LOSS;
```

## Login Synchronization Across Replicas

AGs replicate databases but **not logins**. After failover, applications fail if logins are missing or have mismatched SIDs on the new primary.

### SID-Matched Login Script

```sql
-- Generate CREATE LOGIN scripts with matching SID and password hash
SELECT
    N'CREATE LOGIN [' + sp.name + N'] WITH PASSWORD = 0x' +
    CONVERT(NVARCHAR(MAX), l.password_hash, 2) + N' HASHED, ' +
    N'SID = 0x' + CONVERT(NVARCHAR(MAX), sp.sid, 2) +
    N', DEFAULT_DATABASE = [' + sp.default_database_name + N']' +
    CASE WHEN sp.is_disabled = 1 THEN N'; ALTER LOGIN [' + sp.name + N'] DISABLE;' ELSE N'' END
FROM sys.server_principals AS sp
JOIN sys.sql_logins AS l ON sp.sid = l.sid
WHERE sp.type = 'S'
    AND sp.name NOT LIKE '##%'
    AND sp.name NOT IN ('sa')
ORDER BY sp.name;
```

### DBATools Automation (Recommended)

```powershell
# Sync logins from primary to all secondary replicas
$primaryReplica = Get-DbaAgReplica -SqlInstance $AGListener | Where Role -eq Primary
$secondaryReplicas = Get-DbaAgReplica -SqlInstance $AGListener | Where Role -eq Secondary
$loginsOnPrimary = Get-DbaLogin -SqlInstance $primaryReplica.Name

$secondaryReplicas | ForEach-Object {
    $loginsOnSecondary = Get-DbaLogin -SqlInstance $_.Name
    $diff = $loginsOnPrimary | Where-Object Name -notin ($loginsOnSecondary.Name)
    if ($diff) {
        Copy-DbaLogin -Source $primaryReplica.Name -Destination $_.Name -Login $diff.Name
    }
}
```

### Orphaned User Detection and Remediation

```sql
-- Find orphaned database users (user exists in DB but no matching server login)
SELECT dp.name AS OrphanedUser, dp.sid
FROM sys.database_principals dp
WHERE dp.type IN ('G', 'S', 'U')
    AND dp.sid NOT IN (SELECT sid FROM sys.server_principals)
    AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys');

-- Remap orphaned user to existing login
ALTER USER [AppUser] WITH LOGIN = [AppUser];

-- Auto-fix (login and user names must match)
EXEC sp_change_users_login 'Auto_Fix', 'AppUser';
```

## MSDTC Configuration for AGs

Distributed transactions across AG databases require MSDTC. Configure a clustered DTC resource per AG for SQL Server 2016+.

```sql
-- Enable DTC support when creating the AG
CREATE AVAILABILITY GROUP [AG_Production]
WITH (DTC_SUPPORT = PER_DB)
FOR DATABASE [AppDB]
REPLICA ON ...;

-- Or enable on existing AG
ALTER AVAILABILITY GROUP [AG_Production] SET (DTC_SUPPORT = PER_DB);
```

Best practice: create a dedicated clustered DTC instance for each AG rather than relying on the default local DTC.

## AG vs Failover Cluster Instance (FCI)

| Feature | Availability Group | Failover Cluster Instance |
|---------|-------------------|--------------------------|
| Failover scope | Per-database group | Entire SQL Server instance |
| Readable secondaries | Yes | No |
| Shared storage required | No | Yes |
| Multiple databases | Select databases | All databases fail over |
| Standard Edition | Basic AG (2 replicas, 1 DB) | Yes |
| Instance-level objects | Not replicated (except Contained AG 2022) | Shared via same instance |

FCIs and AGs can be combined: AG replicas can be hosted on FCI instances for both instance-level and database-level HA.

## Best Practices

- Use synchronous commit mode only between replicas in the same datacenter or low-latency network to avoid commit latency impact.
- Always create a listener for application connectivity. Never hard-code replica names in connection strings.
- Configure at least one synchronous replica for automatic failover to achieve true HA without DBA intervention.
- Use asynchronous commit for cross-datacenter DR replicas where network latency exceeds 5-10ms.
- Test failover regularly (monthly or quarterly) during maintenance windows to verify the process works and the team is prepared.
- Monitor redo queue size and log send queue on all secondaries continuously. Alert when queue exceeds acceptable thresholds.
- Use automatic seeding for simplicity, but be aware it generates significant network traffic. For very large databases, manual seeding with backup/restore may be faster.
- Consider using contained database users or syncing logins with matching SIDs across all replicas to prevent authentication failures after failover.
- Set AUTOMATED_BACKUP_PREFERENCE to SECONDARY to offload backup I/O from the primary replica.
- For read-only routing, ensure the ApplicationIntent=ReadOnly connection string property is set in client applications.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Synchronous commit across high-latency WAN | Write latency spikes; application timeouts | Use asynchronous commit for cross-datacenter replicas |
| Not syncing logins across replicas | Authentication failures after failover; application downtime | Script logins with SIDs to all replicas or use contained database users |
| Forgetting to create or configure the listener | Applications connect directly to replica; manual reconnection after failover | Always create and use an AG listener for client connections |
| Running index rebuilds on primary without monitoring secondary lag | Redo queue balloons; readable secondary falls behind; potential failover issues | Monitor redo queue during maintenance; consider REORGANIZE over REBUILD |
| Using automatic failover with asynchronous commit | Not supported; AG will not auto-failover | Automatic failover requires synchronous commit |
| Not testing failover procedures | Unknown issues discovered during real emergencies | Schedule quarterly failover drills; document the process |
| Configuring too many synchronous replicas | Cumulative latency degrades write performance | Limit synchronous replicas to 1-2; use async for the rest |
| Ignoring AG dashboard warnings in SSMS | Small issues escalate into data movement suspension or failover failures | Monitor AG health proactively using DMVs and alerts |

## SQL Server Version Notes

- **SQL Server 2012**: Availability Groups introduced. Up to 4 secondary replicas. Manual seeding only.
- **SQL Server 2014**: Up to 8 secondary replicas. Readable secondary improvements. Enhanced AG dashboard in SSMS.
- **SQL Server 2016**: Automatic seeding, Basic AG for Standard Edition, distributed availability groups, per-database failover detection (DB_FAILOVER), read-only routing load balancing, enhanced DTC support.
- **SQL Server 2017**: Cluster type EXTERNAL (Linux Pacemaker), NONE (clusterless for read-scale); SQL Server on Linux support.
- **SQL Server 2019**: Readable secondary connection redirection (all connections routed to primary automatically if they are read-write). Up to 5 synchronous replicas. Enhanced parallel redo on secondary replicas for faster catch-up.
- **SQL Server 2022**: Contained availability groups (AG contains system databases like msdb, logins, and agent jobs inside the AG for complete portability). Improved automatic seeding with compression. Cross-platform AG (Windows to Linux).

## Sources

- [Always On Availability Groups Overview](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server)
- [Create and Configure an Availability Group](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/creation-and-configuration-of-availability-groups-sql-server)
- [Failover and Failover Modes](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/failover-and-failover-modes-always-on-availability-groups)
- [Read-Only Routing](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/configure-read-only-routing-for-an-availability-group-sql-server)
- [Distributed Availability Groups](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/distributed-availability-groups)
- [Monitor Availability Groups (Transact-SQL)](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/monitor-availability-groups-transact-sql)
- [Troubleshoot Always On Availability Groups Configuration](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/troubleshoot-always-on-availability-groups-configuration-sql-server)
- [Contained Availability Groups (SQL Server 2022)](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/contained-availability-groups-overview)
- [WSFC Quorum Modes and Voting Configuration](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/windows/wsfc-quorum-modes-and-voting-configuration-sql-server)
- [Force a WSFC Cluster to Start Without a Quorum](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/windows/force-a-wsfc-cluster-to-start-without-a-quorum)
- [WSFC Disaster Recovery Through Forced Quorum](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/windows/wsfc-disaster-recovery-through-forced-quorum-sql-server)
- [Synchronize Logins Between AG Replicas](https://www.sqlshack.com/synchronize-logins-between-availability-replicas-in-sql-server-always-on-availability-group/)
- [Orphaned Database Users](https://www.sqlshack.com/how-to-discover-and-handle-orphaned-database-users-in-sql-server/)
- [Configure Distributed Transactions for AG](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/configure-availability-group-for-distributed-transactions)
- [MSDTC Best Practices with AG](https://techcommunity.microsoft.com/t5/core-infrastructure-and-security/msdtc-best-practices-with-an-availability-group/ba-p/909429)
- [AG vs FCI Comparison](https://www.starwindsoftware.com/blog/always-on-availability-groups-vs-failover-cluster/)
- [Monitor AG with Extended Events](https://www.sqlshack.com/monitor-sql-server-always-on-availability-groups-using-extended-events/)
- [AG Backups on Secondary Replicas](https://www.sqlshack.com/sql-server-always-on-availability-group-log-backup-on-secondary-replicas/)
- [Data Synchronization in AGs](https://www.sqlshack.com/data-synchronization-in-sql-server-always-on-availability-groups/)
- [Upgrade AG Replicas](https://learn.microsoft.com/en-us/sql/database-engine/availability-groups/windows/upgrading-always-on-availability-group-replica-instances)
- [Multi-Subnet AG Configuration](https://www.mssqltips.com/sqlservertip/4597/configure-sql-server-alwayson-availability-group-on-a-multisubnet-cluster/)
- [Configure Read-Only Routing for AG](https://www.sqlshack.com/how-to-configure-read-only-routing-for-an-availability-group-in-sql-server-2016/)
