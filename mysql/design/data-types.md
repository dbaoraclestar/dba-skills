# MySQL Data Types — Choosing, Using, and Avoiding Pitfalls

## Overview

Choosing the right data type is one of the most consequential decisions in MySQL
schema design. Data types determine storage requirements, query performance, index
efficiency, and data integrity. An incorrect choice can lead to silent data
truncation, wasted storage, timezone bugs, or inability to store valid data.

MySQL provides numeric, string, date/time, JSON, binary, spatial, and bit data
types. InnoDB stores rows in 16KB pages, so smaller data types mean more rows per
page, better cache efficiency, and faster full-table scans. The general principle
is to use the smallest type that can hold your data with room for growth.

Use this reference when designing new tables, reviewing existing schemas for
optimization, migrating between database engines, or diagnosing data quality
issues related to type mismatches.

## Key Concepts

- **Fixed-Length vs Variable-Length**: CHAR is fixed; VARCHAR is variable. Fixed
  types are faster for comparisons but waste space when values vary in length.
- **Signed vs Unsigned**: Numeric types are signed by default. `UNSIGNED` doubles
  the positive range but prevents negative values.
- **Precision and Scale**: DECIMAL(M,D) stores exact values; M is total digits,
  D is digits after the decimal point.
- **Fractional Seconds Precision (FSP)**: DATETIME(N) and TIMESTAMP(N) support
  0-6 digits of fractional seconds (microsecond precision).
- **Implicit Conversion**: MySQL silently converts types in comparisons and
  assignments, which can cause unexpected results and prevent index usage.

## Numeric Types

### Integer Types

| Type        | Bytes | Signed Range                        | Unsigned Range          |
|-------------|-------|--------------------------------------|-------------------------|
| TINYINT     | 1     | -128 to 127                          | 0 to 255               |
| SMALLINT    | 2     | -32,768 to 32,767                    | 0 to 65,535            |
| MEDIUMINT   | 3     | -8,388,608 to 8,388,607             | 0 to 16,777,215        |
| INT         | 4     | -2,147,483,648 to 2,147,483,647     | 0 to 4,294,967,295     |
| BIGINT      | 8     | -9.2e18 to 9.2e18                    | 0 to 18.4e18           |

```sql
-- Auto-increment primary key (use BIGINT UNSIGNED for high-volume tables)
CREATE TABLE orders (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    status     TINYINT UNSIGNED NOT NULL DEFAULT 0,
    quantity   SMALLINT UNSIGNED NOT NULL,
    amount_cents INT NOT NULL
);

-- Boolean (MySQL has no true boolean; TINYINT(1) is conventional)
CREATE TABLE features (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    is_enabled TINYINT(1) NOT NULL DEFAULT 0
);
-- Note: BOOL and BOOLEAN are aliases for TINYINT(1)
```

### DECIMAL vs FLOAT/DOUBLE

```sql
-- DECIMAL: exact fixed-point arithmetic (use for money, quantities)
CREATE TABLE products (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    price DECIMAL(10,2) NOT NULL,       -- up to 99,999,999.99
    weight DECIMAL(8,3) NOT NULL         -- up to 99,999.999
);

-- FLOAT/DOUBLE: approximate floating-point (use for scientific data)
CREATE TABLE measurements (
    id        INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    latitude  DOUBLE NOT NULL,           -- 8 bytes, ~15 significant digits
    longitude DOUBLE NOT NULL,
    temp_c    FLOAT NOT NULL             -- 4 bytes, ~7 significant digits
);

-- NEVER use FLOAT/DOUBLE for financial calculations
-- Example of floating-point imprecision:
SELECT CAST(0.1 + 0.2 AS DECIMAL(10,2));  -- 0.30 (exact)
SELECT 0.1e0 + 0.2e0;                      -- 0.30000000000000004 (approximate)
```

### Storage for DECIMAL

DECIMAL storage depends on the number of digits. Each group of 9 digits takes 4 bytes.

| Digits | Bytes |
|--------|-------|
| 1-2    | 1     |
| 3-4    | 2     |
| 5-6    | 3     |
| 7-9    | 4     |

## String Types

