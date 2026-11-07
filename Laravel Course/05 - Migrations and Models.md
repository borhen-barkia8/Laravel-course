---
title: "05 - Migrations and Models"
tags:
  - laravel
  - course
  - database
  - migrations
  - eloquent
aliases:
  - Laravel Migrations
  - Laravel Models
---

# Migrations and Models: Describing and Representing Data

> [!abstract] In one paragraph
> A migration is a PHP file that describes a database table or a change to one. It has an `up()` method that applies the change and a `down()` method that reverses it. A model is a PHP class that represents one table and lets you work with its rows as objects. Together, migrations define the shape of the data and Eloquent models give you a code-level way to use it.

## 1. The three ways Laravel talks to a database

Laravel supports several levels of database access.

| Level | What you write | When to use it |
|---|---|---|
| **Raw SQL** | SQL strings executed through `DB` | Rare; complex queries or database-specific features |
| **Query Builder** | Fluent PHP methods such as `DB::table('contacts')->where(…)` | Building queries without a model |
| **Eloquent** | Model methods such as `Contact::where(…)` | Most application code |

The same query can be written three ways:

```php
// 1. Raw SQL
$contacts = DB::select(
    'SELECT * FROM contacts WHERE email LIKE ? ORDER BY name',
    ['%@gmail.com']
);

// 2. Query Builder
$contacts = DB::table('contacts')
    ->where('email', 'like', '%@gmail.com')
    ->orderBy('name')
    ->get();

// 3. Eloquent
$contacts = Contact::where('email', 'like', '%@gmail.com')
    ->orderBy('name')
    ->get();
```

> [!tip] Start with Eloquent
> Eloquent is the most readable and the most "Laravel-like". Query Builder is still useful when you do not have a model, and raw SQL is a last resort for unusual queries. [[06 - Eloquent CRUD and Queries]] covers the first two in depth.

## 2. What a migration is

A migration is **database schema written as PHP code**. Instead of creating tables by hand in a database tool, you describe them in a file that belongs to the project.

A migration can:

- Create a table.
- Add, rename, or remove a column.
- Change a column's type or modifiers.
- Drop a table.
- Create indexes and foreign keys (not covered here).

Why this is useful:

| Benefit | Explanation |
|---|---|
| **Versioned** | The schema evolves with the code and can be reviewed like code |
| **Reproducible** | A new environment can rebuild the same schema |
| **Reversible** | `down()` describes how to undo the change |
| **Shared** | Every developer gets the same table definitions |

> [!info] Migrations are ordered by timestamp
> Migration files live in `database/migrations` and start with a timestamp such as `2026_03_01_120000_create_contacts_table.php`. Laravel uses that timestamp to decide the order in which migrations run.

A new Laravel project already contains migrations for:

- the `users` table (which also creates `password_reset_tokens` and `sessions`),
- the `cache` table,
- the `jobs` table used by queues.

## 3. The two methods of a migration

Every migration has two important methods:

| Method | When it runs | Job |
|---|---|---|
| `up()` | When the migration is applied | Create or modify the schema |
| `down()` | When the migration is rolled back | Reverse what `up()` did |

```php
public function up()
{
    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('email')->unique();
        $table->timestamp('email_verified_at')->nullable();
        $table->string('password');
        $table->rememberToken();
        $table->timestamps();
    });
}

public function down()
{
    Schema::dropIfExists('users');
}
```

| Line                            | Meaning                                       |
| ------------------------------- | --------------------------------------------- |
| `Schema::create('users', …)`    | Create a table named `users`                  |
| `Blueprint $table`              | The object used to declare columns            |
| `$table->id()`                  | Auto-incrementing primary key named `id`      |
| `$table->string('name')`        | `VARCHAR` column                              |
| `->unique()`                    | Add a uniqueness constraint                   |
| `->nullable()`                  | Allow `NULL` values                           |
| `$table->timestamps()`          | Add `created_at` and `updated_at`             |
| `Schema::dropIfExists('users')` | Drop the table if it exists, used by `down()` |

> [!important] down() must undo up()
> If `up()` creates a table, `down()` drops it. If `up()` adds a column, `down()` removes it. A migration that cannot be reversed is harder to use in development.

## 4. Creating a table: the migration file

