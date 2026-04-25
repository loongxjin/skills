# Agent Skills 🛠️

[English](./README.md) | 中文

个人 Agent Skills 集合，适用于 Claude Code、Cursor、Kimi Code CLI、Gemini CLI 及其他 AI 编程助手。

## 技能列表

| 技能 | 说明 |
|------|------|
| [backend-code-review-zh](./backend-code-review-zh/) | 基于九维思维框架的后端代码审查，涵盖代码质量、架构设计、数据库、并发处理、可靠性、性能、安全、可观测性及工程实践 |
| [backend-code-review](./backend-code-review/) | 英文版本的后端代码审查技能——从6大领域29个维度系统深入审查后端代码质量 |

> 持续更新中，更多技能即将推出。

## 资源文档

| 文档 | 说明 |
|------|------|
| [后端工程师思维框架](./backend_thinking-zh.md) | 后端工程师九维思维自检清单——涵盖代码质量、架构设计、数据库与存储、并发与分布式、可靠性工程、性能与可扩展性、安全防御、可观测性与运维、工程实践与团队协作 |

## 安装

### 方式一：npx skills add（推荐）

```bash
# 安装全部技能
npx skills add loongxjin/skills -y

# 安装指定技能
npx skills add loongxjin/skills --skill backend-code-review-zh -y
npx skills add loongxjin/skills --skill backend-code-review -y

# 全局安装
npx skills add loongxjin/skills --skill backend-code-review-zh -y -g
npx skills add loongxjin/skills --skill backend-code-review -y -g

# 安装到指定 Agent
npx skills add loongxjin/skills --skill backend-code-review-zh -a claude-code -a cursor
npx skills add loongxjin/skills --skill backend-code-review -a claude-code -a cursor
```

### 方式二：手动安装

将目标技能文件夹复制到对应目录：

| Agent | 全局路径 | 项目路径 |
|-------|---------|---------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Codex | `~/.codex/skills/` | `.agents/skills/` |
| Kimi Code CLI | `~/.kimi/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | — |

## 许可证

MIT
