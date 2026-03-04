# NewAgentCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/agent/command.go`
**行数**: 7-30

---

## 🎯 核心功能

`NewAgentCommand` 是一个工厂函数，使用 Cobra 框架创建 CLI 的 `agent` 子命令，用于与 AI 代理进行直接交互。

---

## 📊 结构拆解

### 1️⃣ **命令参数定义** (第 8-13 行)

```go
var (
    message    string  // 单次消息（非交互模式）
    sessionKey string  // 会话标识
    model      string  // 使用的模型
    debug      bool    // 调试日志开关
)
```

#### 参数说明：

| 参数 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `message` | string | "" | 单次对话模式的消息内容 |
| `sessionKey` | string | "cli:default" | 会话标识，用于隔离不同对话 |
| `model` | string | "" | 覆盖配置文件中的模型 |
| `debug` | bool | false | 启用详细的调试日志 |

---

### 2️⃣ **Cobra 命令构建** (第 15-22 行)

```go
cmd := &cobra.Command{
    Use:   "agent",                              // 命令名称
    Short: "Interact with the agent directly",   // 简短描述
    Args:  cobra.NoArgs,                         // 不接受位置参数
    RunE: func(cmd *cobra.Command, _ []string) error {
        return agentCmd(message, sessionKey, model, debug)
    },
}
```

#### Cobra 配置说明：

- **Use**: 定义命令名称为 `agent`
- **Short**: 在 `--help` 中显示的简短描述
- **Args**: `cobra.NoArgs` 表示不接受任何位置参数
- **RunE**: 命令执行函数，调用 `agentCmd` 进行实际处理

---

### 3️⃣ **标志位注册** (第 24-27 行)

```go
cmd.Flags().BoolVarP(&debug, "debug", "d", false, "Enable debug logging")
cmd.Flags().StringVarP(&message, "message", "m", "", "Send a single message (non-interactive mode)")
cmd.Flags().StringVarP(&sessionKey, "session", "s", "cli:default", "Session key")
cmd.Flags().StringVarP(&model, "model", "", "", "Model to use")
```

#### 标志位详解：

| 标志 | 短标志 | 类型 | 默认值 | 描述 |
|------|--------|------|--------|------|
| `--debug` | `-d` | bool | false | 启用调试日志输出 |
| `--message` | `-m` | string | "" | 单次消息内容 |
| `--session` | `-s` | string | "cli:default" | 会话键 |
| `--model` | 无 | string | "" | 模型名称 |

---

## 🔄 执行流程（agentCmd 函数）

**位置**: `cmd/picoclaw/internal/agent/helpers.go:21-77`

### **第一阶段：初始化**

```go
func agentCmd(message, sessionKey, model string, debug bool) error {
    // 1. 会话键处理 (22-24行)
    if sessionKey == "" {
        sessionKey = "cli:default"
    }

    // 2. 调试模式设置 (26-29行)
    if debug {
        logger.SetLevel(logger.DEBUG)
        fmt.Println("🔍 Debug mode enabled")
    }

    // 3. 加载配置 (31-34行)
    cfg, err := internal.LoadConfig()
    if err != nil {
        return fmt.Errorf("error loading config: %w", err)
    }

    // 4. 模型覆盖 (36-38行)
    if model != "" {
        cfg.Agents.Defaults.ModelName = model
    }

    // 5. 创建提供商 (40-48行)
    provider, modelID, err := providers.CreateProvider(cfg)
    if err != nil {
        return fmt.Errorf("error creating provider: %w", err)
    }

    // 使用解析后的模型 ID
    if modelID != "" {
        cfg.Agents.Defaults.ModelName = modelID
    }
```

---

### **第二阶段：Agent 创建**

