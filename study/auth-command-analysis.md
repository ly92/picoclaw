# NewAuthCommand 函数详细拆解

## 📍 函数位置

**文件**: `cmd/picoclaw/internal/auth/command.go`
**行数**: 5-22
**实现文件**: `cmd/picoclaw/internal/auth/helpers.go`
**子命令文件**: `login.go`, `logout.go`, `status.go`, `models.go`

---

## 🎯 核心功能

`NewAuthCommand` 是 PicoClaw 的身份验证管理命令，提供 OAuth 登录、令牌存储、凭证管理等功能。这是一个**父命令**，包含 4 个子命令。

**主要职责**：
1. OAuth 流程管理（浏览器授权 / 设备码授权）
2. 令牌安全存储（系统密钥链）
3. 多提供商支持（OpenAI、Anthropic、Google Antigravity）
4. 凭证状态查询
5. 可用模型列表获取

---

## 📊 结构拆解

### 1️⃣ **父命令定义** (command.go:5-22)

```go
func NewAuthCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "auth",
        Short: "Manage authentication (login, logout, status)",
        RunE: func(cmd *cobra.Command, _ []string) error {
            return cmd.Help()  // 不带子命令时显示帮助
        },
    }

    // 注册子命令
    cmd.AddCommand(
        newLoginCommand(),
        newLogoutCommand(),
        newStatusCommand(),
        newModelsCommand(),
    )

    return cmd
}
```

#### 特点：
- ✅ **父命令无直接功能**: 必须使用子命令
- ✅ **自动显示帮助**: 运行 `picoclaw auth` 显示子命令列表
- ✅ **子命令独立**: 每个子命令有独立的参数和逻辑

---

### 2️⃣ **子命令架构**

```
picoclaw auth
    ├─ login          # OAuth 登录或粘贴令牌
    ├─ logout         # 移除存储的凭证
    ├─ status         # 显示当前认证状态
    └─ models         # 显示可用模型（仅 Google Antigravity）
```

---

## 🔐 子命令详解

### **子命令 1: `login`** (login.go:5-25)

```go
func newLoginCommand() *cobra.Command {
    var (
        provider      string
        useDeviceCode bool
    )

    cmd := &cobra.Command{
        Use:   "login",
        Short: "Login via OAuth or paste token",
        Args:  cobra.NoArgs,
        RunE: func(cmd *cobra.Command, _ []string) error {
            return authLoginCmd(provider, useDeviceCode)
        },
    }

    cmd.Flags().StringVarP(&provider, "provider", "p", "", "Provider to login with")
    cmd.Flags().BoolVar(&useDeviceCode, "device-code", false, "Use device code flow")
    _ = cmd.MarkFlagRequired("provider")  // --provider 必须提供

    return cmd
}
```

#### 参数说明：

| 参数 | 短标志 | 类型 | 必需 | 说明 |
|------|--------|------|------|------|
| `--provider` | `-p` | string | ✅ | 提供商名称 (openai, anthropic, google-antigravity) |
| `--device-code` | 无 | bool | ❌ | 使用设备码流程（无头环境） |

#### 执行流程：

```
authLoginCmd(provider, useDeviceCode)
    ├─ provider == "openai"
    │   └─ authLoginOpenAI()
    │       ├─ useDeviceCode? → LoginDeviceCode()
    │       │                  → LoginBrowser()
    │       ├─ SetCredential("openai", cred)
    │       └─ 更新配置: auth_method = "oauth"
    │
    ├─ provider == "anthropic"
    │   └─ authLoginPasteToken()
    │       ├─ 提示用户粘贴 API 令牌
    │       ├─ SetCredential("anthropic", cred)
    │       └─ 更新配置: auth_method = "token"
    │
    └─ provider == "google-antigravity"
        └─ authLoginGoogleAntigravity()
            ├─ LoginBrowser() - OAuth 流程
            ├─ fetchGoogleUserEmail() - 获取邮箱
            ├─ FetchAntigravityProjectID() - 获取项目 ID
            ├─ SetCredential("google-antigravity", cred)
            └─ 更新配置: auth_method = "oauth"
```

---

### **子命令 2: `logout`** (logout.go:5-20)

```go
func newLogoutCommand() *cobra.Command {
    var provider string

    cmd := &cobra.Command{
        Use:   "logout",
        Short: "Remove stored credentials",
        Args:  cobra.NoArgs,
        RunE: func(cmd *cobra.Command, _ []string) error {
            return authLogoutCmd(provider)
        },
    }

    cmd.Flags().StringVarP(&provider, "provider", "p", "",
        "Provider to logout from; empty = all")

    return cmd
}
```

#### 参数说明：

