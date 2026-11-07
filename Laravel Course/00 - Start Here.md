---
title: Laravel Course - Start Here
tags:
  - laravel
  - course
  - moc
aliases:
  - Laravel Course
  - Laravel for Beginners
---

# Laravel: A Code-First Course

> [!info] What this course is
> A complete reading path through the Laravel material in `Laravel/ressources`, rewritten for someone who has **never used a framework before**. It follows the four source PDFs but removes installation, local servers, database configuration, and terminal commands.

> [!tip] Who this is for
> You know a little PHP — variables, functions, arrays, classes — but you have never built a Laravel application. If you can read `class Contact extends Model` and understand that it defines a class, you are ready.

## The one idea to remember

Every Laravel request follows the same path:

![[Request Flow.excalidraw|100%]]

Solid arrows are the request path; dashed arrows are data coming back.

A URL is matched to a **route**. The route calls a **controller method**. The controller asks a **model** for data. The model talks to the **database**. The database returns rows to the model, the model returns them to the controller, and the controller passes that data to a **view**. The view becomes the HTML sent back to the browser.

That is the whole framework in one sentence. Everything else is detail around this loop.

## Course map

| Note | What you learn | Read it when |
|---|---|---|
| [[01 - The Big Picture]] | PHP, frameworks, MVC, Laravel's folders, the request lifecycle | First |
| [[02 - Routing]] | How URLs become code: routes, names, parameters, resource routes | After the big picture |
| [[03 - Controllers and Middleware]] | Controller classes, dependency injection, responses, middleware | After routing |
| [[04 - Blade Templates]] | Views, parameterized views, `{{ }}`, conditions, loops, templates, sections, includes (pages 1–25) | After controllers |
| [[05 - Migrations and Models]] | How tables are described in PHP, and how Eloquent models map to tables | After Blade |
| [[06 - Eloquent CRUD and Queries]] | Reading, creating, updating, deleting data; Query Builder; subqueries; collections | After models |
| [[07 - End-to-End Feature]] | One complete "contacts" feature, file by file | After chapters 1–6 |
| [[08 - Code Reference]] | Cheat sheet, conventions, common mistakes | While reading and revising |

## What is intentionally left out

> [!warning] No setup in this course
> There are **no** installation steps, no web-server configuration, no database server setup, and no terminal commands. When a note says "the migration file contains…", imagine the file already exists in the project.

The course is about **the code Laravel expects you to write**: the classes, methods, routes, templates, and model definitions.

## Vocabulary you will meet

| Term                | Meaning in this course                                        |
| ------------------- | ------------------------------------------------------------- |
| **Route**           | A rule that maps an HTTP method + URL to code                 |
| **Controller**      | A class whose methods handle requests                         |
| **Model**           | A PHP class that represents a database table                  |
| **Migration**       | A PHP file that describes a database table or a change to one |
| **Eloquent**        | Laravel's ORM: the layer that turns model calls into SQL      |
| **Blade**           | Laravel's template engine, used in `.blade.php` files         |
| **Layout**          | A base Blade template with `@yield` areas that child views fill |
| **Partial**         | A small Blade fragment inserted with `@include`                |
| **Component**       | A reusable Blade UI element used as `<x-…>`                    |
| **Middleware**      | A filter that runs before or after a controller               |
| **Mass assignment** | Filling a model from an array of request data                 |
