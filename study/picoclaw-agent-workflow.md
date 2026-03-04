# 📖 PicoClaw Agent 完整工作流程

## 🎯 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI Entry Point                         │
│                    picoclaw agent [-m "msg"]                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      1. 初始化阶段                               │
├─────────────────────────────────────────────────────────────────┤
│ • 加载配置 (config.json)                                         │
│ • 创建 LLM Provider (OpenAI/Claude/etc)                          │
│ • 创建消息总线 (MessageBus)                                      │
│ • 创建 AgentLoop                                                 │
│   ├─ AgentRegistry (多代理支持)                                 │
│   ├─ 工具注册 (文件操作、网络搜索、MCP等)                        │
│   ├─ 会话管理器 (SQLite)                                         │
│   └─ 上下文构建器 (AGENTS.md, IDENTITY.md等)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
              ┌───────────────┴────────────────┐
              │                                │
┌─────────────▼──────────────┐  ┌─────────────▼─────────────┐
│   2A. 单次模式 (-m flag)    │  │   2B. 交互模式 (默认)      │
├────────────────────────────┤  ├───────────────────────────┤
│ • ProcessDirect(message)   │  │ • 使用 readline 交互       │
│ • 处理一次后退出            │  │ • 命令历史                 │
│                            │  │ • 持续对话循环             │
└────────────────────────────┘  └───────────────────────────┘
                │                               │
                └───────────────┬───────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                      3. 消息处理核心                             │
│                    processMessage(msg)                           │
├─────────────────────────────────────────────────────────────────┤
│ • 路由决策 (选择代理和会话)                                      │
│ • 命令处理 (/show, /list, /switch)                              │
│ • 更新工具上下文 (channel, chatID)                               │
│ • 加载会话历史 (session.GetHistory)                              │
│ • 构建上下文消息 (ContextBuilder.BuildMessages)                  │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    4. LLM 迭代循环                               │
│                  runLLMIteration(messages)                       │
├─────────────────────────────────────────────────────────────────┤
│  iteration = 0                                                  │
│  while iteration < MaxIterations (default: 20):                 │
│    ┌─────────────────────────────────────────────────┐         │
│    │ 4.1 调用 LLM (provider.Chat)                     │         │
│    │     • 发送: messages + tool_definitions           │         │
│    │     • 选项: max_tokens, temperature, model       │         │
│    │     • 支持: 回退链（fallback chain）             │         │
│    │     • 重试: 处理超时和上下文溢出                 │         │
│    └─────────────────────────────────────────────────┘         │
│                        ↓                                         │
│    ┌─────────────────────────────────────────────────┐         │
│    │ 4.2 检查响应                                     │         │
│    │  ├─ 无工具调用? → 返回最终响应                  │         │
│    │  └─ 有工具调用? → 继续下一步                    │         │
│    └─────────────────────────────────────────────────┘         │
│                        ↓                                         │
│    ┌─────────────────────────────────────────────────┐         │
│    │ 4.3 执行工具调用 (Tools.ExecuteWithContext)     │         │
│    │  • read_file     • write_file    • exec         │         │
│    │  • web_search    • message       • spawn        │         │
│    │  • mcp_tool      • 自定义技能                   │         │
│    └─────────────────────────────────────────────────┘         │
│                        ↓                                         │
│    ┌─────────────────────────────────────────────────┐         │
│    │ 4.4 将工具结果添加到消息历史                    │         │
│    │  messages.append({                               │         │
│    │    role: "tool",                                 │         │
│    │    content: tool_result,                         │         │
│    │    tool_call_id: id                              │         │
│    │  })                                              │         │
│    └─────────────────────────────────────────────────┘         │
│                        ↓                                         │
│    iteration++                                                   │
│    继续循环 ──────────┘                                         │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    5. 后处理与持久化                             │
├─────────────────────────────────────────────────────────────────┤
│ • 保存会话历史 (Sessions.AddMessage)                             │
│ • 触发记忆总结 (maybeSummarize)                                  │
│ • 上下文压缩 (forceCompression)                                  │
│ • 发布响应到消息总线 (bus.PublishOutbound)                       │
│ • 记录最后活跃渠道 (RecordLastChannel)                           │
└─────────────────────────────────────────────────────────────────┘
                                ↓
                         返回最终响应给用户
