# 测试标准

本文档定义 Go 项目的测试规范，包括 TDD 流程、覆盖率要求、Mock 策略和测试组织。

---

## 核心原则

1. **测试驱动开发（TDD）** - **应该**遵循"红-绿-重构"流程
2. **高覆盖率** - 核心业务**必须**不低于 95%，整体**必须**不低于 90%
3. **正确使用 Mock** - **应该** Mock 外部依赖，**禁止** Mock 核心业务逻辑
4. **测试独立** - 测试**必须**互不依赖，**必须**可并行运行
5. **测试清晰** - 测试**应该**作为文档，**必须**一看就懂

---

## 测试驱动开发（TDD）

### TDD 三阶段

```
红（Red） -> 绿（Green） -> 重构（Refactor）
```

1. **红（Red）**：先写测试，运行失败
2. **绿（Green）**：写最简实现，让测试通过
3. **重构（Refactor）**：优化代码，保持测试通过

### TDD 常见误区

| 误区 | 问题 | 改进方法 |
|------|------|---------|
| 只写测试不重构 | 代码质量下降 | **必须**重构实现代码 |
| 一次写完所有测试 | 偏离 TDD 流程 | **应该**逐个测试编写 |
| 测试过于复杂 | 维护成本高 | **应该**保持测试简单 |
| 为覆盖率而测试 | 测试无实际意义 | **必须**测试行为而非代码行 |

---

## 测试覆盖率要求

| 代码类型 | 最低覆盖率 | 要求级别 |
|---------|----------|---------|
| **核心业务逻辑** | 95% | MUST |
| **API 层** | 90% | MUST |
| **工具函数** | 90% | SHOULD |
| **数据模型** | 80% | SHOULD |
| **配置管理** | 70% | SHOULD |
| **整体代码** | 90% | MUST |

### **可以**不覆盖的代码

- `main.go` 入口文件
- 自动生成的代码（protobuf、mock）
- 简单的 getter/setter
- 已 Mock 的外部依赖

### 查看覆盖率

```bash
go test ./... -cover -coverprofile=coverage.out
go tool cover -func=coverage.out
go tool cover -html=coverage.out -o coverage.html
```

---

## Mock 策略

### **应该** Mock 的场景

**外部依赖（数据库、网络、消息队列、时间）**：

```go
type MockRepository struct {
    mock.Mock
}

func (m *MockRepository) FindByID(ctx context.Context, id string) (*Order, error) {
    args := m.Called(ctx, id)
    return args.Get(0).(*Order), args.Error(1)
}
```

**时间相关操作**：

```go
type TimeProvider interface {
    Now() time.Time
}

type MockTimeProvider struct {
    mock.Mock
}

func (m *MockTimeProvider) Now() time.Time {
    return m.Called().Get(0).(time.Time)
}
```

### **禁止** Mock 的场景

**核心业务逻辑**：

```go
// 错误：Mock 核心业务逻辑
mockMatcher := new(MockMatcher)
mockMatcher.On("Match", ...).Return(expected, nil)

// 正确：测试真实业务逻辑
mockRepo := new(MockRepository)  // 只 Mock 外部依赖
matcher := NewMatcher(mockRepo)  // 使用真实业务逻辑
result, err := matcher.Match(ctx, criteria)
```

**简单数据结构和纯函数**：

```go
// 错误：Mock 简单结构体
type MockOrder struct { mock.Mock }

// 正确：直接使用真实结构体
order := &Order{ID: "001", Name: "test"}
```

### 过度 Mock 的危害

| 问题 | 影响 |
|------|------|
| 测试变成测 Mock | 未验证真实逻辑，测试失去意义 |
| 维护成本高 | 代码变更需同步修改大量 Mock |
| 假阳性 | 测试通过但实际代码有缺陷 |

---

## 单元测试规范

### 测试文件组织

```
internal/matcher/
+-- matcher.go           # 业务代码
+-- matcher_test.go      # 单元测试
+-- rule.go
+-- rule_test.go
+-- testdata/            # 测试数据
    +-- orders.json
    +-- rules.json
```

