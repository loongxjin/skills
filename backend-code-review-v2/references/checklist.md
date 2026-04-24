# 快速检查清单

快速审查时逐项搜索验证，在报告末尾输出汇总表格。

| # | 维度 | 核心规则 | 搜索模式 | ❌ 触发条件 |
|---|------|----------|----------|-------------|
| 1 | 命名规范 | 自解释，无无意义缩写 | `\btmp\b\|\btemp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b\|\bbuf\b\|\bnum\b` | `a`/`b`/`tmp`/`data`/`handle`/`data1`/`data2` 等 |
| 2 | 函数聚焦 | 只做一件事，< 50 行 | `^func\|^def\|^fn\|^public\|^private\|^function` + 函数名含 `and`/`or` | 函数内有分段注释（`// step 1`）/ 超过 50 行 |
| 3 | 参数控制 | ≤3 个，禁止 bool 参数 | 4+ 参数函数定义 + `bool` 参数搜索 | `doSomething(true, false)` / 参数超过 3 个 |
| 4 | Guard Clause | 先处理错误再主逻辑，≤3 层嵌套 | `^\t\t\t`（Go/Java）或 `^            `（Python）+ `} else {` / `else:` | 超过 3 层嵌套 if / 可提前 return 未提前 |
| 5 | 魔法数字 | 提取为常量/枚举 | `\b\d{3,}\b` + `== [0-9]\|!= [0-9]` + `= "`（排除 final/static） | 裸数字 ≥3 位 / 硬编码状态码 / 业务字符串未提常量 |
| 6 | DRY | 重复逻辑抽象，不过早抽象 | 统计 `if err != nil` / `catch (` / `except` 频次 + CRUD 模板重复 | 相同代码出现 3+ 行 / 相同错误处理模式重复 |
| 7 | 接口抽象 | 依赖接口非实现 | `type.*interface` / `@Autowired`/`@Inject`/`Protocol`/`ABC` / `trait` | 业务层直接 `new MySQLClient()` / 直接依赖具体类 |
| 8 | 配置分离 | 禁止硬编码配置 | `3306\|5432\|6379\|localhost\|127.0.0.1` + `password\s*=\s*"\|secret\s*=\s*"\|token\s*=` | 硬编码密钥/URL/密码/端口地址 |
| 9 | 分层清晰 | 禁止跨层调用 | Controller/Handler 定义后跟 `DB\|db\|sql\|Query\|\.Find` + `controller.*dao\|handler.*mapper` | Controller 直接调 DAO / 上帝函数 >100 行 |
| 10 | API 兼容 | 向前兼容，版本化 | `/v\d+/\|version\|X-API-Version` + `ALTER TABLE\|DROP COLUMN\|@JsonProperty\|json:"` | 删除字段/改类型/breaking change/无默认值新字段 |
| 11 | 数据库索引 | 核心字段有索引，防 N+1 | `\.Find\|\.Where\|@Select\|session.query\|\.findMany` + 循环内查询检测 | WHERE/JOIN/ORDER BY 字段无索引 / N+1 查询 / 无分页 |
| 12 | 事务边界 | 无 IO，范围最小 | `Begin\|@Transactional\|transaction.atomic\|\$transaction` + `http.Get\|grpc\.\|requests\.` | 事务内有 HTTP/RPC/文件操作 / 长事务 |
| 13 | 批量操作 | 分批处理 | `for.*INSERT\|for.*UPDATE\|for.*DELETE\|saveAll\|bulk_create` | 无分批，单批次 >2000 / 循环内单条操作 |
| 14 | 数据增长 | 考虑 100 倍增长 | `append\|push\|\.add\|map\[` + `int32\|int16\|Integer\|float\b` + 嵌套循环检测 | INT 溢出风险 / float 金额 / 全量加载 / O(n²) |
| 15 | Schema 变更 | 向下兼容，幂等 | `ALTER TABLE\|migration\|migrate\|schema\|DDL` | 直接删字段/改类型/非幂等脚本 |
| 16 | 缓存一致 | 更新 DB 后失效缓存 | `cache\.Set\|redis\.Set\|@Cacheable` + `func.*Update\|def.*save` | 更新缓存而非失效缓存 / 无 TTL / 无防穿透 |
| 17 | 并发安全 | 临界区加锁，防 TOCTOU | `sync.Mutex\|synchronized\|threading.Lock\|atomic\|Lock(` + `go func` + 共享状态 | 先查后改无锁保护 / goroutine 无退出机制 / 死锁风险 |
| 18 | 幂等设计 | 重复调用结果一致 | `idempoten\|dedup\|request_id` + `Create\|Insert\|Pay\|Charge\|Deduct\|callback` | 支付/库存/发消息无去重 / 无唯一请求 ID |
| 19 | 限流熔断 | 对外限流，外部有超时 | `timeout\|rate.*limit\|circuit.*break\|RateLimiter\|Resilience4j` + 外部调用检测 | 外部调用无超时/限流/熔断 / 无降级方案 |
| 20 | 重试策略 | 可重试才重试，有退避 | `retry\|backoff\|maxRetries\|@Retryable\|tenacity` + `while True\|for {` | 业务错误也重试 / 无退避 / 无最大次数 / 无限重试 |
| 21 | 消息队列 | 消费幂等，有死信队列 | `consumer\|kafka\|rabbitmq\|amqp\|subscriber` + `dead.letter\|DLQ` | 无 DLQ / 消费不幂等 / 静默丢消息 / 无对账 |
| 22 | 资源释放 | 连接/句柄/锁必释放 | `os.Open\|sql.Open\|net.Dial` + `defer.*Close\|try-with-resources\|with open\|impl Drop` | 打开资源无对应释放 / goroutine 无退出机制 |
| 23 | 日志规范 | 关键路径有结构化日志 | `log\.\|logger\.\|slog\.\|SLF4J\|print(` + 错误处理无日志检测 | 无日志/打敏感信息/生产 DEBUG / `fmt.Println`/`console.log` |
| 24 | 监控告警 | 关键路径有指标埋点 | `prometheus\|Counter\|Gauge\|Histogram\|@Timed\|healthz\|readyz` + `signal.Notify\|SIGTERM` | 核心业务无指标/无告警 / 无健康检查 / 无优雅停机 |
| 25 | 链路追踪 | 日志含 trace_id | `otel\|opentelemetry\|traceID\|trace_id\|span_id\|correlationID` + `context.Context\|MDC\|ContextVar` | 跨服务调用无 trace_id / 链路断裂 |
| 26 | 优雅上下线 | 处理完请求再退出 | `graceful\|shutdown\|warmup\|terminationGracePeriodSeconds` | 直接 kill -9 / 无预热 / 无健康检查 |
| 27 | 输入校验 | 入口层校验过滤 | `ShouldBind\|@Valid\|request.json\|req.body` + `fmt.Sprintf.*SELECT\|f".*WHERE` | 用户输入直达 DB / SQL 拼接 / 无校验 |
| 28 | 最小权限 | DB/API 只给必要权限 | `root@\|admin@\|sa@\|@PreAuthorize\|@login_required\|DROP TABLE\|TRUNCATE` | 敏感操作无鉴权/用 root 账号 / 危险操作无限制 |
| 29 | 敏感数据 | 加密存储，TLS 传输 | `password\s*=\s*"\|secret\s*=\s*"\|token\s*=` + `bcrypt\|argon2\|md5` | 明文存储密码/密钥 / HTTP 传输敏感数据 / md5 哈希 |
| 30 | 自动化测试 | 核心逻辑有单测 | `*_test.go\|*Test.java\|test_*.py\|*.test.ts` + `mock\|Mockito\|@patch\|jest.mock` | 核心函数（支付/金额/状态）无测试 / 无 mock |
| 31 | Code Review | 关注逻辑正确性 | `CODEOWNERS\|.github/\|branch_protection\|linter\|golangci` | 无 Review 机制 / 无 CI 门禁 / 纠结风格 |
| 32 | 文档同步 | API/设计文档及时更新 | `@Router\|@Summary\|@Operation\|swagger\|openapi\|trpc\.` + 路由定义对比 | 接口变了文档未更新 / 废弃 API 未标注 |
| 33 | 易错点自问 | 主动思考 9 大易错类别 | `\.Field\|map\[\|int32\|float\b\|time.Now\|time.Parse`（辅助搜索） | 空指针/越界/除零/时区/float 金额/并发边界 |
| 34 | 长时间运行稳定性 | 定时任务/连接池/内存 | `time.Ticker\|@Scheduled\|cron` + `SetMaxOpenConns\|HikariCP\|pool_size` | 定时任务堆积 / 连接池未配置 / 缓存无淘汰 / 内存泄漏 |
| 35 | 向后兼容性 | 变更考虑旧数据/旧客户端 | `ALTER TABLE\|DROP COLUMN\|json:"\|@JsonProperty\|@ApiProperty\|enum\|Enum` | 新字段无默认值 / 删字段影响旧客户端 / 枚举新增影响旧逻辑 |
