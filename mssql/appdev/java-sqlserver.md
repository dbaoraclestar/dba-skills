# Java and SQL Server — Microsoft JDBC Driver for SQL Server

## Overview

The Microsoft JDBC Driver for SQL Server is the official Type 4 JDBC driver for connecting Java applications to SQL Server, Azure SQL Database, and Azure SQL Managed Instance. It implements the JDBC 4.2 specification and communicates directly over the TDS protocol without requiring any native ODBC or OLE DB layers.

Use this driver for any Java application (Spring Boot, Jakarta EE, standalone) that needs SQL Server connectivity. It supports connection pooling through standard JDBC DataSource integration with pools like HikariCP and Apache DBCP, Always Encrypted for client-side column encryption, Azure Active Directory authentication, and Kerberos-based Windows integrated authentication from Linux and macOS.

The driver is distributed via Maven Central as `com.microsoft.sqlserver:mssql-jdbc` and follows semantic versioning. Major versions align with minimum Java requirements: the 12.x line requires Java 11+, while older 9.x/10.x versions support Java 8.

## Key Concepts

- **Type 4 JDBC Driver** — Pure Java, communicates directly with SQL Server over TCP/TDS. No native libraries needed.
- **Connection URL** — Format: `jdbc:sqlserver://host:port;databaseName=db;encrypt=true`. Semicolon-delimited properties.
- **PreparedStatement** — Parameterized queries with `?` positional placeholders. Prevents SQL injection and enables plan reuse.
- **Connection pooling** — The driver provides a `SQLServerConnectionPoolDataSource`, but production apps should use HikariCP or DBCP2 for pool management.
- **Integrated authentication** — Kerberos (cross-platform) or NTLM via `integratedSecurity=true` and `authenticationScheme=JavaKerberos`.
- **Always Encrypted** — Client-side encryption with `columnEncryptionSetting=Enabled` in the connection string. Requires a keystore provider.
- **Batch operations** — `addBatch()`/`executeBatch()` for efficient multi-row INSERT/UPDATE/DELETE.

## Maven Dependency

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
    <version>12.6.1.jre11</version>  <!-- use jre8 artifact for Java 8 -->
</dependency>

<!-- HikariCP connection pool (recommended) -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

## Connection Setup

```java
import java.sql.*;

// Basic SQL Authentication
String url = "jdbc:sqlserver://myserver:1433;"
    + "databaseName=mydb;"
    + "user=myuser;"
    + "password=mypassword;"
    + "encrypt=true;"
    + "trustServerCertificate=false;";

try (Connection conn = DriverManager.getConnection(url)) {
    System.out.println("Connected: " + conn.getCatalog());
}

// Windows Integrated Authentication (Kerberos)
String url = "jdbc:sqlserver://myserver:1433;"
    + "databaseName=mydb;"
    + "integratedSecurity=true;"
    + "authenticationScheme=JavaKerberos;";

// Azure AD with username/password
String url = "jdbc:sqlserver://myserver.database.windows.net:1433;"
    + "databaseName=mydb;"
    + "authentication=ActiveDirectoryPassword;"
    + "user=user@domain.com;"
    + "password=mypassword;"
    + "encrypt=true;";

// Azure AD with Managed Identity
String url = "jdbc:sqlserver://myserver.database.windows.net:1433;"
    + "databaseName=mydb;"
    + "authentication=ActiveDirectoryManagedIdentity;"
    + "encrypt=true;";
```

## Connection Pooling with HikariCP

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;

HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:sqlserver://myserver:1433;databaseName=mydb;encrypt=true");
config.setUsername("myuser");
config.setPassword("mypassword");
config.setMaximumPoolSize(20);
config.setMinimumIdle(5);
config.setIdleTimeout(300_000);      // 5 minutes
config.setMaxLifetime(1_800_000);    // 30 minutes
config.setConnectionTimeout(10_000); // 10 seconds
config.setPoolName("SqlServerPool");

// SQL Server specific tuning
config.addDataSourceProperty("sendStringParametersAsUnicode", "false");
config.addDataSourceProperty("selectMethod", "direct");
config.setConnectionTestQuery("SELECT 1");

HikariDataSource dataSource = new HikariDataSource(config);

