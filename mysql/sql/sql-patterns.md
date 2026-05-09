# MySQL SQL Patterns — Pagination, Upsert, Window Functions, CTEs, and Anti-Joins

## Overview

Well-chosen SQL patterns can eliminate application-side processing, reduce round trips, and leverage MySQL's optimizer effectively. This skill covers the most common and most impactful query patterns: pagination strategies that scale, upsert idioms, conditional aggregation, window functions for analytics, Common Table Expressions for readability and recursion, LATERAL derived tables, and anti-join patterns.

Many of these patterns depend on MySQL version. Window functions, CTEs, and recursive CTEs require MySQL 8.0+. LATERAL derived tables require 8.0.14+. The older patterns (LIMIT/OFFSET, INSERT ... ON DUPLICATE KEY UPDATE, LEFT JOIN ... IS NULL) work across all InnoDB-backed MySQL versions.

This skill focuses on correct, performant SQL. Each pattern includes when to use it, when to avoid it, and how the optimizer handles it.

## Key Concepts

- **Keyset pagination**: Navigating result pages by filtering on the last-seen value rather than using OFFSET. Avoids scanning and discarding rows.
- **Upsert**: An atomic insert-or-update operation. MySQL provides `INSERT ... ON DUPLICATE KEY UPDATE` and `REPLACE` (which deletes then inserts).
- **Window function**: A function that operates on a set of rows related to the current row (the "window") without collapsing them into a single output row. Available in MySQL 8.0+.
- **CTE (Common Table Expression)**: A named subquery defined in a WITH clause. Can be referenced multiple times. Recursive CTEs enable hierarchical queries.
- **LATERAL**: A derived table that can reference columns from preceding tables in the same FROM clause. Enables correlated derived tables.
- **Anti-join**: A pattern that returns rows from one table that have no matching rows in another.

## Pagination

### LIMIT / OFFSET (simple but limited)

```sql
-- Page 1 (rows 1-20)
SELECT id, name, email FROM users
ORDER BY id
LIMIT 20 OFFSET 0;

-- Page 100 (rows 1981-2000)
SELECT id, name, email FROM users
ORDER BY id
LIMIT 20 OFFSET 1980;
-- Problem: MySQL scans and discards 1980 rows before returning 20
```

### Keyset pagination (scalable)

```sql
-- Page 1
SELECT id, name, email FROM users
ORDER BY id
LIMIT 20;
-- Application stores the last id seen (e.g., 20)

-- Page 2: filter by last seen value
SELECT id, name, email FROM users
WHERE id > 20
ORDER BY id
LIMIT 20;
-- Application stores the last id (e.g., 40)

-- Page N: always O(1) index seek, regardless of page number
SELECT id, name, email FROM users
WHERE id > :last_seen_id
ORDER BY id
LIMIT 20;

-- Multi-column keyset (e.g., sort by created_at, id)
SELECT id, name, created_at FROM users
WHERE (created_at, id) > (:last_created_at, :last_id)
ORDER BY created_at, id
LIMIT 20;
-- Requires a composite index on (created_at, id)
```

### Deferred join (OFFSET with reduced cost)

```sql
-- Instead of selecting all columns with OFFSET:
SELECT id, name, email, bio, avatar_url FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;

-- Fetch only the PK with OFFSET, then join back:
SELECT u.id, u.name, u.email, u.bio, u.avatar_url
FROM users u
INNER JOIN (
    SELECT id FROM users
    ORDER BY created_at DESC
    LIMIT 20 OFFSET 10000
) AS page ON u.id = page.id
ORDER BY u.created_at DESC;
-- The inner query scans a covering index (smaller), then the join fetches full rows
```

## Upsert Patterns

### INSERT ... ON DUPLICATE KEY UPDATE

```sql
-- Insert if not exists, update if PK or UNIQUE key matches
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (42, 1, NOW())
ON DUPLICATE KEY UPDATE
    login_count = login_count + 1,
    last_login = NOW();

-- Access the attempted insert values with VALUES() (5.7) or alias (8.0.19+)
-- 8.0.19+ syntax (VALUES() is deprecated):
INSERT INTO inventory (product_id, warehouse_id, quantity)
VALUES (100, 1, 50)
AS new_row
ON DUPLICATE KEY UPDATE
    quantity = inventory.quantity + new_row.quantity;

-- Bulk upsert
INSERT INTO daily_metrics (metric_date, page, views, clicks)
VALUES
    ('2025-01-01', '/home', 1000, 50),
    ('2025-01-01', '/about', 200, 10),
    ('2025-01-01', '/home', 500, 25)
AS new_vals
ON DUPLICATE KEY UPDATE
    views = daily_metrics.views + new_vals.views,
    clicks = daily_metrics.clicks + new_vals.clicks;
```

