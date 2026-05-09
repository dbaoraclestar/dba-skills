# Dapper with SQL Server -- High-Performance Micro-ORM

## Overview

Dapper is a lightweight micro-ORM for .NET that extends `IDbConnection` with methods like `Query<T>`, `Execute`, and `QueryMultiple`. Unlike Entity Framework, Dapper does not generate SQL -- you write your own T-SQL and Dapper handles parameter binding, result mapping, and connection management. This gives you full control over query performance while eliminating boilerplate ADO.NET code.

Dapper is the right choice when you need maximum query performance, want full control over the SQL being executed, are working with stored procedures, or have complex queries that are easier to express in T-SQL than LINQ. It is used extensively in high-throughput systems -- Stack Overflow built Dapper for their own performance-critical data access layer.

Use this skill when implementing data access with Dapper against SQL Server, optimizing query patterns, handling complex result sets, or deciding between Dapper and EF Core for specific use cases.

## Key Concepts

- **Micro-ORM**: Maps query results to objects but does not track changes, generate SQL, or manage schema migrations. You own the SQL.
- **Extension methods**: Dapper adds methods to `IDbConnection` -- no special context class needed. Works with `SqlConnection` directly.
- **Parameterized queries**: Dapper converts anonymous objects or `DynamicParameters` into `SqlParameter` instances, preventing SQL injection.
- **Multi-mapping**: Map a single row to multiple objects (for JOINs) or process multiple result sets from one command.
- **Buffered vs unbuffered**: By default, Dapper buffers all results into a `List<T>`. Pass `buffered: false` for large result sets to stream rows.

## Basic Query and Execute

```csharp
using Microsoft.Data.SqlClient;
using Dapper;

// Connection setup
await using var connection = new SqlConnection(
    "Server=myserver;Database=MyApp;Trusted_Connection=True;Encrypt=True");

// Query: map rows to objects
var customers = await connection.QueryAsync<Customer>(
    "SELECT CustomerID, FirstName, LastName, Email FROM dbo.Customers WHERE IsActive = 1");

// QueryFirst / QueryFirstOrDefault: single row
var customer = await connection.QueryFirstOrDefaultAsync<Customer>(
    "SELECT CustomerID, FirstName, LastName, Email FROM dbo.Customers WHERE CustomerID = @Id",
    new { Id = 42 });

// QuerySingle: exactly one row (throws if 0 or 2+)
var count = await connection.QuerySingleAsync<int>(
    "SELECT COUNT(*) FROM dbo.Customers WHERE Email = @Email",
    new { Email = "jane@example.com" });

// Execute: INSERT, UPDATE, DELETE, DDL
var rowsAffected = await connection.ExecuteAsync(
    @"INSERT INTO dbo.Customers (FirstName, LastName, Email)
      VALUES (@FirstName, @LastName, @Email)",
    new { FirstName = "Jane", LastName = "Doe", Email = "jane@example.com" });

// Execute with OUTPUT clause to get inserted ID
var newId = await connection.QuerySingleAsync<int>(
    @"INSERT INTO dbo.Customers (FirstName, LastName, Email)
      OUTPUT INSERTED.CustomerID
      VALUES (@FirstName, @LastName, @Email)",
    new { FirstName = "John", LastName = "Smith", Email = "john@example.com" });
```

## Parameterized Queries

```csharp
// Anonymous object parameters (most common)
var orders = await connection.QueryAsync<Order>(
    @"SELECT OrderID, CustomerID, OrderDate, TotalAmount
      FROM dbo.Orders
      WHERE CustomerID = @CustId AND OrderDate >= @Since",
    new { CustId = 42, Since = DateTime.UtcNow.AddDays(-30) });

// DynamicParameters for conditional queries
var parameters = new DynamicParameters();
parameters.Add("@MinAmount", 100.0m, DbType.Decimal);

var sql = "SELECT * FROM dbo.Orders WHERE TotalAmount >= @MinAmount";

if (statusFilter != null)
{
    parameters.Add("@Status", statusFilter, DbType.String);
    sql += " AND Status = @Status";
}

var filtered = await connection.QueryAsync<Order>(sql, parameters);

// IN clause with Dapper (automatic expansion)
var ids = new[] { 1, 2, 3, 4, 5 };
var selected = await connection.QueryAsync<Customer>(
    "SELECT * FROM dbo.Customers WHERE CustomerID IN @Ids",
    new { Ids = ids });

// Output parameters
var p = new DynamicParameters();
p.Add("@CustomerID", 42);
p.Add("@OrderCount", dbType: DbType.Int32, direction: ParameterDirection.Output);
p.Add("@ReturnValue", dbType: DbType.Int32, direction: ParameterDirection.ReturnValue);

await connection.ExecuteAsync("dbo.usp_GetCustomerStats", p,
    commandType: CommandType.StoredProcedure);

int orderCount = p.Get<int>("@OrderCount");
int returnValue = p.Get<int>("@ReturnValue");
```