A modern Laravel migration is an anonymous class:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('employees', function (Blueprint $table) {
            $table->id();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('employees');
    }
};
```

The file name follows the convention `create_xxx_table`, and Laravel infers the table name `employees`.

> [!note] The suffix matters
> Naming a migration `create_employees_table` is not cosmetic: the convention tells Laravel (and the reader) that this file creates the `employees` table. For a change to an existing table, names such as `add_details_to_employees_table` or `change_emp_name_in_employees_table` describe the intent.

Once the fields are added to `up()`, the migration describes the table completely:

```php
Schema::create('employees', function (Blueprint $table) {
    $table->id();
    $table->string('emp_name', 100);
    $table->string('email')->unique();
    $table->decimal('salary', 10, 2)->nullable();
    $table->date('hire_date');
    $table->timestamps();
});
```

## 5. Column types

### 5.1 Primary key

```php
$table->id();   // shortcut for bigIncrements('id')
```

This creates an unsigned big-integer `id` that auto-increments.

### 5.2 Numbers

```php
$table->integer('age');
$table->float('height');
$table->double('weight');
$table->decimal('prix', 8, 3);    // total 8 digits, 3 after the decimal point
$table->tinyInteger('rating');
$table->bigInteger('views');
```

### 5.3 Text and booleans

```php
$table->char('initial', 1);
$table->string('name', 100);
$table->text('description');
$table->boolean('is_active');
$table->set('flavors', ['orange', 'vanille']);
```

### 5.4 Dates and times

```php
$table->date('birth_date');
$table->dateTime('published_at');
$table->time('meeting_time');
$table->timestamps();              // created_at + updated_at
```

### 5.5 Special types

```php
$table->json('settings');
$table->binary('photo');
```

> [!tip] Choose the narrowest type that fits
> `string('name', 100)` documents an intentional limit. `text()` is for long free-form content. `decimal()` is for money; do not store money in a `float`, because binary floating-point arithmetic introduces rounding errors.

## 6. Changing an existing table

When the table already exists, use `Schema::table()` instead of `Schema::create()`.

### 6.1 Changing a column

```php
// up(): make emp_name nullable and limit it to 100 characters
Schema::table('employees', function (Blueprint $table) {
    $table->string('emp_name', 100)->nullable()->change();
});

// down(): restore the previous definition
Schema::table('employees', function (Blueprint $table) {
    $table->string('emp_name')->change();   // VARCHAR(255) NOT NULL
});
```

> [!warning] change() replaces the whole definition
> Any modifier you do not repeat is lost. If the column must stay nullable, write `->nullable()->change()`. If it has a default, repeat `->default(…)`. In `down()`, omitting `->nullable()` is precisely what makes the column `NOT NULL` again.

### 6.2 Adding columns

```php
// up() 
Schema::table('employees', function (Blueprint $table) { $table->string('phone'); // added at the end 
$table->string('address')->after('emp_name'); // placed right after emp_name 
$table->string('city')->after('address'); // placed right after address 
}); 
// down() 
Schema::table('employees', function (Blueprint $table) { $table->dropColumn(['phone', 'address', 'city']); });
```

`after()` only controls column position; it does not affect behaviour.

### 6.3 Renaming and dropping columns

```php
Schema::table('employees', function (Blueprint $table) {
    $table->renameColumn('phone', 'phone_number');
});

// Drop one column
$table->dropColumn('phone');

// Drop several columns at once
$table->dropColumn(['phone', 'address', 'city']);
```

> [!important] Every change has an inverse
> In `up()`, add or rename. In `down()`, drop or rename back. A migration is a pair of directions, not a one-way script.

## 7. What an Eloquent model is

Eloquent is Laravel's ORM — Object-Relational Mapping. It is the bridge between PHP objects and relational tables.

| Database concept | Eloquent concept |
|---|---|
| Table | Model class |
| Row | Model instance |
| Column | Attribute/property |
| SQL query | Model or Query Builder call |

A model class lives in `app/Models` and extends `Illuminate\Database\Eloquent\Model`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Contact extends Model
{
    //
}
```

That empty class is already useful. Eloquent assumes:

- The model `Contact` uses the table `contacts` (singular class → plural table).
- The table has an auto-incrementing integer primary key named `id`.
- The table has `created_at` and `updated_at` timestamp columns.

> [!tip] An empty model is not useless
> Because of conventions, `class Contact extends Model {}` can already query the `contacts` table. You only add properties when your schema or naming differs from the convention.

## 8. Model conventions

### 8.1 Table name

By default, Eloquent pluralises the class name:

```php
class Contact extends Model
{
    // Uses the "contacts" table automatically
}
```

If the table has another name, declare it explicitly:

```php
class Contact extends Model
{
    protected $table = 'my_contacts';
}
```

### 8.2 Primary key

By default, Eloquent expects a column named `id`, of type integer, auto-incrementing.

