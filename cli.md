### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Command Line Tool
The command line tool helps you bootstrap a project, edit database configs, and test connections. It is a separate package from the framework.

## Installation
Make sure you have Composer installed.

`composer global require divergence/cli`

Make sure Composer's global bin directory is on your `PATH`. `composer global config bin-dir --absolute` shows where it is. Run `divergence --help` to see the commands available in your installed version.

## Initializing a Project
Make sure a `composer.json` file exists in the folder where you run this command.

`divergence init`

The command adds `divergence/divergence` to your dependencies, sets up a PSR-4 namespace under `src`, copies the bootstrap files into your project, and offers a database configuration wizard. Run it in the project directory and review the files it creates or changes.

The wizard is MySQL-oriented. For PostgreSQL or SQLite, use the [database configuration examples](database.md#connection-configurations). Then finish the [App and root controller setup](gettingstarted.md#take-over-control-from-the-framework); initialization does not write your application's routes.

You can watch a video of the process below.
[![asciicast](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs.png)](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs)

## Testing Your Database Config

To have a select menu come up with all the database configs run this command:

`divergence test database`

You can optionally provide the label, but if you do not you can select from the menu that comes up.

It returns a simple success or failure message after trying to connect.

```bash
divergence test database dev-mysql
```

The CLI connection tester uses its own MySQL connection code. Don't use it to judge PostgreSQL or SQLite configurations; test those through the framework's `Connections::getConnection()`.

## Change Your Database Config

To have a select menu come up with all the database configs run this command:

`divergence config database`

You can optionally provide the label, but if you do not you can select from the menu that comes up.

Once you select which one to edit a wizard will start using the old config as default. The config is rewritten to disk once the wizard is done.

## Tool Usage Reference
The basic command set:

```text
Divergence Command Line Tool

 divergence [command] [arguments]


        Available Arguments
        --version, -v           Version information

        help, --help, -h        This help information

        Available Commands

        init                    Bootstraps a new Divergence project.
        status                  Shows information on the current project.

        config database         Reconfigure database setting.

        test database           Checks if database configuration works by trying to connect to it. Asks you to choose a label name or provide one as the next argument.
```
