# 重构流程准则

本文档定义代码重构的流程、兼容性保证、灰度发布和回滚策略。

---

## 核心原则

1. **功能等价** - 重构前后行为完全一致
2. **上游透明** - 调用方无需修改代码
3. **下游无感** - 依赖方接口保持兼容
4. **渐进式** - 小步快跑，持续验证

---

## 重构流程（7 步）

### 1. 保持旧代码运行

重构期间旧代码**必须**可用。

### 2. 新代码并行实现

实现新代码，但不删除旧代码。

### 3. 完整测试

- 单元测试覆盖率不低于 95%
- 集成测试验证上下游
- 回归测试确保旧功能不变

### 4. 灰度切流

部分流量切到新代码，观察运行。

### 5. 监控验证

对比新旧代码的行为、性能、错误率。

### 6. 全量切流

确认无问题后，全量切到新代码。

### 7. 移除旧代码

删除旧代码、清理注释、更新文档。

---

## 兼容性保证

### 接口兼容

**保留旧接口，内部调用新实现**：

```go
// 旧接口：保留
func Query(id string) (*Order, error) {
    return QueryWithOptions(id, nil)
}

// 新接口：扩展功能
func QueryWithOptions(id string, opts *QueryOptions) (*Order, error) {
    // 新实现
}
```

**使用可选参数（Functional Options）**：

```go
type QueryOption func(*QueryConfig)

func WithTimeout(d time.Duration) QueryOption {
    return func(c *QueryConfig) { c.Timeout = d }
}

func Query(id string, opts ...QueryOption) (*Order, error) {
    cfg := &QueryConfig{}
    for _, opt := range opts {
        opt(cfg)
    }
    // ...
}
```

### 类型兼容

**使用类型别名**：

```go
type OldOrder = Order  // 别名
type Order struct { ... }
```

**提供转换函数**：

```go
func ConvertOldToNew(old *OldOrder) *NewOrder { ... }
func ConvertNewToOld(new *NewOrder) *OldOrder { ... }
```

---

## 常见重构场景

### 重命名函数/类型

**方案 1：别名**

```go
var OldFunctionName = NewFunctionName

func NewFunctionName() { ... }
```

**方案 2：适配器**

```go
func OldFunction(a int) { NewFunction(a) }

func NewFunction(a int) { ... }
```

### 拆分函数

保留旧函数，内部调用拆分后的函数：

```go
func ProcessData(data []byte) error {
    if err := ValidateData(data); err != nil {
        return err
    }
    if err := TransformData(data); err != nil {
        return err
    }
    return SaveData(data)
}
```

### 修改函数签名

**新函数，旧函数调用新函数**：

```go
// 旧签名：保留
func Query(id string) (*Order, error) {
    return QueryWithContext(context.Background(), id)
}

// 新签名：增加 context
func QueryWithContext(ctx context.Context, id string) (*Order, error) {
    // ...
}
```

### 修改数据结构

**增加字段**：向后兼容

```go
type Order struct {
    ID   string
    Name string
    Tags []string  // 新增字段，可以为空
}
```

**删除字段**：标记废弃，延迟删除

```go
type Order struct {
    ID   string
    Name string
    // Deprecated: use Tags instead
    Tag  string    // 废弃字段，保留一段时间
    Tags []string
}
```

---

## 测试策略

### 单元测试

- 覆盖率不低于 95%
- 测试所有边界条件
- Mock 外部依赖

### 集成测试

- 验证上下游集成
- 测试完整流程
- 使用真实依赖

### 回归测试

- 对比新旧实现的输出
- 确保行为一致
- 记录差异并修复

### 性能测试

```bash
# 基准测试
go test -bench=. -benchmem

# 对比新旧性能
go test -bench=BenchmarkOld -benchmem > old.txt
go test -bench=BenchmarkNew -benchmem > new.txt
benchcmp old.txt new.txt
```

---

## 灰度发布

### 流量切分

```go
func Query(id string) (*Order, error) {
    if rand.Intn(100) < 10 {
        return queryNew(id)  // 10% 新实现
    }
    return queryOld(id)  // 90% 旧实现
}
```

### 特性开关（Feature Flag）

```go
func Query(id string) (*Order, error) {
    if config.EnableNewQuery {
        return queryNew(id)
    }
    return queryOld(id)
}
```

### 用户分组

```go
func Query(ctx context.Context, id string) (*Order, error) {
    userID := getUserID(ctx)
    if isInWhitelist(userID) {
        return queryNew(id)
    }
    return queryOld(id)
}
```

---

## 监控与回滚

### 监控指标

- 请求成功率
- 响应时间（P50/P90/P99）
- 错误率
- 业务指标

### 告警设置

- 成功率下降超过 5%
- P99 响应时间增加超过 50%
- 错误率增加超过 10%

### 回滚策略

**方案 1：配置回滚**

```yaml
enable_new_query: false
```

**方案 2：代码回滚**

```bash
git revert <commit>
git push
```

**方案 3：流量切换**

直接切回旧实现。

---

## 重构检查清单

### 开始前

- [ ] 确认重构目标明确
- [ ] 评估风险和收益
- [ ] 准备回滚方案

### 实现中

- [ ] 旧代码保持运行
- [ ] 新代码完整测试
- [ ] 接口向后兼容
- [ ] 添加监控指标

### 测试验证

- [ ] 单元测试覆盖率不低于 95%
- [ ] 集成测试通过
- [ ] 回归测试通过
- [ ] 性能测试无退化

### 发布后

- [ ] 灰度发布观察指标
- [ ] 监控无异常告警
- [ ] 上下游无反馈问题
- [ ] 全量后删除旧代码

---

## 禁止行为

- **禁止**直接删除旧接口
- **禁止**修改旧接口签名
- **禁止**重构没有测试覆盖
- **禁止**一次性全量切换
- **禁止**没有监控和告警
- **禁止**没有回滚方案