| 参数 | 短标志 | 类型 | 必需 | 说明 |
|------|--------|------|------|------|
| `--provider` | `-p` | string | ❌ | 指定提供商；留空则登出所有 |

#### 执行逻辑：

```go
func authLogoutCmd(provider string) error {
    if provider != "" {
        // 登出单个提供商
        DeleteCredential(provider)
        // 清除配置中的 auth_method
        appCfg.ModelList[i].AuthMethod = ""
        fmt.Printf("Logged out from %s\n", provider)
    } else {
        // 登出所有提供商
        DeleteAllCredentials()
        // 清除所有 auth_method
        for i := range appCfg.ModelList {
            appCfg.ModelList[i].AuthMethod = ""
        }
        fmt.Println("Logged out from all providers")
    }
}
```

---

### **子命令 3: `status`** (status.go:5-16)

```go
func newStatusCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "status",
        Short: "Show current auth status",
        Args:  cobra.NoArgs,
        RunE: func(cmd *cobra.Command, _ []string) error {
            return authStatusCmd()
        },
    }
    return cmd
}
```

#### 执行逻辑：

```go
func authStatusCmd() error {
    store := LoadStore()  // 从密钥链加载凭证

    if len(store.Credentials) == 0 {
        fmt.Println("No authenticated providers.")
        return nil
    }

    // 遍历所有凭证
    for provider, cred := range store.Credentials {
        status := "active"
        if cred.IsExpired() {
            status = "expired"
        } else if cred.NeedsRefresh() {
            status = "needs refresh"
        }

        // 显示详细信息
        fmt.Printf("  %s:\n", provider)
        fmt.Printf("    Method: %s\n", cred.AuthMethod)
        fmt.Printf("    Status: %s\n", status)
        fmt.Printf("    Account: %s\n", cred.AccountID)
        fmt.Printf("    Email: %s\n", cred.Email)
        fmt.Printf("    Expires: %s\n", cred.ExpiresAt)
    }
}
```

---

### **子命令 4: `models`** (models.go:5-15)

```go
func newModelsCommand() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "models",
        Short: "Show available models",
        RunE: func(_ *cobra.Command, _ []string) error {
            return authModelsCmd()
        },
    }
    return cmd
}
```

#### 执行逻辑（仅支持 Google Antigravity）：

```go
func authModelsCmd() error {
    // 1. 获取凭证
    cred := GetCredential("google-antigravity")
    if cred == nil {
        return fmt.Errorf("not logged in")
    }

    // 2. 刷新令牌（如需要）
    if cred.NeedsRefresh() {
        cred = RefreshAccessToken(cred)
        SetCredential("google-antigravity", cred)
    }

    // 3. 获取模型列表
    models := FetchAntigravityModels(cred.AccessToken, cred.ProjectID)

    // 4. 显示模型
    for _, m := range models {
        status := "✓"
        if m.IsExhausted {
            status = "✗ (quota exhausted)"
        }
        fmt.Printf("  %s %s\n", status, m.DisplayName)
    }
}
```

---

## 🔄 OAuth 流程详解

### **1. 浏览器授权流程** (默认)

```
┌─────────────────────────────────────────────────┐
│           authLoginOpenAI() / Browser            │
└─────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 1. 启动本地 HTTP 服务器   │
        │    监听: http://localhost:8080
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 2. 打开浏览器              │
        │    跳转到: OAuth 授权页面  │
        │    https://api.openai.com/oauth/authorize
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 3. 用户授权                │
        │    浏览器重定向到:          │
        │    http://localhost:8080/callback?code=...
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 4. 本地服务器接收 code     │
        │    交换 access_token       │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 5. 存储到系统密钥链        │
        │    Keychain (macOS)        │
        │    SecretService (Linux)   │
        │    DPAPI (Windows)         │
        └───────────────────────────┘
```

---

### **2. 设备码授权流程** (无头环境)

```
┌─────────────────────────────────────────────────┐
│      authLoginOpenAI(useDeviceCode=true)        │
└─────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 1. 请求设备码              │
        │    POST /oauth/device      │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 2. 显示用户码              │
        │    "请访问: https://openai.com/activate" │
        │    "输入代码: ABCD-1234"   │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 3. 轮询授权状态            │
        │    每 5 秒查询一次         │
        │    POST /oauth/token       │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 4. 用户授权完成            │
        │    获得 access_token       │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 5. 存储到密钥链            │
        └───────────────────────────┘
```

---

### **3. 粘贴令牌流程** (Anthropic)

```
┌─────────────────────────────────────────────────┐
│          authLoginPasteToken("anthropic")       │
└─────────────────────────────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 1. 提示用户                │
        │    "Paste your API key:"   │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 2. 从标准输入读取          │
        │    token := os.Stdin       │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 3. 验证格式（可选）        │
        │    检查: sk-ant-...        │
        └───────────────────────────┘
                    ↓
        ┌───────────────────────────┐
        │ 4. 存储到密钥链            │
        │    SetCredential()         │
        └───────────────────────────┘
```

