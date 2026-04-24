# 并发与分布式审查指南

本文件覆盖并发与分布式相关审查维度 #17-#22，每个维度包含四部分：
1. **核心规则** — 简明的审查要点
2. **搜索模式** — 多语言 grep 命令，快速定位可疑代码
3. **检查清单** — 逐项核查
4. **深挖方法** — 逐步分析流程

---

## #17 并发安全与锁策略

### 核心规则

- 多协程/线程访问共享资源**必须**考虑同步机制
- **不是"查询前加锁"，而是先查后判断，再对临界区加锁**
- 读多写少优先**乐观锁**（版本号/CAS），写多读少考虑**悲观锁**
- 锁粒度尽可能小，锁住的代码尽可能少，持有时间尽可能短
- 避免死锁：加锁顺序一致，设置锁超时

**语言特定的锁**：
- Go: `sync.Mutex`, `sync.RWMutex`, `sync/atomic`
- Java: `synchronized`, `ReentrantLock`, `ReadWriteLock`, `AtomicInteger`
- Python: `threading.Lock`, `threading.RLock`
- C++: `std::mutex`, `std::lock_guard`, `std::atomic`

### 搜索模式

```bash
# Go：锁
grep -rn 'sync.Mutex\|sync.RWMutex\|Lock()\|RLock()\|Unlock()\|RUnlock()\|sync.Map' --include='*.go'
# Go：goroutine
grep -rn 'go func\|go\s\+\w\+(' --include='*.go'
# Go：共享状态
grep -rn 'var\s\+\w\+\s\+=' --include='*.go' | grep 'map\|slice\|[]'

# Java：锁与并发
grep -rn 'synchronized\|ReentrantLock\|AtomicInteger\|AtomicLong\|ConcurrentHashMap\|volatile' --include='*.java'
# Java：线程池
grep -rn 'ThreadPool\|ExecutorService\|CompletableFuture\|@Async' --include='*.java'

# Python：并发
grep -rn 'threading.Lock\|multiprocessing\|asyncio\|async def\|await\|concurrent.futures' --include='*.py'

# Rust：并发原语
grep -rn 'Mutex::new\|RwLock::new\|Arc::new\|mpsc::\|spawn\|tokio::spawn\|std::thread' --include='*.rs'

# TypeScript：异步
grep -rn 'Promise\|async \|await \|Worker\|cluster' --include='*.ts'

# C++：锁
grep -rn 'std::mutex\|std::lock_guard\|std::atomic\|std::unique_lock\|std::shared_mutex' --include='*.cpp' --include='*.h'
```

### 检查清单

- [ ] 共享可变状态是否有锁保护
- [ ] 锁粒度是否合理（只锁临界区）
- [ ] 是否存在 TOCTOU 问题（Time of Check to Time of Use）
- [ ] 查询+更新是否在同一锁内
- [ ] Goroutine/Thread/协程是否有明确的退出机制
- [ ] 是否有泄漏风险
- [ ] 分布式场景下是否需要分布式锁

### 深挖方法

1. **画出并发访问路径**：哪些线程/协程读/写了同一份数据
2. **对每个共享状态**，检查是否有竞态条件
3. **检查锁的获取/释放是否成对**（Go: `defer Unlock`; Java: `try-finally`; Python: `with` 语句）
4. **给出具体的加锁方案**或改为 channel/消息传递的建议

---

## #18 幂等设计

### 核心规则

任何可能被重复调用的接口（支付、扣库存、发消息、回调通知）**必须幂等**。

- 通过**唯一请求 ID** 或**业务唯一键**去重
- 数据库唯一索引是幂等的最可靠实现
- **常见遗漏**：第三方支付回调、消息队列重投、用户重复点击

### 搜索模式