## Multi-Mapping (JOIN Results)

```csharp
// One-to-one: Order with Customer
var orders = await connection.QueryAsync<Order, Customer, Order>(
    @"SELECT o.OrderID, o.OrderDate, o.TotalAmount,
             c.CustomerID, c.FirstName, c.LastName, c.Email
      FROM dbo.Orders o
      INNER JOIN dbo.Customers c ON o.CustomerID = c.CustomerID
      WHERE o.OrderDate >= @Since",
    (order, customer) =>
    {
        order.Customer = customer;
        return order;
    },
    new { Since = DateTime.UtcNow.AddDays(-7) },
    splitOn: "CustomerID");

// One-to-many: Customer with multiple Orders
var customerDict = new Dictionary<int, Customer>();

var result = await connection.QueryAsync<Customer, Order, Customer>(
    @"SELECT c.CustomerID, c.FirstName, c.LastName,
             o.OrderID, o.OrderDate, o.TotalAmount
      FROM dbo.Customers c
      LEFT JOIN dbo.Orders o ON c.CustomerID = o.CustomerID
      ORDER BY c.CustomerID",
    (customer, order) =>
    {
        if (!customerDict.TryGetValue(customer.CustomerID, out var existing))
        {
            existing = customer;
            existing.Orders = new List<Order>();
            customerDict[customer.CustomerID] = existing;
        }
        if (order != null) existing.Orders.Add(order);
        return existing;
    },
    splitOn: "OrderID");

var customers = customerDict.Values.ToList();
```

## Multiple Result Sets

```csharp
// QueryMultiple: process multiple SELECT statements in one round-trip
await using var multi = await connection.QueryMultipleAsync(
    @"SELECT CustomerID, FirstName, LastName FROM dbo.Customers WHERE CustomerID = @Id;
      SELECT OrderID, OrderDate, TotalAmount FROM dbo.Orders WHERE CustomerID = @Id;
      SELECT COUNT(*) FROM dbo.Orders WHERE CustomerID = @Id;",
    new { Id = 42 });

var customer = await multi.ReadFirstOrDefaultAsync<Customer>();
var orders = (await multi.ReadAsync<Order>()).ToList();
var orderCount = await multi.ReadSingleAsync<int>();
```

## Stored Procedure Execution

```csharp
// Simple stored procedure call
var customers = await connection.QueryAsync<Customer>(
    "dbo.usp_SearchCustomers",
    new { SearchTerm = "smith", MaxResults = 50 },
    commandType: CommandType.StoredProcedure);

// Stored procedure with table-valued parameter
var orderItems = new DataTable();
orderItems.Columns.Add("ProductID", typeof(int));
orderItems.Columns.Add("Quantity", typeof(int));
orderItems.Columns.Add("UnitPrice", typeof(decimal));
orderItems.Rows.Add(101, 2, 29.99m);
orderItems.Rows.Add(205, 1, 49.99m);

var p = new DynamicParameters();
p.Add("@CustomerID", 42);
p.Add("@OrderItems", orderItems.AsTableValuedParameter("dbo.OrderItemType"));

var orderId = await connection.QuerySingleAsync<int>(
    "dbo.usp_CreateOrder", p,
    commandType: CommandType.StoredProcedure);
```

```sql
-- Supporting SQL Server objects for TVP
CREATE TYPE dbo.OrderItemType AS TABLE (
    ProductID INT NOT NULL,
    Quantity  INT NOT NULL,
    UnitPrice DECIMAL(18,2) NOT NULL
);
GO

CREATE PROCEDURE dbo.usp_CreateOrder
    @CustomerID INT,
    @OrderItems dbo.OrderItemType READONLY
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @OrderID INT;

    INSERT INTO dbo.Orders (CustomerID, TotalAmount)
    SELECT @CustomerID, SUM(Quantity * UnitPrice) FROM @OrderItems;

    SET @OrderID = SCOPE_IDENTITY();

    INSERT INTO dbo.OrderItems (OrderID, ProductID, Quantity, UnitPrice)
    SELECT @OrderID, ProductID, Quantity, UnitPrice FROM @OrderItems;

    SELECT @OrderID;
END;
GO
```

## Transactions

