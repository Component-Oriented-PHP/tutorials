---
layout: default
next: 2-vanilla-router
---

# Front Controller

If you have used any modern PHP framework, you already know that they place a file `index.php` inside `public` dir and the web server is configured to point to the `public` dir.

This `index.php` file is called a front controller, since it acts as an entry point for your PHP application.

Have you ever seen the github repo of [WordPress](https://github.com/WordPress/WordPress)? Look at how every file is dumped into the root of the project and the front controller index.php is itself in the root dir. This practice was quite common in the early days of web development.

But this is a bad practice. Since the web server points to the root dir, the files are often prone to be exposed to the web if something goes wrong (we can't control everything, can we?). This is a security issue. Want that to happen to you? No? Then start using public dir and front controller right away, if you're not using it already.

Second thing, WordPress has multiple points of entry. One `index.php` in root dir, another one in wp-admin dir for the admin panel, `wp-login.php` for login/registration, etc. It only creates complexity. Yes, you can create a common file and include it in both, but why bother? Just have a single point of entry to your application for both front and back ends.

In the root dir of your projet, create a `public` directory and add an `index.php` file in it. Put the following content in it:

```php
<?php

declare(strict_types=1);

echo "This tuts rocks!";
```

Run `php -S localhost:8080 -t public` in CLI in project root.

Now open `http://localhost:8080` in your browser. You should see the message "This tuts rocks!"

You should be aware of what `php -S localhost:8080 -t public` does. It starts a development web server (comes with PHP) on port 8080 and points to the `public` dir.

Anyway, in the index.php, we have added `declare(strict_types=1);`. In case you don't know, this is a PHP directive that enables strict type checking for scalar type declarations in that file. When enabled, PHP will throw a TypeError if you pass a value of the wrong type to a function parameter or return statement that has a type declaration. We will be adding this directive to all classes (even helpers) to ensure consistency throughout our code.

For example, without strict types:

```php
function addNumbers(int $a, int $b): int { // we want integer
    return $a + $b;
}

echo addNumbers("5", "10"); // Works, outputs 15 (strings auto-converted)
```

With strict types enabled:

```php
declare(strict_types=1);

function addNumbers(int $a, int $b): int { // we want integer
    return $a + $b;
}

echo addNumbers("5", "10"); // Fatal error: TypeError; we wanted integers, but got strings
echo addNumbers(5, 10);     // Works, outputs 15; we wanted integers, we got integers
```

This helps catch type-related bugs early and makes your code more predictable. It's considered a best practice in modern PHP development, especially when building robust applications.

While most modern frameworks like Symfony and Laravel use type hinting in their classes and functions, they don't always enforce strictness for scalar types with `declare(strict_types=1)` (like Mezzio does). I personally prefer to enable strict types everywhere in my codebase. It's your choice, but I highly recommend you use it! This practice is also a key part of PSR-12 (the Extended Coding Style guide), which sets the standard for modern PHP code.

Remember that the `declare` statement must be the very first statement in the file (after the opening `<?php` tag) to take effect for the entire file. This is why we place it at the top of our front controller.

The next lesson is on [Routing](./2-vanilla-router.md).
