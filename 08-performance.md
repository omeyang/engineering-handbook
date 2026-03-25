# 性能标准

本文档定义 Go 项目的性能意识、数据库优化、缓存策略、并发优化和内存优化标准。

---

## 核心原则

1. **性能意识** - 每个函数**应该**考虑性能影响
2. **先测量，再优化** - **禁止**过早优化
3. **优化热点** - **应该**专注于 20% 的关键代码
4. **平衡复杂度** - **禁止**过度追求性能牺牲可读性
5. **性能回归测试** - **必须**防止性能退化

---

## 性能意识

### **必须**避免的问题

**1. N+1 查询**

```go
// 错误：N+1 查询
for _, item := range items {
    details := repo.FindDetails(item.ID)  // 每次都查数据库
}

// 正确：批量查询
itemIDs := extractIDs(items)
detailsMap := repo.BatchFindDetails(itemIDs)  // 一次查询
```

**2. 重复计算**

```go
// 错误：重复计算
for _, item := range items {
    if item.Value > calculateThreshold() {  // 每次循环都计算
        // 处理
    }
}

// 正确：提前计算
threshold := calculateThreshold()
for _, item := range items {
    if item.Value > threshold {
        // 处理
    }
}
```

**3. 不必要的内存分配**

```go
// 错误：频繁分配
for i := 0; i < 1000; i++ {
    data := make([]byte, 1024)  // 每次循环都分配
}

// 正确：重用 buffer
data := make([]byte, 1024)
for i := 0; i < 1000; i++ {
    // 重用 data
}
```

---

## 数据库优化

### 索引优化

- 常用查询字段**必须**建索引
- 复合索引**应该**按查询频率排序
- 唯一字段**应该**使用唯一索引

### 批量操作

**应该**使用批量操作替代逐个操作：

```go
// 错误：逐个插入
for _, item := range items {
    repo.Create(ctx, item)  // N 次数据库调用
}

// 正确：批量插入
repo.BatchCreate(ctx, items)  // 1 次数据库调用
```

### 投影优化

**应该**只查询需要的字段：

```go
// 错误：查询所有字段
item, err := repo.FindByID(ctx, id)

// 正确：只查询需要的字段
item, err := repo.FindByIDWithProjection(ctx, id, []string{"id", "name", "status"})
```

---

## 缓存策略

### 多级缓存

```
L1: 本地内存缓存 (LRU/LFU)
 | miss
L2: Redis 缓存
 | miss
L3: 数据库
```

### 缓存更新策略

缓存**必须**有过期时间，缓存更新策略**必须**合理。

**1. Cache Aside（旁路缓存）**

```go
// 读：先查缓存，miss 则查数据库并更新缓存
// 写：先更新数据库，再删除缓存
func Update(ctx context.Context, item *Item) error {
    if err := repo.Update(ctx, item); err != nil {
        return err
    }
    cache.Delete(ctx, fmt.Sprintf("item:%s", item.ID))
    return nil
}
```

**2. Read Through**

```go
func (r *ReadThroughCache) Get(ctx context.Context, key string) (*Item, error) {
    if val, found := r.cache.Get(key); found {
        return val.(*Item), nil
    }
    return r.loadFromDB(ctx, key)
}
```

**3. Write Through**

```go
func (r *WriteThroughCache) Set(ctx context.Context, key string, item *Item) error {
    if err := r.db.Save(ctx, item); err != nil {
        return err
    }
    r.cache.Set(key, item, ttl)
    return nil
}
```

---

## 并发优化

### Goroutine 池

**应该**使用 Goroutine 池限制并发数量：

```go
import "github.com/panjf2000/ants/v2"

pool, _ := ants.NewPool(100)
defer pool.Release()

for _, item := range items {
    item := item
    _ = pool.Submit(func() {
        ProcessItem(item)
    })
}
```

### sync.Pool 对象复用

**可以**使用 sync.Pool 复用对象减少内存分配：

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func ProcessData(data []byte) {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)
    // 使用 buf 处理数据
}
```

### errgroup 并发控制

**应该**使用 errgroup 管理并发错误：

```go
import "golang.org/x/sync/errgroup"

