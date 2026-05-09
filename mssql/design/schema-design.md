# Schema Design — Normalization, Index Strategy, and Modern Table Features

## Overview

Schema design in SQL Server encompasses table structure, normalization level, primary key selection, index strategy, constraint design, and the use of modern features like temporal tables and computed columns. Good schema design balances data integrity, query performance, storage efficiency, and developer productivity.

The foundation of relational schema design is normalization — structuring tables to eliminate redundancy and enforce data integrity through constraints. However, production systems often require strategic denormalization for read-heavy workloads. Understanding the trade-offs between normalized and denormalized schemas, and knowing when to apply each, is essential for building systems that scale.

Beyond normalization, the choice of clustered index key is the single most impactful design decision for a SQL Server table. The clustered index determines the physical sort order of data and its key is embedded in every nonclustered index. A poor clustered key choice cascades into wasted storage, increased fragmentation, excessive page splits, and degraded query performance across the entire table.

## Key Concepts

- **Normalization** — The process of structuring tables to reduce data redundancy and prevent update anomalies. Measured in normal forms (1NF through 5NF; 3NF is the practical target).
- **Denormalization** — Intentionally adding redundancy to improve read performance, typically in reporting or OLAP contexts.
- **Clustered index** — Determines physical row order. One per table. The NUSE principle: Narrow, Unique, Static, Ever-increasing.
- **Surrogate key** — An artificial key (IDENTITY, SEQUENCE, GUID) with no business meaning. Preferred for clustered keys.
- **Natural key** — A key composed of business data (SSN, email, product code). Used for uniqueness constraints but rarely optimal as clustered key.
- **Covering index** — A nonclustered index that includes all columns needed by a query, eliminating key lookups.

## Normalization

### First Normal Form (1NF)

```sql
-- VIOLATION: repeating groups / multi-valued columns
CREATE TABLE dbo.Orders_Bad (
    order_id   INT PRIMARY KEY,
    customer   VARCHAR(100),
    products   VARCHAR(MAX)   -- "Widget,Gadget,Sprocket" — NOT 1NF
);

-- 1NF: atomic values, no repeating groups
CREATE TABLE dbo.Orders (
    order_id    INT IDENTITY PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE NOT NULL
);

CREATE TABLE dbo.OrderItems (
    order_item_id INT IDENTITY PRIMARY KEY,
    order_id      INT NOT NULL REFERENCES dbo.Orders(order_id),
    product_id    INT NOT NULL,
    quantity      INT NOT NULL,
    unit_price    DECIMAL(19,4) NOT NULL
);
```

### Second Normal Form (2NF)

```sql
-- VIOLATION: non-key column depends on only part of composite PK
CREATE TABLE dbo.CourseEnrollments_Bad (
    student_id   INT NOT NULL,
    course_id    INT NOT NULL,
    student_name VARCHAR(100),   -- depends only on student_id, NOT on course_id
    grade        CHAR(2),
    PRIMARY KEY (student_id, course_id)
);

-- 2NF: every non-key column depends on the ENTIRE primary key
CREATE TABLE dbo.Students (
    student_id   INT IDENTITY PRIMARY KEY,
    student_name VARCHAR(100) NOT NULL
);

CREATE TABLE dbo.CourseEnrollments (
    student_id INT NOT NULL REFERENCES dbo.Students(student_id),
    course_id  INT NOT NULL REFERENCES dbo.Courses(course_id),
    grade      CHAR(2) NULL,
    PRIMARY KEY (student_id, course_id)
);
```

### Third Normal Form (3NF)

```sql
-- VIOLATION: transitive dependency (city depends on zip, not on PK)
CREATE TABLE dbo.Customers_Bad (
    customer_id INT IDENTITY PRIMARY KEY,
    name        VARCHAR(100),
    zip_code    CHAR(5),
    city        VARCHAR(100),   -- city depends on zip_code, not customer_id
    state       CHAR(2)         -- state depends on zip_code, not customer_id
);

-- 3NF: remove transitive dependencies
CREATE TABLE dbo.ZipCodes (
    zip_code CHAR(5) PRIMARY KEY,
    city     VARCHAR(100) NOT NULL,
    state    CHAR(2) NOT NULL
);

CREATE TABLE dbo.Customers (
    customer_id INT IDENTITY PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    zip_code    CHAR(5) NOT NULL REFERENCES dbo.ZipCodes(zip_code)
);
```

