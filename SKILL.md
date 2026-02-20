---
name: pi-agent
description: Invoke the pi coding agent CLI as a sub-agent. Use when delegating work to pi, running pi programmatically, sending prompts to a specific LLM model via pi, or when users say "use pi", "run pi", "ask pi", "pi agent", "delegate to pi". Includes orchestration best practices for context passing, tool scoping, and multi-agent workflows.
---

# Pi Agent: Sub-Agent Orchestration Guide

## Default Configuration

Unless the user specifies a different model, always use **gemini-3.1-pro-preview** with **high thinking**:

```bash
pi -p --no-session --model google/gemini-3.1-pro-preview --thinking high "your prompt here"
```

## Always Use Print Mode

Use `pi -p` (print mode) for non-interactive, single-shot execution. Never use bare `pi` from another agent — it requires a TTY. Always add `--no-session` to avoid polluting the user's session list.

## Execution Focus

Pi sub-agents must focus on **execution, not exploration**. Every invocation should produce a concrete deliverable — code written, files changed, a structured analysis, a direct answer.

**The orchestrator (you) handles all exploration.** Use your own tools and subagents to read files, grep for patterns, understand the codebase, and gather context. Then pass the distilled context to pi for execution. Never spawn pi to "explore" or "investigate" — that's your job. Pi receives context and acts on it.

---

## Sub-Agent Prompt Template

Every task string passed to pi should have 4 sections. This is the single most important factor for reliable sub-agent results.

```
Objective: <one sentence — what to produce>

Output Format:
<exact structure of the response, e.g. markdown headings, JSON schema, bullet list>

Context:
<only what this agent needs — not everything you know>

Boundaries:
<what NOT to do — prevents tool drift and scope creep>
```

### Why this matters

Vague prompts like "Review the code in src/" produce unpredictable results. The 4-section template constrains the agent's behavior:
- **Objective** prevents wandering — the agent knows when it's done
- **Output Format** enables compressed handoffs between agents
- **Context** avoids bloating the agent's context window with irrelevant info
- **Boundaries** prevents analysis agents from editing files, implementation agents from refactoring unrelated code, etc.

### Example: well-structured task

The orchestrator has already explored the codebase and found the relevant files. Now it passes that context to pi for execution:

```bash
pi -p --no-session \
  --model google/gemini-3.1-pro-preview --thinking high \
  "Objective: Add rate limiting middleware to the login endpoint (max 5 attempts per IP per 15 min).

Output Format:
## Completed
- what was done

## Files Changed
- path — description of change

Context:
- Login route: src/routes/auth.ts:42 (POST /api/login)
- Auth middleware: src/middleware/auth.ts (exports: requireAuth, validateToken)
- Existing rate limiter dep: express-rate-limit@7.1.0 in package.json
- Pattern: middleware is registered in src/app.ts:15-30

Boundaries:
Do not modify test files. Do not refactor existing auth code. Do not add new dependencies."
```

---

## Context Passing Patterns

Pi has 4 mechanisms for injecting context. Choose based on what the sub-agent needs.

| Mechanism | When to use | Example |
|-----------|-------------|---------|
| `@file` | Agent needs full file content (<200 lines) | `@src/config.ts "Explain this config"` |
| `--system-prompt <path>` | Custom agent identity — replaces pi's defaults | `--system-prompt /tmp/auditor.md` |
| `--append-system-prompt <path>` | Add constraints to pi's default identity | `--append-system-prompt /tmp/rules.md` |
| Piped stdin | Compressed summary from a prior agent | `echo "$scout_output" \| pi -p ...` |

### Rules

1. **Don't @file large files.** If a file is >200 lines, have the agent use `read`/`grep` tools to explore it instead of injecting the whole thing.
2. **Use `--append-system-prompt` for role specialization.** Write the role definition to a temp file, pass the path. This preserves pi's built-in capabilities while adding your constraints.
3. **Use `--system-prompt` only when you need a blank-slate agent.** Pair with `--no-skills --no-extensions` to strip all default context.
4. **Compress handoffs.** When chaining agents, don't pass the full prior output. Extract the relevant sections and pass a summary.

### System prompt via temp file (recommended pattern)

