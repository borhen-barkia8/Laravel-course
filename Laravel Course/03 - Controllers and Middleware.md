---
title: "03 - Controllers and Middleware"
tags:
  - laravel
  - course
  - controllers
  - middleware
aliases:
  - Laravel Controllers
  - Laravel Middleware
---

# Controllers and Middleware

> [!abstract] In one paragraph
> A controller is a class that receives a request and produces a response. It may call models, ask a service to do work, and return a view, JSON, a redirect, or a file. Middleware sits between the request and the controller and can inspect, modify, or stop the request before the controller runs.

> [!info] The view layer now has its own course
> Blade syntax, templates, sections, and file inclusions were split out of this note and are covered in [[04 - Blade Templates]]. This note stays on the request-handling side: controllers and middleware.

## 1. What a controller is

A controller is a class that groups the request-handling logic of a feature.

| Responsibility | Example |
|---|---|
| Handle HTTP methods | `GET`, `POST`, `PUT`, `DELETE` |
| Talk to models | Read or save data |
| Return a response | A view, JSON, a redirect, or a file |
| Keep routes thin | `routes/web.php` stays a map, not a program |

Advantages:

- **Centralisation** of the logic for one feature.
- **Reusability**: several routes can call the same controller method or share helper methods.
- **Clear MVC organisation**: the route file, the request logic, and the view stay separate.

> [!note] A controller is not the model
> The controller decides *what to do*. The model represents *what data is*. A controller method might say "find contact 3 and show it"; the model knows how to find a contact in the `contacts` table.

## 2. The shape of a controller

A controller lives in `app/Http/Controllers`, declares the matching namespace, and extends Laravel's base `Controller` class.

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

| Part | Meaning |
|---|---|
| `namespace App\Http\Controllers;` | The class belongs to this folder/namespace |
| `class WelcomeController extends Controller` | It inherits common controller helpers |
| `public function index()` | A public method that can be called by a route |
| `return view('home');` | The method returns `resources/views/home.blade.php` as the response |

The matching route:

```php
use App\Http\Controllers\WelcomeController;

Route::get('/', [WelcomeController::class, 'index']);
```

> [!tip] Method names are conventions, not magic
> `index`, `create`, `store`, `show`, `edit`, `update`, and `destroy` are the names Laravel's resource routing expects. For your own routes, any public method name works.

## 3. A resource controller

A resource controller groups the seven standard CRUD actions.

```php
<?php

namespace App\Http\Controllers;

use App\Models\Produit;
use Illuminate\Http\Request;

class ProduitController extends Controller
{
    public function index()
    {
        // Display a list of products
    }

    public function create()
    {
        // Display the creation form
    }

    public function store(Request $request)
    {
        // Save a new product
    }

    public function show($id)
    {
        // Display one product
    }

    public function edit($id)
    {
        // Display the edit form
    }

    public function update(Request $request, $id)
    {
        // Update a product
    }

    public function destroy($id)
    {
        // Delete a product
    }
}
```

| Method | Job | Typical response |
|---|---|---|
| `index()` | List resources | `view('produits.index', …)` |
| `create()` | Show the creation form | `view('produits.create')` |
| `store(Request $request)` | Persist a new resource | `redirect()->route('produits.index')` |
| `show($id)` | Show one resource | `view('produits.show', …)` |
| `edit($id)` | Show the edit form | `view('produits.edit', …)` |
| `update(Request $request, $id)` | Persist changes | `redirect()->route('produits.show', $id)` |
| `destroy($id)` | Delete a resource | `redirect()->route('produits.index')` |

And the route declaration from the previous chapter:

```php
Route::resource('produits', ProduitController::class);
```

> [!warning] Resource routes expect resource methods
> If `Route::resource` declares seven routes, the controller should define the seven matching methods. A missing method produces an error when that URL is visited.

## 4. Dependency injection

Dependency injection means: instead of creating the tools a class needs, you ask for them in the constructor, and Laravel supplies them.

![[Dependency Injection.excalidraw|1000]]

### 4.1 A small service

```php
<?php

namespace App\Services;

class WelcomeService
{
    public function getWelcomeMessage()
    {
        return "Bienvenue sur notre site !";
    }
}
```

This is an ordinary PHP class. It contains reusable logic that does not belong in the controller or the view.

### 4.2 A controller that receives the service

```php
<?php

namespace App\Http\Controllers;

use App\Services\WelcomeService;

class WelcomeController extends Controller
{
    protected $welcomeService;

    public function __construct(WelcomeService $welcomeService)
    {
        $this->welcomeService = $welcomeService;
    }

    public function showWelcome()
    {
        $message = $this->welcomeService->getWelcomeMessage();

        return view('welcome', compact('message'));
    }
}
```

Laravel reads the constructor type hint `WelcomeService` and creates the object automatically. You do not write `new WelcomeService()`.

### 4.3 The route and the view

```php
use App\Http\Controllers\WelcomeController;

Route::get('/welcome', [WelcomeController::class, 'showWelcome']);
```

