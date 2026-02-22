---
name: aux4-config
description: Works with aux4 config.yaml files - get, set, merge values, nested paths, and integrating config into aux4 commands.
user-invocable: true
disable-model-invocation: false
argument-hint: [config-question-or-task]
---

# aux4 Config

Help with aux4 configuration files based on: $ARGUMENTS

## Overview

The `aux4/config` package provides commands to read, write, and merge configuration files. It supports YAML and JSON formats with nested path access using `/` as a delimiter.

## Config File Structure

All config files must have a `config` root key:

### YAML (config.yaml or config.yml)

```yaml
config:
  dev:
    host: localhost
    port: 3000
    debug: true
  prod:
    host: aux4.io
    port: 80
    debug: false
  database:
    host: db.example.com
    port: 5432
    name: myapp
    credentials:
      user: admin
      password: secret
```

### JSON (config.json)

```json
{
  "config": {
    "dev": {
      "host": "localhost",
      "port": 3000,
      "debug": true
    },
    "prod": {
      "host": "aux4.io",
      "port": 80,
      "debug": false
    }
  }
}
```

### Supported Data Types

- Strings, numbers (int and float), booleans
- Nested objects (maps)
- Arrays (of strings, objects, or mixed)
- Null values and empty arrays

## Commands

### config get

Retrieve a configuration value by path.

```bash
# Auto-discovers config.yaml, config.yml, or config.json
aux4 config get dev/host
# Output: localhost

# Specify a config file
aux4 config get --file prod.yaml dev/host
# Output: aux4.io

# Get a nested object (returns JSON)
aux4 config get dev
# Output: {"debug":true,"host":"localhost","port":3000}

# Deep nested access
aux4 config get database/credentials/user
# Output: admin

# Non-existent path returns empty output
aux4 config get nonexistent/path
# Output: (empty)
```

**Variables:**

| Name | Description | Default | Type |
|------|-------------|---------|------|
| `file` | Path to config file | auto-discover | `--file` flag |
| `name` | Config key path | *(required)* | positional arg |

### config set

Set a configuration value and save to file.

```bash
# Set an existing value
aux4 config set --name dev/host --value 127.0.0.1

# Create a new nested path (intermediate objects are auto-created)
aux4 config set --name staging/host --value staging.example.com

# Specify config file
aux4 config set --file prod.yaml --name prod/port --value 443
```

**Variables:**

| Name | Description | Default | Type |
|------|-------------|---------|------|
| `file` | Path to config file | auto-discover | `--file` flag |
| `name` | Config key path | *(required)* | `--name` flag |
| `value` | Value to set | *(required)* | positional arg |

### config merge

Merge JSON from stdin into a configuration file.

```bash
# Merge one config into another (output to stdout)
aux4 config get --file dev.yaml | aux4 config merge --file prod.yaml

# Merge and save to file
aux4 config get --file dev.yaml | aux4 config merge --file prod.yaml --save true

# Merge into a specific nested path
cat db.json | aux4 config merge --file config.json --name dev/db

# Merge raw JSON
echo '{"host":"localhost","port":3000}' | aux4 config merge --file config.yaml --name dev --save true
```

**Variables:**

| Name | Description | Default | Type |
|------|-------------|---------|------|
| `file` | Target config file | auto-discover | `--file` flag |
| `name` | Path within config to merge into | root level | positional arg |
| `save` | Save merged result to file | `false` | `--save` flag |

**Merge behavior:**
- Deep merge (recursive for nested objects)
- Duplicate keys are overwritten by the incoming data
- Keys are alphabetically sorted in output
- File format (YAML/JSON) is preserved when saving

## Using Config in aux4 Commands

### The --config Flag

The `--config` flag lets commands pull variables from a config file. This is built into aux4 core.

```json
{
  "name": "connect",
  "execute": [
    "log:connecting to ${host}:${port}"
  ],
  "help": {
    "text": "Connect to a server",
    "variables": [
      { "name": "host", "text": "Server host" },
      { "name": "port", "text": "Server port" }
    ]
  }
}
```

With `config.yaml`:
```yaml
config:
  dev:
    host: localhost
    port: 3000
  prod:
    host: aux4.io
    port: 80
```

Usage:
```bash
aux4 connect --config dev
# Output: connecting to localhost:3000

aux4 connect --config prod
# Output: connecting to aux4.io:80
```

### Using --configFile

When the config file is not in the default location or has a custom name:

```bash
aux4 connect --configFile environments.yaml --config staging
```

### Reading Config in Execute Arrays

Use `aux4 config get` within execute instructions to read config values programmatically:

```json
{
  "name": "deploy",
  "execute": [
    "nout:aux4 config get --file deploy.yaml ${env}/host",
    "set:host=${response}",
    "nout:aux4 config get --file deploy.yaml ${env}/port",
    "set:port=${response}",
    "log:Deploying to ${host}:${port}"
  ],
  "help": {
    "text": "Deploy to an environment",
    "variables": [
      { "name": "env", "text": "Target environment", "options": ["dev", "staging", "prod"] }
    ]
  }
}
```

### Adding aux4/config as a Dependency

In your package `.aux4` file:

```json
{
  "scope": "myscope",
  "name": "mypackage",
  "version": "0.1.0",
  "dependencies": ["aux4/config"],
  "profiles": [...]
}
```

This ensures the config commands are available when your package is installed.

## File Auto-Discovery

When `--file` is not specified, the config package searches the current directory for:
1. `config.yaml`
2. `config.yml`
3. `config.json`

If none are found, it returns an error:
```
Error: No config file found (config.json, config.yaml, or config.yml)
```

## Output Behavior

- **Scalar values** (string, number, boolean): printed as plain text
- **Objects**: printed as JSON with alphabetically sorted keys
- **Arrays**: printed as JSON
- **Non-existent paths**: empty output (no error)

## Error Handling

| Scenario | Error Message |
|----------|---------------|
| No config file found | `Error: No config file found (config.json, config.yaml, or config.yml)` |
| Invalid JSON | `Error loading config: failed to parse JSON: [details]` |
| Invalid YAML | `Error loading config: yaml: [details]` |
| Missing required args | Command-specific error message |

## Common Patterns

### Environment-based configuration

```yaml
config:
  dev:
    api_url: http://localhost:3000
    debug: true
  staging:
    api_url: https://staging.example.com
    debug: true
  prod:
    api_url: https://api.example.com
    debug: false
```

```bash
aux4 mycommand --config dev    # Uses dev settings
aux4 mycommand --config prod   # Uses prod settings
```

### Database configuration

```yaml
config:
  database:
    host: localhost
    port: 5432
    name: myapp
    user: admin
```

```bash
aux4 config get database/host   # localhost
aux4 config get database        # {"host":"localhost","name":"myapp","port":5432,"user":"admin"}
```

### Updating deployment config

```bash
# Update a single value
aux4 config set --name prod/version --value 2.1.0

# Merge a whole section
echo '{"replicas":3,"memory":"512m"}' | aux4 config merge --file config.yaml --name prod/resources --save true
```

## Instructions

When working with aux4 config files:

1. Always use the `config` root key in YAML and JSON files.
2. Use `/` to separate nested path levels (e.g., `dev/host`, `database/credentials/user`).
3. Prefer `--config` flag for environment-based command execution over manual `config get` calls.
4. Add `aux4/config` to package dependencies when your package needs config support.
5. Remember that `config get` returns JSON for objects and plain text for scalars.
6. Use `config merge --save true` to persist changes; without `--save`, output goes to stdout.
7. Config files are auto-discovered in order: `config.yaml`, `config.yml`, `config.json`.
