---
name: aux4-docs
description: Updates aux4 package documentation — README.md, man pages, and keeping docs in sync with code changes.
user-invocable: true
disable-model-invocation: false
argument-hint: [what-to-document]
---

# Update aux4 Package Documentation

Update documentation for an aux4 package based on: $ARGUMENTS

## Overview

Every aux4 package has two documentation surfaces:

- **`package/README.md`** — Full package documentation for users. Published to hub.aux4.io.
- **`package/man/<command>.md`** — Per-command man pages, displayed via `aux4 aux4 man <command>`.

These must stay in sync with each other and with the code. When a feature is added, modified, or removed, update both.

## README.md Structure

A README should follow this section order. Not all sections are required — include what's relevant to the package:

````markdown
# Package Name

One-paragraph description of what the package does.

## Installation

```bash
aux4 aux4 pkger install scope/package-name
```

## Quick Start

Minimal working example to get started.

## Configuration

Full configuration example showing all options. This is the single source of truth for the config shape — every configurable option should appear here, even if documented in detail later.

```yaml
config:
  option1: value
  option2: value
  nested:
    key: value
```

## Feature Sections

One section per major feature. Each section should include:
- What it does
- Configuration example (if applicable)
- Behavior details
- Usage examples with expected output

## Environment Variables

List any environment variables the package reads.
````

### Key Rules

1. **Keep the main config example complete.** When adding a new config key, add it to both the main Configuration block AND the relevant feature section. Users scan the top-level example to see everything available.

2. **Feature sections are self-contained.** Each section should make sense on its own — include its own config snippet and examples. Don't force users to cross-reference.

3. **Document behavior, not implementation.** Explain what happens from the user's perspective. "Returns 429 when exceeded" is better than "uses a sliding window Map".

4. **Include caveats and warnings.** If a feature has security implications, limitations, or common pitfalls, add a **bold** note inline. Example:

    ```markdown
    **Note:** Query parameter API keys may be logged by reverse proxies. Prefer the header when possible.
    ```

5. **Show config defaults.** Use inline comments in YAML examples:

    ```yaml
    config:
      server:
        maxConcurrency: 50    # default: 50
        timeout: 30000        # default: 30000 (ms)
    ```

6. **Use consistent formatting.** Code values in backticks (`true`, `503`, `/api/*`). Config keys in backticks (`server.timeout`). HTTP methods and status codes in plain text or backticks depending on context.

## Man Page Format

Man pages live in `package/man/`. Each command gets one file.

### Filename Convention

Use double underscores (`__`) to separate command hierarchy levels:

| Command | Filename |
|---------|----------|
| `aux4 mytool` | `mytool.md` |
| `aux4 mytool run` | `mytool__run.md` |
| `aux4 mytool run all` | `mytool__run__all.md` |

If a profile or command name contains special characters (like `:`), replace them with a single `_` (e.g., profile `email:list` command `all` becomes `email_list__all.md`).

### Man Page Structure

Use `####` headings for the three required sections:

````markdown
#### Description

What the command does, when to use it, how flags interact, and important behaviors. Go beyond the short help text — this is the detailed reference.

List supported features as bullet points when there are several:

- **Feature A** — brief explanation
- **Feature B** — brief explanation

#### Usage

```bash
aux4 mytool run [--flag <value>] [--option <value>]
```

--flag     Description of the flag (default: value)
--option   Description of the option

#### Example

```bash
aux4 mytool run --flag value
```

```text
Expected output here
```
````

Include a configuration example if the command reads from a config file:

````markdown
Configuration file:

```yaml
config:
  key: value
```
````

### Man Page Rules

1. **Description depth matters.** The description should cover everything a user needs to know without reading the source code. Include: what it does, all options and their defaults, supported values, error behaviors, and interactions between options.

2. **Show realistic examples.** Use plausible values, not `foo`/`bar`. Show expected output.

3. **Document new features in existing man pages.** When adding a feature to an existing command, add it to the feature bullet list in Description and include a config example if it introduces new config keys.

## When to Update Docs

| Change | README.md | Man Page |
|--------|-----------|----------|
| New command | Add to command reference | Create new man page |
| New feature on existing command | Add feature section | Add to Description + Example |
| New config key | Add to main config block + feature section | Add to config example |
| Behavior change | Update affected sections | Update Description |
| New caveat or limitation | Add inline **Note** | Add to Description |
| Bug fix | Usually no change | Usually no change |

## Markdown Rules

- Use 4 backticks (`````) for outer fenced code blocks when they contain nested 3-backtick code blocks inside. Never escape backticks with backslash.
- Use `yaml` language tag for config examples, `bash` for commands, `text` for output, `json` for JSON.
- **Always format JSON with indentation** in documentation examples. Never use single-line compact JSON. This applies to config examples, request/response bodies, `.aux4` snippets, and event formats.

Good:
```json
{
  "statusCode": 200,
  "body": "hello"
}
```

```json
{
  "execute": [
    "set:greeting hello ${name}",
    "log:${greeting}"
  ]
}
```

Bad:
```json
{"statusCode":200,"body":"hello"}
```

```json
{"execute":["set:greeting hello ${name}","log:${greeting}"]}
```

## Instructions

When updating documentation:

1. Read the existing `package/README.md` and relevant `package/man/*.md` files first.
2. Identify what changed — new feature, new config, behavior change, or caveat.
3. Update the README.md: main config example (if new config key), feature section, and any affected sections.
4. Update the man page: Description bullet list, config example, Usage flags, and Example section.
5. Keep both in sync — if you add something to the README, make sure the man page reflects it too.
6. Do not remove existing documentation unless the feature was removed.
7. Do not add sections for features that don't exist yet.
