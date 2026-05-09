# Spring Data JPA with SQL Server -- Spring Boot Integration and JPA Patterns

## Overview

Spring Data JPA provides a powerful repository abstraction over JPA (Java Persistence API) for SQL Server, using the Microsoft JDBC driver (`mssql-jdbc`) and HikariCP connection pooling. Spring Boot auto-configures the datasource, entity manager, and transaction manager, allowing developers to define JPA entities and repository interfaces with minimal boilerplate. The framework translates method names, JPQL queries, and native T-SQL into efficient database operations.

Spring Data JPA with SQL Server is the standard choice for Java/Kotlin enterprise applications that need type-safe data access, declarative transactions, auditing, and pagination. It integrates naturally with Spring Security, Spring MVC/WebFlux, and the broader Spring ecosystem.

Use this skill when configuring Spring Boot to connect to SQL Server, mapping JPA entities to SQL Server types, calling stored procedures, or integrating Flyway/Liquibase for schema management within a Spring application.

## Key Concepts

- **Repository interface**: Extend `JpaRepository<Entity, ID>` to get CRUD methods, paging, and sorting for free. Spring generates the implementation at runtime.
- **Entity**: A Java/Kotlin class annotated with `@Entity` that maps to a SQL Server table.
- **JPQL**: Java Persistence Query Language. Object-oriented query language that translates to T-SQL.
- **Native query**: Raw T-SQL executed through JPA using `@Query(nativeQuery = true)`.
- **HikariCP**: Default connection pool in Spring Boot. High-performance, production-ready.
- **Spring Data auditing**: Automatic population of `createdDate`, `lastModifiedDate`, `createdBy` fields.

## Application Configuration

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:sqlserver://sqlserver.example.com:1433;databaseName=MyApp;encrypt=true;trustServerCertificate=true
    username: appuser
    password: ${DB_PASSWORD}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver

  jpa:
    database-platform: org.hibernate.dialect.SQLServer2016Dialect
    hibernate:
      ddl-auto: validate  # Never use 'update' or 'create' in production
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        default_schema: dbo
        jdbc:
          batch_size: 50
          order_inserts: true
          order_updates: true
        query:
          in_clause_parameter_padding: true

  # HikariCP pool settings
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000        # 5 minutes
      max-lifetime: 1800000       # 30 minutes
      connection-timeout: 30000   # 30 seconds
      leak-detection-threshold: 60000  # Warn if connection held > 60s
      connection-test-query: SELECT 1
```

```xml
<!-- pom.xml dependencies -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.microsoft.sqlserver</groupId>
        <artifactId>mssql-jdbc</artifactId>
        <scope>runtime</scope>
    </dependency>
    <!-- Optional: Flyway for migrations -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-sqlserver</artifactId>
    </dependency>
</dependencies>
```

```groovy
// build.gradle (Kotlin DSL)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("com.microsoft.sqlserver:mssql-jdbc")
    implementation("org.flywaydb:flyway-sqlserver")
}
```

## JPA Entity Mapping for SQL Server Types

```java
// entity/Customer.java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.UUID;

@Entity
@Table(name = "Customers", schema = "dbo",
       indexes = {
           @Index(name = "IX_Customers_Email", columnList = "email", unique = true)
       })
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // Maps to IDENTITY(1,1)
    @Column(name = "CustomerID")
    private Integer customerID;

    @Column(name = "ExternalID", columnDefinition = "UNIQUEIDENTIFIER")
    private UUID externalID = UUID.randomUUID();

    @Column(name = "FirstName", nullable = false, length = 100)
    private String firstName;  // NVARCHAR(100)

    @Column(name = "LastName", nullable = false, length = 100)
    private String lastName;

    @Column(name = "Email", nullable = false, length = 256, unique = true)
    private String email;

    @Lob
    @Column(name = "Bio", columnDefinition = "NVARCHAR(MAX)")
    private String bio;

    @Column(name = "Balance", precision = 18, scale = 2)
    private BigDecimal balance = BigDecimal.ZERO;  // DECIMAL(18,2)

    @Column(name = "IsActive", nullable = false)
    private Boolean isActive = true;  // BIT

    @Column(name = "CreatedAt", columnDefinition = "DATETIME2(3)")
    private OffsetDateTime createdAt;

    @Column(name = "UpdatedAt", columnDefinition = "DATETIME2(3)")
    private OffsetDateTime updatedAt;

    @OneToMany(mappedBy = "customer", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();

    @PrePersist
    protected void onCreate() {
        createdAt = OffsetDateTime.now();
        updatedAt = OffsetDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = OffsetDateTime.now();
    }

    // Getters and setters omitted for brevity
}
```

```java
// entity/Order.java
@Entity
@Table(name = "Orders", schema = "dbo")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "OrderID")
    private Integer orderID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "CustomerID", nullable = false)
    private Customer customer;

    @Column(name = "OrderDate", columnDefinition = "DATETIME2(3)")
    private OffsetDateTime orderDate;

    @Column(name = "TotalAmount", precision = 18, scale = 2, nullable = false)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(name = "Status", length = 50)
    private OrderStatus status = OrderStatus.PENDING;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

