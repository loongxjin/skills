# Concurrency & Distributed Systems Review Checklist

## Checklist

### 4.1 Concurrency Safety

- [ ] Shared mutable state is protected by locks
- [ ] Lock granularity is minimized (only critical sections)
- [ ] Lock is acquired **before** read, not after (prevents TOCTOU issues)
- [ ] No nested locks; if required, global lock ordering is consistent
- [ ] Locks are released with `defer`
- [ ] Prefer channel/message passing to eliminate shared state
- [ ] Maps have write protection for concurrent access (Go maps are not safe)

### 4.2 Idempotency Design

- [ ] Payment callbacks are idempotent (network retries cause duplicate notifications)
- [ ] Message consumption is idempotent (MQ at-least-once semantics)
- [ ] Inventory deduction is idempotent (retries could double-deduct)
- [ ] State transitions are idempotent (duplicate requests should not change state)
- [ ] Interface retries are idempotent (client timeout auto-retry)

**Idempotency Implementation Options:**

| Approach | Use Case |
|----------|----------|
| Unique request ID + dedup table | General purpose |
| Business unique key (orderNo + operationType) | Business-specific |
| Optimistic locking (version number) | Data update scenarios |
| State machine constraints | State transition scenarios |

### 4.3 Distributed Awareness

- [ ] External calls have retries and timeouts (network is unreliable)
- [ ] Large data transfers consider sharding and compression (bandwidth is finite)
- [ ] Service discovery mechanism exists (topology changes)
- [ ] Service-to-service calls have encryption and auth (network is not secure)

### 4.4 Message Queues

- [ ] Consumer handles idempotently (duplicate delivery is normal)
- [ ] Message ordering is considered per business key (partition order vs global order)
- [ ] Consumer lag has monitoring and alerting
- [ ] Dead letter queue (DLQ) is configured for failed messages
- [ ] Producer send failures have compensation (do not silently drop messages)
- [ ] Async operations have compensation / reconciliation mechanisms

**Message Consistency Patterns:**

| Pattern | Best For |
|---------|----------|
| Outbox Pattern | Strong consistency required |
| Transactional Message (e.g. RocketMQ) | MQ supports half-message |
| Best-effort + Reconciliation | Acceptable brief inconsistency |
