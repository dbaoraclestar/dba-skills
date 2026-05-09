# Sequelize MySQL — ORM Configuration, Models, Migrations, and Query Patterns

## Overview

Sequelize is a promise-based Node.js ORM that supports MySQL, PostgreSQL, SQLite, and MSSQL. When targeting MySQL, Sequelize translates JavaScript model definitions and query calls into MySQL-specific SQL, handling AUTO_INCREMENT, ENUM, JSON columns, character sets, and storage engine configuration through its dialect layer.

Sequelize provides a full lifecycle: define models, generate and run migrations with `sequelize-cli`, establish associations between models, execute queries via the ORM or raw SQL, and manage transactions. Its MySQL dialect driver is `mysql2`, which supports prepared statements, connection pooling, and both the classic and X protocols.

This skill covers connection configuration, model definition with MySQL type mappings, migrations, associations, transactions, raw queries, and MySQL-specific options for production deployments.

## Key Concepts

- **Sequelize instance**: The main entry point that holds the connection pool and dialect configuration.
- **Model**: A JavaScript class that maps to a MySQL table, defining columns, types, and constraints.
- **Migration**: A versioned JavaScript file that describes a forward and backward schema change.
- **Association**: A relationship between models (hasOne, hasMany, belongsTo, belongsToMany).
- **DataTypes**: Sequelize's type system that maps to MySQL column types (STRING to VARCHAR, INTEGER to INT, etc.).
- **Paranoid mode**: Soft-delete behavior using a `deletedAt` timestamp instead of physical DELETE.

## Connection Configuration

### Basic connection

```javascript
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize('mydb', 'user', 'password', {
  host: 'db-primary',
  port: 3306,
  dialect: 'mysql',
  dialectOptions: {
    charset: 'utf8mb4',
    connectTimeout: 10000,
    supportBigNumbers: true,
    bigNumberStrings: true,
  },
  define: {
    charset: 'utf8mb4',
    collate: 'utf8mb4_unicode_ci',
    engine: 'InnoDB',
    timestamps: true,       // Adds createdAt and updatedAt
    underscored: true,      // Use snake_case column names
    freezeTableName: true,  // Don't pluralize table names
  },
  pool: {
    max: 20,
    min: 5,
    acquire: 30000,         // Max time (ms) to acquire connection
    idle: 10000,            // Max idle time before release
    evict: 60000,           // Eviction interval
  },
  logging: false,           // Set to console.log for debugging
  timezone: '+00:00',       // Store timestamps in UTC
});
```

### Connection with read replicas

```javascript
const sequelize = new Sequelize('mydb', null, null, {
  dialect: 'mysql',
  replication: {
    read: [
      { host: 'replica-1', username: 'readonly', password: 'pass' },
      { host: 'replica-2', username: 'readonly', password: 'pass' },
    ],
    write: {
      host: 'primary', username: 'app_user', password: 'pass',
    },
  },
  pool: {
    max: 20,
    min: 5,
  },
  dialectOptions: {
    charset: 'utf8mb4',
  },
});
```

### Connection URL format

```javascript
const sequelize = new Sequelize(
  'mysql://user:password@host:3306/mydb?charset=utf8mb4',
  {
    dialect: 'mysql',
    pool: { max: 20, min: 5, idle: 10000 },
  }
);
```

### Testing the connection

```javascript
async function testConnection() {
  try {
    await sequelize.authenticate();
    console.log('MySQL connection established successfully.');
  } catch (error) {
    console.error('Unable to connect to MySQL:', error);
  }
}
```

## Model Definition

### DataTypes mapping to MySQL

```javascript
const { DataTypes, Model } = require('sequelize');

class Product extends Model {}

Product.init({
  id: {
    type: DataTypes.BIGINT.UNSIGNED,
    autoIncrement: true,
    primaryKey: true,
  },
  name: {
    type: DataTypes.STRING(255),     // VARCHAR(255)
    allowNull: false,
    validate: {
      notEmpty: true,
      len: [2, 255],
    },
  },
  description: {
    type: DataTypes.TEXT('medium'),   // MEDIUMTEXT
  },
  price: {
    type: DataTypes.DECIMAL(10, 2),  // DECIMAL(10,2)
    allowNull: false,
    validate: {
      min: 0,
    },
  },
  quantity: {
    type: DataTypes.INTEGER.UNSIGNED, // INT UNSIGNED
    defaultValue: 0,
  },
  status: {
    type: DataTypes.ENUM('active', 'inactive', 'archived'),
    defaultValue: 'active',
  },
  tags: {
    type: DataTypes.JSON,             // JSON column (MySQL 5.7.8+)
    defaultValue: [],
  },
  metadata: {
    type: DataTypes.JSON,
    defaultValue: {},
  },
  isVisible: {
    type: DataTypes.TINYINT(1),       // TINYINT(1) for boolean
    defaultValue: 1,
  },
  weight: {
    type: DataTypes.DOUBLE,           // DOUBLE
  },
  sku: {
    type: DataTypes.STRING(50),
    unique: true,
  },
}, {
  sequelize,
  modelName: 'Product',
  tableName: 'products',
  charset: 'utf8mb4',
  collate: 'utf8mb4_unicode_ci',
  engine: 'InnoDB',
  timestamps: true,
  underscored: true,
  paranoid: true,    // Soft delete: adds deleted_at column
  indexes: [
    { fields: ['status'] },
    { fields: ['name', 'status'], name: 'idx_name_status' },
    { unique: true, fields: ['sku'], name: 'uk_sku' },
    { type: 'FULLTEXT', fields: ['name', 'description'], name: 'ft_name_desc' },
  ],
});
```

