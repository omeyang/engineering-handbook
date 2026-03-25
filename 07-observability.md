# 可观测性

本文档定义 Go 项目的可测试性、日志、监控、追踪和可读性标准。

---

## 核心原则

**可观测性三支柱**：
1. **可测试性（Testability）** - 代码**必须**易于编写和执行测试
2. **可观测性（Observability）** - 日志、监控、追踪**必须**完善
3. **可读性（Readability）** - 代码**必须**清晰易懂

---

## 可测试性（Testability）

### 核心规则

- 依赖**必须**可注入
- 函数职责**必须**单一
- **禁止**使用全局状态
- 时间**应该**可控（注入 TimeProvider）
- **应该**优先使用纯函数

**1. 依赖可注入**

```go
// 错误：硬编码依赖
func Query(id string) (*Order, error) {
    db := mongo.Connect("mongodb://...")
    return db.FindByID(id)
}

// 正确：依赖注入
func Query(ctx context.Context, repo Repository, id string) (*Order, error) {
    return repo.FindByID(ctx, id)
}
```

**2. 无全局状态**

```go
// 错误：依赖全局状态
var GlobalCache map[string]*Item

func GetItem(id string) *Item {
    return GlobalCache[id]
}

// 正确：通过参数传递
func GetItem(cache Cache, id string) (*Item, error) {
    return cache.Get(id)
}
```

**3. 时间可控**

```go
// 错误：使用 time.Now()
func IsExpired(item *Item) bool {
    return time.Now().After(item.ExpireAt)
}

// 正确：注入时间源
type TimeProvider interface {
    Now() time.Time
}

func IsExpired(tp TimeProvider, item *Item) bool {
    return tp.Now().After(item.ExpireAt)
}
```

---

## 可观测性（Observability）

### 日志（Logging）

#### 日志级别

**必须**根据场景选择合适的日志级别：

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| **Trace** | 最详细的调试信息 | 函数入口/出口、变量值 |
| **Debug** | 调试信息 | 中间结果、条件判断 |
| **Info** | 关键操作 | 记录创建、更新、删除 |
| **Warn** | 警告信息 | 缓存失败、非核心功能降级 |
| **Error** | 错误信息 | 数据库错误、业务异常 |
| **Fatal** | 致命错误 | 启动失败、配置错误 |

#### 结构化日志

**必须**使用结构化日志：

```go
// 正确：结构化日志
log.Info(ctx, "querying item",
    "id", id,
    "user_id", userID,
    "request_id", requestID,
)

log.Error(ctx, "query failed",
    "id", id,
    "error", err,
    "duration_ms", duration.Milliseconds(),
)

// 错误：字符串拼接
log.Info(fmt.Sprintf("querying item: id=%s, user=%s", id, userID))
```

#### 日志实现与抽象

业务代码**必须**只依赖项目内定义的 `Logger` 接口，日志实现**必须**基于标准库 `log/slog` 或其封装。

```go
type Logger interface {
    Info(ctx context.Context, msg string, args ...any)
    Error(ctx context.Context, msg string, args ...any)
}
```

#### 关键路径日志

关键路径**必须**有日志（入口、出口、错误）：

```go
func Query(ctx context.Context, id string) (*Order, error) {
    log.Info(ctx, "query started", "id", id)

    if err := ValidateID(id); err != nil {
        log.Error(ctx, "invalid id", "id", id, "error", err)
        return nil, err
    }

    order, err := repo.FindByID(ctx, id)
    if err != nil {
        log.Error(ctx, "find failed", "id", id, "error", err)
        return nil, fmt.Errorf("find by id %s failed: %w", id, err)
    }

    log.Info(ctx, "query success", "id", id, "order_id", order.ID)
    return order, nil
}
```

### 监控（Metrics）

**应该**暴露监控指标（/metrics）：

**1. 请求指标**
- 请求总数（Counter）
- 请求延迟（Histogram）
- 错误率（Gauge）
- 并发数（Gauge）

**2. 业务指标**
- 业务对象总数（Gauge）
- 每秒查询数（Counter）
- 缓存命中率（Gauge）

