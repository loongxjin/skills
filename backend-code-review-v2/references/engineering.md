# 工程实践审查指南

本文件涵盖工程实践相关的 6 个审查维度（#30 ~ #35），每个维度包含：核心规则、搜索模式、检查清单、深挖方法。

---

## #30 关键路径要有自动化测试

### 核心规则

- 不要求 100% 覆盖率，核心业务逻辑（支付、状态流转、金额计算、权限判断）必须有单测
- 表驱动测试覆盖边界情况（空值、极值、并发、异常）
- 测试要独立、可重复、不依赖外部环境和时序
- 常见错误：测试依赖真实数据库或外部 API，导致不稳定

### 搜索模式

```bash
# Go：测试文件
find . -name '*_test.go'
grep -rn 'func Test\|func Benchmark\|func Fuzz' --include='*_test.go'

# Java：测试文件
find . -name '*Test.java' -o -name '*Tests.java' -o -name '*IT.java'
grep -rn '@Test\|@ParameterizedTest\|@RepeatedTest' --include='*Test.java' --include='*Tests.java'

# Python：测试文件
find . -name 'test_*.py' -o -name '*_test.py'
grep -rn 'def test_\|class Test\|@pytest' --include='test_*.py' --include='*_test.py'

# Rust：测试
find . -name '*.rs' -exec grep -l '#\[test\]' {} \;
grep -rn '#\[test\]\|#\[tokio::test\]' --include='*.rs'

# TypeScript：测试文件
find . -name '*.test.ts' -o -name '*.spec.ts'
grep -rn 'describe(\|it(\|test(' --include='*.test.ts' --include='*.spec.ts'

# Mock 使用
grep -rn 'gomock\|mock\|Mock\|testify\|mockery' --include='*.go'
grep -rn 'Mockito\|@Mock\|@InjectMocks\|mockito' --include='*.java'
grep -rn 'unittest.mock\|@patch\|mocker\|pytest-mock\|freezegun' --include='*.py'
grep -rn 'jest.fn\|jest.mock\|sinon\|vi.fn\|vi.mock' --include='*.ts'

# 核心业务逻辑（应有测试覆盖）
grep -rn 'func.*Pay\|func.*Charge\|func.*Deduct\|func.*Transfer\|func.*Calculate' --include='*.go'
grep -rn 'public.*pay\|public.*charge\|public.*deduct\|public.*transfer\|public.*calculate' --include='*.java'
grep -rn 'def.*pay\|def.*charge\|def.*deduct\|def.*transfer\|def.*calculate' --include='*.py'
```

### 检查清单

- [ ] 核心业务逻辑（支付、状态流转、金额计算）是否有单测
- [ ] 是否使用表驱动测试/参数化测试覆盖边界情况
- [ ] 是否有集成测试覆盖关键链路
- [ ] 是否有 mock 外部依赖
- [ ] 测试是否有有意义的断言
- [ ] 是否覆盖了异常路径

### 深挖方法

1. 列出所有核心业务函数
2. 对比测试文件，找出没有测试覆盖的核心函数
3. 分析现有测试的断言质量
4. 给出缺失的测试用例：Go: 表驱动测试、Java: @ParameterizedTest、Python: @pytest.mark.parametrize

---

## #31 Code Review 要成为习惯

### 核心规则

- 审查时重点关注逻辑正确性、安全性、性能隐患、边界情况
- 代码风格交给自动化工具（Linter、Formatter），Review 时不纠结风格
- 重要变更至少一人 Review 才能合入主干

### 搜索模式

```bash
# Review 配置检查
# GitHub/GitLab CODEOWNERS
find . -name 'CODEOWNERS' -o -name 'codeowners'
cat .github/CODEOWNERS 2>/dev/null || cat .gitlab/CODEOWNERS 2>/dev/null

# CI 中的 Review 门禁
grep -rn 'required_reviewers\|approve\|CODEOWNERS\|reviewers\|require_review' --include='*.yml' --include='*.yaml'

# Linter/Formatter 配置
find . -name '.golangci.yml' -o -name '.eslintrc*' -o -name '.prettierrc*' -o -name 'checkstyle.xml' -o -name '.pylintrc' -o -name 'ruff.toml' -o -name 'pyproject.toml'
grep -rn 'golangci-lint\|eslint\|prettier\|checkstyle\|pylint\|ruff\|black\|gofmt\|go vet' --include='*.yml' --include='*.yaml' --include='Makefile' --include='*.toml'

# PR 模板
find . -name 'pull_request_template.md' -o -name 'PULL_REQUEST_TEMPLATE.md'

# 提交规范
find . -name '.commitlintrc*' -o -name 'commitlint.config.*'
grep -rn 'commitlint\|conventional' --include='*.json' --include='*.js' --include='*.yml'
```

