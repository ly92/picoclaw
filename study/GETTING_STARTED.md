# PicoClaw 学习与配置指南

本指南帮助你快速了解和配置 PicoClaw 项目。

---

## 📚 第一部分：推荐阅读顺序

### 1️⃣ 先从文档开始（5-10分钟）

**必读文档（按顺序）：**
1. `README.zh.md` 或 `README.md` - 了解项目目标、特性、快速开始
2. `CLAUDE.md` - 了解整体架构和代码组织
3. `CONTRIBUTING.md` - 了解开发流程和规范

### 2️⃣ 理解核心架构（按这个顺序看代码）

#### **第一层：入口和配置**
```
cmd/picoclaw/main.go              # CLI 入口，查看所有可用命令
pkg/config/config.go              # 配置结构，理解配置项含义
```

#### **第二层：核心流程**
```
pkg/agent/instance.go             # Agent 实例创建和配置
pkg/agent/loop.go                 # Agent 执行循环（核心！）
pkg/agent/context.go              # 上下文构建（系统提示）
```

#### **第三层：关键子系统**
```
pkg/routing/route.go              # 路由系统（7级优先级）
pkg/providers/                    # LLM 提供商接口
  ├── openai_compat/              # OpenAI 兼容实现
  └── anthropic/                  # Anthropic 原生实现
pkg/tools/                        # 工具系统
  ├── registry.go                 # 工具注册
  ├── filesystem.go               # 文件操作工具
  └── shell.go                    # 命令执行工具
```

#### **第四层：扩展功能**
```
pkg/channels/                     # 聊天渠道集成
pkg/session/                      # 会话管理
pkg/heartbeat/                    # 定期任务
pkg/skills/                       # 技能系统
```

### 3️⃣ 核心概念理解

**PicoClaw 工作流程：**
```
用户消息
   ↓
Routing（路由）→ 选择 Agent
   ↓
AgentInstance
   ↓
Loop（循环）
   ├─→ ContextBuilder → 构建提示
   ├─→ Provider → 调用 LLM
   ├─→ Tool Execution → 执行工具
   └─→ 迭代直到完成
   ↓
返回响应
```

**三个核心特点：**

1. **🪶 超轻量：<10MB 内存占用**
   - 为什么？看 `pkg/agent/loop.go` 的精简实现
   - 如何做到？避免大型依赖，使用 Go 原生库

2. **🔌 高度可扩展：多提供商、多渠道**
   - 看 `pkg/providers/` 的统一接口设计
   - 看 `pkg/channels/` 的渠道抽象

3. **🔒 安全沙盒：工作空间隔离**
   - 看 `pkg/tools/` 的路径限制实现
   - 看 config 中的 `restrict_to_workspace` 配置

### 4️⃣ 动手实践建议

#### **简单任务（了解基本流程）：**
1. 添加一个简单的工具（比如获取当前时间）
2. 修改系统提示（修改 workspace 中的 AGENTS.md）
3. 测试不同的 LLM 提供商

#### **中等任务（理解架构）：**
1. 添加一个新的聊天渠道
2. 实现一个自定义技能
3. 理解会话管理机制

#### **进阶任务（深入核心）：**
1. 理解工具调用的完整生命周期
2. 研究安全沙盒实现
3. 优化内存占用

### 5️⃣ 根据目标选择学习路径

- **想快速上手使用**：直接看下面的「第二部分：配置流程」→ 运行
- **想理解架构**：先看 `CLAUDE.md` → 再看 `pkg/agent/loop.go`（核心执行循环）
- **想贡献代码**：看 `CONTRIBUTING.md` → 选一个感兴趣的 Issue → 读相关代码
- **想学习 AI Agent 开发**：重点看 `pkg/agent/` 和 `pkg/tools/` 的设计模式

---

## 🚀 第二部分：启动和配置流程

### 前置要求

- macOS / Linux / Windows
- Go 1.25+ （如果从源码构建）
- Make 工具

### 步骤 1：构建项目

```bash
# 克隆仓库（如果还没有）
git clone https://github.com/sipeed/picoclaw.git
cd picoclaw

# 构建项目
make build

# 查看构建产物
ls -lh build/
```

构建成功后会生成：
- `build/picoclaw-<platform>-<arch>` - 对应平台的二进制文件
- `build/picoclaw` - 符号链接指向当前平台的二进制

### 步骤 2：初始化配置

```bash
./build/picoclaw onboard
```

这会创建：
- `~/.picoclaw/config.json` - 配置文件
- `~/.picoclaw/workspace/` - 工作空间目录

### 步骤 3：配置 LLM 提供商

#### 选项 A：使用 DeepSeek（推荐，中文友好）

1. **获取 API Key**
   - 访问 https://platform.deepseek.com
   - 注册/登录账号
   - 在控制台获取 API key（格式：`sk-...`）

