# Skills, Plugins, Slash Commands & Todo Tracking

## Skills (Filesystem-based)

Skills are `SKILL.md` files in `.claude/skills/<name>/SKILL.md`. **Skills are filesystem-only — there is no programmatic API for registering skills in the SDK.**

### Directory Structure

```text
my-skill/
├── SKILL.md           # Main instructions (required)
├── reference.md       # Detailed docs (loaded on demand)
├── examples/
│   └── sample.md      # Example output
└── scripts/
    └── helper.sh      # Executable scripts
```

### SKILL.md Format

```yaml
---
name: pdf-processor
description: Extract text and metadata from PDFs. Use when working with PDF documents.
---

# PDF Processing Skill
...
```

**Key:** The `description` determines when Claude autonomously invokes the skill. Keep `SKILL.md` under 500 lines; move detailed reference to separate files referenced from SKILL.md.

### Enabling Skills in SDK

```typescript
options: {
  cwd: "/path/to/project",  // Must contain .claude/skills/
  settingSources: ["user", "project"],  // Load skills from filesystem
  allowedTools: ["Skill", "Read", "Write", "Bash"]  // "Skill" is REQUIRED
}
```

```python
options = ClaudeAgentOptions(
    cwd="/path/to/project",
    setting_sources=["user", "project"],
    allowed_tools=["Skill", "Read", "Write", "Bash"],
)
```

### Skill Locations

| Location | Path | Loaded When |
|---|---|---|
| Project | `<cwd>/.claude/skills/` | `settingSources` includes `"project"` |
| User | `~/.claude/skills/` | `settingSources` includes `"user"` |
| Plugin | `<plugin>/skills/` | Plugin loaded via `plugins` option |

### Skill Frontmatter Fields

| Field | Description |
|---|---|
| `description` | When Claude should apply this skill (**recommended**) |
| `when_to_use` | Additional trigger phrases |
| `disable-model-invocation: true` | Only user can invoke (prevents auto-load) |
| `user-invocable: false` | Only Claude can invoke |
| `allowed-tools` | Tools auto-approved while skill is active |
| `context: fork` | Run skill in isolated subagent |
| `agent` | Which subagent type when `context: fork` |
| `model` | Model override |
| `effort` | Effort level override |
| `paths` | Glob patterns limiting activation scope |
| `argument-hint` | Autocomplete hint for arguments |
| `arguments` | Named positional arguments for `$name` substitution |

**Note:** The `allowed-tools` frontmatter only works with CLI, not SDK. Control tool access via main `allowedTools`.

### Testing & Discovering Skills

```text
"What Skills are available?"  // Claude lists available skills
"/my-skill some argument"     // Invoke directly
```

### Skill Lifecycle

Skill descriptions load at session start (if `settingSources` includes relevant source). Full content loads only when invoked. Invoked content stays in conversation and is carried forward during auto-compaction (first 5,000 tokens retained, 25,000 token combined budget for all skills).

### Skill String Substitutions

| Variable | Description |
|---|---|
| `$ARGUMENTS` | All arguments passed |
| `$ARGUMENTS[N]` / `$N` | Positional argument (0-based) |
| `$name` | Named argument from `arguments` frontmatter |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Skill's directory path |

---

## Plugins

Plugins are packages of Claude Code extensions loaded via the `plugins` option.

### Loading Plugins

```typescript
options: {
  plugins: [
    { type: "local", path: "./my-plugin" },
    { type: "local", path: "/absolute/path/to/plugin" }
  ]
}
```

```python
options = {
    "plugins": [
        {"type": "local", "path": "./my-plugin"},
        {"type": "local", "path": "/absolute/path/to/plugin"},
    ]
}
```

**Note:** `"local"` is the only supported `type` in the SDK. For marketplace plugins, download first, then load via local path.

### Plugin Directory Structure

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Required manifest
├── skills/                   # Agent Skills
│   └── my-skill/
│       └── SKILL.md
├── commands/                 # Legacy format (use skills/ instead)
│   └── custom-cmd.md
├── agents/                   # Custom agents
│   └── specialist.md
├── hooks/                    # Event handlers
│   └── hooks.json
└── .mcp.json                # MCP server definitions
```

### Namespace

Plugin skills use `plugin-name:skill-name` namespace when invoked: `/my-plugin:greet`

### Verification

```typescript
if (message.type === "system" && message.subtype === "init") {
  console.log("Plugins:", message.plugins);
  console.log("Commands:", message.slash_commands);
}
```

```python
if message.type == "system" and message.subtype == "init":
    print("Plugins:", message.data.get("plugins"))
    print("Commands:", message.data.get("slash_commands"))
```

### Plugin Path Resolution

- Relative paths: resolved from `cwd`
- Absolute paths: full filesystem path
- CLI-installed plugins: check `~/.claude/plugins/`

---

## Slash Commands

Slash commands work in the SDK when sent as prompt strings:

```typescript
query({ prompt: "/compact" })  // Trigger manual compaction
```

Commands from `.claude/commands/*.md` files are loaded when `settingSources` includes the relevant source. The `commands/` directory is a legacy format; prefer `skills/` for new development. If both a skill and a command share the same name, the skill takes precedence.

Slash commands available in the SDK include:
- Built-in: `/compact`, `/help`, `/clear`
- Skills (auto-discovered): `/skill-name`
- Plugin skills: `/plugin-name:skill-name`

---

## Todo Tracking

Claude automatically creates todos for multi-step tasks. Monitor via `TodoWrite` tool use blocks.

### Monitoring Todo Changes

```typescript
if (message.type === "assistant") {
  for (const block of message.message.content) {
    if (block.type === "tool_use" && block.name === "TodoWrite") {
      const todos = block.input.todos;
      todos.forEach((todo, i) => {
        const status = todo.status === "completed" ? "✅" : todo.status === "in_progress" ? "🔧" : "❌";
        console.log(`${i + 1}. ${status} ${todo.content}`);
      });
    }
  }
}
```

```python
if isinstance(message, AssistantMessage):
    for block in message.content:
        if isinstance(block, ToolUseBlock) and block.name == "TodoWrite":
            todos = block.input["todos"]
            for i, todo in enumerate(todos):
                status = "✅" if todo["status"] == "completed" else "🔧" if todo["status"] == "in_progress" else "❌"
                print(f"{i + 1}. {status} {todo['content']}")
```

### Todo Lifecycle

1. **Created** as `pending` when tasks identified
2. **Activated** to `in_progress` when work begins
3. **Completed** when finished successfully
4. **Auto-created** for complex multi-step tasks (3+ steps)

### Progress Tracking

```typescript
class TodoTracker {
  private todos: any[] = [];
  displayProgress() {
    const completed = this.todos.filter(t => t.status === "completed").length;
    const total = this.todos.length;
    console.log(`Progress: ${completed}/${total} completed`);
  }
}
```
