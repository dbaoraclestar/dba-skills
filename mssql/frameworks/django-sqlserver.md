# Django with SQL Server -- mssql-django Backend Configuration and Usage

## Overview

Django connects to SQL Server through the `mssql-django` backend (formerly `django-mssql-backend`), maintained by Microsoft. This backend translates Django's ORM operations into T-SQL, handles SQL Server-specific data types, and supports both SQL authentication and Windows/Azure AD authentication. It allows Django applications to use SQL Server as their primary database while retaining full ORM, migration, and admin functionality.

The `mssql-django` backend supports Django 3.2 through 5.x and SQL Server 2016 through 2022, including Azure SQL Database. While most Django features work seamlessly, there are specific differences from the PostgreSQL backend that developers should be aware of -- particularly around JSONField behavior, TextField limitations, and transaction handling.

Use this skill when configuring Django to connect to SQL Server, understanding backend-specific model considerations, or troubleshooting migration issues unique to the SQL Server dialect.

## Key Concepts

- **mssql-django**: The official Microsoft-maintained Django database backend for SQL Server. Install via `pip install mssql-django`.
- **pyodbc**: The underlying DBAPI driver. mssql-django requires pyodbc and an ODBC driver (typically ODBC Driver 17 or 18 for SQL Server).
- **IDENTITY columns**: Django's AutoField maps to SQL Server's IDENTITY(1,1) by default.
- **NVARCHAR vs VARCHAR**: Django's CharField and TextField map to NVARCHAR by default for Unicode support.
- **Collation**: SQL Server's default collation is case-insensitive (`SQL_Latin1_General_CP1_CI_AS`), which affects Django's `__exact` lookups.

## DATABASES Configuration

```python
# settings.py

DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "MyDjangoApp",
        "HOST": "sqlserver.example.com",
        "PORT": "1433",
        "USER": "django_app",
        "PASSWORD": "SecureP@ss!",
        "OPTIONS": {
            "driver": "ODBC Driver 18 for SQL Server",
            "extra_params": "TrustServerCertificate=yes;Encrypt=yes",
        },
    }
}

# Windows authentication (Trusted_Connection)
DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "MyDjangoApp",
        "HOST": "sqlserver.example.com",
        "PORT": "1433",
        "OPTIONS": {
            "driver": "ODBC Driver 18 for SQL Server",
            "extra_params": "Trusted_Connection=yes;TrustServerCertificate=yes",
        },
    }
}

# Azure SQL Database
DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "MyDjangoApp",
        "HOST": "myserver.database.windows.net",
        "PORT": "1433",
        "USER": "django_app@myserver",
        "PASSWORD": "SecureP@ss!",
        "OPTIONS": {
            "driver": "ODBC Driver 18 for SQL Server",
            "extra_params": "Encrypt=yes",
        },
    }
}

# Named instance
DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "MyDjangoApp",
        "HOST": r"sqlserver.example.com\INST01",
        "PORT": "",
        "USER": "django_app",
        "PASSWORD": "SecureP@ss!",
        "OPTIONS": {
            "driver": "ODBC Driver 18 for SQL Server",
        },
    }
}
```

## Installation

```bash
# Install mssql-django and dependencies
pip install mssql-django

# On Linux, install ODBC Driver 18
# Ubuntu/Debian
curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
sudo add-apt-repository "$(curl https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list)"
sudo apt-get update
sudo apt-get install -y msodbcsql18

# macOS
brew install microsoft/mssql-release/msodbcsql18

# Verify ODBC driver
odbcinst -q -d
```

## SQL Server Specific Model Considerations

```python
from django.db import models

class Customer(models.Model):
    # AutoField uses IDENTITY(1,1)
    # BigAutoField uses BIGINT IDENTITY(1,1) -- default in Django 3.2+
    first_name = models.CharField(max_length=100)   # NVARCHAR(100)
    last_name = models.CharField(max_length=100)     # NVARCHAR(100)
    email = models.EmailField(unique=True)            # NVARCHAR(254)

    # TextField maps to NVARCHAR(MAX) in SQL Server
    # IMPORTANT: NVARCHAR(MAX) columns cannot be used in unique constraints or indexes
    bio = models.TextField(blank=True, default="")

    # DecimalField maps to DECIMAL(max_digits, decimal_places)
    balance = models.DecimalField(max_digits=18, decimal_places=2, default=0)

    # DateTimeField maps to DATETIME2(6) by default
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # BooleanField maps to BIT
    is_active = models.BooleanField(default=True)

    # UUIDField maps to UNIQUEIDENTIFIER (native SQL Server type)
    external_id = models.UUIDField(default=None, null=True, unique=True)

    class Meta:
        db_table = "Customers"
        # SQL Server default collation is case-insensitive
        # This means email lookups are case-insensitive by default


class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT, related_name="orders")
    order_date = models.DateTimeField(auto_now_add=True)
    total_amount = models.DecimalField(max_digits=18, decimal_places=2)
    status = models.CharField(max_length=50, default="Pending",
                              choices=[
                                  ("Pending", "Pending"),
                                  ("Processing", "Processing"),
                                  ("Shipped", "Shipped"),
                                  ("Delivered", "Delivered"),
                              ])
    shipped_date = models.DateTimeField(null=True, blank=True)

    class Meta:
        db_table = "Orders"
        indexes = [
            models.Index(fields=["customer", "order_date"], name="IX_Orders_Cust_Date"),
        ]
```

