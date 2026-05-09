# JSON in MySQL — Storage, Querying, Indexing, and JSON_TABLE

## Overview

MySQL has supported a native JSON data type since version 5.7.8. Unlike storing JSON as TEXT or VARCHAR, the JSON type validates documents on insert, stores them in an optimized binary format for fast element access, and provides a rich set of functions for creation, extraction, modification, and aggregation.

The JSON type is ideal for semi-structured data that varies between rows (user preferences, API payloads, configuration blobs) or for evolving schemas where adding columns is impractical. However, JSON should not replace normalized columns for data that is frequently filtered, joined, or reported on -- relational columns with proper indexes will always be faster for structured queries.

This skill covers the JSON data type, all major JSON functions, path expressions, extraction operators, modification functions, aggregation, indexing strategies (generated columns and multi-valued indexes), and JSON_TABLE for converting JSON to relational rows.

## Key Concepts

- **JSON data type**: Binary storage format that validates JSON on insert. Maximum size is the `max_allowed_packet` value (default 64MB). Not the same as a TEXT column.
- **JSON path expression**: A string like `'$.address.city'` that navigates into a JSON document. Uses `$` for the root, `.key` for object members, `[n]` for array elements, and `[*]`/`**` for wildcards.
- **`->` operator**: Shorthand for `JSON_EXTRACT()`. Returns the value with quotes for strings.
- **`->>` operator**: Shorthand for `JSON_UNQUOTE(JSON_EXTRACT())`. Returns the unquoted value. Preferred for comparisons and display.
- **Generated columns**: Virtual or stored columns derived from a JSON expression. Can be indexed for fast lookups.
- **Multi-valued index**: An index on a JSON array (8.0.17+). Uses `CAST(... AS unsigned ARRAY)` syntax.

## JSON Column Definition

```sql
CREATE TABLE events (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    payload JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_event_type (event_type)
);

-- Insert valid JSON
INSERT INTO events (event_type, payload) VALUES
('user_signup', '{"name": "Alice", "email": "alice@example.com", "tags": ["new", "trial"]}'),
('purchase', '{"item_id": 42, "amount": 29.99, "currency": "USD", "metadata": {"coupon": "SAVE10"}}');

-- Invalid JSON is rejected
INSERT INTO events (event_type, payload) VALUES ('bad', '{invalid}');
-- ERROR 3140: Invalid JSON text
```

## JSON Creation Functions

```sql
-- JSON_OBJECT: create an object from key-value pairs
SELECT JSON_OBJECT('name', 'Alice', 'age', 30, 'active', TRUE);
-- {"age": 30, "name": "Alice", "active": true}

-- JSON_ARRAY: create an array
SELECT JSON_ARRAY(1, 'two', 3.0, NULL);
-- [1, "two", 3.0, null]

-- JSON_QUOTE: wrap a string as a JSON string value
SELECT JSON_QUOTE('hello "world"');
-- "hello \"world\""

-- Build JSON from query results
SELECT JSON_OBJECT(
    'id', u.id,
    'name', u.name,
    'orders', (
        SELECT JSON_ARRAYAGG(JSON_OBJECT('id', o.id, 'total', o.total))
        FROM orders o WHERE o.user_id = u.id
    )
) AS user_json
FROM users u
WHERE u.id = 1;
```

## Extraction Operators and Functions

```sql
-- -> operator (returns JSON value, strings are quoted)
SELECT payload->'$.name' FROM events WHERE id = 1;
-- "Alice"

-- ->> operator (returns unquoted text, use for comparisons)
SELECT payload->>'$.name' FROM events WHERE id = 1;
-- Alice

-- JSON_EXTRACT (equivalent to ->)
SELECT JSON_EXTRACT(payload, '$.metadata.coupon') FROM events WHERE id = 2;
-- "SAVE10"

-- Array element access (0-based)
SELECT payload->>'$.tags[0]' FROM events WHERE id = 1;
-- new

-- Wildcard: all array elements
SELECT JSON_EXTRACT(payload, '$.tags[*]') FROM events WHERE id = 1;
-- ["new", "trial"]

-- Multiple paths
SELECT JSON_EXTRACT(payload, '$.name', '$.email') FROM events WHERE id = 1;
-- ["Alice", "alice@example.com"]

-- Check if key exists
SELECT JSON_CONTAINS_PATH(payload, 'one', '$.metadata.coupon') FROM events WHERE id = 2;
-- 1

-- Check if value exists in document
SELECT JSON_CONTAINS(payload->'$.tags', '"trial"') FROM events WHERE id = 1;
-- 1

-- Type inspection
SELECT JSON_TYPE(payload->'$.name') FROM events WHERE id = 1;
-- STRING

-- Get all keys
SELECT JSON_KEYS(payload) FROM events WHERE id = 1;
-- ["email", "name", "tags"]
```

## JSON Path Expressions

