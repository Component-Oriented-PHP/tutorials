---
previous: 5-error-handling
next: 7-inversion-of-control
---

# Dispatching to a Class

Look at our front controller. We have only a few routes right now with nothing more than a few lines of responses. But this is never the case in real world applications. There are tens and hundreds of routes involved and each response returned contains hundreds of lines of code behind it.

The current approach of front controller works only if the application is quite small. If the application is large, it is better to dispatch the route to a class like a controller or a handler. This way, we can keep our front controller simple and focus on the routing and dispatching logic than the main business/application logic.

I believe you have used some kind of framework before. Then you know that they contain classes called controllers.

Let's take a look at Laravel.

```php
Route::get('/some-route', 'SomeController@someMethod');
```

In Laravel, you typically create a route, and pass the controller name and the method name to the route so that when that particular route is hit, the defined method from the defined controller will be executed.

Now let's see how we can do the same thing in our application.

In the root of your project, create a folder called `src` (this will contain the whole application codebase) and inside it create `src/Controller` directory.

Create a HomeController Class -> `src/Controller/HomeController.php`

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;

class HomeController
{
    public function index()
    {
        return new HtmlResponse('This tuts rocks!!! This comes from Home Controller');
    }
}
```

Now, change the route map.

```php

// this
$map->get('home', '/', function () {
    return new HtmlResponse('This tuts rocks!!!');
});

// to this
$map->get('home', '/', function () {
    return (new \App\Controller\HomeController())->index();
});

```

At this stage, if you refresh browser you MUST get this error from Whoops: `Class "App\Controller\HomeController" not found`. This is because the class is not imported included/required yet anywhere in our front controller. However, we are not going to manually import all the classes. Composer will handle that for us.

Our HomeController exists, but Composer's autoloader, which is responsible for loading classes on demand, has no idea where to find the App namespace. We need to tell it that our App namespace lives inside the src directory.

We do this by configuring PSR-4 autoloading in our composer.json file.

Open composer.json and add an autoload section:

```json
{
    "config": {
        "sort-packages": true,
        "optimize-autoloader": true
    },
    "require": {
        "aura/router": "^3.4",
        "dikki/dotenv": "^2.0",
        "filp/whoops": "^2.18",
        "laminas/laminas-diactoros": "^3.6"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

This configuration tells Composer that any class starting with the `App\` namespace prefix can be found in the `src/` directory.

After saving the file, we need to tell Composer to update its autoloader file based on this new information. Run the following command in your project root: `composer dumpautoload`

Go to the browser and you must see: `This tuts rocks!!! This comes from Home Controller` on visiting `/`.

Now now, this way of routing isn't as pretty as you can set routes in Symfony or Laravel. So what do we do? Think with me for a solution (don't just follow, think critically).

We can create a $routes variable containing route definitions and then use it to route to appropriate controller classes. Let's see...

Change the front controller code to this:

```php
<?php

declare(strict_types=1);

use Aura\Router\RouterContainer;
use Dikki\DotEnv\DotEnv;
use Laminas\Diactoros\Response\HtmlResponse;
use Laminas\Diactoros\ServerRequestFactory;

$routes = [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index']
];

require_once __DIR__ . '/../vendor/autoload.php';

(new DotEnv(__DIR__ . '/../'))->load();

// Initialize Whoops for better error handling
if (getenv('APP_ENV') === 'development') {
    $whoops = new \Whoops\Run;
    $whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
    $whoops->register();
}

$routerContainer = new RouterContainer();

$request = ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);

$map = $routerContainer->getMap();

//map the route definitions
foreach ($routes as $name => $route) {
    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];

    $map->$requestMethod($name, $path, function ($request) use ($controller, $method) {
        return (new $controller($request))->$method(); // frameworks pass $request to method directly, will change it below
    });
}

// match the request
$matcher = $routerContainer->getMatcher();
$route = $matcher->match($request);

