# Transactions in SQL Server — Isolation Levels, Locking, and Concurrency Control

## Overview

SQL Server implements ACID transactions with a sophisticated locking and row-versioning system that controls how concurrent sessions interact. Understanding transaction isolation levels, lock types, and deadlock handling is essential for building applications that are both correct and performant under concurrent load.

SQL Server supports five isolation levels ranging from READ UNCOMMITTED (maximum concurrency, minimum consistency) to SERIALIZABLE (maximum consistency, minimum concurrency), plus SNAPSHOT isolation which uses row versioning in tempdb to provide point-in-time consistency without blocking readers or writers. Azure SQL Database defaults to READ COMMITTED SNAPSHOT (RCSI), a row-versioned variant of READ COMMITTED that eliminates reader-writer blocking.

Choosing the right isolation level is a critical architectural decision. Most OLTP applications perform well with READ COMMITTED (the default) or RCSI. SNAPSHOT isolation is ideal for reporting queries that need consistent reads without blocking production writes. SERIALIZABLE should be reserved for scenarios requiring absolute consistency, such as financial reconciliation.

## Key Concepts

- **ACID** — Atomicity (all-or-nothing), Consistency (constraints enforced), Isolation (concurrent transactions don't interfere), Durability (committed data survives crashes).
- **Isolation level** — Controls the degree to which one transaction is isolated from modifications made by other concurrent transactions.
- **Lock** — A mechanism that prevents conflicting access. Types include shared (S), exclusive (X), update (U), intent (IS/IX/IU), and schema (Sch-S/Sch-M).
- **Lock escalation** — SQL Server automatically escalates many fine-grained row/page locks to a table lock to reduce memory overhead (threshold ~5000 locks on a single table).
- **Row versioning** — SNAPSHOT and RCSI use tempdb to store prior versions of modified rows, allowing readers to see consistent data without acquiring shared locks.
- **Deadlock** — Two or more sessions each hold a lock the other needs. SQL Server detects deadlocks and kills the least expensive session (error 1205).
- **XACT_ABORT** — When ON, any runtime error automatically rolls back the entire transaction. Recommended for production stored procedures.

## Transaction Isolation Levels

```sql
-- READ UNCOMMITTED: dirty reads allowed, no shared locks acquired
-- Use case: rough row counts, monitoring queries where accuracy isn't critical
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION;
    SELECT COUNT(*) FROM Orders;  -- may read uncommitted data
COMMIT;

-- Equivalent using table hint
SELECT COUNT(*) FROM Orders WITH (NOLOCK);

-- READ COMMITTED (default): only reads committed data
-- Shared locks held only during the read, then released immediately
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE OrderId = 100;  -- S lock acquired and released
    -- Another transaction could modify this row before we read it again
    SELECT * FROM Orders WHERE OrderId = 100;  -- may return different data
COMMIT;

-- REPEATABLE READ: shared locks held until end of transaction
-- No other transaction can modify rows you've read
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE CustomerId = 42;  -- S locks held on all matching rows
    -- Same query will return the same rows (but new phantom rows could appear)
    SELECT * FROM Orders WHERE CustomerId = 42;
COMMIT;

-- SERIALIZABLE: range locks prevent phantom reads
-- Most restrictive, highest blocking
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
    SELECT * FROM Orders WHERE Amount BETWEEN 100 AND 500;
    -- Range lock prevents INSERT/UPDATE/DELETE of rows in this range
    SELECT * FROM Orders WHERE Amount BETWEEN 100 AND 500;  -- guaranteed identical
COMMIT;

-- SNAPSHOT: row versioning, point-in-time consistency
-- Must be enabled at database level first
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
    -- Sees data as of transaction start time (not statement start)
    SELECT * FROM Orders;
    WAITFOR DELAY '00:00:05';
    -- Still sees data as of original transaction start, even if others committed
    SELECT * FROM Orders;  -- guaranteed identical to first read
COMMIT;
```

## READ COMMITTED SNAPSHOT (RCSI)

```sql
-- Enable RCSI at database level (requires exclusive access on-prem, no downtime on Azure SQL)
ALTER DATABASE MyDB SET READ_COMMITTED_SNAPSHOT ON;

-- After enabling, all READ COMMITTED queries automatically use row versioning
-- Readers don't block writers, writers don't block readers
-- Each STATEMENT sees the most recent committed data at statement start

-- RCSI vs SNAPSHOT comparison:
-- RCSI: consistency at STATEMENT level, automatic, default for Azure SQL
-- SNAPSHOT: consistency at TRANSACTION level, explicit opt-in, update conflicts detected

-- Check if RCSI is enabled
SELECT name, is_read_committed_snapshot_on, snapshot_isolation_state_desc
FROM sys.databases
WHERE name = DB_NAME();

-- Monitor version store usage in tempdb
SELECT 
    DB_NAME(database_id) AS database_name,
    reserved_page_count * 8 / 1024 AS version_store_mb
FROM sys.dm_tran_version_store_space_usage;
```

## Lock Types and Behavior

```sql
-- View current locks in the system
SELECT 
    l.request_session_id AS spid,
    DB_NAME(l.resource_database_id) AS db_name,
    l.resource_type,
    l.resource_description,
    l.request_mode,
    l.request_status,
    t.text AS sql_text
FROM sys.dm_tran_locks l
LEFT JOIN sys.dm_exec_requests r ON l.request_session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE l.resource_database_id = DB_ID()
ORDER BY l.request_session_id;

-- Lock compatibility matrix (simplified):
-- S (Shared): compatible with S, IS, IU; blocks X, U
-- X (Exclusive): blocks everything
-- U (Update): compatible with S, IS; blocks U, X (prevents conversion deadlocks)
-- IS/IX/IU (Intent): table-level markers indicating row/page locks exist below

-- Lock hints
SELECT * FROM Orders WITH (HOLDLOCK);     -- hold shared lock until end of transaction
SELECT * FROM Orders WITH (UPDLOCK);      -- acquire update lock (prevents conversion deadlocks)
SELECT * FROM Orders WITH (XLOCK);        -- acquire exclusive lock
SELECT * FROM Orders WITH (ROWLOCK);      -- force row-level locking
SELECT * FROM Orders WITH (PAGLOCK);      -- force page-level locking
SELECT * FROM Orders WITH (TABLOCK);      -- force table-level lock
SELECT * FROM Orders WITH (TABLOCKX);     -- exclusive table lock
SELECT * FROM Orders WITH (READPAST);     -- skip locked rows instead of waiting

-- Lock escalation control per table
ALTER TABLE Orders SET (LOCK_ESCALATION = DISABLE);     -- never escalate (use cautiously)
ALTER TABLE Orders SET (LOCK_ESCALATION = AUTO);        -- escalate to partition level if partitioned
ALTER TABLE Orders SET (LOCK_ESCALATION = TABLE);       -- default: escalate to table level

-- Set lock timeout (milliseconds, -1 = wait forever)
SET LOCK_TIMEOUT 5000;  -- wait 5 seconds then error 1222
```

## Deadlock Detection and Handling

```sql
-- Deadlock-safe pattern with retry
CREATE PROCEDURE dbo.TransferFunds
    @FromAccount INT,
    @ToAccount INT,
    @Amount DECIMAL(12,2)
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;
    
    DECLARE @retry INT = 0;
    DECLARE @maxRetries INT = 3;
    
    WHILE @retry < @maxRetries
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;
            
            -- Always access tables in consistent order to minimize deadlocks
            -- Lock the lower AccountId first
            IF @FromAccount < @ToAccount
            BEGIN
                UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
                UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;
            END
            ELSE
            BEGIN
                UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;
                UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
            END
            
            COMMIT TRANSACTION;
            RETURN 0;  -- success
        END TRY
        BEGIN CATCH
            IF @@TRANCOUNT > 0
                ROLLBACK TRANSACTION;
            
            IF ERROR_NUMBER() = 1205  -- deadlock victim
            BEGIN
                SET @retry = @retry + 1;
                IF @retry < @maxRetries
                BEGIN
                    WAITFOR DELAY '00:00:00.100';  -- brief pause before retry
                    CONTINUE;
                END
            END
            
            -- Non-deadlock error or retries exhausted
            THROW;
        END CATCH
    END
END;

-- Enable deadlock graph capture via Extended Events
CREATE EVENT SESSION [DeadlockCapture] ON SERVER
ADD EVENT sqlserver.xml_deadlock_report
ADD TARGET package0.event_file (
    SET filename = N'/var/opt/mssql/log/deadlocks.xel',
    max_file_size = 50
)
WITH (STARTUP_STATE = ON);
ALTER EVENT SESSION [DeadlockCapture] ON SERVER STATE = START;

-- Query captured deadlock graphs
SELECT 
    event_data.value('(event/@timestamp)[1]', 'DATETIME2') AS deadlock_time,
    event_data AS deadlock_graph
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file('/var/opt/mssql/log/deadlocks*.xel', NULL, NULL, NULL)
) AS xevents;
```

## XACT_ABORT and Error Handling

```sql
-- SET XACT_ABORT ON: auto-rollback on any runtime error
-- Recommended for all stored procedures with explicit transactions
CREATE PROCEDURE dbo.ProcessOrder
    @OrderId INT,
    @CustomerId INT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON;  -- any error rolls back the entire transaction
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        INSERT INTO OrderHistory (OrderId, CustomerId, ProcessedDate)
        VALUES (@OrderId, @CustomerId, GETUTCDATE());
        
        UPDATE Orders SET Status = 'Processed' WHERE OrderId = @OrderId;
        
        UPDATE Customers SET LastOrderDate = GETUTCDATE() WHERE CustomerId = @CustomerId;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        -- Log the error
        INSERT INTO ErrorLog (ErrorMessage, ErrorSeverity, ErrorState, ErrorTime)
        VALUES (@ErrorMessage, @ErrorSeverity, @ErrorState, GETUTCDATE());
        
        -- Re-throw preserving original error info (SQL 2012+)
        THROW;
    END CATCH
END;
```

## Savepoints

```sql
-- SAVE TRANSACTION creates a named savepoint within a transaction
-- ROLLBACK to savepoint undoes work back to that point without ending the transaction
BEGIN TRANSACTION;

    INSERT INTO AuditLog (Action) VALUES ('Starting batch');
    
    SAVE TRANSACTION BeforeUpdates;
    
    UPDATE Inventory SET Quantity = Quantity - 10 WHERE ProductId = 1;
    
    -- Check if quantity went negative
    IF EXISTS (SELECT 1 FROM Inventory WHERE ProductId = 1 AND Quantity < 0)
    BEGIN
        -- Undo just the update, keep the audit log entry
        ROLLBACK TRANSACTION BeforeUpdates;
        INSERT INTO AuditLog (Action) VALUES ('Update rolled back: insufficient inventory');
    END
    ELSE
    BEGIN
        INSERT INTO AuditLog (Action) VALUES ('Update succeeded');
    END

COMMIT TRANSACTION;
-- Both audit log entries are committed; the inventory update is committed only if quantity >= 0
```

## Optimistic Concurrency with rowversion

```sql
-- rowversion (timestamp) column auto-increments on every row update
CREATE TABLE Products (
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(100),
    Price DECIMAL(12,2),
    RowVer ROWVERSION  -- automatically updated on each modification
);

-- Read the product and its current version
DECLARE @currentVersion BINARY(8);
SELECT @currentVersion = RowVer, * FROM Products WHERE ProductId = 42;

-- Update with optimistic concurrency check
UPDATE Products 
SET Price = 29.99
WHERE ProductId = 42 AND RowVer = @currentVersion;

IF @@ROWCOUNT = 0
BEGIN
    -- No rows updated: either deleted or modified by another session
    RAISERROR('Concurrency conflict: row was modified by another user', 16, 1);
END

-- Application-layer pattern (C#/Java pseudo-code):
-- 1. SELECT product with RowVer
-- 2. Display to user for editing
-- 3. UPDATE ... WHERE ProductId = @id AND RowVer = @originalRowVer
-- 4. If @@ROWCOUNT = 0, show conflict resolution UI
```

## Distributed Transactions (MSDTC)

```sql
-- Distributed transactions span multiple SQL Server instances or heterogeneous resources
-- Requires MSDTC (Microsoft Distributed Transaction Coordinator) running on all nodes

-- Explicit distributed transaction
BEGIN DISTRIBUTED TRANSACTION;

    -- Local server operation
    UPDATE LocalDB.dbo.Accounts SET Balance = Balance - 500 WHERE AccountId = 1;
    
    -- Remote server operation via linked server
    UPDATE RemoteServer.RemoteDB.dbo.Accounts SET Balance = Balance + 500 WHERE AccountId = 2;

COMMIT TRANSACTION;

-- Check MSDTC status
SELECT * FROM sys.dm_tran_distributed_transaction_statistics;

-- Note: Distributed transactions add significant overhead
-- Consider alternatives:
--   Saga pattern (compensating transactions)
--   Eventual consistency with message queues
--   Application-level two-phase commit
```

## Monitoring Transaction Activity

```sql
-- Active transactions
SELECT 
    s.session_id,
    s.login_name,
    t.transaction_id,
    t.name AS transaction_name,
    ta.transaction_begin_time,
    DATEDIFF(SECOND, ta.transaction_begin_time, GETDATE()) AS duration_seconds,
    ta.transaction_type,  -- 1=read/write, 2=read-only, 3=system, 4=distributed
    ta.transaction_state, -- 0=not initialized, 1=initialized not started, 2=active
    r.blocking_session_id,
    st.text AS sql_text
FROM sys.dm_tran_active_transactions ta
JOIN sys.dm_tran_session_transactions t ON ta.transaction_id = t.transaction_id
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
LEFT JOIN sys.dm_exec_requests r ON s.session_id = r.session_id
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) st
ORDER BY ta.transaction_begin_time;

-- Long-running open transactions (potential version store bloat)
SELECT 
    s.session_id,
    s.login_name,
    s.host_name,
    ta.transaction_begin_time,
    DATEDIFF(MINUTE, ta.transaction_begin_time, GETDATE()) AS open_minutes,
    st.text AS last_sql
FROM sys.dm_tran_active_transactions ta
JOIN sys.dm_tran_session_transactions t ON ta.transaction_id = t.transaction_id
JOIN sys.dm_exec_sessions s ON t.session_id = s.session_id
LEFT JOIN sys.dm_exec_connections c ON s.session_id = c.session_id
OUTER APPLY sys.dm_exec_sql_text(c.most_recent_sql_handle) st
WHERE DATEDIFF(MINUTE, ta.transaction_begin_time, GETDATE()) > 5
ORDER BY ta.transaction_begin_time;
```

## Best Practices

- Use READ COMMITTED (default) or RCSI for most OLTP workloads. Avoid SERIALIZABLE unless you specifically need phantom protection.
- Always use `SET XACT_ABORT ON` in stored procedures with explicit transactions to ensure automatic rollback on errors.
- Access tables in a consistent order across all stored procedures to minimize deadlocks.
- Keep transactions as short as possible — do validation, computation, and I/O outside the transaction boundaries.
- Use optimistic concurrency (rowversion) for read-modify-write patterns to avoid long-held locks.
- Implement deadlock retry logic in application code (error 1205) with 3 retries and exponential backoff.
- Use UPDLOCK hint on SELECT when you intend to update the selected rows, preventing conversion deadlocks.
- Monitor version store size in tempdb when using RCSI or SNAPSHOT isolation.
- Avoid holding transactions open during user think-time (common in desktop apps).
- Use TRY/CATCH with THROW (not RAISERROR) in SQL Server 2012+ for cleaner error re-throwing.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using NOLOCK everywhere | Dirty reads, phantom rows, incorrect aggregates | Use RCSI for non-blocking reads with consistency |
| Leaving transactions open during user interaction | Long lock durations, blocking chains | Keep transactions short; read outside, write inside |
| Not checking @@TRANCOUNT before ROLLBACK | Rolling back wrong transaction level | Check `IF @@TRANCOUNT > 0 ROLLBACK` in CATCH |
| Ignoring deadlock error 1205 | Application crashes on contention | Add retry logic (3 attempts with backoff) |
| Using SERIALIZABLE without understanding | Extreme blocking, range locks | Use SNAPSHOT for consistent reads without blocking |
| Accessing tables in different orders across procedures | Frequent deadlocks | Establish and document a consistent table access order |
| Not monitoring version store with RCSI/SNAPSHOT | tempdb bloat from long-running transactions | Monitor `sys.dm_tran_version_store_space_usage` |

## SQL Server Version Notes

- **SQL Server 2016** — All isolation levels available. In-Memory OLTP (Hekaton) uses optimistic MVCC natively. Accelerated database recovery not yet available.
- **SQL Server 2019** — Accelerated Database Recovery (ADR) drastically reduces rollback time and version cleanup. Scalar UDF inlining reduces transaction overhead.
- **SQL Server 2022** — ADR improvements (multi-level persistent version store). Optimized locking reduces lock memory for UPDATE operations. SNAPSHOT isolation versioning performance improvements.
- **Azure SQL Database** — RCSI enabled by default. SNAPSHOT isolation available. Deadlock detection is faster (sub-second). Automatic tuning may influence lock behavior.
- **Azure SQL Managed Instance** — Same behavior as on-premises SQL Server. RCSI not enabled by default (must opt in).

## Sources

- https://learn.microsoft.com/en-us/sql/t-sql/statements/set-transaction-isolation-level-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-transaction-locking-and-row-versioning-guide
- https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-tran-locks-transact-sql
- https://learn.microsoft.com/en-us/sql/relational-databases/errors-events/mssqlserver-1205-database-engine-error
- https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-deadlocks-guide
- https://learn.microsoft.com/en-us/sql/relational-databases/accelerated-database-recovery-concepts
