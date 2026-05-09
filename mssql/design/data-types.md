# SQL Server Data Types — Choosing the Right Type for Performance and Correctness

## Overview

Data type selection is one of the most consequential design decisions in SQL Server. The chosen type determines storage size, index width, memory grant requirements, implicit conversion behavior, and ultimately query performance. A poorly chosen data type can silently degrade performance across every query that touches the column.

SQL Server provides a rich type system spanning numeric, string, date/time, binary, spatial, hierarchical, and XML categories. Each type has specific storage characteristics and behavioral nuances. The general principle is to use the narrowest type that accommodates your data domain — narrower types mean narrower indexes, more rows per page, smaller memory grants, and faster scans.

Understanding data types is especially critical when designing clustered index keys (which are included in every nonclustered index), join columns (where implicit conversions cause scan-instead-of-seek problems), and columns that participate in range queries or aggregations. This reference covers every major data type category with storage sizes, common pitfalls, and practical guidance.

## Key Concepts

- **Storage size** — Fixed-length types (INT, CHAR, DATETIME2) always consume their declared size. Variable-length types (VARCHAR, VARBINARY) consume actual data length plus 2 bytes of overhead.
- **Precision vs. scale** — For DECIMAL/NUMERIC, precision is the total number of digits; scale is the number of digits after the decimal point.
- **Implicit conversion** — When SQL Server must convert between types in a comparison or join, it follows data type precedence rules. The lower-precedence type is converted, often preventing index seeks.
- **Unicode** — NCHAR/NVARCHAR store UTF-16 and consume 2 bytes per character (or 4 for supplementary characters with `_SC` collations). VARCHAR uses the database collation's code page at 1 byte per character.
- **MAX types** — VARCHAR(MAX), NVARCHAR(MAX), and VARBINARY(MAX) can store up to 2 GB. Values over 8000 bytes are stored off-row (LOB pages).

## Numeric Types

### Integer Types

```sql
-- Integer types and their ranges
-- TINYINT:   0 to 255                          (1 byte)
-- SMALLINT:  -32,768 to 32,767                 (2 bytes)
-- INT:       -2,147,483,648 to 2,147,483,647   (4 bytes)
-- BIGINT:    -9.2 quintillion to 9.2 quintillion (8 bytes)

-- Choose the narrowest type that fits your domain
CREATE TABLE dbo.Products (
    product_id      INT IDENTITY(1,1) NOT NULL,  -- 4 bytes, good for most PKs
    category_id     SMALLINT NOT NULL,            -- 2 bytes, < 32K categories
    status_code     TINYINT NOT NULL,             -- 1 byte, 0-255 status values
    global_order_id BIGINT NOT NULL               -- 8 bytes, when INT is not enough
);
```

### DECIMAL and NUMERIC

```sql
-- DECIMAL and NUMERIC are functionally identical in SQL Server
-- Storage varies by precision:
--   1-9 digits:   5 bytes
--   10-19 digits: 9 bytes
--   20-28 digits: 13 bytes
--   29-38 digits: 17 bytes

-- Financial amounts: use DECIMAL, not MONEY
CREATE TABLE dbo.Transactions (
    amount        DECIMAL(19,4) NOT NULL,   -- 9 bytes, handles trillions
    tax_rate      DECIMAL(5,4) NOT NULL,    -- 5 bytes, e.g., 0.0825
    exchange_rate DECIMAL(12,6) NOT NULL    -- 9 bytes, 6 decimal places
);

-- Arithmetic precision
SELECT CAST(1 AS DECIMAL(19,4)) / CAST(3 AS DECIMAL(19,4));
-- Returns 0.3333 (truncated to scale 4, no rounding surprise)
```

### MONEY vs DECIMAL

```sql
-- MONEY:     8 bytes, fixed scale of 4, range ~922 trillion
-- SMALLMONEY: 4 bytes, fixed scale of 4, range ~214,748

-- MONEY has a known arithmetic pitfall:
DECLARE @m MONEY = 1.00, @d MONEY = 3.00;
SELECT @m / @d;           -- 0.3333 (correct)
SELECT @m / @d * @d;      -- 0.9999 (not 1.00!)

-- DECIMAL avoids this:
DECLARE @dm DECIMAL(19,4) = 1.00, @dd DECIMAL(19,4) = 3.00;
SELECT @dm / @dd * @dd;   -- 1.000000 (correct with higher intermediate precision)

-- Recommendation: use DECIMAL(19,4) instead of MONEY for financial data
```

### FLOAT and REAL

