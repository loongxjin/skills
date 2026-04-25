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

### 搜索关键词

```
ORM/查询:
  Go: .Find, .First, .Where, .Raw, .Exec, db.Query, db.Exec
  Java: @Select, @Insert, @Update, @Delete, executeQuery, jdbcTemplate, mapper.
  Python: session.query, .filter, .objects, .raw, cursor.execute, .all()
  Rust: query, execute, fetch, sqlx::, diesel::
  TypeScript: .findMany, .findUnique, .raw, .query, createQueryBuilder, knex(

N+1 信号: for/forEach/map 循环内出现 find/query/select/where/fetch/get

缺 LIMIT 信号: Find/SELECT/query/.all() 结果中无 limit/Limit/page/size/offset

批量操作:
  Go: INSERT INTO ... VALUES, CreateInBatches, Create + []
  Java: saveAll, batchInsert, executeBatch, @BatchInsert
  Python: bulk_create, bulk_update, execute_many, insert_many
  TypeScript: createMany, updateMany, deleteMany

全表操作: DELETE FROM / UPDATE ... SET / TRUNCATE 无 WHERE 条件
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

**N+1查询**：循环100次触发101次查询，每次平均 2ms → 约 200ms。数据量翻10倍 → 1000次/2秒，用户体感从"正常"变成"明显卡顿"。如果数据库连接池只有 50 个连接，高并发下会怎样？

**批量操作**：单次 INSERT 50000 条记录，`max_allowed_packet`（默认 4MB）能承受吗？内存中构建 SQL 的峰值占用是多少？

**缺 LIMIT**：如果有人不传过滤参数，一次性加载几十万行到内存？返回给前端的数据量是否做了截断？

> **深挖信号**：当发现 N+1 或无 LIMIT 查询时，必须联动 #10（数据增长敏感性）——当前"还能接受"的性能，增长10倍后会变成什么？

---

## 8. 并发与分布式安全

### 搜索关键词

```
锁:
  Go: sync.Mutex, sync.RWMutex, Lock(), RLock(), Unlock(), RUnlock(), sync.Map
  Java: synchronized, ReentrantLock, AtomicInteger, AtomicLong, ConcurrentHashMap, volatile
  Python: threading.Lock, multiprocessing, asyncio, async def, await, concurrent.futures
  Rust: Mutex::new, RwLock::new, Arc::new, mpsc::, spawn, tokio::spawn

Goroutine/线程:
  Go: go func, go <func>()
  Java: ThreadPool, ExecutorService, CompletableFuture, @Async
  Rust: std::thread
  TypeScript: Promise, async, await, Worker, cluster

共享状态:
  Go: var <name> + map/slice/[]
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

**并发推演**：两个请求同时执行到"先查后改"代码：线程 A 读到余额为 100，线程 B 也读到 100，A 扣减后写入 80，B 扣减后也写入 80——本应是 60 的余额变成了 80。

**锁范围推演**：锁住的范围包含 HTTP 调用？一次 RPC 耗时 3 秒，锁被持有 3 秒，其他所有请求排队。100个并发 → QPS 约为 0.33。

**分布式推演**：3个实例各依赖内存变量/本地缓存做互斥，两个实例上的线程各自执行 `if !exists then set`，都能通过检查。需要分布式锁还是数据库原子操作？

> **深挖信号**：发现"先查后改"时，联动 #7（数据库性能）检查 TOCTOU——即使有索引优化，查询和更新之间的间隙仍可能被并发请求穿透。

---

## 9. 事务边界

### 搜索关键词

