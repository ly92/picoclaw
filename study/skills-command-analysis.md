# NewSkillsCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/skills/command.go`
**行数**: 19-79
**实现文件**: `cmd/picoclaw/internal/skills/helpers.go`
**子命令文件**: `list.go`, `install.go`, `remove.go`, `search.go`, `show.go`, `listbuiltin.go`, `installbuiltin.go`

---

## 🎯 核心功能

`NewSkillsCommand` 是 PicoClaw 的**技能管理系统**，提供从多个源安装、搜索、管理技能的完整功能。这是一个**父命令**，包含 7 个子命令。

**主要职责**：
1. 从 GitHub 安装技能
2. 从技能注册表安装（ClawHub）
3. 搜索可用技能
4. 列出已安装技能
5. 显示技能详情
6. 安装内置技能
7. 删除技能

---

## 📊 结构拆解

### 1️⃣ **父命令定义** (command.go:19-79)

```go
type deps struct {
    workspace    string
    installer    *skills.SkillInstaller
    skillsLoader *skills.SkillsLoader
}

func NewSkillsCommand() *cobra.Command {
    var d deps

    cmd := &cobra.Command{
        Use:   "skills",
        Short: "Manage skills",
        PersistentPreRunE: func(cmd *cobra.Command, _ []string) error {
            // 加载配置
            cfg, err := internal.LoadConfig()
            if err != nil {
                return fmt.Errorf("error loading config: %w", err)
            }

            // 初始化依赖
            d.workspace = cfg.WorkspacePath()
            d.installer = skills.NewSkillInstaller(d.workspace)

            // 技能目录：
            // - workspace: ~/.picoclaw/workspace/skills
            // - global:    ~/.picoclaw/skills
            // - builtin:   ~/.picoclaw/picoclaw/skills
            globalDir := filepath.Dir(internal.GetConfigPath())
            globalSkillsDir := filepath.Join(globalDir, "skills")
            builtinSkillsDir := filepath.Join(globalDir, "picoclaw", "skills")
            d.skillsLoader = skills.NewSkillsLoader(d.workspace, globalSkillsDir, builtinSkillsDir)

            return nil
        },
        RunE: func(cmd *cobra.Command, _ []string) error {
            return cmd.Help()  // 不带子命令时显示帮助
        },
    }

    // 通过闭包传递依赖到子命令
    installerFn := func() (*skills.SkillInstaller, error) {
        if d.installer == nil {
            return nil, fmt.Errorf("skills installer is not initialized")
        }
        return d.installer, nil
    }

    loaderFn := func() (*skills.SkillsLoader, error) {
        if d.skillsLoader == nil {
            return nil, fmt.Errorf("skills loader is not initialized")
        }
        return d.skillsLoader, nil
    }

    workspaceFn := func() (string, error) {
        if d.workspace == "" {
            return "", fmt.Errorf("workspace is not initialized")
        }
        return d.workspace, nil
    }

    // 注册子命令
    cmd.AddCommand(
        newListCommand(loaderFn),
        newInstallCommand(installerFn),
        newInstallBuiltinCommand(workspaceFn),
        newListBuiltinCommand(),
        newRemoveCommand(installerFn),
        newSearchCommand(),
        newShowCommand(loaderFn),
    )

    return cmd
}
```

---

### 2️⃣ **子命令架构**

```
picoclaw skills
    ├─ list                  # 列出已安装技能
    ├─ install <repo>        # 从 GitHub 安装
    ├─ install --registry    # 从注册表安装
    ├─ search [query]        # 搜索可用技能
    ├─ show <name>           # 显示技能详情
    ├─ remove <name>         # 删除技能
    ├─ list-builtin          # 列出内置技能
    └─ install-builtin       # 安装内置技能
```

---

## 🔐 子命令详解

### **子命令 1: `list`** (list.go + helpers.go:20-36)

```go
func skillsListCmd(loader *skills.SkillsLoader) {
    allSkills := loader.ListSkills()

    if len(allSkills) == 0 {
        fmt.Println("No skills installed.")
        return
    }

    fmt.Println("\nInstalled Skills:")
    fmt.Println("------------------")
    for _, skill := range allSkills {
        fmt.Printf("  ✓ %s (%s)\n", skill.Name, skill.Source)
        if skill.Description != "" {
            fmt.Printf("    %s\n", skill.Description)
        }
    }
}
```

---

### **子命令 2: `install`** (install.go + helpers.go:38-119)

#### **模式 A: 从 GitHub 安装**

