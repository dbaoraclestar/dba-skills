---
name: oracle
description: Oracle Database skills for administration, SQL and PL/SQL development, performance tuning, security, ORDS, SQLcl, migrations, frameworks, Oracle Container Registry guidance, and agent-safe database workflows.
---

# Oracle Database Skills

This domain contains Oracle Database skills for administration, SQL and PL/SQL development, performance tuning, security, ORDS, SQLcl, migrations, frameworks, OCR container guidance, and agent-safe database workflows.

## How to Use This Domain

1. Start with the routing table below.
2. Read only the specific file or category you need.

## Directory Structure

```text
oracle/
├── admin/
├── agent/
├── appdev/
├── architecture/
├── containers/
├── design/
├── devops/
├── features/
├── frameworks/
├── migrations/
├── monitoring/
├── ords/
├── performance/
├── plsql/
├── security/
├── sql-dev/
└── sqlcl/
```

## Category Routing

| Topic | Directory |
|-------|-----------|
| Backup, recovery, RMAN, Data Guard, redo/undo logs, users | `admin/` |
| Safe DML, destructive operation guards, idempotency, schema discovery, ORA- error handling | `agent/` |
| JDBC, pooling, JSON, XML, spatial, Oracle Text, transactions, MLE, language drivers | `appdev/` |
| RAC, Multitenant, Exadata, In-Memory, OCI database services, Data Guard architecture | `architecture/` |
| OCR database-category container images and pull guidance | `containers/` |
| ERD, data modeling, partitioning, tablespaces | `design/` |
| Schema migrations, online operations, edition-based redefinition, testing, version control | `devops/` |
| AQ, DBMS_SCHEDULER, materialized views, DBLinks, APEX, vector search, SELECT AI | `features/` |
| SQLAlchemy, Django, Pandas, Spring JPA, MyBatis, TypeORM, Sequelize, Dapper, GORM | `frameworks/` |
| Migrations from PostgreSQL, MySQL, SQL Server, MongoDB, Snowflake, and more | `migrations/` |
| Alert log, ADR, health monitor, space management, top SQL | `monitoring/` |
| ORDS architecture, installation, REST design, authentication, monitoring | `ords/` |
| AWR, ASH, explain plan, indexes, optimizer stats, wait events, memory | `performance/` |
| Package design, error handling, performance, collections, cursors, debugging | `plsql/` |
| Privileges, VPD, masking, auditing, encryption, network security | `security/` |
| SQL tuning, SQL patterns, dynamic SQL, injection avoidance | `sql-dev/` |
| SQLcl basics, scripting, Liquibase, formatting, DDL generation, data loading, MCP server, scheduler daemon, AWR, background jobs | `sqlcl/` |

## Key Starting Points

- `sqlcl/sqlcl-mcp-server.md`
- `migrations/migration-assessment.md`
- `performance/explain-plan.md`
- `plsql/plsql-package-design.md`
- `devops/schema-migrations.md`
- `agent/schema-discovery.md`
- `containers/container-selection-matrix.md`

## Common Multi-Step Flows

| Task | Recommended Sequence |
|------|----------------------|
| Diagnose a slow query | `explain-plan` → `wait-events` → `optimizer-stats` → `awr-reports` |
| Plan a migration | `migration-assessment` → `oracle-migration-tools` → source-specific `migrate-*.md` → `migration-cutover-strategy` |
| Build RAG on Oracle Database | `ai-profiles` → `vector-search` → `dbms-vector` |
| Perform agent-safe schema change | `schema-discovery` → `destructive-op-guards` → `idempotency-patterns` → `schema-migrations` |
| Set up AI-driven database access via MCP | `sqlcl-basics` (save connections) → `security/privilege-management` (least-privilege user) → `sqlcl-mcp-server` (configure + start) |
