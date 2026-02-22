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

For subcommand groups, use `profile:` to create a hierarchy:

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "db",
          "execute": ["profile:db"],
          "help": { "text": "Database commands" }
        }
      ]
    },
    {
      "name": "db",
      "commands": [
        {
          "name": "migrate",
          "execute": ["echo 'migrating...'"],
          "help": { "text": "Run database migrations" }
        },
        {
          "name": "seed",
          "execute": ["echo 'seeding...'"],
          "help": { "text": "Seed the database" }
        }
      ]
    }
  ]
}
```

Usage: `aux4 db migrate`, `aux4 db seed`

For deeper nesting, chain profiles:

```json
{
  "name": "admin",
  "execute": ["profile:db:admin"],
  "help": { "text": "Admin database commands" }
}
```

Profile name: `"db:admin"` with commands like `reset`, `backup`, etc.
Usage: `aux4 db admin reset`

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

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Variable name, used as `${varname}` in execute |
| `text` | string | Description for help output and interactive prompts |
| `default` | string | Default value. If set, skips the interactive prompt |
| `arg` | boolean | Accept as positional argument (e.g., `aux4 cmd value` instead of `--name value`) |
| `multiple` | boolean | Accept multiple values (e.g., `--tag a --tag b`, accessed as `${tag*}`) |
| `env` | string | Read from this environment variable if set |
| `options` | string[] | Present a select menu instead of free-text prompt |
| `hide` | boolean | Hide input when typing (for passwords) |
| `encrypt` | boolean | Encrypt the stored value |

**Resolution order**: CLI argument > environment variable > config file > encrypted > default value > interactive prompt

### Variable Usage in Execute

- Simple: `${varname}`
- Previous command output: `${response}`
- Package directory: `${packageDir}`
- aux4 home: `${aux4HomeDir}`
- Array access: `${items[$index]}`, `${object[$key]}`
- All multiple values: `${tag*}`, specific: `${tag*[0]}`

## Execute Instructions

### Shell Commands

```json
"execute": ["echo \"Hello, ${name}!\""]
```

### Executor Prefixes

| Prefix | Syntax | Description |
|--------|--------|-------------|
| *(none)* | `command args` | Run shell command, output to stdout |
| `profile:` | `profile:name` | Switch to another profile for subcommands |
| `set:` | `set:key=value` | Set a variable for subsequent commands |
| `set:` | `set:key=!command` | Set variable from command output |
| `log:` | `log:message ${var}` | Print to stdout (faster than echo) |
| `debug:` | `debug:message` | Print to stderr only when AUX4_DEBUG=true |
| `nout:` | `nout:command` | Run command silently, output saved to `${response}` |
| `json:` | `json:${response}` | Parse JSON string into accessible object |
| `each:` | `each:${response}` | Iterate over lines or array elements |
| `confirm:` | `confirm:Are you sure?` | Prompt yes/no, aborts on "no" |
| `stdin:` | `stdin:command args` | Pipe stdin from previous command or terminal |
| `alias:` | `alias:command` | Run command sharing parent's stdio |
| `#` | `# comment text` | No-op comment for documentation |

### Parameter Functions

Use in execute strings to format variables for passing to commands:

| Function | Input | Output |
|----------|-------|--------|
| `value(name)` | name="John" | `John` |
| `values(a, b)` | a="x", b="y" | `x y` |
| `param(name)` | name="John" | `--name 'John'` |
| `params(a, b)` | a="x", b="y" | `--a 'x' --b 'y'` |
| `object(a, b)` | a="x", b="y" | `'{"a":"x","b":"y"}'` |

Parameters with empty/default values are omitted automatically.

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

**Call another aux4 command:**
```json
"execute": [
  "aux4 config get dev/host",
  "set:host=${response}",
  "log:Deploying to ${host}"
]
```

## Man Page

Create a markdown file in the `man/` directory. Use double underscores (`__`) to separate command hierarchy levels in the filename:

| Command | Man Page Filename |
|---------|-------------------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |
| `aux4 db migrate` | `db__migrate.md` |
| `aux4 aux4 releaser release` | `aux4_releaser__release.md` |

### Man Page Format

```markdown
#### Description

Brief description of what the command does and when to use it.

#### Usage

\`\`\`bash
aux4 mytool run --format json [--output file]
\`\`\`

#### Example

\`\`\`bash
aux4 mytool run --format csv --output result.csv
\`\`\`

\`\`\`text
Processing complete. Output written to result.csv
\`\`\`
```

## Test

Add tests to the appropriate `.test.md` file. If testing a new command in an existing file, add a new heading section:

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

If the command needs a `.aux4` fixture for testing, add a `file:.aux4` block at the parent heading level. See `/aux4-test` for the full test format.

## Instructions

When adding a command:

1. Read the existing `.aux4` file first to understand the current structure.
2. Add the command to the correct profile. If it's a subcommand of an existing command group, add it to that profile. If it's a new top-level command, add it to the `main` profile.
3. If the new command starts a subcommand group, create a new profile and use `profile:name` routing.
4. Define all variables with at minimum `name` and `text`. Add `default` for optional params, `arg: true` for positional args, `options` for select menus.
5. Use the appropriate executor prefix for each instruction in the execute array.
6. Use parameter functions (`value()`, `values()`, `param()`, `params()`) when passing variables to external commands or binaries.
7. Create a man page with the correct double-underscore filename.
8. Add tests covering the main use case, edge cases, and error cases.
9. Do not modify existing commands unless explicitly asked.