```bash
# Write role definition to temp file
cat > /tmp/pi-reviewer.md << 'EOF'
You are a code reviewer focused on security vulnerabilities.

## Output Format
For each finding:
### [SEVERITY] Title
- **File:** path:line
- **Issue:** description
- **Fix:** suggested remediation

## Rules
- Only report confirmed vulnerabilities, not style issues
- Use bash only for `git diff` and `git log`, never to modify files
EOF

# Pass as append-system-prompt
pi -p --no-session --append-system-prompt /tmp/pi-reviewer.md \
  --tools read,grep,find,ls,bash \
  --model google/gemini-3.1-pro-preview --thinking high \
  "Objective: Review the changes in the last 3 commits for security issues.

Boundaries: Do not modify files. Do not review test files."
```

---

## Tool Scoping

Match tools to the task type. Giving a review agent write/edit tools is asking for trouble. Exploration/recon is the orchestrator's job — don't delegate it to pi.

| Task type | Tools | Thinking | Rationale |
|-----------|-------|----------|-----------|
| Code review | `read,grep,find,ls,bash` | medium–high | Bash for `git diff/log` only |
| Implementation | `read,bash,edit,write` (default) | high | Full capabilities |
| Targeted analysis | `read,grep,find,ls` | medium–high | Read-only, orchestrator already identified what to analyze |
| Pure reasoning | `--no-tools` | high–xhigh | No file access needed |
| Build / test | `read,bash,ls` | minimal | Run commands, read output |

### Key flags for stripping context

| Flag | Effect |
|------|--------|
| `--no-tools` | Disable all built-in tools |
| `--no-skills` | Don't load any skills |
| `--no-extensions` | Don't load any extensions |

Use `--no-skills --no-extensions` when you're providing a complete `--system-prompt` — otherwise the agent loads skills/extensions that may conflict with your custom identity.

---

## Output Handling

### Text capture (simple)

```bash
result=$(pi -p --no-session --no-tools \
  --model haiku "Objective: Convert this list of endpoints into an OpenAPI snippet.
Output Format: YAML OpenAPI paths block.
Context:
- GET /api/users (returns User[])
- POST /api/users (body: CreateUserDTO, returns User)
- DELETE /api/users/:id (returns 204)
Boundaries: Only output YAML, no explanation.")
echo "$result"
```

### JSON event stream (structured)

Use `--mode json` for programmatic parsing. Pi emits newline-delimited JSON events. The key event types:

| Event type | Contains | Use for |
|------------|----------|---------|
| `message_end` | Full assistant message with `usage`, `stopReason`, `model` | Final answer extraction, cost tracking |
| `tool_result_end` | Tool output message | Monitoring tool usage |

```bash
# Extract the final assistant text from JSON mode
pi -p --no-session --mode json --model sonnet "Summarize this file" @README.md \
  | grep '"type":"message_end"' \
  | python3 -c "
import sys, json
for line in sys.stdin:
    evt = json.loads(line)
    msg = evt.get('message', {})
    if msg.get('role') == 'assistant':
        for block in msg.get('content', []):
            if block.get('type') == 'text':
                print(block['text'])
"
```

### File-based fan-out (parallel)

For parallel sub-agents, have each write to a named output file, then merge:

```bash
# Fan out: 3 pi agents implement the same refactor across different modules
for mod in api auth db; do
  pi -p --no-session \
    --model google/gemini-3.1-pro-preview --thinking high \
    "Objective: Replace all console.log calls with the structured logger in src/$mod/.
Output Format:
## Files Changed
- path:line — what was changed
Context: Logger import is \`import { logger } from '@/lib/logger'\`. Use logger.info/warn/error.
Boundaries: Only modify src/$mod/. Do not change test files." > "/tmp/refactor-$mod.txt" &
done
wait

# Orchestrator merges results
cat /tmp/refactor-*.txt
```

---

## Anti-Patterns

### 1. Broadcasting full context to all agents

```bash
# BAD: every agent gets the entire codebase summary
pi -p "Here is the full architecture doc (2000 lines)... Now find TODOs in src/utils/"
```

Each agent should receive only the context it needs. A TODO-finder doesn't need your architecture doc.

