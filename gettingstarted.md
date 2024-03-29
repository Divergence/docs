### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)
# Getting Started

## Server Prerequisites
- Get either nginx or apache2. You can use the built in PHP web server for testing as well.
- Install PHP 8.0
- Make sure you have composer installed

## Bootstrap a new project
- It is recommended that you install and use the divergence command line tool to bootstrap your project. If you wish to do this manually feel free to look in the section ahead.
    ```
    composer global require divergence/cli`
    mkdir project
    cd project
    composer init
    divergence init
    ```
    A video of this process is available below.
    [![asciicast](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs.png)](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs)

## How to Bootstrap Manually (Advanced)
- Make sure you initialize with composer first.
- In your terminal run this from inside your project directory

    `composer require divergence/divergence`

- Copy some directories to bootstrap your project

    ```
    cp -R vendor/divergence/divergence/public ./
    cp -R vendor/divergence/divergence/views ./
    cp -R vendor/divergence/divergence/bootstrap ./
    cp -R vendor/divergence/divergence/config ./
    ```
- Run `php -S localhost:8080 -t ./public/` from your project root directory.
- Visiting `localhost:8080` in your browser should show a phpinfo dump.

    #### Establish your classes directory
 - Make a source directory for yourself
    
    `mkdir src`

 - Open your `composer.json` and add this config to give yourself a namespace

    ``` json
    "autoload": {
	    "psr-4": {
		    "project\\": "src/"
	    }
    },
    ```
    Remember that your namespace will be whatever you put in for "project". For more details see composer's documentation.

## Configure Database access

 - Open `config/db.php` and give your new project some MySQL database credentials.
 - You can also use the divergence command line tool for this.
 [![asciicast](https://asciinema.org/a/gZHWY2tXwjxDgYPvzjIuUjhEX.png)](https://asciinema.org/a/gZHWY2tXwjxDgYPvzjIuUjhEX)

 ## Take over control from the framework
 - Create a new class `App` in your classes directory with the filename `App.php`. Simply extend `\Divergence\App`
    ``` php
    <?php
    namespace project;

    class App extends \Divergence\App {

    }
    ```
- Edit `bootstrap/app.php` and change the `use` at the top to use your new namespace in this case `project\App`.
    ``` php
        <?php
        use project\App as App;
    ```
- Make a directory in your classes folder called `Controllers` and make a new file named `SiteRequestHandler.php` with these contents:
    ``` php
    <?php
    namespace project\Controllers;

    class SiteRequestHandler extends \Divergence\Controllers\SiteRequestHandler {

    }
    ```
- Make sure `bootstrap/router.php` has your app startup `$app->handleRequest();`

## Configuring nginx or apache2 Servers
### nginx
```nginx
server {
    listen 80;
    listen [::]:80;

    . . .

    root /var/www/yourproject/public;
    index index.php index.html index.htm index.nginx-debian.html;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    . . .
}
```

### apache2
```apache
<VirtualHost *:80>
    ServerName example.com

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/yourproject/public

    . . .

    <Directory /var/www/yourproject/public>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>
    
    . . .
    
</VirtualHost>
```