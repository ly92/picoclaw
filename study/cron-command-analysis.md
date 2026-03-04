# NewCronCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/cron/command.go`
**行数**: 12-44
**实现文件**: `cmd/picoclaw/internal/cron/helpers.go`
**子命令文件**: `list.go`, `add.go`, `remove.go`, `enable.go`, `disable.go`

---

## 🎯 核心功能

`NewCronCommand` 是 PicoClaw 的**定时任务管理命令**，提供完整的 Cron 任务生命周期管理。这是一个**父命令**，包含 5 个子命令。

**主要职责**：
1. 添加定时任务（按时间间隔或 Cron 表达式）
2. 列出所有定时任务
3. 删除指定任务
4. 启用/禁用任务
5. 任务持久化到 JSON 文件

---

## 📊 结构拆解

### 1️⃣ **父命令定义** (command.go:12-44)

```go
func NewCronCommand() *cobra.Command {
    var storePath string

    cmd := &cobra.Command{
        Use:     "cron",              // 命令名称
        Aliases: []string{"c"},       // 简短别名
        Short:   "Manage scheduled tasks",
        Args:    cobra.NoArgs,
        RunE: func(cmd *cobra.Command, _ []string) error {
            return cmd.Help()  // 不带子命令时显示帮助
        },
        // 在所有子命令执行前运行
        PersistentPreRunE: func(_ *cobra.Command, _ []string) error {
            cfg, err := internal.LoadConfig()
            if err != nil {
                return fmt.Errorf("error loading config: %w", err)
            }
            // 动态解析存储路径
            storePath = filepath.Join(cfg.WorkspacePath(), "cron", "jobs.json")
            return nil
        },
    }

    // 注册子命令（通过闭包传递 storePath）
    cmd.AddCommand(
        newListCommand(func() string { return storePath }),
        newAddCommand(func() string { return storePath }),
        newRemoveCommand(func() string { return storePath }),
        newEnableCommand(func() string { return storePath }),
        newDisableCommand(func() string { return storePath }),
    )

    return cmd
}
```

#### 特点：
- ✅ **动态路径解析**: 每次执行时读取配置，支持 `PICOCLAW_HOME` 环境变量
- ✅ **闭包传递路径**: 子命令通过闭包访问共享的 `storePath`
- ✅ **持久化存储**: 任务保存在 `~/.picoclaw/workspace/cron/jobs.json`

---

### 2️⃣ **子命令架构**

```
picoclaw cron
    ├─ list               # 列出所有任务
    ├─ add                # 添加新任务
    ├─ remove <id>        # 删除任务
    ├─ enable <id>        # 启用任务
    └─ disable <id>       # 禁用任务
```

---

## 🔐 子命令详解

### **子命令 1: `list`** (list.go:5-17)

```go
func newListCommand(storePath func() string) *cobra.Command {
    cmd := &cobra.Command{
        Use:   "list",
        Short: "List all scheduled jobs",
        Args:  cobra.NoArgs,
        RunE: func(_ *cobra.Command, _ []string) error {
            cronListCmd(storePath())
            return nil
        },
    }
    return cmd
}
```

#### 执行逻辑 (helpers.go:10-47)：

```go
func cronListCmd(storePath string) {
    cs := cron.NewCronService(storePath, nil)
    jobs := cs.ListJobs(true)  // true = 显示包括禁用的任务

    if len(jobs) == 0 {
        fmt.Println("No scheduled jobs.")
        return
    }

    fmt.Println("\nScheduled Jobs:")
    fmt.Println("----------------")
    for _, job := range jobs {
        // 格式化调度类型
        var schedule string
        if job.Schedule.Kind == "every" {
            schedule = fmt.Sprintf("every %ds", *job.Schedule.EveryMS/1000)
        } else if job.Schedule.Kind == "cron" {
            schedule = job.Schedule.Expr
        } else {
            schedule = "one-time"
        }

        // 格式化下次运行时间
        nextRun := "scheduled"
        if job.State.NextRunAtMS != nil {
            nextTime := time.UnixMilli(*job.State.NextRunAtMS)
            nextRun = nextTime.Format("2006-01-02 15:04")
        }

        // 显示状态
        status := "enabled"
        if !job.Enabled {
            status = "disabled"
        }

        fmt.Printf("  %s (%s)\n", job.Name, job.ID)
        fmt.Printf("    Schedule: %s\n", schedule)
        fmt.Printf("    Status: %s\n", status)
        fmt.Printf("    Next run: %s\n", nextRun)
    }
}
```

