# MySQL Transactions — ACID, Isolation Levels, Locking, and Deadlock Handling

## Overview

Transactions in MySQL are provided by the InnoDB storage engine (the default since MySQL 5.5). A transaction groups one or more SQL statements into an atomic unit of work: either all changes are committed, or all are rolled back. InnoDB implements full ACID compliance with multi-version concurrency control (MVCC), row-level locking, and automatic deadlock detection.

Understanding MySQL's transaction model is critical for application developers because the default isolation level (REPEATABLE READ) behaves differently from most other databases (which default to READ COMMITTED). InnoDB's REPEATABLE READ uses consistent reads (snapshots) for SELECT and next-key locks for locking reads, which prevents phantom rows but can cause unexpected lock contention.

This skill covers ACID properties, isolation levels, lock types, deadlock detection and handling, autocommit behavior, savepoints, and XA distributed transactions.

## Key Concepts

- **ACID**: Atomicity (all or nothing), Consistency (constraints enforced), Isolation (concurrent transactions do not interfere), Durability (committed data survives crashes).
- **MVCC (Multi-Version Concurrency Control)**: InnoDB maintains multiple versions of each row. Readers see a consistent snapshot without blocking writers. Writers create new versions rather than overwriting.
- **Consistent read (snapshot)**: A SELECT without FOR UPDATE/LOCK IN SHARE MODE reads from the transaction's snapshot, not the current state. The snapshot is taken at the time of the first read in REPEATABLE READ, or at each statement in READ COMMITTED.
- **Locking read**: `SELECT ... FOR UPDATE` (exclusive lock) or `SELECT ... FOR SHARE` / `LOCK IN SHARE MODE` (shared lock). These read current data and acquire locks.
- **Gap lock**: A lock on the gap between index records. Prevents other transactions from inserting into that gap. Only used in REPEATABLE READ.
- **Next-key lock**: A combination of a record lock and a gap lock on the gap before that record. InnoDB's default locking mechanism in REPEATABLE READ.

## ACID Properties in InnoDB

```sql
-- Atomicity: all-or-nothing
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- credit
COMMIT;  -- both succeed, or...
-- ROLLBACK;  -- ...both are undone

-- Consistency: constraints enforced at commit
-- Foreign keys, unique constraints, check constraints (8.0.16+) are validated

-- Isolation: concurrent transactions see consistent data
-- Controlled by isolation level (see below)

-- Durability: committed data is safe
-- Controlled by innodb_flush_log_at_trx_commit:
--   1 = flush to disk on each commit (safest, default)
--   2 = flush to OS cache on each commit (1-second durability gap)
--   0 = flush every second regardless of commits (fastest, least safe)
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
```

## Isolation Levels

```sql
-- Check current isolation level
SELECT @@transaction_isolation;         -- 8.0+
SELECT @@tx_isolation;                  -- 5.7

-- Set isolation level for the next transaction
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Set for the session
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Set globally (all new sessions)
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### READ UNCOMMITTED

```sql
-- Can read uncommitted changes from other transactions (dirty reads)
-- Rarely used. No MVCC snapshot, no gap locks.
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM accounts WHERE id = 1;
-- May see data that another transaction has not yet committed
COMMIT;
```

### READ COMMITTED

```sql
-- Each SELECT reads the most recently committed version
-- No dirty reads. Phantom rows possible.
-- Each statement gets a fresh snapshot (not the transaction start).
-- Only record locks, no gap locks --> less lock contention.
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
SELECT * FROM accounts WHERE balance > 1000;  -- snapshot at this moment
-- Another transaction inserts a row with balance = 5000 and commits
SELECT * FROM accounts WHERE balance > 1000;  -- sees the new row (phantom)
COMMIT;
```

### REPEATABLE READ (InnoDB Default)

```sql
-- Snapshot taken at first read. All subsequent reads see the same snapshot.
-- Gap locks prevent phantoms for locking reads.
-- Non-locking reads do NOT see newly committed rows (consistent snapshot).
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

START TRANSACTION;
SELECT * FROM accounts WHERE balance > 1000;  -- snapshot established here
-- Another transaction inserts a row with balance = 5000 and commits
SELECT * FROM accounts WHERE balance > 1000;  -- same result (no phantom)
COMMIT;

-- BUT: locking reads (FOR UPDATE) do see current data
START TRANSACTION;
SELECT * FROM accounts WHERE balance > 1000 FOR UPDATE;
-- This acquires next-key locks on the range, blocking other inserts
COMMIT;
```

### SERIALIZABLE

```sql
-- All SELECTs are implicitly converted to SELECT ... FOR SHARE
-- Maximum isolation, maximum lock contention
-- Essentially: REPEATABLE READ with every SELECT being a locking read
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

