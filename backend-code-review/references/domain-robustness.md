# II. Robustness & Performance

## Table of Contents
- [7. Database Performance & Batch Operations](#7-database-performance--batch-operations)
- [8. Concurrency & Distributed Safety](#8-concurrency--distributed-safety)
- [9. Transaction Boundaries](#9-transaction-boundaries)
- [10. Data Growth Sensitivity](#10-data-growth-sensitivity)
- [11. Long-Running Stability](#11-long-running-stability)
- [12. Observability (Logging)](#12-observability-logging)

---

## 7. Database Performance & Batch Operations

### Search Keywords

```
ORM/Queries:
  Go: .Find, .First, .Where, .Raw, .Exec, db.Query, db.Exec
  Java: @Select, @Insert, @Update, @Delete, executeQuery, jdbcTemplate, mapper.
  Python: session.query, .filter, .objects, .raw, cursor.execute, .all()
  Rust: query, execute, fetch, sqlx::, diesel::
  TypeScript: .findMany, .findUnique, .raw, .query, createQueryBuilder, knex(

N+1 signal: find/query/select/where/fetch/get appearing inside for/forEach/map loops

Missing LIMIT signal: Find/SELECT/query/.all() results without limit/Limit/page/size/offset

Batch operations:
  Go: INSERT INTO ... VALUES, CreateInBatches, Create + []
  Java: saveAll, batchInsert, executeBatch, @BatchInsert
  Python: bulk_create, bulk_update, execute_many, insert_many
  TypeScript: createMany, updateMany, deleteMany

Full table operations: DELETE FROM / UPDATE ... SET / TRUNCATE without WHERE clause
```

### Checklist

#### Query Performance
- [ ] Do WHERE condition fields have indexes
- [ ] Does primary key guarantee uniqueness
- [ ] Is there N+1 query (query inside loop)
- [ ] Does large result set use pagination
- [ ] Are there unnecessary SELECT *
- [ ] Are JOIN operations reasonable
- [ ] Is there slow query risk

#### Batch Operations
- [ ] Are batch insert/update split into batches (suggest 500-1000 per batch)
- [ ] Are large deletions batched (avoid long transaction table locks)
- [ ] Is batch size configurable
- [ ] Are there failure retry and checkpoint/resume mechanisms
- [ ] Do large data queries use streaming

### Deep Dive Method
1. Find all SQL/ORM queries, analyze if WHERE condition fields have indexes
2. For queries inside loops, calculate N+1 impact (100 loop iterations = 101 queries)
3. For batch operations, estimate maximum single-operation data volume, calculate memory/CPU/lock hold time without batching
4. Give index creation SQL or query optimization plan
5. Give batch processing code examples:
   - Go: `LIMIT/OFFSET` loop
   - Java: `JdbcTemplate.queryForRowSet` streaming
   - Python: `yield_per` / `iterator`
   - TypeScript: `cursor` / `take + skip`

### Scenario Simulation

**N+1 Query**: 100 loop iterations trigger 101 queries, each averaging 2ms → ~200ms. Data volume increases 10x → 1000 queries / 2 seconds, user perception changes from "normal" to "obviously laggy". If DB connection pool only has 50 connections, what happens under high concurrency?

**Batch Operations**: Single INSERT of 50000 records, can `max_allowed_packet` (default 4MB) handle it? What's the peak memory usage for building SQL in memory?

**Missing LIMIT**: What if someone doesn't pass filter params and loads hundreds of thousands of rows into memory at once? Is returned data to frontend truncated?

> **Deep dive signal**: When N+1 or no-LIMIT query is found, must link to #10 (Data Growth Sensitivity) — performance that's currently "acceptable", what happens after 10x growth?

---

## 8. Concurrency & Distributed Safety

### Search Keywords

```
Locks:
  Go: sync.Mutex, sync.RWMutex, Lock(), RLock(), Unlock(), RUnlock(), sync.Map
  Java: synchronized, ReentrantLock, AtomicInteger, AtomicLong, ConcurrentHashMap, volatile
  Python: threading.Lock, multiprocessing, asyncio, async def, await, concurrent.futures
  Rust: Mutex::new, RwLock::new, Arc::new, mpsc::, spawn, tokio::spawn

Goroutine/Threads:
  Go: go func, go <func>()
  Java: ThreadPool, ExecutorService, CompletableFuture, @Async
  Rust: std::thread
  TypeScript: Promise, async, await, Worker, cluster

Shared state:
  Go: var <name> + map/slice/[]
```

### Checklist
- [ ] Is shared mutable state protected by locks
- [ ] Is lock granularity reasonable (only lock critical section)
- [ ] Is there TOCTOU issue (Time of Check to Time of Use)
- [ ] Are query + update within the same lock
- [ ] Do Goroutines/Threads/Coroutines have clear exit mechanisms
- [ ] Is there leak risk
- [ ] Does distributed scenario need distributed locks

### Deep Dive Method
1. Draw concurrent access paths: which threads/goroutines read/write the same data
2. For each shared state, check if there's race condition
3. Check if lock acquire/release is paired (Go: defer Unlock; Java: try-finally; Python: with statement)
4. **Trace lock hold scope**: If lock contains IO operations, calculate maximum lock hold time
5. Give specific locking solutions or suggestions to change to channel/message passing

### Scenario Simulation

**Concurrency simulation**: Two requests simultaneously execute "query then update" code: Thread A reads balance as 100, Thread B also reads 100, A deducts and writes 80, B deducts and also writes 80 — balance that should be 60 becomes 80.

**Lock scope simulation**: Lock scope contains HTTP call? One RPC takes 3 seconds, lock held for 3 seconds, all other requests queue. 100 concurrent → QPS ~ 0.33.

**Distributed simulation**: 3 instances each rely on memory variables/local cache for mutual exclusion, threads on two instances each execute `if !exists then set`, both pass the check. Need distributed lock or database atomic operation?

> **Deep dive signal**: When "query then update" is found, link to #7 (Database Performance) to check TOCTOU — even with index optimization, the gap between query and update can still be penetrated by concurrent requests.

---

## 9. Transaction Boundaries

### Search Keywords

```
Transactions:
  Go: Begin, Transaction, tx., Commit, Rollback, WithTransaction
  Java: @Transactional, TransactionTemplate, PlatformTransactionManager
  Python: transaction.atomic, @transaction.atomic, session.begin, commit, rollback
  Rust: transaction, begin, commit, rollback
  TypeScript: $transaction, .transaction, START TRANSACTION, BEGIN

IO inside transactions (should not be):
  Go: http.Get, http.Post, grpc., rpc., .Publish, .Send, .Call
  Java: RestTemplate, WebClient, FeignClient, KafkaTemplate, RabbitTemplate
  Python: requests., aiohttp., celery., broker.
```

### Checklist
- [ ] Are operations needing atomicity within transactions
- [ ] Does transaction contain network calls (HTTP/RPC/MQ) — should not
- [ ] Is transaction scope minimized
- [ ] Are there paths that forget Commit/Rollback
- [ ] Java: Are `@Transactional` propagation and isolation reasonable
- [ ] Python: Is `atomic` decorator scope too large

### Deep Dive Method
1. Find all transaction blocks
2. Check operations inside transaction line by line, mark each as DB or IO
3. If there are non-DB operations inside transaction, analyze how to move them out
4. **Trace error paths**: Step 2 fails inside transaction, do steps before it rollback? Where is Rollback code?
5. Give refactored transaction boundaries

### Scenario Simulation

**Failure simulation**: Transaction fails at step 3, do steps 1-2 rollback? If step 3 is MQ message already sent, database rolls back but message is still in queue — consumer executes logic based on "rolled back" data. Is there compensation mechanism?

**IO in transaction**: HTTP call inside transaction, other side is slow (30s timeout), transaction holds connection and row lock for 30 seconds. Other requests operating on the same row are blocked, database connection pool exhausted, service snowballs.

**Nested transactions**: Method A opens transaction, internally calls Method B which also opens transaction. Inner Rollback → Under Spring REQUIRED propagation, entire transaction marked rollback-only, outer Commit throws `UnexpectedRollbackException` — is this handled?

> **Deep dive signal**: When there are external calls in transaction, link to rate limiting/retry dimensions — external call fails → transaction rolls back → retry → partial operations execute repeatedly (e.g., message sent again).

---

## 10. Data Growth Sensitivity

### Search Keywords

```
Growable collections: append(, push(, .add(, insert(, map[], Map<, dict(, HashMap
Numeric types: int32, int16, int, uint, Integer, Long, float, Float
Memory sort/aggregate: sort., Sort, ORDER BY, GroupBy, GROUP BY, Count, count(
Nested loop signal: for...range/for...in/for...each nested inside for...range/for...in/for...each
Full export: export, Export, download, Download, csv, Excel, xlsx
```

### Checklist

#### Memory Dimension
- [ ] Is there operation loading full table/large data into memory (fine at current volume, OOM after growth)
- [ ] Will collections grow infinitely with business, is there cleanup/eviction mechanism
- [ ] Is sorting/aggregation done in memory that should be done by database
- [ ] Does export/download load all data at once (should stream output)

#### Numeric Dimension
- [ ] Are auto-increment IDs, counters, etc. numeric types sufficient (int → int64 overflow?)
- [ ] Are amount fields using float (precision loss + amplified after amount growth)
- [ ] Are timeout values, retry counts, etc. hard-coded with unreasonable fixed values

#### Query Performance Dimension
- [ ] Does pagination use cursor instead of offset (offset gets slower with large data)
- [ ] Is there nested loop query O(n*m)
- [ ] After table growth, can JOIN still return in reasonable time
- [ ] Are there index-uncovered fuzzy queries (LIKE '%xxx%' full table scan)

#### Storage Dimension
- [ ] Will log tables/record tables grow infinitely (need archiving/cleanup strategy)
- [ ] Will file/attachment storage fill disk
- [ ] Message queue backlog: what if consumption speed can't keep up with production speed

#### Algorithm Dimension
- [ ] Is there O(n²) or higher complexity operation
- [ ] Can hash table optimize lookups (O(n) → O(1))
- [ ] Can index/sort+binary optimize range queries
- [ ] Is there repeated computation that can cache results

### Deep Dive Method

1. **Estimate growth rate**: How much does this table/collection grow monthly? In one year? Three years?
2. **Simulate crash scenarios**: What about 10x? 100x? Is there a crash threshold (OOM, disk full, connection pool exhausted)?
3. **Analyze root cause**: Slow query? → Index/sharding; Too much memory? → Streaming/batch/push down to DB; Algorithm too slow? → Change algorithm; Single table too large? → Sharding/archiving
4. **Give solutions**: offset → cursor-based; full load → streaming/batch; O(n²) → HashMap O(n); int → int64; float → decimal

### Scenario Simulation

**Growth simulation**: Core table grows 100k rows monthly, 1.2M yearly, 3.6M in three years. Current query 50ms, with 3.6M rows if status field only has 3 values (low selectivity), will it degrade to full table scan?

**OOM simulation**: Query fully loads into List, current 1000 items × 2KB = 2MB. 100x growth → 200MB. 5 users concurrently export → 1GB. What's the JVM/Go runtime memory limit?

**Algorithm simulation**: O(n²) nested loop, n=100 takes 10ms/10000 operations, n=10000 takes 100M operations/~100 seconds. Use HashMap to reduce inner O(n) to O(1)?

> **Deep dive signal**: When full load or O(n²) is found, link to #11 (Long-Running Stability) — continuous data growth without elastic scaling, eventually OOM or timeout "suddenly" crashes.

---

## 11. Long-Running Stability

### Search Keywords

```
Scheduled tasks:
  Go: time.Ticker, time.Sleep, cron, Schedule, ticker, interval
  Java: @Scheduled, ScheduledExecutorService, Timer, cron
  Python: schedule, cron, celery.beat, APScheduler, @periodic_task
  Rust: tokio::time::interval, std::thread::sleep

Connection pools:
  Go: sql.Open, SetMaxOpenConns, SetMaxIdleConns, ConnMaxLifetime, redis.NewClient
  Java: HikariCP, DataSource, ConnectionPool, ThreadPoolExecutor, maxPoolSize
  Python: pool, create_engine, pool_size, max_overflow
```

### Checklist
- [ ] Can scheduled task execution time exceed interval (task pile-up)
- [ ] Does cache have eviction strategy (LRU/TTL/max capacity) — see #22
- [ ] Is connection pool configured with reasonable max connections and timeout
- [ ] Is there memory leak risk
- [ ] Does long-running process have panic/exception recovery

### Deep Dive Method
1. For scheduled tasks: calculate single execution maximum time, compare with interval
2. For connection pools: check if maxSize, maxIdle, lifetime are configured
3. **Trace error paths**: What if task panics? Silent failure or alert?
4. Give specific configuration suggestions

### Scenario Simulation

**Task pile-up**: Scheduled task runs every 5 minutes, but single execution takes up to 8 minutes. After exceeding interval, does it start a second instance concurrently (duplicate processing), or skip scheduling? When executing concurrently, will data be processed repeatedly?

**Connection leak**: Connection pool max 20, slow queries occupy 15 for 30 seconds. Do new requests block waiting (queue snowball) or return timeout immediately? Is there connection acquisition timeout and monitoring alert?

**Memory growth**: Starts at 200MB, reaches 2GB after one week — can logs or monitoring see the growth? Is there append-only slice/list without cleanup? Are there global variables never collected? Do long-running goroutines have exit mechanisms?

> **Deep dive signal**: When cache has no eviction strategy or connection pool has no upper limit, link to #10 (Data Growth) — cache entries infinitely expanding, connection pool exhausted under high concurrency, "boiling frog", crashes instantly when traffic spikes.

---

## 12. Observability (Logging)

### Search Keywords

```
Logging frameworks:
  Go: log., logger., slog., Infof, Errorf, Warnf, Debugf, printf, Println
  Java: log., logger., Logger, log.info, log.error, log.warn, log.debug, SLF4J
  Python: logging., logger., log., print(
  Rust: info!, error!, warn!, debug!, tracing::, log::
  TypeScript: console., logger., winston, pino, bunyan

Errors without logs: if err / catch ( / except / Err( → return/throw/raise but no log/logger

Critical business: pay, charge, deduct, refund, transfer, create.*order, update.*status, delete.*account
```

### Checklist
- [ ] Do error paths have logs
- [ ] Do critical business operations have logs (payment, status change, permission change)
- [ ] Do logs contain key context (requestID, userID, businessID, values)
- [ ] Are log levels correct (Error vs Warn vs Info)
- [ ] Is sensitive info (password, token) logged
- [ ] Do key function entry/exit points have logs

### Deep Dive Method
1. List all error handling branches, check which have no logs
2. For critical business functions, trace if logs cover both success and failure paths
3. Check if log content is just "error" without specific info and context
4. **Trace error paths**: After error occurs, can downstream callers trace root cause from logs?
5. Give specific log supplementation suggestions

### Scenario Simulation

**Troubleshooting**: 3 AM user reports "payment failed". Logs only show `log.Println("error")`, no order number, user ID, error reason. How to locate the issue? If logs only have `"payment failed"` without requestID, how to trace the full chain?

**Sensitive info**: `log.Printf("login failed: user=%s password=%s", user, password)` — password is logged, collected by ELK, everyone with ELK access can see it. What level of security incident?

**Log storm**: Error branch triggers 1000 times per second at peak, each 500B → 500KB/s → 300MB in 10 minutes. Is disk sufficient? Is there rate limiting/sampling?

> **Deep dive signal**: When error paths have no logs, link to resource release path check — `defer Close()` failure without logs, silently leaking connections/file handles/memory online until system crash.

---

## Common Missing Items Reference

| If You See | Must Check For | If Missing, Flag |
|------------|----------------|------------------|
| Query inside loop (N+1) | Whether changed to batch query | ❌ N+1 Query |
| `SELECT *` | Whether only needed fields are queried | ⚠️ Querying too much data |
| List query without pagination | Whether has LIMIT/OFFSET or cursor | ❌ Potential full table scan |
| Large batch operation | Whether batched (500-1000 per batch) | ⚠️ Long transaction risk |
| `sync.Mutex` / `Lock()` | Is lock granularity reasonable, does it contain IO | ⚠️ Lock scope too large |
| `go func` | Does it have exit mechanism | ❌ Goroutine leak |
| Shared map/slice concurrent write | Is there lock protection | ❌ Data race |
| Query then update (Get + Update) | Is it within the same lock/transaction | ❌ TOCTOU |
| `Begin` / `@Transactional` | Is there IO inside transaction (HTTP/RPC/MQ) | ❌ Long transaction |
| `Commit` | Is there corresponding `Rollback` on error path | ❌ Transaction leak |
| `int32` auto-increment ID | Is overflow possible | ⚠️ Overflow risk |
| `float` amount calculation | Is decimal/integer cents used | ❌ Precision loss |
| Nested loop | Is time complexity acceptable | ⚠️ Performance bottleneck |
| Full data load into memory | Is there pagination/streaming | ❌ OOM risk |
| `if err != nil` | Is error logged | ⚠️ Silent error swallowing |
| Critical business operation (payment/refund) | Does log contain business ID | ⚠️ Untraceable |
| Log output | Does it contain sensitive info (password/token) | ❌ Info leak |
| Scheduled task | Is pile-up possible (execution time > interval) | ⚠️ Task pile-up |
| Connection pool config | Is maxSize/maxIdle/lifetime configured | ⚠️ Connection leak |
