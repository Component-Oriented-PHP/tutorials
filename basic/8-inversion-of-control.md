---
previous: 7-templates
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

Let's think about our scenario, we want to use twig or plates without removing PlatesRenderer or TwigRenderer (not yet created but we'll add it later). So, how can we access the PlatesRenderer or TwigRenderer class in controllers without explicitly calling an instance of either classes? Take a look at the following code:

```php
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

So, how do we solve this not so significant issue for smaller apps but recommended pratice for bigger apps? Let's see... Look at HomeController... instead of the controller asking for the view service from ServiceLocator class, we're going to make it declare what it needs and let something else worry about providing it. (If your HomeController is screaming I like ServiceLocator then give it a slap and force it to follow my practice)...

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
```