### 检查清单

- [ ] 项目是否配置了 CODEOWNERS 确保关键路径有人 Review
- [ ] CI 是否强制要求至少一人 Review 才能合入
- [ ] 是否有 Linter/Formatter 自动检查代码风格
- [ ] 是否有 PR 模板引导提交者描述变更
- [ ] 是否有提交规范（Conventional Commits 等）
- [ ] Review 是否关注逻辑正确性而非纠结风格

### 深挖方法

1. 检查 CODEOWNERS 文件，确认关键目录是否有指定 Reviewer
2. 检查 CI 配置，确认是否设置了 Review 门禁
3. 检查 Linter/Formatter 是否在 CI 中运行
4. 评估当前 Review 流程的有效性，给出改进建议

---

## #32 文档与代码同步

### 核心规则

- API 文档要及时更新，文档与实现不一致比没有文档更可怕
- 复杂业务逻辑要写设计文档，记录"为什么这样做"而不只是"做了什么"
- 核心决策留下上下文（Architecture Decision Record），避免后人重复踩坑

### 搜索模式

```bash
# Go（Swag/Gin-Swag）
grep -rn '@Router\|@Summary\|@Tags\|@Param\|@Success\|@Failure' --include='*.go'
# Go（Protobuf/gRPC）
grep -rn 'rpc \|message \|service ' --include='*.proto'

# Java（Swagger/OpenAPI）
grep -rn '@ApiOperation\|@Api\|@Schema\|@Operation\|@Tag' --include='*.java'

# Python（DRF Spectacular / Flask-RESTX）
grep -rn 'swagger\|openapi\|@swagger\|serializers\.' --include='*.py'

# TypeScript（OpenAPI / NestJS / tRPC）
grep -rn '@ApiOperation\|@ApiResponse\|@ApiTags' --include='*.ts'
grep -rn 'swagger\|openapi\|trpc\.' --include='*.ts'

# 路由定义（对比文档）
grep -rn 'router\.\|r\.GET\|r\.POST\|r\.PUT\|r\.DELETE\|Group\|Handle' --include='*.go'
grep -rn '@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping\|@RequestMapping' --include='*.java'
grep -rn '@app.route\|@router\.\|path\(' --include='*.py'
grep -rn '@Get\|@Post\|@Put\|@Delete\|router\.' --include='*.ts'
```

### 检查清单

- [ ] 每个 API 是否有文档注解
- [ ] 文档中的参数、返回值是否与代码一致
- [ ] 是否有废弃的 API 但文档未标注
- [ ] 复杂业务逻辑是否有设计文档
- [ ] 核心决策是否有 ADR（Architecture Decision Record）
- [ ] 文档是否记录了"为什么这样做"而不只是"做了什么"

### 深挖方法

1. 列出代码中所有路由定义
2. 对比文档中定义的接口
3. 标记：代码有但文档没有、文档有但代码没有、参数不一致
4. 给出文档更新建议

---

## #33 易错点自问

### 核心规则

- 审查时对每个函数主动思考常见错误类别，避免遗漏边界情况和隐蔽 bug
- 将易错点自问作为 Code Review 的标准步骤，形成肌肉记忆

### 搜索模式