---

### **子命令 2: `add`** (add.go:11-64)

```go
func newAddCommand(storePath func() string) *cobra.Command {
    var (
        name    string
        message string
        every   int64
        cronExp string
        deliver bool
        channel string
        to      string
    )

    cmd := &cobra.Command{
        Use:   "add",
        Short: "Add a new scheduled job",
        Args:  cobra.NoArgs,
        RunE: func(cmd *cobra.Command, _ []string) error {
            // 验证：必须指定 --every 或 --cron
            if every <= 0 && cronExp == "" {
                return fmt.Errorf("either --every or --cron must be specified")
            }

            // 构建调度配置
            var schedule cron.CronSchedule
            if every > 0 {
                everyMS := every * 1000
                schedule = cron.CronSchedule{Kind: "every", EveryMS: &everyMS}
            } else {
                schedule = cron.CronSchedule{Kind: "cron", Expr: cronExp}
            }

            // 添加任务
            cs := cron.NewCronService(storePath(), nil)
            job, err := cs.AddJob(name, schedule, message, deliver, channel, to)
            if err != nil {
                return fmt.Errorf("error adding job: %w", err)
            }

            fmt.Printf("✓ Added job '%s' (%s)\n", job.Name, job.ID)
            return nil
        },
    }

    // 注册标志
    cmd.Flags().StringVarP(&name, "name", "n", "", "Job name")
    cmd.Flags().StringVarP(&message, "message", "m", "", "Message for agent")
    cmd.Flags().Int64VarP(&every, "every", "e", 0, "Run every N seconds")
    cmd.Flags().StringVarP(&cronExp, "cron", "c", "", "Cron expression")
    cmd.Flags().BoolVarP(&deliver, "deliver", "d", false, "Deliver response to channel")
    cmd.Flags().StringVar(&to, "to", "", "Recipient for delivery")
    cmd.Flags().StringVar(&channel, "channel", "", "Channel for delivery")

    // 必需标志
    _ = cmd.MarkFlagRequired("name")
    _ = cmd.MarkFlagRequired("message")

    // 互斥标志
    cmd.MarkFlagsMutuallyExclusive("every", "cron")

    return cmd
}
```

#### 参数说明：

| 参数 | 短标志 | 类型 | 必需 | 说明 |
|------|--------|------|------|------|
| `--name` | `-n` | string | ✅ | 任务名称 |
| `--message` | `-m` | string | ✅ | 发送给 Agent 的消息 |
| `--every` | `-e` | int64 | ❌ | 每隔 N 秒运行（与 --cron 互斥） |
| `--cron` | `-c` | string | ❌ | Cron 表达式（与 --every 互斥） |
| `--deliver` | `-d` | bool | ❌ | 是否发送响应到渠道 |
| `--channel` | 无 | string | ❌ | 目标渠道（需配合 --deliver） |
| `--to` | 无 | string | ❌ | 接收者（需配合 --deliver） |

---

### **子命令 3: `remove`** (remove.go:5-18)

```go
func newRemoveCommand(storePath func() string) *cobra.Command {
    cmd := &cobra.Command{
        Use:     "remove",
        Short:   "Remove a job by ID",
        Args:    cobra.ExactArgs(1),  // 必须提供 1 个参数（任务 ID）
        Example: `picoclaw cron remove 1`,
        RunE: func(_ *cobra.Command, args []string) error {
            cronRemoveCmd(storePath(), args[0])
            return nil
        },
    }
    return cmd
}
```

#### 执行逻辑 (helpers.go:49-56)：

```go
func cronRemoveCmd(storePath, jobID string) {
    cs := cron.NewCronService(storePath, nil)
    if cs.RemoveJob(jobID) {
        fmt.Printf("✓ Removed job %s\n", jobID)
    } else {
        fmt.Printf("✗ Job %s not found\n", jobID)
    }
}
```

---

### **子命令 4: `enable`** (enable.go:5-16)

```go
func newEnableCommand(storePath func() string) *cobra.Command {
    return &cobra.Command{
        Use:     "enable",
        Short:   "Enable a job",
        Args:    cobra.ExactArgs(1),
        Example: `picoclaw cron enable 1`,
        RunE: func(_ *cobra.Command, args []string) error {
            cronSetJobEnabled(storePath(), args[0], true)
            return nil
        },
    }
}
```

---

