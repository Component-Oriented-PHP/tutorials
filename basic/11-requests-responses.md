---
previous: 10-markdown
next: 12-wrapup
---

# HTTP Requests, Responses and Middlewares

This will probably be the last chapter in our basic tutorial. I intend to cover one inconsistency between my teaching and what we have been doing up until now in controllers and also teach you a bit about HTTP responses (and requests). I will also use this chapter to cover middlewares briefly.

Go take a look at any controller, say HomeController.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\PageFetcher;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view
    ) {
    }

    public function index(ServerRequestInterface $request)
    {

        $pages = $this->pageFetcher->fetchAll();

        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'pages' => $pages
        ]));
    }
}
```

Now, if you have thoroughly gone through all previous chapters, especially DI, you may ask, why the heck are we using HtmlResponse directly? What if we are to replace `laminas/diactoros` with some other package? Wouldn't we need to replace all instances of HtmlResponse (or for that matter JsonResponse or RedirectResponse of Laminas Diactoros package)? Yes... and that's troublesome.

And, isn't that inconsistent with everything I've been teaching you so far, especially how controllers should not know what package they are using to get job done and those packages should be provided to the controllers from outside? Throughout this tutorial, we've been obsessing over abstractions and dependency injection. We inject RendererInterface instead of hardcoding a template engine. We use PageFetcher as a service instead of writing markdown logic in controllers. But here? We're directly instantiating HtmlResponse like it's nobody's business.

This is breaking our own rules. We're tightly coupling our controller to a specific HTTP library, which means our controllers highly depend on Laminas Diactoros package only. It just looks inconsistent with the rest of our codebase. If someone reads our code, they'll see beautiful abstractions everywhere and then... boom, `new HtmlResponse()` scattered around like we forgot what we were doing.

But, first and foremost, there is no need to ever replace laminas diactoros with any other package since it is one of the most widely used and highly capable package out there written by some of the best PHP devs out there in the world. So from that perspective, you do not even need to think about replacing the package with an alternative and can continue using HtmlResponse directly. As such, you can skip this section and move on to the section where I have covered HTTP Requests and HTTP responses and Middlewares. However, you can continue reading if you want to know more about this.

## Making Controllers Consistent With Our Approach

Now, this is probably the only chapter in the whole tutorial that is structured with subheadings, otherwise I have kept the flow natural everywhere else. This is because I am teaching here three different approaches you can take to solve the inconsistency in controllers and our codebase.

### Approach 1: AbstractController

This is probably the easiest way to solve the inconsistency, and is seen in Symfony Framework.

Go refactor your existing AbstractController:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\PlatesRenderer;
use Laminas\Diactoros\Response\HtmlResponse;
use Laminas\Diactoros\Response\JsonResponse;
use Laminas\Diactoros\Response\RedirectResponse;

abstract class AbstractController
{
    protected function html(string $html, int $status = 200, array $headers = []): HtmlResponse
    {
        return new HtmlResponse($html, $status, $headers);
    }

    protected function json(array $data, int $status = 200, array $headers = []): JsonResponse
    {
        return new JsonResponse($data, $status, $headers);
    }

    protected function redirect(string $url, int $status = 302, array $headers = []): RedirectResponse
    {
        return new RedirectResponse($url, $status, $headers);
    }
}
```

Observe that I added three new methods, html(), json() and redirect(). All these methods utilize Laminas Diactoros to return responses needed. Now we can just make controllers extend this AbstractController and we can rewrite HomeController like this:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ServerRequestInterface;

class HomeController extends AbstractController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view
    ) {
    }

    public function index(ServerRequestInterface $request)
    {

        $pages = $this->pageFetcher->fetchAll();

        return $this->html($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'pages' => $pages
        ]));
    }
}
```

You can use this approach for simplicity. Look how clean our HomeController is now compared to before. Now you can just swap laminas diactoros and update AbstractController to keep things up.

Even though it is possible to use DI and AbstractController together like we do in Symdony, this approach is certainly not the best one as I have stated in a previous lesson. Remember when we talked about AbstractController for templates? The same issues apply here. Sure, we could add html(), json(), and redirect() methods to AbstractController to solve the response coupling problem, but we'd still have the inheritance limitations we discussed earlier. What if a controller needs to extend a different base class? What if we have 20 controllers that need different combinations of services?

Here is the thing, by using an AbstractController we're forcing controllers into an inheritance hierarchy just to get response helpers. What if we need a controller that extends something else? We're stuck. The html(), json(), redirect() methods hide the fact that we're still tightly coupled to Laminas. It looks cleaner but doesn't actually solve the coupling problem - it just moves it one layer up. Remember what we talked about AbstractControllers in templates chapter? Not every controller needs all three response types. Why force them to inherit json() if they only ever return HTML?

So, the other two approaches I discuss below are better, you can use the one you like.

Now **you may ask (or may have asked in the templates chapter itself) are Laravel, CodeIgniter and Symfony devs idiots to use Base Controllers/Abstract Controllers? Why should we follow you and not them?**

Look, these framework developers are some of the smartest people in the PHP world. They're not idiots - they're solving completely different problems than we are here.

When you're building a framework that millions of developers will use - from complete beginners to seasoned pros - you have to make trade-offs. Developer experience often beats perfect architecture. If a new developer can extend a base controller and call `$this->json($data)` instead of dealing with DI/IoC concepts they don't understand yet, that's a massive win for adoption and learning curve.

You know already that these frameworks have established ecosystems. Laravel's base controller isn't just about responses - it's woven into middleware, validation, authorization, and hundreds of packages. Ripping that out would break half the internet (or not, I don't know).

But the main question in your mind should be - are you building a framework or an application.

Well, to be frank, in this tutorial we are doing neither. We are learning principles to develop PHP apps without frameworks (by using composer packages), and HOW TO THINK TO DEVELOP APP BEFORE YOU BEGIN CODING, how to refactor something you already have, etc. I'm teaching you to think about coupling, abstraction, and trade-offs because these skills will serve you well whether you end up using Laravel, Symfony, or building your own thing from scratch.

Just know that even if you use Laravel tomorrow, knowing why they made certain choices makes you a better developer who can work with the framework instead of just throwing code at it.RetryClaude can make mistakes. Please double-check responses.

### Approach 2: Our Own Response Interfaces

This is slightly better than the first approach. Let's see how.

Create a new file `./src/Library/Http/CustomResponseInterface.php`:

```php
<?php