```sql
-- Root
'$'

-- Object member
'$.name'
'$.address.city'

-- Array element
'$.tags[0]'
'$.tags[last]'           -- 8.0.26+: last element

-- Wildcard object member
'$.*.name'               -- name key at any top-level member

-- Wildcard array element
'$.tags[*]'              -- all elements

-- Recursive descent (double asterisk)
'$**.id'                 -- id key at any depth

-- Array range (8.0.26+)
'$.tags[0 to 2]'         -- elements 0, 1, 2
```

## JSON Modification Functions

```sql
-- JSON_SET: insert or update values
UPDATE events
SET payload = JSON_SET(payload, '$.verified', true, '$.name', 'Alice Smith')
WHERE id = 1;

-- JSON_INSERT: insert only (skip if path exists)
UPDATE events
SET payload = JSON_INSERT(payload, '$.source', 'web', '$.name', 'IGNORED')
WHERE id = 1;
-- $.source is added, $.name is NOT changed because it already exists

-- JSON_REPLACE: update only (skip if path does not exist)
UPDATE events
SET payload = JSON_REPLACE(payload, '$.name', 'Alice Johnson', '$.nonexistent', 'IGNORED')
WHERE id = 1;
-- $.name is updated, $.nonexistent is NOT added because it does not exist

-- JSON_REMOVE: delete paths
UPDATE events
SET payload = JSON_REMOVE(payload, '$.tags[0]', '$.metadata')
WHERE id = 2;

-- JSON_ARRAY_APPEND: add to end of array
UPDATE events
SET payload = JSON_ARRAY_APPEND(payload, '$.tags', 'premium')
WHERE id = 1;

-- JSON_ARRAY_INSERT: insert at specific position
UPDATE events
SET payload = JSON_ARRAY_INSERT(payload, '$.tags[0]', 'vip')
WHERE id = 1;

-- JSON_MERGE_PATCH: RFC 7396 merge (last value wins for duplicates)
UPDATE events
SET payload = JSON_MERGE_PATCH(payload, '{"name": "Bob", "phone": "555-0100"}')
WHERE id = 1;

-- JSON_MERGE_PRESERVE: merge with array concatenation for duplicates
SELECT JSON_MERGE_PRESERVE('{"a": 1}', '{"a": 2}');
-- {"a": [1, 2]}
```

## JSON Aggregation Functions

```sql
-- JSON_ARRAYAGG: aggregate rows into a JSON array
SELECT event_type, JSON_ARRAYAGG(payload->>'$.name') AS names
FROM events
GROUP BY event_type;

-- JSON_OBJECTAGG: aggregate rows into a JSON object
SELECT JSON_OBJECTAGG(event_type, payload->>'$.name') AS type_to_name
FROM events;
-- {"user_signup": "Alice", "purchase": null}

-- Build nested JSON from a join
SELECT JSON_OBJECT(
    'department', d.name,
    'employees', (
        SELECT JSON_ARRAYAGG(
            JSON_OBJECT('name', e.name, 'title', e.title)
        )
        FROM employees e WHERE e.dept_id = d.id
    )
) AS dept_json
FROM departments d;
```

## Indexing JSON Data

### Generated columns for indexing

```sql
-- Add a generated column that extracts a JSON value
ALTER TABLE events
ADD COLUMN email VARCHAR(255)
    GENERATED ALWAYS AS (payload->>'$.email') VIRTUAL,
ADD INDEX idx_email (email);

-- Now this query uses the index
SELECT * FROM events WHERE email = 'alice@example.com';

-- Stored generated column (for composite indexes or frequent reads)
ALTER TABLE events
ADD COLUMN amount DECIMAL(10,2)
    GENERATED ALWAYS AS (CAST(payload->>'$.amount' AS DECIMAL(10,2))) STORED,
ADD INDEX idx_amount (amount);
```

### Multi-valued indexes (MySQL 8.0.17+)

```sql
-- Index on all elements of a JSON array
CREATE TABLE products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(200),
    attributes JSON,
    INDEX idx_tags ((CAST(attributes->'$.tags' AS CHAR(50) ARRAY)))
);

INSERT INTO products (name, attributes) VALUES
('Laptop', '{"tags": ["electronics", "portable", "work"]}'),
('Tablet', '{"tags": ["electronics", "portable", "media"]}');

-- Query uses the multi-valued index via MEMBER OF
SELECT * FROM products
WHERE 'portable' MEMBER OF (attributes->'$.tags');

-- Also works with JSON_CONTAINS and JSON_OVERLAPS
SELECT * FROM products
WHERE JSON_CONTAINS(attributes->'$.tags', '"electronics"');

SELECT * FROM products
WHERE JSON_OVERLAPS(attributes->'$.tags', '["work", "gaming"]');
```

## JSON_TABLE (Relational Mapping)

JSON_TABLE converts JSON data into relational rows and columns, usable in FROM clauses.

