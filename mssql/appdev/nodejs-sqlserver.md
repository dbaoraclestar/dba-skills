# Node.js and SQL Server — tedious, mssql, and Modern Patterns

## Overview

Node.js connects to SQL Server through the tedious library, a pure-JavaScript TDS protocol implementation. While tedious can be used directly, most developers use the higher-level mssql package, which wraps tedious with a cleaner API including connection pooling, prepared statements, transactions, streaming, and TypeScript support.

The mssql package is the de facto standard for Node.js/SQL Server integration. It provides both callback and Promise/async-await interfaces, built-in connection pooling, tagged template literal queries for safe parameterization, and support for bulk table operations. It works with SQL Server 2012+, Azure SQL Database, and Azure SQL Managed Instance.

For applications using ORMs, Knex.js and TypeORM both support SQL Server through the mssql/tedious stack. The driver is cross-platform and requires no native ODBC installation, making it straightforward to deploy in containers and serverless environments.

## Key Concepts

- **tedious** — Low-level pure-JavaScript TDS driver. Handles protocol details, authentication, and data type marshaling. Most apps do not use it directly.
- **mssql** — High-level wrapper around tedious. Provides ConnectionPool, Request, Transaction, PreparedStatement, and streaming APIs.
- **ConnectionPool** — Manages a pool of reusable connections. One pool per application (or per database) is the recommended pattern.
- **Tagged template literals** — `sql.query\`SELECT * FROM Users WHERE id = ${id}\`` — safe parameterization using JavaScript template syntax.
- **Request** — Represents a SQL command execution. Supports input/output parameters, multiple result sets, and streaming.
- **Streaming** — Process large result sets row-by-row without loading everything into memory.

## Installation

```bash
npm install mssql
# mssql includes tedious as a dependency

# For TypeScript
npm install mssql @types/mssql
```

## Connection Setup

```javascript
const sql = require('mssql');

// Configuration object
const config = {
    user: 'myuser',
    password: 'mypassword',
    server: 'myserver',
    database: 'mydb',
    port: 1433,
    options: {
        encrypt: true,                // required for Azure SQL
        trustServerCertificate: false, // set true for self-signed certs
        enableArithAbort: true,
    },
    pool: {
        max: 20,
        min: 5,
        idleTimeoutMillis: 30000,
    },
    connectionTimeout: 15000,
    requestTimeout: 30000,
};

// Create and connect pool (do this once at app startup)
const pool = new sql.ConnectionPool(config);
const poolConnect = pool.connect();

// Handle pool errors
pool.on('error', (err) => {
    console.error('Pool error:', err);
});

// Windows Integrated Authentication
const configWinAuth = {
    server: 'myserver',
    database: 'mydb',
    options: {
        trustedConnection: true,
        encrypt: true,
    },
};

// Azure AD with access token
const configAzureAD = {
    server: 'myserver.database.windows.net',
    database: 'mydb',
    authentication: {
        type: 'azure-active-directory-access-token',
        options: {
            token: await getAzureToken(),
        },
    },
    options: {
        encrypt: true,
    },
};
```

## Basic Queries

```javascript
const sql = require('mssql');

async function getEmployees(department) {
    await poolConnect; // ensure pool is ready
    
    try {
        const result = await pool.request()
            .input('dept', sql.NVarChar(50), department)
            .query('SELECT EmployeeId, Name, Salary FROM Employees WHERE Department = @dept');
        
        console.log(`Found ${result.recordset.length} employees`);
        return result.recordset;
    } catch (err) {
        console.error('Query failed:', err.message);
        throw err;
    }
}

// Using tagged template literals (safe parameterization)
async function getEmployee(id) {
    await poolConnect;
    const result = await pool.query`
        SELECT EmployeeId, Name, Department, Salary 
        FROM Employees 
        WHERE EmployeeId = ${id}
    `;
    return result.recordset[0];
}

// INSERT with output
async function createEmployee(name, department, salary) {
    await poolConnect;
    const result = await pool.request()
        .input('name', sql.NVarChar(100), name)
        .input('dept', sql.NVarChar(50), department)
        .input('salary', sql.Decimal(12, 2), salary)
        .query(`
            INSERT INTO Employees (Name, Department, Salary) 
            OUTPUT INSERTED.EmployeeId
            VALUES (@name, @dept, @salary)
        `);
    return result.recordset[0].EmployeeId;
}

// UPDATE with rows affected
async function updateSalary(employeeId, newSalary) {
    await poolConnect;
    const result = await pool.request()
        .input('id', sql.Int, employeeId)
        .input('salary', sql.Decimal(12, 2), newSalary)
        .query('UPDATE Employees SET Salary = @salary WHERE EmployeeId = @id');
    return result.rowsAffected[0];
}
```

## Stored Procedures