### 2. Vague task strings

```bash
# BAD: no structure, unpredictable output
pi -p "Review the code in src/"

# GOOD: 4-section template
pi -p "Objective: Identify functions with cyclomatic complexity >10 in src/.
Output Format: - path:line functionName (complexity: N)
Context: This is a TypeScript project using ESLint.
Boundaries: Do not modify files. Do not review tests."
```

### 3. Delegating exploration to pi

```bash
# BAD: using pi to explore the codebase
pi -p "Explore the auth module and tell me how it works"
pi -p --tools read,grep,find,ls "Scout the codebase for relevant files..."

# GOOD: explore yourself, then delegate execution
# (use your own tools: Read, Grep, Glob, or spawn subagents)
# Once you understand the codebase, pass distilled context to pi:
pi -p --no-session "Objective: Add rate limiting to POST /api/login.
Context:
- Route: src/routes/auth.ts:42
- Middleware pattern: src/middleware/auth.ts
- Dep available: express-rate-limit@7.1.0
Boundaries: Do not modify tests."
```

### 4. Append-only context chains

```bash
# BAD: each agent gets ALL prior outputs, context explodes
step1=$(pi -p "Analyze the codebase...")
step2=$(pi -p "Given this analysis: $step1 — now plan changes...")
step3=$(pi -p "Given analysis: $step1 and plan: $step2 — now implement...")
```

Compress between steps. Extract only the sections the next agent needs.

### 5. No tool scoping

```bash
# BAD: analysis agent has edit/write tools, might modify files
pi -p "Review this code for bugs" @src/app.ts

# GOOD: read-only tools
pi -p --tools read,grep,find,ls "Review this code for bugs" @src/app.ts
```

### 6. Session pollution

```bash
# BAD: creates a persistent session for a throwaway task
pi -p --model haiku "What does this regex do?"

# GOOD: ephemeral
pi -p --no-session --model haiku "What does this regex do?"
```

### 7. Passing stale/irrelevant prior outputs

```bash
# BAD: passing 5 prior agent outputs when only the last one matters
pi -p "Previous outputs: $step1 $step2 $step3 $step4 $step5 — now summarize"

# GOOD: pass only what's needed
pi -p "Objective: Summarize findings. Context: $step5 Boundaries: ..."
```

---

## Orchestration Examples

### Orchestrator recon, then pi executes

The orchestrator (you) does the exploration using your own tools/subagents, then passes distilled context to pi for implementation.

```bash
# Step 1: YOU (the orchestrator) explore the codebase
# Use your own Read, Grep, Glob tools or spawn Explore subagents to understand:
# - Login route: src/routes/auth.ts:42 (POST /api/login, calls AuthService.login)
# - Auth middleware: src/middleware/auth.ts (exports requireAuth, validateToken)
# - express-rate-limit@7.1.0 already in package.json
# - Middleware registered in src/app.ts:15-30

# Step 2: Pass distilled context to pi for execution
pi -p --no-session \
  --model google/gemini-3.1-pro-preview --thinking high \
  "Objective: Add rate limiting to the login endpoint (max 5 attempts per IP per 15 min).

Output Format:
## Completed
- what was done

## Files Changed
- path — description of change

## Notes
- anything the caller should know

Context:
- Login route: src/routes/auth.ts:42 (POST /api/login, calls AuthService.login)
- Auth middleware pattern: src/middleware/auth.ts (exports requireAuth, validateToken)
- express-rate-limit@7.1.0 already in package.json
- Middleware registered in src/app.ts:15-30

Boundaries: Only modify auth-related files. Do not refactor existing code. Do not change tests."
```

### Parallel fan-out execution with merge

The orchestrator has already identified the audit areas. Each pi instance executes a focused, scoped audit.

