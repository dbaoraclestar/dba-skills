# sqlcmd and bcp -- Command-Line Tools for SQL Server

## Overview

`sqlcmd` and `bcp` are the foundational command-line tools for SQL Server. `sqlcmd` executes T-SQL statements, scripts, and stored procedures from the terminal, making it essential for automation, CI/CD pipelines, and headless server administration. `bcp` (Bulk Copy Program) handles high-speed data import and export between SQL Server tables and flat files, with performance-tuning options for minimal logging, batch sizing, and parallel loads.

Two versions of sqlcmd exist: the **legacy** C-based sqlcmd (included with SQL Server) and the **new** Go-based sqlcmd (installable separately, cross-platform, supports Azure AD authentication natively). The Go-based version is the recommended path forward. bcp remains a single implementation included with SQL Server client tools.

Use this skill when automating SQL Server operations via scripts, performing bulk data loads or extracts, or integrating SQL Server commands into shell scripts and CI/CD pipelines.

## Key Concepts

- **sqlcmd**: Interactive and scripting mode SQL client. Supports variable substitution, output redirection, batch separators (GO), and error handling.
- **bcp**: Bulk Copy Program. Imports data from files to tables or exports table/query results to files. Uses the TDS protocol for high-throughput transfer.
- **Format file**: XML or non-XML descriptor that maps file columns to table columns for bcp. Required for complex data layouts.
- **BULK INSERT**: T-SQL statement (alternative to bcp) that loads file data into a table from within a SQL session.
- **Minimal logging**: Bulk operations can be minimally logged (bulk_logged or simple recovery model) for dramatically faster loads.

## sqlcmd Connection Options

```bash
# Basic connection (SQL authentication)
sqlcmd -S myserver -U appuser -P 'SecureP@ss!' -d MyApp

# Windows authentication
sqlcmd -S myserver -d MyApp -E

# Named instance
sqlcmd -S "myserver\INST01" -d MyApp -E

# Specific port
sqlcmd -S myserver,1434 -d MyApp -U appuser -P 'SecureP@ss!'

# Azure SQL with encryption
sqlcmd -S myserver.database.windows.net -d MyApp -U appuser -P 'SecureP@ss!' -N -C

# Go-based sqlcmd with Azure AD
sqlcmd -S myserver.database.windows.net -d MyApp --authentication-method=ActiveDirectoryDefault

# Connection via environment variable
export SQLCMDPASSWORD='SecureP@ss!'
sqlcmd -S myserver -U appuser -d MyApp
```

## sqlcmd Scripting Mode

```bash
# Execute a single query
sqlcmd -S myserver -d MyApp -Q "SELECT @@VERSION"

# Execute a SQL script file
sqlcmd -S myserver -d MyApp -i migrate.sql

# Execute multiple script files in order
sqlcmd -S myserver -d MyApp -i setup.sql -i seed.sql -i verify.sql

# Output results to file
sqlcmd -S myserver -d MyApp -Q "SELECT * FROM dbo.Customers" -o customers.txt

# Output as CSV (tab-separated by default; use -s for custom separator)
sqlcmd -S myserver -d MyApp -Q "SELECT * FROM dbo.Customers" -s "," -W -o customers.csv

# Suppress headers and row count
sqlcmd -S myserver -d MyApp -Q "SET NOCOUNT ON; SELECT Email FROM dbo.Customers" -h -1 -W

# Error handling: exit on error
sqlcmd -S myserver -d MyApp -i migrate.sql -b
# -b flag: abort batch on error and set ERRORLEVEL to 1

# Specific error handling in scripts
sqlcmd -S myserver -d MyApp -V 16 -i migrate.sql
# -V 16: only report severity 16 and above as errors
```

## Variable Substitution

```bash
# Pass variables from command line
sqlcmd -S myserver -d MyApp \
    -v DatabaseName="MyApp" \
    -v SchemaName="dbo" \
    -v TargetDate="2025-01-01" \
    -i parameterized_report.sql
```

```sql
-- parameterized_report.sql
:setvar DatabaseName "MyApp"
:setvar SchemaName "dbo"

USE [$(DatabaseName)];
GO

SELECT
    CustomerID,
    FirstName,
    LastName,
    Email,
    CreatedAt
FROM [$(SchemaName)].Customers
WHERE CreatedAt >= '$(TargetDate)'
ORDER BY CreatedAt DESC;
GO

-- Conditional execution
:setvar RunCleanup "YES"

IF '$(RunCleanup)' = 'YES'
BEGIN
    DELETE FROM [$(SchemaName)].AuditLog
    WHERE ChangedAt < DATEADD(YEAR, -2, GETDATE());
    PRINT 'Cleanup completed: ' + CAST(@@ROWCOUNT AS VARCHAR) + ' rows deleted.';
END
GO
```

## sqlcmd Output Formatting

