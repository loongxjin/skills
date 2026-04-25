# IV. Fault Tolerance & Security

## Table of Contents
- [16. Idempotency Design](#16-idempotency-design)
- [17. Rate Limiting, Circuit Breaker, Degradation](#17-rate-limiting-circuit-breaker-degradation)
- [18. Retry Strategy](#18-retry-strategy)
- [19. Resource Release & Cleanup](#19-resource-release--cleanup)
- [20. Input Validation](#20-input-validation)
- [21. Least Privilege](#21-least-privilege)

---

## 16. Idempotency Design

### Search Keywords

```
Write operations (need idempotency attention): Create, Insert, Update, Delete, Pay, Charge, Deduct, Send, Publish
Idempotency keys: idempoten, request_id, reqID, dedup, idempotency
Retry scenarios: retry, callback, webhook, reconcil
```

### Checklist
- [ ] Are payment/deduction/stock deduction interfaces idempotent
- [ ] Is message sending idempotent (message duplicate consumption scenario)
- [ ] Are callbacks/webhooks idempotent
- [ ] Are unique request IDs used for deduplication
- [ ] Does database level have unique constraints to guarantee idempotency
- [ ] Are interfaces under retry scenarios naturally idempotent

### Deep Dive Method
1. List all write operation interfaces
2. **For each write operation ask**: What happens if called twice? Trace complete data change chain
3. Analyze if there are idempotency keys (request ID, business unique key)
4. Give idempotency transformation plan:
   - Unique constraint + INSERT IGNORE / ON CONFLICT
   - Redis: SETNX deduplication
   - State machine: only allow unidirectional state transitions

### Scenario Simulation

- **Replay simulation**: User clicks pay button 3 times consecutively, is money deducted 3 times? Third-party payment callback retries, is it processed repeatedly? Deduction interface retries 3 times, last two also succeed (just slow response), user is charged 3 times.
- **Partial failure**: Idempotency record writes successfully but business operation fails, will idempotency record block the request on next retry? Data remains inconsistent forever?
- **Expired idempotency**: When to clean up idempotency records? SETNX that never expires → same business ID can never be re-operated.
- **Message redelivery**: MQ at-least-once delivery, same message consumed 3 times, each time `INSERT INTO orders` without unique constraint → 3 duplicate orders.

> **Deep dive signal**: When write operations have no idempotency protection, check if upstream may retry (#18), MQ may redeliver (#8), or user may double-click. These combinations are the real high risks.

---

## 17. Rate Limiting, Circuit Breaker, Degradation

### Search Keywords

```
Rate limiting: rate.Limit, ratelimit, throttl, RateLimit, limiter, TokenBucket
  Go: golang.org/x/time/rate
  Java: @RateLimiter, Semaphore, Bucket4j, Resilience4j
  Python: ratelimit, throttle, slowapi, django-ratelimit

Timeout: Timeout, timeout, context.WithTimeout, context.WithDeadline, @Timeout
  Rust: tokio::time::timeout

Circuit breaker: circuit, breaker, CircuitBreaker, hystrix, resilience, Resilience4j, sentinel

External calls (need circuit breaker protection):
  Go: http.Client, grpc.Dial, redis.NewClient, .Call(
  Java: RestTemplate, WebClient, FeignClient, JedisCluster, RedisTemplate
  Python: requests., aiohttp., httpx., redis.
  Rust: reqwest::, hyper::, tonic::
  TypeScript: axios, fetch(, prisma., knex
```

### Checklist
- [ ] Do exposed interfaces have rate limiting protection
- [ ] Do external calls have timeout control
- [ ] Is there circuit breaker mechanism
- [ ] Does core chain have degradation plan
- [ ] Are timeout values reasonable

### Deep Dive Method
1. Find all external call points
2. Check if each call has timeout control
3. **Simulate snowball**: External call is down, impact on this service? Will it drag down upstream?
4. Assess risk without circuit breaker
5. Give rate limiting/circuit breaker configuration suggestions

### Scenario Simulation

- **Cascading failure**: Downstream is down without circuit breaker → thread pool exhausted → all requests timeout → upstream services depending on you also go down → snowball.
- **Timeout setting**: Downstream P99 latency is 2 seconds, is 3 second timeout reasonable? What about 30 seconds? If called inside transaction without timeout, how long does transaction hold locks?
- **Rate limit bypass**: Rate limiting only at gateway, internal calls unlimited. One malicious request amplified 100x through internal calls?
- **Unlimited attack**: Script requests 1000 times per second, database connection pool exhausted, all users unable to access.
- **No circuit breaker waste**: Downstream is already down, still calling every time → wait for timeout → fail → retry. With circuit breaker, directly return degraded result.

> **Deep dive signal**: External calls without timeout are the most common hidden risk. Link to #9 (Transaction Boundary) — external call without timeout inside transaction = long transaction + locks.

---

## 18. Retry Strategy

### Search Keywords

```
Retry: retry, Retry, backoff, Backoff, attempt, maxAttempt, maxRetries
  Go: github.com/cenkalti/backoff, github.com/sethvargo/go-retry
  Java: @Retryable, RetryTemplate, spring-retry, Resilience4j
  Python: tenacity, @retry, retrying, backoff
Infinite retry risk: while True, for {
```

### Checklist
- [ ] Before retry, are retriable errors distinguished from non-retriable errors
- [ ] Is there exponential backoff strategy
- [ ] Is there maximum retry count limit
- [ ] Are retried interfaces idempotent
- [ ] Are retry intervals reasonable

### Deep Dive Method
1. Find all retry logic
2. Analyze retry conditions: only retry retriable errors (network timeout yes, parameter error no)
3. Check backoff strategy (has it + random jitter)
4. **Verify if retried interface is idempotent** (non-idempotent cannot be retried)
5. Give retry framework code examples

### Scenario Simulation

- **Retry storm**: Timeout then retry 3 times → downstream bears 4x pressure. 10 instances fail simultaneously and retry simultaneously → thundering herd, downstream instantly crashes.
- **Non-retriable error**: Parameter error(400), insufficient permission(403) also retrying? Wasting resources and amplifying problems.
- **Retry + non-idempotent**: Deduction interface retries 3 times, last two also succeed → charged 3 times.

> **Deep dive signal**: Retry and idempotency are two sides of the same coin. Every retry logic must ask: is the interface idempotent? (#16). Retry + non-idempotent = disaster, especially payment, stock deduction. Link to #17 (Rate Limiting/Circuit Breaker) to check backoff strategy.

---

## 19. Resource Release & Cleanup

### Search Keywords

```
Resource cleanup:
  Go: defer.*Close, defer.*Unlock, defer.*Cancel, os.Open, sql.Open, net.Dial
  Java: try(, finally, .close(), .shutdown()
  Python: with open, with.*as, __enter__, __exit__, contextmanager
  Rust: impl Drop, drop(
  TypeScript: .dispose(), .close(), .destroy(), finally

Concurrency primitives (exit mechanism see #8):
  Go: go func
  Java: new Thread, @Async, CompletableFuture
  Python: threading.Thread, asyncio.create_task
  Rust: std::thread::spawn, tokio::spawn
```

### Checklist
- [ ] Are database connections closed in defer/finally/with
- [ ] Are file handles correctly closed
- [ ] Are locks released in defer/finally/with
- [ ] Do goroutines/threads/coroutines have exit mechanisms
- [ ] Are there resources only leaked on error paths

### Deep Dive Method
1. For each resource creation point, trace corresponding release point
2. **Focus on error paths**: Check if error paths skip release
   - Go: Is `defer` immediately after resource creation (not after error check)
   - Java: Is try-with-resources used (not manual close)
   - Python: Is `with` statement used
3. For goroutines/threads, trace exit conditions (see #8)
4. Give corresponding language resource management patterns

### Scenario Simulation

- **Exception path**: After opening connection, `if err != nil { return err }` returns directly without closing connection. Each error leaks one. Connection pool 50, after 50 errors service can't connect to database.
- **Goroutine leak**: `go processOrder(order)` without exit mechanism. Running for 7 days → 1 million goroutines → memory 200MB → 8GB → OOM.
- **Connection pool exhaustion**: Connections borrowed but not returned, after pool empties do new requests block or error? How long until timeout?

> **Deep dive signal**: The most dangerous thing about resource leaks is **accumulation effect**. One goroutine leak won't immediately cause issues, but after 7 days tens of thousands of goroutines trigger OOM. Link to #11 (Long-Running Stability) to check accumulated leaks, and #12 (Logging) to check if leaks are observable.

---

## 20. Input Validation

### Search Keywords

```
Parameter entry points:
  Go: ShouldBind, ShouldBindJSON, c.Query, c.Param, c.PostForm
  Java: @RequestBody, @PathVariable, @RequestParam, @Valid, @Validated
  Python: request.json, request.args, request.form, request.data
  TypeScript: @Body, @Param, @Query, req.body, req.params, req.query

SQL injection: fmt.Sprintf.*SELECT/INSERT/WHERE, String.format.*SELECT, f".*SELECT/INSERT/WHERE

Validation: validate, required, @NotNull, @NotBlank, @Size, @Min, @Max, validator
```

### Checklist
- [ ] Is all external input validated
- [ ] Is parameter validation completed at the outermost layer
- [ ] Does SQL use parameterized queries
- [ ] Is there XSS protection
- [ ] Are numeric ranges validated
- [ ] Java: Is `@Valid` + Bean Validation used
- [ ] Python: Is Pydantic / Marshmallow validation used

### Deep Dive Method
1. Find all API entry points
2. **Trace input data complete path from entry to usage**: user input → parameter parsing → business processing → data storage, is there validation on the path?
3. Check if there are paths that bypass entry validation directly to core logic (e.g., Service called by both Controller and Consumer)
4. For SQL concatenation, give parameterized query transformation

### Scenario Simulation

- **Injection simulation**: User input `'; DROP TABLE users;--`, what does the concatenated SQL look like? Do parameterized queries cover 100% of SQL?
- **Privilege escalation simulation**: User passes `userID=123` but it's not their ID, is there validation? Checked horizontal and vertical privilege escalation?
- **Abnormal input**: Empty string, 10MB super long string, negative number, JSON format error, special Unicode → how does code handle?
- **Pagination attack**: `page=-1&size=999999999` → `LIMIT 999999999 OFFSET -1`, what happens to database?

> **Deep dive signal**: The most easily overlooked is "trusting internal calls". Service layer is called by both Controller and Consumer, validation is done in Controller but not in Consumer. Link to #21 (Least Privilege) to check privilege escalation risks.

---

## 21. Least Privilege

### Search Keywords

```
Database high-privilege accounts: root@, admin@, sysdba, sa@, superuser, postgres://

Authentication:
  Go: middleware + auth/jwt/token/permission/rbac
  Java: @PreAuthorize, @Secured, @RolesAllowed, SecurityConfig
  Python: @login_required, @permission_required, is_authenticated, permission_classes

Dangerous operations: delete.*all, drop, truncate, DELETE FROM, DROP TABLE, TRUNCATE
```

### Checklist
- [ ] Does database connection use least-privilege account
- [ ] Do APIs have authentication middleware
- [ ] Do sensitive operations have secondary confirmation
- [ ] Is filesystem access restricted to necessary directories
- [ ] Are there operation audit logs

### Deep Dive Method
1. Check username in database connection string
2. Trace API authentication chain
3. **For unauthenticated interfaces, simulate malicious exploitation scenarios**
4. Give privilege minimization suggestions

### Scenario Simulation

- **Database account**: Using root/sa account to connect to database? SQL injection consequence is DROP DATABASE. Queries that can be completed with read-only account, why use read-write account?
- **Horizontal privilege escalation**: Can user A access user B's data by modifying request parameters? Does every API validate "this resource belongs to current user"?
- **Privilege escalation**: Can normal users call admin interfaces? Is permission check at routing layer or business layer? Is there possibility to bypass routing layer and call directly?
- **Unauthenticated interface**: `/api/users/:id` has no authentication, anyone knowing ID can delete. Crawler traverses IDs 1-100000?

> **Deep dive signal**: Having JWT validation ≠ having permission validation — JWT only proves "who you are", not "what you can do". Check if each operation validates "current user has permission to operate this specific resource". Link to #20 (Input Validation) to check parameter tampering privilege escalation.

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| Write operation (Pay/Charge/Deduct) | Idempotency protection (unique ID/unique constraint/state machine) | ❌ No idempotency |
| External call (HTTP/RPC) | Timeout control | ❌ No timeout |
| Exposed API | Rate limiting protection | ⚠️ No rate limiting |
| External call (HTTP/RPC) | Circuit breaker mechanism | ⚠️ No circuit breaker |
| Retry logic | Retriable judgment + backoff strategy + max count | ❌ Blind retry |
| Retried interface | Idempotency protection | ❌ Retrying non-idempotent interface |
| `os.Open` / `sql.Open` / `net.Dial` | `defer Close` / try-with-resources / `with` | ❌ Resource leak |
| `Lock()` | `defer Unlock()` also releases on error path | ❌ Lock leak |
| `go func` | Exit mechanism (ctx.Done / done channel) | ❌ Goroutine leak |
| API entry point | Input validation | ❌ No validation |
| SQL concatenation `fmt.Sprintf` | Parameterized query | ❌ SQL injection |
| Sensitive operation (delete/modify permission) | Authentication + audit log | ❌ Unauthorized access |
| Database connection | Non-root/admin account | ❌ Excessive privilege |
