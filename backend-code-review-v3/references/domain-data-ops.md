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
2. 分析异常路径：缓存失效、缓存错误、DB 更新失败时缓存状态
3. 检查是否存在不一致的时间窗口
4. 给出具体的缓存策略代码示例

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
3. 模拟 kill -SIGTERM：进行中的请求会怎样？
4. 给出缺失的 metrics 定义和 graceful shutdown 代码示例

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
2. 追踪 traceID 在调用链中的传递路径
3. 检查是否有断裂点（丢失 context 的调用）
4. 给出 traceID 传递的完整链路修复方案
