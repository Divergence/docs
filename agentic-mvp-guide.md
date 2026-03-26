### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Agentic MVP Guide

This guide is written for an agent working in a Divergence application repository.

The goal is to get a minimal MVP running with:

- a web entrypoint
- a CLI entrypoint
- either SQLite or MySQL
- a minimal example of writing JSON into the database

## Rules For The Agent

When setting up a CLI program in this codebase:

- put executable PHP scripts in `./bin/`
- use a real shebang: `#!/usr/bin/env php`
- make the script executable with `chmod +x`
- run the script directly as `./bin/name`
- do not tell the user to invoke it as `php ./bin/name`

When choosing a database:

- use SQLite for the fastest MVP and lowest setup friction
- use MySQL when the app is expected to match a production MySQL deployment

When building the minimum viable app:

- subclass `Divergence\App`
- keep `bootstrap/app.php` thin
- route web requests through your own root controller
- keep CLI code out of controllers
- let CLI scripts bootstrap the app and then run one task

## Minimum Structure

For a practical MVP, create or verify these paths:

```text
bootstrap/
config/
public/
src/
src/Controllers/
src/Models/
bin/
views/
```

## Web Setup

Your web entrypoint should stay standard:

```php
<?php
require(__DIR__.'/../bootstrap/autoload.php');
require(__DIR__.'/../bootstrap/app.php');
require(__DIR__.'/../bootstrap/router.php');
```

Your `App` subclass should own web dispatch:

```php
<?php

namespace App;

use App\Controllers\Main;
use Divergence\Responders\Emitter;
use GuzzleHttp\Psr7\ServerRequest;

class App extends \Divergence\App
{
    public function handleRequest()
    {
        $main = new Main();
        $response = $main->handle(ServerRequest::fromGlobals());
        (new Emitter($response))->emit();
    }
}
```

## SQLite MVP Setup

For the fastest path, set SQLite in `config/db.php`:

```php
'sqlite' => [
    'path' => __DIR__ . '/../var/sqlite/app.sqlite',
    'foreign_keys' => true,
    'busy_timeout' => 5000,
],
```

Then set the active connection early in your app bootstrap or CLI script:

```php
\Divergence\IO\Database\Connections::setConnection('sqlite');
```

Use SQLite when:

- you want zero external DB setup
- you need a local prototype quickly
- one-file persistence is enough

## MySQL MVP Setup

For MySQL, configure a label in `config/db.php`:

```php
'mysql' => [
    'host' => '127.0.0.1',
    'database' => 'app',
    'username' => 'app',
    'password' => 'secret',
],
```

Then activate it:

```php
\Divergence\IO\Database\Connections::setConnection('mysql');
```

Use MySQL when:

- production already uses MySQL
- you need to match MySQL behavior from day one
- deployment assumptions already include a server DB

## CLI Setup

Put your script in `./bin/`.

Use a real executable file:

```php
#!/usr/bin/env php
<?php

require __DIR__.'/../bootstrap/autoload.php';
require __DIR__.'/../bootstrap/app.php';

use Divergence\IO\Database\Connections;

Connections::setConnection('sqlite');

$payload = json_decode(stream_get_contents(STDIN), true);

if (!is_array($payload)) {
    fwrite(STDERR, "Invalid JSON\n");
    exit(1);
}

fwrite(STDOUT, "Loaded JSON\n");
```

Then make it executable:

```bash
chmod +x ./bin/import-json
```

Run it like this:

```bash
echo '{"Title":"Hello","Body":"World"}' | ./bin/import-json
```

## Minimum JSON-To-DB Example

This is the minimum useful example with almost no extra context.

Model:

```php
<?php

namespace App\Models;

use Divergence\Models\Model;
use Divergence\Models\Mapping\Column;

class Note extends Model
{
    public static $tableName = 'notes';

    #[Column(type: 'string', notnull: true)]
    protected $Title;

    #[Column(type: 'clob', notnull: true)]
    protected $Body;
}
```

CLI script:

```php
#!/usr/bin/env php
<?php

require __DIR__.'/../bootstrap/autoload.php';
require __DIR__.'/../bootstrap/app.php';

use App\Models\Note;
use Divergence\IO\Database\Connections;

Connections::setConnection('sqlite');

$data = json_decode(stream_get_contents(STDIN), true);

if (!is_array($data)) {
    fwrite(STDERR, "Invalid JSON\n");
    exit(1);
}

$note = Note::create([
    'Title' => $data['Title'] ?? 'Untitled',
    'Body' => $data['Body'] ?? '',
], true);

fwrite(STDOUT, json_encode([
    'success' => true,
    'id' => $note->ID,
], JSON_PRETTY_PRINT) . "\n");
```

Install executable bit:

```bash
chmod +x ./bin/import-note
```

Run it:

```bash
echo '{"Title":"First note","Body":"Saved from JSON"}' | ./bin/import-note
```

## Agent Workflow

If asked to set up a minimal MVP, the agent should do this in order:

1. Create `src/App.php` if the app does not already override `Divergence\App`.
2. Create a root controller under `src/Controllers/Main.php`.
3. Choose `sqlite` unless the user explicitly wants MySQL parity.
4. Add or verify the DB label in `config/db.php`.
5. Create models with the smallest possible schema.
6. Add executable scripts to `./bin/` with `#!/usr/bin/env php`.
7. Mark those scripts executable with `chmod +x`.
8. Prefer piping JSON over inventing a larger CLI argument parser for the MVP.
9. Only add web forms, admin screens, or richer command frameworks after the basic write path works.

## Minimum Standard

For an MVP, the setup is good enough when:

- `./bin/import-note` runs directly
- JSON from stdin is accepted
- one record is written successfully
- the same app still boots on the web side
- the DB backend can be switched by changing the connection label
