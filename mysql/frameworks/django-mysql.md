# Django MySQL — Database Configuration, Model Fields, and MySQL-Specific Features

## Overview

Django provides first-class MySQL support through the `django.db.backends.mysql` backend. While Django's ORM abstracts most database differences, MySQL has unique behaviors around character sets, strict mode, JSON support, and storage engines that require MySQL-specific configuration to get right in production.

Django works with MySQL through either `mysqlclient` (recommended, C extension) or `PyMySQL` (pure Python fallback). The choice affects performance and compatibility. Beyond the built-in backend, the `django-mysql` package extends Django with MySQL-specific model fields, database functions, and utilities that take advantage of MySQL features not covered by Django's cross-database ORM.

This skill covers DATABASES configuration, MySQL-specific model patterns, JSONField handling, read replica routing, connection pooling, and the django-mysql extension package.

## Key Concepts

- **Backend**: The `django.db.backends.mysql` module that translates Django ORM operations to MySQL SQL.
- **Strict mode**: MySQL's `STRICT_TRANS_TABLES` mode that Django enables by default (prevents silent data truncation).
- **mysqlclient**: The recommended MySQL driver for Django (C extension wrapping libmysqlclient).
- **Database router**: A Django class that directs read/write queries to different database connections.
- **django-mysql**: Third-party package adding MySQL-specific fields, locks, and query optimizations.

## DATABASES Configuration

### Basic configuration

```python
# settings.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "myapp",
        "USER": "django_user",
        "PASSWORD": "secret",
        "HOST": "db-primary.internal",
        "PORT": "3306",
        "OPTIONS": {
            "charset": "utf8mb4",
            "init_command": "SET sql_mode='STRICT_TRANS_TABLES'",
            "connect_timeout": 10,
            "read_timeout": 30,
            "write_timeout": 30,
        },
        "CONN_MAX_AGE": 600,       # Keep connections alive for 10 minutes
        "CONN_HEALTH_CHECKS": True, # Django 4.1+: test before reuse
    }
}
```

### Using PyMySQL instead of mysqlclient

```python
# In manage.py or wsgi.py, before Django loads:
import pymysql
pymysql.install_as_MySQLdb()
```

### Read/write split configuration

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "myapp",
        "HOST": "db-primary.internal",
        "USER": "django_user",
        "PASSWORD": "secret",
        "OPTIONS": {"charset": "utf8mb4"},
    },
    "replica": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "myapp",
        "HOST": "db-replica.internal",
        "USER": "django_readonly",
        "PASSWORD": "secret",
        "OPTIONS": {"charset": "utf8mb4"},
        "TEST": {"MIRROR": "default"},  # Use default DB in tests
    },
}
```

### Database router for read replicas

```python
# routers.py
class PrimaryReplicaRouter:
    """Route reads to replica, writes to primary."""

    def db_for_read(self, model, **hints):
        return "replica"

    def db_for_write(self, model, **hints):
        return "default"

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == "default"

# settings.py
DATABASE_ROUTERS = ["myapp.routers.PrimaryReplicaRouter"]
```

## MySQL-Specific Model Fields

### CharField vs TextField

```python
from django.db import models

class Article(models.Model):
    # VARCHAR(255) — indexable, fixed max length
    title = models.CharField(max_length=255)

    # LONGTEXT — for large content, cannot be part of an index key
    body = models.TextField()

    # MySQL ENUM via CharField with choices
    status = models.CharField(
        max_length=20,
        choices=[
            ("draft", "Draft"),
            ("published", "Published"),
            ("archived", "Archived"),
        ],
        default="draft",
        db_index=True,
    )

    class Meta:
        # MySQL-specific table options
        db_table = "articles"
        managed = True
```

### JSONField

```python
from django.db import models

class UserProfile(models.Model):
    user = models.OneToOneField("auth.User", on_delete=models.CASCADE)

    # Django 3.1+ JSONField (native MySQL 5.7.8+ JSON column)
    preferences = models.JSONField(default=dict, blank=True)
    addresses = models.JSONField(default=list, blank=True)

    class Meta:
        db_table = "user_profiles"

# Querying JSON fields
UserProfile.objects.filter(preferences__theme="dark")
UserProfile.objects.filter(preferences__notifications__email=True)
UserProfile.objects.filter(addresses__0__city="Seattle")

