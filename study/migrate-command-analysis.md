# NewMigrateCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/migrate/command.go`
**行数**: 9-52
**核心包**: `pkg/migrate/`

---

## 🎯 核心功能

`NewMigrateCommand` 是 PicoClaw 的**数据迁移命令**，用于从其他类似项目（如 OpenClaw）迁移配置和工作空间到 PicoClaw。这是一个**一次性操作**命令，支持预览和增量同步。

**主要职责**：
1. 配置文件迁移（格式转换）
2. 工作空间文件复制
3. 智能冲突处理
4. 干运行预览
5. 增量刷新支持

---

## 📊 结构拆解

### 1️⃣ **Cobra 命令定义** (command.go:9-52)

```go
func NewMigrateCommand() *cobra.Command {
    var opts migrate.Options

    cmd := &cobra.Command{
        Use:   "migrate",
        Short: "Migrate from xxxclaw(openclaw, etc.) to picoclaw",
        Args:  cobra.NoArgs,
        Example: `  picoclaw migrate
  picoclaw migrate --from openclaw
  picoclaw migrate --dry-run
  picoclaw migrate --refresh
  picoclaw migrate --force`,
        RunE: func(cmd *cobra.Command, _ []string) error {
            // 创建迁移实例
            m := migrate.NewMigrateInstance(opts)

            // 执行迁移
            result, err := m.Run(opts)
            if err != nil {
                return err
            }

            // 打印摘要（非 dry-run）
            if !opts.DryRun {
                m.PrintSummary(result)
            }

            return nil
        },
    }

    // 注册标志
    cmd.Flags().BoolVar(&opts.DryRun, "dry-run", false,
        "Show what would be migrated without making changes")
    cmd.Flags().StringVar(&opts.Source, "from", "openclaw",
        "Source to migrate from (e.g., openclaw)")
    cmd.Flags().BoolVar(&opts.Refresh, "refresh", false,
        "Re-sync workspace files from OpenClaw (repeatable)")
    cmd.Flags().BoolVar(&opts.ConfigOnly, "config-only", false,
        "Only migrate config, skip workspace files")
    cmd.Flags().BoolVar(&opts.WorkspaceOnly, "workspace-only", false,
        "Only migrate workspace files, skip config")
    cmd.Flags().BoolVar(&opts.Force, "force", false,
        "Skip confirmation prompts")
    cmd.Flags().StringVar(&opts.SourceHome, "source-home", "",
        "Override source home directory (default: ~/.openclaw)")
    cmd.Flags().StringVar(&opts.TargetHome, "target-home", "",
        "Override target home directory (default: ~/.picoclaw)")

    return cmd
}
```

#### 参数说明：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--dry-run` | bool | false | 预览模式，不实际修改 |
| `--from` | string | "openclaw" | 源项目名称 |
| `--refresh` | bool | false | 增量同步工作空间文件 |
| `--config-only` | bool | false | 仅迁移配置（与 --workspace-only 互斥） |
| `--workspace-only` | bool | false | 仅迁移工作空间（与 --config-only 互斥） |
| `--force` | bool | false | 跳过确认提示 |
| `--source-home` | string | "" | 自定义源目录 |
| `--target-home` | string | "" | 自定义目标目录 |

---

## 🔄 执行流程详解

### 核心流程 (`migrate.Run()`)

```
┌─────────────────────────────────────────────────┐
│              migrate.Run() 执行流程              │
└─────────────────────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 1. 获取源 Handler                  │
    │    (目前仅支持 openclaw)           │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 2. 检查互斥参数                    │
    │    --config-only                  │
    │    --workspace-only               │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 3. 解析源和目标路径                │
    │    Source: ~/.openclaw            │
    │    Target: ~/.picoclaw            │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 4. 检查源目录是否存在              │
    │    os.Stat(sourceHome)            │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 5. 计划迁移操作 (Plan)             │
    │    • 配置文件转换                  │
    │    • 工作空间文件复制              │
    │    • 冲突检测                      │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 6. 显示迁移计划                    │
    │    • 操作列表                      │
    │    • 警告信息                      │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 7. 用户确认 (除非 --force)         │
    │    "Proceed? (y/n): "             │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 8. 执行操作 (Execute)              │
    │    • 创建备份                      │
    │    • 转换配置                      │
    │    • 复制文件                      │
    └───────────────────────────────────┘
                        ↓
    ┌───────────────────────────────────┐
    │ 9. 打印摘要                        │
    │    • 成功操作数                    │
    │    • 跳过操作数                    │
    │    • 错误信息                      │
    └───────────────────────────────────┘
```

