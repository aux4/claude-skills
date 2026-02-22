---
name: aux4-package
description: Creates aux4 packages with .aux4 metadata, LICENSE, README, tests, man pages, and GitHub Actions for publishing to hub.aux4.io.
user-invocable: true
disable-model-invocation: false
argument-hint: [package-description]
---

# Create an aux4 Package

Create a complete aux4 package based on: $ARGUMENTS

## Package Structure

Every aux4 package lives inside a `package/` directory with this structure:

```
package/
├── .aux4           # REQUIRED: Package metadata and command definitions
├── LICENSE         # REQUIRED: License file
├── README.md       # REQUIRED: Package documentation
├── man/            # Optional: Command manual pages
│   └── command__subcommand.md
├── test/           # Optional: Test files
│   └── command.test.md
├── lib/            # Optional: Scripts and supporting files
└── dist/           # Optional: Platform-specific binaries
    ├── darwin/
    │   ├── amd64/
    │   └── arm64/
    ├── linux/
    │   ├── amd64/
    │   ├── arm64/
    │   └── 386/
    └── windows/
        ├── amd64/
        ├── arm64/
        └── 386/
```

## Step 1: Create the .aux4 File

The `.aux4` file is JSON with package metadata and command profiles.

### Package Metadata Fields

```json
{
  "scope": "<org-or-username>",
  "name": "<package-name>",
  "version": "1.0.0",
  "description": "<short description>",
  "license": "MIT",
  "tags": ["tag1", "tag2"],
  "dependencies": ["aux4/config"],
  "system": [
    ["test:node --version", "brew:node", "apt:nodejs", "dnf:nodejs", "apk:nodejs"]
  ],
  "profiles": [...]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `scope` | Yes | Package namespace (org or username) |
| `name` | Yes | Package name |
| `version` | Yes | Semantic version (e.g., `1.0.0`) |
| `description` | Yes | Short description |
| `license` | Yes | License identifier (MIT, Apache-2.0, etc.) |
| `tags` | No | Searchable tags |
| `dependencies` | No | Other aux4 packages needed (`"scope/name"` or `"scope/name@version"`) |
| `system` | No | System dependencies (see below) |
| `profiles` | Yes | Array of command profiles |

### System Dependencies Format

When a package requires external tools (CLI binaries, libraries, or runtimes), declare them in `system`. Each entry is an array where:

- The first element is a `test:` command that checks if the tool is already installed
- The remaining elements are `<prefix>:<package>` pairs for installing it with different package managers

```json
"system": [
  ["test:node --version", "brew:node", "apt:nodejs", "linux:nodejs"],
  ["test:jq --version", "brew:jq", "apt:jq", "dnf:jq"],
  ["test:prettier --version", "npm:prettier"]
]
```

When the package is installed via pkger, aux4 runs the `test:` command first. If it fails (tool not found), it tries to install using one of the available system installers: `aux4 aux4 pkger system <prefix> install <package>`.

| Prefix | Package Manager | Platform |
|--------|----------------|----------|
| `brew` | Homebrew | macOS |
| `apt` | APT | Debian/Ubuntu |
| `dnf` | DNF | Fedora/RHEL |
| `yum` | YUM | CentOS/RHEL |
| `apk` | APK | Alpine |
| `npm` | npm | Cross-platform (Node.js) |
| `pkgx` | pkgx | Cross-platform |
| `linux` | Alias for `apt` + `dnf` + `yum` + `apk` | All Linux |

Common examples:

```json
"system": [
  ["test:node --version", "brew:node", "linux:nodejs"],
  ["test:go version", "brew:go", "linux:golang"],
  ["test:python3 --version", "brew:python3", "apt:python3"],
  ["test:prettier --version", "npm:prettier"],
  ["test:jq --version", "brew:jq", "apt:jq", "apk:jq"]
]
```

### Profiles and Commands

The `main` profile is the entry point. Use `profile:` executor for subcommand groups:

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "mytool",
          "execute": ["profile:mytool"],
          "help": { "text": "My tool commands" }
        }
      ]
    },
    {
      "name": "mytool",
      "commands": [
        {
          "name": "run",
          "execute": ["echo \"Running ${file}\""],
          "help": {
            "text": "Run a file",
            "variables": [
              {
                "name": "file",
                "text": "File to run",
                "arg": true
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Variable Definitions

```json
{
  "name": "varname",
  "text": "Description shown to user",
  "default": "optional-default",
  "arg": true,
  "multiple": false,
  "env": "ENV_VAR_NAME",
  "options": ["choice1", "choice2"],
  "hide": false,
  "encrypt": false
}
```

### Execute Instructions

Commands can use these prefixes:

- Shell command: `"echo Hello"`
- Profile switch: `"profile:subprofile"`
- Binary: `"${packageDir}/my-binary value(arg1)"`
- Script: `"node ${packageDir}/lib/script.js values(a, b)"`
- Set variable: `"set:url=https://api.com"`
- Log: `"log:Processing ${file}"`
- No output: `"nout:curl -s ${url}"`
- JSON parse: `"json:${response}"`
- Iterate: `"each:${response}"`
- Confirm: `"confirm:Are you sure?"`
- Stdin: `"stdin:command values(a, b)"`
- Alias: `"alias:command"`
- Comment: `"# step description"`

Parameter functions:
- `value(name)` - raw value
- `values(a, b, c)` - multiple raw values
- `param(name)` - as `--name 'value'`
- `params(a, b)` - multiple params
- `object(a, b)` - as JSON object

## Step 2: Create the LICENSE File

Use a standard license text (MIT, Apache-2.0, etc.). You can also generate it with:

```bash
aux4 license use --name mit --owner "Owner" --year 2025 --project "project-name"
```

## Step 3: Create the README.md

Include: description, installation instructions, usage examples, and command reference.

## Step 4: Create Man Pages

Place markdown files in `package/man/`. Name them using double underscores for command hierarchy:

| Command | Man Page Filename |
|---------|-------------------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |

Man pages provide detailed documentation for each command, displayed via `aux4 aux4 man <command>`. Use `####` headings for Description, Usage, and Example:

