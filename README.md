# Agent Skills 🛠️

English | [中文](./README-zh.md)

A personal collection of Agent Skills for Claude Code, Cursor, Kimi Code CLI, Gemini CLI, and other AI coding assistants.

## Skill List

| Skill | Description |
|-------|-------------|
| [business-driven-refactor](./business-driven-refactor/) | Business-first refactoring — understand business processes first, then realign code to faithfully express business intent. Untangle patched-up logic, identify ghost code, and restore clarity |
| [business-driven-refactor-zh](./business-driven-refactor-zh/) | Chinese version of business-driven-refactor
| [backend-code-review](./backend-code-review/) | Backend code review based on a 9-dimension thinking framework, covering code quality, architecture, database, concurrency, reliability, performance, security, observability, and engineering practices |
| [backend-code-review-zh](./backend-code-review-zh/) | Chinese version of backend code review skill — systematically reviews backend code quality across 6 domains and 29 dimensions |
| [atomic-commit](./atomic-commit/) | Enforce atomic commit workflow — break coding tasks into small steps, commit after each logical change with Conventional Commits |

> Continuously updated. More skills coming soon.

## Resources

| Document | Description |
|----------|-------------|
| [Backend Engineer Thinking Framework](./backend_thinking.md) | A comprehensive 9-dimension thinking checklist for backend engineers — covering code quality, architecture, database, concurrency, reliability, performance, security, observability, and engineering practices |

## Installation

### Option 1: npx skills add (Recommended)

```bash
# Install all skills
npx skills add loongxjin/skills -y

# Install a specific skill
npx skills add loongxjin/skills --skill business-driven-refactor -y
npx skills add loongxjin/skills --skill business-driven-refactor-zh -y
npx skills add loongxjin/skills --skill backend-code-review -y
npx skills add loongxjin/skills --skill backend-code-review-zh -y
npx skills add loongxjin/skills --skill atomic-commit -y

# Install globally
npx skills add loongxjin/skills --skill business-driven-refactor -y -g
npx skills add loongxjin/skills --skill business-driven-refactor-zh -y -g
npx skills add loongxjin/skills --skill backend-code-review -y -g
npx skills add loongxjin/skills --skill backend-code-review-zh -y -g
npx skills add loongxjin/skills --skill atomic-commit -y -g

# Install to specific agents
npx skills add loongxjin/skills --skill business-driven-refactor -a claude-code -a cursor
npx skills add loongxjin/skills --skill business-driven-refactor-zh -a claude-code -a cursor
npx skills add loongxjin/skills --skill backend-code-review -a claude-code -a cursor
npx skills add loongxjin/skills --skill backend-code-review-zh -a claude-code -a cursor
npx skills add loongxjin/skills --skill atomic-commit -a claude-code -a cursor
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
