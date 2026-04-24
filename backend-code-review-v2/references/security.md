# 安全与防御审查指南

## #27 输入永远不可信

### 核心规则

- 所有外部输入（用户提交、第三方回调、上游服务响应）都要校验和过滤
- 基础防护：SQL 注入、XSS、CSRF、命令注入、路径遍历
- 参数校验越早越好，在入口层（Controller/Gateway）拦截非法输入
- 常见错误：认为内部服务传递的数据可信，不在 Service 层做二次校验

### 搜索模式

```bash
# 通用：参数绑定/解析（入口点）
grep -rn 'Bind\|Parse\|Unmarshal\|GetParam\|QueryParam\|Body\|PathVariable\|RequestParam' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# Go
grep -rn 'ShouldBind\|ShouldBindJSON\|c.Query\|c.Param\|c.PostForm' --include='*.go'

# Java
grep -rn '@RequestBody\|@PathVariable\|@RequestParam\|@Valid\|@Validated' --include='*.java'

# Python
grep -rn 'request.json\|request.args\|request.form\|request.data' --include='*.py'

# TypeScript
grep -rn '@Body\|@Param\|@Query\|req.body\|req.params\|req.query' --include='*.ts'

# SQL 注入风险
grep -rn 'fmt.Sprintf.*SELECT\|fmt.Sprintf.*INSERT\|fmt.Sprintf.*WHERE\|string.*SQL\|String.format.*SELECT\|f".*SELECT\|f".*INSERT\|f".*WHERE' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 校验逻辑
grep -rn 'validate\|Validate\|required\|@NotNull\|@NotBlank\|@Size\|@Min\|@Max\|validator' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 所有外部输入是否经过校验
- [ ] 参数校验是否在最外层完成
- [ ] SQL 是否使用参数化查询
- [ ] 是否有 XSS 防护
- [ ] 数值范围是否校验
- [ ] Java: 是否使用 `@Valid` / `@Validated` + Bean Validation
- [ ] Python: 是否使用 Pydantic / Marshmallow 校验

### 深挖方法

1. 找到所有 API 入口点
2. 追踪输入数据从入口到使用的完整路径
3. 检查路径上是否有校验
4. 对 SQL 拼接，给出参数化查询改造

---

## #28 最小权限原则

### 核心规则

- 数据库账号按业务拆分，只给必要权限，禁止用 root 账号跑业务
- API 接口做好鉴权与权限校验，敏感操作要二次确认或审批
- 服务间调用也要鉴权，内网并非绝对安全

### 搜索模式

```bash
# 数据库连接配置：root/admin 账号
grep -rn 'root@\|admin@\|sysdba\|sa@\|superuser\|postgres://' --include='*.go' --include='*.java' --include='*.py' --include='*.ts' --include='*.yaml' --include='*.yml' --include='*.env' --include='*.toml'

# 鉴权中间件
grep -rn 'auth\|Auth\|jwt\|JWT\|token\|Token\|permission\|Permission\|rbac\|RBAC' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# Go
grep -rn 'middleware\|Middleware' --include='*.go' | grep -i auth

# Java
grep -rn '@PreAuthorize\|@Secured\|@RolesAllowed\|SecurityConfig\|WebSecurityConfigurerAdapter' --include='*.java'

# Python
grep -rn '@login_required\|@permission_required\|is_authenticated\|permission_classes' --include='*.py'

# 危险操作
grep -rn 'delete.*all\|drop\|truncate\|DELETE FROM\|DROP TABLE\|TRUNCATE' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
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
3. 对无鉴权的接口，评估风险
4. 给出权限最小化建议

---

## #29 敏感数据保护

### 核心规则

- 敏感数据（密码、密钥、token）必须加密存储，禁止明文
- 传输层使用 TLS/HTTPS，禁止明文传输敏感信息
- 密钥不要和代码放在一起，使用 KMS 或 Vault 管理
- 常见错误：把生产环境密钥提交到 Git 仓库

### 搜索模式

```bash
# 硬编码的密钥/token（🔴 CRITICAL 安全风险）
grep -rni 'password\s*=\s*"\|secret\s*=\s*"\|token\s*=\s*"\|api_key\s*=\s*"\|apikey\s*=\s*"' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
grep -rni 'password\s*=\s*'\''\|secret\s*=\s*'\''\|token\s*=\s*'\''' --include='*.py'

# 密码哈希
grep -rn 'bcrypt\|argon2\|scrypt\|pbkdf2\|sha256\|sha512\|md5' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# TLS 配置
grep -rn 'tls\|TLS\|ssl\|SSL\|https\|HTTPS\|cert\|certificate' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 密码是否使用 bcrypt/argon2 等强哈希算法（禁止 md5/sha1）
- [ ] 敏感配置是否通过环境变量/配置中心/密钥管理服务获取
- [ ] 传输层是否使用 TLS/HTTPS
- [ ] 密钥是否提交到 Git 仓库（检查 .gitignore）
- [ ] 是否使用 KMS/Vault 等密钥管理服务
- [ ] 日志中是否泄露敏感信息

### 深挖方法

1. 搜索所有硬编码的密钥/token/密码
2. 检查密码存储是否使用安全的哈希算法
3. 检查 .gitignore 是否排除了敏感文件
4. 检查是否有 TLS 配置
5. 给出密钥管理方案
