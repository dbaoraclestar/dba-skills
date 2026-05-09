# Query Profiling — Stage-Level Timing, Performance Schema, and Bottleneck Identification in MySQL

## Overview

Query profiling is the process of measuring where time is spent during query execution. While EXPLAIN tells you the plan the optimizer chose, profiling tells you where the actual wall-clock time goes -- parsing, optimizing, opening tables, executing, sorting, sending data, and closing. This distinction matters because a query can have a perfect execution plan but still be slow due to lock waits, disk I/O during execution, temporary table creation, or network latency sending results.

MySQL provides several profiling mechanisms. The legacy `SHOW PROFILE` command (deprecated since 5.6 but still functional through 8.0) gives quick per-stage timing. Performance Schema's `events_stages_*` tables provide the modern, production-safe equivalent with much richer data. The `sys` schema wraps Performance Schema into human-friendly views. Starting in MySQL 8.0.18, `EXPLAIN ANALYZE` adds actual timing to the execution plan tree. For deep optimizer investigation, the optimizer trace provides a complete log of every decision the optimizer made.

For production troubleshooting, the recommended approach is: (1) identify slow queries via the slow query log or Performance Schema statement digests, (2) use EXPLAIN/EXPLAIN ANALYZE to understand the plan, (3) use Performance Schema stages to identify which execution phase consumes time, and (4) use optimizer trace only when the plan itself is suspicious.

## Key Concepts

**Query Execution Stages**: Every query passes through a series of stages: starting, checking permissions, opening tables, init, optimizing, executing, sending data, end, closing, freeing items, cleaning up. Each stage has measurable duration.

**Statement Events**: Performance Schema captures statement-level events with total timing, rows examined, rows sent, and error information in the `events_statements_*` tables.

**Stage Events**: Subordinate to statement events. Each stage event records a specific phase of execution with nanosecond-precision timing. Linked to the parent statement via `NESTING_EVENT_ID`.

**Digest**: A normalized query fingerprint. All queries with the same structure but different literal values share the same digest, enabling aggregation across thousands of executions.

## SHOW PROFILE (Legacy, Deprecated)

SHOW PROFILE was the original profiling tool. It is deprecated since MySQL 5.6.7 but still works in MySQL 8.0. It will be removed in a future version.

### Basic Usage

```sql
-- Enable profiling for the current session
SET profiling = 1;

-- Run the query you want to profile
SELECT c.customer_name, COUNT(o.order_id) AS order_count,
       SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = 'US-WEST'
  AND o.order_date >= '2025-01-01'
GROUP BY c.customer_name
ORDER BY total_spent DESC
LIMIT 20;

-- View available profiles
SHOW PROFILES;
-- +----------+------------+-------------------------------------------+
-- | Query_ID | Duration   | Query                                     |
-- +----------+------------+-------------------------------------------+
-- |        1 | 0.23456789 | SELECT c.customer_name, COUNT(o.order... |
-- +----------+------------+-------------------------------------------+

-- View detailed profile for query 1
SHOW PROFILE FOR QUERY 1;
-- +----------------------+-----------+
-- | Status               | Duration  |
-- +----------------------+-----------+
-- | starting             | 0.000045  |
-- | checking permissions | 0.000012  |
-- | Opening tables       | 0.000089  |
-- | init                 | 0.000034  |
-- | System lock          | 0.000015  |
-- | optimizing           | 0.000078  |
-- | statistics           | 0.000156  |
-- | preparing            | 0.000032  |
-- | Creating tmp table   | 0.000234  |
-- | Sorting result       | 0.000012  |
-- | executing            | 0.000005  |
-- | Sending data         | 0.225678  | <-- Most time here
-- | Creating sort index  | 0.004567  |
-- | end                  | 0.000015  |
-- | removing tmp table   | 0.000089  |
-- | end                  | 0.000008  |
-- | query end            | 0.000012  |
-- | closing tables       | 0.000023  |
-- | freeing items        | 0.000156  |
-- | cleaning up          | 0.000015  |
-- +----------------------+-----------+

-- View profile with additional columns
SHOW PROFILE CPU, BLOCK IO FOR QUERY 1;
-- Shows CPU_user, CPU_system, Block_ops_in, Block_ops_out per stage

SHOW PROFILE ALL FOR QUERY 1;
-- Shows all available metrics: CPU, Block IO, Context switches, Source

-- Disable profiling
SET profiling = 0;
```

### Interpreting SHOW PROFILE Stages