// if no route registered for current path, it can be a 404 error, or 405, 406, etc.
if (!$route) {
    // get the first of the best-available non-matched routes
    $failedRoute = $matcher->getFailedRoute();

    // we need to handle the failed route
    $response = match ($failedRoute->failedRule) {
        // if method was not allowed (e.g., received GET request on a POST route)
        'Aura\Router\Rule\Allows' => (function () {
                // 405 METHOD NOT ALLOWED
                return new HtmlResponse('405 METHOD NOT ALLOWED!!!', 405);
            })(),
        // if content type was not accepted (e.g. received HTML request on a JSON route)
        'Aura\Router\Rule\Accepts' => (function () {
                // 406 NOT ACCEPTABLE
                return new HtmlResponse('406 NOT ACCEPTABLE!!!', 406);
            })(),
        // handle as a 404 error for other cases
        default => new HtmlResponse('404 NOT FOUND!!!', 404)
    };
} else {
    // A route was found, so let's handle the "happy path"

    // add route attributes to the request
    foreach ($route->attributes as $key => $val) {
        $request = $request->withAttribute($key, $val);
    }

    // dispatch the route and get the response
    $handler = $route->handler;
    $response = $handler($request); // This executes our closure and gets the HtmlResponse object
}

// emit the response
foreach ($response->getHeaders() as $name => $values) {
    foreach ($values as $value) {
        header(sprintf('%s: %s', $name, $value), false);
    }
}
http_response_code($response->getStatusCode());
echo $response->getBody();
```

As you can see, I have removed the other useless routes for now. We will add dynamic routes later. But for now, let's see how this works.

I created a $routes array configuration with route name, request method, route path, and controller and method to dispatch to.

Then, I loop through this array. Inside the loop, I'm doing a few key things:

- The line `$handler = explode('::', $route[2]);` takes our string (e.g., `\App\Controller\HomeController::index`) and splits it into two parts at the `::`, giving us the controller's class name and the method's name.
- We then map the route just like before, but the magic is in the handler—the closure.
- Notice the `use ($controller, $method)` part of our closure. When Aura Router matches a route, it will execute this function. The use keyword allows the closure to access the `$controller` and `$method` variables from the parent scope (our foreach loop).
- Inside the closure, we instantiate the controller dynamically with `new $controller($request)` and then call the correct method on it with `$controllerInstance->$method()`.
- Notice that I passed the `$request` object to the controller's constructor. This is what allows us to use the `$request` object in the controller's methods like we did earlier in the blog route.

This is a huge improvement! 🎉 Our front controller is now a pure dispatcher. It doesn't know or care about what happens inside the controllers. Its only job is to map incoming requests to the correct controller action based on a clean configuration array.

Before we proceed, we need to update our HomeController to accept the request object in its constructor method, since our new dispatcher is now passing it. This is a good practice as it allows our controller methods to access information about the incoming request.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(private ServerRequestInterface $request) // not the best way, will change it below
    {
    }

    public function index()
    {
        return new HtmlResponse('This tuts rocks!!! This comes from Home Controller');
    }
}
```

Notice that I added a type hint for `Psr\Http\Message\ServerRequestInterface` for the `$request property.

Back in our front controller, we use the `Laminas\Diactoros\ServerRequestFactory` to create a request object, which is an instance of `Laminas\Diactoros\ServerRequest`. So in our controller, why do we type-hint against the `Psr\Http\Message\ServerRequestInterface` instead of that concrete class? This is a core principle of modern application design. By depending on an interface (which is like a contract for how a class should behave) rather than a specific implementation, we decouple our controller from the component that creates the request. If we ever decide to switch from `laminas-diactoros` to another library that also follows the PSR-7 standard, we wouldn't have to change a single line of code in our controller!

Now, you may ask, but aren't we using HtmlResponse class directly? Won't we have to replace it with something else? Yes... and to answer that we are covering inversion of control in the next chapter.

Also, you might have noticed the private keyword directly in the constructor's parameter list. This is a nifty feature from PHP 8 called Constructor Property Promotion. It's just a more concise way of writing this:

```php
// The long-form way (common before PHP 8)
class HomeController
{
    private ServerRequestInterface $request;

    public function __construct(ServerRequestInterface $request)
    {
        $this->request = $request;
    }