// Use in application
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "SELECT EmployeeId, Name FROM Employees WHERE Department = ?")) {
    ps.setString(1, "Engineering");
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            System.out.printf("%d: %s%n", rs.getInt("EmployeeId"), rs.getString("Name"));
        }
    }
}
```

## Parameterized Queries with PreparedStatement

```java
// SELECT with parameters
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "SELECT OrderId, Amount, OrderDate FROM Orders " +
         "WHERE CustomerId = ? AND OrderDate >= ? AND Status = ?")) {
    
    ps.setInt(1, 42);
    ps.setDate(2, java.sql.Date.valueOf("2025-01-01"));
    ps.setString(3, "Shipped");
    
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            int orderId = rs.getInt("OrderId");
            BigDecimal amount = rs.getBigDecimal("Amount");
            LocalDate date = rs.getDate("OrderDate").toLocalDate();
            System.out.printf("Order %d: $%s on %s%n", orderId, amount, date);
        }
    }
}

// INSERT with generated keys
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(
         "INSERT INTO Employees (Name, Department, Salary) VALUES (?, ?, ?)",
         Statement.RETURN_GENERATED_KEYS)) {
    
    ps.setString(1, "Jane Doe");
    ps.setString(2, "Engineering");
    ps.setBigDecimal(3, new BigDecimal("95000.00"));
    ps.executeUpdate();
    
    try (ResultSet keys = ps.getGeneratedKeys()) {
        if (keys.next()) {
            System.out.println("New employee ID: " + keys.getInt(1));
        }
    }
}
```

## Batch Operations

```java
// Batch insert with addBatch/executeBatch
try (Connection conn = dataSource.getConnection()) {
    conn.setAutoCommit(false);
    
    try (PreparedStatement ps = conn.prepareStatement(
            "INSERT INTO Employees (Name, Department, Salary) VALUES (?, ?, ?)")) {
        
        String[][] employees = {
            {"Alice", "Engineering", "95000"},
            {"Bob", "Marketing", "82000"},
            {"Carol", "Engineering", "105000"},
            {"Dave", "Sales", "78000"}
        };
        
        for (String[] emp : employees) {
            ps.setString(1, emp[0]);
            ps.setString(2, emp[1]);
            ps.setBigDecimal(3, new BigDecimal(emp[2]));
            ps.addBatch();
        }
        
        int[] results = ps.executeBatch();
        conn.commit();
        System.out.printf("Inserted %d rows%n", results.length);
    } catch (BatchUpdateException e) {
        conn.rollback();
        System.err.println("Batch failed: " + e.getMessage());
        int[] updateCounts = e.getUpdateCounts();
        for (int i = 0; i < updateCounts.length; i++) {
            if (updateCounts[i] == Statement.EXECUTE_FAILED) {
                System.err.printf("Statement %d failed%n", i);
            }
        }
    }
}

// SQLServerBulkCopy for high-volume inserts
import com.microsoft.sqlserver.jdbc.SQLServerBulkCopy;
import com.microsoft.sqlserver.jdbc.SQLServerBulkCopyOptions;

try (Connection conn = dataSource.getConnection();
     SQLServerBulkCopy bulkCopy = new SQLServerBulkCopy(conn)) {
    
    SQLServerBulkCopyOptions options = new SQLServerBulkCopyOptions();
    options.setBatchSize(5000);
    options.setBulkCopyTimeout(600);
    options.setTableLock(true);
    bulkCopy.setBulkCopyOptions(options);
    bulkCopy.setDestinationTableName("dbo.LargeTable");
    
    // Write from a ResultSet (e.g., from another database)
    try (Statement stmt = sourceConn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT * FROM SourceTable")) {
        bulkCopy.writeToServer(rs);
    }
}
```

## ResultSet Streaming for Large Results

```java
// Stream large result sets to avoid loading everything into memory
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement(
         ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)) {
    
    // Enable adaptive response buffering (default in newer drivers)
    stmt.setFetchSize(1000);
    ((com.microsoft.sqlserver.jdbc.SQLServerStatement) stmt)
        .setResponseBuffering("adaptive");
    
    try (ResultSet rs = stmt.executeQuery("SELECT * FROM VeryLargeTable")) {
        while (rs.next()) {
            processRow(rs);
        }
    }
}
```

## Transactions

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);

Savepoint savepoint = null;
try {
    // First operation
    try (PreparedStatement ps = conn.prepareStatement(
            "UPDATE Accounts SET Balance = Balance - ? WHERE AccountId = ?")) {
        ps.setBigDecimal(1, new BigDecimal("500.00"));
        ps.setInt(2, 1001);
        ps.executeUpdate();
    }
    
    savepoint = conn.setSavepoint("after_debit");
    
    // Second operation
    try (PreparedStatement ps = conn.prepareStatement(
            "UPDATE Accounts SET Balance = Balance + ? WHERE AccountId = ?")) {
        ps.setBigDecimal(1, new BigDecimal("500.00"));
        ps.setInt(2, 1002);
        ps.executeUpdate();
    }
    
    conn.commit();
} catch (SQLException e) {
    if (savepoint != null) {
        conn.rollback(savepoint);
    } else {
        conn.rollback();
    }
    throw e;
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

## Always Encrypted Support

```java
// Enable Always Encrypted in connection string
String url = "jdbc:sqlserver://myserver:1433;"
    + "databaseName=mydb;"
    + "columnEncryptionSetting=Enabled;"
    + "keyStoreAuthentication=JavaKeyStorePassword;"
    + "keyStoreLocation=/path/to/keystore.jks;"
    + "keyStoreSecret=keystorePassword;";

