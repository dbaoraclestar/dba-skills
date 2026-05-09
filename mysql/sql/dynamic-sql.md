# MySQL Dynamic SQL — PREPARE, EXECUTE, Safe Patterns, and Limitations

## Overview

Dynamic SQL in MySQL is built on the PREPARE/EXECUTE/DEALLOCATE PREPARE protocol. It allows constructing and executing SQL statements at runtime from string expressions. This is essential when the table name, column list, sort order, or WHERE conditions are not known until execution time.

Unlike Oracle's EXECUTE IMMEDIATE or SQL Server's sp_executesql, MySQL's dynamic SQL is more limited. Prepared statements can only use positional `?` placeholders for values -- you cannot parameterize identifiers (table names, column names) or SQL keywords. Identifiers must be concatenated into the SQL string, which requires careful validation to prevent SQL injection.

This skill covers the PREPARE/EXECUTE lifecycle, safe patterns for value binding, workarounds for dynamic identifiers, the limitations of dynamic SQL in stored programs, and injection prevention techniques.

## Key Concepts

- **PREPARE**: Parses a SQL string and creates a prepared statement. The string can come from a user variable or a string literal.
- **EXECUTE**: Runs a previously prepared statement, optionally supplying values for `?` placeholders via `USING`.
- **DEALLOCATE PREPARE**: Frees the resources associated with a prepared statement. Always deallocate after use.
- **User variables**: Session-scoped variables (`@var`) used to pass parameters to `EXECUTE ... USING`. Local variables (DECLARE) cannot be used directly with EXECUTE USING.
- **Placeholder `?`**: Only valid for value positions (WHERE, SET, VALUES). Cannot replace identifiers, keywords, or structural SQL elements.

## Basic PREPARE / EXECUTE / DEALLOCATE

```sql
-- Prepare from a string literal
PREPARE stmt FROM 'SELECT * FROM users WHERE id = ?';
SET @user_id = 42;
EXECUTE stmt USING @user_id;
DEALLOCATE PREPARE stmt;

-- Prepare from a user variable
SET @sql = 'SELECT name, email FROM users WHERE status = ? AND created_at > ?';
PREPARE stmt FROM @sql;
SET @status = 'active';
SET @since = '2025-01-01';
EXECUTE stmt USING @status, @since;
DEALLOCATE PREPARE stmt;

-- INSERT with placeholders
SET @sql = 'INSERT INTO audit_log (action, user_id, details) VALUES (?, ?, ?)';
PREPARE stmt FROM @sql;
SET @action = 'login';
SET @uid = 42;
SET @details = 'Logged in from 10.0.1.5';
EXECUTE stmt USING @action, @uid, @details;
DEALLOCATE PREPARE stmt;

-- Check rows affected
SELECT ROW_COUNT();  -- after EXECUTE
```

## Dynamic SQL in Stored Procedures

```sql
DELIMITER //

CREATE PROCEDURE dynamic_search(
    IN p_table VARCHAR(64),
    IN p_column VARCHAR(64),
    IN p_value VARCHAR(255)
)
BEGIN
    -- Validate table name against whitelist (CRITICAL for security)
    IF p_table NOT IN ('users', 'orders', 'products') THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Invalid table name';
    END IF;

    -- Validate column name against whitelist
    IF p_column NOT IN ('id', 'name', 'email', 'status') THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Invalid column name';
    END IF;

    -- Build the SQL with backtick-quoted identifiers
    SET @sql = CONCAT(
        'SELECT * FROM `', p_table, '` WHERE `', p_column, '` = ?'
    );

    SET @val = p_value;
    PREPARE stmt FROM @sql;
    EXECUTE stmt USING @val;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL dynamic_search('users', 'email', 'alice@example.com');
```

### Dynamic DDL in procedures

