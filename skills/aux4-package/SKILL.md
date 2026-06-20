---
name: aux4-package
description: Creates aux4 packages with .aux4 metadata, LICENSE, README, tests, man pages, and GitHub Actions for publishing to hub.aux4.io.
user-invocable: true
disable-model-invocation: false
argument-hint: [package-description]
---

# Create an aux4 Package

Create a complete aux4 package based on: $ARGUMENTS

**Before starting**, load all required skills by invoking `/aux4-test`, `/aux4-command`, and `/aux4-docs` so you have the full test format, command conventions, and documentation format available. Do not wait until you need them — load them now.

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

The `main` profile is the entry point. Use `profile:` executor for subcommand groups. The profile name should match the command name that routes to it. For deeper nesting, use `<parent-profile>:<command>` as the profile name (e.g., `email:list`, `ai:agent:config`):

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

### Naming, Variables, and Executors

- **Command/profile names**: dashes for composed names (`send-email`, `run-all`)
- **Variable names**: camelCase (`firstName`, `configFile`)

See `/aux4` for the full reference on variable definitions (properties, dot notation, resolution order), executor prefixes (`profile:`, `set:`, `log:`, `nout:`, `json:`, `each:`, `confirm:`, `stdin:`, `alias:`, `#`), and parameter functions (`value()`, `values()`, `param()`, `params()`, `object()`).

## Step 2: Create the LICENSE File

Use a standard license text (MIT, Apache-2.0, etc.). You can also generate it with:

```bash
aux4 license use --name mit --owner "Owner" --year 2025 --project "project-name"
```

## Step 3: Create the Root README.md

Create a `README.md` at the **repository root** (not inside `package/`). This is the file GitHub displays on the repo page. It should only contain:

```markdown
# <scope>/<name>

<short description of the package>

- [aux4 hub](https://hub.aux4.io/r/public/packages/<scope>/<name>)
- [README.md](./package/README.md)
```

This points visitors to the full documentation inside `package/README.md`.

## Step 4: Create the Package README.md

Create `package/README.md` with the full package documentation: description, installation instructions, usage examples, and command reference. See `/aux4-docs` for the full README structure, section order, and formatting conventions.

Critical rules for the package README:

- Title must be `# scope/name` (e.g., `# aux4/todo`). Never `@scope/name` or `# name` alone.
- Must include `## Installation` section with `aux4 aux4 pkger install scope/name`.
- Focus on CLI usage only — never include programmatic/library usage (import statements, API examples).
- Never paste the `.aux4` manifest JSON into the README — document commands and options in prose.
- Never reference tests or test expectations in user-facing documentation.
- Avoid implementation details (internal file paths, algorithms, architecture) unless users need to interact with them.

## Step 5: Create Man Pages

Place markdown files in `package/man/`. Name them using double underscores for command hierarchy:

| Command | Man Page Filename |
|---------|-------------------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |
| `aux4 email list all` | `email_list__all.md` |

**Special characters in names**: If a profile or command name contains special characters (like `:`), replace them with a single `_` in the filename. For example, profile `email:list` with command `all` becomes `email_list__all.md`.

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

The description should go beyond the short help text — explain what the command does, when to use it, how flags interact, and any important behaviors. Include realistic examples with expected output. Create one man page per command. See `/aux4-docs` for the full documentation conventions.

## Step 6: Create Tests

Place `.test.md` files in `package/test/`. Name files using double underscores for command hierarchy (e.g., `mytool__run.test.md` for `aux4 mytool run`), matching the man page convention. If the profile or command name contains special characters (like `:`), replace them with a single `_` (e.g., `email_list__all.test.md` for profile `email:list` command `all`). Test files are published to hub.aux4.io as usage examples, so they also serve as documentation. See the `/aux4-test` skill for the full test format.

**Important**: Tests in `package/test/` should call `aux4 <command>` directly — do NOT create `file:.aux4` blocks to redefine the package commands. The test runner automatically discovers `package/.aux4` from the parent directory. Only use `file:.aux4` for standalone command tests that are not part of a package.

Quick example for a package test:

````markdown
# mytool run

## should output the filename

```execute
aux4 mytool run myfile.txt
```

```expect
Running myfile.txt
```
````

## Step 7: Set Up Build, CI/CD, and Publishing

Read `../references/build-configuration.md` for the complete build setup, including:
- Go cross-compilation (8 platform/arch targets, builder profiles)
- JavaScript Rollup bundling (single `.mjs` output)
- JS packages with native dependencies
- Referencing files in commands (`${packageDir}/binary` vs `node ${packageDir}/lib/bundle.mjs`)
- Key differences between Go and JavaScript packages
- `.gitignore` templates for Go, JS, and Shell-only packages
- GitHub Actions workflows using the unified `aux4/action@v1` (a `publish.yml` pipeline + a PR `ci.yml` gate)
- Symlink for local Go testing
- Build, install locally, and publish commands (pkger + package-releaser)

## Instructions

When creating a package, always:

