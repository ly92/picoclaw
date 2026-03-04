# NewVersionCommand 命令分析

## 1. 函数位置

**主要文件**：
- `cmd/picoclaw/internal/version/command.go` (lines 11-22)
- `cmd/picoclaw/internal/helpers.go` (lines 12-55)

**命令路径**：`picoclaw version` 或 `picoclaw v`

---

## 2. 核心功能

`NewVersionCommand()` 创建一个简单的版本信息显示命令，用于展示 PicoClaw 的版本号、构建信息和 Go 版本。

**主要职责**：
1. 显示应用程序版本（带 Git 提交哈希）
2. 显示构建时间
3. 显示 Go 编译器版本
4. 提供品牌标识（🦞 Logo）

---

## 3. 命令结构

### 3.1 函数定义

```go
func NewVersionCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:     "version",
        Aliases: []string{"v"},
        Short:   "Show version information",
        Run: func(_ *cobra.Command, _ []string) {
            printVersion()
        },
    }

    return cmd
}
```

### 3.2 命令参数

**基本信息**：
- **Use**: `"version"` - 命令名称
- **Aliases**: `[]string{"v"}` - 支持简写 `picoclaw v`
- **Short**: 简短描述
- **Run**: 执行函数，调用 `printVersion()`

**参数和标志**：
- ❌ 无命令行参数
- ❌ 无标志（flags）
- ✅ 纯展示命令，只读操作

---

## 4. 版本信息系统

### 4.1 版本变量（helpers.go）

```go
const Logo = "🦞"  // PicoClaw 品牌标识

var (
    version   = "dev"           // 默认版本号
    gitCommit string            // Git 提交哈希（构建时注入）
    buildTime string            // 构建时间（构建时注入）
    goVersion string            // Go 版本（构建时注入）
)
```

**变量说明**：
- 这些变量通过 **ldflags** 在构建时注入
- 默认值 `version = "dev"` 用于开发环境
- Git 提交、构建时间、Go 版本在发布构建时填充

### 4.2 FormatVersion() - 版本格式化

```go
func FormatVersion() string {
    v := version
    if gitCommit != "" {
        v += fmt.Sprintf(" (git: %s)", gitCommit)
    }
    return v
}
```

**输出格式**：
- 无 Git 信息：`dev`
- 有 Git 信息：`v1.0.0 (git: a1b2c3d)`

### 4.3 FormatBuildInfo() - 构建信息格式化

```go
func FormatBuildInfo() (string, string) {
    build := buildTime
    goVer := goVersion
    if goVer == "" {
        goVer = runtime.Version()  // 回退到运行时版本
    }
    return build, goVer
}
```

**返回值**：
1. `build`: 构建时间字符串（如 `"2025-01-15T10:30:45Z"`）
2. `goVer`: Go 版本（如 `"go1.23.4"`）

**回退机制**：
- 如果 `goVersion` 未在构建时注入
- 自动使用 `runtime.Version()` 获取当前运行时版本

---

## 5. 执行流程

### 5.1 完整流程图

```
用户执行命令
    │
    ├─→ picoclaw version
    │   或
    └─→ picoclaw v
        │
        ▼
    NewVersionCommand() 创建命令
        │
        ▼
    Run 函数执行
        │
        ▼
    printVersion() 调用
        │
        ├─→ internal.Logo (🦞)
        ├─→ internal.FormatVersion()
        │   └─→ 格式化版本号和 Git 提交
        │
        ├─→ internal.FormatBuildInfo()
        │   └─→ 获取构建时间和 Go 版本
        │
        ▼
    输出到终端
        │
        ├─→ 🦞 picoclaw v1.0.0 (git: a1b2c3d)
        ├─→   Build: 2025-01-15T10:30:45Z
        └─→   Go: go1.23.4
```

### 5.2 printVersion() 详解

```go
func printVersion() {
    fmt.Printf("%s picoclaw %s\n", internal.Logo, internal.FormatVersion())
    build, goVer := internal.FormatBuildInfo()
    if build != "" {
        fmt.Printf("  Build: %s\n", build)
    }
    if goVer != "" {
        fmt.Printf("  Go: %s\n", goVer)
    }
}
```

