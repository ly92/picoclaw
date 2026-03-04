# NewStatusCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/status/command.go`
**行数**: 7-18
**实现文件**: `cmd/picoclaw/internal/status/helpers.go`

---

## 🎯 核心功能

`NewStatusCommand` 是 PicoClaw 的**状态查看命令**，用于快速诊断系统配置和服务状态。这是一个**只读命令**，不会修改任何配置或数据。

**主要职责**：
1. 显示版本信息
2. 检查配置文件和工作空间是否存在
3. 显示当前使用的模型
4. 列出已配置的 API 提供商
5. 显示 OAuth/Token 认证状态

---

## 📊 结构拆解

### 1️⃣ **Cobra 命令定义** (command.go:7-18)

```go
func NewStatusCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:     "status",           // 命令名称
        Aliases: []string{"s"},      // 简短别名
        Short:   "Show picoclaw status",
        Run: func(cmd *cobra.Command, args []string) {
            statusCmd()
        },
    }

    return cmd
}
```

#### 特点：
- ✅ 提供简短别名 `s`
- ✅ 无需任何参数
- ✅ 无需标志位（flags）
- ✅ 执行简单，快速反馈

---

## 🔄 执行流程详解

### 核心函数：`statusCmd()` (helpers.go:11-100)

```
┌─────────────────────────────────────────────────────┐
│                statusCmd() 执行流程                  │
└─────────────────────────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 1. 加载配置文件                    │
    │    cfg = LoadConfig()             │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 2. 显示版本信息                    │
    │    • 版本号 (git commit)          │
    │    • 构建时间                      │
    │    • Go 版本                       │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 3. 检查配置文件存在性              │
    │    os.Stat(configPath)            │
    │    ✓ 存在 / ✗ 不存在              │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 4. 检查工作空间存在性              │
    │    os.Stat(workspace)             │
    │    ✓ 存在 / ✗ 不存在              │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 5. 显示当前模型                    │
    │    cfg.Agents.Defaults.ModelName  │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 6. 检查 API 提供商配置             │
    │    • OpenRouter                   │
    │    • Anthropic                    │
    │    • OpenAI                       │
    │    • Gemini                       │
    │    • Zhipu                        │
    │    • Qwen                         │
    │    • Groq                         │
    │    • Moonshot                     │
    │    • DeepSeek                     │
    │    • VolcEngine                   │
    │    • Nvidia                       │
    │    • vLLM/Local                   │
    │    • Ollama                       │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 7. 显示 OAuth/Token 认证状态       │
    │    auth.LoadStore()               │
    │    • 提供商名称                    │
    │    • 认证方法 (oauth/token)       │
    │    • 状态 (authenticated/expired) │
    └───────────────────────────────────┘
```

---

## 📊 输出示例

### **完整配置的系统**

```bash
picoclaw status

# 输出:
🦞 picoclaw Status
Version: v1.0.0 (git: bea238c)
Build: 2026-03-04 10:30:00 UTC

Config: /Users/username/.picoclaw/config.json ✓
Workspace: /Users/username/.picoclaw/workspace ✓
Model: gpt-5.2
OpenRouter API: not set
Anthropic API: ✓
OpenAI API: ✓
Gemini API: not set
Zhipu API: not set
Qwen API: not set
Groq API: not set
Moonshot API: not set
DeepSeek API: not set
VolcEngine API: not set
Nvidia API: not set
vLLM/Local: not set
Ollama: ✓ http://localhost:11434

OAuth/Token Auth:
  openai (oauth): authenticated
  anthropic (token): authenticated
```

---

### **未配置的系统**

```bash
picoclaw status

# 输出:
🦞 picoclaw Status
Version: v1.0.0 (git: bea238c)

Config: /Users/username/.picoclaw/config.json ✗
Workspace: /Users/username/.picoclaw/workspace ✗

# 提示用户运行 onboard
```

---

### **部分配置的系统**

```bash
picoclaw status

# 输出:
🦞 picoclaw Status
Version: v1.0.0 (git: bea238c)

Config: /Users/username/.picoclaw/config.json ✓
Workspace: /Users/username/.picoclaw/workspace ✓
Model: claude-sonnet-4.6
OpenRouter API: not set
Anthropic API: ✓
OpenAI API: not set
Gemini API: not set
Zhipu API: not set
Qwen API: not set
Groq API: not set
Moonshot API: not set
DeepSeek API: not set
VolcEngine API: not set
Nvidia API: not set
vLLM/Local: not set
Ollama: not set

OAuth/Token Auth:
  anthropic (token): authenticated
```

---

## 📦 使用示例

### **1. 标准使用**
```bash
picoclaw status
```

### **2. 使用别名**
```bash
picoclaw s    # 等同于 picoclaw status
```

### **3. 故障排查流程**

```bash
# 场景 1: Agent 命令失败
picoclaw agent -m "Hello"
# Error: error loading config

# 检查状态
picoclaw status
# Config: ... ✗  ← 配置文件不存在

# 解决方案
picoclaw onboard
```

