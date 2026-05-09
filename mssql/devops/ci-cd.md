# CI/CD for SQL Server -- Automated Build, Test, and Deploy Pipelines

## Overview

Continuous Integration and Continuous Deployment for SQL Server databases extends the same principles used for application code -- version control, automated builds, testing, and repeatable deployments -- to your database layer. The cornerstone technology is SSDT (SQL Server Data Tools), which compiles a database project into a dacpac artifact that can be published to any target instance.

Modern SQL Server CI/CD also leverages Docker containers for ephemeral test environments, tSQLt for database unit testing, and pipeline orchestrators like GitHub Actions or Azure DevOps to tie everything together. The goal is to make database deployments as predictable and automated as application deployments.

Use this skill when setting up automated database pipelines, containerized SQL Server test environments, or integrating database changes into an existing CI/CD workflow.

## Key Concepts

- **SSDT project (.sqlproj)**: A Visual Studio / MSBuild project that models a SQL Server database. Each object (table, view, procedure) is a `.sql` file. The project compiles to a `.dacpac`.
- **Dacpac**: Data-tier Application Package. A compiled, portable schema model. SqlPackage.exe compares it against a target database and generates/applies the diff.
- **SqlPackage.exe**: CLI tool for dacpac operations: Publish, Script, Extract, Export, Import, DeployReport.
- **tSQLt**: Open-source T-SQL unit testing framework that runs inside SQL Server.
- **Docker SQL Server**: Official Microsoft images for Linux-based SQL Server, ideal for ephemeral CI test databases.

## SSDT Project Structure

```
MyDatabase/
  MyDatabase.sqlproj           # MSBuild project file
  MyDatabase.publish.xml       # Default publish profile
  Tables/
    dbo.Customers.sql
    dbo.Orders.sql
    dbo.OrderItems.sql
  Views/
    dbo.vw_ActiveCustomers.sql
  StoredProcedures/
    dbo.usp_GetCustomerOrders.sql
    dbo.usp_CreateOrder.sql
  Functions/
    dbo.fn_CalculateDiscount.sql
  Security/
    Roles/
      db_app_reader.sql
      db_app_writer.sql
    Schemas/
      staging.sql
  Scripts/
    PreDeployment/
      Script.PreDeployment.sql
    PostDeployment/
      Script.PostDeployment.sql
      SeedData/
        dbo.OrderStatuses.sql
```

```xml
<!-- MyDatabase.sqlproj key sections -->
<Project DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <DSP>Microsoft.Data.Tools.Schema.Sql.Sql150DatabaseSchemaProvider</DSP>
    <ModelCollation>1033, CI</ModelCollation>
    <DefaultCollation>SQL_Latin1_General_CP1_CI_AS</DefaultCollation>
    <TargetDatabaseSet>True</TargetDatabaseSet>
    <DefaultSchema>dbo</DefaultSchema>
  </PropertyGroup>

  <ItemGroup>
    <Build Include="Tables\dbo.Customers.sql" />
    <Build Include="StoredProcedures\dbo.usp_GetCustomerOrders.sql" />
    <PreDeploy Include="Scripts\PreDeployment\Script.PreDeployment.sql" />
    <PostDeploy Include="Scripts\PostDeployment\Script.PostDeployment.sql" />
  </ItemGroup>
</Project>
```

## Building Dacpac in CI

```powershell
# Build with dotnet SDK (cross-platform, preferred)
dotnet build MyDatabase.sqlproj /p:Configuration=Release

# Build with MSBuild (Windows)
msbuild MyDatabase.sqlproj /p:Configuration=Release /t:Build

# Output: bin/Release/MyDatabase.dacpac
```

## Publishing with SqlPackage.exe

```powershell
# Generate diff script without applying (review first)
SqlPackage /Action:Script `
    /SourceFile:"MyDatabase.dacpac" `
    /TargetConnectionString:"Server=staging-sql;Database=MyApp;Trusted_Connection=True;Encrypt=True" `
    /OutputPath:"deploy-script.sql" `
    /p:BlockOnPossibleDataLoss=True

# Apply dacpac directly to target
SqlPackage /Action:Publish `
    /SourceFile:"MyDatabase.dacpac" `
    /TargetConnectionString:"Server=prod-sql;Database=MyApp;Trusted_Connection=True;Encrypt=True" `
    /Profile:"Production.publish.xml"

# Generate deployment report (XML summary of changes)
SqlPackage /Action:DeployReport `
    /SourceFile:"MyDatabase.dacpac" `
    /TargetConnectionString:"Server=prod-sql;Database=MyApp;Trusted_Connection=True;Encrypt=True" `
    /OutputPath:"deploy-report.xml"
```