```
事务:
  Go: Begin, Transaction, tx., Commit, Rollback, WithTransaction
  Java: @Transactional, TransactionTemplate, PlatformTransactionManager
  Python: transaction.atomic, @transaction.atomic, session.begin, commit, rollback
  Rust: transaction, begin, commit, rollback
  TypeScript: $transaction, .transaction, START TRANSACTION, BEGIN

事务内的 IO（不应在事务中）:
  Go: http.Get, http.Post, grpc., rpc., .Publish, .Send, .Call
  Java: RestTemplate, WebClient, FeignClient, KafkaTemplate, RabbitTemplate
  Python: requests., aiohttp., celery., broker.
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
4. **追踪错误路径**：事务内第2步失败，第1步的 DB 操作会回滚吗？Rollback 代码在哪？
5. 给出重构后的事务边界

### 场景推演

**失败推演**：事务执行到第 3 步失败，前 2 步回滚了吗？如果第 3 步是 MQ 消息已发送，数据库回滚后消息还在队列里——消费者基于"已回滚"数据执行逻辑。有没有补偿机制？

**IO在事务中**：事务内 HTTP 调用，对方慢（30s超时），事务持有连接和行锁 30 秒。其他操作同一行的请求被阻塞，数据库连接池耗尽，服务雪崩。

**嵌套事务**：方法 A 开事务，内部调方法 B 也开事务。内层 Rollback → Spring REQUIRED 传播下整个事务标记为 rollback-only，外层 Commit 抛 `UnexpectedRollbackException`——处理了吗？

> **深挖信号**：事务中有外部调用时，联动限流/重试维度——外部调用失败→事务回滚→重试→部分操作重复执行（如消息再次发送）。

---

## 10. 数据增长敏感性

### 搜索关键词

```
可增长集合: append(, push(, .add(, insert(, map[], Map<, dict(, HashMap
数值类型: int32, int16, int, uint, Integer, Long, float, Float
内存排序/聚合: sort., Sort, ORDER BY, GroupBy, GROUP BY, Count, count(
嵌套循环信号: for...range/for...in/for...each 内嵌套 for...range/for...in/for...each
全量导出: export, Export, download, Download, csv, Excel, xlsx
```

### 检查清单

#### 内存维度
- [ ] 是否有把全表/大量数据加载到内存的操作（当前量小没问题，增长后 OOM）
- [ ] 集合是否会随业务增长无限膨胀，是否有清理/淘汰机制
- [ ] 是否在内存中做了本该由数据库完成的排序/聚合
- [ ] 导出/下载功能是否一次性加载全部数据（应流式输出）

#### 数值维度
- [ ] 自增ID、计数器等数值类型是否够用（int → int64 溢出？）
- [ ] 金额字段是否使用了 float（精度丢失 + 金额增长后精度放大）
- [ ] 超时时间、重试次数等是否硬编码了不合理的固定值

#### 查询性能维度
- [ ] 分页查询是否使用 cursor 而非 offset（offset 大数据量越往后越慢）
- [ ] 是否存在嵌套循环查询 O(n*m)
- [ ] JOIN 的表增长后是否还能在合理时间返回
- [ ] 是否有索引覆盖不了的模糊查询（LIKE '%xxx%' 全表扫描）

#### 存储维度
- [ ] 日志表/记录表是否会无限增长（是否需要归档/清理策略）
- [ ] 文件/附件存储是否会占满磁盘
- [ ] 消息队列积压：消费速度跟不上生产速度会怎样

#### 算法维度
- [ ] 是否有 O(n²) 或更高复杂度的操作
- [ ] 是否可以用哈希表优化查找（O(n) → O(1)）
- [ ] 是否可以用索引/排序+二分优化范围查询
- [ ] 是否有重复计算可以缓存结果

### 深挖方法

1. **估算增长速度**：这个表/集合每月增长多少？一年后？三年后？
2. **推演崩溃场景**：10倍后怎样？100倍后呢？是否存在崩溃阈值（OOM、磁盘满、连接池耗尽）？
3. **分析根因**：查询慢？→ 索引/分库分表；内存太多？→ 流式/分批/下推DB；算法太慢？→ 换算法；单表太大？→ 分表/归档
4. **给出方案**：offset → cursor-based；全量 → 流式/分批；O(n²) → HashMap O(n)；int → int64；float → decimal

### 场景推演

**增长推演**：核心表月增 10 万行，一年 120 万，三年 360 万。当前查询 50ms，360万行时如果 status 字段只有 3 种值（选择性低），会退化为全表扫描吗？

**OOM推演**：查询全量加载到 List，当前 1000 条×2KB=2MB。增长100倍→200MB。5个用户并发导出→1GB。JVM/Go runtime 内存上限是多少？

**算法推演**：O(n²) 嵌套循环，n=100 时 10ms/10000次操作，n=10000 时 1亿次操作/约100秒。用 HashMap 把内层 O(n) 降到 O(1)？

> **深挖信号**：发现全量加载或 O(n²) 时，联动 #11（长时间运行稳定性）——数据持续增长无弹性扩容，最终 OOM 或超时"突然"崩溃。

---

## 11. 长时间运行稳定性

### 搜索关键词

```
定时任务:
  Go: time.Ticker, time.Sleep, cron, Schedule, ticker, interval
  Java: @Scheduled, ScheduledExecutorService, Timer, cron
  Python: schedule, cron, celery.beat, APScheduler, @periodic_task
  Rust: tokio::time::interval, std::thread::sleep

连接池:
  Go: sql.Open, SetMaxOpenConns, SetMaxIdleConns, ConnMaxLifetime, redis.NewClient
  Java: HikariCP, DataSource, ConnectionPool, ThreadPoolExecutor, maxPoolSize
  Python: pool, create_engine, pool_size, max_overflow
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
3. **追踪错误路径**：任务 panic 了会怎样？静默失败还是告警？
4. 给出具体配置建议

### 场景推演

**任务堆积**：定时任务每 5 分钟执行，但单次最慢 8 分钟。超过间隔后启动第二个实例并发执行（重复处理），还是跳过调度？并发执行时数据会被重复处理吗？

**连接泄漏**：连接池最大 20，慢查询占 15 个 30 秒。新请求阻塞等待（排队雪崩）还是立即返回超时错误？有没有获取连接超时和监控告警？

**内存增长**：启动 200MB，一周后 2GB——日志或监控能看到增长吗？有没有只 append 不清理的切片/列表？有没有永不回收的全局变量？长时间运行的 goroutine 有退出机制吗？

> **深挖信号**：缓存无淘汰策略或连接池无上限时，联动 #10（数据增长）——缓存条目无限膨胀、连接池高并发耗尽，"温水煮青蛙"，流量突增时瞬间崩溃。

---

## 12. 可观测性（日志）

### 搜索关键词

```
日志框架:
  Go: log., logger., slog., Infof, Errorf, Warnf, Debugf, printf, Println
  Java: log., logger., Logger, log.info, log.error, log.warn, log.debug, SLF4J
  Python: logging., logger., log., print(
  Rust: info!, error!, warn!, debug!, tracing::, log::
  TypeScript: console., logger., winston, pino, bunyan

错误无日志: if err / catch ( / except / Err( → return/throw/raise 但无 log/logger

关键业务: pay, charge, deduct, refund, transfer, create.*order, update.*status, delete.*account
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

**故障排查**：凌晨3点用户反馈"支付失败"。日志里搜到的都是 `log.Println("error")`，没有订单号、用户ID、错误原因。怎么定位问题？如果日志只有 `"payment failed"` 没有 requestID，怎么追踪完整链路？

**敏感信息**：`log.Printf("login failed: user=%s password=%s", user, password)` — 密码打到了日志，被 ELK 收集，所有有 ELK 权限的人都能看到。什么级别的安全事件？

**日志风暴**：Error 分支高峰期每秒触发 1000 次，每条 500B → 每秒 500KB → 持续10分钟 300MB。磁盘够吗？有没有限速/采样？

> **深挖信号**：错误路径无日志时，联动资源释放路径检查——`defer Close()` 失败无日志，线上无感知地泄漏连接/文件句柄/内存，直到系统崩溃。

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