---

## 📦 使用示例

### **1. 标准迁移（首次）**
```bash
picoclaw migrate

# 输出:
# Migrating from Source to PicoClaw
#   Source:      /Users/username/.openclaw
#   Target:      /Users/username/.picoclaw
#
# Actions:
#   [BACKUP]  config.json → config.json.bak.20260304_150000
#   [CONVERT] config.json (openclaw → picoclaw)
#   [COPY]    workspace/AGENTS.md
#   [COPY]    workspace/IDENTITY.md
#   [COPY]    workspace/sessions/sessions.db
#   ...
#
# Warnings:
#   • Target config already exists, will create backup
#
# Proceed? (y/n): y
#
# ✓ Migration completed successfully
# Summary:
#   • 1 config converted
#   • 15 files copied
#   • 1 backup created
```

### **2. 预览模式（不实际修改）**
```bash
picoclaw migrate --dry-run

# 输出:
# [DRY RUN] No changes will be made
# Migrating from Source to PicoClaw
#   Source:      /Users/username/.openclaw
#   Target:      /Users/username/.picoclaw
#
# Actions:
#   [BACKUP]  config.json → config.json.bak.20260304_150000
#   [CONVERT] config.json (openclaw → picoclaw)
#   [COPY]    workspace/AGENTS.md
#   ...
#
# This was a dry run. Use 'picoclaw migrate' to apply changes.
```

### **3. 仅迁移配置**
```bash
picoclaw migrate --config-only

# 只转换配置文件，不复制工作空间
```

### **4. 仅迁移工作空间**
```bash
picoclaw migrate --workspace-only

# 只复制工作空间文件，不修改配置
```

### **5. 增量刷新（可重复执行）**
```bash
# 从 OpenClaw 同步最新的工作空间文件
picoclaw migrate --refresh

# 等同于:
# picoclaw migrate --workspace-only --force
```

### **6. 跳过确认提示**
```bash
picoclaw migrate --force

# 自动化脚本中使用，不需要交互
```

### **7. 自定义源和目标路径**
```bash
picoclaw migrate \
  --source-home /custom/openclaw \
  --target-home /custom/picoclaw
```

---

## 🔍 迁移操作类型

### **ActionType 枚举**

| 操作类型 | 说明 | 示例 |
|---------|------|------|
| `ActionCopy` | 直接复制文件 | 工作空间模板文件 |
| `ActionSkip` | 跳过（已存在且相同） | 未修改的文件 |
| `ActionBackup` | 创建备份 | 覆盖前备份配置 |
| `ActionConvertConfig` | 转换配置格式 | openclaw → picoclaw |
| `ActionCreateDir` | 创建目录 | 目标目录不存在时 |
| `ActionMergeConfig` | 合并配置 | 保留用户自定义 |

---

## 🔐 配置转换逻辑

### **OpenClaw → PicoClaw 配置映射**

```go
// OpenClaw 配置结构
{
  "providers": {
    "openai": {
      "api_key": "sk-..."
    }
  },
  "agents": {
    "default": {
      "model": "gpt-4",
      "workspace": "~/.openclaw/workspace"
    }
  }
}

// 转换为 PicoClaw 配置
{
  "model_list": [
    {
      "model_name": "gpt-4",
      "model": "openai/gpt-4",
      "api_key": "sk-..."
    }
  ],
  "agents": {
    "defaults": {
      "model": "gpt-4",
      "workspace": "~/.picoclaw/workspace"  // 路径更新
    }
  }
}
```

### **转换规则**：

