# SSMS and Azure Data Studio -- SQL Server Management Tools Compared

## Overview

SQL Server Management Studio (SSMS) and Azure Data Studio (ADS) are the two primary graphical tools for SQL Server administration and development. SSMS is the traditional Windows-only powerhouse with deep integration into every SQL Server feature. Azure Data Studio is a modern, cross-platform (Windows, macOS, Linux) tool built on VS Code technology, focusing on query editing, notebooks, and extensibility.

Most SQL Server professionals use both: SSMS for administrative tasks (maintenance plans, Agent jobs, security configuration, Activity Monitor) and Azure Data Studio for day-to-day query writing, source control integration, and collaborative notebooks. Teams working on macOS or Linux must use Azure Data Studio since SSMS is Windows-only.

Use this skill when choosing between the tools, learning key features and shortcuts, or understanding the workflows each tool excels at.

## Key Concepts

- **SSMS (SQL Server Management Studio)**: Full-featured Windows GUI for SQL Server administration. Includes Object Explorer, Activity Monitor, query editor with execution plans, maintenance plan designer, SQL Server Agent management, and security configuration.
- **Azure Data Studio**: Lightweight cross-platform editor for SQL Server and Azure SQL. Features include IntelliSense, integrated terminal, source control, Jupyter-style notebooks, extensions marketplace, and dashboard widgets.
- **Object Explorer**: SSMS tree view showing servers, databases, tables, views, procedures. Supports drag-and-drop scripting.
- **Execution plan**: Visual representation of how SQL Server executes a query. Available in both tools.

## SSMS Key Features

### Object Explorer

```
-- Object Explorer hierarchy:
Server
  +-- Databases
  |     +-- System Databases (master, model, msdb, tempdb)
  |     +-- MyApp
  |           +-- Tables
  |           |     +-- dbo.Customers (Columns, Keys, Indexes, Statistics, Triggers)
  |           |     +-- dbo.Orders
  |           +-- Views
  |           +-- Programmability
  |           |     +-- Stored Procedures
  |           |     +-- Functions
  |           +-- Security (Users, Roles, Schemas)
  +-- Security (Logins, Server Roles)
  +-- Server Objects (Linked Servers, Endpoints)
  +-- SQL Server Agent (Jobs, Alerts, Operators)
  +-- Management (Maintenance Plans, Extended Events)
  +-- Always On High Availability
```

### Activity Monitor

```
-- Launch: right-click server node > Activity Monitor
-- Or: Ctrl+Alt+A

-- Activity Monitor panels:
1. Overview: % Processor Time, Waiting Tasks, Database I/O, Batch Requests/sec
2. Processes: Active sessions (SPID, database, status, wait type, command)
3. Resource Waits: Top wait types with cumulative wait time
4. Data File I/O: Read/write latency per file
5. Recent Expensive Queries: Top queries by CPU, reads, writes, duration
```

### Execution Plan Viewer

```sql
-- Include Actual Execution Plan: Ctrl+M (toggle before executing)
-- Display Estimated Execution Plan: Ctrl+L

-- Right-click plan > Show Execution Plan XML for detailed analysis
-- Warnings appear as yellow triangles on operators:
--   - Missing index suggestions
--   - Implicit conversions
--   - Excessive memory grants
--   - Spills to tempdb

-- Save execution plan: .sqlplan file (XML format)
-- Compare plans: File > Open > select two .sqlplan files
```

### SSMS Shortcuts

```
-- Essential SSMS keyboard shortcuts:

-- Query execution
F5 / Ctrl+E          Execute query
Ctrl+L               Show estimated execution plan
Ctrl+M               Toggle actual execution plan
Ctrl+Shift+E         Execute to file

-- Editor
Ctrl+K, Ctrl+C       Comment selection
Ctrl+K, Ctrl+U       Uncomment selection
Ctrl+Shift+U         UPPERCASE selection
Ctrl+Shift+L         lowercase selection
Ctrl+R               Toggle results pane
Ctrl+T               Results to text
Ctrl+D               Results to grid
Alt+F1               sp_help on selected text (highlight table name, press Alt+F1)

-- Navigation
Ctrl+G               Go to line number
Ctrl+F               Find
Ctrl+H               Find and Replace
F8                   Object Explorer
Ctrl+U               Change database dropdown

-- Object Explorer
F7                   Object Explorer Details (table view)
Ctrl+Alt+A           Activity Monitor

-- Scripting (from Object Explorer)
Right-click > Script As > CREATE TO / ALTER TO / DROP AND CREATE TO
```