| Stage | What It Means | If High |
|---|---|---|
| starting | Query initialization | Rarely a problem |
| checking permissions | ACL check | Too many privilege checks |
| Opening tables | Opening table handles, can involve disk I/O | Table cache too small (`table_open_cache`) |
| System lock | Acquiring system-level locks | Lock contention |
| optimizing | Choosing execution plan | Complex query, many tables/indexes |
| statistics | Computing statistics for plan | Stale statistics or complex joins |
| preparing | Preparing for execution | Rarely a problem |
| Creating tmp table | Building in-memory or on-disk temp table | Large GROUP BY, consider indexes |
| Sending data | Executing and sending results | The main work; slow here = tune query/indexes |
| Sorting result | Sorting after execution | Missing index for ORDER BY |
| Creating sort index | Building sort index | Large sort, increase `sort_buffer_size` if needed |
| removing tmp table | Cleaning up temp table | Proportional to temp table size |

## Performance Schema — Statement-Level Profiling

Performance Schema is the modern, production-ready profiling infrastructure. It is always available (cannot be disabled at runtime in 8.0) and has low overhead when configured properly.

### Enabling Statement and Stage Instrumentation

```sql
-- Check current setup
SELECT * FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%' AND ENABLED = 'NO';

SELECT * FROM performance_schema.setup_consumers
WHERE NAME LIKE '%statements%' OR NAME LIKE '%stages%';

-- Enable stage instrumentation (if not already enabled)
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'stage/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN (
    'events_statements_current',
    'events_statements_history',
    'events_statements_history_long',
    'events_stages_current',
    'events_stages_history',
    'events_stages_history_long'
);
```

### Profiling a Specific Query with Performance Schema

```sql
-- Step 1: Get the current thread ID
SELECT THREAD_ID FROM performance_schema.threads
WHERE PROCESSLIST_ID = CONNECTION_ID();
-- Let's say it returns 48

-- Step 2: Truncate history to get a clean slate
TRUNCATE TABLE performance_schema.events_statements_history;
TRUNCATE TABLE performance_schema.events_stages_history;

-- Step 3: Run the query
SELECT p.product_name, SUM(oi.quantity) AS units_sold
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.order_date BETWEEN '2025-01-01' AND '2025-06-30'
  AND p.category = 'Electronics'
GROUP BY p.product_name
ORDER BY units_sold DESC
LIMIT 10;

-- Step 4: Get the statement event
SELECT
    EVENT_ID,
    SQL_TEXT,
    ROUND(TIMER_WAIT / 1e12, 6) AS exec_time_sec,
    ROWS_EXAMINED,
    ROWS_SENT,
    ROWS_AFFECTED,
    CREATED_TMP_TABLES,
    CREATED_TMP_DISK_TABLES,
    NO_INDEX_USED,
    NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_history
WHERE THREAD_ID = 48
ORDER BY EVENT_ID DESC
LIMIT 1\G

-- Step 5: Get the stage breakdown for that statement
-- Use the EVENT_ID from Step 4 as the NESTING_EVENT_ID
SELECT
    EVENT_NAME AS stage,
    ROUND(TIMER_WAIT / 1e12, 6) AS duration_sec,
    ROUND(TIMER_WAIT / (SELECT SUM(TIMER_WAIT)
        FROM performance_schema.events_stages_history
        WHERE NESTING_EVENT_ID = 1234) * 100, 2) AS pct_of_total
FROM performance_schema.events_stages_history
WHERE NESTING_EVENT_ID = 1234  -- Replace with actual EVENT_ID from Step 4
ORDER BY TIMER_START;
```

### Output Example

```
+----------------------------------------------+--------------+--------------+
| stage                                        | duration_sec | pct_of_total |
+----------------------------------------------+--------------+--------------+
| stage/sql/starting                           |     0.000032 |         0.01 |
| stage/sql/Executing hook on transaction begin|     0.000008 |         0.00 |
| stage/sql/starting                           |     0.000015 |         0.01 |
| stage/sql/checking permissions               |     0.000023 |         0.01 |
| stage/sql/checking permissions               |     0.000012 |         0.01 |
| stage/sql/checking permissions               |     0.000009 |         0.00 |
| stage/sql/Opening tables                     |     0.000145 |         0.06 |
| stage/sql/init                               |     0.000067 |         0.03 |
| stage/sql/System lock                        |     0.000018 |         0.01 |
| stage/sql/optimizing                         |     0.000089 |         0.04 |
| stage/sql/statistics                         |     0.000234 |         0.10 |
| stage/sql/preparing                          |     0.000045 |         0.02 |
| stage/sql/Creating tmp table                 |     0.000312 |         0.13 |
| stage/sql/executing                          |     0.231456 |        95.23 | <--
| stage/sql/end                                |     0.000012 |         0.01 |
| stage/sql/query end                          |     0.000018 |         0.01 |
| stage/sql/waiting for handler commit         |     0.000056 |         0.02 |
| stage/sql/closing tables                     |     0.000034 |         0.01 |
| stage/sql/freeing items                      |     0.000678 |         0.28 |
| stage/sql/cleaning up                        |     0.000023 |         0.01 |
+----------------------------------------------+--------------+--------------+
```