```go
    // 6. 消息总线初始化 (50-51行)
    msgBus := bus.NewMessageBus()
    defer msgBus.Close()

    // 7. Agent 循环创建 (52行)
    agentLoop := agent.NewAgentLoop(cfg, msgBus, provider)

    // 8. 启动信息输出 (55-61行)
    startupInfo := agentLoop.GetStartupInfo()
    logger.InfoCF("agent", "Agent initialized",
        map[string]any{
            "tools_count":      startupInfo["tools"].(map[string]any)["count"],
            "skills_total":     startupInfo["skills"].(map[string]any)["total"],
            "skills_available": startupInfo["skills"].(map[string]any)["available"],
        })
```

---

### **第三阶段：模式选择**

```go
    // 9. 单次模式 (63-71行)
    if message != "" {
        ctx := context.Background()
        response, err := agentLoop.ProcessDirect(ctx, message, sessionKey)
        if err != nil {
            return fmt.Errorf("error processing message: %w", err)
        }
        fmt.Printf("\n%s %s\n", internal.Logo, response)
        return nil
    }

    // 10. 交互模式 (73-74行)
    fmt.Printf("%s Interactive mode (Ctrl+C to exit)\n\n", internal.Logo)
    interactiveMode(agentLoop, sessionKey)

    return nil
}
```

---

## 🎨 交互模式实现

**位置**: `cmd/picoclaw/internal/agent/helpers.go:79-162`

### **主交互模式** (`interactiveMode`)

```go
func interactiveMode(agentLoop *agent.AgentLoop, sessionKey string) {
    prompt := fmt.Sprintf("%s You: ", internal.Logo)

    // 使用 readline 库（支持历史记录）
    rl, err := readline.NewEx(&readline.Config{
        Prompt:          prompt,
        HistoryFile:     filepath.Join(os.TempDir(), ".picoclaw_history"),
        HistoryLimit:    100,
        InterruptPrompt: "^C",
        EOFPrompt:       "exit",
    })
    if err != nil {
        // 降级到简单输入模式
        fmt.Printf("Error initializing readline: %v\n", err)
        fmt.Println("Falling back to simple input mode...")
        simpleInteractiveMode(agentLoop, sessionKey)
        return
    }
    defer rl.Close()

    // 读取输入循环
    for {
        line, err := rl.Readline()
        if err != nil {
            if err == readline.ErrInterrupt || err == io.EOF {
                fmt.Println("\nGoodbye!")
                return
            }
            fmt.Printf("Error reading input: %v\n", err)
            continue
        }

        input := strings.TrimSpace(line)
        if input == "" {
            continue
        }

        if input == "exit" || input == "quit" {
            fmt.Println("Goodbye!")
            return
        }

        // 处理消息
        ctx := context.Background()
        response, err := agentLoop.ProcessDirect(ctx, input, sessionKey)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            continue
        }

        fmt.Printf("\n%s %s\n\n", internal.Logo, response)
    }
}
```

#### 特性：

- ✅ **命令历史**: 保存到 `/tmp/.picoclaw_history`
- ✅ **历史限制**: 最多 100 条
- ✅ **快捷键**: Ctrl+C 退出
- ✅ **命令**: `exit` 或 `quit` 退出
- ✅ **优雅降级**: readline 失败时自动切换到简单模式

---

### **降级模式** (`simpleInteractiveMode`)

```go
func simpleInteractiveMode(agentLoop *agent.AgentLoop, sessionKey string) {
    reader := bufio.NewReader(os.Stdin)
    for {
        fmt.Print(fmt.Sprintf("%s You: ", internal.Logo))
        line, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                fmt.Println("\nGoodbye!")
                return
            }
            fmt.Printf("Error reading input: %v\n", err)
            continue
        }

        input := strings.TrimSpace(line)
        if input == "" {
            continue
        }

        if input == "exit" || input == "quit" {
            fmt.Println("Goodbye!")
            return
        }

        ctx := context.Background()
        response, err := agentLoop.ProcessDirect(ctx, input, sessionKey)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            continue
        }

        fmt.Printf("\n%s %s\n\n", internal.Logo, response)
    }
}
```

#### 特性：

- ✅ **简单可靠**: 使用标准库 `bufio.Reader`
- ✅ **无历史记录**: 不保存命令历史
- ✅ **基本功能**: 支持输入输出和退出命令