## Denormalization Trade-offs

```sql
-- Scenario: reporting query joins 5 tables, runs 500 times/day
-- Normalized query:
SELECT c.name, p.product_name, SUM(oi.quantity * oi.unit_price) AS total
FROM dbo.Orders o
JOIN dbo.OrderItems oi ON o.order_id = oi.order_id
JOIN dbo.Customers c ON o.customer_id = c.customer_id
JOIN dbo.Products p ON oi.product_id = p.product_id
JOIN dbo.Categories cat ON p.category_id = cat.category_id
WHERE o.order_date >= '2026-01-01'
GROUP BY c.name, p.product_name;

-- Denormalized: pre-joined summary table (materialized)
CREATE TABLE dbo.SalesSummary (
    summary_id    INT IDENTITY PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    product_name  VARCHAR(100) NOT NULL,
    category_name VARCHAR(50) NOT NULL,
    sale_date     DATE NOT NULL,
    total_amount  DECIMAL(19,4) NOT NULL,
    INDEX IX_SalesSummary_Date (sale_date) INCLUDE (customer_name, total_amount)
);

-- Trade-offs:
-- + Faster reads (no joins)
-- + Simpler reporting queries
-- - Data redundancy (storage)
-- - Must maintain consistency (triggers, ETL, or indexed views)
-- - Update anomalies if source data changes
```

## Clustered Index Design — The NUSE Principle

### Narrow

```sql
-- The clustered key is included in EVERY nonclustered index
-- Narrow key = smaller indexes = more rows per page = less I/O

-- BAD: wide clustered key (52 bytes)
CREATE TABLE dbo.Logs_Bad (
    log_source  VARCHAR(50),
    log_date    DATETIME2(7),
    PRIMARY KEY CLUSTERED (log_source, log_date)  -- 52 bytes per NC index row
);

-- GOOD: narrow clustered key (4 bytes)
CREATE TABLE dbo.Logs (
    log_id     INT IDENTITY(1,1) PRIMARY KEY CLUSTERED,  -- 4 bytes
    log_source VARCHAR(50) NOT NULL,
    log_date   DATETIME2(7) NOT NULL,
    INDEX IX_Logs_Source (log_source, log_date)
);
```

### Unique

```sql
-- If the clustered index is not unique, SQL Server adds a hidden
-- 4-byte "uniqueifier" to duplicate values — wasting space silently.

-- BAD: non-unique clustered index on a column with many duplicates
CREATE CLUSTERED INDEX CIX_Orders_Status ON dbo.Orders (status);
-- 80% of rows may have status = 'Completed' — all get uniqueifiers

-- GOOD: unique clustered index
CREATE UNIQUE CLUSTERED INDEX CIX_Orders_ID ON dbo.Orders (order_id);
```

### Static (Non-Volatile)

```sql
-- Updating the clustered key physically moves the row to its new
-- sorted position — causing page splits, fragmentation, and
-- updates to EVERY nonclustered index pointer.

-- BAD: clustered on a frequently updated column
CREATE CLUSTERED INDEX CIX_Tickets_Priority
ON dbo.SupportTickets (priority, created_date);
-- Priority changes move rows constantly

-- GOOD: clustered on a static, ever-increasing value
CREATE CLUSTERED INDEX CIX_Tickets_ID
ON dbo.SupportTickets (ticket_id);
```

### Ever-Increasing

