# Version Control for SQL Server -- Schema-as-Code and Change Management

## Overview

Version-controlling a SQL Server database means treating every schema object -- tables, views, stored procedures, functions, indexes, permissions -- as source code that lives in a repository alongside your application. This eliminates "who changed what and when" mysteries, enables code review for DDL changes, and makes schema deployments repeatable and auditable.

The primary vehicle for SQL Server schema-as-code is SSDT (SQL Server Data Tools), which models the entire database as a Visual Studio project. Each object is a separate `.sql` file, diff-friendly and mergeable. For teams not using SSDT, scripting objects via SMO or sqlcmd and committing them to a migrations folder achieves similar traceability.

Use this skill when establishing database version control practices, managing branching strategies for schema changes, or implementing code review workflows for DDL modifications.

## Key Concepts

- **Schema-as-code**: Every database object is a file in version control. The repository is the single source of truth for what the schema should look like.
- **Schema snapshot**: A point-in-time export of all database objects as individual SQL files. Used for baselining an existing database into version control.
- **Schema compare**: A tool (built into SSDT/Visual Studio or Azure Data Studio) that diffs two schemas -- live database vs project, project vs project, or database vs database.
- **Pre/post deployment scripts**: SQL scripts that run before or after the main schema diff is applied. Used for data motion, seed data, and operations that dacpac cannot model (e.g., DBCC commands).
- **Drift detection**: Comparing the version-controlled schema against production to detect unauthorized changes.

## SSDT Project for Schema-as-Code

```
DatabaseProject/
  DatabaseProject.sqlproj
  Tables/
    dbo/
      Customers.sql
      Orders.sql
      OrderItems.sql
      Products.sql
    staging/
      RawImport.sql
  Views/
    dbo/
      vw_OrderSummary.sql
      vw_ActiveCustomers.sql
  StoredProcedures/
    dbo/
      usp_GetCustomerOrders.sql
      usp_ProcessRefund.sql
  Functions/
    dbo/
      fn_CalculateTax.sql
  Indexes/
    IX_Orders_CustomerID.sql
    IX_Products_CategoryID.sql
  Security/
    Schemas/
      staging.sql
      reporting.sql
    Roles/
      AppReader.sql
      AppWriter.sql
  Scripts/
    PreDeployment/
      Script.PreDeployment.sql
    PostDeployment/
      Script.PostDeployment.sql
      SeedData/
        OrderStatuses.sql
        CountryCodes.sql
```

```sql
-- Tables/dbo/Orders.sql
CREATE TABLE [dbo].[Orders]
(
    [OrderID]       INT            IDENTITY(1,1) NOT NULL,
    [CustomerID]    INT            NOT NULL,
    [OrderDate]     DATETIME2(3)   NOT NULL CONSTRAINT [DF_Orders_OrderDate] DEFAULT (SYSUTCDATETIME()),
    [Status]        NVARCHAR(50)   NOT NULL CONSTRAINT [DF_Orders_Status] DEFAULT (N'Pending'),
    [TotalAmount]   DECIMAL(18,2)  NOT NULL,
    [ShippedDate]   DATETIME2(3)   NULL,
    CONSTRAINT [PK_Orders] PRIMARY KEY CLUSTERED ([OrderID]),
    CONSTRAINT [FK_Orders_Customers] FOREIGN KEY ([CustomerID])
        REFERENCES [dbo].[Customers] ([CustomerID]),
    CONSTRAINT [CK_Orders_TotalAmount] CHECK ([TotalAmount] >= 0)
);
GO
```

## Schema Snapshots (Scripting Existing Objects)

```powershell
# Script all objects from existing database using SMO
Import-Module SqlServer

$server = New-Object Microsoft.SqlServer.Management.Smo.Server("prod-sql-01")
$db = $server.Databases["MyApp"]
$scripter = New-Object Microsoft.SqlServer.Management.Smo.Scripter($server)

$scripter.Options.ScriptDrops = $false
$scripter.Options.IncludeHeaders = $true
$scripter.Options.DriAll = $true           # Include all constraints
$scripter.Options.Indexes = $true
$scripter.Options.Permissions = $true
$scripter.Options.ToFileOnly = $true

# Script tables
foreach ($table in $db.Tables | Where-Object { -not $_.IsSystemObject }) {
    $schema = $table.Schema
    $dir = "Tables/$schema"
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
    $scripter.Options.FileName = "$dir/$($table.Name).sql"
    $scripter.Script($table)
}

# Script stored procedures
foreach ($proc in $db.StoredProcedures | Where-Object { -not $_.IsSystemObject }) {
    $schema = $proc.Schema
    $dir = "StoredProcedures/$schema"
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
    $scripter.Options.FileName = "$dir/$($proc.Name).sql"
    $scripter.Script($proc)
}

# Script views
foreach ($view in $db.Views | Where-Object { -not $_.IsSystemObject }) {
    $schema = $view.Schema
    $dir = "Views/$schema"
    New-Item -ItemType Directory -Path $dir -Force | Out-Null
    $scripter.Options.FileName = "$dir/$($view.Name).sql"
    $scripter.Script($view)
}
```

