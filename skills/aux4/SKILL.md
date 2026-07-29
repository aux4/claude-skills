---
name: aux4
description: Explains how aux4 works - CLI generator, .aux4 files, profiles, commands, variables, executors, and the packaging ecosystem.
user-invocable: true
disable-model-invocation: false
---

# aux4 - CLI Generator

aux4 is a CLI (Command-Line Interface) generator that creates high-level scripts to automate daily tasks. It uses JSON-based `.aux4` configuration files to define command hierarchies.

## Core Concepts

### The .aux4 File

A `.aux4` file is a hidden JSON file that defines profiles and commands. aux4 searches for `.aux4` files in the current directory and parent directories.

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": [
            "echo \"Hello, ${name}!\""
          ],
          "help": {
            "text": "Say hello",
            "variables": [
              {
                "name": "name",
                "text": "Name to greet",
                "default": "World"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

Usage: `aux4 hello --name John` outputs `Hello, John!`

### Profiles

Profiles group related commands. The `main` profile is required and is the default entry point.

Commands can switch to another profile using the `profile:` executor:

```json
{
  "name": "deploy",
  "execute": ["profile:deploy"],
  "help": { "text": "Deployment commands" }
}
```

This means `aux4 deploy <subcommand>` will look for commands in the `deploy` profile.

#### Naming Conventions

- **Profile names**: When creating a nested profile, give the profile the same name as the command that routes to it. For deeper nesting, use `<parent-profile>:<command>` format.
- **Command and profile names**: Use dashes `-` for composed names (e.g., `my-command`, `send-email`).
- **Variable names**: Use camelCase (e.g., `firstName`, `configFile`).

**Nesting example**:
- Command `email` in `main` profile → routes to profile `email`
- Command `list` in `email` profile → routes to profile `email:list`
- Command `config` in `ai:agent` profile → routes to profile `ai:agent:config`

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        { "name": "email", "execute": ["profile:email"], "help": { "text": "Email commands" } }
      ]
    },
    {
      "name": "email",
      "commands": [
        { "name": "send", "execute": ["echo 'sending...'"], "help": { "text": "Send an email" } },
        { "name": "list", "execute": ["profile:email:list"], "help": { "text": "List email commands" } }
      ]
    },
    {
      "name": "email:list",
      "commands": [
        { "name": "inbox", "execute": ["echo 'inbox'"], "help": { "text": "List inbox emails" } },
        { "name": "sent", "execute": ["echo 'sent'"], "help": { "text": "List sent emails" } }
      ]
    }
  ]
}
```

Usage: `aux4 email send`, `aux4 email list inbox`, `aux4 email list sent`

#### Private Commands

Add `"private": true` to a command to hide it from `--help` output. The command still works when called directly:

```json
{
  "name": "internal-setup",
  "execute": ["echo 'setting up...'"],
  "private": true,
  "help": { "text": "Internal setup command" }
}
```

Variables support `private` in the same way — hidden from help, man and autocomplete, while still resolving normally:

```json
{
  "name": "internalToken",
  "text": "internal plumbing, not for users",
  "default": "tok-default",
  "private": true
}
```

Do not confuse it with `hide`, which masks the **value** a user types and leaves the variable listed in the help. `private` hides the variable itself.

### Variables

Variables are parameters for commands. They support:

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Variable name, referenced as `${name}` in execute |
| `text` | string | Description shown in help and prompts |
| `default` | string | Default value (skips prompt if set) |
| `arg` | boolean | Accept as positional argument (only one variable per command can be `arg: true`) |
| `multiple` | boolean | Accept multiple values |
| `env` | string | Read from environment variable |
| `options` | string[] | List of choices for select prompt (with `multiple: true`, shows checkbox list) |
| `hide` | boolean | Mask the value while typing (for passwords); the variable is still listed in help |
| `encrypt` | boolean | Encrypt the value |
| `private` | boolean | Hide the variable from help, man and autocomplete; it still resolves normally |

**Resolution order**: argument > environment variable > config > encrypted > default > interactive prompt

**Special variables**:
- `${response}` - Output of previous command
- `${packageDir}` - Directory of the `.aux4` file
- `${aux4HomeDir}` - aux4 home directory (`~/.aux4.config`)

#### Secret References (`secret://`)

Variable defaults and `set:` instructions can reference secrets using the `secret://` URI format:

```
secret://<provider>/<vault>/<item>/<field>
```

Secrets are resolved at runtime by calling the installed secret provider (e.g., `aux4 secret 1password get`). Multiple secrets from the same provider are batched into a single call for efficiency.

**In variable defaults:**

```json
{
  "name": "apiKey",
  "text": "API key",
  "default": "secret://1password/Work/my-service/credential"
}
```

**In `set:` instructions:**

```json
"execute": [
  "set:token=secret://1password/Work/GitHub/token",
  "curl -H 'Authorization: Bearer ${token}' https://api.example.com"
]
```