**输出逻辑**：
1. **第 1 行**：始终输出 Logo + 版本号
2. **第 2 行**：如果有构建时间，输出 Build
3. **第 3 行**：如果有 Go 版本，输出 Go

---

## 6. 构建时版本注入

### 6.1 Makefile 中的 ldflags

根据 CLAUDE.md，版本信息通过 ldflags 注入：

```makefile
VERSION ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
GIT_COMMIT := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
BUILD_TIME := $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
GO_VERSION := $(shell go version | awk '{print $$3}')

LDFLAGS := -X 'github.com/sipeed/picoclaw/cmd/picoclaw/internal.version=$(VERSION)' \
           -X 'github.com/sipeed/picoclaw/cmd/picoclaw/internal.gitCommit=$(GIT_COMMIT)' \
           -X 'github.com/sipeed/picoclaw/cmd/picoclaw/internal.buildTime=$(BUILD_TIME)' \
           -X 'github.com/sipeed/picoclaw/cmd/picoclaw/internal.goVersion=$(GO_VERSION)'
```

### 6.2 构建命令示例

```bash
# 开发构建（使用默认值）
go build -o picoclaw ./cmd/picoclaw

# 发布构建（注入版本信息）
go build -ldflags="$(LDFLAGS)" -o picoclaw ./cmd/picoclaw
```

---

## 7. 使用示例

### 示例 1：查看版本（完整命令）

```bash
$ picoclaw version
🦞 picoclaw v1.0.0 (git: a1b2c3d)
  Build: 2025-01-15T10:30:45Z
  Go: go1.23.4
```

### 示例 2：查看版本（简写）

```bash
$ picoclaw v
🦞 picoclaw v1.0.0 (git: a1b2c3d)
  Build: 2025-01-15T10:30:45Z
  Go: go1.23.4
```

### 示例 3：开发环境（无构建信息）

```bash
$ go run ./cmd/picoclaw version
🦞 picoclaw dev
  Go: go1.23.4
```

**说明**：
- 使用 `go run` 时，没有注入版本信息
- 只显示 `dev` 版本和运行时 Go 版本

### 示例 4：Git 脏构建

```bash
$ git describe --tags --always --dirty
v1.0.0-dirty

$ picoclaw version
🦞 picoclaw v1.0.0-dirty (git: a1b2c3d)
  Build: 2025-01-15T10:30:45Z
  Go: go1.23.4
```

### 示例 5：CI/CD 环境

```bash
# GitHub Actions 中的构建
$ picoclaw version
🦞 picoclaw v1.2.3 (git: f4e5d6c)
  Build: 2025-01-20T08:15:30Z
  Go: go1.23.5
```

### 示例 6：在脚本中使用

```bash
#!/bin/bash
VERSION=$(picoclaw version | grep picoclaw | awk '{print $3}')
echo "Detected version: $VERSION"

if [[ "$VERSION" == "dev" ]]; then
    echo "Development build detected"
else
    echo "Release build: $VERSION"
fi
```

### 示例 7：Docker 镜像版本

```dockerfile
# Dockerfile 中检查版本
RUN /app/picoclaw version && \
    /app/picoclaw version | grep -q "v1.0.0"
```

### 示例 8：版本比较

```bash
# 检查是否为最新版本
CURRENT_VERSION=$(picoclaw version | grep picoclaw | awk '{print $3}')
LATEST_VERSION=$(curl -s https://api.github.com/repos/sipeed/picoclaw/releases/latest | jq -r .tag_name)

if [[ "$CURRENT_VERSION" == "$LATEST_VERSION" ]]; then
    echo "You are running the latest version"
else
    echo "Update available: $LATEST_VERSION (current: $CURRENT_VERSION)"
fi
```

### 示例 9：调试信息收集

