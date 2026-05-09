# DBA Skills for Claude Code

A collection of database administration skills for [Claude Code](https://claude.ai/code). Each skill provides comprehensive reference material covering performance tuning, administration, security, monitoring, schema design, application development, DevOps, framework integrations, and tooling.

## Available Skills

| Skill | Files | Directories | Invoke |
|-------|-------|-------------|--------|
| **MySQL** | 50 | 10 | `/mysql` |
| **MSSQL** | 49 | 10 | `/mssql` |

## Installation

Copy the skill directory to your Claude Code skills folder:

```bash
# Clone this repo
git clone https://github.com/dbaoraclestar/dba-skills.git

# Copy the skills you want
cp -r dba-skills/mysql ~/.claude/skills/mysql
cp -r dba-skills/mssql ~/.claude/skills/mssql
```

Then invoke in Claude Code with `/mysql` or `/mssql` followed by your question.

## MySQL Skill Structure

```
mysql/
├── SKILL.md              # Routing table and entry point
├── performance/           # EXPLAIN, slow query log, indexes, buffer pool, wait events, optimizer
├── admin/                 # Backup/recovery, replication, binary logs, InnoDB, upgrades, users
├── security/              # Privileges, authentication, encryption, auditing, network security
├── monitoring/            # Error log, Performance Schema, sys schema, INFORMATION_SCHEMA
├── design/                # Data types, partitioning, schema design, character sets
├── appdev/                # Python, Java, Node.js, PHP drivers, connection pooling, JSON, transactions
├── sql/                   # SQL patterns, stored programs, dynamic SQL, SQL tuning
├── devops/                # Schema migrations, CI/CD, version control
├── frameworks/            # SQLAlchemy, Django, Spring JPA, Sequelize, TypeORM, Laravel
└── tools/                 # MySQL Shell, Percona Toolkit, MySQL Router
```

## MSSQL Skill Structure

```
mssql/
├── SKILL.md              # Routing table and entry point
├── performance/           # Execution plans, wait stats, indexes, Query Store, memory, statistics, hints
├── admin/                 # Backup/recovery, Always On AG, transaction log, maintenance, upgrades, users
├── security/              # Permissions, authentication, encryption, auditing, row-level security
├── monitoring/            # DMVs, Extended Events, error log, SQL Server Agent
├── design/                # Data types, partitioning, schema design, filegroups
├── appdev/                # .NET, Python, Java, Node.js drivers, JSON, transactions
├── sql/                   # T-SQL patterns, stored procedures, dynamic SQL, SQL tuning
├── devops/                # Schema migrations (SSDT/Flyway/DbUp), CI/CD, version control
├── frameworks/            # Entity Framework, Dapper, SQLAlchemy, Django, Sequelize, Spring JPA
└── tools/                 # SSMS, Azure Data Studio, sqlcmd/bcp, dbatools PowerShell
```

## Content Template

Every file follows a consistent structure:

1. **Overview** - What it is, when to use it
2. **Key Concepts** - Definitions and fundamentals
3. **Topic Sections** - SQL examples, configuration, procedures
4. **Best Practices** - Bulleted recommendations
5. **Common Mistakes** - 3-column table (Mistake | Impact | Fix)
6. **Version Notes** - Version-specific differences (MySQL 5.7/8.0/8.4, SQL Server 2016/2019/2022)
7. **Sources** - Official documentation links

## Usage Examples

```
/mysql how to check slow queries
/mysql set up GTID replication
/mysql InnoDB buffer pool tuning

/mssql how to check wait stats
/mssql set up Always On AG
/mssql Query Store for plan regression
```

## License

MIT
