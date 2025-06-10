---
previous: 11-controller-responses
next: 13-wrapup
---

# Middlewares

Let's create a REST API for our markdown pages. We already have a frontend displaying them, but what if we ever wanted to have an android app for our application? Wouldn't we need an API then?

> NOTES:
>
> - In advanced tutorial, we will be creating an advanced and versioned REST API. But here, I want to keep things simple for demonstration.
> - In this chapter, I will be using our CustomResponseInterface in the ApiController since it is the sweet spot between the other two approaches I discussed in the last chapter.

Create a new file `./src/Controller/Api/PageController.php`:

```php
// src/Controller/Api/PageController.php
<?php

declare(strict_types=1);

namespace App\Controller\Api;

use App\Library\Http\CustomResponseInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class PageController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private CustomResponseInterface $response
    ) {
    }

    public function index(): ResponseInterface
    {
        // get all pages
        $pages = $this->pageFetcher->fetchAll();

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $pages
        ]);
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        // get the slug
        $slug = $request->getAttribute('slug');

        // get the page
        $page = $this->pageFetcher->fetchSingle($slug);

        // if empty
        if (!$page) {
            return $this->response->json([
                'success' => false,
                'message' => 'Page not found'
            ], 404);
        }

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $page
        ]);
    }
}
```

Register the routes:

```php
<?php

return [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index'],
    // 'about' => ['get', '/about', '\App\Controller\AboutController::index'],
    'page' => ['get', '/{slug}', '\App\Controller\PageController::show'],

    'api.page' => ['get', '/api/page', '\App\Controller\Api\PageController::index'],
    'api.page.show' => ['get', '/api/page/{slug}', '\App\Controller\Api\PageController::show'],
];
```

Now, go and visit <http://localhost:8080/api/page> and <http://localhost:8080/api/page/about>. You should see the JSON API responses.

I doubt I need to explain what's happening in the Api PageController since we have been through this multiple times. But briefly, we asked DI for a class implementing CustomResponseInterface. Then we fetched the data (single or all pages) and returned the response with `application/json` content type (JsonResponse sets that automatically).

But, as always, there is an issue with our simple API. It's not protected. What if we wanted only our apps or the apps having some secret to use the endpoints?

We need to add authentication to our app. Let's see a simple approach.

```php
// src/Controller/Api/PageController.php
<?php

declare(strict_types=1);

namespace App\Controller\Api;

use App\Library\Http\CustomResponseInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class PageController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private CustomResponseInterface $response
    ) {
    }

    public function index(ServerRequestInterface $request): ResponseInterface
    {
        // Authenticate the endpoint. If it returns a response, return it immediately.
        if ($authResponse = $this->authenticate($request, $this->response)) {
            return $authResponse;
        }

        // This code will only run if authentication was successful
        $pages = $this->pageFetcher->fetchAll();

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $pages
        ]);
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        // Authenticate the endpoint. If it returns a response, return it immediately.
        if ($authResponse = $this->authenticate($request, $this->response)) {
            return $authResponse;
        }

        // This code will only run if authentication was successful
        $slug = $request->getAttribute('slug');
        $page = $this->pageFetcher->fetchSingle($slug);

        if (!$page) {
            return $this->response->json([
                'success' => false,
                'message' => 'Page not found'
            ], 404);
        }

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $page
        ]);
    }

    private function authenticate(ServerRequestInterface $request, CustomResponseInterface $response)
    {

        // x-api-key is set
        if (!$request->hasHeader('x-api-key')) {
            return $response->json([
                'success' => false,
                'message' => 'Missing API key'
            ], 401);
        }

        // x-api-key is valid
        if ($request->getHeaderLine('x-api-key') !== '1234567890') {
            return $response->json([
                'success' => false,
                'message' => 'Invalid API key'
            ], 401);
        }

        return null;
    }
}
```

What happened here?

