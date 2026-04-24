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
2. **对每个写操作追问**：如果调用两次会发生什么？追踪完整的数据变化链路
3. 分析是否有幂等键（请求ID、业务唯一键）
4. 给出幂等改造方案：
   - Go/Java/Python: 唯一约束 + INSERT IGNORE / ON CONFLICT
   - Redis: SETNX 去重
   - 状态机：只允许单向状态流转

### 场景推演

- **重放推演**：用户网不好，连续点了3次支付按钮，会扣3次钱吗？第三方支付回调重试了，会重复处理吗？
- **部分失败推演**：幂等记录写入成功了但业务操作失败了，下次重试时幂等记录会拦截掉这次请求吗？数据是不是永远不一致了？
- **过期幂等推演**：幂等记录什么时候清理？如果用一个永远不过期的 SETNX 做去重，同一个业务ID就永远不能重新操作了

> **深挖信号**：幂等问题往往不是单独出现的。如果一个写操作没有幂等保护，检查它上游是否有可能重试（#18 重试策略）、是否有消息队列重投（domain-robustness #8）、是否有用户重复点击。这些场景组合在一起才是真正的高风险。

### 推演场景

> **场景1**：用户点"支付"，网络超时了，前端自动重试，支付接口被调用两次。如果没有幂等保护，扣了两次钱。用户投诉，客服怎么查？钱怎么退？
>
> **场景2**：消息队列 at-least-once 投递，同一条消息被消费了 3 次。每次都 `INSERT INTO orders`，如果没有唯一约束，就会创建 3 笔重复订单。

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
3. **推演雪崩场景**：如果这个外部调用挂了，会对本服务造成什么影响？会不会拖垮上游？
4. 评估没有熔断的风险
5. 给出限流/熔断配置建议

### 场景推演

- **级联失败推演**：下游服务挂了，没有熔断保护，你的服务会怎样？线程池耗尽？所有请求超时？然后依赖你的上游也挂了？
- **超时推演**：这个外部调用的超时时间是多少？如果下游 P99 延迟是2秒，你设了3秒超时，合理吗？如果设了30秒呢？
- **限流绕过推演**：限流只在 API 网关做了，内部服务之间互调有没有限流？一个恶意请求进来，会不会通过内部调用放大100倍？

> **深挖信号**：外部调用无超时是最常见的隐性风险。找到每个外部调用点后，一定要追问"如果在事务内调用且无超时，事务会持有锁多久？"（结合 #9 事务边界）。级联失败是分布式系统的经典问题，一个没有熔断的服务可以拖垮整条调用链。

### 推演场景

> **场景1**：这个接口没有限流。有人写了个脚本每秒请求 1000 次，服务扛不住，数据库连接池耗尽，所有用户都无法访问。
>
> **场景2**：调用的下游服务挂了，但你的代码没有超时控制，用的是默认超时（可能无限等待）。你的服务线程全部阻塞在等下游响应，你的服务也挂了。下游的下游也挂了。雪崩。
>
> **场景3**：下游已经挂了，你的代码还在每次都尝试调用、等待超时、失败、重试。有没有熔断？如果熔断了，可以直接返回降级结果，而不是每次都浪费时间去等一个注定失败的调用。

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
2. 分析重试条件：是否只重试可重试的错误（网络超时可以，参数错误不行）
3. 检查退避策略（是否有 + 随机抖动）
4. **验证重试的接口是否幂等**（不幂等就不能重试）
5. 给出重试框架代码示例

### 场景推演

- **重试风暴推演**：超时后重试3次，如果下游已经在处理第1次请求，重试会让下游承受4倍压力。下游能承受吗？
- **不可重试错误推演**：参数校验失败（400）、权限不足（403）、业务规则违反（业务错误码）也在重试吗？这会浪费资源并放大问题
- **重试+幂等推演**：重试的接口保证幂等了吗？如果不幂等，重试=重复执行=数据错误

> **深挖信号**：重试和幂等是一体两面。每个重试逻辑都要问：重试的接口幂等吗？（#16 幂等设计）。重试+不幂等=灾难，尤其是支付、扣库存类操作。同时检查重试是否有退避策略，无退避的重试风暴会压垮下游（#17 限流熔断）。

### 推演场景

> **场景1**：这个重试没有区分错误类型。参数错误（400 Bad Request）也重试 3 次，每次都失败。白白浪费 3 次请求，放大了下游压力。
>
> **场景2**：重试没有退避策略，间隔固定 1 秒。10 个实例同时失败、同时重试、又同时失败、又同时重试。下游服务在重试瞬间收到 10 倍流量，直接被打崩（惊群效应）。
>
> **场景3**：重试的接口不是幂等的。扣款接口重试 3 次，如果后两次也成功了（只是响应慢），用户被扣了 3 次钱。

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
2. **重点追踪 error 路径**：检查是否有 error 路径跳过了释放
   - Go: `defer` 是否紧跟在资源创建之后（而不是在错误检查之后）
   - Java: 是否用了 try-with-resources（而非手动 close）
   - Python: 是否用了 `with` 语句
3. 对 goroutine/线程，追踪退出条件（并发问题详细分析见 domain-robustness.md #8）
4. 给出对应语言的资源管理模式：
   - Go: `defer`
   - Java: try-with-resources
   - Python: `with` 语句
   - Rust: RAII / `Drop`

### 场景推演

