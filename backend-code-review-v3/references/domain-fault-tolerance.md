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

### 搜索关键词

```
写操作（需关注幂等）: Create, Insert, Update, Delete, Pay, Charge, Deduct, Send, Publish
幂等键: idempoten, request_id, reqID, dedup, idempotency
重试场景: retry, callback, webhook, reconcil
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
2. **对每个写操作追问**：调用两次会发生什么？追踪完整数据变化链路
3. 分析是否有幂等键（请求ID、业务唯一键）
4. 给出幂等改造方案：
   - 唯一约束 + INSERT IGNORE / ON CONFLICT
   - Redis: SETNX 去重
   - 状态机：只允许单向状态流转

### 场景推演

- **重放推演**：用户连续点3次支付按钮，扣3次钱吗？第三方支付回调重试了，重复处理吗？扣款接口重试3次，后两次也成功（只是响应慢），用户被扣3次。
- **部分失败**：幂等记录写入成功但业务操作失败，下次重试时幂等记录会拦截掉请求吗？数据永远不一致了？
- **过期幂等**：幂等记录什么时候清理？永远不过期的 SETNX → 同一业务ID永远不能重新操作。
- **消息重投**：MQ at-least-once 投递，同一条消息消费3次，每次 `INSERT INTO orders` 无唯一约束 → 3笔重复订单。

> **深挖信号**：写操作无幂等保护时，检查上游是否可能重试（#18）、MQ 是否重投（#8）、用户是否重复点击。这些组合才是真正的高风险。

---

## 17. 限流、熔断、降级

### 搜索关键词

```
限流: rate.Limit, ratelimit, throttl, RateLimit, limiter, TokenBucket
  Go: golang.org/x/time/rate
  Java: @RateLimiter, Semaphore, Bucket4j, Resilience4j
  Python: ratelimit, throttle, slowapi, django-ratelimit

超时: Timeout, timeout, context.WithTimeout, context.WithDeadline, @Timeout
  Rust: tokio::time::timeout

熔断: circuit, breaker, CircuitBreaker, hystrix, resilience, Resilience4j, sentinel