### **子命令 5: `disable`** (disable.go:5-16)

```go
func newDisableCommand(storePath func() string) *cobra.Command {
    return &cobra.Command{
        Use:     "disable",
        Short:   "Disable a job",
        Args:    cobra.ExactArgs(1),
        Example: `picoclaw cron disable 1`,
        RunE: func(_ *cobra.Command, args []string) error {
            cronSetJobEnabled(storePath(), args[0], false)
            return nil
        },
    }
}
```

#### 执行逻辑 (helpers.go:58-66)：

```go
func cronSetJobEnabled(storePath, jobID string, enabled bool) {
    cs := cron.NewCronService(storePath, nil)
    job := cs.EnableJob(jobID, enabled)
    if job != nil {
        fmt.Printf("✓ Job '%s' enabled\n", job.Name)
    } else {
        fmt.Printf("✗ Job %s not found\n", jobID)
    }
}
```

---

## 📦 使用示例

### **1. 列出所有任务**
```bash
picoclaw cron list

# 输出:
# Scheduled Jobs:
# ----------------
#   morning-report (1)
#     Schedule: every 60s
#     Status: enabled
#     Next run: 2026-03-04 09:00
#
#   daily-summary (2)
#     Schedule: 0 18 * * *
#     Status: enabled
#     Next run: 2026-03-04 18:00
#
#   backup-check (3)
#     Schedule: 0 2 * * 0
#     Status: disabled
#     Next run: 2026-03-08 02:00
```

### **2. 添加按时间间隔的任务**
```bash
# 每 60 秒检查一次系统状态
picoclaw cron add \
  --name "status-check" \
  --message "检查系统状态并报告" \
  --every 60

# 输出:
# ✓ Added job 'status-check' (4)
```

### **3. 添加 Cron 表达式任务**
```bash
# 每天早上 9 点发送早报
picoclaw cron add \
  --name "morning-report" \
  --message "生成今日工作摘要" \
  --cron "0 9 * * *"

# 输出:
# ✓ Added job 'morning-report' (1)
```

### **4. 添加带通知的任务**
```bash
# 每天下午 6 点发送摘要到 Telegram
picoclaw cron add \
  --name "daily-summary" \
  --message "总结今天的重要事项" \
  --cron "0 18 * * *" \
  --deliver \
  --channel telegram \
  --to "@username"

# 输出:
# ✓ Added job 'daily-summary' (2)
```

### **5. 删除任务**
```bash
picoclaw cron remove 2

# 输出:
# ✓ Removed job 2
```

### **6. 禁用任务（保留但不执行）**
```bash
picoclaw cron disable 3

# 输出:
# ✓ Job 'backup-check' disabled
```

### **7. 启用任务**
```bash
picoclaw cron enable 3

# 输出:
# ✓ Job 'backup-check' enabled
```

### **8. 使用别名**
```bash
picoclaw c list    # 等同于 picoclaw cron list
```

---

## 🕐 Cron 表达式语法

PicoClaw 支持标准的 5 字段 Cron 表达式：

```
┌─────────── 分钟 (0 - 59)
│ ┌───────── 小时 (0 - 23)
│ │ ┌─────── 日期 (1 - 31)
│ │ │ ┌───── 月份 (1 - 12)
│ │ │ │ ┌─── 星期 (0 - 6, 0=周日)
│ │ │ │ │
* * * * *
```

### **常用示例**：

| 表达式 | 说明 |
|--------|------|
| `0 9 * * *` | 每天上午 9:00 |
| `0 18 * * *` | 每天下午 6:00 |
| `0 */6 * * *` | 每 6 小时 |
| `0 9-17 * * 1-5` | 工作日 9:00-17:00 每小时 |
| `0 2 * * 0` | 每周日凌晨 2:00 |
| `0 0 1 * *` | 每月 1 号凌晨 |
| `*/15 * * * *` | 每 15 分钟 |
| `0 0 * * 1` | 每周一凌晨 |

---

## 🗂️ 任务存储格式

### 存储位置：
```
~/.picoclaw/workspace/cron/jobs.json
```

### JSON 结构：