1. **提供商整合**: 将 `providers` 合并到 `model_list`
2. **路径更新**: 所有 `~/.openclaw` → `~/.picoclaw`
3. **模型格式**: 添加 `vendor/` 前缀（如 `openai/gpt-4`）
4. **保留自定义**: 用户自定义配置优先级最高

---

## 🗂️ 工作空间迁移

### **复制文件列表**

```
源目录: ~/.openclaw/workspace/
目标目录: ~/.picoclaw/workspace/

迁移文件:
├── AGENTS.md              # 代理行为指南
├── IDENTITY.md            # 代理身份
├── SOUL.md                # 代理灵魂
├── TOOLS.md               # 工具描述
├── USER.md                # 用户偏好
├── HEARTBEAT.md           # 心跳任务
├── sessions/              # 会话历史
│   └── sessions.db
├── memory/                # 长期记忆
│   └── MEMORY.md
├── state/                 # 持久化状态
├── cron/                  # 定时任务
│   └── jobs.json
└── skills/                # 自定义技能
    └── *.md
```

### **冲突处理策略**

| 场景 | 策略 |
|------|------|
| 目标文件不存在 | 直接复制 |
| 目标文件相同 | 跳过（ActionSkip） |
| 目标文件不同 | 创建备份后覆盖 |
| 二进制文件 | 提示警告，需要手动处理 |

---

## 🔑 关键设计要点

### **1. 干运行模式**

```go
if opts.DryRun {
    // 计算所有操作但不执行
    actions, warnings, _ := m.Plan(opts, sourceHome, targetHome)

    // 显示操作列表
    for _, action := range actions {
        fmt.Printf("[DRY RUN] %s\n", action.Description)
    }

    return result, nil  // 不调用 Execute()
}
```

**好处**：
- ✅ 安全预览
- ✅ 提前发现问题
- ✅ 无风险测试

---

### **2. 增量刷新 (`--refresh`)**

```go
if opts.Refresh {
    opts.WorkspaceOnly = true  // 自动设置为仅工作空间
    opts.Force = true          // 自动跳过确认
}
```

**用途**：
```bash
# 场景：OpenClaw 更新了工作空间模板
# 同步到 PicoClaw 而不修改配置
picoclaw migrate --refresh
```

---

### **3. 备份机制**

```go
// 覆盖前自动备份
if fileExists(targetPath) {
    backupPath := fmt.Sprintf("%s.bak.%s", targetPath, timestamp)
    copyFile(targetPath, backupPath)
}
```

**备份命名**：
```
config.json.bak.20260304_150000
config.json.bak.20260304_151500
...
```

---

### **4. 可扩展架构**

```go
type Operation interface {
    GetSourceName() string
    GetSourceHome() (string, error)
    Plan(opts Options) ([]Action, []string, error)
    Execute(actions []Action) (*Result, error)
}

// 注册 Handler
m.Register("openclaw", openclawHandler)
m.Register("nanobot", nanobotHandler)  // 未来可添加
```

**支持的源**：
- ✅ OpenClaw（已实现）
- 🚧 Nanobot（计划中）
- 🚧 其他类似项目

---

### **5. 互斥参数验证**

```go
if opts.ConfigOnly && opts.WorkspaceOnly {
    return nil, fmt.Errorf("--config-only and --workspace-only are mutually exclusive")
}
```

**冲突检测**：
- `--config-only` ⇄ `--workspace-only`
- 必须二选一或都不选（默认两者都迁移）

---

## 🔗 相关函数调用链

```
NewMigrateCommand()
    └─ cmd.RunE → migrate.Run(opts)
        ├─ getCurrentHandler()
        │   └─ 返回 openclawHandler
        │
        ├─ handler.GetSourceHome()
        │   └─ 解析 ~/.openclaw
        │
        ├─ ResolveTargetHome()
        │   └─ 解析 ~/.picoclaw
        │
        ├─ os.Stat(sourceHome)
        │   └─ 检查源目录是否存在
        │
        ├─ Plan(opts, sourceHome, targetHome)
        │   ├─ 分析配置文件
        │   ├─ 遍历工作空间文件
        │   ├─ 检测冲突
        │   └─ 返回 actions[]
        │
        ├─ 显示迁移计划
        │   └─ 打印每个 action
        │
        ├─ 用户确认 (除非 --force)
        │   └─ fmt.Scanln(&response)
        │
        ├─ Execute(actions)
        │   ├─ ActionBackup → copyFile()
        │   ├─ ActionConvert → convertConfig()
        │   ├─ ActionCopy → copyFile()
        │   └─ ActionCreateDir → os.MkdirAll()
        │
        └─ PrintSummary(result)
            └─ 打印成功/跳过/错误统计
```

