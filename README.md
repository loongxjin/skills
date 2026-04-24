# Agent Skills 🛠️

个人维护的 Agent Skills 合集，适用于 Claude Code、Cursor、Kimi Code CLI、Gemini CLI 等 AI 编程助手。

## Skills 列表

| Skill | 说明 |
|-------|------|
| [backend-code-review](./backend-code-review/) | 基于九维度思维框架的后端代码审核，覆盖代码质量、架构、数据库、并发、可靠性、性能、安全、可观测性、工程实践 |

> 持续更新中，更多 skills 即将加入。

## 安装

### 方式一：npx skills add（推荐）

```bash
# 安装全部 skills
npx skills add <your-username>/skills -y

# 安装指定 skill
npx skills add <your-username>/skills --skill backend-code-review -y

# 全局安装
npx skills add <your-username>/skills --skill backend-code-review -y -g

# 安装到指定 Agent
npx skills add <your-username>/skills --skill backend-code-review -a claude-code -a cursor
```

### 方式二：手动安装

将目标 skill 文件夹复制到对应目录：

| Agent | 全局路径 | 项目路径 |
|-------|---------|---------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Codex | `~/.codex/skills/` | `.agents/skills/` |
| Kimi Code CLI | `~/.kimi/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | — |

## License

MIT
