---
name: pi-agent
description: Invoke the pi coding agent CLI for complex coding tasks on a specific model. Use when delegating work to pi, running pi programmatically, sending prompts to a specific LLM model via pi, or when users say "use pi", "run pi", "ask pi", "pi agent", "delegate to pi". Supports specifying provider/model, thinking level, tools, and system prompts.
---

# Pi Agent: Programmatic CLI Usage

## Always Use Print Mode

Use `pi -p` (print mode) for non-interactive, single-shot execution. Never use bare `pi` from another agent — it requires a TTY.

```bash
pi -p "your prompt here"
```

## Specifying a Model

```bash
# Provider + model flags
pi -p --provider anthropic --model claude-opus-4-6 "prompt"

# Provider/model shorthand (no --provider needed)
pi -p --model anthropic/claude-opus-4-6 "prompt"

# Fuzzy match (partial names work)
pi -p --model sonnet "prompt"

# Model with thinking level shorthand
pi -p --model sonnet:high "prompt"
```

## Key Flags

| Flag | Description |
|------|-------------|
| `-p, --print` | Non-interactive mode (required for programmatic use) |
| `--model <pattern>` | Model ID, fuzzy match, or `provider/id` or `id:thinking` |
| `--provider <name>` | Provider name (anthropic, google, openai, etc.) |
| `--thinking <level>` | off, minimal, low, medium, high, xhigh |
| `--system-prompt <text>` | Override system prompt |
| `--append-system-prompt <text>` | Append to default system prompt |
| `--tools <list>` | Comma-separated: read,bash,edit,write,grep,find,ls |
| `--no-tools` | Disable all built-in tools |
| `--no-session` | Ephemeral, don't persist session |
| `--mode json` | Output JSON event stream instead of text |
| `@file` | Include file contents in the prompt |

## Examples

### Simple prompt on a specific model
```bash
pi -p --model anthropic/claude-sonnet-4-5 "Explain this error: ..."
```

### Read-only analysis
```bash
pi -p --tools read,grep,find --model google/gemini-2.5-pro "Review the code in src/"
```

### High-thinking task
```bash
pi -p --model sonnet:xhigh "Design a cache invalidation strategy for..."
```

### Include files in prompt
```bash
pi -p --model claude-opus-4-6 @src/main.ts "Refactor this file"
```

### JSON event stream (for parsing output programmatically)
```bash
pi -p --mode json --model sonnet "List all exported functions in src/"
```

### Ephemeral (no session saved)
```bash
pi -p --no-session --model haiku "Quick: what does this regex do? /^(?=.*\d)/"
```

### Custom system prompt
```bash
pi -p --system-prompt "You are a security auditor." --model sonnet "Audit this code" @app.ts
```

## Available Providers

anthropic, google, openai, xai, groq, cerebras, mistral, openrouter, huggingface, amazon-bedrock, azure-openai-responses, github-copilot, minimax, kimi-coding, and others. Use `pi --list-models` to see all available models.

## RPC Mode (Advanced)

For long-running programmatic control, use RPC mode over stdin/stdout:

```bash
pi --mode rpc --model anthropic/claude-opus-4-6
```

Send JSON commands on stdin:
```json
{"type": "prompt", "message": "Hello"}
{"type": "set_model", "provider": "google", "modelId": "gemini-2.5-pro"}
{"type": "set_thinking_level", "level": "high"}
```