```bash
# 空指针风险
grep -rn '\.Field\|\.Method\|map\[' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' | grep -v 'if.*nil\|if.*null\|if.*None\|if err\|optional\|unwrap\|?'

# 整数溢出
grep -rn 'int32\|int16\|uint32\|uint16\|int\b' --include='*.go' --include='*.java' --include='*.py' --include='*.rs'

# 金额 float
grep -rn 'float\b\|Float\|double\b\|Double' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 时区处理
grep -rn 'time.Now\|time.Parse\|Local\|UTC\|timezone\|timeZone\|getHours\|getMinutes' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] **空指针/None/null**：这个变量可能为 nil/null/None 吗？
- [ ] **越界**：slice/array/list/map 访问是否可能越界？key 是否存在？
- [ ] **除零**：是否有除法运算？除数可能为 0 吗？
- [ ] **并发**：多线程/协程同时访问这段代码会怎样？
- [ ] **边界**：空输入？超长输入？特殊字符？
- [ ] **时区**：时间处理是否考虑时区？
- [ ] **编码**：字符串处理是否考虑编码问题？
- [ ] **精度**：金额计算是否使用 decimal 而非 float？
- [ ] **顺序**：这段逻辑依赖特定的执行顺序吗？

### 深挖方法

1. 对每个函数，逐一回答以上问题
2. 对可能出错的地方，给出具体的防护代码
3. 特别关注金额、状态流转、权限判断等高风险逻辑

---

## #34 长时间运行稳定性

### 核心规则

- 长期运行的服务要考虑定时任务堆积、连接池耗尽、缓存无淘汰、内存泄漏等稳定性隐患
- 生产环境的 bug 大多与时间相关（运行越久越容易暴露）

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

# 缓存
grep -rn 'sync.Map\|lru\|cache\|Cache\|TTL\|ttl\|expiry\|Expire\|Evict\|MaxSize' --include='*.go'
grep -rn 'Caffeine\|GuavaCache\|@Cacheable\|ConcurrentHashMap' --include='*.java'
grep -rn 'lru_cache\|TTLCache\|cachetools\|functools.cache\|@cached' --include='*.py'

# goroutine/线程泄漏
grep -rn 'go func\|go \w+\(' --include='*.go'
grep -rn 'new Thread\|CompletableFuture\|@Async\|submit(' --include='*.java'
grep -rn 'threading.Thread\|asyncio.create_task\|Thread\(' --include='*.py'
```

### 检查清单

- [ ] 定时任务执行时间是否可能超过间隔（任务堆积）
- [ ] 缓存是否有淘汰策略（LRU/TTL/最大容量）
- [ ] 连接池是否配置了合理的最大连接数和超时
- [ ] 是否有内存泄漏风险（未关闭的资源、无限增长的缓存/map）
- [ ] 长期运行的进程是否有 panic/exception recovery
- [ ] goroutine/线程是否有合理的退出机制

### 深挖方法

1. 对定时任务：计算单次执行最长时间，对比间隔
2. 对连接池：检查是否配置了 maxSize、maxIdle、lifetime
3. 对缓存：检查是否有容量限制和淘汰策略
4. 给出具体配置建议

---

## #35 向后兼容性

### 核心规则

- 代码变更必须考虑对旧数据、旧客户端、旧配置的影响，避免不兼容的破坏性变更
- 每次发布前要回答：旧版本客户端能否正常调用？旧数据能否正常处理？

### 搜索模式

```bash
# 数据库 schema 变更
grep -rn 'ALTER TABLE\|ADD COLUMN\|DROP COLUMN\|MODIFY COLUMN\|CREATE TABLE' --include='*.sql' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 序列化/反序列化标签
# Go
grep -rn 'json:"\|xml:"\|form:"\|query:"\|gorm:"' --include='*.go'
# Java
grep -rn '@JsonProperty\|@XmlElement\|@Column' --include='*.java'
# Python
grep -rn 'Field(\|serializer\|@dataclass\|pydantic' --include='*.py'
# TypeScript
grep -rn '@ApiProperty\|@IsOptional\|interface\s\+\w\+\s*{' --include='*.ts'

# 配置项变更
grep -rn 'Config\|config\|\.Env\|viper\|pydantic' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 枚举/常量变更
grep -rn 'const\|iota\|enum\|Enum\|Enumeration' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 新增字段是否有默认值（旧数据不存在该字段）
- [ ] 删除/重命名字段是否影响旧版本客户端
- [ ] 枚举值新增是否影响旧逻辑
- [ ] 配置项新增是否有默认值
- [ ] 数据库 schema 变更是否向后兼容
- [ ] API 版本管理策略是否明确
- [ ] 废弃功能是否有过渡期和迁移文档

### 深挖方法

1. 找到本次变更涉及的 struct/class/DB/API 改动
2. 模拟旧版本客户端调用新接口
3. 模拟旧数据在新代码中处理
4. 给出兼容性修复