```javascript
async function getOrdersByCustomer(customerId, startDate) {
    await poolConnect;
    
    const result = await pool.request()
        .input('CustomerId', sql.Int, customerId)
        .input('StartDate', sql.Date, startDate)
        .output('TotalOrders', sql.Int)
        .output('TotalAmount', sql.Decimal(18, 2))
        .execute('dbo.GetCustomerOrders');
    
    console.log('Orders:', result.recordset);
    console.log('Total orders:', result.output.TotalOrders);
    console.log('Total amount:', result.output.TotalAmount);
    console.log('Return value:', result.returnValue);
    
    return {
        orders: result.recordset,
        totalOrders: result.output.TotalOrders,
        totalAmount: result.output.TotalAmount,
    };
}
```

## Prepared Statements

```javascript
async function batchLookup(employeeIds) {
    await poolConnect;
    
    const ps = new sql.PreparedStatement(pool);
    ps.input('id', sql.Int);
    
    await ps.prepare('SELECT EmployeeId, Name, Salary FROM Employees WHERE EmployeeId = @id');
    
    const results = [];
    try {
        for (const id of employeeIds) {
            const result = await ps.execute({ id });
            if (result.recordset.length > 0) {
                results.push(result.recordset[0]);
            }
        }
    } finally {
        await ps.unprepare();
    }
    
    return results;
}
```

## Transactions

```javascript
async function transferFunds(fromAccount, toAccount, amount) {
    await poolConnect;
    const transaction = new sql.Transaction(pool);
    
    try {
        await transaction.begin();
        
        const request1 = new sql.Request(transaction);
        await request1
            .input('amount', sql.Decimal(12, 2), amount)
            .input('account', sql.Int, fromAccount)
            .query('UPDATE Accounts SET Balance = Balance - @amount WHERE AccountId = @account');
        
        const request2 = new sql.Request(transaction);
        await request2
            .input('amount', sql.Decimal(12, 2), amount)
            .input('account', sql.Int, toAccount)
            .query('UPDATE Accounts SET Balance = Balance + @amount WHERE AccountId = @account');
        
        await transaction.commit();
        return { success: true };
    } catch (err) {
        await transaction.rollback();
        throw err;
    }
}

// Transaction with isolation level
async function readConsistentData() {
    await poolConnect;
    const transaction = new sql.Transaction(pool);
    
    await transaction.begin(sql.ISOLATION_LEVEL.SNAPSHOT);
    try {
        const req = new sql.Request(transaction);
        const result = await req.query('SELECT * FROM Accounts');
        await transaction.commit();
        return result.recordset;
    } catch (err) {
        await transaction.rollback();
        throw err;
    }
}
```

## Streaming Large Result Sets

```javascript
async function streamLargeTable(tableName, processRow) {
    await poolConnect;
    
    const request = new sql.Request(pool);
    request.stream = true;
    
    let rowCount = 0;
    
    request.on('row', (row) => {
        rowCount++;
        processRow(row);
    });
    
    request.on('error', (err) => {
        console.error('Stream error:', err);
    });
    
    request.on('done', (result) => {
        console.log(`Streamed ${rowCount} rows, affected ${result.rowsAffected} rows`);
    });
    
    // Start the query — events fire as data arrives
    request.query(`SELECT * FROM ${sql.escape(tableName)}`);
    
    // Wait for completion with a promise wrapper
    return new Promise((resolve, reject) => {
        request.on('done', () => resolve(rowCount));
        request.on('error', reject);
    });
}
```

## Bulk Table Operations

```javascript
async function bulkInsertEmployees(employees) {
    await poolConnect;
    
    const table = new sql.Table('dbo.Employees');
    table.create = false; // table already exists
    
    // Define columns matching target table
    table.columns.add('Name', sql.NVarChar(100), { nullable: false });
    table.columns.add('Department', sql.NVarChar(50), { nullable: false });
    table.columns.add('Salary', sql.Decimal(12, 2), { nullable: false });
    table.columns.add('HireDate', sql.Date, { nullable: true });
    
    // Add rows
    for (const emp of employees) {
        table.rows.add(emp.name, emp.department, emp.salary, emp.hireDate || null);
    }
    
    const request = new sql.Request(pool);
    const result = await request.bulk(table);
    console.log(`Bulk inserted ${result.rowsAffected} rows`);
    return result.rowsAffected;
}
```

## TypeScript Usage

```typescript
import sql, { ConnectionPool, IResult, IRecordSet } from 'mssql';

interface Employee {
    EmployeeId: number;
    Name: string;
    Department: string;
    Salary: number;
}

const config: sql.config = {
    user: process.env.DB_USER!,
    password: process.env.DB_PASSWORD!,
    server: process.env.DB_SERVER!,
    database: process.env.DB_NAME!,
    options: {
        encrypt: true,
        trustServerCertificate: false,
    },
};

async function getEmployeeById(id: number): Promise<Employee | undefined> {
    const pool: ConnectionPool = await sql.connect(config);
    
    const result: IResult<Employee> = await pool.request()
        .input('id', sql.Int, id)
        .query<Employee>('SELECT * FROM Employees WHERE EmployeeId = @id');
    
    return result.recordset[0];
}

async function getEmployeesByDept(dept: string): Promise<Employee[]> {
    const pool = await sql.connect(config);
    const result = await pool.request()
        .input('dept', sql.NVarChar(50), dept)
        .query<Employee>('SELECT * FROM Employees WHERE Department = @dept');
    return result.recordset;
}
```