```bash
# 收集完整的调试信息
echo "=== PicoClaw Debug Info ===" > debug.txt
picoclaw version >> debug.txt
echo "" >> debug.txt
uname -a >> debug.txt
```

### 示例 10：帮助信息中的版本

```bash
$ picoclaw version --help
Show version information

Usage:
  picoclaw version [flags]

Aliases:
  version, v

Flags:
  -h, --help   help for version
```

---

## 8. 关键设计要点

### 8.1 极简设计

**特点**：
- ✅ 无参数、无标志、无配置
- ✅ 纯展示命令，只读操作
- ✅ 0 依赖（除了标准库）
- ✅ 执行速度 <1ms

**代码行数**：
- 命令定义：12 行
- 辅助函数：11 行（printVersion）
- 版本系统：38 行（helpers.go）
- **总计**：61 行

### 8.2 构建时注入模式

**优势**：
1. **不需要读取文件**：版本信息编译到二进制中
2. **零运行时开销**：直接访问变量，无需解析
3. **单一真实来源**：Git 标签作为版本来源
4. **自动化友好**：构建系统自动注入

**注入路径**：
```
Git 标签 → Makefile 变量 → ldflags → 编译时常量 → 运行时访问
```

### 8.3 回退机制

**3 层回退**：

1. **版本号**：
   - 优先：Git 标签（`v1.0.0`）
   - 回退：`"dev"`

2. **Git 提交**：
   - 优先：短提交哈希（`a1b2c3d`）
   - 回退：不显示（空字符串）

3. **Go 版本**：
   - 优先：构建时注入的版本
   - 回退：`runtime.Version()`（运行时版本）

### 8.4 品牌一致性

**Logo 使用**：
- ✅ 所有输出都包含 🦞
- ✅ 增强品牌识别度
- ✅ 在终端中易于视觉定位

**命名约定**：
- 项目名：`picoclaw`（小写）
- Logo：🦞（龙虾）
- 版本格式：`v1.0.0` 或 `dev`

---

## 9. 与其他命令的关系

### 9.1 内部使用

**其他命令可能使用版本信息**：

```go
// 在网关启动时记录版本
logger.Infof("Starting PicoClaw gateway %s", internal.FormatVersion())

// 在错误报告中包含版本
fmt.Fprintf(os.Stderr, "Error in %s: %v\n", internal.FormatVersion(), err)
```

### 9.2 版本检查（潜在扩展）

**未来可能的功能**：

```go
// 检查更新
picoclaw version --check-update

// 显示详细信息
picoclaw version --verbose

// JSON 输出（用于脚本）
picoclaw version --json
```

---

## 10. 测试场景

### 10.1 单元测试示例

```go
func TestFormatVersion(t *testing.T) {
    tests := []struct {
        version   string
        gitCommit string
        want      string
    }{
        {"v1.0.0", "a1b2c3d", "v1.0.0 (git: a1b2c3d)"},
        {"dev", "", "dev"},
        {"v2.0.0-rc1", "f4e5d6c", "v2.0.0-rc1 (git: f4e5d6c)"},
    }

    for _, tt := range tests {
        version = tt.version
        gitCommit = tt.gitCommit
        got := FormatVersion()
        if got != tt.want {
            t.Errorf("FormatVersion() = %q, want %q", got, tt.want)
        }
    }
}
```

### 10.2 集成测试

```bash
# 测试命令可用性
picoclaw version
picoclaw v

# 测试输出格式
picoclaw version | grep "🦞 picoclaw"

# 测试退出码
picoclaw version && echo "Success" || echo "Failed"
```

---

## 11. 注意事项

### 11.1 构建要求

**正确的版本信息需要**：
1. ✅ 使用 `make build` 而非 `go build`
2. ✅ Git 仓库存在（用于提取标签和提交）
3. ✅ 正确配置 LDFLAGS
4. ✅ Go 1.23+ 编译器

### 11.2 开发环境注意事项

**使用 `go run` 时**：
- ⚠️ 版本显示为 `dev`
- ⚠️ 无 Git 提交信息
- ⚠️ 无构建时间
- ✅ Go 版本仍然可用（通过 runtime.Version()）

