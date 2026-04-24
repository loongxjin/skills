# 四、容错与安全

## 目录
- [16. 幂等设计](#16-幂等设计)
- [17. 限流、熔断、降级](#17-限流熔断降级)
- [18. 重试策略](#18-重试策略)
- [19. 资源释放与清理](#19-资源释放与清理)
- [20. 输入校验](#20-输入校验)
- [21. 最小权限](#21-最小权限)

---

## 16. 幂等设计

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
1. 列出所有写操作接口
2. 对每个写操作追问：如果调用两次会发生什么？
3. 分析是否有幂等键（请求ID、业务唯一键）
4. 给出幂等改造方案：
   - Go/Java/Python: 唯一约束 + INSERT IGNORE / ON CONFLICT
   - Redis: SETNX 去重
   - 状态机：只允许单向状态流转

---

## 17. 限流、熔断、降级

### 搜索模式

```bash
# 限流
grep -rn 'rate.Limit\|ratelimit\|throttl\|RateLimit\|limiter\|TokenBucket\|RateLimiter' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Go
grep -rn 'golang.org/x/time/rate\|ratelimit\.' --include='*.go'
# Java
grep -rn '@RateLimiter\|RateLimiter\|Semaphore\|Bucket4j\|Resilience4j' --include='*.java'
# Python
grep -rn 'ratelimit\|throttle\|slowapi\|django-ratelimit' --include='*.py'

# 超时控制（所有语言）
grep -rn 'Timeout\|timeout\|context.WithTimeout\|context.WithDeadline\|@Timeout\|@TimeLimiter' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Rust
grep -rn 'timeout\|Duration\|tokio::time::timeout' --include='*.rs'

# 熔断
grep -rn 'circuit\|breaker\|CircuitBreaker\|hystrix\|resilience\|Resilience4j\|sentinel' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 外部调用（需要熔断保护）
grep -rn 'http.Client\|grpc.Dial\|redis\.NewClient\|\.Call(' --include='*.go'
grep -rn 'RestTemplate\|WebClient\|FeignClient\|JedisCluster\|RedisTemplate' --include='*.java'
grep -rn 'requests\.\|aiohttp\.\|httpx\.\|redis\.\|aioredis' --include='*.py'
grep -rn 'reqwest::\|hyper::\|tonic::' --include='*.rs'
grep -rn 'axios\|fetch(\|prisma\.\|knex' --include='*.ts'
```

### 检查清单
- [ ] 对外暴露的接口是否有限流保护
- [ ] 外部调用是否有超时控制
- [ ] 是否有熔断机制
- [ ] 核心链路是否有降级方案
- [ ] 超时时间是否合理

### 深挖方法
1. 找到所有外部调用点
2. 检查每个调用是否有超时控制
3. 评估没有熔断的风险
4. 给出限流/熔断配置建议

---

## 18. 重试策略

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
1. 找到所有重试逻辑
2. 分析重试条件：是否只重试可重试的错误
3. 检查退避策略
4. 给出重试框架代码示例

---

## 19. 资源释放与清理

### 搜索模式

```bash
# Go：defer 清理
grep -rn 'defer.*Close\|defer.*Unlock\|defer.*Cancel\|defer.*Body.Close' --include='*.go'
# Go：资源创建但可能未释放
grep -rn 'os.Open\|sql.Open\|net.Dial\|redis.NewClient' --include='*.go'

# Java：try-with-resources / finally
grep -rn 'try\s*(' --include='*.java' | grep -v 'catch'  # try-with-resources
grep -rn 'finally\s*{' --include='*.java'
# Java：连接池关闭
grep -rn '\.close()\|\.shutdown()\|\.shutdownNow()' --include='*.java'

# Python：with 语句 / context manager
grep -rn 'with open\|with.*as\|__enter__\|__exit__\|contextmanager' --include='*.py'

# Rust：RAII（Drop trait）
grep -rn 'impl Drop\|drop(' --include='*.rs'

# TypeScript：清理
grep -rn '\.dispose()\|\.close()\|\.destroy()\|finally' --include='*.ts'

# 通用：goroutine/线程/协程（并发原语详细搜索见 domain-robustness.md #8）
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
1. 对每个资源创建点，追踪对应的释放点
2. 检查是否有 error 路径跳过了释放
3. 对 goroutine/线程，追踪退出条件（并发问题详细分析见 domain-robustness.md #8）
4. 给出对应语言的资源管理模式：
   - Go: `defer`
   - Java: try-with-resources
   - Python: `with` 语句
   - Rust: RAII / `Drop`

---

## 20. 输入校验

### 搜索模式

```bash
# 通用：参数绑定/解析（入口点）
grep -rn 'Bind\|Parse\|Unmarshal\|GetParam\|QueryParam\|Body\|PathVariable\|RequestParam' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Go
grep -rn 'ShouldBind\|ShouldBindJSON\|c.Query\|c.Param\|c.PostForm' --include='*.go'
# Java
grep -rn '@RequestBody\|@PathVariable\|@RequestParam\|@Valid\|@Validated' --include='*.java'
# Python
grep -rn 'request.json\|request.args\|request.form\|request.data' --include='*.py'
# TypeScript
grep -rn '@Body\|@Param\|@Query\|req.body\|req.params\|req.query' --include='*.ts'

# SQL 注入风险
grep -rn 'fmt.Sprintf.*SELECT\|fmt.Sprintf.*INSERT\|fmt.Sprintf.*WHERE\|string.*SQL\|String.format.*SELECT\|f".*SELECT\|f".*INSERT\|f".*WHERE' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 校验逻辑
grep -rn 'validate\|Validate\|required\|@NotNull\|@NotBlank\|@Size\|@Min\|@Max\|validator' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单
- [ ] 所有外部输入是否经过校验
- [ ] 参数校验是否在最外层完成
- [ ] SQL 是否使用参数化查询
- [ ] 是否有 XSS 防护
- [ ] 数值范围是否校验
- [ ] Java: 是否使用 `@Valid` / `@Validated` + Bean Validation
- [ ] Python: 是否使用 Pydantic / Marshmallow 校验

### 深挖方法
1. 找到所有 API 入口点
2. 追踪输入数据从入口到使用的完整路径
3. 检查路径上是否有校验
4. 对 SQL 拼接，给出参数化查询改造

---

## 21. 最小权限

### 搜索模式

```bash
# 数据库连接配置：root/admin 账号
grep -rn 'root@\|admin@\|sysdba\|sa@\|superuser\|postgres://' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' --include='*.yaml' --include='*.yml' --include='*.env' --include='*.toml'

# 鉴权中间件
grep -rn 'auth\|Auth\|jwt\|JWT\|token\|Token\|permission\|Permission\|rbac\|RBAC' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Go
grep -rn 'middleware\|Middleware' --include='*.go' | grep -i auth
# Java
grep -rn '@PreAuthorize\|@Secured\|@RolesAllowed\|SecurityConfig\|WebSecurityConfigurerAdapter' --include='*.java'
# Python
grep -rn '@login_required\|@permission_required\|is_authenticated\|permission_classes' --include='*.py'

# 危险操作
grep -rn 'delete.*all\|drop\|truncate\|DELETE FROM\|DROP TABLE\|TRUNCATE' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单
- [ ] 数据库连接是否使用最小权限账号
- [ ] API 是否有鉴权中间件
- [ ] 敏感操作是否有二次确认
- [ ] 文件系统访问是否限制在必要目录
- [ ] 是否有操作审计日志

### 深挖方法
1. 检查数据库连接字符串中的用户名
2. 追踪 API 鉴权链路
3. 对无鉴权的接口，评估风险
4. 给出权限最小化建议
