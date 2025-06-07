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

But there is a minor issue. What if you create a new Renderer (say MustacheRenderer to use mustace template engine) and mistakenly add renderView method instead of render? You can either replace render with renderView method in all controllers or change renderView to render in the newly created Renderer. But, you can avoid this mistake early while coding by having something called Contracts (Interfaces in PHP).

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

Move the PlatesRenderer.php from Library directory to the View directory inside Library: `./src/Library/View/PlatesRenderer.php`. Then update the code:

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

## Dependency Injection Approach

<explain>

Great. Go check the browser and you should not see any changes in the output whatsoever.
