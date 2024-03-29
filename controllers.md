### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Controllers
Divergence comes with a suite of controllers to aid in building APIs rapidly as well as a build in helper class for building your own controllers.

## Intro to Tree Routing
Divergence does away with routing configuration files. Instead controllers "take over" a directory path during run time bubbling up from the Main application controller to other controllers until eventually one of the controllers responds to the request and ends the PHP thread.

To illustrate how this works in practice let's take a look at this simple example:
```php
<?php
namespace application\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class Main extends \Divergence\Controllers\RequestHandler
{
    public string $path;
    protected ServerRequestInterface $request;
    
    public function __construct()
    {
        $this->path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        $this->request = $request;
        switch ($action = $this->shiftPath()) {
            case 'admin':
                return (new Admin())->handle($request);
                
            case 'api':
                return (new API())->handle($request);

            case 'media':
                return (new Media())->handle($request);
                
            case 'logout':
                return $this->logout();
                
            default:
                return $this->notfound();
        }
    }
}
```
By default our file `/public/index.php` will run 
```php
require(__DIR__.'/../bootstrap/autoload.php');
require(__DIR__.'/../bootstrap/app.php');
require(__DIR__.'/../bootstrap/router.php');
```

Subsequently these 3 requires run this code:
```php
use technexus\App as App;
define('DIVERGENCE_START', microtime(true));
require(__DIR__.'/../vendor/autoload.php');

$app = new App(realpath(__DIR__.'/../'));
$app->handleRequest();
```

Typically the name of the application will be different so of course your namespace will be different. You *should* extend the App class and write your own handler.

### About `$this->shiftPath()`
`$this->shiftPath()` returns the next directory in the request uri every time it is executed. The method comes with 
For example for this path:

`/api/blog/1/edit`

`shiftPath()` will return 'api', then 'blog',' then '1', and finally 'edit'.

If there is nothing left in the stack it will return false.

Utilizing this method it doesn't matter where in the tree the controller is. It will always be able to pick up from where it is currently.


## RequestHandler
All Divergence controllers extend from abstract class `Divergence\Controllers\RequestHandler`.

RequestHandler keeps track of the path, where you are in it, and provides utility methods for responding to a request.

### Response Mode
```php
static::$responseMode
```
By default responseMode is set to 'dwoo' which is the template engine of choice for Divergence. You may choose to change $responseMode to 'json', 'jsonp', or simply 'return'.

| ResponseMode | Description |
| --- | --- |
| dwoo | Responds with a dwoo template looking in `App::$ApplicationPath.'/views/'` for a template. |
| json | Print a JSON string and sends header `Content-type: application/json`. |
| jsonp | Prints valid JS code that sets a variable `var data` to the data being output. |
| return | Returns a raw PHP array of the data and TemplatePath. |

### API Reference
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

### Built in controller classes for your convenience
| Controller | Description |
| --- | --- |
| `RequestHandler` | A basic blank controller. See earlier section. |
| `RecordsRequestHandler` | Provides a basic CRUD API for Models extending `Divergence\Models\ActiveRecord` |
| `MediaRequestHandler` | Provides a basic CRUD API for Media extending `Divergence\Models\ActiveRecord`. Provides automatic thumbnailing, uploading, and other media features. |

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
    /**
     * Routes
     *  /api/blogpost
     *  /api/tags
     */
    public function handle(RequestInterface $request):ResponseInterface
    {
        switch ($action = $this->shiftPath()) {
            case 'blogpost':
                return (new BlogPost())->handle($request);

            case 'tags':
                return (new Tag())->handle($request);
            
        }
    }
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
    public function is()
    {
        /*
         * Here we are simply checking that the user is logged in.
         * In this case that is all we need to verify their permissions.
         * Of course you should use your own logic based on the
         * authentication system you have configured.
         */
        return App::$App->is_loggedin();
    }
    
    public function checkBrowseAccess($arguments)
    {
        return $this->is();
    }

    public function checkReadAccess(ActiveRecord $Record)
    {
        return $this->is();
    }
    
    public function checkWriteAccess(ActiveRecord $Record)
    {
        return $this->is();
    }

    public function checkUploadAccess()
    {
        return $this->is();
    }
    
    /*
     *  have this return false to disable API access entirely
     */
    public function checkAPIAccess()
    {
        return $this->is();
    }
}

