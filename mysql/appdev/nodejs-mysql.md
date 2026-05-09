# Node.js MySQL Connectivity — mysql2, Promises, Streaming, and Knex.js

## Overview

The `mysql2` package is the recommended MySQL driver for Node.js, superseding the older `mysql` package. It is largely API-compatible with `mysql` but adds critical features: prepared statements via the binary protocol, promise-based APIs, built-in connection pooling, streaming support for large result sets, and TypeScript type definitions.

`mysql2` communicates with MySQL using the native binary protocol for prepared statements, which is faster and safer than the text protocol used by the original `mysql` package. It supports `caching_sha2_password` (MySQL 8.0+ default), SSL/TLS, and compressed protocol.

This skill covers connection setup, pooling, prepared statements, streaming, the promise API, TypeScript integration, error handling, and integration with the Knex.js query builder.

## Key Concepts

- **Connection pool**: A managed set of reusable connections. The pool automatically creates, recycles, and destroys connections as needed. Always use a pool in production.
- **Prepared statements**: SQL parsed once, executed many times with different parameters. `mysql2` sends these over the binary protocol, which is both faster and injection-safe.
- **Streaming**: Processing large result sets row-by-row without loading them all into memory. Uses Node.js readable streams.
- **Promise wrapper**: `mysql2/promise` provides `async/await`-compatible versions of all methods. Use this unless you specifically need callbacks.
- **Named placeholders**: `mysql2` supports `:name` style placeholders in addition to `?`.

## Installation

```bash
npm install mysql2

# With Knex.js (query builder)
npm install mysql2 knex

# TypeScript types are bundled with mysql2
```

## Connection Setup

### Single connection (for scripts/CLIs)

```javascript
const mysql = require('mysql2/promise');

async function main() {
  const conn = await mysql.createConnection({
    host: '127.0.0.1',
    port: 3306,
    user: 'app_user',
    password: 'secret',
    database: 'mydb',
    charset: 'utf8mb4',
    connectTimeout: 10000,
    timezone: '+00:00',
  });

  const [rows] = await conn.execute('SELECT VERSION() AS version');
  console.log('MySQL version:', rows[0].version);

  await conn.end();
}

main().catch(console.error);
```

### Connection pool (for servers)

```javascript
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: '127.0.0.1',
  port: 3306,
  user: 'app_user',
  password: 'secret',
  database: 'mydb',
  charset: 'utf8mb4',
  waitForConnections: true,
  connectionLimit: 20,
  maxIdle: 10,
  idleTimeout: 60000,
  queueLimit: 0,          // unlimited queue
  enableKeepAlive: true,
  keepAliveInitialDelay: 30000,
});

// Pool usage in an Express route
app.get('/users/:id', async (req, res) => {
  const [rows] = await pool.execute(
    'SELECT id, name, email FROM users WHERE id = ?',
    [req.params.id]
  );
  res.json(rows[0] || null);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await pool.end();
  process.exit(0);
});
```

## Prepared Statements

```javascript
// execute() uses prepared statements (binary protocol)
const [rows] = await pool.execute(
  'SELECT * FROM orders WHERE customer_id = ? AND status = ?',
  [customerId, 'shipped']
);

// query() uses the text protocol (no preparation)
// Use for DDL, SHOW, SET, or dynamic SQL where binding is not possible
const [rows] = await pool.query('SHOW PROCESSLIST');

// Named placeholders (opt-in)
const mysql = require('mysql2');
const pool = mysql.createPool({ /* config */ , namedPlaceholders: true });
const promisePool = pool.promise();

const [rows] = await promisePool.execute(
  'SELECT * FROM users WHERE email = :email AND status = :status',
  { email: 'alice@example.com', status: 'active' }
);
```

## Streaming Large Results