````markdown
#### Description

The `run` command processes input files and outputs the result. It supports multiple formats and automatically detects the input encoding. If no output file is specified, the result is written to stdout.

#### Usage

```bash
aux4 mytool run --format <json|csv|yaml> [--output <file>]
```

--format  The output format to use (required)
--output  Path to write the output file (default: stdout)

#### Example

```bash
aux4 mytool run --format csv --output result.csv
```

```text
Processing complete. Output written to result.csv
```
````

The description should go beyond the short help text — explain what the command does, when to use it, how flags interact, and any important behaviors. Include realistic examples with expected output. Create one man page per command.

## Step 5: Create Tests (Optional)

Place `.test.md` files in `package/test/`. Name files using double underscores for command hierarchy (e.g., `mytool__run.test.md` for `aux4 mytool run`), matching the man page convention. Test files are published to hub.aux4.io as usage examples, so they also serve as documentation. See the `/aux4-test` skill for the full test format.

Quick example:

````markdown
# mytool run

## should output the filename

```file:.aux4
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "run",
          "execute": ["echo \"Running ${file}\""],
          "help": {
            "text": "Run a file",
            "variables": [
              { "name": "file", "text": "File to run", "arg": true }
            ]
          }
        }
      ]
    }
  ]
}
```

```execute
aux4 run myfile.txt
```

```expect
Running myfile.txt
```
````

## Step 6: Create Build Configuration

Every package needs a root `.aux4` file (outside `package/`) with at least a `build` command. There are two types of packages:

### Go Package (Multi-Platform Binaries)

Go packages compile to native binaries for each platform. The root `.aux4` defines cross-compilation:

```
my-go-package/
├── .aux4              # Build configuration (builder profiles)
├── go.mod
├── go.sum
├── main.go            # Go source code
└── package/
    ├── .aux4          # Package metadata (commands use ${packageDir}/my-binary)
    ├── LICENSE
    ├── README.md
    ├── man/
    ├── test/
    └── dist/          # Compiled binaries per platform
        ├── darwin/amd64/my-binary
        ├── darwin/arm64/my-binary
        ├── linux/amd64/my-binary
        ├── linux/arm64/my-binary
        ├── linux/386/my-binary
        ├── windows/amd64/my-binary.exe
        ├── windows/arm64/my-binary.exe
        └── windows/386/my-binary.exe
```

