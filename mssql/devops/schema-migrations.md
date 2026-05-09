# Schema Migrations -- Tools and Strategies for SQL Server Database Versioning

## Overview

Schema migrations are the disciplined practice of evolving a SQL Server database schema through versioned, repeatable changes. Every DDL modification -- adding a column, creating an index, altering a constraint -- is captured as a discrete migration artifact that can be applied, verified, and (when necessary) rolled back.

Two fundamental philosophies exist: **state-based** (declare the desired end state and let tooling compute the diff) and **migration-based** (author explicit forward/reverse scripts). SQL Server supports both via SSDT dacpac (state-based) and tools like Flyway, Liquibase, DbUp, and Roundhouse (migration-based). Choosing between them depends on team size, deployment frequency, and tolerance for manual intervention.

Use this skill when designing a migration strategy for SQL Server, evaluating tools, or implementing CI/CD-friendly schema deployment pipelines.

## Key Concepts

- **State-based migration**: You define what the schema should look like (e.g., SSDT project). The tool compares the target database to the desired state and generates a diff script. Pros: no migration ordering issues. Cons: data motion (column renames, splits) requires manual pre/post scripts.
- **Migration-based migration**: You write numbered scripts (V1__create_users.sql, V2__add_email.sql). The tool tracks which have run. Pros: explicit control over every change. Cons: merge conflicts in version numbers, ordering matters.
- **Idempotent migrations**: Scripts that can run multiple times without error (using IF NOT EXISTS guards).
- **Online operations**: SQL Server Enterprise supports ALTER TABLE ... WITH (ONLINE = ON) to avoid table locks during index rebuilds and certain schema changes.
- **Dacpac/Bacpac**: Dacpac is a compiled schema model (no data). Bacpac includes schema + data. Both are ZIP files with XML metadata.

## Flyway (SQL-Based Migrations)

Flyway uses plain SQL files with a naming convention. It stores migration history in a `flyway_schema_history` table.

```sql
-- V1__Create_Customers.sql
CREATE TABLE dbo.Customers (
    CustomerID   INT IDENTITY(1,1) PRIMARY KEY,
    FirstName    NVARCHAR(100) NOT NULL,
    LastName     NVARCHAR(100) NOT NULL,
    Email        NVARCHAR(256) NOT NULL,
    CreatedAt    DATETIME2(3) NOT NULL DEFAULT SYSUTCDATETIME()
);
GO

CREATE NONCLUSTERED INDEX IX_Customers_Email
    ON dbo.Customers (Email);
GO
```

```sql
-- V2__Add_PhoneNumber_to_Customers.sql
ALTER TABLE dbo.Customers
    ADD PhoneNumber NVARCHAR(20) NULL;
GO
```

```bash
# Flyway CLI commands
flyway -url="jdbc:sqlserver://localhost:1433;databaseName=MyApp;encrypt=true;trustServerCertificate=true" \
       -user=sa -password=YourPass \
       migrate

flyway info      # Show migration status
flyway validate  # Check applied vs available
flyway repair    # Fix failed migration checksums
```

## Liquibase (XML/YAML/JSON Changelogs)

```xml
<!-- changelog-master.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="1" author="dba-team">
        <createTable tableName="Orders" schemaName="dbo">
            <column name="OrderID" type="INT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="CustomerID" type="INT">
                <constraints nullable="false"
                             foreignKeyName="FK_Orders_Customers"
                             references="dbo.Customers(CustomerID)"/>
            </column>
            <column name="OrderDate" type="DATETIME2(3)"
                    defaultValueComputed="SYSUTCDATETIME()"/>
            <column name="TotalAmount" type="DECIMAL(18,2)"/>
        </createTable>
    </changeSet>

    <changeSet id="2" author="dba-team">
        <addColumn tableName="Orders" schemaName="dbo">
            <column name="Status" type="NVARCHAR(50)" defaultValue="Pending"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
```

## SSDT (State-Based with Dacpac)

SSDT stores every object as an individual `.sql` file in a Visual Studio project. Publishing compares the dacpac model against the target database and generates a diff script.

```sql
-- Tables/dbo.Products.sql (in SSDT project)
CREATE TABLE [dbo].[Products] (
    [ProductID]   INT            IDENTITY(1,1) NOT NULL,
    [Name]        NVARCHAR(200)  NOT NULL,
    [Price]       DECIMAL(18,2)  NOT NULL,
    [CategoryID]  INT            NULL,
    CONSTRAINT [PK_Products] PRIMARY KEY CLUSTERED ([ProductID])
);
GO
```

```powershell
# Build dacpac from project
dotnet build MyDatabase.sqlproj /p:Configuration=Release

# Publish dacpac to target
SqlPackage /Action:Publish `
    /SourceFile:"bin\Release\MyDatabase.dacpac" `
    /TargetConnectionString:"Server=prod-sql;Database=MyApp;Trusted_Connection=True;Encrypt=True" `
    /p:BlockOnPossibleDataLoss=True `
    /p:GenerateSmartDefaults=True
```

## DbUp (.NET Migration Runner)

```csharp
// Program.cs -- DbUp console migration runner
using DbUp;

var connectionString = args[0];

var upgrader = DeployChanges.To
    .SqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(typeof(Program).Assembly)
    .WithTransaction()
    .LogToConsole()
    .Build();

var result = upgrader.PerformUpgrade();

if (!result.Successful)
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine(result.Error);
    return -1;
}

Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("Success!");
return 0;
```