```sql
DELIMITER //

CREATE PROCEDURE create_partition_table(
    IN p_table_name VARCHAR(64),
    IN p_year INT
)
BEGIN
    -- Whitelist validation
    IF p_table_name NOT REGEXP '^[a-zA-Z_][a-zA-Z0-9_]*$' THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Invalid table name characters';
    END IF;

    SET @sql = CONCAT(
        'CREATE TABLE IF NOT EXISTS `', p_table_name, '_', p_year, '` (',
        '  id BIGINT AUTO_INCREMENT PRIMARY KEY,',
        '  data JSON NOT NULL,',
        '  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP',
        ') ENGINE=InnoDB'
    );

    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;
```

## Building Dynamic SQL Safely

### Value parameters (safe -- use `?` placeholders)

```sql
-- SAFE: values are parameterized
SET @sql = 'SELECT * FROM users WHERE status = ? AND age > ?';
PREPARE stmt FROM @sql;
SET @status = 'active';
SET @min_age = 18;
EXECUTE stmt USING @status, @min_age;
DEALLOCATE PREPARE stmt;
```

### Dynamic identifiers (requires validation)

```sql
DELIMITER //

CREATE PROCEDURE get_sorted(
    IN p_sort_column VARCHAR(64),
    IN p_sort_dir VARCHAR(4)
)
BEGIN
    -- Whitelist allowed sort columns
    IF p_sort_column NOT IN ('name', 'email', 'created_at', 'id') THEN
        SET p_sort_column = 'id';
    END IF;

    -- Whitelist sort direction
    IF UPPER(p_sort_dir) NOT IN ('ASC', 'DESC') THEN
        SET p_sort_dir = 'ASC';
    END IF;

    SET @sql = CONCAT(
        'SELECT id, name, email FROM users ORDER BY `',
        p_sort_column, '` ', p_sort_dir, ' LIMIT 50'
    );

    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;
```

### Dynamic WHERE clause construction

```sql
DELIMITER //

CREATE PROCEDURE search_users(
    IN p_name VARCHAR(100),
    IN p_email VARCHAR(255),
    IN p_status VARCHAR(20)
)
BEGIN
    DECLARE v_where TEXT DEFAULT '1=1';
    DECLARE v_params INT DEFAULT 0;

    -- Build WHERE clause dynamically
    IF p_name IS NOT NULL THEN
        SET v_where = CONCAT(v_where, ' AND name LIKE ?');
        SET @p1 = CONCAT('%', p_name, '%');
        SET v_params = v_params + 1;
    END IF;

    IF p_email IS NOT NULL THEN
        SET v_where = CONCAT(v_where, ' AND email = ?');
        SET @p2 = p_email;
        SET v_params = v_params + 2;
    END IF;

    IF p_status IS NOT NULL THEN
        SET v_where = CONCAT(v_where, ' AND status = ?');
        SET @p3 = p_status;
        SET v_params = v_params + 4;
    END IF;

    SET @sql = CONCAT('SELECT id, name, email, status FROM users WHERE ', v_where);
    PREPARE stmt FROM @sql;

    -- Execute with the right number of parameters
    -- (MySQL does not support dynamic USING, so we use a CASE approach)
    CASE v_params
        WHEN 0 THEN EXECUTE stmt;
        WHEN 1 THEN EXECUTE stmt USING @p1;
        WHEN 2 THEN EXECUTE stmt USING @p2;
        WHEN 3 THEN EXECUTE stmt USING @p1, @p2;
        WHEN 4 THEN EXECUTE stmt USING @p3;
        WHEN 5 THEN EXECUTE stmt USING @p1, @p3;
        WHEN 6 THEN EXECUTE stmt USING @p2, @p3;
        WHEN 7 THEN EXECUTE stmt USING @p1, @p2, @p3;
    END CASE;

    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;
```

### Alternative: single query with optional filters (no dynamic SQL)

```sql
-- Often simpler and safer than dynamic SQL
SELECT id, name, email, status
FROM users
WHERE (p_name IS NULL OR name LIKE CONCAT('%', p_name, '%'))
  AND (p_email IS NULL OR email = p_email)
  AND (p_status IS NULL OR status = p_status);
