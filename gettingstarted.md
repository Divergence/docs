### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)
# Getting Started

## Server Prerequisites

- Get either nginx or apache2. You can use the built in PHP web server for testing as well.
- Install PHP 8.4 or newer. The 3.3 source uses property hooks, even though the Composer requirement still says `>=8.1`.
- Make sure you have Composer installed
- For database-backed applications, enable PDO and the driver for your database: `pdo_mysql`, `pdo_pgsql`, or `pdo_sqlite`.
- Media processing also needs the tools used by the media types you enable. See [Media](media.md).

## Bootstrap a New Project
- It is recommended that you install and use the Divergence command line tool to bootstrap your project. If you wish to do this manually feel free to look in the section ahead.

```bash
composer global require divergence/cli
mkdir project
cd project
composer init
divergence init
```

What `divergence init` does:

- add `divergence/divergence` as a dependency
- set up a PSR-4 namespace under `src/`
- copy the framework bootstrap files into your project
- start a database configuration wizard

A video of this process is available below.
[![asciicast](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs.png)](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs)

## How to Bootstrap Manually (Advanced)
- Make sure you initialize with Composer first.
- In your terminal run this from inside your project directory:

`composer require divergence/divergence:^3.3`

- Copy the directories needed to bootstrap your project:

```bash
cp -R vendor/divergence/divergence/public ./
cp -R vendor/divergence/divergence/views ./
cp -R vendor/divergence/divergence/bootstrap ./
cp -R vendor/divergence/divergence/config ./
mkdir -p var/sqlite
```

Finish the App and controller setup below before serving the project. The bundled `SiteRequestHandler` is a placeholder that calls `phpinfo()` and exits. It is not your application's home page, and you should not expose it on a public server.

### Establish Your Classes Directory
- Make a source directory for yourself:

`mkdir src`

- Open your `composer.json` and add this config to give yourself a namespace:

```json
{
    "autoload": {
        "psr-4": {
            "project\\": "src/"
        }
    }
}
```

Merge the `autoload` entry into your existing file; don't replace your dependencies. Your namespace will be whatever you put in for `project`.

- Regenerate the autoloader:

```bash
composer dump-autoload
```

## Configure Database Access

- Open `config/db.php` and give your new project database credentials.
- The framework ships example labels for MySQL, PostgreSQL, and SQLite, plus test labels.
- The CLI's configuration wizard is MySQL-oriented. Configure PostgreSQL and SQLite directly using the examples in [Database](database.md).

[![asciicast](https://asciinema.org/a/gZHWY2tXwjxDgYPvzjIuUjhEX.png)](https://asciinema.org/a/gZHWY2tXwjxDgYPvzjIuUjhEX)

The shipped config currently includes labels like:

- `mysql`
- `dev-mysql`
- `tests-mysql`
- `tests-mysql-socket`
- `pgsql`
- `dev-pgsql`
- `tests-pgsql`
- `sqlite`
- `dev-sqlite`
- `tests-sqlite-memory`
- `tests-sqlite-files`

If you want an uncommitted local override, create `config/db.dev.php`. `config/db.php` returns that file immediately when it exists.

That override replaces the whole returned configuration, and its filename does not restrict it to development mode. Include every label the application needs. Select your label before the first model query; the defaults are `mysql` in production and `dev-mysql` in development.

## Take Over Control From The Framework
- Create a new class `App` in your classes directory with the filename `App.php`. Simply extend `\Divergence\App`

```php
<?php
namespace project;

class App extends \Divergence\App
{
}
```

- Edit `bootstrap/app.php` and change the `use` at the top to use your new namespace, in this case `project\App`.

```php
<?php
use project\App as App;
```

- Override `handleRequest()` in your `App` class so your application boots into your own root controller instead of the framework placeholder `Divergence\Controllers\SiteRequestHandler`.
- Make a directory in your classes folder called `Controllers` and make a new file named `Main.php` with these contents:

```php
<?php
namespace project\Controllers;

use Divergence\Responders\TwigBuilder;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class Main extends \Divergence\Controllers\RequestHandler
{
    public function __construct()
    {
        $this->responseBuilder = TwigBuilder::class;
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        return $this->respond('home.twig', [
            'message' => 'Divergence is running',
        ]);
    }
}
```

- Update your `App` class to dispatch through that root controller:

```php
<?php
namespace project;

use Divergence\Responders\Emitter;
use GuzzleHttp\Psr7\ServerRequest;
use project\Controllers\Main;

class App extends \Divergence\App
{
    public function handleRequest()
    {
        $main = new Main();
        $response = $main->handle(ServerRequest::fromGlobals());
        (new Emitter($response))->emit();
    }
}
```

- Make sure `bootstrap/router.php` has your app startup:

```php
$app->handleRequest();
```

- Add a view so your root controller has something to render:

```twig
{# views/home.twig #}
<h1>{{ message }}</h1>
```

Now run `php -S localhost:8080 -t ./public/` from the project root and visit `http://localhost:8080`. You should see your message. The built-in server is for development; the public directory is the web root in either setup.

## Configuring nginx or apache2 Servers
### nginx
```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/yourproject/public;
    index index.php index.html index.htm;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }
}
```

### apache2
```apache
<VirtualHost *:80>
    ServerName example.com

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/yourproject/public

    <Directory /var/www/yourproject/public>
        Options -Indexes -MultiViews +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

And inside `public/.htaccess`:

```apache
RewriteEngine On

RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*) index.php [L,QSA]
```

Adjust the hostname, document root, and PHP-FPM socket for your server. Enable Apache's rewrite module if you use the `.htaccess` example. These are basic routing examples; configure HTTPS and production logging for your deployment.
