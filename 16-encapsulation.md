# 封装与信息隐藏

本文档定义 Go 项目中封装与信息隐藏的规范，包括导出决策、结构体设计、包 API 面积控制和构造函数设计。

---

## 核心原则

1. **最小暴露** - **必须**只导出外部确实需要的标识符
2. **隐藏实现细节** - 包的内部实现**禁止**对外可见
3. **通过行为暴露，而非数据** - **应该**通过方法暴露能力，而非直接暴露字段
4. **构造函数守护不变量** - 有约束的类型**必须**通过构造函数创建
5. **包是封装的基本单元** - Go 的封装边界是包，**必须**在包级别思考封装

---

## 导出决策

### 核心规则

**默认不导出，有明确需求时才导出。**

| 标识符类型 | 默认 | 导出条件 |
|-----------|------|---------|
| 结构体 | 不导出 | 其他包需要直接使用该类型 |
| 结构体字段 | 不导出 | 需要 JSON 序列化或其他包直接读写 |
| 方法 | 不导出 | 是包 API 的一部分 |
| 函数 | 不导出 | 是包 API 的一部分 |
| 常量 | 不导出 | 其他包需要引用 |
| 接口 | 导出 | 接口通常用于跨包依赖 |

### 导出前的三个问题

在导出任何标识符前，**必须**回答以下三个问题：

1. **谁需要它？** — 如果只有包内使用，不导出
2. **为什么不能通过方法间接提供？** — 优先通过行为暴露
3. **导出后能否自由修改其实现？** — 如果不能，说明封装不足

### 导出的稳定性承诺

导出的标识符是**公开 API**，修改它会影响所有调用方。

- 导出的函数签名**不应该**随意变更
- 导出的结构体字段**不应该**随意删除或改类型
- 如果不确定是否需要导出，**不导出**

---

## 结构体设计

### 字段可见性

**规则**：结构体字段**应该**默认不导出，通过方法访问。

```go
// 错误：所有字段导出，无法控制修改
type Connection struct {
    Host     string
    Port     int
    Pool     *pgxpool.Pool
    MaxRetry int
}

// 正确：字段不导出，通过构造函数和方法控制
type Connection struct {
    host     string
    port     int
    pool     *pgxpool.Pool
    maxRetry int
}

func (c *Connection) Host() string { return c.host }
func (c *Connection) IsHealthy() bool { return c.pool != nil }
```

### 允许导出字段的场景

以下场景**可以**导出字段：

1. **纯数据传输对象（DTO）**：只用于 JSON/BSON 序列化

```go
// DTO：允许导出字段（JSON 序列化需要）
type CreateOrderRequest struct {
    UserID string `json:"user_id"`
    Items  []Item `json:"items"`
}
```

2. **简单值对象**：无行为、无约束的数据容器

```go
// 简单值对象：允许导出字段
type Point struct {
    X float64
    Y float64
}
```

3. **配置结构体**：需要外部赋值的配置

```go
// 配置：允许导出字段
type ServerConfig struct {
    Port         int           `yaml:"port"`
    ReadTimeout  time.Duration `yaml:"read_timeout"`
    WriteTimeout time.Duration `yaml:"write_timeout"`
}
```

### 有约束的字段必须不导出

如果字段有取值约束、一致性要求或不变量（invariant），**必须**不导出：

```go
// 错误：外部可以设置非法状态
type Order struct {
    Status string  // 外部可以随意赋值 "anything"
    Total  float64 // 外部可以设置负数
}

// 正确：通过方法保护不变量
type Order struct {
    status OrderStatus
    total  float64
}

func (o *Order) Status() OrderStatus { return o.status }

func (o *Order) Cancel() error {
    if o.status != StatusPending {
        return fmt.Errorf("cannot cancel order in status %s", o.status)
    }
    o.status = StatusCancelled
    return nil
}
```

---

## 构造函数设计

### 规则

有约束的类型**必须**提供构造函数，**禁止**依赖外部直接构造。

```go
// 错误：外部可以构造不完整的对象
matcher := &Matcher{}  // 缺少 repo 和 cache，后续调用会 panic

// 正确：构造函数确保完整性
func NewMatcher(repo Repository, cache Cache) *Matcher {
    return &Matcher{
        repo:  repo,
        cache: cache,
    }
}
```

### 构造函数中的校验

构造函数**应该**校验参数的合法性：

```go
func NewServer(addr string, opts ...ServerOption) (*Server, error) {
    if addr == "" {
        return nil, errors.New("addr is required")
    }
    s := &Server{
        addr:    addr,
        port:    8080,
        timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    if s.port <= 0 || s.port > 65535 {
        return nil, fmt.Errorf("invalid port: %d", s.port)
    }
    return s, nil
}
```

### 零值可用

如果类型可以安全使用零值，**可以**不提供构造函数：

```go
// 零值可用：sync.Mutex 无需构造函数
var mu sync.Mutex

// 零值可用：bytes.Buffer 无需构造函数
var buf bytes.Buffer
```

**判断标准**：如果零值不会导致 panic 或错误行为，就是零值可用的。

---

## 包 API 面积控制

### 最小 API 面积

包导出的标识符越少越好。API 面积小意味着：
- 使用者学习成本低
- 维护者修改自由度高
- 不兼容变更的风险低

### 量化指标

| 指标 | 推荐范围 | 说明 |
|------|---------|------|
| 导出函数/方法数 | **应该**不超过 15 个 | 超过说明包可能职责过宽 |
| 导出类型数 | **应该**不超过 5 个 | 超过考虑拆包 |
| 导出接口数 | **应该**不超过 3 个 | 接口应小且精 |
| 导出常量/变量数 | 按需 | 尽量少 |

