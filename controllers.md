### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Controllers
Divergence comes with a suite of controllers to aid in building APIs rapidly as well as a build in helper class for building your own controllers.

## Intro to Tree Routing
Divergence does away with routing configuration files. Instead controllers "take over" a directory path during run time bubbling up from the Main application controller to other controllers until eventually one of the controllers responds to the request and ends the PHP thread.

To illustrate how this works in progress let's take a look at this simple example:
```php
<?php
namespace application\Controllers;

class Main extends \Divergence\Controllers\RequestHandler
{
    public static function handleRequest()
    {
        switch ($action = $action ? $action : static::shiftPath()) {
            case 'admin':
                return Admin::handleRequest();
                
            case 'api':
                return API::handleRequest();
                
            case 'logout':
                return static::logout();
                    
            case '':
                return static::home();
                break;

            default:
                return static::error();
        }
    }
}
```
By default our file `/public/index.php` will run 
```php
application\Controllers\Main::handleRequest();
```

Typically the name of the application will be different so of course your namespace will be different however projects *should* use the Main controller to take over the root path of their website.

#### About `shiftPath()`
`shiftPath()` returns the next directory in the request uri every time it is executed.
For example for this path:

`/api/blog/1/edit`

`shiftPath()` will return 'api', then 'blog',' then '1', and finally 'edit'.

If there is nothing left in the stack it will return false.

Utilizing this method it doesn't matter where in the tree the controller is. It will always be able to pick up from where it is currently.


## RequestHandler

## Your Own Controllers