    // ... methods
}
```

By adding the visibility keyword (private, public, or protected), PHP automatically creates the property and assigns the incoming value to it. It's cleaner this way.

One more thing... if we were using League Route for example, or symfony routes, (or any framework) they'd have passed `$request` directly to the method being called (e.g. `HomeController->index($request)`), so we wouldn't have needed to pass it to the constructor. But I did it here so that I could tell you about the Constructor Property Promotion feature. In the later parts we will modify our codebase to follow the best practices, pass `$request` object to the method instead and save constructor for services/libraries we use.

> NOTES:
>
> - You may ask, what to pass in the controller and method then? Why pass `$request` in method not controller?
> - You typically inject long-lived services or dependencies into the constructor. These are objects that the controller will need to perform many of its tasks. Examples: a database connection, a template rendering engine, or a logging service. These services don't change from one request to the next.
> - The `$request` object is different. It represents the current, specific incoming HTTP request. It's highly dynamic and unique to each call (in methods). Therefore, it makes more logical sense to pass it directly to the method that is handling that specific request rather than the whole constructor.

Let's change the Home controller and the front controller.

```php
// HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse('This tuts rocks!!! This comes from Home Controller!');
    }
}

// in public/index.php

# map the route definitions
$routes = require_once __DIR__ . '/../config/routes.php';
foreach ($routes as $name => $route) {
    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];

    $map->$requestMethod($name, $path, function ($request) use ($controller, $method) {
        return (new $controller())->$method($request);
    });
}
```

Go to the browser and see if it works (it should).

However, there is a minor issue in the front controller. You see how we are defining routes (a type of configuration) directly in the `public/index.php` file?

Why don't we separate the routes into a diff file altogether? Let's create a config folder and create a routes.php file in it.

```php
// config/routes.php
<?php

return [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index']
];
```

Update `public/index.php` to this:

```php
// public/index.php
<?php

declare(strict_types=1);

use Aura\Router\RouterContainer;
use Dikki\DotEnv\DotEnv;
use Laminas\Diactoros\Response\HtmlResponse;
use Laminas\Diactoros\ServerRequestFactory;

require_once __DIR__ . '/../vendor/autoload.php';

(new DotEnv(__DIR__ . '/../'))->load();

// Initialize Whoops for better error handling
if (getenv('APP_ENV') === 'development') {
    $whoops = new \Whoops\Run;
    $whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
    $whoops->register();
}

$routerContainer = new RouterContainer();

$request = ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);

$map = $routerContainer->getMap();

//map the route definitions
$routes = require_once __DIR__ . '/../config/routes.php';
foreach ($routes as $name => $route) {
    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];

    $map->$requestMethod($name, $path, function ($request) use ($controller, $method) {
        return (new $controller())->$method($request);
    });
}

// match the request
$matcher = $routerContainer->getMatcher();
$route = $matcher->match($request);

// if no route registered for current path, it can be a 404 error, or 405, 406, etc.
if (!$route) {
    // get the first of the best-available non-matched routes
    $failedRoute = $matcher->getFailedRoute();

    // we need to handle the failed route
    $response = match ($failedRoute->failedRule) {
        // if method was not allowed (e.g., received GET request on a POST route)
        'Aura\Router\Rule\Allows' => (function () {
                // 405 METHOD NOT ALLOWED
                return new HtmlResponse('405 METHOD NOT ALLOWED!!!', 405);
            })(),
        // if content type was not accepted (e.g. received HTML request on a JSON route)
        'Aura\Router\Rule\Accepts' => (function () {
                // 406 NOT ACCEPTABLE
                return new HtmlResponse('406 NOT ACCEPTABLE!!!', 406);
            })(),
        // handle as a 404 error for other cases
        default => new HtmlResponse('404 NOT FOUND!!!', 404)
    };
} else {
    // A route was found, so let's handle the "happy path"

    // add route attributes to the request
    foreach ($route->attributes as $key => $val) {
        $request = $request->withAttribute($key, $val);
    }

    // dispatch the route and get the response
    $handler = $route->handler;
    $response = $handler($request); // This executes our closure and gets the HtmlResponse object
}

// emit the response
foreach ($response->getHeaders() as $name => $values) {
    foreach ($values as $value) {
        header(sprintf('%s: %s', $name, $value), false);
    }
}
http_response_code($response->getStatusCode());
echo $response->getBody();
```

Check the browser, you should see the same thing as before. But now, we have separated the route definitions from the front controller.

In the next lesson, you will learn an important concept called ["Inversion of Control"](./7-inversion-of-control.md).
