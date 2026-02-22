---
name: aux4-agent
description: Creates AI agents using aux4/ai-agent - instruction-driven agents with tools, model configuration, document indexing, and image generation.
user-invocable: true
disable-model-invocation: false
argument-hint: [agent-description]
---

# Create an aux4 AI Agent

Create an AI agent package using aux4/ai-agent based on: $ARGUMENTS

## Overview

aux4 AI agents are aux4 packages that depend on `aux4/ai-agent`. They use markdown instruction files to teach the agent its role and behavior, and YAML config files to select the LLM model. The agent gets access to built-in tools for file operations, search, and aux4 CLI execution.

## Agent Package Structure

```
my-agent/
├── package/
│   ├── .aux4              # Package metadata, commands, dependency on aux4/ai-agent
│   ├── instructions.md    # System prompt teaching the agent its role
│   ├── config.yaml        # Model configuration
│   ├── README.md
│   ├── LICENSE
│   ├── man/
│   └── test/
├── .aux4                  # Build/dev commands (optional)
└── README.md
```

## Package .aux4 File

The package `.aux4` must declare `aux4/ai-agent` as a dependency and define commands that call `aux4 ai agent ask`.

### Simple Agent (single command)

```json
{
  "scope": "myscope",
  "name": "my-agent",
  "version": "0.1.0",
  "description": "Description of the agent",
  "dependencies": ["aux4/ai-agent"],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "my-agent",
          "execute": ["profile:my-agent"],
          "help": { "text": "My AI agent" }
        }
      ]
    },
    {
      "name": "my-agent",
      "commands": [
        {
          "name": "run",
          "execute": [
            "aux4 ai agent ask --configFile ${packageDir}/config.yaml --config my-agent --instructions ${packageDir}/instructions.md param(question)"
          ],
          "help": {
            "text": "Run the agent",
            "variables": [
              {
                "name": "question",
                "text": "The question or task",
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

### Agent with stdin context

When the agent needs to process input from stdin (like a text processor):

```json
{
  "name": "process",
  "execute": [
    "nout:if(configFile==) && echo ${packageDir}/config.yaml || echo ${configFile}",
    "set:agentConfigFile=$response",
    "stdin:aux4 ai agent ask --configFile ${agentConfigFile} --context --config my-agent --instructions ${packageDir}/instructions.md \"process the input\""
  ],
  "help": {
    "text": "Process input text",
    "variables": [
      {
        "name": "configFile",
        "text": "Path to config file",
        "default": ""
      }
    ]
  }
}
```

Usage: `cat input.txt | aux4 my-agent process`

### Agent with save-to-file option

```json
{
  "name": "generate",
  "execute": [
    "nout:if(configFile==) && echo ${packageDir}/config.yaml || echo ${configFile}",
    "set:agentConfigFile=$response",
    "stdin:aux4 ai agent ask --configFile ${agentConfigFile} --context --config my-agent --instructions ${packageDir}/instructions.md \"generate based on the input\"",
    "set:output=$response",
    "if(save!=) && TEXT=value(output) && echo \"$TEXT\" | bash -c 'cat > ${save}' || exit 0"
  ],
  "help": {
    "text": "Generate output from input",
    "variables": [
      {
        "name": "configFile",
        "text": "Path to config file",
        "default": ""
      },
      {
        "name": "save",
        "text": "File to save output",
        "default": ""
      }
    ]
  }
}
```

### Agent with chat mode

```json
{
  "name": "chat",
  "execute": [
    "aux4 ai agent chat --configFile ${packageDir}/config.yaml --config my-agent --history ${history} --instructions ${packageDir}/instructions.md"
  ],
  "help": {
    "text": "Start interactive chat",
    "variables": [
      {
        "name": "history",
        "text": "Path to save conversation history",
        "default": "history.json"
      }
    ]
  }
}
```

### Agent with multiple commands

```json
{
  "name": "my-agent",
  "commands": [
    {
      "name": "analyze",
      "execute": [
        "stdin:aux4 ai agent ask --configFile ${packageDir}/config.yaml --context --config my-agent --instructions ${packageDir}/instructions/analyze.md \"analyze the input\""
      ],
      "help": { "text": "Analyze input" }
    },
    {
      "name": "summarize",
      "execute": [
        "stdin:aux4 ai agent ask --configFile ${packageDir}/config.yaml --context --config my-agent --instructions ${packageDir}/instructions/summarize.md \"summarize the input\""
      ],
      "help": { "text": "Summarize input" }
    }
  ]
}
```

Each command can use a different instructions file for different behaviors.

## Model Configuration (config.yaml)

```yaml
config:
  my-agent:
    model:
      type: openai
      config:
        model: gpt-5-mini
