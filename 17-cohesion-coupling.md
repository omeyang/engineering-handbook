# 内聚耦合度量

本文档定义 Go 项目中内聚性和耦合度的判定标准、度量方法、解耦策略和边界设计规范。

---

## 核心原则

1. **高内聚** - 包内元素**必须**围绕同一业务概念紧密协作
2. **低耦合** - 包之间的依赖**必须**最小化、接口化、单向化
3. **边界清晰** - 每个包**必须**有明确的职责边界和稳定的公开 API
4. **依赖可控** - 依赖关系**必须**是显式的、可追踪的、无环的
5. **变更隔离** - 修改一个包的实现**不应该**导致其他包的代码变更

---

## 内聚性

### 什么是高内聚

高内聚意味着包内的类型、函数和方法围绕同一个业务概念或职责紧密合作。包内元素之间频繁交互，对外则通过少量稳定的 API 通信。

### 内聚等级

从高到低排列，越高越好：

| 等级 | 名称 | 说明 | Go 中的表现 |
|------|------|------|------------|
| 1 | **功能内聚** | 所有元素共同完成一个功能 | `matcher` 包只做匹配判定 |
| 2 | **顺序内聚** | 输出作为下一步输入 | 验证 → 转换 → 保存的流水线 |
| 3 | **通信内聚** | 操作同一组数据 | `order` 包操作订单数据 |
| 4 | **时间内聚** | 在同一时间执行 | `init` 包的初始化逻辑 |
| 5 | **逻辑内聚** | 逻辑上相关但功能不同 | `utils` 包（**禁止**） |
| 6 | **偶然内聚** | 没有关系，碰巧放一起 | `common` 包（**禁止**） |

- 包**必须**达到通信内聚（等级 3）及以上
- **应该**追求功能内聚（等级 1）
- **禁止**出现逻辑内聚和偶然内聚（等级 5、6）

### 内聚度判定方法

**方法 1：删除法**

从包中移除一个类型或函数，问："包的核心功能是否受损？"

- 受损 → 该元素与包内聚
- 不受损 → 该元素可能不属于这个包

```go
// matcher 包
type Matcher struct { ... }       // 移除 → 核心受损 ✓ 属于此包
func Match() bool { ... }        // 移除 → 核心受损 ✓ 属于此包
func FormatDate() string { ... } // 移除 → 核心不受损 ✗ 不属于此包
```

**方法 2：命名法**

包内所有导出标识符是否都能用包名作为自然前缀？

```go
// matcher 包
// matcher.New        ✓ 自然
// matcher.Match      ✓ 自然
// matcher.Rule       ✓ 自然
// matcher.SendEmail  ✗ 不自然 → 不属于此包
```

**方法 3：修改影响法**

修改包内一个类型的定义，需要同时修改包内其他多少元素？

- 需要修改的元素多 → 内聚度高（元素之间紧密协作）
- 几乎不需要修改 → 内聚度低（元素之间独立，可能是偶然放在一起）

### 包拆分信号

出现以下信号时**应该**拆分包：

| 信号 | 说明 |
|------|------|
| 包内有不相关的类型群 | 类型 A 群和类型 B 群之间无交互 |
| 导出标识符超过 15 个 | API 面积过大 |
| 包名需要 "and"来描述 | 如"订单和支付"→ 拆为两个包 |
| 文件超过 5 个且分属不同关注点 | 每组文件应该是独立的包 |
| 修改一组功能不影响另一组 | 说明是两个独立关注点 |

### 包合并信号

出现以下信号时**应该**合并包：

| 信号 | 说明 |
|------|------|
| 两个包总是一起修改 | 耦合过紧，不如合并 |
| 包 A 的大部分导出只有包 B 使用 | A 可能是 B 的内部实现 |
| 包内只有 1-2 个类型 | 太碎，缺乏存在价值 |
| 两个包相互依赖 | 循环依赖，必须合并或重构 |

---

## 耦合度

### 什么是低耦合

低耦合意味着修改一个包的内部实现不需要修改其他包的代码。包之间通过窄接口通信，知道的细节越少越好。

### 耦合类型

从低到高排列，越低越好：