```sql
-- Sequential inserts go to the end of the index (no page splits)
-- Random inserts go to the middle of the index (page splits)

-- BEST: INT IDENTITY — 4 bytes, unique, sequential
CREATE TABLE dbo.Transactions (
    txn_id INT IDENTITY(1,1) PRIMARY KEY CLUSTERED
);

-- ACCEPTABLE: BIGINT IDENTITY — 8 bytes, for very large tables
CREATE TABLE dbo.Events (
    event_id BIGINT IDENTITY(1,1) PRIMARY KEY CLUSTERED
);

-- AVOID: NEWID() GUID — 16 bytes, random, causes page splits
-- USE INSTEAD: NEWSEQUENTIALID() if GUID is required
```

## Primary Key: IDENTITY vs GUID vs SEQUENCE

```sql
-- Option 1: INT/BIGINT IDENTITY (recommended for most cases)
CREATE TABLE dbo.Users (
    user_id INT IDENTITY(1,1) NOT NULL,
    CONSTRAINT PK_Users PRIMARY KEY CLUSTERED (user_id)
);
-- Pros: narrow (4/8 bytes), sequential, auto-incrementing
-- Cons: gaps on rollback/delete, per-table scope

-- Option 2: SEQUENCE (SQL Server 2012+)
CREATE SEQUENCE dbo.OrderSeq AS BIGINT START WITH 1 INCREMENT BY 1 CACHE 100;

CREATE TABLE dbo.Orders (
    order_id BIGINT NOT NULL DEFAULT NEXT VALUE FOR dbo.OrderSeq,
    CONSTRAINT PK_Orders PRIMARY KEY CLUSTERED (order_id)
);
-- Pros: cross-table sequencing, pre-allocation, range allocation
-- Cons: slightly more complex than IDENTITY

-- Option 3: UNIQUEIDENTIFIER (use only when required)
CREATE TABLE dbo.DistributedEntities (
    entity_id UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID(),
    CONSTRAINT PK_Entities PRIMARY KEY CLUSTERED (entity_id)
);
-- Pros: globally unique, merge-safe
-- Cons: 16 bytes, wider indexes, harder to read/debug
```

## Foreign Keys with Cascading Actions

```sql
-- Foreign key with cascade delete
CREATE TABLE dbo.OrderItems (
    item_id    INT IDENTITY PRIMARY KEY,
    order_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    CONSTRAINT FK_OrderItems_Order
        FOREIGN KEY (order_id) REFERENCES dbo.Orders(order_id)
        ON DELETE CASCADE        -- delete items when order is deleted
        ON UPDATE NO ACTION,     -- prevent order_id changes
    CONSTRAINT FK_OrderItems_Product
        FOREIGN KEY (product_id) REFERENCES dbo.Products(product_id)
        ON DELETE NO ACTION      -- prevent product deletion if referenced
);

-- Cascade options:
-- NO ACTION:    error if referenced rows exist (default)
-- CASCADE:      propagate delete/update to child rows
-- SET NULL:     set FK column to NULL in child rows
-- SET DEFAULT:  set FK column to its DEFAULT value

-- Check existing foreign key actions
SELECT
    fk.name AS fk_name,
    OBJECT_NAME(fk.parent_object_id) AS child_table,
    OBJECT_NAME(fk.referenced_object_id) AS parent_table,
    fk.delete_referential_action_desc,
    fk.update_referential_action_desc
FROM sys.foreign_keys fk;
```

## Computed Columns

```sql
-- Non-persisted: calculated on every read (no storage)
ALTER TABLE dbo.OrderItems
    ADD line_total AS (quantity * unit_price);

-- Persisted: stored physically, updated on write (indexable)
ALTER TABLE dbo.OrderItems
    ADD line_total AS (quantity * unit_price) PERSISTED;

-- Index on a persisted computed column
CREATE INDEX IX_OrderItems_LineTotal
ON dbo.OrderItems (line_total);

-- Deterministic computed column for JSON queries
ALTER TABLE dbo.Events
    ADD event_type AS CAST(JSON_VALUE(event_data, '$.type') AS VARCHAR(50)) PERSISTED;

CREATE INDEX IX_Events_Type ON dbo.Events (event_type);

-- Computed column rules:
-- PERSISTED requires the expression to be deterministic
-- Non-persisted can use non-deterministic functions (but cannot be indexed)
-- Cannot reference other computed columns in the same table
-- Cannot use subqueries
```

