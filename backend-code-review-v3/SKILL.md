---
name: backend-code-review
description: |
  后端代码质量审查技能。基于资深后端工程实践，从6大领域29个维度系统深入审查后端代码质量。
  Use when reviewing, auditing, or evaluating backend code quality — including Go, Java, Python, Rust, C++, or any backend language.
  Triggers: 代码审查, code review, 后端代码质量, backend quality, review backend code, 检查代码质量, 审查代码, CR, code review checklist。
---

# 后端代码质量审查

## 核心原则

**深度优先于广度。** 宁可深入分析3个问题，不要浅尝辄止29个点。每个发现的问题必须：定位到具体代码行 → 解释为什么有问题 → 给出修改后的代码。

## 工作流程（必须遵循）

### 第一步：询问审查方向（必须先问）

**开始审查前，必须先问用户打算审查哪个方向。** 用 AskUserQuestion 工具呈现以下选项：

> **你想重点审查哪个方向？可多选。**
>
> | 选项 | 说明 | 涵盖维度 |
> |------|------|----------|
> | 🔧 代码基础 | 命名、函数职责、参数、控制流、魔法值、代码复用 | #1-#6 |
> | 💪 健壮性与性能 | 数据库、并发、事务、数据增长、稳定性、日志 | #7-#12 |
> | 🏗️ 架构与设计 | 分层、接口、配置分离 | #13-#15 |
> | 🛡️ 容错与安全 | 幂等、熔断、重试、资源、输入校验、权限 | #16-#21 |
> | 📊 数据与运维 | 缓存、监控与上下线、链路追踪 | #22-#24 |
> | ✅ 测试与文档 | 测试、CR习惯、文档、易错点、兼容性 | #25-#29 |

用户选择后，**只读取选中方向对应的 reference 文件**。建议每次最多选2个方向，保证审查深度。

### 第二步：识别审查范围

用 Grep/Glob/ReadFile 工具扫描目标代码，确定涉及的模块、函数、数据流。列出要审查的文件和关键函数。

### 第三步：深入审查（每个维度必须做到）

**只读取第一步确定的 reference 文件**，对每个维度严格遵循以下步骤：

1. **扫描**：用 Grep 搜索代码中的相关模式（如查锁就搜 `sync.Mutex|sync.RWMutex|Lock\(|RLock(`）
2. **定位**：找到具体代码行，ReadFile 查看上下文
3. **分析**：对照 references 中的检查清单逐项验证
4. **判断**：给出 ✅/⚠️/❌，附具体代码行号和原因
5. **建议**：❌ 和 ⚠️ 必须给出修改后的完整代码片段

**禁止一句话带过。** 以下是错误示范和正确示范：

> ❌ 错误："并发安全：未发现明显问题"
> ✅ 正确："并发安全：搜索了 `sync.Mutex` 和 `Lock()` 发现 3 处加锁。其中 `service.go:142` 的 `UpdateStock` 方法存在 TOCTOU 问题——先查询库存再判断再扣减，两步操作未在同一个锁内，高并发下可能超卖。应改为 `SELECT ... FOR UPDATE` 或将判断与更新合并为单条 SQL：`UPDATE stock SET count = count - ? WHERE id = ? AND count >= ?`"

### 第四步：输出报告

```markdown
## 审查结果

### 总评：[🟢 通过 / 🟡 建议改进 / 🔴 必须修复]

### 审查范围
- 重点领域：健壮性与性能（#7-#12）
- 文件：xxx.go, yyy.go
- 模块：订单服务

### 深度发现

#### ❌ [CRITICAL] 并发竞态 — service.go:142 UpdateStock
**问题**：先查询再扣减不在同一锁内，TOCTOU 问题
**影响**：高并发下可能超卖
**修复**：
​```go
// Before
stock := s.repo.GetStock(ctx, sku)
if stock.Count >= qty {
    s.repo.DeductStock(ctx, sku, qty)
}
// After
affected, err := s.repo.DeductStockIfEnough(ctx, sku, qty) // 单条 SQL 原子操作
​```

#### ⚠️ [WARN] 魔法数字 — handler.go:58
...

#### ✅ [GOOD] 事务使用 — order.go:200
事务范围最小化，仅包含两条 INSERT，无外部调用...
```

## 六大审查领域索引

| 领域 | 维度 | Reference |
|------|------|-----------|
| 一、代码基础 | #1-#6 命名、函数职责、参数、控制流、魔法值、代码复用 | [domain-basics.md](references/domain-basics.md) |
| 二、健壮性与性能 | #7-#12 数据库与批量、并发、事务、数据增长、稳定性、日志 | [domain-robustness.md](references/domain-robustness.md) |
| 三、架构与设计 | #13-#15 分层、接口、配置分离 | [domain-architecture.md](references/domain-architecture.md) |
| 四、容错与安全 | #16-#21 幂等、熔断、重试、资源、输入校验、权限 | [domain-fault-tolerance.md](references/domain-fault-tolerance.md) |
| 五、数据与运维 | #22-#24 缓存、监控与上下线、链路追踪 | [domain-data-ops.md](references/domain-data-ops.md) |
| 六、测试与文档 | #25-#29 测试、CR习惯、文档、易错点、兼容性 | [domain-testing-docs.md](references/domain-testing-docs.md) |

## 关键提醒

- **必须先问用户选方向**，不要自作主张全维度扫描
- 用户选什么就审什么，只加载对应的 reference 文件
- 一定要用工具（Grep/ReadFile）实际检查代码，不要靠猜测
- 每个 ❌ 必须附带修复代码，不能只说"建议优化"
- 优先报告 ❌ 问题，⚠️ 次之，✅ 简要说明即可
- 如果某个维度不适用于当前代码，标注 N/A 并说明原因
