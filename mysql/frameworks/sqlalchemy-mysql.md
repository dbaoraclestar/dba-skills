# SQLAlchemy MySQL — Engine Configuration, Dialect Features, and ORM Patterns

## Overview

SQLAlchemy is Python's most comprehensive database toolkit, providing both a Core SQL expression language and a full ORM. When used with MySQL, SQLAlchemy's MySQL dialect translates Python models and queries into MySQL-specific DDL and DML, handling AUTO_INCREMENT, ENUM, JSON columns, character sets, and storage engine selection automatically.

The MySQL dialect supports multiple underlying drivers: PyMySQL (pure Python, most common), mysqlclient (C extension, fastest), and mysql-connector-python (Oracle's official driver). Each driver has different connection string prefixes and slight behavioral differences, but SQLAlchemy normalizes them behind a consistent API.

This skill covers engine creation, connection pooling, MySQL-specific types, DDL options, session management, and bulk operation patterns for production MySQL deployments.

## Key Concepts

- **Engine**: The entry point for database connectivity. Holds the connection pool and dialect configuration.
- **Dialect**: SQLAlchemy's adapter layer for MySQL-specific SQL generation and type mapping.
- **Session**: A transactional workspace that tracks object state changes and flushes them to the database.
- **Connection pool**: A pool of persistent database connections reused across requests (QueuePool by default).
- **Mapped class**: A Python class linked to a MySQL table via SQLAlchemy's ORM declarative mapping.

## Engine Creation

### Driver-specific connection URLs

```python
from sqlalchemy import create_engine

# PyMySQL (pure Python, recommended default)
engine = create_engine(
    "mysql+pymysql://user:password@host:3306/dbname"
    "?charset=utf8mb4"
)

# mysqlclient (C extension, fastest)
engine = create_engine(
    "mysql+mysqldb://user:password@host:3306/dbname"
    "?charset=utf8mb4"
)

# mysql-connector-python (Oracle official)
engine = create_engine(
    "mysql+mysqlconnector://user:password@host:3306/dbname"
    "?charset=utf8mb4"
)
```

### Production engine with connection pool

```python
engine = create_engine(
    "mysql+pymysql://app:secret@db-primary:3306/myapp?charset=utf8mb4",
    pool_size=20,              # Steady-state connections
    max_overflow=10,           # Extra connections under load
    pool_timeout=30,           # Seconds to wait for a connection
    pool_recycle=3600,         # Recycle connections after 1 hour (MySQL wait_timeout)
    pool_pre_ping=True,        # Test connections before use (handles MySQL timeout)
    echo=False,                # Set True for SQL logging in dev
    connect_args={
        "connect_timeout": 10,
        "read_timeout": 30,
        "write_timeout": 30,
    },
)
```

### Read replica routing

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

primary_engine = create_engine("mysql+pymysql://app:pass@primary:3306/myapp")
replica_engine = create_engine("mysql+pymysql://app:pass@replica:3306/myapp")

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None, **kw):
        if self._flushing or self.info.get("write"):
            return primary_engine
        return replica_engine

# Usage
session = RoutingSession()
# Reads go to replica
users = session.query(User).all()
# Writes go to primary
session.info["write"] = True
session.add(User(name="Alice"))
session.commit()
```

## MySQL-Specific Column Types

```python
from sqlalchemy import Column, String, Integer, Text, Numeric, DateTime
from sqlalchemy.dialects.mysql import (
    BIGINT, TINYINT, MEDIUMINT, SMALLINT,
    MEDIUMTEXT, LONGTEXT, TINYTEXT,
    ENUM, SET, JSON, YEAR,
    DOUBLE, FLOAT,
    TIMESTAMP, DATETIME,
    BINARY, VARBINARY, MEDIUMBLOB, LONGBLOB,
    BIT, INTEGER
)
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = "products"

    id = Column(BIGINT(unsigned=True), primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False)
    description = Column(MEDIUMTEXT)
    price = Column(Numeric(10, 2), nullable=False)
    quantity = Column(INTEGER(unsigned=True), default=0)
    status = Column(ENUM("active", "inactive", "archived"), default="active")
    tags = Column(SET("sale", "new", "featured", "clearance"))
    metadata_json = Column(JSON)
    is_visible = Column(TINYINT(1), default=1)
    weight = Column(DOUBLE(precision=10, scale=3))
    year_released = Column(YEAR)
    created_at = Column(TIMESTAMP, server_default="CURRENT_TIMESTAMP")
    updated_at = Column(
        TIMESTAMP,
        server_default="CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"
    )