## Repository Interface

```java
// repository/CustomerRepository.java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import java.util.List;
import java.util.Optional;

public interface CustomerRepository extends JpaRepository<Customer, Integer> {

    // Derived query methods (auto-generated SQL)
    Optional<Customer> findByEmail(String email);
    List<Customer> findByLastNameAndIsActiveTrue(String lastName);
    boolean existsByEmail(String email);
    long countByIsActiveTrue();

    // Pagination
    Page<Customer> findByIsActiveTrue(Pageable pageable);

    // JPQL query
    @Query("SELECT c FROM Customer c WHERE c.isActive = true ORDER BY c.createdAt DESC")
    List<Customer> findAllActiveOrderedByCreation();

    // JPQL with projection
    @Query("SELECT c.firstName, c.lastName, SIZE(c.orders) FROM Customer c WHERE c.customerID = :id")
    Object[] findCustomerSummary(Integer id);

    // Native T-SQL query
    @Query(value = """
        SELECT TOP (:limit) c.CustomerID, c.FirstName, c.LastName,
               COUNT(o.OrderID) AS OrderCount
        FROM dbo.Customers c
        LEFT JOIN dbo.Orders o ON c.CustomerID = o.CustomerID
        WHERE c.IsActive = 1
        GROUP BY c.CustomerID, c.FirstName, c.LastName
        ORDER BY OrderCount DESC
        """, nativeQuery = true)
    List<Object[]> findTopCustomersByOrderCount(int limit);

    // Modifying query (UPDATE/DELETE)
    @Modifying
    @Query("UPDATE Customer c SET c.isActive = false WHERE c.lastLogin < :cutoffDate")
    int deactivateInactiveCustomers(OffsetDateTime cutoffDate);
}
```

## Pagination

```java
// service/CustomerService.java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;

@Service
@Transactional(readOnly = true)
public class CustomerService {

    private final CustomerRepository customerRepository;

    public Page<Customer> getActiveCustomers(int page, int size) {
        PageRequest pageRequest = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "createdAt"));
        return customerRepository.findByIsActiveTrue(pageRequest);
        // Generates: SELECT ... FROM Customers WHERE IsActive = 1
        //            ORDER BY CreatedAt DESC
        //            OFFSET ? ROWS FETCH NEXT ? ROWS ONLY
    }
}
```

## Stored Procedure Calls

```java
// entity/Customer.java -- add stored procedure mapping
@Entity
@Table(name = "Customers", schema = "dbo")
@NamedStoredProcedureQuery(
    name = "Customer.searchCustomers",
    procedureName = "dbo.usp_SearchCustomers",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "SearchTerm", type = String.class),
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "MaxResults", type = Integer.class),
    }
)
public class Customer { /* ... */ }

// repository/CustomerRepository.java
@Procedure(name = "Customer.searchCustomers")
List<Customer> searchCustomers(String SearchTerm, Integer MaxResults);
```

```java
// Alternative: EntityManager for stored procedure calls
@Repository
public class CustomerRepositoryCustom {

    @PersistenceContext
    private EntityManager entityManager;

    public List<Customer> searchCustomers(String term, int maxResults) {
        StoredProcedureQuery query = entityManager
            .createStoredProcedureQuery("dbo.usp_SearchCustomers", Customer.class)
            .registerStoredProcedureParameter("SearchTerm", String.class, ParameterMode.IN)
            .registerStoredProcedureParameter("MaxResults", Integer.class, ParameterMode.IN)
            .setParameter("SearchTerm", term)
            .setParameter("MaxResults", maxResults);

        return query.getResultList();
    }

    // Stored procedure with OUTPUT parameter
    public int getCustomerOrderCount(int customerId) {
        StoredProcedureQuery query = entityManager
            .createStoredProcedureQuery("dbo.usp_GetCustomerStats")
            .registerStoredProcedureParameter("CustomerID", Integer.class, ParameterMode.IN)
            .registerStoredProcedureParameter("OrderCount", Integer.class, ParameterMode.OUT)
            .setParameter("CustomerID", customerId);

        query.execute();
        return (Integer) query.getOutputParameterValue("OrderCount");
    }
}
```

