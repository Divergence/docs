### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Security

This section documents the security primitives that exist today across the framework.

The important split is:

- the framework ships a reusable session model and controller access hooks
- your application defines the actual user model and login policy
- authentication policy is still application-defined

## User Model

The framework does not currently ship a built-in `User` model class.

A minimal application user model commonly looks like this:

```php
<?php
namespace project\Models;

class User extends \Divergence\Models\Model
{
    use \Divergence\Models\Relations;

    public static $tableName = 'users';
    private string $Email;
    private string $DisplayName;
    private string $PasswordHash;
}
```

This is the intended pattern:

- extend `Divergence\Models\Model`
- add your identity fields such as `Email` and `DisplayName`
- store hashed credentials in a field like `PasswordHash`

The framework does not impose a specific user schema beyond what your application needs.

## Session Model

The framework does ship a reusable session implementation: `Divergence\Models\Auth\Session`.

It provides:

- cookie-backed session lookup
- persistent session records in the database
- session creation on demand
- timeout handling
- session termination
- handle generation
- `CreatorID` support through the base model fields

Important built-in session fields include:

- `Handle`
- `LastRequest`
- `LastIP`
- `ContextClass`
- `ContextID`

The session model uses a `binary` field for `LastIP`, and its getter/setter logic already normalizes that field correctly for SQLite and PostgreSQL.

A thin subclass is usually enough when the framework session behavior already matches your application:

```php
<?php
namespace project\Models;

class Session extends \Divergence\Models\Auth\Session
{
}
```

## Authentication

Authentication is application-defined, not framework-enforced.

A common app bootstrap flow is:

1. bootstrap a session with `Session::getFromRequest()`
2. inspect posted login credentials
3. look up the user by email
4. verify the password hash
5. write the user ID into `Session->CreatorID`
6. save the session

The relevant shape is:

```php
if ($User = User::getByField('Email', $username)) {
    if (password_verify($password, $User->PasswordHash)) {
        $this->Session->CreatorID = $User->ID;
        $this->Session->save();
    }
}
```

This is intentionally simple:

- user lookup is just a normal model query
- password verification uses PHP's built-in `password_verify`
- session identity is established by storing the user ID in `CreatorID`

## Logged-In State

A simple helper often looks like:

```php
public function is_loggedin()
{
    if ($this->Session) {
        if ($this->Session->CreatorID) {
            return true;
        }
    }
    return false;
}
```

That pattern works because `Session` inherits the standard model fields, including `CreatorID`.

If your app needs richer identity checks, this is the place to extend them.

## Binding Permissions

The normal place to enforce security in HTTP flows is the controller layer.

For `RecordsRequestHandler` and `MediaRequestHandler`, the important hooks are:

- `checkBrowseAccess()`
- `checkReadAccess()`
- `checkWriteAccess()`
- `checkUploadAccess()`
- `checkAPIAccess()`

Reusable permission traits are the normal pattern.

### Logged-In Permissions

```php
<?php
namespace project\Controllers\Records\Permissions;

use project\App as App;
use Divergence\Models\ActiveRecord;

trait LoggedIn
{
    public function is()
    {
        return App::$App->is_loggedin();
    }

    public function checkBrowseAccess($arguments)
    {
        return $this->is();
    }

    public function checkReadAccess(ActiveRecord $Record)
    {
        return $this->is();
    }

    public function checkWriteAccess(ActiveRecord $Record)
    {
        return $this->is();
    }

    public function checkUploadAccess()
    {
        return $this->is();
    }

    public function checkAPIAccess()
    {
        return $this->is();
    }
}
```

### Admin Write, Guest Read

Another common pattern is public browse/read access with restricted writes.

This is a good fit when:

- content should be publicly visible
- edits should be limited to staff or authenticated users

## Practical Minimum

If you want the minimum viable auth stack in a Divergence app:

1. Create a `User` model with `Email` and `PasswordHash`.
2. Subclass `Divergence\Models\Auth\Session` or use it directly.
3. Bootstrap the session in your `App::init()` or equivalent startup path.
4. Verify passwords with `password_verify`.
5. Store the logged-in user ID in `Session->CreatorID`.
6. Enforce access in controller permission hooks.

That is the current practical pattern shown by the framework code.
