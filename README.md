# Backend Code Review Skill 🛡️

基于后端工程师思维框架的 **九维度系统性代码审核技能**，适用于 Claude Code、Cursor、Kimi Code CLI、Gemini CLI 等 AI 编程助手。

## 审核维度

| 维度 | 关注点 |
|------|--------|
| 🎯 代码质量 | 命名、函数设计、魔法值、重复代码、日志 |
| 🏗️ 架构设计 | 分层边界、接口抽象、配置分离、职责单一 |
| 🗄️ 数据库与存储 | 索引、事务、N+1、数据增长、一致性 |
| ⚡ 并发与分布式 | 并发安全、幂等、消息队列、分布式认知 |
| 🛡️ 可靠性工程 | 限流熔断降级、重试、超时、资源释放 |
| 🚀 性能与可扩展性 | 缓存策略、批量操作、内存意识、长期运行 |
| 🔒 安全防御 | 输入校验、注入防护、越权、敏感数据 |
| 📊 可观测性 | 日志规范、指标监控、链路追踪、告警 |
| 🧪 工程实践 | 测试覆盖、API 文档、兼容性、变更安全 |

## 安装

### 方式一：npx skills add（推荐）

```bash
# 安装到所有检测到的 Agent
npx skills add <your-username>/backend-code-review -y

# 全局安装
npx skills add <your-username>/backend-code-review -y -g

# 安装到指定 Agent
npx skills add <your-username>/backend-code-review -a claude-code -a cursor
```

### 方式二：手动安装

将 `backend-code-review/` 文件夹复制到对应目录：

| Agent | 全局路径 | 项目路径 |
|-------|---------|---------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Codex | `~/.codex/skills/` | `.agents/skills/` |
| Kimi Code CLI | `~/.kimi/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | — |

## 使用示例

安装后，直接用自然语言触发：

```
> 帮我审核这个 PR 的代码质量

> 审查一下 payment_service.go，重点关注并发安全和数据库操作

> 对这个变更做一次全面的后端代码审核

> 检查这段代码有没有安全漏洞和性能问题
```

## 工作原理

采用 **渐进式披露** 设计，按需加载审核维度：

```
SKILL.md (审核流程 + 26项快速清单)     ← 触发时加载
  └── references/code-quality.md       ← 审查代码质量时加载
  └── references/architecture.md       ← 审查架构时加载
  └── references/database.md           ← 审查数据库时加载
  └── references/concurrency.md        ← 审查并发时加载
  └── references/reliability.md        ← 审查可靠性时加载
  └── references/performance.md        ← 审查性能时加载
  └── references/security.md           ← 审查安全时加载
  └── references/observability.md      ← 审查可观测性时加载
  └── references/engineering.md        ← 审查工程实践时加载
```

小范围变更（< 5 文件）自动使用 26 项快速清单；大范围变更逐维度深度审查。

问题定级采用三级分类：🔴 严重 / 🟠 重要 / 🟡 建议。

## 适用语言

Go、Java、Python、Node.js 等后端语言均可使用。代码示例以 Go 为主，但审查原则通用。

## 致谢

审核框架源自 [后端工程师思维框架](./backend_thinking.md)，涵盖从代码质量到工程实践的完整知识体系。

## License

MIT
