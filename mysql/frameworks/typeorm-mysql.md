# TypeORM MySQL — DataSource Configuration, Entity Decorators, and Migration Patterns

## Overview

TypeORM is a TypeScript-first ORM that supports both Active Record and Data Mapper patterns. When used with MySQL, TypeORM provides decorator-based entity definitions that map directly to MySQL tables, a powerful QueryBuilder for complex queries, and a migration system that can auto-generate migration files by diffing your entity definitions against the current database schema.

TypeORM's MySQL driver is `mysql2`, which provides prepared statements, connection pooling, and streaming. TypeORM handles MySQL-specific features like AUTO_INCREMENT, ENUM, JSON columns, spatial types, and fulltext indexes through its column type system and decorator options.

This skill covers DataSource configuration, entity decorators for MySQL types, migration workflows, QueryBuilder vs Repository patterns, relations, connection pooling, and subscriber patterns for MySQL deployments.

## Key Concepts

- **DataSource**: The primary configuration object holding connection details, entity metadata, and pool settings.
- **Entity**: A TypeScript class decorated with `@Entity()` that maps to a MySQL table.
- **Repository**: A per-entity object for CRUD operations (find, save, delete, query building).
- **QueryBuilder**: A fluent API for constructing complex SQL queries with type safety.
- **Migration**: A TypeScript file with `up()` and `down()` methods for schema changes.
- **Subscriber**: An event listener that hooks into entity lifecycle events (beforeInsert, afterUpdate, etc.).

## DataSource Configuration

### Basic configuration

```typescript
// data-source.ts
import { DataSource } from "typeorm";

export const AppDataSource = new DataSource({
  type: "mysql",
  host: process.env.DB_HOST || "localhost",
  port: parseInt(process.env.DB_PORT || "3306"),
  username: process.env.DB_USER || "app_user",
  password: process.env.DB_PASSWORD || "secret",
  database: process.env.DB_NAME || "myapp",
  charset: "utf8mb4_unicode_ci",

  // Entity and migration paths
  entities: [__dirname + "/entities/**/*.{ts,js}"],
  migrations: [__dirname + "/migrations/**/*.{ts,js}"],
  subscribers: [__dirname + "/subscribers/**/*.{ts,js}"],

  // Connection pool
  extra: {
    connectionLimit: 20,
    connectTimeout: 10000,
    waitForConnections: true,
    queueLimit: 0,
  },

  // Logging
  logging: process.env.NODE_ENV === "development" ? "all" : ["error", "warn"],
  logger: "advanced-console",

  // Synchronize (NEVER true in production)
  synchronize: false,
  migrationsRun: false,

  // MySQL-specific
  timezone: "+00:00",
  supportBigNumbers: true,
  bigNumberStrings: false,
});
```

### Read replica configuration

```typescript
export const AppDataSource = new DataSource({
  type: "mysql",
  replication: {
    master: {
      host: "primary.db.internal",
      port: 3306,
      username: "app_write",
      password: process.env.DB_WRITE_PASSWORD,
      database: "myapp",
    },
    slaves: [
      {
        host: "replica-1.db.internal",
        port: 3306,
        username: "app_read",
        password: process.env.DB_READ_PASSWORD,
        database: "myapp",
      },
      {
        host: "replica-2.db.internal",
        port: 3306,
        username: "app_read",
        password: process.env.DB_READ_PASSWORD,
        database: "myapp",
      },
    ],
  },
  entities: [__dirname + "/entities/**/*.{ts,js}"],
});
```

### Initializing the DataSource

```typescript
// index.ts
import { AppDataSource } from "./data-source";

async function bootstrap() {
  try {
    await AppDataSource.initialize();
    console.log("MySQL DataSource initialized");
  } catch (error) {
    console.error("DataSource initialization failed:", error);
    process.exit(1);
  }
}
```

## Entity Decorators and MySQL Column Types

### Complete entity example

