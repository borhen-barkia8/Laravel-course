---
title: "02 - Routing"
tags:
  - laravel
  - course
  - routing
aliases:
  - Laravel Routing
  - Routes in Laravel
---

# Routing: Turning URLs into Code

> [!abstract] In one paragraph
> Routing is the mapping between an HTTP request and the code that handles it. In Laravel, routes live in `routes/web.php`. A route combines an HTTP method (`GET`, `POST`, …) with a URL pattern and an action. The action is either a closure or a controller method. Routes can be named, can receive parameters, and are matched from top to bottom.

## 1. What a route is

A route answers one question:

> *When a request arrives for this URL with this HTTP method, which PHP code should run?*

```php
use Illuminate\Support\Facades\Route;

Route::get('/produit', function () {
    return 'Produits disponibles';
});
```

| Piece | Meaning |
|---|---|
| `Route` | The facade/class used to declare routes |
| `get` | The HTTP method being matched |
| `'/produit'` | The URL pattern |
| `function () { … }` | The action executed when the route matches |

When the browser requests `http://site.com/produit`, Laravel runs the closure and returns the string `Produits disponibles` as the HTTP response body.

> [!info] Where routes live
> Browser routes are declared in `routes/web.php`. Laravel loads this file automatically; you do not need to register it manually.

## 2. The homepage and the route file

The homepage is not special. It is simply the route whose URL is `/`:

```php
Route::get('/', function () {
    return view('welcome');
});
```

This is the default route of a new Laravel application. It returns a **view**, not a string. The view is `resources/views/welcome.blade.php`.

![[Welcome Route.excalidraw|100%]]

> [!note] view('welcome') uses a dot, not a path
> The string `welcome` means `resources/views/welcome.blade.php`. A view named `contacts.index` would mean `resources/views/contacts/index.blade.php`. The `.blade.php` extension is added by Laravel.

## 3. What a route action can return

A closure route can produce several kinds of response:

```php
// 1. A plain HTTP response body
Route::get('/hello', function () {
    return '<h1>Hello!</h1>';
});

// 2. A Blade view
Route::get('/', function () {
    return view('welcome');
});

// 3. A controller call (the usual choice for real features)
Route::get('/contacts', [ContactController::class, 'index']);
```

> [!tip] Closures are fine for tiny experiments
> In a real application, most routes delegate to a controller. This keeps `routes/web.php` readable and puts application logic in a testable class.

## 4. Named routes

A route can be given a name. Names are useful when you need to generate a URL or redirect to it.

```php
Route::get('/', function () {
    return 'Bienvenue sur le site';
})->name('home');
```

Once named, the route can be referenced by name:

```php
route('home');   // generates the URL for the route named "home"
url('/');        // generates the URL for the path "/"
```

For the route above, both produce the site's base URL, for example `http://site.com`.

> [!important] Why names matter
> A name decouples your code from the URL. If the URL later changes from `/contacts` to `/repertoire`, views and redirects that use `route('contacts.index')` keep working. Hard-coded URLs break.

## 5. Route parameters

A URL often contains variable parts: an ID, a slug, a name. Parameters are written in braces.

```php
Route::get('/user/{id}', function ($id) {
    return 'User '.$id;
});
```

Requesting `http://site.com/user/2` calls the closure with `$id = '2'`.

### 5.1 Required parameters

A required parameter must always be present in the URL:

```php
Route::get('/user/{id}', function ($id) {
    return 'User '.$id;
});
```

- `http://site.com/user/2` → works, returns `User 2`.
- `http://site.com/user` → does not match this route.

### 5.2 Optional parameters

Add `?` after the parameter name and give the closure argument a default value:

```php
Route::get('/user/{name?}', function ($name = 'Sami' //Sami is the default name) {
    return 'Hello '.$name;
});
```

- `http://site.com/user/Ali` → `Hello Ali`
- `http://site.com/user` → `Hello Sami`

> [!warning] Optional parameters go last
> You may mix required and optional parameters, but optional ones must come after required ones in the URL:
>
> ```php
> Route::get('/post/{id}/{slug?}', function ($id, $slug = null) {
>     return 'Post ID: '.$id.' Slug: '.$slug;
> });
> ```
>
> - `/post/2/my-post` → `Post ID: 2 Slug: my-post`
> - `/post/2` → `Post ID: 2 Slug:`

### 5.3 Constraining a parameter

The `where()` method restricts a parameter with a regular expression:

```php
Route::get('/{n}', function ($n) {
    return 'Je suis la page '.$n.' !';
})->where('n', '[1-3]');
```

Now `/1`, `/2`, and `/3` match; `/4` does not.

You can constrain several parameters at once:

