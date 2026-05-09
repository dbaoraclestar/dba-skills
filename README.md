# DBA Skills for Claude Code

A collection of database administration skills for [Claude Code](https://claude.ai/code). Each skill provides comprehensive reference material covering performance tuning, administration, security, monitoring, schema design, application development, DevOps, framework integrations, and tooling.

## Available Skills

| Skill | Files | Directories | Invoke |
|-------|-------|-------------|--------|
| **MySQL** | 50 | 10 | `/mysql` |

## Installation

Copy the skill directory to your Claude Code skills folder:

```bash
# Clone this repo
git clone https://github.com/dbaoraclestar/dba-skills.git

# Copy the skill you want
cp -r dba-skills/mysql ~/.claude/skills/mysql
```

Then invoke it in Claude Code with `/mysql` followed by your question.

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

## Content Template

Every file follows a consistent structure:

1. **Overview** - What it is, when to use it
2. **Key Concepts** - Definitions and fundamentals
3. **Topic Sections** - SQL examples, configuration, procedures
4. **Best Practices** - Bulleted recommendations
5. **Common Mistakes** - 3-column table (Mistake | Impact | Fix)
6. **Version Notes** - MySQL 5.7 vs 8.0 vs 8.4/9.x differences
7. **Sources** - Official MySQL documentation links

## Usage Examples

```
/mysql how to check slow queries
/mysql explain plan for a slow query
/mysql set up GTID replication
/mysql InnoDB buffer pool tuning
/mysql deadlock troubleshooting
```

## License

MIT
