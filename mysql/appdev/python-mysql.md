# Python MySQL Connectivity — mysql-connector-python, PyMySQL, and SQLAlchemy Integration

## Overview

Python is one of the most popular languages for MySQL application development, with two primary connector options: `mysql-connector-python` (Oracle's official driver) and `PyMySQL` (a pure-Python implementation). Both conform to the Python DB-API 2.0 specification (PEP 249), making them largely interchangeable at the API level.

Choosing between them depends on your deployment constraints. `mysql-connector-python` includes an optional C extension for performance and supports advanced features like the X DevAPI. `PyMySQL` requires no compiled dependencies, making it ideal for environments where installing C extensions is difficult (e.g., serverless, restricted containers). Both integrate cleanly with SQLAlchemy, the de facto Python ORM/toolkit.

This skill covers connection setup, parameterized queries, connection pooling, cursor strategies, batch operations, error handling, and SQLAlchemy integration for both drivers.

## Key Concepts

- **DB-API 2.0 (PEP 249)**: Python's standard database interface specification. Both drivers implement `connect()`, `cursor()`, `execute()`, `fetchone()`, `fetchall()`, `commit()`, `rollback()`.
- **Parameterized queries**: Bind variables that prevent SQL injection. Use `%s` placeholders (both drivers) or `%(name)s` named placeholders.
- **Connection pooling**: Reusing database connections to avoid the overhead of establishing new ones per request.
- **Buffered cursor**: Fetches all rows from the server immediately. Unbuffered cursor fetches rows on demand, reducing memory usage for large result sets.
- **Autocommit**: By default, both drivers have autocommit disabled. You must call `conn.commit()` explicitly or enable autocommit.

## Installing the Drivers

```bash
# Official Oracle driver
pip install mysql-connector-python

# Pure Python alternative
pip install PyMySQL

# SQLAlchemy (works with both)
pip install SQLAlchemy
```

## Connection Setup

### mysql-connector-python

```python
import mysql.connector

# Basic connection
conn = mysql.connector.connect(
    host="127.0.0.1",
    port=3306,
    user="app_user",
    password="secret",
    database="mydb",
    charset="utf8mb4",
    collation="utf8mb4_unicode_ci",
    connect_timeout=10,
    autocommit=False
)

cursor = conn.cursor()
cursor.execute("SELECT VERSION()")
row = cursor.fetchone()
print(f"MySQL version: {row[0]}")

cursor.close()
conn.close()
```

### PyMySQL

```python
import pymysql

conn = pymysql.connect(
    host="127.0.0.1",
    port=3306,
    user="app_user",
    password="secret",
    database="mydb",
    charset="utf8mb4",
    connect_timeout=10,
    autocommit=False,
    cursorclass=pymysql.cursors.DictCursor  # return rows as dicts
)

with conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT VERSION()")
        row = cursor.fetchone()
        print(f"MySQL version: {row['VERSION()']}")
```

## Parameterized Queries (Preventing SQL Injection)

Never use string formatting or f-strings to build SQL. Always use parameterized queries.

```python
# WRONG - SQL injection vulnerability
cursor.execute(f"SELECT * FROM users WHERE email = '{user_input}'")

# CORRECT - parameterized query (positional)
cursor.execute("SELECT * FROM users WHERE email = %s", (user_input,))

# CORRECT - parameterized query (named) -- mysql-connector-python only
cursor.execute(
    "SELECT * FROM users WHERE email = %(email)s AND status = %(status)s",
    {"email": user_input, "status": "active"}
)

# INSERT with parameters
cursor.execute(
    "INSERT INTO users (name, email, created_at) VALUES (%s, %s, NOW())",
    ("Alice", "alice@example.com")
)
conn.commit()
```

## Connection Pooling

### mysql-connector-python built-in pool

```python
import mysql.connector.pooling

pool_config = {
    "pool_name": "mypool",
    "pool_size": 10,
    "pool_reset_session": True,
    "host": "127.0.0.1",
    "user": "app_user",
    "password": "secret",
    "database": "mydb",
    "charset": "utf8mb4"
}

pool = mysql.connector.pooling.MySQLConnectionPool(**pool_config)

# Get a connection from the pool
conn = pool.get_connection()
try:
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM orders")
    print(cursor.fetchone()[0])
finally:
    conn.close()  # returns connection to pool, does not destroy it
```

### PyMySQL with DBUtils

```python
from dbutils.pooled_db import PooledDB
import pymysql

pool = PooledDB(
    creator=pymysql,
    maxconnections=20,
    mincached=5,
    maxcached=10,
    host="127.0.0.1",
    user="app_user",
    password="secret",
    database="mydb",
    charset="utf8mb4"
)

conn = pool.connection()
try:
    with conn.cursor() as cursor:
        cursor.execute("SELECT 1")
finally:
    conn.close()
```

## Cursor Types

### Buffered vs Unbuffered (mysql-connector-python)

```python
# Buffered cursor (default) -- fetches all rows at once
cursor = conn.cursor(buffered=True)
cursor.execute("SELECT * FROM large_table")
# All rows are already in client memory

# Unbuffered cursor -- rows fetched on demand
cursor = conn.cursor(buffered=False)
cursor.execute("SELECT * FROM large_table")
for row in cursor:
    process(row)  # fetched one at a time from server

# Dictionary cursor -- rows as dicts
cursor = conn.cursor(dictionary=True)
cursor.execute("SELECT id, name FROM users LIMIT 1")
row = cursor.fetchone()  # {'id': 1, 'name': 'Alice'}

# Named tuple cursor
cursor = conn.cursor(named_tuple=True)
cursor.execute("SELECT id, name FROM users LIMIT 1")
row = cursor.fetchone()
print(row.name)  # attribute-style access
```

### PyMySQL cursor classes

```python
# Default tuple cursor
cursor = conn.cursor()

# Dictionary cursor
cursor = conn.cursor(pymysql.cursors.DictCursor)

# Server-side cursor (unbuffered, for large results)
cursor = conn.cursor(pymysql.cursors.SSCursor)

# Server-side dictionary cursor
cursor = conn.cursor(pymysql.cursors.SSDictCursor)
```

## Batch Operations (executemany)

```python
# Insert multiple rows efficiently
records = [
    ("Alice", "alice@example.com"),
    ("Bob", "bob@example.com"),
    ("Charlie", "charlie@example.com"),
]

cursor.executemany(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    records
)
conn.commit()
print(f"Inserted {cursor.rowcount} rows")

# For very large inserts, chunk the data
def chunked_insert(cursor, conn, sql, data, chunk_size=1000):
    for i in range(0, len(data), chunk_size):
        chunk = data[i:i + chunk_size]
        cursor.executemany(sql, chunk)
        conn.commit()
```

## Error Handling

```python
import mysql.connector
from mysql.connector import Error, errorcode

try:
    conn = mysql.connector.connect(host="127.0.0.1", user="app", password="pw", database="mydb")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (email) VALUES (%s)", ("dup@example.com",))
    conn.commit()

except mysql.connector.InterfaceError as err:
    print(f"Connection problem: {err}")

except mysql.connector.DatabaseError as err:
    if err.errno == errorcode.ER_DUP_ENTRY:
        print("Duplicate entry, skipping")
    elif err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
        print("Bad credentials")
    elif err.errno == errorcode.ER_BAD_DB_ERROR:
        print("Database does not exist")
    else:
        print(f"Database error {err.errno}: {err.msg}")
    conn.rollback()

except Exception as err:
    print(f"Unexpected: {err}")
    conn.rollback()

finally:
    if conn.is_connected():
        cursor.close()
        conn.close()
```

## SQLAlchemy Integration

```python
from sqlalchemy import create_engine, text

# Using mysql-connector-python
engine = create_engine(
    "mysql+mysqlconnector://app_user:secret@127.0.0.1:3306/mydb",
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
    pool_pre_ping=True,  # test connections before use
    echo=False
)

# Using PyMySQL
engine = create_engine(
    "mysql+pymysql://app_user:secret@127.0.0.1:3306/mydb?charset=utf8mb4",
    pool_size=10,
    pool_pre_ping=True
)

# Execute queries
with engine.connect() as conn:
    result = conn.execute(text("SELECT * FROM users WHERE status = :status"), {"status": "active"})
    for row in result:
        print(row.name, row.email)

# Transaction block
with engine.begin() as conn:
    conn.execute(text("UPDATE users SET status = :s WHERE id = :id"), {"s": "inactive", "id": 42})
    # auto-commits on exit; auto-rolls-back on exception
```

## Best Practices

- Always use parameterized queries (`%s` placeholders) -- never concatenate user input into SQL strings.
- Use connection pooling in any application that handles concurrent requests (web apps, APIs, workers).
- Set `pool_pre_ping=True` in SQLAlchemy to handle stale connections caused by MySQL's `wait_timeout`.
- Use `utf8mb4` charset (not `utf8`) to support full Unicode including emojis.
- Close cursors and connections in `finally` blocks or use context managers (`with`).
- Use unbuffered/server-side cursors for queries that return millions of rows.
- Set `connect_timeout` and `read_timeout` to avoid hanging on network issues.
- Use `executemany()` for bulk inserts instead of looping over single `execute()` calls.
- Pin your driver version in `requirements.txt` to avoid surprise breakage on upgrades.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| String-formatting SQL with user input | SQL injection vulnerability | Use `%s` parameterized placeholders |
| Forgetting `conn.commit()` after DML | Data never persisted; lost on disconnect | Call `commit()` explicitly or set `autocommit=True` |
| Using `utf8` charset instead of `utf8mb4` | 4-byte characters (emojis) cause errors | Specify `charset="utf8mb4"` in connect params |
| Not closing connections in error paths | Connection leaks exhaust `max_connections` | Use `try/finally` or context managers |
| Using buffered cursor for million-row queries | Entire result set loaded into memory | Use `SSCursor` (PyMySQL) or `buffered=False` |
| Creating a new connection per query in a web app | High latency and connection churn | Use connection pooling |
| Catching generic `Exception` and ignoring it | Hides real errors, data corruption | Catch specific `mysql.connector.Error` subclasses |

## MySQL Version Notes

- **5.7**: Both drivers fully supported. No window functions in SQL, so analytics must be done in Python.
- **8.0**: Both drivers work. `caching_sha2_password` is the default auth plugin; older driver versions may need `mysql_native_password` or driver upgrades. SQLAlchemy 1.4+ recommended.
- **8.4 / 9.x**: `mysql_native_password` removed by default. Ensure driver supports `caching_sha2_password`. `mysql-connector-python` 8.2+ and `PyMySQL` 1.1+ handle this natively.

## Sources

- [mysql-connector-python Developer Guide](https://dev.mysql.com/doc/connector-python/en/)
- [PyMySQL Documentation](https://pymysql.readthedocs.io/)
- [SQLAlchemy MySQL Dialects](https://docs.sqlalchemy.org/en/20/dialects/mysql.html)
- [PEP 249 -- DB-API 2.0](https://peps.python.org/pep-0249/)
