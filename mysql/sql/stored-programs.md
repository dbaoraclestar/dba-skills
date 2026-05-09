# MySQL Stored Programs — Procedures, Functions, Triggers, Events, and Cursors

## Overview

MySQL supports four types of stored programs: stored procedures, stored functions, triggers, and events. Stored programs are compiled SQL/procedural code stored in the database catalog and executed on the server. They reduce network round trips, centralize business logic, and enable server-side automation.

Stored procedures and functions use MySQL's procedural language, which includes variables, control flow (IF, CASE, LOOP, WHILE, REPEAT), cursors, and condition handlers. While less powerful than Oracle PL/SQL, MySQL's procedural language covers the most common use cases: data validation, batch processing, audit logging, and scheduled maintenance.

This skill covers creating and managing all four program types, parameter modes, cursor patterns, condition handlers, and the key limitations compared to Oracle PL/SQL.

## Key Concepts

- **Stored procedure**: A named routine invoked with `CALL`. Can have IN, OUT, and INOUT parameters. Cannot be used in SQL expressions.
- **Stored function**: A named routine that returns a single value. Can be used in SQL expressions like built-in functions. Must be declared DETERMINISTIC or NO SQL/READS SQL DATA.
- **Trigger**: Code that fires automatically before or after an INSERT, UPDATE, or DELETE on a table. Accesses the affected row via `NEW` and `OLD` references.
- **Event**: A scheduled task managed by the event scheduler. Runs SQL at specified intervals or one-time schedules. Replacement for OS-level cron for database tasks.
- **Cursor**: A mechanism for iterating over a result set row by row within a stored procedure. Declared, opened, fetched, and closed.
- **Condition handler**: Exception handling mechanism. Catches SQL warnings, errors, or specific conditions (SQLSTATE or MySQL error codes).

## Stored Procedures

### Basic procedure with IN/OUT/INOUT parameters

```sql
DELIMITER //

CREATE PROCEDURE get_user_stats(
    IN p_user_id INT,
    OUT p_order_count INT,
    OUT p_total_spent DECIMAL(10,2),
    INOUT p_status VARCHAR(20)
)
BEGIN
    SELECT COUNT(*), COALESCE(SUM(total), 0)
    INTO p_order_count, p_total_spent
    FROM orders
    WHERE user_id = p_user_id;

    IF p_order_count > 100 THEN
        SET p_status = 'premium';
    ELSEIF p_order_count > 10 THEN
        SET p_status = 'regular';
    ELSE
        SET p_status = CONCAT(p_status, '_new');
    END IF;
END //

DELIMITER ;

-- Call the procedure
SET @status = 'user';
CALL get_user_stats(42, @count, @total, @status);
SELECT @count, @total, @status;
```

### Control flow

```sql
DELIMITER //

CREATE PROCEDURE process_batch(IN p_batch_size INT)
BEGIN
    DECLARE v_processed INT DEFAULT 0;
    DECLARE v_item_id BIGINT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;

    -- WHILE loop
    WHILE v_processed < p_batch_size DO
        SELECT id INTO v_item_id
        FROM work_queue
        WHERE status = 'pending'
        ORDER BY priority DESC, created_at
        LIMIT 1
        FOR UPDATE SKIP LOCKED;

        IF v_item_id IS NULL THEN
            LEAVE;  -- break out of the loop (no more work)
        END IF;

        UPDATE work_queue SET status = 'processing' WHERE id = v_item_id;

        -- Process the item (call another procedure, etc.)
        CALL process_item(v_item_id);

        UPDATE work_queue SET status = 'completed' WHERE id = v_item_id;
        SET v_processed = v_processed + 1;
    END WHILE;

    SELECT v_processed AS items_processed;
END //

DELIMITER ;
```

### CASE statement

```sql
DELIMITER //

CREATE PROCEDURE apply_discount(
    IN p_order_id INT,
    IN p_discount_type VARCHAR(20)
)
BEGIN
    DECLARE v_pct DECIMAL(5,2);

    CASE p_discount_type
        WHEN 'employee' THEN SET v_pct = 20.00;
        WHEN 'member'   THEN SET v_pct = 10.00;
        WHEN 'coupon'   THEN SET v_pct = 5.00;
        ELSE
            SIGNAL SQLSTATE '45000'
                SET MESSAGE_TEXT = 'Invalid discount type';
    END CASE;

    UPDATE orders
    SET total = total * (1 - v_pct / 100),
        discount_applied = p_discount_type
    WHERE id = p_order_id;
END //

DELIMITER ;
```

## Stored Functions