```go
picoclaw skills install <owner>/<repo>/<path>

// 示例:
picoclaw skills install sipeed/picoclaw-skills/weather
```

**执行逻辑**：
```go
func skillsInstallCmd(installer *skills.SkillInstaller, repo string) error {
    fmt.Printf("Installing skill from %s...\n", repo)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 从 GitHub 下载并安装
    if err := installer.InstallFromGitHub(ctx, repo); err != nil {
        return fmt.Errorf("failed to install skill: %w", err)
    }

    fmt.Printf("✓ Skill '%s' installed successfully!\n", filepath.Base(repo))
    return nil
}
```

---

#### **模式 B: 从注册表安装**

```go
picoclaw skills install --registry <registry-name> <slug>

// 示例:
picoclaw skills install --registry clawhub weather
```

**执行逻辑** (helpers.go:54-119)：
```go
func skillsInstallFromRegistry(cfg *config.Config, registryName, slug string) error {
    // 1. 验证输入
    err := utils.ValidateSkillIdentifier(registryName)
    err = utils.ValidateSkillIdentifier(slug)

    fmt.Printf("Installing skill '%s' from %s registry...\n", slug, registryName)

    // 2. 获取注册表管理器
    registryMgr := skills.NewRegistryManagerFromConfig(...)

    // 3. 获取指定注册表
    registry := registryMgr.GetRegistry(registryName)
    if registry == nil {
        return fmt.Errorf("registry '%s' not found", registryName)
    }

    // 4. 检查是否已安装
    targetDir := filepath.Join(workspace, "skills", slug)
    if _, err = os.Stat(targetDir); err == nil {
        return fmt.Errorf("skill '%s' already installed", slug)
    }

    // 5. 下载并安装
    ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
    defer cancel()

    result, err := registry.DownloadAndInstall(ctx, slug, "", targetDir)
    if err != nil {
        // 失败时清理部分安装
        os.RemoveAll(targetDir)
        return fmt.Errorf("failed to install skill: %w", err)
    }

    // 6. 安全检查
    if result.IsMalwareBlocked {
        os.RemoveAll(targetDir)
        return fmt.Errorf("Skill '%s' is flagged as malicious", slug)
    }

    if result.IsSuspicious {
        fmt.Printf("⚠️  Warning: skill '%s' is flagged as suspicious.\n", slug)
    }

    fmt.Printf("✓ Skill '%s' v%s installed successfully!\n", slug, result.Version)
    return nil
}
```

---

### **子命令 3: `search`** (search.go + helpers.go:220-260)

```go
picoclaw skills search [query]

// 示例:
picoclaw skills search weather
picoclaw skills search           # 列出所有
```

**执行逻辑**：
```go
func skillsSearchCmd(query string) {
    fmt.Println("Searching for available skills...")

    cfg, _ := internal.LoadConfig()

    // 创建注册表管理器
    registryMgr := skills.NewRegistryManagerFromConfig(...)

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 搜索所有注册表
    results, err := registryMgr.SearchAll(ctx, query, skillsSearchMaxResults)
    if err != nil {
        fmt.Printf("✗ Failed to fetch skills list: %v\n", err)
        return
    }

    if len(results) == 0 {
        fmt.Println("No skills available.")
        return
    }

    // 显示结果
    fmt.Printf("\nAvailable Skills (%d):\n", len(results))
    fmt.Println("--------------------")
    for _, result := range results {
        fmt.Printf("  📦 %s\n", result.DisplayName)
        fmt.Printf("     %s\n", result.Summary)
        fmt.Printf("     Slug: %s\n", result.Slug)
        fmt.Printf("     Registry: %s\n", result.RegistryName)
        if result.Version != "" {
            fmt.Printf("     Version: %s\n", result.Version)
        }
        fmt.Println()
    }
}
```

---

### **子命令 4: `show`** (show.go + helpers.go:262-272)

```go
picoclaw skills show <skill-name>

// 示例:
picoclaw skills show weather
```

**执行逻辑**：
```go
func skillsShowCmd(loader *skills.SkillsLoader, skillName string) {
    // 加载技能内容
    content, ok := loader.LoadSkill(skillName)
    if !ok {
        fmt.Printf("✗ Skill '%s' not found\n", skillName)
        return
    }

    // 显示技能详情（SKILL.md 内容）
    fmt.Printf("\n📦 Skill: %s\n", skillName)
    fmt.Println("----------------------")
    fmt.Println(content)
}
```

---

### **子命令 5: `remove`** (remove.go + helpers.go:121-130)