| 等级 | 名称 | 说明 | 推荐度 |
|------|------|------|--------|
| 1 | **消息耦合** | 通过消息/事件异步通信 | **推荐** |
| 2 | **数据耦合** | 只通过参数传递简单数据 | **推荐** |
| 3 | **标记耦合** | 通过结构体传递，但只用部分字段 | **应该避免** |
| 4 | **控制耦合** | 传递控制标志影响行为 | **不应该** |
| 5 | **公共耦合** | 通过全局变量通信 | **禁止** |
| 6 | **内容耦合** | 直接访问另一个模块内部数据 | **禁止** |

### 耦合度度量：Fan-in / Fan-out

**Fan-in（扇入）**：有多少包依赖当前包

- Fan-in 高 → 当前包是核心基础包，修改影响大，**必须**保持稳定
- 基础包（如 `model`、`pkg/`）的 Fan-in 高是正常的

**Fan-out（扇出）**：当前包依赖了多少其他包

- Fan-out 高 → 当前包依赖过多，职责可能过宽或抽象不够
- 单个包的 Fan-out **应该**不超过 5

```bash
# 查看包的依赖关系
go list -f '{{.ImportPath}}: {{join .Imports "\n  "}}' ./internal/...
```

| 指标 | 推荐 | 说明 |
|------|------|------|
| 单包 Fan-out | ≤ 5 | 超过说明依赖过多 |
| 单包直接依赖 | ≤ 7 | import 列表中的项目内包 |

### 耦合检测方法

**方法 1：修改影响分析**

修改包 A 的一个内部实现，检查是否需要修改包 B 的代码：

- 需要 → 包 A 和 B 耦合过紧
- 不需要 → 耦合度可接受

**方法 2：独立编译测试**

包 A 的单元测试是否能独立运行，不依赖包 B 的具体实现？

```go
// 耦合过紧：测试 OrderService 必须启动真实的 MongoDB
func TestCreateOrder(t *testing.T) {
    mongoClient := mongo.Connect(...)  // 依赖具体实现
    svc := NewOrderService(mongoClient)
    // ...
}

// 低耦合：通过接口 Mock
func TestCreateOrder(t *testing.T) {
    mockRepo := &mockRepository{...}  // Mock 接口
    svc := NewOrderService(mockRepo)
    // ...
}
```

**方法 3：import 分析**

```bash
# 检查循环依赖
go vet ./...

# 可视化依赖图
go list -f '{{.ImportPath}} -> {{range .Imports}}{{.}} {{end}}' ./internal/...
```

---

## 解耦策略

### 策略 1：接口隔离

依赖接口而非具体实现，是最基本的解耦手段。

```go
// 耦合：直接依赖 MongoRepository
type OrderService struct {
    repo *MongoRepository  // 绑定具体实现
}

// 解耦：依赖接口
type OrderService struct {
    repo Repository  // 接口
}

type Repository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}
```

### 策略 2：依赖注入

通过构造函数注入依赖，而非在内部创建。

```go
// 耦合：内部创建依赖
func NewOrderService() *OrderService {
    repo := NewMongoRepository(...)  // 内部创建，无法替换
    return &OrderService{repo: repo}
}

// 解耦：构造函数注入
func NewOrderService(repo Repository) *OrderService {
    return &OrderService{repo: repo}
}
```

### 策略 3：事件驱动

用事件/消息替代直接调用，实现异步解耦。

```go
// 耦合：直接调用
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }
    s.notifier.SendEmail(ctx, order.UserID)     // 直接调用通知
    s.inventory.Reserve(ctx, order.Items)        // 直接调用库存
    s.analytics.TrackEvent(ctx, "order_created") // 直接调用分析
    return nil
}

// 解耦：事件驱动
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    if err := s.repo.Save(ctx, order); err != nil {
        return err
    }
    return s.events.Publish(ctx, OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  order.UserID,
        Items:   order.Items,
    })
}
// 通知、库存、分析各自订阅事件，独立处理
```

### 策略 4：防腐层（Anti-Corruption Layer）

在当前系统与外部系统之间建立转换层，防止外部模型污染内部模型。

