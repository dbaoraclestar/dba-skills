# MySQL Schema Design — Normalization, InnoDB Considerations, and Anti-Patterns

## Overview

Schema design determines the long-term performance, maintainability, and correctness
of a MySQL database. A well-designed schema minimizes data redundancy, supports
efficient query patterns, and leverages InnoDB's clustered index architecture.
A poorly designed schema leads to anomalies, bloated storage, slow queries, and
painful migrations.

MySQL's default storage engine, InnoDB, has unique characteristics that directly
influence schema design decisions. The clustered index (primary key) physically
orders data on disk. Secondary indexes carry a copy of the primary key. Row format,
page size, and buffer pool behavior all affect how data should be structured.

Use this reference when creating new schemas, refactoring existing ones, performing
code reviews on migration scripts, or diagnosing performance problems rooted in
schema structure.

## Key Concepts

- **Normalization**: The process of organizing columns and tables to reduce data
  redundancy and improve data integrity. Measured in normal forms (1NF through 5NF;
  3NF is the practical target for most applications).
- **Denormalization**: Intentionally introducing redundancy to optimize read
  performance at the cost of write complexity and storage.
- **Clustered Index**: InnoDB stores table data ordered by the primary key. This
  is not a separate structure — the table IS the clustered index.
- **Secondary Index Overhead**: Every secondary index in InnoDB includes a copy of
  the primary key columns. Large PKs inflate all secondary indexes.
- **Covering Index**: An index that contains all columns needed by a query, avoiding
  the need to look up the full row in the clustered index.

## Normalization

### First Normal Form (1NF)

Each column contains atomic (indivisible) values. No repeating groups.

```sql
-- VIOLATION: comma-separated values in a column
CREATE TABLE orders_bad (
    id       INT PRIMARY KEY,
    customer VARCHAR(100),
    items    VARCHAR(500)  -- "widget,gadget,doohickey"
);

-- 1NF: separate table for multi-valued relationships
CREATE TABLE orders (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL
);

CREATE TABLE order_items (
    id        INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_id  INT UNSIGNED NOT NULL,
    item_name VARCHAR(100) NOT NULL,
    quantity  INT UNSIGNED NOT NULL DEFAULT 1,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

### Second Normal Form (2NF)

Must be in 1NF, and every non-key column must depend on the entire primary key
(no partial dependencies).

```sql
-- VIOLATION: item_price depends only on item_id, not the full PK
CREATE TABLE order_items_bad (
    order_id   INT UNSIGNED NOT NULL,
    item_id    INT UNSIGNED NOT NULL,
    quantity   INT UNSIGNED NOT NULL,
    item_price DECIMAL(10,2) NOT NULL,  -- depends only on item_id
    PRIMARY KEY (order_id, item_id)
);

-- 2NF: move item_price to its own table
CREATE TABLE items (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL
);

CREATE TABLE order_items_2nf (
    order_id INT UNSIGNED NOT NULL,
    item_id  INT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL,
    -- unit_price at time of order (snapshot, acceptable denormalization)
    unit_price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, item_id),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (item_id) REFERENCES items(id)
);
```

### Third Normal Form (3NF)

Must be in 2NF, and no non-key column depends on another non-key column
(no transitive dependencies).

```sql
-- VIOLATION: city and state depend on zip_code, not on customer_id
CREATE TABLE customers_bad (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    zip_code CHAR(10) NOT NULL,
    city     VARCHAR(100) NOT NULL,  -- determined by zip_code
    state    CHAR(2) NOT NULL        -- determined by zip_code
);

-- 3NF: separate zip_code details (if referential integrity is needed)
CREATE TABLE zip_codes (
    zip_code CHAR(10) NOT NULL PRIMARY KEY,
    city     VARCHAR(100) NOT NULL,
    state    CHAR(2) NOT NULL
);

CREATE TABLE customers_3nf (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    zip_code CHAR(10) NOT NULL,
    FOREIGN KEY (zip_code) REFERENCES zip_codes(zip_code)
);

-- PRAGMATIC APPROACH: 3NF is often relaxed for address data.
-- Storing city/state alongside zip_code in the same row is common and acceptable
-- when the data comes from user input and is not derived from a lookup.
```

## Denormalization Trade-offs

### When Denormalization Is Justified

```sql
-- Counter cache: store computed count to avoid COUNT(*) joins
CREATE TABLE forums (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    post_count INT UNSIGNED NOT NULL DEFAULT 0  -- denormalized counter
);

