### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Views
Divergence uses the [Twig Template Engine](https://twig.symfony.com/) as it's primary template engine. It is recommended that you reference Dwoo documentation for details on how to use Dwoo templates. This documentation will discuss helpers for your controllers that will allow you to serve templates at a moment's notice.

#### Architecture
A typical Divergence project will have a `views` folder containing all the Twig templates.

## Responding with a Template
To generate a Response use a Twig template like so.
```php
return new Response(new TwigBuilder('blog/posts.twig', [
    'BlogPosts' => $BlogPosts,
    'isLoggedIn' => App::$App->is_loggedin(),
    'Sidebar' => $this->getSidebarData(),
    'Limit' => static::LIMIT,
    'Total' => DB::foundRows(),
]));
```

### Injecting Data - Hello World
```php
new Response(new TwigBuilder('helloworld.twig', [
    'text'=>'Hello World'
]));
```

```twig
{{ text | raw }}
```

Will print the classic
`Hello World`


