---
previous: 4-routing-package
next: 6-dispatching-to-a-class
---

# Error Handling

Great! Now that we have a working router, let's add some error handling to our application.

In the front controller (`public/index.php`), change the following:

```php
// from
require_once __DIR__ . '/../vendor/autoload.php';

// to
require_once __DIR__ . '/../vendor/autoload'; # we are creating a deliberate typo
```

> 🔥NOTE: We are intentionally creating a typo here.

Open the browser and see what happens. You should see a "This page isn’t working" 500 error. Now, we know what the problem is. But what if we did not? This is where error handling comes in handy.

Just above composer require, add the following:

```php
error_reporting(E_ALL);
ini_set('display_errors', '1');

require_once __DIR__ . '/../vendor/autoload'; # keep the typo
```

Now go check the browser again. Now you should be seeing something like this:

```bash
Warning: require_once(I:\SOFTWARE\cophp\basic-application\public/../vendor/autoload): Failed to open stream: No such file or directory in I:\SOFTWARE\cophp\basic-application\public\index.php on line 12
```

This clearly states that the file we are requiring does not exist. Fix the typo and re-check, the error should be gone now.