-- Materialized summary: precompute expensive aggregations
CREATE TABLE daily_sales_summary (
    sale_date    DATE NOT NULL,
    product_id   INT UNSIGNED NOT NULL,
    total_qty    INT UNSIGNED NOT NULL,
    total_amount DECIMAL(14,2) NOT NULL,
    PRIMARY KEY (sale_date, product_id)
);

-- Snapshot data: capture point-in-time values
CREATE TABLE invoices (
    id               INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    customer_id      INT UNSIGNED NOT NULL,
    -- Snapshot of customer name at invoice time (denormalized)
    customer_name    VARCHAR(100) NOT NULL,
    billing_address  TEXT NOT NULL,
    total            DECIMAL(12,2) NOT NULL,
    created_at       DATETIME NOT NULL
);
```

### Denormalization Costs

- Every denormalized field must be updated in multiple places
- Application code must maintain consistency (or use triggers)
- Stale data is possible if update logic has bugs
- Storage increases proportionally to the number of copies

## InnoDB-Specific Design Considerations

### Clustered Index and Primary Key

```sql
-- GOOD: Auto-increment PK — sequential inserts, compact secondary indexes
CREATE TABLE events (
    id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    payload    JSON,
    created_at DATETIME(3) NOT NULL
);

-- GOOD: Natural composite PK when it matches access patterns
CREATE TABLE user_settings (
    user_id     INT UNSIGNED NOT NULL,
    setting_key VARCHAR(50) NOT NULL,
    value       TEXT,
    PRIMARY KEY (user_id, setting_key)
);
-- Queries by user_id are efficient because rows for the same user
-- are physically adjacent in the clustered index

-- BAD: UUID as primary key (random, causes page splits)
CREATE TABLE sessions_bad (
    id CHAR(36) NOT NULL PRIMARY KEY,  -- random UUID
    user_id INT UNSIGNED NOT NULL,
    data JSON
);

-- BETTER: UUID stored as BINARY(16) with ordered generation
CREATE TABLE sessions_good (
    id      BINARY(16) NOT NULL PRIMARY KEY,
    user_id INT UNSIGNED NOT NULL,
    data    JSON
);
-- Insert with: UUID_TO_BIN(UUID(), TRUE) to get time-ordered bytes (8.0+)
```

### Secondary Index Overhead

```sql
-- Every secondary index includes a copy of the PK columns
-- Large PKs inflate ALL secondary indexes

-- With PK = BIGINT (8 bytes):
-- Each secondary index entry adds 8 bytes for the PK reference

-- With PK = (id BIGINT, tenant_id INT, created_at DATETIME):
-- Each secondary index entry adds 8+4+5 = 17 bytes for the PK reference

-- RULE: Keep primary keys as narrow as possible
```

### Covering Indexes

```sql
-- A covering index satisfies the entire query from the index alone
-- (no clustered index lookup needed — shown as "Using index" in EXPLAIN)

CREATE TABLE products (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(200) NOT NULL,
    category VARCHAR(50) NOT NULL,
    price    DECIMAL(10,2) NOT NULL,
    status   TINYINT UNSIGNED NOT NULL DEFAULT 1
);

-- This index covers: SELECT category, price FROM products WHERE category = ?
CREATE INDEX idx_category_price ON products (category, price);

-- This index covers: SELECT name, price WHERE category = ? ORDER BY price
CREATE INDEX idx_cat_price_name ON products (category, price, name);
```

## Naming Conventions

```sql
-- Table names: plural, snake_case
CREATE TABLE order_items (...);
CREATE TABLE user_preferences (...);