```sql
-- Basic JSON_TABLE usage
SELECT jt.*
FROM events e,
JSON_TABLE(
    e.payload,
    '$' COLUMNS (
        name VARCHAR(100) PATH '$.name',
        email VARCHAR(255) PATH '$.email',
        first_tag VARCHAR(50) PATH '$.tags[0]'
    )
) AS jt
WHERE e.id = 1;

-- Unnest a JSON array into rows
SELECT e.id, jt.tag
FROM events e,
JSON_TABLE(
    e.payload,
    '$.tags[*]' COLUMNS (
        tag VARCHAR(50) PATH '$'
    )
) AS jt
WHERE e.event_type = 'user_signup';

-- Nested JSON_TABLE for nested objects/arrays
SELECT jt.*
FROM orders o,
JSON_TABLE(
    o.line_items,
    '$[*]' COLUMNS (
        row_num FOR ORDINALITY,
        product_name VARCHAR(100) PATH '$.product',
        quantity INT PATH '$.qty',
        price DECIMAL(10,2) PATH '$.price',
        NESTED PATH '$.discounts[*]' COLUMNS (
            discount_code VARCHAR(20) PATH '$.code',
            discount_pct DECIMAL(5,2) PATH '$.percent'
        )
    )
) AS jt;

-- Error handling with DEFAULT and NULL ON ERROR
SELECT jt.*
FROM events e,
JSON_TABLE(
    e.payload,
    '$' COLUMNS (
        amount DECIMAL(10,2) PATH '$.amount'
            DEFAULT '0.00' ON ERROR
            NULL ON EMPTY
    )
) AS jt;
```

## Best Practices

- Use the JSON data type for truly semi-structured data (varying schemas, API payloads). Do not use it as a substitute for normalized columns that are frequently queried or joined.
- Always use `->>` (unquoted extraction) for WHERE clauses and comparisons, not `->` (quoted extraction).
- Create generated columns with indexes for any JSON path that appears in WHERE, JOIN, or ORDER BY clauses.
- Use multi-valued indexes (8.0.17+) for queries that search within JSON arrays.
- Prefer `JSON_SET` for upsert operations, `JSON_INSERT` for insert-if-absent, and `JSON_REPLACE` for update-if-exists.
- Use `JSON_MERGE_PATCH` (not `JSON_MERGE_PRESERVE`) when you want last-value-wins semantics for merging objects.
- Validate JSON structure at the application layer before inserting -- MySQL only validates syntax, not schema.
- Avoid deeply nested JSON (>3 levels) in frequently queried paths -- extraction performance degrades with nesting depth.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `->` instead of `->>` in WHERE clauses | Comparison against quoted string fails (`"Alice"` != `Alice`) | Use `->>` for unquoted extraction |
| No index on frequently filtered JSON paths | Full table scan on every query | Add generated column + index or multi-valued index |
| Storing normalized data as JSON to avoid ALTER TABLE | Poor query performance, no referential integrity | Use relational columns for structured, queryable data |
| Using `JSON_MERGE` (deprecated) | Removed in MySQL 8.0.3 | Use `JSON_MERGE_PRESERVE` or `JSON_MERGE_PATCH` |
| Assuming JSON column order is preserved | MySQL sorts JSON object keys alphabetically in binary format | Do not rely on key order in JSON objects |
| Using `LIKE` on JSON columns | Cannot leverage JSON indexes, scans entire document as text | Use `JSON_CONTAINS`, `JSON_EXTRACT`, or `MEMBER OF` |

## MySQL Version Notes

- **5.7**: JSON type introduced (5.7.8). `JSON_EXTRACT`, `JSON_SET`, `JSON_REMOVE`, `JSON_CONTAINS`, `->` and `->>` available. No multi-valued indexes, no `JSON_TABLE`, no `JSON_ARRAYAGG`/`JSON_OBJECTAGG`.
- **8.0**: `JSON_TABLE` added (8.0.4). `JSON_ARRAYAGG` and `JSON_OBJECTAGG` added. Multi-valued indexes on JSON arrays (8.0.17). `JSON_OVERLAPS` and `MEMBER OF` operator (8.0.17). `JSON_VALUE()` function (8.0.21). Array range in paths (`[0 to 2]`, `[last]`) added in 8.0.26.
- **8.4 / 9.x**: Performance improvements for JSON storage and extraction. `JSON_MERGE` fully removed (use `JSON_MERGE_PATCH` or `JSON_MERGE_PRESERVE`).

## Sources

- [MySQL JSON Data Type](https://dev.mysql.com/doc/refman/8.0/en/json.html)
- [MySQL JSON Functions](https://dev.mysql.com/doc/refman/8.0/en/json-functions.html)
- [MySQL JSON Path Syntax](https://dev.mysql.com/doc/refman/8.0/en/json-path-syntax.html)
- [MySQL JSON_TABLE](https://dev.mysql.com/doc/refman/8.0/en/json-table-functions.html)
- [Multi-Valued Indexes](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-multi-valued)
