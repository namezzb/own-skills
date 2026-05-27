# API Reference

> Function/type parity between TypeScript and Python SDKs.

## Top-Level Functions

| Function | TypeScript | Python |
|---|---|---|
| Start agent | `query({ prompt, options })` → `Query` | `async query(*, prompt, options?, transport?)` → `AsyncIterator[Message]` |
| Pre-warm | `startup({ options?, initializeTimeoutMs? })` | — |
| Define tool | `tool(name, desc, schema, handler, extras?)` | `@tool(name, desc, schema, annotations?)` decorator |
| Create server | `createSdkMcpServer({ name, version?, tools? })` | `create_sdk_mcp_server(name, version?, tools?)` |
| List sessions | `listSessions({ dir?, limit?, includeWorktrees? })` | `list_sessions(directory?, limit?, include_worktrees?)` |
| Read messages | `getSessionMessages(sessionId, { dir?, limit?, offset? })` | `get_session_messages(session_id, directory?, limit?, offset?)` |
| Get info | `getSessionInfo(sessionId, { dir? })` | `get_session_info(session_id, directory?)` |
| Rename | `renameSession(sessionId, title, { dir? })` | `rename_session(session_id, title, directory?)` |
| Tag | `tagSession(sessionId, tag, { dir? })` | `tag_session(session_id, tag, directory?)` |

## Query Object Methods (TypeScript)

The `query()` return value is an `AsyncGenerator<SDKMessage>` with additional methods:

| Method | Description |
|---|---|
| `[Symbol.asyncIterator]()` | Iterate messages |
| `interrupt()` | Send interrupt signal |
| `setPermissionMode(mode)` | Change permission mode mid-session |
| `setModel(model?)` | Change model mid-session |
| `rewindFiles(messageUuid)` | Rewind file changes |
| `getMcpStatus()` | Get MCP server statuses |
| `reconnectMcpServer(name)` | Retry failed MCP connection |
| `toggleMcpServer(name, enabled)` | Enable/disable MCP server |
| `stopTask(taskId)` | Stop background task |

## ClaudeSDKClient Methods (Python)

```python
class ClaudeSDKClient:
    def __init__(self, options=None, transport=None)
    async def connect(self, prompt=None)
    async def query(self, prompt, session_id="default")
    async def receive_messages(self) -> AsyncIterator[Message]
    async def receive_response(self) -> AsyncIterator[Message]
    async def interrupt(self)
    async def set_permission_mode(self, mode)
    async def set_model(self, model=None)
    async def rewind_files(self, user_message_id)
    async def get_mcp_status(self) -> McpStatusResponse
    async def reconnect_mcp_server(self, server_name)
    async def toggle_mcp_server(self, server_name, enabled)
    async def stop_task(self, task_id)
    async def get_server_info(self) -> dict | None
    async def disconnect(self)
```

Used as context manager: `async with ClaudeSDKClient(options) as client:`

## Message Types

| Type | TS Check | PY Check | Key Fields |
|---|---|---|---|
| System | `message.type === "system"` | `isinstance(message, SystemMessage)` | `subtype`, `data`, `session_id`, `mcp_servers` |
| Assistant | `message.type === "assistant"` | `isinstance(message, AssistantMessage)` | `message.content` (TS wraps), `usage`, `message_id` |
| User | `message.type === "user"` | `isinstance(message, UserMessage)` | Tool results, `uuid` |
| Stream event | `message.type === "stream_event"` | `isinstance(message, StreamEvent)` | `event` (raw API event) |
| Result | `message.type === "result"` | `isinstance(message, ResultMessage)` | `result`, `subtype`, `total_cost_usd`, `session_id`, `usage`, `num_turns` |
| Compact boundary | `SDKCompactBoundaryMessage` type | `SystemMessage` subtype `"compact_boundary"` | Compaction event |

## Tool Return Format (Both SDKs)

```typescript
interface CallToolResult {
  content: Array<
    | { type: "text"; text: string }
    | { type: "image"; data: string; mimeType: string }
    | { type: "resource"; resource: { uri: string; text?: string; blob?: string; mimeType?: string } }
  >;
  isError?: boolean;  // true = tool failure but loop continues
}
```

## Tool Annotations

```typescript
interface ToolAnnotations {
  title?: string;
  readOnlyHint?: boolean;     // false → can run in parallel with other read-only
  destructiveHint?: boolean;  // true (default)
  idempotentHint?: boolean;   // false (default)
  openWorldHint?: boolean;    // true (default)
}
```

## Session Info Type

```typescript
interface SDKSessionInfo {
  sessionId: string;
  summary: string;
  lastModified: number;
  fileSize?: number;
  customTitle?: string;
  firstPrompt?: string;
  gitBranch?: string;
  cwd?: string;
  tag?: string;
  createdAt?: number;
}
```

## TS V2 Preview (Unstable)

```typescript
const session = createSession();
await session.send("First message");
for await (const msg of session.stream()) {}
await session.send("Follow-up");  // Same session
```
