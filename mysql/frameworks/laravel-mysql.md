# Laravel MySQL — Database Configuration, Eloquent ORM, and MySQL-Specific Features

## Overview

Laravel provides deeply integrated MySQL support through its database layer, which includes the Eloquent ORM, a fluent query builder, a schema builder for migrations, and database-level transaction management. Laravel's MySQL driver uses PDO with the `pdo_mysql` extension, and the framework handles connection management, read/write splitting, and query logging out of the box.

Laravel's migration system uses a `Schema` facade with a `Blueprint` class to define table structures in PHP. While the migration DSL is database-agnostic, Laravel supports MySQL-specific column types (enum, set, mediumText, json), table options (engine, charset, collation), and query features (raw expressions, index hints) that let you take full advantage of MySQL's capabilities.

This skill covers database configuration, Eloquent model patterns, migrations with MySQL-specific types, the query builder, raw expressions, transactions, and read/write connection splitting.

## Key Concepts

- **Eloquent ORM**: Laravel's Active Record implementation where each model class maps to a MySQL table.
- **Query Builder**: A fluent interface for constructing SQL queries without raw SQL strings.
- **Schema Builder**: The `Schema` facade used in migrations to create and modify tables.
- **Blueprint**: The object passed to migration closures that defines columns, indexes, and constraints.
- **Facade**: Laravel's static proxy pattern (e.g., `DB::`, `Schema::`) that resolves to service container bindings.
- **Strict mode**: Laravel enables MySQL strict mode by default to prevent silent data truncation.

## Database Configuration

### config/database.php

```php
// config/database.php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'myapp'),
    'username' => env('DB_USERNAME', 'app_user'),
    'password' => env('DB_PASSWORD', ''),
    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => '',
    'prefix_indexes' => true,
    'strict' => true,
    'engine' => 'InnoDB',
    'options' => extension_loaded('pdo_mysql') ? array_filter([
        PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
        PDO::ATTR_PERSISTENT => false,
        PDO::ATTR_EMULATE_PREPARES => false,
        PDO::MYSQL_ATTR_FOUND_ROWS => true,
    ]) : [],
],
```

### .env file

```bash
DB_CONNECTION=mysql
DB_HOST=db-primary.internal
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=app_user
DB_PASSWORD=secret_password
```

### MySQL strict mode and modes array

```php
// Disable strict mode (not recommended, but sometimes needed for legacy data)
'strict' => false,

// Or fine-tune individual SQL modes
'strict' => true,
'modes' => [
    'STRICT_TRANS_TABLES',
    'NO_ZERO_IN_DATE',
    'NO_ZERO_DATE',
    'ERROR_FOR_DIVISION_BY_ZERO',
    'NO_ENGINE_SUBSTITUTION',
    // Removed ONLY_FULL_GROUP_BY for legacy queries:
    // 'ONLY_FULL_GROUP_BY',
],
```

### Read/write connection splitting

```php
'mysql' => [
    'read' => [
        'host' => [
            env('DB_READ_HOST_1', '192.168.1.1'),
            env('DB_READ_HOST_2', '192.168.1.2'),
        ],
    ],
    'write' => [
        'host' => [env('DB_WRITE_HOST', '192.168.1.10')],
    ],
    'sticky' => true,  // After write, reads go to write connection for remainder of request
    'driver' => 'mysql',
    'database' => env('DB_DATABASE', 'myapp'),
    'username' => env('DB_USERNAME', 'app_user'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => '',
    'strict' => true,
    'engine' => 'InnoDB',
],
```

## Migrations and Schema Builder

### Creating migrations

```bash
# Create a migration
php artisan make:migration create_products_table
php artisan make:migration add_sku_to_products_table --table=products

# Run migrations
php artisan migrate

# Rollback last batch
php artisan migrate:rollback

# Rollback and re-run all
php artisan migrate:fresh    # DROP ALL then migrate (dev only)
php artisan migrate:refresh  # Rollback all then migrate

# Show migration status
php artisan migrate:status
```

### Migration with MySQL-specific column types