外部调用（需熔断保护）:
  Go: http.Client, grpc.Dial, redis.NewClient, .Call(
  Java: RestTemplate, WebClient, FeignClient, JedisCluster, RedisTemplate
  Python: requests., aiohttp., httpx., redis.
  Rust: reqwest::, hyper::, tonic::
  TypeScript: axios, fetch(, prisma., knex
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
3. **推演雪崩**：外部调用挂了，对本服务影响？会不会拖垮上游？
4. 评估没有熔断的风险
5. 给出限流/熔断配置建议

### 场景推演

- **级联失败**：下游挂了无熔断 → 线程池耗尽 → 所有请求超时 → 依赖你的上游也挂了 → 雪崩。
- **超时设置**：下游 P99 延迟 2 秒，超时设 3 秒合理吗？设 30 秒呢？如果在事务内调用且无超时，事务持有锁多久？
- **限流绕过**：限流只在网关，内部互调无限流。一个恶意请求通过内部调用放大100倍？
- **无限流攻击**：脚本每秒请求 1000 次，数据库连接池耗尽，所有用户无法访问。
- **无熔断浪费**：下游已挂，每次都调用→等超时→失败→重试。有熔断就直接返回降级结果。

> **深挖信号**：外部调用无超时是最常见的隐性风险。联动 #9（事务边界）——事务内调用无超时=长事务+锁。

---

## 18. 重试策略

### 搜索关键词

```
重试: retry, Retry, backoff, Backoff, attempt, maxAttempt, maxRetries
  Go: github.com/cenkalti/backoff, github.com/sethvargo/go-retry
  Java: @Retryable, RetryTemplate, spring-retry, Resilience4j
  Python: tenacity, @retry, retrying, backoff
无限重试风险: while True, for {
```

### 检查清单
- [ ] 重试前是否区分可重试错误和不可重试错误
- [ ] 是否有指数退避策略
- [ ] 是否有最大重试次数限制
- [ ] 重试的接口是否幂等
- [ ] 重试间隔是否合理

### 深挖方法
1. 找到所有重试逻辑
2. 分析重试条件：只重试可重试错误（网络超时可以，参数错误不行）
3. 检查退避策略（是否有 + 随机抖动）
4. **验证重试的接口是否幂等**（不幂等就不能重试）
5. 给出重试框架代码示例

### 场景推演

- **重试风暴**：超时后重试3次 → 下游承受4倍压力。10个实例同时失败同时重试 → 惊群效应，下游瞬间被打崩。
- **不可重试错误**：参数错误(400)、权限不足(403)也在重试？白白浪费资源并放大问题。
- **重试+不幂等**：扣款接口重试3次，后两次也成功 → 扣了3次钱。

> **深挖信号**：重试和幂等是一体两面。每个重试逻辑都要问：接口幂等吗？（#16）。重试+不幂等=灾难，尤其是支付、扣库存。联动 #17（限流熔断）检查退避策略。

---

## 19. 资源释放与清理

### 搜索关键词

```
资源清理:
  Go: defer.*Close, defer.*Unlock, defer.*Cancel, os.Open, sql.Open, net.Dial
  Java: try(, finally, .close(), .shutdown()
  Python: with open, with.*as, __enter__, __exit__, contextmanager
  Rust: impl Drop, drop(
  TypeScript: .dispose(), .close(), .destroy(), finally

并发原语（退出机制见 #8）:
  Go: go func
  Java: new Thread, @Async, CompletableFuture
  Python: threading.Thread, asyncio.create_task
  Rust: std::thread::spawn, tokio::spawn
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
   - Go: `defer` 是否紧跟资源创建（而非错误检查之后）
   - Java: 是否用了 try-with-resources（而非手动 close）
   - Python: 是否用了 `with` 语句
3. 对 goroutine/线程，追踪退出条件（详见 #8）
4. 给出对应语言的资源管理模式

### 场景推演

- **异常路径**：打开连接后 `if err != nil { return err }` 直接返回，未关闭连接。每次报错泄漏一个。连接池 50 个，报错 50 次后服务无法连接数据库。
- **goroutine泄漏**：`go processOrder(order)` 无退出机制。运行7天→100万个 goroutine→内存 200MB→8GB→OOM。
- **连接池耗尽**：连接借出不还，池子空后新请求阻塞还是报错？超时多久？

> **深挖信号**：资源泄漏最危险的是**累积效应**。一个 goroutine 泄漏不会马上出问题，运行7天后几万个 goroutine 触发 OOM。联动 #11（长时间运行稳定性）检查累积泄漏，联动 #12（日志）检查泄漏是否可观测。

---

## 20. 输入校验

### 搜索关键词

```
参数入口:
  Go: ShouldBind, ShouldBindJSON, c.Query, c.Param, c.PostForm
  Java: @RequestBody, @PathVariable, @RequestParam, @Valid, @Validated
  Python: request.json, request.args, request.form, request.data
  TypeScript: @Body, @Param, @Query, req.body, req.params, req.query

SQL注入: fmt.Sprintf.*SELECT/INSERT/WHERE, String.format.*SELECT, f".*SELECT/INSERT/WHERE

校验: validate, required, @NotNull, @NotBlank, @Size, @Min, @Max, validator
```

### 检查清单
- [ ] 所有外部输入是否经过校验
- [ ] 参数校验是否在最外层完成
- [ ] SQL 是否使用参数化查询
- [ ] 是否有 XSS 防护
- [ ] 数值范围是否校验
- [ ] Java: 是否使用 `@Valid` + Bean Validation
- [ ] Python: 是否使用 Pydantic / Marshmallow 校验

### 深挖方法
1. 找到所有 API 入口点
2. **追踪输入数据从入口到使用的完整路径**：用户输入 → 参数解析 → 业务处理 → 数据存储，路径上是否有校验？
3. 检查是否有绕过入口校验直接调到核心逻辑的路径（如 Service 既被 Controller 调也被 Consumer 调）
4. 对 SQL 拼接，给出参数化查询改造

### 场景推演

- **注入推演**：用户输入 `'; DROP TABLE users;--`，SQL 拼出来什么样？参数化查询覆盖了 100% 的 SQL 吗？
- **越权推演**：用户传入 `userID=123` 但不是自己的 ID，有校验吗？水平越权和垂直越权分别检查了吗？
- **异常输入**：空字符串、10MB 超长字符串、负数、JSON 格式错误、特殊 Unicode → 代码怎样？
- **分页攻击**：`page=-1&size=999999999` → `LIMIT 999999999 OFFSET -1`，数据库怎样？

> **深挖信号**：最容易忽略的是"信任内部调用"。Service 层既被 Controller 调也被 Consumer 调，校验在 Controller 做了但 Consumer 没做。联动 #21（最小权限）检查越权风险。

---

## 21. 最小权限

### 搜索关键词

```
数据库高权限账号: root@, admin@, sysdba, sa@, superuser, postgres://

鉴权:
  Go: middleware + auth/jwt/token/permission/rbac
  Java: @PreAuthorize, @Secured, @RolesAllowed, SecurityConfig
  Python: @login_required, @permission_required, is_authenticated, permission_classes

危险操作: delete.*all, drop, truncate, DELETE FROM, DROP TABLE, TRUNCATE
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

- **数据库账号**：用 root/sa 账号连接数据库？SQL 注入的后果就是 DROP DATABASE。用只读账号能完成的查询，为什么用读写账号？
- **横向越权**：用户A能通过修改请求参数访问用户B的数据吗？每个API是否都校验了"这个资源属于当前用户"？
- **提权**：普通用户能调用管理员接口吗？权限校验在路由层还是业务层？有没有绕过路由层直接调用的可能？
- **无鉴权接口**：`/api/users/:id` 无鉴权，任何人知道 ID 就能删除。爬虫遍历 1-100000 的 ID 呢？

> **深挖信号**：有 JWT 校验≠有权限校验——JWT 只证明"你是谁"，不证明"你能做什么"。检查每个操作是否校验了"当前用户有权限操作这个特定资源"。联动 #20（输入校验）检查篡改参数越权。

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