### VARCHAR vs CHAR

```sql
-- CHAR: fixed-length, right-padded with spaces. Use for fixed-size codes.
CREATE TABLE countries (
    code  CHAR(2) NOT NULL PRIMARY KEY,        -- ISO 3166 alpha-2
    code3 CHAR(3) NOT NULL UNIQUE,             -- ISO 3166 alpha-3
    name  VARCHAR(100) NOT NULL
);

-- VARCHAR: variable-length, 1-2 byte length prefix.
-- Max effective length depends on character set and row format.
CREATE TABLE users (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email    VARCHAR(255) NOT NULL,
    bio      VARCHAR(1000) DEFAULT NULL
);

-- VARCHAR storage: data + 1 byte (if max <= 255) or 2 bytes (if max > 255)
-- VARCHAR(255) uses 1 byte prefix; VARCHAR(256) uses 2 bytes prefix
```

### TEXT Variants

| Type       | Max Length   | Storage               |
|------------|-------------|------------------------|
| TINYTEXT   | 255 bytes   | Length + 1 byte prefix |
| TEXT       | 64 KB       | Length + 2 byte prefix |
| MEDIUMTEXT | 16 MB       | Length + 3 byte prefix |
| LONGTEXT   | 4 GB        | Length + 4 byte prefix |

```sql
-- TEXT columns cannot have default values (before 8.0.13)
-- TEXT columns are stored off-page if they exceed the inline threshold
CREATE TABLE articles (
    id      INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title   VARCHAR(255) NOT NULL,
    body    MEDIUMTEXT NOT NULL,
    summary TEXT DEFAULT NULL
);

-- INDEX on TEXT requires a prefix length
CREATE INDEX idx_body ON articles (body(100));
```

### ENUM and SET

```sql
-- ENUM: one value from a predefined list. Stored as 1-2 bytes internally.
CREATE TABLE tickets (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    priority ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
    status   ENUM('open', 'in_progress', 'resolved', 'closed') NOT NULL DEFAULT 'open'
);

-- SET: zero or more values from a predefined list. Stored as 1-8 bytes.
CREATE TABLE user_preferences (
    user_id     INT UNSIGNED NOT NULL PRIMARY KEY,
    notifications SET('email', 'sms', 'push', 'slack') NOT NULL DEFAULT ''
);

-- Querying SET columns
SELECT * FROM user_preferences WHERE FIND_IN_SET('email', notifications);
```

## Date and Time Types

### DATETIME vs TIMESTAMP

| Feature       | DATETIME                | TIMESTAMP                     |
|---------------|-------------------------|-------------------------------|
| Range         | 1000-01-01 to 9999-12-31 | 1970-01-01 to 2038-01-19     |
| Storage       | 5 bytes (+ FSP)         | 4 bytes (+ FSP)              |
| Timezone      | Stores as-is            | Converts to UTC on store      |
| Default NULL  | Yes                     | Yes (varies by sql_mode)      |
| Auto-init     | Via DEFAULT/ON UPDATE   | Via DEFAULT/ON UPDATE         |

```sql
CREATE TABLE events (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    event_name  VARCHAR(100) NOT NULL,
    -- DATETIME: stores the literal value (no timezone conversion)
    event_date  DATETIME NOT NULL,
    -- TIMESTAMP: stored as UTC, displayed in session timezone
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Fractional seconds (0-6 digits)
CREATE TABLE high_precision_log (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    logged_at  DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    message    VARCHAR(500) NOT NULL
);

-- FSP storage overhead: 0=0 bytes, 1-2=1 byte, 3-4=2 bytes, 5-6=3 bytes
```

### Other Date/Time Types

```sql
-- DATE: date only, 3 bytes
-- TIME: time only (or duration), 3 bytes + FSP
-- YEAR: 1-byte year value (1901-2155)

CREATE TABLE employees (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    hire_date  DATE NOT NULL,
    birth_year YEAR NOT NULL,
    shift_start TIME NOT NULL,
    shift_end   TIME NOT NULL
);
```

## JSON Type (5.7.8+)

