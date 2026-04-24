# 六、测试与文档

## 目录
- [25. 关键路径自动化测试](#25-关键路径自动化测试)
- [26. 代码审查习惯](#26-代码审查习惯)
- [27. 接口文档与代码同步](#27-接口文档与代码同步)
- [28. 易错点自问](#28-易错点自问)
- [29. 向后兼容性](#29-向后兼容性)

---

## 25. 关键路径自动化测试

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
4. 给出缺失的测试用例：
   - Go: 表驱动测试 `[]struct{ name string; input ...; want ... }`
   - Java: `@ParameterizedTest` + `@MethodSource`
   - Python: `@pytest.mark.parametrize`
   - TypeScript: `test.each()`

---

## 26. 代码审查习惯

此维度为流程审查，不针对具体代码，重点检查：

### 检查清单
- [ ] 变更是否拆分为合理大小的 PR（不超过400行）
- [ ] PR 描述是否清晰说明了"为什么"改
- [ ] 是否有对应的测试变更
- [ ] 是否有破坏性变更需要通知下游

---

## 27. 接口文档与代码同步

### 搜索模式

```bash
# Go（Swag/Gin-Swag）
grep -rn '@Router\|@Summary\|@Tags\|@Param\|@Success\|@Failure' --include='*.go'
# Go（Protobuf/gRPC）
grep -rn 'rpc \|message \|service ' --include='*.proto'

# Java（Swagger/OpenAPI）
grep -rn '@ApiOperation\|@Api\|@Schema\|@Operation\|@Tag' --include='*.java'
# Java（SpringDoc/SpringFox）
grep -rn '@Operation\|@ApiResponse\|@Schema\|@Tag' --include='*.java'

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

### 深挖方法
1. 列出代码中所有路由定义
2. 对比文档中定义的接口
3. 标记：代码有但文档没有、文档有但代码没有、参数不一致
4. 给出文档更新建议

---

## 28. 易错点自问

审查时对每个函数主动思考以下问题：

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

## 29. 向后兼容性

### 搜索模式

```bash
# 数据库 schema 变更
grep -rn 'ALTER TABLE\|ADD COLUMN\|DROP COLUMN\|MODIFY COLUMN\|CREATE TABLE' --include='*.sql' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 序列化/反序列化标签（API 字段定义）
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

### 深挖方法
1. 找到本次变更涉及的 struct/class/DB/API 改动
2. 模拟旧版本客户端调用新接口
3. 模拟旧数据在新代码中处理
4. 给出兼容性修复
