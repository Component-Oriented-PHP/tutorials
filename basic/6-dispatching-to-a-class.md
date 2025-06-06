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