```sql
-- FLOAT(n): approximate numeric
--   FLOAT(1-24)  = REAL = 4 bytes, ~7 digits precision
--   FLOAT(25-53) = FLOAT = 8 bytes, ~15 digits precision

-- Use for scientific/engineering data only, NEVER for financial
CREATE TABLE dbo.SensorReadings (
    temperature REAL NOT NULL,         -- 4 bytes, ~7 significant digits
    latitude    FLOAT(53) NOT NULL,    -- 8 bytes, ~15 significant digits
    longitude   FLOAT(53) NOT NULL
);

-- Floating-point comparison pitfall
DECLARE @f FLOAT = 0.1 + 0.2;
SELECT CASE WHEN @f = 0.3 THEN 'Equal' ELSE 'Not Equal' END;
-- Returns 'Not Equal' due to binary representation
```

## String Types

### VARCHAR vs NVARCHAR

```sql
-- VARCHAR(n):  1 byte/char, max n = 8000, uses database collation code page
-- NVARCHAR(n): 2 bytes/char, max n = 4000, stores Unicode (UTF-16)
-- CHAR(n):     fixed-length, padded with spaces, 1 byte/char
-- NCHAR(n):    fixed-length, padded with spaces, 2 bytes/char

CREATE TABLE dbo.Customers (
    customer_code  CHAR(10) NOT NULL,         -- fixed format, always 10 chars
    name           NVARCHAR(200) NOT NULL,     -- Unicode for international names
    email          VARCHAR(320) NOT NULL,      -- ASCII-safe, RFC 5321 max
    country_code   CHAR(2) NOT NULL,           -- ISO 3166 alpha-2
    notes          NVARCHAR(MAX) NULL          -- up to 2 GB, stored off-row if > 8000
);

-- Implicit conversion warning: VARCHAR column compared with NVARCHAR literal
-- This prevents index seek:
SELECT * FROM dbo.Customers WHERE email = N'user@example.com';  -- BAD: NVARCHAR literal
SELECT * FROM dbo.Customers WHERE email = 'user@example.com';   -- GOOD: VARCHAR literal
```

### VARCHAR(MAX) Behavior

```sql
-- VARCHAR(MAX) can store up to 2 GB
-- Values <= 8000 bytes are stored in-row by default
-- Values > 8000 bytes are stored in LOB pages (off-row)

-- Control in-row vs off-row storage:
EXEC sp_tableoption 'dbo.Documents', 'large value types out of row', '1';
-- Forces all MAX columns off-row (reduces in-row page density issues)

-- MAX types cannot be used as index key columns
-- But they can be INCLUDEd in nonclustered indexes (SQL Server 2016+)
```

## Date/Time Types

### DATETIME2 vs DATETIME vs DATE

```sql
-- DATE:         3 bytes, 0001-01-01 to 9999-12-31 (date only)
-- TIME(n):      3-5 bytes, 00:00:00.0000000 to 23:59:59.9999999
-- DATETIME2(n): 6-8 bytes, 0001-01-01 to 9999-12-31, 100ns precision
-- DATETIME:     8 bytes, 1753-01-01 to 9999-12-31, 3.33ms precision
-- SMALLDATETIME: 4 bytes, 1900-01-01 to 2079-06-06, 1-minute precision

-- DATETIME2 is the recommended replacement for DATETIME
CREATE TABLE dbo.AuditLog (
    event_date     DATE NOT NULL,              -- 3 bytes, date only
    event_time     TIME(3) NOT NULL,           -- 4 bytes, millisecond precision
    created_at     DATETIME2(3) NOT NULL,      -- 7 bytes, ms precision (vs 8 for DATETIME)
    legacy_column  DATETIME NULL               -- avoid for new designs
);

-- DATETIME rounding surprises
DECLARE @dt DATETIME = '2026-05-09 23:59:59.999';
SELECT @dt;  -- Returns '2026-05-10 00:00:00.000' (rounded up!)

-- DATETIME2 does not round
DECLARE @dt2 DATETIME2(3) = '2026-05-09 23:59:59.999';
SELECT @dt2;  -- Returns '2026-05-09 23:59:59.999' (exact)
```

### DATETIMEOFFSET

```sql
-- DATETIMEOFFSET(n): 8-10 bytes, stores UTC offset with the value
-- Essential for global applications

CREATE TABLE dbo.GlobalEvents (
    event_id    INT IDENTITY PRIMARY KEY,
    occurred_at DATETIMEOFFSET(0) NOT NULL,   -- 8 bytes with second precision
    time_zone   VARCHAR(50) NOT NULL
);

INSERT dbo.GlobalEvents (occurred_at, time_zone)
VALUES ('2026-05-09T14:30:00-07:00', 'America/Los_Angeles');

-- Convert to UTC
SELECT occurred_at AT TIME ZONE 'UTC' AS utc_time
FROM dbo.GlobalEvents;

-- Convert between time zones (SQL Server 2016+)
SELECT occurred_at AT TIME ZONE 'Eastern Standard Time' AS eastern_time
FROM dbo.GlobalEvents;
```

