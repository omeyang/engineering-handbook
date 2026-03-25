# 质量意识

本文档定义 Go 项目中代码质量的核心原则、质量维度、权衡策略和业务抽象方法。质量不仅是"能运行"，而是正确性、可维护性、健壮性等多维度的综合体现。

---

## 核心原则

1. **追求卓越** - **必须**设定高标准，持续改进代码质量
2. **多维度质量** - **必须**从正确性、可维护性、健壮性、性能、可观测性、可测试性六个维度综合评估
3. **业务抽象** - **必须**从业务角度抽象概念，避免被技术实现绑架
4. **质疑现状** - **应该**敢于推翻不合理的设计，而非"原来就这样"
5. **平衡权衡** - **必须**在质量各维度间寻找最优平衡点

---

## 质量维度

### 1. 正确性（Correctness）

**定义**：代码功能符合需求，边界条件处理正确。

**核心要求**：
- **必须**满足功能需求和验收标准
- **必须**正确处理边界条件（空值、零值、边界值）
- **必须**通过所有测试用例

**衡量指标**：
- 测试覆盖率 >= 90%（核心业务 >= 95%）
- 功能测试通过率 100%
- 边界条件测试覆盖率 100%

**反模式**：
```go
// 未处理边界条件
func Divide(a, b int) int {
    return a / b  // b=0 会 panic
}

// 处理边界条件
func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

---

### 2. 可维护性（Maintainability）

**定义**：代码易读、易改、易扩展。

**核心要求**：
- **必须**命名清晰（见名知义）
- **必须**函数职责单一（<= 70 行）
- **必须**圈复杂度 <= 10
- **应该**函数参数 <= 3 个
- **应该**嵌套层级 <= 3 层

**衡量指标**：
- 圈复杂度 <= 10（核心业务），<= 15（其他）
- 函数长度 <= 70 行
- 修改耗时：单个功能修改 <= 1 天

**反模式**：
```go
// 职责不清，难以维护
func Process(data interface{}) interface{} {
    // 200 行代码...
}

// 职责清晰，易于维护
func ValidateInput(input *Request) error { ... }
func TransformData(data *Data) *Result { ... }
```

---

### 3. 健壮性（Robustness）

**定义**：代码能优雅处理异常情况，不崩溃。

**核心要求**：
- **必须**处理所有错误（不能用 `_` 忽略）
- **必须**使用 `%w` 包装错误，保留错误链
- **必须**验证外部输入
- **应该**实现超时控制和重试机制
- **应该**实现熔断器保护

**衡量指标**：
- 错误处理覆盖率 100%
- 关键操作超时保护 100%
- 无未捕获的 panic

**反模式**：
```go
// 忽略错误
user, _ := repo.FindByID(id)

// 正确处理错误
user, err := repo.FindByID(id)
if err != nil {
    return fmt.Errorf("find user by id %s failed: %w", id, err)
}
```

---

### 4. 性能（Performance）

**定义**：响应快、资源占用合理。

**核心要求**：
- **必须**避免 N+1 查询问题
- **必须**避免不必要的内存分配
- **应该**热点数据使用缓存
- **应该**批量操作替代逐个操作
- **可以**使用 goroutine 池控制并发

**衡量指标**：
- P99 响应时间 <= 100ms
- 吞吐量 >= 1000 QPS
- 无明显内存泄漏

**反模式**：
```go
// N+1 查询
for _, order := range orders {
    details, _ := repo.FindDetails(order.ID)
}

// 批量查询
orderIDs := extractIDs(orders)
detailsMap, _ := repo.BatchFindDetails(orderIDs)
```

---

### 5. 可观测性（Observability）

**定义**：问题可追踪、可定位、可度量。

**核心要求**：
- **必须**关键路径有日志（入口、出口、错误）
- **必须**使用结构化日志
- **应该**暴露 Prometheus 监控指标
- **应该**支持分布式追踪（OpenTelemetry）
- **可以**实现健康检查接口

**衡量指标**：
- 关键路径日志覆盖率 100%
- 错误日志包含堆栈信息
- 监控指标暴露（/metrics）

**反模式**：
```go
// 无日志
func GetUser(id string) (*User, error) {
    return repo.FindByID(id)
}

// 有日志和监控
func GetUser(ctx context.Context, id string) (*User, error) {
    log.Info(ctx, "get user", "user_id", id)

    user, err := repo.FindByID(ctx, id)
    if err != nil {
        log.Error(ctx, "find user failed", "user_id", id, "error", err)
        return nil, fmt.Errorf("find user by id %s failed: %w", id, err)
    }

    log.Info(ctx, "get user success", "user_id", user.ID)
    return user, nil
}
```

---

### 6. 可测试性（Testability）

**定义**：代码易于编写和执行测试。

**核心要求**：
- **必须**依赖可注入（通过参数或接口）
- **必须**函数职责单一
- **必须**避免全局状态
- **应该**时间可控（注入 TimeProvider）
- **应该**优先使用纯函数

**衡量指标**：
- 单元测试可独立运行
- 测试执行时间 < 1s（单元测试）
- 无 flaky 测试

**反模式**：
```go
// 硬编码依赖
func GetUser(id string) (*User, error) {
    db := mongo.Connect("mongodb://...")
    return db.FindByID(id)
}