```

## MySQL-Specific DDL (Table Options)

```python
class AuditLog(Base):
    __tablename__ = "audit_log"
    __table_args__ = {
        "mysql_engine": "InnoDB",
        "mysql_charset": "utf8mb4",
        "mysql_collate": "utf8mb4_unicode_ci",
        "mysql_row_format": "DYNAMIC",
        "mysql_key_block_size": "8",
        "mysql_auto_increment": "1000",
        "mysql_comment": "Application audit trail",
        "mysql_partition_by": "RANGE (YEAR(created_at))",
        "mysql_partitions": "4",
    }

    id = Column(BIGINT(unsigned=True), primary_key=True, autoincrement=True)
    action = Column(String(50), nullable=False)
    entity_type = Column(String(100), nullable=False)
    entity_id = Column(BIGINT(unsigned=True), nullable=False)
    payload = Column(JSON)
    created_at = Column(TIMESTAMP, server_default="CURRENT_TIMESTAMP")


# Composite index with MySQL-specific options
from sqlalchemy import Index

Index(
    "idx_entity_lookup",
    AuditLog.entity_type,
    AuditLog.entity_id,
    mysql_using="btree",
    mysql_length={"entity_type": 50},
)

# Fulltext index
Index(
    "ft_action",
    AuditLog.action,
    mysql_prefix="ft",
    mysql_using="fulltext",
)
```

## ENUM and SET Handling

```python
import enum