**3. 资源指标**
- CPU 使用率（Gauge）
- 内存使用（Gauge）
- Goroutine 数量（Gauge）
- GC 耗时（Histogram）

#### 监控代码示例

```go
var (
    queryTotal     = metrics.NewCounter("query_total{status=\"success\"}")
    queryErrTotal  = metrics.NewCounter("query_total{status=\"error\"}")
    queryDuration  = metrics.NewHistogram("query_duration_seconds")
)

func Query(ctx context.Context, id string) (*Order, error) {
    start := time.Now()
    defer func() {
        queryDuration.Update(time.Since(start).Seconds())
    }()

    order, err := repo.FindByID(ctx, id)
    if err != nil {
        queryErrTotal.Inc()
        return nil, err
    }

    queryTotal.Inc()
    return order, nil
}
```

### 追踪（Tracing）

**应该**支持分布式追踪：

```go
import (
    "go.opentelemetry.io/otel"
)

func Query(ctx context.Context, id string) (*Order, error) {
    ctx, span := otel.Tracer("order-service").Start(ctx, "Query")
    defer span.End()

    span.SetAttributes(attribute.String("id", id))

    order, err := repo.FindByID(ctx, id)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    return order, nil
}
```

### 观测抽象接口

业务代码**必须**只依赖项目内定义的观测接口（如 `MetricsRecorder`、`Tracer`），具体实现可以替换。

---

## 可读性（Readability）

### 命名清晰

```go
// 错误
func proc(d interface{}) interface{} { ... }
var x int = 10

// 正确
func ProcessOrder(order *Order) (*Result, error) { ... }
var maxRetryCount int = 10
```

### 注释得当

**应该**有必要的注释，解释"为什么"而非"是什么"：

```go
// 好的注释：解释"为什么"
// 使用指数退避重试，避免雪崩效应
backoff := time.Duration(1<<uint(i)) * time.Second

// 坏的注释：重复代码
// 设置 i 为 0
i := 0
```

### 函数简短

函数**应该**不超过 70 行。

### 避免深层嵌套

嵌套**应该**不超过 3 层，使用提前返回：

```go
// 错误：深层嵌套
func Process(order *Order) error {
    if order != nil {
        if order.ID != "" {
            if isValid(order.ID) {
                // 业务逻辑
            }
        }
    }
}

// 正确：提前返回
func Process(order *Order) error {
    if order == nil {
        return errors.New("order is nil")
    }
    if order.ID == "" {
        return errors.New("ID is empty")
    }
    if !isValid(order.ID) {
        return errors.New("invalid ID")
    }
    // 业务逻辑
    return nil
}
```

### 一致的代码风格

```go
// 一致的错误处理
if err != nil {
    return nil, fmt.Errorf("operation failed: %w", err)
}

// 一致的日志格式
log.Info(ctx, "operation completed", "key", value)

// 一致的命名风格
func FindByID(id string) (*Order, error)
func FindByName(name string) (*Order, error)
```

---

## 检查清单

### 可测试性检查

- [ ] 依赖**必须**可注入
- [ ] 函数职责**必须**单一
- [ ] **禁止**使用全局状态
- [ ] 时间**应该**可控（注入 TimeProvider）
- [ ] **应该**优先使用纯函数

### 可观测性检查

- [ ] 关键路径**必须**有日志（入口、出口、错误）
- [ ] **必须**使用结构化日志
- [ ] **应该**暴露监控指标（/metrics）
- [ ] **应该**支持分布式追踪
- [ ] 错误**应该**有堆栈信息

### 可读性检查

- [ ] 命名**必须**见名知义
- [ ] 函数**应该**不超过 70 行
- [ ] 参数**应该**不超过 3 个
- [ ] 嵌套**应该**不超过 3 层
- [ ] **应该**有必要的注释
- [ ] 代码风格**必须**一致

---

## 参考资料

- OpenTelemetry Go：https://opentelemetry.io/docs/instrumentation/go/
- 结构化日志（log/slog）：https://go.dev/blog/slog
- Effective Go：https://go.dev/doc/effective_go
- Uber Go Style Guide：https://github.com/uber-go/guide
