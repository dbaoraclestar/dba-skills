# MySQL Character Sets and Collations — utf8mb4, Collation Types, and Encoding Issues

## Overview

Character sets and collations control how MySQL stores, compares, and sorts text
data. A character set defines the set of characters that can be stored and their
binary encoding. A collation defines the rules for comparing and ordering characters
within a character set — whether comparisons are case-sensitive, accent-sensitive,
or binary.

The most critical decision is choosing `utf8mb4` over the legacy `utf8` (which is
actually `utf8mb3` — a MySQL-specific 3-byte subset of UTF-8 that cannot store
emoji, mathematical symbols, or many CJK characters). Modern applications should
use `utf8mb4` everywhere: server default, database, table, column, and connection.

Use this reference when setting up new MySQL instances, migrating character sets,
debugging encoding issues (mojibake, question marks, truncation), or optimizing
collation choices for performance and correctness.

## Key Concepts

- **Character Set**: Defines which characters can be represented and their byte
  encoding (e.g., `utf8mb4` = full UTF-8, 1-4 bytes per character).
- **Collation**: Rules for comparing and sorting characters. Named as
  `charset_language_sensitivity` (e.g., `utf8mb4_0900_ai_ci`).
- **Case Sensitivity**: `_ci` = case-insensitive, `_cs` = case-sensitive,
  `_bin` = binary (byte-by-byte comparison).
- **Accent Sensitivity**: `_ai` = accent-insensitive (a = a = a),
  `_as` = accent-sensitive (a != a != a).
- **Pad Attribute**: How trailing spaces are handled in comparisons. `PAD SPACE`
  (traditional) ignores trailing spaces; `NO PAD` (8.0 default for 0900 collations)
  treats them as significant.

## utf8mb4 vs utf8mb3 (utf8)

### Why utf8mb4 Is Required

```sql
-- MySQL's "utf8" is actually utf8mb3: a 3-byte subset of UTF-8
-- It CANNOT store characters outside the Basic Multilingual Plane (BMP):
--   - Emoji (U+1F600 and above)
--   - Mathematical symbols (U+1D400+)
--   - Musical symbols (U+1D100+)
--   - Historic scripts
--   - Some CJK ideographs (Extension B and beyond)

-- This INSERT fails with utf8/utf8mb3:
-- INSERT INTO messages (body) VALUES ('Hello World! 😀');
-- ERROR 1366: Incorrect string value '\xF0\x9F\x98\x80' for column 'body'

-- utf8mb4 supports the full Unicode range (1-4 bytes per character)
-- It is TRUE UTF-8 encoding, compatible with all other systems
```

### Storage Differences

| Character Set | Max Bytes/Char | VARCHAR(255) Max Bytes |
|---------------|----------------|------------------------|
| latin1        | 1              | 255                    |
| utf8mb3       | 3              | 765                    |
| utf8mb4       | 4              | 1020                   |

```sql
-- Impact on index key length:
-- InnoDB has a default index key length limit of 3072 bytes
-- VARCHAR(255) with utf8mb4 = 255 * 4 = 1020 bytes (fits in index)
-- VARCHAR(768) with utf8mb4 = 768 * 4 = 3072 bytes (maximum)
-- VARCHAR(769) with utf8mb4 = 769 * 4 = 3076 bytes (exceeds limit!)

-- Check the maximum index prefix length
SHOW VARIABLES LIKE 'innodb_large_prefix';  -- deprecated in 8.0 (always ON)
```

## Collation Types

### General Collations (_general_ci)

```sql
-- utf8mb4_general_ci: fast but less accurate Unicode comparisons
-- Does not handle all Unicode equivalences correctly
-- Example: German sharp-s (ss) is NOT equal to "ss"

-- Best for: legacy systems, when performance matters more than linguistic accuracy
-- Avoid for: multilingual applications requiring correct sorting
```

### Unicode Collations (_unicode_ci)

```sql
-- utf8mb4_unicode_ci: follows Unicode Collation Algorithm (UCA 4.0.0)
-- More accurate than general_ci but slower
-- Handles linguistic equivalences (e.g., a = a in many languages)

-- Better than general_ci for multilingual data
-- Available in both 5.7 and 8.0
```

### UCA 9.0.0 Collations (8.0+ _0900_)

