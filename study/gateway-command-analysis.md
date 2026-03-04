# NewGatewayCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/gateway/command.go`
**行数**: 7-23
**实现文件**: `cmd/picoclaw/internal/gateway/helpers.go`

---

## 🎯 核心功能

`NewGatewayCommand` 是 PicoClaw 的**网关启动命令**，负责启动一个完整的多渠道 AI 助手服务。这是 PicoClaw 作为**常驻服务**运行的核心命令。

**主要职责**：
1. 启动 Agent 处理循环
2. 初始化多个聊天渠道（Telegram、Discord、WhatsApp 等）
3. 启动定时任务（Cron）服务
4. 启动心跳（Heartbeat）服务
5. 启动设备监控服务
6. 启动媒体文件管理
7. 提供健康检查端点

---

## 📊 结构拆解

### 1️⃣ **Cobra 命令定义** (command.go:7-23)

```go
func NewGatewayCommand() *cobra.Command {
    var debug bool

    cmd := &cobra.Command{
        Use:     "gateway",           // 命令名称
        Aliases: []string{"g"},       // 简短别名
        Short:   "Start picoclaw gateway",
        Args:    cobra.NoArgs,
        RunE: func(_ *cobra.Command, _ []string) error {
            return gatewayCmd(debug)
        },
    }

    cmd.Flags().BoolVarP(&debug, "debug", "d", false, "Enable debug logging")

    return cmd
}
```

#### 参数说明：

| 参数 | 短标志 | 类型 | 默认值 | 说明 |
|------|--------|------|--------|------|
| `--debug` | `-d` | bool | false | 启用调试日志 |

#### 特点：
- ✅ 提供简短别名 `g`
- ✅ 支持调试模式
- ✅ 无需其他参数，配置来自 config.json

---

## 🔄 执行流程详解

### 核心函数：`gatewayCmd()` (helpers.go:41-211)

```
┌─────────────────────────────────────────────────────────┐
│                  gatewayCmd() 启动流程                   │
└─────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│ 第一阶段：初始化核心组件                                  │
├──────────────────────────────────────────────────────────┤
│ 1. 设置调试模式 (可选)                                    │
│ 2. 加载配置文件 (config.json)                            │
│ 3. 创建 LLM Provider                                     │
│ 4. 创建消息总线 (MessageBus)                             │
│ 5. 创建 AgentLoop                                        │
│ 6. 打印启动信息（工具/技能数量）                          │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│ 第二阶段：初始化后台服务                                  │
├──────────────────────────────────────────────────────────┤
│ 7. 设置 Cron 服务 (定时任务)                             │
│ 8. 设置 Heartbeat 服务 (定期检查)                        │
│ 9. 创建 MediaStore (媒体文件管理)                        │
│10. 创建 ChannelManager (多渠道管理)                      │
│11. 创建 DeviceService (设备事件监控)                     │
│12. 创建 HealthServer (健康检查端点)                      │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│ 第三阶段：启动所有服务                                    │
├──────────────────────────────────────────────────────────┤
│13. cronService.Start()                                   │
│14. heartbeatService.Start()                              │
│15. deviceService.Start()                                 │
│16. channelManager.StartAll() - 启动所有渠道              │
│17. agentLoop.Run() - 后台 goroutine                      │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│ 第四阶段：运行与信号处理                                  │
├──────────────────────────────────────────────────────────┤
│18. 监听 Ctrl+C 信号                                       │
│19. 接收到信号后：优雅关闭                                 │
│    ├─ provider.Close()                                   │
│    ├─ cancel() - 取消 context                            │
│    ├─ msgBus.Close()                                     │
│    ├─ channelManager.StopAll()                           │
│    ├─ deviceService.Stop()                               │
│    ├─ heartbeatService.Stop()                            │
│    ├─ cronService.Stop()                                 │
│    ├─ mediaStore.Stop()                                  │
│    └─ agentLoop.Stop()                                   │
└──────────────────────────────────────────────────────────┘
```

---

## 📦 服务组件详解

### **1. Agent Loop** (第 63 行)

```go
agentLoop := agent.NewAgentLoop(cfg, msgBus, provider)
```

**功能**：
- 核心消息处理循环
- 从消息总线消费入站消息
- 调用 LLM 处理消息
- 执行工具调用
- 发布出站消息到渠道

**运行方式**：
```go
go agentLoop.Run(ctx)  // 后台 goroutine
```

---

### **2. Cron Service** (第 84-92 行)

```go
cronService := setupCronTool(
    agentLoop,
    msgBus,
    cfg.WorkspacePath(),
    cfg.Agents.Defaults.RestrictToWorkspace,
    execTimeout,
    cfg,
)
```

