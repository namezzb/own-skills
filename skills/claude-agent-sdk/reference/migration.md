# Migration, Troubleshooting & Best Practices

## Migration from Claude Code SDK

### Package Renames

| Old | New |
|---|---|
| `@anthropic-ai/claude-code` | `@anthropic-ai/claude-agent-sdk` |
| `claude-code-sdk` | `claude-agent-sdk` |
| `ClaudeCodeOptions` | `ClaudeAgentOptions` |
| `claude_code_sdk` → `from claude_code_sdk import query` | `from claude_agent_sdk import query` |

### Breaking Changes (v0.1.0)

1. **System prompt defaults to minimal** — set `systemPrompt: { type: "preset", preset: "claude_code" }`
2. **Setting sources not loaded by default** (reverted in current releases) — pass `settingSources: ["user", "project", "local"]` or omit

---

## Troubleshooting

| Issue | Solution |
|---|---|
| `ANTHROPIC_API_KEY` not found | Export env var or set in `.env` file |
| Opus 4.7 `thinking.type.enabled` error | Upgrade SDK: TS v0.2.111+, PY latest |
| Skills not loading | `allowedTools` must include `"Skill"`; `settingSources` must include `"user"`/`"project"` |
| MCP tools not callable | Add to `allowedTools`: `mcp__<server>__<tool>` or `mcp__<server>__*` |
| Hook not firing | Case-sensitive: `PreToolUse` (not `preToolUse`). Check matcher pattern |
| `canUseTool` not invoked (PY) | Needs streaming input + dummy `PreToolUse` hook `{"continue_": True}` |
| Session not resuming | `cwd` must match original. File at `~/.claude/projects/<encoded-cwd>/<id>.jsonl` |
| Subagent not delegated | Include `"Agent"` in `allowedTools`. Write clear `description`. Mention by name |
| File checkpoint UUID missing | Set `extraArgs: { "replay-user-messages": null }` |
| Rewind fails after stream | Resume session with empty prompt, then `rewindFiles()` |
| `total_cost_usd` is `None` (PY) | Guard: `message.total_cost_usd or 0` |
| Parallel tool usage double-counted | Deduplicate by `message.message.id` (TS) / `message.message_id` (PY) |
| `ProcessTransport not ready` | Connection closed after stream. Resume session for rewind |
| Subagent permission prompts multiplying | Use `PreToolUse` hooks to auto-approve, or configure permission rules |
| Recursive subagent hook loops | Check `agent_id`/`agent_type` in hook input before spawning |

---

## Best Practices

1. **Set budget limits** for production — `maxBudgetUsd` + `maxTurns`
2. **Use subagents for isolation** — fresh context per task, summary only returned
3. **Prefer `allowedTools` over `bypassPermissions`** — wildcards grant exactly what's needed
4. **Deduplicate usage by message ID** — parallel tool calls share same ID
5. **Isolate with `settingSources: []`** in CI/CD to prevent config leakage
6. **`"auto"` for `ENABLE_TOOL_SEARCH`** when tool count is moderate
7. **Catch errors in hook handlers** — uncaught exceptions kill the agent loop
8. **`"acceptEdits"` + allow rules** for dev agents — not `bypassPermissions`
9. **Persistent instructions → CLAUDE.md** — survives automatic compaction better than initial prompt
10. **Start with `effort: "high"` or `"xhigh"`** for complex tasks, `"low"` for simple lookups
11. **Keep schemas focused** for structured outputs — deeply nested required fields harder to satisfy
12. **Use `previewFormat` in `toolConfig.askUserQuestion`** for rich HTML previews in approval dialogs
