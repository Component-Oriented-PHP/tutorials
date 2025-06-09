---
previous: 7-templates
next: 9-configurations
---

# Inversion of Control (IoC): Service Locator and Dependency Injection

I am going to explain this IoC concept in a practical way. To do this, let's assume some idiot came to you and said twig is better than platesphp. You got in the mood to replace our existing platesphp with twig. You want to replace the existing templating engine with a new one. So, you went and replaced Plates Engine with Twig Loader in the PlatesRenderer class. Now, we did not need to change anything in the controllers so that's a plus for having a separate view rendering class.

However, what happens is your team lead later decides that they want to bring back platesphp. So now, you need to redo the PlatesRenderer class. However, what if you kept PlatesRenderer as it is and created a separate TwigRenderer class to handle rendering with Twig? Now you have two renderer classes, but your controllers are still hardcoded to use PlatesRenderer. To switch to TwigRenderer, you'd have to go into every single controller and change `new PlatesRenderer()` to `new TwigRenderer()`. If you have 50 controllers, that's 50 places to change. That's a maintenance nightmare.

Even worse, what if you want to support both templating engines and let users choose? Or what if you want to switch between them based on configuration? Our current approach makes this nearly impossible without major code changes.

Now this is where IoC design philosophy comes into the drama. This philosophy simply states that classes should not create their own dependencies; instead, dependencies should be provided to them from the outside.

That is, instead of a controller creating its own templating engine, the templating engine should be "injected" into the controller from somewhere else (where from? Be patient. I will explain). This way, the controller doesn't need to know or care about which specific templating engine is being used - it just works with whatever is given to it.

IoC is implemented in two ways: Service Locator and Dependency Injection. I am going to explain both in the simplest possible manner. Be patient and keep on reading.

## Service Locator Approach

Okay... so if you have used CodeIgniter 4 ever before, you know that it comes with a `Services.php` config class, where you define how a service (any library or business service class) is accessed by service('service_name'). This approach is Service Locator approach.

Let's think about our scenario, we want to use twig or plates without removing PlatesRenderer or TwigRenderer (not yet created but we'll add it later). So, how can we access the PlatesRenderer or TwigRenderer class in controllers without explicitly calling an instance of either classes?

Make the following changes to HomeController:

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    private $view;

    public function __construct()
    {
        $this->view = (new \App\Library\Service\ServiceLocator())->get('view');
    }

    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!'
        ]));
    }
}
```

Notice how I removed PlatesRenderer entirely. Now, the controller is no longer responsible for handling which renderer is used to render views. It's the Servce Locator class (not created yet, will be adding it below) that's responsible for doing that. This is the Service Locator approach.

Let's create the Service Locator class now.

```php
// src/Library/Service/ServiceLocator.php
<?php

declare(strict_types=1);

namespace App\Library\Service;

use App\Library\View\PlatesRenderer;

class ServiceLocator
{
    private array $services = [];

    public function get(string $serviceName)
    {
        if (!isset($this->services[$serviceName])) {
            $this->services[$serviceName] = $this->createService($serviceName);
        }

        return $this->services[$serviceName];
    }

    private function createService(string $serviceName)
    {
        switch ($serviceName) {
            case 'view':
                return new PlatesRenderer();
            default:
                throw new \Exception("Service {$serviceName} not found");
        }
    }
}
```

Now now... this is not the most powerful and perfectly created Service Locator but this will do the job for our demonstration.

Go to the browser and see how `/` still works. Meaning our service locator is working as it should. However, we have not yet changed AboutController so you will get relevant Whoops error. Fix that on your own (Come on, you can do it).

This ServiceLocator class acts as a centralized registry that manages and provides access to application services. Think of it as a phone book for your application's services - instead of hardcoding dependencies everywhere, you ask the service locator "hey, give me the view service" and it handles the rest.

- The `$services` array is where all the magic happens. It's a simple associative array that acts as a cache for instantiated services. The key is the service name (like 'view') and the value is the actual service instance. This array starts empty and gets populated as services are requested.
- When you call `get('view')` {like we just did in HomeController}, the method first checks if that service already exists in the $services array. If it does, boom - it returns the cached instance immediately. This is important because it means each service is a singleton within the service locator's scope. You're not creating new PlatesRenderer instances every single time you need the view service.
- If the service doesn't exist in the cache yet, that's when we are calling `createService()`. This is where the actual instantiation logic lives. The switch statement maps service names to their corresponding class instantiations. In our case, when we asked for 'view', it created a new `PlatesRenderer` instance.
- Also, if we ask for a service that doesn't exist, it throws an exception with a clear message.
- The beauty of this approach is in the controller. Look at that constructor - `$this->view = (new \App\Library\Service\ServiceLocator())->get('view');`. The controller has absolutely no idea whether it's getting `PlatesRenderer`, `TwigRenderer`, or any other renderer. It just knows it's getting something that can render views.

One thing to note though... this implementation creates a new ServiceLocator instance every time in the controller constructor. In a real application, we'd want the ServiceLocator itself to be a singleton or injected as a dependency. But for demonstration purposes, this gets the job done without overcomplicating things.

Anyway, moving on... let's create TwigRenderer to handle rendering with Twig views.

In root dir, run `composer require twig/twig`. Then create `src/Library/View/TwigRenderer.php` file.

```php
<?php