## JSONField Support

```python
# JSONField is supported but with SQL Server-specific behavior
class ProductConfig(models.Model):
    name = models.CharField(max_length=200)

    # Stored as NVARCHAR(MAX) with JSON validation
    # JSON querying uses SQL Server's JSON_VALUE / JSON_QUERY functions
    metadata = models.JSONField(default=dict, blank=True)
    tags = models.JSONField(default=list, blank=True)

    class Meta:
        db_table = "ProductConfigs"

# Querying JSON fields
# Key lookup (uses JSON_VALUE internally)
products = ProductConfig.objects.filter(metadata__color="red")

# Nested key lookup
products = ProductConfig.objects.filter(metadata__dimensions__width=10)

# Contains lookup
products = ProductConfig.objects.filter(tags__contains=["sale"])

# LIMITATION: JSON containment (@>) and key-exists operators from PostgreSQL
# are not fully supported. Complex JSON queries may need raw SQL.
from django.db.models import Value
from django.db.models.functions import Cast

# Raw SQL fallback for complex JSON operations
products = ProductConfig.objects.raw("""
    SELECT * FROM ProductConfigs
    WHERE JSON_VALUE(metadata, '$.color') = %s
    AND ISJSON(metadata) = 1
""", ["red"])
```

## Migrations with SQL Server Constraints

```python
# migrations/0002_add_index.py
from django.db import migrations, models

class Migration(migrations.Migration):
    dependencies = [("myapp", "0001_initial")]

    operations = [
        # Standard index
        migrations.AddIndex(
            model_name="order",
            index=models.Index(fields=["status", "-order_date"], name="IX_Orders_Status_Date"),
        ),

        # RunSQL for SQL Server-specific operations
        migrations.RunSQL(
            sql="""
                CREATE NONCLUSTERED INDEX IX_Orders_Customer_Includes
                ON dbo.Orders (customer_id)
                INCLUDE (order_date, total_amount)
                WHERE status != 'Cancelled'
            """,
            reverse_sql="DROP INDEX IF EXISTS IX_Orders_Customer_Includes ON dbo.Orders",
        ),

        # Adding a column with SQL Server-specific default
        migrations.RunSQL(
            sql="ALTER TABLE dbo.Customers ADD loyalty_tier NVARCHAR(20) NOT NULL DEFAULT N'Bronze'",
            reverse_sql="ALTER TABLE dbo.Customers DROP COLUMN loyalty_tier",
        ),
    ]
```

## Connection Pooling

```python
# settings.py -- connection pooling options

DATABASES = {
    "default": {
        "ENGINE": "mssql",
        "NAME": "MyDjangoApp",
        "HOST": "sqlserver.example.com",
        "USER": "django_app",
        "PASSWORD": "SecureP@ss!",
        "OPTIONS": {
            "driver": "ODBC Driver 18 for SQL Server",
            "extra_params": "TrustServerCertificate=yes",
        },
        # Django's built-in connection pooling (Django 5.1+)
        "CONN_MAX_AGE": 600,          # Keep connections open for 10 minutes
        "CONN_HEALTH_CHECKS": True,   # Test connections before use (Django 4.1+)
    }
}

# For older Django versions or external pooling, use django-db-connection-pool
# pip install django-db-connection-pool[mssql]
# DATABASES = {
#     "default": {
#         "ENGINE": "dj_db_conn_pool.backends.mssql",
#         "POOL_OPTIONS": {
#             "POOL_SIZE": 10,
#             "MAX_OVERFLOW": 20,
#             "RECYCLE": 300,
#         },
#         ...
#     }
# }
```

## Custom Management Commands

