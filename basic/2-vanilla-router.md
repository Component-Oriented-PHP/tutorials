---
title: Creating a Vanilla PHP Router
---

Now that you have your front controller in place, we are going to create a simple vanilla routing system to return different responses on different routes (/about, /contact, etc).

Open `public/index.php` and add replace the code with the following:

```php
<?php

declare(strict_types=1);

$uri = $_SERVER['REQUEST_URI'];

if ($uri === '/') {
    echo 'This tuts rocks!!';
} elseif ($uri === '/about') {
    echo 'This is the about page';
} elseif ($uri === '/contact') {
    echo 'This is the contact page';
} else {
    echo '404';
}
```

Try going to `http://localhost:8080/`, `http://localhost:8080/about` and `http://localhost:8080/contact`. You should see the home page, about page and contact page we set above respectively. If you go to any route other than these three, you should see the 404 page.

> NOTE: Our simple router uses the entire request URI. In a real application, you'd need to parse the URL to separate the path from any query string parameters, but this is perfect for our simple demonstration.

Now, this is a fine start. However, let's use PHP's `match` function to make this even better, cleaner and less verbose.

```php
<?php

declare(strict_types=1);

$uri = $_SERVER['REQUEST_URI'];

echo match ($uri) {
    '/' => 'This tuts rocks!!',
    '/about' => 'This is the about page!',
    '/contact' => 'This is the contact page!',
    default => '404',
};
```

If you refresh the browser tab, you should see the same results as before (Notice the additional exclamation mark).

BUT in real web applications, such a simple router is practically useless due to the sheer amount of routes and necessity for dynamic routing like `/blog/{slug}` or `/user/{username}`.

But now, image handling this:

```php
// Imagine handling dozens of routes with match:
echo match ($uri) {
    '/' => 'Home',
    '/about' => 'About', 
    '/contact' => 'Contact',
    '/blog' => 'Blog index',
    '/blog/post-1' => 'Post 1',
    '/blog/post-2' => 'Post 2',
    '/blog/post-3' => 'Post 3',
    // ... 50+ more routes?
    default => '404',
};
// This gets unwieldy fast!
```

If you have used a framework, let's say Laravel, you know that you can do this:

```php
Route::get('/blog/{slug}', function ($slug) {
    return 'Slug :' . $slug;
});
```

That's dynamic routing. The router will automatically match the request URL to the appropriate route and execute the corresponding callback function by passing the dynamic parameter to it.

Now, one option that we have right now, is to manually modify the router to be able to handle dynamic routing. BUT... Why bother? There already are packages that can do this for you.

And we haven't even considered other common requirements like handling different request methods (GET, POST), grouping routes under a common prefix, or applying middleware. This is where a dedicated routing package becomes essential.

We will use composer to install a custom routing package that will handle the application routing for us. So, let's move on to the third lesson on [Composer](./3-composer.md).
