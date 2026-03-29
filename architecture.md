### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Architecture

## Request Respond Emit

Divergence makes use of `GuzzleHttp\Psr7\ServerRequest::fromGlobals()` to generate a `Request` object. A request object reads the built in PHP superglobals like `$_GET`, `$_POST`, `$_REQUEST`, but it also provides a uniform interface for things like headers, uploaded files, cookies, and the request body stream.

In response to this request object, controllers generate a response object. Divergence's own `Response` implements `Psr\Http\Message\ResponseInterface`, so it can carry status, headers, cookies, and body content in the normal PSR-7 shape.

Finally the response object is passed to an **Emitter**. The emitter does the actual work of sending headers, status, and body content. It also supports streamed responses and avoids continuing to emit data when the client connection has already terminated.

Generally all responses are created by controllers. Divergence comes with several built in controller classes to save you time, but they still return normal PSR-7 responses.

Using this architecture, most code written for Divergence is portable in the PSR-7 sense even though the framework's routing and ORM are custom.

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
- `Routing\Path` is initialized from `$_SERVER['REQUEST_URI']`
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

Real applications usually override that method so that they dispatch into their own root controller instead of the placeholder `SiteRequestHandler`.

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

Current built in builders are:

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

One important current-state change is internal, not conceptual:

- `RequestHandler` now supports endpoint registration
- endpoint objects are lazily instantiated through `__call()`
- `RecordsRequestHandler` and `MediaRequestHandler` dispatch most action handling into focused endpoint classes

That means the framework still feels like the same CRUD/media controller system externally, but the handler internals are now more modular than older docs described.