**OTP support:** Use the `otp` field to generate a one-time password:

```json
"default": "secret://1password/Work/my-service/otp"
```

If the secret URI has fewer than 3 path segments or doesn't start with `secret://`, it's passed through unchanged.

#### Dot Variables (Nested Parameters)

Variables support dot notation for nested object fields. Use `${person.firstName}` in execute to reference nested values. Users can provide them in two ways:

```bash
# Individual dot-notation flags
aux4 print-person --person.firstName John --person.lastName Doe --person.address.city NY

# JSON object on the parent variable
aux4 print-person --person '{"firstName":"John","lastName":"Doe","address":{"city":"NY"}}'
```

Only the parent variable needs to be declared in `variables` — nested fields are resolved automatically:

```json
{
  "name": "print-person",
  "execute": ["echo ${person.firstName} ${person.lastName} ${person.address.city}"],
  "help": {
    "text": "Print person info",
    "variables": [
      { "name": "person", "text": "The person object" }
    ]
  }
}
```

### Executors (Special Prefixes)

Commands in the `execute` array can use special prefixes:

| Prefix | Example | Purpose |
|--------|---------|---------|
| *(none)* | `echo "hi"` | Run shell command |
| `profile:` | `profile:deploy` | Switch to another profile |
| `set:` | `set:url=https://api.com` | Set a variable to a static value |
| `set:` (command) | `set:result=!curl -s ${url}` | Set variable from command output (stderr shown on error) |
| `set:` (json) | `set:obj=json:${data}` | Parse JSON string into navigable object |
| `log:` | `log:Processing ${file}` | Print output |
| `debug:` | `debug:value is ${x}` | Debug output (AUX4_DEBUG=true) |
| `nout:` | `nout:curl -s ${url}` | Run without output, saves to `${response}` |
| `json:` | `json:${response}` | Parse JSON output |
| `each:` | `each:${response}` | Iterate over lines/array |
| `confirm:` | `confirm:Are you sure?` | Yes/no prompt (bypass with `--yes`) |
| `stdin:` | `stdin:command` | Pass stdin to command |
| `file:` | `file:out.txt:content` | Write content to file (`+` prefix to append) |
| `range:` | `range:5` or `range:1-10` | Generate numeric array in `${response}` |
| `alias:` | `alias:command` | Share stdio with parent |
| `#` | `# this is a comment` | Comment (no-op) |

### Functions (Parameter Formatting)

Used in `execute` strings to format variables:

| Function | Example | Output |
|----------|---------|--------|
| `value(name)` | `command value(file)` | `command myfile.txt` |
| `value(*)` | `command value(*)` | `command '{"host":"localhost","port":3000}'` (all params as JSON) |
| `values(a, b)` | `command values(host, port)` | `command 'localhost' '3000'` |
| `param(name)` | `command param(file)` | `command --file 'myfile.txt'` |
| `param(name:alias)` | `command param(file:f)` | `command --f 'myfile.txt'` |
| `param(name**)` | `command param(tag**)` | `command --tag 'a' --tag 'b'` (one flag per value) |
| `params(a, b)` | `command params(host, port)` | `command --host 'localhost' --port '3000'` |
| `object(a, b)` | `command object(host, port)` | `command '{"host":"localhost","port":3000}'` (supports aliases: `object(data.name:name)`) |
| `nvl(a, b, 'c')` | `log:nvl(avatar, 'default.png')` | First non-null, non-empty value (supports quoted literals) |
| `exists(var)` | `exists(file) && cp ${file} out/` | Test if file at variable path exists |
| `if(var)` | `if(name) && echo yes` | Test if variable has a value |
| `if(var==)` | `if(name==) && echo empty` | Test if variable is empty |
| `if(var==val)` | `if(env==prod) && echo prod` | Test equality |
| `if(var!=val)` | `if(env!=dev) && echo not-dev` | Test inequality |
| `if(!var)` | `if(!debug) && echo off` | Negation |

**Passing parameters to external commands**: When a command calls an external binary or script, **always pass known parameters by index using `values()`**. The external program receives them as positional arguments (`args[0]`, `args[1]`, etc.) and does not need to parse flags or implement config lookup — aux4 handles all variable resolution including `--config` binding from config.yaml.

```json
"execute": ["node ${packageDir}/lib/my-tool.mjs action values(session, url, timeout)"]
```

The binary receives: `my-tool.mjs action <session> <url> <timeout>` — each value at a known index.

**Use `value(*)` only for dynamic parameters** — when the list of expected params is not known in advance. It passes all parameters as a single JSON object:

```json
"execute": ["node ${packageDir}/lib/my-tool.mjs value(*)"]
```

The binary receives: `my-tool.mjs '{"host":"localhost","port":3000,...}'` — a JSON string to parse.

