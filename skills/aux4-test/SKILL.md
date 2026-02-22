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

### Testing Commands with .aux4 File Fixtures

The most common pattern is creating a `.aux4` file fixture and testing commands against it:

````markdown
# my-tool

```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "greet",
          "execute": ["profile:greet"],
          "help": { "text": "Greetings" }
        }
      ]
    },
    {
      "name": "greet",
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

## greet hello

### with default name

```execute
aux4 greet hello
```

```expect
Hello, World!
```

### with --name flag

```execute
aux4 greet hello --name Alice
```

```expect
Hello, Alice!
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

1. Determine what commands need testing based on the `.aux4` file
2. Create a `.test.md` file for each major command or feature
3. Include a `file:.aux4` block if testing a package's commands
4. Write tests for all command variations (default values, flags, args, errors)
5. Use appropriate expect modifiers for flexible matching
6. Group related tests under descriptive headings
7. When writing `.test.md` files, use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
