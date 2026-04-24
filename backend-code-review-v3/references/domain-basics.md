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
