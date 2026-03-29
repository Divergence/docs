### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Project Structure
In this section I will discuss the loading order in detail and explain some of the things the framework does for you.

## Loading Order

A typical Divergence project should by default have a public folder with an `.htaccess` file defined as such for typical apache2 configurations.

```apache
RewriteEngine On

RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*) index.php [L,QSA]
```

This will route all paths to the one other file in a public directory by default: `index.php`.

`index.php` requires these files in order:

* `bootstrap/autoload.php`
    ```php
    define('DIVERGENCE_START', microtime(true));
    require(__DIR__.'/../vendor/autoload.php');
    ```
    Store start time for timing and run Composer's autoload.

* `bootstrap/app.php`
    ```php
    use project\App as App;
    $app = new App(realpath(__DIR__.'/../'));
    ```
    Typically you would use your namespace here instead of `project`, with your own App class that extends `Divergence\App`.

* `bootstrap/router.php`
    ```php
    $app->handleRequest();
    ```

## Folder Structure
| Folder | Description |
| --- | --- |
| `bootstrap` | Bootstrap files described above |
| `config` | Config files |
| `public` | Static files and web root |
| `src` | Default location for your project's PHP code |
| `tests` | Unit tests |
| `var` | Runtime files such as SQLite databases |
| `vendor` | Composer dependencies |
| `views` | Templates |

# The App Class
It is highly recommended that you extend the App class or redefine it treating the existing methods as if it were an interface.

`Divergence\App` is responsible for:

- storing `ApplicationPath`
- exposing the current instance at `App::$App`
- loading config files
- initializing `Routing\Path`
- registering error handling
- dispatching the root request handler

## Configs
You can load configs at any time by calling `$config = App::$App->config('file');` to load `config/file.php`. The return will be whatever the file itself returns.

```php
<?php
return [ 'some', 'example', 'config' => 'data' ];
```

Make sure the config exists though because otherwise it will throw an exception.

```php
public function config($Label)
{
    $Config = $this->ApplicationPath . '/config/' . $Label . '.php';
    if (!file_exists($Config)) {
        throw new \Exception($Config . ' not found in '.static::class.'::config()');
    }
    return require $Config;
}
```

Feel free to make your own configs as you see fit.

## Development & Production
To switch to dev mode simply create an empty file in your project directory named `.dev`.

`touch .dev`

The database layer uses these default config labels:

```php
public static $defaultProductionLabel = 'mysql';
public static $defaultDevLabel = 'dev-mysql';
```

The framework now also ships PostgreSQL and SQLite labels in `config/db.php`, and the concrete storage backend is inferred from the chosen label.

#### The existence of the `.dev` file effectively switches your default database connection label.
#### Tip: You can override these default labels for things like unit tests and continuous integration before any database connections are established.

This behavior is part of the `Divergence\App` and `Divergence\IO\Database\Connections` runtime.

#### This is the first method that runs after Composer.
```php
public function init($Path)
{
    $this->ApplicationPath = $Path;

    if (php_sapi_name()!=='cli' || defined('PHPUNIT_TESTSUITE')) {
        $this->Path = new Path($_SERVER['REQUEST_URI']);
    }

    $this->Config = $this->config('app');

    $this->registerErrorHandler();
}
```

The contents of `config/app.php` is where the production or development mode is really set.

```php
use Divergence\App as App;

return [
    'debug' => file_exists(App::$App->ApplicationPath . '/.debug'),
    'environment' => (file_exists(App::$App->ApplicationPath . '/.dev') ? 'dev' : 'production'),
];
```

A simple way to switch environments is provided for you, but you're welcome to put in your own logic right there.

## Database Labels

`Divergence\IO\Database\Connections` resolves the backend type from the active label:

- labels with a `path` use SQLite
- labels with `driver => 'pgsql'` use PostgreSQL
- everything else defaults to MySQL

You can also explicitly force a connection label before the first query:

```php
\Divergence\IO\Database\Connections::setConnection('tests-pgsql');
```

## Error Handling
Development mode turns on Whoops error handling.

Production mode sets `error_reporting(0)` so that a user does not ever see raw PHP errors.

It is recommended that you bind your own logging and production exception reporting around this.

```php
public function registerErrorHandler()
{
    // only show errors in dev environment
    if ($this->Config['environment'] == 'dev') {
        $this->whoops = new \Whoops\Run();

        $Handler = new \Whoops\Handler\PrettyPageHandler();
        $Handler->setPageTitle("Divergence Error");

        $this->whoops->pushHandler($Handler);
        $this->whoops->register();
    } else {
        error_reporting(0);
    }
}
```