```json
{
  "jobs": [
    {
      "id": "1",
      "name": "morning-report",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * *"
      },
      "message": "生成今日工作摘要",
      "deliver": false,
      "channel": "",
      "to": "",
      "state": {
        "next_run_at_ms": 1709539200000,
        "last_run_at_ms": 1709452800000,
        "last_result": "success"
      },
      "created_at": "2026-03-01T10:00:00Z"
    },
    {
      "id": "2",
      "name": "status-check",
      "enabled": true,
      "schedule": {
        "kind": "every",
        "every_ms": 60000
      },
      "message": "检查系统状态",
      "deliver": true,
      "channel": "telegram",
      "to": "@admin",
      "state": {
        "next_run_at_ms": 1709539260000
      },
      "created_at": "2026-03-01T10:05:00Z"
    }
  ]
}
```

---

## 🔄 任务执行流程

```
┌─────────────────────────────────────────────┐
│     CronService (gateway 启动时启动)         │
└─────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 1. 加载 jobs.json      │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 2. 为每个任务计算       │
        │    下次运行时间         │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 3. 等待最近的任务       │
        │    到达运行时间         │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 4. 执行任务            │
        │    • 调用 CronTool     │
        │    • 传递 message      │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 5. Agent 处理消息      │
        │    • LLM 推理          │
        │    • 工具调用          │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 6. 处理响应            │
        │    • 如果 deliver=true │
        │      发送到指定渠道    │
        │    • 否则静默完成      │
        └───────────────────────┘
                    ↓
        ┌───────────────────────┐
        │ 7. 更新任务状态        │
        │    • 计算下次运行时间  │
        │    • 保存到 jobs.json  │
        └───────────────────────┘
                    ↓
        回到步骤 3（持续循环）
```

---

## 🔑 关键设计要点

### **1. 两种调度模式**

#### **模式 A：按时间间隔 (`--every`)**
```bash
picoclaw cron add --name "check" --message "..." --every 300
# 每 300 秒（5 分钟）执行一次
```

**特点**：
- ✅ 简单直观
- ✅ 适合固定间隔的任务
- ✅ 不受系统时间影响

---

#### **模式 B：Cron 表达式 (`--cron`)**
```bash
picoclaw cron add --name "report" --message "..." --cron "0 9 * * *"
# 每天上午 9:00 执行
```

**特点**：
- ✅ 灵活强大
- ✅ 适合特定时间点的任务
- ✅ 遵循标准 Cron 语法

---

### **2. 消息传递机制**

```go
// 任务定义
{
  "message": "生成报告",
  "deliver": true,
  "channel": "telegram",
  "to": "@admin"
}
```

**执行时**：
1. CronService 触发任务
2. 消息发送到 Agent: `"生成报告"`
3. Agent 调用 LLM 处理
4. 如果 `deliver=true`，响应发送到 Telegram `@admin`
5. 如果 `deliver=false`，响应被丢弃（静默执行）

---

### **3. 动态路径解析**

```go
PersistentPreRunE: func(_ *cobra.Command, _ []string) error {
    cfg, err := internal.LoadConfig()
    if err != nil {
        return fmt.Errorf("error loading config: %w", err)
    }
    storePath = filepath.Join(cfg.WorkspacePath(), "cron", "jobs.json")
    return nil
}
```

**好处**：
- ✅ 支持 `PICOCLAW_HOME` 环境变量
- ✅ 多实例部署时路径独立
- ✅ 每次执行时动态计算

---

### **4. 任务状态管理**

```json
"state": {
  "next_run_at_ms": 1709539200000,    // 下次运行时间
  "last_run_at_ms": 1709452800000,    // 上次运行时间
  "last_result": "success"             // 上次执行结果
}
```

**状态追踪**：
- 下次运行时间：用于调度
- 上次运行时间：用于日志和审计
- 执行结果：用于监控和报警

---

### **5. 启用/禁用机制**

```bash
# 禁用任务（保留配置）
picoclaw cron disable 1

# 任务不会被执行，但保留在 jobs.json 中
# 可以随时重新启用

# 启用任务
picoclaw cron enable 1
```

**用途**：
- 临时关闭任务而不删除
- A/B 测试不同的定时策略
- 维护期间暂停任务

---

## 🔗 相关函数调用链