```go
picoclaw skills remove <skill-name>

// 示例:
picoclaw skills remove weather
```

**执行逻辑**：
```go
func skillsRemoveCmd(installer *skills.SkillInstaller, skillName string) {
    fmt.Printf("Removing skill '%s'...\n", skillName)

    if err := installer.Uninstall(skillName); err != nil {
        fmt.Printf("✗ Failed to remove skill: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("✓ Skill '%s' removed successfully!\n", skillName)
}
```

---

### **子命令 6: `list-builtin`** (listbuiltin.go + helpers.go:168-218)

```go
picoclaw skills list-builtin
```

**执行逻辑**：
```go
func skillsListBuiltinCmd() {
    cfg, _ := internal.LoadConfig()
    builtinSkillsDir := filepath.Join(
        filepath.Dir(cfg.WorkspacePath()),
        "picoclaw", "skills",
    )

    fmt.Println("\nAvailable Builtin Skills:")
    fmt.Println("-----------------------")

    entries, _ := os.ReadDir(builtinSkillsDir)

    for _, entry := range entries {
        if entry.IsDir() {
            skillName := entry.Name()
            skillFile := filepath.Join(builtinSkillsDir, skillName, "SKILL.md")

            // 读取描述
            description := "No description"
            if data, err := os.ReadFile(skillFile); err == nil {
                // 解析 SKILL.md 中的描述
                // ...
            }

            fmt.Printf("  ✓  %s\n", entry.Name())
            if description != "" {
                fmt.Printf("     %s\n", description)
            }
        }
    }
}
```

---

### **子命令 7: `install-builtin`** (installbuiltin.go + helpers.go:132-166)

```go
picoclaw skills install-builtin
```

**执行逻辑**：
```go
func skillsInstallBuiltinCmd(workspace string) {
    builtinSkillsDir := "./picoclaw/skills"
    workspaceSkillsDir := filepath.Join(workspace, "skills")

    fmt.Printf("Copying builtin skills to workspace...\n")

    // 预定义的内置技能列表
    skillsToInstall := []string{
        "weather",
        "news",
        "stock",
        "calculator",
    }

    for _, skillName := range skillsToInstall {
        builtinPath := filepath.Join(builtinSkillsDir, skillName)
        workspacePath := filepath.Join(workspaceSkillsDir, skillName)

        if _, err := os.Stat(builtinPath); err != nil {
            fmt.Printf("⊘ Builtin skill '%s' not found: %v\n", skillName, err)
            continue
        }

        // 创建目标目录
        if err := os.MkdirAll(workspacePath, 0o755); err != nil {
            fmt.Printf("✗ Failed to create directory for %s: %v\n", skillName, err)
            continue
        }

        // 复制整个目录
        if err := copyDirectory(builtinPath, workspacePath); err != nil {
            fmt.Printf("✗ Failed to copy %s: %v\n", skillName, err)
        }
    }

    fmt.Println("\n✓ All builtin skills installed!")
    fmt.Println("Now you can use them in your workspace.")
}
```

---

## 📦 使用示例

### **1. 列出已安装技能**
```bash
picoclaw skills list

# 输出:
# Installed Skills:
# ------------------
#   ✓ weather (workspace)
#     Get weather information for any city
#   ✓ calculator (builtin)
#     Perform mathematical calculations
#   ✓ news (workspace)
#     Fetch latest news articles
```

### **2. 从 GitHub 安装技能**
```bash
picoclaw skills install sipeed/picoclaw-skills/weather

# 输出:
# Installing skill from sipeed/picoclaw-skills/weather...
# ✓ Skill 'weather' installed successfully!
```

### **3. 从注册表安装技能**
```bash
picoclaw skills install --registry clawhub weather

# 输出:
# Installing skill 'weather' from clawhub registry...
# ✓ Skill 'weather' v1.0.0 installed successfully!
#   Get weather information for any city
```

### **4. 搜索技能**
```bash
picoclaw skills search weather

# 输出:
# Searching for available skills...
#
# Available Skills (3):
# --------------------
#   📦 Weather Forecast
#      Get weather information for any city
#      Slug: weather
#      Registry: clawhub
#      Version: 1.0.0
#
#   📦 Weather Alerts
#      Receive severe weather alerts
#      Slug: weather-alerts
#      Registry: clawhub
#      Version: 0.5.0
```

### **5. 搜索所有技能**
```bash
picoclaw skills search

# 列出所有可用技能（最多 20 个）
```

