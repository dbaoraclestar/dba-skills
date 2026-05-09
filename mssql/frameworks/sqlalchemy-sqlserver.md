# SQLAlchemy with SQL Server -- Python ORM and Core for MSSQL

## Overview

SQLAlchemy is the most widely used Python SQL toolkit and ORM. It connects to SQL Server through the `mssql+pyodbc` or `mssql+pymssql` dialects, translating Python model definitions and query expressions into T-SQL. SQLAlchemy offers two distinct usage patterns: the ORM layer (session-based, change tracking, relationships) and Core (table-centric, explicit SQL construction without object mapping).

SQLAlchemy is the right choice for Python applications that need structured database access with SQL Server. It handles connection pooling, dialect-specific SQL generation, schema DDL, and transaction management. For data science or ad-hoc queries, consider using it alongside pandas `read_sql()`.

Use this skill when configuring SQLAlchemy with SQL Server, mapping SQL Server-specific data types, managing sessions and transactions, or tuning connection pool settings for production workloads.

## Key Concepts

- **Engine**: Manages the connection pool and dialect. Created once per application via `create_engine()`.
- **Session**: Unit-of-work pattern for the ORM. Tracks object changes and flushes them as SQL on commit.
- **Dialect**: SQLAlchemy's abstraction for database-specific SQL generation. The `mssql` dialect handles T-SQL syntax, IDENTITY columns, TOP vs LIMIT, and SQL Server type mappings.
- **Core vs ORM**: Core uses `Table` objects and `select()` constructs. ORM uses mapped Python classes with `Session`. Both generate the same SQL.
- **Connection pooling**: SQLAlchemy maintains a pool of DBAPI connections. Default is `QueuePool` with 5 connections and 10 overflow.

## Engine Creation

```python
from sqlalchemy import create_engine

# pyodbc with ODBC Driver 18 (recommended)
engine = create_engine(
    "mssql+pyodbc://username:password@myserver/MyApp"
    "?driver=ODBC+Driver+18+for+SQL+Server"
    "&TrustServerCertificate=yes",
    echo=False,    # Set True to log all SQL
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600,  # Recycle connections after 1 hour
)

# pyodbc with DSN
engine = create_engine("mssql+pyodbc://username:password@my_dsn")

# pyodbc with Windows authentication (Trusted_Connection)
engine = create_engine(
    "mssql+pyodbc://@myserver/MyApp"
    "?driver=ODBC+Driver+18+for+SQL+Server"
    "&Trusted_Connection=yes"
    "&TrustServerCertificate=yes"
)

# pymssql dialect (pure Python, no ODBC driver needed)
engine = create_engine(
    "mssql+pymssql://username:password@myserver:1433/MyApp"
)

# Azure SQL with Azure AD token (pyodbc)
from azure.identity import DefaultAzureCredential
import struct

credential = DefaultAzureCredential()
token = credential.get_token("https://database.windows.net/.default")

engine = create_engine(
    "mssql+pyodbc://@myserver.database.windows.net/MyApp"
    "?driver=ODBC+Driver+18+for+SQL+Server",
    connect_args={
        "attrs_before": {
            1256: struct.pack("=IH", token.token.encode("utf-16-le").__len__(),
                              *token.token.encode("utf-16-le"))
        }
    }
)
```

## SQL Server Specific Types

```python
from sqlalchemy import Column, Integer, String, DateTime, Numeric, Text, Boolean
from sqlalchemy.dialects.mssql import (
    UNIQUEIDENTIFIER, XML, NVARCHAR, DATETIME2,
    BIT, MONEY, SMALLMONEY, TINYINT, IMAGE,
    NTEXT, SQL_VARIANT, ROWVERSION, DATETIMEOFFSET
)
from sqlalchemy.orm import DeclarativeBase
import uuid

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "Customers"
    __table_args__ = {"schema": "dbo"}

    CustomerID = Column(Integer, primary_key=True, autoincrement=True)  # IDENTITY(1,1)
    CustomerGUID = Column(UNIQUEIDENTIFIER, default=uuid.uuid4, unique=True)
    FirstName = Column(NVARCHAR(100), nullable=False)
    LastName = Column(NVARCHAR(100), nullable=False)
    Email = Column(NVARCHAR(256), nullable=False, unique=True, index=True)
    Bio = Column(NVARCHAR(None))  # NVARCHAR(MAX)
    ProfileXML = Column(XML)
    Balance = Column(MONEY)
    IsActive = Column(BIT, default=True)
    CreatedAt = Column(DATETIME2(precision=3), server_default="SYSUTCDATETIME()")
    LastLogin = Column(DATETIMEOFFSET)

class AuditLog(Base):
    __tablename__ = "AuditLog"
    __table_args__ = {"schema": "dbo"}

    LogID = Column(Integer, primary_key=True, autoincrement=True)
    TableName = Column(NVARCHAR(128), nullable=False)
    Action = Column(NVARCHAR(10), nullable=False)
    OldData = Column(NVARCHAR(None))  # NVARCHAR(MAX) for JSON
    NewData = Column(NVARCHAR(None))
    ChangedAt = Column(DATETIME2(precision=3), server_default="SYSUTCDATETIME()")
```

