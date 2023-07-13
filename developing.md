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
    'username' =>  'root',
    'password' =>  '',
]
```
---


## Unit Testing
Run `vendor/bin/phpunit --coverage-clover build/logs/clover.xml`
```
PHPUnit 9.6.9 by Sebastian Bergmann and contributors.

Runtime:       PHP 8.1.21
Configuration: /home/akujin/Divergence/framework/phpunit.xml

Summoning Canaries
Starting Divergence Mock Environment for PHPUnit
...............................................................  63 / 201 ( 31%)
............................................................... 126 / 201 ( 62%)
............................................................... 189 / 201 ( 94%)
............                                                    201 / 201 (100%)
Cleaning up Divergence Mock Environment for PHPUnit


Time: 00:00.659, Memory: 16.00 MB

OK (201 tests, 474 assertions)

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
        if ($suite->getName() == 'all') {
            $_SERVER['REQUEST_URI'] = '/';
            $suite->app = new App(__DIR__.'/../../');
            MySQL::setConnection('tests-mysql');
            $suite->app->setUp();
            fwrite(STDERR, 'Starting Divergence Mock Environment for PHPUnit'."\n");
        }
    }
```

The `App` referenced above is actually `tests/MockSite/App`.

The `setUp` method in the above class contains the database setup code.

The mock data is generated at random and from the tests themselves but cleared on every phpunit run.

---
## Style Guide
The styles are defined in `.php_cs.dist.php`

Just run `php-cs-fixer fix`

Currently they are
```php
<?php

$finder = (new PhpCsFixer\Finder())
    //->exclude('somedir')
    ->in(__DIR__)
;

return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR1' => true,
        '@PSR2' => true,
        'array_syntax' => ['syntax' => 'short'],
        'trailing_comma_in_multiline_array' => true,
        'no_trailing_comma_in_singleline_array' => true,
        'ternary_operator_spaces' => true,
        'trim_array_spaces' => true,
        'ordered_imports' => [
            'sortAlgorithm' => 'length'
        ],
        'ordered_class_elements' => true,
        'indentation_type' => true,
        'header_comment' => [
            'header' => $header,
            'comment_type' => 'PHPDoc',
            'location' => 'after_open',
            'separate' => 'none',
        ]
    ])
    ->setFinder($finder)   
;
```
Familiarize yourself with php-cs-fixer if you have not already.