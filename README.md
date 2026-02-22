# aux4 Claude Code Skills

Claude Code skills for creating and working with [aux4](https://aux4.io) packages.

## Installation

```bash
aux4 install
```

This copies all skills to `~/.claude/skills/` and the `CLAUDE.md` to `~/.claude/CLAUDE.md`, making them available globally across all your projects.

## Skills

| Skill | Description |
|-------|-------------|
| `/aux4` | Explains how aux4 works — CLI generator, `.aux4` files, profiles, commands, variables, executors |
| `/aux4-package` | Creates aux4 packages from scratch — Go and JS, build, publish, release |
| `/aux4-command` | Adds commands to existing `.aux4` files — profiles, variables, man pages, tests |
| `/aux4-test` | Creates and runs `.test.md` test files — execute, expect, error, file blocks, hooks |
| `/aux4-config` | Works with `config.yaml` files — get, set, merge, nested paths, `--config` flag |
| `/aux4-agent` | Creates AI agents using `aux4/ai-agent` — instructions, model config, tools, chat |

## Usage

In Claude Code, type the skill name followed by what you want to do:

```
/aux4-package create a CLI tool to manage Docker containers
```

```
/aux4-command add a deploy command that pushes to production
```

```
/aux4-test write tests for the deploy command
```

```
/aux4-agent create an agent that reviews pull requests
```

```
/aux4-config set up environment-based configuration for dev and prod
```

```
/aux4 explain how profile routing works
```

## How It Works

- **CLAUDE.md** loads automatically in every Claude Code session, giving Claude context about the aux4 ecosystem and available skills
- **Skills** are invoked on demand with `/skill-name` or automatically when Claude determines they are relevant to your request
- Each skill contains detailed instructions, examples, and conventions so Claude can generate correct aux4 code