## Error Handling Patterns

```javascript
const sql = require('mssql');

async function safeQuery(queryFn) {
    try {
        return await queryFn();
    } catch (err) {
        if (err instanceof sql.ConnectionError) {
            console.error('Connection failed:', err.message);
            // Pool may need reconnecting
        } else if (err instanceof sql.RequestError) {
            console.error('Query error:', err.message);
            console.error('SQL error number:', err.number);
            console.error('SQL state:', err.state);
            
            switch (err.number) {
                case 2627: // unique constraint violation
                case 2601: // duplicate key
                    throw new DuplicateError(err.message);
                case 547:  // FK constraint violation
                    throw new ReferenceError(err.message);
                case 1205: // deadlock
                    throw new DeadlockError(err.message);
                default:
                    throw err;
            }
        } else if (err instanceof sql.TransactionError) {
            console.error('Transaction error:', err.message);
        }
        throw err;
    }
}

// Retry pattern for transient errors (deadlocks, timeouts)
async function withRetry(fn, maxRetries = 3, delayMs = 1000) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            return await fn();
        } catch (err) {
            const isTransient = err instanceof sql.RequestError &&
                [1205, -2, 40197, 40501, 40613].includes(err.number);
            
            if (isTransient && attempt < maxRetries) {
                const delay = delayMs * Math.pow(2, attempt - 1);
                console.warn(`Transient error (attempt ${attempt}), retrying in ${delay}ms`);
                await new Promise(r => setTimeout(r, delay));
            } else {
                throw err;
            }
        }
    }
}
```

## Express.js Integration Pattern

```javascript
const express = require('express');
const sql = require('mssql');

const app = express();
let pool;

// Initialize pool at startup
async function initDb() {
    pool = await sql.connect(config);
    console.log('Database pool initialized');
}

// Middleware to ensure pool is ready
app.use((req, res, next) => {
    if (!pool || !pool.connected) {
        return res.status(503).json({ error: 'Database unavailable' });
    }
    req.db = pool;
    next();
});

app.get('/api/employees/:id', async (req, res) => {
    try {
        const result = await req.db.request()
            .input('id', sql.Int, parseInt(req.params.id))
            .query('SELECT * FROM Employees WHERE EmployeeId = @id');
        
        if (result.recordset.length === 0) {
            return res.status(404).json({ error: 'Employee not found' });
        }
        res.json(result.recordset[0]);
    } catch (err) {
        console.error('Database error:', err);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
    if (pool) await pool.close();
    process.exit(0);
});

initDb().then(() => app.listen(3000));
```

## Best Practices

- Create one `ConnectionPool` per database at application startup and reuse it across all requests. Never create a pool per request.
- Use tagged template literals (`pool.query\`...\``) for simple queries — they handle parameterization automatically.
- Use `request.input()` with explicit SQL types for stored procedures and when you need precise type control.
- Enable streaming (`request.stream = true`) for queries returning more than ~10K rows to prevent memory exhaustion.
- Always call `pool.close()` on application shutdown (SIGTERM/SIGINT handlers) to release connections gracefully.
- Set `requestTimeout` appropriately for your workload — the default is 15 seconds in mssql.
- Use `sql.Decimal` or `sql.Numeric` for monetary values — never `sql.Float`.
- Handle deadlocks (error 1205) with retry logic and exponential backoff.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Creating a new pool per request | Rapid connection exhaustion, "pool full" errors | Create one pool at startup, reuse it globally |
| Using string interpolation in queries | SQL injection vulnerability | Use tagged templates or `.input()` parameters |
| Not handling pool `error` event | Unhandled promise rejection crashes app | Add `pool.on('error', handler)` |
| Forgetting `await` on async operations | Query runs but result is a pending Promise | Always `await` pool.request().query() |
| Not calling `transaction.rollback()` in catch | Locks held until connection timeout | Always rollback in catch/finally |
| Using `sql.connect()` without pool management | Creates implicit global pool, hard to manage | Use explicit `new sql.ConnectionPool(config)` |
| Not closing PreparedStatements | Connection leak in pool | Always call `ps.unprepare()` in finally |

## SQL Server Version Notes

- **SQL Server 2016** — tedious/mssql fully support all features including Always Encrypted, temporal tables, and JSON.
- **SQL Server 2019** — Full support. UTF-8 collations work transparently. Accelerated database recovery benefits connection pool resilience.
- **SQL Server 2022** — Supported by tedious 16+ and mssql 10+. TDS 8.0 strict encryption mode via `encrypt: 'strict'` in config.
- **Azure SQL** — Always set `encrypt: true`. Use Azure AD authentication types for managed identity. Connection resiliency is critical.
- **mssql 10+/tedious 16+** — `encrypt` defaults to `true` (breaking change from older versions where it defaulted to `false`).

## Sources

- https://www.npmjs.com/package/mssql
- https://tediousjs.github.io/tedious/
- https://github.com/tediousjs/node-mssql
- https://learn.microsoft.com/en-us/sql/connect/node-js/node-js-driver-for-sql-server
- https://github.com/tediousjs/tedious
