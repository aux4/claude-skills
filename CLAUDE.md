# aux4 - CLI Generator

aux4 is a CLI generator that creates high-level scripts using JSON-based `.aux4` configuration files.

## Workspace Structure

```
aux4/                          # Monorepo root
├── aux4/                      # Core CLI (Go)
├── pkger/                     # Package manager (Go)
├── packages/                  # aux4 packages
│   ├── config/                # config.yaml utility (Go)
│   ├── test/                  # .test.md test framework (Go)
│   ├── package-releaser/      # Build/publish/release tool
│   ├── encrypt/               # Encryption utility (Go)
│   ├── validator/             # Input validation (JS)
│   ├── db/                    # Database core (Go)
│   ├── db-mysql/              # MySQL adapter (JS)
│   ├── db-mssql/              # MSSQL adapter (JS)
│   ├── adapter/               # External adapters (JS)
│   ├── template/              # Template engine
│   ├── ai/                    # AI packages
│   │   ├── ai-agent/          # Core AI agent framework (JS/LangChain)
│   │   ├── copilot/           # AI copilot with pluggable skills
│   │   └── agent/             # Example agents (grammar, blog-writer)
│   └── ...                    # Other packages
├── docs.aux4.io/              # Documentation site
├── config-github-actions/     # CI/CD for config package
├── pkger-github-actions/      # CI/CD for pkger
├── claude-skills/             # Claude Code skills for aux4
└── ...
```

## Package Structure

Every aux4 package follows this layout:

```
package-name/
├── package/
│   ├── .aux4              # Package metadata + command definitions
│   ├── README.md          # Package documentation
│   ├── LICENSE            # License file
│   ├── man/               # Man pages (command__subcommand.md)
│   ├── test/              # Tests (.test.md files)
│   ├── lib/               # JS bundles (.mjs) — JS packages only
│   └── dist/              # Compiled binaries — Go packages only
│       ├── darwin/amd64/
│       ├── darwin/arm64/
│       ├── linux/amd64/
│       ├── linux/arm64/
│       ├── linux/386/
│       ├── windows/amd64/
│       ├── windows/arm64/
│       └── windows/386/
├── .aux4                  # Build/dev commands
├── go.mod / package.json  # Language-specific config
└── source files
```

## Key Conventions

- The `main` profile is the required entry point in every `.aux4` file
- Use `profile:name` executor to create subcommand groups
- Variables use `${varname}` syntax in execute arrays
- Man page filenames use `__` (double underscore) for hierarchy: `config__get.md`
- Test files use `.test.md` extension with markdown-based execute/expect blocks
- Go packages cross-compile to 8 platform/arch combinations
- JS packages bundle with Rollup to a single `.mjs` file
- Publishing uses `aux4/publish-package-action@v1` GitHub Action
- Package registry: hub.aux4.io

## Skills Available

| Skill | When to Use |
|-------|-------------|
| `/aux4` | Understanding how aux4 works |
| `/aux4-package` | Creating a new aux4 package from scratch |
| `/aux4-command` | Adding commands to an existing `.aux4` file |
| `/aux4-test` | Writing or running `.test.md` tests |
| `/aux4-config` | Working with config.yaml files and the config package |
| `/aux4-agent` | Creating AI agents using aux4/ai-agent |

## Common Workflows

### Creating a new package
1. Use `/aux4-package` to scaffold the package
2. Use `/aux4-command` to add commands
3. Use `/aux4-test` to write tests
4. Use `aux4 aux4 releaser release` to publish

### Adding features to an existing package
1. Read the existing `package/.aux4` file
2. Use `/aux4-command` to add the new command
3. Use `/aux4-test` to add tests
4. Run `aux4 test run` to verify

### Working with configuration
1. Use `/aux4-config` to understand config.yaml patterns
2. Add `aux4/config` as a dependency in `.aux4`
3. Use `--configFile` and `--config` flags in commands

### Creating an AI agent
1. Use `/aux4-package` to scaffold the package
2. Use `/aux4-agent` to set up the agent (instructions, config, commands)
3. Use `/aux4-test` to write tests
4. Use `aux4 aux4 releaser release` to publish
