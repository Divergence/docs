Documentation for the Divergence Framework

# Getting Started

## Server Prerequisites
- Get either nginx or apache2. You can use the built in PHP web server for testing as well.
- Install PHP 7.1
- Make sure you have composer installed


## Building a new project
- In your terminal run this from inside your project directory

    `composer require divergence/divergence`

- Copy some directories to bootstrap your project

    ```
    cp -R vendor/divergence/divergence/public ./
    cp -R vendor/divergence/divergence/views ./
    cp -R vendor/divergence/divergence/bootstrap ./
    cp -R vendor/divergence/divergence/config ./
    ```

 - Make a classes directory for yourself
    
    `mkdir classes`

 - Open your `composer.json` and add this config to give yourself a namespace

    ```
    "autoload": {
	    "psr-4": {
		    "project\\": "classes/"
	    }
    },
    ```
    Remember that your namespace will be whatever you put in for "project". For more details see composer's documentation.

 - Open `config/db.php` and give your new project some MySQL database credentials.
 - Create a new class `App` in your classes directory with the filename `App.php`. Simply extend `\Divergence\App`
    ```
    <?php
    namespace project;

    class App extends \Divergence\App {

    }
    ```
- Edit `bootstrap/app.php` and change the `use` at the top to use your new namespace in this case `project\App`.
    ```
        <?php
        use project\App as App;
    ```
- Make a directory in your classes folder called `Controllers` and make a new file named `SiteRequestHandler.php` with these contents:
    ```
    <?php
    namespace project\Controllers;

    class SiteRequestHandler extends \Divergence\Controllers\SiteRequestHandler {

    }
    ```
- Edit `bootstrap/router.php` and change the one line to say `project\Controllers\SiteRequestHandler::handleRequest();`


- Run `php -S localhost:8080 -t ./public/` from your project root directory.
- Visiting `localhost:8080` in your browser should show a phpinfo dump.