2. **编辑配置文件**
   ```bash
   vim ~/.picoclaw/config.json
   ```

3. **修改以下配置**
   ```json
   {
     "agents": {
       "defaults": {
         "model": "deepseek-chat",
         "max_tokens": 8192,
         "max_tool_iterations": 50
       }
     },
     "model_list": [
       {
         "model_name": "deepseek-chat",
         "model": "deepseek/deepseek-chat",
         "api_base": "https://api.deepseek.com/v1",
         "api_key": "sk-your-api-key-here"
       }
     ]
   }
   ```

   ⚠️ **重要**：DeepSeek 的 `max_tokens` 限制为 8192，不要设置更大的值！

#### 选项 B：使用 OpenAI GPT

```json
{
  "agents": {
    "defaults": {
      "model": "gpt-4"
    }
  },
  "model_list": [
    {
      "model_name": "gpt-4",
      "model": "openai/gpt-4",
      "api_key": "sk-your-openai-key"
    }
  ]
}
```

#### 选项 C：使用 Anthropic Claude

```json
{
  "agents": {
    "defaults": {
      "model": "claude"
    }
  },
  "model_list": [
    {
      "model_name": "claude",
      "model": "anthropic/claude-sonnet-4.6",
      "api_key": "sk-ant-your-key"
    }
  ]
}
```

#### 选项 D：使用本地 Ollama（免费）

```bash
# 安装 Ollama
brew install ollama

# 启动 Ollama
ollama serve &

# 下载模型
ollama pull llama3
```

配置：
```json
{
  "agents": {
    "defaults": {
      "model": "llama3"
    }
  },
  "model_list": [
    {
      "model_name": "llama3",
      "model": "ollama/llama3",
      "api_base": "http://localhost:11434/v1"
    }
  ]
}
```

### 步骤 4：检查配置状态

```bash
./build/picoclaw status
```

输出示例：
```
🦞 picoclaw Status
Version: bea238c (git: bea238c3)
Config: /Users/you/.picoclaw/config.json ✓
Workspace: /Users/you/.picoclaw/workspace ✓
Model: deepseek-chat
```

### 步骤 5：测试运行

#### 测试 1：基本对话

```bash
./build/picoclaw agent -m "你好，介绍一下你自己"
```

成功标志：收到 AI 的回复

#### 测试 2：工具调用

```bash
./build/picoclaw agent -m "创建一个文件 hello.txt，内容是 Hello PicoClaw"
```

成功标志：
- AI 调用 `write_file` 工具
- 文件被创建在 `~/.picoclaw/workspace/hello.txt`

#### 测试 3：交互式对话

```bash
./build/picoclaw agent
```

进入交互模式，可以持续对话：
```
🦞 > 你好
AI: 你好！...

🦞 > 列出当前目录的文件
AI: [使用 list_dir 工具] ...

🦞 > exit
```

### 步骤 6：配置可选功能

#### A. 启用网络搜索

编辑配置文件：
```json
{
  "tools": {
    "web": {
      "duckduckgo": {
        "enabled": true,
        "max_results": 5
      },
      "brave": {
        "enabled": false,
        "api_key": "YOUR_BRAVE_API_KEY",
        "max_results": 5
      }
    }
  }
}
```

测试：
```bash
./build/picoclaw agent -m "搜索最新的 Go 1.23 新特性"
```

#### B. 配置 Telegram 机器人

1. 获取 Bot Token：
   - 在 Telegram 中找到 `@BotFather`
   - 发送 `/newbot` 创建机器人
   - 复制 token

2. 获取你的 User ID：
   - 找到 `@userinfobot`
   - 发送任意消息获取 ID

3. 配置：
   ```json
   {
     "channels": {
       "telegram": {
         "enabled": true,
         "token": "YOUR_BOT_TOKEN",
         "allow_from": ["YOUR_USER_ID"]
       }
     }
   }
   ```

4. 启动网关：
   ```bash
   ./build/picoclaw gateway
   ```

5. 在 Telegram 中向你的机器人发消息测试

#### C. 配置 Discord 机器人

1. 创建应用：https://discord.com/developers/applications
2. 创建 Bot 并获取 token
3. 启用 MESSAGE CONTENT INTENT
4. 配置：
   ```json
   {
     "channels": {
       "discord": {
         "enabled": true,
         "token": "YOUR_BOT_TOKEN",
         "allow_from": ["YOUR_USER_ID"]
       }
     }
   }
   ```

5. 邀请机器人到服务器并启动网关

---

## 📝 第三部分：常用命令参考

### 基本命令

```bash
# 初始化配置
picoclaw onboard

# 单次对话
picoclaw agent -m "你的问题"

# 交互式对话
picoclaw agent

# 指定模型
picoclaw agent --model deepseek-chat -m "问题"

# 指定会话
picoclaw agent --session my-session -m "问题"

# 启动网关（所有启用的渠道）
picoclaw gateway

# 查看状态
picoclaw status

# 查看版本
picoclaw version
```

