### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Developing

## Get the Source Code

```bash
git clone https://github.com/Divergence/framework.git divergence/framework
git clone https://github.com/Divergence/cli.git divergence/cli
```

### Make sure you run
`composer install`

Unit tests use named labels from `config/db.php`.

#### |> THE TABLES WILL BE DELETED AFTER EACH TEST CYCLE!!!! <|

Representative labels:

```php
'tests-mysql' => [
    'host'     => 'localhost',
    'database' => 'test',
    'username' => 'root',
    'password' => '',
]

'tests-pgsql' => [
    'driver'   => 'pgsql',
    'host'     => '127.0.0.1',
    'port'     => 5432,
    'database' => 'test',
    'username' => 'divergence',
    'password' => 'abc123',
]

'tests-sqlite-memory' => [
    'path' => ':memory:',
]
```

## Unit Testing
The framework currently exposes Composer scripts for the common test paths:

```bash
composer test
composer test:mysql
composer test:sqlite
composer test:pgsql
composer test:coverage
```

What they do today:

- `composer test` runs MySQL, SQLite, and PostgreSQL suites in sequence
- each suite sets `DIVERGENCE_TEST_DB`
- coverage mode generates per-backend coverage blobs and merges them with `phpcov`

You can also still run PHPUnit directly. For example:

```bash
vendor/bin/phpunit
```

or:

```bash
XDEBUG_MODE=coverage DIVERGENCE_TEST_DB=tests-mysql vendor/bin/phpunit --coverage-php build/coverage/mysql.cov
```

Representative current output from `composer test`:

```text
Summoning Canaries
Starting Divergence Mock Environment for PHPUnit (tests-mysql)
PHPUnit 13.0.5 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.3
Configuration: /home/akujin/Divergence/framework/phpunit.xml

...............................................................  63 / 224 ( 28%)
............................................................... 126 / 224 ( 56%)
............................................................... 189 / 224 ( 84%)
...................................                             224 / 224 (100%)

Time: 00:00.483, Memory: 30.00 MB

OK (224 tests, 2767 assertions)

Cleaning up Divergence Mock Environment for PHPUnit (tests-mysql)
Summoning Canaries
Starting Divergence Mock Environment for PHPUnit (tests-sqlite-memory)
PHPUnit 13.0.5 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.3
Configuration: /home/akujin/Divergence/framework/phpunit.xml

...............................................................  63 / 224 ( 28%)
............................................................... 126 / 224 ( 56%)
............................................................... 189 / 224 ( 84%)
...................................                             224 / 224 (100%)

Time: 00:00.406, Memory: 30.00 MB

OK (224 tests, 2744 assertions)

Cleaning up Divergence Mock Environment for PHPUnit (tests-sqlite-memory)
Summoning Canaries
Starting Divergence Mock Environment for PHPUnit (tests-pgsql)
PHPUnit 13.0.5 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.5.3
Configuration: /home/akujin/Divergence/framework/phpunit.xml

...............................................................  63 / 224 ( 28%)
............................................................... 126 / 224 ( 56%)
............................................................... 189 / 224 ( 84%)
...................................                             224 / 224 (100%)

Time: 00:00.669, Memory: 30.00 MB

OK (224 tests, 2763 assertions)

Cleaning up Divergence Mock Environment for PHPUnit (tests-pgsql)
```

### How Mock Data is Made

The current test environment is bootstrapped through:

- `tests/bootstrap.php`
- `tests/Divergence/TestListener.php`
- `tests/MockSite/App.php`

`TestListener` sets up the mock environment per suite:

```php
public function startTestSuite(TestSuite $suite): void
{
    if ($connectionLabel = $this->getConnectionLabel($suite)) {
        $_SERVER['REQUEST_URI'] = '/';
        $suite->app = new App(__DIR__.'/../../');
        $suite->connectionLabel = $connectionLabel;
        Connections::setConnection($suite->connectionLabel);
        $suite->app->setUp();
    }
}
```

The `App` referenced above is `tests/MockSite/App`.

The `setUp` method in that class contains the database setup code.

The mock data is generated at random and from the tests themselves, then cleared on every PHPUnit run.

`tests/MockSite/App.php` currently:

- drops existing test tables for the active backend
- seeds tags
- creates canaries
- creates relational forum-style mock data
- removes generated media on teardown

The mock app under `tests/MockSite` is one of the clearest executable examples of intended framework usage.

## Style Guide
The formatter is wired through Composer:

```bash
composer fix-code
```

The framework currently ships `friendsofphp/php-cs-fixer` in `require-dev`.

If you contribute to the framework, keep test coverage in mind. The project clearly treats the test suite as behavioral documentation, especially around:

- ORM field mapping and persistence behavior
- versioning
- relations
- request handlers
- media streaming and range semantics
- DB helper behavior across multiple backends