declare(strict_types=1);

namespace App\Library\View;

use Twig\Environment;
use Twig\Loader\FilesystemLoader;

class TwigRenderer
{
    private Environment $renderer;

    public function __construct()
    {
        $loader = new FilesystemLoader(__DIR__ . '/../../../templates/twig');
        $this->renderer = new Environment($loader, [
            'debug' => true, // set to false in production
            'cache' => __DIR__  . '/../../../tmp/cache', // set to a proper path in production
        ]);
    }

    public function renderView(string $template, array $data = []): string // deliberately different method name
    {
        return $this->renderer->render($template . '.twig', $data);
    }
}
```

Now, we are going to add the twig templates in `templates/twig` folder.

```html
{% raw %}
<!--templates/twig/layouts/default.twig-->
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }}</title>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css">

    {% block styles %}{% endblock %}
</head>

<body>
    {% include 'partials/header.twig' %}

    <main class="container">
        {% block content %}{% endblock %}
    </main>

    {% include 'partials/footer.twig' %}
</body>

</html>

<!--templates/twig/partials/header.twig-->
<header class="container">
    <a href="/" class="logo">Home</a>
    <a href="/about">About Us</a>
</header>

<!--templates/twig/partials/footer.twig-->
<footer class="container">
    <center>
        &copy; {{ "now"|date("Y") }} All Rights Reserved
    </center>
</footer>

<!--templates/twig/home/index.twig-->
{% extends 'layouts/default.twig' %}

{% block content %}
Welcome to <span class="twig">Twig</span>!
{% endblock %}

{% block styles %}
<style>
    .twig {
        color: lime;
    }
</style>
{% endblock %}

<!--templates/twig/about/index.twig-->
{% extends 'layouts/default.twig' %}

{% block content %}
Welcome to <span class="twig">Twig</span> About Us Page!
{% endblock %}

{% block styles %}
<style>
    .twig {
        color: green;
    }
</style>
{% endblock %}

{% endraw %}
```

Great. Now, let's change service locator to use TwigRenderer instead of PlatesRenderer.

```php
    // ...

    private function createService(string $serviceName)
    {
        switch ($serviceName) {
            case 'view':
                // return new PlatesRenderer();
                return new TwigRenderer(); # now we are returning twig renderer instead
            default:
                throw new \Exception("Service {$serviceName} not found");
        }
    }
```

Go and check the browser, you should see an error saying "Call to undefined method App\Library\View\TwigRenderer::render()". This is because all our controllers use `render` method and not `renderView`. So, go back and change `renderView` to `render` in TwigRenderer and the error should be gone now. BUT... WHY DID I DELIBERATELY USE renderView to begin with?

I did it to explain something. What if you create a new Renderer, say MustacheRenderer to use mustace template engine and mistakenly added renderView method instead of render? You would go back and fix the error ofcourse by changing renderView to render like we just did with TwigRenderer. But, you can avoid this mistake early while coding by having something called Contracts (Interfaces in PHP).

### Contracts (Interfaces)

You need to be aware of interfaces before I can go on.

An interface in PHP is like a contract that defines what methods a class must have, without specifying how those methods should work. Think of it as a blueprint or a promise. When a class "implements" an interface, it's promising to provide all the methods that the interface defines.

Here's a simple example. Create a `src/Library/View/RendererInterface.php` file.

```php
// src/Library/View/RendererInterface.php; you need to put it in a View directory inside Library
<?php

