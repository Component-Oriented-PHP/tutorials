---
previous: 8-inversion-of-control
---

# Configuration and Helpers

Take a look at our TwigRenderer's constructor. Go on. Don't be shy.

```php
// from src/Library/View/TwigRenderer.php
    public function __construct()
    {
        $loader = new FilesystemLoader(__DIR__ . '/../../../templates/twig');
        $this->renderer = new Environment($loader, [
            'debug' => true, // <-- hardcoded value
            'cache' => __DIR__  . '/../../../tmp/cache', // <-- hardcoded value
        ]);
    }
```

Remember how we set Whoops up for Error Handling? We were in a similar situation. We needed the value of environment. So, first we created a hardcoded value for $environment variable and then we installed a dotenv package to load environment variables from a `.env` file.

To begin with, we can use getenv('APP_ENV') value to decide the value of debug. But, what happens if we ever needed to change the value of cache dir? We'd need to change the cache dir in the class itself. Just so you know, this is not the right approach. Why?

- You see, every time you need to change a configuration value, you have to modify the source code, retest, and redeploy.
- Different developers working on your project might want different cache directories or debug settings without modifying the core code.
- Sensitive configuration (like database credentials or API Keys) shouldn't be hardcoded in version-controlled files.

That's why we use a configuration files like `.env`. But .env file is not suitable for all sorts of configurations, especially the ones like dependencies.php and routes.php we created in earlier chapters.

Now, let's take a practical scenario. You see how our view files have titles? There is a layout file that has a titl variable. I want to pass the site name (say, "COPHP") after the current page title. How do we do that?

Let's first modify the twig layout file.

```twig
{% raw %}
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }} - COPHP</title> <!--modify this-->

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css">

    {% block styles %}{% endblock %}
</head>
{% endraw %}
```

Check in the browser if the site name shows up. It does? Good job. Go get some candy 🍭.

Now now, we know that view files MUST NOT BE responsible for having hard coded values. Controllers are the ones responsible for handling data. So, instead we should use a variable (e.g. site_name) in our view files while passing the value for that variable in our controller. Let's do that.

```twig
{% raw %}
// templates/twig/layouts/default.twig
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }} - {{site_name}}</title> <!--modify this-->

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css">

    {% block styles %}{% endblock %}
</head>
{% endraw %}
```

```php
// in src/Controller/HomeController.php only; do not edit AboutController yet
    public function index(ServerRequestInterface $request)
    {
        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'site_name' => 'COPHP'
        ]));
    }
```

Go on and check homepage and the about page. You will notice that the site title is visible on Homepage, but not the about page. Why? Because we did not pass the value of `site_name` variable in the AboutController.

You see the issue? We'd need to pass the value of `site_name` variable in the AboutController and all other controllers we may ever create. That's bad and a lot of work. So, what's the solution? Think with me...

We could create a php array file `config/app.php` containing the value for site_name. But then, remember, we'd again need to include the app.php config file in every controller and then pass the value. That's troublesome.

How do we solve that? Hmm... have you used Laravel before? You know it has a magic function called `config()` that fetches the value from a config file. So if we had that we could maybe use config('app.name') in controllers to pass site_name.

Let's first create our `config/app.php` file.

```php
<?php

return [
    'name' => 'COPHP',
    'url' => 'https://cophp.com'
];
```

Now, to use this, we need a function called `config()`. Where do we place it? Let's create a file called `src/Helpers.php` and add the following code in it:

```php
// src/Helpers.php
<?php

declare(strict_types=1);

/**
 * config function
 */
function config(string $key): mixed
{
    static $configs = [];

    $parts = explode('.', $key);
    $file = array_shift($parts);

    // Load config file if not cached
    if (!isset($configs[$file])) {
        $path = __DIR__ . "/../config/{$file}.php";
        $configs[$file] = file_exists($path) ? require $path : [];
    }

    $value = $configs[$file];
    foreach ($parts as $part) {
        $value = $value[$part] ?? null;
        if ($value === null)
            break;
    }

    return $value;
}
```

So, what's going on above?

- well, to begin with, our function config expects a single argument that has to be a string. But the fetched value can be anything (mixed)
- next, we have `static $configs = []` to cache the config values. We use a static array to cache loaded config files in memory for performance, i.e. each config file is only read from disk once per request, even if accessed multiple times
- next, to support dot notation (like `app.name`), we splits the key by dots, uses the first part as the filename, and loads the corresponding PHP file from the `config/` directory
- in the foreach loop, we traverse through the remaining dot-separated parts to access nested array values in the configuration data

Now to use we need to include it (Helpers.php file) somewhere. But we are not going to be including it in front controller. Instead, we will include it in composer.json file. Modify the composer.json file:

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
        "laminas/laminas-diactoros": "^3.6",
        "league/plates": "^3.6",
        "rdlowrey/auryn": "^1.4",
        "twig/twig": "^3.21"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        },
        "files": [
            "src/Helpers.php"
        ]
    }
}
```

Run `composer dumpautoload`.

Now, we can use `config('app.name')` in our controllers.-

But wait... it did not solve the issue of having to write `config('app.name')` in every controller. So why did this? Well, it seems useless right now, but we can use it in front controller to access routes like config('routes') and dependencies like config('dependencies') instead of having to include the respective files.

However, that's not the biggest issue right now. You may ask, that we covered DI in last lesson. So, should we not be using Container to handle config values? Yes, we should. My purpose of creating a helper was to demonstrate how you can have your own magic if you want. But I am not a magic lover. I prefer explicitness, so should you. Now, I won't be using config function in the app, but rather rely on our DI. However, I am not stopping you from using it.

Let's see how we can handle configurations via DI.

Create two files called `./src/Library/Config/PHPConfigFetcher.php` and `./src/Library/Config/ConfigInterface.php`. ConfigInterface is there to ensure that just in case you want to use ini or yaml files for configuration, you can just create another library that implements the interface.

```php
// src/Library/Config/ConfigInterface.php
<?php