```sql
-- utf8mb4_0900_ai_ci: default collation in MySQL 8.0
-- Based on Unicode Collation Algorithm version 9.0.0
-- "ai" = accent-insensitive, "ci" = case-insensitive
-- Uses NO PAD attribute (trailing spaces are significant)

-- Available 0900 variants:
-- utf8mb4_0900_ai_ci    — accent/case insensitive (default)
-- utf8mb4_0900_as_ci    — accent-sensitive, case-insensitive
-- utf8mb4_0900_as_cs    — accent-sensitive, case-sensitive
-- utf8mb4_0900_bin      — binary comparison (byte-by-byte)

-- Language-specific 0900 collations:
-- utf8mb4_de_pb_0900_ai_ci   — German phonebook ordering
-- utf8mb4_ja_0900_as_cs      — Japanese
-- utf8mb4_zh_0900_as_cs      — Chinese
-- utf8mb4_tr_0900_ai_ci      — Turkish (I/i handling)
```

### Binary Collation (_bin)

```sql
-- utf8mb4_bin: byte-by-byte comparison
-- Case-sensitive, accent-sensitive, exact binary match
-- Fastest comparison (no character transformation)

-- Use for: case-sensitive lookups, hash values, exact matching
CREATE TABLE api_keys (
    id        INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    api_key   VARCHAR(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL UNIQUE,
    user_id   INT UNSIGNED NOT NULL
);
-- 'ApiKey123' != 'apikey123' with _bin collation
```

### Comparison of Collation Behavior

```sql
-- Test how different collations compare strings
SELECT
  'cafe' = 'cafe' COLLATE utf8mb4_general_ci AS general_ci,     -- 1 (equal)
  'cafe' = 'cafe' COLLATE utf8mb4_0900_ai_ci AS ai_ci,         -- 1 (equal)
  'cafe' = 'cafe' COLLATE utf8mb4_0900_as_ci AS as_ci,         -- 0 (not equal)
  'cafe' = 'cafe' COLLATE utf8mb4_bin AS bin_compare;           -- 0 (not equal)

-- Case sensitivity test
SELECT
  'ABC' = 'abc' COLLATE utf8mb4_0900_ai_ci AS ci_result,       -- 1 (equal)
  'ABC' = 'abc' COLLATE utf8mb4_0900_as_cs AS cs_result,       -- 0 (not equal)
  'ABC' = 'abc' COLLATE utf8mb4_bin AS bin_result;              -- 0 (not equal)
```

## Setting Character Set and Collation

### Server Level

```ini
[mysqld]
character_set_server = utf8mb4
collation_server = utf8mb4_0900_ai_ci    # 8.0 default
# collation_server = utf8mb4_unicode_ci  # 5.7 recommendation
```

```sql
-- Check server defaults
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';

-- Change at runtime (affects new databases/tables only)
SET GLOBAL character_set_server = 'utf8mb4';
SET GLOBAL collation_server = 'utf8mb4_0900_ai_ci';
```

### Database Level

```sql
-- Set at creation
CREATE DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- Change existing database default (does NOT change existing tables)
ALTER DATABASE myapp
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- Check database character set
SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME, DEFAULT_COLLATION_NAME
FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME = 'myapp';
```

### Table Level

```sql
-- Set at creation
CREATE TABLE users (
    id    INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name  VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL
) ENGINE=InnoDB
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- Change existing table (converts ALL columns)
ALTER TABLE users
  CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;
-- WARNING: This rebuilds the table and may take a long time on large tables

-- Change table default only (does NOT convert existing columns)
ALTER TABLE users
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_0900_ai_ci;
```

### Column Level

```sql
-- Override table default for specific columns
CREATE TABLE mixed_encoding (
    id          INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,  -- inherits table charset
    binary_hash VARCHAR(64) CHARACTER SET ascii COLLATE ascii_bin NOT NULL,
    notes       TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci
);

-- Change a single column
ALTER TABLE users
  MODIFY COLUMN email VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL;
```

### Connection Level

```sql
-- Set connection character set (affects how client sends/receives data)
SET NAMES 'utf8mb4';
-- Equivalent to setting all three:
-- SET character_set_client = 'utf8mb4';
-- SET character_set_connection = 'utf8mb4';
-- SET character_set_results = 'utf8mb4';

-- Verify connection settings
SHOW VARIABLES LIKE 'character_set%';
SHOW VARIABLES LIKE 'collation%';

-- In application connection strings:
-- JDBC: jdbc:mysql://host/db?characterEncoding=UTF-8&connectionCollation=utf8mb4_0900_ai_ci
-- Python: mysql.connector.connect(charset='utf8mb4', collation='utf8mb4_0900_ai_ci')
-- PHP PDO: new PDO("mysql:host=...;charset=utf8mb4", ...)
```

## Collation Effects on Sorting and Comparison

