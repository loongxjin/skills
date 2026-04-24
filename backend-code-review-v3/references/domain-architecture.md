# 三、架构与设计

## 目录
- [13. 单一职责与分层](#13-单一职责与分层)
- [14. 面向接口编程](#14-面向接口编程)
- [15. 配置与代码分离](#15-配置与代码分离)

---

## 13. 单一职责与分层

### 搜索模式

```bash
# Go：Controller 直接操作数据库
grep -rn 'func.*Controller\|func.*Handler' --include='*.go' -A20 | grep 'DB\|db\|sql\|Query\|Exec\|\.Find\|\.Where'
# Java：Controller 直接操作数据库
grep -rn '@Controller\|@RestController\|@GetMapping\|@PostMapping' --include='*.java' -A10 | grep 'JdbcTemplate\|Repository\|mapper\.\|session\.\|createQuery'
# Python：View/路由直接操作数据库
grep -rn '@app.route\|@router\.\|def .*view' --include='*.py' -A10 | grep 'Model\.\|objects\.\|session\.\|cursor\.\|db\.'
# TypeScript：Controller 直接操作数据库
grep -rn '@Controller\|@Get\|@Post\|@Router' --include='*.ts' -A10 | grep 'prisma\.\|repository\.\|\.query\|\.raw'

# "上帝函数"（超过100行的函数/方法）
# Go
awk '/^func /{name=$0; start=NR} /^}$/{if(NR-start>100) print NR, name}' *.go
# Java
awk '/public.*{/{name=$0; start=NR} /^    }$/{if(NR-start>100) print NR, name}' *.java
# Python
awk '/^def /{name=$0; start=NR} /^$/{if(NR-start>100) print NR, name}' *.py
```

### 检查清单
- [ ] 分层是否明确：Handler/Controller → Service → Repository/DAO
- [ ] Controller 是否只做参数校验和转发，不含业务逻辑
- [ ] Service 层是否只含业务逻辑，不含 SQL/HTTP 细节
- [ ] Repository 层是否只含数据访问，不含业务判断
- [ ] 依赖方向是否正确（上层依赖下层，不反向）
- [ ] 是否存在跨层直接调用

### 深挖方法
1. 画出模块依赖图：谁调用了谁
2. 对超过100行的函数，分析其职责是否可拆分
3. 找到跨层调用的具体位置
4. 给出重构后的分层代码结构

### 场景推演

**跨层调用推演**：Controller 直接操作数据库的话，如果将来要换数据库实现，要改几个地方？如果要在 Service 调用前加权限校验，要在几个地方加？

**上帝函数推演**：这个100行的函数，如果出了 bug，你能快速定位是哪一段出的问题吗？测试时能单独测其中一段吗？

**依赖方向推演**：有没有 Repository 层反过来依赖 Service 层的情况？循环依赖会导致什么问题？

> **深挖信号**：如果跨层调用或上帝函数确实存在，要求开发者画出当前调用链路图，标注每层的边界违规点，并给出拆分方案。

> 如果要把数据库从 MySQL 换成 PostgreSQL，需要改哪些文件？如果 Controller 里直接写了 SQL，那就得改 Controller 了——这合理吗？如果 Service 层的某个函数要被 HTTP 和 gRPC 两个入口复用，它能直接处理 HTTP 请求对象吗？

---

## 14. 面向接口编程

### 搜索模式

```bash
# Go：接口定义
grep -rn 'type\s\+\w\+\s\+interface' --include='*.go'
# Go：具体类型直接注入
grep -rn 'struct{.*\*.*Impl\|struct{.*\*mysql\|struct{.*\*redis\|struct{.*\*postgres' --include='*.go'

# Java：接口 vs 具体类注入
grep -rn '@Autowired\|@Inject\|@Resource' --include='*.java' -A1 | grep 'Impl\b\|MySQL\|Redis\|Postgres'
# Java：接口定义
grep -rn 'interface\s\+\w\+' --include='*.java'

# Python：Protocol/ABC
grep -rn 'Protocol\|ABC\|abstractmethod\|@abstractmethod' --include='*.py'
# Python：具体类直接导入
grep -rn 'from.*import.*Impl\|from.*mysql\|from.*redis' --include='*.py'

# Rust：trait 定义
grep -rn 'trait\s\+\w\+' --include='*.rs'

# TypeScript：interface 定义
grep -rn 'interface\s\+\w\+' --include='*.ts'
# TypeScript：具体类注入
grep -rn 'new.*Impl\|new.*Repository\|new.*Service' --include='*.ts'
```

### 检查清单
- [ ] 核心业务逻辑是否依赖接口/抽象而非具体实现
- [ ] 存储层是否抽象为接口（可替换数据库实现）
- [ ] 外部服务调用是否通过接口（便于 mock 测试）
- [ ] Go: 接口是否在使用方定义（消费者定义接口）
- [ ] Java: 是否面向 interface 编程而非 Impl
- [ ] Python: 是否使用 Protocol/ABC 定义抽象
- [ ] Rust: 是否使用 trait 定义行为
- [ ] 接口粒度是否合理（小接口优于大接口）

### 深挖方法
1. 列出所有 struct/class 的字段/依赖类型
2. 标记哪些是具体实现、哪些是接口
3. 分析具体实现字段是否应改为接口
4. 给出接口定义和依赖注入示例

### 场景推演

**可替换性推演**：如果把 MySQL 换成 PostgreSQL，要改几个文件？如果要给 OrderService 写单测，能 mock 掉数据库依赖吗？

**接口粒度推演**：这个接口有几个方法？如果一个调用方只用到其中一个方法，却被强制依赖了所有方法，这合理吗？

**接口膨胀推演**：接口是否在持续添加方法？是否应该拆分为更小的接口？

> **深挖信号**：如果核心业务逻辑直接依赖具体实现，要求开发者指出替换实现的改动的文件数，并给出接口抽象方案和 mock 测试示例。

> 业务层直接 `new MySQLClient()`。如果要写单元测试，你怎么 mock 这个数据库？如果要换 PostgreSQL，要改几个文件？如果两个服务共用同一套业务逻辑但用不同的存储引擎呢？

---

## 15. 配置与代码分离

### 搜索模式

```bash
# 通用：硬编码的端口、地址
# 注意：硬编码数字的通用搜索在 domain-basics.md #5 消除魔法值，此处聚焦于环境配置类硬编码
grep -rn '3306\|5432\|6379\|27017\|localhost\|127.0.0.1\|0.0.0.0' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'

# 硬编码的密钥/token（🔴 安全风险）
grep -rni 'password\s*=\s*"\|secret\s*=\s*"\|token\s*=\s*"\|api_key\s*=\s*"\|apikey\s*=\s*"' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
grep -rni 'password\s*=\s*'\''\|secret\s*=\s*'\''\|token\s*=\s*'\''' --include='*.py'

# 配置读取方式
grep -rn 'os.Getenv\|viper\.\|config\.' --include='*.go'
grep -rn '@Value\|@ConfigurationProperties\|Environment\|Config' --include='*.java'
grep -rn 'os.environ\|os.getenv\|config\.\|settings\.\|pydantic' --include='*.py'
grep -rn 'std::env::\|dotenv\|config::' --include='*.rs'
grep -rn 'process.env\|config\.\|ConfigModule' --include='*.ts'
```

### 检查清单
- [ ] 数据库地址/端口是否硬编码
- [ ] 密钥/token 是否硬编码（安全风险！）
- [ ] 超时时间、重试次数是否可配置
- [ ] 不同环境（dev/staging/prod）配置是否分离
- [ ] 配置是否有合理的默认值

### 深挖方法
1. 找到所有硬编码的配置值
2. 检查是否有对应的配置文件或环境变量
3. 对密钥/token：标记为 🔴 CRITICAL 安全问题
4. 给出配置管理方案（Go: viper, Java: application.yml, Python: pydantic-settings, Rust: dotenv+config crate）

### 场景推演

**多环境推演**：同一份代码，开发环境和生产环境的配置（数据库地址、密钥、超时时间）不同，怎么切换？硬编码意味着需要改代码重新编译。

**密钥泄露推演**：密钥硬编码在代码里，代码进 Git 后，哪些人能看到？密钥一旦泄露，轮换流程是什么？需要通知所有有代码访问权限的人吗？

**配置变更推演**：修改一个超时时间需要重新部署吗？能否通过配置中心热更新？

> **深挖信号**：如果存在硬编码的密钥或环境配置，要求开发者说明多环境切换机制，并给出配置外置的改造方案和密钥轮换流程。

> 代码里写了 `redis.NewClient("localhost:6379")`。部署到生产环境时，忘了改，会怎样？这个密钥硬编码在代码里，如果代码开源了或泄露了，攻击者能做什么？

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| Controller/Handler | 是否只做参数校验+转发，不含 DB 操作 | ⚠️ 跨层调用 |
| `new MySQLClient()` / `new RedisClient()` | 是否通过接口/依赖注入 | ⚠️ 依赖具体实现 |
| 硬编码端口 `3306` / `localhost` | 是否从配置文件读取 | ⚠️ 配置硬编码 |
| 硬编码密钥 `password = "..."` | 是否使用环境变量/密钥管理服务 | ❌ 密钥泄露 |
| Service 层直接 `db.Query(...)` | 是否通过 Repository 抽象 | ⚠️ 分层不清 |

---

## 跨维度关联提示

| 关联信号 | 指向的维度 | 要追问的问题 |
|----------|-----------|-------------|
| Controller 直接操作数据库 | #8 并发安全、#9 事务边界 | 跨层调用的事务管理是否规范？并发控制是否遗漏？ |
| 密钥硬编码 | #21 最小权限、#20 输入校验 | 用的是什么账号？有没有 root/sa？敏感数据是否也硬编码了？ |
| 接口缺失 | #25 测试 | 核心逻辑依赖具体实现，单测怎么 mock？测试覆盖率会怎样？ |
| 上帝函数 | #18 重试、#19 资源释放 | 函数过长，资源释放是否在每个分支都覆盖了？重试逻辑是否混在业务逻辑中？ |
| 配置未分离 | #17 限流熔断、#18 重试 | 超时/重试/限流配置硬编码，不同环境是否合理？能否按需调整？ |