declare(strict_types=1);

namespace App\Library\Config;

interface ConfigInterface
{
    public function get(string $key): mixed;
}

// src/Library/Config/PHPConfigFetcher.php
<?php

declare(strict_types=1);

namespace App\Library\Config;

class PHPConfigFetcher implements ConfigInterface
{
    public function get(string $key): mixed
    {
        static $configs = [];

        $parts = explode('.', $key);
        $file = array_shift($parts);

        // Load config file if not cached
        if (!isset($configs[$file])) {
            $path = __DIR__ . "/../../../config/{$file}.php";
            $configs[$file] = file_exists($path) ? require $path : [];
        }

        $value = $configs[$file];
        foreach ($parts as $part) {
            $value = $value[$part] ?? null;
            if ($value === null)
                break;
        }

        return $value;
    }

}
```

I just copy-pasted the code from helper function in a class method. No explanation needed here.

Now, how do we make use of it via DI? Well, we register it as a dependency in `config/dependencies.php` file.

```php
<?php

use App\Library\Config\ConfigInterface;
use App\Library\Config\PHPConfigFetcher;
use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class
];
```

Now, let's update our front controller first to make use of our new and shiny ConfigInterface instead of including direct php config files for routes.

```php
// OLD WAY
//map the route definitions
$routes = require_once __DIR__ . '/../config/routes.php';
foreach ($routes as $name => $route) {
    // ... whatever code here is
}

// NEW WAY
$routes = $container->make(\App\Library\Config\ConfigInterface::class)->get('routes');
foreach ($routes as $name => $route) {
    // ... whatever code here is
}
```

Go refresh browser tab and the app should work as it did before.

However, we still haven't solved the issue of having to pass `site_name` in every controller. Let's fix that.

Refactor `TwigRenderer` to use ConfigInterface. Also, we will create a `config/templates.php` file to hold our templates configuration.

```php
// config/templates.php
<?php

<?php

return [
    'twig' => [
        'path' => __DIR__ . '/../templates/twig',
        'cache_path' => __DIR__ . '/../tmp/cache',
        'debug' => getenv('APP_ENV') === 'development' ? true : false // if environment is development, set debug to true
    ],
    'plates' => [
        'path' => __DIR__ . '/../templates/plates',
    ],
];

// src/Library/View/TwigRenderer.php
<?php

declare(strict_types=1);

namespace App\Library\View;

use App\Library\Config\ConfigInterface;
use Twig\Environment;
use Twig\Loader\FilesystemLoader;

class TwigRenderer implements RendererInterface
{
    private Environment $renderer;

    public function __construct(private ConfigInterface $config)
    {
        $loader = new FilesystemLoader($this->config->get('templates.twig.path'));
        $this->renderer = new Environment($loader, [
            'debug' => $this->config->get('templates.twig.debug'),
            'cache' => $this->config->get('templates.twig.cache_path'),
        ]);
    }

    public function render(string $template, array $data = []): string
    {
        return $this->renderer->render($template . '.twig', $data);
    }
}
```

If you're using the PlatesRenderer, you need to refactor that class accordingly.

So, what is going on now in TwigRenderer?

- Nothing special... just like Controllers, it stated that it needs any class that implements ConfigInterface.
- The Container looks for defined dependencies. Remember what we did in config/dependencies.php? We did this: `ConfigInterface::class => PHPConfigFetcher::class`... so the container passed PHPConfigFetcher class (an instance of PHPConfigFetcher) to TwigRenderer.

Okay, but how do we ensure that we don't have to repeat passing site name variable in all controllers? To achieve that, we need to modify the TwigRenderer (or PlatesRenderer) again.

```php
// src/Library/View/TwigRenderer.php
<?php

declare(strict_types=1);

namespace App\Library\View;

use App\Library\Config\ConfigInterface;
use Twig\Environment;
use Twig\Loader\FilesystemLoader;

class TwigRenderer implements RendererInterface
{
    private Environment $renderer;
    protected array $globalConfig = [];

    public function __construct(private ConfigInterface $config)
    {
        $loader = new FilesystemLoader($this->config->get('templates.twig.path'));
        $this->renderer = new Environment($loader, [
            'debug' => $this->config->get('templates.twig.debug'),
            'cache' => $this->config->get('templates.twig.cache_path'),
        ]);

        $this->globalConfig = ['app' => $this->config->get('app')];
    }

    public function render(string $template, array $data = []): string
    {
        return $this->renderer->render($template . '.twig', array_merge($this->globalConfig, $data));
    }
}
```

Refactor the layout view file to use the new variable (i.e. `app.name`).

```twig
{% raw %}
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ title }} - {{app.name}}</title>

    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css">

    {% block styles %}{% endblock %}
</head>
{% endraw %}
```

Congrats... we've got a working configuration setup now.

In the next chapter, I will cover [Markdown Parsing](./10-markdown.md) for our application's pages.
