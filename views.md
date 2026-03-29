### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Views
Divergence uses the [Twig Template Engine](https://twig.symfony.com/) as its primary template engine. It is recommended that you reference Twig documentation for details on how to use Twig templates. This documentation will discuss helpers for your controllers that allow you to serve templates at a moment's notice.

#### Architecture
A typical Divergence project will have a `views` folder containing all the Twig templates.

The current `TwigBuilder`:

- loads templates from `App::$App->ApplicationPath.'/views'`
- uses `Twig\Environment`
- enables `strict_variables`
- adds `Twig\Extension\StringLoaderExtension`

## Responding with a Template
To generate a response using a Twig template:

```php
return new Response(new TwigBuilder('blog/posts.twig', [
    'BlogPosts' => $BlogPosts,
    'isLoggedIn' => App::$App->is_loggedin(),
    'Sidebar' => $this->getSidebarData(),
    'Limit' => static::LIMIT,
    'Total' => DB::foundRows(),
]));
```

If you are already inside a `RequestHandler`, the more common current style is:

```php
$this->responseBuilder = \Divergence\Responders\TwigBuilder::class;

return $this->respond('blog/posts.twig', [
    'BlogPosts' => $BlogPosts,
    'Sidebar' => $this->getSidebarData(),
]);
```

### Injecting Data - Hello World
```php
new Response(new TwigBuilder('helloworld.twig', [
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