func BatchQuery(ctx context.Context, ids []string) ([]*Item, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]*Item, len(ids))

    for i, id := range ids {
        i, id := i, id
        g.Go(func() error {
            item, err := repo.FindByID(ctx, id)
            if err != nil {
                return err
            }
            results[i] = item
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

---

## 内存优化

### 避免内存泄漏

- 资源**必须**关闭
- Goroutine **必须**有生命周期管理
- **应该**避免大切片持有

```go
// 错误：大切片持有
func GetFirst10(data []byte) []byte {
    return data[:10]  // 持有整个 data 的引用
}

// 正确：复制所需部分
func GetFirst10(data []byte) []byte {
    result := make([]byte, 10)
    copy(result, data[:10])
    return result
}
```

### 字符串优化

**应该**使用 strings.Builder 拼接字符串：

```go
// 错误：字符串拼接
var result string
for _, s := range strs {
    result += s  // 每次都分配新内存
}

// 正确：使用 strings.Builder
var builder strings.Builder
for _, s := range strs {
    builder.WriteString(s)
}
result := builder.String()
```

---

## 常见优化技巧

### 1. 预分配切片容量

```go
// 错误：不预分配
var results []*Item
for _, id := range ids {
    results = append(results, query(id))
}

// 正确：预分配
results := make([]*Item, 0, len(ids))
for _, id := range ids {
    results = append(results, query(id))
}
```

### 2. 使用 map 而非线性查找

```go
// 错误：线性查找 O(n)
func Contains(slice []string, target string) bool {
    for _, s := range slice {
        if s == target { return true }
    }
    return false
}

// 正确：使用 map O(1)
seen := make(map[string]bool)
for _, s := range slice {
    seen[s] = true
}
```

### 3. 使用更高效的序列化

```go
// JSON (慢)
data, _ := json.Marshal(item)

// MessagePack (快)
data, _ := msgpack.Marshal(item)

// Protobuf (最快)
data, _ := proto.Marshal(item)
```

---

## 性能测试

### Benchmark 测试

```go
func BenchmarkQuery(b *testing.B) {
    repo := setupTestRepo()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = repo.FindByID(context.Background(), "001")
    }
}
```

### 性能分析

```bash
# CPU Profile
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Memory Profile
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof

# 生成可视化图表
go tool pprof -http=:8080 cpu.prof
```

### 性能基准

| 指标 | 要求 | 说明 |
|------|------|------|
| **响应时间** | P99 **应该**不超过 100ms | 99% 请求在 100ms 内完成 |
| **吞吐量** | **应该**不低于 1000 QPS | 每秒处理 1000 个请求 |
| **并发数** | **应该**不低于 100 | 支持 100 个并发请求 |
| **内存使用** | **应该**不超过 500MB | 单实例内存小于 500MB |
| **CPU 使用** | **应该**不超过 80% | CPU 使用率小于 80% |

---

## 检查清单

### 性能意识检查

- [ ] **必须**避免 N+1 查询
- [ ] **应该**避免重复计算
- [ ] **应该**避免不必要的内存分配
- [ ] 数据库查询**必须**有索引
- [ ] **应该**使用批量操作替代逐个操作

### 缓存检查

- [ ] 热点数据**应该**有缓存
- [ ] 缓存**必须**有过期时间
- [ ] 缓存更新策略**必须**合理

### 并发检查

- [ ] **应该**使用 Goroutine 池限制并发
- [ ] Goroutine **必须**有生命周期管理
- [ ] **禁止** Goroutine 泄漏

### 性能测试检查

- [ ] **应该**有 Benchmark 测试
- [ ] **禁止**性能回归

---

## 参考资料

- Go 性能优化：https://go.dev/doc/diagnostics
- pprof 使用指南：https://go.dev/blog/pprof
- Go 性能优化技巧：https://github.com/dgryski/go-perfbook
- Uber Go Style Guide：https://github.com/uber-go/guide
