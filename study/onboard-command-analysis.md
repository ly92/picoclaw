# NewOnboardCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/onboard/command.go`
**行数**: 13-24
**实现文件**: `cmd/picoclaw/internal/onboard/helpers.go`

---

## 🎯 核心功能

`NewOnboardCommand` 是 PicoClaw 的初始化命令，负责创建默认配置文件和工作空间模板。这是用户首次使用 PicoClaw 时**必须执行**的命令。

**主要职责**：
1. 生成默认配置文件 (`~/.picoclaw/config.json`)
2. 创建工作空间目录结构
3. 复制嵌入的模板文件到工作空间
4. 引导用户完成后续配置

---

## 📊 结构拆解

### 1️⃣ **嵌入式文件系统** (第 9-11 行)

```go
//go:generate cp -r ../../../../workspace .
//go:embed workspace
var embeddedFiles embed.FS
```

#### 说明：
- **go:generate**: 编译前将项目根目录的 `workspace` 文件夹复制到当前包目录
- **go:embed**: 将 `workspace` 文件夹嵌入到二进制文件中
- **作用**: 确保 PicoClaw 单个二进制文件包含所有必要的初始模板

---

### 2️⃣ **Cobra 命令定义** (第 13-24 行)

```go
func NewOnboardCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:     "onboard",              // 命令名称
        Aliases: []string{"o"},          // 简短别名
        Short:   "Initialize picoclaw configuration and workspace",
        Run: func(cmd *cobra.Command, args []string) {
            onboard()
        },
    }

    return cmd
}
```

#### Cobra 配置说明：

| 字段 | 值 | 说明 |
|------|-----|------|
| **Use** | `"onboard"` | 命令名称 |
| **Aliases** | `[]string{"o"}` | 可用 `picoclaw o` 作为快捷方式 |
| **Short** | 简短描述 | 在 `--help` 中显示 |
| **Run** | 执行函数 | 调用 `onboard()` 进行实际初始化 |

**特点**：
- ✅ 无需任何参数或标志
- ✅ 提供简短别名 `o`
- ✅ 执行逻辑独立封装在 `onboard()` 函数中

---

## 🔄 执行流程详解

### 核心函数：`onboard()` (`helpers.go:13-47`)

```
┌─────────────────────────────────────────────────────────────┐
│                    onboard() 执行流程                        │
└─────────────────────────────────────────────────────────────┘
                            ↓
        ┌───────────────────────────────────┐
        │ 1. 检查配置文件是否存在            │
        │    Path: ~/.picoclaw/config.json  │
        └───────────────────────────────────┘
                            ↓
                ┌───────────┴───────────┐
                │                       │
        存在 ──►│  提示用户确认覆盖      │◄─── 不存在
                │  "Overwrite? (y/n)"   │      (跳过确认)
                └───────────┬───────────┘
                            │
                    输入 'n' ─────► 中止执行
                            │
                    输入 'y' 或不存在
                            ↓
        ┌───────────────────────────────────┐
        │ 2. 创建默认配置                    │
        │    cfg = config.DefaultConfig()   │
        │    • workspace: ~/.picoclaw/workspace
        │    • max_tokens: 32768            │
        │    • restrict_to_workspace: true  │
        └───────────────────────────────────┘
                            ↓
        ┌───────────────────────────────────┐
        │ 3. 保存配置到文件                  │
        │    config.SaveConfig(path, cfg)   │
        └───────────────────────────────────┘
                            ↓
        ┌───────────────────────────────────┐
        │ 4. 创建工作空间模板                │
        │    createWorkspaceTemplates()     │
        │    └─ copyEmbeddedToTarget()      │
        └───────────────────────────────────┘
                            ↓
        ┌───────────────────────────────────┐
        │ 5. 显示成功消息和后续步骤          │
        │    "picoclaw is ready! 🦞"        │
        └───────────────────────────────────┘
```

---

## 📁 工作空间模板复制

### 函数：`copyEmbeddedToTarget()` (`helpers.go:56-101`)

