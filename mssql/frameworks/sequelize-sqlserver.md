# Sequelize with SQL Server -- Node.js ORM Using the Tedious Dialect

## Overview

Sequelize is the most popular Node.js ORM, and it connects to SQL Server through the `tedious` driver (dialect: `'mssql'`). Sequelize provides model definition, migrations, associations (one-to-one, one-to-many, many-to-many), transactions, and raw query support. The `tedious` dialect translates Sequelize operations into T-SQL, handling SQL Server-specific syntax like `TOP`, `IDENTITY`, `OUTPUT INSERTED`, and `OFFSET...FETCH`.

Sequelize is the right choice for Node.js/TypeScript applications that need a full-featured ORM with SQL Server. It supports both SQL Server Authentication and Windows Authentication (via integrated security), as well as Azure SQL Database with Azure AD tokens. For lighter-weight alternatives, consider `knex.js` (query builder) or `mssql` (raw driver).

Use this skill when configuring Sequelize with SQL Server, mapping JavaScript types to SQL Server data types, managing migrations, or implementing transactional operations.

## Key Concepts

- **Dialect**: Sequelize uses `dialect: 'mssql'` to select the tedious driver. Tedious is a pure JavaScript TDS implementation -- no native binaries required.
- **Model**: A JavaScript class mapped to a SQL Server table. Columns are defined with Sequelize `DataTypes` that translate to SQL Server types.
- **Migration**: A versioned JavaScript file describing schema changes. Managed by `sequelize-cli`.
- **Association**: Defines relationships between models (hasOne, hasMany, belongsTo, belongsToMany).
- **Instance vs Class methods**: Instance methods operate on a single row. Class methods (finders) query the table.

## Connection Configuration

```javascript
// config/database.js
const { Sequelize } = require('sequelize');

// SQL Server Authentication
const sequelize = new Sequelize('MyApp', 'appuser', 'SecureP@ss!', {
  host: 'sqlserver.example.com',
  port: 1433,
  dialect: 'mssql',
  dialectOptions: {
    options: {
      encrypt: true,
      trustServerCertificate: true,  // For self-signed certs (dev/test)
      requestTimeout: 30000,
    }
  },
  pool: {
    max: 20,
    min: 5,
    acquire: 30000,
    idle: 10000,
    evict: 1000,
  },
  logging: console.log,  // Set to false for production
});

// Windows Authentication (Integrated Security)
const sequelizeWindows = new Sequelize('MyApp', null, null, {
  host: 'sqlserver.example.com',
  dialect: 'mssql',
  dialectOptions: {
    authentication: {
      type: 'ntlm',
      options: {
        domain: 'MYDOMAIN',
        userName: 'appuser',
        password: 'SecureP@ss!',
      }
    },
    options: {
      encrypt: true,
      trustServerCertificate: true,
    }
  }
});

// Azure SQL Database
const sequelizeAzure = new Sequelize('MyApp', 'admin@myserver', 'SecureP@ss!', {
  host: 'myserver.database.windows.net',
  port: 1433,
  dialect: 'mssql',
  dialectOptions: {
    options: {
      encrypt: true,
      enableArithAbort: true,
    }
  }
});

// Test connection
async function testConnection() {
  try {
    await sequelize.authenticate();
    console.log('Connection established successfully.');
  } catch (error) {
    console.error('Unable to connect:', error);
  }
}
```

## Model Definition

```javascript
// models/Customer.js
const { DataTypes, Model } = require('sequelize');

class Customer extends Model {}

Customer.init({
  // INT IDENTITY(1,1) -- auto-generated primary key
  customerID: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true,  // Maps to IDENTITY(1,1)
    field: 'CustomerID',  // Actual column name in SQL Server
  },
  firstName: {
    type: DataTypes.STRING(100),  // NVARCHAR(100)
    allowNull: false,
    field: 'FirstName',
  },
  lastName: {
    type: DataTypes.STRING(100),  // NVARCHAR(100)
    allowNull: false,
    field: 'LastName',
  },
  email: {
    type: DataTypes.STRING(256),  // NVARCHAR(256)
    allowNull: false,
    unique: true,
    validate: {
      isEmail: true,
    },
    field: 'Email',
  },
  bio: {
    type: DataTypes.TEXT,  // NVARCHAR(MAX)
    allowNull: true,
    field: 'Bio',
  },
  balance: {
    type: DataTypes.DECIMAL(18, 2),  // DECIMAL(18,2)
    defaultValue: 0,
    field: 'Balance',
  },
  isActive: {
    type: DataTypes.BOOLEAN,  // BIT
    defaultValue: true,
    field: 'IsActive',
  },
  externalId: {
    type: DataTypes.UUID,  // UNIQUEIDENTIFIER
    defaultValue: DataTypes.UUIDV4,
    field: 'ExternalID',
  },
  metadata: {
    type: DataTypes.JSON,  // NVARCHAR(MAX) with JSON serialization
    defaultValue: {},
    field: 'Metadata',
  },
}, {
  sequelize,
  modelName: 'Customer',
  tableName: 'Customers',
  schema: 'dbo',
  timestamps: true,        // Adds createdAt/updatedAt
  createdAt: 'CreatedAt',  // Map to SQL Server column names
  updatedAt: 'UpdatedAt',
  underscored: false,
});

module.exports = Customer;
```

