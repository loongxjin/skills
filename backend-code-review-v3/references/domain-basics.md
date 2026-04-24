# 一、代码基础

## 目录
- [1. 命名规范](#1-命名规范)
- [2. 函数职责单一](#2-函数职责单一)
- [3. 参数精简](#3-参数精简)
- [4. 控制流扁平化](#4-控制流扁平化)
- [5. 消除魔法值](#5-消除魔法值)
- [6. 代码复用与去重](#6-代码复用与去重)

---

## 1. 命名规范

### 搜索模式

根据项目语言选择对应的后缀和关键字：

```bash
# Go
grep -rn '\btmp\|temp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b\|\bbuf\b\|\bnum\b' --include='*.go'
# Java
grep -rn '\btmp\|temp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b' --include='*.java'
# Python
grep -rn '\btmp\b\|\btemp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b' --include='*.py'
# Rust
grep -rn '\btmp\b\|\btemp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b' --include='*.rs'
# TypeScript
grep -rn '\btmp\b\|\btemp\b\|\bret\b\|\bdata\b\|\binfo\b\|\bresult\b\|\bobj\b\|\bval\b' --include='*.ts' --include='*.tsx'
```

### 检查清单
- [ ] 变量名是否表达了业务含义（`orderCount` 而非 `cnt`）
- [ ] 函数名是否是动词短语（`GetUserByID` / `getUserById` / `get_user_by_id`）
- [ ] bool 变量是否以 `is/has/can/should` 开头
- [ ] 常量是否按语言规范命名（Go: PascalCase, Java: UPPER_SNAKE, Python: UPPER_SNAKE）
- [ ] 文件名是否反映了内容（`user_service.go` / `UserService.java` / `user_service.py` / `user_service.rs`）
- [ ] 是否存在 `data1, data2, result1, result2` 这类编号命名

### 深挖方法
1. 找到每个可疑命名
2. 追踪该变量在函数中的使用链路，确认是否能从上下文理解含义
3. 给出具体的重命名建议

### 场景推演

对每个模糊命名（如 `data`、`result`、`info`），追问：

- **六个月后你还能看懂吗？** 如果一个变量叫 `data`，你需要在函数内来回滚动才能理解它是什么，那就该重命名
- **代码审查时会不会误解？** `result` 到底是成功结果还是错误结果？`info` 是用户信息还是订单信息？
- **grep 能精准定位吗？** 搜 `data` 会匹配到满屏结果，搜 `orderConfirmData` 只会匹配到你想要的地方

> **深挖信号**：如果一个函数内有多个 `data`/`result`/`info` 变量，这通常意味着函数做了太多事，应结合 #2 函数职责单一一起审视。

> 一个新同事看到 `result2`，能否在不看上下文的情况下知道它代表什么？3个月后你自己还能看懂吗？

---

## 2. 函数职责单一

### 搜索模式

```bash
# Go：搜索函数定义
grep -rn 'func ' --include='*.go'
# Java：搜索方法定义
grep -rn 'public\|private\|protected' --include='*.java' | grep '\('
# Python：搜索函数/方法定义
grep -rn 'def ' --include='*.py'
# Rust：搜索函数定义
grep -rn 'fn ' --include='*.rs'
# TypeScript：搜索函数定义
grep -rn 'function \|=>\|async ' --include='*.ts' --include='*.tsx'

# 多职责信号：函数名中包含 and/or
grep -rni '\w[Aa]nd\w\|\w[Oo]r\w' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts' | grep -i 'func\|def\|fn\|function\|public\|private'
```

### 检查清单
- [ ] 函数是否超过 50 行（超过需警惕）
- [ ] 函数名是否暗示多个动作（`CreateAndSendEmail` / `create_and_send_email`）
- [ ] 函数内是否有不同抽象层级的操作混在一起
- [ ] 函数内是否有一段代码可以提取为独立函数并复用

### 深挖方法
1. 对超过50行的函数，逐段标注每段做什么
2. 如果一段代码超过10行且与前后逻辑独立，建议提取
3. 给出拆分后的函数签名示例

### 场景推演

对每个超长函数，追问：

- **改其中一个职责时，会不会误伤另一个？** 比如 `CreateAndSendOrder` 里改了发送逻辑，不小心影响了创建逻辑
- **测试时能单独测其中一个行为吗？** 如果不能，说明耦合太紧
- **调用方真的需要同时做这两件事吗？** 有些场景只需要创建不需要发送

> **深挖信号**：函数内出现"段落注释"（`// 第一步：创建订单`、`// 第二步：发送通知`），说明它在做两件独立的事。如果段落之间有共享状态（临时变量），那拆分时要特别注意数据传递方式。结合 #13 分层 检查是否不同抽象层级的操作混在了一起。

> 这个函数需要修改时，是否会因为多种职责混合在一起，导致改一处影响另一处？如果函数报错了，你能从函数名快速定位是哪个职责出了问题吗？

---

## 3. 参数精简

### 搜索模式

```bash
# Go：4个及以上参数的函数
grep -rn 'func\s\+\w\+\s*(.\+,.\+,.\+,.\+)' --include='*.go'
# Java：多参数方法
grep -rn 'void\s\+\w\+(.\+,.\+,.\+,.\+)' --include='*.java'
# Python：多参数函数
grep -rn 'def\s\+\w\+(.\+,.\+,.\+,.\+)' --include='*.py'
# Rust：多参数函数
grep -rn 'fn\s\+\w\+(.\+,.\+,.\+,.\+)' --include='*.rs'
# TypeScript：多参数函数
grep -rn 'function\s\+\w\+(.\+,.\+,.\+,.\+)' --include='*.ts'

# bool 参数（常用于控制分支）
# Go
grep -rn 'func\s\+.*bool\b' --include='*.go'
# Java
grep -rn 'boolean\b\|Boolean\b' --include='*.java' | grep '('
# Python
grep -rn 'def\s\+.*bool\b' --include='*.py'
```

### 检查清单
- [ ] 参数是否超过3个
- [ ] 是否有 bool 参数控制 if-else 分支（应拆为两个函数）
- [ ] 同一组参数是否在多个函数中重复出现（应封装为结构体/DTO/dataclass）
- [ ] 参数顺序是否符合语言惯例（Go: ctx 在前, Java: request 在前, Python: self 在前）

### 深挖方法
1. 对超过3个参数的函数，分析哪些参数天然属于一组
2. 给出封装建议：Go 用 struct, Java 用 DTO/RequestObject, Python 用 dataclass, Rust 用 struct
3. 对 bool 参数，给出拆分后的两个函数签名

### 场景推演

- **调用方传错参数时，编译器能发现吗？** `SendEmail(user, subject, body, true, false)` —— 第4个和第5个 bool 参数调换位置，编译器不会报错，但行为完全不同
- **新增参数时，所有调用方都要改吗？** 参数越多，新增参数的改动面越大
- **bool 参数的两个分支，错误处理一样吗？** 如果不一样，说明本来就该是两个函数

> **深挖信号**：如果同一组参数在多个函数中反复出现（如 `userID, orderID, amount`），这不仅是一个参数精简问题，还可能意味着缺少一个领域对象（如 `OrderContext`）。结合 #13 分层 和 #14 面向接口 检查是否缺少必要的抽象。

> 调用方写 `CreateOrder(user, items, true, false, nil, "express")` 时，能看懂每个参数的含义吗？如果第5个参数要改成必填，所有调用方都要改吗？

---

## 4. 控制流扁平化

### 搜索模式

```bash
# Go：深层嵌套（3层 tab 或 8+ spaces）
grep -rn '^\t\t\t' --include='*.go'
# Java
grep -rn '^\t\t\t' --include='*.java'
# Python（Python 用缩进定义作用域，嵌套尤其致命）
grep -rn '^            ' --include='*.py'  # 3层缩进（12 spaces）

# else 分支
grep -rn '} else {' --include='*.go' --include='*.java'
grep -rn 'else:' --include='*.py'
```

### 检查清单
- [ ] 是否存在超过2层嵌套的 if-else
- [ ] 是否可以提前 return/break/continue 减少嵌套
- [ ] 错误处理是否在正常逻辑之前（guard clause 模式）
- [ ] 是否有复杂的条件表达式可提取为有意义的 bool 变量

### 深挖方法
1. 对每个深层嵌套，画出控制流结构
2. 展示如何用 guard clause 重写
3. 对比重写前后的可读性

**Go 示例：**
```go
// Bad
if err == nil {
    if user != nil {
        // happy path
    }
}

// Good
if err != nil { return err }
if user == nil { return nil }
// happy path
```

**Java 示例：**
```java
// Bad
if (result != null) {
    if (result.isValid()) {
        // happy path
    }
}

// Good
if (result == null) { return; }
if (!result.isValid()) { return; }
// happy path
```

**Python 示例：**
```python
# Bad
if data is not None:
    if data.is_valid():
        # happy path

# Good
if data is None:
    return
if not data.is_valid():
    return
# happy path
```

### 场景推演

对每个深层嵌套，追问：

- **每个分支的异常路径都处理了吗？** 嵌套越深，越容易遗漏某个分支的错误处理
- **happy path 是不是藏在了最内层？** 如果读代码需要滚动到最深处才能看到正常逻辑，说明嵌套有问题
- **哪个分支最可能出 bug？** 嵌套中的 else 分支往往是最少被测试的，也最容易藏 bug

> **深挖信号**：深层嵌套往往伴随着隐含的业务规则。`if A then if B then if C` 实际上是一个业务条件 `A && B && C`，但它被拆成了三层。这不仅影响可读性，还意味着如果 B 失败了，开发者可能忘记处理。结合 #28 易错点自问 检查每个分支的错误路径是否都有处理。

> 这段嵌套代码有 4 层 if，如果要在最内层加一个条件，你要在第几层加？加完后可读性会怎样？

---

## 5. 消除魔法值

### 搜索模式

```bash
# 通用：搜索硬编码数字（排除 0, 1, -1）
grep -rn '[^a-zA-Z_][2-9][0-9]*[^a-zA-Z_0-9.]' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
# 通用：搜索状态码硬编码
grep -rn '== [0-9]\|!= [0-9]' --include='*.go' --include='*.java' --include='*.py' --include='*.rs' --include='*.ts'
# Java：搜索硬编码字符串常量
grep -rn '= "' --include='*.java' | grep -v 'final\|static final'
# Python：搜索硬编码字符串
grep -rn '= "' --include='*.py' | grep -v 'CLASS\|ENUM'
```

### 检查清单
- [ ] 数字常量是否有命名（`maxRetryCount` 而非 `3`）
- [ ] 状态/类型是否用枚举/常量组定义
- [ ] 错误码是否有常量定义
- [ ] 超时时间、重试次数等配置值是否可配置
- [ ] Go: 是否用 `const` 或 `iota`；Java: 是否用 `enum`；Python: 是否用 `Enum` 类

### 深挖方法
1. 列出所有硬编码值
2. 分析哪些是真正的魔法数字，哪些是数学上固定的（如 `*2`, `/2`）
3. 给出对应语言的常量/枚举定义代码

### 场景推演

对每个魔法值，追问：

- **业务规则变了，要改几个地方？** 状态码 `3` 出现在 5 个文件中，业务说"已取消"要改成 `4`，你能确保全部改到吗？
- **这个数字的含义，新人能看懂吗？** `if status == 3` vs `if status == OrderStatus.Cancelled`
- **这个超时时间在不同环境下都合理吗？** 本地测试 3 秒够了，生产环境可能需要 10 秒。硬编码意味着无法按环境调整

> **深挖信号**：如果硬编码的数字用于状态判断，检查是否有遗漏的状态值。比如代码只处理了 `status == 1` 和 `status == 2`，但数据库里实际有 3 种状态，第 3 种状态会被默认逻辑错误处理。结合 #10 数据增长敏感性 和 #28 易错点自问 中的"边界"检查。

> `if status == 3` 这个 3 代表什么？如果要改成 4，你确定能找到所有该改的地方吗？如果另一个开发者也用了 `3` 但含义不同呢？

---

## 6. 代码复用与去重

> **核心关注点**：消除复制粘贴式的重复代码，提炼通用的逻辑和模式，降低维护成本（一处修改多处生效）。

### 搜索模式

```bash
# 搜索相似的错误处理模式（重复的 if err != nil / try-catch）
grep -rn 'if err != nil' --include='*.go' | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -20
grep -rn 'catch (' --include='*.java' | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -20
grep -rn 'except ' --include='*.py' | awk -F: '{print $1}' | sort | uniq -c | sort -rn | head -20

# 搜索重复的参数校验逻辑
grep -rn 'required\|Required\|@NotNull\|@NotBlank\|validate\|Validate' --include='*.go' --include='*.java' --include='*.py' --include='*.ts'

# 搜索重复的 CRUD 模板（相似的 Create/Update/Delete 函数签名）
grep -rn 'func.*Create\|func.*Update\|func.*Delete' --include='*.go'
grep -rn 'public.*create\|public.*update\|public.*delete' --include='*.java'
grep -rn 'def.*create\|def.*update\|def.*delete' --include='*.py'

# 搜索重复的配置/常量定义
grep -rn 'const\|static final\|MAX_\|MIN_\|DEFAULT_' --include='*.go' --include='*.java' --include='*.py'

# 搜索重复的工具函数（同一个操作在多处实现）
grep -rn 'func.*Is\|func.*Has\|func.*Check\|func.*Parse\|func.*Format' --include='*.go'
grep -rn 'public.*is\|public.*has\|public.*check\|public.*parse\|public.*format' --include='*.java'
grep -rn 'def.*is_\|def.*has_\|def.*check_\|def.*parse_\|def.*format_' --include='*.py'
```

### 检查清单

#### 代码块重复
- [ ] 是否有复制粘贴的代码块（3行以上相同或高度相似）
- [ ] 多个函数中是否有相同的错误处理模式（应抽取中间件或公共包装器）
- [ ] 多个接口中是否有相同的参数校验逻辑（应抽取公共校验函数）

#### 业务逻辑重复
- [ ] 同一业务规则是否在多个地方重复实现（如订单状态判断、权限检查）
- [ ] 相似的分支条件是否出现在多个函数中（应统一为策略/状态机）
- [ ] 数据转换逻辑（DTO ↔ Model）是否有大量重复的字段映射

#### 模板代码冗余
- [ ] CRUD 操作是否每个实体都写一遍相同的模板代码（应考虑泛型基类/代码生成）
- [ ] Repository/DAO 层是否有大量相似的增删改查方法
- [ ] 是否可以用泛型/模板方法模式减少重复

#### 配置与定义重复
- [ ] 相同的常量/配置是否在多个文件中重复定义
- [ ] 相同的错误码/状态码是否分散在多处
- [ ] 相同的 struct/class 定义是否可以合并

### 深挖方法

1. **定位重复**：对以上搜索结果，按文件分组统计，高频模式即为重复信号

2. **分析复用价值**（不是所有重复都需要消除）：
   - 两处相同且业务相关度高 → 必须抽取
   - 两处相似但上下文不同 → 先合并抽象再看
   - 两处相同但各自独立演进的可能性大 → 可以暂时保留

3. **给出重构建议**：
   - 重复代码块 → 提取为公共函数，参数化差异部分
   - 重复错误处理 → 抽取中间件或 `WrapHandler` 包装器
   - 重复校验 → 抽取公共校验函数或校验标签
   - 重复业务逻辑 → 统一到 Service 层或领域模型中
   - 重复 CRUD → 用泛型基类（Go: 泛型 + 接口; Java: 泛型 Repository; Python: 抽象基类）
   - 重复常量 → 集中到 constants 包/类中

### 场景推演

对每组重复代码，追问：

- **修改一边忘了改另一边，会发生什么？** 这是重复代码最大的风险。比如支付校验逻辑在两个 Service 里各有一份，后来加了优惠码校验但只改了一处
- **两份"相同"的代码，细节上有没有微妙差异？** 看起来一样的代码，可能在边界条件处理、错误码返回上有细微差别。这种差异往往是有意为之（合并时要保留）还是无意产生的（合并时以哪份为准？）
- **重复的业务规则是否意味着领域模型缺失？** 如果"订单状态流转"的逻辑散落在 5 个地方，可能不是代码复用的问题，而是缺少一个 `OrderStateMachine` 领域对象

> **深挖信号**：重复代码最危险的不是维护成本，而是**一致性风险**。当一份重复的业务逻辑被修改了但另一份没有，就产生了隐性 bug。结合 #16 幂等设计 检查重复的写操作逻辑是否一致，结合 #28 易错点自问 检查重复的边界条件处理是否一致。

> 这段校验逻辑在3个文件里各有一份。现在业务规则变了（比如手机号要支持+86前缀），你能确保3处都改了吗？如果漏改了一处呢？

---

## 常见缺失对照表

| 看到 | 必须检查是否有 | 缺失则标记 |
|------|---------------|-----------|
| 函数名含 `and`/`or` | 是否拆分为单一职责函数 | ⚠️ 多职责函数 |
| 3个以上参数 | 是否封装为结构体/DTO | ⚠️ 参数过多 |
| `true`/`false` 作为参数 | 是否拆为两个语义明确的函数 | ⚠️ 布尔陷阱 |
| 3层以上嵌套 | 是否可用 guard clause 扁平化 | ⚠️ 箭头代码 |
| 裸数字 `== 3` / `>= 200` | 是否提取为命名常量/枚举 | ⚠️ 魔法值 |
| 相似代码块出现3次+ | 是否提取为公共函数 | ⚠️ 重复代码 |

---

## 跨维度关联提示

审查完代码基础维度后，检查以下关联风险：

| 关联信号 | 指向的维度 | 要追问的问题 |
|----------|-----------|-------------|
| 魔法值用于状态判断 | #8 并发安全、#16 幂等 | 状态流转是否原子？状态值变更会不会导致逻辑分支覆盖不全？ |
| 函数职责混合了业务逻辑和外部调用 | #9 事务边界、#17 限流熔断 | 拆分后，外部调用是否在事务外？是否有超时和熔断？ |
| 重复的业务规则散落多处 | #25 测试、#16 幂等 | 修改时能保证所有副本一致吗？测试覆盖了所有副本吗？ |
| 深层嵌套包含错误处理 | #12 日志、#19 资源释放 | 内层分支的错误是否被正确处理和记录？资源是否在所有分支都释放了？ |
| bool 参数控制不同业务分支 | #16 幂等 | 两个分支的幂等策略是否一致？调用方传错参数会怎样？ |
| 硬编码的超时/重试配置 | #18 重试策略 | 超时值是否可按环境调整？重试次数是否合理？ |