## Uniqueidentifier (GUID)

```sql
-- UNIQUEIDENTIFIER: 16 bytes, stores a GUID
-- Common as PK but has significant performance implications

CREATE TABLE dbo.Sessions (
    session_id UNIQUEIDENTIFIER NOT NULL
        DEFAULT NEWSEQUENTIALID(),   -- sequential GUIDs reduce page splits
    user_id    INT NOT NULL,
    created_at DATETIME2(3) NOT NULL
);

-- NEWID() vs NEWSEQUENTIALID()
-- NEWID():            random GUID — causes massive page splits as clustered key
-- NEWSEQUENTIALID():  sequential GUID — acceptable as clustered key, but still 16 bytes
-- INT IDENTITY:       4 bytes, sequential — best clustered key for most tables

-- GUID as clustered key: 16-byte key in every nonclustered index row
-- INT as clustered key:  4-byte key in every nonclustered index row
-- On a table with 5 nonclustered indexes and 10M rows:
--   GUID overhead: 5 * 10M * 12 extra bytes = 600 MB wasted
```

## XML Type

```sql
-- XML type: stored as LOB, supports XQuery, can be indexed

CREATE TABLE dbo.Configurations (
    config_id   INT IDENTITY PRIMARY KEY,
    config_data XML NOT NULL
);

-- Typed XML with schema collection
CREATE XML SCHEMA COLLECTION dbo.ConfigSchema AS N'
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
  <xs:element name="config">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="setting" maxOccurs="unbounded">
          <xs:complexType>
            <xs:attribute name="key" type="xs:string" use="required"/>
            <xs:attribute name="value" type="xs:string" use="required"/>
          </xs:complexType>
        </xs:element>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>';

ALTER TABLE dbo.Configurations
    ALTER COLUMN config_data XML(dbo.ConfigSchema);

-- Querying XML
SELECT
    config_id,
    config_data.value('(/config/setting[@key="timeout"]/@value)[1]', 'INT') AS timeout_val
FROM dbo.Configurations;

-- XML indexes for performance
CREATE PRIMARY XML INDEX IX_Config_XML ON dbo.Configurations(config_data);
```

## JSON Handling

```sql
-- SQL Server has NO native JSON type — use NVARCHAR(MAX) with JSON functions
-- Validate with ISJSON(), query with JSON_VALUE(), JSON_QUERY(), OPENJSON()

CREATE TABLE dbo.Events (
    event_id   INT IDENTITY PRIMARY KEY,
    event_data NVARCHAR(MAX) NOT NULL
        CONSTRAINT CK_Events_ValidJSON CHECK (ISJSON(event_data) = 1)
);

INSERT dbo.Events (event_data)
VALUES (N'{"type":"click","page":"/home","user_id":42,"tags":["web","mobile"]}');

-- Scalar value extraction
SELECT
    JSON_VALUE(event_data, '$.type') AS event_type,
    JSON_VALUE(event_data, '$.user_id') AS user_id
FROM dbo.Events;

-- Computed column + index for fast JSON field queries
ALTER TABLE dbo.Events
    ADD event_type AS JSON_VALUE(event_data, '$.type');

CREATE INDEX IX_Events_Type ON dbo.Events(event_type);

-- OPENJSON for shredding arrays
SELECT e.event_id, t.value AS tag
FROM dbo.Events e
CROSS APPLY OPENJSON(e.event_data, '$.tags') t;
```

## Spatial Types

```sql
-- GEOMETRY: planar (flat-earth) coordinate system
-- GEOGRAPHY: geodetic (round-earth) with latitude/longitude

CREATE TABLE dbo.Stores (
    store_id  INT IDENTITY PRIMARY KEY,
    name      NVARCHAR(100) NOT NULL,
    location  GEOGRAPHY NOT NULL
);

INSERT dbo.Stores (name, location)
VALUES ('Downtown', geography::Point(37.7749, -122.4194, 4326));  -- lat, lon, SRID

-- Find stores within 10 km of a point
DECLARE @search GEOGRAPHY = geography::Point(37.78, -122.41, 4326);
SELECT name, location.STDistance(@search) / 1000 AS distance_km
FROM dbo.Stores
WHERE location.STDistance(@search) < 10000
ORDER BY location.STDistance(@search);

-- Spatial index for performance
CREATE SPATIAL INDEX SIX_Stores_Location ON dbo.Stores(location);
```

