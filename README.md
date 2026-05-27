# own-skills

SailorZ 的个人 OpenClaw skill 仓库。

## Skills

| Skill | Description |
|---|---|
| [claude-agent-sdk](skills/claude-agent-sdk/) | Build production AI agents with the Claude Agent SDK (Python & TypeScript) |
| [dev-conventions](skills/dev-conventions/) | 个人开发习惯：老逻辑不动分流原则、AI 交互表达规范、HTML 可视化输出原则 |

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # 主 skill 文件
    references/       # 补充参考文档
```

## Usage

复制 skill 目录到 OpenClaw workspace 即可使用：

```bash
cp -r skills/<skill-name> ~/.openclaw/workspace/skills/
```
