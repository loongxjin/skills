# 常见反模式速查

发现以下代码 pattern 时，直接匹配对应反模式，给出定位和修复。

---

## 代码质量

### #3 布尔陷阱
```java
// ❌ 反模式 — 调用方看不懂 true/false 含义
sendEmail(user, subject, body, true, false);

// ✅ 修复 — 拆分为语义明确的函数
sendEmailWithAttachment(user, subject, body, attachment);
sendEmailWithoutAttachment(user, subject, body);
// 或用配置对象
sendEmail(user, subject, body, SendOptions{WithAttachment: true});
```

### #4 箭头代码（深层嵌套）
```python
# ❌ 反模式
if user:
    if user.active:
        if user.balance > 0:
            process_order(order)

# ✅ 修复 — Guard Clause
if not user or not user.active:
    raise Error("invalid user")
if user.balance <= 0:
    raise Error("insufficient balance")
process_order(order)
```

### #7 依赖具体实现
```go
// ❌ 反模式 — 业务层直接依赖 MySQL
func (s *OrderService) Create(o Order) {
    db := mysql.NewClient("localhost:3306")  // 硬编码 + 无法 mock
    db.Insert(o)
}

// ✅ 修复 — 依赖接口
func (s *OrderService) Create(o Order) {
    s.repo.Save(o)  // repo 是接口，测试时注入 mock
}
```

---

## 数据与存储

### #12 事务内 IO
```java
// ❌ 反模式 — RPC 在事务内，长事务 + 锁持有时间过长
@Transactional
public void order() {
    orderDao.create(order);
    paymentService.charge(order);  // ❌ HTTP/RPC 调用！
    inventoryDao.deduct(order.sku);
}

// ✅ 修复 — 事务只包含 DB 操作，RPC 放外面
public void order() {
    paymentService.charge(order);  // 先 RPC
    txManager.run(() -> {          // 再事务
        orderDao.create(order);
        inventoryDao.deduct(order.sku);
    });
}
```

### #15 不安全 Schema 变更
```sql
-- ❌ 反模式 — 直接删字段，旧代码会崩溃
ALTER TABLE orders DROP COLUMN status;

-- ✅ 修复 — 分阶段灰度
-- Step 1: 新代码不再读写该字段（代码兼容）
-- Step 2: 确认无旧代码依赖后，标记字段 deprecated
-- Step 3: 下一版本才真正删除
```

### #16 缓存更新（非失效）
```go
// ❌ 反模式 — 高并发下可能写入旧值
func UpdateUser(u User) {
    db.Update(u)
    cache.Set("user:"+u.ID, u)  // A、B 并发更新，后写入的可能是旧值
}

// ✅ 修复 — 更新 DB 后删除缓存，下次读取加载最新值
func UpdateUser(u User) {
    db.Update(u)
    cache.Del("user:" + u.ID)
}
```

---

## 并发与分布式

### #17 TOCTOU（先查后改）
```go
// ❌ 反模式 — 查询和修改不是原子操作
func DeductStock(sku string, qty int) error {
    stock := repo.GetStock(sku)      // 此时 stock=10
    if stock.Count >= qty {          // A 线程判断通过
        repo.Deduct(sku, qty)        // B 线程同时扣减，A 再扣就变成负数
    }
}

// ✅ 修复 1 — 单条原子 SQL
func DeductStock(sku string, qty int) error {
    affected := repo.Exec(
        "UPDATE stock SET count = count - ? WHERE sku = ? AND count >= ?",
        qty, sku, qty,
    )
    if affected == 0 { return Error("insufficient stock") }
}

// ✅ 修复 2 — SELECT FOR UPDATE
func DeductStock(sku string, qty int) error {
    return repo.WithTx(func(tx) error {
        stock := tx.LockStock(sku)   // SELECT ... FOR UPDATE
        if stock.Count < qty { return Error("insufficient stock") }
        return tx.Deduct(sku, qty)
    })
}
```

### #18 无幂等保护
```go
// ❌ 反模式 — 重复调用会导致重复扣款/发货
func Pay(orderID string, amount int) error {
    db.Exec("UPDATE orders SET status='paid' WHERE id=?", orderID)
    wallet.Deduct(amount)              // 重复调用会重复扣钱！
}

// ✅ 修复 — 唯一请求 ID + 数据库唯一索引去重
func Pay(orderID string, amount int, reqID string) error {
    _, err := db.Exec("INSERT INTO idempotent_log (req_id) VALUES (?)", reqID)
    if isDuplicate(err) { return nil }  // 已处理过，直接返回
    db.Exec("UPDATE orders SET status='paid' WHERE id=?", orderID)
    wallet.Deduct(amount)
}
```

### #19 外部调用无超时
```go
// ❌ 反模式 — 默认超时无限长，下游挂了会拖死自己
resp, err := http.Get("http://payment.api/charge")

// ✅ 修复 — 必须设置超时和上下文
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
resp, err := http.NewRequestWithContext(ctx, "POST", url, body)
```

### #20 重试业务错误
```python
# ❌ 反模式 — 参数错误也重试，浪费资源且放大问题
for i in range(3):
    resp = api.call(params)
    if resp.code != 0:          # ❌ 400 Bad Request 也重试！
        time.sleep(1)
        continue

# ✅ 修复 — 只重试可重试的错误，指数退避
for i in range(3):
    resp = api.call(params)
    if resp.code == 0: break
    if not is_retryable(resp.code):  # 400/401/403 不重试
        raise Error(resp.msg)
    time.sleep((2 ** i) + random())  # 指数退避 + 抖动
```

---