For a different primary key, define it both in the migration and in the model.

**Migration:**

```php
$table->string('matricule', 10)->primary();
```

**Model:**

```php
class Etudiant extends Model
{
    protected $primaryKey = 'matricule';
    protected $keyType = 'string';   // 'int' by default
    public $incrementing = false;    // true by default; note: public, not protected
}
```

> [!warning] The model must be told
> Creating a custom primary key in the migration is not enough. Eloquent still assumes `id` until you set `$primaryKey`. It also assumes an integer key unless you set `$keyType` and disable incrementing.

### 8.3 UUID primary keys

A UUID is a 36-character unique identifier. It is useful when IDs must be generated without asking the database, or when you do not want sequential IDs.

**Migration:**

```php
Schema::create('contacts', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('phone')->nullable();
    $table->text('message');
    $table->timestamps();
});
```

> [!note] uuid() needs two details
> `$table->uuid()` creates a column named `uuid` by default. To call it `id`, write `uuid('id')`. The method does not make it a primary key automatically, so add `->primary()`.

**Model:**

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Concerns\HasUuids;

class Contact extends Model
{
    use HasUuids;

    protected $fillable = [
        'name', 'email', 'phone', 'message',
    ];
}
```

The `HasUuids` trait generates UUIDs automatically when a new contact is created. It also declares a string key and disables incrementing, so you do not need to set `$keyType` and `$incrementing` manually.

## 9. Timestamps

Eloquent manages `created_at` and `updated_at` automatically when the table has them (created by `$table->timestamps()`).

- `created_at` is set when the record is created.
- `updated_at` is changed every time the record is saved after a modification.

### 9.1 Disabling timestamps

If the table does not have timestamp columns, disable them:

```php
class Contact extends Model
{
    public $timestamps = false;
}
```

Set it back to `true` to re-enable them (this is the default).

### 9.2 Renaming timestamp columns

If the columns have different names, declare the constants:

```php
class Contact extends Model
{
    const CREATED_AT = 'creation_date';
    const UPDATED_AT = 'last_update';
}
```

Laravel will still manage them automatically, but under the new names.

### 9.3 Using timestamps in code

```php
$contact = Contact::find(1);

echo $contact->created_at;   // date of creation
echo $contact->updated_at;   // date of last update
```

These values are Carbon instances, Laravel's date/time library. They support methods such as `addDay()`, `diffForHumans()`, and `format(…)`.

## 10. Default attribute values

A newly instantiated model normally has no attribute values. You can give some attributes a default value with `$attributes`:

```php
class Contact extends Model
{
    protected $attributes = [
        'status' => 'pending',
    ];
}
```

```php
$contact = new Contact();
echo $contact->status;      // "pending"

$contact->status = 'approved';
echo $contact->status;      // "approved"
```

> [!warning] Defaults are stored in raw, database-ready form
> If a column stores an integer, the default in `$attributes` must be an integer, not a formatted string. For defaults that depend on complex logic, use a mutator or the model constructor instead of `$attributes`.

## 11. Choosing a database connection

By default, every Eloquent model uses the application's main database connection. If a project defines a secondary connection named, for example, `mysql2` in `config/database.php`, a model can choose it:

```php
class Contact extends Model
{
    protected $connection = 'mysql2';
}
```

Most applications never need this. It matters when one model legitimately belongs to a different database.

## 12. Mass assignment: a first look

When creating a model from an array of request data, Eloquent needs to know which attributes may be filled in bulk. This is controlled by `$fillable` or `$guarded`:

```php
class Contact extends Model
{
    protected $fillable = ['name', 'email', 'phone', 'message'];
}
```

> [!danger] Never leave mass assignment wide open
> `protected $guarded = [];` allows every column to be filled from request data. That lets a malicious user set columns you never intended, such as `is_admin`. Prefer an explicit `$fillable` list. Chapter [[06 - Eloquent CRUD and Queries]] explains the full mechanism.

## 13. Migrations and models are two halves of one idea

![[Migrations and Models.excalidraw|1000]]

- The **migration** defines the table: columns, types, constraints.
- The **model** gives that table a PHP identity: `Contact`, with attributes and methods.

If the migration creates a `contacts` table, the model should be `Contact`. If the model is named differently, or the table is named differently, you must state the mapping explicitly with `$table`.

## 14. What comes next

You now know how data is *described*. The next note shows how it is *used*: reading rows, filtering them, creating records, updating them, and deleting them with Eloquent.

→ Continue with [[06 - Eloquent CRUD and Queries]].