---

## 📦 使用示例

### **1. OpenAI OAuth 登录（浏览器）**
```bash
picoclaw auth login --provider openai

# 输出:
# Opening browser for authentication...
# Waiting for authorization...
# ✓ Login successful!
# Account: user-abc123
# Default model set to: gpt-5.2
```

### **2. OpenAI 设备码登录（无头环境）**
```bash
picoclaw auth login --provider openai --device-code

# 输出:
# Please visit: https://api.openai.com/activate
# Enter code: ABCD-1234
#
# Waiting for authorization... (press Ctrl+C to cancel)
# ✓ Login successful!
# Account: user-abc123
# Default model set to: gpt-5.2
```

### **3. Anthropic 令牌登录**
```bash
picoclaw auth login --provider anthropic

# 输出:
# Paste your Anthropic API key (sk-ant-...):
# [用户输入令牌]
# ✓ Token saved for anthropic!
# Default model set to: claude-sonnet-4.6
```

### **4. Google Antigravity OAuth 登录**
```bash
picoclaw auth login --provider google-antigravity

# 输出:
# Opening browser for authentication...
# Email: user@example.com
# Project: my-cloud-project-123
# ✓ Google Antigravity login successful!
# Default model set to: gemini-flash
# Try it: picoclaw agent -m "Hello world"
```

### **5. 查看认证状态**
```bash
picoclaw auth status

# 输出:
# Authenticated Providers:
# ------------------------
#   openai:
#     Method: oauth
#     Status: active
#     Account: user-abc123
#     Expires: 2026-04-04 15:30
#
#   anthropic:
#     Method: token
#     Status: active
```

### **6. 查看可用模型（Google Antigravity）**
```bash
picoclaw auth models

# 输出:
# Fetching models for project: my-cloud-project-123
#
# Available Antigravity Models:
# -----------------------------
#   ✓ gemini-3-flash (Gemini 3 Flash)
#   ✓ gemini-3-pro (Gemini 3 Pro)
#   ✗ gemini-3-ultra (quota exhausted)
```

### **7. 登出单个提供商**
```bash
picoclaw auth logout --provider openai

# 输出:
# Logged out from openai
```

### **8. 登出所有提供商**
```bash
picoclaw auth logout

# 输出:
# Logged out from all providers
```

---

## 🛡️ 安全机制

### **1. 凭证存储位置**

#### macOS
```
系统密钥链（Keychain）
- 位置: /Users/username/Library/Keychains/login.keychain
- 服务名: picoclaw-auth
- 账户名: <provider>
```

#### Linux
```
Secret Service API
- 使用: GNOME Keyring / KWallet
- DBus 接口: org.freedesktop.secrets
```

#### Windows
```
DPAPI (Data Protection API)
- 用户级加密
- 存储位置: %APPDATA%\picoclaw\auth
```

---

### **2. 凭证结构**

```go
type AuthCredential struct {
    Provider     string    // "openai" / "anthropic" / "google-antigravity"
    AuthMethod   string    // "oauth" / "token"
    AccessToken  string    // OAuth: access_token; Token: API key
    RefreshToken string    // OAuth 刷新令牌（可选）
    ExpiresAt    time.Time // 过期时间
    AccountID    string    // 账户 ID
    Email        string    // 邮箱地址
    ProjectID    string    // GCP 项目 ID（Antigravity）
}
```

---

### **3. 令牌刷新机制**

```go
if cred.NeedsRefresh() && cred.RefreshToken != "" {
    // 令牌即将过期（提前 5 分钟刷新）
    newCred := RefreshAccessToken(cred, oauthConfig)
    SetCredential(provider, newCred)
}
```

**刷新条件**：
- 距离过期时间 < 5 分钟
- 存在有效的 `refresh_token`

---

## 🔑 关键设计要点

### **1. 支持的认证方法**

| 提供商 | OAuth 浏览器 | OAuth 设备码 | 粘贴令牌 |
|--------|-------------|-------------|---------|
| OpenAI | ✅ | ✅ | ✅ |
| Anthropic | ❌ | ❌ | ✅ |
| Google Antigravity | ✅ | ❌ | ❌ |

---

### **2. 自动配置更新**

登录成功后，自动更新 `config.json`：

```json
{
  "model_list": [
    {
      "model_name": "gpt-5.2",
      "model": "openai/gpt-5.2",
      "auth_method": "oauth"  // ← 自动添加
    }
  ],
  "agents": {
    "defaults": {
      "model": "gpt-5.2"  // ← 自动设置默认模型
    }
  }
}
```

