# 数据与存储审查指南（#11-#16）

## #11 数据库索引与设计规范

### 核心规则

- 核心查询条件、关联字段、排序字段必须加索引
- 索引不是越多越好，写操作（INSERT/UPDATE/DELETE）会因索引变慢
- 联合索引遵循**最左前缀原则**
- 大表加索引使用 Online DDL 工具（pt-online-schema-change、gh-ost），避免锁表
- 每个表必须有主键，主键尽量选择单调递增整数，避免 UUID 导致页分裂
- 用 `EXPLAIN` 分析慢查询执行计划

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

- [ ] WHERE 条件字段是否有索引
- [ ] 主键是否保证唯一性
- [ ] 是否存在 N+1 查询（循环内查询）
- [ ] 大结果集是否使用分页
- [ ] 是否有不必要的 SELECT *
- [ ] JOIN 操作是否合理
- [ ] 是否有慢查询风险
- [ ] 批量插入/更新是否分批（每批建议 500-1000 条）
- [ ] 大量删除是否分批（避免长事务锁表）
- [ ] 是否有失败重试和断点续传机制
- [ ] 大数据量查询是否使用流式处理

### 深挖方法

1. 找到所有 SQL/ORM 查询，分析 WHERE 条件字段是否有索引
2. 对循环中的查询，计算 N+1 影响（循环100次=101次查询）
3. 对批量操作，估算单次操作的最大数据量，计算不分批的内存/CPU/锁持有时间
4. 给出索引创建 SQL 或查询优化方案
5. 给出分批处理代码示例

---

## #12 事务边界与性能

### 核心规则

- 判断操作是否需要在同一事务内完成
- **严禁在事务中执行 IO 操作**（RPC 调用、文件读写、网络请求），事务应只包含数据库操作
- 事务范围越小越好，长事务是锁等待和死锁的元凶
- **常见错误**：为了"代码整洁"把 HTTP 调用包在事务装饰器里

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

## #13 大批量操作必须分批

### 核心规则

- 批量插入、更新、查询、删除都要分批处理
- 避免一次操作数据过大导致锁表、OOM、网关超时
- 删除大量数据尤其要分批，避免长事务锁住整张表
- 批量大小通常 500~2000 条，根据实际数据量和系统负载调整

### 搜索模式

```bash
# 循环中的单条操作（应改为批量）
grep -rn 'for.*range.*{[^}]*INSERT\|for.*range.*{[^}]*UPDATE\|for.*range.*{[^}]*DELETE' --include='*.go'
grep -rn 'for.*in.*{[^}]*session.add\|for.*{[^}]*save\(' --include='*.java' --include='*.py'

# 批量操作是否分批
grep -rn 'batch\|Batch\|bulk\|Bulk\|BATCH\|BULK\|chunk\|Chunk' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 大量数据的 DELETE/UPDATE（无 LIMIT 的风险信号）
grep -rn 'DELETE FROM\|UPDATE.*SET' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' | grep -v 'LIMIT\|limit\|WHERE.*LIMIT\|batch\|chunk'

# 一次性加载全部数据的查询
grep -rn '\.Find(\|\.all()\|SELECT \*\|fetchall(\|\.findMany(' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' | grep -iv 'limit\|where\|filter'
```

### 检查清单

- [ ] 批量插入是否使用批量接口而非循环单条插入
- [ ] 批量操作是否按 500~2000 条分批执行
- [ ] 大量删除是否分批 + 使用 LIMIT（避免长事务锁表）
- [ ] 批量操作是否有失败重试和断点续传机制
- [ ] 循环中的数据库操作是否可以提取为批量操作
- [ ] 批量查询结果是否一次性全部加载到内存（应考虑流式/分页）
- [ ] 是否有单次操作数据量过大的风险（OOM、锁表、网关超时）

### 深挖方法

1. 找到所有循环中的单条数据库操作，计算循环次数 × 单次耗时 = 总耗时
2. 找到所有批量操作，检查是否有分批逻辑（chunk/batch）
3. 对无分批的批量操作，估算最大数据量和对应资源消耗
4. 给出分批处理代码示例，包括失败重试和进度跟踪

---

## #14 数据增长与冷热分离

### 核心规则

- 设计阶段考虑数据量扩大 100 倍会怎样
- ID 是否会溢出？计数器是否会超界？磁盘/内存是否会打满？
- 历史数据要有**归档、分表、TTL 清理**策略
- 热数据放高速存储，冷数据下沉到低成本存储

### 搜索模式

