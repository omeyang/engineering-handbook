# 函数设计规范

本文档定义 Go 项目中函数的设计原则，包括职责划分、参数设计、返回值、副作用管理、组合与拆分策略。

---

## 核心原则

1. **做且只做一件事** - 函数**必须**只完成一个明确的任务
2. **抽象层次一致** - 函数体内的操作**必须**处于同一抽象层次
3. **命令-查询分离** - 函数**应该**要么执行动作，要么返回数据，不同时做两件事
4. **最小惊讶** - 函数的行为**必须**与名称一致，不产生意外副作用
5. **可测试性优先** - 函数设计**必须**便于单元测试

---

## 函数职责

### 单一职责判定

函数是否"只做一件事"的判定方法：

**方法 1：能否再拆？**

如果函数中的某段逻辑可以提取为一个有意义命名的子函数，说明原函数做了不止一件事。

```go
// 错误：做了两件事（验证 + 保存）
func CreateOrder(ctx context.Context, req *CreateOrderRequest) error {
    // 验证逻辑（20行）
    if req.UserID == "" {
        return errors.New("user_id is required")
    }
    if len(req.Items) == 0 {
        return errors.New("items cannot be empty")
    }
    for _, item := range req.Items {
        if item.Quantity <= 0 {
            return fmt.Errorf("invalid quantity for item %s", item.ID)
        }
    }

    // 保存逻辑（15行）
    order := &Order{
        ID:     GenerateID(),
        UserID: req.UserID,
        Items:  req.Items,
    }
    return s.repo.Save(ctx, order)
}

// 正确：拆分为独立职责
func CreateOrder(ctx context.Context, req *CreateOrderRequest) error {
    if err := validateCreateOrderRequest(req); err != nil {
        return fmt.Errorf("validate request: %w", err)
    }
    order := newOrderFromRequest(req)
    return s.repo.Save(ctx, order)
}
```

**方法 2：能否用一句话描述？**

如果描述函数功能时必须用"并且"、"然后"、"同时"连接，说明做了不止一件事。

```go
// "验证订单请求并保存订单并发送通知" -> 三件事
// 应该拆分为三个函数
```

**方法 3：是否混合了不同抽象层次？**

如果函数中既有高层业务概念又有底层实现细节，说明职责混乱。详见"抽象层次一致性"小节。

### 函数大小

- 函数体**必须**不超过 70 行（不含注释和空行）
- 函数体**应该**不超过 40 行
- 超过 40 行的函数**应该**检查是否可以拆分

### 嵌套深度

- 函数嵌套深度**必须**不超过 3 层
- 超过 2 层嵌套**应该**使用提前返回或提取子函数

```go
// 错误：嵌套过深
func Process(items []*Item) error {
    for _, item := range items {
        if item.IsActive() {
            for _, rule := range item.Rules {
                if rule.Match() {
                    if err := rule.Apply(item); err != nil {
                        return err
                    }
                }
            }
        }
    }
    return nil
}

// 正确：提取子函数降低嵌套
func Process(items []*Item) error {
    for _, item := range items {
        if err := processItem(item); err != nil {
            return fmt.Errorf("process item %s: %w", item.ID, err)
        }
    }
    return nil
}

func processItem(item *Item) error {
    if !item.IsActive() {
        return nil
    }
    return applyMatchingRules(item, item.Rules)
}

func applyMatchingRules(item *Item, rules []*Rule) error {
    for _, rule := range rules {
        if !rule.Match() {
            continue
        }
        if err := rule.Apply(item); err != nil {
            return fmt.Errorf("apply rule %s: %w", rule.ID, err)
        }
    }
    return nil
}
```

---

## 抽象层次一致性

### 规则

同一个函数内的所有操作**必须**处于同一抽象层次。

高层函数协调流程，低层函数处理细节。**禁止**在同一函数中混合不同层次。

