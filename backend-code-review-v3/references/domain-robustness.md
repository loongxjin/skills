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
4. 给出具体的加锁方案或改为 channel/消息传递的建议

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
4. 给出重构后的事务边界

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
3. 给出具体配置建议

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
4. 给出具体的日志补充建议