## Online Schema Changes

```sql
-- Online index rebuild (Enterprise only)
ALTER INDEX IX_Customers_Email ON dbo.Customers
    REBUILD WITH (ONLINE = ON, MAXDOP = 4, RESUMABLE = ON);
GO

-- Online column add (SQL Server 2012+, nullable or with runtime default)
ALTER TABLE dbo.Customers
    ADD MiddleName NVARCHAR(100) NULL;
GO

-- Adding NOT NULL column with default (online in SQL 2012+)
ALTER TABLE dbo.Customers
    ADD IsActive BIT NOT NULL
    CONSTRAINT DF_Customers_IsActive DEFAULT (1);
GO
```

## Idempotent Migration Patterns

```sql
-- Guard: table existence
IF NOT EXISTS (SELECT 1 FROM sys.tables WHERE name = 'AuditLog' AND schema_id = SCHEMA_ID('dbo'))
BEGIN
    CREATE TABLE dbo.AuditLog (
        LogID       BIGINT IDENTITY(1,1) PRIMARY KEY,
        TableName   NVARCHAR(128) NOT NULL,
        Action      NVARCHAR(10)  NOT NULL,
        ChangedAt   DATETIME2(3)  NOT NULL DEFAULT SYSUTCDATETIME()
    );
END
GO

-- Guard: column existence
IF NOT EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('dbo.Customers') AND name = 'LoyaltyPoints'
)
BEGIN
    ALTER TABLE dbo.Customers ADD LoyaltyPoints INT NULL;
END
GO

-- Guard: index existence
IF NOT EXISTS (
    SELECT 1 FROM sys.indexes
    WHERE object_id = OBJECT_ID('dbo.Orders') AND name = 'IX_Orders_CustomerID'
)
BEGIN
    CREATE NONCLUSTERED INDEX IX_Orders_CustomerID
        ON dbo.Orders (CustomerID)
        INCLUDE (OrderDate, TotalAmount);
END
GO
```

## Rollback Strategies

```sql
-- Flyway undo migration: U2__Remove_PhoneNumber.sql (Flyway Teams only)
ALTER TABLE dbo.Customers DROP COLUMN PhoneNumber;
GO

-- Manual rollback with safety check
IF EXISTS (
    SELECT 1 FROM sys.columns
    WHERE object_id = OBJECT_ID('dbo.Customers') AND name = 'PhoneNumber'
)
BEGIN
    -- Check if column has data before dropping
    DECLARE @hasData BIT = 0;
    EXEC sp_executesql N'IF EXISTS (SELECT 1 FROM dbo.Customers WHERE PhoneNumber IS NOT NULL)
                          SET @has = 1',
                       N'@has BIT OUTPUT', @has = @hasData OUTPUT;

    IF @hasData = 0
        ALTER TABLE dbo.Customers DROP COLUMN PhoneNumber;
    ELSE
        RAISERROR('Column PhoneNumber contains data -- manual review required.', 16, 1);
END
GO
```

## Best Practices

- Always wrap migrations in explicit transactions when the tool supports it
- Use idempotent guards (IF NOT EXISTS) for every DDL statement
- Never rename columns directly -- add new, copy data, drop old across multiple migrations
- Test migrations against a restored production backup before deploying
- Keep migrations small and focused -- one logical change per file
- Include both schema and reference/seed data changes in the migration pipeline
- Set BlockOnPossibleDataLoss=True when publishing dacpac to production
- Version your migration tool configuration alongside your application code
- Use ONLINE = ON for index operations on high-traffic tables (Enterprise edition)

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Dropping a column without checking for data | Permanent data loss in production | Add a pre-check query; use soft-delete (rename + deprecate) first |
| Running migrations outside a transaction | Partial schema changes on failure leave DB inconsistent | Wrap in BEGIN TRAN / COMMIT or use tool-level transaction support |
| Hardcoding server names in migration scripts | Script fails in different environments | Use variables, SQLCMD mode `:setvar`, or tool-level placeholders |
| Skipping idempotent guards | Re-running a migration fails with "object already exists" | Wrap every DDL in IF NOT EXISTS checks |
| Not testing rollback scripts | Rollback fails during an incident, extending downtime | Test rollback in staging after every migration deployment |

## SQL Server Version Notes

- **SQL Server 2016**: Resumable index rebuild not available. Online index operations require Enterprise. No UTF-8 collation support.
- **SQL Server 2019**: Resumable online index create (not just rebuild). Accelerated Database Recovery reduces rollback time. UTF-8 collations available.
- **SQL Server 2022**: Resumable ADD/DROP CONSTRAINT operations. Ledger tables for tamper-evident schema. GREATEST/LEAST functions simplify migration logic. JSON-native data type.

## Sources

- https://learn.microsoft.com/en-us/sql/ssdt/sql-server-data-tools
- https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage
- https://documentation.red-gate.com/fd/flyway-documentation-138346877.html
- https://docs.liquibase.com/start/tutorials/microsoft-sql-server.html
- https://dbup.readthedocs.io/en/latest/
- https://learn.microsoft.com/en-us/sql/relational-databases/indexes/perform-index-operations-online