```bash
# 通用：可能增长的集合
grep -rn 'append(\|push(\|\.add(\|insert(' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
# 通用：map 定义
grep -rn 'map\[.*\]\|Map<\|dict(\|HashMap\|{\w\+:}' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 数值类型
grep -rn 'int32\|int16\|int\b\|uint\b\|Integer\|Long\|float\b\|Float' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'

# 内存中排序/聚合
grep -rn 'sort\.\|Sort\|ORDER BY\|GroupBy\|group by\|GROUP BY\|Count\|count(' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 嵌套循环（O(n²) 信号）
grep -rn 'for.*range\|for.*in\|for.*each' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' -A3 | grep 'for.*range\|for.*in\|for.*each'

# 全量数据导出/下载
grep -rn 'export\|Export\|download\|Download\|csv\|Excel\|xlsx' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

#### 内存维度
- [ ] 是否有把全表/大量数据加载到内存的操作（当前量小没问题，增长后 OOM）
- [ ] 集合是否会随业务增长无限膨胀，是否有清理/淘汰机制
- [ ] 是否在内存中做了本该由数据库完成的排序/聚合
- [ ] 导出/下载功能是否一次性加载全部数据（应流式输出）

#### 数值维度
- [ ] 自增ID、计数器等数值类型是否够用（int → int64 溢出？）
- [ ] 金额字段是否使用了 float（精度丢失）
- [ ] 超时时间、重试次数等是否硬编码了不合理的固定值

#### 查询性能维度
- [ ] 分页查询是否使用 cursor 而非 offset
- [ ] 是否存在嵌套循环查询（O(n\*m)）
- [ ] JOIN 的表数据量增长后是否还能在合理时间内返回
- [ ] 是否有索引覆盖不了的模糊查询（LIKE '%xxx%'）

#### 存储维度
- [ ] 日志表/记录表是否会无限增长
- [ ] 文件/附件存储是否会占满磁盘
- [ ] 消息队列积压场景：消费速度跟不上生产速度会怎样

#### 算法维度
- [ ] 是否有 O(n²) 或更高复杂度的操作
- [ ] 是否可以用哈希表优化查找（O(n) → O(1)）
- [ ] 是否有重复计算可以缓存结果

### 深挖方法

1. 估算增长速度：这个表/集合/文件每月增长多少？一年后多少？三年后多少？
2. 推演崩溃场景：数据量 10 倍后怎样？100 倍后呢？
3. 分析根因：是数据量大？内存处理多？算法复杂度高？单表太大？
4. 给出具体方案：分页优化、内存控制、算法优化、存储优化、数值安全

---

## #15 数据库迁移与 Schema 变更

### 核心规则

- Schema 变更脚本必须**向下兼容**
- 加字段流程：先加 nullable → 部署新代码写入 → 回填历史数据 → 加非空约束
- 禁止直接删字段、改字段类型，要分版本灰度
- 迁移脚本纳入版本控制，具备**幂等性**（重复执行不报错）

### 搜索模式

```bash
# 迁移文件/目录
find . -type d \( -name 'migrations' -o -name 'migration' -o -name 'db/migrate' -o -name 'alembic' -o -name 'flyway' -o -name 'liquibase' \)
find . -type f \( -name '*.sql' -o -name '*_migration*' -o -name '*_migration.go' \)

# DDL 操作
grep -rn 'CREATE TABLE\|ALTER TABLE\|DROP TABLE\|DROP COLUMN\|ADD COLUMN\|MODIFY COLUMN\|CHANGE COLUMN\|RENAME COLUMN' --include='*.sql' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# ORM 自动迁移（生产环境应禁用）
grep -rn 'AutoMigrate\|autoMigrate\|migrate\|sync_schema\|create_all' --include='*.go' --include='*.python' --include='*.ts'

# 字段类型变更
grep -rn 'MODIFY\|CHANGE\|ALTER.*TYPE\|ALTER.*COLUMN.*TYPE' --include='*.sql'

# 删除操作（危险信号）
grep -rn 'DROP TABLE\|DROP COLUMN\|DROP INDEX' --include='*.sql'

# 非 NULL 约束直接添加
grep -rn 'ADD COLUMN.*NOT NULL\|ALTER.*NOT NULL' --include='*.sql' | grep -v 'DEFAULT'

# 锁表风险（大表无 Online DDL）
grep -rn 'ALTER TABLE\|CREATE INDEX\|DROP INDEX' --include='*.sql' | grep -iv 'pt-online\|gh-ost\|ONLINE\|ALGORITHM.*INPLACE\|LOCK.*NONE'
```

### 检查清单

- [ ] 迁移脚本是否纳入版本控制（git）
- [ ] 迁移脚本是否具备幂等性（重复执行不报错）
- [ ] 加字段是否遵循流程：nullable → 部署 → 回填 → NOT NULL
- [ ] 是否存在直接删除字段/改字段类型的操作
- [ ] 是否存在大表直接加索引（无 Online DDL）
- [ ] 生产环境是否禁用了 ORM 自动迁移
- [ ] 迁移脚本是否有回滚方案（rollback）
- [ ] 新旧代码交替部署期间 Schema 是否向下兼容
- [ ] 是否有数据回填脚本及进度跟踪机制

### 深挖方法

1. 找到所有迁移文件，按版本号排序，检查演进链路
2. 对每个 DDL 操作，验证是否向下兼容（旧代码能否正常运行在新 Schema 上）
3. 对加 NOT NULL 约束的操作，检查是否有 DEFAULT 值或分步执行
4. 对大表 DDL 操作，评估锁表时间和对线上服务的影响
5. 检查是否缺少回滚脚本，给出回滚方案

---

## #16 缓存一致性策略

### 核心规则

- 明确缓存与数据库的一致性策略（Cache Aside、Write Through、Write Behind）
- 缓存必须设置**过期时间**和兜底方案
- 高一致性场景：更新数据库后**失效缓存**，而非更新缓存
- 警惕缓存穿透、击穿、雪崩三类经典问题
- **常见错误**：先删缓存再更新 DB（并发下会读到旧值），正确顺序是先更新 DB 再删缓存

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
