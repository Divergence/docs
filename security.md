### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Security

The framework gives you a session model and access hooks. It does not decide who your users are or what they are allowed to do. You have to wire that up.

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
    public static $tableName = 'users';

    #[\Divergence\Models\Mapping\Column]
    private string $Email;

    #[\Divergence\Models\Mapping\Column]
    private string $DisplayName;

    #[\Divergence\Models\Mapping\Column(length: 255)]
    private string $PasswordHash;

    public function getData(): array
    {
        $data = parent::getData();
        unset($data['PasswordHash']);
        return $data;
    }
}
```

This is the intended pattern:

- extend `Divergence\Models\Model`
- add your identity fields such as `Email` and `DisplayName`
- store hashed credentials in a field like `PasswordHash`

The framework does not impose a specific user schema beyond what your application needs.

Set `PasswordHash` using `password_hash($password, PASSWORD_DEFAULT)` when creating or changing credentials. The `password` field type does not perform that step for you. Also define the validation and unique email index your application needs.

Private mapped properties are still readable through the model API. They are not a data-redaction policy. `getData()` normally includes mapped fields, and the JSON builder uses that output. The override above excludes the hash from that path; raw-record getters and your own responses still need care. Don't expose a generic writable User endpoint that lets clients assign `PasswordHash` or privileged fields.

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

`getFromRequest()` checks the configured cookie and can also accept a handle from `$_REQUEST`. It updates the last request time and IP, creates a session by default, and returns `false` instead when called with `false` and no valid session is found. It reads `$_SERVER['REMOTE_ADDR']`, so use it in an initialized HTTP request, not an arbitrary CLI script.

The defaults include cookie name `s`, a one-year session timeout, and `cookieSecure = false`. The built-in cookie call does not set HttpOnly or SameSite. Review or override cookie handling for your deployment; don't assume those protections are already enabled. Set a Secure cookie on HTTPS and keep session handles out of URLs.

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
5. replace the session handle after authentication and write the user ID into `Session->CreatorID`
6. save the session before sending output

The relevant shape is:

```php
if ($User = User::getByField('Email', $username)) {
    if (password_verify($password, $User->PasswordHash)) {
        $this->Session->Handle = Session::generateUniqueHandle();
        $this->Session->CreatorID = $User->ID;
        $this->Session->save();
    }
}
```

This is intentionally simple:

- user lookup is just a normal model query
- password verification uses PHP's built-in `password_verify`
- session identity is established by storing the user ID in `CreatorID`

This snippet belongs in your application's login flow, with your `User` and `Session` classes imported. It assumes `$this->Session` has already been initialized. Invalid credentials, disabled accounts, rate limiting, CSRF checks, and login responses still belong to that flow. Call `terminate()` for logout; it deletes the session record and attempts to clear the cookie.

## Logged-In State

A simple helper often looks like:

```php
public function isLoggedIn(): bool
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

`isLoggedIn()` is your application method, not a method built into `Divergence\App`. A stored ID also doesn't prove that the user still exists or has permission for a particular record.

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

The default checks return `true`. Settings such as `accountLevelWrite = 'User'` don't enforce anything by themselves. Your hooks must implement that policy, including record ownership where appropriate.

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
        return App::$App->isLoggedIn();
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

    public function checkAPIAccess()
    {
        return $this->is();
    }
}
```

This trait covers the boolean checks used by the records controller. Media needs particular care: the upload endpoint calls `checkUploadAccess()` but does not test its returned boolean. Returning `false` alone does not stop an upload. Enforce denial in your application before dispatching the media handler, or use a hook that stops execution with an exception your application turns into an HTTP response. Audit the media actions you expose; don't assume the records controller's checks are applied to each one.

An error payload is not automatically an HTTP error status. The default unauthorized response contains `success: false` but uses the ordinary response status unless you override it. Your API can explicitly return a response with `withStatus(403)`.

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
5. Rotate the session handle and store the logged-in user ID in `Session->CreatorID`.
6. Enforce access in controller permission hooks.

Keep this wiring explicit. Session storage is not a complete authentication system, and a logged-in user is not automatically authorized to write every record.