```typescript
import {
  Entity, PrimaryGeneratedColumn, Column, Index,
  CreateDateColumn, UpdateDateColumn, DeleteDateColumn,
  VersionColumn, OneToMany, ManyToOne, JoinColumn,
} from "typeorm";

@Entity("products")
@Index("idx_category_status", ["category", "status"])
export class Product {
  @PrimaryGeneratedColumn("increment", { type: "bigint", unsigned: true })
  id: number;

  @Column({ type: "varchar", length: 255, nullable: false })
  name: string;

  @Column({ type: "mediumtext", nullable: true })
  description: string;

  @Column({ type: "decimal", precision: 10, scale: 2, nullable: false })
  price: string; // DECIMAL returned as string for precision

  @Column({ type: "int", unsigned: true, default: 0 })
  quantity: number;

  @Column({ type: "varchar", length: 100, nullable: true })
  category: string;

  @Column({
    type: "enum",
    enum: ["active", "inactive", "archived"],
    default: "active",
  })
  status: string;

  @Column({ type: "json", nullable: true })
  metadata: Record<string, any>;

  @Column({ type: "json", nullable: true })
  tags: string[];

  @Column({ type: "tinyint", width: 1, default: 1 })
  isVisible: boolean;

  @Column({ type: "double", nullable: true })
  weight: number;

  @Column({ type: "varchar", length: 50, unique: true })
  sku: string;

  @CreateDateColumn({ type: "timestamp" })
  createdAt: Date;

  @UpdateDateColumn({ type: "timestamp" })
  updatedAt: Date;

  @DeleteDateColumn({ type: "timestamp", nullable: true })
  deletedAt: Date; // Soft delete

  @VersionColumn()
  version: number; // Optimistic locking

  @OneToMany(() => OrderItem, (item) => item.product)
  orderItems: OrderItem[];
}
```

### MySQL-specific column types

```typescript
// All MySQL column types available in TypeORM
@Column({ type: "varchar", length: 255 })       // VARCHAR(255)
@Column({ type: "char", length: 36 })            // CHAR(36) — for UUIDs
@Column({ type: "text" })                        // TEXT
@Column({ type: "tinytext" })                    // TINYTEXT
@Column({ type: "mediumtext" })                  // MEDIUMTEXT
@Column({ type: "longtext" })                    // LONGTEXT
@Column({ type: "int" })                         // INT
@Column({ type: "int", unsigned: true })         // INT UNSIGNED
@Column({ type: "tinyint" })                     // TINYINT
@Column({ type: "smallint" })                    // SMALLINT
@Column({ type: "mediumint" })                   // MEDIUMINT
@Column({ type: "bigint", unsigned: true })      // BIGINT UNSIGNED
@Column({ type: "float" })                       // FLOAT
@Column({ type: "double" })                      // DOUBLE
@Column({ type: "decimal", precision: 10, scale: 2 }) // DECIMAL(10,2)
@Column({ type: "date" })                        // DATE
@Column({ type: "datetime" })                    // DATETIME
@Column({ type: "timestamp" })                   // TIMESTAMP
@Column({ type: "time" })                        // TIME
@Column({ type: "year" })                        // YEAR
@Column({ type: "json" })                        // JSON
@Column({ type: "blob" })                        // BLOB
@Column({ type: "mediumblob" })                  // MEDIUMBLOB
@Column({ type: "longblob" })                    // LONGBLOB
@Column({ type: "binary", length: 16 })          // BINARY(16)
@Column({ type: "varbinary", length: 255 })      // VARBINARY(255)
@Column({ type: "enum", enum: ["a", "b", "c"] }) // ENUM('a','b','c')
@Column({ type: "set", enum: ["x", "y", "z"] })  // SET('x','y','z')
@Column({ type: "geometry" })                     // GEOMETRY
@Column({ type: "point" })                        // POINT
```

## Migrations

### CLI commands

```bash
# Generate migration by diffing entities against current DB schema
npx typeorm migration:generate -d src/data-source.ts src/migrations/AddProductFields

# Create empty migration file
npx typeorm migration:create src/migrations/SeedCategories

# Run pending migrations
npx typeorm migration:run -d src/data-source.ts

# Revert last migration
npx typeorm migration:revert -d src/data-source.ts

# Show migration status
npx typeorm migration:show -d src/data-source.ts
```

### Auto-generated migration example

