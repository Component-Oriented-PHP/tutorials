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

### Approach 2

### Approach 3

## HTTP Requests and Responses

### Requests

### Responses

### Middlewares
