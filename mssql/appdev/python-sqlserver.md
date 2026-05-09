# Python and SQL Server — pyodbc, pymssql, and SQLAlchemy Integration

## Overview

Python connects to SQL Server primarily through pyodbc, which uses the Microsoft ODBC Driver for SQL Server (versions 17 and 18) to communicate via the TDS protocol. pyodbc is the most widely used and recommended driver, offering full feature support including Windows/Azure AD authentication, Always Encrypted, and connection resilience.

pymssql is an alternative pure-Python-friendly driver built on FreeTDS. It requires no ODBC driver installation, making it simpler to set up on Linux, but it lacks some advanced features like Always Encrypted. For most production workloads, pyodbc with ODBC Driver 18 is the recommended stack.

SQLAlchemy provides a higher-level ORM and schema management layer using the `mssql+pyodbc` dialect. Combined with pandas `read_sql` and `to_sql`, this stack covers everything from ad hoc analytics to production application development.

## Key Concepts

- **pyodbc** — Python DB-API 2.0 driver that wraps unixODBC/Windows ODBC. Requires Microsoft ODBC Driver 17 or 18 installed on the system.
- **pymssql** — Lighter alternative using FreeTDS. No ODBC driver needed. Good for simple use cases but less actively maintained.
- **ODBC Driver 17 vs 18** — Driver 18 defaults `Encrypt=yes` (breaking change). Driver 17 defaults `Encrypt=no`. Both support TLS 1.2.
- **Connection pooling** — pyodbc does not pool connections natively. Use SQLAlchemy's pool or a connection pool wrapper.
- **Parameterized queries** — Use `?` placeholders in pyodbc (positional), never f-strings or string formatting.
- **DB-API 2.0** — Python's standard database interface specification. Both pyodbc and pymssql conform to it.

## Connection Setup with pyodbc

```python
import pyodbc

# Basic SQL Authentication
conn = pyodbc.connect(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=myserver.database.windows.net;"
    "DATABASE=mydb;"
    "UID=myuser;"
    "PWD=mypassword;"
    "Encrypt=yes;"
    "TrustServerCertificate=no;"
)

# Windows Integrated Authentication
conn = pyodbc.connect(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=myserver;"
    "DATABASE=mydb;"
    "Trusted_Connection=yes;"
)

# Azure AD with access token
import struct

token = get_azure_ad_token()  # from azure.identity or msal
token_bytes = token.encode("UTF-16-LE")
token_struct = struct.pack(f"<I{len(token_bytes)}s", len(token_bytes), token_bytes)

conn = pyodbc.connect(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=myserver.database.windows.net;"
    "DATABASE=mydb;",
    attrs_before={1256: token_struct}  # SQL_COPT_SS_ACCESS_TOKEN
)

# Connection with timeout and autocommit
conn = pyodbc.connect(connection_string, timeout=30, autocommit=False)
```

## Connection Setup with pymssql

```python
import pymssql

# Basic connection
conn = pymssql.connect(
    server="myserver",
    user="myuser",
    password="mypassword",
    database="mydb",
    port=1433,
    login_timeout=30,
    charset="UTF-8"
)

# Windows Authentication (Linux with FreeTDS Kerberos)
conn = pymssql.connect(
    server="myserver",
    database="mydb"
    # omit user/password for integrated auth
)
```

## Parameterized Queries

```python
# pyodbc — use ? placeholders (positional)
cursor = conn.cursor()

# SELECT with parameters
cursor.execute(
    "SELECT EmployeeId, Name, Salary FROM Employees WHERE Department = ? AND Salary > ?",
    ("Engineering", 80000)
)

for row in cursor.fetchall():
    print(f"{row.EmployeeId}: {row.Name} - ${row.Salary:,.2f}")

# INSERT with parameters
cursor.execute(
    "INSERT INTO Employees (Name, Department, Salary) VALUES (?, ?, ?)",
    ("Jane Doe", "Engineering", 95000)
)
conn.commit()

# UPDATE returning rows affected
cursor.execute(
    "UPDATE Employees SET Salary = ? WHERE EmployeeId = ?",
    (100000, 42)
)
print(f"Updated {cursor.rowcount} rows")
conn.commit()

# NEVER do this — SQL injection risk
# cursor.execute(f"SELECT * FROM Employees WHERE Name = '{user_input}'")  # BAD
```

