# 文件与包设计

本文档定义 Go 项目的文件规模、包组织、依赖管理和目录结构规范。

---

## 核心原则

1. **文件规模控制** - 单个 Go 文件**必须**不超过 800 行
2. **单一职责** - 每个包/文件**必须**只做一件事
3. **高内聚低耦合** - 包内功能**必须**高度相关，包间依赖**应该**最小化
4. **按功能分包** - **应该**按业务功能分包，**禁止**按技术层次分包
5. **依赖选型** - **应该**优先使用标准库，**必须**严格评估第三方依赖

---

## 文件规模控制

### 文件长度限制

- 单个 Go 文件**必须**不超过 800 行
- 核心业务文件**应该**不超过 600 行
- 接口定义文件**应该**不超过 400 行

### 文件拆分策略

#### 1. 按职责拆分

```go
// 错误：一个文件包含所有功能（1200 行）
// order.go - 所有逻辑

// 正确：按职责拆分
// order.go - 核心类型定义（150 行）
// order-crud.go - CRUD 操作（200 行）
// order-query.go - 查询操作（180 行）
// order-validation.go - 验证逻辑（120 行）
```

#### 2. 按业务场景拆分

```go
// matcher/endpoint-match.go - 端侧匹配（250 行）
// matcher/network-match.go - 网侧匹配（280 行）
// matcher/rule.go - 规则处理（220 行）
```

#### 3. 按生命周期拆分

```go
// server/init.go - 初始化逻辑（180 行）
// server/cache.go - 缓存管理（200 行）
// server/cron.go - 定时任务（150 行）
// server/api.go - API 服务器（120 行）
```

---

## 包设计

### 包组织原则

**按功能分包（应该）**：

```
internal/
+-- matcher/        # 匹配功能
+-- model/          # 数据模型
|   +-- order/      # 订单模型
|   +-- config/     # 配置模型
+-- eventchange/    # 事件变更处理
```

**按技术分包（禁止）**：

```
internal/
+-- handlers/       # 所有 HTTP handler
+-- services/       # 所有业务逻辑
+-- repositories/   # 所有数据访问
+-- models/         # 所有数据模型
```

### 包命名规范

- 包名**必须**全小写，**禁止**下划线或驼峰
- 包名**应该**简短且有意义（1～2 个单词）
- 包名**应该**是单数形式

```go
// 推荐
package order
package matcher
package config

// 不推荐
package orders        // 不必要的复数
package orderService  // 驼峰命名
package order_model   // 下划线
```

### 包的单一职责

每个包**必须**只负责一个业务领域：

```go
// 正确：职责清晰
package matcher      // 只负责匹配判定
package eventchange  // 只负责事件变更处理

// 错误：职责不清
package utils        // 包含各种不相关的工具函数
package common       // 包含各种通用代码
```

### 包内聚度判定

**方法 1：删除法** — 从包中移除一个类型，问"包的核心功能是否受损？"

- 受损 → 属于此包
- 不受损 → 可能不属于此包，考虑移出

**方法 2：命名法** — 包内导出标识符是否都能用包名作为自然前缀？

```go
// matcher 包
// matcher.Match  ✓ 自然
// matcher.Rule   ✓ 自然
// matcher.SendEmail  ✗ 不属于此包
```

**拆分信号**：

| 信号 | 行动 |
|------|------|
| 包内有不相关的类型群 | 拆为多个包 |
| 导出标识符超过 15 个 | 考虑拆包 |
| 包名需要"and"描述 | 拆为两个包 |

**合并信号**：

| 信号 | 行动 |
|------|------|
| 两个包总是一起修改 | 合并为一个包 |
| 包 A 的大部分导出只有包 B 用 | A 并入 B |
| 包内只有 1-2 个类型 | 考虑合并到相关包 |

### 包 API 面积控制

包导出的标识符越少越好：

| 指标 | 推荐范围 | 说明 |
|------|---------|------|
| 导出函数/方法数 | **应该**不超过 15 个 | 超过说明职责过宽 |
| 导出类型数 | **应该**不超过 5 个 | 超过考虑拆包 |
| 导出接口数 | **应该**不超过 3 个 | 接口应小且精 |

**规则**：默认不导出，有明确外部需求时才导出。

### internal vs pkg