-- Column names: snake_case
-- id for primary key, <table_singular>_id for foreign keys
CREATE TABLE comments (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    post_id    INT UNSIGNED NOT NULL,     -- references posts.id
    user_id    INT UNSIGNED NOT NULL,     -- references users.id
    body       TEXT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Index names: idx_<table>_<columns>
CREATE INDEX idx_comments_post_id ON comments (post_id);
CREATE INDEX idx_comments_user_id_created ON comments (user_id, created_at);

-- Foreign key names: fk_<child_table>_<parent_table>
ALTER TABLE comments
  ADD CONSTRAINT fk_comments_posts FOREIGN KEY (post_id) REFERENCES posts(id),
  ADD CONSTRAINT fk_comments_users FOREIGN KEY (user_id) REFERENCES users(id);

-- Unique constraint names: uq_<table>_<columns>
ALTER TABLE users
  ADD CONSTRAINT uq_users_email UNIQUE (email);
```

## Foreign Key Considerations

### ON DELETE / ON UPDATE Actions

```sql
-- CASCADE: delete/update child rows when parent row is deleted/updated
CREATE TABLE order_items (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    order_id INT UNSIGNED NOT NULL,
    item_id  INT UNSIGNED NOT NULL,
    quantity INT UNSIGNED NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (item_id) REFERENCES items(id) ON DELETE RESTRICT
);

-- RESTRICT (default): prevent delete/update if child rows exist
-- SET NULL: set FK column to NULL when parent is deleted
-- NO ACTION: same as RESTRICT in MySQL (deferred check in other RDBMS)

-- Common patterns:
-- CASCADE: order_items when order is deleted
-- RESTRICT: prevent deleting a user who has orders
-- SET NULL: set assigned_to = NULL when an employee leaves
```

### Foreign Keys: Performance Impact

```sql
-- Foreign keys add overhead:
-- 1. INSERT: parent table is checked for existence (shared lock on parent row)
-- 2. DELETE parent: child table is scanned for references
-- 3. UPDATE parent PK: cascading updates to all children

-- For high-throughput OLTP, some teams enforce FK logic in application code
-- and omit FK constraints in the database for performance.
-- This is a trade-off: faster writes vs risk of orphaned data.
```

## Generated Columns (STORED vs VIRTUAL)

```sql
CREATE TABLE products (
    id             INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    price_cents    INT UNSIGNED NOT NULL,
    tax_rate       DECIMAL(5,4) NOT NULL DEFAULT 0.0875,
    -- VIRTUAL: computed on read, no storage, cannot be indexed alone before 8.0
    price_dollars  DECIMAL(10,2) GENERATED ALWAYS AS (price_cents / 100.0) VIRTUAL,
    -- STORED: computed on write, takes disk space, can be indexed
    price_with_tax DECIMAL(10,2) GENERATED ALWAYS AS (price_cents / 100.0 * (1 + tax_rate)) STORED
);

-- Index on STORED generated column
CREATE INDEX idx_price_with_tax ON products (price_with_tax);

-- VIRTUAL generated column with functional index (8.0+)
-- No need for STORED if only used in index
ALTER TABLE products ADD INDEX idx_price_dollars ((price_cents / 100.0));

-- JSON extraction as generated column
CREATE TABLE configs (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    data     JSON NOT NULL,
    env      VARCHAR(20) GENERATED ALWAYS AS (data->>'$.environment') STORED,
    INDEX idx_env (env)
);
```

## Common Anti-Patterns

### Entity-Attribute-Value (EAV)

```sql
-- ANTI-PATTERN: EAV table
CREATE TABLE object_attributes (
    object_id    INT UNSIGNED NOT NULL,
    attribute    VARCHAR(100) NOT NULL,
    value        VARCHAR(500),
    PRIMARY KEY (object_id, attribute)
);
-- Problems: no type safety, no constraints, complex queries, slow joins

-- BETTER: Use JSON column or proper columns
CREATE TABLE products (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    attributes  JSON NOT NULL DEFAULT '{}'
    -- OR: add proper typed columns for known attributes
);
```

### Polymorphic Associations

```sql
-- ANTI-PATTERN: one FK column referencing multiple parent tables
CREATE TABLE comments_bad (
    id              INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    commentable_type VARCHAR(50) NOT NULL,  -- "Post" or "Photo"
    commentable_id   INT UNSIGNED NOT NULL,  -- FK to posts.id OR photos.id
    body             TEXT NOT NULL
);
-- No foreign key constraint possible; no referential integrity

-- BETTER: Separate FK columns (exclusive arcs)
CREATE TABLE comments_better (
    id       INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    post_id  INT UNSIGNED DEFAULT NULL,
    photo_id INT UNSIGNED DEFAULT NULL,
    body     TEXT NOT NULL,
    FOREIGN KEY (post_id) REFERENCES posts(id),
    FOREIGN KEY (photo_id) REFERENCES photos(id),
    CHECK (
        (post_id IS NOT NULL AND photo_id IS NULL) OR
        (post_id IS NULL AND photo_id IS NOT NULL)
    )
);

-- BETTER: Separate join tables
CREATE TABLE post_comments (
    comment_id INT UNSIGNED NOT NULL PRIMARY KEY,
    post_id    INT UNSIGNED NOT NULL,
    FOREIGN KEY (comment_id) REFERENCES comments(id),
    FOREIGN KEY (post_id) REFERENCES posts(id)
);
```

### Comma-Separated Values in Columns

```sql
-- ANTI-PATTERN: storing multiple values in one column
CREATE TABLE articles_bad (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    tags  VARCHAR(500)  -- "mysql,performance,tuning"
);
-- Cannot efficiently query, index, or enforce constraints on individual tags

-- BETTER: junction/bridge table
CREATE TABLE tags (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE article_tags (
    article_id INT UNSIGNED NOT NULL,
    tag_id     INT UNSIGNED NOT NULL,
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id)
);
```

### Over-Indexing

```sql
-- ANTI-PATTERN: index on every column
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name  VARCHAR(50) NOT NULL,
    email      VARCHAR(255) NOT NULL,
    phone      VARCHAR(20),
    city       VARCHAR(100),
    state      CHAR(2),
    INDEX idx_first (first_name),
    INDEX idx_last (last_name),
    INDEX idx_email (email),
    INDEX idx_phone (phone),
    INDEX idx_city (city),
    INDEX idx_state (state),
    INDEX idx_first_last (first_name, last_name)
);
-- idx_first is redundant (left prefix of idx_first_last)
-- Many indexes may never be used
-- Each index slows INSERT/UPDATE/DELETE