START TRANSACTION;
SELECT * FROM accounts WHERE id = 1;  -- acquires shared lock
-- Another transaction trying to UPDATE this row will block
COMMIT;
```

## Lock Types

### Record locks

```sql
-- Exclusive lock on a specific row
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Shared lock on a specific row
SELECT * FROM accounts WHERE id = 1 FOR SHARE;       -- 8.0+
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;  -- 5.7/8.0

-- NOWAIT and SKIP LOCKED (8.0+)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;       -- error if locked
SELECT * FROM accounts WHERE status = 'pending'
    FOR UPDATE SKIP LOCKED LIMIT 5;  -- skip locked rows, useful for job queues
```

### Gap locks and next-key locks

```sql
-- Given an index with values 10, 20, 30:
-- Gap locks cover: (-inf, 10), (10, 20), (20, 30), (30, +inf)
-- Next-key locks cover: (-inf, 10], (10, 20], (20, 30], (30, +inf)

-- Example: this locks the gap between 10 and 20
START TRANSACTION;
SELECT * FROM t WHERE id BETWEEN 12 AND 18 FOR UPDATE;
-- No rows matched, but the gap (10, 20) is locked
-- Another transaction cannot INSERT id=15

-- Gap locks only exist in REPEATABLE READ and SERIALIZABLE
-- In READ COMMITTED, only record locks are used
```

### Intention locks

```sql
-- InnoDB uses table-level intention locks for compatibility checking
-- IS (Intention Shared): transaction intends to set shared locks on rows
-- IX (Intention Exclusive): transaction intends to set exclusive locks on rows
-- These are automatic and transparent to the user

-- View current locks
SELECT * FROM performance_schema.data_locks;        -- 8.0+
SELECT * FROM information_schema.innodb_locks;       -- 5.7
SELECT * FROM information_schema.innodb_lock_waits;  -- 5.7
```

## Deadlock Detection and Handling

```sql
-- InnoDB automatically detects deadlocks and rolls back the "cheapest" transaction
-- (the one with the fewest undo log entries)

-- View the most recent deadlock
SHOW ENGINE INNODB STATUS\G
-- Look for the "LATEST DETECTED DEADLOCK" section

-- Monitor deadlocks in performance_schema (8.0+)
SELECT * FROM performance_schema.data_lock_waits;

-- Deadlock detection settings
SHOW VARIABLES LIKE 'innodb_deadlock_detect';    -- ON by default
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';  -- 50 seconds default
```

### Application-side deadlock handling

```python
import mysql.connector
import time

MAX_RETRIES = 3

def execute_with_retry(conn, operations):
    for attempt in range(MAX_RETRIES):
        try:
            conn.start_transaction()
            operations(conn)
            conn.commit()
            return  # success
        except mysql.connector.errors.InternalError as e:
            conn.rollback()
            if e.errno == 1213:  # ER_LOCK_DEADLOCK
                if attempt < MAX_RETRIES - 1:
                    wait = (2 ** attempt) * 0.1  # exponential backoff
                    time.sleep(wait)
                    continue
            raise  # re-raise non-deadlock errors or final attempt
```

```java
// Java deadlock retry
int maxRetries = 3;
for (int attempt = 0; attempt < maxRetries; attempt++) {
    try {
        conn.setAutoCommit(false);
        // ... execute statements ...
        conn.commit();
        break;
    } catch (SQLException e) {
        conn.rollback();
        if (e.getErrorCode() == 1213 && attempt < maxRetries - 1) {
            Thread.sleep((long) Math.pow(2, attempt) * 100);
            continue;
        }
        throw e;
    }
}
```

## Autocommit Behavior

```sql
-- MySQL defaults to autocommit = ON
-- Each individual statement is its own transaction (implicitly committed)
SHOW VARIABLES LIKE 'autocommit';

-- Disable autocommit for the session
SET autocommit = 0;
-- Now every statement starts an implicit transaction
-- You MUST call COMMIT or ROLLBACK

-- START TRANSACTION temporarily disables autocommit
-- until COMMIT or ROLLBACK
SET autocommit = 1;
START TRANSACTION;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 42);
COMMIT;  -- autocommit is re-enabled after this

-- DDL statements cause an implicit commit
START TRANSACTION;
INSERT INTO t1 VALUES (1);
ALTER TABLE t2 ADD COLUMN x INT;  -- implicit COMMIT happens here
ROLLBACK;  -- too late, the INSERT is already committed
```

## Savepoints

```sql
START TRANSACTION;