Root `.aux4` for Go packages:

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "build",
          "execute": ["aux4 builder all"],
          "help": { "text": "Build for all platforms" }
        },
        {
          "name": "builder",
          "execute": ["profile:builder"],
          "help": { "text": "Platform-specific builds" }
        }
      ]
    },
    {
      "name": "builder",
      "commands": [
        {
          "name": "darwin",
          "execute": [
            "GOOS=darwin GOARCH=amd64 go build -o package/dist/darwin/amd64/my-binary .",
            "GOOS=darwin GOARCH=arm64 go build -o package/dist/darwin/arm64/my-binary ."
          ],
          "help": { "text": "Build for macOS" }
        },
        {
          "name": "linux",
          "execute": [
            "GOOS=linux GOARCH=amd64 go build -o package/dist/linux/amd64/my-binary .",
            "GOOS=linux GOARCH=arm64 go build -o package/dist/linux/arm64/my-binary .",
            "GOOS=linux GOARCH=386 go build -o package/dist/linux/386/my-binary ."
          ],
          "help": { "text": "Build for Linux" }
        },
        {
          "name": "windows",
          "execute": [
            "GOOS=windows GOARCH=amd64 go build -o package/dist/windows/amd64/my-binary.exe .",
            "GOOS=windows GOARCH=arm64 go build -o package/dist/windows/arm64/my-binary.exe .",
            "GOOS=windows GOARCH=386 go build -o package/dist/windows/386/my-binary.exe ."
          ],
          "help": { "text": "Build for Windows" }
        },
        {
          "name": "all",
          "execute": [
            "aux4 builder darwin",
            "aux4 builder linux",
            "aux4 builder windows",
            "chmod -R +x package/dist/*"
          ],
          "help": { "text": "Build for all platforms" }
        }
      ]
    }
  ]
}
```

`${packageDir}` always points to the directory where the `package/.aux4` file lives (the `package/` folder), so `${packageDir}/dist/...` resolves to `package/dist/...`. Commands reference the binary directly:

```json
"execute": ["${packageDir}/my-binary value(arg1)"]
```

Or with stdin:

```json
"execute": ["stdin:${packageDir}/my-binary command values(a, b)"]
```

### JavaScript Package (Rollup Bundle)

JS packages bundle source + dependencies into a single file using Rollup. They require Node.js at runtime and declare it as a system dependency. The bundled output goes to `package/lib/`.

`${packageDir}` always points to the directory where the `package/.aux4` file lives (the `package/` folder), so `${packageDir}/lib/my-tool.mjs` resolves to `package/lib/my-tool.mjs`.

```
my-js-package/
├── .aux4              # Build configuration (npm run build)
├── package.json       # NPM dependencies and build scripts
├── rollup.config.js   # Rollup bundler configuration
├── bin/
│   └── executable.js  # Entry point (#!/usr/bin/env node)
├── lib/               # Source code
│   └── ...
└── package/
    ├── .aux4          # Package metadata (commands use node ${packageDir}/lib/...)
    ├── LICENSE
    ├── README.md
    ├── man/
    ├── test/
    └── lib/
        └── my-tool.mjs  # Bundled output (single file)
```

Root `.aux4` for JS packages:

```json
{
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "build",
          "execute": ["npm run build"],
          "help": { "text": "Build the package" }
        }
      ]
    }
  ]
}
```

`package.json` with Rollup build:

```json
{
  "name": "@scope/my-tool",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "rollup -c"
  },
  "dependencies": {
    "colors": "^1.4.0"
  },
  "devDependencies": {
    "@rollup/plugin-commonjs": "^28.0.6",
    "@rollup/plugin-json": "^6.1.0",
    "@rollup/plugin-node-resolve": "^16.0.1",
    "rollup": "^4.46.3"
  }
}
```

`rollup.config.js` (ES Module output):

```javascript
import { nodeResolve } from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import json from '@rollup/plugin-json';

