### [⤺ Back to Table of Contents](README.md#divergence-framework-documentation)

# Database

Divergence supports MySQL, PostgreSQL, and SQLite through PDO. Your models use the active connection. You don't need a different model class for each backend.

## Connection Configurations

`config/db.php` returns an array of named connections. The names are labels, not driver names. A config containing `path` selects SQLite; `driver => 'pgsql'` selects PostgreSQL; otherwise the resolver uses MySQL.

Here is a complete example. Replace the server credentials with your own:

```php
<?php
return [
    'mysql' => [
        'host' => '127.0.0.1',
        'port' => 3306,
        'database' => 'myapp',
        'username' => 'myapp',
        'password' => 'replace-me',
    ],
    'pgsql' => [
        'driver' => 'pgsql',
        'host' => '127.0.0.1',
        'port' => 5432,
        'database' => 'myapp',
        'username' => 'myapp',
        'password' => 'replace-me',
    ],
    'sqlite' => [
        'path' => dirname(__DIR__).'/var/sqlite/app.sqlite',
        'foreign_keys' => true,
        'busy_timeout' => 5000,
    ],
];
```

MySQL also accepts a `socket` instead of a TCP host. PostgreSQL accepts `sslmode`. SQLite's `busy_timeout` is in milliseconds. Create the SQLite parent directory and make it writable by the application user. `:memory:` creates a database that lives only for that connection.

The shipped `config/db.php` checks for `config/db.dev.php` and returns it if present. This replaces the whole config regardless of the application's environment; it is not a per-key merge.

## Choosing the Connection

After bootstrapping the App, select your label before the first model query:

```php
use Divergence\IO\Database\Connections;

Connections::setConnection('pgsql');
$connection = Connections::getConnection(); // PDO
```

Without an explicit selection, the default labels are `mysql` in production and `dev-mysql` in development. You can change `Connections::$defaultProductionLabel` and `Connections::$defaultDevLabel` in your bootstrap if you want different defaults.

Connections and model metadata are cached in the PHP process. Select the intended backend before querying or hydrating records. Don't treat changing a label halfway through a unit of work as moving already-loaded objects to another database.

## Queries

For model data, start with model getters:

```php
$tag = \yourApp\Models\Tag::getByField('Slug', $slug);
$tags = \yourApp\Models\Tag::getAllByWhere([], [
    'order' => ['Tag' => 'ASC'],
    'limit' => 25,
    'offset' => 0,
]);
```

For raw SQL, resolve the active storage class:

```php
$storage = Connections::getConnectionType();
$count = $storage::oneValue('SELECT COUNT(*) FROM tags');
```

| Helper | Result |
| --- | --- |
| `query($sql, $parameters = [])` | PDO statement |
| `nonQuery($sql, $parameters = [])` | Runs a statement without returning rows |
| `oneRecord($sql, $parameters = [])` | One associative row or `false` |
| `allRecords($sql, $parameters = [])` | Array of associative rows |
| `oneValue($sql, $parameters = [])` | First column of the first row, or `false` |
| `allValues($field, $sql, $parameters = [])` | Values from one result field |
| `table($keyField, $sql, $parameters = [])` | Rows keyed by a result field |
| `affectedRows()` | Count from the last relevant statement |
| `insertID()` | Last generated identifier |

The storage helpers' `$parameters` are `sprintf`/`vsprintf` substitutions. They are **not PDO bind parameters**. Passing an untrusted string as `%s` does not escape it. `quote()` returns a complete quoted SQL string literal; `escape()` returns its escaped contents without the surrounding quotes. Neither is for quoting table or column names.

When you want bound parameters, use PDO directly:

```php
$statement = Connections::getConnection()->prepare(
    'SELECT * FROM tags WHERE "Slug" = :slug'
);
$statement->execute(['slug' => $slug]);
$rows = $statement->fetchAll(\PDO::FETCH_ASSOC);
```

That identifier quoting is PostgreSQL syntax. Direct PDO queries go straight to the selected driver; the framework does not rewrite them. Use the syntax for your backend and allowlist any dynamic identifiers separately from values.

## Transactions

```php
$connection = Connections::getConnection();
$connection->beginTransaction();

try {
    $first->save();
    $second->save();
    $connection->commit();
} catch (\Throwable $exception) {
    $connection->rollBack();
    throw $exception;
}
```

The database rollback does not restore arbitrary PHP objects. If you're saving an ORM collection and want its models and indexes restored too, use [`RecordCollection::saveWithTransaction()`](collections.md#orm-collections). Neither approach rolls back file writes or other external work performed by your hooks.

## Schema Creation and Portability

Models can auto-create missing tables. This is convenient when starting a project, but it is not a migration system. Changing a field definition does not migrate an existing table. Set `public static $autoCreateTables = false;` when your deployment manages schema changes explicitly.

The PostgreSQL and SQLite storage classes translate parts of the MySQL-style SQL used by the framework. That compatibility layer is not a general SQL parser. Test custom joins, expressions, schema changes, and pagination against each backend you intend to support.

`foundRows()` is backend-specific. MySQL uses `FOUND_ROWS()`; PostgreSQL and SQLite count a rewritten version of the preceding SELECT. Call it immediately after the paginated query, before another query changes the tracked statement. An explicit count query is often clearer for custom SQL.

SQL `decimal` storage does not guarantee exact decimal math in PHP. See [Accounting With Integer Cents](math.md#accounting-with-integer-cents).
