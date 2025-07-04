---
previous: 10-markdown
next: 12-middlewares
---

# Fixing Response Coupling in Controllers

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

Noticed an inconsistency between what we have learned and what we kept doing here?

In case you have thoroughly gone through all previous chapters, especially DI, you may ask, **why the heck are we using HtmlResponse directly?** What if we were to replace `laminas/diactoros` with some other package? Wouldn't we need to replace all instances of HtmlResponse (or for that matter JsonResponse or RedirectResponse from the Diactoros package)?

Moreover, isn't that inconsistent with everything we have learned so far, especially how, for better separation, controllers should not know what package they are using to get job done and those packages should be provided to the controllers from outside?

Throughout this tutorial, we've been obsessing over abstractions and dependency injection. We inject RendererInterface instead of hardcoding a template engine. We use PageFetcher as a service instead of writing markdown logic in controllers. But here? We're directly instantiating HtmlResponse like we don't really care.

This is breaking our own rules. We're tightly coupling our controller to a specific HTTP library, which means our controllers highly depend on Laminas Diactoros package only. It just looks inconsistent with the rest of our codebase. If someone reads our code, they'll see beautiful abstractions everywhere and then... boom, `new HtmlResponse()` scattered around like we forgot what we were doing.

Now, **from a purely practical standpoint, you might never need to replace Laminas Diactoros - it's mature, stable, and PSR-7 compliant. So using HtmlResponse directly won't hurt you in most real applications.**

So technically, there is no need to ever replace laminas diactoros with any other package . It is one of the most widely used and highly capable package out there written by some of the best PHP devs out there in the world. So from that perspective, you do not even need to think about replacing the package with an alternative and can continue using HtmlResponse directly. As such, **you can skip this chapter and move on to the next chapter where I have covered Middlewares**. However, _you can continue reading if you want to know more about this_.

## Approach 1: AbstractController

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

Before we go on, **you may ask (or may have asked in the templates chapter itself) are Laravel, CodeIgniter and Symfony devs idiots to use Base Controllers/Abstract Controllers? Why should we follow you and not them?**

Look, these framework developers are some of the smartest people in the PHP world. They're not idiots - they're solving completely different problems than we are here.

When you're building a framework that millions of developers will use - from complete beginners to seasoned pros - you have to make trade-offs. Developer experience often beats perfect architecture. If a new developer can extend a base controller and call `$this->json($data)` instead of dealing with DI/IoC concepts they don't understand yet, that's a massive win for adoption and learning curve.

You know already that these frameworks have established ecosystems. Laravel's base controller isn't just about responses - it's woven into middleware, validation, authorization, and hundreds of packages. Ripping that out would break half the internet (or not, I don't know).

But the main question in your mind should be - are you building a framework or an application.

Well, to be frank, in this tutorial we are doing neither. We are learning principles to develop PHP apps without frameworks (by using composer packages), and HOW TO THINK TO DEVELOP APP BEFORE YOU BEGIN CODING, how to refactor something you already have, etc. I'm teaching you to think about coupling, abstraction, and trade-offs because these skills will serve you well whether you end up using Laravel, Symfony, or building your own thing from scratch.

Just know that even if you use Laravel tomorrow, knowing why they made certain choices makes you a better developer who can work with the framework instead of just throwing code at it.RetryClaude can make mistakes. Please double-check responses.

## Approach 2: Our Own Response Interfaces

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

I doubt any additional explanation is needed here, since we have been doing the same with templates renderer or markdown parser for the past few chapters. To ensure that we can easily swap diactoros package with some other http implementation, we created an interface first that defined the methods we need. Then we created a class that implements this interface using diactoros and injected it into our container. That's it.

You can use this CustomResponse approach if you like it, or use AbstractController. Or, the third one below. Your call, it's your application.

## Approach 3: Response Factory

This one is the most dense section since it is the most complicated one of the three.