```


## JSON API Reference
This API Reference is for classes that extend `Divergence\RecordsRequestHandler`.

For simplicity lets assume we have our api controller at `/blogposts/`.

### Browse
`URI: /blogposts/json`

`Method: GET, POST`

### Parameters
| Name | Type | Description |
| --- | --- | --- |
| offset | number | Position offset in the database. |
| limit | number | Number of records to pull from offset. |
| sort | json array | An array of order key value pairs. |
| filter | json array | An array of key value pairs. By default filters will use the `AND` operator. |

##### All of these are accepted as GET or POST

### Example Sorting
Specify sort rules with a JSON encoded array.
```php
[
    [
        'property' => 'LastName',
        'direction' => 'ASC',
    ],
    [
        'property' =>  'FirstName',
        'direction' =>  'ASC',
    ]
]
```

### Filtering
Specify filter rules with a JSON encoded array.
```php
[
    [
        'property' => 'FirstName',
        'value'     => 'John',
    ],
    [
        'property'  =>  'LastName',
        'value'     =>  'Doe',
    ]
]
```

### Example Return
`Content-type: application/json`
```js
{
    "success": true,
    "data":[ /* array of objects corresponding to your model */ ],
    "conditions":[], // the calculated conditions for the query based on filters you provided and controller configurables
    "total":"5", // number of records in the database total
    "limit":false, // the number of objects actually returned or false if unlimited
    "offset":false // the offset provided by you
}
```

### Example
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

### Example Failure
`$ curl -s http://localhost:8080/api/blogposts/json | jq`
```js
{
  "success": false,
  "failed": {
    "errors": "API access required."
  }
}
```
This will be returned if your controller's `checkAPIAccess()` method returns false.

It returns true by default. You must redefine it to setup permissions.

If you plan to share permissions you should build yourself a permissions trait to re-use.


### One Record
`URI: /blogposts/json/:id`

`Method: GET, POST`



### Example
`$ curl -s http://localhost:8080/api/tags/json/2 | jq`

```js
{
  "success": true,
  "data": {
      "ID": "2",
      "Class": "technexus\\Models\\Tag",
      "Created": 1523870415,
      "CreatorID": "1",
      "Tag": "linux",
      "Slug": "linux"
  }
}
```

### Edit One Record
`URI: /blogposts/json/:id/edit`

`Method: POST`

### Examples
Values that do not belong to this model are completely ignored. The record is returned.

`$ curl -d "param1=value1&param2=value2" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "technexus\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "linux",
    "Slug": "linux"
  }
}
```

Trying to change the primary key will be ignored.

`$ curl -d "ID=2" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "technexus\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "linux",
    "Slug": "linux"
  }
}
```

Changes will be returned with the record.

`$ curl -d "Tag=curl&Slug=curl" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`
```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "technexus\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "curl",
    "Slug": "curl"
  }
}
```

Feel free to use a JSON string as your data payload with `Content-Type: application/json`

`curl -d '{"Tag":"JSON", "Slug":"json"}' -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "technexus\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "JSON",
    "Slug": "json"
  }
}
```

Validation failures bubble up from the model cleanly.

`$ curl -d '{"Tag":"J", "Slug":"j"}' -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": false,
  "data": {
    "ID": "2",
    "Class": "technexus\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "J",
    "Slug": "j",
    "validationErrors": {
      "Tag": "Tag must be at least two characters."
    }
  }
}
```
Know which field caused the error and why.

[Click here for many validation definition examples.](/orm.md#validation)

### Create One Record
`URI: /blogposts/json/create`

`Method: POST`

The return will provide you with the new primary key and unix timestamp of when it was created.

### Example

`curl $ curl -d '{"Tag":"ActiveRecord", "Slug":"activerecord"}' -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/create | jq`

```js
{
  "success": true,
  "data": {
    "ID": "8",
    "Class": "technexus\\Models\\Tag",
    "Created": 1527133465,
    "CreatorID": null,
    "Tag": "ActiveRecord",
    "Slug": "activerecord"
  }
}
```

### Delete One Record
`URI: /blogposts/json/:id/delete`

`Method: POST, DELETE`

The return will provide you with the data from the model that was deleted.

`$ curl -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/8/delete | jq`

```js
{
  "success": true,
  "data": {
    "ID": "8",
    "Class": "technexus\\Models\\Tag",
    "Created": 1527133465,
    "CreatorID": null,
    "Tag": "ActiveRecord",
    "Slug": "activerecord"
  }
}
```

### Create or Edit Multiple Records
`URI: /blogposts/json/save`

`METHOD: POST`

### Delete Multiple Records
`URI: /blogposts/json/delete`

`METHOD: POST`
