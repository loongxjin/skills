---
name: backend-code-review-en
description: |
  Backend code quality review skill. Based on senior backend engineering practices, systematically and thoroughly reviews backend code quality across 6 domains and 29 dimensions.
  Use when reviewing, auditing, or evaluating backend code quality — including Go, Java, Python, Rust, C++, or any backend language.
  Triggers: 代码审查, code review, 后端代码质量, backend quality, review backend code, 检查代码质量, 审查代码, CR, code review checklist。
---

# Backend Code Quality Review

## Core Principles

**Depth over breadth.** Better to deeply analyze 3 issues than to superficially check 29 dimensions. Every found issue must: pinpoint the exact code line → explain why it's problematic → provide corrected code.

**Follow the vine, don't checkbox.** Valuable issues often hide in the call chain, not in individual functions. First map out the critical paths, then dig deep along them — this is more effective than mechanically searching by dimension.

## Approval Criteria

**The goal of review is continuous improvement of code health, not pursuit of perfect code.**

| Overall Verdict | Condition |
|-----------------|-----------|
| 🔴 Must Fix | Has ❌ CRITICAL level issues (data correctness / security risk / production availability), must be fixed before merge |
| 🟡 Suggest Improvement | Has ⚠️ WARN level issues but does not block merge, record as follow-up issue |
| 🟢 Pass | No ❌ level issues, code improves system health. Does not need to be perfect, do not block for style preferences |

**Blocking principle**: Only block issues that "would cause a production incident if not fixed." Style preferences and better-ways-to-write suggestions should be marked with 💡 SUGGEST, not blocking merge. Don't block just because "I wouldn't write it this way" — if it improves the codebase and follows project conventions, approve it.

## Finding Severity Levels

| Marker | Meaning | Author Action |
|--------|---------|---------------|
| ❌ CRITICAL | Blocks merge | Security vulnerabilities, data correctness, production availability risks — must fix |
| ⚠️ WARN | Suggest improvement | Can be addressed in follow-up PRs, record as issue to track |
| 💡 SUGGEST | Optional suggestion | Better way to write, style suggestions — author can ignore |
| ℹ️ INFO | For your information | Context info, no action needed |
| ✅ GOOD | Worth praising | Good practices, encourage maintaining |

## Workflow (Must Follow)

### Step 1: Determine Review Scope and Direction

**Flexibly choose based on user intent, don't mechanically execute.** Decision logic:

| User Intent | Action | Should You Ask? |
|-------------|--------|-----------------|
| Already has a clear direction (e.g., "review concurrency safety" / "check SQL issues") | Go directly to corresponding dimension review | ❌ Don't ask |
| Small changes (< 5 files) and no specific direction | Quick full-dimension scan | ❌ Don't ask |
| Large amount of code and no specific direction | **Ask** the user which direction they want to focus on | ✅ Ask |

**When asking, present the following options via AskUserQuestion:**

> **Which direction(s) do you want to focus on? Multiple selections allowed.**
>
> | Option | Description | Covered Dimensions |
> |--------|-------------|-------------------|
> | 🔧 Code Fundamentals | Naming, function responsibilities, parameters, control flow, magic values, code reuse | #1-#6 |
> | 💪 Robustness & Performance | Database, concurrency, transactions, data growth, stability, logging | #7-#12 |
> | 🏗️ Architecture & Design | Layering, interfaces, config separation | #13-#15 |
> | 🛡️ Fault Tolerance & Security | Idempotency, circuit breaker, retry, resources, input validation, permissions | #16-#21 |
> | 📊 Data & Operations | Caching, monitoring & deployment, distributed tracing | #22-#24 |
> | ✅ Testing & Documentation | Tests, CR habits, docs, pitfalls, compatibility | #25-#29 |

After user selection, **only read the reference files for selected directions**. It's recommended to select 2-3 directions to ensure review depth; selecting more directions will proportionally reduce depth.