declare(strict_types=1);

namespace App\Library\View;

interface RendererInterface
{
    public function render(string $template, array $data = []): string;
}
```

Pay attention to the code above. The interface does nothing but define what the render method should be like. First, it states that the `render` method must be implemented by the class that implements this interface. Second, the render method MUST be public (not private, not protected). Third, it expects two arguments: one, a string of the template to render, and the second, an array of data to pass to the template. Last, it returns a string of the rendered template (the HTML).

Now, any class that implements this interface MUST have a render method with exactly this signature. If it doesn't, PHP will throw an error. This is the "contract" part - the class is promising to provide this method.

Move the PlatesRenderer.php from Library directory to the View directory inside Library: `./src/Library/View/PlatesRenderer.php`, if you haven't done so already. Then update the code:

```php
// src/Library/View/PlatesRenderer.php
<?php

declare(strict_types=1);

namespace App\Library\View;

use League\Plates\Engine;

class PlatesRenderer implements RendererInterface
{
    private Engine $engine;

    public function __construct()
    {
        $this->engine = new Engine(__DIR__ . '/../../../templates/plates');
    }

    public function render(string $template, array $data = []): string
    {
        return $this->engine->render($template, $data);
    }
}
```

Notice the `implements RendererInterface` part. This means PlatesRenderer is now bound by the contract - it must have a public render method that takes a string and an array, and returns a string.

Do the same for TwigRenderer: `./src/Library/View/TwigRenderer.php`

```php
<?php

declare(strict_types=1);

namespace App\Library\View;

use Twig\Environment;
use Twig\Loader\FilesystemLoader;

class TwigRenderer implements RendererInterface
{
    private Environment $renderer;

    public function __construct()
    {
        $loader = new FilesystemLoader(__DIR__ . '/../../../templates/twig');
        $this->renderer = new Environment($loader, [
            'debug' => true, // set to false in production
            'cache' => __DIR__  . '/../../../tmp/cache', // set to a proper path in production
        ]);
    }

    public function render(string $template, array $data = []): string
    {
        return $this->renderer->render($template . '.twig', $data);
    }
}
```

Go back to the browser and check that everything works fine. Now we can easily avoid this kind of mistake in the future. But, the use of interfaces is not just limited to this. It has a greater role to play in Dependency Injection. I will show you how, be patient.

## Dependency Injection Approach

> You can read this beautiful article on DI from Nette after/before going through this tutorial: <https://doc.nette.org/en/dependency-injection/introduction>

Alright, you've seen how the Service Locator removes the responsibility of creating the renderer from the controller. The controller now just asks for it.

With Dependency Injection (DI), we take it one final, crucial step further: the controller doesn't even ask, it just states that it uses that particular service (like a View Renderer). The dependency is given, or "injected," into it from the outside (where from? Once again... Patience my boy... patience...).

Remember how our HomeController constructor currently looks?

```php
// Current approach with Service Locator
public function __construct()
{
    $this->view = (new \App\Library\Service\ServiceLocator())->get('view');
}
```

There is no major issue in it. However, the controller still knows about the ServiceLocator. It's still actively fetching its own dependency. Dependency Injection approach flips this around. The controller becomes completely passive. It simply states what it needs in its constructor, and something else provides it.

So, how do we solve this not so significant issue for smaller apps but recommended practice for bigger apps? Let's see... Look at HomeController... instead of the controller asking for the view service from ServiceLocator class, we're going to make it declare what it needs and let something else worry about providing it. (If your HomeController is screaming I like ServiceLocator then give it a slap and force it to follow my practice)...

Let's modify our HomeController and AboutController:

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(private RendererInterface $view)
    {
    }

    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!!!!!'
        ]));
    }
}

// src/Controller/AboutController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class AboutController
{


    public function __construct(private RendererInterface $view)
    {
    }

    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('about/index', [
            'title' => 'This is a title for About Us!!!!!'
        ]));
    }
}
```

See what the hell happened? If you run the code right now, you will get an error saying "Too few arguments to function App\Controller\AboutController::__construct(), 0 passed in ...". Don't worry. We will implement dependency injection in front controller a bit later.

