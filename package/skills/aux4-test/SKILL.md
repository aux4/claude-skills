---
name: aux4-test
description: Creates and runs aux4 .test.md files for testing aux4 packages. Markdown-based test format with execute, expect, error, file blocks, and hooks.
user-invocable: true
disable-model-invocation: false
argument-hint: [test-description]
---

# aux4 Test Framework

Create or run aux4 tests based on: $ARGUMENTS

## Overview

aux4 uses markdown-based `.test.md` files for testing. Tests are placed in the `package/test/` directory. The test runner parses markdown headings and fenced code blocks to build and execute test suites.

## Running Tests

```bash
# Run all tests in package/test/
aux4 test run

# Run a specific test file
aux4 test run package/test/mycommand.test.md

# Add a test programmatically
aux4 test add mytest.test.md --level 2 --name "Test Name" --execute "echo hello"
```

## Test File Format

Test files use markdown with special fenced code blocks. The heading hierarchy (`#`, `##`, `###`, etc.) defines test grouping (like `describe` blocks).

### Basic Test Structure

````markdown
# Command Name

## Scenario Description

### should do something

```execute
aux4 mycommand --flag value
```

```expect
expected output
```
````

### Complete Example

````markdown
# greet hello

## with default name

### should greet World

```execute
aux4 greet hello
```

```expect
Hello, World!
```

## with custom name

### should greet the given name

```execute
aux4 greet hello --name John
```

```expect
Hello, John!
```
````

## Code Block Types

### `execute` - Run a Command

The shell command to execute. Captures stdout and stderr separately.

````markdown
```execute
aux4 config get dev/host
```
````

### `expect` - Validate stdout

Exact match against stdout by default. Must follow an `execute` block.

````markdown
```expect
localhost
```
````

### `expect` with Modifiers

#### `:partial` - Substring/Wildcard Matching

Matches a substring of the output. Supports wildcards:
- `*?` - matches any single line content
- `*` - matches any content on the same line
- `**` - matches any content across multiple lines

````markdown
```expect:partial
Hello *?
```
````

````markdown
```expect:partial
Start ** end
```
````

#### `:ignoreCase` - Case-Insensitive

````markdown
```expect:ignoreCase
hello world
```
````

#### `:regex` - Regular Expression

````markdown
```expect:regex
Hello, \w+!
```
````

#### Combined Modifiers

Modifiers can be combined with `:`:

````markdown
```expect:regex:ignoreCase
error code: [a-z]+\d+
```
````

````markdown
```expect:partial:ignoreCase
hello *?
```
````

### `error` - Validate stderr

Same syntax and modifiers as `expect`, but matches against stderr:

````markdown
```error
Error: file not found
```
````

````markdown
```error:partial
Error: *?
```
````

### Combined stdout and stderr

You can validate both stdout and stderr for the same execute:

````markdown
```execute
echo "Success" && echo "Warning" >&2
```

```expect
Success
```

```error
Warning
```
````

### `file:<filename>` - Create Test Fixtures

Creates a file before the test runs. Scoped to the current heading and nested headings.

````markdown
```file:config.yaml
config:
  dev:
    host: localhost
    port: 3000
```
````

````markdown
```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": ["echo \"Hello, ${name}!\""],
          "help": {
            "text": "Say hello",
            "variables": [
              { "name": "name", "text": "Name", "default": "World" }
            ]
          }
        }
      ]
    }
  ]
}
```
````

**Scoping rules:**
- Files declared at a parent heading are available to all nested tests
- Files are NOT shared between sibling headings
- Files are automatically cleaned up after tests

### `timeout` - Set Command Timeout

Override the default timeout (in milliseconds). Place before the `execute` block:

````markdown
```timeout
5000
```

```execute
sleep 2 && echo "Done"
```

```expect
Done
```
````

### `beforeAll` / `afterAll` - Scenario Hooks

Run once before/after all tests in the current heading scope:

````markdown
## My Scenario

```beforeAll
mkdir -p test-dir
echo "ready" > test-dir/setup.log
```

```afterAll
rm -rf test-dir
```

### test 1

```execute
cat test-dir/setup.log
```

```expect
ready
```
````

### `beforeEach` / `afterEach` - Per-Test Hooks

Run before/after each individual test in the current heading scope:

````markdown
## My Scenario

```beforeEach
echo "fresh" > state.txt
```

```afterEach
rm -f state.txt
```

### test 1

```execute
cat state.txt
```

```expect
fresh
```
````

## Writing Tests for aux4 Packages

### Package Tests (in `package/test/`)

When tests live inside `package/test/`, the test runner automatically discovers the `package/.aux4` file from the parent directory. **Do NOT create `file:.aux4` blocks** — just call `aux4 <command>` directly:

````markdown
# greet hello

## with default name

### should greet World

```execute
aux4 greet hello
```

```expect
Hello, World!
```

## with custom name

### should greet the given name

```execute
aux4 greet hello --name Alice
```

```expect
Hello, Alice!
```
````

### Standalone Tests (outside a package)

