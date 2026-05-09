# Entity Framework Core -- SQL Server Provider and ORM Patterns

## Overview

Entity Framework Core (EF Core) is Microsoft's recommended ORM for .NET applications accessing SQL Server. The `Microsoft.EntityFrameworkCore.SqlServer` provider translates LINQ queries into optimized T-SQL, manages connection lifecycle, handles migrations, and maps .NET types to SQL Server column types. EF Core supports both code-first (define models in C#, generate schema) and database-first (scaffold models from existing database) workflows.

EF Core is the right choice when your team values developer productivity, type-safe queries, and automatic change tracking. It handles 80-90% of data access needs out of the box. For the remaining performance-critical paths, EF Core supports raw SQL queries and can coexist with Dapper or ADO.NET.

Use this skill when configuring EF Core with SQL Server, optimizing query performance, implementing migrations, or leveraging SQL Server-specific features like temporal tables or HierarchyId.

## Key Concepts

- **DbContext**: The primary class for database interaction. Represents a session with the database, exposes DbSet<T> properties for each entity, and manages change tracking and transactions.
- **Migrations**: Code-first schema versioning. Each migration is a C# class describing schema changes. Applied via `dotnet ef database update` or at runtime.
- **Code-first**: Define your C# model classes first; EF Core generates the database schema.
- **Database-first (scaffolding)**: Generate C# models from an existing SQL Server database using `dotnet ef dbcontext scaffold`.
- **Change tracking**: EF Core automatically detects modifications to tracked entities and generates UPDATE statements on SaveChanges().

## DbContext Configuration

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Customer>(entity =>
        {
            entity.ToTable("Customers", "dbo");
            entity.HasKey(e => e.CustomerID);
            entity.Property(e => e.Email).HasMaxLength(256).IsRequired();
            entity.HasIndex(e => e.Email).IsUnique();
            entity.Property(e => e.CreatedAt)
                  .HasDefaultValueSql("SYSUTCDATETIME()")
                  .HasColumnType("datetime2(3)");
        });

        modelBuilder.Entity<Order>(entity =>
        {
            entity.ToTable("Orders", "dbo");
            entity.HasOne(e => e.Customer)
                  .WithMany(c => c.Orders)
                  .HasForeignKey(e => e.CustomerID)
                  .OnDelete(DeleteBehavior.Restrict);
            entity.Property(e => e.TotalAmount).HasColumnType("decimal(18,2)");
        });
    }
}
```

## Connection String Patterns

```csharp
// Program.cs (ASP.NET Core)
var builder = WebApplication.CreateBuilder(args);

// SQL authentication
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// appsettings.json
// "ConnectionStrings": {
//   "DefaultConnection": "Server=myserver;Database=MyApp;User Id=appuser;Password=secret;Encrypt=True;TrustServerCertificate=False"
// }

// Windows authentication
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("Server=myserver;Database=MyApp;Trusted_Connection=True;Encrypt=True"));

// Azure AD authentication (passwordless)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("AzureSQL"),
        sqlOptions => sqlOptions.UseAzureSqlDefaults()));
```

## Migrations

```bash
# Add a new migration
dotnet ef migrations add CreateInitialSchema

# Apply migrations to database
dotnet ef database update

# Generate SQL script from migrations (for DBA review)
dotnet ef migrations script --idempotent --output migrate.sql

# Revert to a specific migration
dotnet ef database update PreviousMigrationName

# Scaffold from existing database (database-first)
dotnet ef dbcontext scaffold \
    "Server=myserver;Database=MyApp;Trusted_Connection=True;Encrypt=True" \
    Microsoft.EntityFrameworkCore.SqlServer \
    --output-dir Models \
    --context AppDbContext \
    --schema dbo
```

```csharp
// Generated migration example
public partial class AddShippedDateToOrders : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<DateTime>(
            name: "ShippedDate",
            table: "Orders",
            schema: "dbo",
            type: "datetime2(3)",
            nullable: true);

        migrationBuilder.CreateIndex(
            name: "IX_Orders_ShippedDate",
            table: "Orders",
            schema: "dbo",
            column: "ShippedDate")
            .Annotation("SqlServer:Include", new[] { "CustomerID", "TotalAmount" });
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropIndex("IX_Orders_ShippedDate", schema: "dbo", table: "Orders");
        migrationBuilder.DropColumn("ShippedDate", schema: "dbo", table: "Orders");
    }
}
```

## Query Optimization

```csharp
// Eager loading -- single query with JOINs
var customers = await context.Customers
    .Include(c => c.Orders)
        .ThenInclude(o => o.OrderItems)
    .Where(c => c.IsActive)
    .ToListAsync();

// Explicit loading -- load related data on demand
var customer = await context.Customers.FindAsync(customerId);
await context.Entry(customer).Collection(c => c.Orders).LoadAsync();

// AsNoTracking -- read-only queries, skip change tracking overhead
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.Price > 50)
    .OrderBy(p => p.Name)
    .ToListAsync();

// Projection -- only select needed columns
var orderSummaries = await context.Orders
    .Where(o => o.OrderDate >= startDate)
    .Select(o => new OrderSummaryDto
    {
        OrderID = o.OrderID,
        CustomerName = o.Customer.FirstName + " " + o.Customer.LastName,
        TotalAmount = o.TotalAmount,
        ItemCount = o.OrderItems.Count()
    })
    .ToListAsync();

