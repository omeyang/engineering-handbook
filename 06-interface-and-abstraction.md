# 接口设计

本文档定义 Go 项目中接口设计、抽象时机、泛型使用和抽象层次的规范。

---

## 核心原则

1. **面向接口编程** - 依赖抽象而非具体实现
2. **小接口原则** - 接口**应该**小而专注（ISP）
3. **合理抽象** - **必须**有 2 种及以上实现时才抽象
4. **善用泛型** - Go 1.18+ 泛型**应该**简化通用代码
5. **清晰的抽象层次** - **禁止**抽象泄漏

---

## 接口设计原则

### 小接口原则（Interface Segregation Principle）

客户端**禁止**依赖它不需要的方法。

**错误：大接口**

```go
type Repository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
    Create(ctx context.Context, order *Order) error
    Update(ctx context.Context, order *Order) error
    Delete(ctx context.Context, id string) error
    Count(ctx context.Context) (int64, error)
}
```

**正确：拆分为小接口**

```go
type Finder interface {
    FindByID(ctx context.Context, id string) (*Order, error)
}

type Creator interface {
    Create(ctx context.Context, order *Order) error
}

type Updater interface {
    Update(ctx context.Context, order *Order) error
}

// 组合接口（按需组合）
type ReadRepository interface {
    Finder
}

type WriteRepository interface {
    Creator
    Updater
}

type Repository interface {
    ReadRepository
    WriteRepository
}
```

### 单方法接口

Go 惯例：**应该**优先使用单方法接口。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}
```

### 避免包名重复

接口名**禁止**包含包名：

```go
// 错误
package order
type OrderRepository interface { ... }   // order.OrderRepository

// 正确
package order
type Repository interface { ... }  // order.Repository
```

---

## 面向接口编程

### 依赖抽象而非实现

**必须**依赖接口而非具体实现（DIP）：

```go
// 错误：依赖具体实现
type OrderProcessor struct {
    mongoRepo *MongoRepository  // 依赖具体实现
}

// 正确：依赖接口
type OrderProcessor struct {
    repo Repository  // 依赖抽象
}

func NewOrderProcessor(repo Repository) *OrderProcessor {
    return &OrderProcessor{repo: repo}
}
```

### 依赖注入

**应该**使用构造函数注入：

```go
type Matcher struct {
    repo   Repository
    cache  Cache
    logger Logger
}

func NewMatcher(repo Repository, cache Cache, logger Logger) *Matcher {
    return &Matcher{
        repo:   repo,
        cache:  cache,
        logger: logger,
    }
}
```

---

## 抽象的时机

### 何时抽象？

**核心规则**：**必须**有 2 种及以上实现时才抽象。

**应该抽象的场景**：

1. 已有 2 种及以上实现
2. 明确需要支持多种实现
3. 外部依赖（便于测试）

**禁止抽象的场景**：

1. 只有 1 种实现，且未来不会变
2. 简单的数据结构

```go
// 错误：不必要的抽象
type IDGenerator interface {
    Generate() string
}

// 正确：直接使用函数
func GenerateID() string {
    return uuid.New().String()
}
```

### YAGNI 原则

**禁止**实现还不需要的功能。

```go
// 错误：过度抽象
type AbstractFactory interface {
    CreateItem() ItemInterface
}

// 正确：简单直接
type Item struct {
    ID   string
    Name string
}

func NewItem(id, name string) *Item {
    return &Item{ID: id, Name: name}
}
```

---

## Go 泛型（Go 1.18+）

### 何时使用泛型？

- **应该**用于通用容器、算法、数据结构
- **禁止**用于业务逻辑
- 类型参数**应该**不超过 2 个

### **应该**使用泛型的场景

**通用算法**：

```go
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}
```

**通用数据结构**：

```go
type Cache[K comparable, V any] struct {
    data sync.Map
}

func (c *Cache[K, V]) Set(key K, value V) {
    c.data.Store(key, value)
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    val, ok := c.data.Load(key)
    if !ok {
        var zero V
        return zero, false
    }
    return val.(V), true
}
```

### **禁止**使用泛型的场景

```go
// 错误：用泛型写业务逻辑
func ProcessItem[T ItemInterface](item T) error {
    // 业务逻辑必须具体，禁止泛化
}

// 正确：业务逻辑必须具体
func ProcessItem(item *Item) error {
    // 具体的业务处理
}
```

---

## 抽象层次

### 抽象层次一致性

同一个函数/模块**必须**保持相同的抽象层次。

```go
// 错误：抽象层次混乱
func ProcessOrder(order *Order) error {
    if err := ValidateOrder(order); err != nil {
        return err
    }
    // 低层实现细节混在一起
    db := mongo.Connect("mongodb://...")
    collection := db.Database("orders").Collection("orders")
    _, err := collection.InsertOne(context.Background(), order)
    return err
}

// 正确：抽象层次一致
func ProcessOrder(order *Order) error {
    if err := ValidateOrder(order); err != nil {
        return err
    }
    if err := SaveOrder(order); err != nil {
        return err
    }
    return NotifyOrderCreated(order)
}
```

### 避免抽象泄漏

接口**禁止**暴露实现细节：

```go
// 错误：暴露了 MongoDB 的 bson.M
type Repository interface {
    FindByFilter(ctx context.Context, filter bson.M) (*Order, error)
}

// 正确：使用通用查询条件
type Repository interface {
    FindByID(ctx context.Context, id string) (*Order, error)
}
```

---

## 可扩展性与耦合

### 开闭原则（OCP）

新增功能**应该**不修改现有代码：

```go
// 使用策略模式替代 if-else
type Matcher interface {
    Match(ctx context.Context, criteria *Criteria) (*Result, error)
}

type MatcherFactory struct {
    matchers map[string]Matcher
}

func (f *MatcherFactory) Register(typeName string, matcher Matcher) {
    f.matchers[typeName] = matcher
}

// 新增类型只需注册，不修改现有代码
factory.Register("new-type", &NewTypeMatcher{})
```

### 低耦合

| 耦合类型 | 说明 | 推荐度 |
|---------|------|--------|
| 内容耦合 | 直接访问另一个模块的内部数据 | **禁止** |
| 公共耦合 | 通过全局变量通信 | **禁止** |
| 控制耦合 | 传递控制标志 | **不应该** |
| 数据耦合 | 通过参数传递数据 | **应该** |
| 消息耦合 | 通过消息/事件通信 | **应该** |

---

## 检查清单

### 接口设计检查

- [ ] 接口小而专注（单一职责）
- [ ] **应该**优先使用单方法接口
- [ ] 接口名**禁止**包含包名
- [ ] 接口**应该**按职责拆分
- [ ] 接口**应该**可组合

### 抽象时机检查

- [ ] **必须**有 2 种及以上实现时才抽象
- [ ] **禁止**为了抽象而抽象
- [ ] **必须**遵循 YAGNI 原则

### 泛型使用检查

- [ ] **应该**用泛型实现通用容器/算法
- [ ] **禁止**用泛型写业务逻辑
- [ ] 类型参数**应该**不超过 2 个

### 抽象层次检查

- [ ] 同一函数抽象层次**必须**一致
- [ ] **禁止**抽象泄漏
- [ ] 接口**禁止**暴露实现细节

---

## 参考资料

- SOLID 原则：https://en.wikipedia.org/wiki/SOLID
- YAGNI 原则：https://martinfowler.com/bliki/Yagni.html
- Go 泛型：https://go.dev/doc/tutorial/generics
- Uber Go Style Guide：https://github.com/uber-go/guide