```bash
# Alternative: use mssql-scripter (cross-platform Python tool)
pip install mssql-scripter

mssql-scripter -S prod-sql-01 -d MyApp -U sa -P 'Pass' \
    --file-per-object \
    --output-path ./schema-snapshot/ \
    --include-types Table View StoredProcedure UserDefinedFunction
```

## Schema Compare

```powershell
# Schema compare via SqlPackage (extract + compare)

# Step 1: Extract dacpac from live database
SqlPackage /Action:Extract `
    /TargetFile:"live-snapshot.dacpac" `
    /SourceConnectionString:"Server=prod-sql;Database=MyApp;Trusted_Connection=True;Encrypt=True"

# Step 2: Generate diff report between project dacpac and live
SqlPackage /Action:DeployReport `
    /SourceFile:"project/bin/Release/MyDatabase.dacpac" `
    /TargetFile:"live-snapshot.dacpac" `
    /OutputPath:"schema-drift-report.xml"

# Step 3: Generate diff script (review before applying)
SqlPackage /Action:Script `
    /SourceFile:"project/bin/Release/MyDatabase.dacpac" `
    /TargetFile:"live-snapshot.dacpac" `
    /OutputPath:"schema-sync.sql"
```

## Migration File Management

For teams using migration-based tools (Flyway, DbUp) alongside or instead of SSDT:

```
migrations/
  V001__baseline.sql
  V002__add_customer_email_index.sql
  V003__create_audit_log_table.sql
  V004__add_order_shipped_date.sql
  V005__create_reporting_schema.sql
  R__refresh_materialized_views.sql    # Repeatable migration (Flyway)
  U004__remove_order_shipped_date.sql  # Undo migration (Flyway Teams)
```

```sql
-- V004__add_order_shipped_date.sql
-- Description: Add ShippedDate column to track fulfillment
-- Author: jsmith
-- Date: 2025-03-15
-- Ticket: JIRA-4521

IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('dbo.Orders') AND name = 'ShippedDate'
)
BEGIN
    ALTER TABLE dbo.Orders
        ADD ShippedDate DATETIME2(3) NULL;

    PRINT 'Added ShippedDate column to dbo.Orders';
END
ELSE
BEGIN
    PRINT 'ShippedDate column already exists on dbo.Orders -- skipping';
END
GO
```

## Branching Strategies for Database Changes

```
main (production schema)
  |
  +-- feature/add-loyalty-program
  |     V010__create_loyalty_points_table.sql
  |     V011__add_loyalty_tier_to_customers.sql
  |
  +-- feature/order-refunds
  |     V012__create_refunds_table.sql
  |     V013__add_refund_procedure.sql
  |
  +-- hotfix/fix-order-total-constraint
        V009__fix_order_total_check.sql
```

```
Strategy 1: Sequential numbering (simple teams)
- Use integer prefixes: V001, V002, V003
- Reserve ranges per sprint: sprint-12 gets V100-V199
- Merge conflicts in numbering resolved by renumbering the later branch

Strategy 2: Timestamp-based (larger teams)
- Use timestamps: V20250315_1430__description.sql
- Near-zero merge conflicts on file names
- Ordering matches creation time

Strategy 3: Hybrid (SSDT + migrations)
- SSDT project for schema definition (state-based)
- Pre/post deployment scripts for data motion
- Git branch per feature; dacpac build validates compilability
```

## Pre/Post Deployment Scripts in SSDT

```sql
-- Scripts/PreDeployment/Script.PreDeployment.sql
-- Runs BEFORE the dacpac schema diff is applied

-- Backup critical data before column type change
IF EXISTS (SELECT 1 FROM sys.columns
           WHERE object_id = OBJECT_ID('dbo.Products')
             AND name = 'Price' AND system_type_id = TYPE_ID('money'))
BEGIN
    PRINT 'Backing up Products.Price before type change...';
    SELECT ProductID, Price AS OldPrice
    INTO #PriceBackup
    FROM dbo.Products;
END
GO
```

```sql
-- Scripts/PostDeployment/Script.PostDeployment.sql
-- Runs AFTER the dacpac schema diff is applied

-- Load seed data (idempotent)
:r .\SeedData\OrderStatuses.sql
:r .\SeedData\CountryCodes.sql

-- Rebuild statistics after schema changes
PRINT 'Updating statistics...';
EXEC sp_updatestats;
GO
```

```sql
-- Scripts/PostDeployment/SeedData/OrderStatuses.sql
MERGE dbo.OrderStatuses AS target
USING (VALUES
    (1, N'Pending'),
    (2, N'Processing'),
    (3, N'Shipped'),
    (4, N'Delivered'),
    (5, N'Cancelled'),
    (6, N'Refunded')
) AS source (StatusID, StatusName)
ON target.StatusID = source.StatusID
WHEN MATCHED AND target.StatusName <> source.StatusName THEN
    UPDATE SET StatusName = source.StatusName
WHEN NOT MATCHED THEN
    INSERT (StatusID, StatusName) VALUES (source.StatusID, source.StatusName);
GO
```