```javascript
// Callback-style streaming (uses the non-promise pool)
const mysql = require('mysql2');
const pool = mysql.createPool({ /* config */ });

const stream = pool.query('SELECT * FROM huge_table WHERE created_at > ?', ['2025-01-01'])
  .stream();

stream.on('data', (row) => {
  // process one row at a time
  processRow(row);
});

stream.on('end', () => {
  console.log('Stream complete');
});

stream.on('error', (err) => {
  console.error('Stream error:', err);
});

// With backpressure handling using pipeline (Node 16+)
const { pipeline } = require('stream/promises');
const { Transform } = require('stream');

const queryStream = pool.query('SELECT * FROM huge_table').stream();

const transformer = new Transform({
  objectMode: true,
  transform(row, encoding, callback) {
    const result = processRow(row);
    callback(null, JSON.stringify(result) + '\n');
  }
});

await pipeline(queryStream, transformer, process.stdout);
```

## Promise-Based API

```javascript
const mysql = require('mysql2/promise');

async function transferFunds(pool, fromId, toId, amount) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();

    await conn.execute(
      'UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?',
      [amount, fromId, amount]
    );

    await conn.execute(
      'UPDATE accounts SET balance = balance + ? WHERE id = ?',
      [amount, toId]
    );

    await conn.commit();
    return { success: true };

  } catch (err) {
    await conn.rollback();
    throw err;
  } finally {
    conn.release(); // return to pool
  }
}
```

## TypeScript Support

```typescript
import mysql, { RowDataPacket, ResultSetHeader, FieldPacket } from 'mysql2/promise';

// Define row interface
interface User extends RowDataPacket {
  id: number;
  name: string;
  email: string;
  created_at: Date;
}

const pool = mysql.createPool({
  host: '127.0.0.1',
  user: 'app_user',
  password: 'secret',
  database: 'mydb',
});

// Typed query result
const [users] = await pool.execute<User[]>(
  'SELECT id, name, email, created_at FROM users WHERE status = ?',
  ['active']
);

users.forEach(user => {
  console.log(user.name); // TypeScript knows this is a string
});

// Typed insert result
const [result] = await pool.execute<ResultSetHeader>(
  'INSERT INTO users (name, email) VALUES (?, ?)',
  ['Alice', 'alice@example.com']
);

console.log('Inserted id:', result.insertId);
console.log('Affected rows:', result.affectedRows);
```

## Error Handling

```javascript
const mysql = require('mysql2/promise');

try {
  const [rows] = await pool.execute('INSERT INTO users (email) VALUES (?)', ['dup@test.com']);
} catch (err) {
  if (err.code === 'ER_DUP_ENTRY') {
    console.log('Duplicate entry, ignoring');
  } else if (err.code === 'ER_ACCESS_DENIED_ERROR') {
    console.error('Authentication failed');
  } else if (err.code === 'ECONNREFUSED') {
    console.error('Cannot connect to MySQL');
  } else if (err.code === 'PROTOCOL_CONNECTION_LOST') {
    console.error('Connection lost, will auto-reconnect via pool');
  } else if (err.code === 'ER_LOCK_WAIT_TIMEOUT') {
    console.error('Lock wait timeout, retry the transaction');
  } else {
    console.error(`MySQL error [${err.code}] ${err.errno}: ${err.message}`);
  }
}

// Pool-level error event
pool.on('connection', (conn) => {
  console.log(`New connection ${conn.threadId}`);
});

// For callback-style pools
const cbPool = require('mysql2').createPool({ /* config */ });
cbPool.on('enqueue', () => {
  console.warn('Waiting for available connection slot -- pool exhausted');
});
```

## Connection Configuration Options

