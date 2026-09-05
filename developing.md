### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Developing

## Get the Source Code

```bash
git clone https://github.com/Divergence/framework.git divergence/framework
cd divergence/framework
composer install
```

Use PHP 8.4 or newer and install the PDO drivers for the backends you plan to test. The CLI lives in the separate `Divergence/cli` repository; you don't need it to run framework tests.

## Unit Testing

#### |> THE TABLES WILL BE DELETED AFTER EACH TEST CYCLE!!!! <|

Use disposable databases. The mock app also drops existing test tables during setup. Do not point a test label at application data.

Tests use labels returned by `config/db.php`. A local `config/db.dev.php` replaces that configuration when present. These are example entries, not credentials you should deploy:

```php
<?php
return [
    'tests-mysql' => [
        'host' => '127.0.0.1',
        'database' => 'divergence_test',
        'username' => 'divergence_test',
        'password' => 'replace-me',
    ],
    'tests-pgsql' => [
        'driver' => 'pgsql',
        'host' => '127.0.0.1',
        'port' => 5432,
        'database' => 'divergence_test',
        'username' => 'divergence_test',
        'password' => 'replace-me',
    ],
    'tests-sqlite-memory' => [
        'path' => ':memory:',
    ],
];
```

Run one backend or the whole matrix:

```bash
composer test:pgsql
composer test:sqlite
composer test:mysql
composer test
```

`composer test` runs MySQL, SQLite, then PostgreSQL. Each backend script sets `DIVERGENCE_TEST_DB` and invokes PHPUnit. To narrow a run, pass that label explicitly:

```bash
DIVERGENCE_TEST_DB=tests-pgsql vendor/bin/phpunit --filter CollectionMath
```

Don't assume an unqualified `vendor/bin/phpunit` selects the backend you wanted. Without the environment variable, the bootstrap doesn't initialize the selected database fixture environment.

### Coverage

Ordinary test runs don't generate coverage reports. Request coverage explicitly when you need it:

```bash
composer test:coverage
```

That runs all three backends in coverage mode, writes `build/coverage/mysql.cov`, `sqlite.cov`, and `pgsql.cov`, and merges them into `build/logs/clover.xml` with `phpcov`. It requires a working coverage driver; the scripts enable Xdebug's coverage mode. There are also `test:mysql:coverage`, `test:sqlite:coverage`, and `test:pgsql:coverage` scripts for individual backends.

### How Mock Data is Made

`tests/bootstrap.php` loads Composer, reads `DIVERGENCE_TEST_DB`, initializes `tests/MockSite/App.php`, selects the connection, and calls the mock app's `setUp()`. A shutdown handler calls `tearDown()`.

The mock app builds tags, canaries, and relational forum data. Individual tests also build their own fixtures. The accounting tests use a deterministic 200-receipt ledger, so expected totals and distributions are repeatable instead of depending on random prices.

The bootstrap starts a fresh `tests/test-errors.log` for each run and appends captured errors and exceptions. Test setup and teardown have filesystem and database effects even when coverage is disabled.

The mock app and tests are useful examples of field mapping, versioning, relationships, controllers, media, collection rollback, and Math behavior. Read the test for the feature you're changing and run it as you work.

## Static Analysis

```bash
composer analyze
```

This runs Phan with `--allow-polyfill-parser --no-progress-bar` and the project's `.phan/config.php`. The Phan GitHub Action runs on pushes and pull requests across all branches. Requiring the `Phan` check before merging into `develop` is a repository branch-protection setting, not something the workflow file enforces by itself.

Keep annotations accurate. Use real type hints where the runtime contract permits them. When a namespace is only needed in a DocBlock, write the full namespace there; don't add a `use` just for a comment. An annotation should explain the code, not hide a bug.

## Style Guide

Preview the formatter first:

```bash
vendor/bin/php-cs-fixer fix --dry-run --diff
```

To apply the configured rules:

```bash
composer fix-code
```

Use the repository's `.php-cs-fixer.dist.php`, including its custom copyright-header handling and annotation overrides. Don't substitute a generic ruleset. Review the resulting edits: formatting is not permission to change behavior, visibility, or annotations just to make a tool quiet.