### REPLACE (delete + insert)

```sql
-- Deletes the existing row (if PK/UNIQUE matches) then inserts
-- WARNING: this changes the auto-increment ID and triggers DELETE + INSERT
REPLACE INTO sessions (session_id, user_id, expires_at)
VALUES ('abc123', 42, DATE_ADD(NOW(), INTERVAL 1 HOUR));

-- Use INSERT ... ON DUPLICATE KEY UPDATE instead in most cases
-- REPLACE fires DELETE triggers and resets auto_increment
```

### INSERT IGNORE

```sql
-- Insert if not exists, silently skip if duplicate
INSERT IGNORE INTO tags (name) VALUES ('mysql'), ('postgres'), ('mysql');
-- Inserts 'mysql' and 'postgres', silently skips the duplicate 'mysql'
-- WARNING: also ignores other errors (type conversion, NOT NULL violations)
```

## Conditional Aggregation

```sql
-- Pivot-style query: count orders by status per month
SELECT
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    COUNT(*) AS total,
    SUM(status = 'completed') AS completed,
    SUM(status = 'cancelled') AS cancelled,
    SUM(status = 'pending') AS pending,
    ROUND(SUM(status = 'completed') / COUNT(*) * 100, 1) AS completion_pct
FROM orders
WHERE created_at >= '2025-01-01'
GROUP BY DATE_FORMAT(created_at, '%Y-%m')
ORDER BY month;

-- Conditional SUM with CASE (more explicit)
SELECT
    department_id,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count,
    AVG(CASE WHEN tenure_years > 5 THEN salary END) AS avg_senior_salary
FROM employees
GROUP BY department_id;
```

## Window Functions (MySQL 8.0+)

### Running totals and moving averages

```sql
-- Running total of daily revenue
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date) AS running_total,
    AVG(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM daily_revenue;

-- Running total with partition
SELECT
    department_id,
    employee_id,
    salary,
    SUM(salary) OVER (PARTITION BY department_id ORDER BY employee_id) AS dept_running_total
FROM employees;
```

### Ranking functions

```sql
-- ROW_NUMBER, RANK, DENSE_RANK
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_val,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank_val
FROM employees;

-- Top-N per group (e.g., top 3 earners per department)
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn <= 3;
```

### LAG / LEAD

```sql
-- Compare each month's revenue to the previous month
SELECT
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) AS prev_month,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) AS month_over_month,
    ROUND((revenue - LAG(revenue, 1) OVER (ORDER BY month))
        / LAG(revenue, 1) OVER (ORDER BY month) * 100, 1) AS pct_change
FROM monthly_revenue;
```

### NTILE and percentiles

```sql
-- Divide employees into salary quartiles
SELECT
    name,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;

-- Percent rank
SELECT
    name,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS percentile
FROM employees;
```

## CTEs and Recursive CTEs (MySQL 8.0+)

### Non-recursive CTE

```sql
-- Named subquery for readability and reuse
WITH active_users AS (
    SELECT id, name, email
    FROM users
    WHERE status = 'active' AND last_login > DATE_SUB(NOW(), INTERVAL 30 DAY)
),
user_orders AS (
    SELECT user_id, COUNT(*) AS order_count, SUM(total) AS total_spent
    FROM orders
    WHERE created_at > DATE_SUB(NOW(), INTERVAL 90 DAY)
    GROUP BY user_id
)
SELECT
    au.name,
    au.email,
    COALESCE(uo.order_count, 0) AS orders_90d,
    COALESCE(uo.total_spent, 0) AS spent_90d
FROM active_users au
LEFT JOIN user_orders uo ON au.id = uo.user_id
ORDER BY uo.total_spent DESC;
```

### Recursive CTE (hierarchical data)

```sql
-- Org chart traversal
WITH RECURSIVE org_tree AS (
    -- Anchor: start from the CEO
    SELECT id, name, manager_id, 0 AS depth, CAST(name AS CHAR(1000)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: join children to parents
    SELECT e.id, e.name, e.manager_id, ot.depth + 1,
           CONCAT(ot.path, ' > ', e.name)
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.id
)
SELECT * FROM org_tree ORDER BY path;

-- Bill of materials (BOM) explosion
WITH RECURSIVE bom AS (
    SELECT component_id, parent_id, quantity, 1 AS level
    FROM components
    WHERE parent_id = 1000  -- top-level product

    UNION ALL

    SELECT c.component_id, c.parent_id, c.quantity * b.quantity, b.level + 1
    FROM components c
    INNER JOIN bom b ON c.parent_id = b.component_id
)
SELECT * FROM bom;

-- Safety: limit recursion depth
SET SESSION cte_max_recursion_depth = 100;  -- default 1000
```

## LATERAL Derived Tables (MySQL 8.0.14+)

