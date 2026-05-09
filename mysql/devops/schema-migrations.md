# Schema Migrations — Tools, Strategies, and Zero-Downtime DDL for MySQL

## Overview

Schema migrations are the process of evolving a database schema over time in a controlled, versioned, and repeatable manner. In MySQL environments, migrations range from simple ALTER TABLE statements to complex online schema changes that must execute without downtime on multi-terabyte tables serving production traffic.

Choosing the right migration tool depends on your table size, replication topology, uptime requirements, and team workflow. Small tables can use native MySQL DDL; medium tables benefit from MySQL 8.0's instant DDL or INPLACE algorithms; large tables under heavy write load often require external tools like pt-online-schema-change or gh-ost to avoid blocking writes.

This skill covers five major approaches: Flyway, Liquibase, pt-online-schema-change, gh-ost, and MySQL 8.0+ instant/online DDL. Each has trade-offs in complexity, safety, rollback support, and operational overhead.

## Key Concepts

- **Migration**: A versioned, atomic change to the database schema (DDL) or reference data (DML).
- **Idempotency**: A migration that can be applied multiple times without error or side effects.
- **Online DDL**: MySQL's built-in ability to perform certain ALTER TABLE operations without blocking concurrent DML.
- **ALGORITHM**: The DDL execution strategy MySQL uses: INSTANT (metadata-only), INPLACE (no table rebuild but may rebuild index), or COPY (full table rebuild).
- **LOCK**: The lock level during DDL: NONE, SHARED, EXCLUSIVE. Lower is better for availability.
- **Trigger-based migration**: Tools like pt-osc that use triggers on the original table to capture changes during migration.
- **Triggerless migration**: Tools like gh-ost that use binlog parsing instead of triggers.

## Flyway (SQL-Based, Java)

Flyway uses plain SQL files with a versioned naming convention. It tracks applied migrations in a `flyway_schema_history` table.

### File naming convention

```
V1__create_users_table.sql
V2__add_email_index.sql
V3__add_orders_table.sql
R__refresh_views.sql          -- Repeatable migration (re-run when checksum changes)
U2__undo_add_email_index.sql  -- Undo migration (Teams/Enterprise only)
```

### Example migration file (V1__create_users_table.sql)

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    username VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### CLI usage

```bash
# Apply pending migrations
flyway -url=jdbc:mysql://localhost:3306/mydb -user=root -password=secret migrate

# Check migration status
flyway -url=jdbc:mysql://localhost:3306/mydb info

# Validate applied migrations match local files
flyway -url=jdbc:mysql://localhost:3306/mydb validate

# Repair metadata table (remove failed entries)
flyway -url=jdbc:mysql://localhost:3306/mydb repair
```

### flyway.conf

```properties
flyway.url=jdbc:mysql://localhost:3306/mydb?useSSL=false&allowPublicKeyRetrieval=true
flyway.user=flyway_user
flyway.password=${FLYWAY_PASSWORD}
flyway.locations=filesystem:./sql/migrations
flyway.baselineOnMigrate=true
flyway.outOfOrder=false
flyway.validateMigrationNaming=true
```

## Liquibase (XML/YAML/SQL ChangeSets)

Liquibase tracks changes via a `DATABASECHANGELOG` table and supports multiple formats.

### YAML changelog example

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: dba-team
      changes:
        - createTable:
            tableName: products
            columns:
              - column:
                  name: id
                  type: BIGINT UNSIGNED
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: name
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
              - column:
                  name: price
                  type: DECIMAL(10,2)
      rollback:
        - dropTable:
            tableName: products

  - changeSet:
      id: 2
      author: dba-team
      preConditions:
        - onFail: MARK_RAN
        - not:
            - indexExists:
                indexName: idx_products_name
      changes:
        - createIndex:
            indexName: idx_products_name
            tableName: products
            columns:
              - column:
                  name: name
```

### CLI usage

```bash
liquibase --changelog-file=changelog.yaml update
liquibase --changelog-file=changelog.yaml rollbackCount 1
liquibase --changelog-file=changelog.yaml status
liquibase --changelog-file=changelog.yaml diff --referenceUrl=jdbc:mysql://localhost/staging
```

## pt-online-schema-change (Percona, Trigger-Based)

pt-online-schema-change creates a new table with the desired schema, copies data in chunks, and uses triggers to capture live DML during the copy.

```bash
# Add a column
pt-online-schema-change \
  --alter "ADD COLUMN phone VARCHAR(20) DEFAULT NULL" \
  --host=db1 --user=root --ask-pass \
  --chunk-size=1000 \
  --max-lag=1s \
  --check-replication-filters \
  --recurse=1 \
  --execute \
  D=mydb,t=users

# Add an index with throttling
pt-online-schema-change \
  --alter "ADD INDEX idx_created (created_at)" \
  --max-load="Threads_running:25" \
  --critical-load="Threads_running:100" \
  --chunk-time=0.5 \
  --execute \
  D=mydb,t=orders
