# Deployment, Observability & System Prompts

## Hosting Patterns

### Container Security (Docker)

```bash
docker run \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --network none \
  --memory 2g --cpus 2 --pids-limit 100 \
  --user 1000:1000 \
  -v /path/to/code:/workspace:ro \
  -v /var/run/proxy.sock:/var/run/proxy.sock:ro \
  agent-image
```

### Deployment Patterns

| Pattern | Use Case |
|---|---|
| **Ephemeral** | New container per task, destroy on completion |
| **Long-running** | Persistent container, multiple agent processes |
| **Hybrid** | Ephemeral with history hydration |
| **Single container** | Multiple agents in one container |

### Isolation Technologies

| Tech | Isolation | Overhead |
|---|---|---|
| Sandbox runtime | Good | Very low |
| Containers (Docker) | Setup dependent | Low |
| gVisor | Excellent | Medium/High |
| VMs (Firecracker) | Excellent | High |

### Credential Management — Proxy Pattern

- Agent sends requests without credentials → proxy outside container injects them
- Claude API: `ANTHROPIC_BASE_URL=http://proxy:8080`
- Other services: `HTTP_PROXY` / `HTTPS_PROXY` + TLS-terminating proxy

### Filesystem Security

- Mount code read-only (`:ro`) for analysis-only agents
- Exclude credential files (`.env`, `.aws/`, `.git-credentials`)
- Use `tmpfs` for ephemeral writable spaces

---

## Observability (OpenTelemetry)

### Enable

```bash
CLAUDE_CODE_ENABLE_TELEMETRY=1
CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1  # Required for traces
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318
OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer token"
```

### Signals

| Signal | Content | Enable |
|---|---|---|
| Metrics | Token/cost counters, sessions, tool decisions | `OTEL_METRICS_EXPORTER` |
| Log Events | Prompts, API requests, tool results | `OTEL_LOGS_EXPORTER` |
| Traces (beta) | Spans per interaction | `OTEL_TRACES_EXPORTER` + beta flag |

### Trace Spans

- `claude_code.interaction` — one turn
- `claude_code.llm_request` — each API call
- `claude_code.tool` — tool invocation
- `claude_code.tool.blocked_on_user` — permission wait
- `claude_code.tool.execution` — execution
- `claude_code.hook` — hook execution

### Service Tagging

```bash
OTEL_SERVICE_NAME=support-triage-agent
OTEL_RESOURCE_ATTRIBUTES="service.version=1.4.0,deployment.environment=production"
```

### Sensitive Data (Opt-in Only)

Content is NOT exported by default. Enable:
- `OTEL_LOG_USER_PROMPTS=1`
- `OTEL_LOG_TOOL_DETAILS=1`
- `OTEL_LOG_TOOL_CONTENT=1`
- `OTEL_LOG_RAW_API_BODIES`

### Flush for Short-lived Calls

```bash
OTEL_METRIC_EXPORT_INTERVAL=1000
OTEL_LOGS_EXPORT_INTERVAL=1000
OTEL_TRACES_EXPORT_INTERVAL=1000
```

---

## Cost Tracking

### Get Total Cost

```typescript
if (message.type === "result") {
  console.log(`Cost: $${message.total_cost_usd}`);
  // Also: message.usage, message.modelUsage (per-model breakdown)
}
```

```python
if isinstance(message, ResultMessage):
    cost = message.total_cost_usd or 0
    print(f"Cost: ${cost:.4f}")
```

### Per-Step (Deduplicated)

```typescript
const seenIds = new Set<string>();
if (message.type === "assistant") {
  if (!seenIds.has(message.message.id)) {
    seenIds.add(message.message.id);
    totalInput += message.message.usage.input_tokens;
    totalOutput += message.message.usage.output_tokens;
  }
}
```

### Cache Tokens

Track `cache_read_input_tokens` and `cache_creation_input_tokens`. Set `ENABLE_PROMPT_CACHING_1H=1` for 1-hour TTL (default: 5 min).

### Important Notes

- `total_cost_usd` is a **client-side estimate** — not authoritative billing
- In Python, `total_cost_usd` is `Optional` — guard with `message.total_cost_usd or 0`
- Accumulate across `query()` calls yourself — each result is per-call only

---

## Context Window & Compaction

The context window accumulates across turns: system prompt, CLAUDE.md, tool definitions, conversation history, tool I/O.

### What Consumes Context

| Source | Impact |
|---|---|
| System prompt | Small fixed cost, always present (prompt-cached) |
| CLAUDE.md | Full content every request (prompt-cached after first) |
| Tool definitions | Each tool adds its schema. Use tool search for large sets |
| Conversation history | Grows with each turn |
| Skill descriptions | Short summaries; full content only when invoked |

### Automatic Compaction

When context nears limit, older history is summarized. The SDK emits:
- **TS:** `SDKCompactBoundaryMessage` type
- **PY:** `SystemMessage` with `subtype: "compact_boundary"`

**Important:** Early conversation instructions may be lost. Persistent rules belong in CLAUDE.md (re-injected on every request).

### Compaction Control

