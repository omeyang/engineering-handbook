# 错误处理与健壮性

本文档定义 Go 项目的错误处理、边界条件、防御式编程、并发安全和优雅降级标准。

---

## 核心原则

1. **完善的错误处理** - **必须**处理所有错误，**禁止**忽略错误返回值
2. **边界条件覆盖** - **必须**检查 nil、空值、边界值
3. **防御式编程** - **必须**验证所有外部输入
4. **故障隔离** - **应该**实现超时控制、重试机制、熔断器
5. **优雅降级** - **应该**设计降级策略，非核心功能失败不影响核心流程
6. **并发安全** - **必须**保护共享状态，避免数据竞争
7. **安全编码** - **必须**防止注入攻击，保护敏感数据

---

## 错误处理

### 错误处理要求

```go
// 错误：忽略错误
result, _ := repo.FindByID(id)

// 正确：处理错误
result, err := repo.FindByID(id)
if err != nil {
    return fmt.Errorf("find by id %s failed: %w", id, err)
}
```

### 错误包装

**必须**使用 `%w` 包装错误，保留错误链：

```go
func Query(id string) (*Order, error) {
    order, err := repo.FindByID(id)
    if err != nil {
        return nil, fmt.Errorf("query by id %s failed: %w", id, err)
    }
    return order, nil
}
```

**必须**使用 `errors.Is` 和 `errors.As` 判断错误：

```go
if errors.Is(err, ErrNotFound) {
    return nil  // 未找到不是错误
}

var valErr *ValidationError
if errors.As(err, &valErr) {
    log.Error("validation failed", "field", valErr.Field)
}
```

### 自定义错误类型

```go
var (
    ErrNotFound      = errors.New("not found")
    ErrInvalidInput  = errors.New("invalid input")
    ErrDuplicate     = errors.New("duplicate record")
)

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: field=%s, message=%s", e.Field, e.Message)
}
```

---

## 边界条件处理

### **必须**检查的边界条件

**1. nil 检查**

```go
func Process(order *Order) error {
    if order == nil {
        return errors.New("order is nil")
    }
    // 处理逻辑
}
```

**2. 空字符串检查**

```go
func FindByID(id string) (*Order, error) {
    if id == "" {
        return nil, ErrInvalidInput
    }
    // 查询逻辑
}
```

**3. 空切片检查**

```go
func BatchCreate(orders []*Order) error {
    if len(orders) == 0 {
        return errors.New("empty order list")
    }
    // 批量创建逻辑
}
```

**4. 数值边界检查**

```go
func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

**5. 索引越界检查**

```go
func GetItem(items []*Item, index int) (*Item, error) {
    if index < 0 || index >= len(items) {
        return nil, errors.New("index out of bounds")
    }
    return items[index], nil
}
```

---

## 输入验证

### 验证层级

```
API 层（参数验证） -> Service 层（业务验证） -> Model 层（数据完整性验证）
```

### API 层参数验证

```go
func ValidateInput(input *CreateInput) error {
    var errs []string

    if input.Name == "" {
        errs = append(errs, "name is required")
    }

    if input.Email != "" && !isValidEmail(input.Email) {
        errs = append(errs, "invalid email address")
    }

    if len(errs) > 0 {
        return fmt.Errorf("validation failed: %s", strings.Join(errs, "; "))
    }
    return nil
}
```

---

## 并发安全

### **必须**保护共享状态

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]*Item
}

func (c *Cache) Get(id string) (*Item, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    item, ok := c.items[id]
    return item, ok
}

func (c *Cache) Set(id string, item *Item) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[id] = item
}
```

### **必须**避免数据竞争

使用 `-race` 标志检测竞争条件：

```bash
go test -race ./...
go build -race
```

常见数据竞争解决方案：

```go
// 使用互斥锁
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// 使用原子操作
type Counter struct {
    count int64
}

func (c *Counter) Increment() {
    atomic.AddInt64(&c.count, 1)
}
```

