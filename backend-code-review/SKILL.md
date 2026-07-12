---
name: backend-code-review
description: Backend code quality review across 6 domains and 29 dimensions. Use when reviewing, auditing, or evaluating backend code.
---

# Backend Code Quality Review

**Depth over breadth.** Find the critical path, then dig deep along it. Three real issues beat 29 superficial checks.

**Follow the vine, don't checkbox.** Valuable issues hide in call chains and cross-layer interactions, not in isolated functions. Map critical paths first; let them guide where to dig.

## Severity

| Marker | Meaning | Action |
|--------|---------|--------|
| ❌ CRITICAL | Security vulnerability, data correctness, production availability risk | Blocks merge |
| ⚠️ WARN | Improvement suggested | Record as follow-up |
| 💡 SUGGEST | Optional — better approach, style preference | Author may ignore |
| ℹ️ INFO | Context, no action needed | — |
| ✅ GOOD | Worth praising | — |

**Blocking principle**: Only block for issues that would cause a production incident if unfixed. Style preferences and alternative approaches are 💡 SUGGEST, not ❌ CRITICAL.

## Domains

| Domain | Dimensions | Reference |
|--------|-----------|-----------|
| 🔧 Code Fundamentals | #1-#6 Naming, Function Responsibility, Parameters, Control Flow, Magic Values, Code Reuse | [domain-basics.md](references/domain-basics.md) |
| 💪 Robustness & Performance | #7-#12 Database & Batch, Concurrency, Transaction, Data Growth, Stability, Logging | [domain-robustness.md](references/domain-robustness.md) |
| 🏗️ Architecture & Design | #13-#15 Layering, Interface, Config Separation | [domain-architecture.md](references/domain-architecture.md) |
| 🛡️ Fault Tolerance & Security | #16-#21 Idempotency, Circuit Breaker, Retry, Resources, Input Validation, Permissions | [domain-fault-tolerance.md](references/domain-fault-tolerance.md) |
| 📊 Data & Operations | #22-#24 Caching, Monitoring & Deployment, Distributed Tracing | [domain-data-ops.md](references/domain-data-ops.md) |
| ✅ Testing & Documentation | #25-#29 Testing, CR Habits, Documentation, Pitfalls, Compatibility | [domain-testing-docs.md](references/domain-testing-docs.md) |

Load only the reference files for domains relevant to the current review. For small changes (<5 files) or quick scans, review code directly against the checklists. For large changes, trace call chains before diving into individual dimensions.

## Execution

1. **Map critical paths.** Identify entry points, trace call chains, mark high-risk nodes — external calls, transaction boundaries, shared state, write operations, error branches. Output the call chain summary; it's the most valuable part of the review.
2. **Deep dive along critical paths.** For each high-risk node, simulate concrete failure scenarios. For each error branch, trace: error silently swallowed? transaction rolled back? resources released?
3. **Cross-dimension check.** Flag high-risk combinations even if the dimensions weren't explicitly selected.
4. **Missing item detection.** Check what should be present but isn't.

## Scenario Simulation

For each suspicious point, test with concrete scenarios:

| Concern | Ask |
|---------|-----|
| Concurrency | 100 requests hit this simultaneously — what happens to shared state? |
| Transaction + External Call | RPC times out, DB rolls back but downstream already processed — consistent? |
| Idempotency | Called 3 times — what does the data look like? |
| Data Growth | At 100M rows, does this query still return in 1 second? |
| Resource Release | Error at line X — are connections/locks from earlier lines released? |
| Cache | Cache expires between read and write — what happens? |
| Retry | Succeeds on 3rd attempt — side effects from attempts 1 and 2? |
| Tracing | User reports a bug — can logs pinpoint the service and line? |

## Cross-Dimension Risks

| Combination | Check |
|-------------|-------|
| Transaction + External Call | RPC/MQ inside transaction? Rollback on call failure? |
| Concurrency + Database | Query-then-update atomic? TOCTOU? |
| Idempotency + Retry | Repeated calls cause duplicate charges/shipments? |
| Cache + Database | Cache eviction strategy correct? Consistent on error path? |
| Message Queue + Transaction | Send and DB in same transaction? Send fails but DB committed? |
| Input Validation + SQL | User input concatenated into SQL? Parameterized everywhere? |
| Retry + Rate Limiting | Retry storm risk? Backoff strategy? |
| Data Growth + Memory | Full load cause OOM? Pagination/streaming used? |

## Missing Item Patterns

| If you see | Must exist | Severity if missing |
|------------|-----------|---------------------|
| External call (HTTP/RPC) | Timeout + error handling | ❌ |
| Transaction Begin | No IO inside + has Rollback | ❌ |
| Write operation | Idempotent protection | ❌ |
| go func / thread / coroutine | Exit mechanism | ❌ |
| Lock | Corresponding Unlock (including error paths) | ❌ |
| Error handling | Log recording | ❌ |
| Cache read | TTL + anti-penetration | ❌ |
| Retry logic | Retriable judgment + backoff + max count | ❌ |
| File/connection open | defer/close | ❌ |
| Core business logic | Automated test | ⚠️ |

## Done When

1. Critical paths mapped with high-risk nodes identified
2. Every ❌ CRITICAL includes: exact code location, concrete failure scenario, corrected code
3. Every ⚠️ WARN and 💡 SUGGEST has location and rationale
4. Missing items and cross-dimension risks flagged
5. Overall verdict: 🔴 Must Fix / 🟡 Suggest Improvement / 🟢 Pass

## Integrity

- "No issues found" without stating what was searched and which files were reviewed is not a review.
- Call bugs what they are. "Might need attention" doesn't describe something that could cause financial loss.
- Quantify: "50 extra queries when listing 100 items" beats "performance issue."
- "Tests passed" is necessary, not sufficient. "I'll clean it up later" — experience says later never comes.
