# 命名准则

本文档定义 Go 项目中包、文件、类型、函数、变量等命名的统一规范。

---

## 核心原则

1. **见名知义** - 不看注释就知道用途
2. **简短准确** - 避免冗长，但必须准确
3. **禁止泛化** - 不用 manager/service/handler/util 等模糊词
4. **避免包名重复** - 导出名不要包含包名
5. **符合 Uber Go 规范**

---

## 避免包名重复

导出的类型、函数、接口名**不要包含包名**，因为调用时会重复。

```go
// 错误：包名重复
package user

type UserRepository interface { ... }
func QueryUser(id string) { ... }

// 调用时：
user.UserRepository  // 重复了 User
user.QueryUser()     // 重复了 User
```

```go
// 正确：简洁清晰
package user

type Repository interface { ... }
func Query(id string) { ... }

// 调用时：
user.Repository  // 清晰
user.Query()     // 简洁
```

---

## 包（Package）

### 规则

- 小写字母
- 简短（1～2 个单词）
- 单数形式
- 无下划线、无驼峰

### 正确示例

```
user
config
order
eventchange
```

### 错误示例

```
users          // 不用复数
userManager    // 不用驼峰
user_config    // 不用下划线
utils          // 太泛化
```

---

## 文件

### 规则

- kebab-case（短横线分隔）
- 见名知义
- 避免缩写

### 正确示例

```
order-handler.go
query-by-id.go
branch-tree.go
```

### 错误示例

```
orderHandler.go  // 不用驼峰
order_handler.go // 不用下划线
oh.go            // 不用缩写
utils.go         // 太泛化
```

---

## 类型（Type）

### 规则

- PascalCase
- 名词或名词短语
- 准确描述用途
- 禁止 Manager/Service/Handler 后缀
- **不要包含包名**

### 正确示例

```go
// 在 order 包中
type Matcher struct { ... }      // 调用：order.Matcher
type Repository struct { ... }   // 调用：order.Repository
type Validator struct { ... }    // 调用：order.Validator
```

### 错误示例

```go
// 在 order 包中
type OrderMatcher struct { ... }     // 包名重复
type OrderRepository struct { ... }  // 包名重复
type OrderManager struct { ... }     // 太泛化 + 包名重复
type Handler struct { ... }          // 完全不知道干什么
type Data struct { ... }             // 太模糊
```

---

## 接口（Interface）

### 规则

- PascalCase
- 单方法接口用 `-er` 后缀
- 多方法接口用名词
- **不要包含包名**
- 不用 I 前缀或 Interface 后缀

### 正确示例

```go
package order

type Repository interface { ... }  // 调用：order.Repository
type Matcher interface { ... }     // 调用：order.Matcher
type Finder interface { ... }      // 调用：order.Finder
```

### 错误示例

```go
package order

type OrderRepository interface { ... }  // 包名重复
type IRepository interface { ... }      // 不用 I 前缀
type RepositoryInterface interface { ... } // 不用 Interface 后缀
```

---

## 函数/方法

### 规则

- PascalCase（公开）/ camelCase（私有）
- 动词开头
- 准确描述操作
- **不要包含包名**

### 正确示例

```go
package order

func Query(id string) (*Order, error) { ... }
func Create(o *Order) error { ... }
func MatchByGroup(groupID string) ([]*Order, error) { ... }
func validateInput(id string) error { ... }
```

### 错误示例

```go
package order

func QueryOrder(id string) { ... }    // 包名重复
func Handle(data interface{}) { ... }  // 处理什么？
func Process(data interface{}) { ... } // 处理什么？
func Do() { ... }                      // 做什么？
```

---

## 变量

### 规则

- camelCase
- 简短但清晰
- 缩写全大写（ID/IP/HTTP）

### 正确示例

```go
orderID
userIP
httpClient
branchTree
```

### 错误示例

```go
OrderID       // 非导出变量不用 PascalCase
orderId       // 缩写不全大写
data          // 太模糊
temp          // 太模糊
```

---

## 常量

### 规则

- PascalCase 或全大写（枚举用 PascalCase）
- 有意义的名字

### 正确示例

```go
const MaxRetryCount = 3
const DefaultTimeout = 30 * time.Second

const (
    StatusActive   = "active"
    StatusInactive = "inactive"
)
```

### 错误示例

```go
const MAX_RETRY = 3    // 不用全大写+下划线
const max = 3          // 太模糊
```

---

## 特殊命名

### 接收者（Receiver）

- 1～2 个字母缩写
- 统一使用相同缩写

```go
func (o *Order) Create() { ... }
func (o *Order) Update() { ... }
```

### 上下文（Context）

- 第一个参数
- 命名为 `ctx`

```go
func Query(ctx context.Context, id string) (*Order, error) { ... }
```

### 错误（Error）

- 命名为 `err`
- 自定义错误类型用 `Err` 前缀

```go
var ErrNotFound = errors.New("not found")
var ErrInvalidInput = errors.New("invalid input")
```

---

## 禁止列表

### 绝对禁止的词

- `Manager` - 用具体动作代替（如 `Matcher`、`Locator`）
- `Service` - 用具体职责代替（如 `Repository`、`Validator`）
- `Handler` - 用具体处理代替（如 `Querier`、`Processor`）
- `Util/Utils` - 用具体功能代替（如 `StringValidator`、`TimeFormatter`）
- `Helper` - 用具体功能代替
- `Common` - 用具体功能代替
- `Base` - 用具体功能代替
- `Data` - 用具体类型代替

### 替代方案

```
OrderManager    ->  OrderRepository / OrderMatcher
UserService     ->  UserRepository / UserValidator
DataHandler     ->  DataProcessor / DataTransformer
StringUtil      ->  StringValidator / StringParser
```

---

## 检查清单

- [ ] 包名小写、简短、无下划线
- [ ] 文件名 kebab-case
- [ ] 类型名 PascalCase、见名知义
- [ ] 函数名动词开头、准确描述
- [ ] 变量名 camelCase、清晰
- [ ] **导出名不包含包名**
- [ ] 没有使用禁止词（Manager/Service/Handler/Util）
- [ ] 缩写全大写（ID/IP/HTTP/URL）
- [ ] 接收者名称统一