## IDENTITY and Autoincrement

```python
from sqlalchemy import Sequence, Identity

# Default: IDENTITY(1,1)
class Order(Base):
    __tablename__ = "Orders"
    __table_args__ = {"schema": "dbo"}

    OrderID = Column(Integer, Identity(start=1, increment=1), primary_key=True)
    CustomerID = Column(Integer, nullable=False)
    OrderDate = Column(DATETIME2(precision=3), server_default="SYSUTCDATETIME()")
    TotalAmount = Column(Numeric(18, 2), nullable=False)

# Sequence-based (less common in SQL Server)
order_seq = Sequence("order_id_seq", schema="dbo")
class OrderWithSeq(Base):
    __tablename__ = "OrdersSeq"
    OrderID = Column(Integer, order_seq, server_default=order_seq.next_value(), primary_key=True)
```

## Session Management

```python
from sqlalchemy.orm import sessionmaker, Session

SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

# Basic usage
with SessionLocal() as session:
    customer = Customer(FirstName="Jane", LastName="Doe", Email="jane@example.com")
    session.add(customer)
    session.commit()

    # Query
    active_customers = session.query(Customer).filter(Customer.IsActive == True).all()

# SQLAlchemy 2.0 style
from sqlalchemy import select

with Session(engine) as session:
    stmt = select(Customer).where(Customer.Email == "jane@example.com")
    customer = session.execute(stmt).scalar_one_or_none()

    if customer:
        customer.LastName = "Smith"
        session.commit()

# FastAPI dependency injection pattern
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/customers/{customer_id}")
async def get_customer(customer_id: int, db: Session = Depends(get_db)):
    stmt = select(Customer).where(Customer.CustomerID == customer_id)
    customer = db.execute(stmt).scalar_one_or_none()
    return customer
```

## Relationships and Joins

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Customer(Base):
    __tablename__ = "Customers"
    __table_args__ = {"schema": "dbo"}

    CustomerID = Column(Integer, primary_key=True, autoincrement=True)
    FirstName = Column(NVARCHAR(100), nullable=False)
    LastName = Column(NVARCHAR(100), nullable=False)

    orders = relationship("Order", back_populates="customer", lazy="selectin")

class Order(Base):
    __tablename__ = "Orders"
    __table_args__ = {"schema": "dbo"}

    OrderID = Column(Integer, primary_key=True, autoincrement=True)
    CustomerID = Column(Integer, ForeignKey("dbo.Customers.CustomerID"), nullable=False)
    TotalAmount = Column(Numeric(18, 2), nullable=False)

    customer = relationship("Customer", back_populates="orders")
    items = relationship("OrderItem", back_populates="order", lazy="selectin")
```

```python
# Querying with joins (2.0 style)
from sqlalchemy import func

stmt = (
    select(Customer.FirstName, Customer.LastName, func.count(Order.OrderID).label("order_count"))
    .join(Order, Customer.CustomerID == Order.CustomerID)
    .group_by(Customer.FirstName, Customer.LastName)
    .having(func.count(Order.OrderID) > 5)
    .order_by(func.count(Order.OrderID).desc())
)

results = session.execute(stmt).all()
```

## Bulk Operations

```python
# Bulk insert (ORM)
customers = [
    Customer(FirstName=f"User{i}", LastName="Test", Email=f"user{i}@test.com")
    for i in range(10000)
]
session.add_all(customers)
session.commit()

# Bulk insert (Core -- faster, bypasses ORM overhead)
session.execute(
    Customer.__table__.insert(),
    [
        {"FirstName": f"User{i}", "LastName": "Test", "Email": f"user{i}@test.com"}
        for i in range(10000)
    ]
)
session.commit()

# Bulk update (Core)
from sqlalchemy import update

stmt = (
    update(Customer)
    .where(Customer.IsActive == False)
    .values(Email=func.concat(Customer.Email, ".deactivated"))
)
session.execute(stmt)
session.commit()
```

## Raw SQL with text()

```python
from sqlalchemy import text

# Raw SELECT
result = session.execute(
    text("SELECT TOP 10 * FROM dbo.Customers WHERE LastName = :last_name ORDER BY CreatedAt DESC"),
    {"last_name": "Smith"}
)
rows = result.fetchall()