```go
// 错误：外部 API 的数据结构直接渗透到业务逻辑
func (s *OrderService) SyncFromLegacy(ctx context.Context) error {
    // legacy API 返回的奇怪字段名直接使用
    resp := legacyAPI.GetOrders()
    for _, item := range resp.Lst {  // 外部命名泄漏
        s.repo.Save(ctx, &Order{
            ID:    item.OrdNo,       // 外部命名泄漏
            Total: item.TtlAmt,     // 外部命名泄漏
        })
    }
}

// 正确：防腐层隔离外部模型
// internal/adapter/legacy.go
type LegacyAdapter struct {
    client *legacyAPI.Client
}

func (a *LegacyAdapter) FetchOrders(ctx context.Context) ([]*Order, error) {
    resp, err := a.client.GetOrders(ctx)
    if err != nil {
        return nil, fmt.Errorf("fetch legacy orders: %w", err)
    }
    return convertLegacyOrders(resp.Lst), nil
}

func convertLegacyOrders(items []legacyAPI.Item) []*Order {
    orders := make([]*Order, 0, len(items))
    for _, item := range items {
        orders = append(orders, &Order{
            ID:    item.OrdNo,
            Total: item.TtlAmt,
        })
    }
    return orders
}
```

### 策略 5：中间包提取

当两个包相互依赖时，将共享的类型提取到第三个包。

```go
// 错误：循环依赖
// order → payment（需要支付信息）
// payment → order（需要订单信息）

// 正确：提取共享类型
// model/order.go  — 订单数据模型
// order → model
// payment → model
// order 和 payment 不再相互依赖
```

---

## 边界设计

### 三层边界

```
┌─────────────────────────────────────────────┐
│  系统边界（最外层）                          │
│  HTTP API / gRPC / 消息队列                  │
│  职责：协议转换、认证、限流、输入验证        │
│  ┌─────────────────────────────────────────┐ │
│  │  模块边界（中间层）                      │ │
│  │  包的公开 API（导出函数/接口）           │ │
│  │  职责：封装业务逻辑、隐藏实现细节        │ │
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │  函数边界（最内层）                  │ │ │
│  │  │  函数签名（参数和返回值）            │ │ │
│  │  │  职责：明确输入输出契约              │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 系统边界

系统边界是系统与外部世界的交互点。

**规则**：

- 所有外部输入**必须**在系统边界校验
- 系统边界**必须**做协议转换（HTTP Request → 内部模型）
- 系统边界**禁止**包含业务逻辑

```go
// API 层：系统边界
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
    // 1. 协议转换：HTTP → 内部模型
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    // 2. 输入验证（边界职责）
    if err := req.Validate(); err != nil {
        writeError(w, http.StatusBadRequest, err.Error())
        return
    }

    // 3. 调用业务逻辑（不在此处实现业务）
    order, err := h.service.CreateOrder(r.Context(), &req)
    if err != nil {
        writeError(w, http.StatusInternalServerError, err.Error())
        return
    }

    // 4. 协议转换：内部模型 → HTTP Response
    writeJSON(w, http.StatusCreated, order)
}
```

### 模块边界

模块边界是包与包之间的交互点。

**规则**：

- 包的公开 API **必须**是稳定的接口或函数
- 包之间通信**必须**使用导出类型，**禁止**使用具体实现类型
- 数据跨包传递时，**应该**在边界处做转换

### 边界处的数据转换

不同层次**应该**使用不同的数据模型，在边界处转换：

```go
// API 层的 DTO
type CreateOrderRequest struct {
    UserID string `json:"user_id"`
    Items  []Item `json:"items"`
}

// Service 层的领域模型
type Order struct {
    id     string
    userID string
    items  []orderItem
    status OrderStatus
}

// 边界处的转换函数
func toOrder(req *CreateOrderRequest) *Order {
    return &Order{
        id:     GenerateID(),
        userID: req.UserID,
        items:  toOrderItems(req.Items),
        status: StatusPending,
    }
}
```

### 分层验证策略

不同边界层次**应该**验证不同的内容：

| 边界 | 验证内容 | 示例 |
|------|---------|------|
| **系统边界（API层）** | 格式、类型、必填字段 | JSON 格式、字段非空、类型正确 |
| **模块边界（Service层）** | 业务规则、权限、状态 | 用户是否存在、余额是否足够、状态是否允许 |
| **数据边界（Model层）** | 数据完整性、约束 | 唯一性、外键、字段范围 |

```go
// API 层：格式验证
func (r *CreateOrderRequest) Validate() error {
    if r.UserID == "" {
        return errors.New("user_id is required")
    }
    if len(r.Items) == 0 {
        return errors.New("items cannot be empty")
    }
    return nil
}