### 测试函数命名

```go
// 格式：Test<FunctionName>_<Scenario>
func TestFindByID_Success(t *testing.T) { ... }
func TestFindByID_NotFound(t *testing.T) { ... }
func TestFindByID_DatabaseError(t *testing.T) { ... }
```

### 测试结构（Given-When-Then）

```go
func TestMatcher_Match_Success(t *testing.T) {
    // Given（准备测试数据和依赖）
    mockRepo := new(MockRepository)
    mockRepo.On("FindByID", mock.Anything, "001").
        Return(&Order{ID: "001", GroupID: "g-01"}, nil)
    matcher := NewMatcher(mockRepo)

    // When（执行被测试的代码）
    result, err := matcher.Match(context.Background(), &Criteria{ID: "001"})

    // Then（验证结果）
    assert.NoError(t, err)
    assert.Equal(t, "g-01", result.GroupID)
    mockRepo.AssertExpectations(t)
}
```

### 表驱动测试

```go
func TestValidator_Validate(t *testing.T) {
    tests := []struct {
        name    string
        input   *Order
        wantErr bool
    }{
        {"valid order", &Order{ID: "001", Name: "test"}, false},
        {"empty ID", &Order{Name: "test"}, true},
        {"invalid name", &Order{ID: "001", Name: ""}, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := Validate(tt.input)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

### 测试边界条件

**必须**覆盖以下边界：
- 正常情况
- 空值/nil
- 零值
- 边界值（最大/最小）
- 错误情况

---

## 集成测试规范

### 适用场景

- 多个模块协作
- 涉及真实数据库操作
- 涉及消息队列
- 端到端业务流程

### 运行集成测试

```bash
# 只运行单元测试
go test ./... -short

# 运行包括集成测试
go test ./... -tags=integration

# 运行特定集成测试
go test ./internal/matcher -tags=integration -run TestQueryFlow
```

---

## 性能测试（Benchmark）

### 适用场景

- 核心算法
- 频繁调用的函数
- 性能敏感的操作

### Benchmark 示例

```go
func BenchmarkMatcher_Match(b *testing.B) {
    mockRepo := new(MockRepository)
    mockRepo.On("FindByID", mock.Anything, mock.Anything).
        Return(&Order{ID: "001"}, nil)

    matcher := NewMatcher(mockRepo)
    criteria := &Criteria{ID: "001"}

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = matcher.Match(context.Background(), criteria)
    }
}
```

### 运行性能测试

```bash
go test -bench=. ./...
go test -bench=. -benchmem ./...
go test -bench=. -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof
```

---

## 测试工具

### 断言库

**应该**使用 testify/assert：

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// assert：失败后继续执行
assert.Equal(t, expected, actual)

// require：失败后立即终止
require.NoError(t, err)
```

---

## 检查清单

### 测试编写检查

- [ ] **应该**遵循 TDD 流程（红 -> 绿 -> 重构）
- [ ] 测试覆盖率达标（核心不低于 95%，整体不低于 90%）
- [ ] **应该**使用 Given-When-Then 结构
- [ ] 测试命名清晰（Test\<Function\>\_\<Scenario\>）
- [ ] 边界条件有覆盖

### Mock 使用检查

- [ ] 只 Mock 外部依赖
- [ ] **禁止** Mock 核心业务逻辑
- [ ] **禁止** Mock 简单结构体
- [ ] **应该**验证 Mock 调用（AssertExpectations）

### 测试质量检查

- [ ] 测试独立（可并行运行）
- [ ] 单元测试执行时间小于 1 秒
- [ ] 测试清晰（一看就懂）
- [ ] 无 flaky 测试

### 运行检查

- [ ] `go test ./...` 全部通过
- [ ] `go test ./... -cover` 覆盖率达标
- [ ] `go test -bench=.` 性能无退化

---

## 参考资料

- Go Testing 官方文档：https://pkg.go.dev/testing
- 表驱动测试：https://go.dev/wiki/TableDrivenTests
- testify：https://github.com/stretchr/testify
- Uber Go Style Guide（Testing）：https://github.com/uber-go/guide