```

### Supported Model Providers

| Type | Provider | Example Models |
|------|----------|----------------|
| `openai` | OpenAI | gpt-5-mini, gpt-4, gpt-4o |
| `anthropic` | Anthropic | claude-sonnet-4-5-20250929, claude-opus-4-6 |
| `bedrock` | AWS Bedrock | anthropic.claude-3-sonnet |
| `gemini` | Google | gemini-pro, gemini-1.5-pro |
| `groq` | Groq | llama-3.1-70b-versatile |
| `mistral` | Mistral | mistral-large-latest |
| `ollama` | Ollama (local) | llama3, codellama |
| `xai` | xAI | grok-2 |
| `cohere` | Cohere | command-r-plus |
| `databricks` | Databricks | databricks-meta-llama |
| `vertex` | Google Vertex | gemini-1.5-pro |

### Model config options

```yaml
config:
  my-agent:
    model:
      type: anthropic
      config:
        model: claude-sonnet-4-5-20250929
        temperature: 0.7
        # Provider-specific options accepted
```

API keys are read from environment variables by default (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).

## Instructions File (instructions.md)

The instructions file is a markdown system prompt that teaches the agent its role, rules, and behavior.

### Structure

```markdown
# Agent Role Title

You are an AI agent that [description of what the agent does].

## Process

Follow these steps:

### Step 1: [First step]

[Detailed instructions for step 1]

### Step 2: [Second step]

[Detailed instructions for step 2]

## Rules

**Always:**
- [Rule 1]
- [Rule 2]

**Never:**
- [Anti-pattern 1]
- [Anti-pattern 2]

## Output

[Describe the expected output format]
```

### Variable Substitution

Use `{variableName}` in instructions. Values are passed from the command's params:

```markdown
You are an expert at answering questions about {topic}.
```

When called with `--topic "machine learning"`, `{topic}` becomes `machine learning`.

### Best Practices for Instructions

1. Be specific about the agent's role and boundaries
2. Provide step-by-step processes
3. Include examples of expected input and output
4. Define explicit ALWAYS/NEVER rules
5. Specify the output format clearly
6. Tell the agent to return ONLY the result (no explanations, no preamble)
7. Keep token management in mind — tell the agent not to read every file

## Available Tools

The agent automatically gets access to these 10 built-in tools:

### File Operations

| Tool | Description |
|------|-------------|
| `readFile` | Read file contents (cwd and ~/.aux4.config/packages) |
| `writeFile` | Create or overwrite files (cwd only, tracks for cleanup) |
| `editFile` | Partial file updates with find/replace (`old_string` → `new_string`, optional `replace_all`) |
| `listFiles` | List directory contents (optional `recursive`, `exclude` prefixes) |
| `createDirectory` | Create directories (auto-creates parents, tracks for cleanup) |
| `removeFiles` | Delete only agent-created files (safety: cannot delete pre-existing files) |
| `saveImage` | Save base64 image data to disk |

### Search & Discovery

| Tool | Description |
|------|-------------|
| `searchFiles` | Search file contents by pattern (`include`/`exclude` extensions, `maxResults` default 50) |
| `searchContext` | Semantic search in vector store (for learned documents) |

### aux4 Integration

| Tool | Description |
|------|-------------|
| `executeAux4` | Run aux4 CLI commands (without the `aux4` prefix) |

### Teaching Tools in Instructions

The agent already knows about the tools, but you can guide how it should use them:

```markdown
## How to Work

1. Use `listFiles` to explore the project structure
2. Use `readFile` to understand existing code
3. Use `searchFiles` to find specific patterns
4. Use `writeFile` to create new files
5. Use `editFile` for small changes to existing files
6. Use `executeAux4` to run aux4 commands (without the `aux4` prefix)

### Important
- Always read a file before editing it
- Use `searchFiles` before `readFile` to find the right files
- Use `editFile` for small changes, `writeFile` for new files or full rewrites
```

## aux4 ai agent Commands

### ask — Single question/task

```bash
aux4 ai agent ask \
  --instructions instructions.md \
  --configFile config.yaml \
  --config my-agent \
  --question "do something"
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--instructions` | `instructions.md` | Path to instructions file |
| `--configFile` | *(none)* | Path to config YAML |
| `--config` | *(none)* | Config section name |
| `--question` | *(positional arg)* | The question or task |
| `--context` | `false` | Read additional context from stdin |
| `--history` | *(empty)* | Path to conversation history JSON |
| `--image` | *(empty)* | Image path(s), comma-separated |
| `--storage` | `.llm` | Vector store directory |
| `--stream` | `false` | Enable streaming output |
| `--role` | `user` | Message role |
| `--outputSchema` | `schema.json` | JSON schema for structured output |

### chat — Interactive conversation

```bash
aux4 ai agent chat \
  --instructions instructions.md \
  --configFile config.yaml \
  --config my-agent \
  --history history.json
