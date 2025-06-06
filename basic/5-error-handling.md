---
previous: 4-routing-package
next: 6-dispatching-to-a-class
---

# Error Handling

Great! Now that we have a working router, let's add some error handling to our application.

In the front controller (`public/index.php`), change the following:

```php
// from
require_once __DIR__ . '/../vendor/autoload.php';

// to
require_once __DIR__ . '/../vendor/autoload'; # we are creating a deliberate typo by removing .php
```

> 🔥NOTE: We are intentionally creating a typo here.

Open the browser and see what happens. You should see a "This page isn’t working" 500 error. Now, we know what the problem is. But what if we did not? This is where error handling comes in handy.

Just above composer require, add the following:

```php
error_reporting(E_ALL);
ini_set('display_errors', '1');

require_once __DIR__ . '/../vendor/autoload'; # keep the typo
```

Now go check the browser again. Now you should be seeing something like this:

```bash
Warning: require_once(I:\SOFTWARE\cophp\basic-application\public/../vendor/autoload): Failed to open stream: No such file or directory in I:\SOFTWARE\cophp\basic-application\public\index.php on line 12
```

This clearly states that the file we are requiring does not exist. Fix the typo and re-check, the error should be gone now.

This is good enough. In development, I want to see all the errors, warnings, and notices so I can fix them. But... TBH, that default error message is pretty ugly and not as helpful as the ones you may have seen in Laravel or Symfony or CodeIgniter or any other framework. We can do much, much better.

To achieve a similar pretty error messages, we can use some composer packages. I have used `filp/whoops`, `tracy/tracy`, and `symfony/error-handler` a lot. But we will use `filp/whoops` in this tutorial and move on to `tracy/tracy` in the advanced tutorial. But feel free to use whatever you like.

In your project root, run:

```bash
composer require filp/whoops
```

Now, open `public/index.php` and add the following code:

```php
<?php

declare(strict_types=1);

use Aura\Router\RouterContainer;
use Laminas\Diactoros\Response\HtmlResponse;
use Laminas\Diactoros\ServerRequestFactory;

require_once __DIR__ . '/../vendor/autoload.php';

// Initialize Whoops for better error handling
$whoops = new \Whoops\Run;
$whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
$whoops->register();

$routerContainer = new RouterContainer();

$request = ServerRequestFactory::fromGlobals(
    $_SERVER,
    $_GET,
    $_POST,
    $_COOKIE,
    $_FILES
);

$map = $routerContainer->getMap();

$map->get('home', '/', function () {
    return new HtmlResponse('This tuts rocks!!!');
});

$map->get('about', '/about', function () {
    return new HtmlResponse('This is the about page!!!');
});

$map->get('contact', '/contact', function () {
    return new HtmlResponse('This is the contact page!!!');
});

$map->get('blog_slug', '/blog/{slug}', function ($request) {
    // Get the slug from the route attributes
    $slug = (string) $request->getAttribute('slug');
    $html = 'This is the blog page for ' . $slug . '!!!';
    return new HtmlResponse($html);
});

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

I removed error reporting and ini_set('display_errors', '1') because we will be using Whoops now to handle errors.

Notice this part:

```php
// Initialize Whoops for better error handling
$whoops = new \Whoops\Run;
$whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
$whoops->register();
```

This code registers the Whoops error handler. Now let's experiment.

Replace:

```php
// this
$routerContainer = new RouterContainer();

// with this (temporarily)
$routerContainer = new NonExistentRouterContainer();
```

Go to the browser and enjoy a beautiful error reporting.

Revert the temporary change and move on.

Pay attention - **we do not want error reporting to occur in production!** Why? Because it can expose sensitive information about your application and potentially server configurations.

So, modify the front controller:

```php

// change this
$whoops = new \Whoops\Run;
$whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
$whoops->register();

// to this
if ($environment === 'development') {
    $whoops = new \Whoops\Run;
    $whoops->pushHandler(new \Whoops\Handler\PrettyPageHandler);
    $whoops->register();
}
```

But we haven't defined the environment variable yet. Have we? Define a variable `$environment` and set it to `development` in the front controller just above composer autoload.

```php
// ...

$environment = 'development';

require_once __DIR__ . '/../vendor/autoload.php';

// ...
```

Now, you can experiment setting $environment to "production" and "development" and replacing $routerContainer with NonExistentRouterContainer (like we did before) to see how error handling works in production vs development.

BUT... THERE IS A MAJOR ISSUE WITH THIS APPROACH OF SETTING CONFIGURATIONS.

You see, it is not the right approach to be setting $environment directly in front controller. It is more of a configuration than a code. What we should instead be doing is either to have a config file containing environment variable or a .env file in project root containing the value of environment variable.

So, which approach should we take?

While both methods work, the industry standard and best practice for modern applications is to use a .env file.

Your application's code should not contain configuration that changes between environments (like development, testing, production). A .env file keeps your environment-specific settings completely separate from your codebase.

The .env file is meant to be stored on the server but never committed to your version control (like Git). This prevents sensitive information like database passwords or API keys from being exposed in your repository. You'll typically commit a .env.example file as a template for other developers.

So, let's refactor our application to use .env files. We'll use a very popular package that can read from .env files. I am going to install `dikki/dotenv` (my own package that I ripped from CodeIgniter 4's Dotenv class and launched as a composer package). You can use `vlucas/phpdotenv` instead if you like. It's a very popular package and I have used it in many projects.

In your project root, run:

```bash
composer require dikki/dotenv
```

Now, let's refactor our front controller.

```php
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

$map->get('home', '/', function () {
    return new HtmlResponse('This tuts rocks!!!');
});

$map->get('about', '/about', function () {
    return new HtmlResponse('This is the about page!!!');
});

$map->get('contact', '/contact', function () {
    return new HtmlResponse('This is the contact page!!!');
});

$map->get('blog_slug', '/blog/{slug}', function ($request) {
    // Get the slug from the route attributes
    $slug = (string) $request->getAttribute('slug');
    $html = 'This is the blog page for ' . $slug . '!!!';
    return new HtmlResponse($html);
});

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

I removed the `$environment` variable since we will be reading the environment from the .env file using getenv() function.

Create a .env file in your project root with the following content:

```ini
APP_ENV=development
```

Now go on and experiment setting APP_ENV to "production" and "development" and replacing RouterContainer with NonExistentRouterContainer (like we did before) to see how error handling works in production vs development.

Look at how we register the DotEnv class in a single line.

```php
(new DotEnv(__DIR__ . '/../'))->load();
```

All it does is instantiate the DotEnv class and call the load() method to load the .env file. We passed the root dir where the .env file is located. That's it. You can either use $_ENV or getenv() to read the environment variables set in the .env file. Note that it is recommended to keep the variables in capital letters. Why?

I don't know. I believe it's a long-standing convention that comes from the world of Unix and shell scripting. Using all-caps for environment variables makes them stand out visually, helping you instantly distinguish them from other variables in your code. It's a signal to developers that this value is coming from the external environment and is likely to be constant throughout the application's runtime.

> 🔒 Don't Forget to gitignore .env! The .env file should NEVER be committed to your version control system (like Git), as it may contain sensitive credentials in the future. Add `.env` in your .gitignore file.

Next lesson is on [dispatching the routes to a class (like controllers)](./6-dispatching-to-a-class.md)
