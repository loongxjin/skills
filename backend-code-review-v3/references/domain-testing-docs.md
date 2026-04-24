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
3. 分析现有测试的断言质量（是真断言还是 `assert True`）
4. **重点检查边界测试**：0、负数、空值、极大值、并发场景
5. 给出缺失的测试用例：
   - Go: 表驱动测试 `[]struct{ name string; input ...; want ... }`
   - Java: `@ParameterizedTest` + `@MethodSource`
   - Python: `@pytest.mark.parametrize`
   - TypeScript: `test.each()`

### 场景推演

**无测试推演**：支付金额计算逻辑改了一行代码，没有单测。上线后发现100元的订单收了1000元。这个问题多久能被发现？需要多少用户投诉？

**测试质量推演**：测试只验证了 happy path，边界条件测了吗？金额为0、负数、超大数？并发场景？网络超时？

**Mock vs 真实推演**：mock 了数据库返回成功，但真实数据库有唯一约束。这种 bug 单测能发现吗？需要集成测试吗？

> **深挖信号**：如果核心业务逻辑没有测试覆盖，或者测试只有 happy path，应立即追问边界条件和异常路径的测试情况。

> **推演场景补充**：这段支付逻辑有 4 种状态（待支付、已支付、已退款、已取消）和 6 种状态转换规则。手动测试能覆盖所有组合吗？如果现在要加第 5 种状态"部分退款"，你能确保不改坏现有逻辑吗？如果没有单测，你敢改吗？

---

## 26. 代码审查习惯

此维度为流程审查，不针对具体代码，重点检查：

### 检查清单
- [ ] 变更是否拆分为合理大小的 PR（不超过400行）
- [ ] PR 描述是否清晰说明了"为什么"改
- [ ] 是否有对应的测试变更
- [ ] 是否有破坏性变更需要通知下游

### 推演场景
> 这个 PR 有 2000 行改动，涉及 30 个文件。Review 的人能仔细看完吗？还是草草扫一眼就 Approve 了？如果里面藏了一个 bug，能被发现吗？

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

### 场景推演

**文档不一致推演**：文档说返回 `user_id`，代码返回 `userId`。前端按文档开发的，上线后解析失败。这种问题 CR 能发现吗？

**废弃 API 推演**：某个接口已经标记废弃3个月了，还有调用方在用吗？直接删除会怎样？有没有下线计划？

**字段变更推演**：新增了一个必填字段但没有更新文档，调用方不知道要传这个字段，生产环境报 400。

> **深挖信号**：文档与代码不一致是协作问题的源头。发现任何字段名、类型、必填/可选的不一致，都应立即标记并追踪影响范围。

> **推演场景补充**：接口文档说返回 `status: number`，代码里已经改成了 `status: string`。下游团队按文档开发的客户端，解析 `status` 时按 number 处理，线上全部报错。文档和代码不一致，比没有文档更危险。

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

### 场景推演

**空指针推演**：这个函数的返回值在调用方有判空吗？如果数据库查询返回 nil/null，下游代码会 panic/NPE 吗？

**金额精度推演**：`0.1 + 0.2 == 0.3` 成立吗？金额字段用的 float 还是 decimal？累计100万笔交易后精度偏差有多大？

**时区推演**：服务器在不同时区，`time.Now()` 返回的时间一样吗？存入数据库的时间是 UTC 还是本地时间？跨时区查询会怎样？

**除零推演**：平均值的除数可能是 0 吗？百分比计算的分母可能是 0 吗？

> **深挖信号**：易错点是 bug 的高发区域。空指针、金额精度、时区、除零——每一个看似小问题，在生产环境都可能引发大面积故障。审查时应逐一排查，不可跳过。

> **推演场景补充（精度）**：`0.1 + 0.2 == 0.3` 在 float 中是 `False`。如果这是在计算金额，`price * quantity` 的误差在单价小、数量大时会放大。10000 次 `0.1 + 0.2` 的累积误差有多少？
>
> **推演场景补充（时区）**：`time.Now()` 返回的是服务器本地时间。如果服务器在上海（UTC+8），但数据库用 UTC 存储查询条件 `WHERE created_at > '2024-01-01'`，会差 8 小时。

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
2. **模拟旧版本客户端调用新接口**：哪些字段会缺失？缺失后逻辑会怎样？
3. **模拟旧数据在新代码中处理**：新字段旧数据为 NULL，代码能正确处理吗？
4. 给出兼容性修复

### 场景推演

**旧客户端推演**：新版 API 删除了一个字段，还有10%的旧版客户端在读这个字段。这些用户会看到什么？崩溃？空白？

**数据库变更推演**：给表加了一个 NOT NULL 字段但没有默认值，旧代码插入数据时会报错。部署顺序是先代码还是先数据库？

**枚举新增推演**：在枚举中新增了一个值，旧版本的 switch/match 能处理这个新值吗？会走到 default 分支吗？default 分支的行为正确吗？

> **深挖信号**：向后兼容问题往往不在开发时暴露，而在部署或版本切换时集中爆发。任何 schema、枚举、API 字段的变更，都应模拟新旧版本共存的场景。

> **推演场景补充（删字段）**：API 响应里删了 `discount_price` 字段。旧版本客户端代码是 `if (response.discount_price > 0)`，删了之后变成 `undefined > 0`，JavaScript 里是 `false`，业务逻辑错误但没有报错。静默出错最可怕。
>
> **推演场景补充（新字段无默认值）**：DB 新增了 `is_verified` 字段但没有默认值。旧数据全是 `NULL`。代码 `if user.is_verified { ... }` 对旧用户不生效。如果这影响的是风控逻辑呢？

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| 核心业务函数（Pay/Charge/Calculate） | 单元测试 | ⚠️ 无测试覆盖 |
| 测试文件 | 边界测试（0/空值/负数/极大值） | ⚠️ 测试不充分 |
| API 路由定义 | 对应的文档注解 | ⚠️ 文档缺失 |
| `ALTER TABLE DROP COLUMN` | 是否有灰度策略（先代码兼容再删字段） | ❌ 破坏性变更 |
| 新增 struct 字段 | 是否有默认值 / `omitempty` | ⚠️ 旧数据兼容 |
| 枚举新增值 | 旧逻辑是否兼容新值 | ⚠️ 枚举扩展 |
| `json:"field_name"` | 字段名变更是否影响旧客户端 | ❌ 字段重命名 |
| 金额计算 | 是否使用 decimal（非 float） | ❌ 精度丢失 |
| `time.Now()` | 是否考虑时区 | ⚠️ 时区问题 |
