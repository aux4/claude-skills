---
name: aux4-command
description: Adds commands to existing .aux4 files with correct profiles, variables, executors, man pages, and tests.
user-invocable: true
disable-model-invocation: false
argument-hint: [command-description]
---

# Add an aux4 Command

Add a command to an existing `.aux4` file based on: $ARGUMENTS

## How to Add a Command

1. Read the existing `.aux4` file to understand the current profiles and commands.
2. Determine if the new command belongs in an existing profile or needs a new one.
3. Add the command with proper variable definitions and execute instructions.
4. Create a man page in `man/` (or `package/man/` if inside a package).
5. Add tests in `test/` (or `package/test/`).

## Command Structure

```json
{
  "name": "command-name",
  "execute": [
    "instruction1",
    "instruction2"
  ],
  "help": {
    "text": "Short description of what this command does",
    "variables": [
      {
        "name": "varname",
        "text": "Description for help and prompts"
      }
    ]
  }
}
```

### Optional Command Properties

```json
{
  "name": "command-name",
  "private": true,
  "execute": [...],
  "help": { ... }
}
```

- `private: true` hides the command from help output (useful for internal/helper commands).

## Profile Routing

For subcommand groups, use `profile:name` to route to another profile. The profile name must match the command that routes to it. For deeper nesting, use `<parent>:<command>` as the profile name (e.g., `email:list`, `ai:agent:config`).

- **Command/profile names**: dashes for composed names (`send-email`, `my-command`)
- **Variable names**: camelCase (`firstName`, `configFile`)

See `/aux4` for full profile routing examples, nesting patterns, and naming conventions.

## Variable Definitions

```json
{
  "name": "varname",
  "text": "Description shown in help and prompts",
  "default": "value",
  "arg": true,
  "multiple": false,
  "env": "ENV_VAR_NAME",
  "options": ["choice1", "choice2", "choice3"],
  "hide": false,
  "encrypt": false
}
```

See `/aux4` for the full property reference, resolution order, special variables (`${response}`, `${packageDir}`, `${aux4HomeDir}`), and dot variable (nested parameter) syntax.

## Execute Instructions

### Shell Commands

```json
"execute": ["echo \"Hello, ${name}!\""]
```

### Executor Prefixes

Common prefixes: `profile:`, `set:`, `log:`, `nout:`, `json:`, `each:`, `confirm:`, `stdin:`, `alias:`, `debug:`, `#`. Use parameter functions (`value()`, `values()`, `param()`, `params()`, `object()`) to format variables for external commands. See `/aux4` for the full executor and parameter function reference tables.

### Conditional Execution

```json
"if(env==prod) && deploy.sh || log:skipping deploy"
```

Operators: `==`, `!=`

### Common Patterns

**Run command, capture output, use it:**
```json
"execute": [
  "nout:curl -s https://api.example.com/data",
  "json:${response}",
  "log:Got ${response.name} with ${response.count} items"
]
```

**Iterate over results:**
```json
"execute": [
  "nout:cat items.json",
  "json:${response}",
  "each:${response} echo \"Item: ${value}\""
]
```

**Set variables from command output:**
```json
"execute": [
  "set:version=!cat package/.aux4 | jq -r .version",
  "log:Current version is ${version}"
]
```

**Conditional build:**
```json
"execute": [
  "if(noBuild==false) && aux4 build || true"
]
```

**Confirm before destructive action:**
```json
"execute": [
  "confirm:This will delete all data. Continue?",
  "rm -rf data/"
]
```

**Call binary with stdin:**
```json
"execute": [
  "stdin:${packageDir}/my-tool process values(format, output)"
]
```

**Use config values directly (--config auto-populates variables):**
```json
"execute": [
  "log:Deploying to ${host}:${port}"
]
```
Usage: `aux4 deploy --config dev` — aux4 extracts `host` and `port` from the config file automatically. No need to call `aux4 config get`.

## Man Page

Every command should have a man page. Create a markdown file in the `man/` directory (or `package/man/` if inside a package). Use double underscores (`__`) to separate command hierarchy levels in the filename:

| Command | Man Page Filename |
|---------|-------------------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |
| `aux4 db migrate` | `db__migrate.md` |
| `aux4 aux4 releaser release` | `aux4_releaser__release.md` |
| `aux4 email list all` | `email_list__all.md` |

**Special characters in names**: If a profile or command name contains special characters (like `:`), replace them with a single `_` in the filename. For example, profile `email:list` with command `all` becomes `email_list__all.md`, not `email:list__all.md`.