### Template Explorer

```sql
-- View > Template Explorer (Ctrl+Alt+T)
-- Templates provide parameterized SQL snippets:

-- Example: Create Table template
-- Ctrl+Shift+M to fill in template parameters

CREATE TABLE <schema_name, sysname, dbo>.<table_name, sysname, MyTable> (
    <column_1, sysname, ID> <datatype_1, sysname, INT> IDENTITY(1,1) NOT NULL,
    <column_2, sysname, Name> <datatype_2, sysname, NVARCHAR(100)> NOT NULL,
    CONSTRAINT <pk_name, sysname, PK_MyTable> PRIMARY KEY (<column_1, sysname, ID>)
);
GO
```

### Query Execution Options

```
-- Tools > Options > Query Execution > SQL Server > Advanced:

SET STATISTICS IO ON     -- Show logical/physical reads per table
SET STATISTICS TIME ON   -- Show CPU and elapsed time per statement
SET NOCOUNT ON           -- Suppress rowcount messages
SET ANSI_NULLS ON        -- Standard NULL comparison behavior
SET QUOTED_IDENTIFIER ON -- Allow double-quoted identifiers

-- Query > Query Options > Results > Grid:
Maximum Characters Retrieved: 65535 (for large text columns)
Discard Results After Execution: useful for performance testing
```

### Registered Servers

```
-- View > Registered Servers (Ctrl+Alt+G)
-- Group servers by environment:

Production
  +-- PROD-SQL-01 (Primary)
  +-- PROD-SQL-02 (Secondary)
Staging
  +-- STG-SQL-01
Development
  +-- DEV-SQL-01
  +-- DEV-SQL-02

-- Multi-server queries: right-click a group > New Query
-- Executes the same query against all servers in the group simultaneously
```

## Azure Data Studio Key Features

### Notebooks

```sql
-- Azure Data Studio notebooks combine SQL, text, and results
-- File > New Notebook > Kernel: SQL

-- Markdown cell:
-- ## Monthly Order Report
-- Analysis of order trends for the current quarter.

-- Code cell (SQL):
SELECT
    YEAR(OrderDate) AS OrderYear,
    MONTH(OrderDate) AS OrderMonth,
    COUNT(*) AS OrderCount,
    SUM(TotalAmount) AS Revenue
FROM dbo.Orders
WHERE OrderDate >= DATEADD(QUARTER, -1, GETDATE())
GROUP BY YEAR(OrderDate), MONTH(OrderDate)
ORDER BY OrderYear, OrderMonth;

-- Results render inline below each code cell
-- Notebooks saved as .ipynb files -- versionable in git
-- Export to HTML/PDF for sharing
```

### Extensions

```
-- Essential Azure Data Studio extensions:

1. SQL Server Admin Pack
   - Server dashboard, backup/restore wizards

2. SQL Database Projects
   - SSDT-like .sqlproj support, schema compare

3. Schema Compare
   - Visual diff between databases or dacpac files

4. PostgreSQL / MySQL extensions
   - Multi-database support in one tool

5. Profiler
   - Lightweight query profiler (Extended Events based)

6. Query Plan Viewer
   - Graphical execution plan display

7. SandDance
   - Data visualization directly from query results

-- Install: Extensions sidebar (Ctrl+Shift+X) > search > Install
```

### Source Control Integration

```
-- Azure Data Studio has built-in git support:

-- 1. Open a folder containing SQL files (File > Open Folder)
-- 2. Source Control sidebar shows changed files
-- 3. Stage, commit, push, pull without leaving the editor
-- 4. View file diffs inline (side-by-side or inline)

-- Workflow:
-- Edit SQL files > stage changes > commit > push to remote
-- Perfect for version-controlling stored procedures, views, functions
```

