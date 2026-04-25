# 五、数据与运维

## 目录
- [22. 缓存一致性](#22-缓存一致性)
- [23. 监控与优雅上下线](#23-监控与优雅上下线)
- [24. 链路追踪](#24-链路追踪)

---

## 22. 缓存一致性

### 搜索关键词

```
缓存操作:
  Go: cache., Cache., redis.Get, redis.Set, redis.Del, ristretto, bigcache, freecache
  Java: @Cacheable, @CacheEvict, @CachePut, CacheManager, RedisTemplate, Caffeine, Ehcache
  Python: cache, @lru_cache, @cached, redis.get, redis.set, redis.delete
  TypeScript: cache., .get(, .set(, .del(, redis., ioredis

写操作（缓存更新时机）:
  Go: func.*Update, func.*Save, func.*Delete
  Java/Python/TS: update, save, delete 相关方法
```

### 检查清单
- [ ] 缓存更新策略是否明确（Cache Aside / Write Through / Write Behind）
- [ ] 缓存是否有 TTL（过期时间）
- [ ] 缓存失效时是否有防击穿保护（singleflight / 互斥锁）
- [ ] 缓存雪崩风险：大量 key 是否同时过期
- [ ] 缓存穿透风险：不存在的 key 是否缓存空值
- [ ] 写操作时缓存更新顺序是否正确（先更新DB再删缓存）

### 深挖方法
1. 画出缓存读写路径
2. **对每条路径推演异常场景**：
   - 缓存刚好在查询和写入之间过期，会怎样？
   - DB 更新成功但删缓存失败，会怎样？
   - 并发读写同一个 key，各种时序下的结果是什么？
3. 检查是否存在不一致的时间窗口
4. 给出具体的缓存策略代码示例

### 场景推演

**并发更新**：A、B 同时更新同一条数据，都先更新 DB 再删缓存。时序：A更新DB→B更新DB→B删缓存→A删缓存。A删缓存后下一个读请求加载B的新值，最终一致。但在A删缓存之前有短暂不一致窗口。

**缓存击穿**：热点 key 过期，1000个请求同时 miss 缓存→1000个请求同时查 DB。DB 瞬间扛不住。有没有 singleflight 或互斥锁？

**缓存雪崩**：10000个 key 同一 TTL 同时过期，10000个请求同时打 DB。连接池 500 → 耗尽→雪崩。解决方案：TTL 加随机偏移 `baseTTL + random(0, 300s)`，或多级缓存。

**缓存穿透**：大量不存在的 ID 查询，缓存 miss→全部打 DB→崩溃。解决方案：布隆过滤器、缓存空值（短 TTL）、入口 ID 合法性校验。

**删缓存失败**：DB 更新成功但删缓存失败→读到旧数据。多久恢复？TTL 1小时→1小时不一致。有没有重试删缓存、订阅 binlog 异步删、兜底 TTL？

> **深挖信号**：批量设置相同 TTL → 缓存雪崩（→ #23 监控）。恶意查询不存在的 ID → 是输入校验还是缓存策略问题？（→ #20 输入校验）。缓存命中率暴跌有没有告警？（→ #23 监控）

---

## 23. 监控与优雅上下线

### 搜索关键词

```
指标监控:
  Go: prometheus, promhttp, Counter, Gauge, Histogram, Summary
  Java: MeterRegistry, @Timed, @Counted, micrometer
  Python: prometheus_client, Counter, Gauge, Histogram, Summary
  Rust: prometheus, metrics
  TypeScript: prom-client, prometheus

健康检查: health, Health, healthz, readyz, livez, /health, readiness, liveness

优雅停机:
  Go: signal.Notify, os.Signal, SIGTERM, SIGINT, syscall.SIGTERM, Shutdown
  Java: @PreDestroy, DisposableBean, ShutdownHook, GracefulShutdown
  Python: signal.signal, SIGTERM, shutdown, lifespan
  Rust: tokio::signal, ctrl_c, tokio::select
  TypeScript: process.on.*SIGTERM, process.on.*SIGINT, graceful, shutdown
```

### 检查清单

#### 指标监控
- [ ] 是否有 QPS/延迟/错误率的基础指标
- [ ] 关键业务指标是否监控
- [ ] 是否有资源使用率监控（CPU、内存、连接池）
- [ ] 告警阈值是否合理

#### 健康检查
- [ ] 是否有健康检查端点（/healthz、/readyz、/livez）
- [ ] 就绪探针是否检查了所有依赖（数据库、Redis、MQ）
- [ ] 启动时是否预热后再接流量

#### 优雅停机
- [ ] 服务是否有 SIGTERM 信号处理
- [ ] 关闭时是否等待进行中的请求完成
- [ ] 是否有关闭超时（强制退出时间）

### 深挖方法
1. 列出所有已定义的 metrics，对照业务流程标记缺失项
2. 检查健康检查端点是否覆盖了所有关键依赖
3. **模拟 kill -SIGTERM**：进行中的请求怎样？事务回滚吗？消息丢吗？
4. 给出缺失的 metrics 定义和 graceful shutdown 代码示例

### 场景推演

**无监控**：P99 延迟从 200ms 涨到 2 秒，无告警，用户投诉后才发现。如果有 QPS/延迟/错误率指标，5分钟就能发现并告警。支付接口 P99 从 200ms 涨到 5s 呢？

**滚动更新**：K8s 发版，旧 Pod 收到 SIGTERM 但还有 50-100 个请求在处理。直接退出→连接重置→用户看到"操作失败"甚至"数据丢失"（写请求）。正确做法：停止接收新请求→等待进行中请求完成（30s超时）→强制退出。

**健康检查假阳性**：健康检查只返回 200 但不检查依赖。数据库挂了但健康检查通过→K8s 不摘除 Pod→所有请求失败。正确：就绪探针检查 DB/Redis/MQ 连通性，任一不可用返回 503。

> **深挖信号**：停机时正在处理的消息会丢失吗？重启后重新消费能幂等吗？（→ #16 幂等）。关键接口有延迟分位图吗？缓存命中率有监控吗？

---

## 24. 链路追踪

### 搜索关键词

```
Tracing 框架:
  Go: otel, opentelemetry
  Java: opentelemetry, brave, sleuth, TraceId, SpanId, @NewSpan
  Python: opentelemetry, jaeger, zipkin, tracer
  Rust: opentelemetry, tracing::
  TypeScript: opentelemetry

Context 传递:
  Go: context.Context
  Java: RequestContextHolder, MDC, ThreadLocal
  Python: contextvars, ContextVar

traceID 变量: traceID, TraceID, trace_id, span_id, SpanId, requestID, RequestID, correlationID
```

### 检查清单
- [ ] 是否集成了 tracing 框架（OpenTelemetry / Jaeger 等）
- [ ] 是否有 traceID 生成和传递
- [ ] context 是否在调用链中完整传递
- [ ] 跨服务调用是否传递 traceID（HTTP header / gRPC metadata）

### 深挖方法
1. 找到入口点，检查 traceID 生成
2. **追踪 traceID 在调用链中的传递路径**：逐层检查 context 是否被传递
3. 检查是否有断裂点（丢失 context 的调用）
4. **重点检查**：goroutine / 异步调用 / 消息队列 是否传递了 context
5. 给出 traceID 传递的完整链路修复方案

### 场景推演

**断链**：请求从网关有 traceID，到 Service A 还有，但 A 调 B 时 traceID 丢了（没放 HTTP header/gRPC metadata）。出了问题只能靠时间窗口和业务参数猜。根因：跨服务调用没传播 trace context。

**异步断链**：主流程有 traceID，通过 MQ 触发的异步流程没有。异步出问题，怎么跟原始请求关联？只能靠业务 ID 搜日志。正确做法：traceID 作为消息 header 传递。

**日志无 traceID**：线上报错，日志里 1000 条错误，不知道哪些是同一个请求的。并发场景下完全无法区分。修复：所有日志框架从 context 提取 traceID 写入每条日志（Go slog + traceID，Java MDC，Python contextvars）。

> **深挖信号**：跨服务调用的客户端是否正确注入了 trace header？（→ #12 日志中的 traceID 覆盖率）。消息队列生产者和消费者是否传递了 traceID？（→ #16 幂等、#12 日志）

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| `cache.Set(key, value)` | 写操作后是否 `cache.Del`（删缓存而非更新缓存） | ⚠️ 缓存不一致 |
| `redis.Set(key, value)` | 是否有 TTL | ⚠️ 缓存永不过期 |
| 缓存读取 miss | 是否有防击穿（singleflight/互斥锁） | ⚠️ 缓存击穿风险 |
| 大量 key 同时设置 | TTL 是否加了随机偏移 | ⚠️ 缓存雪崩风险 |
| 对外 API | 是否有 QPS/延迟/错误率监控 | ⚠️ 无监控盲区 |
| 关键业务操作（支付/下单） | 是否有业务指标埋点 | ⚠️ 无业务监控 |
| `ListenAndServe` / `app.run` | 是否有 SIGTERM 信号处理 | ❌ 无优雅停机 |
| 健康检查端点 | 是否检查了所有依赖（DB/Redis/MQ） | ⚠️ 虚假健康 |
| `context.Context` 传递 | traceID 是否在 goroutine/异步调用中传递 | ⚠️ 链路断裂 |
| 跨服务调用（HTTP/gRPC） | 是否传递 traceID（header/metadata） | ⚠️ 链路断裂 |