## DataTypes Mapping to SQL Server Types

```javascript
// Complete DataTypes reference for SQL Server (tedious dialect)

DataTypes.INTEGER        // INT
DataTypes.BIGINT         // BIGINT
DataTypes.SMALLINT       // SMALLINT
DataTypes.TINYINT        // TINYINT
DataTypes.FLOAT          // FLOAT
DataTypes.REAL           // REAL
DataTypes.DOUBLE         // FLOAT (same as FLOAT in SQL Server)
DataTypes.DECIMAL(18,2)  // DECIMAL(18,2)

DataTypes.STRING         // NVARCHAR(255) -- default length
DataTypes.STRING(100)    // NVARCHAR(100)
DataTypes.STRING(4000)   // NVARCHAR(4000)
DataTypes.TEXT           // NVARCHAR(MAX)
DataTypes.CHAR(10)       // NCHAR(10)

DataTypes.BOOLEAN        // BIT
DataTypes.DATE           // DATETIMEOFFSET
DataTypes.DATEONLY       // DATE
DataTypes.TIME           // TIME

DataTypes.UUID           // UNIQUEIDENTIFIER
DataTypes.JSON           // NVARCHAR(MAX) -- serialized JSON
DataTypes.BLOB           // VARBINARY(MAX)
DataTypes.BLOB('tiny')   // VARBINARY(256)
```

## Migrations

```bash
# Initialize sequelize-cli
npx sequelize-cli init

# Generate a migration
npx sequelize-cli migration:generate --name create-customers

# Run migrations
npx sequelize-cli db:migrate

# Undo last migration
npx sequelize-cli db:migrate:undo

# Undo all migrations
npx sequelize-cli db:migrate:undo:all
```

```javascript
// migrations/20250101000001-create-customers.js
'use strict';

module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable(
      { tableName: 'Customers', schema: 'dbo' },
      {
        CustomerID: {
          type: Sequelize.INTEGER,
          primaryKey: true,
          autoIncrement: true,
        },
        FirstName: {
          type: Sequelize.STRING(100),
          allowNull: false,
        },
        LastName: {
          type: Sequelize.STRING(100),
          allowNull: false,
        },
        Email: {
          type: Sequelize.STRING(256),
          allowNull: false,
          unique: true,
        },
        IsActive: {
          type: Sequelize.BOOLEAN,
          defaultValue: true,
        },
        CreatedAt: {
          type: Sequelize.DATE,
          defaultValue: Sequelize.fn('SYSUTCDATETIME'),
        },
        UpdatedAt: {
          type: Sequelize.DATE,
          defaultValue: Sequelize.fn('SYSUTCDATETIME'),
        },
      }
    );

    // Add index
    await queryInterface.addIndex(
      { tableName: 'Customers', schema: 'dbo' },
      ['Email'],
      { name: 'IX_Customers_Email', unique: true }
    );
  },

  async down(queryInterface) {
    await queryInterface.dropTable({ tableName: 'Customers', schema: 'dbo' });
  }
};
```

## Associations

```javascript
// models/index.js -- define associations after all models are loaded
const Customer = require('./Customer');
const Order = require('./Order');
const OrderItem = require('./OrderItem');
const Product = require('./Product');

// One-to-many: Customer has many Orders
Customer.hasMany(Order, { foreignKey: 'CustomerID', as: 'orders' });
Order.belongsTo(Customer, { foreignKey: 'CustomerID', as: 'customer' });

// One-to-many: Order has many OrderItems
Order.hasMany(OrderItem, { foreignKey: 'OrderID', as: 'items' });
OrderItem.belongsTo(Order, { foreignKey: 'OrderID', as: 'order' });

// Many-to-many: Products and Tags through ProductTags
Product.belongsToMany(Tag, { through: 'ProductTags', foreignKey: 'ProductID' });
Tag.belongsToMany(Product, { through: 'ProductTags', foreignKey: 'TagID' });

// Usage: eager loading
const customersWithOrders = await Customer.findAll({
  include: [{
    model: Order,
    as: 'orders',
    where: { status: 'Shipped' },
    required: false,  // LEFT JOIN
    include: [{
      model: OrderItem,
      as: 'items',
    }]
  }],
  order: [['CreatedAt', 'DESC']],
  limit: 50,
});
```

## Transactions