**Prefer `values()` over `value(*)`** when you know the expected parameters. Positional args are simpler, more explicit, and don't require JSON parsing in the target program.

**Nested field access**: Use dot notation to access nested config/variable values. For example, with `person: { firstName: John, lastName: Doe }` in config:
- `values(person.firstName, person.lastName)` resolves to `'John' 'Doe'`
- `values(person, person.firstName, person.lastName)` resolves to `'{"firstName":"John","lastName":"Doe"}' 'John' 'Doe'` — passing the parent object as JSON and the individual fields as separate arguments

This works with `value()`, `values()`, `param()`, `params()`, and `object()`. The external command (JS, Go, etc.) receives these as fixed positional arguments at known indices — it does not need to implement config parsing or key lookup.

**How `--config` binds parameters**: When a user runs `aux4 mytool run --config dev`, aux4 reads the config file and populates command variables automatically. The `values()` function then resolves those variables just like any other. The external command doesn't know or care where the values came from — it just receives positional args.

### Configuration Files (config.yaml)

Requires `aux4/config` package. Use `--configFile config.yaml --config` flags.

```yaml
config:
  dev:
    host: localhost
    port: 3000
```

The `--config` flag automatically extracts values from the config file and populates command variables. You do **not** need to call `aux4 config get` to retrieve individual values — aux4 handles it automatically.

Access nested values with `/`: `aux4 config get dev/host` returns `localhost`. The `config get` command is only needed for programmatic access outside of normal command variable resolution.

### Package Ecosystem

- **hub.aux4.io** - Package registry
- **pkger** - Package manager (`aux4 aux4 pkger install scope/name`)
- **Packages** are ZIP files containing `.aux4`, `LICENSE`, `README.md`, and optional `lib/`, `dist/`, `man/`, `test/` directories
- **Dependencies** - aux4 packages (`"dependencies": ["aux4/config"]`) and system packages (`"system": [["test:node --version", "brew:node"]]`)

### Installation

```bash
curl https://aux4.sh | sh     # Easy install
brew install aux4              # Homebrew
npm install -global aux4       # npm
```

### Investigating Commands

Use `--help` on any command to see its subcommands, variables, and descriptions:

```bash
aux4 --help                    # List all top-level commands
aux4 deploy --help             # List subcommands under deploy
aux4 deploy run --help         # Show variables for deploy run
```

Use `--showSource` to see the execute instructions for a command:

```bash
aux4 deploy run --showSource   # Show the execute array for deploy run
```

### Interactive Shell

`aux4 shell` starts an interactive REPL where you type aux4 commands without the `aux4` prefix. Supports line editing and history. Type `exit` to quit. Also works in batch mode via piped input.

### Built-in Commands

```bash
aux4 aux4 version              # Show version
aux4 aux4 man <command>        # Show command man page
aux4 aux4 source <command>     # Show command source (.aux4 definition)
aux4 aux4 which <command>      # Show where command is defined
```

### Hooks

Hooks are cross-cutting interceptors defined at the package level alongside `profiles`. They run before, after, or on error of any command — including commands from other packages.

```json
{
  "profiles": [],
  "hooks": [
    {
      "command": "main/deploy",
      "order": 10,
      "before": ["log:deploying to ${env}..."],
      "after": ["log:deployed: ${__response}"],
      "error": ["log:failed: ${__error}"]
    }
  ]
}
```

- **Pattern matching**: Command pattern uses `profile/command` format — `*/deploy` matches any `deploy` command, `*/*` matches all commands
- **Conditional params**: `params` field matches variable values — `"env": "production"` or `"env": "dev|staging"` (pipe for alternatives, AND across multiple params)
- **Variable scope**: `before` hooks can inject variables with `set:` that are available in the command and `after`/`error` hooks
- **Built-in variables**: `${__command}`, `${__scope}`, `${__package}`, `${__response}`, `${__error}`, `${__exitCode}`
- **No recursion**: Hooks never trigger other hooks
- **Blocked executors**: `profile:` and `stdin:` are not allowed in hooks
- **Skip hooks**: `--noHooks` flag or `AUX4_NO_HOOKS=true` env var
- **Block hooks per command**: `"noHooks": true` on a command definition
- **Discovery**: `aux4 aux4 hooks`, `aux4 aux4 hooks --command "pattern"`, `aux4 <cmd> --showHooks`

### Package Manager (pkger)

```bash
aux4 aux4 pkger list           # List installed packages
aux4 aux4 pkger man <package>  # Show package documentation (README.md)
aux4 aux4 pkger install <pkg>  # Install a package from hub.aux4.io
aux4 aux4 pkger uninstall <pkg> # Uninstall a package
```

Examples:

```bash
aux4 aux4 pkger list                     # See all installed packages
aux4 aux4 pkger man aux4/config          # Read the config package docs
aux4 aux4 pkger install aux4/db-sqlite   # Install db-sqlite package
```
