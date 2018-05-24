### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Views
Divergence uses the [Dwoo Template Engine](https://github.com/dwoo-project/dwoo) as it's primary template engine. It is recommended that you reference Dwoo documentation for details on how to use Dwoo templates. This documentation will discuss helpers for your controllers that will allow you to serve templates at a moment's notice.

#### Architecture
A typical Divergence project will have a `views/dwoo` folder containing all the Dwoo templates. This is so that if you decide you wish to use Twig or something else you can simply make a folder in `views/` for that template engine instead of having to share a single folder and without having to create a new one in the project root.

## Responding with a Template
You can always use `\Divergence\Controllers\RequestHandler::respond('template.tpl')` from anywhere. It will look for the file in `views/dwoo/template.tpl`.

If you are inside of a controller it's best practice to use `static::respond('template.tpl')` which is much less verbose.

### Injecting Data - Hello World
```php
static::respond('helloworld.tpl', [
    'text'=>'Hello World'
]);
```

```smarty
{$data.text}
```

Will print the classic
`Hello World`