```sql
-- Top 3 recent orders per customer (without window functions)
SELECT c.id, c.name, recent.order_id, recent.total, recent.created_at
FROM customers c,
LATERAL (
    SELECT o.id AS order_id, o.total, o.created_at
    FROM orders o
    WHERE o.customer_id = c.id  -- references outer table
    ORDER BY o.created_at DESC
    LIMIT 3
) AS recent
WHERE c.status = 'active';

-- Running aggregates with LATERAL
SELECT d.dept_name, stats.avg_salary, stats.max_salary
FROM departments d,
LATERAL (
    SELECT AVG(salary) AS avg_salary, MAX(salary) AS max_salary
    FROM employees e
    WHERE e.department_id = d.id
) AS stats;
```

## Anti-Join Patterns (EXISTS vs IN vs LEFT JOIN)

### LEFT JOIN ... IS NULL

```sql
-- Find customers with no orders
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;
-- Optimizer often converts this to an anti-join internally
```

### NOT EXISTS (usually best performance)

```sql
-- Find customers with no orders
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
-- Short-circuits as soon as one match is found
-- Generally the fastest anti-join pattern in MySQL
```

### NOT IN (use with caution)

```sql
-- Find customers with no orders
SELECT id, name
FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- DANGER: if orders.customer_id has ANY NULL value, this returns NO rows
-- Because NULL comparisons with NOT IN produce UNKNOWN, not TRUE

-- Safe version (exclude NULLs)
SELECT id, name
FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);
```

### Performance comparison

```sql
-- NOT EXISTS: best for correlated subqueries with an index on the join column
-- LEFT JOIN IS NULL: equivalent performance in most cases after optimization
-- NOT IN: avoid with nullable columns; optimizer may not use anti-join optimization
-- Use EXPLAIN to verify the optimizer's choice
```

## Best Practices

- Use keyset pagination for any list that could grow beyond a few thousand rows. LIMIT/OFFSET degrades linearly with page depth.
- Prefer `INSERT ... ON DUPLICATE KEY UPDATE` over `REPLACE` to avoid unnecessary deletes, auto-increment gaps, and trigger side effects.
- Use `NOT EXISTS` for anti-joins rather than `NOT IN` to avoid the NULL trap.
- Name your CTEs clearly and use them to break complex queries into readable steps.
- Set `cte_max_recursion_depth` when using recursive CTEs to prevent runaway queries.
- Use conditional aggregation (`SUM(CASE ...)` or `SUM(condition)`) to pivot data in SQL rather than making multiple queries.
- Use LATERAL derived tables instead of correlated subqueries in SELECT lists when you need multiple columns from the correlated query.
- Always check EXPLAIN output when switching between equivalent patterns -- the optimizer may handle them differently depending on indexes and data distribution.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using OFFSET for deep pagination (page 1000+) | O(n) scan and discard, very slow | Use keyset pagination |
| `NOT IN` with nullable subquery column | Returns zero rows due to NULL comparison semantics | Use `NOT EXISTS` or filter out NULLs |
| Using `REPLACE` without understanding it deletes first | Auto-increment changes, cascade deletes, trigger side effects | Use `INSERT ... ON DUPLICATE KEY UPDATE` |
| Window functions without ORDER BY in the OVER clause | Undefined ordering, non-deterministic results | Always specify ORDER BY in OVER() for ordered window functions |
| Recursive CTE without a termination condition | Infinite loop until `cte_max_recursion_depth` is hit | Ensure the recursive member converges (e.g., depth limit, no circular references) |
| Using CTE as an optimization barrier (assuming materialization) | MySQL 8.0 may or may not materialize CTEs | Use `WITH ... AS /* materialized */` hint or restructure if materialization is needed |

## MySQL Version Notes

- **5.7**: No window functions, no CTEs, no LATERAL. Use self-joins, subqueries, and user variables for equivalent logic. `INSERT ... ON DUPLICATE KEY UPDATE` with `VALUES()` function.
- **8.0**: Window functions (8.0.2+), CTEs and recursive CTEs (8.0.1+), LATERAL derived tables (8.0.14+). `VALUES()` in ON DUPLICATE KEY deprecated in 8.0.20; use row alias syntax instead. CTE materialization hints available.
- **8.4 / 9.x**: Optimizer improvements for CTE merging. Better cost estimation for window function pushdown. `VALUES()` function removed; row alias syntax required.

## Sources

- [MySQL Window Functions](https://dev.mysql.com/doc/refman/8.0/en/window-functions.html)
- [MySQL WITH (CTE)](https://dev.mysql.com/doc/refman/8.0/en/with.html)
- [MySQL INSERT ... ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html)
- [MySQL LATERAL Derived Tables](https://dev.mysql.com/doc/refman/8.0/en/lateral-derived-tables.html)
