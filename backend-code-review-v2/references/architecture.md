# 架构与 API 设计审查指南

---

## #9 单一职责与分层清晰

### 核心规则

每个模块、服务只做一件事，禁止"上帝类"和"上帝函数"。

- **分层明确**：
  - **Controller/Handler/API** → 接收请求、参数校验、组装响应
  - **Service/UseCase** → 业务编排与核心逻辑
  - **Repository/DAO/Mapper** → 数据存取
- **禁止跨层调用**：Controller 不直接调 DAO
- 依赖方向：上层依赖下层，禁止反向依赖

**各语言分层命名惯例**：
- Go/Java: `controller`/`handler` → `service` → `repository`/`dao`/`mapper`
- Python: `views` → `services` → `repositories`/`models`
- Node.js: `routes` → `controllers` → `services` → `models`

### 搜索模式

```bash
# ========== 跨层调用检测 ==========

# Go：Controller 直接操作数据库
grep -rn 'func.*Controller\|func.*Handler' --include='*.go' -A20 | grep 'DB\|db\|sql\|Query\|Exec\|\.Find\|\.Where'

# Java：Controller 直接操作数据库
grep -rn '@Controller\|@RestController\|@GetMapping\|@PostMapping' --include='*.java' -A10 | grep 'JdbcTemplate\|Repository\|mapper\.\|session\.\|createQuery'

# Python：View/路由直接操作数据库
grep -rn '@app.route\|@router\.\|def .*view' --include='*.py' -A10 | grep 'Model\.\|objects\.\|session\.\|cursor\.\|db\.'

# TypeScript：Controller 直接操作数据库
grep -rn '@Controller\|@Get\|@Post\|@Router' --include='*.ts' -A10 | grep 'prisma\.\|repository\.\|\.query\|\.raw'

# ========== 上帝函数检测（超过100行的函数/方法） ==========

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

1. **画出模块依赖图**：谁调用了谁，识别循环依赖和反向依赖
2. **定位上帝函数**：对超过100行的函数，分析其职责是否可拆分为多个独立函数
3. **找到跨层调用的具体位置**：确认 Controller 是否直接 import/调用了 Repository/DAO 层
4. **给出重构后的分层代码结构**：按职责拆分，确保每层职责单一、依赖方向正确

---

## #10 API 版本化与兼容性

### 核心规则

接口升级要考虑向前兼容，避免 breaking change。

- **版本化方式**：URL（`/v1/users`）或 Header（`X-API-Version: 2`）
- **向前兼容原则**：
  - 只新增字段，不删除已有字段
  - 不改变已有字段的语义和数据类型
  - 废弃字段标记 `deprecated` 后再删除
- 修改旧逻辑时评估对旧客户端、旧数据、旧状态的影响

### 搜索模式

```bash
# ========== 数据库 schema 变更 ==========

grep -rn 'ALTER TABLE\|ADD COLUMN\|DROP COLUMN\|MODIFY COLUMN\|CREATE TABLE' --include='*.sql' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# ========== 序列化/反序列化标签（API 字段定义） ==========

# Go
grep -rn 'json:"\|xml:"\|form:"\|query:"\|gorm:"' --include='*.go'

# Java
grep -rn '@JsonProperty\|@XmlElement\|@Column' --include='*.java'

# Python
grep -rn 'Field(\|serializer\|@dataclass\|pydantic' --include='*.py'

# TypeScript
grep -rn '@ApiProperty\|@IsOptional\|interface\s\+\w\+\s*{' --include='*.ts'

# ========== 配置项变更 ==========

grep -rn 'Config\|config\|\.Env\|viper\|pydantic' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# ========== 枚举/常量变更 ==========

grep -rn 'const\|iota\|enum\|Enum\|Enumeration' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'
```

### 检查清单

- [ ] 新增字段是否有默认值（旧数据不存在该字段）
- [ ] 删除/重命名字段是否影响旧版本客户端
- [ ] 枚举值新增是否影响旧逻辑
- [ ] 配置项新增是否有默认值
- [ ] 数据库 schema 变更是否向后兼容
- [ ] API 版本管理策略是否明确

### 深挖方法

1. **找到本次变更涉及的 struct/class/DB/API 改动**：列出所有影响面
2. **模拟旧版本客户端调用新接口**：检查响应结构是否仍然可被旧客户端正确解析
3. **模拟旧数据在新代码中处理**：检查新增字段缺失时是否有合理的默认/容错处理
4. **给出兼容性修复**：如需要，提供版本迁移方案或兼容层代码
