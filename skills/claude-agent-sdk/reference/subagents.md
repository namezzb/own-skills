# Subagents Reference

> Define, invoke, and manage subagents. Programmatic + filesystem definitions, inheritance rules, per-agent configuration.

## AgentDefinition Fields

| Field | Required | Type | Description |
|---|---|---|---|
| `description` | **Yes** | `string` | When Claude should use this agent |
| `prompt` | **Yes** | `string` | System prompt defining role and behavior |
| `tools` | No | `string[]` | Restricted tool set (default: all parent tools) |
| `disallowedTools` | No | `string[]` | Tools to remove from agent's tool set |
| `model` | No | `string` | `"sonnet"`, `"opus"`, `"haiku"`, `"inherit"`, or full ID |
| `skills` | No | `string[]` | Skill names available to this agent |
| `memory` | No | `"user" \| "project" \| "local"` | Memory source |
| `mcpServers` | No | `(string \| object)[]` | MCP servers by name or inline config |
| `maxTurns` | No | `number` | Max turns |
| `background` | No | `boolean` | Non-blocking background task |
| `effort` | No | `"low" \| "medium" \| "high" \| "xhigh" \| "max"` | Reasoning effort |
| `permissionMode` | No | `PermissionMode` | Permission mode |

## Programmatic Subagent Example

```typescript
agents: {
  "code-reviewer": {
    description: "Expert code reviewer. Use for quality, security reviews.",
    prompt: "You are a code review specialist...",
    tools: ["Read", "Grep", "Glob"],  // Read-only
    model: "sonnet"
  },
  "test-runner": {
    description: "Test execution specialist. Use for test analysis.",
    prompt: "You run and analyze test suites...",
    tools: ["Bash", "Read", "Grep"]
  }
}
// Required: "Agent" in allowedTools
allowedTools: ["Read", "Grep", "Glob", "Agent"]
```

```python
from claude_agent_sdk import AgentDefinition

options = ClaudeAgentOptions(
    allowed_tools=["Read", "Grep", "Glob", "Agent"],
    agents={
        "code-reviewer": AgentDefinition(
            description="Expert code reviewer. Use for quality, security reviews.",
            prompt="You are a code review specialist...",
            tools=["Read", "Grep", "Glob"],
            model="sonnet",
        ),
        "test-runner": AgentDefinition(
            description="Test execution specialist. Use for test analysis.",
            prompt="You run and analyze test suites...",
            tools=["Bash", "Read", "Grep"],
        ),
    },
)
```

## What Subagents Inherit

| Subagent receives | Subagent does NOT receive |
|---|---|
| Its own system prompt + Agent tool prompt | Parent's conversation history |
| Project CLAUDE.md (via settingSources) | Skills (unless in `AgentDefinition.skills`) |
| Tool definitions (subset or all) | Parent's system prompt |

**Key:** The only channel parent→subagent is the Agent tool's prompt string. Include all needed context (file paths, errors, decisions) in that prompt.

## Invocation Patterns

### Automatic (by description)
Write clear `description` → Claude matches tasks to subagents.

### Explicit (by name)
```text
"Use the code-reviewer agent to check auth.ts"
```

### Factory Pattern (Dynamic Config)
```typescript
function createSecurityAgent(level: "basic" | "strict"): AgentDefinition {
  return {
    description: "Security reviewer",
    prompt: `You are a ${level === "strict" ? "strict" : "balanced"} reviewer...`,
    model: level === "strict" ? "opus" : "sonnet",
    tools: ["Read", "Grep", "Glob"]
  };
}
```

## Detecting Subagent Invocation

Check tool_use blocks: `name === "Agent"` (or `"Task"` for older SDKs). Messages from subagent context have `parent_tool_use_id` set.

```typescript
if (block.type === "tool_use" && (block.name === "Agent" || block.name === "Task")) {
  console.log("Subagent:", block.input.subagent_type);
}
```

## Resuming Subagents

1. Capture `session_id` from `ResultMessage`
2. Parse `agentId` from content: regex `/agentId:\s*([a-f0-9-]+)/`
3. Resume: `resume: sessionId` with prompt referencing agent ID

## Tool Combination Patterns

| Use Case | Tools |
|---|---|
| Read-only analysis | `Read`, `Grep`, `Glob` |
| Test execution | `Bash`, `Read`, `Grep` |
| Code modification | `Read`, `Edit`, `Write`, `Grep`, `Glob` |
| Full access | Omit `tools` field (inherits all) |

## Important Rules

- **Subagents cannot spawn subagents** — don't include `Agent` in subagent's `tools`
- **Permission inheritance:** When parent uses `bypassPermissions`/`acceptEdits`/`auto`, all subagents inherit that mode. Cannot be overridden per subagent
- **Programmatic takes precedence** over filesystem-based agents with same name
- **Filesystem agents** loaded from `.claude/agents/` directories via `settingSources`