```

---

## 🔍 核心组件详解

### 1️⃣ **初始化阶段** (`agentCmd`)

**位置**: `cmd/picoclaw/internal/agent/helpers.go:21-77`

```go
// 执行流程
1. 设置会话键 (默认: "cli:default")
2. 启用调试模式 (可选)
3. 加载配置文件 (~/.picoclaw/config.json)
4. 创建 LLM Provider (支持 20+ 提供商)
5. 创建消息总线 (用于组件间通信)
6. 创建 AgentLoop
   └─ 这是核心！
```

### 2️⃣ **AgentLoop 构建** (`NewAgentLoop`)

**位置**: `pkg/agent/loop.go:63-92`

```go
func NewAgentLoop(cfg, msgBus, provider) *AgentLoop {
    // 1. 创建代理注册表
    registry := NewAgentRegistry(cfg, provider)

    // 2. 注册共享工具
    registerSharedTools(cfg, msgBus, registry, provider)
    //    - web_search (Brave/Tavily/DuckDuckGo)
    //    - web_fetch
    //    - message (发送消息到用户)
    //    - spawn (创建子代理)
    //    - find_skills / install_skill
    //    - I2C/SPI (硬件工具，仅 Linux)

    // 3. 设置回退链 (模型故障转移)
    fallbackChain := NewFallbackChain(cooldown)

    // 4. 创建状态管理器
    stateManager := state.NewManager(workspace)

    return &AgentLoop{...}
}
```

### 3️⃣ **AgentInstance 构建** (`NewAgentInstance`)

**位置**: `pkg/agent/instance.go:39-169`

每个代理实例包含：

```go
type AgentInstance struct {
    ID             string                    // 代理 ID (例如: "main")
    Workspace      string                    // 工作空间路径
    Model          string                    // 使用的模型
    MaxIterations  int                       // 最大工具调用迭代 (20)
    MaxTokens      int                       // 最大令牌数 (8192)
    Temperature    float64                   // 温度参数 (0.7)

    Provider       providers.LLMProvider     // LLM 提供商
    Sessions       *session.SessionManager   // 会话管理器 (SQLite)
    ContextBuilder *ContextBuilder           // 上下文构建器
    Tools          *tools.ToolRegistry       // 工具注册表

    // 注册的工具:
    // - read_file    : 读取文件
    // - write_file   : 写入文件
    // - list_dir     : 列出目录
    // - edit_file    : 编辑文件
    // - append_file  : 追加到文件
    // - exec         : 执行命令 (受沙盒限制)
}
```

---

## 🔄 消息处理流程

### 4️⃣ **ProcessDirect** → **processMessage** → **runAgentLoop**

**位置**: `pkg/agent/loop.go:385-663`

```go
// 入口函数
ProcessDirect(ctx, content, sessionKey) {
    msg := InboundMessage{
        Channel:    "cli",
        ChatID:     "direct",
        Content:    content,
        SessionKey: sessionKey,
    }
    return processMessage(ctx, msg)
}

// 路由和命令处理
processMessage(ctx, msg) {
    // 1. 检查系统消息
    if msg.Channel == "system" {
        return processSystemMessage(ctx, msg)
    }

    // 2. 处理命令 (/show, /list, /switch)
    if handled := handleCommand(ctx, msg) {
        return handled
    }

    // 3. 路由到正确的代理 (7 级优先级)
    route := registry.ResolveRoute(RouteInput{
        Channel:    msg.Channel,
        Peer:       extractPeer(msg),
        ParentPeer: extractParentPeer(msg),
        GuildID:    msg.Metadata["guild_id"],
        // ...
    })

    agent := registry.GetAgent(route.AgentID)

    // 4. 执行代理循环
    return runAgentLoop(ctx, agent, processOptions{
        SessionKey:    sessionKey,
        UserMessage:   msg.Content,
        EnableSummary: true,
    })
}