## sys Schema Statement Analysis Views

The sys schema provides pre-built views that wrap Performance Schema data in human-readable format.

### statement_analysis — Top Queries by Total Latency

```sql
-- Top queries by total execution time
SELECT
    query,
    db,
    exec_count,
    total_latency,
    avg_latency,
    max_latency,
    rows_sent_avg,
    rows_examined_avg,
    tmp_tables,
    tmp_disk_tables,
    full_scans
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 20\G
```

### statements_with_errors_or_warnings

```sql
-- Queries that produce errors or warnings
SELECT
    query,
    db,
    exec_count,
    errors,
    warnings,
    total_latency
FROM sys.statements_with_errors_or_warnings
ORDER BY errors DESC
LIMIT 10;
```

### statements_with_full_table_scans

```sql
-- Queries performing full table scans
SELECT
    query,
    db,
    exec_count,
    total_latency,
    no_index_used_count,
    no_good_index_used_count,
    rows_examined_avg,
    rows_sent_avg
FROM sys.statements_with_full_table_scans
WHERE db NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY no_index_used_count DESC
LIMIT 20;
```

### statements_with_temp_tables

```sql
-- Queries creating temporary tables (especially on disk)
SELECT
    query,
    db,
    exec_count,
    total_latency,
    tmp_tables,
    tmp_disk_tables,
    ROUND(tmp_disk_tables / NULLIF(tmp_tables, 0) * 100, 1) AS pct_on_disk
FROM sys.statements_with_temp_tables
WHERE tmp_disk_tables > 0
ORDER BY tmp_disk_tables DESC
LIMIT 20;
```

### statements_with_sorting

```sql
-- Queries with sorting activity
SELECT
    query,
    db,
    exec_count,
    total_latency,
    sort_merge_passes,
    sort_rows,
    sort_scan,
    sort_range
FROM sys.statements_with_sorting
ORDER BY sort_merge_passes DESC
LIMIT 20;
```

## EXPLAIN ANALYZE (MySQL 8.0.18+)

EXPLAIN ANALYZE executes the query and provides actual timing data in a tree format. This is covered in detail in the explain-plan skill, but here we focus on using it for profiling.

```sql
EXPLAIN ANALYZE
SELECT department, COUNT(*) AS emp_count, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date >= '2020-01-01'
  AND status = 'active'
GROUP BY department
HAVING emp_count > 10
ORDER BY avg_salary DESC\G

-- -> Sort: avg_salary DESC  (actual time=12.345..12.347 rows=8 loops=1)
--     -> Filter: (emp_count > 10)  (actual time=12.280..12.310 rows=8 loops=1)
--         -> Table scan on <temporary>  (actual time=12.275..12.290 rows=15 loops=1)
--             -> Aggregate using temporary table  (actual time=12.270..12.270 rows=15 loops=1)
--                 -> Filter: ((employees.status = 'active') and (employees.hire_date >= DATE'2020-01-01'))
--                    (cost=4523.10 rows=3245) (actual time=0.045..8.567 rows=3102 loops=1)
--                     -> Table scan on employees  (cost=4523.10 rows=32450)
--                        (actual time=0.038..5.234 rows=32450 loops=1)
```

### Profiling with EXPLAIN ANALYZE: What to Look For

1. **Large gap between estimated and actual rows**: Indicates stale statistics
2. **High actual time on leaf nodes**: I/O or inefficient access method
3. **Loops > 1 with high actual time per loop**: Nested loop with expensive inner operation
4. **Actual time first row vs last row spread**: First row time >> 0 means blocking operation (sort, aggregate)

## Optimizer Trace

The optimizer trace records every decision the optimizer makes: which indexes it considered, cost estimates for each alternative, and why it chose a particular plan.

### Enabling and Using Optimizer Trace

