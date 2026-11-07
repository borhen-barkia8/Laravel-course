---
title: "06 - Eloquent CRUD and Queries"
tags:
  - laravel
  - course
  - eloquent
  - database
aliases:
  - Eloquent CRUD
  - Laravel Database Queries
---

# Eloquent in Practice: CRUD and Queries

> [!abstract] In one paragraph
> Each Eloquent model is both a representation of a table and a query builder for it. You can read all rows, find one row, filter with `where()`, order and limit results, create records with `save()` or `create()`, update them with `save()` or `update()`, and delete them with `delete()` or `destroy()`. For everyday data access, you rarely need to write SQL by hand.

## 1. The model as a query builder

Eloquent is Laravel's ORM. It maps rows to objects and lets you call methods instead of writing SQL.

```php
$contacts = Contact::all();
```

This returns all rows from the `contacts` table as `Contact` objects. Each object exposes its columns as properties:

```php
foreach ($contacts as $contact) {
    echo $contact->name;
    echo $contact->email;
}
```

> [!note] Eloquent returns collections
> `Contact::all()` and `Contact::where(…)->get()` return an Eloquent **Collection**, not a plain array. A collection supports `foreach`, `count()`, `isEmpty()`, `first()`, and many other helpers. Blade's `@foreach` works with it directly.

## 2. Reading data

### 2.1 All records

```php
foreach (Contact::all() as $contact) {
    echo $contact->name;
    echo $contact->email;
}
```

### 2.2 One record by primary key

```php
$contact = Contact::find(1);
```

`find()` returns `null` if no row has that ID.

In a controller, a common variant is `findOrFail()`:

```php
$contact = Contact::findOrFail(1);
```

`findOrFail()` sends a 404 response automatically when the record does not exist, which is usually the desired behaviour for a `show` page.

### 2.3 Pagination

Loading thousands of rows into memory is a bad idea. Pagination limits the number of records per page:

```php
$contacts = Contact::paginate(10);   // 10 contacts per page
```

In a Blade view:

```PHP
@foreach ($contacts as $contact)
    <p>{{ $contact->name }}</p>
@endforeach

{{ $contacts->links() }}
```

> [!tip] Pagination belongs in the controller
> The controller decides how many records a page contains. The view only renders the page links. Do not paginate inside a Blade template.

## 3. Filtering with Query Builder methods

Eloquent exposes Query Builder methods directly. Add conditions, then call `get()` to execute the query.

```php
$activeContacts = Contact::where('status', 'active')->get();

foreach ($activeContacts as $contact) {
    echo $contact->name;
    echo $contact->email;
}
```

| Method | Purpose |
|---|---|
| `where('column', 'value')` | Equality condition |
| `where('column', 'operator', 'value')` | Comparison, e.g. `>=` |
| `orderBy('column', 'asc'\|'desc')` | Sort results |
| `take(n)` | Limit the number of rows |
| `select('a', 'b')` | Choose specific columns |
| `get()` | Execute and return a collection |

### 3.1 Combining conditions

```php
$contacts = Contact::where('status', 'active')
    ->where('created_at', '>=', now()->subMonth())
    ->get();
```

`now()` returns the current date/time as a Carbon instance; `subMonth()` returns one month earlier. This returns contacts that are active **and** created in the last month.

### 3.2 Ordering and limiting

```php
$recentContacts = Contact::orderBy('created_at', 'desc')
    ->take(5)
    ->get();
```

This returns the five most recently created contacts.

### 3.3 Selecting specific columns

```php
$contacts = Contact::where('status', 'active')
    ->select('name', 'email')
    ->get();
```

Only `name` and `email` are loaded. This is useful for lists where the other columns are not needed.

> [!warning] Select what you use
> If you select only some columns, do not access other attributes on the resulting models. They were not loaded from the database.

## 4. Subqueries

Eloquent supports subqueries: a query nested inside another query, executed by the database engine as part of the same statement.

Consider an `articles` table with `id`, `reference`, `designation`, `prix`, `stock`, and `created_at`. Suppose you want to display each article with the average price of the catalogue, to see where it stands.

### 4.1 A calculated column with addSelect()

```php
use App\Models\Article;

$articles = Article::select('designation', 'prix')
    ->addSelect([
        'prix_moyen' => Article::selectRaw('AVG(prix)'),
    ])
    ->get();

foreach ($articles as $article) {
    echo $article->designation . ' : '
        . $article->prix . ' (moyenne : '
        . $article->prix_moyen . ")\n";
}
```

| Method | Meaning |
|---|---|
| `select('designation', 'prix')` | Selects the normal columns |
| `addSelect([…])` | Adds a computed column that does not exist in the table |
| `selectRaw('AVG(prix)')` | Expresses the calculation in raw SQL, because the builder cannot translate `AVG` itself |
| `$article->prix_moyen` | The computed value is available as an attribute |

The database computes `prix_moyen` in the same query; you do not loop over articles and run one query per row.

### 4.2 Comparing to an aggregate

A subquery can also provide the value used by a `where` condition. To get articles more expensive than the catalogue average:

```php
$articles = Article::where('prix', '>', function ($query) {
    $query->selectRaw('AVG(prix)')->from('articles');
})
->orderByDesc('prix')
->get();
```

| Piece | Meaning |
|---|---|
| `where('prix', '>', function ($query) { … })` | Compares `prix` to the value returned by the subquery |
| `$query->selectRaw('AVG(prix)')` | The subquery itself |
| `->from('articles')` | The subquery is independent of the outer query and must name its own table |
| `orderByDesc('prix')` | Sort from most expensive to cheapest |