```bash
# 通用：写操作（需要关注幂等的接口）
grep -rn 'Create\|Insert\|Update\|Delete\|Pay\|Charge\|Deduct\|Send\|Publish' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'

# 通用：幂等键
grep -rn 'idempoten\|Idempoten\|request_id\|reqID\|dedup\|idempotency' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 通用：重复调用场景
grep -rn 'retry\|Retry\|reconcil\|Reconcil\|callback\|Callback\|webhook\|Webhook' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 支付/扣款/扣库存接口是否幂等
- [ ] 消息发送是否幂等（消息重复消费场景）
- [ ] 回调/webhook 是否幂等
- [ ] 是否使用唯一请求 ID 做去重
- [ ] 数据库层面是否有唯一约束保障幂等
- [ ] 重试场景下的接口是否天然幂等

### 深挖方法

1. **列出所有写操作接口**
2. **对每个写操作追问**：如果调用两次会发生什么？
3. **分析是否有幂等键**（请求ID、业务唯一键）
4. **给出幂等改造方案**：唯一约束 + `INSERT IGNORE` / `ON CONFLICT`、Redis `SETNX`、状态机

---

## #19 限流、熔断、降级

### 核心规则

- 对外暴露的接口**必须**有限流保护（令牌桶、漏桶、计数器）
- 依赖的外部服务要有**超时控制**和**熔断机制**，防止故障扩散
- 核心链路要有降级方案，非核心服务挂了不能拖垮全盘

### 搜索模式

```bash
# === 限流 ===
# 通用
grep -rn 'rate.Limit\|ratelimit\|throttl\|RateLimit\|limiter\|TokenBucket\|RateLimiter' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Go
grep -rn 'golang.org/x/time/rate\|ratelimit\.' --include='*.go'
# Java
grep -rn '@RateLimiter\|RateLimiter\|Semaphore\|Bucket4j\|Resilience4j' --include='*.java'
# Python
grep -rn 'ratelimit\|throttle\|slowapi\|django-ratelimit' --include='*.py'

# === 超时控制 ===
# 所有语言
grep -rn 'Timeout\|timeout\|context.WithTimeout\|context.WithDeadline\|@Timeout\|@TimeLimiter' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Rust
grep -rn 'timeout\|Duration\|tokio::time::timeout' --include='*.rs'

# === 熔断 ===
grep -rn 'circuit\|breaker\|CircuitBreaker\|hystrix\|resilience\|Resilience4j\|sentinel' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# === 外部调用（需要熔断保护） ===
# Go
grep -rn 'http.Client\|grpc.Dial\|redis\.NewClient\|\.Call(' --include='*.go'
# Java
grep -rn 'RestTemplate\|WebClient\|FeignClient\|JedisCluster\|RedisTemplate' --include='*.java'
# Python
grep -rn 'requests\.\|aiohttp\.\|httpx\.\|redis\.\|aioredis' --include='*.py'
# Rust
grep -rn 'reqwest::\|hyper::\|tonic::' --include='*.rs'
# TypeScript
grep -rn 'axios\|fetch(\|prisma\.\|knex' --include='*.ts'
```

### 检查清单

- [ ] 对外暴露的接口是否有限流保护
- [ ] 外部调用是否有超时控制
- [ ] 是否有熔断机制
- [ ] 核心链路是否有降级方案
- [ ] 超时时间是否合理

### 深挖方法

1. **找到所有外部调用点**
2. **检查每个调用是否有超时控制**
3. **评估没有熔断的风险**
4. **给出限流/熔断配置建议**

---

## #20 重试策略

### 核心规则

- 重试前判断**是否可重试**：网络超时、连接失败可以重试；业务错误、参数校验失败**不可**重试
- 必须有**退避策略**（指数退避 + 随机抖动），避免惊群效应
- 必须设置**最大重试次数**
- 重试操作本身必须**幂等**

### 搜索模式

```bash
# 通用：重试逻辑
grep -rn 'retry\|Retry\|backoff\|Backoff\|attempt\|maxAttempt\|maxRetries' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
# Go
grep -rn 'github.com/cenkalti/backoff\|github.com/sethvargo/go-retry' --include='*.go'
# Java
grep -rn '@Retryable\|RetryTemplate\|spring-retry\|Resilience4j' --include='*.java'
# Python
grep -rn 'tenacity\|@retry\|retrying\|backoff' --include='*.py'