```go
func copyEmbeddedToTarget(targetDir string) error {
    // 1. 确保目标目录存在
    os.MkdirAll(targetDir, 0o755)

    // 2. 遍历嵌入的文件系统
    fs.WalkDir(embeddedFiles, "workspace", func(path, d, err) {
        // 跳过目录
        if d.IsDir() { return nil }

        // 3. 读取嵌入的文件
        data := embeddedFiles.ReadFile(path)

        // 4. 计算目标路径
        relPath := filepath.Rel("workspace", path)
        targetPath := filepath.Join(targetDir, relPath)

        // 5. 创建目标目录
        os.MkdirAll(filepath.Dir(targetPath), 0o755)

        // 6. 写入文件
        os.WriteFile(targetPath, data, 0o644)
    })
}
```

#### 工作空间目录结构：

根据 CLAUDE.md，工作空间会创建以下结构：

```
~/.picoclaw/workspace/
├── sessions/          # 会话历史（SQLite）
├── memory/            # 长期记忆（MEMORY.md）
├── state/             # 持久化状态
├── cron/              # 定时任务数据库
├── skills/            # 自定义技能
├── AGENTS.md          # 代理行为指南
├── HEARTBEAT.md       # 定期任务提示
├── IDENTITY.md        # 代理身份
├── SOUL.md            # 代理灵魂
├── TOOLS.md           # 工具描述
└── USER.md            # 用户偏好
```

---

## 📦 使用示例

### **1. 标准初始化**
```bash
picoclaw onboard

# 输出示例:
# 🦞 picoclaw is ready!
#
# Next steps:
#   1. Add your API key to /Users/username/.picoclaw/config.json
#
#      Recommended:
#      - OpenRouter: https://openrouter.ai/keys (access 100+ models)
#      - Ollama:     https://ollama.com (local, free)
#
#      See README.md for 17+ supported providers.
#
#   2. Chat: picoclaw agent -m "Hello!"
```

### **2. 使用别名**
```bash
picoclaw o    # 等同于 picoclaw onboard
```

### **3. 覆盖现有配置**
```bash
picoclaw onboard

# 如果配置已存在:
# Config already exists at /Users/username/.picoclaw/config.json
# Overwrite? (y/n): y
# 🦞 picoclaw is ready!
```

### **4. 取消覆盖**
```bash
picoclaw onboard

# 如果配置已存在:
# Config already exists at /Users/username/.picoclaw/config.json
# Overwrite? (y/n): n
# Aborted.
```

---

## 🛠️ 默认配置内容

### 配置文件路径优先级：
```
1. $PICOCLAW_CONFIG (环境变量)
2. $PICOCLAW_HOME/config.json
3. ~/.picoclaw/config.json (默认)
```

### 默认配置结构 (`config.DefaultConfig()`)：

```go
{
  "agents": {
    "defaults": {
      "workspace": "~/.picoclaw/workspace",
      "restrict_to_workspace": true,
      "provider": "",               // 需要用户配置
      "model": "",                  // 需要用户配置
      "max_tokens": 32768,
      "temperature": null,          // 使用提供商默认值
      "max_tool_iterations": 20,
      "allow_read_outside_workspace": false
    }
  },
  "tools": {
    "web": {
      "brave": {"enabled": false},
      "tavily": {"enabled": false},
      "duckduckgo": {"enabled": true}
    },
    "mcp": {
      "enabled": false,
      "servers": {}
    }
  },
  "channels": {
    "telegram": {"enabled": false},
    "discord": {"enabled": false},
    "webhook_server": {"enabled": false}
  }
}
```

---

## 🔑 关键设计要点

### **1. 嵌入式文件系统**
- ✅ **优势**: 单一二进制文件，无需额外的安装包
- ✅ **可移植性**: 可在任何系统上直接运行
- ✅ **构建时生成**: 通过 `go:generate` 自动化

### **2. 安全确认机制**
```go
if _, err := os.Stat(configPath); err == nil {
    fmt.Print("Overwrite? (y/n): ")
    // 防止意外覆盖现有配置
}
```

### **3. 用户引导**
- 清晰的"下一步"指引
- 推荐的 API 提供商（OpenRouter, Ollama）
- 快速验证命令