```go
// 错误：混合了"业务流程编排"和"数据库操作细节"
func PlaceOrder(ctx context.Context, req *PlaceOrderRequest) error {
    if err := validateOrder(req); err != nil {
        return err
    }

    // 突然跌落到底层实现细节
    collection := s.db.Database("shop").Collection("orders")
    _, err := collection.InsertOne(ctx, bson.M{
        "user_id":    req.UserID,
        "items":      req.Items,
        "created_at": time.Now(),
    })
    if err != nil {
        return fmt.Errorf("insert order: %w", err)
    }

    // 又回到高层
    return s.notifier.OrderPlaced(ctx, req.UserID)
}

// 正确：统一在"业务流程编排"层次
func PlaceOrder(ctx context.Context, req *PlaceOrderRequest) error {
    if err := validateOrder(req); err != nil {
        return err
    }
    order := newOrder(req)
    if err := s.repo.Save(ctx, order); err != nil {
        return fmt.Errorf("save order: %w", err)
    }
    return s.notifier.OrderPlaced(ctx, order.UserID)
}
```

### 抽象层次的三层模型

```
┌─────────────────────────────────────┐
│  编排层：协调业务流程               │  validateOrder → saveOrder → notify
├─────────────────────────────────────┤
│  操作层：执行具体业务操作           │  repo.Save, cache.Invalidate
├─────────────────────────────────────┤
│  实现层：底层技术细节               │  collection.InsertOne, redis.Set
└─────────────────────────────────────┘
```

每个函数**应该**只在一个层次工作，通过调用下一层函数来完成工作。

---

## 命令-查询分离（CQS）

### 规则

- **命令函数**（Command）：执行动作，改变状态，返回 `error` 或不返回
- **查询函数**（Query）：返回数据，**禁止**修改状态

函数**应该**要么是命令，要么是查询，**不应该**同时是两者。

```go
// 错误：既修改状态又返回数据
func GetAndUpdateUser(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }
    user.LastAccessAt = time.Now()
    if err := s.repo.Update(ctx, user); err != nil {
        return nil, err
    }
    return user, nil
}

// 正确：分离命令和查询
func GetUser(ctx context.Context, id string) (*User, error) {
    return s.repo.FindByID(ctx, id)  // 查询：只读
}

func RecordUserAccess(ctx context.Context, id string) error {
    return s.repo.UpdateLastAccess(ctx, id, time.Now())  // 命令：只写
}
```

### 允许的例外

以下场景**可以**违反 CQS：

1. **栈/队列操作**：`Pop()` 必须同时移除并返回元素
2. **原子操作**：`CompareAndSwap`、`GetAndDelete` 等需要原子性的操作
3. **生成操作**：`GenerateID()` 创建并返回新 ID

违反 CQS 时**必须**在函数名中体现（如 `PopFirst`、`GetAndDelete`）。

---

## 参数设计

### 参数数量

- 参数**必须**不超过 3 个（不含 `context.Context`）
- 超过 3 个参数**必须**使用结构体

```go
// 错误：参数过多
func CreateUser(ctx context.Context, name, email, phone string, age int, role Role) error

// 正确：使用结构体
func CreateUser(ctx context.Context, req *CreateUserRequest) error

type CreateUserRequest struct {
    Name  string
    Email string
    Phone string
    Age   int
    Role  Role
}
```

### 参数顺序

参数**应该**按以下顺序排列：

1. `context.Context`（始终第一个）
2. 主要输入（操作的对象）
3. 配置/选项（控制行为）

```go
// 正确的参数顺序
func UpdateOrder(ctx context.Context, id string, opts ...UpdateOption) error
func SendMessage(ctx context.Context, msg *Message, opts ...SendOption) error
```

### 禁止输出参数

**禁止**使用指针参数作为输出（out parameter）。**必须**使用返回值。

```go
// 错误：输出参数
func ParseConfig(data []byte, result *Config) error {
    // 通过指针修改 result
}

// 正确：使用返回值
func ParseConfig(data []byte) (*Config, error) {
    // 返回结果
}
```

### Functional Options 模式

当函数有可选配置时，**应该**使用 Functional Options：

```go
type ServerOption func(*Server)

func WithPort(port int) ServerOption {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) ServerOption {
    return func(s *Server) { s.timeout = d }
}

func NewServer(addr string, opts ...ServerOption) *Server {
    s := &Server{addr: addr, port: 8080, timeout: 30 * time.Second}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### 布尔参数

**禁止**使用裸布尔参数。调用处无法理解含义。

```go
// 错误：调用处看不懂 true 是什么意思
s.Process(ctx, order, true)

// 正确：使用具名类型或 Option
s.Process(ctx, order, WithForceRetry())