## Flyway/Liquibase Integration

```yaml
# application.yml -- Flyway configuration
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    schemas:
      - dbo
    sql-migration-prefix: V
    sql-migration-separator: __
    validate-on-migrate: true
```

```
# Flyway migration files location
src/main/resources/db/migration/
  V1__create_customers_table.sql
  V2__create_orders_table.sql
  V3__add_customer_email_index.sql
```

```yaml
# application.yml -- Liquibase configuration
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
    default-schema: dbo
```

## Azure AD Authentication

```yaml
# application.yml for Azure AD (passwordless)
spring:
  datasource:
    url: jdbc:sqlserver://myserver.database.windows.net:1433;databaseName=MyApp;encrypt=true;authentication=ActiveDirectoryDefault
```

```java
// Programmatic Azure AD token authentication
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:sqlserver://myserver.database.windows.net:1433;databaseName=MyApp;encrypt=true");
        config.setUsername("appuser@myserver");

        // Azure Identity SDK for token
        DefaultAzureCredential credential = new DefaultAzureCredentialBuilder().build();
        AccessToken token = credential.getToken(
            new TokenRequestContext().addScopes("https://database.windows.net/.default")
        ).block();

        config.addDataSourceProperty("accessToken", token.getToken());
        return new HikariDataSource(config);
    }
}
```

## Best Practices

- Use `GenerationType.IDENTITY` for primary keys -- it maps directly to SQL Server IDENTITY
- Set `hibernate.ddl-auto=validate` in production to prevent accidental schema changes
- Enable Hibernate batch inserts/updates (`batch_size`, `order_inserts`, `order_updates`) for bulk operations
- Use `@Transactional(readOnly = true)` on read-only service methods for performance
- Prefer `FetchType.LAZY` for all associations and use `JOIN FETCH` in JPQL when eager loading is needed
- Set `max-lifetime` in HikariCP shorter than SQL Server's connection timeout
- Use `@Query(nativeQuery = true)` for complex T-SQL that JPQL cannot express efficiently
- Enable `leak-detection-threshold` during development to catch connection leaks
- Use projections (interfaces or DTOs) for read-only queries to avoid loading full entities

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `GenerationType.AUTO` instead of `IDENTITY` | Hibernate may try to use a sequence, which fails on SQL Server | Explicitly use `@GeneratedValue(strategy = GenerationType.IDENTITY)` |
| Setting `ddl-auto=update` in production | Hibernate modifies schema without review, potential data loss | Use `validate` in production; manage schema with Flyway or Liquibase |
| Not setting `fetch = FetchType.LAZY` on `@ManyToOne` | Eager loading causes N+1 queries or cartesian products | Always set `FetchType.LAZY` and use `JOIN FETCH` when needed |
| Forgetting `@Modifying` on UPDATE/DELETE `@Query` methods | Spring throws InvalidDataAccessApiUsageException | Add `@Modifying` and `@Transactional` to non-SELECT query methods |
| Not configuring HikariCP `max-lifetime` | Connection pool holds connections that SQL Server has already closed | Set `max-lifetime` to 1800000 (30 min), less than SQL Server timeout |

## SQL Server Version Notes

- **SQL Server 2016**: Use `SQLServer2016Dialect`. Temporal tables queryable via native SQL but not mapped by Hibernate. JSON functions available in native queries. Always Encrypted requires JDBC driver 6.0+.
- **SQL Server 2019**: Use `SQLServer2016Dialect` (still valid). UTF-8 collation support. Accelerated Database Recovery improves long-transaction rollback. Batch mode on rowstore benefits analytical JPA queries. JDBC driver 7.4+ recommended.
- **SQL Server 2022**: Use `SQLServer2016Dialect` or Hibernate 6.2+ `SQLServerDialect` (auto-detects version). Ledger tables usable via native queries. Native JSON type not yet mapped by Hibernate. JDBC driver 12.2+ recommended for full feature support. Free edition available for dev/test.

## Sources

- https://learn.microsoft.com/en-us/sql/connect/jdbc/microsoft-jdbc-driver-for-sql-server
- https://spring.io/projects/spring-data-jpa
- https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql
- https://github.com/microsoft/mssql-jdbc
- https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html
- https://learn.microsoft.com/en-us/azure/developer/java/spring-framework/configure-spring-data-jpa-with-azure-sql
