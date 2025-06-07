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

Then run `composer require league/plates` in the root to install League Plates. You can install [Twig](https://twig.symfony.com/), [Nette Latte](https://latte.nette.org/), [Aura View](https://github.com/auraphp/Aura.View), [davanich view renderer](https://github.com/devanych/view-renderer), [Mustance](https://github.com/bobthecow/mustache.php) the good old [Smarty](https://www.smarty.net/) or any other template engine you like, even [Laravel Blade](https://github.com/jenssegers/blade).

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

Note that we are creating separate directories for each controller (home for HomeController, about for AboutController [to be created below]) and separated directories for layout and partials (includes). This is not a strict requirement, but a good practice. Your call, structure your templates directory however the fuck you want, but always keep things clean.

One more thing, I am not a designer/frontend dev. So if the app looks ugly by design, blame yourself. Go get me a tea.

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
```

Let's add AboutController and relevant route as well.

```php
```