```php
Route::get('/user/{id}/{name}', function ($id, $name) {
    return $id.' - '.$name;
})->where([
    'id' => '[0-9]+',
    'name' => '[A-Za-z]+',
]);
```

> [!caution] Constraints are not validation of business data
> `where()` decides whether a route matches at all. It is not a replacement for validating a form. If a URL should return a 404 for a non-numeric ID, a route constraint is appropriate; if a form field must contain a valid email, use validation in the controller.

## 6. Route order matters

Laravel reads `routes/web.php` from top to bottom and keeps the **first** route whose pattern matches. It does not look for the "best" match.

```php
// Trap: the first route captures everything, including /post/creer
Route::get('/post/{id}', function ($id) {
    // $id will be "creer" when the URL is /post/creer
});

Route::get('/post/creer', function () {
    // This route is never reached.
});
```

The correct order is: **fixed routes before parameterised routes**.

```php
Route::get('/post/creer', function () {
    return view('posts.create');
});

Route::get('/post/{id}', function ($id) {
    return 'Post '.$id;
});
```


> [!important] Rule of thumb
> Declare the most specific URLs first. Put fixed paths (`/post/creer`, `/post/archives`) above dynamic paths (`/post/{id}`).

## 7. Routes to controllers

Instead of a closure, a route can point to a controller method. The action is an array: `[ControllerClass::class, 'methodName']`.

```php
use App\Http\Controllers\WelcomeController;

Route::get('/', [WelcomeController::class, 'index']);
```

The controller lives in `app/Http/Controllers/WelcomeController.php`:

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

![[Route to Controller.excalidraw|100%]]

> [!note] Keeping routes/web.php tidy
> Add `use App\Http\Controllers\…;` at the top of the route file, then refer to the class by its short name. This is the same `use` mechanism you use in any PHP file.

## 8. Resource routes

Most CRUD features need the same seven actions: list, show form, store, show one, show edit form, update, delete. Laravel can register all of them with one declaration:

```php
use App\Http\Controllers\ProduitController;

Route::resource('produits', ProduitController::class);
```

This creates the following routes automatically:

| HTTP method     | URL                        | Controller method | Route name         |
| --------------- | -------------------------- | ----------------- | ------------------ |
| `GET`           | `/produits`                | `index`           | `produits.index`   |
| `GET`           | `/produits/create`         | `create`          | `produits.create`  |
| `POST`          | `/produits`                | `store`           | `produits.store`   |
| `GET`           | `/produits/{produit}`      | `show`            | `produits.show`    |
| `GET`           | `/produits/{produit}/edit` | `edit`            | `produits.edit`    |
| `PUT` / `PATCH` | `/produits/{produit}`      | `update`          | `produits.update`  |
| `DELETE`        | `/produits/{produit}`      | `destroy`         | `produits.destroy` |

The controller must provide these methods. The next chapter shows their signatures.

> [!tip] Resource routes are named for you
> That is why a view can write `route('produits.index')`, `route('produits.create')`, or `route('produits.show', $produit->id)` without any extra naming work.

> [!warning] Order is handled for you here
> `Route::resource` places `/produits/create` before `/produits/{produit}`, so the create form is not swallowed by the `show` route. When you write routes manually, you must respect that order yourself.

## 9. HTTP methods in one table

| Method | Typical use | Example route |
|---|---|---|
| `GET` | Read a page or a resource | `Route::get('/contacts', …)` |
| `POST` | Submit a form that creates data | `Route::post('/contacts', …)` |
| `PUT` / `PATCH` | Update existing data | `Route::put('/contacts/{id}', …)` |
| `DELETE` | Delete data | `Route::delete('/contacts/{id}', …)` |

> [!note] HTML forms and HTTP verbs
> A browser form can only send `GET` or `POST`. Laravel's Blade directives add a hidden `_method` field so a `POST` form can be treated as `PUT`, `PATCH`, or `DELETE`. You will see this in [[07 - End-to-End Feature]].

## 10. Common routing mistakes

| Mistake | Result | Fix |
|---|---|---|
| Dynamic route before a fixed route | Fixed URL is captured by the dynamic route | Put fixed routes first |
| Optional parameter before a required one | Route pattern does not behave as expected | Put optional parameters last |
| Route points to a missing method | Laravel cannot dispatch the request | Check the controller method name |
| Wrong namespace in `use` | Class not found | Match the namespace to the folder |
| Using `view('home.blade.php')` | Wrong view name | Use `view('home')`; Laravel adds the extension |
| Hard-coding `/contacts` in links | Links break when the URL changes | Use named routes and `route()` |

## 11. What comes next

Routing tells Laravel *which code to run*. The next note explains the class that actually runs: the controller, plus the views it returns and the middleware that can interrupt it.

→ Continue with [[03 - Controllers and Middleware]].