// Compiled queries -- pre-compile for repeated execution
private static readonly Func<AppDbContext, int, Task<Customer?>> _getCustomerById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Customers.FirstOrDefault(c => c.CustomerID == id));

// Usage
var customer = await _getCustomerById(context, 42);

// Split queries -- avoid cartesian explosion with multiple Includes
var orders = await context.Orders
    .Include(o => o.OrderItems)
    .Include(o => o.Customer)
    .AsSplitQuery()
    .ToListAsync();
```

## SQL Server Specific Features

```csharp
// Temporal tables (SQL Server 2016+)
modelBuilder.Entity<Product>(entity =>
{
    entity.ToTable("Products", b => b.IsTemporal(t =>
    {
        t.HasPeriodStart("ValidFrom");
        t.HasPeriodEnd("ValidTo");
        t.UseHistoryTable("ProductsHistory", "dbo");
    }));
});

// Query temporal data
var productAsOf = await context.Products
    .TemporalAsOf(DateTime.UtcNow.AddDays(-30))
    .FirstOrDefaultAsync(p => p.ProductID == 42);

var productHistory = await context.Products
    .TemporalAll()
    .Where(p => p.ProductID == 42)
    .OrderBy(p => EF.Property<DateTime>(p, "ValidFrom"))
    .ToListAsync();

// HierarchyId (SQL Server specific type)
public class Employee
{
    public int EmployeeID { get; set; }
    public string Name { get; set; } = string.Empty;
    public HierarchyId HierarchyNode { get; set; } = null!;
}

// Spatial types (NetTopologySuite)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connStr, x => x.UseNetTopologySuite()));
```

## Connection Resiliency

```csharp
// Enable retry on transient failures (Azure SQL recommended)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: new List<int> { 4060, 40197, 40501, 40613, 49918, 49919, 49920 });
        sqlOptions.CommandTimeout(60);
        sqlOptions.MinBatchSize(5);
        sqlOptions.MaxBatchSize(100);
    }));
```

## Bulk Operations

```csharp
// EFCore.BulkExtensions (NuGet: EFCore.BulkExtensions)
using EFCore.BulkExtensions;

// Bulk insert -- 100x faster than AddRange + SaveChanges for large datasets
var newProducts = Enumerable.Range(1, 10000)
    .Select(i => new Product { Name = $"Product {i}", Price = i * 1.5m })
    .ToList();
await context.BulkInsertAsync(newProducts);

// Bulk update
var productsToUpdate = await context.Products
    .Where(p => p.CategoryID == 5)
    .ToListAsync();
productsToUpdate.ForEach(p => p.Price *= 1.10m); // 10% increase
await context.BulkUpdateAsync(productsToUpdate);

// Bulk delete
await context.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteDeleteAsync(); // EF Core 7+ built-in bulk delete

// Bulk update without loading entities (EF Core 7+)
await context.Products
    .Where(p => p.CategoryID == 5)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.Price, p => p.Price * 1.10m));
```

## Best Practices

- Use AsNoTracking() for all read-only queries to reduce memory and CPU overhead
- Prefer projections (Select) over loading full entities when you only need a few columns
- Enable connection resiliency (EnableRetryOnFailure) for Azure SQL and any network-attached SQL Server
- Use AsSplitQuery() when including multiple collection navigations to avoid cartesian explosion
- Generate idempotent SQL scripts from migrations for production deployments reviewed by DBAs
- Set CommandTimeout appropriately for long-running migration operations
- Use compiled queries for hot paths executed thousands of times per second
- Avoid lazy loading in web applications -- it causes N+1 query problems
- Use ExecuteUpdate/ExecuteDelete (EF Core 7+) for bulk operations instead of loading entities

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Calling SaveChanges() inside a loop | N separate round-trips instead of one batch | Collect all changes, call SaveChanges() once after the loop |
| Using lazy loading in API controllers | N+1 queries; response time grows with data volume | Use eager loading (Include) or projections (Select) |
| Not disposing DbContext | Connection pool exhaustion, memory leaks | Use dependency injection with scoped lifetime; use `await using` for manual creation |
| Applying migrations automatically in production startup | Race conditions in multi-instance deployments; risky | Generate SQL scripts and apply through CI/CD pipeline with DBA review |
| Ignoring the generated SQL (not logging queries) | Undetected performance issues reach production | Enable query logging: `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)` |

## SQL Server Version Notes

- **SQL Server 2016**: Temporal tables supported via `IsTemporal()`. No JSON column type (use NVARCHAR(MAX) with manual serialization). Row-level security can be configured but not modeled in EF Core.
- **SQL Server 2019**: UTF-8 collations supported. Accelerated Database Recovery improves migration rollback speed. Graph table support not modeled in EF Core.
- **SQL Server 2022**: Native JSON type supported in EF Core 8+. Ledger tables configurable via fluent API. GREATEST/LEAST usable in raw SQL queries. DATE_BUCKET simplifies temporal queries.

## Sources

- https://learn.microsoft.com/en-us/ef/core/providers/sql-server/
- https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/
- https://learn.microsoft.com/en-us/ef/core/querying/
- https://learn.microsoft.com/en-us/ef/core/performance/
- https://learn.microsoft.com/en-us/ef/core/providers/sql-server/azure-sql-database
- https://github.com/borisdj/EFCore.BulkExtensions