// 核心执行逻辑
runAgentLoop(ctx, agent, opts) {
    // 步骤 0: 记录最后活跃渠道
    RecordLastChannel(opts.Channel)

    // 步骤 1: 更新工具上下文
    updateToolContexts(agent, opts.Channel, opts.ChatID)

    // 步骤 2: 构建消息列表
    history := agent.Sessions.GetHistory(opts.SessionKey)
    summary := agent.Sessions.GetSummary(opts.SessionKey)
    messages := agent.ContextBuilder.BuildMessages(
        history, summary, opts.UserMessage, opts.Media,
        opts.Channel, opts.ChatID
    )

    // 步骤 3: 保存用户消息到会话
    agent.Sessions.AddMessage(opts.SessionKey, "user", opts.UserMessage)

    // 步骤 4: 运行 LLM 迭代循环 ⚡ (核心)
    finalContent, iteration := runLLMIteration(ctx, agent, messages, opts)

    // 步骤 5: 保存助手消息到会话
    agent.Sessions.AddMessage(opts.SessionKey, "assistant", finalContent)
    agent.Sessions.Save(opts.SessionKey)

    // 步骤 6: 触发记忆总结 (可选)
    if opts.EnableSummary {
        maybeSummarize(agent, opts.SessionKey)
    }

    return finalContent
}
```

---

## ⚡ LLM 迭代循环 (最核心部分)

### 5️⃣ **runLLMIteration** - 工具调用循环

**位置**: `pkg/agent/loop.go:722-1058`

```go
runLLMIteration(ctx, agent, messages, opts) {
    iteration := 0

    for iteration < agent.MaxIterations {  // 默认 20 次
        iteration++

        // ═══════════════════════════════════════════
        // 步骤 1: 构建工具定义
        // ═══════════════════════════════════════════
        providerToolDefs := agent.Tools.ToProviderDefs()
        // 转换为 provider 格式：
        // [{
        //   "type": "function",
        //   "function": {
        //     "name": "read_file",
        //     "description": "读取文件内容",
        //     "parameters": {...}
        //   }
        // }, ...]

        // ═══════════════════════════════════════════
        // 步骤 2: 调用 LLM (带回退支持)
        // ═══════════════════════════════════════════
        response, err := callLLM()  // 内部实现：
        // if 配置了多个候选模型 {
        //     fallbackChain.Execute(candidates, ...)
        //     // 例如: gpt-4 失败 → claude-sonnet → gemini-pro
        // } else {
        //     provider.Chat(messages, tools, model, options)
        // }

        // 重试逻辑 (最多 2 次):
        for retry := 0; retry <= 2; retry++ {
            response, err = callLLM()
            if err == nil { break }

            // 检测错误类型:
            if isTimeoutError {
                sleep(5s * retry)  // 指数退避
                continue
            }

            if isContextWindowError {
                // 强制压缩历史记录
                forceCompression(agent, sessionKey)
                messages = rebuildMessages()
                continue
            }
        }

        // ═══════════════════════════════════════════
        // 步骤 3: 处理推理内容 (Claude 特有)
        // ═══════════════════════════════════════════
        if response.Reasoning != "" {
            // 异步发送到推理渠道 (Discord/Slack 等)
            go handleReasoning(ctx, response.Reasoning)
        }

        // ═══════════════════════════════════════════
        // 步骤 4: 检查是否有工具调用
        // ═══════════════════════════════════════════
        if len(response.ToolCalls) == 0 {
            // 没有工具调用 → 这是最终答案
            finalContent = response.Content
            break  // 退出循环
        }

        // ═══════════════════════════════════════════
        // 步骤 5: 构建助手消息 (带工具调用)
        // ═══════════════════════════════════════════
        assistantMsg := Message{
            Role:      "assistant",
            Content:   response.Content,
            ToolCalls: normalizedToolCalls,
        }
        messages.append(assistantMsg)
        agent.Sessions.AddFullMessage(sessionKey, assistantMsg)

        // ═══════════════════════════════════════════
        // 步骤 6: 执行每个工具调用
        // ═══════════════════════════════════════════
        for _, toolCall := range response.ToolCalls {
            // 示例: toolCall = {
            //   Name: "read_file",
            //   Arguments: {"path": "/path/to/file.txt"}
            // }

            toolResult := agent.Tools.ExecuteWithContext(
                ctx,
                toolCall.Name,
                toolCall.Arguments,
                opts.Channel,
                opts.ChatID,
                asyncCallback,  // 用于异步工具 (spawn)
            )

            // 工具结果可能包含:
            // - ForLLM:   发送给 LLM 的内容
            // - ForUser:  直接显示给用户的内容
            // - Silent:   是否静默 (不发送 ForUser)
            // - Media:    媒体引用 (图片、视频等)
            // - Err:      错误信息

            // 发送 ForUser 到用户 (如果不是 Silent)
            if !toolResult.Silent && toolResult.ForUser != "" {
                bus.PublishOutbound(toolResult.ForUser)
            }

            // 发送媒体 (如果有)
            if len(toolResult.Media) > 0 {
                bus.PublishOutboundMedia(toolResult.Media)
            }

            // 构建工具结果消息
            contentForLLM := toolResult.ForLLM
            if contentForLLM == "" && toolResult.Err != nil {
                contentForLLM = toolResult.Err.Error()
            }

            toolResultMsg := Message{
                Role:       "tool",
                Content:    contentForLLM,
                ToolCallID: toolCall.ID,
            }
            messages.append(toolResultMsg)
            agent.Sessions.AddFullMessage(sessionKey, toolResultMsg)
        }

        // ═══════════════════════════════════════════
        // 步骤 7: 继续下一次迭代
        // ═══════════════════════════════════════════
        // 现在 messages 包含:
        // 1. 系统提示
        // 2. 历史对话
        // 3. 当前用户消息
        // 4. 助手消息 (带工具调用)
        // 5. 工具结果消息
        //
        // 循环回到步骤 1，再次调用 LLM
    }

    return finalContent, iteration
}
```

---

## 📊 消息流转示例

让我们通过一个实际例子理解整个流程：

### 示例：用户问 "当前目录有哪些文件？"

```
┌────────────────────────────────────────────────────────┐
│ Iteration 1                                            │
├────────────────────────────────────────────────────────┤
│ LLM 输入:                                               │
│  - 系统提示: "You are a helpful AI assistant..."       │
│  - 用户消息: "当前目录有哪些文件？"                     │
│  - 可用工具: [list_dir, read_file, write_file, ...]   │
│                                                        │
│ LLM 输出:                                               │
│  Content: ""                                           │
│  ToolCalls: [{                                         │
│    Name: "list_dir",                                   │
│    Arguments: {"path": "."}                            │
│  }]                                                    │
│                                                        │
│ 工具执行:                                               │
│  list_dir(".") → "file1.txt\nfile2.py\nREADME.md"     │
│                                                        │
│ 添加到消息历史:                                         │
│  - 助手消息 (带工具调用)                                │
│  - 工具结果消息 (role: "tool")                          │
└────────────────────────────────────────────────────────┘
                         ↓