INSERT INTO orders (user_id, total) VALUES (1, 99.99);
SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 42);
-- Something went wrong with this item
ROLLBACK TO SAVEPOINT after_order;

-- The order insert is still intact
INSERT INTO order_items (order_id, product_id) VALUES (LAST_INSERT_ID(), 43);

RELEASE SAVEPOINT after_order;  -- optional, frees the savepoint name
COMMIT;
```

## XA Transactions (Distributed)

```sql
-- XA transactions span multiple MySQL instances or heterogeneous databases
-- Two-phase commit protocol

-- Phase 1: Prepare
XA START 'txn_001';
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
XA END 'txn_001';
XA PREPARE 'txn_001';

-- Phase 2: Commit (after all participants have prepared)
XA COMMIT 'txn_001';

-- Or rollback
XA ROLLBACK 'txn_001';

-- View prepared XA transactions (for recovery)
XA RECOVER;

-- XA limitations:
-- No support for replication filters
-- Binary log must use ROW format
-- InnoDB only
-- Performance overhead of two-phase commit
```

## innodb_lock_wait_timeout

```sql
-- How long a transaction waits for a row lock before giving up
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';  -- default: 50 seconds

-- Set per session (for batch operations)
SET SESSION innodb_lock_wait_timeout = 10;

-- When exceeded, you get:
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
-- Unlike deadlocks, lock wait timeouts do NOT automatically rollback the transaction
-- (only the timed-out statement is rolled back unless innodb_rollback_on_timeout = ON)

SHOW VARIABLES LIKE 'innodb_rollback_on_timeout';  -- OFF by default
```

## Best Practices

- Use READ COMMITTED isolation level if your application can tolerate phantom reads and you want reduced lock contention. Many production MySQL deployments (including those using replication) switch from the default REPEATABLE READ to READ COMMITTED.
- Keep transactions as short as possible. Long-running transactions hold locks and prevent undo log purging, growing the undo tablespace.
- Always handle deadlock errors (errno 1213) in application code with retry logic and exponential backoff.
- Access tables and rows in a consistent order across all transactions to minimize deadlock frequency.
- Use `SELECT ... FOR UPDATE SKIP LOCKED` (8.0+) for job queue patterns instead of polling with `SLEEP`.
- Avoid mixing DDL and DML in the same transaction -- DDL causes an implicit commit.
- Monitor `Innodb_row_lock_waits` and `Innodb_row_lock_time` status variables to detect lock contention trends.
- Set `innodb_print_all_deadlocks = ON` to log every deadlock to the error log, not just the latest one.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Assuming REPEATABLE READ prevents all anomalies | Write skew is possible (two transactions read overlapping data and make non-conflicting writes) | Use `SELECT ... FOR UPDATE` or switch to SERIALIZABLE for critical sections |
| Long-running transactions with autocommit OFF | Undo log bloat, lock accumulation, replication lag | Keep transactions short; commit or rollback promptly |
| Not retrying on deadlock (errno 1213) | Application returns error to user on a transient condition | Implement retry loop with backoff |
| DDL inside a transaction | Implicit commit before DDL, partial atomicity | Separate DDL from DML transactions |
| Assuming lock wait timeout rolls back the whole transaction | Only the timed-out statement is rolled back (unless `innodb_rollback_on_timeout = ON`) | Explicitly ROLLBACK after catching error 1205 |
| Using MyISAM tables expecting transactions | MyISAM has no transaction support | Use InnoDB exclusively |
| Not indexing columns used in WHERE of UPDATE/DELETE | Full table scan acquires locks on every row | Add appropriate indexes |

## MySQL Version Notes

- **5.7**: Default isolation is REPEATABLE READ. `tx_isolation` variable. `SELECT ... LOCK IN SHARE MODE` syntax. XA support with some limitations. No `SKIP LOCKED` or `NOWAIT`.
- **8.0**: `transaction_isolation` replaces `tx_isolation`. Added `FOR SHARE` as alias for `LOCK IN SHARE MODE`. Added `NOWAIT` and `SKIP LOCKED` modifiers. `data_locks` and `data_lock_waits` tables in performance_schema replace `INFORMATION_SCHEMA.INNODB_LOCKS`. Check constraints added (8.0.16). Atomic DDL for InnoDB.
- **8.4 / 9.x**: Improved deadlock detection performance. Better undo log purging under high concurrency. `INFORMATION_SCHEMA.INNODB_LOCKS` fully removed (use `performance_schema.data_locks`).

## Sources

- [MySQL InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- [MySQL Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [InnoDB Deadlocks](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
- [XA Transactions](https://dev.mysql.com/doc/refman/8.0/en/xa.html)
