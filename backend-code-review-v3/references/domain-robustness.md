# 二、健壮性与性能

## 目录
- [7. 数据库性能与批量操作](#7-数据库性能与批量操作)
- [8. 并发与分布式安全](#8-并发与分布式安全)
- [9. 事务边界](#9-事务边界)
- [10. 数据增长敏感性](#10-数据增长敏感性)
- [11. 长时间运行稳定性](#11-长时间运行稳定性)
- [12. 可观测性（日志）](#12-可观测性日志)

---

## 7. 数据库性能与批量操作

### 搜索模式

```bash
# Go（GORM/标准库）
grep -rn '\.Find\|\.First\|\.Where\|\.Raw\|\.Exec\|db.Query\|db.Exec' --include='*.go'
# Java（MyBatis/JPA/JDBC）
grep -rn '@Select\|@Insert\|@Update\|@Delete\|createQuery\|executeQuery\|jdbcTemplate\|mapper\.' --include='*.java'
# Python（SQLAlchemy/Django ORM）
grep -rn 'session.query\|\.filter\|\.objects\|\.raw\|cursor.execute\|\.all()' --include='*.py'
# Rust（SQLx/Diesel）
grep -rn 'query\|execute\|fetch\|sqlx::\|diesel::' --include='*.rs'
# TypeScript（Prisma/TypeORM/Sequelize/Knex）
grep -rn '\.findMany\|\.findUnique\|\.findFirst\|\.raw\|\.query\|createQueryBuilder\|knex(' --include='*.ts'

# 循环中的查询（N+1 信号）
grep -rn 'for\b\|forEach\|map(' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' -A5 | grep -i 'find\|query\|select\|where\|fetch\|get'

# 缺少 LIMIT 的查询
grep -rn 'Find\|SELECT\|query\|\.all()' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' | grep -iv 'limit\|Limit\|page\|size\|offset'

# 批量操作
grep -rn 'batch\|Batch\|bulk\|Bulk\|BATCH\|BULK' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
# Go
grep -rn 'INSERT INTO.*VALUES\|CreateInBatches\|Create.*\[\]' --include='*.go'
# Java
grep -rn 'saveAll\|batchInsert\|executeBatch\|BatchSqlParameterSource\|@BatchInsert' --include='*.java'
# Python
grep -rn 'bulk_create\|bulk_update\|execute_many\|executemany\|insert_many' --include='*.py'
# TypeScript
grep -rn 'createMany\|updateMany\|deleteMany\|$executeRaw' --include='*.ts'

# 全表操作（无 WHERE 条件）
grep -rn 'DELETE FROM\|UPDATE.*SET\|TRUNCATE' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' | grep -v 'WHERE\|where'

# 循环中处理大量数据
grep -rn 'for.*range.*items\|for.*range.*records\|for.*range.*data\|for.*in.*items\|for item in' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

#### 查询性能
- [ ] WHERE 条件字段是否有索引
- [ ] 主键是否保证唯一性
- [ ] 是否存在 N+1 查询（循环内查询）
- [ ] 大结果集是否使用分页
- [ ] 是否有不必要的 SELECT *
- [ ] JOIN 操作是否合理
- [ ] 是否有慢查询风险

#### 批量操作
- [ ] 批量插入/更新是否分批（每批建议 500-1000 条）
- [ ] 大量删除是否分批（避免长事务锁表）
- [ ] 分批大小是否可配置
- [ ] 是否有失败重试和断点续传机制
- [ ] 大数据量查询是否使用流式处理

### 深挖方法
1. 找到所有 SQL/ORM 查询，分析 WHERE 条件字段是否有索引
2. 对循环中的查询，计算 N+1 影响（循环100次=101次查询）
3. 对批量操作，估算单次操作的最大数据量，计算不分批的内存/CPU/锁持有时间
4. 给出索引创建 SQL 或查询优化方案
5. 给出分批处理代码示例：
   - Go: `LIMIT/OFFSET` 循环
   - Java: `JdbcTemplate.queryForRowSet` 流式
   - Python: `yield_per` / `iterator`
   - TypeScript: `cursor` / `take + skip`

### 场景推演

**N+1查询场景**：假设循环100次触发101次查询，以每次查询平均 2ms 计算，当前耗时约 200ms。数据量翻 10 倍后，循环变为 1000 次，耗时约 2 秒——用户体感会从"正常"变为"明显卡顿"。如果这个接口还被其他服务调用，超时风险如何传导？

**批量操作场景**：这段批量 INSERT 代码，单次操作的最大数据量是多少？如果业务方一次性传入 50000 条记录，数据库的 `max_allowed_packet`（默认 4MB）能承受吗？内存中构建 SQL 语句的峰值占用是多少？

**缺少LIMIT场景**：这个查询在生产环境当前数据量下会返回多少行？如果有人通过接口传入一个超大范围的查询条件（如不传过滤参数），会不会一次性加载几十万行数据到内存？返回给前端的数据量是否做了截断？

> **深挖信号**：当发现 N+1 查询或无 LIMIT 查询时，必须联动 #10（数据增长敏感性）推演——当前"还能接受"的性能，数据增长 10 倍后会变成什么？

> 这个接口循环内查询用户信息，100个订单 = 101次 SQL。如果某次请求有 1000 个订单呢？数据库连接池只有 50 个连接，高并发下会怎样？

---

## 8. 并发与分布式安全

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
1. 画出并发访问路径：哪些线程/协程读/写了同一份数据
2. 对每个共享状态，检查是否有竞态条件
3. 检查锁的获取/释放是否成对（Go: defer Unlock; Java: try-finally; Python: with 语句）
4. **追踪锁的持有范围**：如果锁内包含 IO 操作，计算锁最长持有时间
5. 给出具体的加锁方案或改为 channel/消息传递的建议

### 场景推演

**并发推演**：两个请求同时执行到这段代码，会发生什么？请具体描述竞态条件：线程 A 读到余额为 100，线程 B 也读到 100，A 扣减后写入 80，B 扣减后也写入 80——本应是 60 的余额变成了 80。这个场景在你的代码中以什么形式存在？

**锁的范围推演**：锁住的范围是否包含了网络 IO 操作（如 HTTP 调用、数据库查询）？如果一次 RPC 调用耗时 3 秒，锁就会被持有 3 秒，期间其他所有请求都在排队。这个锁粒度是否合理？

**分布式推演**：如果这个服务部署了 3 个实例，当前依赖内存中的变量/本地缓存做互斥，还有保护作用吗？两个实例上的线程各自执行 `if !exists then set`，是不是都能通过检查？需要引入分布式锁还是改用数据库层面的原子操作？

> **深挖信号**：当发现"先查后改"模式时，必须联动 #7（数据库性能）检查是否存在 TOCTOU——即使有索引优化，查询和更新之间的间隙仍然可能被并发请求穿透。

> **场景1**：100个并发请求同时执行 `GetStock → 判断 → Deduct`，库存=10，每个请求查到的都是10，都判断通过，最终扣成负数。
>
> **场景2**：这段代码加了锁，但锁的范围包含了 HTTP 调用。HTTP 超时 30s，锁最多持有 30s，其他99个协程全部阻塞30s。系统的 QPS 会变成多少？

---

## 9. 事务边界

### 搜索模式

```bash
# Go
grep -rn 'Begin\|Transaction\|tx\.\|Commit\|Rollback\|WithTransaction' --include='*.go'
# Java（Spring）
grep -rn '@Transactional\|TransactionTemplate\|PlatformTransactionManager' --include='*.java'
# Python（Django/SQLAlchemy）
grep -rn 'transaction.atomic\|@transaction.atomic\|session.begin\|commit\|rollback' --include='*.py'
# Rust
grep -rn 'transaction\|begin\|commit\|rollback' --include='*.rs'
# TypeScript（TypeORM/Prisma/Sequelize）
grep -rn '\$transaction\|\.transaction\|START TRANSACTION\|BEGIN' --include='*.ts'

# 事务内的 IO（HTTP/RPC/MQ —— 不应在事务中）
grep -rn 'http.Get\|http.Post\|grpc\.\|rpc\.\|\.Publish\|\.Send\|\.Call' --include='*.go'
grep -rn 'RestTemplate\|WebClient\|FeignClient\|KafkaTemplate\|RabbitTemplate' --include='*.java'
grep -rn 'requests\.\|aiohttp\.\|celery\.\|broker\.' --include='*.py'
```

### 检查清单
- [ ] 需要原子性的操作是否在事务内
- [ ] 事务中是否包含网络调用（HTTP/RPC/MQ）——不应有
- [ ] 事务范围是否最小化
- [ ] 是否有忘记 Commit/Rollback 的路径
- [ ] Java: `@Transactional` 的 propagation 和 isolation 是否合理
- [ ] Python: `atomic` 装饰器范围是否过大

### 深挖方法
1. 找到所有事务块
2. 逐行检查事务内操作，标记每个操作是 DB 还是 IO
3. 如果有非 DB 操作在事务内，分析如何移出
4. **追踪错误路径**：如果事务内第2步失败了，第1步的 DB 操作会回滚吗？Rollback 代码在哪？
5. 给出重构后的事务边界

### 场景推演

**失败推演**：事务执行到第 3 步时失败了，前 2 步的数据库操作回滚了吗？如果第 3 步是发送消息到 MQ 且已经发出去了，数据库回滚后消息还在队列里——消费者会基于一条"已回滚"的数据执行业务逻辑。你有没有补偿机制来处理这种不一致？

**IO在事务中推演**：这段代码在事务里发起了一个 HTTP 调用。如果对方服务响应很慢（比如 30 秒超时），事务会持有数据库连接和行锁 30 秒。这期间其他需要操作同一行数据的请求会怎样？是被阻塞直到超时，还是能正常处理？数据库连接池够不够撑？

**嵌套事务推演**：这里是否存在嵌套事务（方法 A 开了事务，内部调用方法 B 也开了事务）？当内层事务 Rollback 时，外层事务是继续执行还是也回滚？Spring 的默认传播行为是 REQUIRED，内层回滚会标记整个事务为 rollback-only，外层 Commit 时会抛 `UnexpectedRollbackException`——你处理了这个异常吗？

> **深挖信号**：当发现事务中有外部调用时，必须联动限流/重试维度思考——外部调用失败后事务回滚，但如果有重试机制，重试时事务会重新执行，可能导致部分操作重复执行（如已发送的消息再次发送）。

> **场景1**：事务内先 INSERT 订单，再 RPC 调用支付服务。RPC 成功了，但 Commit 前数据库挂了，事务回滚。结果：支付已扣款，但订单不存在。用户钱没了，订单也没了。
>
> **场景2**：事务内包含一个 HTTP 调用，平均耗时 200ms，最慢 5s。事务持有行锁期间，其他需要操作同一行的事务全部阻塞。100个并发请求排队等锁，数据库连接池耗尽，服务雪崩。

---

## 10. 数据增长敏感性

> **核心关注点**：业务数据增长可能造成的系统隐患、崩溃、性能下降，以及是否需要算法优化。
> 审查时要带着"数据量10倍、100倍后会怎样"的思维去审视每一段代码。

### 搜索模式

```bash
# 通用：可能增长的集合
grep -rn 'append(\|push(\|\.add(\|insert(' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
# 通用：map 定义
grep -rn 'map\[.*\]\|Map<\|dict(\|HashMap\|{\w\+:}' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 数值类型
grep -rn 'int32\|int16\|int\b\|uint\b\|Integer\|Long\|float\b\|Float' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'

# 内存中排序/聚合（大数据量下性能杀手）
grep -rn 'sort\.\|Sort\|ORDER BY\|GroupBy\|group by\|GROUP BY\|Count\|count(' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 嵌套循环（O(n²) 信号）
grep -rn 'for.*range\|for.*in\|for.*each' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' -A3 | grep 'for.*range\|for.*in\|for.*each'

# 全量数据导出/下载
grep -rn 'export\|Export\|download\|Download\|csv\|Excel\|xlsx' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

#### 内存维度
- [ ] 是否有把全表/大量数据加载到内存的操作（当前量小没问题，增长后 OOM）
- [ ] 集合（slice/list/map）是否会随业务增长无限膨胀，是否有清理/淘汰机制
- [ ] 是否在内存中做了本该由数据库完成的排序/聚合（数据量大时内存爆炸）
- [ ] 导出/下载功能是否一次性加载全部数据（应流式输出）

#### 数值维度
- [ ] 自增ID、计数器等数值类型是否够用（int → int64 溢出？uint 翻转？）
- [ ] 金额字段是否使用了 float（精度丢失 + 金额增长后精度放大）
- [ ] 超时时间、重试次数等是否硬编码了不合理的固定值

#### 查询性能维度
- [ ] 分页查询是否使用 cursor 而非 offset（offset 在数据量大时越往后越慢）
- [ ] 是否存在嵌套循环查询（O(n) 次查询外层再 O(m) 次查询内层 = O(n*m)）
- [ ] JOIN 的表数据量增长后是否还能在合理时间内返回
- [ ] 是否有索引覆盖不了的模糊查询（LIKE '%xxx%' 全表扫描）

#### 存储维度
- [ ] 日志表/记录表是否会无限增长（是否需要定期归档/清理策略）
- [ ] 文件/附件存储是否会占满磁盘
- [ ] 消息队列积压场景：消费速度跟不上生产速度会怎样

#### 算法维度
- [ ] 是否有 O(n²) 或更高复杂度的操作（嵌套循环、全量对比）
- [ ] 是否可以用哈希表优化查找（O(n) → O(1)）
- [ ] 是否可以用索引/排序+二分优化范围查询
- [ ] 是否有重复计算可以缓存结果

### 深挖方法

对每段涉及数据处理的代码，按以下步骤推演：

1. **估算增长速度**：这个表/集合/文件每月增长多少？一年后多少？三年后多少？

2. **推演崩溃场景**：
   - 数据量 10 倍后，这个查询/操作会怎样？（秒级？分钟级？超时？OOM？）
   - 数据量 100 倍后呢？
   - 是否存在某个阈值会导致系统突然崩溃（如内存耗尽、磁盘写满、连接池耗尽）

3. **分析根因**：
   - 是数据量太大导致查询慢？→ 需要索引优化 / 分库分表 / 读写分离
   - 是内存中处理的数据太多？→ 需要流式处理 / 分批处理 / 下推到数据库
   - 是算法复杂度太高？→ 需要换算法（哈希、二分、布隆过滤器等）
   - 是单表数据量太大？→ 需要分表 / 归档 / 冷热分离

4. **给出具体方案**：
   - 分页优化：offset → cursor-based（基于有序唯一键）
   - 内存控制：全量加载 → 流式处理 / 固定大小 buffer / 分批处理
   - 算法优化：嵌套循环 O(n²) → 哈希表 O(n)、排序+二分 O(n log n)
   - 存储优化：单表 → 按时间/ID分表、冷数据归档、定期清理策略
   - 数值安全：int → int64、float → decimal

### 场景推演

**增长推演**：这个核心业务表每月增长多少行？假设月增 10 万行，一年后 120 万行，三年后 360 万行。当前这条查询在 10 万行时耗时 50ms，在 360 万行时由于失去了索引选择性（如 status 字段只有 3 种值，选择性太低），会不会退化为全表扫描？到那时接口超时了你怎么发现？

**OOM推演**：这段代码把查询结果全量加载到一个 `List` 里。当前每次加载 1000 条，每条 2KB，占用约 2MB。如果数据增长 100 倍到 10 万条，就是 200MB。如果是导出场景同时有 5 个用户并发导出，就是 1GB——会触发 OOM 吗？JVM/Go runtime 的内存上限是多少？

**算法推演**：这里的嵌套循环是 O(n²) 复杂度。当 n=100 时耗时 10ms（10000 次操作），当 n 增长到 10000 时，操作次数变成 1 亿次，耗时约 100 秒——用户体验从"瞬间完成"变成"直接超时"。有没有考虑用 HashMap 把内层查找从 O(n) 降到 O(1)？

> **深挖信号**：当发现全量加载或 O(n²) 算法时，必须联动 #11（长时间运行稳定性）——数据持续增长但系统没有弹性扩容或淘汰机制，最终会以 OOM 或超时的方式"突然"崩溃。

> **场景**：这个表每月增长 10 万条，一年后 120 万条，三年后 360 万条。
> - 10倍后（1200万条）：这个 `SELECT * FROM orders WHERE user_id = ?` 还能在 100ms 内返回吗？user_id 有索引吗？
> - 100倍后（1.2亿条）：这个 `int32` 的自增ID还够用吗？（int32 最大约 21 亿）
> - 崩溃阈值：是否存在某个数据量级会导致系统突然崩溃？（OOM、磁盘写满、连接池耗尽）

---

## 11. 长时间运行稳定性

### 搜索模式

```bash
# 定时任务
grep -rn 'time.Ticker\|time.Sleep\|cron\|Schedule\|ticker\|interval' --include='*.go'
grep -rn '@Scheduled\|ScheduledExecutorService\|Timer\|cron' --include='*.java'
grep -rn 'schedule\|cron\|celery.beat\|APScheduler\|@periodic_task' --include='*.py'
grep -rn 'tokio::time::interval\|std::thread::sleep\|cron' --include='*.rs'

# 连接池
grep -rn 'sql.Open\|SetMaxOpenConns\|SetMaxIdleConns\|ConnMaxLifetime\|redis.NewClient' --include='*.go'
grep -rn 'HikariCP\|DataSource\|ConnectionPool\|ThreadPoolExecutor\|maxPoolSize' --include='*.java'
grep -rn 'pool\|Pool\|create_engine\|pool_size\|max_overflow' --include='*.py'

# 注意：缓存相关搜索在 domain-data-ops.md #22 缓存一致性
```

### 检查清单
- [ ] 定时任务执行时间是否可能超过间隔（任务堆积）
- [ ] 缓存是否有淘汰策略（LRU/TTL/最大容量）—— 详见 #22
- [ ] 连接池是否配置了合理的最大连接数和超时
- [ ] 是否有内存泄漏风险
- [ ] 长期运行的进程是否有 panic/exception recovery

### 深挖方法
1. 对定时任务：计算单次执行最长时间，对比间隔
2. 对连接池：检查是否配置了 maxSize、maxIdle、lifetime
3. **追踪错误路径**：任务 panic 了会怎样？是静默失败还是告警？
4. 给出具体配置建议

### 场景推演

**任务堆积推演**：这个定时任务配置了每 5 分钟执行一次，但单次执行最长时间是多少？如果某次执行因为数据量突增而耗时 8 分钟，超过了间隔时间，系统是会启动第二个实例并发执行（导致重复处理），还是会跳过本次调度？并发执行时数据会被重复处理吗？

**连接泄漏推演**：如果数据库连接池的最大连接数是 20，而某个慢查询占住了 15 个连接长达 30 秒，新来的请求需要获取连接时会怎样？是阻塞等待（导致请求排队雪崩），还是立即返回超时错误（用户体验差）？连接池有没有配置获取连接的超时时间和监控告警？

**内存增长推演**：这个服务启动时内存占用 200MB，运行一周后增长到了 2GB——你能在日志或监控中看到内存的持续增长吗？代码中有没有不断 `append` 到切片/列表但从不清理的数据结构？有没有只在初始化时创建但永远不会被回收的全局变量？长时间运行的 goroutine/thread 是否有退出机制？

> **深挖信号**：当发现缓存无淘汰策略或连接池无上限时，必须联动 #10（数据增长敏感性）——缓存条目随业务增长无限膨胀，连接池在高并发下耗尽，这些都是"温水煮青蛙"式的问题，平时看不出，流量突增时瞬间崩溃。

> **场景1**：这个定时任务每分钟执行一次，但单次执行最慢要 3 分钟。任务会堆积吗？如果上一个还没跑完，下一个又启动了，会怎样？数据会被重复处理吗？
>
> **场景2**：这个 Redis 连接池没配 `ConnMaxLifetime`。Redis 服务端 30 分钟断开空闲连接，但客户端不知道，还在用已断开的连接。会怎样？

---

## 12. 可观测性（日志）

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

# 注意：链路追踪（traceID）相关搜索在 domain-data-ops.md #24
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
4. **追踪错误路径**：错误发生后，下游调用方能从日志追踪到根因吗？
5. 给出具体的日志补充建议

### 场景推演

**故障排查推演**：假设这个支付功能在线上出了问题——用户说"扣了钱但订单没创建"。你打开日志，能看到什么？能追踪到这个用户这笔交易从请求进入到最终结果的完整链路吗？日志里有 requestID 可以串联上下游吗？如果日志只有 `"payment failed"` 没有订单号和用户ID，你怎么定位是哪笔交易出了问题？

**敏感信息推演**：这段日志代码会打印完整的请求参数吗？请求体中是否包含密码、token、身份证号、银行卡号？谁有权限查看这些日志（开发、运维、外包）？如果日志被采集到了 ELK/Splunk 等集中式平台，敏感信息是否已经脱敏？

**日志风暴推演**：如果这个 Error 分支在流量高峰期每秒触发 1000 次（比如下游服务超时），日志系统扛得住吗？每条日志 500 字节，每秒写入 500KB，一分钟 30MB——如果持续 10 分钟，磁盘空间够吗？有没有对高频日志做限速（rate limiting）或采样（sampling）？WARN 级别的日志会不会因为太频繁而被直接忽略？

> **深挖信号**：当发现错误处理路径没有日志时，必须联动资源释放路径检查——如果 `defer Close()` 或 `finally` 块中的资源释放失败但没有日志，你会在线上完全无感知地泄漏连接、文件句柄或内存，直到系统崩溃才被动发现。

> **场景1**：凌晨3点用户反馈"支付失败"。你打开日志，搜索到的都是 `log.Println("error")`，没有任何订单号、用户ID、错误原因。你怎么定位问题？
>
> **场景2**：生产环境出了 bug，日志里有 `log.Printf("login failed: user=%s password=%s", user, password)`。密码被打到日志里了，日志被 ELK 收集，所有有 ELK 权限的人都能看到。这是一个什么级别的安全事件？

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| 循环内查询（N+1） | 是否改为批量查询 | ❌ N+1 查询 |
| `SELECT *` | 是否只查需要的字段 | ⚠️ 查询过多数据 |
| 无分页的列表查询 | 是否有 LIMIT/OFFSET 或 cursor | ❌ 潜在全表扫描 |
| 大批量操作 | 是否分批处理（每批 500-1000） | ⚠️ 长事务风险 |
| `sync.Mutex` / `Lock()` | 锁粒度是否合理，是否包含 IO | ⚠️ 锁粒度过大 |
| `go func` | 是否有退出机制 | ❌ goroutine 泄漏 |
| 共享 map/slice 并发写入 | 是否有锁保护 | ❌ 数据竞态 |
| 先查后改（Get + Update） | 是否在同一锁/事务内 | ❌ TOCTOU |
| `Begin` / `@Transactional` | 事务内是否有 IO（HTTP/RPC/MQ） | ❌ 长事务 |
| `Commit` | 是否有对应的 `Rollback` 在错误路径 | ❌ 事务泄漏 |
| `int32` 自增ID | 是否可能溢出 | ⚠️ 溢出风险 |
| `float` 金额计算 | 是否使用 decimal/整数分 | ❌ 精度丢失 |
| 嵌套循环 | 时间复杂度是否可接受 | ⚠️ 性能瓶颈 |
| 全量数据加载到内存 | 是否有分页/流式处理 | ❌ OOM 风险 |
| `if err != nil` | 是否有日志记录错误 | ⚠️ 静默吞错 |
| 关键业务操作（支付/退款） | 是否有日志含业务ID | ⚠️ 不可追踪 |
| 日志输出 | 是否包含敏感信息（密码/token） | ❌ 信息泄露 |
| 定时任务 | 是否可能堆积（执行时间 > 间隔） | ⚠️ 任务堆积 |
| 连接池配置 | 是否配了 maxSize/maxIdle/lifetime | ⚠️ 连接泄漏 |