# Raw INSERT with OUTPUT
result = session.execute(
    text("""
        INSERT INTO dbo.Customers (FirstName, LastName, Email)
        OUTPUT INSERTED.CustomerID
        VALUES (:first, :last, :email)
    """),
    {"first": "Jane", "last": "Doe", "email": "jane@test.com"}
)
new_id = result.scalar()

# Calling stored procedures
result = session.execute(
    text("EXEC dbo.usp_SearchCustomers @SearchTerm = :term, @MaxResults = :max"),
    {"term": "smith", "max": 50}
)
customers = result.fetchall()
```

## SQL Server Specific DDL

```python
from sqlalchemy import Index, event

# Filtered index (SQL Server 2008+)
Index("IX_Customers_ActiveEmail", Customer.Email,
      mssql_where=Customer.IsActive == True)

# Included columns
Index("IX_Orders_CustomerID", Order.CustomerID,
      mssql_include=["OrderDate", "TotalAmount"])

# Clustered index
Index("IX_AuditLog_ChangedAt", AuditLog.ChangedAt,
      mssql_clustered=True)

# Filegroup placement
class LargeTable(Base):
    __tablename__ = "LargeData"
    __table_args__ = {"mssql_data_filegroup": "SECONDARY"}

# Schema creation
from sqlalchemy import DDL
event.listen(Base.metadata, "before_create",
    DDL("IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = 'staging') EXEC('CREATE SCHEMA staging')"))
```

## Connection Pooling Configuration

```python
# Production-grade pool settings
engine = create_engine(
    connection_string,
    pool_size=20,           # Steady-state connections
    max_overflow=30,        # Additional connections under burst
    pool_timeout=30,        # Seconds to wait for a connection before error
    pool_recycle=1800,      # Recycle connections after 30 minutes
    pool_pre_ping=True,     # Test connection health before use
    echo_pool="debug",      # Log pool checkout/checkin events
    connect_args={
        "timeout": 10,      # Connection timeout in seconds
    }
)

# For short-lived scripts (no pooling needed)
engine = create_engine(connection_string, poolclass=NullPool)
```

## Best Practices

- Use `mssql+pyodbc` with ODBC Driver 18 for production; pymssql for quick scripts without ODBC
- Set `pool_pre_ping=True` to handle stale connections gracefully
- Use `NVARCHAR` instead of `String`/`VARCHAR` for Unicode support
- Prefer SQLAlchemy 2.0 style (`select()`, `Session.execute()`) over legacy Query API
- Use `expire_on_commit=False` to avoid lazy-loading after commit in web frameworks
- Use Core bulk operations for large inserts (10x faster than ORM `add_all`)
- Set `pool_recycle` shorter than SQL Server's connection timeout to avoid broken pipe errors
- Use `text()` for complex T-SQL that does not map naturally to SQLAlchemy constructs

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `VARCHAR` instead of `NVARCHAR` for Unicode data | Corrupted characters for non-ASCII text | Use `NVARCHAR(length)` from `sqlalchemy.dialects.mssql` |
| Not setting `pool_pre_ping=True` | Stale connections cause intermittent failures after idle periods | Enable pre-ping: `create_engine(..., pool_pre_ping=True)` |
| Forgetting schema prefix in foreign keys | ForeignKey references fail with "table not found" | Use full dotted path: `ForeignKey("dbo.Customers.CustomerID")` |
| Using `DATETIME` instead of `DATETIME2` | 3.33ms precision, larger storage, no timezone offset | Use `DATETIME2(precision=3)` for new columns |
| Not closing sessions in web request handlers | Connection pool exhaustion under load | Use context manager (`with Session() as s:`) or framework dependency injection |

## SQL Server Version Notes

- **SQL Server 2016**: Full pyodbc support. JSON functions available in raw SQL but no native JSON column type. Row-level security usable but not modeled in SQLAlchemy. Always Encrypted requires ODBC Driver 17+.
- **SQL Server 2019**: UTF-8 collation support (use `NVARCHAR` regardless for portability). Accelerated Database Recovery reduces long-transaction rollback impact on pool. ODBC Driver 17.4+ recommended.
- **SQL Server 2022**: Native JSON type (use `NVARCHAR(MAX)` in SQLAlchemy until dialect adds support). ODBC Driver 18 enables encryption by default -- add `TrustServerCertificate=yes` for self-signed certs. Ledger tables not modeled in SQLAlchemy.

## Sources

- https://docs.sqlalchemy.org/en/20/dialects/mssql.html
- https://docs.sqlalchemy.org/en/20/core/pooling.html
- https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server
- https://github.com/mkleehammer/pyodbc/wiki
- https://docs.sqlalchemy.org/en/20/orm/session_basics.html
