# Spring Data JPA MySQL — Datasource Configuration, Entity Mapping, and Repository Patterns

## Overview

Spring Data JPA provides a high-level abstraction over JPA (Java Persistence API) that simplifies database access in Spring Boot applications. When used with MySQL, Spring Data JPA handles connection pooling via HikariCP, entity-to-table mapping, transaction management, and query generation, while allowing MySQL-specific optimizations through native queries and dialect configuration.

The Spring Boot auto-configuration detects the MySQL driver on the classpath and configures a HikariCP connection pool, Hibernate as the JPA provider, and the MySQL dialect automatically. However, production deployments require explicit tuning of pool sizes, timeouts, and Hibernate behaviors to match MySQL's capabilities and limitations.

This skill covers datasource configuration, HikariCP pool tuning, JPA entity mapping for MySQL types, repository patterns, pagination, error handling, and integration with Flyway/Liquibase for schema migrations.

## Key Concepts

- **DataSource**: The connection factory managed by HikariCP that provides pooled MySQL connections.
- **Entity**: A Java class annotated with `@Entity` that maps to a MySQL table.
- **Repository**: A Spring Data interface that auto-generates CRUD and query methods from method names.
- **Dialect**: Hibernate's MySQL-specific SQL generator (e.g., `MySQL8Dialect` for MySQL 8.0 features).
- **HikariCP**: The default connection pool in Spring Boot, known for low latency and high throughput.
- **JPQL**: Java Persistence Query Language, an object-oriented query language translated to MySQL SQL.

## DataSource Configuration

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:mysql://db-primary:3306/myapp?useSSL=true&serverTimezone=UTC&characterEncoding=utf8mb4&useUnicode=true
    username: app_user
    password: ${DB_PASSWORD}
    driver-class-name: com.mysql.cj.jdbc.Driver

    hikari:
      pool-name: MyApp-HikariPool
      minimum-idle: 5
      maximum-pool-size: 20
      idle-timeout: 300000         # 5 minutes
      max-lifetime: 1800000        # 30 minutes (< MySQL wait_timeout)
      connection-timeout: 10000    # 10 seconds
      validation-timeout: 3000
      connection-test-query: SELECT 1
      leak-detection-threshold: 60000  # Log warning if connection held >60s

  jpa:
    hibernate:
      ddl-auto: validate           # Never use 'update' or 'create' in production
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        format_sql: true
        jdbc:
          batch_size: 50
          batch_versioned_data: true
        order_inserts: true
        order_updates: true
        generate_statistics: false  # Enable for debugging
    open-in-view: false            # Disable OSIV for predictable transaction boundaries
    show-sql: false                # Enable for debugging
```

### application.properties (alternative format)

```properties
spring.datasource.url=jdbc:mysql://db-primary:3306/myapp?useSSL=true&serverTimezone=UTC
spring.datasource.username=app_user
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=10000

spring.jpa.hibernate.ddl-auto=validate
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.open-in-view=false
```

## Entity Mapping for MySQL Types

```java
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.LocalDate;

@Entity
@Table(name = "products", indexes = {
    @Index(name = "idx_category", columnList = "category"),
    @Index(name = "idx_name_category", columnList = "name, category"),
    @Index(name = "idx_created", columnList = "createdAt")
})
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // Maps to AUTO_INCREMENT
    @Column(columnDefinition = "BIGINT UNSIGNED")
    private Long id;

    @Column(nullable = false, length = 255)
    private String name;

    @Column(columnDefinition = "MEDIUMTEXT")
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false, columnDefinition = "INT UNSIGNED DEFAULT 0")
    private Integer quantity;

    @Column(length = 50)
    private String category;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20,
            columnDefinition = "ENUM('ACTIVE','INACTIVE','ARCHIVED') DEFAULT 'ACTIVE'")
    private ProductStatus status = ProductStatus.ACTIVE;

    @Column(columnDefinition = "JSON")
    private String metadataJson;

    @Column(nullable = false, updatable = false,
            columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime createdAt;

    @Column(columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    private LocalDateTime updatedAt;

    @Version
    private Integer version;  // Optimistic locking

    // Getters, setters, constructors omitted for brevity

    public enum ProductStatus {
        ACTIVE, INACTIVE, ARCHIVED
    }

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### Relationship mappings

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_order_customer"))
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal total;

    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private OrderStatus status;

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
}
```

## Repository Patterns

### Basic repository

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // Derived query methods (auto-generated SQL)
    List<Product> findByCategory(String category);
    List<Product> findByStatusAndCategory(ProductStatus status, String category);
    Optional<Product> findByNameIgnoreCase(String name);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
    boolean existsByName(String name);
    long countByStatus(ProductStatus status);

    // Pagination
    Page<Product> findByCategory(String category, Pageable pageable);

    // JPQL query
    @Query("SELECT p FROM Product p WHERE p.price > :minPrice ORDER BY p.price DESC")
    List<Product> findExpensiveProducts(@Param("minPrice") BigDecimal minPrice);

    // Native MySQL query
    @Query(value = "SELECT * FROM products WHERE JSON_EXTRACT(metadata_json, '$.featured') = true",
           nativeQuery = true)
    List<Product> findFeaturedProducts();

    // Native query with pagination
    @Query(value = "SELECT * FROM products WHERE category = :category",
           countQuery = "SELECT COUNT(*) FROM products WHERE category = :category",
           nativeQuery = true)
    Page<Product> findByCategoryNative(@Param("category") String category, Pageable pageable);

    // Modifying query
    @Modifying
    @Query("UPDATE Product p SET p.status = :status WHERE p.id IN :ids")
    int bulkUpdateStatus(@Param("ids") List<Long> ids, @Param("status") ProductStatus status);
}
```

### Pagination and sorting

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public Page<Product> getProducts(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, sortBy));
        return productRepository.findAll(pageable);
    }

    public Page<Product> searchByCategory(String category, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());
        return productRepository.findByCategory(category, pageable);
    }
}
```

## Error Handling for MySQL

```java
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.dao.CannotAcquireLockException;
import org.springframework.dao.DeadlockLoserDataAccessException;
import org.springframework.orm.jpa.JpaSystemException;

