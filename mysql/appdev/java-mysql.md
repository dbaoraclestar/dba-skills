# Java MySQL Connectivity — JDBC, Connector/J, and Connection Pooling

## Overview

Java applications connect to MySQL through JDBC (Java Database Connectivity), using MySQL's official driver, Connector/J. JDBC provides a standard API for database access, and Connector/J implements the MySQL-specific wire protocol. Modern Java applications typically use a connection pool like HikariCP on top of the JDBC driver.

Connector/J supports all MySQL features including prepared statements, stored procedures, SSL/TLS, the `caching_sha2_password` authentication plugin, and failover configurations. It is available as a single JAR dependency via Maven Central and is compatible with Java 8 through 21+.

This skill covers JDBC fundamentals, connection URL format, prepared statements, batch operations, transaction management, connection pooling with HikariCP and c3p0, ResultSet handling, and common JDBC errors with their fixes.

## Key Concepts

- **JDBC**: Java's standard API for relational database access. Applications code to `java.sql.*` interfaces; the driver provides the implementation.
- **DriverManager**: Factory that creates connections from a JDBC URL. Suitable for simple apps, not for production connection management.
- **DataSource**: A factory interface that connection pools implement. Preferred over DriverManager in production.
- **PreparedStatement**: Pre-compiled SQL with bind parameters (`?`). Prevents SQL injection and improves performance on repeated execution.
- **ResultSet**: The cursor-like object returned by queries. Supports forward-only or scrollable navigation.
- **Autocommit**: JDBC defaults to autocommit=true. You must disable it explicitly for multi-statement transactions.

## Maven Dependency

```xml
<!-- Connector/J 8.x (works with MySQL 5.7, 8.0, 8.4, 9.x) -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.4.0</version>
</dependency>

<!-- HikariCP connection pool -->
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>5.1.0</version>
</dependency>
```

## Connection URL Format

```
jdbc:mysql://host:port/database?param1=value1&param2=value2
```

```java
// Basic connection URL
String url = "jdbc:mysql://127.0.0.1:3306/mydb";

// With parameters
String url = "jdbc:mysql://127.0.0.1:3306/mydb"
    + "?useSSL=true"
    + "&requireSSL=true"
    + "&serverTimezone=UTC"
    + "&characterEncoding=utf8mb4"
    + "&connectTimeout=10000"    // millis
    + "&socketTimeout=30000"
    + "&allowMultiQueries=false" // keep false for safety
    + "&rewriteBatchedStatements=true"; // critical for batch perf

// Failover / HA
String url = "jdbc:mysql://primary:3306,secondary:3306/mydb"
    + "?failOverReadOnly=true"
    + "&autoReconnect=true";
```

## Basic Connection with DriverManager

```java
import java.sql.*;

public class BasicExample {
    public static void main(String[] args) {
        String url = "jdbc:mysql://127.0.0.1:3306/mydb?serverTimezone=UTC";
        String user = "app_user";
        String pass = "secret";

        try (Connection conn = DriverManager.getConnection(url, user, pass);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT VERSION()")) {

            if (rs.next()) {
                System.out.println("MySQL version: " + rs.getString(1));
            }
        } catch (SQLException e) {
            System.err.println("SQLState: " + e.getSQLState());
            System.err.println("Error Code: " + e.getErrorCode());
            System.err.println("Message: " + e.getMessage());
        }
    }
}
```

## PreparedStatement (Parameterized Queries)