### Additional MySQL DataTypes

```javascript
// All available MySQL-relevant DataTypes
DataTypes.STRING(100)         // VARCHAR(100)
DataTypes.STRING.BINARY       // VARCHAR BINARY
DataTypes.TEXT                 // TEXT
DataTypes.TEXT('tiny')         // TINYTEXT
DataTypes.TEXT('medium')       // MEDIUMTEXT
DataTypes.TEXT('long')         // LONGTEXT
DataTypes.BOOLEAN             // TINYINT(1)
DataTypes.INTEGER             // INT
DataTypes.INTEGER.UNSIGNED    // INT UNSIGNED
DataTypes.BIGINT              // BIGINT
DataTypes.BIGINT.UNSIGNED     // BIGINT UNSIGNED
DataTypes.FLOAT               // FLOAT
DataTypes.DOUBLE              // DOUBLE
DataTypes.DECIMAL(10, 2)      // DECIMAL(10,2)
DataTypes.DATE                // DATETIME
DataTypes.DATEONLY            // DATE
DataTypes.TIME                // TIME
DataTypes.JSON                // JSON
DataTypes.BLOB                // BLOB
DataTypes.BLOB('medium')      // MEDIUMBLOB
DataTypes.BLOB('long')        // LONGBLOB
DataTypes.UUID                // CHAR(36)
DataTypes.ENUM('a', 'b')      // ENUM('a','b')
DataTypes.GEOMETRY            // GEOMETRY
DataTypes.GEOMETRY('POINT')   // POINT
```

## Migrations with sequelize-cli

### Setup

```bash
npx sequelize-cli init
# Creates: config/, models/, migrations/, seeders/
```

### config/config.json

```json
{
  "development": {
    "username": "root",
    "password": "password",
    "database": "myapp_dev",
    "host": "127.0.0.1",
    "dialect": "mysql",
    "dialectOptions": { "charset": "utf8mb4" },
    "define": { "charset": "utf8mb4", "collate": "utf8mb4_unicode_ci" }
  },
  "production": {
    "use_env_variable": "DATABASE_URL",
    "dialect": "mysql",
    "dialectOptions": { "charset": "utf8mb4", "ssl": { "rejectUnauthorized": true } }
  }
}
```

### Creating and running migrations

```bash
# Generate a migration
npx sequelize-cli migration:generate --name create-products

# Run pending migrations
npx sequelize-cli db:migrate

# Undo last migration
npx sequelize-cli db:migrate:undo

# Undo all migrations
npx sequelize-cli db:migrate:undo:all
```

### Migration file example

```javascript
// migrations/20241115120000-create-products.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('products', {
      id: {
        type: Sequelize.BIGINT.UNSIGNED,
        autoIncrement: true,
        primaryKey: true,
      },
      name: {
        type: Sequelize.STRING(255),
        allowNull: false,
      },
      price: {
        type: Sequelize.DECIMAL(10, 2),
        allowNull: false,
      },
      status: {
        type: Sequelize.ENUM('active', 'inactive', 'archived'),
        defaultValue: 'active',
      },
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP'),
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.literal('CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP'),
      },
    }, {
      charset: 'utf8mb4',
      collate: 'utf8mb4_unicode_ci',
      engine: 'InnoDB',
    });

    await queryInterface.addIndex('products', ['status'], { name: 'idx_status' });
  },

  async down(queryInterface) {
    await queryInterface.dropTable('products');
  },
};
```

## Associations

```javascript
// models/index.js — define associations after all models are loaded

const Customer = require('./customer');
const Order = require('./order');
const OrderItem = require('./orderItem');
const Product = require('./product');
const Tag = require('./tag');

// One-to-Many
Customer.hasMany(Order, { foreignKey: 'customer_id', as: 'orders' });
Order.belongsTo(Customer, { foreignKey: 'customer_id', as: 'customer' });

// One-to-Many with cascade
Order.hasMany(OrderItem, {
  foreignKey: 'order_id',
  as: 'items',
  onDelete: 'CASCADE',
  onUpdate: 'CASCADE',
});
OrderItem.belongsTo(Order, { foreignKey: 'order_id' });

// Many-to-Many through junction table
Product.belongsToMany(Tag, {
  through: 'product_tags',
  foreignKey: 'product_id',
  otherKey: 'tag_id',
  as: 'tags',
});
Tag.belongsToMany(Product, {
  through: 'product_tags',
  foreignKey: 'tag_id',
  otherKey: 'product_id',
  as: 'products',
});
```