-- BETTER: Index based on actual query patterns
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name  VARCHAR(50) NOT NULL,
    email      VARCHAR(255) NOT NULL,
    phone      VARCHAR(20),
    city       VARCHAR(100),
    state      CHAR(2),
    UNIQUE INDEX uq_email (email),
    INDEX idx_name (last_name, first_name),
    INDEX idx_state_city (state, city)
);
```

## Best Practices

- Start with 3NF and denormalize selectively based on measured query performance,
  not speculation.
- Always define an explicit primary key. Use auto-increment integers or ordered
  UUIDs (UUID_TO_BIN with swap flag in 8.0+).
- Keep primary keys narrow (ideally a single BIGINT UNSIGNED) to minimize secondary
  index overhead in InnoDB.
- Name tables, columns, indexes, and constraints consistently using snake_case
  conventions.
- Use foreign keys for data integrity unless you have measured performance reasons
  to omit them and have application-level enforcement.
- Add `created_at` and `updated_at` DATETIME columns to every table for audit
  and debugging purposes.
- Avoid EAV patterns. Use JSON columns or proper typed columns instead.
- Design indexes based on actual query patterns, not theoretical access. Use
  `sys.schema_unused_indexes` to verify.
- Document schema decisions (why a column is denormalized, why a FK was omitted)
  in migration script comments or a schema documentation table.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using UUID CHAR(36) as primary key | 36 bytes per PK, random inserts cause page splits, bloats all secondary indexes | Use BIGINT AUTO_INCREMENT or BINARY(16) with ordered UUID |
| No primary key on InnoDB tables | InnoDB creates a hidden 6-byte clustered key; cannot be referenced; poor performance | Always define an explicit primary key |
| Storing comma-separated values | Cannot index, query, or enforce constraints on individual values | Use a junction table or JSON array with multi-valued index |
| Over-indexing every column | Slows writes, wastes storage, creates maintenance burden | Index based on actual query patterns; audit with sys.schema_unused_indexes |
| Ignoring InnoDB clustered index behavior | Random PK inserts cause page splits; wide PKs bloat secondary indexes | Use sequential PKs; keep PKs narrow |
| Using EAV for extensible attributes | No type safety, complex queries, poor performance, unmaintainable | Use JSON column with generated columns and indexes |

## MySQL Version Notes

- **5.7**: InnoDB is the default engine. Generated columns supported (VIRTUAL and
  STORED). CHECK constraints are parsed but not enforced. JSON type available.
  No functional indexes. Foreign keys supported only on InnoDB.
- **8.0**: CHECK constraints are enforced. Functional indexes (index on expression)
  added. Invisible indexes for testing index removal. Multi-valued indexes on JSON
  arrays. `DEFAULT` expressions for BLOB/TEXT columns (8.0.13). Descending indexes.
  Instant ADD COLUMN for InnoDB (appending columns only).
- **8.4 / 9.x**: Instant DDL improvements. Check release notes for new index types,
  DDL capabilities, and constraint features.

## Sources

- [MySQL 8.0 Reference Manual: InnoDB Table and Page Structure](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
- [MySQL 8.0: CREATE TABLE](https://dev.mysql.com/doc/refman/8.0/en/create-table.html)
- [MySQL 8.0: Foreign Key Constraints](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html)
- [MySQL 8.0: Generated Columns](https://dev.mysql.com/doc/refman/8.0/en/create-table-generated-columns.html)
- [MySQL 8.0: CHECK Constraints](https://dev.mysql.com/doc/refman/8.0/en/create-table-check-constraints.html)
- [Use The Index, Luke — SQL Indexing and Tuning](https://use-the-index-luke.com/)