- **CLAUDE.md instructions:** Add a section telling the compactor what to preserve
- **`PreCompact` hook:** Run custom logic before compaction (archive transcript)
- **Manual:** Send `/compact` as prompt string

### Context Efficiency

- **Subagents:** Each starts fresh — only summary returns to parent
- **Tool scoping:** Use `tools` field on `AgentDefinition` to minimize tool defs
- **Tool search:** Load MCP tools on demand, not all upfront
- **Lower effort:** `effort: "low"` for simple tasks reduces token usage

---

## System Prompts

### CLAUDE.md Load Hierarchy

| Level | Path | When Loaded |
|---|---|---|
| Project (root) | `<cwd>/CLAUDE.md` or `<cwd>/.claude/CLAUDE.md` | `settingSources` includes `"project"` |
| Project (rules) | `<cwd>/.claude/rules/*.md` | `settingSources` includes `"project"` |
| Project (parent dirs) | `CLAUDE.md` in directories above `cwd` | `settingSources` includes `"project"`, loaded at session start |
| Project (child dirs) | `CLAUDE.md` in subdirectories | Loaded on demand when agent reads a file in that subtree |
| Local (gitignored) | `<cwd>/CLAUDE.local.md` | `settingSources` includes `"local"` |
| User | `~/.claude/CLAUDE.md` | `settingSources` includes `"user"` |
| User (rules) | `~/.claude/rules/*.md` | `settingSources` includes `"user"` |

All levels are additive. No hard precedence; if instructions conflict, write non-conflicting rules or state precedence explicitly.

### What settingSources Does NOT Control

Even with `settingSources: []`, these are still read:
- **Managed policy settings** (enterprise config)
- **`~/.claude.json`** (global config — relocate with `CLAUDE_CONFIG_DIR`)
- **Auto memory** at `~/.claude/projects/<project>/memory/` (disable with `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`)

### Output Styles

Output styles are saved configurations that modify Claude's system prompt.

**Creation:**
```bash
mkdir -p ~/.claude/output-styles
```

```markdown
<!-- ~/.claude/output-styles/code-reviewer.md -->
---
name: Code Reviewer
description: Thorough code review assistant
---

You are an expert code reviewer.
For every code submission:
1. Check for bugs and security issues
2. Evaluate performance
3. Suggest improvements
4. Rate code quality (1-10)
```

**Usage:** Loaded when `settingSources` includes `"user"` or `"project"`. Activated via `/output-style [name]` or settings. Each style is a markdown file with YAML frontmatter in `~/.claude/output-styles/` or `.claude/output-styles/`.

### System Prompt Methods Compared

| Feature | CLAUDE.md | Output Styles | `systemPrompt` with append | Custom `systemPrompt` |
|---|---|---|---|---|
| Persistence | Per-project file | Saved as files | Session only | Session only |
| Reusability | Per-project | Across projects | Code duplication | Code duplication |
| Default tools | Preserved | Preserved | Preserved | Lost (must re-add) |
| Built-in safety | Maintained | Maintained | Maintained | Must be added |
| Scope | Project-specific | User or project | Code session | Code session |
| Version control | With project | Yes | With code | With code |

---

## User Input & Approvals

### canUseTool Callback

```typescript
canUseTool: async (toolName, input) => {
  const approved = await askUser(`Allow ${toolName}?`);
  return approved
    ? { behavior: "allow", updatedInput: input }
    : { behavior: "deny", message: "User rejected" };
}
```

```python
async def can_use_tool(tool_name, input_data, context):
    approved = input(f"Allow {tool_name}? (y/n): ")
    if approved.lower() == "y":
        return PermissionResultAllow(updated_input=input_data)
    return PermissionResultDeny(message="User rejected")
```

### Response Types

| Action | TS | PY |
|---|---|---|
| **Allow** | `{ behavior: "allow", updatedInput }` | `PermissionResultAllow(updated_input=...)` |
| **Deny** | `{ behavior: "deny", message }` | `PermissionResultDeny(message=...)` |

### Handling AskUserQuestion

When `toolName === "AskUserQuestion"`:
1. Parse `input.questions[]` → each has `question`, `header`, `options[{label, description}]`, `multiSelect`
2. Display to user, collect answers
3. Return: `{ questions: input.questions, answers: { "Question text": "Selected Label" } }`
   - Multi-select: join labels with `", "`
   - Free-text: use typed text as value

```python
async def can_use_tool(tool_name, input_data, context):
    if tool_name == "AskUserQuestion":
        answers = {}
        for q in input_data["questions"]:
            print(f"\n{q['header']}: {q['question']}")
            for i, opt in enumerate(q["options"]):
                print(f"  {i+1}. {opt['label']} - {opt['description']}")
            resp = input("Your choice: ").strip()
            try:
                idx = int(resp) - 1
                answers[q["question"]] = q["options"][idx]["label"]
            except ValueError:
                answers[q["question"]] = resp  # Free-text
        return PermissionResultAllow(
            updated_input={"questions": input_data["questions"], "answers": answers}
        )
    return PermissionResultAllow(updated_input=input_data)
```

**Important (Python):** `can_use_tool` requires streaming input + dummy `PreToolUse` hook returning `{"continue_": True}`.