Only use `file:.aux4` blocks when testing commands that are NOT part of a package (e.g., standalone test files or testing inline command definitions):

````markdown
# my standalone test

```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "hello",
          "execute": ["echo \"Hello, ${name}!\""],
          "help": {
            "text": "Say hello",
            "variables": [
              { "name": "name", "text": "Name", "default": "World" }
            ]
          }
        }
      ]
    }
  ]
}
```

## should greet World

```execute
aux4 hello
```

```expect
Hello, World!
```
````

### Testing Config Files

````markdown
# config get

```file:config.yaml
config:
  dev:
    host: localhost
    port: 3000
```

## get nested value

```execute
aux4 config get dev/host
```

```expect
localhost
```
````

### Testing Error Cases

````markdown
# error handling

## invalid input

```execute
aux4 config get --file nonexistent.yaml dev
```

```error:partial
Error *?
```
````

### Testing JSON Output

````markdown
# json output

```execute
aux4 config get dev | jq .
```

```expect
{
  "host": "localhost",
  "port": 3000
}
```
````

### Testing Long-Running Processes (Servers, Daemons)

When tests require a background server or daemon, use `beforeAll` to start it and `afterAll` to stop it. Use `nohup` with output redirection to prevent the hook from blocking:

````markdown
# queue create

```beforeAll
nohup aux4 queue start --port 18420 >/dev/null 2>&1 &
sleep 1
```

```afterAll
aux4 queue stop --port 18420
```

## should create a queue

```execute
aux4 queue create --name myqueue --port 18420
```

```expect
queue "myqueue" created
```
````

**Key points:**
- Use `nohup ... >/dev/null 2>&1 &` to background the process without blocking the `beforeAll` hook
- Add `sleep 1` after starting to allow the server time to initialize
- Always clean up in `afterAll` (stop server, remove temp files)
- Use unique ports per test file to avoid conflicts when running tests in parallel

### Preparing Go Packages for Testing

For Go packages (with `dist/` binaries), you must build and create a symlink before running tests. The `package/.aux4` references the binary as `${packageDir}/<binary-name>`, so a symlink must exist at `package/<binary-name>`:

```bash
# Build all platforms
aux4 build

# Create symlink for current platform
ln -sf dist/$(uname -s | tr '[:upper:]' '[:lower:]')/$(uname -m | sed 's/x86_64/amd64/;s/aarch64\|arm64/arm64/;s/i[36]86/386/')/<binary-name> package/<binary-name>

# Run tests
aux4 test run
```

## Test File Naming Conventions

- Place in `package/test/` directory
- Use double underscores (`__`) to separate command hierarchy levels in the filename, matching the man page convention:

| Command | Test File |
|---------|-----------|
| `aux4 greet` | `greet.test.md` |
| `aux4 greet hello` | `greet__hello.test.md` |
| `aux4 config get` | `config__get.test.md` |
| `aux4 pdf parse` | `pdf__parse.test.md` |
| `aux4 repository read` | `repository__read.test.md` |
| `aux4 email list all` | `email_list__all.test.md` |

**Special characters in names**: If a profile or command name contains special characters (like `:`), replace them with a single `_` in the filename. For example, profile `email:list` with command `all` becomes `email_list__all.test.md`.

- Test files are published to hub.aux4.io as usage examples alongside man pages, so they also serve as documentation
- Use descriptive heading hierarchy:
  - `#` - Command or feature name
  - `##` - Scenario or context
  - `###` - Individual test case (prefixed with "should" for clarity)

## Best Practices

1. Start each test file with a `#` heading describing the command being tested
2. Use `file:` blocks at the appropriate scope level for test fixtures
3. Prefer `expect:partial` for output that may vary (timestamps, paths)
4. Use `expect:regex` for pattern matching complex output
5. Keep tests focused - one `execute` block per test case
6. Clean up side effects with `afterAll` or `afterEach` hooks
7. Test both success and error cases
8. Use meaningful heading names that describe the behavior being tested

## Instructions

When creating tests:

1. Determine what commands need testing based on the `.aux4` file.
2. Create a `.test.md` file for each major command or feature.
3. For package tests (in `package/test/`), call `aux4 <command>` directly — do NOT use `file:.aux4` blocks. The test runner auto-discovers `package/.aux4` from the parent directory. Only use `file:.aux4` for standalone tests outside a package.
4. Write tests for all command variations (default values, flags, args, errors).
5. Use appropriate expect modifiers for flexible matching.
6. Group related tests under descriptive headings.
7. For tests that need a background server or daemon, use `nohup ... >/dev/null 2>&1 &` in `beforeAll` and clean up in `afterAll`.
8. For Go packages, ensure the binary is built and a symlink exists at `package/<binary-name>` before running tests.
9. When writing `.test.md` files, use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
10. **Always format JSON with indentation** in `file:` blocks (`.aux4`, `config.yaml`, fixture files). Never use single-line compact JSON for `.aux4` definitions or config fixtures. Each item in the `execute` array must be on its own line.