// 依赖注入
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

func GetUser(ctx context.Context, repo UserRepository, id string) (*User, error) {
    return repo.FindByID(ctx, id)
}
```

---

## 质量权衡

不同质量维度之间存在权衡，**应该**根据场景选择平衡点。

### 1. 抽象 vs 过度设计

**原则**：**必须**有 >= 2 种实现时才抽象，遵循 YAGNI（You Aren't Gonna Need It）。

```go
// 过度抽象（只有 1 种实现）
type OrderProcessor interface {
    Process(*Order) error
}

// 直接实现
func ProcessOrder(order *Order) error { ... }
```

### 2. 性能 vs 可读性

**原则**：**应该**优先保证可读性，**必须**先测量再优化热点代码。

```go
// 过度优化，牺牲可读性
func calc(a, b int) int {
    return (a << 1) + (b << 1) - (a & b)
}

// 清晰实现
func calculateTotal(price, quantity int) int {
    return (price + quantity) * 2
}
```

### 3. Mock vs 过度 Mock

**原则**：**必须** Mock 外部依赖，**禁止** Mock 核心业务逻辑。

```go
// 过度 Mock（Mock 业务逻辑）
type OrderService struct {
    validator OrderValidator
}

// 合理 Mock（Mock 外部依赖）
type OrderService struct {
    repo Repository
}
```

---

## 业务抽象

### 原则

1. **从业务角度抽象** - **必须**使用业务术语而非技术术语
2. **避免技术绑架** - 业务逻辑**禁止**依赖具体技术实现
3. **显式建模** - 关键业务概念**必须**有显式类型

### 业务概念显式化

```go
// 技术术语，不明确
func ProcessData(data map[string]interface{}) error { ... }

// 业务术语，清晰
type TransferRequest struct {
    FromAccount string
    ToAccount   string
    Amount      int64
}

func ExecuteTransfer(req *TransferRequest) (*TransferResult, error) { ... }
```

### 分层隔离

```go
// 业务层依赖存储细节
func FindOrder(filter bson.M) (*Order, error) {
    return db.Collection("orders").FindOne(ctx, filter)
}

// 业务层使用业务概念
type OrderFinder interface {
    FindByID(ctx context.Context, id string) (*Order, error)
}
```

---

## 质疑现状

### 原则

1. **敢于质疑** - **应该**质疑不合理设计，而非"原来就这样"
2. **推倒重来** - **可以**完全重写设计不良的代码
3. **有理有据** - **必须**提供改进理由和具体方案

### 反模式检测

以下情况**应该**警惕并重构：

| 现象 | 问题 | 改进方向 |
|-----|------|---------|
| "这个函数 300 行，改不动了" | 职责不单一 | 拆分函数，提取子功能 |
| "加一个 if-else 就行" | 违反开闭原则 | 使用策略模式 |
| "测试太难写了" | 可测试性差 | 依赖注入，提取接口 |
| "这段代码很复杂，但能用" | 可维护性差 | 简化逻辑，降低复杂度 |
| "原来就这样，不宜改动" | 缺乏改进意识 | 评估技术债务，制定重构计划 |

---

## 检查清单

### 代码质量自查

- [ ] 正确性：测试覆盖率达标，边界条件处理完善
- [ ] 可维护性：命名清晰，函数简短，复杂度低
- [ ] 健壮性：错误处理完善，输入验证充分
- [ ] 性能：无明显性能问题，热点有优化
- [ ] 可观测性：日志完善，监控到位
- [ ] 可测试性：依赖可注入，测试易编写

### 设计质量自查

- [ ] 业务概念抽象清晰，使用业务术语
- [ ] 分层合理，业务逻辑不依赖技术细节
- [ ] 抽象适度，无过度设计
- [ ] 可扩展，符合开闭原则
- [ ] 耦合度低，内聚度高

---

## 参考资料

**设计原则**：
- SOLID 原则：https://en.wikipedia.org/wiki/SOLID
- YAGNI 原则：https://martinfowler.com/bliki/Yagni.html
- DRY 原则：https://en.wikipedia.org/wiki/Don%27t_repeat_yourself

**Go 最佳实践**：
- Effective Go：https://go.dev/doc/effective_go
- Go Code Review Comments：https://go.dev/wiki/CodeReviewComments
- Uber Go Style Guide：https://github.com/uber-go/guide
