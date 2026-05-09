# PHP MySQL Connectivity — PDO, mysqli, Prepared Statements, and Connection Strategies

## Overview

PHP has two primary APIs for MySQL access: PDO (PHP Data Objects) and mysqli. PDO is the recommended choice for new projects because it provides a database-agnostic API, supports named parameters, and has cleaner error handling. mysqli is MySQL-specific but offers a few features PDO lacks, such as asynchronous queries and multi-query execution.

Both APIs support prepared statements (essential for security), transactions, and SSL connections. PDO supports 12+ database drivers with the same API, making it portable. mysqli provides both an object-oriented and procedural interface, but only works with MySQL and MariaDB.

This skill covers PDO and mysqli connection setup, prepared statements, transactions, error modes, persistent connections, the distinction between emulated and native prepared statements, and connection pooling strategies.

## Key Concepts

- **PDO (PHP Data Objects)**: Database-agnostic abstraction layer. One API for MySQL, PostgreSQL, SQLite, and others. Preferred for new code.
- **mysqli**: MySQL-specific extension. Supports MySQL-only features (LOAD DATA LOCAL INFILE, multi-query, async). Object-oriented and procedural APIs.
- **Prepared statements**: SQL templates with bound parameters. Both PDO and mysqli support them. Critical for preventing SQL injection.
- **Emulated prepared statements**: PDO can prepare statements client-side (default). The driver escapes parameters and sends a single text query. Faster for simple queries but less safe.
- **Native prepared statements**: The server parses the SQL template separately from the parameters. Safer and required for certain data types.
- **Persistent connections**: Connections that survive the end of a PHP request and are reused by subsequent requests from the same process/thread.

## Connection Setup

### PDO (Recommended)

```php
<?php
$dsn  = 'mysql:host=127.0.0.1;port=3306;dbname=mydb;charset=utf8mb4';
$user = 'app_user';
$pass = 'secret';

$options = [
    PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,   // throw exceptions on error
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,         // return assoc arrays
    PDO::ATTR_EMULATE_PREPARES   => false,                    // use native prepared stmts
    PDO::ATTR_STRINGIFY_FETCHES  => false,                    // preserve native types
    PDO::MYSQL_ATTR_FOUND_ROWS   => true,                     // UPDATE returns matched rows
    PDO::ATTR_TIMEOUT            => 10,                       // connection timeout in seconds
];

try {
    $pdo = new PDO($dsn, $user, $pass, $options);
} catch (PDOException $e) {
    error_log('Database connection failed: ' . $e->getMessage());
    throw $e;
}
```

### mysqli

```php
<?php
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);  // throw exceptions

$mysqli = new mysqli('127.0.0.1', 'app_user', 'secret', 'mydb', 3306);
$mysqli->set_charset('utf8mb4');
$mysqli->options(MYSQLI_OPT_CONNECT_TIMEOUT, 10);

// Verify connection
if ($mysqli->connect_errno) {
    throw new RuntimeException('MySQL connect error: ' . $mysqli->connect_error);
}
```

## Prepared Statements

### PDO prepared statements

```php
// Positional placeholders
$stmt = $pdo->prepare('SELECT id, name, email FROM users WHERE status = ? AND role = ?');
$stmt->execute(['active', 'admin']);
$users = $stmt->fetchAll();

// Named placeholders (PDO-only feature)
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email AND status = :status');
$stmt->execute([
    ':email'  => $email,
    ':status' => 'active',
]);
$user = $stmt->fetch();

// INSERT with bound parameters
$stmt = $pdo->prepare('INSERT INTO users (name, email, status) VALUES (:name, :email, :status)');
$stmt->bindParam(':name',   $name,   PDO::PARAM_STR);
$stmt->bindParam(':email',  $email,  PDO::PARAM_STR);
$stmt->bindParam(':status', $status, PDO::PARAM_STR);

$name = 'Alice'; $email = 'alice@example.com'; $status = 'active';
$stmt->execute();
$newId = $pdo->lastInsertId();

// Re-execute with different values
$name = 'Bob'; $email = 'bob@example.com';
$stmt->execute();
```

### mysqli prepared statements

```php
$stmt = $mysqli->prepare('SELECT id, name FROM users WHERE status = ? AND role = ?');
$stmt->bind_param('ss', $status, $role);  // 'ss' = two strings

$status = 'active';
$role   = 'admin';
$stmt->execute();

$result = $stmt->get_result();
while ($row = $result->fetch_assoc()) {
    echo $row['name'] . "\n";
}
$stmt->close();

// Type characters for bind_param:
// s = string, i = integer, d = double, b = blob
```

## Transactions

### PDO transactions

```php
try {
    $pdo->beginTransaction();

    $pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
        ->execute([$amount, $fromId]);

    $pdo->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?')
        ->execute([$amount, $toId]);

    $pdo->commit();
} catch (Exception $e) {
    $pdo->rollBack();
    throw $e;
}
```

### mysqli transactions

```php
$mysqli->begin_transaction();

try {
    $stmt1 = $mysqli->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?');
    $stmt1->bind_param('di', $amount, $fromId);
    $stmt1->execute();

    $stmt2 = $mysqli->prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?');
    $stmt2->bind_param('di', $amount, $toId);
    $stmt2->execute();

    $mysqli->commit();
} catch (Exception $e) {
    $mysqli->rollback();
    throw $e;
}
```

## Error Modes (PDO)