## Other Types

```sql
-- SQL_VARIANT: stores values of various data types (up to 8016 bytes)
-- Use sparingly; cannot be used in LIKE, keys of distributed queries
DECLARE @v SQL_VARIANT = CAST(42 AS INT);
SELECT SQL_VARIANT_PROPERTY(@v, 'BaseType');  -- 'int'

-- HIERARCHYID: compact representation of position in a tree
CREATE TABLE dbo.OrgChart (
    node      HIERARCHYID PRIMARY KEY,
    level_num AS node.GetLevel(),
    emp_name  NVARCHAR(100)
);

INSERT dbo.OrgChart VALUES (hierarchyid::GetRoot(), 'CEO');
INSERT dbo.OrgChart VALUES ('/1/', 'VP Engineering');
INSERT dbo.OrgChart VALUES ('/1/1/', 'Dev Manager');

-- Find all descendants of CEO
SELECT emp_name, node.ToString() AS path
FROM dbo.OrgChart
WHERE node.IsDescendantOf(hierarchyid::GetRoot()) = 1;
```

## Best Practices

- Use the narrowest integer type that fits your domain: TINYINT for status codes, SMALLINT for lookup tables, INT for most PKs, BIGINT only when INT range is genuinely insufficient.
- Use DECIMAL(p,s) instead of MONEY for financial calculations to avoid intermediate rounding errors.
- Use DATETIME2 instead of DATETIME for new designs — smaller storage at equal or better precision, no rounding surprises.
- Use VARCHAR for ASCII-only data (emails, codes, URLs) and NVARCHAR for user-facing text that may contain international characters.
- Avoid NVARCHAR when VARCHAR suffices — doubling column width doubles index width and memory grants.
- Never use FLOAT/REAL for data requiring exact comparison or financial calculations.
- Add a CHECK constraint with `ISJSON()` on NVARCHAR columns storing JSON to enforce validity.
- Use NEWSEQUENTIALID() instead of NEWID() if GUID must be the clustered key, but prefer INT IDENTITY when possible.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using NVARCHAR(MAX) for all string columns | Inflated memory grants, poor cardinality estimates, no inline storage | Size columns appropriately: NVARCHAR(100), VARCHAR(255), etc. |
| FLOAT for financial amounts | Silent rounding errors in comparisons and arithmetic | Use DECIMAL(19,4) for money |
| Comparing VARCHAR column with NVARCHAR literal | Implicit conversion prevents index seek (full scan) | Match literal type to column type; avoid the N prefix on VARCHAR columns |
| DATETIME for high-precision timestamps | 3.33ms rounding; '23:59:59.999' rounds to next day | Use DATETIME2(3) or DATETIME2(7) |
| NEWID() as clustered index key | Random GUIDs cause massive page splits and fragmentation | Use INT IDENTITY or NEWSEQUENTIALID() |
| Using MAX types as join/filter columns | Cannot be index keys; poor query performance | Use fixed-length or bounded-length types for searchable columns |

## SQL Server Version Notes

- **SQL Server 2016** — JSON functions introduced (JSON_VALUE, JSON_QUERY, OPENJSON, FOR JSON). AT TIME ZONE for DATETIMEOFFSET conversion. Temporal tables require DATETIME2 for period columns. DROP IF EXISTS syntax.
- **SQL Server 2017** — STRING_AGG function (reduces need for XML PATH string concatenation). TRIM function. SELECT INTO with filegroup targeting.
- **SQL Server 2019** — UTF-8 collations (`_UTF8` suffix) allow VARCHAR to store Unicode with 1-byte ASCII efficiency and 2-4 bytes for non-ASCII. Reduces storage by up to 50% for predominantly ASCII multilingual data vs NVARCHAR.
- **SQL Server 2022** — JSON_OBJECT and JSON_ARRAY constructors. IS JSON predicate for type testing. GREATEST/LEAST functions. DATE_BUCKET function. DATETRUNC function. Bit manipulation functions. GENERATE_SERIES for sequence generation.

## Sources

- [Data Types (Transact-SQL)](https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-types-transact-sql)
- [decimal and numeric](https://learn.microsoft.com/en-us/sql/t-sql/data-types/decimal-and-numeric-transact-sql)
- [datetime2](https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetime2-transact-sql)
- [JSON Data in SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server)
- [Spatial Data Types Overview](https://learn.microsoft.com/en-us/sql/relational-databases/spatial/spatial-data-types-overview)
- [nchar and nvarchar](https://learn.microsoft.com/en-us/sql/t-sql/data-types/nchar-and-nvarchar-transact-sql)