Just know that the controller constructor now simply screams at top of its voice - *"I need something (a class obviously) that implements RendererInterface."* It doesn't know how to create it, it doesn't know where to get it from - it just knows it needs it. This is pure dependency injection. Now remember that we have two renderer classes, PlatesRenderer and TwigRenderer. Moving on... (what? you thought I was going to teach you how to use these renderers directly? Not so fast...)

As I said, you will encounter error like too few arguments to function on whichever route you visit. This is because our controllers now state that they expect a class that implements RendererInterface to be passed when the Controller itself is instantiated during routing, but look at the front controller where we are mapping the routes.

```php
// public/index.php

// ... lots of code before this

//map the route definitions
$routes = require_once __DIR__ . '/../config/routes.php';
foreach ($routes as $name => $route) {
    $requestMethod = $route[0];
    $path = $route[1];
    $handler = explode('::', $route[2]);
    $controller = $handler[0];
    $method = $handler[1];

    $map->$requestMethod($name, $path, function ($request) use ($controller, $method) {
        return (new $controller())->$method($request); // <-- notice this part
    });
}

// ... lots of code after this
```

Did you see the issue? No? Then slap yourself one time and see again...

You see it now, right? Yep... we are not passing any class (PlatesRenderer or TwigRenderer in our case) that implements RendererInterface: `return (new $controller())->$method($request);`.

But there is another issue. This part of code is used to instantiate any controller passed in route config. However, diff controllers may have diff requirements. For instance, an AuthController may require a Session object, while a HomeController may not. So, how do we solve this?

Let's see how we will do it.

> NOTE: Now that we are transitioning to DI, the ServiceLocator class we created will no longer be used, but keep it for reference. No harm there (or remove it, it's your call).

To solve such an issue, we use something called a DI Container package. The DI Container (a class) will automatically fetch, and instantiate the required classes/services and pass them onto the controllers.

### Setting up DI Container

I have used several DI packages in my projects. I recommend you pick any one you like/have used: [Symfony DI](https://symfony.com/doc/current/components/dependency_injection.html), [Nette DI](https://github.com/nette/di), [League Container](https://container.thephpleague.com/) {my personal favorite}, [Aryun Dependency Injector](https://github.com/rdlowrey/auryn), [Aura DI](https://github.com/auraphp/Aura.Di), [Pimple](https://github.com/silexphp/Pimple), and [PHP DI](https://php-di.org/).

I will use Aryun Dependency Injector in this tutorial. It's a simple package that does what we need to solve our current error. Run `composer require rdlowrey/auryn` in root dir.

Now, to make use of this DI Container, we have to make a few changes to our front controller:

```php
// public/index.php
<?php

declare(strict_types=1);

use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;
use Aura\Router\RouterContainer;
use Auryn\Injector;
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

// DI
$container = new Injector();

# register services
// this tells the container: "When someone asks for RendererInterface, give them TwigRenderer"
$container->alias(RendererInterface::class, TwigRenderer::class);

// OR: to use PlatesRenderer instead, uncomment the line below and comment the one above
// $container->alias(RendererInterface::class, \App\Library\View\PlatesRenderer::class);

// Router
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

    // we need to make use of DI container to pass the necessary services to controller for usage
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container) {
        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
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

Now, check the browser. Error should be gone by now. Let me explain what happened.

Notice this part:

```php
use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;
use Auryn\Injector;

// DI
$container = new Injector();

# register services
// this tells the container: "When someone asks for RendererInterface, give them TwigRenderer"
$container->alias(RendererInterface::class, TwigRenderer::class);

// OR: to use PlatesRenderer instead, uncomment the line below and comment the one above
// $container->alias(RendererInterface::class, \App\Library\View\PlatesRenderer::class);

```

So what's going on? First, we create an instance of the Injector class which Aryun's package provides for dependency management.

But this won't work alone. Why? You know our controller is asking for RendererInterface. We have two classes that implement that: PlatesRenderer and TwigRenderer. Which one should the container be able to pass? It does not know that.

So, we are telling the container, to register that when someone asks for RendererInterface, give them TwigRenderer.

```php
$container->alias(RendererInterface::class, TwigRenderer::class);
```

Now container knows, that if a controller asks for RendererInterface, give them TwigRenderer. You can uncomment the PlatesRenderer line if you want to use PlatesRenderer instead (don't forget to comment TwigRenderer line; one alias can have multiple implementations but the one registered last will be used by Container).

Next, move on to route mapping part because without that DI won't work.

```php

    // OLD WAY
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method) {
        return (new $controller())->$method($request);
    });

    // NEW WAY
    // we need to make use of DI container to pass the necessary services to controller for usage
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container) {
        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
    });