```bash
# Orchestrator has identified these areas to audit (via its own exploration)
areas=("SQL injection:src/db" "XSS:src/views" "Auth bypass:src/auth")

# Fan out parallel executions — each pi does targeted analysis, not exploration
for entry in "${areas[@]}"; do
  IFS=: read -r vuln_type dir <<< "$entry"
  pi -p --no-session --tools read,grep,find,ls \
    --model sonnet --thinking high \
    "Objective: Audit $dir/ for $vuln_type vulnerabilities.

Output Format:
### $vuln_type Findings
- **[HIGH|MED|LOW]** path:line — description

Boundaries: Do not modify files. Only report $vuln_type issues. Do not explore outside $dir/." \
    > "/tmp/audit-$(echo $vuln_type | tr ' ' '-').txt" &
done
wait

# Merge — orchestrator can do this itself, or use pi for structured formatting
cat /tmp/audit-*.txt | pi -p --no-session --no-tools \
  --model haiku --thinking minimal \
  "Objective: Merge these security audit results into a single prioritized report.
Output Format:
## Critical
## High
## Medium
## Low
Each item: - path:line — vulnerability type — description

Boundaries: Do not add findings not present in the input. Do not suggest fixes."
```

### System prompt for role specialization

```bash
# Write a specialized agent definition to a temp file
cat > /tmp/pi-architect.md << 'EOF'
You are a senior software architect. You analyze codebases and produce
architectural decision records (ADRs).

You must NOT make any changes to files. Only read, analyze, and produce ADRs.

## Output Format
# ADR-NNN: Title
## Status: Proposed
## Context
## Decision
## Consequences
EOF

pi -p --no-session \
  --system-prompt /tmp/pi-architect.md \
  --no-skills --no-extensions \
  --tools read,grep,find,ls \
  --model google/gemini-3.1-pro-preview --thinking high \
  "Objective: Produce an ADR for migrating from Express to Fastify.

Context: The project is in /Users/me/app. Entry point is src/index.ts.

Boundaries: Do not modify files. Focus only on the migration decision."
```

### Structured output with JSON mode

```bash
# Orchestrator already read the file and extracted the function signatures.
# Now use pi to generate documentation from that context.
pi -p --no-session --mode json --no-tools \
  --model sonnet --thinking medium \
  "Objective: Generate JSDoc comments for these functions.

Output Format: For each function, output the JSDoc block followed by the signature.

Context:
- createSession(userId: string, options?: SessionOptions): Promise<Session> — creates a new auth session
- validateToken(token: string): TokenPayload | null — validates JWT, returns null if invalid
- revokeSession(sessionId: string): Promise<void> — invalidates an active session

Boundaries: Only output JSDoc + signatures. No explanation." \
  2>/dev/null \
  | while IFS= read -r line; do
      type=$(echo "$line" | python3 -c "import sys,json; print(json.loads(sys.stdin.read()).get('type',''))" 2>/dev/null)
      if [ "$type" = "message_end" ]; then
        echo "$line" | python3 -c "
import sys, json
evt = json.loads(sys.stdin.read())
msg = evt.get('message', {})
if msg.get('role') == 'assistant':
    for block in msg.get('content', []):
        if block.get('type') == 'text':
            print(block['text'])
"
      fi
    done
```

---

## Key Flags

| Flag | Description |
|------|-------------|
| `-p, --print` | Non-interactive mode (required for programmatic use) |
| `--model <pattern>` | Model ID, fuzzy match, or `provider/id` or `id:thinking` |
| `--provider <name>` | Provider name (anthropic, google, openai, etc.) |
| `--thinking <level>` | off, minimal, low, medium, high, xhigh |
| `--system-prompt <path>` | Replace system prompt with contents of file at path |
| `--append-system-prompt <path>` | Append file contents to default system prompt |
| `--tools <list>` | Comma-separated: read,bash,edit,write,grep,find,ls |
| `--no-tools` | Disable all built-in tools |
| `--no-skills` | Don't load skills |
| `--no-extensions` | Don't load extensions |
| `--no-session` | Ephemeral, don't persist session |
| `--mode json` | Output JSON event stream (newline-delimited) |
| `--mode rpc` | Long-running RPC mode over stdin/stdout |
| `@file` | Include file contents in the prompt |

## Specifying a Model

```bash
# Provider/model shorthand (recommended)
pi -p --model anthropic/claude-opus-4-6 "prompt"

# Fuzzy match (partial names work)
pi -p --model sonnet "prompt"

# Model with thinking level shorthand
pi -p --model sonnet:high "prompt"
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
