---
previous: 9-configurations
next: 11-requests-responses
---

# Markdown Parsing

Look at AboutController and HomeController. They render views `home/index.twig` and `about/index.twig` respectively. What if we wanted 100s of other pages on our site? Contact page, privacy policy, etc.? Would we be creating new controllers and views for each page? No... that's not efficient.

Normally, we'd use a database to store the pages and then render them using a PageController. However, for the sake of simplicity, we are going to keep our site flat-file, i.e. use markdown files to store the pages. This is a good approach for a small site (and may be medium ones). But for a large site, you would want to use a database.

Create a directory `content` in project root and add the following two files:

```markdown
about.md
---
title: About Us
description: Learn more about this site
---

This is an about us written in markdown.
```

```markdown
contact.md
---
title: Contact Us
description: Get in touch with us
---

This is a contact us written in markdown.
```

The section between the `---` lines is called "front matter" and contains metadata about the page. In this case, we are using the `title` key to store the title of the page. You can add description, date, author, etc.

Now, we need a way to parse markdown to html. PHP does not have inbuilt markdown rendering capabilities. So, we need to use a markdown parser.

There are many markdown parsers available on packagist. We are going to use [League CommonMark](https://commonmark.thephpleague.com/) as it has everything we need (and it's the only one I have used extensively). There may be several other markdown parsers available, but I could not find a better one. If you find one or know one, fork this repo, edit this line, and send a pull request please (or just let me know).

Run `composer require league/commonmark symfony/yaml` to install the the commonmark package and symfony/yaml package to parse front matter.

Now, before I proceed, you need to know something about the league commonmark library. First, it comes with lots of inbuilt extensions that we need to enable to get it to work for our use-case. Second, we cannot do this directly in controllers since (a) it will clutter the controller we use it in (b) we may need markdown parsing in more than one controller (e.g. PageController to show individual pages and in HomeController to show a list of pages). So, we need to create a LeagueMarkdownParser service class to avoid code duplication at multiple places.

> Did you notice something above? I said "MarkdownParser service" class, not a library. You need to be aware of the distinction between a library and a service. A service is meant to contain business logic (e.g., how many items to show per page) while a library does not. A library is simply meant to achieve some task (like returning items), but how many it returns is not the library's responsibility. It will ask for the quantity, and the service class will be the one to tell the library to return a particular number of items.
> So, in our case, the league/commonmark package is the library—a generic tool for parsing Markdown. Our LeagueMarkdownParser class is the service—it will contain our application's specific logic by deciding which extensions to enable and how to configure the library to fit our exact needs

But before we proceed, I urge you to go through the commonmark library documentation, so that it becomes easier to understand what I am doing here (don't ask chatgpt for how to use league commonmark, go read the docs).

## Creating Our Markdown Parser

Create a new file `src/Service/Markdown/LeagueMarkdownParser.php` and add the following code:

```php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

use League\CommonMark\Environment\Environment;
use League\CommonMark\Extension\Attributes\AttributesExtension;
use League\CommonMark\Extension\Autolink\AutolinkExtension;
use League\CommonMark\Extension\CommonMark\CommonMarkCoreExtension;
use League\CommonMark\Extension\DisallowedRawHtml\DisallowedRawHtmlExtension;
use League\CommonMark\Extension\FrontMatter\FrontMatterExtension;
use League\CommonMark\Extension\FrontMatter\Output\RenderedContentWithFrontMatter;
use League\CommonMark\MarkdownConverter;

class LeagueMarkdownParser
{
    private MarkdownConverter $markdownConverter;

    public function __construct()
    {
        $config = [
            'attributes' => [
                'allow' => ['id', 'class', 'align'],
            ],
            'autolink' => [
                'allowed_protocols' => ['https'],
                'default_protocol' => 'https',
            ],
            'disallowed_raw_html' => [
                'disallowed_tags' => ['title', 'textarea', 'style', 'xmp', 'iframe', 'noembed', 'noframes', 'script', 'plaintext'],
            ],
        ];
        $environment = new Environment($config);
        $environment->addExtension(new AttributesExtension());
        $environment->addExtension(new AutolinkExtension());
        $environment->addExtension(new CommonMarkCoreExtension());
        $environment->addExtension(new DisallowedRawHtmlExtension());
        $environment->addExtension(new FrontMatterExtension());
        $this->markdownConverter = new MarkdownConverter($environment);
    }

    public function parse(string $markdown): array
    {
        $output = $this->markdownConverter->convert($markdown);
        $data = [];

        if ($output instanceof RenderedContentWithFrontMatter) {
            $frontMatter = $output->getFrontMatter();
            $content = (string) $output->getContent();

            // Add front matter data to result
            foreach ($frontMatter as $key => $value) {
                $data[$key] = $value;
            }
        } else {
            // Handle regular markdown without front matter
            $content = (string) $output;
        }

        $data['content'] = $content;

        return $data;
    }
}
```

So, what happened in the LeagueMarkdownParser service class?

- First, we imported all the extensions we needed.
  - AttributesExtension is used to add attributes to HTML tags. (like we can add `id` and `class` attributes to HTML tags): [see full details in docs](https://commonmark.thephpleague.com/2.7/extensions/attributes/)
  - AutolinkExtension is used to add links to raw URLs automatically: [see full details in docs](https://commonmark.thephpleague.com/2.7/extensions/autolinks/)
  - CommonMarkCoreExtension provides all the standard CommonMark features like headings, paragraphs, lists, emphasis, strong text, code blocks, links, images, etc. This is the foundation that gives us basic markdown functionality: [see full details in docs](https://commonmark.thephpleague.com/2.7/customization/environment/)
  - DisallowedRawHtmlExtension is used to prevent raw HTML from being rendered: [see full details in docs](https://commonmark.thephpleague.com/2.7/extensions/disallowed-raw-html/)
  - FrontMatterExtension is used to parse front matter: [see full details in docs](https://commonmark.thephpleague.com/2.7/extensions/front-matter/)
  - MarkdownConverter is the main class that does the actual markdown-to-HTML conversion by making use of the config and extensions we imported above.

- Second, in the constructor, we created a configuration array that defines how each extension should behave:
  - For `attributes`, we only allow `id`, `class`, and `align` attributes for security reasons
  - For `autolink`, we only allow HTTPS protocol and set it as the default to ensure secure links
  - For `disallowed_raw_html`, we block dangerous HTML tags like `<script>`, `<style>`, `<iframe>` that could be used for XSS attacks

- Third, we created an Environment object with our configuration and added all the extensions to it. The Environment is like a container that holds all the parsing rules and extensions.

- Fourth, we created a MarkdownConverter instance using our configured environment and stored it as a private property. Why private? Because other classes don't need direct access to the converter—they should use our service's `parse()` method instead. This follows the principle of encapsulation: we hide the implementation details and provide a clean, simple interface for parsing markdown.

- Finally, we created a `parse()` method that takes a markdown string and returns an array containing the converted HTML content and, if present, the front matter data.

Did you notice this part of the code above:

```php
// Add front matter data to result
foreach ($frontMatter as $key => $value) {
    $data[$key] = $value;
}
```

Typically, for the about.md we created above, the parser will return something like this:

```php
// returned value for $frontMatter variable
[
    'title' => 'About Us',
    'description' => 'Learn more about this site',
]
```

The `$content` variable will just be a parsed html string.

So, in order to get a unified array like `['title'=>'About Us', 'description'=>'Learn more about this site', 'content'=>'<p>This is an about us written in markdown.</p>']`, I created an empty array called `$data` and used a foreach loop to add each front matter key-value pair to it, then added the parsed HTML content under the 'content' key.

### Solving the Issues

Can you find the issues with our LeagueMarkdownParser service class? No? Give yourself a slap and think again.

Found them (yes, 'them', not 'it')? Good. I believe in you.

There are two crucial issues:

1. We are storing configurations in the class itself, which is not a good idea as we have discussed in the last lesson only.
2. What if we ever wanted to replace `league/commonmark` with another markdown parser? We would have to change the code in the LeagueMarkdownParser service class.

So, how do we solve these issues? Think again my boy, I keep telling you to think and come up with solutions on your own. Passively reading it won't do any good.

Anyway, ofcourse, since we have our configuration system in place, we are going to use it. Let's solve the first issue and then only move on to the next.

Create a `config/markdown.php` config file and move all the configurations there.

```php
// config/markdown.php
<?php

return [
    'attributes' => [
        'allow' => ['id', 'class', 'align'],
    ],
    'autolink' => [
        'allowed_protocols' => ['https'],
        'default_protocol' => 'https',
    ],
    'disallowed_raw_html' => [
        'disallowed_tags' => ['title', 'textarea', 'style', 'xmp', 'iframe', 'noembed', 'noframes', 'script', 'plaintext'],
    ],
];
```

Now remember what we did in configurations chapter? We did two things: create a config helper to magically fetch values from the config file and use it in our service class. Second, we also created a ConfigFetcher library (not service; remember the diff between service and library?) that would fetch the value from the config file via DI.

You're free to use the config helper if you want, but I suggest you use the library since we worked so hard to learn DI.

```php
// src/Service/Markdown/LeagueMarkdownParser.php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

use App\Library\Config\ConfigInterface;
use League\CommonMark\Environment\Environment;
use League\CommonMark\Extension\FrontMatter\Output\RenderedContentWithFrontMatter;
use League\CommonMark\MarkdownConverter;

class LeagueMarkdownParser
{
    private MarkdownConverter $markdownConverter;

    public function __construct(
        private ConfigInterface $configInterface
    ) {
        $config = $this->configInterface->get('markdown');
        $environment = new Environment($config);
        $environment->addExtension(new AttributesExtension());
        $environment->addExtension(new AutolinkExtension());
        $environment->addExtension(new CommonMarkCoreExtension());
        $environment->addExtension(new DisallowedRawHtmlExtension());
        $environment->addExtension(new FrontMatterExtension());
        $this->markdownConverter = new MarkdownConverter($environment);
    }

    public function parse(string $markdown): array
    {
        $output = $this->markdownConverter->convert($markdown);
        $data = [];

        if ($output instanceof RenderedContentWithFrontMatter) {
            $frontMatter = $output->getFrontMatter();
            $content = (string) $output->getContent();

            // Add front matter data to result
            foreach ($frontMatter as $key => $value) {
                $data[$key] = $value;
            }
        } else {
            // Handle regular markdown without front matter
            $content = (string) $output;
        }

        $data['content'] = $content;

        return $data;
    }
}
```

But but but... we did not solve the issue in its entirety, did we? What if we wanted to add more extensions? Do we keep on changing the parser service? No. We move the extensions to config as well. Let's see how.

```php
// config/markdown.php
<?php

use League\CommonMark\Extension\Attributes\AttributesExtension;
use League\CommonMark\Extension\Autolink\AutolinkExtension;
use League\CommonMark\Extension\CommonMark\CommonMarkCoreExtension;
use League\CommonMark\Extension\DisallowedRawHtml\DisallowedRawHtmlExtension;
use League\CommonMark\Extension\FrontMatter\FrontMatterExtension;

return [
    'config' => [
        'attributes' => [
            'allow' => ['id', 'class', 'align'],
        ],
        'autolink' => [
            'allowed_protocols' => ['https'],
            'default_protocol' => 'https',
        ],
        'disallowed_raw_html' => [
            'disallowed_tags' => ['title', 'textarea', 'style', 'xmp', 'iframe', 'noembed', 'noframes', 'script', 'plaintext'],
        ],
    ],
    'extensions' => [
        AttributesExtension::class,
        AutolinkExtension::class,
        CommonMarkCoreExtension::class,
        DisallowedRawHtmlExtension::class,
        FrontMatterExtension::class,
    ]
];
```

What did we do here? We simply moved the logic to decide which extention to use, to the config file. Notice how we nested the original configuration under a 'config' key? This is because the Environment class expects this structure, while our extensions are a separate concern that our service handles.

Now, change the LeagueMarkdownParser service class:

```php
// src/Service/Markdown/LeagueMarkdownParser.php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

use App\Library\Config\ConfigInterface;
use League\CommonMark\Environment\Environment;
use League\CommonMark\Extension\FrontMatter\Output\RenderedContentWithFrontMatter;
use League\CommonMark\MarkdownConverter;

class LeagueMarkdownParser
{
    private MarkdownConverter $markdownConverter;

    public function __construct(
        private ConfigInterface $configInterface
    ) {
        $config = $this->configInterface->get('markdown');
        $environment = new Environment($config['config']);
        foreach ($config['extensions'] as $extension) {
            if (class_exists($extension)) {
                $environment->addExtension(new $extension());
            }
        }
        $this->markdownConverter = new MarkdownConverter($environment);
    }

    public function parse(string $markdown): array
    {
        $output = $this->markdownConverter->convert($markdown);
        $data = [];

        if ($output instanceof RenderedContentWithFrontMatter) {
            $frontMatter = $output->getFrontMatter();
            $content = (string) $output->getContent();

            // Add front matter data to result
            foreach ($frontMatter as $key => $value) {
                $data[$key] = $value;
            }
        } else {
            // Handle regular markdown without front matter
            $content = (string) $output;
        }

        $data['content'] = $content;

        return $data;
    }
}
```

See how clean our parser service is now?

But but but... yes, we did not solve the issue completely. Now you may not see it in its entirety, but the thing is that we almost always need a few extensions for the Parser to work: the CommonMarkCoreExtension and the FrontMatterExtension (for our use-case). So what do we do? Well, we go for a middle-ground. We import the CommonMarkCoreExtension and the FrontMatterExtension in the LeagueMarkdownParser service class, and keep the other not-to-important extensions to the config file. (Yeah, yeah... it's annoying to not teach you everything in one go, but I want you to develop problem thinking skills!)

Let's make the final round of changes to solve our issue #1 (configuration):

```php
// config/markdown.php
<?php

use League\CommonMark\Extension\Attributes\AttributesExtension;
use League\CommonMark\Extension\Autolink\AutolinkExtension;
use League\CommonMark\Extension\DisallowedRawHtml\DisallowedRawHtmlExtension;

return [
    'config' => [
        'attributes' => [
            'allow' => ['id', 'class', 'align'],
        ],
        'autolink' => [
            'allowed_protocols' => ['https'],
            'default_protocol' => 'https',
        ],
        'disallowed_raw_html' => [
            'disallowed_tags' => ['title', 'textarea', 'style', 'xmp', 'iframe', 'noembed', 'noframes', 'script', 'plaintext'],
        ],
    ],
    'extensions' => [
        AttributesExtension::class,
        AutolinkExtension::class,
        DisallowedRawHtmlExtension::class,
    ]
];

// src/Service/Markdown/LeagueMarkdownParser.php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

use App\Library\Config\ConfigInterface;
use League\CommonMark\Environment\Environment;
use League\CommonMark\Extension\CommonMark\CommonMarkCoreExtension;
use League\CommonMark\Extension\FrontMatter\FrontMatterExtension;
use League\CommonMark\Extension\FrontMatter\Output\RenderedContentWithFrontMatter;
use League\CommonMark\MarkdownConverter;

class LeagueMarkdownParser
{
    private MarkdownConverter $markdownConverter;

    public function __construct(
        private ConfigInterface $configInterface
    ) {
        $config = $this->configInterface->get('markdown');
        $environment = new Environment($config['config']);
        $environment->addExtension(new CommonMarkCoreExtension());
        $environment->addExtension(new FrontMatterExtension());
        foreach ($config['extensions'] as $extension) {
            if (class_exists($extension)) {
                $environment->addExtension(new $extension());
            }
        }
        $this->markdownConverter = new MarkdownConverter($environment);
    }

    public function parse(string $markdown): array
    {
        $output = $this->markdownConverter->convert($markdown);
        $data = [];

        if ($output instanceof RenderedContentWithFrontMatter) {
            $frontMatter = $output->getFrontMatter();
            $content = (string) $output->getContent();

            // Add front matter data to result
            foreach ($frontMatter as $key => $value) {
                $data[$key] = $value;
            }
        } else {
            // Handle regular markdown without front matter
            $content = (string) $output;
        }

        $data['content'] = $content;

        return $data;
    }
}
```

However, if you try to use this parser service, you'll get an error saying LeagueMarkdownParser expects one argument, but got 0. You can easily solve it by registering the LeagueMarkdownParser service in `config/dependencies.php`, but be patient, we will solve it soon.

Anyway, let's move on to the next issue: what happens if we ever want to replace `league/commonmark` with another markdown parser? We CAN potentially change the code in the LeagueMarkdownParser service class, but then we will encounter the same issue that we learned about in templates chapter (remember, twig and plates?)

So, yeah, we are going to create a MarkdownParserInterface and register it in DI config.

Create a `src/Service/Markdown/MarkdownParserInterface.php`:

```php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

interface MarkdownParserInterface
{
    public function parse(string $markdown): array;
}
```

Make sure that the LeagueMarkdownParser class implements the MarkdownParserInterface:

```php
<?php

declare(strict_types=1);

namespace App\Service\Markdown;

use App\Library\Config\ConfigInterface;
use League\CommonMark\Environment\Environment;
use League\CommonMark\Extension\CommonMark\CommonMarkCoreExtension;
use League\CommonMark\Extension\FrontMatter\FrontMatterExtension;
use League\CommonMark\Extension\FrontMatter\Output\RenderedContentWithFrontMatter;
use League\CommonMark\MarkdownConverter;

class LeagueMarkdownParser implements MarkdownParserInterface
{
    // ... code remains the same
}
```

Now, register the MarkdownParserInterface in `config/dependencies.php`:

```php
<?php

use App\Library\Config\ConfigInterface;
use App\Library\Config\PHPConfigFetcher;
use App\Library\View\RendererInterface;
use App\Library\View\TwigRenderer;
use App\Service\Markdown\LeagueMarkdownParser;
use App\Service\Markdown\MarkdownParserInterface;

return [
    RendererInterface::class => TwigRenderer::class,
    // or RendererInterface::class => \App\Library\View\PlatesRenderer::class

    ConfigInterface::class => PHPConfigFetcher::class,

    MarkdownParserInterface::class => LeagueMarkdownParser::class
];
```

Okay, we have solved the second issue. How do we test if the current setup works? Well, let's implement dynamic pages on our site to see if it works and finish our basic application.

## Dynamic Pages

Now, you know that we will be using markdown files to store site pages. So that makes the current AboutController obsoltete. Remove the AboutController (or keep it if you want; but don't forget to deregister the /about route in `config/routes.php` then).

Let's create a `src/Controller/PageController.php` controller and its `templates/twig/page/show.twig` view to handle dynamic pages.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\Markdown\MarkdownParserInterface;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class PageController
{

    public function __construct(
        private MarkdownParserInterface $markdown,
        private RendererInterface $view
    ) {
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        $slug = $request->getAttribute('slug');
        $markdownFile = __DIR__ . '/../../content/' . $slug . '.md';

        if (!file_exists($markdownFile)) {
            return new HtmlResponse($this->view->render('404'), 404);
        }

        $page = $this->markdown->parse(file_get_contents($markdownFile));

        return new HtmlResponse($this->view->render('page/show', [
            'title' => $page['title'],
            'description' => $page['description'],
            'page' => $page
        ]));
    }

}
```

```twig
<!-- templates/twig/page/show.twig -->
{% raw %}
{% extends 'layouts/default.twig' %}

{% block content %}

<h1>{{ page.title }}</h1>

<p>{{ page.description }}</p>

<article>
    {{ page.content|raw }}
</article>

{% endblock %}

<!-- templates/twig/404.twig -->
{% raw %}
<!DOCTYPE html>
<html lang="en">
 <head>
  <meta charset="UTF-8">
  <title>404 Not Found</title>
  <style>
   body {
    font-family: sans-serif;
    text-align: center;
    padding: 50px;
    background: #f8f9fa;
   }
   h1 {
    font-size: 60px;
    margin-bottom: 10px;
    color: #dc3545;
   }
   p {
    font-size: 20px;
    color: #6c757d;
   }
   a {
    display: inline-block;
    margin-top: 20px;
    text-decoration: none;
    color: #007bff;
   }
   a:hover {
    text-decoration: underline;
   }
  </style>
 </head>
 <body>
  <h1>404</h1>
  <p>Page Not Found</p>
  <a href="/">Go back to homepage</a>
 </body>
</html>
{% endraw %}
```

Do not forget to add the routes.

```php
// config/routes.php
<?php

return [
    // route name => [route method, route path, controller::method]
    'home' => ['get', '/', '\App\Controller\HomeController::index'],
    // 'about' => ['get', '/about', '\App\Controller\AboutController::index'],
    'page' => ['get', '/{slug}', '\App\Controller\PageController::show'],
];
```

So, what happened above?

- in the page controller we created, the page controller screams/declares that it uses MarkdownParserInterface and the RendererInterface. It does not know or care which class implementing the said interfaces will be passed to it. The DI container we have setup checks `config/dependencies.php` for the interfaces and their implementations and wires them up, and during controller instantiation in our front controller, it passes the implementations to the controller.
- We created a show method (which receives `$request` object from the front controller during method call)
- In the show method, it fetches the value of passed slug to the route (e.g. "about" if a user visits <http://localhost:8080/about> path on our app).
- We use this slug to see if a markdown file with that basename exists. If it does not, we simply return a 404 error response.
- If it does, we use our markdown parse to parse the markdown file (after getting its contents as string; remember that method parse() in MarkdownParserInterface accepts a string)
- The parsed markdown file is then passed to the view to render a HTML.
- This HTML string is then used to generate response via HtmlResponse class instance.
- If you pay attention to the show method, it expects a `ResponseInterface` as a return type. `]Laminas\Diactoros\Response\HtmlResponse` is an implementation of ResponseInterface, so are `\Laminas\Diactoros\Response\JsonResponse` and `\Laminas\Diactoros\Response\RedirectResponse` classes.
- Now notice the twig template file we created. In the `{{page.content|raw}}` tag, we use the raw filter to output the parsed HTML string as it is. Because if we don't, Twig's automatic security feature will escape the HTML. This means instead of seeing a formatted paragraph, your users would see the actual HTML tags printed on the screen, like `<p>This is an about us written in markdown.</p>`."

Go check the browser (don't forget to start the local server). Individual pages shoud work as expected (/about, /contact).

Time to modify HomeController to display list of all pages. I am not going to complicate things for now by introducing pagination and stuff; let's leave that for the advanced tutorial.

```php
// src/Controller/HomeController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\Markdown\MarkdownParserInterface;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ServerRequestInterface;

class HomeController
{
    public function __construct(
        private MarkdownParserInterface $markdown,
        private RendererInterface $view
    ) {
    }

    public function index(ServerRequestInterface $request)
    {

        $pages = glob(__DIR__ . '/../../content/*.md');

        $data = [];
        foreach ($pages as $page) {
            $slug = basename($page, '.md');
            $page = $this->markdown->parse(file_get_contents($page));
            $data[] = [
                'title' => $page['title'],
                'description' => $page['description'],
                'slug' => $slug
            ];
        }

        return new HtmlResponse($this->view->render('home/index', [
            'title' => 'This is a title for Homepage!',
            'pages' => $data
        ]));
    }
}
```

```twig
<!-- src/templates/twig/home/index.twig -->
{% raw %}
{% extends 'layouts/default.twig' %}

{% block content %}
Welcome to <span class="twig">Twig</span>!

<h2>Top Pages</h2>

<ul>
    {% for page in pages %}
    <li><a href="/{{ page.slug }}">{{ page.title }}</a>: {{ page.description }}</li>
    {% endfor %}
</ul>
{% endblock %}

{% block styles %}
<style>
    .twig {
        color: lime;
    }
</style>
{% endblock %}
{% endraw %}
```

Now, let me explain the HomeController index method breifly. We used glob function to fetch all markdown files inside content dir. Then, we looped through them and parsed them one by one, and added their title, slug, and description to the `$data` array. Finally, we used the `$data` array to render the HTML template file. That's it.

### Improving the Controllers

While this much is enough to get job done, there is a minor problem. Let us assume that we ever wanted to expose an API endpoint to list pages and individual pages so that the API can be used in other apps (External; like a VueJS or Sveltekit frontend). Would we not need to create separate controllers and repeat the same code we just did here?

So, what is a better approach to resolve such an issue? Of course... you are on the right track - we create a new PageFetcher service class to fetch pages. That way, we can use the same service in both current controller and API controllers if we ever needed to.

Let's do that.

```php
// src/Service/PageFetcher.php
<?php

declare(strict_types=1);

namespace App\Service;

use App\Library\Config\ConfigInterface;
use App\Service\Markdown\MarkdownParserInterface;

class PageFetcher
{

    private string $markdownPath;

    public function __construct(
        private ConfigInterface $config,
        private MarkdownParserInterface $markdown
    ) {
        $this->markdownPath = $config->get('markdown.content_dir'); // prevent hardcoding path
    }

    public function fetchAll(): array
    {
        $files = glob($this->markdownPath . '*.md');

        $data = [];
        foreach ($files as $file) {
            $slug = basename($file, '.md');
            $page = $this->markdown->parse(file_get_contents($file));
            $data[] = [
                'title' => $page['title'],
                'description' => $page['description'],
                'slug' => $slug
            ];
        }

        return $data;
    }

    public function fetchSingle(string $pageName): array
    {
        $path = $this->markdownPath . $pageName . '.md';

        if (!file_exists($path)) {
            return [];
        }

        return $this->markdown->parse(file_get_contents($path));
    }

}
```

Ok... so what did we do in the PageFetcher service?

- First thing we did was to define the path to the content dir so that we can make use of it. Note that I did not hardcode the path as we have a working configuration system via DI that we can use to fetch path to content dir via the config file.
- Then, we made the class declare that it relies on two classes that implement the declared interfaces: ConfigInterface and MarkdownParserInterface. Once again I will repeat: the PageFetcher service does not know which class DI will pass to it, nor does it care. It simply relies on the methods defined in relevant interfaces to work.
- In the constructor, we used ConfigInterface to fetch the path to the content dir.
- We defined two methods: fetchAll() and fetchSingle(). fetchAll() will return an array of all pages and fetchSingle() will return an array of a single parsed page.
- I will not be explaining the contents of each methods since I already did that when we used the same code in controllers.

Don't forget to set the content_dir in config/markdown.php as well.

```php
// config/markdown.php
<?php

use League\CommonMark\Extension\Attributes\AttributesExtension;
use League\CommonMark\Extension\Autolink\AutolinkExtension;
use League\CommonMark\Extension\DisallowedRawHtml\DisallowedRawHtmlExtension;

return [
    'content_dir' => __DIR__ . '/../content', // <-- add this in config
    'config' => [..],
    // ...
];
```

I wanted to skip telling you about the part below so that when you get an error in app you could figure the issue and fix it on your own. But then I decided to cover it since it is a bit diff from how we defined services earlier to this point.

Make the following changes to `config/dependencies.php`.

```php
// config/dependencies.php
<?php

use App\Library\Config\ConfigInterface;
use App\Library\Config\PHPConfigFetcher;
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
];
```

Noticed something? Up until now we have defined interfaces and told the DI that if a class requests that particular interface, give them the defined concrete class. But in case of PageFetcher class, we did not define any interface. Hence told the DI that if some class wants PageFetcher, give them the PageFetcher class. Our DI setup in frontcontroller expects "alias" => "value" format. See:

```php
# register services
$dependencies = require_once __DIR__ . '/../config/dependencies.php';
foreach ($dependencies as $key => $value) {
    $container->alias($key, $value);
}
```

Even though we could refactor it, I will keep it as is for now. Will cover better approaches in advanced turorial.

Now that this is clear, edit the controllers to make use of the PageFetcher service.

```php
// src/Controller/HomeController.php
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

// src/Controller/PageController.php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Library\View\RendererInterface;
use App\Service\Markdown\MarkdownParserInterface;
use App\Service\PageFetcher;
use Laminas\Diactoros\Response\HtmlResponse;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

class PageController
{

    public function __construct(
        private PageFetcher $pageFetcher,
        private RendererInterface $view
    ) {
    }

    public function show(ServerRequestInterface $request): ResponseInterface
    {
        $slug = $request->getAttribute('slug');

        $page = $this->pageFetcher->fetchSingle($slug);

        if (!$page) {
            return new HtmlResponse($this->view->render('404'), 404);
        }

        return new HtmlResponse($this->view->render('page/show', [
            'title' => $page['title'],
            'description' => $page['description'],
            'page' => $page
        ]));
    }
}
```

And... that's it. You have a working application now. Go check it out!

I've intentionally left the controller changes for you to analyze. Take a moment to compare the before and after code - you'll notice how much cleaner and more focused each controller became once we extracted the PageFetcher service.

This is exactly the kind of architectural thinking I've been trying to teach throughout this tutorial: recognizing patterns, identifying opportunities for abstraction, and building maintainable systems. The code changes here follow the same principles we've applied throughout - can you spot them?

If you get stuck, don't worry - the important thing is developing that analytical mindset. That's what separates good developers from great ones. I want you to learn HOW TO THINK before you learn HOW TO CODE.

In the next chapter, I will cover one last thing before the wrapup... [responses](./11-responses.md).