declare(strict_types=1);

namespace App\Library\Http;

use Psr\Http\Message\ResponseInterface;

interface CustomResponseInterface 
{

    public function html(string $html, int $status = 200, array $headers = []): ResponseInterface;

    public function json(array $data, int $status = 200, array $headers = []): ResponseInterface;

    public function redirect(string $url, int $status = 302, array $headers = []): ResponseInterface;

    public function xml(string $xml, int $status = 200, array $headers = []): ResponseInterface;

}
```

This response interface defines four methods that other classes using it must implement for four diff content types (html, json, redirect, xml). We can then create a class that implements this interface and inject it into our controllers.

Create `src/Library/Http/CustomResponse.php`:

```php
<?php

declare(strict_types=1);

namespace App\Library\Http;

use Laminas\Diactoros\Response\HtmlResponse;
use Laminas\Diactoros\Response\JsonResponse;
use Laminas\Diactoros\Response\RedirectResponse;
use Laminas\Diactoros\Response\XmlResponse;
use Psr\Http\Message\ResponseInterface;

class CustomResponse implements CustomResponseInterface
{

    public function html(string $html, int $status = 200, array $headers = []): ResponseInterface
    {
        return new HtmlResponse($html, $status, $headers);
    }

    public function json(array $data, int $status = 200, array $headers = []): ResponseInterface
    {
        return new JsonResponse($data, $status, $headers);
    }

    public function redirect(string $url, int $status = 302, array $headers = []): ResponseInterface
    {
        return new RedirectResponse($url, $status, $headers);
    }

    public function xml(string $xml, int $status = 200, array $headers = []): ResponseInterface
    {
        return new XmlResponse($xml, $status, $headers);
    }

}
```

Nothing special going on here, we are just doing what we have been doing so far: create an interface, make a class implement it and use an existing library (here, laminas diactoros) to create the responses. Now we can use this library in our controllers after we inject it into the container.

```php
// config/dependencies.php
<?php

use App\Library\Config\ConfigInterface;
use App\Library\Config\PHPConfigFetcher;
use App\Library\Http\CustomResponse;
use App\Library\Http\CustomResponseInterface;
use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;
use App\Service\Markdown\LeagueMarkdownParser;
use App\Service\Markdown\MarkdownParserInterface;
use App\Service\PageFetcher;

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class,

    MarkdownParserInterface::class => LeagueMarkdownParser::class,

    PageFetcher::class => PageFetcher::class,

    CustomResponseInterface::class => CustomResponse::class
];
```

Now, let's modify HomeController (and other controllers if you want).

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\Http\CustomResponseInterface;
use App\Library\View\RendererInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view,
        private CustomResponseInterface $response,
    ) {
    }

    public function index(ServerRequestInterface $request)
    {
        $pages = $this->pageFetcher->fetchAll();

        // separated this view part for better clarity
        $html = $this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'pages' => $pages
        ]);

        return $this->response->html($html);
    }
}
```

You can use this CustomResponse approach if you like it or use AbstractController. Or, the third one below.

### Approach 3: Response Factory

Now, if you did not know, PSR-17 defines standard factory interfaces for creating HTTP responses - ResponseFactoryInterface and StreamFactoryInterface. Laminas Diactoros implements these interfaces, which means we could use these standardized factories in our controllers instead of directly creating response objects (i.e. `new HtmlResponse()`). Our container can inject the Laminas implementations when these interfaces are requested.

Before we get into coding, someone may ask, why can't we simply use `ResponseInterface` and make our container pass `\Laminas\Diactoros\Response` instead, and then use it to return responses? Well, theoretically we can do that but the simplest explanation (though it is more complex than this) I can give to avoid doing it is that `ResponseInterface` is not a service/library, but represents a Response object. On the other hand, the two factory interfaces are services. When we inject services via DI, we should be injecting things that do work for us (services), not data objects that represent state.