## Queries and Transactions

### Common queries

```javascript
// Find with eager loading (JOIN)
const orders = await Order.findAll({
  where: { status: 'pending' },
  include: [
    { model: Customer, as: 'customer', attributes: ['name', 'email'] },
    { model: OrderItem, as: 'items', include: [{ model: Product }] },
  ],
  order: [['created_at', 'DESC']],
  limit: 20,
  offset: 0,
});

// Upsert (INSERT ... ON DUPLICATE KEY UPDATE)
const [product, created] = await Product.upsert({
  sku: 'ABC-123',
  name: 'Widget',
  price: 9.99,
});

// Bulk create with ignoreDuplicates
await Product.bulkCreate(products, {
  ignoreDuplicates: true,        // INSERT IGNORE
  updateOnDuplicate: ['price'],  // ON DUPLICATE KEY UPDATE price
});
```

### Transactions

```javascript
const t = await sequelize.transaction();
try {
  const order = await Order.create({ customerId: 1, total: 99.99 }, { transaction: t });
  await OrderItem.bulkCreate([
    { orderId: order.id, productId: 1, quantity: 2, price: 49.99 },
    { orderId: order.id, productId: 2, quantity: 1, price: 49.99 },
  ], { transaction: t });
  await t.commit();
} catch (error) {
  await t.rollback();
  throw error;
}
```

### Raw queries

```javascript
const [results, metadata] = await sequelize.query(
  'SELECT * FROM products WHERE JSON_CONTAINS(tags, :tag)',
  {
    replacements: { tag: JSON.stringify('featured') },
    type: sequelize.QueryTypes.SELECT,
  }
);
```

## Hooks (Lifecycle Events)

```javascript
Product.beforeCreate((product) => {
  product.sku = product.sku.toUpperCase();
});

Product.afterDestroy(async (product) => {
  await AuditLog.create({
    action: 'DELETE',
    entityType: 'Product',
    entityId: product.id,
  });
});
```

## Best Practices

- Always set `charset: 'utf8mb4'` and `collate: 'utf8mb4_unicode_ci'` in model and connection config.
- Use `underscored: true` to follow MySQL snake_case naming conventions.
- Set `freezeTableName: true` to prevent Sequelize from pluralizing table names unpredictably.
- Use `BIGINT.UNSIGNED` for primary keys on high-volume tables.
- Enable `paranoid: true` for tables requiring soft deletes (audit, compliance).
- Configure `pool.max` based on MySQL's `max_connections` divided by the number of application instances.
- Use transactions for multi-table writes to maintain data consistency.
- Prefer `bulkCreate` with `updateOnDuplicate` over individual upserts for batch operations.
- Use `include` for eager loading instead of making separate queries (avoid N+1).
- Set `logging: false` in production; use `logging: console.log` only in development.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not setting `charset: utf8mb4` on model or connection | Emoji and 4-byte Unicode cause `ER_TRUNCATED_WRONG_VALUE_FOR_FIELD` | Set `charset` and `collate` at both connection and model level |
| Using `DataTypes.INTEGER` for primary keys on large tables | INT overflow at ~2.1 billion rows | Use `DataTypes.BIGINT.UNSIGNED` |
| Missing `{ transaction: t }` on one query in a transaction block | That query runs outside the transaction, causing partial commits on rollback | Pass the transaction option to every query inside the block |
| Using `findAll` without `limit` on large tables | Returns millions of rows, causes OOM in Node.js | Always add `limit` and `offset` or use pagination |
| Defining associations in model files instead of after all models load | Circular dependency errors; associations silently missing | Define all associations in a central `models/index.js` after importing all models |

## MySQL Version Notes

- **5.7**: JSON column type supported (5.7.8+). No window functions in raw queries. `ENUM` and `SET` work fully. FULLTEXT indexes supported on InnoDB.
- **8.0**: JSON path expressions, `JSON_TABLE`, window functions available in raw queries. Default auth `caching_sha2_password` requires `mysql2` driver v2.0+. Atomic DDL for safer migrations.
- **8.4/9.x**: `mysql_native_password` removed by default. Ensure `mysql2` v3.0+ is used. Enhanced generated columns and functional indexes available via raw SQL migrations.

## Sources

- [Sequelize v6 Documentation](https://sequelize.org/docs/v6/)
- [Sequelize MySQL Dialect](https://sequelize.org/docs/v6/other-topics/dialect-specific-things/#mysql)
- [mysql2 npm Package](https://github.com/sidorares/node-mysql2)
- [sequelize-cli Documentation](https://github.com/sequelize/cli)
- [MySQL JSON Functions](https://dev.mysql.com/doc/refman/8.0/en/json-functions.html)