**功能**：
- 管理定时任务
- 支持 Cron 表达式（如：`0 9 * * *`）
- 任务持久化到 `workspace/cron/jobs.json`
- 执行超时保护

**Cron 工具注册**：
```go
agentLoop.RegisterTool(cronTool)
// LLM 可以调用 cron 工具来创建定时任务
```

**任务执行**：
```go
cronService.SetOnJob(func(job *cron.CronJob) (string, error) {
    result := cronTool.ExecuteJob(context.Background(), job)
    return result, nil
})
```

---

### **3. Heartbeat Service** (第 94-117 行)

```go
heartbeatService := heartbeat.NewHeartbeatService(
    cfg.WorkspacePath(),
    cfg.Heartbeat.Interval,  // 默认 30 分钟
    cfg.Heartbeat.Enabled,
)
```

**功能**：
- 定期读取 `workspace/HEARTBEAT.md`
- 执行定期检查任务
- 可用于健康监控、数据同步等

**处理器**：
```go
heartbeatService.SetHandler(func(prompt, channel, chatID string) *tools.ToolResult {
    // 使用 ProcessHeartbeat - 无会话历史，每次独立
    response, err := agentLoop.ProcessHeartbeat(ctx, prompt, channel, chatID)
    if response == "HEARTBEAT_OK" {
        return tools.SilentResult("Heartbeat OK")
    }
    return tools.SilentResult(response)
})
```

**特点**：
- ✅ 每次心跳**独立执行**，不累积上下文
- ✅ 静默模式，不发送响应给用户
- ✅ 适合后台任务（如：spawn 工具）

---

### **4. Media Store** (第 119-125 行)

```go
mediaStore := media.NewFileMediaStoreWithCleanup(media.MediaCleanerConfig{
    Enabled:  cfg.Tools.MediaCleanup.Enabled,
    MaxAge:   time.Duration(cfg.Tools.MediaCleanup.MaxAge) * time.Minute,
    Interval: time.Duration(cfg.Tools.MediaCleanup.Interval) * time.Minute,
})
mediaStore.Start()
```

**功能**：
- 管理临时媒体文件（图片、视频、音频）
- 自动清理过期文件（默认 TTL）
- 引用计数管理
- 支持 `media://` 协议

**典型流程**：
```
1. 渠道接收媒体 → Store 保存 → 返回 media://ref
2. Agent 处理消息时引用 media://ref
3. 工具生成媒体 → Store 保存 → 返回 media://ref
4. 渠道发送媒体时解析 media://ref
5. TTL 到期后自动删除文件
```

---

### **5. Channel Manager** (第 127-142 行)

```go
channelManager, err := channels.NewManager(cfg, msgBus, mediaStore)
```

**功能**：
- 管理多个聊天渠道
- 统一的消息入站/出站接口
- 共享 Webhook 服务器（默认端口 18790）

**支持的渠道**：
```go
import (
    _ "pkg/channels/telegram"        // Telegram Bot
    _ "pkg/channels/discord"         // Discord Bot
    _ "pkg/channels/whatsapp"        // WhatsApp (Web)
    _ "pkg/channels/whatsapp_native" // WhatsApp (Native)
    _ "pkg/channels/slack"           // Slack Bot
    _ "pkg/channels/qq"              // QQ Bot
    _ "pkg/channels/dingtalk"        // 钉钉机器人
    _ "pkg/channels/feishu"          // 飞书机器人
    _ "pkg/channels/wecom"           // 企业微信
    _ "pkg/channels/line"            // LINE Bot
    _ "pkg/channels/onebot"          // OneBot 协议
    _ "pkg/channels/pico"            // Pico 设备
    _ "pkg/channels/maixcam"         // MaixCam 设备
)
```

**启动渠道**：
```go
channelManager.StartAll(ctx)
// 只启动配置中 enabled: true 的渠道
```

---

### **6. Device Service** (第 160-170 行)

```go
deviceService := devices.NewService(devices.Config{
    Enabled:    cfg.Devices.Enabled,
    MonitorUSB: cfg.Devices.MonitorUSB,
}, stateManager)
```

**功能**：
- 监控 USB 设备插拔事件
- 通知 Agent 设备状态变化
- 用于硬件集成（如：树莓派、MaixCam）

**事件流**：
```
USB 设备插入 → DeviceService 检测 → 发送消息到 MessageBus
→ Agent 处理 → 执行相应操作（如：自动运行脚本）
```

---

### **7. Health Server** (第 172-182 行)

```go
healthServer := health.NewServer(cfg.Gateway.Host, cfg.Gateway.Port)
channelManager.SetupHTTPServer(addr, healthServer)
```

**功能**：
- 提供健康检查端点
- 共享 HTTP 服务器（与 Webhook 服务器合并）