### API 面积检查方法

```bash
# 统计包的导出符号数量
go doc ./internal/matcher | grep -c "^func\|^type\|^var\|^const"

# 或使用 go vet 的 API 兼容性检查
go vet ./...
```

### 收窄 API 的手段

**手段 1：不导出辅助类型**

```go
// 错误：辅助类型也导出了
type MatchResult struct { ... }  // 只在包内用
type MatchConfig struct { ... }  // 只在包内用

// 正确：只导出必要的
type matchResult struct { ... }  // 不导出
type matchConfig struct { ... }  // 不导出
```

**手段 2：使用 internal 包**

```go
// 将不应该被外部使用的代码放入 internal
project/
├── matcher/
│   ├── matcher.go          // 公开 API
│   └── internal/
│       └── algorithm/      // 内部算法，外部不可导入
│           └── fuzzy.go
```

**手段 3：接口收窄**

返回接口而非具体类型，隐藏实现细节：

```go
// 错误：暴露了具体类型
func NewCache() *RedisCache { ... }

// 正确：返回接口
func NewCache() Cache { ... }

type Cache interface {
    Get(ctx context.Context, key string) ([]byte, error)
    Set(ctx context.Context, key string, val []byte, ttl time.Duration) error
}
```

---

## 封装层次

### 三个封装层次

```
┌───────────────────────────────────────────┐
│  包级封装（最外层）                        │
│  exported/unexported 控制包的 API 面积     │
│  ┌───────────────────────────────────────┐ │
│  │  类型级封装（中间层）                  │ │
│  │  字段可见性 + 方法控制状态变更          │ │
│  │  ┌───────────────────────────────────┐ │ │
│  │  │  函数级封装（最内层）              │ │ │
│  │  │  局部变量、闭包隐藏实现细节        │ │ │
│  │  └───────────────────────────────────┘ │ │
│  └───────────────────────────────────────┘ │
└───────────────────────────────────────────┘
```

### 各层次的职责

**包级封装**：决定哪些类型和函数可以被其他包使用
- **必须**最小化导出标识符
- **必须**通过 `internal/` 隐藏不应暴露的包

**类型级封装**：决定类型的哪些字段和方法对外可见
- 有约束的字段**必须**不导出
- 状态变更**必须**通过方法

**函数级封装**：函数内部的实现细节
- 局部变量**禁止**泄漏到函数外
- 复杂逻辑**应该**封装在子函数中

---

## 反模式

### 反模式 1：贫血暴露

结构体导出所有字段，没有行为方法，逻辑散落在外部。

```go
// 反模式：贫血对象
type Order struct {
    Status string
    Total  float64
    Items  []Item
}

// 逻辑散落在外部函数中
func CanCancelOrder(o *Order) bool { return o.Status == "pending" }
func CalculateTotal(o *Order) float64 { ... }

// 正确：行为内聚在类型中
type Order struct {
    status OrderStatus
    total  float64
    items  []item
}

func (o *Order) CanCancel() bool { return o.status == StatusPending }
func (o *Order) Total() float64  { return o.total }
```

### 反模式 2：接口泄漏实现

接口方法暴露了底层技术细节。

```go
// 反模式：暴露 MongoDB 细节
type Repository interface {
    FindByFilter(ctx context.Context, filter bson.M) ([]*Order, error)
    Aggregate(ctx context.Context, pipeline mongo.Pipeline) ([]*Result, error)
}

// 正确：技术无关的接口
type Repository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
    FindByStatus(ctx context.Context, status OrderStatus) ([]*Order, error)
}
```

### 反模式 3：全局可变状态

包级变量导出且可变，任何人随时可改。

```go
// 反模式：全局可变状态
var DefaultTimeout = 30 * time.Second  // 任何包都能改

// 正确：不可变或通过函数访问
const DefaultTimeout = 30 * time.Second  // 常量不可变

// 或通过函数控制
func SetTimeout(d time.Duration) {
    mu.Lock()
    defer mu.Unlock()
    timeout = d
}
```

---

## 检查清单

### 导出决策检查

- [ ] 所有导出的标识符都有明确的外部使用者
- [ ] 没有"以防万一"的导出
- [ ] 导出前回答了三个问题（谁需要、为什么不能通过方法、能否自由修改实现）

### 结构体设计检查

- [ ] 有约束的字段不导出
- [ ] DTO/配置结构体可以导出字段
- [ ] 有约束的类型提供了构造函数
- [ ] 构造函数校验了参数合法性

### 包 API 面积检查

- [ ] 导出函数/方法不超过 15 个
- [ ] 导出类型不超过 5 个
- [ ] 使用 `internal/` 隐藏实现包
- [ ] 返回接口而非具体类型（当有多种实现时）

### 封装层次检查

- [ ] 没有贫血暴露（行为与数据在一起）
- [ ] 接口不泄漏实现细节
- [ ] 没有导出的全局可变状态
- [ ] 包级变量优先使用常量

---

## 参考资料

- Effective Go — Exported Identifiers：https://go.dev/doc/effective_go#names
- Go Code Review Comments — Package Names：https://go.dev/wiki/CodeReviewComments
- 信息隐藏原则（Parnas, 1972）：On the Criteria To Be Used in Decomposing Systems into Modules
- Uber Go Style Guide：https://github.com/uber-go/guide