```sql
CREATE TABLE configurations (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    settings JSON NOT NULL
);

-- Insert JSON data
INSERT INTO configurations (name, settings)
VALUES ('app', '{"theme": "dark", "lang": "en", "features": ["search", "export"]}');

-- Query JSON fields
SELECT name,
       settings->>'$.theme' AS theme,
       settings->>'$.lang' AS lang,
       JSON_LENGTH(settings->'$.features') AS feature_count
FROM configurations;

-- JSON_TABLE (8.0+): Extract JSON array to rows
SELECT c.name, f.feature
FROM configurations c,
     JSON_TABLE(c.settings, '$.features[*]'
       COLUMNS(feature VARCHAR(50) PATH '$')
     ) AS f;

-- Generated column + index for fast JSON lookups
ALTER TABLE configurations
  ADD COLUMN theme VARCHAR(20) GENERATED ALWAYS AS (settings->>'$.theme') STORED,
  ADD INDEX idx_theme (theme);

-- Multi-valued index on JSON array (8.0.17+)
CREATE TABLE products (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    tags JSON NOT NULL
);
CREATE INDEX idx_tags ON products ((CAST(tags->'$[*]' AS CHAR(50) ARRAY)));
SELECT * FROM products WHERE JSON_CONTAINS(tags, '"electronics"');
```

## Binary Types

```sql
-- BINARY/VARBINARY: byte strings (not character strings)
CREATE TABLE sessions (
    session_id BINARY(16) NOT NULL PRIMARY KEY,  -- UUID stored as bytes
    user_id    INT UNSIGNED NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Store UUID efficiently as BINARY(16) instead of CHAR(36)
INSERT INTO sessions (session_id, user_id)
VALUES (UUID_TO_BIN(UUID(), TRUE), 42);
-- UUID_TO_BIN with swap_flag=TRUE improves index locality (8.0+)

SELECT BIN_TO_UUID(session_id, TRUE) AS session_uuid, user_id
FROM sessions;

-- BLOB variants: same size tiers as TEXT
-- TINYBLOB (255), BLOB (64KB), MEDIUMBLOB (16MB), LONGBLOB (4GB)
```

## BIT Type

```sql
-- BIT(N): stores N-bit values (1-64 bits)
CREATE TABLE permissions (
    user_id INT UNSIGNED NOT NULL PRIMARY KEY,
    flags   BIT(8) NOT NULL DEFAULT b'00000000'
    -- bit 0: read, bit 1: write, bit 2: delete, bit 3: admin
);

INSERT INTO permissions (user_id, flags) VALUES (1, b'00001011');

-- Check a specific bit
SELECT user_id, flags,
       flags & b'00000001' AS can_read,
       flags & b'00000010' AS can_write,
       flags & b'00001000' AS is_admin
FROM permissions;
```

## Spatial Types

```sql
-- POINT, LINESTRING, POLYGON, GEOMETRY, etc.
CREATE TABLE locations (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    coord POINT NOT NULL SRID 4326,
    SPATIAL INDEX (coord)
);

INSERT INTO locations (name, coord)
VALUES ('Office', ST_GeomFromText('POINT(-122.4194 37.7749)', 4326));

-- Find locations within 10km
SELECT name, ST_Distance_Sphere(coord, ST_GeomFromText('POINT(-122.4 37.8)', 4326)) AS dist_m
FROM locations
HAVING dist_m < 10000
ORDER BY dist_m;
```

## Storage Requirements Summary

| Type          | Storage    | Notes                          |
|---------------|------------|--------------------------------|
| TINYINT       | 1 byte     |                                |
| SMALLINT      | 2 bytes    |                                |
| MEDIUMINT     | 3 bytes    |                                |
| INT           | 4 bytes    |                                |
| BIGINT        | 8 bytes    |                                |
| FLOAT         | 4 bytes    | Approximate                    |
| DOUBLE        | 8 bytes    | Approximate                    |
| DECIMAL(M,D)  | Variable   | ~4 bytes per 9 digits          |
| CHAR(N)       | N * charset bytes | Fixed, right-padded       |
| VARCHAR(N)    | Len + 1-2  | Variable                       |
| DATE          | 3 bytes    |                                |
| TIME          | 3 + FSP    |                                |
| DATETIME      | 5 + FSP    |                                |
| TIMESTAMP     | 4 + FSP    |                                |
| JSON          | Variable   | Stored as binary, max 1GB      |
| BINARY(N)     | N bytes    | Fixed                          |
| VARBINARY(N)  | Len + 1-2  | Variable                       |

