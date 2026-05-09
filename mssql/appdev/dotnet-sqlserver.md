# .NET and SQL Server — Microsoft.Data.SqlClient for Modern Data Access

## Overview

Microsoft.Data.SqlClient is the official, actively developed ADO.NET data provider for SQL Server. It supersedes the legacy System.Data.SqlClient namespace and is the recommended choice for all new .NET applications targeting SQL Server, Azure SQL Database, and Azure SQL Managed Instance.

Use Microsoft.Data.SqlClient when building any .NET application (ASP.NET Core, console, worker service, desktop) that needs to communicate with SQL Server. It provides connection pooling out of the box, async/await support, Always Encrypted client-side encryption, and modern authentication methods including Azure Active Directory.

The library supports .NET Framework 4.6.2+, .NET Core 3.1+, and .NET 6/7/8. It is distributed via NuGet and receives regular updates independent of the .NET runtime, allowing faster adoption of new SQL Server features.

## Key Concepts

- **SqlConnection** — Represents a connection to SQL Server. Connections are pooled automatically; always use `using` statements to return connections to the pool promptly.
- **SqlCommand** — Executes T-SQL statements or stored procedures against a connection. Always use parameterized queries to prevent SQL injection.
- **SqlDataReader** — Forward-only, read-only cursor for streaming result sets with minimal memory overhead.
- **SqlBulkCopy** — High-performance bulk insert API that maps to the TDS bulk insert protocol (equivalent to bcp).
- **SqlTransaction** — Explicit local transaction control with commit/rollback semantics.
- **Connection pooling** — Managed automatically by the provider. Connections with identical connection strings share a pool (default max 100).
- **SqlRetryLogicBaseProvider** — Built-in configurable retry logic for transient fault handling (available in Microsoft.Data.SqlClient 3.0+).

## Connection Strings

```csharp
// SQL Authentication
var connStr = "Server=myserver.database.windows.net;Database=mydb;User Id=myuser;Password=mypassword;Encrypt=True;TrustServerCertificate=False;";

// Windows Integrated Authentication
var connStr = "Server=myserver;Database=mydb;Integrated Security=True;Encrypt=True;";

// Azure AD with Managed Identity
var connStr = "Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity;Encrypt=True;";

// Azure AD Interactive (browser prompt)
var connStr = "Server=myserver.database.windows.net;Database=mydb;Authentication=Active Directory Interactive;Encrypt=True;";

// Connection pooling parameters (defaults shown)
var connStr = "Server=myserver;Database=mydb;Integrated Security=True;Min Pool Size=0;Max Pool Size=100;Connection Lifetime=0;Pooling=True;";
```

## Basic CRUD Operations

```csharp
using Microsoft.Data.SqlClient;

// Read with parameterized query
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

await using var command = new SqlCommand(
    "SELECT EmployeeId, Name, Department FROM Employees WHERE Department = @dept", 
    connection);
command.Parameters.AddWithValue("@dept", "Engineering");

await using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    var id = reader.GetInt32(0);
    var name = reader.GetString(1);
    var dept = reader.GetString(2);
    Console.WriteLine($"{id}: {name} ({dept})");
}
```

```csharp
// Insert with output parameter
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

await using var command = new SqlCommand(
    @"INSERT INTO Employees (Name, Department, HireDate) 
      VALUES (@name, @dept, @hireDate);
      SELECT SCOPE_IDENTITY();", connection);

command.Parameters.Add("@name", SqlDbType.NVarChar, 100).Value = "Jane Doe";
command.Parameters.Add("@dept", SqlDbType.NVarChar, 50).Value = "Engineering";
command.Parameters.Add("@hireDate", SqlDbType.Date).Value = DateTime.Today;

var newId = Convert.ToInt32(await command.ExecuteScalarAsync());
Console.WriteLine($"Inserted employee with ID: {newId}");
```

## Async Operations