### **必须**安全使用 channel

```go
// 发送方关闭 channel
func Producer(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// 接收方检查 channel 状态
func Consumer(ch <-chan int) {
    for val := range ch {
        process(val)
    }
}
```

---

## 安全编码

### **必须**防止注入

**SQL 参数化查询**：

```go
// 错误：拼接 SQL
query := fmt.Sprintf("SELECT * FROM orders WHERE id='%s'", id)

// 正确：参数化查询
query := "SELECT * FROM orders WHERE id=?"
db.Query(query, id)
```

### **必须**保护敏感数据

```go
// 错误：记录密码
log.Info("user login", "username", username, "password", password)

// 正确：脱敏处理
log.Info("user login", "username", username, "password", "***")
```

---

## 故障隔离

### **应该**实现超时控制

```go
func QueryWithTimeout(ctx context.Context, id string) (*Order, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    return repo.FindByID(ctx, id)
}
```

### **应该**实现重试机制

```go
func QueryWithRetry(ctx context.Context, id string, maxRetries int) (*Order, error) {
    var lastErr error

    for i := 0; i < maxRetries; i++ {
        order, err := repo.FindByID(ctx, id)
        if err == nil {
            return order, nil
        }

        lastErr = err
        if !isRetryableError(err) {
            break
        }

        // 指数退避
        backoff := time.Duration(1<<uint(i)) * time.Second
        time.Sleep(backoff)
    }

    return nil, fmt.Errorf("query failed after %d retries: %w", maxRetries, lastErr)
}
```

---

## 优雅降级

### **应该**设计非核心功能降级策略

```go
func Query(ctx context.Context, id string) (*Order, error) {
    // 核心功能：查询数据
    order, err := repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 非核心功能：丰富信息（可以降级）
    if err := enrichInfo(ctx, order); err != nil {
        log.Warn(ctx, "enrich info failed, using default", "error", err)
        // 继续执行，不影响核心功能
    }

    return order, nil
}
```

### **应该**实现缓存降级

```go
func Get(ctx context.Context, id string) (*Order, error) {
    // 1. 优先从缓存获取
    if order, err := cache.Get(id); err == nil {
        return order, nil
    }

    // 2. 缓存失败，从数据库获取
    order, err := repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    // 3. 更新缓存（失败不影响结果）
    if err := cache.Set(id, order); err != nil {
        log.Warn(ctx, "update cache failed", "error", err)
    }

    return order, nil
}
```

---

## 检查清单

### 错误处理检查

- [ ] 所有错误都有处理（**禁止**用 `_` 忽略）
- [ ] 使用 `%w` 包装错误
- [ ] 错误携带足够上下文
- [ ] 使用 `errors.Is` 和 `errors.As` 判断错误

### 边界条件检查

- [ ] nil 检查
- [ ] 空字符串检查
- [ ] 空切片检查
- [ ] 数值边界检查
- [ ] 索引越界检查

### 并发安全检查

- [ ] 保护共享状态
- [ ] 使用 `-race` 检测数据竞争
- [ ] 安全使用 channel
- [ ] 使用 sync.Once 保证单次初始化

### 安全编码检查

- [ ] 防止 SQL/NoSQL 注入
- [ ] 防止命令注入
- [ ] 不记录敏感信息
- [ ] 使用加密存储密码

### 故障隔离检查

- [ ] 关键操作有超时控制
- [ ] 网络调用有重试机制
- [ ] 使用 context 传递取消信号

### 优雅降级检查

- [ ] 非核心功能失败不影响核心功能
- [ ] 缓存失败有降级策略

---

## 参考资料

- Go 错误处理最佳实践：https://go.dev/blog/error-handling-and-go
- Go 并发模式：https://go.dev/blog/pipelines
- Go 数据竞争检测器：https://go.dev/doc/articles/race_detector
- OWASP Go Secure Coding：https://owasp.org/www-project-go-secure-coding-practices-guide/
