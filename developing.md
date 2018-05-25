### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Developing

## Get the Source Code

```bash
$ git clone https://github.com/Divergence/framework.git divergence/framework`
$ git clone https://github.com/Divergence/cli.git divergence/cli`
```

### Make sure you run
`composer install`

Unit tests use this config by default defined in config/db.php
#### |> THE TABLE WILL BE DELETED AFTER EACH TEST CYCLE!!!! <|
```php
'tests-mysql' => [
    'host'     =>   'localhost',
    'database' =>  'test',
    'username' =>  'travis',
    'password' =>  '',
]
```
---


## Unit Testing
Run `vendor/bin/phpunit --coverage-clover build/logs/clover.xml`
```
PHPUnit 7.1.5 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.1.16 with Xdebug 2.6.0
Configuration: /Users/admin/divergence/framework/phpunit.xml

Summoning Canaries
Starting Divergence Mock Environment for PHPUnit
...............................................................  63 / 183 ( 34%)
............................................................... 126 / 183 ( 68%)
.........................................................       183 / 183 (100%)
Cleaning up Divergence Mock Environment for PHPUnit


Time: 6.78 seconds, Memory: 16.00MB

OK (183 tests, 437 assertions)

Generating code coverage report in Clover XML format ... done
```

### How Mock Data is Made
`phpunit.xml` lets you define a "TestListener"
```html
<listeners>
    <listener class="Divergence\Tests\TestListener" file="./tests/Divergence/TestListener.php"></listener>
</listeners>
```

Which of course then runs `test/Divergence/TestListener.php`

The listener is a
```php
class TestListener implements PHPUnit_TestListener
```
Which then contains
```php
    public function startTestSuite(TestSuite $suite): void
    {
        //printf("TestSuite '%s' started.\n", $suite->getName());
        if ($suite->getName() == 'all') {
            MySQL::$defaultProductionLabel = 'tests-mysql';
            App::init(__DIR__.'/../../');
            App::setUp();
            fwrite(STDERR, 'Starting Divergence Mock Environment for PHPUnit'."\n");
        }
    }
```

The `App` referenced above is actually `tests/MockSite/App`.

The `setUp` method in the above class contains the database setup code.

The mock data is generated at random and from the tests themselves but cleared on every phpunit run.

---
## Style Guide
The styles are defined in `.php_cs.dist`

Just run `php-cs-fixer fix`

Currently they are
```php
<?php

$finder = PhpCsFixer\Finder::create()
    //->exclude('somedir')
    ->in(__DIR__)
;

return PhpCsFixer\Config::create()
    ->setRules([
        '@PSR1' => true,
        '@PSR2' => true,
        'array_syntax' => ['syntax' => 'short'],
        'trailing_comma_in_multiline_array' => true,
        'ternary_operator_spaces' => true,
        'trim_array_spaces' => true,
        'ordered_imports' => [
            'sortAlgorithm' => 'length'
        ],
        'ordered_class_elements' => true
    ])
    ->setFinder($finder)
;
```
Familiarize yourself with php-cs-fixer if you have not already.