### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Controllers
Divergence comes with a suite of controllers to aid in building APIs rapidly as well as a built in helper class for building your own controllers.

## Intro to Tree Routing
Divergence does away with routing configuration files. Instead controllers "take over" a directory path during runtime, passing control from the main application controller to other controllers until one returns a response. The App passes that response to the emitter.

To illustrate how this works in practice, let's take a look at this simple example:

```php
<?php
namespace application\Controllers;

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
        switch ($action = $this->shiftPath()) {
            case 'admin':
                return (new Admin())->handle($request);

            case 'api':
                return (new API())->handle($request);

            case 'media':
                return (new Media())->handle($request);

            case 'thumbnail':
                $this->unshiftPath('thumbnail');
                return (new Media())->handle($request);

            default:
                return $this->respond('home.twig');
        }
    }
}
```

By default our file `/public/index.php` will run:

```php
require(__DIR__.'/../bootstrap/autoload.php');
require(__DIR__.'/../bootstrap/app.php');
require(__DIR__.'/../bootstrap/router.php');
```

Subsequently these 3 requires run code equivalent to:

```php
define('DIVERGENCE_START', microtime(true));
require(__DIR__.'/../vendor/autoload.php');

use project\App as App;

$app = new App(realpath(__DIR__.'/../'));
$app->handleRequest();
```

Typically the name of the application will be different, so of course your namespace will be different. You *should* extend the App class and write your own root handler.

`Admin`, `API`, and `Media` above are application controllers in the same namespace. Your `Media` controller can extend `Divergence\Controllers\MediaRequestHandler` and enforce your media access policy. The thumbnail branch puts the action back on the path so the media handler can initialize the request and dispatch it normally.

### About `$this->shiftPath()`
`$this->shiftPath()` returns the next directory in the request URI every time it is executed.

For this path:

`/api/blog/1/edit`

`shiftPath()` will return `api`, then `blog`, then `1`, and finally `edit`.

If there is nothing left in the stack it will return `false`.

Utilizing this method it does not matter where in the tree the controller is. It will always be able to pick up from where it is currently.

## RequestHandler
All Divergence controllers extend abstract class `Divergence\Controllers\RequestHandler`.

RequestHandler keeps track of the path, where you are in it, and provides utility methods for responding to a request.

### Response Mode
```php
$this->responseBuilder
```

Divergence chooses response format through the response builder attached to the controller. The base helper is:

```php
respond($responseID, $responseData = [])
```

| Response Builder | Description |
| --- | --- |
| `TwigBuilder` | Responds with a Twig template looking in `App::$App->ApplicationPath.'/views/'` for a template. |
| `JsonBuilder` | Builds JSON with `Content-Type: application/json`. |
| `JsonpBuilder` | Builds JavaScript assigning `var data`; this is not a configurable JSONP callback. |
| `MediaBuilder` | Streams file data and supports byte ranges. |
| `EmptyBuilder` | Returns an empty response body with status and headers only. |

### API Reference
| Method | Purpose |
| --- | --- |
| `peekPath` | Returns the next path without moving the marker over. |
| `shiftPath` | Returns the next path while moving the marker over. |
| `unshiftPath($path)` | Lets you add a path to the internal path stack. |
| `respond($responseID, $responseData = [])` | Builds a response using the current response builder. |

These are controller methods intended to be used from inside your handlers.

`respond()` builds a response; it does not send it. Return the response and let the emitter handle output. When setting a status or header, use the returned response, for example `return $this->respond('error.twig')->withStatus(404);`.

### Endpoint Registration
The current controller base also supports endpoint registration internally:

- handlers register endpoint classes
- `__call()` lazily instantiates endpoint objects
- request-handling methods like `handleBrowseRequest()` are delegated into those endpoint classes

This keeps the action implementations in `Controllers/Records/Endpoints` and `Controllers/Media/Endpoints` while preserving the handler methods used by application code.