```java
// SELECT with parameters
String sql = "SELECT id, name, email FROM users WHERE status = ? AND created_at > ?";

try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    pstmt.setString(1, "active");
    pstmt.setTimestamp(2, Timestamp.valueOf("2025-01-01 00:00:00"));

    try (ResultSet rs = pstmt.executeQuery()) {
        while (rs.next()) {
            int id = rs.getInt("id");
            String name = rs.getString("name");
            String email = rs.getString("email");
            System.out.printf("id=%d name=%s email=%s%n", id, name, email);
        }
    }
}

// INSERT with parameters and generated keys
String insertSql = "INSERT INTO users (name, email, status) VALUES (?, ?, ?)";

try (PreparedStatement pstmt = conn.prepareStatement(insertSql, Statement.RETURN_GENERATED_KEYS)) {
    pstmt.setString(1, "Alice");
    pstmt.setString(2, "alice@example.com");
    pstmt.setString(3, "active");
    int affected = pstmt.executeUpdate();

    try (ResultSet keys = pstmt.getGeneratedKeys()) {
        if (keys.next()) {
            long newId = keys.getLong(1);
            System.out.println("Inserted with id: " + newId);
        }
    }
}
```

## Batch Operations

```java
// rewriteBatchedStatements=true in the URL is critical for performance
String sql = "INSERT INTO orders (customer_id, product_id, quantity) VALUES (?, ?, ?)";

conn.setAutoCommit(false);
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    for (Order order : orderList) {
        pstmt.setInt(1, order.getCustomerId());
        pstmt.setInt(2, order.getProductId());
        pstmt.setInt(3, order.getQuantity());
        pstmt.addBatch();

        // Execute in chunks to manage memory
        if (pstmt.getUpdateCount() % 1000 == 0) {
            pstmt.executeBatch();
        }
    }
    pstmt.executeBatch(); // flush remaining
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

## Connection Pooling with HikariCP

```java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import javax.sql.DataSource;

public class DataSourceFactory {

    private static HikariDataSource ds;

    public static DataSource getDataSource() {
        if (ds == null) {
            HikariConfig config = new HikariConfig();
            config.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/mydb"
                + "?serverTimezone=UTC&rewriteBatchedStatements=true");
            config.setUsername("app_user");
            config.setPassword("secret");
            config.setMaximumPoolSize(20);
            config.setMinimumIdle(5);
            config.setIdleTimeout(300_000);        // 5 min
            config.setMaxLifetime(1_800_000);      // 30 min
            config.setConnectionTimeout(10_000);   // 10 sec
            config.setConnectionTestQuery("SELECT 1");
            config.setPoolName("mysql-pool");

            // MySQL-specific optimizations
            config.addDataSourceProperty("cachePrepStmts", "true");
            config.addDataSourceProperty("prepStmtCacheSize", "250");
            config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
            config.addDataSourceProperty("useServerPrepStmts", "true");

            ds = new HikariDataSource(config);
        }
        return ds;
    }
}

// Usage
DataSource ds = DataSourceFactory.getDataSource();
try (Connection conn = ds.getConnection();
     PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
    pstmt.setInt(1, 42);
    try (ResultSet rs = pstmt.executeQuery()) {
        // process results
    }
}
```

## Connection Pooling with c3p0

```java
import com.mchange.v2.c3p0.ComboPooledDataSource;

ComboPooledDataSource cpds = new ComboPooledDataSource();
cpds.setDriverClass("com.mysql.cj.jdbc.Driver");
cpds.setJdbcUrl("jdbc:mysql://127.0.0.1:3306/mydb?serverTimezone=UTC");
cpds.setUser("app_user");
cpds.setPassword("secret");
cpds.setMinPoolSize(5);
cpds.setMaxPoolSize(20);
cpds.setAcquireIncrement(3);
cpds.setMaxIdleTime(300);
cpds.setPreferredTestQuery("SELECT 1");
cpds.setTestConnectionOnCheckin(true);
cpds.setIdleConnectionTestPeriod(60);

// Usage is the same DataSource interface
try (Connection conn = cpds.getConnection()) {
    // ...
}
```

## Transaction Management

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);
Savepoint sp = null;

try {
    // Operation 1
    try (PreparedStatement p1 = conn.prepareStatement(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?")) {
        p1.setBigDecimal(1, amount);
        p1.setInt(2, fromAccountId);
        p1.executeUpdate();
    }

    sp = conn.setSavepoint("after_debit");

    // Operation 2
    try (PreparedStatement p2 = conn.prepareStatement(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?")) {
        p2.setBigDecimal(1, amount);
        p2.setInt(2, toAccountId);
        p2.executeUpdate();
    }

    conn.commit();

} catch (SQLException e) {
    if (sp != null) {
        conn.rollback(sp);  // rollback to savepoint
    } else {
        conn.rollback();    // rollback everything
    }
    throw e;
} finally {
    conn.setAutoCommit(true);
    conn.close();
}
```

