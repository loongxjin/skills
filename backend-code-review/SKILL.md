---
name: backend-code-review
description: Backend code review skill based on a 9-dimension thinking framework. Use when reviewing backend code, conducting PR/MR reviews, assessing code quality, or identifying potential risks in backend systems. Covers code quality, architecture design, database and storage, concurrency and distributed systems, reliability engineering, performance and scalability, security, observability, and engineering practices. Applicable to Go, Java, Python, and other backend languages.
---

# Backend Code Review

Systematic code quality and risk assessment for backend systems based on a backend engineer thinking framework.

## Review Process

### 1. Determine Review Scope

Identify the code files, modules, and layers involved in the change. Mark critical files for focused review.

### 2. Dimension-by-Dimension Review

Review each of the following nine dimensions sequentially. Detailed checklists and anti-pattern examples for each dimension are in the corresponding file under `references/`:

| Dimension | Reference File | Core Focus |
|-----------|---------------|------------|
| Code Quality | [code-quality.md](references/code-quality.md) | Naming, function design, magic values, duplication, logging |
| Architecture | [architecture.md](references/architecture.md) | Layer boundaries, interface abstraction, config separation, SRP |
| Database & Storage | [database.md](references/database.md) | Indexing, transactions, N+1, data growth, consistency |
| Concurrency & Distributed | [concurrency.md](references/concurrency.md) | Thread safety, idempotency, message queues, distributed awareness |
| Reliability | [reliability.md](references/reliability.md) | Rate limiting, circuit breaker, retry, timeout, resource cleanup |
| Performance & Scalability | [performance.md](references/performance.md) | Caching, batch operations, memory awareness, long-running concerns |
| Security | [security.md](references/security.md) | Input validation, injection prevention, authorization, sensitive data |
| Observability | [observability.md](references/observability.md) | Logging standards, metrics, tracing, alerting |
| Engineering Practices | [engineering.md](references/engineering.md) | Test coverage, API docs, compatibility, safe migrations |

Read the corresponding reference file before reviewing each dimension.

### 3. Issue Severity Classification

Classify all findings into three levels:

| Level | Meaning | Examples |
|-------|---------|----------|
| 🔴 **Critical** | Could cause production incidents or data loss | SQL injection, RPC inside transactions, data races |
| 🟠 **Important** | Impacts system stability or maintainability | Missing timeouts, N+1 queries, missing idempotency |
| 🟡 **Suggestion** | Good to fix but not urgent | Unclear naming, incomplete logging, extractable common logic |

### 4. Output Review Report

Use the following format:

```
## Review Summary
Scope: [files / modules reviewed]
Overall Assessment: [1-2 sentence summary of code quality]

## Issues Found

### 🔴 Critical Issues
1. **[file:line]** Description of the issue
   - Why: Explain why this is a problem
   - Suggestion: Concrete fix or approach

### 🟠 Important Issues
...

### 🟡 Suggestions
...

## Positives
- Good practices worth acknowledging
```

## Quick Review Mode

For small changes (< 5 files), use the 26-item quick checklist below without loading reference files:

**Pre-commit checks**: Clear naming, short focused functions, early returns, no magic values, logging on critical paths, no code duplication

**Database**: Index coverage, short transactions without external I/O, batched bulk operations, data growth considered, no `SELECT *`

**Concurrency & Reliability**: Shared state protected by locks, lock-before-read, idempotent interfaces, timeout & retry strategies, resource cleanup, degradation protection

**Security & Compatibility**: Input validation, parameterized queries, sensitive data masking, endpoint authorization, backward compatibility

**Operations**: Key metrics monitored (QPS/latency/error rate), distributed tracing with TraceID, graceful startup/shutdown, externalized config, secure key storage

## Review Principles

- **Ask, don't command**: "Would it be better to use XXX here?" instead of "Change to XXX"
- **Explain why**: Every suggestion should include reasoning
- **Distinguish must-fix from nice-to-have**: Use severity levels
- **Give positive feedback**: Good design decisions are worth calling out