**Review Mode Selection**:

| Mode | Applicable Scenario | Process |
|------|---------------------|---------|
| ⚡ Quick Review | Small changes (< 5 files) or user says "quick look" | Skip step 2, search directly by selected dimensions |
| 🔍 Deep Review | Large code changes or user requests thorough review | Full 6-step process |

**Dimension Categories**: Different domains require different review approaches, don't apply heavy processes to all dimensions.

| Type | Applicable Domains | Characteristics | Review Method |
|------|-------------------|-----------------|---------------|
| 🪶 Lightweight Dimensions | 🔧 Code Fundamentals (#1-#6), ✅ Testing & Docs (#25-#29) | Local issues, found by reading code, don't depend on call chains | **Direct scan**: Read code → Check against reference checklist → Record findings directly. No need to trace call chains, scenario simulation, or cross-dimension association |
| 🔬 Deep Dimensions | 💪 Robustness & Performance (#7-#12), 🏗️ Architecture & Design (#13-#15), 🛡️ Fault Tolerance & Security (#16-#21), 📊 Data & Operations (#22-#24) | Involve runtime behavior, cross-layer interaction, data flow, require context to judge | **Full process**: Trace call chains → Scenario simulation → Cross-dimension association check |

**Execution Rules**:
- If user only selected lightweight dimensions → Skip step 2 (call chain tracing), step 4 (cross-dimension association), step 5 (missing item detection), directly read and scan code, output report
- If user selected deep dimensions → Follow full 6-step process, but any lightweight dimensions mixed in should still be treated as direct scan
- Lightweight dimension findings are typically 💡 SUGGEST or ⚠️ WARN level, rarely ❌ CRITICAL

**Change Size Assessment**: When determining review scope, also assess change size:

| Size | Changed Lines | Recommendation |
|------|---------------|----------------|
| Reasonable | < 300 lines | Normal review |
| Large | 300-1000 lines | Suggest user split into multiple PRs, each focusing on one logical change |
| Too Large | > 1000 lines | Strongly suggest splitting. Prompt: "This change is too large to guarantee review depth. Suggest splitting and reviewing individually." |

**Splitting Strategy**:

| Strategy | Method | Applicable Scenario |
|----------|--------|---------------------|
| By File Groups | Group files needing different review perspectives | Cross-layer changes |
| By Layers | First submit shared code / interface definitions, then submit consumers | Layered architecture |
| By Feature Slices | Split into smaller full-stack feature slices | Feature development |

**One change, one thing**: Refactoring and new features should be separate submissions, bugfix and optimization should be separate. Small cleanups (variable renaming) can be accepted at reviewer's discretion.

### Step 2: Map Core Call Chain + Identify Critical Paths (Only for Deep Dimensions)

**Don't rush to search by dimension. Spend 2-3 minutes mapping out the core call paths first.**

Use Grep/Glob/ReadFile tools to:

1. **Identify project language and framework**, subsequent review only focuses on corresponding language search patterns
2. **Find entry points**: Public methods in Controller/Handler, or MQ Consumer processing functions
3. **Read down the call chain**: Entry → Service → Repository/External calls, read code layer by layer
4. **Draw the call chain**: List critical paths in text, e.g.:

```
POST /orders
  → OrderHandler.Create()          # Entry: parameter validation, binding
    → OrderService.CreateOrder()    # Business: stock check, amount calculation
      → StockRepo.Deduct()          # DB: stock deduction (query then update?)
      → PaymentService.Charge()     # RPC: external payment call (in transaction?)
      → OrderRepo.Save()            # DB: save order
    → MessageProducer.Publish()     # MQ: send message (what if send fails?)
```

5. **Mark high-risk nodes**: Annotate the following types of locations on the call chain for prioritized deep dive:
   - **External calls** (HTTP/RPC/MQ) → Timeout? Circuit breaker? Failure handling?
   - **Transaction boundaries** (Begin/Commit) → IO inside transaction? Forgot rollback?
   - **Shared state** (locks/global variables/cache) → Race conditions? Consistency?
   - **Write operations** (Insert/Update/Delete) → Idempotent? Data growth?
   - **Error branches** (if err / catch / except) → Errors swallowed? Resource leaks?

6. **Output critical path summary**:

```
Critical Path: POST /api/orders → OrderHandler.Create → OrderService.CreateOrder → tx.Begin → OrderRepo.Insert + StockRepo.Deduct → MQ.Publish → tx.Commit
High-risk nodes: tx.Begin~tx.Commit (includes MQ.Publish), StockRepo.Deduct (concurrent deduction), MQ.Publish (send failure handling)
```

**⚠️ Must output call chain summary before continuing.** This step is the most valuable part of the review; skipping it significantly reduces review quality.

**Why is this step important?** Because searching by dimension is "carrying a hammer looking for nails" — you'll find many small issues but easily miss cross-layer combination issues. Mapping the call chain first lets you see the complete data flow and control flow, discovering cross-dimensional issues like "RPC called inside transaction", "query-then-update without atomic protection", or "message send fails but transaction already committed".

### Step 3: Deep Dive (Around Critical Paths + Selected Dimensions)

**Only read reference files determined in Step 1**.

**Lightweight Dimensions** (Code Fundamentals #1-#6, Testing & Docs #25-#29): Directly read code, check against reference checklist item by item, record findings as you go, no need for scenario simulation.

**Deep Dimensions** (Robustness & Performance #7-#12, Architecture & Design #13-#15, Fault Tolerance & Security #16-#21, Data & Operations #22-#24): Strictly follow these steps:

1. **Scan**: Use search keywords in reference to Grep search code
2. **Locate**: Find specific code lines, ReadFile to view context
3. **Analyze**: Check against references checklist item by item
4. **Scenario Simulation** (this determines review depth): For each suspicious point, **use concrete scenario simulation** rather than static judgment

**Scenario Simulation Checklist**:

| Dimension | Simulation Question |
|-----------|---------------------|
| Concurrency Safety | If 100 requests execute this code simultaneously, what happens to shared state? |
| Transaction | If this RPC inside the transaction times out, DB rolls back but downstream already processed, is state consistent? |
| Idempotency | If this API is called 3 times, what does the data look like? |
| Data Growth | When this table reaches 100 million rows, will this query still return in 1 second? |
| Resource Release | If this function errors at line X, are previously opened connections/locks released? |
| Caching | If cache expires between query and write, what happens? |
| Retry | Retry succeeds on the 3rd attempt, but what about the side effects of the first two? |
| Tracing | User reports a bug, can logs pinpoint which service and which line of code? |

**Error Path Tracing Requirements**:

For each error/exception branch, must ask:
- Is the error **silently swallowed**? (`if err != nil { return nil }` or `catch (e) {}`)
- Does transaction **rollback on error path**? (Or only Commit without Rollback?)
- Are resources **released on error path**? (Or only happy path handled?)
- Can caller **correctly handle this error**? (Or will upstream panic/crash?)

5. **Judge**: Give ✅/⚠️/❌ with specific line numbers and reasons
6. **Suggest**: ❌ and ⚠️ must provide complete corrected code snippets

**No one-liners allowed.** Below are wrong and right examples:

> ❌ Wrong: "Concurrency safety: no obvious issues found"
> ✅ Right: "Concurrency safety: searched `sync.Mutex` and `Lock()` and found 3 lock locations. The `UpdateStock` method at `service.go:142` has a TOCTOU issue — query stock then judge then deduct, two operations not within the same lock, may oversell under high concurrency. Should change to `SELECT ... FOR UPDATE` or merge judgment and update into single SQL: `UPDATE stock SET count = count - ? WHERE id = ? AND count >= ?`"

**Effort Allocation Suggestion**:
- Call chain tracing (Step 2): 20% time
- High-risk node deep dive: 50% time
- Other dimension scanning: 20% time
- Report organization: 10% time

### Step 4: Cross-Dimension Association Check

**After reviewing selected dimensions, must perform a cross-dimension association check.** Review the call chain from Step 2, check the following high-risk combinations:

| Combination | Question to Ask |
|-------------|-----------------|
| Transaction + External Call | Does transaction scope include HTTP/RPC/MQ calls? Does transaction rollback on call failure? |
| Concurrency + Database | Is query-then-update within the same lock/transaction? Is there TOCTOU? |
| Idempotency + Retry | Is the retried interface idempotent? Will repeated calls cause duplicate charges/shipments? |
| Cache + Database | Is cache update strategy correct? Update DB first then delete cache? On exception path are cache and DB consistent? |
| Message Queue + Transaction | Are message sending and DB operations within the same transaction? What if message send fails but DB already committed? |
| Input Validation + SQL | Is user input directly concatenated into SQL? Are parameterized queries covering everything? |
| Retry + Rate Limiting | Will retry storm overwhelm downstream? Is there backoff strategy? |
| Data Growth + Memory | Will fully loaded data cause OOM after growth? Is pagination/streaming used? |

If any of the above combinations exist on the call chain, even if the corresponding dimension is not in the user's selected scope, must flag and explain in the report.

### Step 5: Missing Item Detection

**Review what's "supposed to be there but isn't" in the code.** Compare against high-risk nodes marked in Step 2, check one by one:

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| External call (HTTP/RPC) | Timeout control + error handling | ❌ No timeout / No error handling |
| Transaction Begin | No IO inside transaction + has Rollback | ❌ IO mixed in transaction / No Rollback |
| Write operation (Insert/Update/Delete) | Idempotent protection | ❌ No idempotency |
| go func / thread / coroutine | Exit mechanism | ❌ No exit mechanism |
| Lock Lock | Corresponding Unlock (including error path) | ❌ Lock leak |
| Error handling if err / catch | Log recording | ❌ Silent error swallowing |
| Cache read | Cache invalidation strategy + anti-penetration | ❌ No TTL / No anti-penetration |
| Retry logic | Retriable judgment + backoff strategy + max count | ❌ Blind retry |
| File/connection open | defer/finally/with close | ❌ Resource leak |
| Core business logic | Automated test | ⚠️ No test coverage |

### Step 6: Output Report

```markdown
## Review Results

### Overall Verdict: [🟢 Pass / 🟡 Suggest Improvement / 🔴 Must Fix]

### Critical Path
POST /api/orders → OrderHandler.Create → OrderService.CreateOrder → ...
High-risk nodes: tx.Begin~tx.Commit (includes MQ.Publish), StockRepo.Deduct (concurrent deduction)

### Core Call Chain
(List call chain from Step 2, annotate locations where issues were found)

### Deep Findings

#### ❌ [CRITICAL] Race Condition — service.go:142 UpdateStock
**Issue**: Query then deduct not within the same lock, TOCTOU issue
**Simulation**: 100 concurrent requests execute simultaneously, thread A reads stock=1, thread B also reads stock=1, both pass judgment, deduct twice resulting in -1
**Impact**: Oversell, direct financial loss
**Fix**:
```go
// Before
stock := s.repo.GetStock(ctx, sku)
if stock.Count >= qty {
    s.repo.DeductStock(ctx, sku, qty)
}
// After
affected, err := s.repo.DeductStockIfEnough(ctx, sku, qty) // single SQL atomic operation
```

#### ⚠️ [WARN] Magic Number — handler.go:58
...

#### ✅ [GOOD] Transaction Usage — order.go:200
Transaction scope minimized, only contains two INSERTs, no external calls...

### Missing Items
| If You See | Missing | Location | Severity |
|------------|---------|----------|----------|
| External call paymentService.charge | No timeout control | service.go:89 | ❌ |
| goroutine go processOrder | No exit mechanism | worker.go:45 | ❌ |
| Error handling if err != nil | No log recording | handler.go:33 | ⚠️ |

### Cross-Dimension Association Risks
(List cross-dimensional issues found in Step 4)

### Dimension Check Summary
| # | Dimension | Status | Location | Notes |
|---|-----------|--------|----------|-------|
| 1 | Naming Conventions | ✅ | — | |
| 17 | Concurrency Safety | ❌ | service.go:142 | TOCTOU |
| 23 | Logging Standards | ⚠️ | logger.go:30 | Missing trace_id |
```

**❌ Issues sorted by fix priority**: Security vulnerability > Data correctness > Production availability > Performance > Maintainability

## Six Review Domain Index

| Domain | Dimensions | Reference |
|--------|-----------|-----------|
| I. Code Fundamentals | #1-#6 Naming, Function Responsibility, Parameters, Control Flow, Magic Values, Code Reuse | [domain-basics.md](references/domain-basics.md) |
| II. Robustness & Performance | #7-#12 Database & Batch, Concurrency, Transaction, Data Growth, Stability, Logging | [domain-robustness.md](references/domain-robustness.md) |
| III. Architecture & Design | #13-#15 Layering, Interface, Config Separation | [domain-architecture.md](references/domain-architecture.md) |
| IV. Fault Tolerance & Security | #16-#21 Idempotency, Circuit Breaker, Retry, Resources, Input Validation, Permissions | [domain-fault-tolerance.md](references/domain-fault-tolerance.md) |
| V. Data & Operations | #22-#24 Caching, Monitoring & Deployment, Distributed Tracing | [domain-data-ops.md](references/domain-data-ops.md) |
| VI. Testing & Documentation | #25-#29 Testing, CR Habits, Documentation, Pitfalls, Compatibility | [domain-testing-docs.md](references/domain-testing-docs.md) |

## Key Reminders

- **Flexibly choose based on user intent**: If direction is already clear, don't ask again; only ask when there's a large amount of code and no direction
- **Deep review must first identify critical paths**, dig deep around high-risk nodes, rather than evenly distributing effort
- **Lightweight dimensions (code fundamentals, testing & docs) get direct scan**, no need to trace call chains or simulate scenarios
- Review what the user selected, only load corresponding reference files
- Must use tools (Grep/ReadFile) to actually check code, don't guess
- **❌ for deep dimensions must include scenario simulation**: If X happens, what then
- Each ❌ must come with corrected code, cannot just say "suggest optimization"
- **Error paths and missing items are where issues most easily hide**, must check carefully
- If a dimension doesn't apply to current code, mark N/A and explain why
- **Cross-dimension association check cannot be skipped**, even if dimensions aren't in user's selection scope, risks must be flagged

## Review Integrity

- **Don't go through the motions.** "No obvious issues found" must be accompanied by what you actually searched and which files you reviewed. "LGTM" without evidence is no review.
- **Don't downplay real issues.** If it's a bug, call it a bug. Don't use "might need attention" to describe an issue that could cause financial loss.
- **Quantify issues.** "This N+1 query adds about 50 extra database queries when listing 100 items" is better than "there's a performance issue here." Quantify whenever possible.
- **Beware common excuses:**
  - "It works" → Code that works but is unreadable/unsafe will accumulate technical debt that compounds
  - "Tests passed" → Tests are necessary, not sufficient. Passing tests don't mean there are no architecture, security, or readability issues
  - "AI-generated code should be fine" → AI code needs stricter review; it's confident but can be wrong
  - "I'll clean it up later" → Experience shows "later" never comes. Require cleanup in the current change, or create an issue and self-assign it
- **Review code, not people.** Review comments target the code itself, not the author. Use "this code has a race condition" rather than "you wrote unsafe code".