```csharp
public async Task<List<Employee>> GetEmployeesAsync(string department)
{
    var employees = new List<Employee>();
    
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(
        "SELECT EmployeeId, Name, Salary FROM Employees WHERE Department = @dept",
        connection);
    command.Parameters.Add("@dept", SqlDbType.NVarChar, 50).Value = department;

    await using var reader = await command.ExecuteReaderAsync();
    while (await reader.ReadAsync())
    {
        employees.Add(new Employee
        {
            Id = reader.GetInt32(reader.GetOrdinal("EmployeeId")),
            Name = reader.GetString(reader.GetOrdinal("Name")),
            Salary = reader.GetDecimal(reader.GetOrdinal("Salary"))
        });
    }

    return employees;
}

// ExecuteNonQueryAsync for INSERT/UPDATE/DELETE
public async Task<int> UpdateSalaryAsync(int employeeId, decimal newSalary)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand(
        "UPDATE Employees SET Salary = @salary WHERE EmployeeId = @id", connection);
    command.Parameters.Add("@salary", SqlDbType.Decimal).Value = newSalary;
    command.Parameters.Add("@id", SqlDbType.Int).Value = employeeId;

    return await command.ExecuteNonQueryAsync(); // returns rows affected
}
```

## SqlBulkCopy for Bulk Operations

```csharp
public async Task BulkInsertEmployeesAsync(DataTable employees)
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "dbo.Employees",
        BatchSize = 5000,
        BulkCopyTimeout = 600,
        EnableStreaming = true
    };

    // Map source columns to destination columns
    bulkCopy.ColumnMappings.Add("Name", "Name");
    bulkCopy.ColumnMappings.Add("Department", "Department");
    bulkCopy.ColumnMappings.Add("Salary", "Salary");

    // Optional: track progress
    bulkCopy.NotifyAfter = 1000;
    bulkCopy.SqlRowsCopied += (sender, e) =>
        Console.WriteLine($"Copied {e.RowsCopied} rows");

    await bulkCopy.WriteToServerAsync(employees);
}

// Bulk insert from IDataReader (streaming from another source)
public async Task BulkCopyFromReaderAsync(SqlDataReader sourceReader)
{
    await using var destConn = new SqlConnection(_destConnectionString);
    await destConn.OpenAsync();

    using var bulkCopy = new SqlBulkCopy(destConn)
    {
        DestinationTableName = "dbo.EmployeesArchive",
        EnableStreaming = true
    };

    await bulkCopy.WriteToServerAsync(sourceReader);
}
```

## Transactions

```csharp
// Local transaction with SqlTransaction
await using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();
await using var transaction = (SqlTransaction)await connection.BeginTransactionAsync();

try
{
    await using var cmd1 = new SqlCommand(
        "UPDATE Accounts SET Balance = Balance - @amount WHERE AccountId = @from",
        connection, transaction);
    cmd1.Parameters.AddWithValue("@amount", 500m);
    cmd1.Parameters.AddWithValue("@from", 1001);
    await cmd1.ExecuteNonQueryAsync();

    await using var cmd2 = new SqlCommand(
        "UPDATE Accounts SET Balance = Balance + @amount WHERE AccountId = @to",
        connection, transaction);
    cmd2.Parameters.AddWithValue("@amount", 500m);
    cmd2.Parameters.AddWithValue("@to", 1002);
    await cmd2.ExecuteNonQueryAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}

// TransactionScope for ambient transactions (spans multiple connections)
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

await using var conn1 = new SqlConnection(connectionString1);
await conn1.OpenAsync();
// ... operations on conn1

await using var conn2 = new SqlConnection(connectionString2);
await conn2.OpenAsync();
// ... operations on conn2

scope.Complete(); // commits both
```

## Configurable Retry Logic

```csharp
// Available in Microsoft.Data.SqlClient 3.0+
var retryLogic = SqlConfigurableRetryFactory.CreateExponentialRetryProvider(
    new SqlRetryLogicOption
    {
        NumberOfTries = 5,
        DeltaTime = TimeSpan.FromSeconds(1),
        MaxTimeInterval = TimeSpan.FromSeconds(30),
        TransientErrors = new[] { 4060, 40197, 40501, 40613, 49918, 49919, 49920, 11001 }
    });

await using var connection = new SqlConnection(connectionString);
connection.RetryLogicProvider = retryLogic;
await connection.OpenAsync(); // retries automatically on transient errors

// Apply retry logic to commands
await using var command = connection.CreateCommand();
command.RetryLogicProvider = retryLogic;
command.CommandText = "SELECT COUNT(*) FROM Orders";
var count = await command.ExecuteScalarAsync();
```

## Always Encrypted

```csharp
// Connection string enables Always Encrypted
var connStr = "Server=myserver;Database=mydb;Integrated Security=True;"
    + "Column Encryption Setting=Enabled;";

await using var connection = new SqlConnection(connStr);
await connection.OpenAsync();

// Parameterized queries work transparently — encryption/decryption is automatic
await using var command = new SqlCommand(
    "SELECT PatientName, SSN FROM Patients WHERE SSN = @ssn", connection);
command.Parameters.Add("@ssn", SqlDbType.NVarChar, 11).Value = "123-45-6789";

await using var reader = await command.ExecuteReaderAsync();
// SSN is decrypted client-side automatically
```

