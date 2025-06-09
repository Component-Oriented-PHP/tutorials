---
previous: 6-dispatching-to-a-class
next: 8-inversion-of-control
---

# Templates (Views)

Now that we have a controller, it's time to introduce some templates (views).

Traditionally, PHP devs (including me, like 10 years ago) used to write php files and have everything including database logic and views in the same file. This is the worst approach one could take often leading to the so called spaghetti code.

So, we are going to focus on separation of concerns (i.e., keeping business logic and views separated). This way, we can keep our application clean and simple.

However, for that, we don't really need an external package as PHP itself is a templating engine. But using a templating engine is better because they come with inbuilt features like security (escaping values), layout support, etc.

In the root of your project, create a `templates` directory. We are going to place all our view files in this directory. If you want, you can place them in `./src/View` or `./src/Templates` directories instead. Your call, it's your application.

Then run `composer require league/plates` in the root to install League Plates. You can install [Twig](https://twig.symfony.com/), [Nette Latte](https://latte.nette.org/), [Aura View](https://github.com/auraphp/Aura.View), [davanich view renderer](https://github.com/devanych/view-renderer), [Mustache](https://github.com/bobthecow/mustache.php), the good old [Smarty](https://www.smarty.net/) or any other template engine you like, even [Laravel Blade](https://github.com/jenssegers/blade).

In the templates directory add the following code (I am adding all plates view files in plates directory as I will be teaching you how to replace plates with another template engine in Inversion of Control chapter, wherein we will place those view files in separate directory for clarity):

```html
// templates/plates/layouts/default.php
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?= $this->e($title) ?></title>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css">

    <?= $this->section('styles') ?>
</head>

<body>
    <?= $this->insert('partials/header') ?>

    <main class="controller">
        <?= $this->section('content') ?>
    </main>

    <?= $this->insert('partials/footer') ?>
</body>

</html>

// templates/plates/partials/header.php
<header class="container">
    <a href="/" class="logo">Home</a>
    <a href="/about">About Us</a>
</header>

// templates/plates/partials/footer.php
<footer class="container">
    <center>
        &copy; <?= date('Y') ?> All Rights Reserved
    </center>
</footer>

// templates/plates/home/index.php
<?php $this->layout('layouts/default', ['title' => 'Home' /* this 'Home' is to be replaced with  a variable passed by our controller*/]) ?>

Welcome to <span class="platesphp">PlatesPHP</span>!

<?php $this->start('styles') ?>
<style>
    .platesphp {
        color: red;
    }
</style>
<?php $this->end() ?>

// templates/plates/about/index.php
<?php $this->layout('layouts/default', ['title' => 'About' /*this 'About' is to be replaced with  a variable passed by our controller*/]) ?>

Welcome to <span class="platesphp">PlatesPHP</span> About Us Page!

<?php $this->start('styles') ?>
<style>
    .platesphp {
        color: blue;
    }
</style>
<?php $this->end() ?>
```

Note that we are creating separate directories for each controller (home for HomeController, about for AboutController [to be created below]) and separated directories for layout and partials (includes). This is not a strict requirement, but a good practice. Your call, structure your templates directory however you want, but always keep things clean.

One more thing, I am not a designer/frontend dev. So if the app looks ugly by design, don't blame me.

Before we move on to using the views, let me explain how platesphp (the templating engine we just installed) works.

- First in default.php layout file, notice `$this->e()` wrapped around `$title` variable (we will pass $title value in controller). This function escapes the value (i.e., converting potentially dangerous characters into safe representations that won't be interpreted as executable code by the browser or cause other security issues). Escaping is important to ensure that some shady user of your site does not execute arbitrary code on your server which was quite possible in your spaghetti code.
- Second, in the same default.php file, we have `$this->section('content')` and `$this->section('head')`. These are placeholders where content from child templates will be inserted. The `content` section is special - it automatically captures all the content from child templates that isn't wrapped in other sections. The head section and other custom sections need to be explicitly defined in child templates using `$this->start('section_name')` and `$this->end()` {observe the home template and about template for clarity}.
- Third, the `$this->insert('partials/header')` and `$this->insert('partials/footer')` statements include reusable template fragments. This promotes code reusability and keeps templates DRY (Don't Repeat Yourself).
- Fourth, In the individual page templates (home/index.php, about/index.php), we use `$this->layout('layouts/default', ['title' => 'Home'])` to extend the default layout and pass variables to it. This creates a parent-child relationship where the *layout acts as a wrapper* around individual view files.
- Fifth, notice how both home/index.php and about/index.php have their own CSS for the .platesphp span class - home page makes it red while about page makes it blue. This demonstrates how each template can have its own specific styles that only apply to that page. The styles are injected into the layout's head section, creating page-specific styling. If you wanted the same styling across all pages globally, you would move the CSS to the layout file (default.php) instead of defining it in individual page templates. This gives you flexibility to have both global styles (in layouts) and page-specific styles (in individual templates).
- Data can be passed from controllers to templates, and from child templates to parent layouts using arrays in the layout declaration.

This separation allows us to maintain clean, secure, and reusable templates while keeping our business logic separate from presentation logic.

Anyway, now we have our view files that we can use. But how to we use these in controllers?

Well, let's open HomeController and change it.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function index(ServerRequestInterface $request)
    {
        $view = new \League\Plates\Engine(__DIR__ . '/../../templates/plates');
        return new HtmlResponse($view->render('home/index', [
            'title' => 'This is a title for Homepage'
        ]));
    }
}
```

Let's add AboutController and relevant route as well.

```php
// src/Controller/AboutController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class AboutController
{
    public function index(ServerRequestInterface $request)
    {
        $view = new \League\Plates\Engine(__DIR__ . '/../../templates/plates');
        return new HtmlResponse($view->render('about/index', [
            'title' => 'This is a title for About Us'
        ]));
    }
}

// config/routes.php
<?php

return [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index'],
    'about' => ['get', '/about', '\App\Controller\AboutController::index'],
];
```

Refresh the browser and visit home and about pages. You should get the expected results.

But notice that the HTML title is not changing to the ones we set in controller. We need to change home/index.php and about/index.php to use $title instead of passing explicit title values to layout.

```html
// templates/plates/home/index.php
<?php $this->layout('layouts/default', ['title' => $title]) ?>

Welcome to <span class="platesphp">PlatesPHP</span>!

<?php $this->start('styles') ?>
<style>
    .platesphp {
        color: red;
    }
</style>
<?php $this->end() ?>

// templates/plates/about/index.php
<?php $this->layout('layouts/default', ['title' => $title]) ?>

Welcome to <span class="platesphp">PlatesPHP</span> About Us Page!

<?php $this->start('styles') ?>
<style>
    .platesphp {
        color: blue;
    }
</style>
<?php $this->end() ?>
```

Try again. You should see the same titles set in controllers Now.

**HOWEVER**, there is ONE BIG PROBLEM. What if we had 100 controllers? Would we repeat the same code of creating an engine, passing path to templates dir? Hell no... that's stupidity. What if we decided we wanted to move templates dir to src folder instead? We wouldn't go around changing all the controllers.

We need a separate class. Let's create `PlatesRenderer` inside ./src/Library to ensure code reusability.

```php
// src/Library/PlatesRenderer.php
<?php

declare(strict_types=1);

namespace App\Library;

use League\Plates\Engine;

class PlatesRenderer
{
    private Engine $engine;

    public function __construct()
    {
        $this->engine = new Engine(__DIR__ . '/../../templates/plates');
    }

    public function render(string $template, array $data): string
    {
        return $this->engine->render($template, $data);
    }
}
```

Now let's update HomeController and AboutController to use PlatesRenderer instead of PlatesEngine.

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\PlatesRenderer;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    private PlatesRenderer $view;

    public function __construct()
    {
        $this->view = new PlatesRenderer();
    }

    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!'
        ]));
    }
}

// src/Controller/AboutController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\PlatesRenderer;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class AboutController
{

    private PlatesRenderer $view;

    public function __construct()
    {
        $this->view = new PlatesRenderer();
    }

    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('about/index', [
            'title' => 'This is a title for About Us!'
        ]));
    }
}
```

Do I need to tell you what to do next? Go on... check the browser (notice the additional exclaimation mark in HTML title).

Now, you may ask, why did we move view to constructor? Because index is not the only method that may need to render view. We are not going to keep creating new instances of PlatesRenderer in every method that will use it. It's not efficient.

Next question, why is PlatesRenderer private? Because it's not going to be used outside of this class. By making the $view property private, we are hiding the internal details of the controller. Other parts of our application don't need to know how the controller renders a view, just that it can.

Now, also notice how both HomeController and AboutController have the same constructor code. What if we had 20 such controllers? One option to not repeat is to create an AbstractController (abstract class) that both HomeController and AboutController extend. AbstractController can have a method that returns PlatesRenderer (or any other renderer) and we can use it in HomeController and AboutController. Though, we are not going to be using that much in this tutorial and instead be relying on dependency injection, which we will cover in the next chapter, I will show you how to setup and use a simple AbstractController implementation. Another issue is that some controllers may need to extend a different controller. That may lead to code duplication as multiple abstract controllers may share some common services.

However, you are free to use AbstractController  approach if you are more comfortable with that than dependency injection.

Create a new `./src/Controller/AbstractController.php` (you can name it anything else, like CI4 names it BaseController) file and add:

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\PlatesRenderer;

abstract class AbstractController
{

    protected PlatesRenderer $view;

    public function __construct()
    {
        $this->view = new PlatesRenderer();
    }

    /**
     * you can even have a separate method to render the view
     */
    protected function render(string $template, array $data = []): string
    {
        return $this->view->render($template, $data);
    }

    /**
     * Or, a separate method to only return an instance of PlatesRenderer;
     * but then it becomes something akin to a Service Locator (gonna cover it in IoC chapter)
     */
    protected function getView(): PlatesRenderer
    {
        return $this->view;
    }
}
```

Alright, now I need to explain what is happenning in the code above

- Notice the word `abstract` which we used for this AbstractController but in not HomeController or AboutController. In PHP, the word `abstract` is used for classes that are not meant to be used directly, but are meant to be extended by other classes. That is, AbstractController cannot be instantiated directly - you cannot do `new AbstractController()`. It serves as a blueprint that other controllers (HomeController and AboutController for example) must extend.
- Now move your attention to the `protected` keyword used for `$view` property as well as the methods. Why did we use protected and not public or private? First, not private because we want the property or methods to be accessible to our controllers only and obviously not public because we do not want them accessible anywhere but the child classes.
- We have the same constructor logic here that was previously duplicated in HomeController and AboutController. Now this initialization happens once in the parent class.
- Don't add all three ways of rendering view in HomeController and AboutController. Just add the one you prefer.

Now, we can potentially do this in our controllers:

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

class HomeController extends AbstractController
{
    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->render('home/index', [
            'title' => 'This is a title for Homepage'
        ]));

        /**
         * or you can use the following method
         * return new HtmlResponse($this->getView()->render('home/index', [
         *     'title' => 'This is a title for Homepage'
         * ]));
         * or you can use the following method
         * return new HtmlResponse($this->view->render('home/index', [
         *     'title' => 'This is a title for Homepage'
         * ]));
         */
    }
}

// src/Controller/AboutController.php
<?php

declare(strict_types=1);

namespace App\Controller;

class AboutController extends AbstractController
{
    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->render('about/index', [
            'title' => 'This is a title for About Us'
        ]));

        /**
         * or you can use the following method
         * return new HtmlResponse($this->getView()->render('about/index', [
         *     'title' => 'This is a title for About Us'
         * ]));
         * or you can use the following method
         * return new HtmlResponse($this->view->render('about/index', [
         *     'title' => 'This is a title for About Us'
         * ]));
         */
    }
}
```

But you see there is a problem here. Assume you have two controllers, AuthController and HomeController. In AbstractController, you have `$this->view`, and `$this->session` objects. Now, you will use `$this->view` in both the controllers, but HomeController won't need `$this->session` object. So, technically, what you're doing by using AbstractController is that even when a controller does not need a session object, it will still have access to `$this->session`. So, session is instantiated even though it is not needed.

This creates unnecessary overhead and violates the principle of "only take what you need." With dependency injection, AuthController would inject both view and session services, while HomeController would only inject the view service. Each controller gets exactly what it requires, nothing more, nothing less.

Also, if you have 10 controllers and only 3 need session functionality, AbstractController would still instantiate session objects for all 10 controllers. That's wasteful. DI ensures each controller only carries the dependencies it actually uses, making your application more efficient and thus your code more explicit about its requirements.

In the next chapter, I will cover the most important thing IMO - [Inversion of Control](./8-inversion-of-control.md).
