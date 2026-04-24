# 五、数据与运维

## 目录
- [22. 缓存一致性](#22-缓存一致性)
- [23. 监控与优雅上下线](#23-监控与优雅上下线)
- [24. 链路追踪](#24-链路追踪)

---

## 22. 缓存一致性

### 搜索模式

```bash
# Go
grep -rn 'cache\.\|Cache\.\|redis\.Get\|redis\.Set\|redis\.Del\|ristretto\|bigcache\|freecache' --include='*.go'
# Java
grep -rn '@Cacheable\|@CacheEvict\|@CachePut\|CacheManager\|RedisTemplate\|Caffeine\|Ehcache' --include='*.java'
# Python
grep -rn 'cache\|@lru_cache\|@cached\|redis.get\|redis.set\|redis.delete' --include='*.py'
# TypeScript
grep -rn 'cache\.\|\.get(\|\.set(\|\.del(\|redis\.\|ioredis' --include='*.ts'

# 写操作（缓存更新时机）
grep -rn 'func.*Update\|func.*Save\|func.*Delete\|def.*update\|def.*save\|def.*delete' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
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
   - 缓存刚好在查询和写入之间过期，会出现什么？
   - DB 更新成功但删缓存失败，会怎样？
   - 并发读写同一个 key，各种时序下的结果是什么？
3. 检查是否存在不一致的时间窗口
4. 给出具体的缓存策略代码示例

### 场景推演

#### 并发更新推演
A、B 两个请求同时更新同一条数据，都先更新 DB 再删缓存。时序：A 先更新 DB → B 后更新 DB → B 先删缓存 → A 后删缓存。结果：缓存被 A 删除后，下一个读请求会从 DB 加载 B 的新值，数据最终一致。但如果在 A 删缓存之前有一个读请求，读到的可能是 A 的旧值。这种时序下存在短暂的不一致窗口，需评估业务能否接受。

> **深挖信号**：并发更新缓存时是否需要分布式锁？是否需要 singleflight 防止缓存击穿？→ 追问 #8 并发安全

#### 缓存失效风暴推演
缓存中 10000 个 key 同时过期（比如设置了相同的 TTL），瞬间会有 10000 个请求同时打到数据库。如果数据库最大连接数是 500，会出现连接池耗尽，后续请求全部超时，引发雪崩。解决方案：给 TTL 加随机偏移（如 `baseTTL + random(0, 300s)`），或使用多级缓存（L1 本地缓存 + L2 Redis）。

> **深挖信号**：是否存在批量设置相同 TTL 的代码？缓存雪崩的保护措施是否到位？→ 追问 #23 监控

#### 缓存穿透推演
有人用大量不存在的 ID 查询（如 ID 从 1 到 1000000），缓存没有这些 key，每个请求都会穿透到数据库。数据库每秒承受百万级无效查询，很快就会崩溃。解决方案：布隆过滤器过滤不存在的 ID、缓存空值（短 TTL）、在入口层做 ID 合法性校验。

> **深挖信号**：恶意查询不存在的 ID，是输入校验的问题还是缓存策略的问题？→ 追问 #20 输入校验

#### 异常路径推演
DB 更新成功了但删缓存失败了，接下来读到的都是旧数据。多久才能恢复？取决于 TTL：如果 TTL 是 1 小时，就有 1 小时的数据不一致。有没有补偿机制？方案：重试删缓存（带退避）、订阅 DB binlog 异步删缓存、或设置合理的兜底 TTL。

> **深挖信号**：缓存删除失败有没有重试机制？缓存命中率暴跌有没有告警？→ 追问 #23 监控

#### 补充推演场景

> **场景1（缓存更新 vs 失效）**：`UpdateUser` 里先更新 DB，再 `cache.Set(user)`。线程A和B并发更新同一条数据：A更新DB值为1，B更新DB值为2，B先写缓存为2，A后写缓存为1。缓存里是旧值1，DB里是新值2。**不一致。**
>
> **场景2（缓存击穿）**：一个热点 key 过期了，瞬间 1000 个请求同时 miss 缓存，1000 个请求同时查 DB。DB 瞬间扛不住。有没有 singleflight 或互斥锁？
>
> **场景3（缓存雪崩）**：所有 key 的 TTL 都是 3600 秒，且都是服务启动时写入的。1小时后，所有 key 同时过期，全部请求打到 DB。DB 扛不住。
>
> **场景4（缓存穿透）**：黑客用大量不存在的 ID 查询，缓存 miss，全部打到 DB。DB 扛不住。

---

## 23. 监控与优雅上下线

### 搜索模式

```bash
# 指标监控
# Go
grep -rn 'prometheus\|promhttp\|metric\|Counter\|Gauge\|Histogram\|Summary' --include='*.go'
# Java（Micrometer/Prometheus）
grep -rn 'MeterRegistry\|@Timed\|@Counted\|Counter\|Gauge\|Timer\|micrometer' --include='*.java'
# Python
grep -rn 'prometheus_client\|Counter\|Gauge\|Histogram\|Summary\|metrics' --include='*.py'
# Rust
grep -rn 'prometheus\|metrics\|Counter\|Gauge\|Histogram' --include='*.rs'
# TypeScript
grep -rn 'prom-client\|prometheus\|metrics\|Histogram\|Counter\|Gauge' --include='*.ts'

# 健康检查端点
grep -rn 'health\|Health\|healthz\|readyz\|livez\|/health\|readiness\|liveness' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 优雅停机
# Go
grep -rn 'signal.Notify\|os.Signal\|SIGTERM\|SIGINT\|syscall.SIGTERM\|Shutdown\|ListenAndServe' --include='*.go'
# Java
grep -rn '@PreDestroy\|DisposableBean\|ShutdownHook\|GracefulShutdown\|server.shutdown' --include='*.java' --include='*.yml' --include='*.yaml' --include='*.properties'
# Python
grep -rn 'signal.signal\|SIGTERM\|shutdown\|on_event.*shutdown\|lifespan' --include='*.py'
# Rust
grep -rn 'tokio::signal\|ctrl_c\|tokio::select' --include='*.rs'
# TypeScript
grep -rn 'process.on.*SIGTERM\|process.on.*SIGINT\|graceful\|shutdown' --include='*.ts'
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
3. **模拟 kill -SIGTERM**：进行中的请求会怎样？事务会回滚吗？消息会丢吗？
4. 给出缺失的 metrics 定义和 graceful shutdown 代码示例

### 场景推演

#### 无监控推演
这个接口的 P99 延迟从 200ms 涨到了 2 秒，没有告警，用户投诉了你才知道。损失已经造成了。哪些指标能提前发现这个问题？至少需要：请求延迟 Histogram（P50/P95/P99）、错误率 Counter、依赖服务延迟。告警规则应该是：P99 > 500ms 持续 3 分钟 → 告警。

> **深挖信号**：关键接口有没有延迟分位图？告警阈值是否覆盖了业务场景？→ 追问缓存命中率、依赖服务状态等关联指标

#### 滚动更新推演
K8s 发版时，旧 Pod 收到 SIGTERM 但还有 100 个请求在处理。如果直接退出，这 100 个请求会收到连接重置错误，用户体验是"操作失败"甚至"数据丢失"（如果是写请求）。正确做法：收到 SIGTERM 后停止接收新请求，等待进行中的请求完成（设超时，如 30s），超时后再强制退出。

> **深挖信号**：停机时正在处理的消息会丢失吗？重启后重新消费能幂等吗？→ 追问 #16 幂等

#### 健康检查假阳性推演
健康检查只返回 200 但不检查依赖，数据库挂了但健康检查通过。K8s 不会摘除这个 Pod，所有请求都会失败。正确做法：就绪探针（readiness）应检查所有关键依赖（DB、Redis、MQ）的连通性，任一依赖不可用就返回 503，让 K8s 自动摘除。

> **深挖信号**：健康检查是否真正验证了服务的可用性？是否检查了所有关键依赖？→ 追问 #23 监控的完整性

#### 补充推演场景

> **场景1（无监控）**：支付接口的 P99 延迟从 200ms 涨到 5s，但没有指标监控，没人发现。直到用户大量投诉才知道。如果有 QPS/延迟/错误率指标，5分钟就能发现并告警。
>
> **场景2（无优雅停机）**：K8s 滚动更新，向 Pod 发送 SIGTERM。Pod 直接退出，正在处理的 50 个支付请求被中断。用户扣了钱但订单状态还是"未支付"。如果有优雅停机，会等这 50 个请求处理完再退出。

---

## 24. 链路追踪

### 搜索模式

```bash
# Tracing 框架集成
# Go
grep -rn 'otel\|opentelemetry' --include='*.go'
# Java
grep -rn 'opentelemetry\|brave\|sleuth\|TraceId\|SpanId\|@NewSpan' --include='*.java'
# Python
grep -rn 'opentelemetry\|jaeger\|zipkin\|tracer' --include='*.py'
# Rust
grep -rn 'opentelemetry\|tracing::' --include='*.rs'
# TypeScript
grep -rn 'opentelemetry\|@nestjs/microservices' --include='*.ts'

# Context 传递
grep -rn 'context.Context' --include='*.go'
grep -rn 'RequestContextHolder\|MDC\|ThreadLocal' --include='*.java'
grep -rn 'contextvars\|ContextVar' --include='*.py'

# traceID 变量命名
grep -rn 'traceID\|TraceID\|trace_id\|span_id\|SpanId\|requestID\|RequestID\|correlationID' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 注意：日志内容中的 traceID 检查在 domain-robustness.md #12 可观测性
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

#### 断链推演
请求从网关进来有 traceID，到了 Service A 还有，但 Service A 调 Service B 时 traceID 丢了。出了问题你只能看到"Service B 收到一个没有 traceID 的请求"，怎么关联到原始请求？——没法直接关联，只能靠时间窗口和业务参数去猜。根因：Service A 调 Service B 时没有把 traceID 放到 HTTP header（如 `traceparent`）或 gRPC metadata 中。修复：确保所有跨服务调用都正确传播 trace context。

> **深挖信号**：跨服务调用的 HTTP/gRPC 客户端是否正确注入了 trace header？→ 追问 #12 日志中的 traceID 覆盖率

#### 异步调用断链推演
主流程有 traceID，但通过消息队列触发的异步流程没有。异步流程出了问题，怎么跟原始请求关联？——没有 traceID，只能靠业务 ID 去搜索日志，效率极低。正确做法：将 traceID 作为消息 header 传递，异步消费者从消息 header 中提取 traceID 并设为新 span 的 parent（或使用 `Links` 关联）。

> **深挖信号**：消息队列的生产者和消费者是否传递了 traceID？异步任务的上下文是否完整？→ 追问 #16 幂等、#12 日志

#### 日志无 traceID 推演
日志里只有时间戳和消息内容，没有 traceID。线上报错了，日志里有 1000 条错误，你怎么知道哪些是同一个请求的？——没法知道，只能靠时间窗口粗略筛选，但并发场景下完全无法区分。修复：确保所有日志框架都从 context 中提取 traceID 并写入每条日志（如 Go 的 `slog` + `slog.Int64("traceID", ...)`，Java 的 MDC，Python 的 `contextvars`）。

> **深挖信号**：traceID 丢失后，日志还怎么关联？关键路径的日志能定位问题吗？→ 追问 #12 日志的 traceID 注入

#### 补充推演场景

> **场景**：用户反馈"下单失败"。日志里有订单服务的错误，但错误发生在调用了库存服务之后。你需要看库存服务的日志，但没有 traceID 关联，你只能按时间范围搜索。如果有 traceID，1秒就能定位跨服务的完整调用链。如果 context 在中间某层断了，链路就断了。

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