```csharp
await connection.OpenAsync();
await using var transaction = await connection.BeginTransactionAsync();

try
{
    var orderId = await connection.QuerySingleAsync<int>(
        @"INSERT INTO dbo.Orders (CustomerID, TotalAmount)
          OUTPUT INSERTED.OrderID
          VALUES (@CustomerID, @TotalAmount)",
        new { CustomerID = 42, TotalAmount = 79.98m },
        transaction);

    await connection.ExecuteAsync(
        @"INSERT INTO dbo.OrderItems (OrderID, ProductID, Quantity, UnitPrice)
          VALUES (@OrderID, @ProductID, @Quantity, @UnitPrice)",
        new { OrderID = orderId, ProductID = 101, Quantity = 2, UnitPrice = 39.99m },
        transaction);

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

## Custom Type Handlers

```csharp
// Handle custom types that Dapper doesn't know about natively
public class JsonTypeHandler<T> : SqlMapper.TypeHandler<T>
{
    public override void SetValue(IDbDataParameter parameter, T? value)
    {
        parameter.Value = value == null ? DBNull.Value : JsonSerializer.Serialize(value);
        parameter.DbType = DbType.String;
    }

    public override T? Parse(object value)
    {
        return value is string json ? JsonSerializer.Deserialize<T>(json) : default;
    }
}

// Register at startup
SqlMapper.AddTypeHandler(new JsonTypeHandler<Address>());

// Now Dapper auto-serializes/deserializes Address to/from NVARCHAR column
var customer = await connection.QueryFirstAsync<Customer>(
    "SELECT CustomerID, FirstName, AddressJson AS Address FROM dbo.Customers WHERE CustomerID = @Id",
    new { Id = 42 });
```

## Dapper vs EF Core: When to Use Which

```
Dapper is better when:
- You need maximum query performance (2-10x faster than EF Core for reads)
- You have complex queries easier to write in T-SQL than LINQ
- You are calling stored procedures with TVPs or OUTPUT parameters
- You want zero magic -- see exactly what SQL runs
- You are building a read-heavy API (dashboards, reports, search)

EF Core is better when:
- You need change tracking and automatic UPDATE generation
- You want schema migrations managed by the framework
- You have a CRUD-heavy application with simple queries
- You want LINQ-based query composition with compile-time checking
- You need lazy/eager loading of navigation properties

Hybrid approach (common in production):
- EF Core for writes (change tracking, validation, migrations)
- Dapper for reads (performance-critical queries, reports, search)
- Both can share the same SqlConnection
```

## Best Practices

- Always use parameterized queries -- never concatenate user input into SQL strings
- Prefer `QueryFirstOrDefault` over `QueryFirst` when the result may not exist
- Use `buffered: false` for large result sets (10K+ rows) to avoid memory spikes
- Wrap multi-statement operations in explicit transactions
- Use `CommandType.StoredProcedure` for procedure calls -- it adds `EXEC` and proper parameter binding
- Close connections promptly; use `await using` to ensure disposal
- Consider `QueryMultiple` to batch related queries into a single round-trip
- Register custom type handlers at application startup, not per-query
- Use `splitOn` parameter carefully in multi-mapping -- it defaults to "Id"

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Missing `splitOn` parameter in multi-mapping | Dapper splits on "Id" by default; wrong column mapping | Explicitly set `splitOn` to the first column of each mapped type |
| Forgetting `CommandType.StoredProcedure` for proc calls | Dapper sends the proc name as raw SQL text; fails or behaves unexpectedly | Always pass `commandType: CommandType.StoredProcedure` |
| Not disposing SqlConnection | Connection pool exhaustion under load | Use `await using var connection = new SqlConnection(...)` |
| Using string concatenation for WHERE IN clauses | SQL injection vulnerability | Use Dapper's built-in IN expansion: `WHERE Id IN @Ids` with array parameter |
| Buffering millions of rows | OutOfMemoryException | Pass `buffered: false` and process rows in a streaming fashion |

## SQL Server Version Notes

- **SQL Server 2016**: All Dapper features work. JSON_VALUE/JSON_QUERY usable in raw SQL queries. No native JSON type -- use NVARCHAR(MAX) columns with JsonTypeHandler.
- **SQL Server 2019**: UTF-8 collation support. Batch mode on rowstore improves analytical query performance called from Dapper. Table variable deferred compilation improves TVP performance.
- **SQL Server 2022**: Native JSON type can be mapped with custom type handler. GREATEST/LEAST functions simplify query logic. Parameter-sensitive plan optimization improves parameterized query performance.

## Sources

- https://github.com/DapperLib/Dapper
- https://www.learndapper.com/
- https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql/table-valued-parameters
- https://learn.microsoft.com/en-us/sql/relational-databases/tables/use-table-valued-parameters-database-engine
