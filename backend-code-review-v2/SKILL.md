---
name: backend-code-review-v2
description: |
  后端代码质量审查技能（v2）。基于一线后端工程实践，从8大领域35个维度深度审查后端代码，含常见反模式速查和修复示例。
  Use when reviewing, auditing, or evaluating backend code quality — including Go, Java, Python, Rust, C++, Node.js or any backend language.
  Triggers: 代码审查, code review, 后端代码质量, backend quality, review backend code, 检查代码质量, 审查代码, CR, code review checklist。
---

# 后端代码质量审查

## 核心原则

1. **证据优先**：每个结论必须有代码行号，禁止凭感觉
2. **深度优先于广度**：宁可深入分析3个问题，不要浅尝辄止
3. **可执行**：每个 ❌/⚠️ 必须给修复代码

## 判断标准

| 标记 | 含义 | 判定标准 |
|------|------|----------|
| ❌ | 严重问题 | 会导致 Bug、数据错误、安全漏洞、性能灾难、资源泄漏 |
| ⚠️ | 潜在隐患 | 增加维护成本、有潜在风险、不符合最佳实践、边界情况未处理 |
| ✅ | 良好实践 | 符合规范，值得保持 |
| N/A | 不适用 | 当前代码不涉及此维度，需说明原因 |

## 工作流程

### Step 1: 判断模式

- **快速模式**（< 5 个文件 或 用户说"快速看看"）
  - 加载 [checklist.md](references/checklist.md)，按 35 个维度逐项搜索验证
  - 输出：检查清单汇总表格（❌/⚠️/✅/N/A）
- **深度模式**（大量代码 或 用户要求仔细 review）
  - 用 AskUserQuestion 让选 1-2 个重点方向
  - 加载对应 reference（含搜索模式 + 检查清单 + 深挖方法）+ [anti-patterns.md](references/anti-patterns.md)
  - 输出：深度发现（含反模式匹配）+ 检查清单汇总

### Step 2: 扫描范围

Glob/Grep 扫描，列出：语言框架、关键文件、模块边界。

### Step 3: 深入审查

对每个维度循环执行：
1. **搜索** → 用对应 reference 中的搜索模式 Grep 搜索代码
2. **定位** → ReadFile 看上下文
3. **匹配反模式** → 对照 [anti-patterns.md](references/anti-patterns.md)，看到对应 pattern 直接定位问题
4. **判断** → 按上方标准标记 ❌/⚠️/✅/N/A，参考对应 reference 的检查清单逐项验证
5. **建议** → ❌ 和 ⚠️ 必须给修复代码，优先用 [anti-patterns.md](references/anti-patterns.md) 中的修复示例
6. **深挖** → 对关键问题参考对应 reference 的深挖方法做更深入分析

**正确示范 vs 错误示范：**

> ❌ 错误："并发安全：未发现明显问题"
> ✅ 正确："并发安全：`service.go:142` 的 `UpdateStock` 先查询库存再判断再扣减，存在 TOCTOU 反模式（[anti-patterns.md #17](references/anti-patterns.md)）。应改为原子操作：`UPDATE stock SET count = count - ? WHERE id = ? AND count >= ?`"

### Step 4: 输出报告

```markdown
## 审查结果

### 总评：[🟢 通过 / 🟡 建议改进 / 🔴 必须修复]

### 审查范围
- 模式：快速 / 深度
- 方向：代码质量、数据存储
- 文件：xxx.go, yyy.go (共 N 个)
- 模块：订单服务

### 深度发现
#### ❌ [CRITICAL] 并发竞态 — service.go:142 UpdateStock
**反模式**：TOCTOU（先查后改，非原子操作）
**影响**：高并发下可能超卖
**修复**：
​```go
affected, err := s.repo.DeductStockIfEnough(ctx, sku, qty)
​```

#### ⚠️ [WARN] 魔法数字 — handler.go:58
...

#### ✅ [GOOD] 事务范围 — order.go:200
事务仅含 INSERT，无 IO

### 检查清单汇总
| # | 维度 | 状态 | 位置 | 备注 |
|---|------|------|------|------|
| 1 | 命名规范 | ✅ | — | |
| 17 | 并发安全 | ❌ | service.go:142 | TOCTOU |
| 23 | 日志规范 | ⚠️ | logger.go:30 | 缺 trace_id |
```

## 领域索引

| 领域 | 维度 | Reference |
|------|------|-----------|
| 代码质量 | #1-#8 命名、函数、参数、Guard Clause、魔法值、DRY、接口抽象、配置分离 | [code-quality.md](references/code-quality.md) |
| 架构与API | #9-#10 分层、API 兼容 | [architecture.md](references/architecture.md) |
| 数据与存储 | #11-#16 索引、事务、批量、数据增长、Schema 变更、缓存 | [data-storage.md](references/data-storage.md) |
| 并发与分布式 | #17-#22 并发、幂等、限流熔断、重试、消息队列、资源释放 | [concurrency.md](references/concurrency.md) |
| 稳定性与可观测 | #23-#26 日志、监控、链路追踪、优雅上下线 | [stability.md](references/stability.md) |
| 安全与防御 | #27-#29 输入校验、最小权限、敏感数据 | [security.md](references/security.md) |
| 工程实践 | #30-#35 测试、Code Review、文档、易错点、长运行稳定性、向后兼容 | [engineering.md](references/engineering.md) |
| 反模式速查 | 全维度 | [anti-patterns.md](references/anti-patterns.md) |
| 快速清单 | 全维度 | [checklist.md](references/checklist.md) |