```typescript
// migrations/1700000000000-AddProductFields.ts
import { MigrationInterface, QueryRunner } from "typeorm";

export class AddProductFields1700000000000 implements MigrationInterface {
  name = "AddProductFields1700000000000";

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE \`products\`
      ADD \`weight\` double NULL
    `);
    await queryRunner.query(`
      ALTER TABLE \`products\`
      ADD \`sku\` varchar(50) NOT NULL
    `);
    await queryRunner.query(`
      ALTER TABLE \`products\`
      ADD UNIQUE INDEX \`uk_sku\` (\`sku\`)
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE \`products\` DROP INDEX \`uk_sku\``);
    await queryRunner.query(`ALTER TABLE \`products\` DROP COLUMN \`sku\``);
    await queryRunner.query(`ALTER TABLE \`products\` DROP COLUMN \`weight\``);
  }
}
```

### Custom migration with MySQL-specific DDL

```typescript
import { MigrationInterface, QueryRunner } from "typeorm";

export class AddFulltextIndex1700000001000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // MySQL FULLTEXT index
    await queryRunner.query(`
      ALTER TABLE products
      ADD FULLTEXT INDEX ft_name_desc (name, description)
    `);
    // Partitioning
    await queryRunner.query(`
      ALTER TABLE audit_log
      PARTITION BY RANGE (YEAR(created_at)) (
        PARTITION p2023 VALUES LESS THAN (2024),
        PARTITION p2024 VALUES LESS THAN (2025),
        PARTITION pmax VALUES LESS THAN MAXVALUE
      )
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE products DROP INDEX ft_name_desc`);
    await queryRunner.query(`ALTER TABLE audit_log REMOVE PARTITIONING`);
  }
}
```

## QueryBuilder vs Repository Pattern

### Repository pattern (simple CRUD)

```typescript
const productRepo = AppDataSource.getRepository(Product);

// Find
const product = await productRepo.findOneBy({ id: 1 });
const products = await productRepo.find({
  where: { status: "active", category: "electronics" },
  order: { createdAt: "DESC" },
  take: 20,
  skip: 0,
});

// Save (insert or update)
const newProduct = productRepo.create({ name: "Widget", price: "9.99", sku: "WDG-001" });
await productRepo.save(newProduct);

// Soft delete (requires @DeleteDateColumn)
await productRepo.softDelete(1);

// Restore soft-deleted
await productRepo.restore(1);
```

### QueryBuilder (complex queries)

```typescript
// Complex join with aggregation
const results = await AppDataSource
  .getRepository(Order)
  .createQueryBuilder("order")
  .innerJoin("order.customer", "customer")
  .leftJoinAndSelect("order.items", "item")
  .leftJoinAndSelect("item.product", "product")
  .where("order.status = :status", { status: "pending" })
  .andWhere("order.createdAt > :since", { since: new Date("2024-01-01") })
  .andWhere("customer.tier = :tier", { tier: "premium" })
  .orderBy("order.total", "DESC")
  .take(50)
  .getMany();

// Subquery
const avgPrice = AppDataSource
  .getRepository(Product)
  .createQueryBuilder("p")
  .select("AVG(p.price)", "avg")
  .getQuery();

const expensiveProducts = await AppDataSource
  .getRepository(Product)
  .createQueryBuilder("product")
  .where(`product.price > (${avgPrice})`)
  .getMany();

// Raw SQL via QueryBuilder
const stats = await AppDataSource.query(`
  SELECT
    DATE(created_at) AS date,
    COUNT(*) AS order_count,
    SUM(total) AS revenue
  FROM orders
  WHERE created_at >= ?
  GROUP BY DATE(created_at)
  ORDER BY date DESC
`, [startDate]);
```

## Relations (Lazy and Eager)

```typescript
@Entity("orders")
export class Order {
  @PrimaryGeneratedColumn("increment", { type: "bigint", unsigned: true })
  id: number;

  // Eager: always loaded with the order
  @ManyToOne(() => Customer, { eager: true })
  @JoinColumn({ name: "customer_id" })
  customer: Customer;

  // Lazy: loaded on first access (returns Promise)
  @OneToMany(() => OrderItem, (item) => item.order, { lazy: true })
  items: Promise<OrderItem[]>;