## Temporal Tables (System-Versioned, SQL Server 2016+)

```sql
-- Create a system-versioned temporal table
CREATE TABLE dbo.Employees (
    employee_id  INT IDENTITY PRIMARY KEY,
    name         NVARCHAR(100) NOT NULL,
    department   NVARCHAR(50) NOT NULL,
    salary       DECIMAL(12,2) NOT NULL,
    -- Period columns (required, must be DATETIME2)
    valid_from   DATETIME2 GENERATED ALWAYS AS ROW START NOT NULL,
    valid_to     DATETIME2 GENERATED ALWAYS AS ROW END NOT NULL,
    PERIOD FOR SYSTEM_TIME (valid_from, valid_to)
)
WITH (
    SYSTEM_VERSIONING = ON (
        HISTORY_TABLE = dbo.EmployeesHistory,
        DATA_CONSISTENCY_CHECK = ON
    )
);

-- Querying temporal data
-- Current data (default behavior)
SELECT * FROM dbo.Employees WHERE employee_id = 1;

-- Point-in-time query
SELECT * FROM dbo.Employees
FOR SYSTEM_TIME AS OF '2026-01-15 10:00:00'
WHERE employee_id = 1;

-- All historical versions
SELECT * FROM dbo.Employees
FOR SYSTEM_TIME ALL
WHERE employee_id = 1
ORDER BY valid_from;

-- Changes between two dates
SELECT * FROM dbo.Employees
FOR SYSTEM_TIME BETWEEN '2026-01-01' AND '2026-06-01'
WHERE employee_id = 1;

-- Turn off versioning (for maintenance)
ALTER TABLE dbo.Employees SET (SYSTEM_VERSIONING = OFF);
-- Re-enable
ALTER TABLE dbo.Employees SET (
    SYSTEM_VERSIONING = ON (
        HISTORY_TABLE = dbo.EmployeesHistory
    )
);
```

## Graph Tables (SQL Server 2017+)

```sql
-- Node table (entities)
CREATE TABLE dbo.Person AS NODE (
    name NVARCHAR(100),
    age  INT
);

-- Edge table (relationships)
CREATE TABLE dbo.FriendOf AS EDGE (
    since DATE,
    strength INT
);

-- Insert nodes
INSERT dbo.Person (name, age) VALUES ('Alice', 30), ('Bob', 25), ('Carol', 28);

-- Insert edges (using $node_id from the node table)
INSERT dbo.FriendOf ($from_id, $to_id, since, strength)
SELECT p1.$node_id, p2.$node_id, '2025-01-01', 5
FROM dbo.Person p1, dbo.Person p2
WHERE p1.name = 'Alice' AND p2.name = 'Bob';

-- Query graph with MATCH
SELECT
    p1.name AS person,
    p2.name AS friend,
    f.since
FROM dbo.Person p1, dbo.FriendOf f, dbo.Person p2
WHERE MATCH(p1-(f)->p2);

-- Shortest path (SQL Server 2019+)
SELECT
    p1.name,
    STRING_AGG(p2.name, ' -> ') WITHIN GROUP (GRAPH PATH) AS path
FROM dbo.Person p1,
     dbo.FriendOf FOR PATH f,
     dbo.Person FOR PATH p2
WHERE MATCH(SHORTEST_PATH(p1(-(f)->p2)+))
  AND p1.name = 'Alice';
```

## Naming Conventions