---

## 📦 使用示例

### **1. 单次对话**
```bash
# 发送一条消息后退出
picoclaw agent -m "你好，请介绍一下自己"

# 输出:
# 🦞 你好！我是 PicoClaw，一个超轻量级的 AI 助手...
```

### **2. 交互模式**
```bash
# 启动交互式对话
picoclaw agent

# 输出:
# 🦞 Interactive mode (Ctrl+C to exit)
#
# 🦞 You: 当前目录有哪些文件？
# 🦞 当前目录包含以下文件：
#    1. main.go
#    2. README.md
#    ...
#
# 🦞 You: exit
# Goodbye!
```

### **3. 指定模型**
```bash
# 使用特定模型
picoclaw agent --model claude-sonnet-4-6 -m "写一首诗"
```

### **4. 指定会话**
```bash
# 使用自定义会话键（隔离对话历史）
picoclaw agent -s "project-alpha" -m "继续上次的讨论"

# 另一个独立会话
picoclaw agent -s "project-beta" -m "开始新项目规划"
```

### **5. 调试模式**
```bash
# 启用详细日志
picoclaw agent -d -m "测试命令"

# 输出:
# 🔍 Debug mode enabled
# [DEBUG] Loading config from ~/.picoclaw/config.json
# [DEBUG] Creating OpenAI provider
# [DEBUG] Agent initialized with 15 tools
# ...
```

---

## 🔑 关键设计要点

### **1. 模式切换**
- 通过 `-m` 参数区分单次 vs 交互模式
- 单次模式：适合脚本集成、CI/CD
- 交互模式：适合人机对话、调试

### **2. 会话隔离**
- 每个会话通过 `sessionKey` 标识
- 不同会话的历史记录完全独立
- 默认会话: `cli:default`

### **3. 模型灵活性**
- 支持命令行动态指定模型
- 命令行参数优先级高于配置文件
- 自动解析模型 ID（vendor/model 格式）

### **4. 优雅降级**
- readline 失败时自动切换到 bufio
- 确保在任何环境下都能工作
- 保持基本功能可用

### **5. 资源管理**
- `defer msgBus.Close()` 确保资源清理
- 交互模式中正确处理 Ctrl+C 信号
- 错误处理完善，不会意外崩溃

---

## 🔗 相关函数调用链

```
NewAgentCommand()
    └─ cmd.RunE → agentCmd()
        ├─ internal.LoadConfig()
        ├─ providers.CreateProvider()
        ├─ bus.NewMessageBus()
        ├─ agent.NewAgentLoop()
        │   ├─ NewAgentRegistry()
        │   ├─ registerSharedTools()
        │   └─ NewFallbackChain()
        │
        ├─ [单次模式]
        │   └─ agentLoop.ProcessDirect()
        │       └─ processMessage()
        │           └─ runAgentLoop()
        │               └─ runLLMIteration()
        │
        └─ [交互模式]
            └─ interactiveMode()
                └─ agentLoop.ProcessDirect()
                    └─ (同上)
```

---

## 📚 相关文件

- `cmd/picoclaw/internal/agent/command.go` - 命令定义
- `cmd/picoclaw/internal/agent/helpers.go` - 命令实现
- `pkg/agent/loop.go` - Agent 循环核心逻辑
- `pkg/bus/bus.go` - 消息总线
- `pkg/providers/factory.go` - Provider 创建工厂

---

## 🎓 总结

`NewAgentCommand` 是 PicoClaw CLI 的核心入口之一，它：

1. ✅ 使用 Cobra 框架构建清晰的 CLI 接口
2. ✅ 支持单次和交互两种模式
3. ✅ 提供灵活的配置覆盖机制
4. ✅ 实现优雅的错误处理和资源管理
5. ✅ 通过会话键实现对话历史隔离

这个命令体现了"简单易用、功能强大"的设计理念，是用户与 AI 代理交互的主要方式。

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