```bash
# 场景 2: 认证失败
picoclaw agent -m "Hello"
# Error: authentication required

# 检查状态
picoclaw status
# OpenAI API: not set  ← API Key 未配置

# 解决方案
picoclaw auth login --provider openai
```

```bash
# 场景 3: 模型错误
picoclaw agent -m "Hello"
# Error: model not found

# 检查状态
picoclaw status
# Model: invalid-model  ← 模型名称错误

# 解决方案
vim ~/.picoclaw/config.json
# 修正 agents.defaults.model
```

### **4. 脚本集成**

```bash
#!/bin/bash
# 健康检查脚本

if ! picoclaw status | grep -q "Config.*✓"; then
    echo "PicoClaw not configured"
    exit 1
fi

if ! picoclaw status | grep -q "Workspace.*✓"; then
    echo "Workspace missing"
    exit 1
fi

echo "PicoClaw ready"
exit 0
```

### **5. 监控集成**

```bash
# Prometheus exporter 示例
picoclaw status | awk '/API:/ {print $1, $2}' | while read provider status; do
    if [[ "$status" == "✓" ]]; then
        echo "picoclaw_provider_configured{provider=\"$provider\"} 1"
    else
        echo "picoclaw_provider_configured{provider=\"$provider\"} 0"
    fi
done
```

---

## 🔍 详细信息解析

### **1. 版本信息**

```go
fmt.Printf("Version: %s\n", internal.FormatVersion())
// 格式: v1.0.0 (git: bea238c)

build, _ := internal.FormatBuildInfo()
fmt.Printf("Build: %s\n", build)
// 格式: 2026-03-04 10:30:00 UTC
```

**构建时注入**：
```bash
# Makefile 中通过 ldflags 注入
go build -ldflags "\
  -X 'main.version=v1.0.0' \
  -X 'main.gitCommit=$(git rev-parse --short HEAD)' \
  -X 'main.buildTime=$(date -u +%Y-%m-%d\ %H:%M:%S\ UTC)' \
  -X 'main.goVersion=$(go version)'"
```

---

### **2. 文件系统检查**

```go
if _, err := os.Stat(configPath); err == nil {
    fmt.Println("Config:", configPath, "✓")
} else {
    fmt.Println("Config:", configPath, "✗")
}
```

**检查项**：
- ✅ 配置文件存在且可读
- ✅ 工作空间目录存在

**路径**：
- 配置: `~/.picoclaw/config.json`
- 工作空间: `~/.picoclaw/workspace`

---

### **3. API 提供商检查**

```go
hasOpenAI := cfg.Providers.OpenAI.APIKey != ""
status := func(enabled bool) string {
    if enabled {
        return "✓"
    }
    return "not set"
}
fmt.Println("OpenAI API:", status(hasOpenAI))
```

**检查逻辑**：
- API Key 提供商：检查 `api_key` 字段是否非空
- 本地提供商：检查 `api_base` 字段是否非空

**支持的提供商**（13 个）：
| 提供商 | 检查字段 | 类型 |
|--------|---------|------|
| OpenRouter | `api_key` | Cloud |
| Anthropic | `api_key` | Cloud |
| OpenAI | `api_key` | Cloud |
| Gemini | `api_key` | Cloud |
| Zhipu (智谱) | `api_key` | Cloud |
| Qwen (通义千问) | `api_key` | Cloud |
| Groq | `api_key` | Cloud |
| Moonshot | `api_key` | Cloud |
| DeepSeek | `api_key` | Cloud |
| VolcEngine | `api_key` | Cloud |
| Nvidia | `api_key` | Cloud |
| vLLM | `api_base` | Local |
| Ollama | `api_base` | Local |

---

### **4. OAuth/Token 认证状态**

```go
store, _ := auth.LoadStore()
if store != nil && len(store.Credentials) > 0 {
    fmt.Println("\nOAuth/Token Auth:")
    for provider, cred := range store.Credentials {
        status := "authenticated"
        if cred.IsExpired() {
            status = "expired"
        } else if cred.NeedsRefresh() {
            status = "needs refresh"
        }
        fmt.Printf("  %s (%s): %s\n", provider, cred.AuthMethod, status)
    }
}
```

**状态分类**：
| 状态 | 含义 | 操作建议 |
|------|------|---------|
| `authenticated` | 认证有效 | 无需操作 |
| `expired` | 令牌已过期 | 重新登录 |
| `needs refresh` | 即将过期 | 自动刷新（OAuth） |

**认证方法**：
- `oauth`: OAuth 2.0 流程
- `token`: API Key 粘贴

---

## 🔑 关键设计要点

### **1. 只读操作**

```go
// 不修改任何配置或状态
func statusCmd() {
    cfg, _ := internal.LoadConfig()  // 只读
    os.Stat(configPath)              // 只检查
    auth.LoadStore()                 // 只读
}
```

**特点**：
- ✅ 安全：不会破坏现有配置
- ✅ 快速：无需等待网络请求
- ✅ 无副作用：可重复执行

---

### **2. 容错设计**

```go
cfg, err := internal.LoadConfig()
if err != nil {
    fmt.Printf("Error loading config: %v\n", err)
    return  // 优雅退出，不崩溃
}
```

