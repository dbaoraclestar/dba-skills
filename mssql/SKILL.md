---
name: mssql
description: SQL Server Database skills for administration, performance tuning, security, Always On availability, monitoring with DMVs and Extended Events, schema design, T-SQL development, application development, migrations, frameworks, and tooling.
---

# SQL Server Database Skills

This domain contains SQL Server Database skills for administration, performance tuning, security, Always On high availability, monitoring with DMVs and Extended Events, schema design, T-SQL development, application development, DevOps, framework integrations, and tooling.

## How to Use This Domain

1. Start with the routing table below.
2. Read only the specific file or category you need.

## Directory Structure

```text
mssql/
├── admin/
├── appdev/
├── design/
├── devops/
├── frameworks/
├── monitoring/
├── performance/
├── security/
├── sql/
└── tools/
```

## Category Routing

| Topic | Directory |
|-------|-----------|
| Backup, recovery, Always On AG, log shipping, user management, maintenance, upgrades | `admin/` |
| Language drivers (.NET, Python, Java, Node.js), connection pooling, JSON, transactions | `appdev/` |
| Schema design, data types, partitioning, filegroups, columnstore | `design/` |
| Schema migrations, CI/CD, SSDT, version control for SQL | `devops/` |
| Entity Framework, Dapper, SQLAlchemy, Django, Sequelize, Spring JPA | `frameworks/` |
| DMVs, Extended Events, error log, Activity Monitor, SQL Server Agent | `monitoring/` |
| Execution plans, wait statistics, indexes, Query Store, memory tuning, statistics | `performance/` |
| Permissions, authentication, TDE, Always Encrypted, auditing, row-level security | `security/` |
| T-SQL patterns, stored procedures, dynamic SQL, SQL tuning | `sql/` |
| SSMS, sqlcmd, Azure Data Studio, bcp, SSIS, dbatools PowerShell | `tools/` |

## Key Starting Points

- `performance/execution-plans.md`
- `performance/wait-statistics.md`
- `admin/backup-recovery.md`
- `admin/always-on-ag.md`
- `security/permissions.md`
- `monitoring/dmvs.md`
- `tools/dbatools.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Diagnose a slow query | `execution-plans` → `index-strategy` → `query-store` → `wait-statistics` |
| Investigate performance issues | `wait-statistics` → `dmvs` → `extended-events` → `memory-tuning` |
| Set up high availability | `always-on-ag` → `backup-recovery` → `monitoring/dmvs` |
| Secure a SQL Server instance | `user-management` → `permissions` → `authentication` → `encryption` → `auditing` |
| Online schema change | `schema-migrations` → `online-index-ops` (in index-strategy) |
| Capacity planning | `memory-tuning` → `dmvs` → `filegroups` (in design) |