## ResultSet Handling

```java
// Forward-only (default, most efficient)
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM users");

// Scrollable ResultSet
Statement stmt = conn.createStatement(
    ResultSet.TYPE_SCROLL_INSENSITIVE,
    ResultSet.CONCUR_READ_ONLY
);
ResultSet rs = stmt.executeQuery("SELECT * FROM users");
rs.absolute(5);    // jump to row 5
rs.previous();     // move back
rs.last();         // jump to last row
int rowCount = rs.getRow();

// Streaming large results (fetchSize trick for Connector/J)
Statement stmt = conn.createStatement(
    ResultSet.TYPE_FORWARD_ONLY,
    ResultSet.CONCUR_READ_ONLY
);
stmt.setFetchSize(Integer.MIN_VALUE);  // signals row-by-row streaming
ResultSet rs = stmt.executeQuery("SELECT * FROM huge_table");
while (rs.next()) {
    // process one row at a time, low memory footprint
}
```

## Best Practices

- Always use `PreparedStatement` with `?` placeholders -- never concatenate strings into SQL.
- Use HikariCP for connection pooling; it outperforms c3p0, DBCP, and Tomcat JDBC Pool in benchmarks.
- Set `rewriteBatchedStatements=true` in the JDBC URL for batch insert performance (10x-50x faster).
- Enable prepared statement caching (`cachePrepStmts`, `useServerPrepStmts`) via HikariCP data source properties.
- Always close `ResultSet`, `Statement`, and `Connection` in `finally` blocks or use try-with-resources.
- Set `maxLifetime` in HikariCP to less than MySQL's `wait_timeout` (default 28800s) to avoid broken connections.
- Use `setFetchSize(Integer.MIN_VALUE)` for streaming large result sets to avoid OOM errors.
- Log `SQLState` and `ErrorCode` on exceptions for precise diagnosis.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| String concatenation in SQL queries | SQL injection vulnerability | Use `PreparedStatement` with `?` |
| Not closing connections/statements | Connection pool exhaustion, resource leaks | Use try-with-resources |
| Missing `rewriteBatchedStatements=true` | Batch inserts execute row-by-row (slow) | Add parameter to JDBC URL |
| Default `fetchSize` on huge queries | JVM OutOfMemoryError | Set `fetchSize(Integer.MIN_VALUE)` for streaming |
| Autocommit left on during transactions | Each statement is its own transaction; no atomicity | Call `conn.setAutoCommit(false)` before multi-step logic |
| `maxLifetime` exceeds MySQL `wait_timeout` | "Connection closed" errors from stale pool entries | Set `maxLifetime` below `wait_timeout` |
| Using deprecated `com.mysql.jdbc.Driver` | Warnings or class-not-found in Connector/J 8.x | Use `com.mysql.cj.jdbc.Driver` |

## MySQL Version Notes

- **5.7**: Connector/J 8.x works. Default auth is `mysql_native_password`. No `caching_sha2_password` issues.
- **8.0**: Default auth switched to `caching_sha2_password`. Connector/J 8.0.11+ handles it. SSL required by default on some configurations; set `useSSL=true&requireSSL=true` explicitly.
- **8.4 / 9.x**: `mysql_native_password` plugin removed. Connector/J 8.2+ required. `com.mysql.jdbc.Driver` fully removed; only `com.mysql.cj.jdbc.Driver` works.

## Sources

- [MySQL Connector/J Developer Guide](https://dev.mysql.com/doc/connector-j/en/)
- [HikariCP GitHub](https://github.com/brettwooldridge/HikariCP)
- [JDBC API Specification](https://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/)
- [c3p0 Documentation](https://www.mchange.com/projects/c3p0/)
