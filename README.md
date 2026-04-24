# Agent Skills 🛠️

English | [中文](./README.zh.md)

A personal collection of Agent Skills for Claude Code, Cursor, Kimi Code CLI, Gemini CLI, and other AI coding assistants.

## Skill List

| Skill | Description |
|-------|-------------|
| [backend-code-review](./backend-code-review/) | Backend code review based on a 9-dimension thinking framework, covering code quality, architecture, database, concurrency, reliability, performance, security, observability, and engineering practices |

> Continuously updated. More skills coming soon.

## Resources

| Document | Description |
|----------|-------------|
| [Backend Engineer Thinking Framework](./backend_thinking.en.md) | A comprehensive 9-dimension thinking checklist for backend engineers — covering code quality, architecture, database, concurrency, reliability, performance, security, observability, and engineering practices |

## Installation

### Option 1: npx skills add (Recommended)

```bash
# Install all skills
npx skills add loongxjin/skills -y

# Install a specific skill
npx skills add loongxjin/skills --skill backend-code-review -y

# Install globally
npx skills add loongxjin/skills --skill backend-code-review -y -g

# Install to specific agents
npx skills add loongxjin/skills --skill backend-code-review -a claude-code -a cursor
```

### Option 2: Manual Installation

Copy the target skill folder to the corresponding directory:

| Agent | Global Path | Project Path |
|-------|-------------|--------------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Codex | `~/.codex/skills/` | `.agents/skills/` |
| Kimi Code CLI | `~/.kimi/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | — |

## License

MIT