# JSON containment (MySQL 8.0+)
from django.db.models import Q
UserProfile.objects.filter(
    preferences__contains={"theme": "dark"}
)
```

### Auto-incrementing primary keys

```python
class Order(models.Model):
    # Django default: id = AutoField (INT AUTO_INCREMENT)
    # For large tables, use BigAutoField:
    id = models.BigAutoField(primary_key=True)

    customer = models.ForeignKey("Customer", on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "orders"
        # Use index_together for composite indexes (deprecated in 4.2+, use indexes)
        indexes = [
            models.Index(fields=["customer", "-created_at"], name="idx_cust_created"),
            models.Index(fields=["status", "created_at"], name="idx_status_created"),
        ]
```

### Setting default auto field globally

```python
# settings.py — Use BigAutoField for all new models (Django 3.2+)
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

## Migrations with MySQL

### Creating and running migrations

```bash
# Generate migration from model changes
python manage.py makemigrations myapp

# Show SQL without applying
python manage.py sqlmigrate myapp 0001

# Apply migrations
python manage.py migrate

# Show migration status
python manage.py showmigrations
```

### MySQL-specific migration operations

```python
# migrations/0005_add_fulltext_index.py
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [("myapp", "0004_auto")]

    operations = [
        # Raw SQL for MySQL-specific DDL
        migrations.RunSQL(
            sql="ALTER TABLE articles ADD FULLTEXT INDEX ft_title_body (title, body)",
            reverse_sql="ALTER TABLE articles DROP INDEX ft_title_body",
        ),
        # Add column with ALGORITHM=INSTANT hint
        migrations.RunSQL(
            sql=(
                "ALTER TABLE orders ADD COLUMN notes TEXT NULL "
                "AFTER total, ALGORITHM=INSTANT"
            ),
            reverse_sql="ALTER TABLE orders DROP COLUMN notes",
            state_operations=[
                migrations.AddField(
                    model_name="order",
                    name="notes",
                    field=models.TextField(null=True, blank=True),
                ),
            ],
        ),
    ]
```

## django-mysql Package

### Installation and setup

```bash
pip install django-mysql
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    "django_mysql",
]
```

### MySQL-specific fields

```python
from django.db import models
from django_mysql.models import (
    ListCharField,
    SetCharField,
    ListTextField,
    SetTextField,
    SizedBinaryField,
    SizedTextField,
    Bit1BooleanField,
    NullBit1BooleanField,
)

class Event(models.Model):
    name = models.CharField(max_length=255)

    # Comma-separated list stored as VARCHAR
    tags = ListCharField(
        base_field=models.CharField(max_length=50),
        size=10,             # Max 10 items
        max_length=550,      # Total VARCHAR length
    )

    # MySQL SET type stored as VARCHAR
    categories = SetCharField(
        base_field=models.CharField(max_length=30),
        size=5,
        max_length=160,
    )

    # Efficient boolean using BIT(1)
    is_active = Bit1BooleanField(default=True)

    class Meta:
        db_table = "events"

# Querying ListCharField
Event.objects.filter(tags__contains="python")
Event.objects.filter(tags__len=3)

# Querying SetCharField
Event.objects.filter(categories__contains="tech")
```

### Django-mysql locks

```python
from django_mysql.locks import Lock

# Named lock (MySQL GET_LOCK / RELEASE_LOCK)
with Lock("import-users", acquire_timeout=10):
    # Only one process can hold this lock
    import_users_from_csv()

# Check if lock is held
lock = Lock("import-users")
if lock.is_held():
    print("Import already in progress")
```

### Query optimization

```python
from django_mysql.models import QuerySetMixin

class UserQuerySet(QuerySetMixin, models.QuerySet):
    pass

class User(models.Model):
    objects = UserQuerySet.as_manager()
    ...

# Use pt-visual-explain-compatible EXPLAIN
qs = User.objects.filter(is_active=True)
print(qs.pt_visual_explain())

# Straight join hint
User.objects.straight_join().filter(orders__total__gt=100)

# SQL_NO_CACHE hint
User.objects.sql_no_cache().filter(is_active=True)
```

## Connection Pooling

### Built-in persistent connections (Django 4.1+)

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        ...
        "CONN_MAX_AGE": 600,        # Reuse connections for 10 minutes
        "CONN_HEALTH_CHECKS": True,  # Verify connection before reuse
    }
}
```

### django-db-connection-pool

```bash
pip install django-db-connection-pool[mysql]
```

```python
DATABASES = {
    "default": {
        "ENGINE": "dj_db_conn_pool.backends.mysql",
        "NAME": "myapp",
        "HOST": "db-primary.internal",
        "POOL_OPTIONS": {
            "POOL_SIZE": 20,
            "MAX_OVERFLOW": 10,
            "RECYCLE": 3600,
            "PRE_PING": True,
        },
    }
}
```

## Raw SQL and Database Functions

```python
from django.db import connection
from django.db.models import Func, CharField

