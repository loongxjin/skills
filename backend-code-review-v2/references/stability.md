# 稳定性与可观测性审查指南

## #23 日志规范

### 核心规则

- 关键逻辑、分支判断、异常捕获处必须打日志
- 日志要包含关键数据、关键状态、关键错误信息，不能只打 "error happened"
- 分级：DEBUG → INFO → WARN → ERROR，生产环境禁止输出 DEBUG
- 禁止在日志中打印敏感信息（密码、token、身份证号、银行卡号）
- 日志格式建议结构化（JSON），带上 trace_id、timestamp、level、msg
- 常见错误：fmt.Println(err) 或 console.log(data) 直接打到 stdout，无法收集分析

### 搜索模式

```bash
# Go
grep -rn 'log\.\|logger\.\|slog\.\|Infof\|Errorf\|Warnf\|Debugf\|printf\|Println' --include='*.go'

# Java
grep -rn 'log\.\|logger\.\|Logger\|log\.info\|log\.error\|log\.warn\|log\.debug\|SLF4J\|Logback' --include='*.java'

# Python
grep -rn 'logging\.\|logger\.\|log\.\|print(' --include='*.py'

# Rust
grep -rn 'info!\|error!\|warn!\|debug!\|tracing::\|log::' --include='*.rs'

# TypeScript
grep -rn 'console\.\|logger\.\|winston\|pino\|bunyan' --include='*.ts'

# 错误处理但无日志（所有语言）
grep -rn 'if err\|catch (\|except\|Err(' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' -A2 | grep 'return\|throw\|raise' | grep -v 'log\|logger\|logging'

# 关键业务操作
grep -rn 'pay\|charge\|deduct\|refund\|transfer\|create.*order\|update.*status\|delete.*account' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
```

### 检查清单

- [ ] 错误路径是否有日志
- [ ] 关键业务操作是否有日志（支付、状态变更、权限变更）
- [ ] 日志是否包含关键上下文（requestID, userID, 业务ID, 数值）
- [ ] 日志级别是否正确（Error vs Warn vs Info）
- [ ] 是否有敏感信息（密码、token）被打到日志中
- [ ] 关键函数入口/出口是否有日志

### 深挖方法

1. 列出所有错误处理分支，检查哪些没有日志
2. 对关键业务函数，追踪日志是否覆盖了成功/失败两个路径
3. 检查日志内容是否只有 "error" 而没有具体信息和上下文
4. 给出具体的日志补充建议

---

## #24 关键指标监控与告警

### 核心规则

- 指标（Metrics）：QPS、延迟 P99/P95、错误率、资源使用率（CPU/内存/磁盘/连接数）
- 关键业务指标：订单量、支付成功率、队列积压长度、任务执行耗时
- 告警阈值要合理，避免"狼来了"效应——告警太多等于没有告警
- 常见遗漏：只监控机器指标，不监控业务指标（如支付失败率突增）

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

## #25 链路追踪

### 核心规则

- 分布式系统中，一个请求可能跨越多个服务，必须传递 trace_id 串联全链路
- 每条日志都带上 trace_id，方便排查跨服务问题
- 调用下游服务时也要透传追踪标识
- 常见错误：入口服务生成了 trace_id，但调用下游时未透传，导致链路断裂

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

---

## #26 优雅上下线

### 核心规则

- 服务关闭时要优雅退出：停止接收新请求，处理完进行中的请求后再退出
- 启动时做好健康检查再接入流量，避免冷启动时请求失败或触发热数据加载瓶颈
- 常见错误：容器化部署时未配置优雅退出时间，直接被 kill -9

### 搜索模式

```bash
# 启动预热
grep -rn 'warmup\|Warmup\|warm_up\|preheat\|preload' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 容器配置
grep -rn 'terminationGracePeriodSeconds\|stopTimeout\|shutdown_timeout\|SIGKILL\|SIGTERM' --include='*.yaml' --include='*.yml' --include='*.json' --include='*.toml'

# 优雅停机信号处理（与 #24 搜索模式互补）
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

- [ ] 停止接收新请求后是否等待进行中的请求完成
- [ ] 启动时是否有预热机制（缓存预热、连接池预热）
- [ ] 容器 terminationGracePeriodSeconds 是否大于业务最长处理时间
- [ ] 是否处理了 SIGTERM 而不仅是 SIGKILL

### 深挖方法

1. 找到服务启动入口，检查预热逻辑
2. 模拟滚动更新：旧容器停止时进行中的请求会怎样？
3. 检查容器编排配置中的 graceful shutdown 参数
4. 给出 graceful shutdown 代码示例
