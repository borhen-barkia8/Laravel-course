---
title: "01 - The Big Picture"
tags:
  - laravel
  - course
  - fundamentals
aliases:
  - Laravel Big Picture
  - PHP MVC and Laravel
---

# The Big Picture: PHP, MVC, and Laravel

> [!abstract] In one paragraph
> Laravel is a PHP framework for building web applications. It gives your project a fixed structure: a URL enters through one public file, Laravel starts the application, filters the request through middleware, finds a matching route, calls a controller, optionally uses a model to read or write data, and returns a view or another HTTP response. This note explains the concepts behind that sentence and the folders that support it.

## 1. Laravel is still PHP

PHP is an **interpreted, server-side** language. The browser never executes PHP; the server does. The result sent to the browser is usually HTML, JSON, or a redirect.

A plain PHP page can mix PHP and HTML:

```php
<h1>Bonjour</h1>
<?php
$nom = "Sami";
echo "<p>Bienvenue, $nom</p>";
?>
```

Laravel does not replace PHP. It organises PHP into classes and gives those classes a shared structure. When you write a Laravel controller, you are writing a PHP class. When you write a Blade view, you are writing a template that becomes PHP behind the scenes.

> [!note] Practical consequence
> Everything you already know about PHP — variables, arrays, functions, classes, namespaces, `use` statements — still applies. Laravel adds conventions and ready-made components on top.

## 2. Why a framework exists

A framework is a set of tools, components, and conventions that speeds up application development. Without one, every project would re-implement the same infrastructure:

| Need | What a framework provides |
|---|---|
| Routing | Map a URL to a piece of code and extract parameters |
| Data access | An ORM instead of hand-written SQL everywhere |
| Views | Reusable templates and layout inheritance |
| Validation | Check incoming data and show errors |
| Security | CSRF protection, output escaping, prepared queries, password hashing |
| Common features | Authentication, sessions, emails, file uploads, scheduled tasks, tests |

Writing all of this yourself is possible. Writing it correctly and maintaining it for years is much harder.

> [!warning] A framework is not these things
> - **Not an IDE.** VS Code is an editor, not a framework.
> - **Not a CMS.** WordPress is an application you configure; Laravel is a toolkit you program.
> - **Not a library.** A library is a set of functions you call. A framework also imposes a structure and a way of writing code.

## 3. MVC: the architecture Laravel follows

MVC separates an application into three responsibilities.

![[MVC Architecture.excalidraw|100%]]

| Part | Responsibility | In Laravel |
|---|---|---|
| **Controller** | Receives the request, decides what to do, prepares the response | `app/Http/Controllers` |
| **Model** | Represents data and the business rules around it | `app/Models` |
| **View** | Formats what the user sees | `resources/views` |

The important word is **separation**. A controller should not contain HTML, and a view should not contain SQL. Each layer has one job.

> [!tip] Why beginners should care
> MVC is not bureaucracy. It means you can change the HTML without touching the database logic, or change a query without rewriting the page. It also makes code testable.

## 4. What Laravel is, exactly

Laravel is an open-source PHP framework it provides:

- **MVC architecture** with a conventional folder structure.
- **Eloquent**, an ORM for database access.
- **Blade**, a template engine for views.
- **Routing**, middleware, validation, authentication, queues, caching, and testing tools.
- **Security defaults**: CSRF protection, automatic escaping in views, prepared statements, password hashing.

## 5. The project structure that matters

A Laravel project has many folders. For this course, these are the ones you need to recognise:

| Path                   | Role                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `app/Http/Controllers` | Controller classes: the application's request logic          |
| `app/Http/Middleware`  | Middleware classes: filters around requests                  |
| `app/Models`           | Eloquent model classes: data and relationships               |
| `routes/web.php`       | URL-to-code mapping for browser requests                     |
| `resources/views`      | Blade templates (`.blade.php`)                               |
| `database/migrations`  | Table definitions and schema changes, written in PHP         |
| `public`               | The only folder exposed to the browser; contains `index.php` |
| `config`               | Application configuration files                              |
| `bootstrap`            | Application startup: routing, middleware, exception handling |
| `storage`              | Generated files: compiled views, cache, logs, uploads        |
| `tests`                | Automated tests                                              |
| `vendor`               | Laravel and third-party packages installed for the project   |

> [!important] The public folder is the front door
> A browser can only reach files inside `public`. That is why every request goes through `public/index.php`, whatever the URL. The rest of the application code stays outside the web root.

A typical feature touches four places:

```text
routes/web.php                  ← URL is declared here
app/Http/Controllers/…          ← request is handled here
app/Models/…                    ← data is read or written here
resources/views/…               ← HTML is produced here
```

## 6. The lifecycle of a request

This is the most important sequence in Laravel. Learn it once, and every file in the project has an obvious place.

| Step | What happens                                                                         | Where                                             |
| ---- | ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| 1    | The web server receives the request and always forwards it to the same file          | `public/index.php`                                |
| 2    | Laravel starts: it loads configuration, environment values, and application services | `bootstrap/app.php`                               |
| 3    | The request passes through a series of filters called middleware                     | `app/Http/Middleware` and bootstrap configuration |
| 4    | Laravel scans the declared routes and keeps the **first** one matching the URL       | `routes/web.php`                                  |
| 5    | The controller method attached to that route runs                                    | `app/Http/Controllers`                            |
| 6    | The returned value becomes an HTTP response sent to the browser                      | response                                          |

![[Request Lifecycle.excalidraw|1000]]

> [!warning] "First matching route" is literal
> Laravel does not search for the most specific route. It reads `routes/web.php` from top to bottom and stops at the first match. Chapter [[02 - Routing]] shows the classic trap this creates.

## 7. A first look at the three layers

You do not need to understand every detail yet. Notice how the same feature is split across three files.

### Route:routes/web.php

```php
use App\Http\Controllers\WelcomeController;

Route::get('/', [WelcomeController::class, 'index']);
```

### Controller:app/Http/Controllers/WelcomeController.php

```php
<?php

namespace App\Http\Controllers;

class WelcomeController extends Controller
{
    public function index()
    {
        return view('home');
    }
}
```

### View:resources/views/home.blade.php

```php
<h1>Bienvenue</h1>
<p>Ceci est la page d'accueil.</p>
```

The route says *which URL*. The controller says *what to do*. The view says *what to display*. The next chapters examine each layer in isolation, then put them back together.

## 8. How to read Laravel code

Three PHP features appear constantly:

```php
<?php

namespace App\Http\Controllers;          // 1. Where this class lives

use App\Models\Contact;                  // 2. Which other class it uses

class ContactController extends Controller // 3. What it inherits from
{
    // ...
}
```

| Element | Meaning |
|---|---|
| `namespace` | The logical path of the class; it matches the folder under `app` |
| `use` | Imports a class so you can write `Contact` instead of the full path |
| `extends` | Inherits behaviour from a parent class |

> [!note] Namespaces are not optional decoration
> `App\Http\Controllers` corresponds to `app/Http/Controllers`. If the namespace and the folder disagree, Laravel will not find your class.

## 9. What comes next

The lifecycle gives you the map. The next note zooms into step 4: how a URL is turned into a route, and how a route reaches a controller.

→ Continue with [[02 - Routing]].
