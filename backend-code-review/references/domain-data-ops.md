# V. Data & Operations

## Table of Contents
- [22. Cache Consistency](#22-cache-consistency)
- [23. Monitoring & Graceful Shutdown](#23-monitoring--graceful-shutdown)
- [24. Distributed Tracing](#24-distributed-tracing)

---

## 22. Cache Consistency

### Search Keywords

```
Cache operations:
  Go: cache., Cache., redis.Get, redis.Set, redis.Del, ristretto, bigcache, freecache
  Java: @Cacheable, @CacheEvict, @CachePut, CacheManager, RedisTemplate, Caffeine, Ehcache
  Python: cache, @lru_cache, @cached, redis.get, redis.set, redis.delete
  TypeScript: cache., .get(, .set(, .del(, redis., ioredis

Write operations (cache update timing):
  Go: func.*Update, func.*Save, func.*Delete
  Java/Python/TS: update, save, delete related methods
```

### Checklist
- [ ] Is cache update strategy clear (Cache Aside / Write Through / Write Behind)
- [ ] Does cache have TTL (expiration time)
- [ ] Does cache invalidation have anti-breakdown protection (singleflight / mutex)
- [ ] Cache avalanche risk: do large numbers of keys expire simultaneously
- [ ] Cache penetration risk: are non-existent keys cached as empty values
- [ ] Is cache update order correct on write operations (update DB first then delete cache)

### Deep Dive Method
1. Draw cache read/write paths
2. **For each path simulate exception scenarios**:
   - Cache expires between query and write, what happens?
   - DB update succeeds but cache deletion fails, what happens?
   - Concurrent read/write on same key, what are results under various timings?
3. Check for inconsistent time windows
4. Give specific cache strategy code examples

### Scenario Simulation

**Concurrent update**: A and B simultaneously update same data, both update DB first then delete cache. Timing: A updates DB → B updates DB → B deletes cache → A deletes cache. After A deletes cache, next read request loads B's new value, eventually consistent. But there's a brief inconsistency window before A deletes cache.

**Cache breakdown**: Hot key expires, 1000 requests simultaneously miss cache → 1000 requests simultaneously query DB. DB can't handle it instantly. Is there singleflight or mutex?

**Cache avalanche**: 10000 keys with same TTL expire simultaneously, 10000 requests simultaneously hit DB. Connection pool 500 → exhausted → avalanche. Solution: TTL with random offset `baseTTL + random(0, 300s)`, or multi-level cache.

**Cache penetration**: Large number of non-existent ID queries, cache miss → all hit DB → crash. Solutions: Bloom filter, cache empty values (short TTL), entry ID legitimacy validation.

**Cache deletion failure**: DB update succeeds but cache deletion fails → read stale data. How long until recovery? TTL 1 hour → 1 hour inconsistency. Is there retry deletion, subscribe to binlog for async deletion, or fallback TTL?

> **Deep dive signal**: Batch setting same TTL → cache avalanche (→ #23 Monitoring). Malicious query for non-existent IDs → is it input validation or cache strategy issue? (→ #20 Input Validation). Is there alert for sudden cache hit rate drop? (→ #23 Monitoring)

---

## 23. Monitoring & Graceful Shutdown

### Search Keywords

```
Metrics monitoring:
  Go: prometheus, promhttp, Counter, Gauge, Histogram, Summary
  Java: MeterRegistry, @Timed, @Counted, micrometer
  Python: prometheus_client, Counter, Gauge, Histogram, Summary
  Rust: prometheus, metrics
  TypeScript: prom-client, prometheus

Health checks: health, Health, healthz, readyz, livez, /health, readiness, liveness

Graceful shutdown:
  Go: signal.Notify, os.Signal, SIGTERM, SIGINT, syscall.SIGTERM, Shutdown
  Java: @PreDestroy, DisposableBean, ShutdownHook, GracefulShutdown
  Python: signal.signal, SIGTERM, shutdown, lifespan
  Rust: tokio::signal, ctrl_c, tokio::select
  TypeScript: process.on.*SIGTERM, process.on.*SIGINT, graceful, shutdown
```

### Checklist

#### Metrics Monitoring
- [ ] Are there basic metrics for QPS/latency/error rate
- [ ] Are critical business metrics monitored
- [ ] Is there resource usage monitoring (CPU, memory, connection pool)
- [ ] Are alert thresholds reasonable

#### Health Checks
- [ ] Are there health check endpoints (/healthz, /readyz, /livez)
- [ ] Does readiness probe check all dependencies (database, Redis, MQ)
- [ ] Is there warmup before accepting traffic on startup

#### Graceful Shutdown
- [ ] Does service have SIGTERM signal handling
- [ ] Does shutdown wait for in-flight requests to complete
- [ ] Is there shutdown timeout (forced exit time)

### Deep Dive Method
1. List all defined metrics, mark missing items against business processes
2. Check if health check endpoints cover all critical dependencies
3. **Simulate kill -SIGTERM**: What about in-flight requests? Does transaction rollback? Are messages lost?
4. Give missing metrics definitions and graceful shutdown code examples

### Scenario Simulation

**No monitoring**: P99 latency rises from 200ms to 2 seconds, no alert, only discovered after user complaints. If there are QPS/latency/error rate metrics, can discover and alert in 5 minutes. What if payment interface P99 rises from 200ms to 5s?

**Rolling update**: K8s deploys, old Pod receives SIGTERM but still has 50-100 requests in progress. Direct exit → connection reset → user sees "operation failed" or even "data loss" (write requests). Correct approach: stop accepting new requests → wait for in-flight requests to complete (30s timeout) → force exit.

**Health check false positive**: Health check only returns 200 but doesn't check dependencies. Database is down but health check passes → K8s doesn't remove Pod → all requests fail. Correct: readiness probe checks DB/Redis/MQ connectivity, return 503 if any is unavailable.

> **Deep dive signal**: Will messages being processed during shutdown be lost? Can re-consumption after restart be idempotent? (→ #16 Idempotency). Does critical interface have latency histogram? Is cache hit rate monitored?

---

## 24. Distributed Tracing

### Search Keywords

```
Tracing frameworks:
  Go: otel, opentelemetry
  Java: opentelemetry, brave, sleuth, TraceId, SpanId, @NewSpan
  Python: opentelemetry, jaeger, zipkin, tracer
  Rust: opentelemetry, tracing::
  TypeScript: opentelemetry

Context propagation:
  Go: context.Context
  Java: RequestContextHolder, MDC, ThreadLocal
  Python: contextvars, ContextVar

traceID variables: traceID, TraceID, trace_id, span_id, SpanId, requestID, RequestID, correlationID
```

### Checklist
- [ ] Is tracing framework integrated (OpenTelemetry / Jaeger, etc.)
- [ ] Is there traceID generation and propagation
- [ ] Is context completely propagated through call chain
- [ ] Does cross-service call propagate traceID (HTTP header / gRPC metadata)

### Deep Dive Method
1. Find entry point, check traceID generation
2. **Trace traceID propagation path through call chain**: check layer by layer if context is propagated
3. Check for break points (calls that lose context)
4. **Focus check**: Are goroutines / async calls / message queues propagating context
5. Give complete traceID propagation chain fix plan

### Scenario Simulation

**Broken chain**: Request has traceID from gateway, still has it at Service A, but loses traceID when A calls B (didn't put in HTTP header/gRPC metadata). When issues occur, can only guess by time window and business parameters. Root cause: cross-service call didn't propagate trace context.

**Async broken chain**: Main process has traceID, but async process triggered by MQ doesn't. When async issues occur, how to associate with original request? Can only search logs by business ID. Correct approach: propagate traceID as message header.

**Logs without traceID**: Error occurs online, 1000 error logs, don't know which belong to same request. In concurrent scenarios completely indistinguishable. Fix: all logging frameworks extract traceID from context and write to every log (Go slog + traceID, Java MDC, Python contextvars).

> **Deep dive signal**: Does cross-service call client correctly inject trace header? (→ #12 traceID coverage in logs). Do message queue producer and consumer propagate traceID? (→ #16 Idempotency, #12 Logging)

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| `cache.Set(key, value)` | After write operation, is `cache.Del` done (delete cache rather than update cache) | ⚠️ Cache inconsistency |
| `redis.Set(key, value)` | Is there TTL | ⚠️ Cache never expires |
| Cache read miss | Is there anti-breakdown (singleflight/mutex) | ⚠️ Cache breakdown risk |
| Large number of keys set simultaneously | Is TTL with random offset | ⚠️ Cache avalanche risk |
| External API | Is there QPS/latency/error rate monitoring | ⚠️ Monitoring blind spot |
| Critical business operation (payment/order) | Is there business metrics instrumentation | ⚠️ No business monitoring |
| `ListenAndServe` / `app.run` | Is there SIGTERM signal handling | ❌ No graceful shutdown |
| Health check endpoint | Does it check all dependencies (DB/Redis/MQ) | ⚠️ False health |
| `context.Context` propagation | Is traceID propagated in goroutine/async calls | ⚠️ Chain break |
| Cross-service call (HTTP/gRPC) | Is traceID propagated (header/metadata) | ⚠️ Chain break |