```sql
-- Sorting differences between collations
CREATE TEMPORARY TABLE sort_test (word VARCHAR(20));
INSERT INTO sort_test VALUES ('apple'), ('Apple'), ('APPLE'),
                             ('apfel'), ('Apfel'), ('banana');

-- Case-insensitive sort (utf8mb4_0900_ai_ci)
SELECT word FROM sort_test ORDER BY word COLLATE utf8mb4_0900_ai_ci;
-- apfel, Apfel, apple, Apple, APPLE, banana

-- Case-sensitive sort (utf8mb4_0900_as_cs)
SELECT word FROM sort_test ORDER BY word COLLATE utf8mb4_0900_as_cs;
-- APPLE, Apfel, Apple, apfel, apple, banana

-- Binary sort (utf8mb4_bin) — by byte value
SELECT word FROM sort_test ORDER BY word COLLATE utf8mb4_bin;
-- APPLE, Apfel, Apple, apfel, apple, banana (uppercase first: A=0x41, a=0x61)

-- COLLATE in WHERE clause
SELECT * FROM sort_test
WHERE word = 'apple' COLLATE utf8mb4_0900_as_cs;
-- Returns only 'apple' (not 'Apple' or 'APPLE')

-- WARNING: Using COLLATE in WHERE can prevent index usage
-- Better to define the column with the correct collation
```

## Character Set Conversion

### CONVERT Function

```sql
-- Convert string between character sets
SELECT CONVERT('Hello World' USING utf8mb4);
SELECT CONVERT(column_name USING utf8mb4) FROM some_table;

-- CAST with character set
SELECT CAST('text' AS CHAR CHARACTER SET utf8mb4);
```

### ALTER TABLE Conversion

```sql
-- Convert entire table from utf8mb3 to utf8mb4
ALTER TABLE legacy_table
  CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- Check before converting: find tables still using utf8mb3
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_COLLATION
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_COLLATION LIKE 'utf8_%'
  AND TABLE_COLLATION NOT LIKE 'utf8mb4_%'
  AND TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys');

-- Find columns with non-utf8mb4 character sets
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME,
       CHARACTER_SET_NAME, COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE CHARACTER_SET_NAME IS NOT NULL
  AND CHARACTER_SET_NAME != 'utf8mb4'
  AND TABLE_SCHEMA NOT IN ('mysql', 'performance_schema',
                           'information_schema', 'sys')
ORDER BY TABLE_SCHEMA, TABLE_NAME, ORDINAL_POSITION;
```

### Bulk Conversion Script

```sql
-- Generate ALTER statements for all non-utf8mb4 tables in a database
SELECT CONCAT('ALTER TABLE `', TABLE_SCHEMA, '`.`', TABLE_NAME,
              '` CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;')
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_TYPE = 'BASE TABLE'
  AND (TABLE_COLLATION NOT LIKE 'utf8mb4_%'
       OR TABLE_COLLATION IS NULL)
ORDER BY TABLE_NAME;
```

## Diagnosing Mojibake (Encoding Corruption)

Mojibake occurs when text is stored or read using the wrong character set. Common
symptoms: `Ã©` instead of `e`, `â€™` instead of `'`, `???` instead of emoji.

```sql
-- Check what encoding a column actually has
SELECT COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'messages'
  AND COLUMN_NAME = 'body';

-- Inspect raw bytes of corrupt data
SELECT body, HEX(body), LENGTH(body), CHAR_LENGTH(body)
FROM messages
WHERE id = 12345;
-- If LENGTH > CHAR_LENGTH, multi-byte characters are present
-- If data looks like double-encoding, bytes were UTF-8 encoded twice

-- Common mojibake: UTF-8 bytes interpreted as latin1 then stored as UTF-8
-- Fix: convert latin1 bytes back to binary, then interpret as UTF-8
UPDATE messages
SET body = CONVERT(CAST(CONVERT(body USING latin1) AS BINARY) USING utf8mb4)
WHERE id = 12345;

-- Verify the connection character set matches the client
SHOW VARIABLES LIKE 'character_set_client';
SHOW VARIABLES LIKE 'character_set_connection';
SHOW VARIABLES LIKE 'character_set_results';
-- All three should be utf8mb4 for modern applications
```

### Prevention Checklist

```sql
-- 1. Server default
SHOW VARIABLES LIKE 'character_set_server';  -- should be utf8mb4

-- 2. Database default
SELECT DEFAULT_CHARACTER_SET_NAME FROM INFORMATION_SCHEMA.SCHEMATA
WHERE SCHEMA_NAME = 'mydb';  -- should be utf8mb4

-- 3. Table and column level
SELECT TABLE_NAME, TABLE_COLLATION FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'mydb';

-- 4. Connection level
SHOW VARIABLES LIKE 'character_set_client';     -- utf8mb4
SHOW VARIABLES LIKE 'character_set_connection'; -- utf8mb4
SHOW VARIABLES LIKE 'character_set_results';    -- utf8mb4

-- 5. Application connection string includes charset=utf8mb4
```