---

## ⚠️ 注意事项

### **1. 源目录必须存在**
```bash
picoclaw migrate
# Error: Source installation not found at /Users/username/.openclaw

# 解决方案：确认 OpenClaw 已安装
ls ~/.openclaw
```

### **2. 配置冲突**
```bash
# 如果目标配置已存在，会自动备份
# 备份位置: ~/.picoclaw/config.json.bak.20260304_150000

# 恢复备份:
mv ~/.picoclaw/config.json.bak.20260304_150000 ~/.picoclaw/config.json
```

### **3. 会话数据库兼容性**
```bash
# SQLite 数据库通常兼容，但谨慎处理
# 建议先备份:
cp ~/.openclaw/workspace/sessions/sessions.db \
   ~/.openclaw/workspace/sessions/sessions.db.backup
```

### **4. 路径引用更新**
```bash
# 迁移后，检查配置中的绝对路径
grep -r "openclaw" ~/.picoclaw/config.json

# 手动更新（如果自动转换遗漏）
sed -i 's/openclaw/picoclaw/g' ~/.picoclaw/config.json
```

### **5. 重复执行**
```bash
# 标准迁移：首次执行后，再次执行会检测到文件已存在
picoclaw migrate
# Warning: Target config already exists, will create backup

# 增量刷新：可安全重复执行
picoclaw migrate --refresh  # 可多次运行
```

---

## 🎓 总结

`NewMigrateCommand` 是 PicoClaw 的**一键迁移工具**，它：

1. ✅ **智能转换**: 自动转换配置格式
2. ✅ **安全备份**: 覆盖前自动备份
3. ✅ **干运行预览**: 风险评估无副作用
4. ✅ **增量同步**: 支持多次刷新
5. ✅ **冲突处理**: 智能检测和解决冲突
6. ✅ **可扩展**: 易于添加新的迁移源

**推荐迁移流程**：
```bash
# 步骤 1: 预览迁移计划
picoclaw migrate --dry-run

# 步骤 2: 执行迁移
picoclaw migrate

# 步骤 3: 验证配置
picoclaw status

# 步骤 4: 测试功能
picoclaw agent -m "Hello"

# 可选: 同步 OpenClaw 的更新
picoclaw migrate --refresh
```

**迁移检查清单**：

```
☐ 备份重要数据
☐ 运行 --dry-run 预览
☐ 检查警告信息
☐ 执行迁移
☐ 验证配置文件
☐ 测试基本功能
☐ 检查会话历史
☐ 验证定时任务
☐ 测试自定义技能
```

**与其他命令对比**：

| 特性 | `migrate` | `onboard` |
|------|----------|-----------|
| 用途 | 从其他项目迁移 | 全新初始化 |
| 数据保留 | ✅ 保留所有数据 | ❌ 创建空白 |
| 配置转换 | ✅ 智能转换 | ✅ 默认配置 |
| 会话历史 | ✅ 迁移 | ❌ 空 |
| 适用场景 | 从 OpenClaw 切换 | 首次安装 |

---

## 📚 相关文件

- `cmd/picoclaw/internal/migrate/command.go` - 命令定义
- `pkg/migrate/migrate.go` - 核心迁移逻辑
- `pkg/migrate/internal/types.go` - 类型定义
- `pkg/migrate/internal/common.go` - 公共函数
- `pkg/migrate/sources/openclaw/` - OpenClaw 迁移实现

---

## 🔍 与其他命令的关系

```
[从 OpenClaw 迁移]
    ↓
migrate (数据迁移)
    ↓
status (验证迁移)
    ↓
agent / gateway (正常使用)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