- first I created a separate private method `authenticate`.
  - In the authenticate method, first we are checking if the incoming request (from the user) comes with the api key inside X-API-KEY header... if not we just return a 401 UNAUTHORIZED ERROR. **NOTE**: This is not the best way to secure an API (we will use JWT TOKENS IN ADVANCED TUTORIAL) but will do for now.
  - If an api key was provided inside X-API-KEY header, we check (hardcoded for now) if it is equal to 1234567890. If not, we return a 401 UNAUTHORIZED ERROR.
  - If the api key is valid, we return null (no error).

- second, we modify both methods, index and show, checks if running the authenticate method returns any response (remember, the method will either return null or a 401 response). If it does not (i.e. null), it simply means that the authentication was successful and we can continue with the rest of the code. If the authenticate method returns a response (i.e. 401), we just return that response (401 UNAUTHORIZED ERROR) and stop the execution of the rest of the code.

I know my boy, I know what's going on in your mind... you have come a long way... you already figured out that this is a bad approach to API authentication because we would need to duplicate the authenticate method in each controller. And even if we separate the authenticate method into a diff class, we'd be relying on adding auth to each method.

So, what do we do to solve this issue? Have you ever used CodeIgniter 4? It comes with something called filters. You typically create a filter class (e.g. an ApiAuthFilter that checks if API key was provided and is right). Then you can simply add this filter to a single, or multiple, or a group of routes.

Can we do the same in our app? Why not? But we will take a different approach, something called a Middleware.

I am sorry, but I need to get a bit into theory here for a brief moment.

## Explaining the Middlewares

Think of middlewares like security checkpoints at an airport. When you want to board a plane (reach your controller), you don't go directly to the gate. Instead, you pass through several checkpoints:

1. Ticket verification - Is your ticket valid?
2. Security screening - Do you have prohibited items?
3. Passport control - Are you authorized to travel?

Each checkpoint can either let you continue to the next one, or stop you entirely and send you back.
Middlewares work the same way with HTTP requests:

```txt
Request → Middleware 1 → Middleware 2 → Middleware 3 → Controller
         (Auth check)   (Rate limiting)  (Logging)    (Your code)
```

By using middlewares, we can technically stop a user from ever reaching our controller if they are not permitted to.

Now think about it for a sec... what benefits do we get from such a beautiful approach as compared to our authenticate method type API authentication? Well, first of course, we can reduce code repetion by using the same middleware (e.g. ApiAuthMiddleware.php class) in multiple API controllers. We can create any number of middlewares before/after a response is returned. We could limit how many requests a user can make (e.g. a rate limiter middleware). We could log every request (e.g. a logging middleware).

Each middleware can examine/modify the request before passing it along to the next middleware/controller, examine/modify the response after getting it back or stop the chain entirely by returning early. Note that middlewares can be run before or after a controller. So, technically, we can also modify the response returned by controller before it is sent to the users (see the diagram below).

```txt
Request → Middleware A (before) → Middleware B (before) → Controller → Middleware B (after) → Middleware A (after) → Response
```

Another benefit is that you can change the order some middlewares are run in. But it is important to note that the order of middlewares is important and can potentially affect how our application work. Example, you should always run auth middleware before the log middleware to ensure that you don't log an auth failure

## Implementing the Middlewares

First, let's begin by moving the API auth logic out of the controller.

Where should we place the auth logic?

- we can create a separate class to check for authentication? Hmmm... but a service/library is not supposed to return responses that our current authenticate method is doing. They are only supposed to tell us whether user is authenticated or not. So, that's (while possible) not the right approach. Moreover, even if we did use that, would we not need to make use of that class in multiple controllers/methods leading to duplication?
- keep the logic in controllers only? As stated, code duplication...
- move it to the location where we are directly instantiating controllers, checking for routes for granular control over responses? Maybe... and which place is that? Correct! The front controller (`public/index.php`)!

### Phase 1: Front Controller

Let's modify our `public/index.php` to support API auth and clean our Api PageController. Also, let's secure our secret by moving it to the .env file.