```sql
DELIMITER //

CREATE FUNCTION calculate_tax(
    p_amount DECIMAL(10,2),
    p_state VARCHAR(2)
) RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_rate DECIMAL(5,4);

    SELECT tax_rate INTO v_rate
    FROM state_tax_rates
    WHERE state_code = p_state;

    IF v_rate IS NULL THEN
        SET v_rate = 0.0000;
    END IF;

    RETURN ROUND(p_amount * v_rate, 2);
END //

DELIMITER ;

-- Use in SQL expressions
SELECT
    id,
    subtotal,
    calculate_tax(subtotal, shipping_state) AS tax,
    subtotal + calculate_tax(subtotal, shipping_state) AS total
FROM orders;

-- Pure computation (no data access)
CREATE FUNCTION mask_email(p_email VARCHAR(255))
RETURNS VARCHAR(255)
DETERMINISTIC
NO SQL
BEGIN
    DECLARE v_at_pos INT;
    SET v_at_pos = LOCATE('@', p_email);
    IF v_at_pos <= 1 THEN
        RETURN p_email;
    END IF;
    RETURN CONCAT(
        LEFT(p_email, 1),
        REPEAT('*', v_at_pos - 2),
        SUBSTRING(p_email, v_at_pos)
    );
END //
```

## Triggers

### BEFORE and AFTER triggers

```sql
DELIMITER //

-- Audit trail on UPDATE
CREATE TRIGGER trg_users_before_update
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    -- Validate email format
    IF NEW.email NOT LIKE '%@%.%' THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Invalid email format';
    END IF;

    -- Auto-set updated_at
    SET NEW.updated_at = NOW();
END //

CREATE TRIGGER trg_users_after_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO users_audit (
        user_id, field_name, old_value, new_value, changed_at, changed_by
    )
    SELECT OLD.id, 'email', OLD.email, NEW.email, NOW(), CURRENT_USER()
    WHERE OLD.email != NEW.email

    UNION ALL

    SELECT OLD.id, 'name', OLD.name, NEW.name, NOW(), CURRENT_USER()
    WHERE OLD.name != NEW.name;
END //

-- Prevent deletes (soft delete enforcement)
CREATE TRIGGER trg_users_before_delete
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'Direct deletes not allowed. Use soft delete (SET status = "deleted").';
END //

DELIMITER ;
```

### INSERT trigger for derived values

```sql
DELIMITER //

CREATE TRIGGER trg_orders_before_insert
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    -- Auto-generate order number
    SET NEW.order_number = CONCAT(
        DATE_FORMAT(NOW(), '%Y%m%d'),
        '-',
        LPAD(FLOOR(RAND() * 100000), 5, '0')
    );

    -- Set default status
    IF NEW.status IS NULL THEN
        SET NEW.status = 'pending';
    END IF;
END //

DELIMITER ;
```

## Events (Scheduled Tasks)

```sql
-- Enable the event scheduler
SET GLOBAL event_scheduler = ON;
-- Or in my.cnf: event_scheduler = ON

DELIMITER //

-- Recurring event: purge old sessions every hour
CREATE EVENT evt_purge_sessions
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
DO
BEGIN
    DELETE FROM sessions WHERE expires_at < NOW();
END //

-- Recurring event: archive old orders daily at 2 AM
CREATE EVENT evt_archive_orders
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 02:00:00'
COMMENT 'Archive orders older than 2 years'
DO
BEGIN
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR);

    DELETE FROM orders
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 2 YEAR);
END //

-- One-time event
CREATE EVENT evt_send_reminder
ON SCHEDULE AT CURRENT_TIMESTAMP + INTERVAL 1 DAY
ON COMPLETION NOT PRESERVE  -- auto-drop after execution
DO
BEGIN
    INSERT INTO notifications (user_id, message)
    SELECT id, 'Your trial expires tomorrow'
    FROM users
    WHERE trial_end = DATE_ADD(CURDATE(), INTERVAL 1 DAY);
END //

DELIMITER ;

-- Manage events
SHOW EVENTS;
ALTER EVENT evt_purge_sessions DISABLE;
ALTER EVENT evt_purge_sessions ENABLE;
DROP EVENT IF EXISTS evt_purge_sessions;
```

## Cursors in Stored Procedures

```sql
DELIMITER //

CREATE PROCEDURE recalculate_all_totals()
BEGIN
    DECLARE v_order_id BIGINT;
    DECLARE v_done BOOLEAN DEFAULT FALSE;
    DECLARE v_count INT DEFAULT 0;

    -- Declare cursor
    DECLARE cur_orders CURSOR FOR
        SELECT id FROM orders WHERE status = 'pending';

    -- Declare handler for end of cursor
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET v_done = TRUE;

    OPEN cur_orders;

    read_loop: LOOP
        FETCH cur_orders INTO v_order_id;
        IF v_done THEN
            LEAVE read_loop;
        END IF;

        -- Recalculate order total from line items
        UPDATE orders o
        SET total = (
            SELECT COALESCE(SUM(quantity * unit_price), 0)
            FROM order_items
            WHERE order_id = v_order_id
        )
        WHERE o.id = v_order_id;

        SET v_count = v_count + 1;
    END LOOP;

    CLOSE cur_orders;

    SELECT v_count AS orders_updated;
END //

DELIMITER ;
```

## Condition Handlers (Exception Handling)