export default {
  input: 'bin/executable.js',
  output: {
    file: 'package/lib/my-tool.mjs',
    format: 'es',
    inlineDynamicImports: true
  },
  plugins: [
    nodeResolve({ preferBuiltins: true }),
    commonjs(),
    json()
  ],
  external: ['fs', 'path', 'stream', 'util', 'events', 'buffer', 'string_decoder', 'crypto', 'os', 'tty', 'process']
};
```

### JS Package with Native Dependencies

Some dependencies have native binaries (e.g., SQLite, libsql) that Rollup cannot bundle. In this case:

1. Exclude those dependencies from Rollup's `external` array
2. Add a `package.json` inside `package/lib/` with only those excluded dependencies
3. Update the build script to run `npm install` in `package/lib/` after bundling

```
my-js-package/
├── .aux4
├── package.json       # "build": "rollup -c && cd package/lib && npm install"
├── rollup.config.js   # Excludes native deps
└── package/
    ├── .aux4
    └── lib/
        ├── my-tool.js     # Bundled output
        └── package.json   # Only the excluded native dependencies
```

`rollup.config.js` — exclude native deps:

```javascript
export default {
  input: 'main.js',
  output: {
    file: 'package/lib/my-tool.js',
    format: 'es'
  },
  plugins: [
    nodeResolve({ preferBuiltins: true }),
    commonjs(),
    json()
  ],
  external: [
    'fs', 'path', 'crypto', 'util', 'stream', 'os', 'process',
    'libsql',           // native dependency — cannot be bundled
    /^@libsql\/.*/      // platform-specific binaries
  ]
};
```

`package.json` (root) — build script runs `npm install` in `package/lib/`:

```json
{
  "scripts": {
    "build": "rollup -c && cd package/lib && npm install"
  }
}
```

`package/lib/package.json` — only the excluded native dependencies:

```json
{
  "type": "module",
  "dependencies": {
    "libsql": "^0.5.20"
  }
}
```

Root `.aux4` stays the same (`aux4 build` runs `npm run build`), which triggers Rollup and then installs the native deps in `package/lib/`.

### Referencing Files in Commands

In `package/.aux4`, commands reference files relative to `${packageDir}` (the `package/` directory):

```json
{
  "system": [
    ["test:node --version", "brew:node", "apt:nodejs", "linux:nodejs"]
  ],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "mytool",
          "execute": ["stdin:node ${packageDir}/lib/my-tool.mjs action values(a, b)"],
          "help": { "text": "Run my tool" }
        }
      ]
    }
  ]
}
```

### Key Differences: Go vs JavaScript

| Aspect | Go Package | JavaScript Package |
|--------|-----------|-------------------|
| Runtime | None (standalone binary) | Node.js required |
| Build | `GOOS=... go build` per platform | `rollup -c` (single bundle) |
| Output | `package/dist/{os}/{arch}/binary` | `package/lib/tool.mjs` |
| Execute | `${packageDir}/binary args` | `node ${packageDir}/lib/tool.mjs args` |
| Root .aux4 | Builder profiles (darwin/linux/windows/all) | Simple `npm run build` |
| System deps | Usually none | `["test:node --version", "brew:node", ...]` |
| Distribution | Platform-specific zips + master zip | Single zip (cross-platform) |

## Step 7: Set Up GitHub Actions (Optional)

Create `.github/workflows/publish.yml` to auto-publish to hub.aux4.io:

```yaml
name: Publish Package

on:
  push:
    branches:
      - main
    paths:
      - '**'
      - '!.github/**'
      - '!README.md'
      - '!LICENSE'
      - '!.gitignore'

  workflow_dispatch:
    inputs:
      level:
        description: 'Release level (patch, minor, major)'
        required: true
        default: 'patch'
        type: choice
        options:
          - patch
          - minor
          - major