**端点**：
```
GET /health  - 健康检查（200 OK）
GET /ready   - 就绪检查（所有服务启动后返回 200）
```

**用途**：
- Kubernetes liveness/readiness probes
- 负载均衡器健康检查
- 监控系统集成

---

## 📊 启动信息示例

```bash
picoclaw gateway

# 输出:
📦 Agent Status:
  • Tools: 15 loaded
  • Skills: 8/12 available
✓ Channels enabled: telegram, discord
✓ Gateway started on 127.0.0.1:18790
Press Ctrl+C to stop
✓ Cron service started
✓ Heartbeat service started
✓ Device event service started
✓ Health endpoints available at http://127.0.0.1:18790/health and /ready
```

---

## 📦 使用示例

### **1. 标准启动**
```bash
picoclaw gateway

# 网关启动，监听 Ctrl+C 停止
```

### **2. 调试模式**
```bash
picoclaw gateway --debug

# 输出:
# 🔍 Debug mode enabled
# [DEBUG] Loading config...
# [DEBUG] Creating provider...
# [DEBUG] Starting channels...
# ...
```

### **3. 使用别名**
```bash
picoclaw g    # 等同于 picoclaw gateway
```

### **4. 后台运行（systemd）**

创建服务文件 `/etc/systemd/system/picoclaw.service`：
```ini
[Unit]
Description=PicoClaw Gateway
After=network.target

[Service]
Type=simple
User=picoclaw
WorkingDirectory=/home/picoclaw
ExecStart=/usr/local/bin/picoclaw gateway
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

启动服务：
```bash
sudo systemctl enable picoclaw
sudo systemctl start picoclaw
sudo systemctl status picoclaw
```

### **5. Docker 运行**
```bash
# 使用项目提供的 Makefile
make docker-run

# 或手动运行
docker run -d \
  -v ~/.picoclaw:/root/.picoclaw \
  -p 18790:18790 \
  picoclaw:latest \
  gateway
```

### **6. 健康检查**
```bash
# 检查服务是否健康
curl http://localhost:18790/health
# 输出: {"status": "ok"}

# 检查服务是否就绪
curl http://localhost:18790/ready
# 输出: {"status": "ready"}
```

---

## 🛠️ 配置文件结构

### 网关相关配置 (`config.json`)

```json
{
  "gateway": {
    "host": "127.0.0.1",
    "port": 18790
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "bot_token": "123456:ABC-DEF..."
    },
    "discord": {
      "enabled": true,
      "bot_token": "MTk..."
    },
    "webhook_server": {
      "enabled": true,
      "host": "127.0.0.1",
      "port": 18790
    }
  },
  "heartbeat": {
    "enabled": true,
    "interval": 30
  },
  "devices": {
    "enabled": false,
    "monitor_usb": false
  },
  "tools": {
    "cron": {
      "exec_timeout_minutes": 10
    },
    "media_cleanup": {
      "enabled": true,
      "max_age": 60,
      "interval": 10
    }
  }
}
```

---

## 🔑 关键设计要点

### **1. 优雅关闭机制**

```go
// 监听 Ctrl+C 信号
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, os.Interrupt)
<-sigChan

// 按顺序关闭服务（15秒超时）
shutdownCtx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
defer cancel()

channelManager.StopAll(shutdownCtx)
deviceService.Stop()
heartbeatService.Stop()
cronService.Stop()
mediaStore.Stop()
agentLoop.Stop()
```

**关闭顺序**：
1. 停止接收新消息（关闭渠道）
2. 停止后台服务
3. 等待正在处理的消息完成
4. 清理资源

---

### **2. 消息总线架构**

```
┌─────────────┐
│   Channels  │ (Telegram, Discord, ...)
└──────┬──────┘
       │ PublishInbound()
       ↓
┌─────────────┐
│ MessageBus  │
└──────┬──────┘
       │ ConsumeInbound()
       ↓
┌─────────────┐
│ AgentLoop   │
└──────┬──────┘
       │ PublishOutbound()
       ↓
┌─────────────┐
│ MessageBus  │
└──────┬──────┘
       │ ConsumeOutbound()
       ↓
┌─────────────┐
│   Channels  │ (发送响应)
└─────────────┘
```

**特点**：
- ✅ 解耦渠道和 Agent
- ✅ 统一的消息格式
- ✅ 支持多对多通信

---

### **3. 上下文传播**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// 所有服务共享同一个 context
agentLoop.Run(ctx)
channelManager.StartAll(ctx)
deviceService.Start(ctx)

// Ctrl+C 触发 cancel()，所有服务优雅退出
```

---

### **4. 渠道自动注册**

通过 `import _` 实现渠道自动注册：

