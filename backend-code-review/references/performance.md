# Performance & Scalability Review Checklist

## Checklist

### 6.1 Performance Methodology

- [ ] Current state is quantified before optimization (PProf / APM / load testing)
- [ ] Bottleneck is identified before optimizing (CPU / memory / IO / lock contention / network)
- [ ] Only the bottleneck is optimized; avoid premature optimization
- [ ] Post-optimization load test confirms improvement with no side effects

### 6.2 Caching Strategy

- [ ] Hot read data uses caching
- [ ] Caching strategy matches business characteristics:

| Data Characteristic | Recommended Strategy |
|---------------------|----------------------|
| Infrequently changed | Cache directly with reasonable TTL |
| Frequently changed, brief inconsistency OK | Cache Aside (update DB first, delete cache, + delayed double delete) |
| Frequently changed, no inconsistency allowed | Write Through (sync update DB and cache) |

- [ ] Cache penetration prevented (nonexistent keys → bloom filter / cache null values)
- [ ] Cache avalanche prevented (TTL with random offset)
- [ ] Cache hot key breakdown prevented (mutex lock / never expire + async refresh)
- [ ] Cache has a capacity limit to prevent unbounded memory growth

### 6.3 Batch Operations & Data Volume

- [ ] Never load all data into memory at once
- [ ] Bulk processing uses batches (batch size ~500)
- [ ] API responses have pagination and upper limits
- [ ] Scheduled job data volume growth does not exceed execution interval
- [ ] Arrays / slices have bounded growth to prevent OOM

### 6.4 Long-running Considerations

| Problem | Cause | Mitigation |
|---------|-------|------------|
| Memory持续增长 | Goroutine leak, unbounded cache, unreleased connections | Monitor trends, set cache limits, pprof analysis |
| Connection pool exhaustion | Connection leak, slow queries | Max connections, query timeouts, leak detection |
| File descriptor exhaustion | Unclosed files / connections / sockets | Always `defer close`, monitor fd usage |
| Goroutine失控 | Goroutines without exit mechanism | Context cancellation, monitor goroutine count |
| Gradual performance degradation | Data growth, index fragmentation | Periodic archiving, index rebuild, capacity planning |
