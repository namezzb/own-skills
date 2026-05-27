# OpenClaw Skills

A curated collection of OpenClaw agent skills.

## Skills

| Skill | Description |
|---|---|
| [claude-agent-sdk](skills/claude-agent-sdk/) | Build production AI agents with the Claude Agent SDK (Python & TypeScript) |

## Structure

Each skill lives in its own directory under `skills/`:

```
skills/
  <skill-name>/
    SKILL.md          # Main skill file (loaded by OpenClaw)
    reference/        # Supplementary reference docs
    README.md         # Human-readable docs
```

## Usage

Install a skill by copying the skill directory to your OpenClaw workspace:

```bash
cp -r skills/<skill-name> ~/.openclaw/workspace/skills/
```

## Contributing

PRs welcome! Each skill should include a `SKILL.md` with a clear `name` and `description` frontmatter.