## 安全

### #27 SQL 注入
```java
// ❌ 反模式 — 字符串拼接 SQL
String sql = "SELECT * FROM users WHERE name = '" + name + "'";

// ✅ 修复 — 参数化查询
String sql = "SELECT * FROM users WHERE name = ?";
jdbcTemplate.query(sql, rs -> ..., name);
```

### #28 缺失权限校验
```go
// ❌ 反模式 — 敏感操作无鉴权
func DeleteUser(userID string) {
    db.Delete(userID)  // 任何人都能调！
}

// ✅ 修复 — 入口处校验权限
func DeleteUser(ctx Context, userID string) error {
    if !ctx.HasPermission("user:delete") {
        return Error("forbidden")
    }
    db.Delete(userID)
}
```

### #29 明文存储密码
```python
# ❌ 反模式
user.password = request.password  # 明文存入数据库
db.save(user)

# ✅ 修复 — 加盐哈希
import bcrypt
user.password_hash = bcrypt.hashpw(
    request.password.encode(), bcrypt.gensalt()
)
db.save(user)
```

---

## 可观测性

### #23 日志打印敏感信息
```go
// ❌ 反模式
log.Printf("login failed: user=%s password=%s", user, password)

// ✅ 修复 — 脱敏或只打 ID
log.Printf("login failed: user_id=%d", user.ID)
// 敏感信息进安全审计日志，与业务日志分离
```

### #26 无优雅退出
```go
// ❌ 反模式 — 收到信号直接退出，进行中的请求被中断
func main() {
    http.ListenAndServe(":8080", handler)
}

// ✅ 修复 — 处理完进行中的请求再退出
func main() {
    srv := &http.Server{Addr: ":8080", Handler: handler}
    go srv.ListenAndServe()

    sig := make(chan os.Signal, 1)
    signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
    <-sig

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)  // 优雅关闭
}
```

---

## 架构

### #9 跨层调用
```java
// ❌ 反模式 — Controller 直接操作数据库，跳过 Service 层
@RestController
public class OrderController {
    @Autowired
    private JdbcTemplate jdbcTemplate;  // ❌ Controller 不应直接访问数据库

    @PostMapping("/orders")
    public Result create(@RequestBody Order order) {
        jdbcTemplate.update("INSERT INTO orders ...", order.getId());
        return Result.ok();
    }
}

// ✅ 修复 — Controller → Service → Repository
@RestController
public class OrderController {
    @Autowired
    private OrderService orderService;  // 只依赖 Service

    @PostMapping("/orders")
    public Result create(@RequestBody Order order) {
        orderService.create(order);  // 业务逻辑在 Service 层
        return Result.ok();
    }
}
```

### #11 N+1 查询
```go
// ❌ 反模式 — 循环内查询，100 个订单 = 101 次 SQL
func GetOrdersWithUsers(orderIDs []string) ([]OrderWithUser, error) {
    var result []OrderWithUser
    for _, id := range orderIDs {
        order := db.GetOrder(id)    // 第 1 次
        user := db.GetUser(order.UserID)  // 第 2~101 次！
        result = append(result, OrderWithUser{order, user})
    }
    return result, nil
}

// ✅ 修复 — 批量查询 + 内存关联
func GetOrdersWithUsers(orderIDs []string) ([]OrderWithUser, error) {
    orders := db.GetOrdersByIDs(orderIDs)         // 1 次
    userIDs := extractUserIDs(orders)
    users := db.GetUsersByIDs(userIDs)            // 1 次
    userMap := toUserMap(users)                    // O(n) 构建 map
    return join(orders, userMap), nil              // 内存关联
}
```

### #21 消息丢失
```go
// ❌ 反模式 — 消息消费失败直接丢弃，无 DLQ，无重试
func HandleOrderCreated(msg *Message) error {
    order, err := parseOrder(msg.Body)
    if err != nil {
        log.Printf("parse failed: %v", err)
        return nil  // ❌ 消息被确认，数据永久丢失！
    }
    return processOrder(order)
}

// ✅ 修复 — 失败进死信队列 + 告警
func HandleOrderCreated(msg *Message) error {
    order, err := parseOrder(msg.Body)
    if err != nil {
        log.Printf("parse failed: %v, sending to DLQ", err)
        dlq.Publish(msg)  // 进死信队列，后续可人工处理或对账
        alert("order parse failed", msg)
        return nil
    }
    return processOrder(order)
}
```

---

## 工程实践

### #33 float 金额计算
```python
# ❌ 反模式 — float 精度丢失
price = 0.1 + 0.2  # 结果是 0.30000000000000004
assert price == 0.3  # ❌ AssertionError！
total = 999999.99 * 100  # 大数时精度更离谱

# ✅ 修复 — 使用 decimal
from decimal import Decimal
price = Decimal('0.1') + Decimal('0.2')  # 0.3
assert price == Decimal('0.3')  # ✅
# Go 使用 int64（以分为单位）或 shopspring/decimal
# Java 使用 BigDecimal
// Java:
BigDecimal price = new BigDecimal("0.1").add(new BigDecimal("0.2"));
```

### #33 整数溢出
```go
// ❌ 反模式 — int32 存订单 ID，超过 21 亿后溢出变负数
type Order struct {
    ID   int32  // ❌ 21 亿后溢出
    // ...
}

// ✅ 修复 — 使用 int64
// Go:
type Order struct {
    ID   int64  // ✅ 9.2 * 10^18，足够安全
}
// Java: 使用 Long 而非 Integer
// Python: 天然支持大整数，但存储到 DB 时需注意字段类型
```