## Your Own Controllers
Typically your app should have a controller namespace under your main application namespace, which means you should have a `src/Controllers` directory. This directory is recommended for storing all your controllers so that they are easy to find. You can create subdirectories for various types of controllers.

You should organize your controllers by type or subdivision of your project. For example you might organize controllers related to an admin control panel in a folder called `Admin`.

## Using a Third Party Routing Library
Divergence in no way prevents you from using third-party routing libraries. Simply register the third party library in your application's root controller `handle(...)` flow.

### Built in Controller Classes for Your Convenience
| Controller | Description |
| --- | --- |
| `RequestHandler` | A basic blank controller. |
| `RecordsRequestHandler` | Provides a basic CRUD API for models extending `Divergence\Models\ActiveRecord`. |
| `MediaRequestHandler` | Provides a basic CRUD/media API for `Divergence\Models\Media\Media` and subclasses. Includes upload, thumbnail, streaming, and other media features. |

*Feel free to write your own*

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

Internally the current implementation registers dedicated endpoint classes for:

- browse
- record
- create
- edit
- delete
- multi-save
- multi-destroy

The public API is still the familiar CRUD surface, but the internals are now split into focused endpoint classes instead of being one large handler.

### Don't Forget to Add This Controller to Another Controller's `handle` Tree
```php
/**
 * Routes
 *  /api/blogpost
 *  /api/tags
 */
public function handle(ServerRequestInterface $request): ResponseInterface
{
    switch ($action = $this->shiftPath()) {
        case 'blogpost':
            return (new BlogPost())->handle($request);

        case 'tags':
            return (new Tag())->handle($request);

        default:
            return (new \GuzzleHttp\Psr7\Response())->withStatus(404);
    }
}
```

### Permissions

*Example Trait*
```php
<?php
namespace project\Controllers\Records\Permissions;

use project\App as App;
use Divergence\Models\ActiveRecord;

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
        return App::$App->isLoggedIn();
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

    /*
     * have this return false to disable API access entirely
     */
    public function checkAPIAccess()
    {
        return $this->is();
    }
}
```

## Security
Security in Divergence is mostly enforced at the controller layer. The framework gives you hooks, but your application is responsible for defining the actual policy.

The main security hooks you should care about are:

- `checkBrowseAccess()`
- `checkReadAccess()`
- `checkWriteAccess()`
- `checkUploadAccess()`
- `checkAPIAccess()`

In practice:

- `checkAPIAccess()` gates JSON API access
- `checkWriteAccess()` protects create, edit, delete, and batch save/destroy paths
- the upload endpoint calls `checkUploadAccess()`, but currently ignores a returned `false`; enforce media denial before dispatch or through an exception handled by your application
- shared authorization logic belongs in reusable traits