# Python enum mapped to MySQL ENUM
class OrderStatus(enum.Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class Order(Base):
    __tablename__ = "orders"

    id = Column(BIGINT(unsigned=True), primary_key=True, autoincrement=True)
    # Using Python enum (stored as string in MySQL ENUM column)
    status = Column(
        ENUM(OrderStatus, values_callable=lambda e: [m.value for m in e]),
        default=OrderStatus.PENDING,
        nullable=False,
    )
    # Direct MySQL ENUM without Python enum
    priority = Column(ENUM("low", "medium", "high", "critical"), default="medium")
    # MySQL SET type
    flags = Column(SET("urgent", "fragile", "oversized", "insured"))
```

## Session Management Patterns

```python
from sqlalchemy.orm import sessionmaker, scoped_session
from contextlib import contextmanager

SessionFactory = sessionmaker(bind=engine)
ScopedSession = scoped_session(SessionFactory)

# Context manager pattern (recommended)
@contextmanager
def get_session():
    session = SessionFactory()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
with get_session() as session:
    user = User(name="Alice", email="alice@example.com")
    session.add(user)
    # Auto-commits on exit, rolls back on exception
```

## Bulk Operations

```python
from sqlalchemy import insert, update, text
from sqlalchemy.dialects.mysql import insert as mysql_insert

# Bulk insert (Core API, fastest)
with engine.begin() as conn:
    conn.execute(
        insert(User),
        [
            {"name": "Alice", "email": "alice@example.com"},
            {"name": "Bob", "email": "bob@example.com"},
            {"name": "Charlie", "email": "charlie@example.com"},
        ],
    )

# MySQL INSERT ... ON DUPLICATE KEY UPDATE (upsert)
stmt = mysql_insert(User).values(
    name="Alice", email="alice@example.com"
)
upsert = stmt.on_duplicate_key_update(
    name=stmt.inserted.name,
    updated_at=text("CURRENT_TIMESTAMP"),
)
with engine.begin() as conn:
    conn.execute(upsert)

# Bulk update with executemany
with engine.begin() as conn:
    conn.execute(
        update(User).where(User.id == text(":id")),
        [
            {"id": 1, "name": "Alice Updated"},
            {"id": 2, "name": "Bob Updated"},
        ],
    )

# LOAD DATA INFILE via raw SQL for maximum throughput
with engine.begin() as conn:
    conn.execute(text("""
        LOAD DATA LOCAL INFILE '/tmp/users.csv'
        INTO TABLE users
        FIELDS TERMINATED BY ','
        ENCLOSED BY '"'
        LINES TERMINATED BY '\\n'
        IGNORE 1 LINES
        (name, email, created_at)
    """))
```

## Best Practices

- Always set `pool_pre_ping=True` to handle MySQL's `wait_timeout` disconnections gracefully.
- Set `pool_recycle` to a value less than MySQL's `wait_timeout` (default 28800 seconds).
- Use `charset=utf8mb4` in the connection URL to support full Unicode including emoji.
- Prefer `BIGINT(unsigned=True)` for primary keys to avoid running out of ID space.
- Use `server_default` instead of `default` for timestamps so the database sets the value, not Python.
- Use the Core `insert()` API with lists of dicts for bulk inserts instead of ORM `session.add_all()`.
- Use `mysql_insert().on_duplicate_key_update()` for upsert operations instead of manual SELECT-then-INSERT.
- Set explicit `mysql_engine="InnoDB"` and `mysql_charset="utf8mb4"` on every table definition.
- Use `scoped_session` in web frameworks for thread-safe session management.
- Profile queries with `echo=True` or SQLAlchemy events during development.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not setting `pool_recycle` | "MySQL server has gone away" errors after idle periods | Set `pool_recycle=3600` (less than MySQL `wait_timeout`) |
| Using `String` without length for MySQL | SQLAlchemy generates `TEXT` instead of `VARCHAR`, cannot be indexed efficiently | Always specify `String(255)` or use `Text` explicitly |
| Using Python `default=datetime.now` instead of `server_default` | Time is set at Python process start, not at INSERT time; stale timestamps | Use `server_default=text("CURRENT_TIMESTAMP")` |
| Forgetting `charset=utf8mb4` in connection URL | 4-byte Unicode characters (emoji) cause "Incorrect string value" errors | Add `?charset=utf8mb4` to connection URL |
| Creating sessions without closing them | Connection pool exhaustion; "Too many connections" error | Use context managers or `scoped_session` with `remove()` |

## MySQL Version Notes

- **5.7**: JSON column type supported but without generated column indexes. ENUM and SET work identically. `utf8mb4` available but not default.
- **8.0**: Native JSON functions (`JSON_TABLE`, `JSON_VALUE`) available in raw SQL. Multi-valued indexes on JSON arrays. `caching_sha2_password` may require `pymysql>=0.9.3` or `mysqlclient>=2.0`. Default charset is `utf8mb4`.
- **8.4/9.x**: Removed `mysql_native_password` by default. Ensure PyMySQL or mysqlclient supports `caching_sha2_password`. Enhanced optimizer hints available via `text()` in SQLAlchemy queries.

## Sources

- [SQLAlchemy MySQL Dialect](https://docs.sqlalchemy.org/en/20/dialects/mysql.html)
- [SQLAlchemy Engine Configuration](https://docs.sqlalchemy.org/en/20/core/engines.html)
- [SQLAlchemy Connection Pooling](https://docs.sqlalchemy.org/en/20/core/pooling.html)
- [PyMySQL Documentation](https://pymysql.readthedocs.io/)
- [MySQL Connector/Python](https://dev.mysql.com/doc/connector-python/en/)
