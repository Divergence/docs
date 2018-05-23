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
All Divergence controllers extend from abstract class RequestHandler.

RequestHandler keeps track of the path, where you are in it, and provides utility methods for responding to a request.

#### Response Mode
```php
static::$responseMode
```
By default respondeMode is set to 'dwoo' which is the template engine of choice for Divergence. You may choose to change $respondeMode to 'json', 'jsonp', or simply 'return'.

| ResponseMode | Description |
| --- | --- |
| dwoo | Responds with a dwoo template looking in `App::$ApplicationPath.'/views/'` for a template. |
| json | Print a JSON string and sends header `Content-type: application/json`. |
| jsonp | Prints valid JS code that sets a variable `var data` to the data being output. |
| return | Returns a raw PHP array of the data and TemplatePath. |

#### API Reference
| Method | Purpose |
| --- | --- |
| setPath | Internal method for getting the path from `$_SERVER['REQUEST_URI']`. |
| peekPath | Returns the next path without moving the marker over. |
| shiftPath | Returns the next path while moving the marker over. |
| getPath | Returns the internal array derived from `$_SERVER['REQUEST_URI']`. |
| unshiftPath($path) | Lets you add a path to the internal path stack. |

*All of these are protected methods so you can only run them from inside a controller.*

## Your Own Controllers
Typically your app should have a Controller namespace under your main Application namespace which means you should have a `src/Controllers` directory. This directory is recommended for storing all your controllers so that they are easy to find. You can create sub directories for various types of controllers.

You should organize your controllers by type or subdivision of your project. For example you might organize controllers related to an admin control panel in a folder called admin.

## Using a third party routing library
Divergence in no way prevents you from using third-party routing libraries. Simply register the third party library in `project\Controllers\Main::handleRequest();`.

#### Built in controller classes for your conveniance
| Controller | Description |
| --- | --- |
| `RequestHandler` | A basic blank controller. See earlier section. |
| `RecordsRequestHandler` | Provides a basic CRUD API for Models extending `Divergence\Models\ActiveRecord` |

*Feel free to write your own

## RecordsRequestHandler
`Divergence\Controllers\RecordsRequestHandler` gives you a pre-made controller for doing REST operations on a `Divergence\Models\ActiveRecord` model.

### Building an API
*Example class*
```php
<?php
namespace project\Controllers\Records;

class BlogPost extends \Divergence\Controllers\RecordsRequestHandler
{
    use Permissions\LoggedIn;
    
    public static $recordClass = 'project\\Models\\BlogPost';
}
```

### Don't forget to add this controller to another controller's handleRequest tree.
```php
public static function handleRequest()
    {
        switch ($action = $action ? $action : static::shiftPath()) {
            case 'blogposts':
                return project\Controllers\Records\BlogPost::handleRequest();
```

### Permissions

*Example Trait*
```php
<?php
namespace project\Controllers\Records\Permissions;

use \project\App as App;

use \Divergence\Models\ActiveRecord as ActiveRecord;

trait LoggedIn
{
    public static function is()
    {
        /*
         * Here we are simply checking that the user is logged in.
         * In this case that is all we need to verify their permissions.
         * Of course you should use your own logic based on the
         * authentication system you have configured.
         */
        return App::$Session->CreatorID ? true : false;
    }
    
    public static function checkBrowseAccess($arguments)
    {
        return static::is();
    }

    public static function checkReadAccess(ActiveRecord $Record)
    {
        return static::is();
    }
    
    public static function checkWriteAccess(ActiveRecord $Record)
    {
        return static::is();
    }
    
    /*
     *  have this return false to disable API access entirely
     */
    public static function checkAPIAccess()
    {
        return static::is();
    }
}
```


## JSON API Reference
##### This API Reference is for classes that extend `Divergence\RecordsRequestHandler`.

For simplicity lets assume we have our api controller at `/blogposts/`.

### Browse
`URI: /blogposts/json`

`Method: GET, POST`

#### Parameters
| Name | Type | Description |
| --- | --- | --- |
| offset | number | Position offset in the database. |
| limit | number | Number of records to pull from offset. |
| sort | array | An array of order key value pairs. Can also be a JSON string. |
| filter | array | An array of filter key value pairs. Can also be a JSON string. By default filters will use the `AND` operator. |

##### All of these are accepted as GET or POST

#### Example Return
`Content-type: application/json`
```js
{
    "success": true,
    "data":[ /* array of objects corrosponding to your model */ ],
    "conditions":[], // the calculated conditions for the query based on filters you provided and controller configurables
    "total":"5", // number of records in the database total
    "limit":false, // the number of objects actually returned or false if unlimited
    "offset":false // the offset provided by you
}
```

#### Example
`$ curl -s http://localhost:8080/api/tags/json | jq`
```js
{
  "success": true,
  "data": [
    {
      "ID": "1",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523869087,
      "CreatorID": null,
      "Tag": "ssh",
      "Slug": "ssh"
    },
    {
      "ID": "2",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523870415,
      "CreatorID": "1",
      "Tag": "linux",
      "Slug": "linux"
    },
    {
      "ID": "3",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523870424,
      "CreatorID": "1",
      "Tag": "osx",
      "Slug": "osx"
    },
    {
      "ID": "4",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523870431,
      "CreatorID": "1",
      "Tag": "terminal",
      "Slug": "terminal"
    },
    {
      "ID": "5",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523870455,
      "CreatorID": "1",
      "Tag": "bash",
      "Slug": "bash"
    },
    {
      "ID": "6",
      "Class": "technexus\\Models\\Tag",
      "Created": 1524502523,
      "CreatorID": null,
      "Tag": "daemon",
      "Slug": "daemon"
    },
    {
      "ID": "7",
      "Class": "technexus\\Models\\Tag",
      "Created": 1524514976,
      "CreatorID": null,
      "Tag": "openssh",
      "Slug": "openssh"
    }
  ],
  "conditions": [],
  "total": "7",
  "limit": false,
  "offset": false
}
```

#### Example Failure
`$ curl -s http://localhost:8080/api/blogposts/json | jq`
```js
{
  "success": false,
  "failed": {
    "errors": "API access required."
  }
}
```
This will be returned if your controller's `checkAPIAccess()` method returns false. It returns true by default.


##### Get One Record


##### Edit One Record


##### Create One Record


##### Delete One Record


##### Create or Edit Multiple Records


##### Delete Multiple Records