// 或使用枚举
type RetryPolicy int
const (
    NoRetry    RetryPolicy = iota
    ForceRetry
)
s.Process(ctx, order, ForceRetry)
```

---

## 返回值设计

### 返回值规则

- 返回值**必须**不超过 3 个（包含 error）
- `error` **必须**是最后一个返回值
- 有 `error` 返回值时，调用方**必须**检查

### 零值可用

函数在错误时**应该**返回类型的零值：

```go
// 正确：错误时返回零值
func FindUser(ctx context.Context, id string) (*User, error) {
    user, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("find user %s: %w", id, err)
    }
    return user, nil
}
```

### 命名返回值

**禁止**使用命名返回值来"省代码"。**可以**在以下场景使用命名返回值：

1. 多个相同类型的返回值需要区分含义
2. `defer` 中需要修改返回值

```go
// 允许：defer 需要修改返回值
func ReadFile(path string) (content []byte, err error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("open %s: %w", path, err)
    }
    defer func() {
        if closeErr := f.Close(); closeErr != nil && err == nil {
            err = fmt.Errorf("close %s: %w", path, closeErr)
        }
    }()
    return io.ReadAll(f)
}
```

---

## 提前返回（Guard Clause）

### 规则

**应该**使用 Guard Clause 提前处理异常情况，保持主逻辑在最低缩进级别。

```go
// 错误：主逻辑嵌套在条件中
func ProcessOrder(ctx context.Context, order *Order) error {
    if order != nil {
        if order.Status == StatusPending {
            if len(order.Items) > 0 {
                // 主逻辑深埋在三层嵌套中
                return s.fulfill(ctx, order)
            } else {
                return ErrEmptyOrder
            }
        } else {
            return ErrInvalidStatus
        }
    } else {
        return ErrNilOrder
    }
}

// 正确：Guard Clause 提前返回
func ProcessOrder(ctx context.Context, order *Order) error {
    if order == nil {
        return ErrNilOrder
    }
    if order.Status != StatusPending {
        return ErrInvalidStatus
    }
    if len(order.Items) == 0 {
        return ErrEmptyOrder
    }
    return s.fulfill(ctx, order)
}
```

### Guard Clause 的顺序

Guard Clause **应该**按以下顺序排列：

1. nil 检查
2. 权限/状态检查
3. 业务规则检查
4. 主逻辑

---

## 副作用管理

### 什么是副作用

副作用是指函数执行时对外部状态产生的可观察变化：

- 修改全局变量或包级变量
- 修改传入的引用参数
- 写文件、发网络请求
- 写日志、发消息、发通知

### 规则

- 纯计算函数**禁止**有副作用
- 有副作用的函数**必须**在命名中体现（动词开头：`Save`、`Send`、`Update`）
- 副作用**应该**集中在函数的末尾，而非分散在逻辑中间

```go
// 错误：副作用分散在逻辑中间
func CalculateDiscount(order *Order) float64 {
    log.Info("calculating discount", "order_id", order.ID)  // 副作用
    discount := 0.0
    if order.Total > 1000 {
        discount = 0.1
        s.metrics.RecordDiscount(order.ID, discount)  // 副作用
    }
    return discount
}

// 正确：纯计算，无副作用
func CalculateDiscount(order *Order) float64 {
    if order.Total > 1000 {
        return 0.1
    }
    return 0.0
}
```

### 优先纯函数

相同输入始终产生相同输出、无副作用的函数称为**纯函数**。

- 计算逻辑**应该**提取为纯函数
- 纯函数天然适合单元测试和并发安全
- 业务核心逻辑**应该**尽可能设计为纯函数

```go
// 纯函数：匹配规则判定
func MatchRule(order *Order, rule *Rule) bool {
    return rule.MinAmount <= order.Total && order.Total <= rule.MaxAmount
}