-- Caveat: the optimizer may not use indexes as effectively as dynamic SQL
-- with explicit conditions. Check EXPLAIN.
```

## User Variables in Prepared Statements

```sql
-- User variables (@var) are the ONLY way to pass values to EXECUTE USING
-- Local DECLARE variables CANNOT be used with EXECUTE USING

-- CORRECT
SET @val = 42;
EXECUTE stmt USING @val;

-- WRONG (will not work)
DECLARE v_val INT DEFAULT 42;
EXECUTE stmt USING v_val;  -- ERROR: syntax error

-- Workaround: copy local variable to user variable
DECLARE v_val INT DEFAULT 42;
SET @val = v_val;
EXECUTE stmt USING @val;
```

## Dynamic Column Selection

```sql
DELIMITER //

CREATE PROCEDURE get_columns(
    IN p_columns TEXT  -- comma-separated column names
)
BEGIN
    -- Validate each column against a whitelist
    -- This is a simplified approach; production code should parse and validate each column
    SET @validated = '';
    SET @remaining = p_columns;

    validate_loop: LOOP
        SET @comma_pos = LOCATE(',', @remaining);
        IF @comma_pos = 0 THEN
            SET @col = TRIM(@remaining);
            SET @remaining = '';
        ELSE
            SET @col = TRIM(SUBSTRING(@remaining, 1, @comma_pos - 1));
            SET @remaining = SUBSTRING(@remaining, @comma_pos + 1);
        END IF;

        IF @col IN ('id', 'name', 'email', 'status', 'created_at') THEN
            IF @validated = '' THEN
                SET @validated = CONCAT('`', @col, '`');
            ELSE
                SET @validated = CONCAT(@validated, ', `', @col, '`');
            END IF;
        END IF;

        IF @remaining = '' THEN
            LEAVE validate_loop;
        END IF;
    END LOOP;

    IF @validated = '' THEN
        SET @validated = '*';
    END IF;

    SET @sql = CONCAT('SELECT ', @validated, ' FROM users LIMIT 100');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;

CALL get_columns('name, email, status');
```

## Dynamic Table Names (Workarounds)

```sql
-- Cannot parameterize table names with ?
-- WRONG:
PREPARE stmt FROM 'SELECT * FROM ? WHERE id = ?';

-- CORRECT: concatenate validated table name into the SQL string
DELIMITER //

CREATE PROCEDURE query_table(
    IN p_table_name VARCHAR(64),
    IN p_id BIGINT
)
BEGIN
    -- Option 1: whitelist
    IF p_table_name NOT IN ('users', 'orders', 'products', 'categories') THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Table not allowed';
    END IF;

    -- Option 2: verify table exists in INFORMATION_SCHEMA
    -- SELECT COUNT(*) INTO @exists
    -- FROM INFORMATION_SCHEMA.TABLES
    -- WHERE TABLE_SCHEMA = DATABASE() AND TABLE_NAME = p_table_name;
    -- IF @exists = 0 THEN SIGNAL ... END IF;

    SET @sql = CONCAT('SELECT * FROM `', p_table_name, '` WHERE id = ?');
    SET @id = p_id;
    PREPARE stmt FROM @sql;
    EXECUTE stmt USING @id;
    DEALLOCATE PREPARE stmt;
END //

DELIMITER ;
```

## Injection Prevention Patterns

```sql
-- RULE 1: Always use ? for values
-- RULE 2: Always whitelist identifiers
-- RULE 3: Never trust user input for structural SQL elements

-- DANGEROUS (SQL injection via table name)
SET @sql = CONCAT('SELECT * FROM ', user_input);  -- NEVER

-- SAFE (whitelist + backtick quoting)
IF user_input IN ('users', 'orders') THEN
    SET @sql = CONCAT('SELECT * FROM `', user_input, '`');
END IF;

-- DANGEROUS (SQL injection via column)
SET @sql = CONCAT('SELECT ', column_input, ' FROM users');  -- NEVER

-- SAFE (validate with INFORMATION_SCHEMA)
SELECT COUNT(*) INTO @col_exists
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users'
  AND COLUMN_NAME = column_input;

