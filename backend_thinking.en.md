# Backend Engineer Thinking Framework

English | [中文](./backend_thinking.md)

> An engineer's level is not determined by how much code they write, but by how much they think before writing code.
> This document is not dogma, but a thinking checklist — a self-check guide for every time you write code, design systems, or troubleshoot issues.

---

## Table of Contents

- [I. Code Quality: The Art of Writing for Humans](#i-code-quality-the-art-of-writing-for-humans)
- [II. Architecture Design: Making Systems Sustainable](#ii-architecture-design-making-systems-sustainable)
- [III. Database & Storage: Data Is the Core Asset](#iii-database--storage-data-is-the-core-asset)
- [IV. Concurrency & Distributed Systems: The Source of Complexity](#iv-concurrency--distributed-systems-the-source-of-complexity)
- [V. Reliability Engineering: Keeping Systems Alive Under Adversity](#v-reliability-engineering-keeping-systems-alive-under-adversity)
- [VI. Performance & Scalability: Meeting the Challenge of Growth](#vi-performance--scalability-meeting-the-challenge-of-growth)
- [VII. Security Defense: Trust No External Input](#vii-security-defense-trust-no-external-input)
- [VIII. Observability & Operations: Making Systems Transparent and Understandable](#viii-observability--operations-making-systems-transparent-and-understandable)
- [IX. Engineering Practices & Team Collaboration](#ix-engineering-practices--team-collaboration)
- [Appendix: Quick Self-Check Checklist](#appendix-quick-self-check-checklist)

---

## I. Code Quality: The Art of Writing for Humans

> Code is read far more often than it is written. Good code should be like a good article — the reader can quickly understand the author's intent.

### 1.1 Naming Is Documentation

**Principle: Names should accurately convey intent and eliminate ambiguity.**

| Type | Anti-pattern ❌ | Correct Example ✅ | Explanation |
|------|----------------|-------------------|-------------|
| Variable | `d`, `data`, `tmp` | `elapsedDays`, `userData`, `tmpFileHandle` | Avoid meaningless abbreviations; describe "what it is" |
| Function | `process()`, `handle()` | `validateUserPermission()`, `calculateOrderTotal()` | Start with a verb; describe "what it does" |
| Class | `Manager`, `Util` | `PaymentProcessor`, `OrderRepository` | Noun; describe "what role it plays" |
| Constant | `3.14`, `60` | `MAX_RETRY_COUNT`, `CACHE_TTL_SECONDS` | Use names instead of comments |
| Filename | `util.go`, `helper.py` | `payment_processor.go`, `order_repository.py` | Filename reflects module responsibility |

**Deep Thinking:**
- Naming length should be proportional to its scope: local variables can be short (e.g., loop variable `i`), while global/exported names must be long and clear
- Boolean variables use `is/has/can/should` prefixes: `isAuthenticated`, `hasPermission`, `canRetry`
- Avoid pun naming: one name should express only one meaning; don't reuse the same name in different contexts

### 1.2 Function Design Principles

**Functions should be short, focused, and predictable.**

```
Characteristics of an ideal function:
├── Length: Within 20 lines (extremely complex logic should not exceed 50 lines)
├── Parameters: 0-2 is best, 3 is the upper limit; beyond that, encapsulate into a struct
├── Responsibility: Does only one thing; the function name describes everything it does
├── Level: Does not mix different levels of abstraction within the same function
└── Side effects: Ideally none; if present, must be reflected in the function name and comments
```

**Early returns, reduce nesting (Guard Clauses pattern):**

```go
// ❌ Arrow-shaped code, hard to read
func ProcessOrder(order *Order) error {
    if order != nil {
        if order.IsValid() {
            if order.HasStock() {
                // Real business logic buried at the third level
                return doProcess(order)
            } else {
                return errors.New("out of stock")
            }
        } else {
            return errors.New("invalid order")
        }
    } else {
        return errors.New("nil order")
    }
}

// ✅ Guard clause pattern: errors return early, main logic stays at the outermost level
func ProcessOrder(order *Order) error {
    if order == nil {
        return errors.New("nil order")
    }
    if !order.IsValid() {
        return errors.New("invalid order")
    }
    if !order.HasStock() {
        return errors.New("out of stock")
    }
    // Main business logic, clearly visible
    return doProcess(order)
}
```

### 1.3 Eliminate Magic Values

**Principle: No unexplained literals should appear in code.**

```go
// ❌ Magic numbers: readers cannot understand their meaning
if user.Status == 3 {
    sendEmail(user)
}

// ✅ Enum/constant, self-explanatory
if user.Status == model.UserStatusActive {
    sendWelcomeEmail(user)
}

// ✅ Better approach: use a method to encapsulate intent
if user.IsActive() {
    sendWelcomeEmail(user)
}
```

**Applicable Scenarios:**
- Business status codes → Enum (Enum/Iota)
- Configuration thresholds → Named constants or configuration items
- Error codes → Error types with semantics
- Timeout values, retry counts → Structured configuration

### 1.4 Duplicated Code Is Debt

**Principle: DRY (Don't Repeat Yourself), but don't over-abstract.**

Judgment criteria:
- **Rule of Three**: When copied for the third time, it must be refactored
- When extracting, first consider whether semantics have changed — two seemingly identical code fragments that may change independently in different business contexts should not be forcibly merged
- Prefer composition over inheritance for code reuse

### 1.5 Logs Are the Lifeline of Troubleshooting

**Principle: Critical paths must have logs, and logs must contain enough information to locate problems.**

```go
// ❌ Useless log: completely unable to locate issues when problems occur
log.Error("process failed")

// ✅ Valuable log: includes context, key data, and error reasons
log.Error("order payment callback failed",
    zap.String("order_id", orderID),
    zap.String("channel", paymentChannel),
    zap.Int64("amount", amount),
    zap.String("callback_status", callbackStatus),
    zap.Error(err))
```

**Log Level Strategy:**

| Level | When to Use | Examples |
|-------|------------|----------|
| DEBUG | During development debugging; usually disabled in production | Function parameters, intermediate variables |
| INFO | Key business process nodes | Order created successfully, payment completed |
| WARN | Recoverable anomalies that need attention | Retry succeeded, degradation triggered, slow query |
| ERROR | Business-impacting errors that require intervention | External service call failed, data inconsistency |

**Log Discipline:**
- Prohibit logging at INFO level or above inside loops (high volume leads to IO bottlenecks and storage explosion)
- Sensitive information desensitization: passwords, tokens, ID numbers, phone numbers
- Structured logs are better than formatted strings, facilitating log platform searches

---

## II. Architecture Design: Making Systems Sustainable

> Good architecture makes adding new features easy and reduces the cost of making mistakes.

### 2.1 Layering and Responsibility Boundaries

```
Classic three-layer architecture (each layer has clear responsibility boundaries):

┌─────────────────────────────────┐
│  Controller / Handler Layer     │  ← Parameter validation, permission check, protocol conversion
│  (Entry layer: receive in/out, no business logic) │
├─────────────────────────────────┤
│  Service / Domain Layer         │  ← Core business logic, orchestration, transaction management
│  (Core layer: business rules, protocol-agnostic) │
├─────────────────────────────────┤
│  Repository / DAO Layer         │  ← Data access, storage details
│  (Storage layer: only cares about data read/write) │
└─────────────────────────────────┘

Dependency direction: Upper layer → Lower layer; reverse dependencies are strictly prohibited
```

**Common Mistakes:**
- Writing business logic in Controller → Cannot reuse, cannot unit test
- Service directly manipulating HTTP objects (Request/Response) → Coupled to protocol layer
- Writing business judgments in Repository → Confusion between storage layer and business layer responsibilities

### 2.2 Programming to Interfaces and Dependency Inversion

**Principle: Core logic depends on abstract interfaces, not concrete implementations.**

```
Core value of dependency inversion:
1. Replaceability: Swap database, cache, or message queue by changing only the implementation, not the business logic
2. Testability: Mock interfaces for unit testing without starting real infrastructure
3. Evolvability: New implementations don't affect existing logic (Open/Closed Principle)
```

```go
// Define abstraction
type OrderRepository interface {
    GetByID(ctx context.Context, id string) (*Order, error)
    Save(ctx context.Context, order *Order) error
}

// Business layer only depends on interfaces
type OrderService struct {
    repo OrderRepository  // Doesn't care whether the implementation is MySQL or MongoDB
}

// Concrete implementation injected at initialization
func NewOrderService(repo OrderRepository) *OrderService {
    return &OrderService{repo: repo}
}
```

### 2.3 Separation of Configuration and Code

**Principle: Any value that changes depending on the environment should not be hard-coded.**

```
Configurations that must be externalized:
├── Infrastructure: database address, Redis address, message queue address
├── Timeouts & retries: HTTP timeout, DB connection timeout, retry count
├── Keys & credentials: API Key, Secret, JWT Signing Key
├── Feature toggles: gray release switch, degradation switch, A/B experiment flags
└── Business thresholds: limits, rate limiting values, cache TTL
```

**Configuration Management Best Practices:**
- Different environments (dev/staging/prod) are distinguished through environment variables or config centers; same code deploys to multiple environments
- Secret-type configurations go through Vault/Secret Manager, not in Git
- Configuration changes should ideally support hot updates, avoiding restarts for every config change

### 2.4 Single Responsibility and Avoiding "God Classes"

**Judgment criterion: If a class has more than one reason to change, it is taking on too many responsibilities.**

Anti-pattern signals:
- A Service file exceeds 500 lines
- A class has more than a dozen dependencies injected
- Changing one business feature requires modifying multiple places in the same file

Splitting strategies:
- Split by business domain: `OrderService` → `OrderCreationService` + `OrderPaymentService` + `OrderFulfillmentService`
- Split by technical concern: Make validation logic, notification logic, and persistence logic independent

---

## III. Database & Storage: Data Is the Core Asset

> The database is often the bottleneck of the system and the place most prone to problems. Be extra cautious with every line of code that operates on the database.

### 3.1 Indexing and Query Optimization

**Design Principles:**
- All columns involved in WHERE / JOIN / ORDER BY must be evaluated for indexing needs
- Columns with high distinctiveness (high selectivity) should be prioritized for indexing; columns like gender with only two values are not suitable for standalone indexing
- For composite indexes, follow the leftmost prefix principle; place high-selectivity columns first
- Control the number of indexes per table to within 5-6; more indexes are not better — writes incur index maintenance costs

**Query Anti-pattern Quick Reference:**

| Anti-pattern | Problem | Optimization |
|--------------|---------|--------------|
| `SELECT *` | Returns useless columns, wastes IO, affects covering index | Explicitly list required fields |
| `LIKE '%keyword%'` | Prefix fuzzy match cannot use index | Full-text index / Elasticsearch |
| `ORDER BY RAND()` | Full table scan + sort | Pre-randomization or application-layer randomization |
| Large offset pagination `LIMIT 100000,10` | Deep pagination scans large amounts of data | Cursor pagination / ID-based pagination |
| N+1 queries | Querying one by one in a loop | JOIN or batch IN queries |
| Implicit type conversion | Querying a `varchar` column with `int` causes index invalidation | Ensure parameter type matches column type |

### 3.2 Correct Use of Transactions

**Transactions are not a panacea; improper use can create problems.**

```
Three iron rules of transaction usage:
1. Transactions should be short — long transactions hold locks for a long time, blocking other operations
2. No external IO in transactions — RPC calls, HTTP requests should not be placed inside transactions
3. Transaction granularity should be minimized — only wrap operations that must guarantee consistency
```

```go
// ❌ Transaction contains RPC call: slow external service causes DB lock to be held continuously
tx := db.Begin()
tx.Create(&order)
paymentClient.Charge(order)  // Dangerous! Network call may take several seconds
tx.Commit()

// ✅ Complete local transaction first, then call external service; ensure eventual consistency via state machine + compensation
tx := db.Begin()
tx.Create(&order)
order.Status = StatusPendingPayment
tx.Commit()

// Call external service outside the transaction
err := paymentClient.Charge(order)
if err != nil {
    // Mark payment failure, handle via compensation mechanism later
    order.Status = StatusPaymentFailed
    db.Save(order)
}
```

### 3.3 Dealing with Data Growth

**Constantly ask yourself: What scale will this table's data volume grow to?**

| Data Scale | Strategy |
|------------|----------|
| Ten-thousands | Single table is sufficient; add good indexes |
| Hundred-thousands ~ Millions | Evaluate query patterns; consider covering indexes, read-write separation |
| Ten-millions | Sharding, hot-cold separation, archiving old data |
| Hundred-millions+ | Re-evaluate storage choices; consider TiDB, ClickHouse, Elasticsearch, etc. |

**Growth issues that must be considered:**
- Querying `COUNT(*)` is extremely slow on large tables → Use statistics tables or cache-maintained counts
- `DELETE` large amounts of data → Delete in batches to avoid long transactions locking the table
- Field length → Set VARCHAR as needed; TEXT types affect indexing strategy
- Join queries → Large table JOIN performance degrades; consider redundant fields or wide tables

### 3.4 Data Consistency Guarantees

```
Consistency strategy selection matrix:
┌─────────────────┬────────────────────────────────────┐
│  Strong Consistency│ Completed within the same transaction; suitable for monetary operations │
│  (Transaction)  │ Cost: high performance overhead, long lock hold time              │
├─────────────────┼────────────────────────────────────┤
│  Eventual Consistency│ Achieved via message queue, state machine, compensation mechanism │
│  (Saga/TCC)     │ Suitable for cross-service operations; delayed but eventually consistent │
├─────────────────┼────────────────────────────────────┤
│  Reconciliation │ Scheduled task compares upstream/downstream data differences      │
│  (Reconciliation)│ Last line of defense: discover and repair inconsistent data        │
└─────────────────┴────────────────────────────────────┘
```

---

## IV. Concurrency & Distributed Systems: The Source of Complexity

> Single-machine programming is deterministic; distributed programming is full of uncertainty. Networks fail, clocks drift, processes die, messages are lost or duplicated.

### 4.1 Concurrency Safety

**Shared mutable state is the root of concurrency bugs.**

```go
// ❌ Not concurrency-safe: multiple goroutines modify map simultaneously
var counter = make(map[string]int)
go func() { counter["key"]++ }()  // data race!
go func() { counter["key"]++ }()  // data race!

// ✅ Solution 1: Lock protection
var mu sync.Mutex
var counter = make(map[string]int)
mu.Lock()
counter["key"]++
mu.Unlock()

// ✅ Solution 2: Use concurrency-safe data structures
var counter sync.Map
counter.Store("key", newValue)

// ✅ Solution 3 (Best): Communicate via channels, eliminate shared state
ch := make(chan int, 100)
go func() { ch <- 1 }()
result := <-ch
```

**Lock Usage Discipline:**
- Lock granularity should be as small as possible: only lock the code segments that must be locked
- **Lock before querying, not after querying** — otherwise, the data queried may have been modified by other goroutines before the lock is acquired
- Avoid nested locks to prevent deadlocks; if nesting is necessary, ensure a globally consistent lock ordering
- Use `defer` to ensure locks are always released

### 4.2 Idempotency Design

**Principle: Any operation that may be called repeatedly must be idempotent.**

```
Scenarios that must be idempotent:
├── Payment callbacks (network retries cause duplicate notifications)
├── Message consumption (message queues' at-least-once semantics)
├── Inventory deduction (retries may cause over-deduction)
├── State transitions (duplicate requests should not change state)
└── Interface retries (client timeout triggers automatic retries)
```

**Idempotency Implementation Solutions:**

| Solution | Applicable Scenario | Implementation |
|----------|---------------------|----------------|
| Unique Request ID + Deduplication Table | General scenarios | Each request carries a unique ID; server records processed IDs |
| Business Unique Key | Business-related scenarios | E.g., orderNo + operationType combination for deduplication |
| Optimistic Lock (Version Number) | Data update scenarios | `UPDATE t SET val=new, ver=ver+1 WHERE id=x AND ver=old` |
| State Machine Constraints | State transition scenarios | Only allow transitions from specific states; duplicate requests become invalid because the state has already changed |

### 4.3 Basic Understanding of Distributed Systems

**Fallacies of Distributed Computing:**
1. ~~The network is reliable~~ → Networks fail; must have retries and timeouts
2. ~~Latency is zero~~ → Every remote call has latency; reasonable timeouts are needed
3. ~~Bandwidth is infinite~~ → Transmitting large data requires consideration of sharding and compression
4. ~~The network is secure~~ → Must encrypt and authenticate
5. ~~Topology doesn't change~~ → Services go up and down; service discovery is needed
6. ~~There is one administrator~~ → Multi-team collaboration requires contracts and documentation
7. ~~Transport cost is zero~~ → Serialization/deserialization has overhead
8. ~~The network is homogeneous~~ → Different environments vary greatly; compatibility is needed

### 4.4 Asynchronous and Message Queues

**Message queues are the "decoupler" and "buffer" of distributed systems, but introducing queues also introduces complexity.**

**Core Concerns:**

```
Message Queue Usage Self-Check List:
├── Idempotent consumption: Message queues usually provide at-least-once semantics; duplicate delivery is normal; consumers must be idempotent
├── Message ordering: Do messages with the same business key require strict ordering? Partition ordering vs global ordering
├── Consumer backlog: What if consumption speed can't keep up with production speed? Is there monitoring and alerting?
├── Dead letter handling: Where do failed messages go? Is a Dead Letter Queue (DLQ) configured?
├── Message loss: Does the producer have a compensation strategy for send failures? Don't silently drop messages
└── Eventual consistency: Asynchronous operations can't synchronize in real-time; is a compensation/reconciliation mechanism needed?
```

**Producer-Side Best Practices:**

```go
// Reliability guarantee for sending messages
func ProduceMessage(ctx context.Context, mq MessageQueue, msg *OrderMessage) error {
    // 1. Write to local transaction table first (Outbox pattern), ensuring message and business data are in the same transaction
    tx := db.Begin()
    tx.Create(&order)
    tx.Create(&OutboxMessage{
        Topic:   "order_created",
        Payload: msg,
        Status:  OutboxStatusPending,
    })
    tx.Commit()

    // 2. Async dispatcher scans Outbox table and sends messages to the queue
    // 3. After successful send, mark Outbox status as sent
    // 4. Scheduled task as safety net: resend messages that timed out without confirmation
    return nil
}
```

**Consumer-Side Best Practices:**

```go
func ConsumeOrderMessage(ctx context.Context, msg *OrderMessage) error {
    // 1. Idempotency check: determine if already processed based on messageID or business unique key
    if isProcessed(msg.ID) {
        return nil  // Duplicate message, acknowledge directly
    }

    // 2. Business processing (completed within a local transaction)
    tx := db.Begin()
    // ... execute business logic ...
    // 3. Record processing status (within the same transaction)
    tx.Create(&ConsumeRecord{MessageID: msg.ID, Status: StatusConsumed})
    tx.Commit()

    return nil
}
```

**Comparison of Three Common Message Consistency Patterns:**

| Pattern | Principle | Pros | Cons |
|---------|-----------|------|------|
| **Outbox Pattern** | Business data and messages written to the same database; background thread polls and sends | Strong consistency, no message loss | Requires additional Outbox table and polling mechanism |
| **Transaction Message** | MQ provides half-message mechanism (e.g., RocketMQ) | Good performance, no local table needed | Depends on specific MQ support |
| **Best-effort + Reconciliation** | Send message first; log on failure; scheduled reconciliation and resend | Simple to implement | Brief inconsistency window |

---

## V. Reliability Engineering: Keeping Systems Alive Under Adversity

> System reliability is not determined by performance during normal operation, but by performance when problems occur.

### 5.1 Rate Limiting, Circuit Breaker, Degradation

```
Three Lines of Defense:
┌─────────────────────────────────────────────┐
│ Rate Limiting                               │
│ Purpose: Protect yourself; reject excessive requests │
│ Strategy: Token Bucket / Leaky Bucket / Sliding Window │
│ Location: Gateway layer / Interface layer   │
├─────────────────────────────────────────────┤
│ Circuit Breaker                             │
│ Purpose: Protect downstream; stop calling when downstream error rate exceeds threshold │
│ States: Closed → Open → Half-Open → Closed  │
│ Implementation: Hystrix / Sentinel / resilience4go │
├─────────────────────────────────────────────┤
│ Degradation                                 │
│ Purpose: Protect core; sacrifice non-core features to ensure system availability │
│ Strategy: Return cached data / disable recommendations / disable comments, etc. │
└─────────────────────────────────────────────┘
```

### 5.2 Retry Strategies

**Three prerequisites for retries:**
1. **The error is retryable**: Network timeouts/connection resets are retryable; business errors (e.g., insufficient balance) are not
2. **The operation is idempotent**: Retries won't produce side effects
3. **Retries have a backoff strategy**: Avoid all clients retrying simultaneously causing a "thundering herd" effect

```go
// Standard implementation of exponential backoff + jitter
func RetryWithBackoff(ctx context.Context, fn func() error, maxRetries int) error {
    baseDelay := 100 * time.Millisecond
    maxDelay := 10 * time.Second

    for attempt := 0; attempt <= maxRetries; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }
        if !isRetryable(err) {
            return err  // Non-retryable error, return directly
        }
        if attempt == maxRetries {
            return fmt.Errorf("max retries (%d) exceeded: %w", maxRetries, err)
        }

        // Exponential backoff + random jitter
        delay := time.Duration(float64(baseDelay) * math.Pow(2, float64(attempt)))
        jitter := time.Duration(rand.Int63n(int64(delay) / 2))
        delay = min(delay+jitter, maxDelay)

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
        }
    }
    return nil
}
```

### 5.3 Timeout Control

**Principle: Every external call must have a timeout; never wait indefinitely.**

```
Timeout hierarchy:
HTTP request        → 3~10s (depending on business)
Database query      → 1~5s (complex reports may be relaxed)
Redis operation     → 100~500ms
RPC call            → Depends on downstream SLA, usually 1~5s
Connection timeout  → 3~5s
End-to-end timeout  → Global timeout for the entire request, should be greater than the sum of individual timeouts
```

### 5.4 Resource Release and Leak Prevention

```go
// Iron rule of resource release: use defer to ensure
func ProcessFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()  // No matter what happens later, it will be closed

    // Database connections, locks, HTTP Body are the same
    mu.Lock()
    defer mu.Unlock()

    resp, err := http.Get(url)
    if err != nil { return err }
    defer resp.Body.Close()
}
```

**Goroutine Leak Prevention:**
- Every goroutine must have a clear exit mechanism (context cancellation / done channel)
- Before starting a goroutine, ask yourself: when does it end? If you don't know, don't start it
- Monitor goroutine count with `runtime.NumGoroutine()`; continuous growth indicates a leak

### 5.5 Graceful Startup and Shutdown

```
Graceful Startup:
1. Start all dependency connections (DB, Cache, MQ)
2. Initialize cache (warm up hot data)
3. Register health check endpoint
4. Only accept traffic after health check passes
5. For multiple instances, consider rolling out one by one (canary release)

Graceful Shutdown:
1. Deregister from service registry, stop receiving new requests
2. Wait for in-flight requests to complete (set a graceful wait time, e.g., 15s)
3. Close connections, flush buffers
4. Exit process
```

---

## VI. Performance & Scalability: Meeting the Challenge of Growth

> The first step of performance optimization is not optimizing code, but confirming where the bottleneck is.

### 6.1 Performance Optimization Methodology

```
Correct steps for performance optimization:
1. Quantify current state → Use PProf / APM / load testing tools to get baseline data
2. Locate bottleneck → Which is the bottleneck: CPU / memory / IO / lock contention / network?
3. Optimize bottleneck → Only optimize the bottleneck, not non-bottlenecks (over-optimization is waste)
4. Verify effect → Compare load tests to confirm optimization is effective and has no side effects
5. Continuous monitoring → Prevent performance degradation
```

### 6.2 Caching Strategies

```
Caching Decision Tree:
             Data changes frequently?
            /             \
          Yes              No
          |                |
     Allow brief inconsistency?   Cache directly, set reasonable TTL
      /           \
    Yes            No
    |              |
 Cache Aside      Write Through
 (Update DB first, (Sync update DB and cache)
  then delete cache) Cost: high write latency
  + delayed double delete
```

**Common Caching Pitfalls:**
- **Cache penetration**: Querying non-existent data → Bloom filter / cache null values
- **Cache avalanche**: Large amounts of cache expire simultaneously → Add random offset to TTL
- **Cache hotspot**: Massive requests hit DB when hot key expires → Mutex lock / never expire + async refresh
- **Cache-DB inconsistency** → Define clear consistency strategy, accept eventual consistency, and use reconciliation as safety net

### 6.3 Batch Operations and Data Volume Awareness

**Principle: Never process large amounts of data at once.**

```go
// ❌ Load all data at once
var allOrders []Order
db.Find(&allOrders)  // OOM or timeout when data volume is large
for _, order := range allOrders {
    process(order)
}

// ✅ Batch processing
const batchSize = 500
var lastID int64 = 0
for {
    var batch []Order
    db.Where("id > ?", lastID).Order("id ASC").Limit(batchSize).Find(&batch)
    if len(batch) == 0 {
        break
    }
    for _, order := range batch {
        process(order)
    }
    lastID = batch[len(batch)-1].ID
}
```

**Data Volume Awareness Checklist:**
- How large will arrays/slices grow? Will they burst memory?
- How many rows will the database table have? Can queries still use indexes at large data volumes?
- Is there an upper limit on API response data volume? Is pagination needed?
- Will the data volume processed by scheduled tasks grow over time, exceeding the scheduling interval?

### 6.4 Considerations for Long-Running Programs

**What happens when a program runs for a long time?**

| Problem | Cause | Countermeasure |
|---------|-------|----------------|
| Continuous memory growth | Goroutine leak, unbounded cache, unreleased connections | Monitor memory trends, set cache limits, pprof analysis |
| Connection pool exhaustion | Connection leak, slow queries occupying connections | Set max connections, query timeouts, connection leak detection |
| File descriptor exhaustion | Unclosed files/connections/sockets | Always use defer close, monitor fd usage |
| Goroutine count out of control | Goroutines without exit mechanisms | Context cancellation mechanism, monitor goroutine count |
| Gradual performance degradation | Data volume growth, index fragmentation | Regular archiving, rebuild indexes, capacity planning |

---

## VII. Security Defense: Trust No External Input

> Security is not a feature, but a mindset. Every external input is a potential attack vector.

### 7.1 Input Validation Principles

```
Three Lines of Defense for Input Validation:
┌──────────────────────────────────────┐
│ First Line: Controller Layer         │
│ - Parameter type validation (required, format, range) │
│ - Use Validator framework for declarative validation │
│ - Intercept obviously invalid input  │
├──────────────────────────────────────┤
│ Second Line: Service Layer           │
│ - Business rule validation (state allowed, permissions) │
│ - Cross-field combined validation    │
├──────────────────────────────────────┤
│ Third Line: Database Layer           │
│ - Unique constraints, foreign key constraints, CHECK constraints │
│ - Last line of defense, preventing dirty data from entering the database │
└──────────────────────────────────────┘
```

### 7.2 Common Attack Protection

| Attack Type | Protection Measures |
|-------------|---------------------|
| SQL Injection | Always use parameterized queries/prepared statements; prohibit SQL concatenation |
| XSS | Output encoding (HTML/CSS/JS contexts each have corresponding encoding methods) |
| CSRF | SameSite Cookie + CSRF Token |
| Unauthorized Access | Interface-level authorization + data-level authorization (don't just check login status; also check data ownership) |
| Sensitive Information Leakage | Response desensitization, log desensitization, error messages don't expose internal details |
| Replay Attack | Request signature + timestamp + nonce |

### 7.3 Principle of Least Privilege

```
Least privilege practice checklist:
├── Database account: Application account only granted SELECT/INSERT/UPDATE/DELETE permissions, no DDL permissions
├── API calls: Token/Key only granted necessary scope, no global permissions
├── Server access: Application process does not run as root
├── Internal services: Service-to-service calls use internal network, with mTLS or service mesh authentication
└── CI/CD: Build environment credentials minimized, no production keys used
```

### 7.4 Sensitive Data Protection

**Principle: Sensitive data must have protective measures at every stage of storage, transmission, and display.**

```
Sensitive Data Protection Three-Layer Model:
┌────────────────────────────────────────────────────────┐
│  Storage Layer Protection                              │
│  - Passwords must be hashed with bcrypt/scrypt/argon2, plaintext prohibited │
│  - PII data such as ID numbers, bank card numbers encrypted at rest (AES-256-GCM) │
│  - Database encryption: Transparent Data Encryption (TDE) or application-layer encryption │
│  - Backup files must also be encrypted; many data leaks come from backups │
├────────────────────────────────────────────────────────┤
│  Transmission Layer Protection                         │
│  - All external communication uses TLS 1.2+, plaintext HTTP prohibited │
│  - Internal service calls also use TLS or mTLS; internal networks are not absolutely secure │
│  - API keys, tokens not passed via URL parameters, use Header instead │
├────────────────────────────────────────────────────────┤
│  Display Layer Protection                              │
│  - Log desensitization: mask passwords, tokens, ID numbers, phone numbers │
│  - Response desensitization: hide sensitive fields or partially mask in API responses │
│  - Error messages don't expose internal details (stack traces, SQL, file paths) │
└────────────────────────────────────────────────────────┘
```

**Key Management Best Practices:**

| Practice | Explanation |
|----------|-------------|
| ❌ Prohibited | Keys hard-coded in code, committed to Git, written in plaintext in config files |
| ❌ Prohibited | Multiple environments sharing the same set of keys |
| ✅ Recommended | Use KMS (Key Management Service) or Vault for unified key management |
| ✅ Recommended | Regular key rotation; quickly invalidate if leaked |
| ✅ Recommended | Different keys for different environments; production keys only accessible by operations/Secret management systems |
| ✅ Recommended | Keys dynamically fetched from Secret Manager when used, not persisted to disk |

**Common Desensitization Rules Examples:**

```
Phone:    138****1234          (Keep first 3 and last 4)
ID:       310***********1234   (Keep first 3 and last 4)
Bank Card:**** **** **** 5678  (Keep last 4)
Email:    c***@example.com    (Keep first letter and domain)
Password: [REDACTED]          (Completely hidden)
Token:    tk_****...****89ab  (Keep prefix and last 4 digits, for troubleshooting)
```

---

## VIII. Observability & Operations: Making Systems Transparent and Understandable

> A system without monitoring is like driving with your eyes covered — you don't know when an accident will happen.

### 8.1 Three Pillars of Observability

```
┌─────────────────────────────────────────────────────┐
│ Logs                                                 │
│ Purpose: Record discrete events, troubleshoot specific issues │
│ Key: Structured, with traceID, key business data    │
│ Tools: ELK / Loki + Grafana                          │
├─────────────────────────────────────────────────────┤
│ Metrics                                              │
│ Purpose: Aggregate data, discover trends and anomalies │
│ Essential metrics: QPS, latency P50/P95/P99, error rate │
│ Business metrics: Order volume, payment success rate, queue backlog depth │
│ Tools: Prometheus + Grafana                          │
├─────────────────────────────────────────────────────┤
│ Traces                                               │
│ Purpose: Restore the complete call chain of a request │
│ Key: Every request has a globally unique TraceID, passed across services │
│ Tools: Jaeger / Zipkin / SkyWalking / OpenTelemetry  │
└─────────────────────────────────────────────────────┘
```

### 8.2 Alert Design

**Alert Principles:**
- Alerts must be actionable — when you receive an alert, you should know what to do. Alerts that you don't know how to handle are noise
- Distinguish P0/P1/P2/P3 severity levels; different levels have different notification channels and response times
- Alerts should have clear thresholds; avoid "roughly okay"
- Regularly review alerts; delete alerts that have been ignored for a long time ("alert fatigue")

```
Typical alert rules examples:
├── P0 (Immediate response): Error rate > 5%, service unavailable, database connections exhausted
├── P1 (30-minute response): P99 latency > 2s, queue backlog > 10000
├── P2 (Handle same day): Disk usage > 80%, slow queries increasing
└── P3 (Plan to handle): Slow memory growth trend, technical debt alerts
```

### 8.3 Distributed Tracing Practice

```go
// Generate TraceID at request entry, pass through the entire chain
func OrderHandler(c *gin.Context) {
    ctx := context.WithValue(c.Request.Context(), "traceID", uuid.New().String())

    // All logs carry traceID
    logger := log.With(zap.String("traceID", getTraceID(ctx)))

    // When calling downstream services, pass via HTTP Header or gRPC Metadata
    req := http.NewRequest("GET", paymentServiceURL, nil)
    req.Header.Set("X-Trace-ID", getTraceID(ctx))
}
```

---

## IX. Engineering Practices & Team Collaboration

> One person can write good code; a team can build good systems.

### 9.1 Code Review

**Core dimensions Code Review should focus on:**

| Dimension | Focus | Not Focus |
|-----------|-------|-----------|
| Correctness | Is the logic correct? Are boundary conditions handled? | |
| Security | Are there injection, unauthorized access, and other security risks? | Code style (leave to lint tools) |
| Maintainability | Is it over-designed/under-designed? Is naming clear? | Personal preference differences |
| Performance | Are there obvious performance issues (N+1, IO inside large loops)? | |
| Compatibility | Does it break existing interfaces/data compatibility? | |

**Reviewer's Attitude:**
- Ask rather than command ("Would XXX be better here?" instead of "Change to XXX")
- Explain reasons ("Suggest changing because..." instead of "Change")
- Distinguish between must-fix (Must) and nice-to-have suggestions (Nice to have)

### 9.2 Testing Strategy

```
Testing Pyramid:
        ╱  E2E Tests  ╲        ← Few, verify core paths
       ╱────────────────╲
      ╱  Integration Tests ╲   ← Moderate, verify module interactions
     ╱────────────────────────╲
    ╱     Unit Tests            ╲  ← Many, cover core logic
   ╱──────────────────────────────╲
```

**Pragmatic Testing Principles:**
- Don't pursue 100% coverage, but core business must have tests (payment, amount calculation, state transitions)
- Use table-driven tests to cover normal/boundary/exception scenarios
- Test code is also code; maintain it with the same quality as business code
- Tests must be fast — slow tests will be skipped, and skipped tests are as good as none

### 9.3 Documentation and Knowledge Management

**Principle: Documentation is not a burden, but a multiplier of team communication efficiency.**

```
Documentation System:
├── API Documentation → Swagger/OpenAPI, updated synchronously with code
├── Design Documentation → Record "why it was done this way", not just "what was done"
├── Operations Documentation → Deployment architecture, runbooks, emergency plans
├── Change Log → CHANGELOG, what changed in each version
└── Knowledge Base → Pitfall records, technical decision records (ADR)
```

**Core Requirements for API Documentation:**
- Every interface must be documented: URL, Method, parameters, return values, error codes, examples
- Documentation inconsistent with implementation is worse than no documentation — must establish a documentation update mechanism
- Breaking changes must be notified in advance, with migration plans and transition periods provided

### 9.4 Compatibility Thinking for Changes

**Ask yourself every time you modify code:**

```
Compatibility Self-Check List:
├── [ ] Can old-version data work normally with the new code?
├── [ ] Can old-version clients call the new interface normally?
├── [ ] Is the database schema change backward compatible?
├── [ ] Does the config item change have a default value? Is the old config still valid?
├── [ ] Does the message format change affect messages being processed?
└── [ ] Is a gray release needed, to roll out gradually?
```

**Safe Principles for Database Changes:**
- Adding columns is fine; removing columns must be done in steps (stop reading first, then clean up)
- Changing types must be done in steps (add new column → migrate data → switch reads to new column → delete old column)
- Never directly delete columns/tables that are still in use; mark as deprecated first, clean up in the next version

---

## Appendix: Quick Self-Check Checklist

> Go through this checklist every time before committing code.

### Pre-Commit Quick Check

| # | Check Item | ✅ |
|---|------------|----|
| 1 | Is naming clear, unambiguous, and compliant with team conventions? | |
| 2 | Are functions short, single-responsibility, with reasonable parameters? | |
| 3 | Do errors return early, avoiding deep nesting? | |
| 4 | Are there magic numbers/strings not extracted as constants or enums? | |
| 5 | Are critical paths logged, with sufficient information to locate issues? | |
| 6 | Is there duplicated code that can be extracted? | |

### Database Related

| # | Check Item | ✅ |
|---|------------|----|
| 7 | Are WHERE/JOIN/ORDER BY fields indexed? | |
| 8 | Are transactions short, and do they contain external IO calls? | |
| 9 | Are large-batch operations executed in batches? | |
| 10 | Is performance after data growth considered? | |
| 11 | Do queries SELECT only the fields needed? | |

### Concurrency and Reliability

| # | Check Item | ✅ |
|---|------------|----|
| 12 | Is shared state protected by locks, with reasonable lock granularity? | |
| 13 | Is the lock acquired before querying (avoiding TOCTOU issues)? | |
| 14 | Are interfaces that may be called repeatedly idempotent? | |
| 15 | Do external calls have timeout control and retry strategies? | |
| 16 | Are resources (connections/locks/goroutines) guaranteed to be released? | |
| 17 | Is there degradation/circuit breaker protection? | |

### Security and Compatibility

| # | Check Item | ✅ |
|---|------------|----|
| 18 | Is all external input validated? | |
| 19 | Are SQL queries using parameterized queries? | |
| 20 | Is sensitive data desensitized? | |
| 21 | Are interfaces authenticated and authorized? | |
| 22 | Is the change backward compatible? Are old data/old clients affected? | |

### Operations and Observability

| # | Check Item | ✅ |
|---|------------|----|
| 23 | Are key metrics (QPS/latency/error rate) monitored? | |
| 24 | Is there distributed tracing and TraceID? | |
| 25 | Can the service start and shut down gracefully? | |
| 26 | Is configuration externalized, and are keys stored securely? | |

---

> **Final words: Writing code that runs is not hard; writing code that runs stably in production for the long term is where true skill shows.**
> Every rule behind this is a lesson learned from painful on-call incidents. Turn them into muscle memory, and your code quality will take a qualitative leap.