// Service 层：业务规则验证
func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    user, err := s.userRepo.FindByID(ctx, req.UserID)
    if err != nil {
        return nil, fmt.Errorf("find user: %w", err)
    }
    if !user.IsActive() {
        return nil, ErrUserInactive
    }
    // ...
}
```

---

## 依赖方向

### 依赖规则

```
API Layer → Service Layer → Model Layer → Infrastructure Layer
                ↓
          pkg/（公共库）
```

- 上层**必须**只依赖下层
- 下层**禁止**依赖上层
- 同层之间**必须**通过接口依赖
- **禁止**循环依赖

### 依赖倒置

高层模块不依赖低层模块，两者都依赖抽象。

```go
// 错误：Service 直接依赖 Mongo 实现
import "project/internal/infra/mongo"

type OrderService struct {
    repo *mongo.OrderRepository
}

// 正确：Service 依赖接口，接口定义在 Service 层
type Repository interface {
    Save(ctx context.Context, order *Order) error
}

type OrderService struct {
    repo Repository  // 依赖抽象
}

// Mongo 实现在 infra 层，实现 Service 层定义的接口
// 组装在 main 或 wire 中完成
```

### 接口归属

**规则**：接口**应该**定义在使用方，而非实现方。

```go
// 错误：接口定义在实现方
// internal/infra/mongo/repository.go
type Repository interface { ... }
type MongoRepository struct { ... }

// 正确：接口定义在使用方
// internal/order/service.go
type Repository interface { ... }

// internal/infra/mongo/repository.go
type MongoRepository struct { ... }
// MongoRepository 隐式实现 order.Repository
```

---

## 稳定性分析

### 稳定性原则

被大量依赖的包（Fan-in 高）**必须**比依赖它的包更稳定。

**稳定包的特征**：
- 以接口和数据类型为主
- 很少修改
- 向后兼容

**不稳定包的特征**：
- 以业务实现为主
- 经常修改
- 可以自由变更

### 稳定依赖原则

```
不稳定（经常变化） → 稳定（很少变化）
```

依赖方向**必须**指向更稳定的方向。

```
具体业务逻辑（不稳定）
    ↓ 依赖
业务接口定义（稳定）
    ↓ 依赖
基础数据模型（非常稳定）
```

**禁止**让稳定的包依赖不稳定的包。

---

## 检查清单

### 内聚性检查

- [ ] 包达到通信内聚（等级 3）及以上
- [ ] 没有 `utils`、`common`、`helpers` 包
- [ ] 包内所有元素都能通过"删除法"验证归属
- [ ] 包名能自然作为导出标识符的前缀
- [ ] 识别并处理了拆分/合并信号

### 耦合度检查

- [ ] 单包 Fan-out 不超过 5
- [ ] 包之间通过接口通信
- [ ] 无循环依赖（`go vet` 通过）
- [ ] 无全局可变状态共享
- [ ] 无控制耦合（不传递控制标志）

### 边界设计检查

- [ ] 系统边界做了输入验证和协议转换
- [ ] 系统边界不包含业务逻辑
- [ ] 不同层使用不同数据模型（必要时）
- [ ] 分层验证（API 格式 → Service 业务 → Model 数据）
- [ ] 外部系统通过防腐层隔离

### 依赖方向检查

- [ ] 依赖方向从上层到下层
- [ ] 接口定义在使用方
- [ ] 稳定的包不依赖不稳定的包
- [ ] 依赖注入通过构造函数

---

## 参考资料

- 结构化设计（Larry Constantine）：内聚耦合理论来源
- Clean Architecture - Robert C. Martin：依赖倒置与稳定性分析
- Go Code Review Comments：https://go.dev/wiki/CodeReviewComments
- Uber Go Style Guide：https://github.com/uber-go/guide
- 反腐层模式：https://learn.microsoft.com/zh-cn/azure/architecture/patterns/anti-corruption-layer
