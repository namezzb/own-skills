# Sessions, Streaming & Structured Outputs

## Sessions

### Core Operations

| Operation | TypeScript | Python |
|---|---|---|
| New session | `query({ prompt })` (default) | `query(prompt=...)` |
| Resume by ID | `options: { resume: sessionId }` | `resume=session_id` |
| Continue recent | `options: { continue: true }` | `continue_conversation=True` |
| Fork | `resume: id, forkSession: true` | `resume=id, fork_session=True` |
| In-memory (TS) | `persistSession: false` | — |

### Capturing Session ID

```typescript
// TypeScript
if (message.type === "result") {
  sessionId = message.session_id;
}
// Also available earlier: message.type === "system" && message.subtype === "init"

// Python
if isinstance(message, ResultMessage):
    session_id = message.session_id
```

### ClaudeSDKClient (Python Multi-turn)

```python
async with ClaudeSDKClient(options=options) as client:
    await client.query("First question")
    async for msg in client.receive_response():
        print(msg)

    await client.query("Follow-up")  # Same session!
    async for msg in client.receive_response():
        print(msg)
```

### Session Management Functions

| Function | TS | Python |
|---|---|---|
| List sessions | `listSessions({ dir?, limit?, includeWorktrees? })` | `list_sessions(directory?, limit?, include_worktrees?)` |
| Read messages | `getSessionMessages(sessionId, { dir?, limit?, offset? })` | `get_session_messages(session_id, directory?, limit?, offset?)` |
| Get info | `getSessionInfo(sessionId, { dir? })` | `get_session_info(session_id, directory?)` |
| Rename | `renameSession(sessionId, title, { dir? })` | `rename_session(session_id, title, directory?)` |
| Tag | `tagSession(sessionId, tag \| null, { dir? })` | `tag_session(session_id, tag, directory?)` |

### Cross-host Resume

Session files: `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`
Copy the JSONL file and restore to same path on target host. Or use `SessionStore` adapter.

---

## Streaming Output

### Enable

```typescript
options: { includePartialMessages: true }
// Python: include_partial_messages=True
```

### The StreamEvent

```typescript
if (message.type === "stream_event") {
  const event = message.event;
  // event.type: "message_start" | "content_block_start" | "content_block_delta" | "content_block_stop" | "message_delta" | "message_stop"
}
```

### Text Streaming

```typescript
if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
  process.stdout.write(event.delta.text);
}
```

```python
if isinstance(message, StreamEvent):
    event = message.event
    if event.get("type") == "content_block_delta":
        delta = event.get("delta", {})
        if delta.get("type") == "text_delta":
            print(delta.get("text", ""), end="", flush=True)
```

### Tool Call Streaming

```typescript
let currentTool: string | null = null;
let toolInput = "";

if (event.type === "content_block_start" && event.content_block.type === "tool_use") {
  currentTool = event.content_block.name;
  toolInput = "";
} else if (event.type === "content_block_delta" && event.delta.type === "input_json_delta") {
  toolInput += event.delta.partial_json;
} else if (event.type === "content_block_stop" && currentTool) {
  console.log(`Tool ${currentTool}: ${toolInput}`);
  currentTool = null;
}
```

### Streaming UI Pattern

Track `inTool` flag to show tool status indicators while streaming text only when not in a tool.

### Limitations

- **Extended thinking:** `StreamEvent` not emitted when `maxThinkingTokens` set
- **Structured output:** JSON only in final `ResultMessage.structured_output`

---

## Streaming Input

Use `AsyncIterable` (TS) or async generator (PY) as prompt instead of string:

```typescript
async function* messages() {
  yield { type: "user", message: { role: "user", content: "First message" } };
  await delay(1000);
  yield { type: "user", message: { role: "user", content: [
    { type: "text", text: "With image" },
    { type: "image", source: { type: "base64", media_type: "image/png", data: "..." } }
  ]}};
}
query({ prompt: messages(), options });
```

**Streaming input supports:** image uploads, queued messages, interrupts, hooks. **Single message lacks:** images, dynamic queueing, real-time interrupts.

**Python streaming input:**
```python
async def message_generator():
    yield {"type": "user", "message": {"role": "user", "content": "First message"}}
    await asyncio.sleep(1)
    yield {"type": "user", "message": {"role": "user", "content": "Follow-up"}}

async with ClaudeSDKClient(options) as client:
    await client.query(message_generator())
    async for message in client.receive_response():
        print(message)
```

---

## Structured Outputs

### TypeScript (Zod)

```typescript
import { z } from "zod";

const Recipe = z.object({
  name: z.string(),
  prep_time_minutes: z.number(),
  ingredients: z.array(z.object({ item: z.string(), amount: z.number(), unit: z.string() })),
  steps: z.array(z.string())
});

options: {
  outputFormat: { type: "json_schema", schema: z.toJSONSchema(Recipe) }
}

// Handle result
if (message.type === "result" && message.subtype === "success" && message.structured_output) {
  const recipe = Recipe.safeParse(message.structured_output);
  if (recipe.success) console.log(recipe.data.name);
}
```

### Python (Pydantic)

```python
from pydantic import BaseModel

class Recipe(BaseModel):
    name: str
    prep_time_minutes: int
    ingredients: list[dict]
    steps: list[str]

options = ClaudeAgentOptions(
    output_format={"type": "json_schema", "schema": Recipe.model_json_schema()}
)

# Handle result
async for message in query(prompt="...", options=options):
    if isinstance(message, ResultMessage) and message.structured_output:
        recipe = Recipe.model_validate(message.structured_output)
        print(recipe.name)
```

### Error Handling

| Subtype | Meaning |
|---|---|
| `success` + `structured_output` | Valid output |
| `error_max_structured_output_retries` | Validation failed after retries |

---

## File Checkpointing

### Enable

```typescript
options: {
  enableFileCheckpointing: true,
  extraArgs: { "replay-user-messages": null }  // REQUIRED
}
```

### Capture & Rewind

```typescript
let checkpointId: string | undefined;
let sessionId: string | undefined;

// Capture: first user message UUID
for await (const message of response) {
  if (message.type === "user" && message.uuid && !checkpointId) {
    checkpointId = message.uuid;
  }
  if ("session_id" in message) sessionId = message.session_id;
}

// Rewind: resume + rewindFiles()
const rewindQ = query({ prompt: "", options: { ...opts, resume: sessionId } });
for await (const msg of rewindQ) {
  await rewindQ.rewindFiles(checkpointId);
  break;
}
```

**Tracks:** `Write`, `Edit`, `NotebookEdit`. **Not tracked:** Bash changes.