┌────────────────────────────────────────────────────────┐
│ Iteration 2                                            │
├────────────────────────────────────────────────────────┤
│ LLM 输入:                                               │
│  - 系统提示                                             │
│  - 用户消息: "当前目录有哪些文件？"                     │
│  - 助手消息: [工具调用 list_dir]                        │
│  - 工具结果: "file1.txt\nfile2.py\nREADME.md"          │
│                                                        │
│ LLM 输出:                                               │
│  Content: "当前目录包含以下文件:                        │
│            1. file1.txt                                │
│            2. file2.py                                 │
│            3. README.md"                               │
│  ToolCalls: []  ← 无工具调用，循环结束                  │
│                                                        │
│ 返回最终响应给用户 ✓                                    │
└────────────────────────────────────────────────────────┘
```

---

## 🛡️ 关键特性

### **1. 工作空间沙盒**
```go
// 所有文件操作受限于工作空间目录
Workspace: ~/.picoclaw/workspace/

// 可以配置例外路径
allow_read_paths:  ["/tmp/*", "/var/log/*"]
allow_write_paths: ["/tmp/output/*"]
```

### **2. 会话管理** (`pkg/session/`)
```go
// SQLite 存储
~/.picoclaw/workspace/sessions/sessions.db

// 表结构:
// - session_messages: 消息历史
// - session_metadata: 会话摘要、令牌窗口
```

### **3. 上下文构建** (`pkg/agent/context.go`)
```go
ContextBuilder.BuildMessages() {
    // 从工作空间文件构建系统提示:
    1. AGENTS.md    - 代理行为指南
    2. IDENTITY.md  - 代理身份
    3. SOUL.md      - 代理灵魂
    4. TOOLS.md     - 工具描述
    5. USER.md      - 用户偏好
    6. skills/*.md  - 技能定义

    // 构建消息数组:
    [
        {role: "system", content: systemPrompt + summary},
        ...history,  // 历史对话
        {role: "user", content: userMessage}
    ]
}
```

### **4. 记忆总结**
```go
maybeSummarize() {
    threshold := contextWindow * 75%

    if tokenEstimate > threshold {
        // 异步总结历史对话
        go summarizeSession(agent, sessionKey)

        // 总结策略:
        // 1. 保留最后 4 条消息
        // 2. 总结之前的对话
        // 3. 将摘要存储到 session_metadata
    }
}
```

### **5. 强制压缩** (上下文溢出时)
```go
forceCompression() {
    // 1. 保留系统提示
    // 2. 丢弃最老的 50% 消息
    // 3. 保留最后一条用户消息
    // 4. 添加压缩通知到系统提示
}
```

---

## 📦 工具生态

### **内置工具** (`pkg/tools/`)
| 工具 | 功能 | 沙盒限制 |
|------|------|---------|
| `read_file` | 读取文件 | ✓ 工作空间 |
| `write_file` | 写入文件 | ✓ 工作空间 |
| `edit_file` | 编辑文件 | ✓ 工作空间 |
| `append_file` | 追加内容 | ✓ 工作空间 |
| `list_dir` | 列出目录 | ✓ 工作空间 |
| `exec` | 执行命令 | ✓ 工作空间 + 危险命令阻止 |
| `web_search` | 网络搜索 | ✗ 无限制 |
| `web_fetch` | 获取网页 | ✗ 无限制 |
| `message` | 发送消息 | ✗ 无限制 |
| `spawn` | 创建子代理 | ✓ 代理白名单 |
| `mcp_tool` | MCP 协议工具 | 取决于 MCP 服务器 |

### **危险命令阻止** (`exec` 工具)
```go
BlockedPatterns := [
    "rm -rf",
    "format",
    "dd if=",
    ":(){ :|:& };:",  // fork 炸弹
    "> /dev/sda",
    // ...
]
```

---

## 🎓 总结

**PicoClaw Agent 的核心工作原理**：

1. **初始化**: 加载配置 → 创建 Provider → 构建 AgentLoop
2. **消息路由**: 解析消息 → 选择代理 → 加载会话历史
3. **LLM 循环**:
   - 调用 LLM（带工具定义）
   - 检查工具调用
   - 执行工具
   - 将结果添加到消息历史
   - 重复，直到 LLM 返回最终答案
4. **持久化**: 保存会话 → 触发总结 → 返回响应

**关键设计理念**：
- ✅ 最小内存占用（工作空间沙盒）
- ✅ 会话隔离（SQLite 持久化）
- ✅ 工具安全（白名单 + 危险命令阻止）
- ✅ 智能记忆管理（自动总结 + 强制压缩）
- ✅ 高可用性（回退链 + 重试机制）

---

## 📚 相关文件索引

### 命令层
- `cmd/picoclaw/internal/agent/command.go` - NewAgentCommand 定义
- `cmd/picoclaw/internal/agent/helpers.go` - agentCmd 实现，交互模式

### 核心逻辑
- `pkg/agent/loop.go` - AgentLoop 主循环，消息处理，LLM 迭代
- `pkg/agent/instance.go` - AgentInstance 定义和初始化
- `pkg/agent/context.go` - ContextBuilder 上下文构建
- `pkg/agent/registry.go` - AgentRegistry 多代理管理

### 基础设施
- `pkg/session/` - 会话管理（SQLite）
- `pkg/tools/` - 工具注册表和各种工具实现
- `pkg/providers/` - LLM 提供商抽象和实现
- `pkg/bus/` - 消息总线（组件间通信）
- `pkg/routing/` - 代理路由逻辑

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
