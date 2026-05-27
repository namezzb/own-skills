# Hooks Reference

> Intercept agent behavior at key lifecycle points. 19 hook events, callback API, matchers, and patterns.

## All Hook Events

| Event | TS | PY | Fires When | Common Uses |
|---|---|---|---|---|
| `PreToolUse` | ✓ | ✓ | Before tool execution | Block dangerous ops, validate, modify input |
| `PostToolUse` | ✓ | ✓ | After tool returns | Audit, log, trigger side effects |
| `PostToolUseFailure` | ✓ | ✓ | Tool execution fails | Handle tool errors |
| `PostToolBatch` | ✓ | — | Full tool batch resolves | Inject conventions per batch |
| `UserPromptSubmit` | ✓ | ✓ | User prompt submitted | Inject context into prompts |
| `Stop` | ✓ | ✓ | Agent execution stops | Save state, validate result |
| `SubagentStart` | ✓ | ✓ | Subagent spawns | Track parallel tasks |
| `SubagentStop` | ✓ | ✓ | Subagent completes | Aggregate results |
| `PreCompact` | ✓ | ✓ | Before context compaction | Archive transcript |
| `PermissionRequest` | ✓ | ✓ | Permission dialog would show | Custom permission handling |
| `Notification` | ✓ | ✓ | Agent status messages | Forward to Slack/PagerDuty |
| `SessionStart` | — | ✓ | Session starts | Init logging |
| `SessionEnd` | — | ✓ | Session ends | Clean up |
| `Setup` | — | ✓ | Session setup/maintenance | Init tasks |
| `TeammateIdle` | — | ✓ | Teammate idle | Reassign work |
| `TaskCompleted` | — | ✓ | Background task completes | Aggregate |
| `ConfigChange` | — | ✓ | Config file changes | Reload settings |
| `WorktreeCreate` | — | ✓ | Git worktree created | Track workspaces |
| `WorktreeRemove` | — | ✓ | Git worktree removed | Clean up |

## Configuration

```typescript
// TypeScript
options: {
  hooks: {
    PreToolUse: [
      { matcher: "Write|Edit", hooks: [protectEnvFiles] },
      { matcher: "^mcp__", hooks: [mcpAudit] },
      { hooks: [globalLogger] }  // No matcher = all tools
    ]
  }
}
```

```python
# Python
ClaudeAgentOptions(
    hooks={
        "PreToolUse": [
            HookMatcher(matcher="Write|Edit", hooks=[protect_env_files]),
            HookMatcher(hooks=[global_logger]),
        ]
    }
)
```

## Callback Signature

**TypeScript:** `HookCallback = async (input: HookInput, toolUseID: string, context: { signal: AbortSignal }) => HookJSONOutput`

**Python:** `async def callback(input_data, tool_use_id, context) -> dict`

## Output Fields

| Field | For | Description |
|---|---|---|
| `hookSpecificOutput.permissionDecision` | PreToolUse | `"allow"`, `"deny"`, `"ask"`, `"defer"` (TS only) |
| `hookSpecificOutput.permissionDecisionReason` | PreToolUse | Why deny |
| `hookSpecificOutput.updatedInput` | PreToolUse | Modified tool input (requires `"allow"`) |
| `hookSpecificOutput.additionalContext` | PostToolUse | Append to tool result |
| `hookSpecificOutput.updatedToolOutput` | PostToolUse | Replace tool output entirely |
| `systemMessage` | Any | Inject message into conversation |
| `continue` / `continue_` | Any | Control agent continuation |
| `async` / `async_` + `asyncTimeout` | Any | Don't wait for hook (side effects only) |

**Priority:** `deny` > `defer` > `ask` > `allow`

## Matchers

Matchers are regex against **tool names only** (not file paths). Filter by path inside callback.

```typescript
// Built-in tool matchers: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, Agent, Skill, AskUserQuestion, TodoWrite
// MCP tool matchers: mcp__<server>__<action>

// Filter by file path inside callback:
if (input.hook_event_name !== "PreToolUse") return {};
const filePath = (input.tool_input as any)?.file_path;
if (!filePath?.endsWith(".md")) return {};  // Skip non-markdown
```

## Common Patterns

### Block Dangerous Commands

```typescript
const blockRm: HookCallback = async (input) => {
  const pre = input as PreToolUseHookInput;
  const cmd = (pre.tool_input as any)?.command;
  if (cmd?.includes("rm -rf")) {
    return { hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "Blocked destructive command" } };
  }
  return {};
};
```

### Modify Tool Input (Sandbox Redirect)

```typescript
const redirect: HookCallback = async (input) => {
  const pre = input as PreToolUseHookInput;
  const ti = pre.tool_input as Record<string, unknown>;
  return { hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: "allow", updatedInput: { ...ti, file_path: `/sandbox${ti.file_path}` } } };
};
```

### Auto-Approve Read-Only Tools

```typescript
const autoApprove: HookCallback = async (input) => {
  if (input.hook_event_name !== "PreToolUse") return {};
  if (["Read", "Glob", "Grep"].includes((input as PreToolUseHookInput).tool_name)) {
    return { hookSpecificOutput: { hookEventName: "PreToolUse", permissionDecision: "allow" } };
  }
  return {};
};
```

### Async Side-Effect (Logging)

```typescript
const logAsync: HookCallback = async (input) => {
  sendToLogService(input).catch(console.error);
  return { async: true, asyncTimeout: 30000 };
};
```

### Notification → Slack

```typescript
const notifySlack: HookCallback = async (input) => {
  const n = input as NotificationHookInput;
  await fetch("https://hooks.slack.com/...", {
    method: "POST", body: JSON.stringify({ text: `Agent: ${n.message}` })
  });
  return {};
};
```

### Chain Multiple Hooks (Ordered Execution)

```typescript
hooks: {
  PreToolUse: [
    { hooks: [rateLimiter] },       // 1st
    { hooks: [authorizationCheck] }, // 2nd
    { hooks: [inputSanitizer] },     // 3rd
    { hooks: [auditLogger] }         // Last
  ]
}
```

## Troubleshooting

| Issue | Fix |
|---|---|
| Hook not firing | Event name is case-sensitive (`PreToolUse`). Check matcher matches tool name |
| Matcher not filtering | Matchers match tool NAMES only. Filter by path inside callback |
| Modified input not applied | Must include `permissionDecision: "allow"` AND `hookEventName` in `hookSpecificOutput` |
| `canUseTool` not invoked (PY) | Requires streaming input + dummy `PreToolUse` hook returning `{"continue_": True}` |
| SessionStart/SessionEnd not in PY | Only available as shell command hooks via `settingSources: ["project"]` |