```sql
-- Tables: PascalCase, singular nouns
CREATE TABLE dbo.Customer (...);       -- not Customers, not tbl_customer
CREATE TABLE dbo.OrderItem (...);      -- not order_items, not tblOrderItem

-- Columns: PascalCase or snake_case (pick one, be consistent)
-- PascalCase:
CREATE TABLE dbo.SalesOrder (
    SalesOrderId INT IDENTITY PRIMARY KEY,
    CustomerId   INT NOT NULL,
    OrderDate    DATE NOT NULL
);

-- snake_case:
CREATE TABLE dbo.sales_order (
    sales_order_id INT IDENTITY PRIMARY KEY,
    customer_id    INT NOT NULL,
    order_date     DATE NOT NULL
);

-- Constraints: prefix with type
-- PK_TableName               — Primary key
-- FK_ChildTable_ParentTable   — Foreign key
-- UQ_TableName_Column         — Unique constraint
-- CK_TableName_Rule           — Check constraint
-- DF_TableName_Column         — Default constraint
-- IX_TableName_Columns        — Nonclustered index
-- CIX_TableName               — Clustered index (if not PK)

-- Avoid: Hungarian notation (tbl_, sp_, fn_),
-- reserved words as names (User, Order, Index)
```

## Best Practices

- Normalize to 3NF as the starting point. Denormalize strategically where read performance demands it, with a documented maintenance plan.
- Follow the NUSE principle for clustered keys: Narrow, Unique, Static, Ever-increasing. INT IDENTITY is the default choice.
- Always name constraints explicitly. Auto-generated names like `PK__Users__3214EC07` are hard to manage in deployment scripts.
- Use SEQUENCE over IDENTITY when you need cross-table sequencing, range allocation, or cycling.
- Prefer persisted computed columns with indexes over trigger-maintained denormalized columns.
- Use temporal tables for audit requirements instead of building custom history tracking with triggers.
- Design foreign keys deliberately. Use CASCADE with caution — accidental cascade deletes can be catastrophic.
- Establish naming conventions early and enforce them via code review. Consistency matters more than which convention you choose.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using GUID (NEWID()) as clustered primary key | 16-byte key in every index, random inserts cause page splits | Use INT IDENTITY; if GUID required, use NEWSEQUENTIALID() |
| Over-normalizing an OLAP/reporting schema | Excessive joins degrade query performance | Denormalize reporting tables or use indexed views |
| No foreign key constraints ("the app handles it") | Orphaned rows, data integrity violations, silent corruption | Always define FKs; disable during bulk loads if needed, then re-enable |
| VARCHAR(MAX) or NVARCHAR(MAX) as default string type | Inflated memory grants, no inline storage optimization | Size columns to their actual maximum (e.g., VARCHAR(100)) |
| Not including the clustered key columns in analysis | Hidden overhead in every nonclustered index goes unnoticed | Audit clustered key width; multiply by total NC index rows for true cost |
| Using EAV (Entity-Attribute-Value) tables for everything | Poor query performance, no type safety, no constraint enforcement | Use proper relational design; consider JSON columns for truly dynamic attributes |

## SQL Server Version Notes

- **SQL Server 2016** — System-versioned temporal tables. DROP IF EXISTS syntax. Stretch Database. Row-Level Security. Dynamic Data Masking. Always Encrypted columns (limited data type support).
- **SQL Server 2017** — Graph tables (CREATE TABLE AS NODE / EDGE). Resumable online index rebuild. SELECT INTO with filegroup. Adaptive query processing.
- **SQL Server 2019** — SHORTEST_PATH for graph queries. Scalar UDF inlining (schema design implications for computed columns calling UDFs). Accelerated Database Recovery (reduces impact of long transactions on schema changes). Memory-optimized tempdb metadata.
- **SQL Server 2022** — Ledger tables for tamper-evident data (append-only ledger and updatable ledger). JSON_OBJECT/JSON_ARRAY native constructors. IS JSON predicate. XML compression. Contained Availability Group with per-AG system databases. GREATEST/LEAST functions.

## Sources

- [Clustered and Nonclustered Indexes](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described)
- [Temporal Tables](https://learn.microsoft.com/en-us/sql/relational-databases/tables/temporal-tables)
- [Graph Processing with SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/graphs/sql-graph-overview)
- [Computed Columns](https://learn.microsoft.com/en-us/sql/relational-databases/tables/specify-computed-columns-in-a-table)
- [Primary and Foreign Key Constraints](https://learn.microsoft.com/en-us/sql/relational-databases/tables/primary-and-foreign-key-constraints)
- [CREATE SEQUENCE](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-sequence-transact-sql)