## Batch Operations with executemany

```python
# executemany for batch inserts
employees = [
    ("Alice", "Engineering", 95000),
    ("Bob", "Marketing", 82000),
    ("Carol", "Engineering", 105000),
    ("Dave", "Sales", 78000),
]

cursor.executemany(
    "INSERT INTO Employees (Name, Department, Salary) VALUES (?, ?, ?)",
    employees
)
conn.commit()
print(f"Inserted {len(employees)} rows")

# For large batches, use fast_executemany (pyodbc 4.0.19+)
cursor.fast_executemany = True
cursor.executemany(
    "INSERT INTO LargeTable (Col1, Col2, Col3) VALUES (?, ?, ?)",
    large_dataset  # list of tuples
)
conn.commit()
```

## Cursor Types and Result Handling

```python
# Default cursor — forward-only
cursor = conn.cursor()
cursor.execute("SELECT * FROM Employees")

# Fetch methods
row = cursor.fetchone()          # single row (or None)
rows = cursor.fetchmany(100)     # up to 100 rows
all_rows = cursor.fetchall()     # all remaining rows

# Access columns by name (pyodbc rows support attribute access)
cursor.execute("SELECT EmployeeId, Name FROM Employees")
for row in cursor:
    print(row.Name)              # attribute access
    print(row[1])                # index access

# Get column metadata
cursor.execute("SELECT * FROM Employees")
columns = [desc[0] for desc in cursor.description]
print(columns)  # ['EmployeeId', 'Name', 'Department', 'Salary']

# Dictionary-like rows
def rows_as_dicts(cursor):
    columns = [desc[0] for desc in cursor.description]
    for row in cursor:
        yield dict(zip(columns, row))

cursor.execute("SELECT * FROM Employees WHERE Department = ?", ("Engineering",))
for emp in rows_as_dicts(cursor):
    print(emp["Name"], emp["Salary"])
```

## Pandas Integration

```python
import pandas as pd
import pyodbc

conn = pyodbc.connect(connection_string)

# Read SQL results into DataFrame
df = pd.read_sql(
    "SELECT * FROM Sales WHERE SaleDate >= ? AND Region = ?",
    conn,
    params=["2025-01-01", "West"],
    parse_dates=["SaleDate"]
)

# Read with chunking for large datasets
chunks = pd.read_sql(
    "SELECT * FROM LargeTable",
    conn,
    chunksize=10000
)
for chunk_df in chunks:
    process(chunk_df)

# Write DataFrame to SQL Server
df.to_sql(
    name="SalesSummary",
    con=engine,  # SQLAlchemy engine required for to_sql
    if_exists="append",    # 'fail', 'replace', or 'append'
    index=False,
    dtype={
        "Amount": sa.types.Numeric(12, 2),
        "Region": sa.types.NVARCHAR(50),
    },
    chunksize=1000,
    method="multi"         # faster batch inserts
)
```

## SQLAlchemy Integration

```python
from sqlalchemy import create_engine, text
import urllib

# Connection string for SQLAlchemy
params = urllib.parse.quote_plus(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=myserver;"
    "DATABASE=mydb;"
    "Trusted_Connection=yes;"
    "Encrypt=yes;"
    "TrustServerCertificate=yes;"
)
engine = create_engine(f"mssql+pyodbc:///?odbc_connect={params}")

# Or with explicit credentials
engine = create_engine(
    "mssql+pyodbc://user:password@myserver/mydb?driver=ODBC+Driver+18+for+SQL+Server"
)

# Using the engine
with engine.connect() as conn:
    result = conn.execute(
        text("SELECT * FROM Employees WHERE Department = :dept"),
        {"dept": "Engineering"}
    )
    for row in result:
        print(row.Name)

# Connection pooling is built into SQLAlchemy
engine = create_engine(
    connection_url,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=3600,  # recycle connections after 1 hour
    pool_pre_ping=True  # test connections before use
)
```

## Error Handling