```javascript
const pool = mysql.createPool({
  // Connection
  host: '127.0.0.1',
  port: 3306,
  user: 'app_user',
  password: 'secret',
  database: 'mydb',
  charset: 'utf8mb4',

  // SSL
  ssl: {
    ca: fs.readFileSync('/path/to/ca.pem'),
    rejectUnauthorized: true
  },

  // Pool
  connectionLimit: 20,
  maxIdle: 10,
  idleTimeout: 60000,
  waitForConnections: true,
  queueLimit: 0,

  // Timeouts
  connectTimeout: 10000,

  // Protocol
  namedPlaceholders: false,   // enable :name syntax
  multipleStatements: false,  // keep false for security
  dateStrings: false,         // true = return dates as strings
  decimalNumbers: false,      // true = return DECIMAL as Number (lossy)
  typeCast: true,             // auto-convert MySQL types to JS types
  supportBigNumbers: true,    // handle BIGINT properly
  bigNumberStrings: false,    // return BIGINT as string if too large for Number
});
```

## Integration with Knex.js

```javascript
const knex = require('knex')({
  client: 'mysql2',
  connection: {
    host: '127.0.0.1',
    port: 3306,
    user: 'app_user',
    password: 'secret',
    database: 'mydb',
    charset: 'utf8mb4',
  },
  pool: {
    min: 5,
    max: 20,
    acquireTimeoutMillis: 10000,
    idleTimeoutMillis: 60000,
  },
});

// Select with builder
const users = await knex('users')
  .where('status', 'active')
  .andWhere('created_at', '>', '2025-01-01')
  .orderBy('name')
  .limit(50);

// Insert
const [id] = await knex('users').insert({
  name: 'Alice',
  email: 'alice@example.com',
  status: 'active'
});

// Transaction
await knex.transaction(async (trx) => {
  await trx('accounts').where('id', fromId).decrement('balance', amount);
  await trx('accounts').where('id', toId).increment('balance', amount);
});

// Raw SQL when needed
const results = await knex.raw(
  'SELECT * FROM users WHERE email = ? AND status = ?',
  ['alice@example.com', 'active']
);
```

## Best Practices

- Always use `mysql2/promise` and `async/await` over the callback API.
- Use `pool.execute()` (prepared statements) for parameterized queries, and `pool.query()` only for DDL or SHOW commands.
- Keep `multipleStatements: false` (the default) to block multi-statement SQL injection.
- Use connection pools in all server applications. Never create individual connections per request.
- Release connections with `conn.release()` in a `finally` block when using `pool.getConnection()`.
- Set `supportBigNumbers: true` if your schema uses BIGINT columns.
- Stream results for queries expected to return more than a few thousand rows.
- Set `enableKeepAlive: true` on pools to prevent connections from being silently dropped by network infrastructure.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `query()` for parameterized SQL | Text protocol, no injection protection from prepared statements | Use `execute()` for parameterized queries |
| Forgetting `conn.release()` after `getConnection()` | Pool exhaustion, app hangs waiting for connections | Always call `release()` in `finally` |
| Setting `multipleStatements: true` | Opens door to SQL injection even with parameters | Keep default `false` |
| Not handling `PROTOCOL_CONNECTION_LOST` | App crashes on transient network errors | Use pool (auto-reconnects) and catch the error |
| Using `Number` for BIGINT values > 2^53 | Silent precision loss | Set `supportBigNumbers: true`, `bigNumberStrings: true` |
| Creating a new pool per request | Thousands of connections, MySQL refuses new ones | Create one pool at app startup, reuse it |

## MySQL Version Notes

- **5.7**: `mysql2` fully supported. Default auth plugin is `mysql_native_password`.
- **8.0**: Default auth switched to `caching_sha2_password`. `mysql2` 2.x+ handles it. Ensure `mysql2` is updated if migrating from 5.7.
- **8.4 / 9.x**: `mysql_native_password` removed. Requires `mysql2` 3.x+ which defaults to `caching_sha2_password`.

## Sources

- [mysql2 GitHub and Documentation](https://github.com/sidorares/node-mysql2)
- [Knex.js Documentation](https://knexjs.org/)
- [Node.js Streams API](https://nodejs.org/api/stream.html)
- [MySQL Connector/Node.js (X DevAPI)](https://dev.mysql.com/doc/dev/connector-nodejs/latest/)
