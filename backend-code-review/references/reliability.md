# Reliability Engineering Review Checklist

## Checklist

### 5.1 Rate Limiting, Circuit Breaker, Degradation

- [ ] External interfaces have rate limiting (token bucket / leaky bucket / sliding window)
- [ ] Downstream service calls have circuit breaker (trip when error rate exceeds threshold)
- [ ] Circuit breaker has a Half-Open state for recovery probing
- [ ] Degradation strategy exists (return cached data / disable non-critical features)
- [ ] Thresholds are reasonable and dynamically adjustable

### 5.2 Retry Strategy

- [ ] Error is confirmed retryable before retrying (network timeout yes, business error no)
- [ ] Retried operation is idempotent
- [ ] Retry has backoff (exponential backoff + random jitter to prevent thundering herd)
- [ ] Retry count has an upper limit
- [ ] Retry respects context cancellation

**Standard Backoff Key Points:**
- `baseDelay * 2^attempt + random_jitter`
- Set `maxDelay` ceiling
- Support context cancellation interrupt

### 5.3 Timeout Control

- [ ] **Every external call has a timeout** (never wait indefinitely)
- [ ] Timeouts are reasonable:

| Call Type | Suggested Timeout |
|-----------|-------------------|
| HTTP request | 3-10s |
| DB query | 1-5s |
| Redis operation | 100-500ms |
| RPC call | Based on downstream SLA, usually 1-5s |
| Connection establishment | 3-5s |

- [ ] End-to-end timeout is set and exceeds the sum of individual component timeouts

### 5.4 Resource Cleanup

- [ ] File handles closed with `defer`
- [ ] DB connections released with `defer`
- [ ] Locks released with `defer`
- [ ] HTTP Response Body closed with `defer`
- [ ] Goroutines have explicit exit mechanisms (context cancellation / done channel)
- [ ] Goroutine lifecycle is defined before launch

### 5.5 Graceful Startup & Shutdown

- [ ] Startup initializes dependencies before accepting traffic
- [ ] Health check endpoint is registered
- [ ] Shutdown deregisters from service registry and waits for in-flight requests
- [ ] Graceful wait time is configured (e.g. 15s)
- [ ] Buffers are flushed and resources are released on exit