```

Starts a loop: user types a message, agent responds, repeat. Type `exit` to quit.

### learn — Index documents for semantic search

```bash
aux4 ai agent learn --storage .llm document.pdf
aux4 ai agent learn --storage .llm codebase.js --type js
```

Indexes documents into a FAISS vector store. Supported types: `.csv`, `.json`, `.txt`, `.pdf`, `.docx`, `.pptx`, plus 40+ code file types.

### search — Query indexed documents

```bash
aux4 ai agent search --storage .llm --limit 5 "how does authentication work"
```

### image — Generate images

```bash
aux4 ai agent image --image output.png --size 1024x1024 "a futuristic city"
```

### history — View conversation history

```bash
aux4 ai agent history history.json
```

## Complete Example: Code Review Agent

### package/.aux4

```json
{
  "scope": "tools",
  "name": "code-reviewer",
  "version": "0.1.0",
  "description": "AI agent that reviews code for quality and best practices",
  "dependencies": ["aux4/ai-agent"],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "review",
          "execute": ["profile:review"],
          "help": { "text": "Code review commands" }
        }
      ]
    },
    {
      "name": "review",
      "commands": [
        {
          "name": "check",
          "execute": [
            "aux4 ai agent ask --configFile ${packageDir}/config.yaml --config reviewer --instructions ${packageDir}/instructions.md \"review the code in ${path}\""
          ],
          "help": {
            "text": "Review code in a file or directory",
            "variables": [
              {
                "name": "path",
                "text": "File or directory to review",
                "arg": true
              }
            ]
          }
        },
        {
          "name": "diff",
          "execute": [
            "stdin:aux4 ai agent ask --configFile ${packageDir}/config.yaml --context --config reviewer --instructions ${packageDir}/instructions/diff-review.md \"review this diff\""
          ],
          "help": {
            "text": "Review a git diff from stdin"
          }
        }
      ]
    }
  ]
}
```

### package/config.yaml

```yaml
config:
  reviewer:
    model:
      type: openai
      config:
        model: gpt-5-mini
```

### package/instructions.md

```markdown
# Code Review Agent

You are a code review agent. Your job is to review code for quality, best practices, and potential issues.

## Process

1. Use `listFiles` to understand the project structure
2. Use `readFile` to read the files to review
3. Analyze the code for issues
4. Provide a clear, actionable review

## Review Checklist

- Code correctness and logic errors
- Security vulnerabilities (injection, XSS, etc.)
- Performance issues
- Error handling
- Code style and readability
- DRY principle violations

## Output Format

For each issue found:
- **File**: path and line number
- **Severity**: critical / warning / suggestion
- **Issue**: what's wrong
- **Fix**: how to fix it

If no issues found, say "No issues found."
```

## Complete Example: Simple Question Agent

### package/.aux4

```json
{
  "scope": "myscope",
  "name": "my-assistant",
  "version": "0.1.0",
  "dependencies": ["aux4/ai-agent"],
  "profiles": [
    {
      "name": "main",
      "commands": [
        {
          "name": "ask",
          "execute": [
            "aux4 ai agent ask --instructions ${packageDir}/instructions.md param(question)"
          ],
          "help": {
            "text": "Ask a question",
            "variables": [
              {
                "name": "question",
                "text": "Your question",
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

### package/instructions.md

```markdown
You are a helpful assistant. Answer questions clearly and concisely.
Return only the answer, no preamble.
```

This is the simplest possible agent — no config file needed (uses default model), just instructions and a question.

## Instructions

When creating an aux4 AI agent:

1. Create the package structure following `/aux4-package` conventions.
2. Add `"aux4/ai-agent"` to the `dependencies` array in `package/.aux4`.
3. Write an `instructions.md` that clearly defines the agent's role, process, rules, and output format.
4. Create a `config.yaml` with the desired model provider and model name.
5. Define commands that call `aux4 ai agent ask` with `--instructions`, `--configFile`, and `--config` flags.
6. Use `--context` flag and `stdin:` executor prefix when the agent needs to receive input from stdin.
7. Use `param(variable)` for optional flags and `value(variable)` for positional values when building the command.
8. For agents with multiple behaviors, create separate instruction files (e.g., `instructions/analyze.md`, `instructions/summarize.md`).
9. Add a `chat` command if the agent should support interactive conversation.
10. Create man pages and tests following `/aux4-command` and `/aux4-test` conventions.