```php
// ERRMODE_SILENT (default) -- errors stored in errorInfo()
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_SILENT);
$stmt = $pdo->query('SELECT * FROM nonexistent');
if ($stmt === false) {
    $err = $pdo->errorInfo();
    // $err[0] = SQLSTATE, $err[1] = driver error code, $err[2] = message
}

// ERRMODE_WARNING -- issues PHP warnings
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_WARNING);

// ERRMODE_EXCEPTION (recommended) -- throws PDOException
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
try {
    $pdo->exec('INSERT INTO users (email) VALUES ("dup@test.com")');
} catch (PDOException $e) {
    if ($e->getCode() === '23000') {  // integrity constraint violation
        echo "Duplicate entry\n";
    }
}
```

## Persistent Connections

```php
// PDO persistent connection
$pdo = new PDO($dsn, $user, $pass, [
    PDO::ATTR_PERSISTENT => true,
    PDO::ATTR_ERRMODE    => PDO::ERRMODE_EXCEPTION,
]);

// mysqli persistent connection (prefix host with "p:")
$mysqli = new mysqli('p:127.0.0.1', 'app_user', 'secret', 'mydb');
```

Persistent connections are reused across requests within the same PHP-FPM worker process. They avoid the TCP handshake and MySQL authentication overhead on each request.

Caveats: temporary tables, session variables, and uncommitted transactions from a prior request may leak into the next. Always reset state or avoid persistent connections if your app relies on clean session state.

## Emulated vs Native Prepared Statements

```php
// Emulated (PDO default) -- client-side escaping
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, true);
// Sends: SELECT * FROM users WHERE id = '42'

// Native -- server-side preparation
$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
// Sends: PREPARE + EXECUTE with binary parameter binding
```

When to use native (`false`):
- When you need strict type checking (integers, dates).
- When executing the same statement many times (server caches the plan).
- When you want MySQL to reject invalid SQL at prepare time.

When emulated (`true`) may be acceptable:
- Simple queries executed once.
- When using multiple statements in a single `execute()` call (not possible with native).
- Compatibility with older MySQL versions or edge-case drivers.

## Connection Pooling Strategies

PHP does not have built-in connection pooling like Java or Node.js. Strategies:

```php
// 1. Persistent connections (simplest)
// Reuses connections within PHP-FPM worker processes
$pdo = new PDO($dsn, $user, $pass, [PDO::ATTR_PERSISTENT => true]);

// 2. ProxySQL (external connection pooler)
// Application connects to ProxySQL on port 6033
// ProxySQL maintains a pool of real MySQL connections
$dsn = 'mysql:host=127.0.0.1;port=6033;dbname=mydb;charset=utf8mb4';

// 3. Swoole connection pool (for async PHP)
// Only if using Swoole coroutine-based PHP framework
use Swoole\Database\PDOConfig;
use Swoole\Database\PDOPool;

$pool = new PDOPool(
    (new PDOConfig)
        ->withHost('127.0.0.1')
        ->withDbname('mydb')
        ->withCharset('utf8mb4')
        ->withUsername('app_user')
        ->withPassword('secret'),
    20  // pool size
);
```

## Best Practices

- Use PDO for new projects. It is more portable, has cleaner error handling, and supports named placeholders.
- Always set `PDO::ATTR_ERRMODE` to `PDO::ERRMODE_EXCEPTION`.
- Disable emulated prepares (`PDO::ATTR_EMULATE_PREPARES => false`) for better security and type safety.
- Use `utf8mb4` charset, not `utf8`, in both the DSN and MySQL table definitions.
- Bind parameters with explicit types (`PDO::PARAM_INT`, `PDO::PARAM_STR`) when type precision matters.
- Use transactions for any multi-step write operation to ensure atomicity.
- If using persistent connections, never leave uncommitted transactions or temporary tables.
- Use `PDO::MYSQL_ATTR_FOUND_ROWS => true` if you need UPDATE to report matched rows rather than changed rows.
- Set `mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT)` at the top of any mysqli project.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Interpolating `$_GET`/`$_POST` into SQL | SQL injection | Use prepared statements with bound parameters |
| Leaving `PDO::ERRMODE_SILENT` (default) | Errors silently ignored, data corruption | Set `PDO::ERRMODE_EXCEPTION` |
| Using `utf8` instead of `utf8mb4` in DSN | Emojis and some CJK characters truncated or rejected | Use `charset=utf8mb4` |
| Not calling `$stmt->close()` in mysqli loops | Exhausts server prepared statement limit | Close statements after use |
| Persistent connections with dirty state | Transaction or temp table leak between requests | Reset session state or avoid persistent connections |
| Emulated prepares with LIMIT parameters | `LIMIT '10'` (string) causes syntax error | Disable emulated prepares or cast to int |
| Using `PDO::query()` with user input | No parameterization at all | Use `PDO::prepare()` + `execute()` |

## MySQL Version Notes

- **5.7**: Both PDO and mysqli fully supported. Default auth: `mysql_native_password`. No issues with standard PHP 7.x/8.x drivers.
- **8.0**: Default auth is `caching_sha2_password`. PHP 7.4+ / 8.0+ PDO and mysqli handle it. Older PHP versions may need `default_authentication_plugin=mysql_native_password` server-side.
- **8.4 / 9.x**: `mysql_native_password` removed. PHP 8.1+ with mysqlnd (native driver, default) handles `caching_sha2_password`. Update PHP if running pre-8.1.

## Sources

- [PHP PDO Manual](https://www.php.net/manual/en/book.pdo.php)
- [PHP mysqli Manual](https://www.php.net/manual/en/book.mysqli.php)
- [PHP PDO MySQL Driver](https://www.php.net/manual/en/ref.pdo-mysql.php)
- [MySQL caching_sha2_password and PHP](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)