**internal/** - 项目私有代码：
- **必须**放置所有业务逻辑
- 外部项目**禁止**导入

**pkg/** - 可复用的公共库：
- **应该**放置可被其他项目使用的代码
- **必须**保持稳定的 API

```
project/
+-- internal/       # 业务逻辑（外部不可导入）
|   +-- matcher/
|   +-- model/
+-- pkg/            # 公共库（外部可导入）
|   +-- identifier/ # 接口定义
+-- cmd/            # 程序入口
```

---

## 第三方依赖管理

### 依赖选型原则

**应该**优先使用标准库：

```go
// 优先使用标准库
import (
    "encoding/json"
    "net/http"
    "sync"
)

// 仅在标准库无法满足时使用第三方库
import "github.com/knadh/koanf/v2"
```

### 依赖评估标准

| 评估项 | 要求 | 说明 |
|--------|------|------|
| **活跃度** | 最近 6 个月**必须**有更新 | 检查最后提交时间 |
| **维护性** | Issue 响应时间**应该**不超过 7 天 | 查看 Issue 列表 |
| **成熟度** | Star 数**应该**不少于 1000 | 社区认可度 |
| **许可证** | **必须**兼容项目许可证 | 避免法律风险 |
| **依赖树** | 依赖数**应该**不超过 10 | 避免依赖地狱 |
| **安全性** | **必须**无已知漏洞 | 使用 `govulncheck` |

### 推荐的开源库

| 类别 | 库 | 说明 |
|------|------|------|
| HTTP | `net/http`（标准库） | 优先使用 |
| MongoDB | `go.mongodb.org/mongo-driver/v2` | 推荐 |
| Redis | `github.com/redis/go-redis/v9` | 推荐 |
| 日志 | `log/slog`（标准库） | 推荐 |
| 测试 | `github.com/stretchr/testify` | 推荐 |
| 错误处理 | `errors`、`fmt`（标准库） | 优先使用 |

### 依赖版本管理

**必须**锁定依赖版本，**应该**定期更新依赖：

```bash
# 检查可更新的依赖
go list -u -m all

# 更新所有依赖
go get -u ./...

# 扫描已知漏洞
govulncheck ./...
```

---

## 目录结构最佳实践

### 标准项目布局

```
project/
+-- cmd/                # 程序入口
|   +-- app/
|       +-- main.go
+-- internal/           # 私有代码
|   +-- matcher/        # 业务功能
|   +-- model/          # 数据模型
|   +-- server/         # 服务管理
+-- pkg/                # 公共库
+-- api/                # API 定义
+-- configs/            # 配置文件
+-- scripts/            # 脚本
+-- deploy/             # 部署配置
+-- docs/               # 文档
+-- go.mod              # 依赖管理
+-- README.md           # 项目说明
```

---

## 文件组织最佳实践

### 文件内容组织顺序

```go
package packagename

// 1. 包注释
// Package packagename 提供核心功能

// 2. import 分组
import (
    // 标准库
    "context"
    "fmt"

    // 第三方库
    "github.com/redis/go-redis/v9"

    // 项目内部库
    "project/internal/model"
)

// 3. 常量定义
const DefaultTimeout = 5 * time.Second

// 4. 变量定义
var ErrNotFound = errors.New("not found")

// 5. 类型定义
type Item struct { ... }

// 6. 接口定义
type Repository interface { ... }

// 7. 函数实现
func NewItem() *Item { ... }
```

### 文件命名规范

```
item.go           # 核心类型定义
item_test.go      # 测试文件
item-crud.go      # CRUD 操作
item-query.go     # 查询操作
doc.go            # 包文档
```

---

## 检查清单

### 文件规模检查

- [ ] 单个文件**必须**不超过 800 行
- [ ] 核心文件**应该**不超过 600 行
- [ ] 超长文件**必须**按职责拆分

### 包设计检查

- [ ] 包**必须**按功能分组
- [ ] 包名**必须**全小写，无下划线
- [ ] 每个包**必须**职责单一
- [ ] **禁止**有 "utils"、"common" 包
- [ ] 包内聚度通过删除法/命名法验证
- [ ] 导出函数/方法不超过 15 个
- [ ] 导出类型不超过 5 个
- [ ] 默认不导出，有需求才导出

### 依赖管理检查

- [ ] **应该**优先使用标准库
- [ ] 第三方依赖**必须**通过评估
- [ ] 依赖版本**必须**锁定
- [ ] **必须**定期运行 `govulncheck`

### 目录结构检查

- [ ] 遵循标准项目布局
- [ ] `internal/` 用于业务代码
- [ ] `pkg/` 用于可复用库
- [ ] 配置文件**必须**分离

---

## 参考资料

- Go 项目布局标准：https://github.com/golang-standards/project-layout
- Go 包命名规范：https://go.dev/blog/package-names
- Go Modules 官方文档：https://go.dev/ref/mod
- Uber Go Style Guide：https://github.com/uber-go/guide