```PHP
<body>
    <h1>{{ $message }}</h1>
</body>
```

> [!important] Why this is worth learning early
> Constructor injection makes dependencies visible. A controller that asks for `WelcomeService` cannot silently depend on ten hidden global objects. It also makes the controller easy to test: you can give it a fake service.

## 5. Passing data from a controller to a view

A view is a template. It does not fetch data by itself; the controller passes data to it.

There are two common forms.

**Associative array:**

```php
public function show($id)
{
    $contact = Contact::find($id);

    return view('contacts.show', [
        'contact' => $contact,
        'title' => 'Contact details',
    ]);
}
```

**`compact()`:**

```php
public function show($id)
{
    $contact = Contact::find($id);

    return view('contacts.show', compact('contact'));
}
```

`compact('contact')` is shorthand for `['contact' => $contact]`.

In the Blade view, each array key becomes a variable:

```php
<h1>{{ $contact->name }}</h1>
<p>{{ $contact->email }}</p>
```

> [!warning] Only pass what the view needs
> Do not load the entire database in the controller "just in case". Pass the data the view actually displays. This keeps the controller readable and the view predictable.

> [!info] The view layer has its own course
> Blade syntax, layouts, sections, includes, and reusable view fragments are covered in [[04 - Blade Templates]]. This note stays on the controller and middleware side.

## 6. Response types

A controller method can return different kinds of responses.

**A Blade view:**

```php
return view('contacts.index', compact('contacts'));
```

**JSON (common for APIs):**

```php
return response()->json([
    'status' => 'ok',
    'count' => $contacts->count(),
]);
```

**A redirect:**

```php
return redirect()->route('contacts.index');
```

**A file download:**

```php
return response()->download(storage_path('app/report.pdf'));
```

**A custom status code and headers:**

```php
return response('Not found', 404)
    ->header('X-App', 'Laravel Course');
```

> [!tip] Choose the response based on the client
> A browser page usually wants a view or a redirect. A JavaScript front-end or mobile app usually wants JSON. The controller is the place where that decision belongs.

## 7. Middleware

Middleware is a filter that runs **around** a request. It can inspect the request, modify it, stop it, or pass it to the next layer.

Typical uses:

- Authentication: is the user logged in?
- Authorisation: does the user have the right role?
- Validation of URL parameters.
- Logging, rate limiting, locale selection.

![[Middleware Flow.excalidraw|1000]]

### 7.1 A middleware class

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class ValidateName
{
    public function handle(Request $request, Closure $next)
    {
        $name = $request->route('name');

        if (empty($name) || strlen($name) < 3 || strlen($name) > 50) {
            return redirect()
                ->route('welcome.error')
                ->withErrors(['name' => 'Le nom doit contenir entre 3 et 50 caractères.']);
        }

        return $next($request);
    }
}
```

| Piece | Meaning |
|---|---|
| `handle(Request $request, Closure $next)` | The middleware entry point |
| `$request->route('name')` | Reads the `{name}` route parameter |
| early `return` | Stops the request and sends a response immediately |
| `return $next($request)` | Continues to the next middleware or to the controller |

### 7.2 Registering the middleware

In recent Laravel versions, middleware aliases are declared in `bootstrap/app.php`:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use App\Http\Middleware\ValidateName;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'validateName' => ValidateName::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

The alias `validateName` is the short name you use on routes.

### 7.3 Applying the middleware to a route

```php
use App\Http\Controllers\WelcomeController;

Route::get('/welcome/error', function () {
    return view('welcome-error');
})->name('welcome.error');

Route::get('/welcome/{name}', [WelcomeController::class, 'showWelcome'])
    ->middleware('validateName')
    ->name('welcome');
```

> [!warning] The fixed route must come first
> `/welcome/error` is a fixed URL and must be declared **before** `/welcome/{name}`. Otherwise the dynamic route captures the word `error` as a name, and the error page is never reached. This is the same ordering rule as in [[02 - Routing]].

When a request reaches `/welcome/Ali`:

1. The route matches and its middleware runs.
2. `ValidateName` checks the `name` route parameter.
3. If the name is invalid, the user is redirected to `welcome.error`.
4. If the name is valid, `$next($request)` continues to `showWelcome`.

## 8. Controllers and middleware together

| Layer | Sees the request | Can stop the request | Typical code |
|---|---|---|---|
| Middleware | Yes, before the controller | Yes | `return redirect(…)` |
| Controller | Yes, after middleware | Yes, by returning another response | `return view(…)` |
| View | No; it only receives data | No | `{{ $contact->name }}` |

> [!important] Keep filters out of controllers
> Authentication, role checks, and repeated request checks belong in middleware. If the same `if` appears at the top of ten controller methods, it probably belongs in a middleware.

## 9. What comes next

Controllers and middleware explain how a request is handled. The view layer is now covered in the next note: Blade syntax, parameterized views, templates, sections, and includes.

→ Continue with [[04 - Blade Templates]].