## Dependency Injection Pattern (ASP.NET Core)

```csharp
// Program.cs / Startup.cs
builder.Services.AddTransient<IDbConnection>(sp =>
    new SqlConnection(builder.Configuration.GetConnectionString("DefaultConnection")));

// Or use a factory pattern
builder.Services.AddSingleton<IDbConnectionFactory>(sp =>
    new SqlConnectionFactory(builder.Configuration.GetConnectionString("DefaultConnection")));

public class SqlConnectionFactory : IDbConnectionFactory
{
    private readonly string _connectionString;
    public SqlConnectionFactory(string connectionString) => _connectionString = connectionString;
    
    public async Task<IDbConnection> CreateConnectionAsync()
    {
        var connection = new SqlConnection(_connectionString);
        await connection.OpenAsync();
        return connection;
    }
}
```

## Best Practices

- Always use `Microsoft.Data.SqlClient` over `System.Data.SqlClient` for new projects — it receives all new features and security fixes.
- Always wrap SqlConnection in `using`/`await using` to return connections to the pool promptly.
- Use parameterized queries exclusively — never concatenate user input into SQL strings.
- Prefer `Add()` with explicit SqlDbType over `AddWithValue()` to avoid implicit type conversions that hurt performance.
- Set `CommandTimeout` appropriately for long-running queries (default is 30 seconds).
- Use `async` methods throughout to avoid thread pool starvation under load.
- Enable `Encrypt=True` for all connections, especially Azure SQL.
- Use `SqlBulkCopy` instead of individual inserts when loading more than a few hundred rows.
- Configure retry logic for cloud deployments where transient errors are expected.
- Monitor connection pool usage with performance counters or `SqlConnection.ClearAllPools()` during diagnostics.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `AddWithValue` with string parameters | Causes implicit conversion from NVARCHAR(MAX), preventing index seeks | Use `Add("@p", SqlDbType.NVarChar, 50).Value = val` |
| Not disposing SqlConnection | Pool exhaustion, "Timeout expired" errors | Always use `using` or `await using` |
| Synchronous calls in async context | Thread pool starvation, deadlocks in ASP.NET | Use `OpenAsync`, `ExecuteReaderAsync`, etc. |
| Catching all exceptions in transactions | Swallows errors, data inconsistency | Catch specific exceptions, always rollback on failure |
| Hardcoding connection strings | Security risk, inflexible deployment | Use configuration/secrets management |
| Using MARS without understanding it | Unexpected blocking, performance issues | Prefer separate connections or sequential reads |
| Not setting CommandTimeout for reports | 30-second default timeout kills long queries | Set `command.CommandTimeout = 300` for reports |

## SQL Server Version Notes

- **SQL Server 2016** — Introduced Always Encrypted, temporal tables, JSON support. Microsoft.Data.SqlClient fully supports 2016+.
- **SQL Server 2019** — Added Accelerated Database Recovery, UTF-8 support. SqlClient supports new collations and features transparently.
- **SQL Server 2022** — Ledger tables, Always Encrypted with secure enclaves (enhanced), parameter-sensitive plan optimization. Requires Microsoft.Data.SqlClient 5.0+ for full feature support.
- **Azure SQL Database** — Always use `Encrypt=True;TrustServerCertificate=False`. Connection retry logic is essential. AAD authentication recommended over SQL auth.
- **Microsoft.Data.SqlClient 5.x** — Changed `Encrypt` default to `true` (breaking change from System.Data.SqlClient). TLS 1.2 minimum enforced.

## Sources

- https://learn.microsoft.com/en-us/sql/connect/ado-net/microsoft-ado-net-sql-server
- https://learn.microsoft.com/en-us/sql/connect/ado-net/introduction-microsoft-data-sqlclient-namespace
- https://learn.microsoft.com/en-us/sql/connect/ado-net/sqlclient-troubleshooting-guide
- https://learn.microsoft.com/en-us/sql/connect/ado-net/sql/sqlclient-support-always-encrypted
- https://learn.microsoft.com/en-us/sql/connect/ado-net/configurable-retry-logic
- https://github.com/dotnet/SqlClient
