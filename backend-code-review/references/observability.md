# Observability Review Checklist

## Checklist

### 8.1 Logging

- [ ] Logs are structured (JSON format for log platform searchability)
- [ ] Critical business nodes have INFO logs (order creation, payment completion)
- [ ] Recoverable exceptions have WARN logs (retry success, degradation trigger, slow query)
- [ ] Business-impacting errors have ERROR logs (external service failure, data inconsistency)
- [ ] Logs include traceID for correlation
- [ ] Logs include key business data (orderID, userID, etc.)
- [ ] No INFO or higher level logging inside loops
- [ ] Sensitive data is masked

### 8.2 Metrics

- [ ] Basic metrics collected: QPS, latency P50/P95/P99, error rate
- [ ] Key business metrics tracked (order volume, payment success rate, queue depth)
- [ ] Infrastructure metrics monitored (CPU, memory, disk, connections, goroutine count)

### 8.3 Distributed Tracing

- [ ] Request entry generates a TraceID
- [ ] TraceID propagated across services (HTTP Header / gRPC Metadata)
- [ ] All logs carry TraceID
- [ ] Standard tools used: OpenTelemetry / Jaeger / Zipkin

### 8.4 Alerting

- [ ] Alerts are actionable (recipient knows what to do)
- [ ] Severity levels are distinguished:

| Level | Response Time | Typical Scenario |
|-------|--------------|------------------|
| P0 | Immediate | Error rate > 5%, service down, DB connection pool exhausted |
| P1 | 30 minutes | P99 latency > 2s, queue backlog > 10000 |
| P2 | Same day | Disk usage > 80%, increased slow queries |
| P3 | Planned | Gradual memory growth, tech debt |

- [ ] Alert thresholds are explicit (no "roughly" estimates)
- [ ] Alerts are regularly reviewed; stale alerts are removed (prevent alert fatigue)