Man pages are displayed when running `aux4 aux4 man <command>`.

### Man Page Format

Use `####` headings for Description, Usage, and Example sections. The description should explain what the command does in detail, not just repeat the help text. Include all flags and their defaults, and show realistic examples with expected output.

````markdown
#### Description

The `run` command processes input files using the specified format and writes the result to stdout or an output file. It supports multiple output formats and automatically detects the input encoding.

#### Usage

```bash
aux4 mytool run --format <json|csv|yaml> [--output <file>]
```

--format  The output format to use (required)
--output  Path to write the output file. If not provided, output goes to stdout

#### Example

```bash
aux4 mytool run --format csv --output result.csv
```

```text
Processing complete. Output written to result.csv
```
````

### Real-World Example

````markdown
#### Description

The `install` command automates packaging and installing a local build of your aux4 package. It reads the project metadata from the `.aux4` file in the specified directory, temporarily appends a `-local` suffix to the version, and generates a zip archive. After building, it uninstalls any existing version, installs the freshly built local package, and restores the original version in your project file.

#### Usage

```bash
aux4 aux4 releaser install [--dir <path>] [--rm <true|false>]
```

--dir   The path to the directory containing your `.aux4` package definition (default: `.`)
--rm    Remove the generated local zip file after installation (default: `false`)

#### Example

```bash
aux4 aux4 releaser install --dir ./my-package --rm true
```

This command will:
1. Load the version from `./my-package/.aux4`.
2. Update the version to `<current>-local` and build a zip artifact.
3. Uninstall any existing published version of this package.
4. Install the newly built local package using aux4 pkger.
5. Restore the original version string in `.aux4`.
6. Remove the local zip file, since `--rm true` was specified.

```text
The package myscope/my-package:0.1.0-local has been installed
```
````

## Test

Add tests in `test/` (or `package/test/`). Name the file using double underscores for command hierarchy, matching the man page convention (e.g., `mytool__run.test.md` for `aux4 mytool run`). If the profile or command name contains special characters (like `:`), replace them with a single `_` (e.g., `email_list__all.test.md` for profile `email:list` command `all`). Test files are published to hub.aux4.io as usage examples, so they also serve as documentation.

If testing a new command in an existing file, add a new heading section:

````markdown
## mytool run

### with default format

```execute
aux4 mytool run
```

```expect
Processing with format: json
```

### with csv format

```execute
aux4 mytool run --format csv
```

```expect
Processing with format: csv
```

### with invalid format

```execute
aux4 mytool run --format xml
```

```error:partial
Error: unsupported format *?
```
````

**Important**: If the test lives inside `package/test/`, do NOT create a `file:.aux4` block — the test runner auto-discovers `package/.aux4` from the parent directory. Only use `file:.aux4` for standalone tests outside a package. See `/aux4-test` for the full test format.

## Instructions

When adding a command:

1. Read the existing `.aux4` file first to understand the current structure.
2. Add the command to the correct profile. If it's a subcommand of an existing command group, add it to that profile. If it's a new top-level command, add it to the `main` profile.
3. If the new command starts a subcommand group, create a new profile and use `profile:name` routing.
4. Define all variables with at minimum `name` and `text`. Add `default` for optional params, `arg: true` for positional args, `options` for select menus.
5. Use the appropriate executor prefix for each instruction in the execute array.
6. Use parameter functions (`value()`, `values()`, `param()`, `params()`) when passing variables to external commands or binaries.
7. Create a man page and update the README.md. See `/aux4-docs` for the full documentation format, conventions, and what to update.
8. Add tests covering the main use case, edge cases, and error cases. For package tests (in `package/test/`), call `aux4 <command>` directly — do NOT use `file:.aux4` blocks. The test runner auto-discovers `package/.aux4` from the parent directory.
10. If the new command requires an external tool (CLI binary, library, or runtime), add it to the `system` array in the package `.aux4`: `["test:tool --version", "brew:tool", "linux:tool"]`. The `test:` entry checks if it's already installed; the remaining entries are install options. See `/aux4-package` for the full system dependencies format.
11. Do not modify existing commands unless explicitly asked.
12. When writing markdown files (man pages, `.test.md`), use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
13. **Always format JSON with indentation** in `.aux4` files, documentation, and test fixtures. Never use single-line compact JSON. Each item in the `execute` array must be on its own line.