```bash
# Default output: column headers + dashes + data + row count
sqlcmd -S myserver -d MyApp -Q "SELECT TOP 5 FirstName, LastName FROM dbo.Customers"
# FirstName   LastName
# ----------- -----------
# Jane        Doe
# John        Smith
# (5 rows affected)

# Clean output: no headers, no row count, trimmed whitespace
sqlcmd -S myserver -d MyApp -h -1 -W \
    -Q "SET NOCOUNT ON; SELECT FirstName, LastName FROM dbo.Customers"
# Jane Doe
# John Smith

# Custom column separator
sqlcmd -S myserver -d MyApp -s "|" -W \
    -Q "SET NOCOUNT ON; SELECT FirstName, LastName, Email FROM dbo.Customers"
# Jane|Doe|jane@example.com
# John|Smith|john@example.com

# Fixed-width output for reports
sqlcmd -S myserver -d MyApp -w 200 \
    -Q "SELECT * FROM dbo.Customers"
# -w 200: set screen width to 200 characters (prevents line wrapping)

# XML output
sqlcmd -S myserver -d MyApp \
    -Q "SELECT * FROM dbo.Customers FOR XML PATH('Customer'), ROOT('Customers')" \
    -o customers.xml -y 0
# -y 0: unlimited XML display width
```

## sqlcmd Batch Operations

```sql
-- Interactive mode commands (prefixed with colon)
:connect myserver          -- Connect to a different server
:r setup.sql               -- Include/execute another script file
:setvar varname "value"    -- Set a scripting variable
:listvar                   -- List all variables
:error stderr.log          -- Redirect errors to file
:out results.txt           -- Redirect output to file
:out stdout                -- Reset output to screen
:reset                     -- Clear the statement cache
:quit / :exit              -- Exit sqlcmd

-- GO with count (repeat a batch)
INSERT INTO dbo.TestData (Value) VALUES (NEWID());
GO 1000
-- Executes the INSERT statement 1000 times
```

## bcp Export (Data Out)

```bash
# Export entire table to file
bcp dbo.Customers out customers.dat -S myserver -d MyApp -T -c
# -T: trusted connection (Windows auth)
# -c: character mode (text, tab-delimited)

# Export with query (queryout)
bcp "SELECT CustomerID, FirstName, LastName, Email FROM dbo.Customers WHERE IsActive = 1" \
    queryout active_customers.csv -S myserver -d MyApp -T -c -t "," -r "\n"
# -t ",": field terminator (comma)
# -r "\n": row terminator (newline)

# Export with native format (binary, fastest for SQL-to-SQL transfer)
bcp dbo.Customers out customers.bcp -S myserver -d MyApp -T -n
# -n: native mode (binary). Use for moving data between SQL Server instances.

# Export with Unicode
bcp dbo.Customers out customers_unicode.dat -S myserver -d MyApp -T -w
# -w: Unicode character mode (NCHAR/NVARCHAR friendly)

# Generate format file (for customized imports)
bcp dbo.Customers format nul -S myserver -d MyApp -T -c -f customers.fmt
bcp dbo.Customers format nul -S myserver -d MyApp -T -c -x -f customers.xml
# -x: generate XML format file
```

## bcp Import (Data In)

```bash
# Import from file
bcp dbo.Customers in customers.dat -S myserver -d MyApp -T -c

# Import CSV with custom delimiters
bcp dbo.Customers in customers.csv -S myserver -d MyApp -T \
    -c -t "," -r "\n" -F 2
# -F 2: skip first row (header)

# Import with format file
bcp dbo.Customers in customers.dat -S myserver -d MyApp -T -f customers.fmt

# Import with performance tuning
bcp dbo.Customers in customers.dat -S myserver -d MyApp -T -c \
    -b 10000 \
    -h "TABLOCK" \
    -a 65535
# -b 10000: batch size (rows per batch, enables partial rollback)
# -h "TABLOCK": table-level lock for minimal logging
# -a 65535: network packet size (max 65535 bytes)

# Import with error file
bcp dbo.Customers in customers.dat -S myserver -d MyApp -T -c \
    -e errors.log \
    -m 100
# -e: error file for rejected rows
# -m 100: maximum number of errors before abort
```

## BULK INSERT (T-SQL Alternative to bcp)

```sql
-- Basic BULK INSERT
BULK INSERT dbo.Customers
FROM 'C:\Data\customers.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2,          -- Skip header row
    CODEPAGE = '65001',    -- UTF-8
    TABLOCK,               -- Table lock for minimal logging
    BATCHSIZE = 10000,
    ERRORFILE = 'C:\Data\customers_errors.log',
    MAXERRORS = 100
);
GO

-- BULK INSERT with format file
BULK INSERT dbo.Customers
FROM 'C:\Data\customers.dat'
WITH (
    FORMATFILE = 'C:\Data\customers.xml',
    TABLOCK,
    BATCHSIZE = 50000
);
GO

-- OPENROWSET BULK for SELECT-style loading
INSERT INTO dbo.Customers (FirstName, LastName, Email)
SELECT FirstName, LastName, Email
FROM OPENROWSET(
    BULK 'C:\Data\customers.csv',
    FORMATFILE = 'C:\Data\customers.xml',
    FIRSTROW = 2
) AS Source;
GO
```