```python
# management/commands/sqlserver_health.py
from django.core.management.base import BaseCommand
from django.db import connection

class Command(BaseCommand):
    help = "Check SQL Server connection and basic health"

    def handle(self, *args, **options):
        with connection.cursor() as cursor:
            cursor.execute("SELECT @@VERSION")
            version = cursor.fetchone()[0]
            self.stdout.write(f"Connected to: {version[:80]}")

            cursor.execute("""
                SELECT DB_NAME() AS db_name,
                       SUSER_SNAME() AS login,
                       USER_NAME() AS db_user,
                       @@SERVERNAME AS server
            """)
            row = cursor.fetchone()
            self.stdout.write(f"Database: {row[0]}, Login: {row[1]}, "
                              f"DB User: {row[2]}, Server: {row[3]}")

            cursor.execute("""
                SELECT COUNT(*) FROM sys.dm_exec_sessions
                WHERE database_id = DB_ID()
            """)
            sessions = cursor.fetchone()[0]
            self.stdout.write(f"Active sessions: {sessions}")
```

## Limitations vs PostgreSQL Backend

```python
# Features with differences or limitations on SQL Server:

# 1. TextField cannot be used in unique constraints
# This FAILS on SQL Server (NVARCHAR(MAX) cannot be in UNIQUE):
# class Slug(models.Model):
#     value = models.TextField(unique=True)  # ERROR

# Fix: Use CharField with explicit max_length
class Slug(models.Model):
    value = models.CharField(max_length=500, unique=True)

# 2. Case sensitivity -- SQL Server default is case-INSENSITIVE
# Customer.objects.filter(email="Jane@Example.com")  # Matches "jane@example.com"
# Use collation override if case-sensitive comparison needed:
from django.db.models.functions import Collate
Customer.objects.annotate(
    email_cs=Collate("email", "SQL_Latin1_General_CP1_CS_AS")
).filter(email_cs="Jane@Example.com")

# 3. Distinct on specific fields not supported
# PostgreSQL: QuerySet.distinct("field")  -- not available on SQL Server
# Workaround: use values() + annotate() or raw SQL

# 4. ArrayField not available (PostgreSQL-specific)
# Use JSONField with a list or a separate many-to-many table

# 5. Full-text search differs from PostgreSQL
# Use raw SQL with SQL Server's CONTAINS / FREETEXT
from django.db.models import Q
results = Customer.objects.raw("""
    SELECT * FROM Customers
    WHERE CONTAINS((first_name, last_name), %s)
""", ["smith"])
```

## Best Practices

- Install ODBC Driver 18 and use `Encrypt=yes` for all connections
- Set `CONN_HEALTH_CHECKS = True` (Django 4.1+) to handle stale connections
- Use `CharField(max_length=N)` instead of `TextField` for any column needing indexing or unique constraints
- Be aware of case-insensitive collation -- it affects filter() exact lookups
- Use `RunSQL` in migrations for SQL Server-specific DDL (filtered indexes, included columns, compression)
- Test migrations against a SQL Server instance, not just SQLite, during development
- Use UUIDField for external-facing IDs -- it maps to native UNIQUEIDENTIFIER
- Set `CONN_MAX_AGE` to avoid reconnection overhead on every request
- Pin your mssql-django version to avoid unexpected behavior changes on upgrade

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `TextField` with `unique=True` | Migration fails: NVARCHAR(MAX) cannot have unique constraint | Use `CharField(max_length=N)` instead |
| Assuming case-sensitive filter() lookups | Queries match more rows than expected due to CI collation | Use `Collate()` annotation or configure database with CS collation |
| Not installing ODBC driver on Linux CI/CD | `django.db.utils.Error: Can't open lib 'ODBC Driver 18'` | Add ODBC driver installation to Dockerfile and CI setup steps |
| Using `distinct("field")` (PostgreSQL-only) | `NotSupportedError` at runtime | Use `values().annotate()` or subquery workarounds |
| Forgetting `TrustServerCertificate=yes` with self-signed certs | Connection fails with SSL/TLS error | Add to `extra_params` in OPTIONS for dev/test environments |

## SQL Server Version Notes

- **SQL Server 2016**: Full mssql-django support. Temporal tables available but not modeled in Django ORM. JSON_VALUE works for JSONField queries but with limited operator support. Row-level security not integrated.
- **SQL Server 2019**: Improved query performance with batch mode on rowstore. UTF-8 collation support available. OPTIMIZE_FOR_SEQUENTIAL_KEY improves insert-heavy workloads common in Django.
- **SQL Server 2022**: Ledger tables not supported by mssql-django. Native JSON type not yet mapped (still uses NVARCHAR(MAX)). Parameter-sensitive plan optimization improves Django's parameterized query performance. GREATEST/LEAST available in raw SQL.

## Sources

- https://github.com/microsoft/mssql-django
- https://pypi.org/project/mssql-django/
- https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server
- https://docs.djangoproject.com/en/5.0/ref/databases/
- https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/installing-the-microsoft-odbc-driver-for-sql-server