# Raw SQL with parameterized queries
with connection.cursor() as cursor:
    cursor.execute(
        "SELECT * FROM users WHERE created_at > %s LIMIT %s",
        [start_date, limit],
    )
    columns = [col[0] for col in cursor.description]
    results = [dict(zip(columns, row)) for row in cursor.fetchall()]

# Custom MySQL function
class GroupConcat(Func):
    function = "GROUP_CONCAT"
    template = "%(function)s(%(distinct)s%(expressions)s%(ordering)s%(separator)s)"

    def __init__(self, expression, distinct=False, ordering=None, separator=",", **extra):
        super().__init__(
            expression,
            distinct="DISTINCT " if distinct else "",
            ordering=f" ORDER BY {ordering}" if ordering else "",
            separator=f" SEPARATOR '{separator}'" if separator else "",
            output_field=CharField(),
            **extra,
        )
```

## Best Practices

- Use `mysqlclient` over `PyMySQL` for production workloads (2-5x faster).
- Set `DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"` in all new projects.
- Always specify `charset: utf8mb4` in OPTIONS to support full Unicode.
- Enable `CONN_HEALTH_CHECKS = True` (Django 4.1+) to handle stale connections.
- Use database routers to split reads to replicas for read-heavy applications.
- Use `RunSQL` migrations with `ALGORITHM=INSTANT` or `ALGORITHM=INPLACE` for large table changes.
- Always include `state_operations` when using `RunSQL` so Django's migration state stays consistent.
- Use `select_related()` and `prefetch_related()` to minimize N+1 queries against MySQL.
- Set `CONN_MAX_AGE` to a value less than MySQL's `wait_timeout`.
- Use `django-mysql` `Lock` for distributed locking instead of file-based or Redis-based locks when MySQL is already available.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not setting `charset: utf8mb4` in OPTIONS | Emoji and 4-byte Unicode characters cause `Incorrect string value` errors | Add `"charset": "utf8mb4"` to DATABASES OPTIONS |
| Using `AutoField` on high-volume tables | INT overflow at ~2.1 billion rows; application crashes | Use `BigAutoField` or set `DEFAULT_AUTO_FIELD` globally |
| Setting `CONN_MAX_AGE = None` (infinite) | Connections persist forever, potentially holding stale transactions or locks | Set `CONN_MAX_AGE = 600` with `CONN_HEALTH_CHECKS = True` |
| Running `makemigrations` in production | Auto-generated migration may include unintended model state changes | Run `makemigrations` in dev, commit the files, and only `migrate` in production |
| Using `TextField` where `CharField` is needed for indexing | MySQL cannot create full-length index on TEXT columns; migration fails with key length error | Use `CharField(max_length=N)` for indexed columns |

## MySQL Version Notes

- **5.7**: JSONField requires MySQL 5.7.8+. No JSON_TABLE or multi-valued indexes. `utf8mb4` available but server default is usually `latin1` or `utf8mb3`.
- **8.0**: Full JSONField support including lookups, containment, and key transforms. `caching_sha2_password` default may require `mysqlclient>=2.0`. Atomic DDL for safer migrations. Default charset is `utf8mb4`.
- **8.4/9.x**: `mysql_native_password` removed by default; ensure `mysqlclient>=2.2` or `PyMySQL>=1.1`. CHECK constraints fully enforced (Django's `CheckConstraint` works correctly).

## Sources

- [Django MySQL Notes](https://docs.djangoproject.com/en/5.0/ref/databases/#mysql-notes)
- [django-mysql Documentation](https://django-mysql.readthedocs.io/)
- [Django Database Routers](https://docs.djangoproject.com/en/5.0/topics/db/multi-db/#database-routers)
- [mysqlclient Documentation](https://mysqlclient.readthedocs.io/)
- [Django Connection Pooling](https://docs.djangoproject.com/en/5.0/ref/databases/#persistent-database-connections)