```python
import pyodbc

try:
    conn = pyodbc.connect(connection_string, timeout=10)
    cursor = conn.cursor()
    cursor.execute("INSERT INTO Orders (OrderId, Amount) VALUES (?, ?)", (1, 99.99))
    conn.commit()
except pyodbc.OperationalError as e:
    # Connection failures, timeouts
    print(f"Connection error: {e}")
except pyodbc.IntegrityError as e:
    # Constraint violations (PK, FK, unique)
    print(f"Integrity error: {e}")
    conn.rollback()
except pyodbc.ProgrammingError as e:
    # SQL syntax errors, invalid object names
    print(f"SQL error: {e}")
    conn.rollback()
except pyodbc.DatabaseError as e:
    # General database errors
    sqlstate = e.args[0] if e.args else "Unknown"
    message = e.args[1] if len(e.args) > 1 else str(e)
    print(f"Database error [{sqlstate}]: {message}")
    conn.rollback()
finally:
    if conn:
        conn.close()

# Context manager pattern (preferred)
with pyodbc.connect(connection_string) as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT 1")
        # connection auto-commits on successful exit
        # rolls back on exception
```

## Best Practices

- Use pyodbc with ODBC Driver 18 for production workloads; fall back to pymssql only when ODBC driver installation is impossible.
- Always use parameterized queries with `?` placeholders — never use string formatting or f-strings for SQL values.
- Enable `fast_executemany = True` on cursors before `executemany()` calls for 10-100x speedup on batch inserts.
- Use SQLAlchemy's connection pooling (`pool_pre_ping=True`) instead of managing raw pyodbc connections in multi-threaded apps.
- Close connections explicitly or use context managers (`with`) to prevent connection leaks.
- Set `autocommit=False` (the default) and call `conn.commit()` explicitly for write operations.
- When upgrading from ODBC Driver 17 to 18, add `TrustServerCertificate=yes` if using self-signed certs, since Driver 18 defaults to `Encrypt=yes`.
- Use `pd.read_sql` with `chunksize` for large result sets to control memory usage.
- Log the SQLSTATE code from pyodbc exceptions for troubleshooting.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| String formatting SQL values | SQL injection vulnerability | Use `?` placeholders with parameter tuples |
| Not enabling `fast_executemany` | Batch inserts 10-100x slower than necessary | Set `cursor.fast_executemany = True` before `executemany()` |
| Using ODBC Driver 18 without `Encrypt` awareness | Connection failures with self-signed certs | Add `TrustServerCertificate=yes` or install proper certs |
| Creating new connections per query | Exhausts server connections, high latency | Use SQLAlchemy connection pool or reuse connections |
| Forgetting `conn.commit()` | Data silently not persisted | Always commit after write operations |
| Using `fetchall()` on million-row queries | Out of memory | Use `fetchmany(size)` or iterate cursor directly |
| Mixing pyodbc and pymssql placeholder styles | `?` (pyodbc) vs `%s` (pymssql) causes errors | Match placeholder to your driver |

## SQL Server Version Notes

- **SQL Server 2016** — Full support from pyodbc/pymssql. JSON functions available via T-SQL.
- **SQL Server 2019** — UTF-8 collation support. ODBC Driver 17.4+ recommended for full compatibility.
- **SQL Server 2022** — ODBC Driver 18 recommended. New T-SQL functions (GREATEST, LEAST, GENERATE_SERIES) work through any driver.
- **ODBC Driver 17** — Encrypt defaults to `no`. Last version to support older TLS. Still supported but in maintenance mode.
- **ODBC Driver 18** — Encrypt defaults to `yes` (breaking change). TLS 1.2 minimum. Recommended for all new deployments.
- **Azure SQL** — Always use ODBC Driver 18 with `Encrypt=yes`. Azure AD token auth requires pyodbc 4.0.30+ and Driver 17.6+/18.

## Sources

- https://learn.microsoft.com/en-us/sql/connect/python/pyodbc/python-sql-driver-pyodbc
- https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server
- https://github.com/mkleehammer/pyodbc/wiki
- https://docs.sqlalchemy.org/en/20/dialects/mssql.html
- https://pymssql.readthedocs.io/en/stable/
- https://pandas.pydata.org/docs/reference/api/pandas.read_sql.html
