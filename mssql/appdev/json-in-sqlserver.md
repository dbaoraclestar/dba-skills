# JSON in SQL Server — Generating, Parsing, and Querying JSON Data

## Overview

SQL Server provides comprehensive built-in JSON support starting with SQL Server 2016. Unlike XML which has a dedicated data type, JSON is stored in standard NVARCHAR columns and manipulated through a set of built-in functions. This approach leverages the existing string infrastructure while providing efficient JSON parsing and generation capabilities.

Use JSON in SQL Server when integrating with web APIs, storing semi-structured data, exchanging data between services, or when your application layer works natively with JSON. The FOR JSON clause generates JSON output from query results, OPENJSON parses JSON text into relational rows and columns, and scalar functions like JSON_VALUE and JSON_MODIFY provide targeted extraction and manipulation.

SQL Server 2022 added JSON_ARRAY and JSON_OBJECT constructors for more flexible JSON generation, along with ISJSON enhancements. While SQL Server does not have a native JSON data type, computed columns with indexes on JSON properties deliver excellent query performance for indexed lookups.

## Key Concepts

- **FOR JSON** — Converts query results to JSON. `PATH` mode gives you full control over output structure; `AUTO` mode infers structure from table joins.
- **OPENJSON** — Table-valued function that parses JSON text into rows. Can produce key-value pairs (default schema) or typed columns (explicit schema with WITH clause).
- **JSON_VALUE** — Extracts a scalar value from a JSON string using a JSON path expression. Returns NVARCHAR(4000).
- **JSON_QUERY** — Extracts a JSON object or array (not scalar values) from a JSON string.
- **JSON_MODIFY** — Updates a value in a JSON string and returns the modified JSON.
- **ISJSON** — Validates whether a string is well-formed JSON. Returns 1 (valid) or 0 (invalid).
- **JSON path expressions** — Use `$.property` syntax with dot notation: `$.address.city`, `$.orders[0].amount`.
- **Lax vs strict mode** — `lax $.path` (default) returns NULL for missing paths; `strict $.path` raises an error.

## Generating JSON with FOR JSON

```sql
-- FOR JSON PATH: full control over structure
SELECT 
    EmployeeId AS [id],
    Name AS [name],
    Department AS [dept],
    Salary AS [salary],
    HireDate AS [dates.hired]
FROM Employees
WHERE Department = 'Engineering'
FOR JSON PATH;
-- [{"id":1,"name":"Alice","dept":"Engineering","salary":95000.00,"dates":{"hired":"2023-01-15"}}]

-- FOR JSON PATH with ROOT
SELECT EmployeeId, Name, Department
FROM Employees
FOR JSON PATH, ROOT('employees');
-- {"employees":[{"EmployeeId":1,"Name":"Alice","Department":"Engineering"}, ...]}

-- FOR JSON AUTO: structure from joins
SELECT 
    d.DepartmentName,
    e.EmployeeId,
    e.Name
FROM Departments d
JOIN Employees e ON d.DepartmentId = e.DepartmentId
FOR JSON AUTO;
-- [{"DepartmentName":"Engineering","e":[{"EmployeeId":1,"Name":"Alice"},{"EmployeeId":2,"Name":"Bob"}]}]

-- Single object (not array) with WITHOUT_ARRAY_WRAPPER
SELECT TOP 1
    EmployeeId AS id,
    Name AS name,
    Department AS dept
FROM Employees
WHERE EmployeeId = 42
FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
-- {"id":42,"name":"Alice","dept":"Engineering"}

-- Include NULL values (normally omitted)
SELECT EmployeeId, Name, MiddleName, Department
FROM Employees
FOR JSON PATH, INCLUDE_NULL_VALUES;
-- [{"EmployeeId":1,"Name":"Alice","MiddleName":null,"Department":"Engineering"}]

-- Nested JSON with subqueries
SELECT 
    d.DepartmentId AS [id],
    d.DepartmentName AS [name],
    (SELECT e.EmployeeId AS [id], e.Name AS [name]
     FROM Employees e
     WHERE e.DepartmentId = d.DepartmentId
     FOR JSON PATH) AS [employees]
FROM Departments d
FOR JSON PATH;
```