```sql
DELIMITER //

CREATE PROCEDURE safe_insert_user(
    IN p_name VARCHAR(100),
    IN p_email VARCHAR(255),
    OUT p_result VARCHAR(50)
)
BEGIN
    -- Handler for duplicate key
    DECLARE CONTINUE HANDLER FOR 1062
    BEGIN
        SET p_result = 'DUPLICATE_EMAIL';
    END;

    -- Handler for any other SQL exception
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1
            @sqlstate = RETURNED_SQLSTATE,
            @errno = MYSQL_ERRNO,
            @msg = MESSAGE_TEXT;
        SET p_result = CONCAT('ERROR: ', @errno, ' - ', @msg);
        ROLLBACK;
    END;

    SET p_result = 'OK';

    START TRANSACTION;
    INSERT INTO users (name, email) VALUES (p_name, p_email);
    COMMIT;
END //

-- Raising custom errors
CREATE PROCEDURE validate_age(IN p_age INT)
BEGIN
    IF p_age < 0 OR p_age > 150 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Age must be between 0 and 150',
                MYSQL_ERRNO = 50001;
    END IF;
END //

-- Re-raising after logging
CREATE PROCEDURE logged_operation()
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1
            @msg = MESSAGE_TEXT;
        INSERT INTO error_log (message, logged_at)
        VALUES (@msg, NOW());
        RESIGNAL;  -- re-raise the original error
    END;

    -- ... operations that might fail ...
END //

DELIMITER ;
```

## Limitations Compared to Oracle PL/SQL

```
Feature                          Oracle PL/SQL        MySQL
-----------------------------------------------------------
Packages                         Yes                  No
Autonomous transactions          Yes (PRAGMA)         No
Bulk collect / FORALL             Yes                  No (use INSERT...SELECT)
Associative arrays / collections  Yes (TABLE, VARRAY)  No
Object types                     Yes                  No
Pipelined functions               Yes                  No
Native compilation                Yes                  No
Exception variables (%ROWCOUNT)  Yes                  ROW_COUNT()
RETURNING INTO on DML            Yes                  LAST_INSERT_ID() only
Nested functions in packages     Yes                  No (separate routines)
```

## Best Practices

- Use stored procedures for multi-step operations that benefit from reduced round trips (batch processing, data validation, audit logging).
- Declare functions as `DETERMINISTIC` when they always return the same output for the same input -- the optimizer can cache results.
- Keep triggers lightweight. Heavy logic in triggers slows down every DML operation on the table and makes debugging difficult.
- Use `SIGNAL SQLSTATE '45000'` to raise custom application errors from procedures and triggers.
- Always declare a `CONTINUE HANDLER FOR NOT FOUND` when using cursors to avoid infinite loops.
- Prefer set-based operations (`INSERT ... SELECT`, `UPDATE ... JOIN`) over cursor-based row-by-row processing whenever possible.
- Use events for recurring database maintenance (purging, archiving) instead of external cron jobs when the logic is purely SQL.
- Document each stored program with a `COMMENT` clause for discoverability.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Missing `DELIMITER` change before `CREATE PROCEDURE` | Syntax error: the first `;` inside the body terminates the statement | Use `DELIMITER //` before and `DELIMITER ;` after |
| Not declaring `CONTINUE HANDLER FOR NOT FOUND` with cursors | Infinite loop or unexpected behavior when cursor is exhausted | Always declare the handler before opening the cursor |
| Heavyweight trigger logic | Every INSERT/UPDATE/DELETE on the table is slowed | Move complex logic to application or stored procedure |
| Function without `DETERMINISTIC` or `NO SQL` declaration | Binary logging rejects it (`log_bin_trust_function_creators` must be ON) | Declare the correct data access characteristic |
| Using `SIGNAL` without `SQLSTATE` | Syntax error | Use `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...'` |
| Not enabling `event_scheduler` | Events are created but never execute | `SET GLOBAL event_scheduler = ON` or set in my.cnf |

## MySQL Version Notes

- **5.7**: Full support for procedures, functions, triggers, events, cursors, and condition handlers. One trigger per action timing per event per table (e.g., only one BEFORE INSERT trigger). `GET DIAGNOSTICS` available.
- **8.0**: Multiple triggers per action timing per event per table (use `FOLLOWS`/`PRECEDES` to order them). Check constraints (8.0.16+) can sometimes replace BEFORE INSERT/UPDATE triggers. `RESIGNAL` improvements. Atomic DDL means `CREATE PROCEDURE` is atomic.
- **8.4 / 9.x**: No major procedural language changes. Performance improvements for stored program execution.

## Sources

- [MySQL Stored Procedures](https://dev.mysql.com/doc/refman/8.0/en/create-procedure.html)
- [MySQL Stored Functions](https://dev.mysql.com/doc/refman/8.0/en/create-function.html)
- [MySQL Triggers](https://dev.mysql.com/doc/refman/8.0/en/triggers.html)
- [MySQL Events](https://dev.mysql.com/doc/refman/8.0/en/events.html)
- [MySQL Condition Handling](https://dev.mysql.com/doc/refman/8.0/en/condition-handling.html)