### Terminal and Command Palette

```bash
# Integrated terminal (Ctrl+`)
# Run sqlcmd, bcp, dbatools directly

sqlcmd -S myserver -d MyApp -Q "SELECT @@VERSION"

# Command Palette (Ctrl+Shift+P)
# Quick access to all commands:
# > "New Query" -- opens new SQL tab
# > "Run Query" -- executes current query
# > "Export Results" -- save to CSV/JSON/Excel
# > "Format Document" -- auto-format SQL
```

## When to Use Which

```
SSMS is better for:
  - SQL Server Agent job management (create, schedule, monitor)
  - Maintenance plans (backup, reindex, integrity checks)
  - Security administration (logins, server roles, auditing)
  - Activity Monitor for real-time performance monitoring
  - Replication setup and monitoring
  - Always On Availability Group configuration
  - SSIS package management
  - Import/Export wizard
  - Linked server configuration
  - Full-text search setup

Azure Data Studio is better for:
  - Cross-platform work (macOS, Linux)
  - Day-to-day query writing with modern IntelliSense
  - Jupyter-style SQL notebooks for documentation and analysis
  - Version-controlled SQL file editing (git integration)
  - Multi-database support (SQL Server + PostgreSQL + MySQL)
  - Extension-based customization
  - Dashboard widgets for monitoring
  - Schema compare (via extension)
  - Collaborative query sharing (notebooks)
```

## Best Practices

- Use SSMS for all administrative operations (Agent, security, maintenance plans)
- Use Azure Data Studio for query development, notebooks, and version-controlled scripts
- Enable "Include Actual Execution Plan" (Ctrl+M) when troubleshooting slow queries in SSMS
- Set up Registered Server groups in SSMS for multi-server management
- Save commonly used queries as SSMS snippets or Azure Data Studio code snippets
- Use Azure Data Studio notebooks for runbook documentation (SQL + markdown)
- Configure SSMS to disconnect idle connections (Tools > Options > Environment > AutoRecover)
- Keep both tools updated -- SSMS and ADS release independently of SQL Server

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Running queries with "Results to Text" when results have wide columns | Output truncated or hard to read | Use Ctrl+D for "Results to Grid" or increase max characters in options |
| Not using "Include Actual Execution Plan" for performance tuning | Only estimated plan available, which may differ from actual | Toggle Ctrl+M before executing to get actual row counts and timing |
| Editing production data directly in SSMS grid view | Accidental data modification without audit trail | Use explicit UPDATE statements in a transaction with BEGIN TRAN / ROLLBACK |
| Ignoring SSMS warnings about unsaved query tabs | Losing work when SSMS crashes or is closed | Enable AutoRecover and save queries to files proactively |
| Using SSMS for collaborative query sharing | No version history, no diff capability | Use Azure Data Studio with git integration or save queries in a repository |

## SQL Server Version Notes

- **SQL Server 2016**: SSMS 17.x first release as standalone installer (decoupled from SQL Server install). Azure Data Studio (originally SQL Operations Studio) launched as preview. Live Query Statistics introduced in SSMS.
- **SQL Server 2019**: SSMS 18.x required for full feature support (Big Data Clusters, Always Encrypted with enclaves). Azure Data Studio gained SQL Database Projects extension and schema compare. Notebooks first available for SQL kernel.
- **SQL Server 2022**: SSMS 19.x and 20.x releases. Azure Data Studio fully supports Ledger tables, contained availability groups. SSMS adds Query Store hints integration and plan forcing improvements. Azure Data Studio extensions ecosystem expanded significantly.

## Sources

- https://learn.microsoft.com/en-us/sql/ssms/sql-server-management-studio-ssms
- https://learn.microsoft.com/en-us/azure-data-studio/what-is-azure-data-studio
- https://learn.microsoft.com/en-us/sql/ssms/tutorials/ssms-tricks
- https://learn.microsoft.com/en-us/azure-data-studio/extensions/sql-database-project-extension
- https://learn.microsoft.com/en-us/azure-data-studio/notebooks/notebooks-guidance