```xml
<!-- Production.publish.xml -->
<Project ToolsVersion="Current" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <BlockOnPossibleDataLoss>True</BlockOnPossibleDataLoss>
    <DropObjectsNotInSource>False</DropObjectsNotInSource>
    <GenerateSmartDefaults>True</GenerateSmartDefaults>
    <AllowIncompatiblePlatform>False</AllowIncompatiblePlatform>
    <ExcludeObjectTypes>Users;RoleMembership;Logins</ExcludeObjectTypes>
    <ScriptDatabaseOptions>False</ScriptDatabaseOptions>
  </PropertyGroup>
</Project>
```

## Docker SQL Server for Dev/Test

```yaml
# docker-compose.yml
version: '3.8'
services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "StrongP@ssw0rd!"
      MSSQL_PID: "Developer"
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "StrongP@ssw0rd!" -C -Q "SELECT 1"
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  sqldata:
```

```bash
# Quick single-container startup
docker run -d --name sql-test \
  -e "ACCEPT_EULA=Y" \
  -e "MSSQL_SA_PASSWORD=StrongP@ssw0rd!" \
  -e "MSSQL_PID=Developer" \
  -p 1433:1433 \
  mcr.microsoft.com/mssql/server:2022-latest

# Wait for SQL Server to be ready
until docker exec sql-test /opt/mssql-tools18/bin/sqlcmd \
  -S localhost -U sa -P "StrongP@ssw0rd!" -C -Q "SELECT 1" 2>/dev/null; do
  sleep 2
done
```

## Database Unit Testing with tSQLt

```sql
-- Install tSQLt framework (run once per database)
-- Download from https://tsqlt.org and execute tSQLt.class.sql

-- Create a test class (schema)
EXEC tSQLt.NewTestClass 'OrderTests';
GO

-- Unit test: verify stored procedure creates an order
CREATE PROCEDURE OrderTests.[test usp_CreateOrder inserts row and returns ID]
AS
BEGIN
    -- Arrange: fake the tables to isolate the test
    EXEC tSQLt.FakeTable 'dbo.Orders';
    EXEC tSQLt.FakeTable 'dbo.Customers';

    INSERT INTO dbo.Customers (CustomerID, FirstName, LastName, Email)
    VALUES (1, 'Jane', 'Doe', 'jane@example.com');

    -- Act
    DECLARE @OrderID INT;
    EXEC dbo.usp_CreateOrder @CustomerID = 1, @TotalAmount = 99.99, @OrderID = @OrderID OUTPUT;

    -- Assert
    EXEC tSQLt.AssertEquals @Expected = 1,
                            @Actual = (SELECT COUNT(*) FROM dbo.Orders WHERE CustomerID = 1);

    EXEC tSQLt.AssertEquals @Expected = 99.99,
                            @Actual = (SELECT TotalAmount FROM dbo.Orders WHERE OrderID = @OrderID);
END;
GO

-- Run all tests
EXEC tSQLt.RunAll;

-- Run a specific test class
EXEC tSQLt.Run 'OrderTests';
```

## GitHub Actions Pipeline

```yaml
# .github/workflows/database-ci.yml
name: Database CI/CD

on:
  push:
    branches: [main]
    paths: ['database/**']
  pull_request:
    paths: ['database/**']

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      sqlserver:
        image: mcr.microsoft.com/mssql/server:2022-latest
        env:
          ACCEPT_EULA: "Y"
          MSSQL_SA_PASSWORD: "TestP@ssw0rd!"
          MSSQL_PID: Developer
        ports:
          - 1433:1433
        options: >-
          --health-cmd "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P 'TestP@ssw0rd!' -C -Q 'SELECT 1'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'

      - name: Build dacpac
        run: dotnet build database/MyDatabase.sqlproj /p:Configuration=Release

      - name: Install SqlPackage
        run: dotnet tool install -g microsoft.sqlpackage

      - name: Deploy to test instance
        run: |
          sqlpackage /Action:Publish \
            /SourceFile:"database/bin/Release/MyDatabase.dacpac" \
            /TargetConnectionString:"Server=localhost,1433;Database=MyApp;User Id=sa;Password=TestP@ssw0rd!;Encrypt=True;TrustServerCertificate=True"

      - name: Run tSQLt tests
        run: |
          sqlcmd -S localhost,1433 -U sa -P "TestP@ssw0rd!" -C \
            -d MyApp -i database/Tests/install-tsqlt.sql
          sqlcmd -S localhost,1433 -U sa -P "TestP@ssw0rd!" -C \
            -d MyApp -i database/Tests/all-tests.sql
          sqlcmd -S localhost,1433 -U sa -P "TestP@ssw0rd!" -C \
            -d MyApp -Q "EXEC tSQLt.RunAll"

      - name: Upload dacpac artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: database-dacpac
          path: database/bin/Release/MyDatabase.dacpac
```

## Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main]
  paths:
    include: ['database/*']

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildDacpac
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Build database project'
            inputs:
              command: 'build'
              projects: 'database/MyDatabase.sqlproj'
              arguments: '/p:Configuration=Release'

          - task: PublishBuildArtifacts@1
            displayName: 'Publish dacpac'
            inputs:
              PathtoPublish: 'database/bin/Release'
              ArtifactName: 'dacpac'

  - stage: DeployStaging
    dependsOn: Build
    jobs:
      - deployment: DeployToStaging
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: SqlAzureDacpacDeployment@1
                  displayName: 'Deploy dacpac to staging'
                  inputs:
                    azureSubscription: 'my-azure-subscription'
                    ServerName: 'staging-sql.database.windows.net'
                    DatabaseName: 'MyApp'
                    SqlUsername: '$(DB_USER)'
                    SqlPassword: '$(DB_PASSWORD)'
                    DacpacFile: '$(Pipeline.Workspace)/dacpac/MyDatabase.dacpac'
                    AdditionalArguments: '/p:BlockOnPossibleDataLoss=True'
```

## Integration Testing Pattern

```csharp
// DatabaseIntegrationTests.cs (xUnit + Testcontainers)
using Testcontainers.MsSql;
using Microsoft.Data.SqlClient;

public class OrderServiceTests : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        // Deploy dacpac or run migrations
        var connStr = _container.GetConnectionString();
        await DeployDacpac(connStr, "MyDatabase.dacpac");
    }

    [Fact]
    public async Task CreateOrder_ValidCustomer_ReturnsOrderId()
    {
        await using var conn = new SqlConnection(_container.GetConnectionString());
        await conn.OpenAsync();

        // Seed test data
        await using var seedCmd = new SqlCommand(
            "INSERT INTO dbo.Customers (FirstName, LastName, Email) VALUES ('Test', 'User', 'test@test.com')",
            conn);
        await seedCmd.ExecuteNonQueryAsync();

        // Execute the procedure under test
        await using var cmd = new SqlCommand("dbo.usp_CreateOrder", conn);
        cmd.CommandType = CommandType.StoredProcedure;
        cmd.Parameters.AddWithValue("@CustomerID", 1);
        cmd.Parameters.AddWithValue("@TotalAmount", 49.99m);
        var outParam = cmd.Parameters.Add("@OrderID", SqlDbType.Int);
        outParam.Direction = ParameterDirection.Output;

        await cmd.ExecuteNonQueryAsync();

        Assert.True((int)outParam.Value > 0);
    }

    public async Task DisposeAsync() => await _container.DisposeAsync();
}
```

## Best Practices

- Build dacpac on every pull request to catch syntax and reference errors early
- Use Docker SQL Server containers for test isolation -- never test against shared databases
- Keep publish profiles separate per environment (Dev, Staging, Production)
- Set BlockOnPossibleDataLoss=True for all non-dev deployments
- Generate a deployment script for manual review before production publishes
- Store connection strings in CI/CD secrets, never in source control
- Run tSQLt tests as a required check before merging database PRs
- Pin your SQL Server Docker image version to avoid surprise behavior changes
- Use the DeployReport action to get a summary of changes before applying

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `DropObjectsNotInSource=True` in production | Drops tables/views not in your project, causing data loss | Set to False for production; use explicit drop migrations |
| Not setting health checks on Docker SQL containers in CI | Pipeline proceeds before SQL Server is ready, tests fail intermittently | Add health check with retry loop using sqlcmd |
| Storing SA password in docker-compose.yml committed to repo | Credential exposure in version control | Use environment variables or Docker secrets; use `.env` files excluded from git |
| Building dacpac with wrong target platform (DSP) | Generated scripts use syntax not supported by target version | Match DSP in .sqlproj to your target SQL Server version |
| Running integration tests against a shared staging database | Tests interfere with each other, flaky results | Use Testcontainers or per-pipeline Docker instances |

## SQL Server Version Notes

- **SQL Server 2016**: DSP = `Sql130DatabaseSchemaProvider`. No UTF-8 support. Docker images available but less stable.
- **SQL Server 2019**: DSP = `Sql150DatabaseSchemaProvider`. Improved Docker image stability. Accelerated Database Recovery makes test teardown faster. Graph tables and UTF-8 collation support.
- **SQL Server 2022**: DSP = `Sql160DatabaseSchemaProvider`. Ledger tables for auditable schemas. Contained availability groups simplify containerized HA testing. JSON native type simplifies test data fixtures.

## Sources

- https://learn.microsoft.com/en-us/sql/ssdt/sql-server-data-tools
- https://learn.microsoft.com/en-us/sql/tools/sqlpackage/sqlpackage-publish
- https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-docker-container-deployment
- https://tsqlt.org/user-guide/
- https://github.com/testcontainers/testcontainers-dotnet
- https://learn.microsoft.com/en-us/azure/devops/pipelines/targets/azure-sqldb