concurrency:
  group: publish-package
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # Add build steps here if needed (see below)

      - name: Publish package
        uses: aux4/publish-package-action@v1
        with:
          level: ${{ inputs.level || 'patch' }}
          aux4_token: ${{ secrets.AUX4_ACCESS_TOKEN }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

**Required secrets:**
- `AUX4_ACCESS_TOKEN` - Authentication token for hub.aux4.io
- `GITHUB_TOKEN` - Provided automatically by GitHub

For **Go** packages, add a build step before publish:

```yaml
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'

      - name: Build
        run: |
          GOOS=darwin GOARCH=amd64 go build -o package/dist/darwin/amd64/my-binary .
          GOOS=darwin GOARCH=arm64 go build -o package/dist/darwin/arm64/my-binary .
          GOOS=linux GOARCH=amd64 go build -o package/dist/linux/amd64/my-binary .
          GOOS=linux GOARCH=arm64 go build -o package/dist/linux/arm64/my-binary .
          GOOS=linux GOARCH=386 go build -o package/dist/linux/386/my-binary .
          GOOS=windows GOARCH=amd64 go build -o package/dist/windows/amd64/my-binary.exe .
          GOOS=windows GOARCH=arm64 go build -o package/dist/windows/arm64/my-binary.exe .
          GOOS=windows GOARCH=386 go build -o package/dist/windows/386/my-binary.exe .
```

For **JavaScript** packages, add a build step before publish:

```yaml
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Build
        run: npm ci && npm run build
```

## Step 8: Build, Install Locally, and Publish

### Using pkger directly

```bash
# Build the package
aux4 aux4 pkger build ./out ./package

# Test locally from file
aux4 aux4 pkger install --fromFile ./out/scope_name_1.0.0.zip

# Login and publish
aux4 aux4 login
aux4 aux4 pkger publish ./out/scope_name_1.0.0.zip

# Others install from hub
aux4 aux4 pkger install scope/name
```

### Using package-releaser (recommended)

Install the releaser: `aux4 aux4 pkger install aux4/package-releaser`

```bash
# Install locally for testing (builds, bumps version to X-local, installs, restores version)
aux4 aux4 releaser install --dir ./package

# Release (increments version, builds, publishes, tags git, creates GitHub release)
aux4 aux4 releaser release --dir ./package --level patch

# Skip build step if already built
aux4 aux4 releaser install --dir ./package --noBuild true
aux4 aux4 releaser release --dir ./package --level minor --noBuild true
```

The `release` command does everything in one step:
1. `git pull -r` to get latest changes
2. Prompts for confirmation and increments version (patch/minor/major)
3. Runs `aux4 build` (unless `--noBuild true`)
4. Runs `aux4 aux4 pkger build` to create the zip
5. Runs `aux4 aux4 pkger publish` to upload to hub.aux4.io
6. Creates git tag and GitHub release with the zip attached
7. Cleans up the zip file

## Instructions

When creating a package, always:

1. Ask the user for scope, name, description, and license if not provided in the arguments.
2. Ask if it's a Go package (multi-platform binaries) or JavaScript package (Node.js bundle) or a simple shell-only package.
3. Create the `package/` directory with `.aux4`, `LICENSE`, and `README.md`.
4. Design a clean command hierarchy using profiles for subcommand groups.
5. Add tests in `package/test/` using `.test.md` format.
6. Add man pages in `package/man/` for each command.
7. Create the root `.aux4` with the appropriate build configuration:
   - Go: builder profiles for darwin/linux/windows with `go build` commands
   - JS: simple `npm run build` + `rollup.config.js` + `package.json`
   - Shell-only: empty build or no build needed
8. If the user wants CI/CD, create `.github/workflows/publish.yml` with the right build steps.
9. Use `${packageDir}` to reference files within the package.
10. Use `value()`, `values()`, `param()`, `params()` for parameter formatting in execute commands.
11. For Go packages, commands reference `${packageDir}/binary-name`.
12. For JS packages, commands reference `node ${packageDir}/lib/bundle.mjs` and include Node.js in system dependencies.
13. When writing markdown files (man pages, `.test.md`, `README.md`), use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
