### [⤺ Back to Table of Contents](/README.md#divergence-framework-documentation)

# Command Line Tool
The command line tool is designed to help you bootstrap a project, build models, edit configs, test database connections, and above all speed up your work.

## Installation
Make sure you have Composer installed.

`composer global require divergence/cli`

The CLI is a separate package from the framework itself, so if your local installed command set differs from the historical examples below, use `divergence --help` as the final authority for your installed version.

## Initializing a Project
Make sure a `composer.json` file exists in the folder where you run this command.

`divergence init`

The command is expected to add `divergence/divergence` to your dependencies, set up a PSR-4 namespace under the `src` directory, copy necessary framework files into your project folder, and start a database configuration wizard which edits the database config for you.

You can watch a video of the process below.
[![asciicast](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs.png)](https://asciinema.org/a/FhE9hATLKDhH7oQfFbeNG5hzs)

## Testing Your Database Config

To have a select menu come up with all the database configs run this command:

`divergence test config`

Some historical help output and docs also refer to this as:

`divergence test database`

You can optionally provide the label, but if you do not you can select from the menu that comes up.

It returns a simple success or failure message after trying to connect.

## Change Your Database Config

To have a select menu come up with all the database configs run this command:

`divergence config database`

You can optionally provide the label, but if you do not you can select from the menu that comes up.

Once you select which one to edit a wizard will start using the old config as default. The config is rewritten to disk once the wizard is done.

## Tool Usage Reference
Historically documented help output:

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
