---
previous: 3-composer
next: 5-error-handling
---

# Routing 2: Installing a Routing Package

Head over to [packagist](https://packagist.org) and pay attention at the top center. You will see an important thing...

"Composer v1 support is coming to an end"... Oh... sorry... not this one...

You will see the search bar. Search for "Routing". You will see hundreds of routing packages that we can use in our project. But I will list only the ones I have personally used in my projects.

- [symfony/routing](https://packagist.org/packages/symfony/routing): the definition of perfection; It comes with attributes based routing that you use in Symfony framework like (#[Route('/')]) although you can use the old way to place routes in a single file as well.
- [nikic/fast-route](https://packagist.org/packages/nikic/fast-route): this is the one Patrick Louys used in his original No Framework Tutorial. I love this as it is quite fast.
- [league/route](https://packagist.org/packages/league/route): THIS IS MY PERSONAL FAVORITE routing package as it comes with an inbuilt-middleware support, PSR compliance, support for Dependency Injection. In fact, it is built on top of nikic/fast-route and is like a wrapper around it.
- [pecee/simple-router](https://github.com/skipperbent/simple-php-router): If you use Laravel, this is the one you'd love the most. It comes with its own Request and Response classes, middleware support, and more.
- [aura/router](https://packagist.org/packages/aura/router): We will use this one for this tutorial.
- [nette/routing](https://packagist.org/packages/nette/routing): Haven't used it outside Nette Framework so you may need to check its docs to learn more about it. (or just move on to the router I use)
- [aplus/routing](https://packagist.org/packages/aplus/routing): used it only twice but included it to showoff my experience with routing packages 😏

But I suggest you use the one you like the most and that actually serves the purpose of your project. I mostly use league/route and pecee/simple-router.

In this tutorial (basic) we will use Aura Router and then move on to use League Route in advanced tutorial. Why? Because I said so. But you're free to use whatever router you want. You need to read their docs for proper direction.

Head over to your project root and run `composer require aura/router`. Wait for it to finish and open composer.json.

> NOTE: Here, aura is the vendor name and router is the package name. A vendor can have multiple packages like aura/session or aura/di.

You should see this:

```json
{
    "config": {
        "sort-packages": true,
        "optimize-autoloader": true
    },
    "require": {
        "aura/router": "^3.4"
    }
}
```

As you can see, composer will add the routing package to your `require` section. It installed the package and added vendor dir and composer.lock files as we discussed in last lesson. Composer automatically installed the latest version of the package. You can also install an explicit version using `composer require vendor-name/package-name:version`.

Now, we need to update the front controller to use this router instead of our vanilla one.

```php
<?php

declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';

$routerContainer = new \Aura\Router\RouterContainer();

$map = $routerContainer->getMap();

$map->get('home', '/', function () {
    echo 'This tuts rocks!!!';
});

$map->get('about', '/about', function () {
    echo 'This is the about page';
});

$map->get('contact', '/contact', function () {
    echo 'This is the contact page';
});
```

I removed the vanilla routing code and added the routing code from the package. Here, we included the composer's autoload.php file to ensure we can use the installed packages in our app (here, aura/router). Then we instantiated the router and added the routes to it. Now, let's get our app up and running.

When you head over to the browser, you will see an empty page. No response is there when you to go `http://localhost:8080`. What's going on?

It so happens that we have only defined the routes. We've created a map telling the router that if a request for `/` comes in, it should execute our function and return a response (e.g. "This tuts rocks!!").

However, the router has not yet dispatched a request. What does that mean? It simply means that the router has not yet been told to actually look at the current request and match it against the map. That process is called dispatching.

## Dispatching a Request

In simplest possible words, dispatching is the process where the router takes the current incoming request (e.g., the URL path `/about` and the `GET` method) and tries to find a matching route in the map we defined. If it finds a match, it "dispatches" to the handler we provided (i.e., our function() {...} which handles what is returned).

So, how do we do this?

Well, to begin with, you need to read the documentation for respective routing package you're using. Aura has a great documentation. Let's continue.

Run `composer require laminas/laminas-diactoros` in root and then edit the `public/index.php` file (I will explain below):

```php
<?php

declare(strict_types=1);

require_once __DIR__ . '/../vendor/autoload.php';

$routerContainer = new \Aura\Router\RouterContainer();

$request = \Laminas\Diactoros\ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);

$map = $routerContainer->getMap();

$map->get('home', '/', function () {
    echo 'This tuts rocks!!!';
});

$map->get('about', '/about', function () {
    echo 'This is the about page!!!';
});

$map->get('contact', '/contact', function () {
    echo 'This is the contact page!!!';
});

$map->get('blog_slug', '/blog/{slug}', function ($request, $route) {
    // Get the slug from the route attributes
    $slug = (string) $route->attributes['slug'];
    echo 'This is the blog page for ' . $slug;
});

// match the request
$matcher = $routerContainer->getMatcher();
$route = $matcher->match($request);

// if no route registered for current path
if (!$route) {
    http_response_code(404);
    echo '404';
    exit;
}

// dispatch the route
$handler = $route->handler; // get the callable function we defined for each route
$handler($request, $route); // run the function and pass the request object to it for further usage in the function
```

Go and check the browser for all the defined routes and you SHOULD be able to see the defined responses.

So, what's hapenning here?

In short, we are creating a standardized representation of the incoming web request and then asking the router to match it against our defined maps. If it finds a match for the URI, it executes the corresponding function we wrote. If not, it shows 404 error we defined.

Focus on this:

```php
$request = \Laminas\Diactoros\ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);
```

This is a very important step in modern PHP development. Instead of working directly with messy and insecure global variables like `$_GET` and `$_SERVER`, we use a library (here, `laminas/laminas-diactoros`; but there are packages like `symfony/http-foundation`, or `nette/http1 for the same; I am more used to laminas diactoros, so I used it here) to bundle all of that information into a clean, standardized Request object.

- **Why, you ask?** This object is predictable and easy to work with. Any modern PHP component, not just our router, can understand its structure. This is part of a standard called PSR-7.
- `ServerRequestFactory::fromGlobals(...)` is a handy helper that does the dirty work of reading all those global variables and creating the `$request` object for us that we can use in callable functions/controllers/handlers.

Let's shift our focus to a dynamic route.

```php
$map->get('blog_slug', '/blog/{slug}', function ($request, $route) {
    // Get the slug from the route attributes
    $slug = (string) $route->attributes['slug'];
    echo 'This is the blog page for ' . $slug;
});
```

The cool part here is the `/{slug}`. This is a placeholder. It tells the router to match any URL that starts with `/blog/` followed by some value. That value will be captured and named `slug`. This is how you create dynamic pages, like `yourhotsite.com/blog/my-first-post` or `yoursexysite.com/posts/learning-php` instead of using query parameters like `somehub.com/video.php?id=1234`.

Now, let's check the matching process for the incoming request.

```php
// match the request
$matcher = $routerContainer->getMatcher();
$route = $matcher->match($request);
```

This is the heart of the dispatching process!

- `$routerContainer->getMatcher()`: We get the "Matcher" service from our router. Its only job is to find a match for the incoming request.
- `$matcher->match($request)`: We pass our shiny new `$request` object to the matcher. It looks at the request's URL path (e.g., `/about` or `/blog/my-first-post`) and HTTP method (e.g., `GET`) and compares it against all the routes we have defined in the `$map`.
- If it finds a match, it returns a `$route` object containing all the information about the matched route (including its name, path, and handler/callable-function we defined in the map). If no match is found, it returns false or null (and this false/null value is used to return a 404 error).

```php
// if no route registered for current path
if (!$route) {
    http_response_code(404);
    echo '404';
    exit;
}
```

Self explanatory this 404 part is. FOCUS BELOW INSTEAD.

```php
// dispatch the route
$handler = $route->handler; // get the callable function we defined
$handler($request, $route); // run the function
```

This is the grand finale! If a route was found:

- `$handler = $route->handler;`: We get the handler that you defined for that specific route. In our case, this is the callable function() {...}.
- `$handler($request, $route);`: We execute that function! The router is smart enough to pass the $request object and the $route object into your handler function. This is why you can access the slug inside the blog handler using `$route->attributes['slug']`. It gives your route's code access to all the information about the incoming request and the matched route.

And that's it! We've successfully connected an incoming request to a specific piece of your code. Awesome job! Go get me some tea now.

Anyway, before going on to the next chapter (error handling), there is ONE MINOR IMPROVEMENT (not necessary, but this is my tutorial... I will do whatever the hell I want) that can be made to the front controller.

See how we are passing $request and $route both to $handler and then passing both in callble function is blog map? What if we could pass only $request and not $route? Well, if we were using league/route package, it could have done that for us. But we will do it manually here.

Modify the front controller.