## Performance Tuning for Bulk Loads

```sql
-- Step 1: Set recovery model for minimal logging
ALTER DATABASE MyApp SET RECOVERY BULK_LOGGED;

-- Step 2: Drop non-clustered indexes before bulk load
DROP INDEX IX_Customers_Email ON dbo.Customers;

-- Step 3: Run bulk load with TABLOCK
BULK INSERT dbo.Customers
FROM 'C:\Data\large_dataset.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2,
    TABLOCK,
    BATCHSIZE = 100000,
    ORDER (CustomerID ASC)  -- Match clustered index order
);

-- Step 4: Rebuild indexes
CREATE NONCLUSTERED INDEX IX_Customers_Email
    ON dbo.Customers (Email)
    WITH (SORT_IN_TEMPDB = ON, ONLINE = ON, MAXDOP = 4);

-- Step 5: Update statistics
UPDATE STATISTICS dbo.Customers WITH FULLSCAN;

-- Step 6: Restore recovery model
ALTER DATABASE MyApp SET RECOVERY FULL;

-- Step 7: Take a full backup (required after BULK_LOGGED)
BACKUP DATABASE MyApp TO DISK = 'C:\Backups\MyApp_post_bulk.bak';
```

```bash
# Parallel bcp import (split file, multiple bcp processes)
# Split large file into chunks
split -l 1000000 large_dataset.csv chunk_

# Run multiple bcp imports concurrently
for chunk in chunk_*; do
    bcp dbo.StagingTable in "$chunk" -S myserver -d MyApp -T -c -t "," \
        -b 50000 -h "TABLOCK" &
done
wait
echo "All chunks loaded."
```

## Best Practices

- Use the Go-based sqlcmd for new projects -- it is cross-platform and supports Azure AD natively
- Always use `-b` flag with sqlcmd scripts to abort on error and return non-zero exit code
- Use variable substitution (`:setvar` / `-v`) to make scripts environment-agnostic
- For bcp imports, use TABLOCK + BATCHSIZE + ORDER for minimal logging and maximum throughput
- Always generate a format file for complex table layouts to avoid column mapping issues
- Drop non-clustered indexes before large bulk loads, rebuild after
- Set BULK_LOGGED recovery during large loads, then switch back to FULL and take a backup
- Use `-e` error file with bcp to capture rejected rows without aborting the entire load
- Validate row counts after bcp import: compare source file line count with table row count

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not using `-b` flag in sqlcmd CI/CD scripts | Script continues after errors; pipeline reports success despite failures | Always pass `-b` to abort on error and check `$?` / `%ERRORLEVEL%` |
| Using bcp without TABLOCK on empty table | Fully logged operation, 10-100x slower than minimally logged | Add `-h "TABLOCK"` for bcp or `WITH (TABLOCK)` for BULK INSERT |
| Forgetting `-F 2` to skip CSV header row | Header row inserted as data, causing type conversion errors | Use `-F 2` for bcp or `FIRSTROW = 2` for BULK INSERT |
| Not taking a backup after BULK_LOGGED recovery | Log chain broken; point-in-time recovery impossible until next full backup | Always take a full backup immediately after switching back to FULL recovery |
| Using legacy sqlcmd with Azure AD authentication | Legacy sqlcmd does not support Azure AD tokens | Install Go-based sqlcmd: `winget install sqlcmd` or `brew install sqlcmd` |

## SQL Server Version Notes

- **SQL Server 2016**: Legacy sqlcmd and bcp included. BULK INSERT supports `DATA_SOURCE` for Azure Blob Storage files. JSON output available via `FOR JSON` in sqlcmd queries. bcp `-G` flag for Azure AD added in later tool updates.
- **SQL Server 2019**: Go-based sqlcmd introduced (separate install). bcp supports UTF-8 codepage (65001) natively. `BULK INSERT ... WITH (FORMAT = 'CSV')` added for proper CSV parsing (handles quoted fields). Accelerated Database Recovery speeds up large transaction rollback during failed bulk loads.
- **SQL Server 2022**: Go-based sqlcmd is the recommended default. `BULK INSERT` supports `FORMAT = 'PARQUET'` (preview with S3-compatible storage). Ledger tables support bulk insert. `sqlcmd` supports `--authentication-method` flag for multiple auth types including managed identity.

## Sources

- https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility
- https://learn.microsoft.com/en-us/sql/tools/sqlcmd/go-sqlcmd-utility
- https://learn.microsoft.com/en-us/sql/tools/bcp-utility
- https://learn.microsoft.com/en-us/sql/relational-databases/import-export/bulk-import-and-export-of-data-sql-server
- https://learn.microsoft.com/en-us/sql/relational-databases/import-export/use-a-format-file-to-bulk-import-data
- https://learn.microsoft.com/en-us/sql/t-sql/statements/bulk-insert-transact-sql