```sql
-- Enable optimizer trace for the session
SET optimizer_trace = 'enabled=on';
SET optimizer_trace_max_mem_size = 1048576;  -- 1MB (default is often too small)

-- Run the query
SELECT c.customer_name, o.order_date, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE c.region = 'US-WEST'
  AND o.order_date >= '2025-01-01'
  AND o.total_amount > 500
ORDER BY o.order_date DESC
LIMIT 100;

-- Read the trace
SELECT * FROM information_schema.OPTIMIZER_TRACE\G

-- Disable when done (it has overhead)
SET optimizer_trace = 'enabled=off';
```

### Key Optimizer Trace Sections

```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {"expanded_query": "/* full expanded query */"}
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "...",
              "steps": [
                {"transformation": "equality_propagation", "resulting_condition": "..."},
                {"transformation": "constant_propagation", "resulting_condition": "..."},
                {"transformation": "trivial_condition_removal", "resulting_condition": "..."}
              ]
            }
          },
          {
            "rows_estimation": [
              {
                "table": "`customers` `c`",
                "range_analysis": {
                  "table_scan": {"rows": 50000, "cost": 5123.45},
                  "potential_range_indexes": [
                    {"index": "idx_region", "usable": true, "key_parts": ["region"]}
                  ],
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_region",
                        "ranges": ["US-WEST <= region <= US-WEST"],
                        "rows": 8500,
                        "cost": 2345.67,
                        "chosen": true
                      }
                    ]
                  }
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [],
                "table": "`customers` `c`",
                "best_access_path": {
                  "considered_access_paths": [
                    {"access_type": "ref", "index": "idx_region", "rows": 8500, "cost": 2345.67, "chosen": true}
                  ]
                },
                "cost_for_plan": 2345.67,
                "rows_for_plan": 8500,
                "chosen": true
              }
            ]
          }
        ]
      }
    }
  ]
}
```

### Using Optimizer Trace to Diagnose Plan Choices

```sql
-- Why did the optimizer choose a full table scan over an index?
-- Check the "range_analysis" section for each table:
-- - "table_scan" cost vs index scan cost
-- - If table_scan cost < index cost, MySQL chose the scan

-- Why did the optimizer pick this join order?
-- Check "considered_execution_plans" for all orderings and their costs

-- Why was my index not considered?
-- Check "potential_range_indexes" -> "usable": false with reason
```

## Identifying Bottleneck Stages

### Methodology

```sql
-- 1. Identify the slow query (from slow log or Performance Schema)
-- 2. Get its stage breakdown
-- 3. Focus on the stage consuming the most time

-- Common bottleneck stages and their causes:

-- "executing" takes most time:
--   -> The query is doing real work. Optimize the query plan (indexes, rewrites).

-- "Sending data" takes most time (5.7 terminology):
--   -> The query is scanning and filtering rows. May need better indexes.
--   -> May also mean large result set being sent to client.

-- "Creating tmp table" is significant:
--   -> GROUP BY, DISTINCT, or UNION creating temp tables.
--   -> If disk temp tables, check tmp_table_size and max_heap_table_size.

-- "Creating sort index" is significant:
--   -> ORDER BY not served by an index.
--   -> Consider adding an index that includes ORDER BY columns.

-- "Opening tables" is slow:
--   -> Table cache is too small.
--   -> Check table_open_cache setting.

-- "System lock" or "Waiting for table metadata lock" is slow:
--   -> DDL or long transaction holding metadata lock.
--   -> Check for blocking sessions.
```

### Quick Bottleneck Finder Query

```sql
-- Find the top consuming stage for each of the most recent slow statements
SELECT
    es.EVENT_ID AS stmt_event_id,
    LEFT(es.SQL_TEXT, 80) AS query_preview,
    ROUND(es.TIMER_WAIT / 1e12, 3) AS stmt_time_sec,
    stg.EVENT_NAME AS bottleneck_stage,
    ROUND(stg.TIMER_WAIT / 1e12, 3) AS stage_time_sec,
    ROUND(stg.TIMER_WAIT / es.TIMER_WAIT * 100, 1) AS pct_of_total
FROM performance_schema.events_statements_history_long es
JOIN (
    SELECT NESTING_EVENT_ID, EVENT_NAME, TIMER_WAIT,
           ROW_NUMBER() OVER (PARTITION BY NESTING_EVENT_ID ORDER BY TIMER_WAIT DESC) AS rn
    FROM performance_schema.events_stages_history_long
) stg ON stg.NESTING_EVENT_ID = es.EVENT_ID AND stg.rn = 1
WHERE es.TIMER_WAIT / 1e12 > 1  -- Only statements > 1 second
ORDER BY es.TIMER_WAIT DESC
LIMIT 10;
```