# 无限重试风险
grep -rn 'while True\|for {' --include='*.go' --include='*.py'
```

### 检查清单

- [ ] 重试前是否区分可重试错误和不可重试错误
- [ ] 是否有指数退避策略
- [ ] 是否有最大重试次数限制
- [ ] 重试的接口是否幂等
- [ ] 重试间隔是否合理

### 深挖方法

1. **找到所有重试逻辑**
2. **分析重试条件**：是否只重试可重试的错误
3. **检查退避策略**
4. **给出重试框架代码示例**

---

## #21 异步与消息队列

### 核心规则

- 消息消费要考虑**顺序性**和**重复投递**，业务侧必须幂等
- 消息队列要配置**死信队列（DLQ）**，处理消费失败的消息
- 异步任务要保障**最终一致性**，必要时引入对账机制
- 生产者发送失败要有补偿策略，不能静默丢消息
- **常见错误**：消息消费异常直接丢弃，没有进入死信队列，导致数据丢失

### 搜索模式

```bash
# 消息队列框架
grep -rn 'kafka\|Kafka\|rabbitmq\|amqp\|nsq\|NSQ\|rocketmq\|pulsar' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 消费者
grep -rn 'consumer\|Consumer\|subscriber\|Subscriber\|handler.*Message\|OnMessage' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 死信队列
grep -rn 'dead.letter\|DLQ\|dlq\|dead_letter\|retry.queue' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 消息发送
grep -rn 'Publish\|publish\|Produce\|produce\|Send\|\.send(' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 消费者是否幂等处理重复消息
- [ ] 是否配置了死信队列（DLQ）
- [ ] 消息消费异常是否有重试机制
- [ ] 生产者发送失败是否有补偿策略
- [ ] 是否有对账机制保障最终一致性
- [ ] 消息是否有顺序性要求（有则需分区/有序消费）

### 深挖方法

1. **找到所有消息生产者和消费者**
2. **检查消费者是否有幂等处理**
3. **检查是否有 DLQ 配置**
4. **分析消息丢失风险点**
5. **给出对账和补偿方案**

---

## #22 资源释放与协程生命周期

### 核心规则

- 打开的连接、文件句柄、锁必须在 `finally`、`defer` 或 `try-with-resources` 中释放
- Goroutine / 线程 / 协程**必须有退出机制**，不能无限泄漏

**语言特定**：
- Go: `defer`, goroutine 需监听 `ctx.Done()` 或退出 channel
- Java: `try-with-resources`, `finally`, `ExecutorService.shutdown()`
- Python: `with`, `finally`, `contextlib`
- C++: RAII, `std::unique_ptr`

### 搜索模式

```bash
# Go：defer 清理
grep -rn 'defer.*Close\|defer.*Unlock\|defer.*Cancel\|defer.*Body.Close' --include='*.go'
# Go：资源创建但可能未释放
grep -rn 'os.Open\|sql.Open\|net.Dial\|redis.NewClient' --include='*.go'

# Java：try-with-resources / finally
grep -rn 'try\s*(' --include='*.java' | grep -v 'catch'
grep -rn 'finally\s*{' --include='*.java'
# Java：连接池关闭
grep -rn '\.close()\|\.shutdown()\|\.shutdownNow()' --include='*.java'

# Python：with 语句 / context manager
grep -rn 'with open\|with.*as\|__enter__\|__exit__\|contextmanager' --include='*.py'

# Rust：RAII（Drop trait）
grep -rn 'impl Drop\|drop(' --include='*.rs'

# TypeScript：清理
grep -rn '\.dispose()\|\.close()\|\.destroy()\|finally' --include='*.ts'

# C++：RAII
grep -rn 'std::unique_ptr\|std::shared_ptr\|std::lock_guard\|RAII' --include='*.cpp' --include='*.h'

# === goroutine/线程/协程 ===
grep -rn 'go func' --include='*.go'
grep -rn 'new Thread\|@Async\|CompletableFuture' --include='*.java'
grep -rn 'threading\.Thread\|multiprocessing\|asyncio\.create_task' --include='*.py'
grep -rn 'std::thread::spawn\|tokio::spawn' --include='*.rs'
```

### 检查清单

- [ ] 数据库连接是否在 defer/finally/with 中关闭
- [ ] 文件句柄是否正确关闭
- [ ] 锁是否在 defer/finally/with 中释放
- [ ] goroutine/线程/协程是否有退出机制
- [ ] 是否有资源只在 error 路径上泄漏

### 深挖方法

1. **对每个资源创建点**，追踪对应的释放点
2. **检查是否有 error 路径跳过了释放**
3. **对 goroutine/线程**，追踪退出条件
4. **给出对应语言的资源管理模式**
