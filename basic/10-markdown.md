---
previous: 9-configurations
next: 11-wrapup
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

Now, before I proceed, you need to know something about the league commonmark library. First, it comes with lots of inbuilt extensions that we need to enable to get it to work for our use-case. Second, we cannot do this directly in controllers since (a) it will clutter the controller we use it in (b) we may need markdown parsing in more than one controller. So, we need to create a LeagueMarkdownParser service class to avoid code duplication at multiple places.

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