### **6. 显示技能详情**
```bash
picoclaw skills show weather

# 输出:
# 📦 Skill: weather
# ----------------------
# # Weather Skill
#
# Get real-time weather information for any city.
#
# ## Usage
# Ask the agent: "What's the weather in Tokyo?"
#
# ## Parameters
# - city: City name
# - units: metric/imperial (default: metric)
```

### **7. 删除技能**
```bash
picoclaw skills remove weather

# 输出:
# Removing skill 'weather'...
# ✓ Skill 'weather' removed successfully!
```

### **8. 列出内置技能**
```bash
picoclaw skills list-builtin

# 输出:
# Available Builtin Skills:
# -----------------------
#   ✓  weather
#      Get weather forecasts
#   ✓  news
#      Fetch latest news
#   ✓  stock
#      Track stock prices
#   ✓  calculator
#      Mathematical calculations
```

### **9. 安装内置技能**
```bash
picoclaw skills install-builtin

# 输出:
# Copying builtin skills to workspace...
# ✓ All builtin skills installed!
# Now you can use them in your workspace.
```

---

## 🗂️ 技能目录结构

### **技能存储位置**

```
~/.picoclaw/
├── workspace/skills/        # 用户安装的技能（最高优先级）
│   ├── weather/
│   │   └── SKILL.md
│   ├── news/
│   │   └── SKILL.md
│   └── custom-skill/
│       └── SKILL.md
│
├── skills/                  # 全局共享技能
│   └── shared-skill/
│       └── SKILL.md
│
└── picoclaw/skills/         # 内置技能（最低优先级）
    ├── weather/
    ├── news/
    ├── stock/
    └── calculator/
```

### **加载优先级**

```
1. workspace/skills/    (最高) - 用户自定义
2. skills/              (中等) - 全局共享
3. picoclaw/skills/     (最低) - 内置默认
```

**冲突解决**：同名技能时，优先级高的覆盖优先级低的。

---

## 🔐 技能结构（SKILL.md）

### **标准格式**

```markdown
# Weather Skill

Get real-time weather information for any city.

## Usage

Ask the agent: "What's the weather in Tokyo?"

## Parameters

- city: City name (required)
- units: metric/imperial (default: metric)

## Examples

- "What's the weather in New York?"
- "Show me the forecast for London in Fahrenheit"

## API Requirements

Requires API key from OpenWeatherMap.
Set in config: tools.weather.api_key

## Version

1.0.0

## Author

PicoClaw Team
```

---

## 🔍 技能注册表

### **ClawHub 注册表**

**配置** (`config.json`):
```json
{
  "tools": {
    "skills": {
      "registries": {
        "clawhub": {
          "enabled": true,
          "base_url": "https://clawhub.picoclaw.org"
        }
      },
      "max_concurrent_searches": 3
    }
  }
}
```

**功能**：
- ✅ 集中式技能仓库
- ✅ 版本管理
- ✅ 恶意软件扫描
- ✅ 可疑内容标记
- ✅ 搜索和发现

---

## 🔑 关键设计要点

### **1. 多源安装**

```
源 1: GitHub
  └─ picoclaw skills install owner/repo/path

源 2: 注册表 (ClawHub)
  └─ picoclaw skills install --registry clawhub slug

源 3: 内置
  └─ picoclaw skills install-builtin
```

---

### **2. 安全机制**

```go
// 恶意软件检测
if result.IsMalwareBlocked {
    os.RemoveAll(targetDir)
    return fmt.Errorf("Skill is flagged as malicious")
}

// 可疑内容警告
if result.IsSuspicious {
    fmt.Printf("⚠️  Warning: skill is flagged as suspicious.\n")
}
```

**安全级别**：
- 🚫 **Malicious**: 阻止安装
- ⚠️ **Suspicious**: 允许但警告
- ✅ **Safe**: 正常安装

---

### **3. 闭包依赖注入**

```go
// 在 PersistentPreRunE 中初始化
d.installer = skills.NewSkillInstaller(d.workspace)
d.skillsLoader = skills.NewSkillsLoader(...)

// 通过闭包传递给子命令
installerFn := func() (*skills.SkillInstaller, error) {
    if d.installer == nil {
        return nil, fmt.Errorf("not initialized")
    }
    return d.installer, nil
}

cmd.AddCommand(newInstallCommand(installerFn))
```

**好处**：
- ✅ 延迟初始化
- ✅ 共享依赖
- ✅ 错误检查

---

### **4. 超时保护**

```go
// GitHub 安装: 30秒
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)

// 注册表安装: 60秒
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)

// 搜索: 30秒
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
```

---

### **5. 错误回滚**

