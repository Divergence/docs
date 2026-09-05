### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Architecture

## Request Respond Emit

Divergence makes use of `GuzzleHttp\Psr7\ServerRequest::fromGlobals()` to generate a `Request` object. A request object reads the built in PHP superglobals like `$_GET`, `$_POST`, `$_REQUEST`, but it also provides a uniform interface for things like headers, uploaded files, cookies, and the request body stream.

In response to this request object, controllers generate a response object. Divergence's own `Response` implements `Psr\Http\Message\ResponseInterface`, so it can carry status, headers, cookies, and body content in the normal PSR-7 shape.

Finally the response object is passed to an **Emitter**. The emitter does the actual work of sending headers, status, and body content. It also supports streamed responses and avoids continuing to emit data when the client connection has already terminated.

Generally all responses are created by controllers. Divergence comes with several built in controller classes to save you time, but they still return normal PSR-7 responses.

The request and response interfaces are PSR-compatible. The built-in handlers still use PHP superglobals and `App::$App`, though. Passing in a different request object does not replace all that global state. Keep that in mind when writing tests or embedding the framework in a long-running server.

## Boot Flow

By default `public/index.php` runs:

```php
require(__DIR__.'/../bootstrap/autoload.php');
require(__DIR__.'/../bootstrap/app.php');
require(__DIR__.'/../bootstrap/router.php');
```

Those files then do this work in order:

```php
define('DIVERGENCE_START', microtime(true));
require(__DIR__.'/../vendor/autoload.php');

use Divergence\App as App;

$app = new App(realpath(__DIR__.'/../'));
$app->handleRequest();
```

Inside `Divergence\App`:

- `ApplicationPath` is stored
- `Routing\Path` is initialized from `$_SERVER['REQUEST_URI']` for web requests; ordinary CLI startup skips it
- `config/app.php` is loaded
- the error handler is registered

And inside `handleRequest()`:

```php
public function handleRequest()
{
    $main = new SiteRequestHandler();
    $response = $main->handle(ServerRequest::fromGlobals());
    (new Emitter($response))->emit();
}
```

Override that method to dispatch into your own root controller. The bundled `SiteRequestHandler` calls `phpinfo()` and exits; it is a placeholder, not a production route. See [Getting Started](gettingstarted.md#take-over-control-from-the-framework) for the complete setup.

## Path Stack Routing

Divergence does not use route config files. Instead controllers consume the URI one segment at a time.

For example, with the path:

`/api/blog/1/edit`

successive calls to `shiftPath()` return:

1. `api`
2. `blog`
3. `1`
4. `edit`

If there is nothing left in the stack it returns `false`.

That means each controller can take over a branch of the URL tree, then hand off to a more specific controller as needed.

## Response Builders

Controllers normally choose their response format through `$this->responseBuilder`.

Built in builders are:

- `TwigBuilder`
- `JsonBuilder`
- `JsonpBuilder`
- `MediaBuilder`
- `EmptyBuilder`

`RequestHandler::respond($responseID, $responseData)` wraps the chosen builder in `Divergence\Responders\Response`.

So in practice a controller usually does one of these:

```php
$this->responseBuilder = \Divergence\Responders\TwigBuilder::class;
return $this->respond('posts.twig', ['Posts' => $Posts]);
```

or:

```php
$this->responseBuilder = \Divergence\Responders\JsonBuilder::class;
return $this->respond('ignored-in-json-mode', ['success' => true, 'data' => $Posts]);
```

## Endpoint Based Controllers

Endpoint classes keep individual actions out of the main handler:

- `RequestHandler` supports endpoint registration
- endpoint objects are lazily instantiated through `__call()`
- `RecordsRequestHandler` and `MediaRequestHandler` dispatch most action handling into focused endpoint classes

The public handler methods still dispatch those actions. Override the relevant handler hooks or register an endpoint when you need application-specific behavior; you don't need to copy the entire CRUD controller.

## Data Layer

The model API stays small, but the work is split up:

| Component | Responsibility |
| --- | --- |
| `Models\ActiveRecord` and `Models\Model` | Fields, state, validation, and the public persistence API |
| `Models\Factory` | Model getter dispatch and coordination |
| `Models\Factory\ModelMetadata` | Cached model and field definitions |
| `Models\Factory\Instantiator` and `EventBinder` | Hydrate records and initialize model state |
| `IO\Database\Connections` | Resolve the selected PDO connection and storage backend |
| Database query and writer classes | Build backend-specific SQL |
| `Data\Collections` | In-memory storage, indexes, queries, and Math |
| `Models\Collections\RecordCollection` | Collection behavior tied to model events and persistence |

These are different layers. `Tag::getAllByWhere()` queries the database. `$tags->getAllByCriteria()` queries the records already loaded into a collection. `$tags->sum()` does arithmetic in PHP; it does not issue `SELECT SUM(...)`.

Model definitions, reflection information, getter instances, and connections have process-local caches. That helps ordinary PHP request execution, but it is not a distributed cache or a promise that global state is isolated between requests in a persistent worker.

The [ORM](orm.md), [Database](database.md), [Collections](collections.md), and [Math](math.md) chapters cover each part in detail.