- **异常路径推演**：正常路径上资源释放了，但异常路径呢？`if err != nil { return err }` 之前，前面的资源都释放了吗？
- **goroutine泄漏推演**：启动了一个 goroutine 处理任务，它什么时候退出？如果任务卡住了，这个 goroutine 会永远存在吗？1000个请求后会有1000个泄漏的 goroutine 吗？
- **连接池耗尽推演**：连接借出去不还，池子会空。当池子空了，新的请求会阻塞还是报错？超时时间是多久？

> **深挖信号**：资源泄漏最危险的不是单次泄漏，而是**累积效应**。一个 goroutine 泄漏不会马上出问题，但运行7天后几万个 goroutine 会让 OOM killer 惩罚你的服务。结合 #11 长时间运行稳定性 检查是否有累积泄漏的风险，结合 #12 日志 检查泄漏是否可观测。

### 推演场景

> **场景1**：这段代码打开了数据库连接，但在第 10 行 `if err != nil { return err }` 直接返回了，没有关闭连接。每次报错都泄漏一个连接。连接池 50 个连接，报错 50 次后，服务就再也连不上数据库了。
>
> **场景2**：`go processOrder(order)` 启动的 goroutine 没有退出机制。服务运行 7 天后，积累了 100 万个 goroutine，内存从 200MB 涨到 8GB，OOM 崩溃。

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
2. **追踪输入数据从入口到使用的完整路径**：用户输入 → 参数解析 → 业务处理 → 数据存储，路径上是否有校验？
3. 检查路径上是否有校验
4. 对 SQL 拼接，给出参数化查询改造

### 场景推演

- **注入推演**：用户输入 `'; DROP TABLE users;--`，SQL拼出来是什么样的？参数化查询覆盖了100%的SQL吗？有没有漏掉的动态SQL？
- **越权推演**：用户传入 `userID=123`，但这个ID不是他自己的，代码里有校验吗？水平越权（访问同级别其他用户数据）和垂直越权（普通用户访问管理员功能）分别检查了吗？
- **异常输入推演**：传入空字符串、超长字符串（10MB）、负数、JSON 格式错误的字符串、特殊的 Unicode 字符，代码会怎样？

> **深挖信号**：输入校验最容易被忽略的是"信任内部调用"。如果 Service 层的函数既被 Controller 调用也被 MQ Consumer 调用，校验在 Controller 做了但 Consumer 没做。追踪输入数据的完整路径，看有没有"绕过入口校验直接调到核心逻辑"的路径。结合 #21 最小权限 检查越权风险。

### 推演场景

> **场景1**：这个接口直接把用户输入拼进了 SQL：`fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)`。如果用户输入 `'; DROP TABLE users; --`，会发生什么？
>
> **场景2**：这个接口没有校验分页参数。用户传了 `page=-1&size=999999999`。SQL 变成 `LIMIT 999999999 OFFSET -1`。数据库会怎样？

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
3. **对无鉴权的接口，推演被恶意利用的场景**
4. 给出权限最小化建议

### 场景推演

- **数据库账号推演**：应用用的是 root/sa 账号连接数据库？那 SQL 注入的后果就是 DROP DATABASE。用只读账号能完成的查询，为什么要用读写账号？
- **横向越权推演**：用户A能通过修改请求参数访问到用户B的数据吗？每个API是否都校验了"这个资源属于当前用户"？
- **提权推演**：普通用户能调用管理员接口吗？权限校验是在路由层做的还是业务层做的？有没有绕过路由层直接调用的可能？

> **深挖信号**：越权问题往往藏在"看起来有鉴权"的代码里。有 JWT 校验≠有权限校验——JWT只证明"你是谁"，不证明"你能做什么"。检查每个操作是否校验了"当前用户有权限操作这个特定资源"。结合 #20 输入校验 检查是否通过篡改参数实现越权。

### 推演场景

> **场景1**：数据库用的是 root 账号。代码里有一个 SQL 注入漏洞，攻击者可以通过注入 `DROP TABLE` 删掉所有表。如果用的是只读账号，影响会怎样？
>
> **场景2**：这个删除用户的接口 `/api/users/:id` 没有鉴权。任何人只要知道用户 ID 就能删除。如果有爬虫遍历 1-100000 的 ID 呢？

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| 写操作（Pay/Charge/Deduct） | 幂等保护（唯一ID/唯一约束/状态机） | ❌ 无幂等 |
| 外部调用（HTTP/RPC） | 超时控制 | ❌ 无超时 |
| 对外暴露的 API | 限流保护 | ⚠️ 无限流 |
| 外部调用（HTTP/RPC） | 熔断机制 | ⚠️ 无熔断 |
| 重试逻辑 | 可重试判断 + 退避策略 + 最大次数 | ❌ 盲目重试 |
| 重试的接口 | 幂等保护 | ❌ 重试非幂等接口 |
| `os.Open` / `sql.Open` / `net.Dial` | `defer Close` / try-with-resources / `with` | ❌ 资源泄漏 |
| `Lock()` | `defer Unlock()` 在 error 路径也释放 | ❌ 锁泄漏 |
| `go func` | 退出机制（ctx.Done / done channel） | ❌ goroutine 泄漏 |
| API 入口点 | 输入校验 | ❌ 无校验 |
| SQL 拼接 `fmt.Sprintf` | 参数化查询 | ❌ SQL 注入 |
| 敏感操作（删除/修改权限） | 鉴权 + 审计日志 | ❌ 未授权访问 |
| 数据库连接 | 非 root/admin 账号 | ❌ 权限过大 |