Without a subquery, you could write:

```php
$moyenne = Article::avg('prix');

$articles = Article::where('prix', '>', $moyenne)
    ->orderByDesc('prix')
    ->get();
```

The result is the same, but it runs **two** queries instead of one.

> [!note] Subqueries and ordering
> A subquery can also be used inside `orderByDesc()`. That pattern becomes especially useful between related tables, where the subquery computes a value from the related table.

## 5. Creating data

There are two common ways to insert a row.

### 5.1 New model + save()

```php
$contact = new Contact();
$contact->name = 'Sami Sellam';
$contact->email = 'sellam@example.com';
$contact->phone = '123456789';
$contact->message = 'Bonjour, je veux en savoir plus.';
$contact->save();
```

`save()` writes the object to the database. After that, `$contact->id` contains the generated primary key.

### 5.2 create() and mass assignment

`create()` takes an array and fills the model in one call:

```php
Contact::create([
    'name' => 'Sami Sellam',
    'email' => 'sellam@example.com',
    'phone' => '123456789',
    'message' => 'Merci pour l\'information.',
]);
```

Before this works, the model must state which attributes may be assigned in bulk:

```php
class Contact extends Model
{
    protected $fillable = ['name', 'email', 'phone', 'message'];
}
```

### 5.3 $fillable vs $guarded

| Property | Meaning | Example |
|---|---|---|
| `$fillable` | Only these attributes may be mass-assigned | `protected $fillable = ['name', 'email'];` |
| `$guarded` | These attributes may **not** be mass-assigned; all others may | `protected $guarded = ['is_admin'];` |

> [!danger] Do not use both
> `$fillable` and `$guarded` are two ways to express the same protection. Define only one. If you define `$fillable`, only those columns are assignable. If you define `$guarded`, every column not listed is assignable.

> [!warning] Never use $guarded = [] casually
> An empty `$guarded` array protects nothing: every column can be filled from request data. If a form posts an unexpected `is_admin` field, it will be written. Prefer an explicit `$fillable` list.

> [!tip] Why mass assignment protection exists
> Without it, `Contact::create($request->all())` would let the client decide which columns to write. Mass assignment protection is the difference between "the user may set their name and email" and "the user may set anything in the table".

## 6. Updating data

### 6.1 Load, modify, save

```php
$contact = Contact::find(1);
$contact->phone = '111222333';
$contact->save();
```

This updates only the changed attributes, and Laravel updates `updated_at` automatically.

### 6.2 update() on an instance

```php
$contact = Contact::find(1);
$contact->update([
    'phone' => '111222333',
]);
```

This is equivalent to setting the attributes and calling `save()`.

### 6.3 Update without loading the model

```php
Contact::where('id', 1)->update([
    'phone' => '111222333',
]);
```

> [!note] Which form should you use?
> Use an instance when you need to read the record anyway (for example, to display it first). Use a direct `where()->update()` when you only want to change values and do not need the object. Direct updates do **not** fire model events, but that is beyond this course.

## 7. Deleting data

### 7.1 Delete an instance

```php
$contact = Contact::find(1);
$contact->delete();
```

### 7.2 Delete by primary key without loading

```php
Contact::destroy(1);
```

> [!warning] Deletion is permanent by default
> These calls remove the row from the table. Laravel also supports "soft deletes" (marking a row as deleted while keeping it), but that requires an additional column and trait and is outside this course.

## 8. CRUD in one table

| Operation | Method | Example |
|---|---|---|
| Create | `save()` | `$contact->save();` |
| Create | `create()` | `Contact::create([…]);` |
| Read all | `all()` | `Contact::all();` |
| Read one | `find()` / `findOrFail()` | `Contact::find(1);` |
| Read filtered | `where()->get()` | `Contact::where('status', 'active')->get();` |
| Read paginated | `paginate()` | `Contact::paginate(10);` |
| Update | `save()` | `$contact->phone = '…'; $contact->save();` |
| Update | `update()` | `$contact->update([…]);` |
| Delete | `delete()` | `$contact->delete();` |
| Delete | `destroy()` | `Contact::destroy(1);` |

## 9. Query Builder vs Eloquent

Use Query Builder directly when you do not need model instances:

```php
$rows = DB::table('contacts')
    ->where('status', 'active')
    ->orderBy('name')
    ->get();
```

The rows are plain objects, not `Contact` instances. You lose model features such as attribute casting, relationships, and model events, but the query is lighter.

Use Eloquent when the result is a domain object you will use as such:

```php
$contacts = Contact::where('status', 'active')
    ->orderBy('name')
    ->get();
```

> [!tip] Default choice for this course
> If a model exists for the table, start with Eloquent. Reach for `DB::table()` when the query is genuinely table-oriented and does not benefit from model behaviour.

## 10. A note on security

Eloquent and Query Builder use parameter binding. Values passed to `where()` are not concatenated into the SQL string:

```php
Contact::where('email', $email)->first();
```

This is the safe way to include user input. Avoid building SQL strings by hand with concatenation.

> [!danger] Never interpolate request data into raw SQL
> `DB::select("SELECT * FROM contacts WHERE email = '$email'")` is an SQL injection risk. Prefer Eloquent, Query Builder, or bound parameters.

## 11. What comes next

Every layer has now been introduced separately. The next note assembles them into one complete feature, with the migration, model, controller, routes, and views for a small contact book.

→ Continue with [[07 - End-to-End Feature]].