The defaults allow access. The `isLoggedIn()` method in the example is one you implement on your App, not a built-in method. Account-level settings alone do not enforce a policy. See [Security](security.md#binding-permissions) before exposing these endpoints.

## JSON API Reference
This API reference is for classes that extend `Divergence\Controllers\RecordsRequestHandler`.

For simplicity let's assume we have our API controller mounted at `/api/tags/`.

The response bodies below are illustrative, not snapshots from a particular database. IDs, timestamps, counts, and validation messages depend on the model and fixtures. A `success: false` payload does not automatically change the HTTP status; customize error responses if your API requires specific status codes.

### Route Shape
The `json` path segment chooses the response builder:

- JSON mode is entered by routing through `/json`
- that means JSON paths look like `/api/tags/json/...`
- they do **not** look like `/api/tags/.../json`

That matches the current `RecordsRequestHandler::handle()` implementation.

An `Accept: application/json` header alone does not select this mode.

### Browse
`URI: /api/tags/json`

`Method: GET, POST`

### Parameters
| Name | Type | Description |
| --- | --- | --- |
| `offset` | number | Position offset in the database. |
| `start` | number | Alias for `offset`. |
| `limit` | number | Number of records to pull from offset. |
| `sort` | JSON array | An array of order key-value pairs. |
| `filter` | JSON array | An array of key-value pairs. By default filters use `AND`. |

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
        'property' => 'FirstName',
        'direction' => 'ASC',
    ]
]
```

### Filtering
Specify filter rules with a JSON encoded array.

These are model field names and values, not the in-memory `CriteriaGroup` API. Filters are added to the model's conditions by field name; repeating a property replaces its earlier value. Use an application endpoint when you need a different query shape or a restricted field allowlist.

```php
[
    [
        'property' => 'FirstName',
        'value' => 'John',
    ],
    [
        'property' => 'LastName',
        'value' => 'Doe',
    ]
]
```

### Example Return
`Content-Type: application/json`
```js
{
    "success": true,
    "data": [ /* array of objects corresponding to your model */ ],
    "conditions": [],
    "total": "5",
    "limit": false,
    "offset": false
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
      "Class": "project\\Models\\Tag",
      "Created": 1523869087,
      "CreatorID": null,
      "Tag": "ssh",
      "Slug": "ssh"
    },
    {
      "ID": "2",
      "Class": "project\\Models\\Tag",
      "Created": 1523870415,
      "CreatorID": "1",
      "Tag": "linux",
      "Slug": "linux"
    }
  ],
  "conditions": [],
  "total": "7",
  "limit": false,
  "offset": false
}
```

Browse with sort and filter:

```bash
curl -sG http://localhost:8080/api/tags/json \
  --data-urlencode 'limit=10' \
  --data-urlencode 'offset=0' \
  --data-urlencode 'sort=[{"property":"Created","direction":"DESC"}]' \
  --data-urlencode 'filter=[{"property":"Tag","value":"linux"}]' | jq
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

It returns true by default. You must redefine it to set up permissions.

### One Record
`URI: /api/tags/json/:handle`

`Method: GET, POST`

The generic `RecordsRequestHandler` resolves records through `getRecordByHandle($action)`. In practice that means this path segment is whatever your model exposes through `getByHandle(...)`, not necessarily a numeric primary key.

### Example
`$ curl -s http://localhost:8080/api/tags/json/2 | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "project\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "linux",
    "Slug": "linux"
  }
}
```

### Edit One Record
`URI: /api/tags/json/:handle/edit`

`Method: POST, PUT`

### Examples
Values that do not belong to this model are ignored. The record is returned.

`$ curl -d "param1=value1&param2=value2" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "project\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "linux",
    "Slug": "linux"
  }
}
```

Trying to change an auto-increment field such as the default `ID` will be ignored. That is not a blanket guarantee for every custom primary key or other sensitive field. Restrict writable fields in your application.

`$ curl -d "ID=2" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "project\\Models\\Tag",
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
    "Class": "project\\Models\\Tag",
    "Created": 1523870415,
    "CreatorID": "1",
    "Tag": "curl",
    "Slug": "curl"
  }
}
```

Feel free to use a JSON string as your data payload with `Content-Type: application/json`.

`$ curl -d '{"Tag":"JSON", "Slug":"json"}' -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/2/edit | jq`

```js
{
  "success": true,
  "data": {
    "ID": "2",
    "Class": "project\\Models\\Tag",
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
    "Class": "project\\Models\\Tag",
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
`URI: /api/tags/json/create`

`Method: POST, PUT`

The return will provide you with the new primary key and timestamp of when it was created.

### Example
`$ curl -d '{"Tag":"ActiveRecord", "Slug":"activerecord"}' -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/create | jq`

```js
{
  "success": true,
  "data": {
    "ID": "8",
    "Class": "project\\Models\\Tag",
    "Created": 1527133465,
    "CreatorID": null,
    "Tag": "ActiveRecord",
    "Slug": "activerecord"
  }
}
```

### Delete One Record
`URI: /api/tags/json/:handle/delete`

`Method: POST`

The return will provide you with the data from the model that was deleted.

`$ curl -H "Content-Type: application/json" -X POST -s http://localhost:8080/api/tags/json/8/delete | jq`