- `ResponseInterface` = single, immutable response/data object that holds HTTP response information (with specific status, headers, and body)
- `ResponseFactoryInterface`/`StreamFactoryInterface` = services that create response objects for us
- `ResponseFactoryInterface` = A tool that can create many different response objects as needed

Take a look below:

```php
// for example only
// This would be weird - injecting a specific response instance
public function __construct(ResponseInterface $response) {} 

// This makes sense - injecting a factory that creates responses
public function __construct(ResponseFactoryInterface $responseFactory) {}
```

The core issue though isn't whether ResponseInterface represents a "state" or "service" - it's that injecting a **single response instance** doesn't make sense when you need to create multiple different responses.

What does "not being able to create multiple different responses" mean? Look at the below example:

```php
// for example only

// This would be weird and problematic:
class UserController
{
    public function __construct(
        private ResponseInterface $response  // A single response instance?
    ) {}
    
    public function show($id)
    {
        $user = $this->userService->find($id);
        
        // What do we do with this single $response instance?
        // We need to return different responses based on the situation:
        
        if (!$user) {
            // Need a 404 response
            return ??? 
        }
        
        if ($user->isBlocked()) {
            // Need a 403 response  
            return ???
        }
        
        // Need a 200 response with user data
        return ???
    }
    
    public function update($id) 
    {
        // Need different responses here too:
        // - 422 for validation errors
        // - 200 for success
        // - 404 if user not found
        return ???
    }
}

// Compare that mess with this clean factory approach:
class UserController
{
    public function __construct(
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface $streamFactory
    ) {}
    
    public function show($id)
    {
        if (!$user) {
            return $this->responseFactory->createResponse(404);
        }
        
        // Create fresh response for success case
        $body = $this->streamFactory->createStream(json_encode($user));
        return $this->responseFactory->createResponse(200)
            ->withHeader('Content-Type', 'application/json')
            ->withBody($body);
    }
}
```

You see the problem? **A single response instance can't handle all these different scenarios.** Why not? Because PSR-7 responses (here, ResponseInterface) are designed to be immutable, meaning methods like withStatus(), withHeader(), etc. don't modify the original response - they return a new instance entirely.

```php
// This is what actually happens:
$originalResponse = $this->response;
$newResponse = $originalResponse->withStatus(404); // Returns NEW instance

// $originalResponse is unchanged!
// $newResponse is a different object entirely
```

It's like the difference between being handed one pre-written letter (ResponseInterface) vs. being given a pen and paper to write different letters as needed (ResponseFactoryInterface). I hope that's clear.

So if you inject a single response instance, you'd have this weird situation:

```php
// for example only
class UserController
{
    public function __construct(
        private ResponseInterface $response
    ) {}
    
    public function show($id) 
    {
        // This creates a NEW response, doesn't modify the injected one
        return $this->response->withStatus(404)->withHeader('Content-Type', 'text/plain');
    }
    
    public function update($id)
    {
        // This ALSO creates a NEW response from the same original
        // What if the original response already has a body from somewhere else?
        return $this->response->withStatus(200)->withHeader('Content-Type', 'application/json');
    }
}
```

Now let's experiment with this real code in our app:

```php
// temporary code, remove after you're done

// HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\PageFetcher;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view,
        private ResponseInterface $response
    ) {
        $this->response->getBody()->write('Hello World!');
    }

    public function index(ServerRequestInterface $request)
    {

        $pages = $this->pageFetcher->fetchAll();

        $html =($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'pages' => $pages
        ]));

        $this->response->getBody()->write($html);

        return $this->response;
    }
}

// config/dependencies.php
<?php

use App\Library\Config\ConfigInterface;
use App\Library\Config\PHPConfigFetcher;
use App\Library\Http\CustomResponse;
use App\Library\Http\CustomResponseInterface;
use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;
use App\Service\Markdown\LeagueMarkdownParser;
use App\Service\Markdown\MarkdownParserInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ResponseInterface;

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class,

    MarkdownParserInterface::class => LeagueMarkdownParser::class,

    PageFetcher::class => PageFetcher::class,

    CustomResponseInterface::class => CustomResponse::class

    ResponseInterface::class => \Laminas\Diactoros\Response::class
];
```

Go check the browser. You will see that "Hello World is present in the response body. It was not overwritten. What if something like this happens such that some other part of the codebase has written something into the body and one of our controllers has written something else alongside? The response will contain both. That's a problem. (Now don't forget to undo our HomeController and dependencies.php config file). Ok, this Hello World issue does not represent the whole problem... but it atleast demonstrates the shared state problem: when we inject the same response instance across different parts of your application, we risk unintended data accumulation and interference.

So, the real issues are basically (a) the injected response might have existing data that interferes (b) we're using a single "response" to create many different responses, which defeats the purpose of injection (c) when you call `$this->response->withStatus(404)`, you're essentially using the response as a factory anyway

Enough theory for now... go refactor your HomeController.php:

```php

```

## HTTP Requests and Responses

### Requests

### Responses

### Middlewares