```go
result, err := registry.DownloadAndInstall(ctx, slug, "", targetDir)
if err != nil {
    // 失败时清理部分安装
    rmErr := os.RemoveAll(targetDir)
    if rmErr != nil {
        fmt.Printf("✗ Failed to remove partial install: %v\n", rmErr)
    }
    return fmt.Errorf("failed to install skill: %w", err)
}
```

---

## 🔗 相关函数调用链

```
NewSkillsCommand()
    ├─ PersistentPreRunE
    │   ├─ LoadConfig()
    │   ├─ NewSkillInstaller()
    │   └─ NewSkillsLoader()
    │
    ├─ newListCommand()
    │   └─ skillsListCmd()
    │       └─ loader.ListSkills()
    │
    ├─ newInstallCommand()
    │   ├─ skillsInstallCmd()
    │   │   └─ installer.InstallFromGitHub()
    │   │
    │   └─ skillsInstallFromRegistry()
    │       ├─ NewRegistryManagerFromConfig()
    │       ├─ GetRegistry()
    │       └─ registry.DownloadAndInstall()
    │
    ├─ newSearchCommand()
    │   └─ skillsSearchCmd()
    │       └─ registryMgr.SearchAll()
    │
    ├─ newShowCommand()
    │   └─ skillsShowCmd()
    │       └─ loader.LoadSkill()
    │
    ├─ newRemoveCommand()
    │   └─ skillsRemoveCmd()
    │       └─ installer.Uninstall()
    │
    ├─ newListBuiltinCommand()
    │   └─ skillsListBuiltinCmd()
    │       └─ os.ReadDir()
    │
    └─ newInstallBuiltinCommand()
        └─ skillsInstallBuiltinCmd()
            └─ copyDirectory()
```

---

## ⚠️ 注意事项

### **1. 网络依赖**
```bash
# 从 GitHub 或注册表安装需要网络连接
picoclaw skills install sipeed/picoclaw-skills/weather
# Error: failed to fetch: connection timeout

# 内置技能不需要网络
picoclaw skills install-builtin  # 离线可用
```

### **2. 技能冲突**
```bash
# 同名技能已存在
picoclaw skills install --registry clawhub weather
# Error: skill 'weather' already installed at ~/.picoclaw/workspace/skills/weather

# 解决方案：先删除旧版本
picoclaw skills remove weather
picoclaw skills install --registry clawhub weather
```

### **3. API 密钥要求**
```bash
# 某些技能需要 API 密钥
picoclaw skills install --registry clawhub weather

# 配置 API 密钥
vim ~/.picoclaw/config.json
# {
#   "tools": {
#     "weather": {
#       "api_key": "your-openweathermap-key"
#     }
#   }
# }
```

### **4. 技能命名规范**
```bash
# 有效名称
weather
news-aggregator
my_custom_skill

# 无效名称
Weather               # 大写
news aggregator       # 空格
my-skill@v1           # 特殊字符
```

---

## 🎓 总结

`NewSkillsCommand` 是 PicoClaw 的**技能生态系统管理中心**，它：

1. ✅ **多源安装**: GitHub + 注册表 + 内置
2. ✅ **安全扫描**: 恶意软件检测和可疑内容警告
3. ✅ **搜索发现**: 集中式技能仓库
4. ✅ **版本管理**: 支持版本号
5. ✅ **优先级加载**: workspace > global > builtin
6. ✅ **完整生命周期**: 安装 → 使用 → 更新 → 删除

**推荐工作流**：
```bash
# 1. 搜索技能
picoclaw skills search weather

# 2. 查看详情（可选）
picoclaw skills show weather

# 3. 安装技能
picoclaw skills install --registry clawhub weather

# 4. 配置 API 密钥（如需要）
vim ~/.picoclaw/config.json

# 5. 测试技能
picoclaw agent -m "What's the weather in Tokyo?"

# 6. 列出已安装
picoclaw skills list

# 7. 删除不需要的技能
picoclaw skills remove old-skill
```

---

## 📚 相关文件

- `cmd/picoclaw/internal/skills/command.go` - 命令定义
- `cmd/picoclaw/internal/skills/helpers.go` - 核心实现
- `pkg/skills/installer.go` - 安装器
- `pkg/skills/loader.go` - 加载器
- `pkg/skills/registry.go` - 注册表管理
- `pkg/utils/validation.go` - 输入验证

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
skills search (发现技能)
    ↓
skills install (安装技能)
    ↓
agent (使用技能)
    ↓
skills remove (清理)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
