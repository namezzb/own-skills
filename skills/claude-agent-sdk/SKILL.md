---
name: claude-agent-sdk
description: Build production AI agents with the Claude Agent SDK (Python & TypeScript). Covers installation, agent loop, configuration, tools, MCP, permissions, hooks, subagents, sessions, streaming, structured outputs, deployment, and troubleshooting. Use when building with @anthropic-ai/claude-agent-sdk or claude-agent-sdk, implementing tool-using agents, or configuring agent permissions/hooks/subagents.
---

# Claude Agent SDK

> Build production AI agents with Claude Code as a library. Full coverage of every SDK feature.

## Quick Install

```bash
npm install @anthropic-ai/claude-agent-sdk   # TypeScript
pip install claude-agent-sdk                  # Python
export ANTHROPIC_API_KEY=your-api-key
```

## Minimal Agent

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix the bug in auth.ts",
  options: { allowedTools: ["Read", "Edit", "Bash"] }
})) {
  if ("result" in message) console.log(message.result);
}
```

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"]),
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(main())
```

## How the Agent Loop Works

1. **Receive prompt** → SDK yields `SystemMessage` (subtype `"init"`)
2. **Evaluate** → Claude responds with text/tool calls → `AssistantMessage`
3. **Execute tools** → SDK runs tools, yields `UserMessage` with results
4. **Repeat** until text-only response → final `ResultMessage` with `subtype`, `total_cost_usd`, `session_id`

### Result Subtypes

| Subtype | Meaning |
|---|---|
| `success` | Task completed (has `result` field) |
| `error_max_turns` | Hit maxTurns |
| `error_max_budget_usd` | Hit maxBudgetUsd |
| `error_during_execution` | API failure or cancelled |
| `error_max_structured_output_retries` | Validation failed |

### Message Types

| Type | Key Check |
|---|---|
| `SystemMessage` (PY) / `type: "system"` (TS) | `subtype === "init"` for session metadata |
| `AssistantMessage` (PY) / `type: "assistant"` (TS) | `content` blocks, `usage`, `message_id` |
| `UserMessage` (PY) / `type: "user"` (TS) | Tool results, `uuid` (checkpoint) |
| `ResultMessage` (PY) / `type: "result"` (TS) | `result`, `subtype`, `total_cost_usd`, `session_id` |
| `StreamEvent` (PY) / `type: "stream_event"` (TS) | Raw API events (only when `includePartialMessages: true`) |

## Reference Files

Detailed reference material is organized into separate files loaded on demand:

| File | Content |
|---|---|
| [reference/configuration.md](reference/configuration.md) | All `Options` / `ClaudeAgentOptions` fields, setting sources, permission modes |
| [reference/tools.md](reference/tools.md) | Built-in tools, custom tools (SDK MCP servers), external MCP integration, tool search |
| [reference/hooks.md](reference/hooks.md) | All 19 hook events, callback signatures, matchers, output fields, patterns |
| [reference/subagents.md](reference/subagents.md) | AgentDefinition, inheritance, programmatic/filesystem definitions, resuming |
| [reference/sessions-streaming.md](reference/sessions-streaming.md) | Sessions (resume/fork/continue), streaming output/input, structured outputs (Zod/Pydantic), file checkpointing |
| [reference/skills-plugins.md](reference/skills-plugins.md) | Skills (filesystem-based), Plugins, Slash Commands, Todo Tracking |
| [reference/deployment.md](reference/deployment.md) | Hosting patterns, container security, observability (OpenTelemetry), cost tracking, system prompts, user input, context window |
| [reference/api-reference.md](reference/api-reference.md) | Complete TS/Python function parity table, `ClaudeSDKClient` methods, `Query` object methods |
| [reference/migration.md](reference/migration.md) | Migration from Claude Code SDK, troubleshooting, best practices |

## Common Patterns

### Read-only Analysis Agent
```typescript
options: { allowedTools: ["Read", "Glob", "Grep"], permissionMode: "dontAsk" }
```

### Autonomous Dev Agent
```typescript
options: { allowedTools: ["Read", "Edit", "Bash", "Glob", "Grep"], permissionMode: "acceptEdits", maxTurns: 30, effort: "high" }
```

### Production-Safe Agent
```typescript
options: { allowedTools: ["Read", "Glob", "Grep"], permissionMode: "dontAsk", settingSources: [], maxBudgetUsd: 0.50, maxTurns: 10 }
```

### Web-Connected Agent
```typescript
options: { mcpServers: { playwright: { command: "npx", args: ["@playwright/mcp@latest"] } }, allowedTools: ["mcp__playwright__*", "Read", "WebSearch"] }
```

## Package Names

| | TypeScript | Python |
|---|---|---|
| Package | `@anthropic-ai/claude-agent-sdk` | `claude-agent-sdk` |
| Import | `import { query } from "..."` | `from claude_agent_sdk import query` |
| Options type | `Options` (interface) | `ClaudeAgentOptions` (dataclass) |
| Session mgmt | `continue: true` | `ClaudeSDKClient` context manager |

## Permission Evaluation Order

1. **Hooks** (`PreToolUse`) → can allow/deny/modify
2. **Deny rules** (`disallowedTools`) → always block
3. **Permission mode** → `bypassPermissions` / `acceptEdits` / `dontAsk` / `default` / `plan`
4. **Allow rules** (`allowedTools`) → auto-approve listed
5. **`canUseTool` callback** → interactive approval

Priority: `deny` > `defer` > `ask` > `allow` (when multiple hooks conflict).

## Key Conventions

- **MCP tool naming:** `mcp__<server_name>__<tool_name>`
- **Permission scoping:** `"Bash(npm *)"`, `"Edit(src/**/*.ts)"`
- **Session files:** `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`
- **Cost fields are estimates** — not authoritative billing
- **Guard `total_cost_usd`** in Python — `None` on some error paths