// 非纯函数：只在需要副作用的地方出现
func ApplyRule(ctx context.Context, order *Order, rule *Rule) error {
    if !MatchRule(order, rule) {
        return nil
    }
    return s.repo.SaveMatch(ctx, order.ID, rule.ID)
}
```

---

## 函数组合与拆分

### 何时拆分

出现以下信号时**应该**拆分函数：

| 信号 | 说明 |
|------|------|
| 超过 40 行 | 可能职责过多 |
| 超过 2 层嵌套 | 逻辑过于复杂 |
| 需要注释分隔逻辑块 | 每个块应该是独立函数 |
| 参数超过 3 个 | 职责可能过宽 |
| 多个 if-else 分支 | 考虑策略模式或查表 |
| 描述需要"并且" | 做了不止一件事 |

### 拆分策略

**策略 1：按阶段拆分**

```go
// 原函数：验证 -> 转换 -> 保存 -> 通知
func HandleOrder(ctx context.Context, req *Request) error {
    if err := validateRequest(req); err != nil {
        return err
    }
    order := convertToOrder(req)
    if err := saveOrder(ctx, order); err != nil {
        return err
    }
    return notifyOrderCreated(ctx, order)
}
```

**策略 2：按条件拆分**

```go
// 错误：长 switch
func process(eventType string, data []byte) error {
    switch eventType {
    case "create":
        // 20 行
    case "update":
        // 25 行
    case "delete":
        // 15 行
    }
}

// 正确：每个分支独立函数
func process(eventType string, data []byte) error {
    switch eventType {
    case "create":
        return processCreate(data)
    case "update":
        return processUpdate(data)
    case "delete":
        return processDelete(data)
    default:
        return fmt.Errorf("unknown event type: %s", eventType)
    }
}
```

**策略 3：提取纯计算**

将可测试的纯计算从有副作用的流程中提取出来。

```go
// 原函数：计算和保存混在一起
func UpdateOrderTotal(ctx context.Context, order *Order) error {
    total := calculateTotal(order.Items)      // 纯计算，提取
    discount := calculateDiscount(total)       // 纯计算，提取
    order.Total = total - discount
    return s.repo.Update(ctx, order)
}
```

### 何时不拆分

- 拆分后的子函数只有一个调用点，且逻辑简单（< 10 行）
- 拆分会破坏逻辑的完整性，导致理解困难
- 函数已经足够简单清晰

---

## 方法设计

### Receiver 选择

| 场景 | 类型 | 说明 |
|------|------|------|
| 需要修改状态 | `*T`（指针） | 方法需要改变 receiver 字段 |
| 大结构体 | `*T`（指针） | 避免拷贝开销 |
| 不修改状态且结构体小 | `T`（值） | 安全，无副作用 |
| 实现接口 | `*T`（指针） | 统一使用指针 receiver |

**规则**：同一类型的所有方法**必须**统一使用值 receiver 或指针 receiver，**禁止**混用。

### 方法与函数的选择

- 操作与类型强绑定 → 方法
- 操作独立于类型 → 函数
- 需要多态 → 方法（通过接口）

```go
// 方法：与 Order 强绑定
func (o *Order) CanCancel() bool {
    return o.Status == StatusPending
}

// 函数：独立的工具逻辑
func FormatPrice(amount float64) string {
    return fmt.Sprintf("%.2f", amount)
}
```

---

## 闭包设计

### 规则

- 闭包**应该**短小（不超过 15 行）
- 闭包**禁止**捕获可变的外部变量（除非有并发保护）
- 闭包**应该**只捕获必要的变量

```go
// 错误：闭包捕获可变变量，并发不安全
count := 0
for i := 0; i < n; i++ {
    go func() {
        count++  // 数据竞争
    }()
}

// 正确：通过参数传递
var count atomic.Int64
for i := 0; i < n; i++ {
    go func() {
        count.Add(1)
    }()
}
```

---

## 检查清单

### 职责检查

- [ ] 函数能用一句话（无"并且"）描述功能
- [ ] 函数体内操作处于同一抽象层次
- [ ] 函数不超过 70 行，**应该**不超过 40 行
- [ ] 嵌套深度不超过 3 层

### 参数与返回值检查

- [ ] 参数不超过 3 个（不含 `ctx`），超过用结构体
- [ ] 参数顺序：`ctx` → 主输入 → 选项
- [ ] 无裸布尔参数
- [ ] 无输出参数（out parameter）
- [ ] 返回值不超过 3 个，`error` 在最后

### 设计原则检查

- [ ] 命令函数和查询函数分离（CQS）
- [ ] 纯计算逻辑无副作用
- [ ] 使用 Guard Clause 提前返回
- [ ] 副作用集中在函数末尾
- [ ] Receiver 类型统一（值或指针不混用）

---

## 参考资料

- Clean Code - Robert C. Martin：函数设计章节
- Go Code Review Comments：https://go.dev/wiki/CodeReviewComments
- Effective Go：https://go.dev/doc/effective_go
- Uber Go Style Guide：https://github.com/uber-go/guide