```javascript
// Managed transaction (auto-commit/rollback)
const result = await sequelize.transaction(async (t) => {
  const customer = await Customer.create({
    firstName: 'Jane',
    lastName: 'Doe',
    email: 'jane@example.com',
  }, { transaction: t });

  const order = await Order.create({
    CustomerID: customer.customerID,
    totalAmount: 79.98,
    status: 'Pending',
  }, { transaction: t });

  await OrderItem.bulkCreate([
    { OrderID: order.orderID, ProductID: 101, quantity: 2, unitPrice: 39.99 },
  ], { transaction: t });

  return order;
});
// Transaction auto-committed if no error thrown; auto-rolled back on error

// Unmanaged transaction (explicit control)
const t = await sequelize.transaction();
try {
  await Customer.update(
    { balance: sequelize.literal('Balance - 79.98') },
    { where: { customerID: 42 }, transaction: t }
  );
  await Order.update(
    { status: 'Paid' },
    { where: { orderID: 100 }, transaction: t }
  );
  await t.commit();
} catch (error) {
  await t.rollback();
  throw error;
}

// Transaction isolation levels
const { Transaction } = require('sequelize');
await sequelize.transaction({
  isolationLevel: Transaction.ISOLATION_LEVELS.READ_COMMITTED, // Default for SQL Server
  // Also: READ_UNCOMMITTED, REPEATABLE_READ, SERIALIZABLE, SNAPSHOT
}, async (t) => {
  // queries here
});
```

## Raw Queries

```javascript
// Raw SELECT with replacements (parameterized)
const [results] = await sequelize.query(
  `SELECT TOP(:limit) c.CustomerID, c.FirstName, c.LastName,
          COUNT(o.OrderID) AS OrderCount
   FROM dbo.Customers c
   LEFT JOIN dbo.Orders o ON c.CustomerID = o.CustomerID
   WHERE c.IsActive = 1
   GROUP BY c.CustomerID, c.FirstName, c.LastName
   ORDER BY OrderCount DESC`,
  {
    replacements: { limit: 10 },
    type: sequelize.QueryTypes.SELECT,
  }
);

// Call stored procedure
const [spResult] = await sequelize.query(
  'EXEC dbo.usp_SearchCustomers @SearchTerm = :term, @MaxResults = :max',
  {
    replacements: { term: 'smith', max: 50 },
    type: sequelize.QueryTypes.SELECT,
  }
);
```

## Best Practices

- Use `field` option in model definitions to map camelCase JS properties to PascalCase SQL Server columns
- Always specify `schema: 'dbo'` explicitly in model and migration table references
- Use managed transactions (`sequelize.transaction(async (t) => {...})`) for automatic rollback on error
- Set `encrypt: true` in dialectOptions for all connections, especially Azure SQL
- Use `DataTypes.UUID` for external-facing IDs -- it maps to native UNIQUEIDENTIFIER
- Configure connection pool (`pool.max`) based on your SQL Server max connections and app instance count
- Use `logging: false` in production to avoid performance overhead from console logging
- Run `sequelize.authenticate()` at startup to fail fast on connection issues
- Use `bulkCreate` with `ignoreDuplicates: true` for upsert-like operations

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Forgetting `dialect: 'mssql'` in config | Sequelize defaults to MySQL, connection fails | Always set `dialect: 'mssql'` explicitly |
| Using `DataTypes.NOW` for defaults | Generates client-side timestamp, not server-side | Use `Sequelize.fn('SYSUTCDATETIME')` for server-side default |
| Not specifying `schema: 'dbo'` | Sequelize may create tables in the wrong schema or fail to find them | Set schema in model definition and migrations |
| Using `SNAPSHOT` isolation without enabling it on database | Transaction fails with error | Run `ALTER DATABASE MyApp SET ALLOW_SNAPSHOT_ISOLATION ON` first |
| Not handling `requestTimeout` for long queries | Queries killed after 15s default | Set `dialectOptions.options.requestTimeout` to an appropriate value |

## SQL Server Version Notes

- **SQL Server 2016**: Full Sequelize support via tedious. JSON_VALUE available for raw queries. Temporal tables usable via raw SQL but not modeled in Sequelize. STRING_SPLIT useful in raw queries.
- **SQL Server 2019**: UTF-8 collation support. Batch mode on rowstore improves analytical queries executed via `sequelize.query()`. Scalar UDF inlining improves performance for function-heavy schemas.
- **SQL Server 2022**: Native JSON type (not yet mapped by tedious -- still uses NVARCHAR). GREATEST/LEAST simplify raw SQL. Parameter-sensitive plan optimization improves Sequelize's parameterized query performance. GENERATE_SERIES useful for seed data in raw SQL.

## Sources

- https://sequelize.org/docs/v6/other-topics/dialect-specific-things/#mssql
- https://sequelize.org/docs/v6/getting-started/
- https://github.com/tediousjs/tedious
- https://sequelize.org/docs/v6/other-topics/transactions/
- https://learn.microsoft.com/en-us/sql/connect/node-js/node-js-driver-for-sql-server