```go
import (
    _ "pkg/channels/telegram"  // init() 函数自动注册
)

// channels 包内部：
func init() {
    channels.Register("telegram", &TelegramChannel{})
}
```

---

## 🔗 相关函数调用链

```
NewGatewayCommand()
    └─ cmd.RunE → gatewayCmd(debug)
        ├─ internal.LoadConfig()
        ├─ providers.CreateProvider()
        ├─ bus.NewMessageBus()
        ├─ agent.NewAgentLoop()
        ├─ setupCronTool()
        │   ├─ cron.NewCronService()
        │   ├─ tools.NewCronTool()
        │   └─ agentLoop.RegisterTool()
        ├─ heartbeat.NewHeartbeatService()
        ├─ media.NewFileMediaStoreWithCleanup()
        ├─ channels.NewManager()
        ├─ devices.NewService()
        ├─ health.NewServer()
        ├─ [启动阶段]
        │   ├─ cronService.Start()
        │   ├─ heartbeatService.Start()
        │   ├─ deviceService.Start()
        │   ├─ channelManager.StartAll()
        │   └─ agentLoop.Run() (goroutine)
        └─ [关闭阶段]
            ├─ provider.Close()
            ├─ msgBus.Close()
            ├─ channelManager.StopAll()
            ├─ deviceService.Stop()
            ├─ heartbeatService.Stop()
            ├─ cronService.Stop()
            ├─ mediaStore.Stop()
            └─ agentLoop.Stop()
```

---

## ⚠️ 注意事项

### **1. 端口冲突**
```bash
# 如果 18790 端口已被占用
# Error: listen tcp 127.0.0.1:18790: bind: address already in use

# 解决方案：修改配置文件
{
  "gateway": {
    "port": 18791  // 改用其他端口
  }
}
```

### **2. 渠道凭证**
```bash
# 启动前确保渠道凭证已配置
# Telegram: bot_token
# Discord: bot_token
# WhatsApp: 扫码授权（首次）
```

### **3. 防火墙设置**
```bash
# 如果需要外部访问（Webhook）
sudo ufw allow 18790/tcp
```

### **4. 资源占用**
- CPU：空闲 < 1%，处理消息时 5-20%
- 内存：~10-50 MB（取决于渠道数量）
- 磁盘：主要是会话历史和媒体文件

### **5. 日志文件**
```bash
# 日志位置（如果配置）
~/.picoclaw/logs/gateway.log

# 查看实时日志
tail -f ~/.picoclaw/logs/gateway.log
```

---

## 🎓 总结

`NewGatewayCommand` 是 PicoClaw 的**服务化运行命令**，它：

1. ✅ **完整的服务栈**: Agent + 渠道 + 定时任务 + 心跳 + 设备监控
2. ✅ **多渠道支持**: 14+ 聊天平台集成
3. ✅ **优雅关闭**: 15秒超时，确保消息处理完成
4. ✅ **健康检查**: 标准 HTTP 端点，支持容器化部署
5. ✅ **自动清理**: 媒体文件 TTL 管理
6. ✅ **可扩展**: 通过 import _ 轻松添加新渠道

**推荐部署方式**：
```bash
# 开发环境：直接运行
picoclaw gateway

# 生产环境：systemd 服务
sudo systemctl start picoclaw

# 容器化：Docker
docker-compose up -d picoclaw

# 分布式：Kubernetes
kubectl apply -f picoclaw-deployment.yaml
```

**与 agent 命令的区别**：

| 特性 | `picoclaw agent` | `picoclaw gateway` |
|------|-----------------|-------------------|
| 运行模式 | 交互式 CLI | 常驻后台服务 |
| 渠道支持 | 仅 CLI | 多渠道（14+） |
| 定时任务 | ❌ | ✅ |
| 心跳服务 | ❌ | ✅ |
| 健康检查 | ❌ | ✅ |
| 设备监控 | ❌ | ✅ |
| 适用场景 | 本地测试、脚本调用 | 生产部署、机器人服务 |

---

## 📚 相关文件

- `cmd/picoclaw/internal/gateway/command.go` - 命令定义
- `cmd/picoclaw/internal/gateway/helpers.go` - 核心实现
- `pkg/channels/` - 渠道实现
- `pkg/bus/` - 消息总线
- `pkg/cron/` - 定时任务服务
- `pkg/heartbeat/` - 心跳服务
- `pkg/devices/` - 设备监控服务
- `pkg/media/` - 媒体文件管理
- `pkg/health/` - 健康检查服务器

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
auth login (认证)
    ↓
┌────────┴─────────┐
│                  │
agent           gateway
(本地测试)      (生产部署)
│                  │
└───── status ─────┘
    (查看状态)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