```

Notice what we are doing here. In the function, we added $container [`use ($controller, $method, $container)`] to ensure that we can make use of container in the function where we are instantiating controller.

Notice that earlier we had `return (new $controller())->$method($request);`. Here we were directly instantiating the controller, but now we are doing this: `$controllerInstance = $container->make($controller);`.

What's that? Basically, we are asking the Container to create an instance of the controller that route expects rather than manually instantiating like earlier. Why? Because if we don't, the container will not be able to pass the necessary dependencies to the controller.

In the following line `return $controllerInstance->$method($request);`, we are calling a method on the controller instance to handle the request.

That's it. It may sound complicated right now, as it did to me when I read about it for the first time, but I promise that overtime as you use this approach, it will become easier to understand. I believe in your capabilities (if you don't believe yourself, give yourself a hard slap and then read what I said here again).

> NOTE: You may ask now, if we have to use hundreds of services in our controllers, then are we going to register all 100 services in the container? Well, yes and no.
> Some DI container packages like League Container or Symfony's DI come with something called **Autowiring**. With autowiring enabled, you can typehint concrete classes in your controllers and the container will automatically instantiate them and their dependencies - no explicit registration needed.
> However, for interfaces (like our RendererInterface), you still need explicit registration because the container can't guess which implementation to use.
> It is highly recommended to register your key services explicitly in the container for the sake of explicitness, testability, and clarity of what is going on and where. Autowiring is great to get job done, but explicit registration gives you full control.

Anyway, before we move on to the next turorial, we can do ONE LAST THING FOR ASTHETICS.

You see, as we continue building our application the number of registered services will be in tens if not hundreds. This will clutter our front controller. Think with me. What can we do?

We have several options (Think... think... don't just read... use your brain to come up with solutions... keep thinking on problem-solution line instead of passively reading). One option is to do the same thing we did to our routes. Like create a `config/dependencies.php` php array file, register all dependencies as php array there, then include that config file, register each defined dependency via foreach loop. That's simple enough.

```php
// example only
// config/dependencies.php

return [
    \App\Library\View\RendererInterface::class => \App\Library\View\TwigRenderer::class,
];

// public/index.php
$dependencies = require_once __DIR__ . '/../config/dependencies.php';

foreach ($dependencies as $interface => $class) {
    $container->alias($interface, $class);
}
```

What else?

Another option is to create a separate class (e.g., `src/Library/Service/Container.php`) and register all our dependencies in a register method that will stay private. Then have a getContainer() public method that creates an instance of Auryn Injector, registers services by calling our private method, and then return the Injector itself. Then we can use this class itself to only create controller instance (remember, we'd have registered all dependencies in the private method).

```php
// example only
// src/Library/Service/Container.php

class Container
{
    public function getContainer()
    {
        $container = new Aryun\Injector();

        $this->registerServices($container);

        return $container;
    }

    private function registerServices(Injector $container)
    {
        $container->alias(\App\Library\View\RendererInterface::class, \App\Library\View\TwigRenderer::class);
    }
}
```

I will use the config approach for the sake of simplicity in this basic tutorial. And maybe... we will take a different approach in the advanced one.

Create a file `config/dependencies.php` and add the following code:

```php
<?php

use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class
];
```

Now, refactor the `public/index.php` file:

```php
<?php

declare(strict_types=1);

use Aura\Router\RouterContainer;
use Auryn\Injector;
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

// DI
$container = new Injector();

# register services
$dependencies = require_once __DIR__ . '/../config/dependencies.php';
foreach ($dependencies as $key => $value) {
    $container->alias($key, $value);
}

// Router
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

    // we need to make use of DI container to pass the necessary services to controller for usage
    $map->$requestMethod($name, $path, function ($request) use ($controller, $method, $container) {
        // Here's the magic! The container automatically creates the controller
        // and injects all its dependencies
        $controllerInstance = $container->make($controller); // this is needed; we need to instantiate controller via DI
        return $controllerInstance->$method($request);
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

Go on. Check the browser. App should work as it should.

Next up, I am covering [configurations](./9-configurations.md).