Now, if you did not know, PSR-17 defines standard factory interfaces for creating HTTP responses - ResponseFactoryInterface and StreamFactoryInterface. Laminas Diactoros implements these interfaces, which means we could use these standardized factories in our controllers instead of directly creating response objects (i.e. `new HtmlResponse()`). Our container can inject the Laminas implementations when these interfaces are requested.

Before we get into coding, someone may ask, why can't we simply use `ResponseInterface` and make our container pass `\Laminas\Diactoros\Response` instead, and then use it to return responses? Well, theoretically we can do that but the simplest explanation (though it is more complex than this) I can give is that `ResponseInterface` is not a service/library, but represents a Response object. On the other hand, the two factory interfaces are services. When we inject services via DI, we should be injecting things that do work for us (services), not data objects that represent state.

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

However, the core issue here isn't whether ResponseInterface represents a "state/data" or "service" - it's that injecting a **single response instance** doesn't make sense when you need to create multiple different responses. Single Response Instance means that the ResponseInterface represents a single data object and not multiple ones that we may need in certain scenarios (examples below).

What does "multiple different responses" mean? Look at the example below:

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

You see the problem? **A single response instance can't handle all these different scenarios.** Why not? Because PSR-7 responses (here, ResponseInterface) are designed to be immutable, i.e. methods like withStatus(), withHeader(), etc. don't modify the original response - they return a *new instance entirely*.

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
use Psr\Http\Message\ResponseInterface; // <-- remove this after experimentation

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class,

    MarkdownParserInterface::class => LeagueMarkdownParser::class,

    PageFetcher::class => PageFetcher::class,

    CustomResponseInterface::class => CustomResponse::class

    ResponseInterface::class => \Laminas\Diactoros\Response::class // <-- remove this after experimentation
];
```

Go check the browser. You will see that "Hello World is present in the response body. It was not overwritten. What if something like this happens such that some other part of the codebase has written something into the body and one of our controllers has written something else alongside? The response will contain both. That's a problem. (Now don't forget to undo our HomeController and dependencies.php config file). Ok, this Hello World issue does not represent the whole problem... but it atleast demonstrates the shared state problem: when we inject the same response instance across different parts of your application, we risk unintended data accumulation and interference.

So, the real issues are basically (a) the injected response might have existing data that interferes (b) we're using a single "response" to create many different responses, which defeats the purpose of injection (c) when you call `$this->response->withStatus(404)`, you're essentially using the response as a factory anyway

Enough theory for now... go refactor your PageController.php to make use of ResponseFactoryInterface and StreamFactoryInterface.

```php
// src/Controller/PageController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\PageFetcher;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\StreamFactoryInterface;