## Code Review for DDL Changes

```sql
-- Pull request checklist for DDL changes:

-- 1. Does the migration have IF NOT EXISTS guards?
-- 2. Will this lock a large table? Check row count:
SELECT
    SCHEMA_NAME(t.schema_id) AS SchemaName,
    t.name AS TableName,
    SUM(p.rows) AS RowCount
FROM sys.tables t
JOIN sys.partitions p ON t.object_id = p.object_id AND p.index_id IN (0, 1)
WHERE t.name = 'Orders'
GROUP BY t.schema_id, t.name;

-- 3. Is there an index to support the new foreign key?
-- 4. Does the rollback script work?
-- 5. Has the migration been tested against a production-size dataset?

-- Automated drift detection (run nightly)
-- Compare live database against version-controlled dacpac
-- Alert on any differences
```

```yaml
# .github/CODEOWNERS for database changes
# Require DBA review for all schema changes
/database/Tables/    @org/dba-team
/database/Indexes/   @org/dba-team
/database/Security/  @org/dba-team @org/security-team
/database/StoredProcedures/  @org/dba-team @org/backend-team
```

## Drift Detection Automation

```powershell
# drift-check.ps1 -- Run nightly to detect unauthorized schema changes
param(
    [string]$DacpacPath = "latest/MyDatabase.dacpac",
    [string]$ConnectionString
)

# Generate deploy report (XML diff summary)
$reportPath = "drift-report-$(Get-Date -Format 'yyyyMMdd').xml"

SqlPackage /Action:DeployReport `
    /SourceFile:$DacpacPath `
    /TargetConnectionString:$ConnectionString `
    /OutputPath:$reportPath

# Parse the report for unexpected changes
[xml]$report = Get-Content $reportPath
$operations = $report.DeploymentReport.Operations.Operation

if ($operations.Count -gt 0) {
    $message = "Schema drift detected: $($operations.Count) differences found.`n"
    foreach ($op in $operations) {
        $message += "  - $($op.Name): $($op.Item.Value)`n"
    }
    # Send alert (email, Slack, etc.)
    Write-Warning $message
    exit 1
} else {
    Write-Host "No schema drift detected." -ForegroundColor Green
    exit 0
}
```

## Best Practices

- One file per database object -- never combine multiple CREATE statements in one file
- Use consistent file naming: `SchemaName.ObjectName.sql` (e.g., `dbo.Customers.sql`)
- Organize folders by object type (Tables, Views, StoredProcedures) then by schema
- Require DBA team review via CODEOWNERS for all table and index changes
- Run schema compare as a nightly CI job to detect drift from version control
- Include migration description, author, date, and ticket reference in every migration file header
- Never commit generated scripts (diff output) to the repository -- only source definitions
- Use MERGE for seed data to make post-deployment scripts idempotent
- Tag releases in git to correlate schema versions with application versions

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Editing production directly without updating version control | Schema drift; next deployment may revert manual changes | Enforce all changes through version control; run nightly drift detection |
| Putting multiple objects in one SQL file | Merge conflicts, hard to review, cannot track individual object history | One file per object; use SSDT project structure conventions |
| Not reviewing DDL changes in pull requests | Destructive changes (column drops, type changes) reach production unchecked | Require CODEOWNERS approval; add automated migration lint checks |
| Committing absolute file paths or server names | Scripts break across environments | Use SQLCMD variables, publish profiles, or environment-specific config files |
| Skipping seed data versioning | Lookup tables diverge between environments | Include seed data in post-deployment scripts using MERGE |

## SQL Server Version Notes

- **SQL Server 2016**: Schema compare available in SSDT for Visual Studio 2015+. No built-in JSON type complicates seed data handling. Extended Events schema changes are not modeled in dacpac.
- **SQL Server 2019**: SSDT supports Sql150 provider. Graph table CREATE syntax supported in projects. Accelerated Database Recovery makes large migration rollbacks faster. External data source definitions can be versioned.
- **SQL Server 2022**: Sql160 provider. Ledger table definitions can be versioned in SSDT. JSON type simplifies configuration/seed data patterns. System versioned temporal table improvements reduce migration complexity for audit tables.

## Sources

- https://learn.microsoft.com/en-us/sql/ssdt/sql-server-data-tools
- https://learn.microsoft.com/en-us/sql/ssdt/how-to-compare-and-synchronize-the-data-of-two-databases
- https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-extract
- https://learn.microsoft.com/en-us/sql/ssdt/how-to-specify-predeployment-or-postdeployment-scripts
- https://documentation.red-gate.com/soc/common-tasks/work-with-source-control