## Profiling Active Queries in Real Time

```sql
-- What is a currently running query doing right now?
SELECT
    t.PROCESSLIST_ID,
    t.PROCESSLIST_INFO AS current_query,
    esc.EVENT_NAME AS current_stage,
    ROUND(esc.TIMER_WAIT / 1e12, 3) AS stage_duration_sec,
    ROUND(essc.TIMER_WAIT / 1e12, 3) AS stmt_elapsed_sec
FROM performance_schema.threads t
JOIN performance_schema.events_stages_current esc
    ON esc.THREAD_ID = t.THREAD_ID
JOIN performance_schema.events_statements_current essc
    ON essc.THREAD_ID = t.THREAD_ID
WHERE t.PROCESSLIST_COMMAND != 'Sleep'
  AND t.PROCESSLIST_INFO IS NOT NULL
ORDER BY essc.TIMER_WAIT DESC;
```

## Best Practices

- Use Performance Schema stages instead of SHOW PROFILE for production profiling
- Enable `events_stages_history_long` to capture stage data for post-mortem analysis
- Combine EXPLAIN ANALYZE (for plan-level timing) with stage profiling (for execution-phase timing)
- Use optimizer trace sparingly -- it has significant overhead and produces large output
- Profile representative queries under realistic load, not just on idle development servers
- Focus on the single stage consuming the most time; optimizing minor stages yields negligible improvement
- Use sys schema views for quick analysis; drop to Performance Schema tables for detailed investigation
- Truncate history tables before profiling a specific query to isolate its events
- Set `optimizer_trace_max_mem_size` large enough (1MB+) to avoid truncated traces
- Disable profiling instrumentation you are not actively using to minimize overhead

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Relying solely on SHOW PROFILE | Deprecated feature, will be removed; limited in production | Migrate to Performance Schema `events_stages_*` tables |
| Not enabling stage consumers in Performance Schema | Stage profiling data is not collected | Enable `events_stages_current/history/history_long` consumers |
| Leaving optimizer trace enabled in production | High CPU and memory overhead per query | Enable only in the profiling session; disable immediately after |
| Profiling on a dev server and assuming production behavior | Different data size, buffer pool state, and concurrency change timing | Profile on production replicas or with representative data |
| Focusing on query parse/optimize time when execution dominates | Wasted effort optimizing millisecond stages when seconds are in execution | Always identify the top-consuming stage first before optimizing |
| Not resetting Performance Schema history before profiling | History contains events from other queries, confusing analysis | TRUNCATE history tables before the test query |
| Ignoring `loops` multiplier in EXPLAIN ANALYZE | Underestimating time spent in nested loop inner tables | Total time = actual_time * loops |

## MySQL Version Notes

**MySQL 5.7**:
- SHOW PROFILE available (deprecated)
- Performance Schema statement and stage events available
- sys schema included (5.7.7+)
- No EXPLAIN ANALYZE
- No EXPLAIN FORMAT=TREE
- "Sending data" stage covers most of execution time (not very granular)
- Optimizer trace available

**MySQL 8.0**:
- EXPLAIN ANALYZE added (8.0.18) with actual timing in tree format
- EXPLAIN FORMAT=TREE added (8.0.16)
- "Sending data" stage split into more granular stages
- Performance Schema always enabled (cannot be fully disabled at runtime)
- Improved sys schema with additional profiling views
- Optimizer trace improvements with more readable output
- Performance Schema setup_instruments enabled by default for statements and stages

**MySQL 8.4 / 9.x**:
- Enhanced EXPLAIN ANALYZE with additional iterator details
- Improved stage granularity for better bottleneck identification
- Performance Schema telemetry component integration
- Better tracing of parallel query execution (where applicable)

## Sources

- [MySQL SHOW PROFILE Statement](https://dev.mysql.com/doc/refman/8.0/en/show-profile.html)
- [MySQL Performance Schema Statement Events](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-statement-tables.html)
- [MySQL Performance Schema Stage Events](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-stage-tables.html)
- [MySQL sys Schema](https://dev.mysql.com/doc/refman/8.0/en/sys-schema.html)
- [MySQL EXPLAIN ANALYZE](https://dev.mysql.com/doc/refman/8.0/en/explain.html#explain-analyze)
- [MySQL Optimizer Trace](https://dev.mysql.com/doc/dev/mysql-server/latest/PAGE_OPT_TRACE.html)
