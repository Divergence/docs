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

This will route all paths to the one other file in a public directory by default: an `index.php` file.

`index.php` requires these files in order
* `bootstrap/autoload.php`
    ```php
    define('DIVERGENCE_START', microtime(true));
    require(__DIR__.'/../vendor/autoload.php');
    ```
    Store start time for timing & run composer's autoload.

* `bootstrap/app.php`
    ```php
    use project\App as App;
    App::init(realpath(__DIR__.'/../'));
    ```
    Typically you would use your namespace here instead of project with your own App class that extends `Divergence\App`.

* `bootstrap/router.php`
    ```php
    project\Controllers\SiteRequestHandler::handleRequest();
    ```
    Typically you would use your namespace here as well.

## Folder Structure
| Folder | Description |
| --- | --- |
| bootstrap | Bootstrap files described above |
| config | Config files |
| public | Static files |
| src | Default location for your project's php code |
| tests | Unit tests |
| vendor | Composer dependencies |
| views | templates |



# The App Class
It is highly recommended that you extend the App class or redefine it treating the existing methods as if it were an interface.

## Configs
---
You can load configs at anytime by calling `$config = App::config('file');` to load `config/file.php`. The return will be whatever the file itself returns.

```<?php
return [ 'some', 'example', 'config' => 'data' ];
```

Make sure the config exists though because otherwise it will throw an excepton.
```php
    public static function config($Label)
    {
        $Config = static::$ApplicationPath . '/config/' . $Label . '.php';
        if (!file_exists($Config)) {
            throw new \Exception($Config . ' not found in '.static::class.'::config()');
        }
        return require $Config;
    }
```

Feel free to make your own configs as you see fit.

## Development & Production
---
To switch to dev mode simply create an empty file in your project directory named `.dev`.

`$ touch .dev`

The MySQL class uses these default database configs.
```php
    public static $defaultProductionLabel = 'mysql';
    public static $defaultDevLabel = 'dev-mysql';
```

#### The existence of the .dev file effectively switches your database connection.
#### Tip: You can override these default labels for things like unit tests and continuous integrations before any database connections are established!

This behavior is part of the `Divergence\App` class.

#### This is the first method that runs after composer.
```php
    public static function init($Path)
    {
        static::$ApplicationPath = $Path;
        
        Controllers\RequestHandler::$templateDirectory = $Path.'/views/';

        static::$Config = static::config('app');
        
        static::registerErrorHandler();
    }
```


The contents of `config/app.php` is where the production or development mode is really set.
```php
use \Divergence\App as App;

return [
    'debug'			=>	file_exists(App::$ApplicationPath . '/.debug'),
    'environment'	=>	(file_exists(App::$ApplicationPath . '/.dev') ? 'dev' : 'production'),
];
```

A simple way to switch environments provided for you but you're welcome to put in your own right here.

#### Error Handling
Development mode turns on Whoops error handling.

Production mode sets error_reporting to 0 so that a user does not ever see an error.

It is recommended that you bind your own error handler to log and potentially email critical errors.

```php
    public static function registerErrorHandler()
    {
        // only show errors in dev environment
        if (static::$Config['environment'] == 'dev') {
            static::$whoops = new \Whoops\Run;
            
            $Handler = new \Whoops\Handler\PrettyPageHandler;
            $Handler->setPageTitle("Divergence Error");
            
            static::$whoops->pushHandler($Handler);
            static::$whoops->register();
        } else {
            error_reporting(0);
        }
    }
```