## Parsing JSON with OPENJSON

```sql
-- Default schema: returns key, value, type columns
DECLARE @json NVARCHAR(MAX) = N'{"name":"Alice","age":30,"active":true}';

SELECT * FROM OPENJSON(@json);
-- key     | value  | type
-- name    | Alice  | 1 (string)
-- age     | 30     | 2 (number)
-- active  | true   | 3 (boolean)

-- Explicit schema with WITH clause
DECLARE @employees NVARCHAR(MAX) = N'[
    {"id": 1, "name": "Alice", "salary": 95000, "skills": ["Python","SQL"]},
    {"id": 2, "name": "Bob", "salary": 82000, "skills": ["Java","SQL"]}
]';

SELECT * 
FROM OPENJSON(@employees)
WITH (
    EmployeeId   INT           '$.id',
    Name         NVARCHAR(100) '$.name',
    Salary       DECIMAL(12,2) '$.salary',
    Skills       NVARCHAR(MAX) '$.skills' AS JSON  -- preserve as JSON
);

-- Nested JSON arrays
DECLARE @orders NVARCHAR(MAX) = N'{
    "customer": "Acme Corp",
    "orders": [
        {"orderId": 101, "items": [{"product": "Widget", "qty": 5}]},
        {"orderId": 102, "items": [{"product": "Gadget", "qty": 3}]}
    ]
}';

SELECT 
    customer,
    o.orderId,
    i.product,
    i.qty
FROM OPENJSON(@orders) WITH (
    customer NVARCHAR(100) '$.customer',
    orders   NVARCHAR(MAX) '$.orders' AS JSON
) AS root
CROSS APPLY OPENJSON(root.orders) WITH (
    orderId INT '$.orderId',
    items   NVARCHAR(MAX) '$.items' AS JSON
) AS o
CROSS APPLY OPENJSON(o.items) WITH (
    product NVARCHAR(100) '$.product',
    qty     INT           '$.qty'
) AS i;

-- Insert JSON data into a relational table
INSERT INTO Employees (EmployeeId, Name, Department, Salary)
SELECT EmployeeId, Name, Department, Salary
FROM OPENJSON(@jsonPayload)
WITH (
    EmployeeId  INT            '$.id',
    Name        NVARCHAR(100)  '$.name',
    Department  NVARCHAR(50)   '$.department',
    Salary      DECIMAL(12,2)  '$.salary'
);
```

## Scalar JSON Functions

```sql
-- JSON_VALUE: extract scalar values
DECLARE @json NVARCHAR(MAX) = N'{"name":"Alice","address":{"city":"Seattle","zip":"98101"}}';

SELECT JSON_VALUE(@json, '$.name');           -- Alice
SELECT JSON_VALUE(@json, '$.address.city');   -- Seattle
SELECT JSON_VALUE(@json, '$.address.zip');    -- 98101
SELECT JSON_VALUE(@json, '$.missing');        -- NULL (lax mode, default)
-- SELECT JSON_VALUE(@json, 'strict $.missing'); -- ERROR (strict mode)

-- JSON_QUERY: extract objects or arrays (not scalars)
SELECT JSON_QUERY(@json, '$.address');
-- {"city":"Seattle","zip":"98101"}

-- Use in WHERE clauses
SELECT *
FROM Products
WHERE JSON_VALUE(Attributes, '$.color') = 'red'
  AND CAST(JSON_VALUE(Attributes, '$.weight') AS DECIMAL(10,2)) < 5.0;

-- JSON_MODIFY: update JSON values
DECLARE @doc NVARCHAR(MAX) = N'{"name":"Alice","age":30,"skills":["SQL"]}';

-- Update existing property
SET @doc = JSON_MODIFY(@doc, '$.age', 31);

-- Add new property
SET @doc = JSON_MODIFY(@doc, '$.department', 'Engineering');

-- Append to array
SET @doc = JSON_MODIFY(@doc, 'append $.skills', 'Python');

-- Delete property (set to NULL with strict mode or just set NULL)
SET @doc = JSON_MODIFY(@doc, '$.age', NULL);

-- Insert a JSON object (use JSON_QUERY to avoid double-escaping)
SET @doc = JSON_MODIFY(@doc, '$.address', 
    JSON_QUERY('{"city":"Seattle","state":"WA"}'));

-- ISJSON: validate JSON
SELECT ISJSON('{"valid": true}');    -- 1
SELECT ISJSON('not json');            -- 0
SELECT ISJSON(NULL);                  -- NULL
```

