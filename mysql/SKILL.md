---
name: mysql
description: MySQL Database skills for administration, performance tuning, security, replication, InnoDB internals, monitoring, schema design, application development, migrations, frameworks, and Percona/MySQL Shell tooling.
---

# MySQL Database Skills

This domain contains MySQL Database skills for administration, performance tuning, security, replication, InnoDB internals, monitoring, schema design, application development, SQL patterns, DevOps, framework integrations, and tooling.

## How to Use This Domain

1. Start with the routing table below.
2. Read only the specific file or category you need.

## Directory Structure

```text
mysql/
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
| Backup, recovery, replication, binary logs, InnoDB internals, users, upgrades | `admin/` |
| Language drivers (Python, Java, Node.js, PHP), connection pooling, JSON, transactions | `appdev/` |
| Schema design, data types, partitioning, character sets, collations | `design/` |
| Schema migrations, CI/CD, version control for SQL | `devops/` |
| SQLAlchemy, Django, Spring JPA, Sequelize, TypeORM, Laravel | `frameworks/` |
| Error log, Performance Schema, sys schema, INFORMATION_SCHEMA | `monitoring/` |
| EXPLAIN plans, slow query log, indexes, buffer pool, query profiling, wait events, optimizer | `performance/` |
| Privileges, authentication plugins, encryption, auditing, network security | `security/` |
| SQL patterns, stored programs, dynamic SQL, SQL tuning | `sql/` |
| MySQL Shell, Percona Toolkit, MySQL Router | `tools/` |

## Key Starting Points

- `performance/explain-plan.md`
- `performance/slow-query-log.md`
- `admin/backup-recovery.md`
- `admin/replication.md`
- `security/privilege-management.md`
- `monitoring/performance-schema.md`
- `tools/percona-toolkit.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Diagnose a slow query | `explain-plan` → `index-strategy` → `optimizer-hints` → `query-profiling` |
| Investigate performance issues | `slow-query-log` → `performance-schema` → `wait-events` → `innodb-buffer-pool` |
| Set up replication | `binary-log` → `replication` → `monitoring/performance-schema` |
| Secure a MySQL instance | `user-management` → `privilege-management` → `authentication` → `encryption` → `network-security` |
| Online schema change | `schema-migrations` → `percona-toolkit` (pt-online-schema-change) |
| Capacity planning | `innodb-buffer-pool` → `sys-schema` → `information-schema` |