```php
// database/migrations/2024_11_15_000001_create_products_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            // Primary key
            $table->id();                                // BIGINT UNSIGNED AUTO_INCREMENT

            // String types
            $table->string('name', 255);                 // VARCHAR(255)
            $table->string('sku', 50)->unique();         // VARCHAR(50) UNIQUE
            $table->char('country_code', 2);             // CHAR(2)

            // Text types
            $table->tinyText('short_desc')->nullable();  // TINYTEXT
            $table->text('summary')->nullable();         // TEXT
            $table->mediumText('description')->nullable(); // MEDIUMTEXT
            $table->longText('full_content')->nullable();  // LONGTEXT

            // Numeric types
            $table->decimal('price', 10, 2);             // DECIMAL(10,2)
            $table->unsignedInteger('quantity')->default(0); // INT UNSIGNED
            $table->unsignedBigInteger('view_count')->default(0); // BIGINT UNSIGNED
            $table->double('weight', 8, 3)->nullable();  // DOUBLE(8,3)
            $table->float('rating', 3, 2)->nullable();   // FLOAT(3,2)
            $table->tinyInteger('priority')->default(0); // TINYINT

            // MySQL ENUM and SET
            $table->enum('status', ['active', 'inactive', 'archived'])->default('active');
            $table->set('flags', ['featured', 'sale', 'new', 'clearance'])->nullable();

            // JSON
            $table->json('metadata')->nullable();        // JSON
            $table->json('tags')->nullable();             // JSON

            // Boolean
            $table->boolean('is_visible')->default(true); // TINYINT(1)

            // Date/Time
            $table->date('launch_date')->nullable();      // DATE
            $table->dateTime('published_at')->nullable(); // DATETIME
            $table->timestamp('indexed_at')->nullable();  // TIMESTAMP
            $table->year('model_year')->nullable();       // YEAR

            // Binary
            $table->binary('thumbnail')->nullable();      // BLOB

            // Foreign key
            $table->foreignId('category_id')
                  ->constrained('categories')
                  ->onDelete('cascade')
                  ->onUpdate('cascade');

            // Timestamps and soft delete
            $table->timestamps();                         // created_at, updated_at TIMESTAMP
            $table->softDeletes();                        // deleted_at TIMESTAMP NULL

            // Indexes
            $table->index('status');
            $table->index(['category_id', 'status'], 'idx_cat_status');
            $table->fullText(['name', 'description'], 'ft_name_desc');

            // Table options
            $table->engine('InnoDB');
            $table->charset('utf8mb4');
            $table->collation('utf8mb4_unicode_ci');
            $table->comment('Product catalog');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

### ALTER TABLE migration

```php
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('products', function (Blueprint $table) {
            $table->string('barcode', 100)->nullable()->after('sku');
            $table->unsignedInteger('reorder_level')->default(10)->after('quantity');
            $table->index('barcode');
        });
    }

    public function down(): void
    {
        Schema::table('products', function (Blueprint $table) {
            $table->dropIndex(['barcode']);
            $table->dropColumn(['barcode', 'reorder_level']);
        });
    }
};
```

## Eloquent ORM

### Model definition

```php
// app/Models/Product.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Product extends Model
{
    use SoftDeletes;

    protected $table = 'products';

    protected $fillable = [
        'name', 'sku', 'description', 'price',
        'quantity', 'status', 'category_id', 'metadata', 'tags',
    ];

    protected $casts = [
        'price' => 'decimal:2',
        'metadata' => 'array',     // JSON <-> PHP array
        'tags' => 'array',
        'is_visible' => 'boolean',
        'launch_date' => 'date',
        'published_at' => 'datetime',
    ];

    protected $attributes = [
        'status' => 'active',
        'quantity' => 0,
        'is_visible' => true,
    ];

    // Relationships
    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function orderItems(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function relatedProducts(): BelongsToMany
    {
        return $this->belongsToMany(Product::class, 'related_products', 'product_id', 'related_id');
    }

    // Scopes
    public function scopeActive($query)
    {
        return $query->where('status', 'active');
    }

    public function scopeInCategory($query, string $category)
    {
        return $query->whereHas('category', fn($q) => $q->where('slug', $category));
    }

    // Accessor
    protected function formattedPrice(): Attribute
    {
        return Attribute::make(
            get: fn() => '$' . number_format($this->price, 2),
        );
    }
}
```

### Eloquent queries

```php
// Basic CRUD
$product = Product::find(1);
$product = Product::findOrFail(1);
$products = Product::where('status', 'active')->get();

// Eager loading (prevent N+1)
$products = Product::with(['category', 'orderItems'])->get();
$products = Product::with(['category:id,name,slug'])->paginate(20);

// Scopes
$active = Product::active()->inCategory('electronics')->get();

// Mass operations
Product::where('status', 'archived')
    ->where('updated_at', '<', now()->subYear())
    ->delete();

// Chunk processing for large result sets
Product::where('status', 'active')->chunk(1000, function ($products) {
    foreach ($products as $product) {
        // Process each product
    }
});

// Upsert (INSERT ... ON DUPLICATE KEY UPDATE)
Product::upsert(
    [
        ['sku' => 'ABC-1', 'name' => 'Widget A', 'price' => 9.99],
        ['sku' => 'ABC-2', 'name' => 'Widget B', 'price' => 14.99],
    ],
    uniqueBy: ['sku'],
    update: ['name', 'price']
);
```

## Query Builder and Raw Expressions

```php
use Illuminate\Support\Facades\DB;

// Query builder
$orders = DB::table('orders')
    ->join('customers', 'orders.customer_id', '=', 'customers.id')
    ->select('customers.name', DB::raw('SUM(orders.total) as revenue'))
    ->where('orders.status', 'completed')
    ->groupBy('customers.name')
    ->havingRaw('SUM(orders.total) > ?', [1000])
    ->orderByDesc('revenue')
    ->limit(10)
    ->get();

// Raw expressions
$products = DB::table('products')
    ->select(DB::raw('DATE(created_at) as date, COUNT(*) as count'))
    ->whereRaw("JSON_CONTAINS(tags, ?)", [json_encode('featured')])
    ->groupByRaw('DATE(created_at)')
    ->get();

// Full raw query
$results = DB::select('SELECT * FROM products WHERE MATCH(name, description) AGAINST(? IN BOOLEAN MODE)', ['+"widget" -cheap']);

// Insert with raw default
DB::table('audit_log')->insert([
    'action' => 'login',
    'user_id' => $userId,
    'created_at' => DB::raw('CURRENT_TIMESTAMP'),
]);
```

## Database Transactions

```php
use Illuminate\Support\Facades\DB;

// Closure-based (auto commit/rollback)
DB::transaction(function () use ($fromId, $toId, $amount) {
    DB::table('accounts')->where('id', $fromId)->decrement('balance', $amount);
    DB::table('accounts')->where('id', $toId)->increment('balance', $amount);
}, attempts: 3); // Retry on deadlock up to 3 times

// Manual transaction control
DB::beginTransaction();
try {
    $order = Order::create([...]);
    OrderItem::insert([...]);
    DB::commit();
} catch (\Exception $e) {
    DB::rollBack();
    throw $e;
}
```

## Best Practices

- Always use `utf8mb4` charset and `utf8mb4_unicode_ci` collation in database config.
- Keep `strict => true` to prevent silent data truncation and invalid dates.
- Use `sticky => true` with read/write splitting to avoid stale reads after writes.
- Use `foreignId()->constrained()` instead of manual `unsignedBigInteger` + `foreign()` for cleaner migrations.
- Cast JSON columns to `array` in Eloquent models for automatic serialization/deserialization.
- Use `chunk()` or `lazy()` for processing large result sets to avoid memory exhaustion.
- Use `upsert()` for bulk insert-or-update operations instead of individual `updateOrCreate()` calls.
- Always use `with()` for eager loading to prevent N+1 query problems.
- Set `engine => 'InnoDB'` explicitly in database config to ensure transactional table creation.
- Use the `attempts` parameter on `DB::transaction()` for automatic deadlock retry.

## Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `migrate:fresh` in production | Drops ALL tables and data; catastrophic data loss | Only use in development; use `migrate` in production |
| Not using `with()` for eager loading | N+1 query problem: 1 query for parent + N queries for related models | Always eager-load relations: `Model::with(['relation'])->get()` |
| Using `string()` without length for indexed columns | MySQL may reject index on VARCHAR(255) with utf8mb4 (key too long on older configs) | Specify explicit length: `$table->string('email', 191)` or use `innodb_large_prefix=ON` |
| Setting `strict => false` globally to fix legacy queries | Silently truncates data, allows invalid dates, non-deterministic GROUP BY | Fix the queries instead; use `modes` array to selectively disable specific SQL modes |
| Using `DB::raw()` with unescaped user input | SQL injection vulnerability | Always use parameter binding: `DB::raw('FIELD = ?')` with bound values |

## MySQL Version Notes

- **5.7**: `json` column type requires 5.7.8+. `$table->id()` generates BIGINT UNSIGNED. Default `innodb_large_prefix=OFF` on some installations may cause "Specified key was too long" for `string(255)` with `utf8mb4`.
- **8.0**: Full JSON support including `whereJsonContains`, `whereJsonLength`. Default charset is `utf8mb4`. Window functions available via `DB::raw()`. `caching_sha2_password` default may require `PDO::MYSQL_ATTR_SSL_CA` configuration.
- **8.4/9.x**: `mysql_native_password` removed by default. Ensure PHP `pdo_mysql` extension supports `caching_sha2_password`. Laravel 11+ handles this transparently with modern PHP versions.

## Sources

- [Laravel Database Configuration](https://laravel.com/docs/11.x/database)
- [Laravel Migrations](https://laravel.com/docs/11.x/migrations)
- [Laravel Eloquent ORM](https://laravel.com/docs/11.x/eloquent)
- [Laravel Query Builder](https://laravel.com/docs/11.x/queries)
- [MySQL Strict Mode](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sql-mode-strict)