## Best Practices

- Use the smallest integer type that fits your range. Do not default to INT when
  TINYINT or SMALLINT suffices.
- Use BIGINT UNSIGNED for auto-increment primary keys on high-volume tables to
  avoid running out of ID space.
- Always use DECIMAL for financial amounts, never FLOAT or DOUBLE.
- Prefer VARCHAR over CHAR unless values are truly fixed-length (country codes,
  MD5 hashes stored as hex).
- Use DATETIME instead of TIMESTAMP for dates that must survive timezone changes
  or extend beyond 2038.
- Store UUIDs as BINARY(16) with UUID_TO_BIN(uuid, TRUE) for compact storage and
  better index performance.
- Choose utf8mb4 as the default character set for all VARCHAR/TEXT columns to
  support the full Unicode range including emoji.
- Avoid ENUM for values that change frequently — adding/removing values requires
  ALTER TABLE.
- Add fractional seconds precision (DATETIME(3) or DATETIME(6)) when sub-second
  accuracy matters.
- Prefer JSON type over serialized TEXT for structured data — it validates on
  insert and supports indexed access via generated columns.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using FLOAT/DOUBLE for monetary values | Floating-point rounding errors accumulate; financial calculations become inaccurate | Use DECIMAL(M,D) for all financial data |
| Using INT for auto-increment on billion-row tables | Reaches 2.1B signed max and inserts fail with duplicate key error | Use BIGINT UNSIGNED (18.4 quintillion max) |
| Using TIMESTAMP for dates beyond 2038 | Values wrap around or become 0000-00-00 when exceeding 2038-01-19 03:14:07 UTC | Use DATETIME for long-lived data |
| Storing UUIDs as CHAR(36) | 36 bytes vs 16 bytes; poor index locality due to random ordering | Use BINARY(16) with UUID_TO_BIN(uuid, TRUE) |
| Using utf8 (utf8mb3) instead of utf8mb4 | Cannot store emoji or supplementary Unicode characters; data silently truncated | Set character set to utf8mb4 everywhere |
| Declaring VARCHAR(4000) when max value is 50 chars | Memory temp tables allocate the declared max length, not actual; wastes memory | Size VARCHAR to realistic maximum |
| Using ENUM with dozens of values | Hard to maintain; ALTER TABLE required for changes; ordering is by index not alphabetically | Use a lookup/reference table with a foreign key instead |

## MySQL Version Notes

- **5.7**: JSON type introduced in 5.7.8. No multi-valued indexes on JSON arrays.
  Generated columns available (STORED and VIRTUAL). TIMESTAMP still has implicit
  DEFAULT CURRENT_TIMESTAMP behavior depending on `explicit_defaults_for_timestamp`.
  INT display width (e.g., INT(11)) is cosmetic only.
- **8.0**: Multi-valued indexes on JSON arrays (8.0.17). UUID_TO_BIN/BIN_TO_UUID
  functions added. INT display width deprecated (INT(11) display width is ignored
  and generates warnings). DEFAULT expressions allowed on BLOB/TEXT (8.0.13).
  Functional indexes for expressions. CHECK constraints enforced.
  `explicit_defaults_for_timestamp` defaults to ON.
- **8.4 / 9.x**: INT display width fully removed. Continued JSON improvements.
  Check release notes for new type features or deprecations.

## Sources

- [MySQL 8.0 Reference Manual: Data Types](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)
- [MySQL 8.0: Numeric Type Storage](https://dev.mysql.com/doc/refman/8.0/en/storage-requirements.html)
- [MySQL 8.0: The JSON Data Type](https://dev.mysql.com/doc/refman/8.0/en/json.html)
- [MySQL 8.0: Date and Time Types](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-types.html)
- [MySQL 8.0: The ENUM Type](https://dev.mysql.com/doc/refman/8.0/en/enum.html)
