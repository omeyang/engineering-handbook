# 质量门禁

本文档定义代码质量的自动化检查标准和 CI/CD 集成规范。

---

## 核心原则

1. **自动化检查** - 所有质量检查**必须**自动化执行
2. **左移质量** - 质量问题**必须**在开发阶段检出
3. **快速反馈** - 检查结果**必须**在提交后 5 分钟内反馈
4. **零容忍** - 不符合门禁标准的代码**禁止**合并
5. **持续改进** - 门禁规则**应该**根据实践调整优化

---

## 静态分析

### golangci-lint 配置

```yaml
version: "2"
linters:
  default: none
  enable:
    - errcheck
    - govet
```

**执行命令**：

```bash
golangci-lint run ./...
```

**门禁要求**：
- **必须**通过所有启用的 linters 检查
- **禁止**存在未处理的错误（errcheck）
- **禁止**存在未使用的变量/函数（unused）
- **禁止**存在安全漏洞（gosec）

### 常见问题修复

**errcheck：未处理错误**

```go
// 修复前
repo.Create(item)

// 修复后
if err := repo.Create(item); err != nil {
    return fmt.Errorf("create failed: %w", err)
}
```

**gosec：安全问题**

```go
// 修复前：SQL 注入风险
query := fmt.Sprintf("SELECT * FROM items WHERE id='%s'", id)

// 修复后：参数化查询
query := "SELECT * FROM items WHERE id=?"
db.Query(query, id)
```

---

## 复杂度控制

### 圈复杂度标准

| 复杂度 | 等级 | 处理方式 |
|--------|------|---------|
| 1～10  | 合格 | 允许提交 |
| 11～15 | 警告 | 建议重构，可提交 |
| 16～20 | 不合格 | **必须**重构后提交 |
| >20    | 严重 | **必须**立即重构 |

**检查命令**：

```bash
gocyclo -over 10 internal/
```

**门禁要求**：
- 核心业务逻辑复杂度**必须**不超过 10
- 其他代码复杂度**应该**不超过 15
- 函数长度**应该**不超过 70 行
- 函数参数**应该**不超过 3 个
- 嵌套层级**应该**不超过 3 层

### 降低复杂度策略

**策略 1：提取函数**

```go
// 提取前：复杂度 15
func Process(item *Item) error {
    // 验证、查询、更新、通知逻辑混在一起
}

// 提取后：复杂度 4
func Process(item *Item) error {
    if err := validate(item); err != nil { return err }
    if err := query(item); err != nil { return err }
    if err := update(item); err != nil { return err }
    return notify(item)
}
```

**策略 2：表驱动替代 if-else**

```go
// 替代前：复杂度 10
func GetMatcher(t string) Matcher {
    if t == "A" { return &MatcherA{} }
    // ... 10 个 if-else
}

// 替代后：复杂度 1
var matchers = map[string]Matcher{
    "A": &MatcherA{}, "B": &MatcherB{},
}
func GetMatcher(t string) Matcher { return matchers[t] }
```

---

## 测试覆盖率

### 覆盖率标准

| 代码类型 | 最低覆盖率 | 推荐覆盖率 |
|---------|----------|----------|
| 核心业务逻辑 | 90% | 95% |
| API 层 | 85% | 90% |
| 整体代码 | 80% | 90% |

**检查命令**：

```bash
go test ./... -cover -coverprofile=coverage.out
go tool cover -func=coverage.out
go tool cover -html=coverage.out -o coverage.html
```

**门禁要求**：
- 核心业务逻辑覆盖率**必须**不低于 90%
- 整体覆盖率**必须**不低于 80%
- 所有测试**必须**通过
- **禁止**存在 flaky 测试

---

## 安全检查

### govulncheck 漏洞扫描

```bash
govulncheck ./...
```

**门禁要求**：
- **禁止**存在已知漏洞
- **必须**定期更新依赖（每月一次）
- 高危漏洞**必须**在 24 小时内修复
- 中危漏洞**应该**在 1 周内修复

---

## CI/CD 集成

### 本地 Pre-commit Hook

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 格式化检查
if [ -n "$(gofmt -l .)" ]; then
    echo "Code is not formatted"
    exit 1
fi

# Lint 检查
golangci-lint run ./... || exit 1

# 测试
go test ./... -short || exit 1

# 复杂度检查
if gocyclo -over 15 internal/ | grep .; then
    echo "Complexity too high"
    exit 1
fi

echo "All checks passed"
```

### CI 配置示例

```yaml
stages:
  - quality
  - test

code-quality:
  stage: quality
  script:
    - golangci-lint run ./...
    - gocyclo -over 15 internal/
  allow_failure: false

test:
  stage: test
  script:
    - go test ./... -cover -coverprofile=coverage.out
    - go tool cover -func=coverage.out
  coverage: '/total:\s+\(statements\)\s+(\d+\.\d+)%/'
```

---

## 规格到工具映射

| 质量规格 | 检查工具 | 阈值/配置 | 门禁要求 |
|---------|---------|---------|---------|
| 未处理错误 | golangci-lint (errcheck) | 0 errors | MUST |
| 圈复杂度 | gocyclo | 不超过 10（核心），不超过 15（其他） | MUST |
| 函数长度 | golangci-lint (funlen) | 不超过 70 行 | SHOULD |
| 测试覆盖率 | go test -cover | 不低于 80%（整体），不低于 90%（核心） | MUST |
| 安全漏洞 | govulncheck | 0 vulnerabilities | MUST |
| 安全审计 | golangci-lint (gosec) | 0 issues | MUST |
| 代码格式 | gofmt / goimports | 100% formatted | MUST |
| 未使用代码 | golangci-lint (unused) | 0 unused | MUST |

---

## 质量门禁标准

### 提交前检查

- [ ] `gofmt -l .` 无输出
- [ ] `golangci-lint run ./...` 无错误
- [ ] `gocyclo -over 10 internal/` 无输出
- [ ] `go test ./...` 全部通过
- [ ] `go test ./... -cover` 覆盖率达标

### 合并前检查

- [ ] CI/CD 流水线全部通过
- [ ] Code Review 批准（不少于 2 人）
- [ ] 没有未解决的讨论
- [ ] 测试覆盖率不低于 80%
- [ ] 性能测试无退化

### 发布前检查

- [ ] 所有测试通过（单元、集成、E2E）
- [ ] 性能测试通过
- [ ] 安全扫描通过
- [ ] 文档已更新

---

## 参考资料

- golangci-lint：https://golangci-lint.run/
- gocyclo：https://github.com/fzipp/gocyclo
- govulncheck：https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck
- Go Code Review Comments：https://go.dev/wiki/CodeReviewComments
