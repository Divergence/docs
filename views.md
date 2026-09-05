### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Views
Divergence uses the [Twig Template Engine](https://twig.symfony.com/) as its primary template engine. It is recommended that you reference Twig documentation for details on how to use Twig templates. This documentation will discuss helpers for your controllers that allow you to serve templates at a moment's notice.

#### Architecture
A typical Divergence project will have a `views` folder containing all the Twig templates.

`TwigBuilder`:

- loads templates from `App::$App->ApplicationPath.'/views'`
- uses `Twig\Environment`
- enables `strict_variables`
- adds `Twig\Extension\StringLoaderExtension`

## Responding with a Template
To generate a response using a Twig template:

```php
use Divergence\Responders\Response;
use Divergence\Responders\TwigBuilder;

return new Response(new TwigBuilder('blog/posts.twig', [
    'BlogPosts' => $BlogPosts,
]));
```

If you are already inside a `RequestHandler`, use its response helper:

```php
$this->responseBuilder = \Divergence\Responders\TwigBuilder::class;

return $this->respond('blog/posts.twig', [
    'BlogPosts' => $BlogPosts,
]);
```

### Injecting Data - Hello World
```php
return new Response(new TwigBuilder('helloworld.twig', [
    'text' => 'Hello World'
]));
```

```twig
{{ text }}
```

Will print the classic:
`Hello World`

## Template Lookup Rules

Templates are resolved relative to your app's `views/` directory, so:

- `home.twig` maps to `views/home.twig`
- `blog/posts.twig` maps to `views/blog/posts.twig`

Also note that `RecordsRequestHandler` still uses model nouns to derive HTML template names when you are not in JSON mode, so `$singularNoun` and `$pluralNoun` still matter even in current Divergence.

The records controller uses response IDs such as `tags`, `tagEdit`, and `tagSaved` for a model with nouns `tag` and `tags`. Those are literal template names; the builder does not append `.twig` for you. Either provide templates with those names or customize the handler's naming.

## Template Data

Pass the values the template actually uses. With `strict_variables` enabled, a missing value can throw instead of silently becoming an empty string. Use Twig's `is defined` or `default` only when absence is part of the intended input.

```twig
{% for post in BlogPosts %}
    <h2>{{ post.Title }}</h2>
{% else %}
    <p>No posts yet.</p>
{% endfor %}
```

Twig's normal HTML escaping applies. Don't use `raw` for user-supplied text just to make formatting work. `StringLoaderExtension` is available, but rendering user input as a Twig template is a different operation from displaying it as data.

The default builder creates a Twig environment when it builds the response. It enables strict variables but does not configure a persistent compiled-template cache or add your application's helper functions. Use a custom response builder if you need those settings.
