# Configuration Reference

> All `ClaudeAgentOptions` (Python) / `Options` (TypeScript) fields.

## Core Options

| Option | Type | Default | Description |
|---|---|---|---|
| `model` | `string` | CLI default | Model ID (e.g. `"claude-sonnet-4-6"`, `"claude-opus-4-7"`) |
| `fallbackModel` | `string` | `undefined` | Model to use if primary fails |
| `maxTurns` | `number` | `undefined` | Max tool-use round trips |
| `maxBudgetUsd` | `number` | `undefined` | Max cost estimate before stopping |
| `effort` | `"low" \| "medium" \| "high" \| "xhigh" \| "max"` | TS: `"high"`, PY: unset | Reasoning depth per turn. `"xhigh"` recommended for Opus 4.7 |
| `cwd` | `string` | `process.cwd()` | Working directory |
| `env` | `Record<string, string>` | `process.env` | Environment variables. **TS replaces**, **PY merges** inherited env |
| `extraArgs` | `Record<string, string \| null>` | `{}` | Additional CLI arguments |
| `abortController` (TS) | `AbortController` | `new AbortController()` | Cancel the query |
| `betas` | `SdkBeta[]` | `[]` | Enable beta features |
| `maxBufferSize` (PY) | `int` | `undefined` | Max stdout/stderr buffer size |

## Tool & Permission Options

| Option | Type | Default | Description |
|---|---|---|---|
| `tools` | `string[] \| ToolsPreset` | `undefined` | Built-in tools in context. `[]` removes all built-ins. MCP tools unaffected |
| `allowedTools` | `string[]` | `[]` | Auto-approve listed tools. Supports scoping: `"Bash(npm *)"`, `"Edit(src/**)"` |
| `disallowedTools` | `string[]` | `[]` | Always deny (even in `bypassPermissions`). Checked before allow rules |
| `permissionMode` | `PermissionMode` | `"default"` | Global permission mode |
| `canUseTool` (TS) / `can_use_tool` (PY) | callback | `undefined` | Interactive approval callback |
| `additionalDirectories` (TS) / `add_dirs` (PY) | `string[]` | `[]` | Additional accessible directories |
| `permissionPromptToolName` | `string` | `undefined` | MCP tool for permission prompts |
| `allowDangerouslySkipPermissions` (TS) | `boolean` | `false` | Required for `bypassPermissions` |

### Permission Modes

| Mode | TS | PY | Behavior |
|---|---|---|---|
| `"default"` | ✓ | ✓ | Unmatched tools → `canUseTool`; no callback = deny |
| `"acceptEdits"` | ✓ | ✓ | Auto-approves file edits + `mkdir`, `touch`, `rm`, `mv`, `cp`, `sed` |
| `"dontAsk"` | ✓ | ✓ | Anything not in `allowedTools` denied silently |
| `"bypassPermissions"` | ✓ | ✓ | All tools run. **Subagents inherit this mode** |
| `"plan"` | ✓ | ✓ | No tool execution; Claude produces plans |
| `"auto"` | ✓ | — | Model classifier approves/denies each call |

**Dynamic change:** `query.setPermissionMode(mode)` (TS) / `client.set_permission_mode(mode)` (PY)

## Session Options

| Option | TS | PY | Description |
|---|---|---|---|
| `resume` | ✓ | ✓ | Resume session by ID |
| `forkSession` (TS) / `fork_session` (PY) | ✓ | ✓ | Fork from `resume` session |
| `continue` (TS) / `continue_conversation` (PY) | ✓ | ✓ | Continue most recent session in directory |
| `persistSession` (TS) | — | `boolean` | `false` = in-memory only |
| `sessionStore` (TS) / `session_store` (PY) | ✓ | ✓ | Custom session storage backend |
| `enableFileCheckpointing` | ✓ | ✓ | Track file changes for rewinding. Requires `extraArgs: { "replay-user-messages": null }` |

## System Prompt & Context Options

| Option | Type | Default | Description |
|---|---|---|---|
| `systemPrompt` (TS) / `system_prompt` (PY) | `string \| { type: "preset", preset: "claude_code", append?, excludeDynamicSections? }` | `undefined` | Custom or preset prompt. Default is minimal; use `preset: "claude_code"` for full |
| `settingSources` (TS) / `setting_sources` (PY) | `SettingSource[]` | `["user", "project", "local"]` | Filesystem settings to load |
| `outputFormat` (TS) / `output_format` (PY) | `{ type: "json_schema", schema: JSONSchema }` | `undefined` | Structured output |
| `includePartialMessages` | `boolean` | `false` | Enable streaming `StreamEvent` messages |
| `thinking` | `ThinkingConfig` | `undefined` | Extended thinking config (replaces deprecated `maxThinkingTokens`) |

### Setting Sources

| Source | Location | Loads |
|---|---|---|
| `"project"` | `<cwd>/.claude/` + parent dirs | CLAUDE.md, rules, skills, hooks, settings.json |
| `"user"` | `~/.claude/` | User CLAUDE.md, rules, skills, settings |
| `"local"` | `<cwd>/` | CLAUDE.local.md, settings.local.json |

**Isolation:** Use `settingSources: []` in CI/CD to prevent filesystem config leakage. Note: managed policies and `~/.claude.json` are read regardless.

## MCP, Hooks, Agents, Plugins

| Option | Type | Description |
|---|---|---|
| `mcpServers` (TS) / `mcp_servers` (PY) | `Record<string, McpServerConfig>` | MCP servers (stdio/http/sse/SDK) |
| `hooks` | `Record<HookEvent, HookMatcher[]>` | Hook callbacks |
| `agents` | `Record<string, AgentDefinition>` | Programmatic subagents |
| `plugins` | `SdkPluginConfig[]` | Load plugins from local paths |
| `sandbox` | `SandboxSettings` | Sandbox config (TS only in V1) |
| `toolConfig` (TS) | `ToolConfig` | Tool-specific config (`askUserQuestion.previewFormat`) |

## Runtime & Executable Options (TS)

| Option | Type | Default | Description |
|---|---|---|---|
| `executable` | `"bun" \| "deno" \| "node"` | Auto-detect | JS runtime |
| `executableArgs` | `string[]` | `[]` | Runtime args |
| `debug` | `boolean` | `false` | Debug mode |
| `debugFile` | `string` | `undefined` | Debug log path |
| `pathToClaudeCodeExecutable` | `string` | Bundled native | Override CLI binary path |
| `agent` | `string` | `undefined` | Agent name for main thread. Must be defined in `agents` or settings |
| `allowDangerouslySkipPermissions` | `boolean` | `false` | Required for `bypassPermissions` |
| `cli_path` (PY) | `str \| Path` | `undefined` | Custom path to Claude CLI binary |
| `settings` (PY) | `str` | `undefined` | Custom settings string |
| `stderr` (PY) | `Callable[[str], None]` | `undefined` | stderr callback |
| `user` (PY) | `str` | `undefined` | User identifier |
| `transport` (PY) | `Transport` | `undefined` | Custom transport for CLI communication |

## TypeScript `startup()` — Pre-warming

```typescript
import { startup } from "@anthropic-ai/claude-agent-sdk";

// Pay spawn cost upfront (application boot)
const warm = await startup({ options: { maxTurns: 3 } });

// Later, when prompt is ready — immediate response
for await (const message of warm.query("What files are here?")) {
  console.log(message);
}
```

`startup()` pre-warms the CLI subprocess. Returns `WarmQuery` with a `.query(prompt)` method. Saves subprocess spawn + init time on first real query.