## SQL Server 2022 JSON Constructors

```sql
-- JSON_OBJECT: construct JSON objects
SELECT JSON_OBJECT(
    'id': EmployeeId,
    'name': Name,
    'salary': Salary
) AS employee_json
FROM Employees;
-- {"id":1,"name":"Alice","salary":95000.00}

-- JSON_OBJECT with nested objects
SELECT JSON_OBJECT(
    'id': EmployeeId,
    'name': Name,
    'address': JSON_OBJECT(
        'city': City,
        'state': State
    )
) AS employee_json
FROM Employees;

-- JSON_ARRAY: construct JSON arrays
SELECT JSON_ARRAY(1, 2, 3, 'four', NULL);
-- [1,2,3,"four"]

SELECT JSON_ARRAY(1, 2, 3, NULL NULL ON NULL);
-- [1,2,3,null]   (NULL ON NULL includes nulls)

-- JSON_ARRAY with subquery
SELECT JSON_OBJECT(
    'department': d.DepartmentName,
    'employees': (
        SELECT JSON_ARRAY(e.Name)
        FROM Employees e
        WHERE e.DepartmentId = d.DepartmentId
    )
) 
FROM Departments d;

-- ISJSON with type argument (2022)
SELECT ISJSON('{"a":1}', OBJECT);  -- 1
SELECT ISJSON('[1,2,3]', ARRAY);   -- 1
SELECT ISJSON('"hello"', SCALAR);  -- 1
SELECT ISJSON('123', VALUE);       -- 1
```

## Indexing JSON Data

```sql
-- Strategy: computed column + index
-- Step 1: Add computed column extracting the JSON property
ALTER TABLE Products
ADD Color AS CAST(JSON_VALUE(Attributes, '$.color') AS NVARCHAR(50)) PERSISTED;

ALTER TABLE Products
ADD Weight AS CAST(JSON_VALUE(Attributes, '$.weight') AS DECIMAL(10,2)) PERSISTED;

-- Step 2: Create index on computed column
CREATE INDEX IX_Products_Color ON Products(Color);
CREATE INDEX IX_Products_Weight ON Products(Weight);

-- Now queries using JSON_VALUE can seek on the index
SELECT ProductId, Name
FROM Products
WHERE JSON_VALUE(Attributes, '$.color') = 'red';
-- Optimizer recognizes this matches the computed column and uses IX_Products_Color

-- Full-text search on JSON values
ALTER TABLE Products
ADD Description_FT AS CAST(JSON_VALUE(Attributes, '$.description') AS NVARCHAR(500));

CREATE FULLTEXT INDEX ON Products(Description_FT)
    KEY INDEX PK_Products
    ON FT_Catalog;

-- Filtered index for JSON existence checks
CREATE INDEX IX_Products_HasColor 
ON Products(Color)
WHERE Color IS NOT NULL;
```

## JSON vs XML in SQL Server

```sql
-- JSON: lighter syntax, better for web APIs
SELECT Name, Department, Salary
FROM Employees
FOR JSON PATH;
-- [{"Name":"Alice","Department":"Engineering","Salary":95000}]

-- XML: native data type, schema validation, XQuery
SELECT Name, Department, Salary
FROM Employees
FOR XML PATH('Employee'), ROOT('Employees');
-- <Employees><Employee><Name>Alice</Name><Department>Engineering</Department></Employee></Employees>

-- JSON advantages:
--   Simpler syntax, smaller payload, native to JavaScript/REST APIs
--   Easier to read and write
--   Better interop with modern application frameworks

-- XML advantages:
--   Native XML data type with schema validation (XML Schema Collections)
--   XQuery/XPath for complex querying
--   Namespace support
--   Stronger typing guarantees
--   Primary XML indexes for large XML documents
```

## Performance Considerations

