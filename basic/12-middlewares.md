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

Register the rotues:

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

Now, go and visit <http://localhost:8000/api/page> and <http://localhost:8000/api/page/about>. You should see the JSON API responses.

I doubt I need to explain what's hapenning in the Api PageController since we have been through this multiple times. But breifly, we asked DI for a class implementing CustomResponseInterface. Then we fetched the data (single or all pages) and returned the response with `application/json` content type (JsonResponse sets that automatically).

But, as always, there is an issue with our simple API. It's not protected. What if we wanted only our apps or the apps having some secret to use the endpoints?

We need to add authentication to our app. Let's see a simple approach.