IF @col_exists > 0 THEN
    SET @sql = CONCAT('SELECT `', column_input, '` FROM users');
END IF;

-- DANGEROUS (unquoted string in LIKE)
SET @sql = CONCAT("SELECT * FROM users WHERE name LIKE '%", search_term, "%'");  -- NEVER

-- SAFE (parameterized LIKE)
SET @sql = 'SELECT * FROM users WHERE name LIKE ?';
SET @search = CONCAT('%', search_term, '%');
PREPARE stmt FROM @sql;
EXECUTE stmt USING @search;
```

## Limitations

```sql
-- 1. Cannot use PREPARE/EXECUTE in stored FUNCTIONS (only procedures)
-- ERROR 1336: Dynamic SQL is not allowed in stored function

-- 2. Cannot use DECLARE (local) variables with EXECUTE USING
-- Must use @user_variables

-- 3. Cannot parameterize identifiers (table, column, database names)
-- Must concatenate after validation

-- 4. Cannot parameterize SQL keywords (ORDER BY direction, LIMIT)
-- Must concatenate after validation

-- 5. Maximum number of prepared statements per connection
SHOW VARIABLES LIKE 'max_prepared_stmt_count';  -- default: 16382

-- 6. Prepared statements are session-scoped
-- They are lost when the connection closes

-- 7. Not all statements can be prepared
-- DDL (CREATE, ALTER, DROP), SHOW, SET, and utility statements
-- can be prepared in recent versions, but not all are documented
```

## Best Practices

- Always use `?` placeholders for values. Never concatenate user-supplied values into SQL strings.
- Validate all identifiers (table names, column names) against a whitelist or against `INFORMATION_SCHEMA` before concatenation.
- Always `DEALLOCATE PREPARE` after use to free server resources.
- Use backtick quoting (`` ` ``) around concatenated identifiers to handle reserved words and special characters.
- Prefer static SQL with optional filters (`WHERE (? IS NULL OR col = ?)`) over dynamic SQL when the set of possible conditions is small and fixed.
- Copy `DECLARE` local variables to `@user_variables` before passing to `EXECUTE USING`.
- Limit the use of dynamic SQL to cases where static SQL truly cannot solve the problem (dynamic table names, dynamic column lists, dynamic sort orders).
- Log dynamic SQL strings during development for debugging; disable logging in production.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Concatenating user input directly into SQL | SQL injection vulnerability | Use `?` for values; whitelist identifiers |
| Using DECLARE variables with EXECUTE USING | Syntax error at runtime | Copy to `@user_variable` first |
| Forgetting DEALLOCATE PREPARE | Server accumulates prepared statements up to `max_prepared_stmt_count` | Always DEALLOCATE in a handler or after use |
| Using dynamic SQL in a stored function | Error 1336: not allowed | Move logic to a stored procedure |
| Not quoting identifiers with backticks | Fails on reserved words (e.g., table named `order`) | Always backtick-quote concatenated identifiers |
| Assuming PREPARE validates the SQL | PREPARE parses syntax but may not validate semantics until EXECUTE | Handle errors on both PREPARE and EXECUTE |

## MySQL Version Notes

- **5.7**: PREPARE/EXECUTE/DEALLOCATE fully supported. Limited DDL support in prepared statements. `max_prepared_stmt_count` default is 16382.
- **8.0**: More statement types can be prepared (most DDL). Prepared statements work with CTEs and window functions. No changes to the USING mechanism.
- **8.4 / 9.x**: No significant changes to dynamic SQL capabilities. Same PREPARE/EXECUTE model.

## Sources

- [MySQL PREPARE Statement](https://dev.mysql.com/doc/refman/8.0/en/prepare.html)
- [MySQL EXECUTE Statement](https://dev.mysql.com/doc/refman/8.0/en/execute.html)
- [MySQL DEALLOCATE PREPARE](https://dev.mysql.com/doc/refman/8.0/en/deallocate-prepare.html)
- [MySQL SQL Injection Prevention](https://dev.mysql.com/doc/refman/8.0/en/security-against-attack.html)
