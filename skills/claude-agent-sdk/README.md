# Claude Agent SDK Skill

> **OpenCode / Claude Code Skill** — 100% coverage of the official [Claude Agent SDK documentation](https://code.claude.com/docs/en/agent-sdk/overview).

Build production AI agents with the Claude Agent SDK (Python & TypeScript). This skill covers every feature, configuration option, tool, pattern, and best practice across **29 official documentation pages**.

## Quick Start

```bash
# Install the skill
cp -r claude-agent-sdk ~/.claude/skills/

# Or clone directly
git clone https://github.com/namezzb/claude-agent-sdk-skill.git ~/.claude/skills/claude-agent-sdk
```

Then ask Claude:

```text
How do I set up a read-only analysis agent with the Claude Agent SDK?
How do I create custom tools with the SDK?
Show me how to configure hooks to block dangerous Bash commands
```

## Structure

```
claude-agent-sdk/
├── SKILL.md                          # Navigation hub (136 lines, under 500 guideline)
└── reference/                        # Detailed docs loaded on demand
    ├── configuration.md              # All Options / ClaudeAgentOptions fields
    ├── tools.md                       # Built-in tools, custom tools, MCP, tool search
    ├── hooks.md                       # 19 hook events, callback API, patterns
    ├── subagents.md                   # AgentDefinition, inheritance, resuming
    ├── sessions-streaming.md          # Sessions, streaming I/O, structured outputs, checkpointing
    ├── skills-plugins.md              # Skills, Plugins, Slash Commands, Todo Tracking
    ├── deployment.md                  # Hosting, observability, cost, system prompts, user input
    ├── api-reference.md               # TS/Python parity tables
    └── migration.md                   # Migration, troubleshooting, best practices
```

## Coverage

| # | Topic | File | TS | PY |
|---|-------|------|----|----|
| 1 | Overview & Quickstart | SKILL.md | ✅ | ✅ |
| 2 | Agent Loop & Context Window | SKILL.md + deployment.md | ✅ | ✅ |
| 3 | Configuration (All Fields) | configuration.md | ✅ | ✅ |
| 4 | Built-in Tools | tools.md | ✅ | ✅ |
| 5 | Custom Tools (SDK MCP) | tools.md | ✅ | ✅ |
| 6 | External MCP Integration | tools.md | ✅ | ✅ |
| 7 | Tool Search | tools.md | ✅ | ✅ |
| 8 | Hooks (19 Events) | hooks.md | ✅ | ✅ |
| 9 | Subagents | subagents.md | ✅ | ✅ |
| 10 | Sessions (Resume/Fork/Continue) | sessions-streaming.md | ✅ | ✅ |
| 11 | Streaming Output & Input | sessions-streaming.md | ✅ | ✅ |
| 12 | Structured Outputs (Zod/Pydantic) | sessions-streaming.md | ✅ | ✅ |
| 13 | File Checkpointing | sessions-streaming.md | ✅ | ✅ |
| 14 | Skills (Filesystem-based) | skills-plugins.md | ✅ | ✅ |
| 15 | Plugins | skills-plugins.md | ✅ | ✅ |
| 16 | Slash Commands | skills-plugins.md | ✅ | ✅ |
| 17 | Todo Tracking | skills-plugins.md | ✅ | ✅ |
| 18 | Hosting & Deployment | deployment.md | ✅ | ✅ |
| 19 | Secure Deployment | deployment.md | ✅ | ✅ |
| 20 | Observability (OpenTelemetry) | deployment.md | ✅ | ✅ |
| 21 | Cost Tracking | deployment.md | ✅ | ✅ |
| 22 | System Prompts & CLAUDE.md | deployment.md | ✅ | ✅ |
| 23 | User Input & Approvals | deployment.md | ✅ | ✅ |
| 24 | Permissions | configuration.md + SKILL.md | ✅ | ✅ |
| 25 | Migration Guide | migration.md | ✅ | ✅ |
| 26 | API Reference (TS/Python Parity) | api-reference.md | ✅ | ✅ |
| 27 | TS V2 Preview | api-reference.md | ✅ | — |

## Features

- **10 files**, all well under the 500-line SKILL.md guideline
- **Every example in both Python AND TypeScript**
- Follows [official Agent Skills conventions](https://code.claude.com/docs/en/skills):
  - SKILL.md as navigation hub with YAML frontmatter
  - Reference files loaded on demand via markdown links
  - Supporting directory structure following `reference/` pattern

## License

MIT — see [LICENSE](LICENSE)

## Publishing on ClawHub

To publish this skill on [ClawHub](https://clawhub.ai):

1. Push the repo to GitHub
2. Visit ClawHub and submit the repo URL
3. The skill follows the standard ClawHub/OpenCode skill format with YAML frontmatter and supporting files

## Related

- [Claude Agent SDK Official Docs](https://code.claude.com/docs/en/agent-sdk/overview)
- [Agent Skills Specification](https://agentskills.io)
- [Claude Code Skills Guide](https://code.claude.com/docs/en/skills)