```js
{
  "success": true,
  "data": {
    "ID": "8",
    "Class": "project\\Models\\Tag",
    "Created": 1527133465,
    "CreatorID": null,
    "Tag": "ActiveRecord",
    "Slug": "activerecord"
  }
}
```

If you request the delete URL without POST, the generic endpoint returns a confirmation response instead of destroying the record immediately.

### Create or Edit Multiple Records
`URI: /api/tags/json/save`

`METHOD: POST, PUT`

The current batch save endpoint expects a payload shaped like:

```json
{
  "data": [
    { "Tag": "one", "Slug": "one" },
    { "ID": 2, "Tag": "updated", "Slug": "updated" }
  ]
}
```

Example:

```bash
curl -s -X POST http://localhost:8080/api/tags/json/save \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      { "Tag": "Batch One", "Slug": "batch-one" },
      { "ID": 2, "Tag": "Batch Updated", "Slug": "batch-updated" }
    ]
  }' | jq
```

Representative response:

```js
{
  "success": true,
  "data": [
    {
      "ID": "9",
      "Tag": "Batch One",
      "Slug": "batch-one"
    },
    {
      "ID": "2",
      "Tag": "Batch Updated",
      "Slug": "batch-updated"
    }
  ],
  "failed": []
}
```

### Delete Multiple Records

Batch operations process records individually. They are not all-or-nothing transactions, and `success` can be true when some records failed. Always inspect `failed` as well as `data` for both batch saves and batch deletes.

`URI: /api/tags/json/destroy`

`METHOD: POST, PUT, DELETE`

The current batch delete endpoint expects `data` as either:

- an array of numeric IDs
- an array of objects containing a numeric primary key

Example:

```bash
curl -s -X DELETE http://localhost:8080/api/tags/json/destroy \
  -H "Content-Type: application/json" \
  -d '{ "data": [8, 9] }' | jq
```

Representative response:

```js
{
  "success": true,
  "data": [
    { "ID": "8" },
    { "ID": "9" }
  ],
  "failed": []
}
```

If `data` is malformed you will get an error similar to:

```js
{
  "success": false,
  "failed": {
    "errors": "Save expects \"data\" field as array of records."
  }
}
```

## MediaRequestHandler
`Divergence\Controllers\MediaRequestHandler` extends the generic records controller with media-specific behavior for `Divergence\Models\Media\Media`.

In addition to the record endpoints above, it registers endpoints for:

- upload
- open/media streaming
- info
- download
- caption
- thumbnail
- media browse
- media delete

Important current behavior:

- byte-range requests are supported for streamed media
- cache headers and `ETag` headers are added on media responses
- uploads default to the file field name `mediaFile`
- JSON mode is again entered through `/json` at the start of the media controller path

Examples:

Browse media as JSON:

```bash
curl -s http://localhost:8080/media/json/browse | jq
```

Upload media:

```bash
curl -s -X POST http://localhost:8080/media/json/upload \
  -F 'mediaFile=@./tests/assets/logo.png' \
  -F 'Caption=Example upload' | jq
```

Get media metadata:

```bash
curl -s http://localhost:8080/media/json/info/1 | jq
```

Download the original media:

```bash
curl -OJ http://localhost:8080/media/download/1/logo.png
```

Stream partial media:

```bash
curl -i http://localhost:8080/media/open/1 \
  -H 'Range: bytes=0-1023'
```

The [Media chapter](media.md) covers storage paths, import behavior, required tools, thumbnails, and streaming. Media routes use action-first paths such as `/media/open/1`, not the records controller's `/1/edit` pattern.