### **4. 错误处理**
```go
if err := config.SaveConfig(configPath, cfg); err != nil {
    fmt.Printf("Error saving config: %v\n", err)
    os.Exit(1)  // 失败时立即退出
}
```

### **5. 工作空间隔离**
- 默认启用 `restrict_to_workspace: true`
- 所有文件操作受限于工作空间目录
- 提高安全性，防止意外修改系统文件

---

## 🔗 相关函数调用链

```
NewOnboardCommand()
    └─ cmd.Run → onboard()
        ├─ internal.GetConfigPath()
        │   └─ 读取环境变量 PICOCLAW_CONFIG 或使用默认路径
        │
        ├─ os.Stat(configPath)
        │   └─ 检查配置文件是否存在
        │
        ├─ config.DefaultConfig()
        │   ├─ 读取 PICOCLAW_HOME 环境变量
        │   ├─ 构建默认配置结构
        │   └─ 设置工作空间路径
        │
        ├─ config.SaveConfig(path, cfg)
        │   └─ 将配置序列化为 JSON 并写入文件
        │
        └─ createWorkspaceTemplates(workspace)
            └─ copyEmbeddedToTarget(workspace)
                ├─ os.MkdirAll() - 创建目录
                ├─ fs.WalkDir() - 遍历嵌入文件
                ├─ embeddedFiles.ReadFile() - 读取文件
                └─ os.WriteFile() - 写入目标文件
```

---

## 📊 执行效果

### **执行前**
```
系统中没有任何 PicoClaw 相关文件
```

### **执行后**
```
~/.picoclaw/
├── config.json          # 配置文件
└── workspace/           # 工作空间
    ├── AGENTS.md
    ├── IDENTITY.md
    ├── SOUL.md
    ├── TOOLS.md
    ├── USER.md
    ├── HEARTBEAT.md
    ├── sessions/
    ├── memory/
    ├── state/
    ├── cron/
    └── skills/
```

---

## ⚠️ 注意事项

### **1. 首次运行必须**
```bash
# 如果没有运行 onboard，其他命令会失败
picoclaw agent -m "Hello"
# Error: config file not found at ~/.picoclaw/config.json
# Please run 'picoclaw onboard' first
```

### **2. 配置覆盖风险**
- 如果已有配置，会提示确认
- 输入 `y` 会**完全覆盖**现有配置
- 建议先备份重要配置

### **3. 工作空间路径**
```bash
# 使用自定义路径
export PICOCLAW_HOME=/custom/path
picoclaw onboard
# 工作空间将创建在: /custom/path/workspace
```

### **4. 权限要求**
- 需要对 `~/.picoclaw` 目录有写权限
- 目录权限：`0755` (rwxr-xr-x)
- 文件权限：`0644` (rw-r--r--)

---

## 🎓 总结

`NewOnboardCommand` 是 PicoClaw 的**入门命令**，它：

1. ✅ **简洁高效**: 无需任何参数，一键初始化
2. ✅ **嵌入式设计**: 所有模板打包在二进制中
3. ✅ **安全可靠**: 防止意外覆盖，提供确认机制
4. ✅ **用户友好**: 清晰的后续步骤指引
5. ✅ **可定制**: 支持环境变量自定义路径

**推荐工作流**：
```bash
# 1. 初始化
picoclaw onboard

# 2. 编辑配置文件，添加 API Key
vim ~/.picoclaw/config.json

# 3. 验证配置
picoclaw agent -m "你好"

# 4. 启动网关（可选）
picoclaw gateway
```

---

## 📚 相关文件

- `cmd/picoclaw/internal/onboard/command.go` - 命令定义
- `cmd/picoclaw/internal/onboard/helpers.go` - 实现逻辑
- `pkg/config/defaults.go` - 默认配置生成
- `pkg/config/config.go` - 配置结构定义
- `workspace/` - 嵌入的模板文件（项目根目录）

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
┌───┴────────────────┐
│                    │
agent              gateway
(本地对话)        (启动服务)
    │                │
    └────── status ──┘
         (查看状态)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