class PageController
{

    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view,
        private ResponseFactoryInterface $responseFactory,
        private StreamFactoryInterface $streamFactory
    ) {
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        $slug = $request->getAttribute('slug');

        $page = $this->pageFetcher->fetchSingle($slug);

        if (!$page) {
            $html = $this->view->render('404');
            $stream = $this->streamFactory->createStream($html);

            return $this->responseFactory->createResponse(404)->withBody($stream);
        }

        // separated for clarity and readability
        $html = $this->view->render('page/show', [
            'title' => $page['title'],
            'description' => $page['description'],
            'page' => $page
        ]);

        $stream = $this->streamFactory->createStream($html);

        return $this->responseFactory->createResponse(200)->withBody($stream);
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
use Laminas\Diactoros\ResponseFactory;
use Laminas\Diactoros\StreamFactory;
use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\StreamFactoryInterface;

return [
    RendererInterface::class => TwigRenderer::class,
        // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class,

    MarkdownParserInterface::class => LeagueMarkdownParser::class,

    PageFetcher::class => PageFetcher::class,

    CustomResponseInterface::class => CustomResponse::class,

    ResponseFactoryInterface::class => ResponseFactory::class,
    StreamFactoryInterface::class => StreamFactory::class,
];
```

Ok... a lot is going on here. Let me explain each part of the code. It seems complex and hard work as compared to the previous two approaches I have discussed (and let me tell you again: you are free to use any of the three approaches you feel comfortable with, no pressure to use this one).

- Notice what we are importing in the constructor other than PageFetcher and RendererInterface. That's the other two additional dependencies: ResponseFactoryInterface and StreamFactoryInterface.
  - You already know what that means: PageController screams that it now relies on ResponseFactoryInterface and StreamFactoryInterface. DI Container looks at the registered services and injects Laminas\Diactoros\ResponseFactory and Laminas\Diactoros\StreamFactory into PageController.

- Next up, in the show method
  - we fetch slug of the page to render from the attribute of the request object
  - we use our PageFetcher service to get the page data
  - if page does not exist
    - we are using the RendererInterface to get html string for a 404 page
    - then we are passing this html string to our StreamFactory to create a stream object required by the ResponseFactory to generate Response
  - if page exists we are doing the same thing, just rendering a different twig template and passing different status code (200 in this case, 404 if page did not exist)

Let me explain how StreamFactory and ResponseFactory actually work.

The `StreamFactoryInterface` is responsible for creating PSR-7 stream objects. In HTTP responses, the body content MUST be wrapped in a stream object rather than being a plain string. But why does it only want a stream object and not a plain string? Well, to put it simply, if you use strings, the entire content must be loaded into memory at once. A 2GB video file would consume 2GB of RAM, and with multiple concurrent users, you'd quickly exhaust server memory. Streams on the other hand load content on-demand in small chunks (typically 8KB), so memory usage remains constant regardless of file size (thus, prevent memory exhaustion/server crashes). Plus, HTTP supports sending data in chunks rather than waiting for the complete response, improving user experience and server efficiency (and since they do, why not use Streams?). Streams also, provide explicit control over file handles and other system resources and can wrap various data sources (files, memory, network connections, temporary storage) while strings can only be strings.

Anyway, the Laminas implementation of StreamFactoryInterface looks something like this:

```php
// illustrative code only

// What happens when you call:
$stream = $this->streamFactory->createStream($html);

// Internally, Laminas StreamFactory does something like:
class StreamFactory implements StreamFactoryInterface
{
    public function createStream(string $content = ''): StreamInterface
    {
        $resource = fopen('php://memory', 'r+');
        fwrite($resource, $content);
        rewind($resource);
        
        return new Stream($resource);
    }
}
```

The stream here acts as a wrapper around the actual content,providing methods like getContents() [gets full content as string], read() [reads specific number of bytes], write() [writes content to the stream], and rewind() [resets the stream pointer to the beginning].

Moving on, the `ResponseFactoryInterface` creates fresh response objects with the specified status code like this:

```php
// illustrative code only

// What happens when you call:
$response = $this->responseFactory->createResponse(404);

// Internally, Laminas ResponseFactory does something like:
class ResponseFactory implements ResponseFactoryInterface
{
    public function createResponse(int $code = 200, string $reasonPhrase = ''): ResponseInterface
    {
        return new Response(
            'php://memory',  // empty body stream
            $code,           // status code
            []               // empty headers array
        );
    }
}
```

Notice our PageController. ResponseFactoryInterface gives us a clean, fresh response object that you can then modify using the immutable (not able to be changed) methods:

```php
// illustrative code only
$response = $this->responseFactory->createResponse(200)
    ->withHeader('Content-Type', 'text/html')
    ->withHeader('Cache-Control', 'no-cache')
    ->withBody($stream);
```

That's it. That's how we can use ResponseFactoryInterface and StreamFactoryInterface in our controllers for better separation.

Now, we could also refactor our controllers for better error handling and separation, but I will leave all that for the advacned tutorial.

[Next: Middlewares](./12-middlewares.md)