// Queries work transparently — encryption/decryption is automatic
try (Connection conn = DriverManager.getConnection(url);
     PreparedStatement ps = conn.prepareStatement(
         "SELECT PatientId, SSN FROM Patients WHERE SSN = ?")) {
    ps.setString(1, "123-45-6789");  // encrypted automatically
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            // SSN decrypted automatically
            System.out.println(rs.getString("SSN"));
        }
    }
}
```

## Spring Boot Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:sqlserver://myserver:1433;databaseName=mydb;encrypt=true
    username: myuser
    password: mypassword
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1800000
      connection-timeout: 10000
```

## Best Practices

- Use HikariCP for connection pooling in production; the driver's built-in `SQLServerConnectionPoolDataSource` is adequate only for lightweight usage.
- Always use `PreparedStatement` with `?` placeholders — never concatenate values into SQL strings.
- Set `sendStringParametersAsUnicode=false` in HikariCP if your database uses VARCHAR columns to avoid implicit NVARCHAR conversions.
- Use `addBatch()`/`executeBatch()` for multi-row operations and `SQLServerBulkCopy` for very large loads (10K+ rows).
- Enable `responseBuffering=adaptive` (default since driver 2.0) and set appropriate `fetchSize` for large result sets.
- Close all JDBC resources (Connection, Statement, ResultSet) in try-with-resources blocks.
- Set `loginTimeout` and `queryTimeout` to prevent indefinite hangs.
- Use `JavaKerberos` authentication scheme on Linux for Windows integrated auth.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not closing ResultSet/Statement/Connection | Connection pool exhaustion, resource leaks | Use try-with-resources for all JDBC objects |
| String concatenation in SQL | SQL injection, no plan cache reuse | Use PreparedStatement with `?` placeholders |
| Using wrong `jre` artifact version | `UnsupportedClassVersionError` at runtime | Match `jre8`/`jre11` artifact to your Java version |
| Not setting `sendStringParametersAsUnicode=false` | Implicit NVARCHAR conversion prevents index seeks | Set in DataSource properties for VARCHAR-heavy schemas |
| Calling `executeBatch()` without `setAutoCommit(false)` | Each batch statement auto-commits individually | Disable autocommit, batch, then commit once |
| Ignoring `BatchUpdateException.getUpdateCounts()` | No visibility into which rows failed | Check update counts to identify failed statements |
| Using `ResultSet.TYPE_SCROLL_INSENSITIVE` unnecessarily | Server-side cursor overhead, slower performance | Default `TYPE_FORWARD_ONLY` unless scrolling is needed |

## SQL Server Version Notes

- **SQL Server 2016** — JDBC driver 6.0+ required. Always Encrypted and temporal table support added.
- **SQL Server 2019** — JDBC driver 7.4+ recommended. UTF-8 collation support, accelerated database recovery.
- **SQL Server 2022** — JDBC driver 12.2+ recommended. Ledger tables, enhanced Always Encrypted with secure enclaves, TDS 8.0 with strict encryption.
- **JDBC Driver 12.x** — Requires Java 11 minimum. Adds `accessTokenCallbackClass` for custom token providers. `encrypt=true` by default (breaking from older drivers where it defaulted to false).
- **Azure SQL** — Azure AD authentication requires JDBC driver 9.2+. Use `authentication=ActiveDirectoryManagedIdentity` for Azure-hosted apps.

## Sources

- https://learn.microsoft.com/en-us/sql/connect/jdbc/microsoft-jdbc-driver-for-sql-server
- https://learn.microsoft.com/en-us/sql/connect/jdbc/building-the-connection-url
- https://learn.microsoft.com/en-us/sql/connect/jdbc/using-always-encrypted-with-the-jdbc-driver
- https://learn.microsoft.com/en-us/sql/connect/jdbc/using-bulk-copy-with-the-jdbc-driver
- https://github.com/microsoft/mssql-jdbc
- https://github.com/brettwooldridge/HikariCP