```

### Limitations

- Cannot use on tables with triggers (MySQL trigger-per-event limit, relaxed in 5.7.2+).
- Foreign key handling requires `--alter-foreign-keys-method`.
- Requires unique key for chunking (preferably PRIMARY KEY).

## gh-ost (GitHub, Triggerless)

gh-ost reads the binary log to capture changes instead of using triggers, reducing load on the source.

```bash
# Basic usage
gh-ost \
  --host=db1 \
  --database=mydb \
  --table=users \
  --alter="ADD COLUMN phone VARCHAR(20)" \
  --allow-on-master \
  --chunk-size=1000 \
  --max-lag-millis=1500 \
  --serve-socket-file=/tmp/gh-ost.sock \
  --execute

# Interactive throttling via socket
echo "throttle" | nc -U /tmp/gh-ost.sock
echo "no-throttle" | nc -U /tmp/gh-ost.sock

# Postpone cut-over until manual approval
gh-ost ... --postpone-cut-over-flag-file=/tmp/gh-ost.postpone --execute
# When ready:
rm /tmp/gh-ost.postpone
```

## MySQL 8.0+ Instant and Online DDL

### ALGORITHM=INSTANT (metadata-only, no table rebuild)

```sql
-- Supported instant operations (8.0.12+):
ALTER TABLE users ADD COLUMN nickname VARCHAR(100), ALGORITHM=INSTANT;
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active', ALGORITHM=INSTANT;
ALTER TABLE users RENAME COLUMN old_name TO new_name, ALGORITHM=INSTANT;  -- 8.0.28+

-- 8.0.29+: instant ADD COLUMN at any position
ALTER TABLE users ADD COLUMN middle_name VARCHAR(50) AFTER first_name, ALGORITHM=INSTANT;
```

### ALGORITHM=INPLACE (no table copy, may rebuild indexes)

```sql
ALTER TABLE orders ADD INDEX idx_status (status), ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE orders DROP INDEX idx_old, ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE orders CHANGE COLUMN qty quantity INT, ALGORITHM=INPLACE, LOCK=NONE;
```

### Operations requiring COPY (full rebuild)

```sql
-- Changing column data type (e.g., INT to BIGINT) requires COPY
ALTER TABLE orders MODIFY COLUMN id BIGINT UNSIGNED NOT NULL, ALGORITHM=COPY;
-- Changing CHARSET requires COPY
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4, ALGORITHM=COPY;
```

## Best Practices

- Always test migrations on a staging replica with production-sized data before applying to production.
- Use `--dry-run` (Flyway) or `--no-execute` (pt-osc) to preview changes.
- Set `--max-lag` or `--max-load` thresholds to auto-throttle during replication-aware migrations.
- Keep each migration small and focused on a single logical change.
- Include explicit rollback SQL or reverse changesets for every migration.
- For large tables (>10GB), prefer gh-ost or pt-osc over native DDL to avoid metadata lock waits.
- Use `ALGORITHM=INSTANT` whenever possible on MySQL 8.0+ for zero-impact column additions.
- Never run DDL during peak traffic windows without online DDL tooling.
- Version-control all migration files alongside application code.
- Use a dedicated migration user with limited privileges (ALTER, CREATE, DROP, INDEX, REFERENCES on target schemas).

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running ALTER TABLE on a large table without ALGORITHM hint | Full table copy with EXCLUSIVE lock, blocking all DML for hours | Use `ALGORITHM=INPLACE, LOCK=NONE` or external tools (pt-osc, gh-ost) |
| Not checking replication lag during migration | Replicas fall hours behind, causing read inconsistency or failover triggers | Set `--max-lag=1s` in pt-osc/gh-ost; monitor `Seconds_Behind_Master` |
| Applying migrations out of order across branches | Schema drift between environments, failed deployments | Use Flyway's `outOfOrder=false` or Liquibase preconditions; enforce linear ordering in CI |
| Dropping a column without verifying application compatibility | Application crashes with "Unknown column" errors | Deploy application changes first (stop reading the column), then drop in a subsequent release |
| Using pt-osc on a table that already has triggers | Migration fails or existing triggers are silently dropped | Audit triggers with `SHOW TRIGGERS`; use gh-ost instead (triggerless) |

## MySQL Version Notes

- **5.7**: Online DDL supports INPLACE for most index and column operations. No INSTANT algorithm. `pt-online-schema-change` is the primary tool for large table DDL.
- **8.0**: Introduces `ALGORITHM=INSTANT` (8.0.12) for adding columns at the end. 8.0.29 extends INSTANT to adding columns at any position and dropping columns. Atomic DDL ensures crash-safe schema changes.
- **8.4/9.x**: Further expands INSTANT DDL coverage. Improved online DDL progress monitoring. Deprecates some legacy DDL behaviors.

## Sources

- [MySQL 8.0 Online DDL Operations](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)
- [Flyway Documentation](https://documentation.red-gate.com/fd)
- [Liquibase Documentation](https://docs.liquibase.com/)
- [pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [gh-ost Documentation](https://github.com/github/gh-ost/tree/master/doc)
- [MySQL 8.0 Instant DDL](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-column-operations)