**解决方案**：
```bash
# 方法 1：使用构建后的二进制
make build
./bin/picoclaw version

# 方法 2：手动传递 ldflags
go run -ldflags="-X 'github.com/sipeed/picoclaw/cmd/picoclaw/internal.version=dev-test'" \
    ./cmd/picoclaw version
```

### 11.3 版本命名规范

**遵循语义化版本（SemVer）**：
- `v1.0.0` - 主版本.次版本.补丁版本
- `v1.0.0-rc1` - 发布候选
- `v1.0.0-beta1` - 测试版本
- `v1.0.0-dirty` - 脏构建（有未提交更改）

---

## 12. 相关文件索引

### 12.1 核心文件

| 文件 | 路径 | 行数 | 说明 |
|------|------|------|------|
| command.go | `cmd/picoclaw/internal/version/command.go` | 34 | 命令定义 |
| helpers.go | `cmd/picoclaw/internal/helpers.go` | 56 | 版本系统 |

### 12.2 构建文件

| 文件 | 路径 | 说明 |
|------|------|------|
| Makefile | `Makefile` | 构建规则和 LDFLAGS |
| CLAUDE.md | `CLAUDE.md` | 版本注入文档 |

### 12.3 相关包

| 包 | 说明 |
|----|------|
| `runtime` | 获取运行时 Go 版本 |
| `fmt` | 格式化输出 |

---

## 13. 总结

### 13.1 命令特点

| 特性 | 说明 |
|------|------|
| **复杂度** | ⭐☆☆☆☆ 极简 |
| **依赖性** | 0 外部依赖 |
| **执行时间** | <1ms |
| **代码行数** | 61 行 |
| **用户交互** | 无（纯输出） |
| **错误可能性** | 极低 |

### 13.2 核心价值

1. **快速诊断**：
   - 用户报告问题时，立即获取版本信息
   - CI/CD 中验证部署的版本

2. **可追溯性**：
   - Git 提交哈希确保每个构建可追溯
   - 构建时间帮助识别部署时间线

3. **调试支持**：
   - Go 版本信息帮助诊断编译器相关问题
   - 版本信息可包含在错误报告中

### 13.3 与其他命令的对比

| 命令 | 复杂度 | 子命令 | 外部依赖 | 用途 |
|------|--------|--------|----------|------|
| **version** | ⭐ | 0 | 0 | 显示版本 |
| status | ⭐⭐ | 0 | Config | 显示状态 |
| onboard | ⭐⭐ | 0 | Config | 初始化 |
| auth | ⭐⭐⭐ | 4 | OAuth | 身份验证 |
| cron | ⭐⭐⭐⭐ | 5 | SQLite | 定时任务 |
| skills | ⭐⭐⭐⭐⭐ | 7 | Git/HTTP | 技能管理 |

### 13.4 最佳实践

**✅ 推荐**：
- 使用 `make build` 构建发布版本
- 在 CI/CD 中验证版本信息
- 在错误报告中包含 `picoclaw version` 输出
- 使用 Git 标签管理版本号

**❌ 避免**：
- 不要手动编辑版本变量
- 不要在开发环境中依赖版本号逻辑
- 不要跳过 Git 标签直接发布

---

## 14. 扩展阅读

### 14.1 相关文档

- CLAUDE.md：版本信息和构建系统
- Makefile：构建规则和 LDFLAGS
- README.md：版本管理说明

### 14.2 相关概念

- **语义化版本（SemVer）**：版本号命名规范
- **ldflags**：Go 构建时的链接器标志
- **runtime.Version()**：Go 运行时版本获取
- **Git describe**：从 Git 历史生成版本字符串

---

**文档版本**：1.0
**更新时间**：2025-01-XX
**对应代码版本**：PicoClaw main 分支

---

*本文档为 PicoClaw 命令分析系列第 8 篇，完整分析了 `NewVersionCommand()` 函数的实现、版本注入系统、使用场景和最佳实践。*