```env
APP_ENV=development
API_KEY=1234567890
```

```php
// public/index.php

//map the route definitions
$routes = $container->make(\App\Library\Config\ConfigInterface::class)->get('routes');
foreach ($routes as $name => $route) {
    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];

    // we need to make use of DI container to pass the necessary services to controller for usage
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container) {
        // api auth
        if (str_contains($path, '/api')) {
            // get response
            $response = $container->make(\App\Library\Http\CustomResponseInterface::class);

            // x-api-key is set
            if (!$request->hasHeader('x-api-key')) {
                return $response->json([
                    'success' => false,
                    'message' => 'Missing API key'
                ], 401);
            }

            // x-api-key is valid
            if ($request->getHeaderLine('x-api-key') !== '1234567890') {
                return $response->json([
                    'success' => false,
                    'message' => 'Invalid API key'
                ], 401);
            }
        }

        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
    });
}
```

But there is a minor issue. We do not want to keep editing the front controller in case we change the route from which APIs are served. Also, what if we only wanted a few API routes to be protected and others to be public? Why don't we make use of route config file instead? Let's see how.

```php
// config/routes.php
<?php

return [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index'],
    // 'about' => ['get', '/about', '\App\Controller\AboutController::index'],
    'page' => ['get', '/{slug}', '\App\Controller\PageController::show'],

    // all routes containing 'apiauth' will be protected and others will be public
    'api.page' => ['get', '/api/page', '\App\Controller\Api\PageController::index'], // public
    'api.page.show' => ['get', '/api/page/{slug}', '\App\Controller\Api\PageController::show', 'apiauth'], // protected
];

// public/index.php

    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];
    $filter = $route[3] ?? null; // <-- get the filter (if any)

    // we need to make use of DI container to pass the necessary services to controller for usage
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container, $path, $filter) { // <-- notice the change
        // api auth
        if ($filter === 'apiauth') { // <-- notice the change?
            // get response
            $response = $container->make(\App\Library\Http\CustomResponseInterface::class);

            // x-api-key is set
            if (!$request->hasHeader('x-api-key')) {
                return $response->json([
                    'success' => false,
                    'message' => 'Missing API key'
                ], 401);
            }

            // x-api-key is valid
            if ($request->getHeaderLine('x-api-key') !== getenv('API_KEY')) {
                return $response->json([
                    'success' => false,
                    'message' => 'Invalid API key'
                ], 401);
            }
        }

        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
    });
```

Don't forget to remove auth from Api\PageController.

```php
// src/Controller/Api/PageController.php
<?php

declare(strict_types=1);

namespace App\Controller\Api;

use App\Library\Http\CustomResponseInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class PageController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private CustomResponseInterface $response
    ) {
    }

    public function index(ServerRequestInterface $request): ResponseInterface
    {
        // This code will only run if authentication was successful
        $pages = $this->pageFetcher->fetchAll();

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $pages
        ]);
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        // This code will only run if authentication was successful
        $slug = $request->getAttribute('slug');
        $page = $this->pageFetcher->fetchSingle($slug);

        if (!$page) {
            return $this->response->json([
                'success' => false,
                'message' => 'Page not found'
            ], 404);
        }

        // create a response
        return $this->response->json([
            'success' => true,
            'data' => $page
        ]);
    }
}
```

Now, go and visit <http://localhost:8080/api/page> and <http://localhost:8080/api/page/about> to see if individual page api route is protected while the index api route is public. You can use Postman to test if passing X-API-KEY works (I will cover postman when we develop APIs in Advanced Tutorial).

Now that we have protected our APIs, I want you to go through the front controller code and let me know what issues you notice with our current approach. Got any? No? Slap yourself hard if you did not (pat your back if you did) and think again!

Found it? Good... here are the key issues:

- notice how bloated our route mapping has become. Even though it is not such a big issue and we can continue working with it, why not look for a better solution?
- we are mixing routing logic with auth logic. Where is the separation of concerns?
- what if we need to add more filters like CORS? Wouldn't the front controller get even more bloated than it already is?
- this approach is not as good as the Middleware approach we talked about. Why? Well, we cannot modify the response after it has been sent by the controller if we ever need to.

Take a look yourself:

```php
// illustrative code only; do not modify anything in your project

// Before (clean):
$map->get($name, $path, function ($request) use ($controller, $method, $container) {
    $controllerInstance = $container->make($controller);
    return $controllerInstance->$method($request);
});

// After (bloated):
$map->get($name, $path, function ($request) use ($controller, $method, $container, $path, $filter) {
    // Auth logic
    if ($filter === 'apiauth') { /* 10+ lines */ }
    
    // CORS logic (hypothetical)
    if ($filter === 'cors') { /* another 10+ lines */ }
    
    // Rate limiting (hypothetical)  
    if ($filter === 'ratelimit') { /* another 10+ lines */ }
    
    // Finally, the actual controller call
    $controllerInstance = $container->make($controller);
    return $controllerInstance->$method($request);
});
```

So, can you come up with a better approach? Well, how about we move the filters to a separate class of its own?

### Phase 2: Separating the Filters

Create a new class `./src/Service/Filters.php`:

```php
<?php

declare(strict_types=1);

namespace App\Service;

use App\Library\Http\CustomResponseInterface;
use Psr\Http\Message\ResponseInterface;

class Filters
{

    public function __construct(private CustomResponseInterface $response)
    {
    }

    public function runFilter(string $filter, $request): ?ResponseInterface
    {
        return match ($filter) {
            'apiauth' => $this->apiFilter($request),
            default => null,
        };
    }

    private function apiFilter($request): ?ResponseInterface
    {
        // x-api-key is set
        if (!$request->hasHeader('x-api-key')) {
            return $this->response->json([
                'success' => false,
                'message' => 'Missing API key'
            ], 401);
        }

        // x-api-key is valid
        if ($request->getHeaderLine('x-api-key') !== getenv('API_KEY')) {
            return $this->response->json([
                'success' => false,
                'message' => 'Invalid API key'
            ], 401);
        }

        return null;
    }
}

// public/index.php
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container, $path, $filter) {
        if ($filter) {
            $filterService = $container->make(\App\Service\Filters::class);
            $filterResponse = $filterService->runFilter(filter: $filter, request: $request);
            if ($filterResponse) {
                return $filterResponse;
            }
        }

        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
    });
```

Let me explain what we did here:

- We created a new class Filters to handle the filters and separate the auth logic from front controller.
- the class expected an implementation of CustomResponseInterface instance as a dependency to generate responses.
- the method `runFilter()` takes the filter name (e.g. "apiauth") and the request object as parameter, both to be provided by the front controller.
- the method `runFilter()` matches the filter name and runs the corresponding private method in the class.
- the private method `apiFilter()` checks if the request has the x-api-key header and returns corresponding responses.
- in the front controller, we check if a filter name is set in route config. If it is, we use DI to create a new instance of Filters class via DI (so that it can pass the CustomResponseInterface dependency) and call its `runFilter()` method.

BUT my boy... you might have guessed what I am about to say... it has several issues:

- a service class like Filters.php is not meant to return responses. It should be atmost used to check is auth is allowed but not return responses. Services should contain business logic, not HTTP response generation.
- The Filter class is doing too many things. It is on one hand checking if a user is authorized to view the response and also returns the response on other hand.
- Issues similar to phase 1 approach persist: we cannot modify the response after it has been returned by the controllers if we ever need to.

So, what do we do next? Well... let's think about the primary concern here... services should not return responses. But what else can? Controllers... yes... but it should not be responsible for authenticating requests. What else can return response and check for authentication? Bingo! Remember the word "Middleware"? We are going to create a middleware that does the job.

### Phase 3

...

[Next: Wrapup](./13-wrapup.md)
