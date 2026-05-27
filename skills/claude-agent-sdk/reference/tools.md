# Tools Reference

> Built-in tools, custom SDK tools, MCP integration, and tool search.

## Built-in Tools

| Tool | Category | Description |
|---|---|---|
| `Read` | File | Read files with offset/limit |
| `Write` | File | Create or overwrite files |
| `Edit` | File | Find-and-replace edits (`old_string`/`new_string`) |
| `Bash` | Execution | Run shell commands (`description`, `timeout` params) |
| `Monitor` | Execution | Watch background script, react per line |
| `Glob` | Search | Find files by pattern (`**/*.ts`) |
| `Grep` | Search | Search file contents with regex |
| `WebSearch` | Web | Search the web (results summarized) |
| `WebFetch` | Web | Fetch and parse web pages |
| `Agent` (prev. `Task`) | Orchestration | Spawn subagents. **Must be in `allowedTools` for subagents** |
| `Skill` | Orchestration | Invoke filesystem skills. **Must be in `allowedTools`** |
| `AskUserQuestion` | Interaction | Ask user clarifying questions. **Triggers `canUseTool`** |
| `TodoWrite` | Tracking | Task tracking (auto for multi-step) |
| `ToolSearch` | Discovery | Dynamic tool discovery (enabled by default) |
| `NotebookEdit` | File | Edit Jupyter notebooks (`.ipynb`) |

### Tool Availability vs Permission

- **`tools: ["Read", "Grep"]`** → Only those built-ins in context. `tools: []` removes ALL built-ins
- **`allowedTools: ["Read"]`** → Pre-approves `Read`. Other tools still available but fall through to permission mode
- **`disallowedTools: ["Bash"]`** → Always denies `Bash`, even in `bypassPermissions`

## Permission Scoping

```typescript
allowedTools: [
  "Bash(npm *)",           // Only npm commands
  "Bash(git:*)",           // All git subcommands
  "Edit(src/**/*.ts)",     // Only TS in src/
  "mcp__github__list_*",   // Wildcard for MCP tools
  "mcp__filesystem__*"     // All from filesystem server
]
```

## Custom Tools (SDK MCP Servers)

Custom tools use in-process MCP servers. Tool name format: `mcp__<server_name>__<tool_name>`.

### TypeScript

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const getWeather = tool(
  "get_weather",
  "Get current weather for a city",
  { city: z.string().describe("City name") },
  async (args) => {
    const resp = await fetch(`https://api.weather.com/${args.city}`);
    return { content: [{ type: "text", text: `${resp.json().temp}°C` }] };
  },
  { annotations: { readOnlyHint: true } }  // Parallel-safe
);

const server = createSdkMcpServer({ name: "weather", tools: [getWeather] });
```

### Python

```python
from claude_agent_sdk import tool, create_sdk_mcp_server

@tool("get_weather", "Get current weather for a city", {"city": str})
async def get_weather(args):
    return {"content": [{"type": "text", "text": f"{args['city']}: 22°C"}]}

@tool("search", "Search with filters",
    {"type": "object", "properties": {
        "query": {"type": "string"},
        "limit": {"type": "integer", "minimum": 1, "maximum": 100}
    }, "required": ["query"]})
async def search(args): ...

server = create_sdk_mcp_server("weather", tools=[get_weather])
```

### Tool Return Format

```typescript
{
  content: [  // Array of blocks - mix text, image, resource
    { type: "text", text: "result" },
    { type: "image", data: "base64string", mimeType: "image/png" },
    { type: "resource", resource: { uri: "file:///x", text: "content" } }
  ],
  isError: true  // Optional: signals tool failure, agent loop continues
}
```

### Tool Annotations

| Field | Default | Effect |
|---|---|---|
| `readOnlyHint` | `false` | If `true`, tool runs in parallel with other read-only tools |
| `destructiveHint` | `true` | Informational |
| `idempotentHint` | `false` | Informational |
| `openWorldHint` | `true` | Informational |

### Error Handling

```typescript
// ✅ Return isError → agent loop continues, Claude sees error
return { content: [{ type: "text", text: "API error: 500" }], isError: true };

// ❌ Throw exception → agent loop STOPS, query fails
throw new Error("API failed");
```

```python
# ✅ Return is_error → agent loop continues
return {"content": [{"type": "text", "text": "API error: 500"}], "is_error": True}

# ❌ Throw exception → agent loop STOPS
raise Exception("API failed")
```

### Complete Custom Tool Example with Error Handling

```python
@tool("fetch_data", "Fetch data from an API", {"endpoint": str})
async def fetch_data(args):
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(args["endpoint"])
            if response.status_code != 200:
                return {
                    "content": [{"type": "text", "text": f"API error: {response.status_code}"}],
                    "is_error": True,
                }
            return {"content": [{"type": "text", "text": response.text}]}
    except Exception as e:
        return {
            "content": [{"type": "text", "text": f"Failed: {str(e)}"}],
            "is_error": True,
        }
```

## External MCP Servers

### Transport Types

| Transport | Config | Use Case |
|---|---|---|
| **stdio** | `{ command: "npx", args: ["-y", "pkg"], env: {...} }` | Local MCP servers |
| **http** | `{ type: "http", url: "https://...", headers: {...} }` | Remote non-streaming |
| **sse** | `{ type: "sse", url: "https://...", headers: {...} }` | Remote streaming |
| **SDK** | `createSdkMcpServer({...})` | In-process custom tools |

### `.mcp.json` Config

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```

Loaded when `settingSources` includes `"project"` (default).

### Auth Pattern

```typescript
mcpServers: {
  github: {
    command: "npx", args: ["-y", "@modelcontextprotocol/server-github"],
    env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN }
  },
  "remote-api": {
    type: "sse", url: "https://api.example.com/mcp/sse",
    headers: { Authorization: `Bearer ${token}` }
  }
}
```

```python
options = ClaudeAgentOptions(
    mcp_servers={
        "github": {
            "command": "npx",
            "args": ["-y", "@modelcontextprotocol/server-github"],
            "env": {"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
        },
        "remote-api": {
            "type": "sse",
            "url": "https://api.example.com/mcp/sse",
            "headers": {"Authorization": f"Bearer {token}"},
        },
    },
    allowed_tools=["mcp__github__list_issues"],
)
```

### Checking MCP Connection Status

```typescript
if (message.type === "system" && message.subtype === "init") {
  const failed = message.mcp_servers.filter(s => s.status !== "connected");
}
```

## Tool Search

Enabled by default. `ENABLE_TOOL_SEARCH` env var:

| Value | Behavior |
|---|---|
| (unset) / `true` | Always on — never preload tools |
| `auto` | Activate when tool defs > 10% of context |
| `auto:N` | Custom threshold (e.g. `auto:5` = 5%) |
| `false` | Disable — all tools preloaded |

Requires: Claude Sonnet 4+ or Opus 4+. Max 10,000 tools, returns 3-5 per search.