```sql
-- 1. Use computed columns for frequently queried JSON properties
-- This avoids repeated JSON parsing at query time

-- 2. Avoid JSON_VALUE in WHERE without a computed column index
-- BAD: full table scan, parses JSON for every row
SELECT * FROM Products
WHERE JSON_VALUE(Attributes, '$.color') = 'red';

-- GOOD: seeks on index via computed column
ALTER TABLE Products ADD Color AS JSON_VALUE(Attributes, '$.color');
CREATE INDEX IX_Color ON Products(Color);

-- 3. Use OPENJSON with explicit schema (WITH clause)
-- The WITH clause is faster than default schema because it
-- only extracts the columns you need and applies type conversion once

-- 4. Avoid storing large JSON documents (>8KB) if you query individual properties
-- Consider normalizing frequently accessed properties into columns

-- 5. JSON_VALUE returns NVARCHAR(4000) max
-- For values > 4000 chars, use JSON_QUERY (returns NVARCHAR(MAX))

-- 6. Batch JSON operations
-- Instead of calling JSON_MODIFY multiple times, rebuild the JSON object:
UPDATE Products
SET Attributes = (
    SELECT 
        JSON_VALUE(Attributes, '$.name') AS name,
        'newColor' AS color,
        99.99 AS price
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
)
WHERE ProductId = 1;
```

## Best Practices

- Store JSON in NVARCHAR(MAX) columns, not VARCHAR — JSON often contains Unicode data.
- Add CHECK constraints with ISJSON() to ensure data integrity: `ALTER TABLE t ADD CONSTRAINT CK_ValidJSON CHECK (ISJSON(JsonCol) = 1)`.
- Create persisted computed columns with indexes on frequently filtered JSON properties.
- Use explicit schema in OPENJSON (WITH clause) rather than default schema for better performance and type safety.
- Prefer JSON_VALUE for scalar extractions and JSON_QUERY for object/array extractions — using the wrong one returns NULL.
- Use `lax` mode (default) in application code for graceful NULL handling; use `strict` mode in data validation scenarios.
- Consider normalizing JSON properties that are frequently updated, filtered, or joined — relational columns are more efficient for these operations.
- Use JSON_OBJECT/JSON_ARRAY (SQL Server 2022) instead of string concatenation for constructing JSON.
- Validate incoming JSON with ISJSON() before storing it.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using JSON_VALUE to extract an object/array | Returns NULL silently | Use JSON_QUERY for objects and arrays |
| Using JSON_QUERY to extract a scalar | Returns NULL silently | Use JSON_VALUE for scalar values |
| Filtering on JSON_VALUE without computed column index | Full table scan on every query | Create persisted computed column + index |
| Storing JSON in VARCHAR instead of NVARCHAR | Unicode data corruption, emoji/special chars lost | Use NVARCHAR(MAX) |
| Exceeding JSON_VALUE's 4000-char limit | Truncated or NULL result | Use JSON_QUERY for large values, or OPENJSON |
| Using string concatenation to build JSON | Injection risk, malformed JSON with special chars | Use FOR JSON PATH, JSON_OBJECT, or JSON_MODIFY |
| Not adding ISJSON check constraint | Invalid JSON stored, runtime errors on read | Add CHECK constraint: `CHECK (ISJSON(col) = 1)` |

## SQL Server Version Notes

- **SQL Server 2016** — Initial JSON support: FOR JSON, OPENJSON, JSON_VALUE, JSON_QUERY, JSON_MODIFY, ISJSON. No native JSON data type.
- **SQL Server 2017** — No new JSON features (STRING_AGG added but not JSON-specific).
- **SQL Server 2019** — No new JSON-specific features. Performance improvements in string handling benefit JSON operations.
- **SQL Server 2022** — Added JSON_OBJECT() and JSON_ARRAY() constructors. ISJSON() enhanced with type argument (OBJECT, ARRAY, SCALAR, VALUE). JSON_PATH_EXISTS() function added.
- **Azure SQL Database** — All JSON features from 2022 are available. Identical syntax and behavior.

## Sources

- https://learn.microsoft.com/en-us/sql/relational-databases/json/json-data-sql-server
- https://learn.microsoft.com/en-us/sql/t-sql/functions/json-value-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/functions/openjson-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/queries/select-for-clause-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/functions/json-object-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/json/index-json-data