## Collation Mismatch Errors

```sql
-- ERROR 1267: Illegal mix of collations
-- Happens when comparing columns with different collations

-- Fix option 1: COLLATE clause in query
SELECT * FROM t1 JOIN t2
ON t1.name COLLATE utf8mb4_0900_ai_ci = t2.name COLLATE utf8mb4_0900_ai_ci;

-- Fix option 2: Change column collation to match
ALTER TABLE t2
  MODIFY COLUMN name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- Fix option 3: Use CONVERT
SELECT * FROM t1 JOIN t2
ON t1.name = CONVERT(t2.name USING utf8mb4);
```

## Best Practices

- Use `utf8mb4` as the character set for all new databases, tables, and columns.
  There is no reason to use `utf8mb3` (utf8) in new deployments.
- Use `utf8mb4_0900_ai_ci` as the default collation in MySQL 8.0+ for general-purpose
  applications. Use `utf8mb4_unicode_ci` in MySQL 5.7.
- Set `character_set_server = utf8mb4` in my.cnf so all new objects inherit it.
- Always include `charset=utf8mb4` in application connection strings (SET NAMES alone
  can be forgotten or overridden).
- Use `_bin` collation for columns that require exact case-sensitive matching
  (API keys, hashes, tokens) rather than relying on BINARY casts in queries.
- When migrating from utf8mb3 to utf8mb4, check index key lengths — columns that
  were at the 767-byte limit with utf8mb3 may exceed limits with utf8mb4.
- Test collation behavior with representative data before deployment — especially
  for languages with special sorting rules (Turkish I/i, German a/a, etc.).
- Ensure consistent collation across all tables that are joined — mismatches cause
  errors or prevent index usage.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using utf8 (utf8mb3) instead of utf8mb4 | Cannot store emoji or supplementary Unicode; data truncated or rejected | Change to utf8mb4 everywhere |
| Not setting connection charset | Client sends latin1; server stores garbled bytes as utf8mb4 | Add `charset=utf8mb4` to connection string and/or `SET NAMES utf8mb4` |
| Mixing collations across joined tables | `Illegal mix of collations` error or index not used for join | Standardize collation across all tables |
| Converting utf8mb3 VARCHAR(255) to utf8mb4 without checking indexes | Index key length exceeds 3072 bytes (255*4=1020 is fine, but 768*4=3072 is the limit) | Check index key lengths before conversion; reduce VARCHAR size if needed |
| Using general_ci for multilingual data | Incorrect sorting and comparison for many languages | Use unicode_ci (5.7) or 0900_ai_ci (8.0) for accurate Unicode handling |
| Fixing mojibake with ALTER TABLE without understanding the encoding layers | Double-conversion makes corruption permanent | Diagnose with HEX() first; apply targeted CONVERT/CAST fixes |

## MySQL Version Notes

- **5.7**: Default character set is `latin1` (unless changed). Default collation
  is `latin1_swedish_ci`. `utf8` refers to `utf8mb3` (3-byte subset).
  Recommended collation for utf8mb4 is `utf8mb4_unicode_ci`. No 0900 collations.
  `utf8mb4_general_ci` is the default collation for utf8mb4.
- **8.0**: Default character set changed to `utf8mb4`. Default collation changed
  to `utf8mb4_0900_ai_ci`. `utf8mb3` is deprecated (use `utf8mb4` instead).
  New UCA 9.0.0 collations with NO PAD attribute. `utf8` is still an alias for
  `utf8mb3` but generates deprecation warnings. Language-specific 0900 collations
  added (German, Japanese, Chinese, Turkish, etc.).
- **8.4 / 9.x**: `utf8mb3` may be removed in a future release. `utf8` alias will
  eventually point to `utf8mb4`. Plan migration now. Check release notes for
  collation additions or deprecation timeline updates.

## Sources

- [MySQL 8.0 Reference Manual: Character Sets](https://dev.mysql.com/doc/refman/8.0/en/charset.html)
- [MySQL 8.0: Unicode Character Sets](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode.html)
- [MySQL 8.0: Collation Naming Conventions](https://dev.mysql.com/doc/refman/8.0/en/charset-collation-names.html)
- [MySQL 8.0: Connection Character Sets](https://dev.mysql.com/doc/refman/8.0/en/charset-connection.html)
- [MySQL 8.0: utf8mb3 Deprecation](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb3.html)
- [MySQL 5.7: Character Set Configuration](https://dev.mysql.com/doc/refman/5.7/en/charset-configuration.html)