  @Column({ type: "decimal", precision: 10, scale: 2 })
  total: string;
}

// Accessing lazy relation
const order = await orderRepo.findOneBy({ id: 1 });
const items = await order.items; // SQL executed here
```

## Subscriber Patterns

```typescript
import {
  EntitySubscriberInterface, EventSubscriber,
  InsertEvent, UpdateEvent, RemoveEvent,
} from "typeorm";
import { Product } from "../entities/Product";

@EventSubscriber()
export class ProductSubscriber implements EntitySubscriberInterface<Product> {
  listenTo() {
    return Product;
  }

  beforeInsert(event: InsertEvent<Product>) {
    event.entity.sku = event.entity.sku?.toUpperCase();
  }

  afterInsert(event: InsertEvent<Product>) {
    console.log(`Product created: ${event.entity.id}`);
  }

  afterUpdate(event: UpdateEvent<Product>) {
    // Audit trail
    if (event.updatedColumns.some((col) => col.propertyName === "price")) {
      console.log(`Price changed for product ${event.entity?.id}`);
    }
  }

  afterSoftRemove(event: RemoveEvent<Product>) {
    console.log(`Product soft-deleted: ${event.entityId}`);
  }
}
```

## Best Practices

- Never set `synchronize: true` in production; always use migrations for schema changes.
- Use `migration:generate` to auto-generate migration files from entity changes, then review before running.
- Set `charset: "utf8mb4_unicode_ci"` in the DataSource configuration for full Unicode support.
- Use `@PrimaryGeneratedColumn("increment")` with `bigint unsigned` for high-volume tables.
- Prefer the Repository pattern for simple CRUD; use QueryBuilder for joins, subqueries, and aggregations.
- Use `@DeleteDateColumn` for soft deletes instead of a manual `isDeleted` boolean column.
- Use `@VersionColumn` for optimistic locking on entities with concurrent write patterns.
- Set `extra.connectionLimit` based on MySQL `max_connections` divided by application instance count.
- Use `@Index` decorators on entities rather than creating indexes in separate migrations.
- Always review auto-generated migrations before running; they sometimes produce suboptimal DDL.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `synchronize: true` in production | TypeORM silently alters tables, potentially dropping columns or indexes | Set `synchronize: false` and use `migration:generate` + `migration:run` |
| Not setting `charset` to `utf8mb4` | 4-byte characters (emoji) cause MySQL errors on insert | Set `charset: "utf8mb4_unicode_ci"` in DataSource config |
| Using eager relations on entities loaded in bulk | Every `findAll()` triggers N+1 joins, massive query overhead | Default to lazy relations; use `relations` option or QueryBuilder `leftJoinAndSelect` when needed |
| Calling `migration:generate` without running pending migrations first | Generated migration includes changes already in previous unapplied migrations, causing conflicts | Always run `migration:run` before generating new migrations |
| Using `@PrimaryGeneratedColumn("uuid")` without understanding MySQL impact | CHAR(36) primary keys are 4x larger than BIGINT, degrade index performance | Prefer `@PrimaryGeneratedColumn("increment")` for MySQL; use UUID only when distributed ID generation is required |

## MySQL Version Notes

- **5.7**: JSON columns supported (5.7.8+). No window functions. TypeORM generates `utf8` charset by default on older driver versions.
- **8.0**: Full JSON support including `JSON_TABLE`. Window functions available in raw queries. Default auth `caching_sha2_password` requires `mysql2` v2.0+. Atomic DDL for crash-safe migrations.
- **8.4/9.x**: `mysql_native_password` removed. Use `mysql2` v3.0+. CHECK constraints enforced. Multi-valued indexes on JSON arrays available via raw migration SQL.

## Sources

- [TypeORM Documentation](https://typeorm.io/)
- [TypeORM MySQL Connection Options](https://typeorm.io/data-source-options#mysql--mariadb-data-source-options)
- [TypeORM Migrations](https://typeorm.io/migrations)
- [TypeORM Entity Decorators](https://typeorm.io/entities)
- [mysql2 Driver](https://github.com/sidorares/node-mysql2)