@Service
public class OrderService {

    @Transactional
    public Order createOrder(OrderRequest request) {
        try {
            Order order = new Order();
            // ... populate order
            return orderRepository.save(order);
        } catch (DataIntegrityViolationException e) {
            // Duplicate key, FK violation, NOT NULL constraint
            if (e.getMessage().contains("Duplicate entry")) {
                throw new DuplicateOrderException("Order already exists");
            }
            throw new InvalidOrderException("Data integrity violation: " + e.getMessage());
        } catch (DeadlockLoserDataAccessException e) {
            // MySQL InnoDB deadlock — retry the transaction
            throw new RetryableException("Deadlock detected, please retry");
        } catch (CannotAcquireLockException e) {
            // Lock wait timeout exceeded
            throw new RetryableException("Lock timeout, please retry");
        }
    }
}
```

## Flyway / Liquibase Integration

### Flyway with Spring Boot

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false
    table: flyway_schema_history
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

### Liquibase with Spring Boot

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    default-schema: myapp
```

```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

## Transaction Management

```java
@Service
public class TransferService {

    @Transactional(isolation = Isolation.READ_COMMITTED, timeout = 30)
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new NotFoundException("Source account not found"));
        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new NotFoundException("Target account not found"));

        from.debit(amount);
        to.credit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
    }

    // Read-only transaction hint for MySQL optimizer
    @Transactional(readOnly = true)
    public List<Account> getAccounts() {
        return accountRepository.findAll();
    }
}
```

## Best Practices

- Always set `spring.jpa.open-in-view=false` to prevent lazy loading outside transactions.
- Use `GenerationType.IDENTITY` for MySQL AUTO_INCREMENT; avoid `SEQUENCE` (not native to MySQL) or `TABLE` (poor performance).
- Set `hibernate.jdbc.batch_size=50` with `order_inserts=true` and `order_updates=true` for efficient batch operations.
- Use `@Transactional(readOnly = true)` for read-only methods to enable MySQL read optimizations.
- Set HikariCP `max-lifetime` to less than MySQL's `wait_timeout` to prevent stale connections.
- Use `ddl-auto=validate` in production; never `update` or `create-drop`.
- Prefer JPQL over native queries for portability; use native queries only for MySQL-specific features (JSON functions, FULLTEXT search).
- Use `@Version` for optimistic locking to handle concurrent updates without database-level locks.
- Enable Hibernate batch operations for bulk inserts/updates (significant performance improvement).
- Handle `DeadlockLoserDataAccessException` with retry logic in services that perform concurrent writes.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `GenerationType.AUTO` instead of `IDENTITY` | Hibernate may use TABLE strategy, creating a `hibernate_sequence` table with poor locking performance | Use `@GeneratedValue(strategy = GenerationType.IDENTITY)` |
| Leaving `open-in-view=true` (default) | Lazy loading triggers SQL queries outside service layer, potential N+1 problems in controllers | Set `spring.jpa.open-in-view=false` |
| Not setting `max-lifetime` on HikariCP | MySQL closes idle connections after `wait_timeout`; HikariCP returns dead connections | Set `max-lifetime` to 30 minutes (less than MySQL `wait_timeout`) |
| Using `ddl-auto=update` in production | Hibernate auto-DDL can create unintended schema changes, miss indexes, and cannot rename columns | Use `ddl-auto=validate` with Flyway/Liquibase for migrations |
| Fetching `@OneToMany` with `FetchType.EAGER` | Loads all related rows on every query; causes cartesian product with multiple eager collections | Default to `FetchType.LAZY`; use `JOIN FETCH` in JPQL when needed |

## MySQL Version Notes

- **5.7**: Use `MySQL57Dialect` or `MySQL5InnoDBDialect`. No window functions in JPQL. JSON column support limited; use `columnDefinition = "JSON"` and native queries for JSON operations.
- **8.0**: Use `MySQL8Dialect`. Full support for window functions, CTEs, and JSON_TABLE in native queries. Default auth plugin `caching_sha2_password` requires MySQL Connector/J 8.0.11+. `useSSL=true` recommended; `allowPublicKeyRetrieval=true` may be needed for dev environments.
- **8.4/9.x**: Continue using `MySQL8Dialect` (Hibernate 6.x auto-detects). `mysql_native_password` removed; ensure Connector/J 8.2+ or 9.0+. Enhanced EXPLAIN output accessible via native queries.

## Sources

- [Spring Boot MySQL Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.datasource)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/jpa.html)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#gear-configuration-knobs-baby)
- [Hibernate MySQL Dialect](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#database-dialect)
- [Flyway Spring Boot](https://documentation.red-gate.com/fd/spring-boot-184127564.html)