1. Ask the user for scope, name, description, and license if not provided in the arguments.
2. Ask if it's a Go package (multi-platform binaries) or JavaScript package (Node.js bundle) or a simple shell-only package.
3. Create the `package/` directory with `.aux4`, `LICENSE`, and `README.md`.
4. Design a clean command hierarchy using profiles for subcommand groups.
5. Add man pages in `package/man/` for **every command** — do not skip any.
6. Add tests in `package/test/` using `.test.md` format. Tests should call `aux4 <command>` directly — do NOT use `file:.aux4` blocks. The test runner auto-discovers `package/.aux4` from the parent directory.
7. Create the root `.aux4` with the appropriate build configuration:
   - Go: builder profiles for darwin/linux/windows with `go build` commands (use `cd ${packageDir} &&` prefix)
   - JS: simple `npm run build` + `rollup.config.js` + `package.json`
   - Shell-only: empty build or no build needed
8. Create a `.gitignore` file excluding build artifacts (`package/dist/`, `node_modules/`, etc.), platform files (`.DS_Store`), and language-specific generated files (`go.sum`, symlinks).
9. Create GitHub Actions workflows in the **repo root** `.github/workflows/` (sibling to `package/`) using the unified **`aux4/action@v1`** action (`command: lint | test | publish`). Scaffold BOTH files:
   - **`publish.yml`** — multi-job pipeline `lint` → `test-linux` + `test-darwin` → `publish`, triggered on push to `main` and `workflow_dispatch` (with a `level` choice input: patch/minor/major). The `lint` job uses `command: lint` with `strict: 'true'`; the test jobs use `command: test`; the `publish` job uses `command: publish` with `level: ${{ inputs.level || 'patch' }}`, `aux4_token: ${{ secrets.AUX4_ACCESS_TOKEN }}`, `github_token: ${{ secrets.GITHUB_TOKEN }}` (needs `permissions: contents: write` and `concurrency: { group: publish-package, cancel-in-progress: false }`).
   - **`ci.yml`** — PR gate triggered on `pull_request` + `push: branches-ignore: [main]`, with `concurrency` cancel-in-progress. Jobs: `lint` (`command: lint`, `strict: 'true'`) and `test` (`command: test`). NO publish job.

   **Build-vs-no-build variant (decide per package):** a package is JS-with-build if its repo-root `package.json` has a `scripts.build`. For JS-with-build, add `Set up Node.js` (v22) + `npm install` + `npm run build` steps to the lint/publish jobs, and pass `build_command: npm run build` to the `test` steps. For shell-only / plain-`.mjs` packages with no build script, OMIT the Node/npm/build steps and `build_command` — just `actions/checkout@v6` then the `aux4/action@v1` step. Never reference the legacy component actions (`aux4/lint-package-action`, `aux4/test-package-action`, `aux4/publish-package-action`).
10. Use `${packageDir}` to reference files within the package.
11. **Always pass known parameters by index using `values(var1, var2, ...)`** when calling external binaries or scripts. The target program receives them as positional args and doesn't need to parse flags — aux4 handles all variable resolution including `--config` binding. Only use `value(*)` (all params as JSON) when the parameter list is dynamic.
12. For Go packages, commands reference `${packageDir}/binary-name`.
13. For JS packages, commands reference `node ${packageDir}/lib/bundle.mjs` and include Node.js in system dependencies.
14. When writing markdown files (man pages, `.test.md`, `README.md`), use 4 backticks (````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
15. **Always format JSON with indentation** in `.aux4`, `.test.md`, `.md`, and any other files. Never use single-line compact JSON. Each item in the `execute` array must be on its own line.
16. **Always specify a language tag** on fenced code blocks (`bash`, `json`, `yaml`, `text`, etc.). Never use bare ` ``` ` without a language.
17. For Go packages, before running tests: build the binary (`aux4 build`), then create a symlink from `package/<binary-name>` to `dist/<os>/<arch>/<binary-name>` for the current platform.
18. **Always build before running tests.** Run `npm run build` (JS) or `aux4 build` (Go) from the project root, then run `aux4 test run` from the `package/` or `package/test/` directory.
19. **Run `aux4 lint run` before pushing.** After building, run `aux4 lint run --strict true` from the project root to validate all `.aux4` files. Fix any errors before committing. If `aux4/lint` is not installed, install it with `aux4 aux4 pkger install aux4/lint`.
20. When any command is added or modified, always update the corresponding man page, test file, and README.md to stay in sync. See `/aux4-docs` for documentation conventions.
21. Load all needed skills at the start of package creation (e.g., `/aux4-package`, `/aux4-test`, `/aux4-command`, `/aux4-docs`) rather than loading them one by one as needed.
22. **Test local packages with `aux4 aux4 releaser install`** — run from the `package/` directory to build, install locally, and test. Never manually update `~/.aux4.config/packages/`.
23. **Never browse or modify `~/.aux4.config/packages/` directly** — use `aux4 aux4 pkger list` to inspect installed packages, `aux4 aux4 pkger uninstall <scope>/<name>` to remove packages. Never copy files to or delete files from that directory manually.
24. **Remove zip files from `package/` before building** — leftover `.zip` files get included in the package by mistake. Clean them before running `aux4 aux4 releaser install` or `aux4 aux4 pkger build`.
25. **If reinstalling fails** because the existing version is used by other packages and uninstall is blocked, bump the version and install without uninstalling first.
26. **Never commit `.aux4` files with `-local` version suffix** — the `-local` suffix is added temporarily by `aux4 aux4 releaser install` for local testing. Always verify the version is clean before committing.
27. **Check security and vulnerabilities.** For JS packages, run `npm audit` after installing dependencies. Review code for common vulnerabilities (command injection, path traversal, etc.).
28. **Check that dependencies are up to date.** For JS packages, run `npm outdated` to verify. For Go packages, run `go list -m -u all`.
