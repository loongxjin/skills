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
