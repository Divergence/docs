### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Architecture
## Request Respond Emit

Divergence make use of Psr7's `ServerRequest::fromGlobals()` method to generate a `Request` object. A Request object reads the build in PHP super globals like `$_GET`, `$_POST`, `$_REQUEST` but it also builds on that by providing a unified interface for anything else you might want to do such as get headers, cookier, or read a file stream attached to the request. In response to this Request we actually generate a Response object. Like the Request object the Response object has everything a Response might want to do. Generally this could mean setting cookies, headers, and content. The complexity of a Response object could be very low such as in the case of an empty response where you might just return an HTTP status code and nothing else. Or it could be highly complex such as when using the `HTTP_RANGE` feature for seeking a large file. The response object 

Finally the Response object is passed to an **Emitter**. The emitter actually does the work of doing the response. It has a uniform way of setting all headers, cookies, and output streams. The emitter also has some tricks to avoid sending data once the connection has been terminated saving you bandwidth and server processing time. You generally don't need to ever bother touching the emitter since you can create any HTTP response with the provided Response object. The `Response` object `implements ResponseInterface` another Psr7 provided configuration.

Generally all Responses are created by Controllers. Divergence comes with several built in standard Controllers to save you time.

By default Divergence will always `start` evaluating a request by instantiating the class `App` and then running ``$app->handleRequest()` which should run `ResponseInterface SiteRequestHandler->handle` and then pass the Response to the Emitter. Handle always returns a $response. You'll notice by looking at the built in `RecordsRequestHandler`` and `MediaRequestHandler` that all the end points handled by these classes return a `ResponseInterface` interface.

Using this architecture. Any code written for Divergence would easily be portable to any other framework that implements Psr7
