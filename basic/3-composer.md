---
layout: default
previous: 2-vanilla-router
next: 4-routing-package
---

# Composer and Packagist

Now, it's time to get serious. Composer, if you don't know, is a package manager for PHP. It allows you to install packages from the [Packagist](https://packagist.org/) repository.

A package basically is a collection of classes/functions/code that can be reused in any application. For instance, if you find yourself copy-pasting your own classes over and over again across your projects, you can release it as a composer package instead and then install it in any project by running `composer require vendor-name/package-name`. But this tutorial is not about teaching you that.

We will use composer to install a custom routing package that will handle the application routing for us in the next lesson. But for now, let's just focus on composer and packagist.

If you go to packagist.org and search for the term "router", you will see hundreds of packages. In fact, I have my own (but not ganna share it here, you'll make fun of me 🥲).

Anyway, first you need to install composer either globally or locally in your project as `phar` (PHP Archive) file.
You will see the download instructions for windows/linux/mac on the [official site here](https://getcomposer.org/download/).

Now, remember what the goal of this project is. It's to create web apps without using a framework, but by glueing composer packages together. That's what we are going to do.

In the root of your project, create a `composer.json` file and add the following content:

```json
{
    "config": {
        "sort-packages": true,
        "optimize-autoloader": true
    }
}
```

Let me explain what is going on.

Composer relies on `composer.json` to know which packages to install. It automatically creates a composer.lock file that contains the list of installed packages and their versions. It also creates a vendor directory that contains all the installed packages.

Here in the composer.json file, we added a `config` key containing two options:

- `sort-packages` tells composer to sort the packages alphabetically
- `optimize-autoloader` tells composer to optimize the autoloading process by scanning and mapping all class files to their paths. This means PHP can directly load a class without having to search for it on the filesystem. It's especially helpful in production environments, where performance matters.

From this point onwards, we’ll start installing packages from [Packagist](https://packagist.org/) using the `composer require vendor-name/package-name` command. Each time we do this, Composer will update the `composer.json` file under the require key and install the package into a `vendor` directory.

Along with this, Composer also generates a crucial file: `vendor/autoload.php`. This file is the glue that ties everything together. By simply including it in our application's entry point (i.e., `public/index.php`), we can automatically use any class from any installed package—no more manual `require_once` statements. It’s a true game-changer.

> 🔥NOTE: Never commit vendor dir to your git repo. The sheer size of the packages will make GitHub kick you out of their platform (Just kidding, but never commit vendor dir to your git repo). Add `vendor/` to your `.gitignore` file in root of your project.

Now that we have our composer config set up, it's time to install our first package — the router — and see how it fits into our application.

👉 Head to the next lesson to [install the routing package](./4-routing-package.md) and hook it up to your app.