### 定时任务

```bash
# 列出所有定时任务
picoclaw cron list

# 添加定时任务
picoclaw cron add "0 9 * * *" "每天早上9点提醒我查看邮件"

# 删除定时任务
picoclaw cron delete <job-id>
```

### 技能管理

```bash
# 列出可用技能
picoclaw skills list

# 安装技能
picoclaw skills install <skill-name>
```

### 身份验证

```bash
# OAuth 登录（Anthropic, Gemini）
picoclaw auth login --provider anthropic
```

---

## 🛠️ 第四部分：常见问题排查

### 问题 1：API 调用失败

**错误信息**：`API request failed: Status: 401`

**解决方案**：
1. 检查 API key 是否正确
2. 检查 API key 是否有效（未过期）
3. 检查 `api_base` 是否正确

### 问题 2：max_tokens 错误

**错误信息**：`Invalid max_tokens value, the valid range of max_tokens is [1, 8192]`

**解决方案**：
不同模型有不同的 token 限制：
- DeepSeek: 8192
- GPT-4: 通常 4096-8192
- Claude: 取决于具体模型

修改配置中的 `agents.defaults.max_tokens`

### 问题 3：工具调用权限错误

**错误信息**：`Command blocked by safety guard (path outside working dir)`

**原因**：安全沙盒限制

**解决方案**：
1. 检查 `restrict_to_workspace` 设置
2. 使用工作空间内的路径：`~/.picoclaw/workspace/`
3. 或者配置 `allow_read_paths` / `allow_write_paths`

### 问题 4：构建失败

**错误信息**：`go: module not found`

**解决方案**：
```bash
# 清理并重新下载依赖
make clean
make deps
make build
```

### 问题 5：内存不足

**现象**：程序崩溃或变慢

**解决方案**：
1. 减少 `max_tokens`
2. 减少 `max_tool_iterations`
3. 清理旧会话：`rm -rf ~/.picoclaw/workspace/sessions/*`

---

## 🎯 第五部分：进阶配置

### 多模型配置

同时配置多个模型，按需切换：

```json
{
  "model_list": [
    {
      "model_name": "deepseek",
      "model": "deepseek/deepseek-chat",
      "api_key": "sk-..."
    },
    {
      "model_name": "gpt4",
      "model": "openai/gpt-4",
      "api_key": "sk-..."
    },
    {
      "model_name": "claude",
      "model": "anthropic/claude-sonnet-4.6",
      "api_key": "sk-ant-..."
    }
  ],
  "agents": {
    "defaults": {
      "model": "deepseek"  // 默认使用 deepseek
    }
  }
}
```

使用时：
```bash
# 使用默认模型
picoclaw agent -m "问题"

# 使用指定模型
picoclaw agent --model gpt4 -m "问题"
picoclaw agent --model claude -m "问题"
```

### 多 Agent 配置

为不同场景配置不同的 Agent：

```json
{
  "agents": {
    "defaults": {
      "model": "deepseek-chat"
    },
    "list": [
      {
        "id": "coder",
        "name": "编程助手",
        "model": {"primary": "gpt4"},
        "skills": ["github", "code-review"]
      },
      {
        "id": "writer",
        "name": "写作助手",
        "model": {"primary": "claude"},
        "skills": ["writing", "translation"]
      }
    ]
  }
}
```

### 工作空间自定义

编辑工作空间文件来定制 Agent 行为：

```bash
# Agent 身份
vim ~/.picoclaw/workspace/IDENTITY.md

# Agent 行为指南
vim ~/.picoclaw/workspace/AGENTS.md

# 用户偏好
vim ~/.picoclaw/workspace/USER.md

# 定期任务
vim ~/.picoclaw/workspace/HEARTBEAT.md
```

---

## 📚 第六部分：参考资源

### 官方文档
- GitHub: https://github.com/sipeed/picoclaw
- 官网: https://picoclaw.io

### 学习资源
- `README.md` - 项目介绍
- `CLAUDE.md` - 架构文档
- `CONTRIBUTING.md` - 贡献指南
- `ROADMAP.md` - 路线图

### 社区
- Discord: https://discord.gg/V4sAZ9XWpN
- WeChat: 见 README 中的二维码

---

## 🎓 总结

完成以上步骤后，你应该已经：
- ✅ 理解 PicoClaw 的核心架构
- ✅ 成功构建并运行 PicoClaw
- ✅ 配置好至少一个 LLM 提供商
- ✅ 测试了基本对话和工具调用
- ✅ 了解如何配置可选功能

**下一步建议**：
1. 探索更多工具和技能
2. 尝试配置聊天渠道（Telegram/Discord）
3. 阅读源码，理解实现细节
4. 参与社区，贡献代码

祝你使用愉快！ 🦐