**容错策略**：
- 配置加载失败：显示错误但不退出进程
- 工作空间缺失：标记 ✗ 但继续检查其他项
- 认证状态异常：跳过认证部分

---

### **3. 视觉反馈**

```go
status := func(enabled bool) string {
    if enabled {
        return "✓"  // 绿色勾号（在支持的终端）
    }
    return "not set"  // 明确的未配置提示
}
```

**符号含义**：
- `✓`: 已配置且可用
- `✗`: 缺失或不可访问
- `not set`: 未配置

---

### **4. 分层信息展示**

```
第一层：基本信息
  • 版本号
  • 构建信息

第二层：核心配置
  • 配置文件路径
  • 工作空间路径
  • 当前模型

第三层：提供商状态
  • 13 个 API 提供商
  • 配置状态

第四层：认证状态（可选）
  • OAuth/Token 认证
  • 过期状态
```

---

## 🔗 相关函数调用链

```
NewStatusCommand()
    └─ cmd.Run → statusCmd()
        ├─ internal.LoadConfig()
        │   └─ config.Load(configPath)
        │
        ├─ internal.GetConfigPath()
        │   └─ 返回配置文件路径
        │
        ├─ internal.FormatVersion()
        │   └─ 格式化版本信息（git commit）
        │
        ├─ internal.FormatBuildInfo()
        │   └─ 格式化构建信息（时间、Go 版本）
        │
        ├─ os.Stat(configPath)
        │   └─ 检查文件是否存在
        │
        ├─ os.Stat(workspace)
        │   └─ 检查目录是否存在
        │
        ├─ cfg.Agents.Defaults.GetModelName()
        │   └─ 获取当前模型名称
        │
        ├─ 检查各提供商 API Key
        │   ├─ cfg.Providers.OpenAI.APIKey
        │   ├─ cfg.Providers.Anthropic.APIKey
        │   └─ ... (13 个提供商)
        │
        └─ auth.LoadStore()
            └─ 从密钥链加载认证信息
```

---

## ⚠️ 注意事项

### **1. 权限问题**

```bash
# 如果配置文件权限错误
picoclaw status
# Error loading config: permission denied

# 修复权限
chmod 644 ~/.picoclaw/config.json
```

### **2. 配置文件损坏**

```bash
# JSON 格式错误
picoclaw status
# Error loading config: invalid JSON

# 验证 JSON 格式
cat ~/.picoclaw/config.json | jq .
```

### **3. 多环境配置**

```bash
# 使用自定义配置路径
export PICOCLAW_CONFIG=/path/to/custom/config.json
picoclaw status
# Config: /path/to/custom/config.json ✓
```

### **4. 信息安全**

```bash
# status 命令不会显示完整的 API Key
# 只显示是否配置（✓ / not set）

# API Key 存储在配置文件中
# 确保配置文件权限安全：
chmod 600 ~/.picoclaw/config.json
```

---

## 🎓 总结

`NewStatusCommand` 是 PicoClaw 的**快速诊断工具**，它：

1. ✅ **零副作用**: 只读操作，不修改任何配置
2. ✅ **快速反馈**: 秒级完成，无网络请求
3. ✅ **全面检查**: 覆盖版本、配置、提供商、认证
4. ✅ **视觉清晰**: ✓/✗ 符号直观显示状态
5. ✅ **容错友好**: 配置缺失时优雅降级
6. ✅ **故障排查**: 快速定位配置问题

**推荐使用场景**：
```bash
# 1. 首次安装后验证
picoclaw onboard
picoclaw status  # 确认配置成功

# 2. 运行失败时排查
picoclaw agent -m "Hello"  # 失败
picoclaw status           # 查看哪里有问题

# 3. 定期健康检查
*/5 * * * * picoclaw status | grep -q "Config.*✓" || alert

# 4. 切换模型前确认
picoclaw status  # 查看当前模型
vim ~/.picoclaw/config.json  # 修改
picoclaw status  # 确认更改

# 5. CI/CD 管道验证
docker run picoclaw:latest status  # 验证镜像配置
```

**与其他命令对比**：

| 命令 | 功能 | 修改配置 | 网络请求 | 执行时间 |
|------|------|---------|---------|---------|
| `status` | 查看状态 | ❌ | ❌ | < 1s |
| `onboard` | 初始化 | ✅ | ❌ | < 1s |
| `auth status` | 认证状态 | ❌ | ❌ | < 1s |
| `agent` | 对话 | ❌ | ✅ | 5-30s |
| `gateway` | 启动服务 | ❌ | ✅ | 持续运行 |

---

## 📚 相关文件

- `cmd/picoclaw/internal/status/command.go` - 命令定义
- `cmd/picoclaw/internal/status/helpers.go` - 核心实现
- `cmd/picoclaw/internal/helpers.go` - 版本信息格式化
- `pkg/config/config.go` - 配置结构
- `pkg/auth/store.go` - 认证存储

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
status (验证配置) ← 故障排查入口
    ↓
auth login (认证)
    ↓
status (确认认证)
    ↓
agent / gateway (使用)
    ↓
status (定期检查)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