---

### **3. 错误处理**

```go
// 未登录时尝试使用
picoclaw agent -m "Hello"
// Error: authentication required for openai
// Run: picoclaw auth login --provider openai

// 令牌过期
picoclaw agent -m "Hello"
// Error: access token expired
// Run: picoclaw auth login --provider openai
```

---

### **4. 向后兼容**

同时更新两种配置格式：

```go
// 新格式：model_list
appCfg.ModelList[i].AuthMethod = "oauth"

// 旧格式：providers（向后兼容）
appCfg.Providers.OpenAI.AuthMethod = "oauth"
```

---

## 🔗 相关函数调用链

```
NewAuthCommand()
    ├─ newLoginCommand()
    │   └─ authLoginCmd(provider, useDeviceCode)
    │       ├─ authLoginOpenAI()
    │       │   ├─ auth.LoginBrowser() / LoginDeviceCode()
    │       │   ├─ auth.SetCredential()
    │       │   └─ config.SaveConfig()
    │       │
    │       ├─ authLoginPasteToken()
    │       │   ├─ auth.LoginPasteToken(os.Stdin)
    │       │   ├─ auth.SetCredential()
    │       │   └─ config.SaveConfig()
    │       │
    │       └─ authLoginGoogleAntigravity()
    │           ├─ auth.LoginBrowser()
    │           ├─ fetchGoogleUserEmail()
    │           ├─ providers.FetchAntigravityProjectID()
    │           ├─ auth.SetCredential()
    │           └─ config.SaveConfig()
    │
    ├─ newLogoutCommand()
    │   └─ authLogoutCmd(provider)
    │       ├─ auth.DeleteCredential() / DeleteAllCredentials()
    │       └─ config.SaveConfig()
    │
    ├─ newStatusCommand()
    │   └─ authStatusCmd()
    │       └─ auth.LoadStore()
    │
    └─ newModelsCommand()
        └─ authModelsCmd()
            ├─ auth.GetCredential()
            ├─ auth.RefreshAccessToken()
            └─ providers.FetchAntigravityModels()
```

---

## ⚠️ 注意事项

### **1. 首次使用流程**
```bash
# 正确顺序
picoclaw onboard                          # 1. 初始化
picoclaw auth login --provider openai     # 2. 登录
picoclaw agent -m "Hello"                 # 3. 使用
```

### **2. 设备码流程适用场景**
- SSH 远程服务器
- Docker 容器
- CI/CD 环境
- 无法打开浏览器的环境

### **3. 令牌安全**
```bash
# ❌ 错误：不要在命令行传递令牌
picoclaw auth login --provider anthropic --token sk-ant-...

# ✅ 正确：通过交互式输入
picoclaw auth login --provider anthropic
# Paste your API key: [安全输入]
```

### **4. 多账户支持**
```bash
# 当前不支持同一提供商的多账户
# 后续登录会覆盖前一个账户
```

---

## 🎓 总结

`NewAuthCommand` 是 PicoClaw 的**身份验证中心**，它：

1. ✅ **多种认证方式**: OAuth（浏览器/设备码）+ API 令牌
2. ✅ **安全存储**: 系统级密钥链保护
3. ✅ **自动刷新**: OAuth 令牌自动续期
4. ✅ **配置集成**: 登录后自动更新配置
5. ✅ **状态透明**: 清晰显示认证状态和过期时间
6. ✅ **多提供商**: OpenAI、Anthropic、Google Antigravity

**推荐工作流**：
```bash
# OpenAI (OAuth - 推荐)
picoclaw auth login --provider openai

# Anthropic (令牌)
picoclaw auth login --provider anthropic

# 查看状态
picoclaw auth status

# 登出（必要时）
picoclaw auth logout --provider openai
```

---

## 📚 相关文件

- `cmd/picoclaw/internal/auth/command.go` - 父命令定义
- `cmd/picoclaw/internal/auth/login.go` - 登录子命令
- `cmd/picoclaw/internal/auth/logout.go` - 登出子命令
- `cmd/picoclaw/internal/auth/status.go` - 状态子命令
- `cmd/picoclaw/internal/auth/models.go` - 模型列表子命令
- `cmd/picoclaw/internal/auth/helpers.go` - 核心实现逻辑
- `pkg/auth/` - 认证基础设施
- `pkg/providers/` - 提供商 OAuth 配置

---

## 🔍 与其他命令的关系

```
onboard (初始化)
    ↓
auth login (认证)
    ↓
agent / gateway (使用)
```

---

**生成时间**: 2026-03-04
**文档版本**: 1.0
**基于代码版本**: main (最新提交: bea238c)