```
NewCronCommand()
    ├─ PersistentPreRunE
    │   └─ 解析 storePath
    │
    ├─ newListCommand()
    │   └─ cronListCmd()
    │       └─ cron.NewCronService()
    │           └─ ListJobs()
    │
    ├─ newAddCommand()
    │   └─ cron.NewCronService()
    │       └─ AddJob()
    │           ├─ 验证参数
    │           ├─ 生成任务 ID
    │           ├─ 计算下次运行时间
    │           └─ 保存到 jobs.json
    │
    ├─ newRemoveCommand()
    │   └─ cronRemoveCmd()
    │       └─ cron.NewCronService()
    │           └─ RemoveJob()
    │
    ├─ newEnableCommand()
    │   └─ cronSetJobEnabled(..., true)
    │       └─ cron.NewCronService()
    │           └─ EnableJob()
    │
    └─ newDisableCommand()
        └─ cronSetJobEnabled(..., false)
            └─ cron.NewCronService()
                └─ EnableJob()
```

---

## ⚠️ 注意事项

### **1. Gateway 必须运行**
```bash
# Cron 任务只在 gateway 运行时执行
picoclaw gateway

# 如果 gateway 停止，所有定时任务停止
# 重启 gateway 后，任务会恢复执行
```

### **2. 时区问题**
```bash
# Cron 表达式使用服务器本地时区
# 如果需要 UTC 时间，请调整表达式

# 例如：服务器在 UTC+8，想要 UTC 9:00 执行
# 本地时间 = UTC + 8 = 17:00
picoclaw cron add --cron "0 17 * * *" ...
```

### **3. 任务重叠**
```bash
# 如果任务执行时间超过调度间隔
# 新的任务会等待当前任务完成

# 例如：--every 60，但任务执行需要 90 秒
# 第二次执行会在第一次完成后立即开始
```

### **4. 消息长度限制**
```bash
# --message 参数长度有限制
# 对于长消息，考虑使用文件引用

picoclaw cron add \
  --name "report" \
  --message "读取 /path/to/instructions.txt 并执行" \
  --cron "0 9 * * *"
```

### **5. 错误处理**
```bash
# 如果任务执行失败，不会自动重试
# 检查日志：
tail -f ~/.picoclaw/logs/gateway.log | grep cron
```

---

## 🎓 总结

`NewCronCommand` 是 PicoClaw 的**定时任务管理中心**，它：

1. ✅ **双模式调度**: 时间间隔 + Cron 表达式
2. ✅ **完整生命周期**: 添加 → 列出 → 禁用 → 删除
3. ✅ **持久化存储**: JSON 文件，易于备份和迁移
4. ✅ **渠道集成**: 支持将结果发送到聊天渠道
5. ✅ **状态追踪**: 记录运行时间和执行结果
6. ✅ **动态配置**: 支持环境变量和自定义路径

**推荐使用场景**：
```bash
# 1. 定期报告
picoclaw cron add --name "weekly-report" \
  --message "总结本周工作" \
  --cron "0 9 * * 1" \
  --deliver --channel telegram --to "@team"

# 2. 健康检查
picoclaw cron add --name "health-check" \
  --message "检查系统状态" \
  --every 300

# 3. 数据备份提醒
picoclaw cron add --name "backup-reminder" \
  --message "提醒备份数据库" \
  --cron "0 2 * * 0"

# 4. 定期清理
picoclaw cron add --name "cleanup" \
  --message "清理临时文件" \
  --cron "0 3 * * *"
```

**与其他命令对比**：

| 特性 | `picoclaw cron` | `picoclaw agent` | Heartbeat |
|------|----------------|-----------------|-----------|
| 执行时机 | 定时调度 | 立即执行 | 固定间隔 |
| 配置方式 | CLI 命令 | CLI 参数 | HEARTBEAT.md |
| 持久化 | ✅ JSON 文件 | ❌ 一次性 | ❌ 只读文件 |
| 结果通知 | ✅ 可选渠道 | ✅ 终端输出 | ❌ 静默 |
| 管理界面 | ✅ 完整 CLI | ❌ 无 | ❌ 无 |

---

## 📚 相关文件

- `cmd/picoclaw/internal/cron/command.go` - 父命令定义
- `cmd/picoclaw/internal/cron/list.go` - 列表子命令
- `cmd/picoclaw/internal/cron/add.go` - 添加子命令
- `cmd/picoclaw/internal/cron/remove.go` - 删除子命令
- `cmd/picoclaw/internal/cron/enable.go` - 启用子命令
- `cmd/picoclaw/internal/cron/disable.go` - 禁用子命令
- `cmd/picoclaw/internal/cron/helpers.go` - 辅助函数
- `pkg/cron/` - Cron 服务核心实现

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
auth login (认证)
    ↓
gateway (启动服务) ← cron 依赖 gateway 运行
    ↓
cron add (添加定时任务)
    ↓
cron list (查看任务)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
