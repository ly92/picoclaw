# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码仓库中工作时提供指导。

## 项目概览

PicoClaw 是一个用 Go 编写的超轻量级个人 AI 助手。它设计为在最小硬件上运行（<10MB RAM，$10 硬件），并作为一个支持多个 LLM 提供商和聊天渠道的 AI 代理框架。代码库通过 AI 辅助开发方式进行引导构建。

**核心理念**：最小化内存占用，最大化可移植性，尽可能只依赖 Go 标准库。

## 构建与测试

### 构建命令

```bash
# 为当前平台构建
make build

# 为所有支持的平台构建
make build-all

# 为特定平台构建
make build-linux-arm      # ARMv7（32位树莓派）
make build-linux-arm64    # ARM64
make build-pi-zero        # 为 Pi Zero 2 W 构建 32 位和 64 位版本

# 构建带原生 WhatsApp 支持的版本（二进制文件更大）
make build-whatsapp-native

# 运行代码生成（构建前必须执行）
make generate
```

### 测试与质量检查

```bash
# 运行所有测试
make test

# 运行特定测试
go test -run TestName -v ./pkg/session/

# 运行基准测试
go test -bench=. -benchmem -run='^$' ./...

# 代码质量检查
make fmt          # 格式化代码
make vet          # 静态分析
make lint         # 完整的代码检查（golangci-lint）
make fix          # 自动修复代码检查问题
make check        # 完整的提交前检查：deps + fmt + vet + test
```

### 开发工作流

```bash
# 安装依赖
make deps

# 更新依赖
make update-deps

# 将二进制文件安装到 ~/.local/bin
make install

# 清理构建产物
make clean
```

### Docker

```bash
# 构建最小化 Docker 镜像（基于 Alpine）
make docker-build
make docker-build-full        # 完整功能镜像（包含 Node.js 24）

# 运行网关
make docker-run
make docker-run-full

# 交互式运行代理
make docker-run-agent
make docker-run-agent-full

# 在 Docker 中测试 MCP 工具
make docker-test

# 清理 Docker 产物
make docker-clean
```

## 架构

### 核心组件

1. **Agent 系统** (`pkg/agent/`)
   - `instance.go`: AgentInstance - 完全配置的代理，包含工作空间、工具、会话
   - `loop.go`: 主代理执行循环 - 处理工具调用、LLM 交互
   - `context.go`: ContextBuilder - 从工作空间文件构建系统提示（AGENTS.md、IDENTITY.md 等）
   - `memory.go`: 长期记忆管理
   - `registry.go`: 多代理支持的代理注册表

2. **路由系统** (`pkg/routing/`)
   - 代理选择的 7 级优先级级联：peer > parent_peer > guild > team > account > channel_wildcard > default
   - 带身份链接的会话密钥构建
   - AgentID 规范化和解析

3. **Providers（提供商）** (`pkg/providers/`)
   - 统一的 LLM 提供商接口
   - `openai_compat/`: OpenAI 兼容提供商（OpenRouter、Groq、Zhipu、DeepSeek、Gemini、Moonshot、Qwen、NVIDIA、Ollama、LiteLLM、vLLM、Cerebras、GitHub Copilot）
   - `anthropic/`: 原生 Anthropic/Claude 集成
   - 以模型为中心的配置，支持回退和轮询负载均衡

4. **Channels（渠道）** (`pkg/channels/`)
   - 每个渠道是一个独立的包：`telegram/`、`discord/`、`whatsapp/`、`qq/`、`dingtalk/`、`feishu/`、`line/`、`slack/`、`wecom/`
   - 基于 webhook 的渠道共享 webhook 服务器（默认：127.0.0.1:18790）
   - 通过 whatsmeow 原生支持 WhatsApp（构建标签：`whatsapp_native`）

5. **Tools（工具）** (`pkg/tools/`)
   - 带工作空间沙盒的工具注册表
   - 文件操作：`read_file`、`write_file`、`edit_file`、`append_file`、`list_dir`
   - `exec`: 带安全防护的命令执行（阻止危险模式，限制路径）
   - `message`: 向用户发送消息
   - `spawn`: 为异步任务创建子代理
   - `cron`: 安排定期任务
   - `web_search`: 通过 Brave/Tavily/DuckDuckGo 进行网络搜索
   - `mcp_tool`: 模型上下文协议工具集成

6. **会话管理** (`pkg/session/`)
   - 基于 SQLite 的会话存储
   - 带令牌窗口管理的消息历史
   - 按代理/渠道/对等方隔离会话

7. **配置** (`pkg/config/`)
   - 基于 JSON 的配置，位于 `~/.picoclaw/config.json`
   - 环境变量覆盖：`PICOCLAW_CONFIG`、`PICOCLAW_HOME`
   - 统一提供商配置的模型列表
   - 用于路由的代理绑定

### 关键架构模式

**Agent 工作空间结构**：
```
~/.picoclaw/workspace/
├── sessions/          # 会话历史（SQLite）
├── memory/            # 长期记忆（MEMORY.md）
├── state/             # 持久化状态
├── cron/              # 定时任务数据库
├── skills/            # 自定义技能
├── AGENTS.md          # 代理行为指南
├── HEARTBEAT.md       # 定期任务提示（每 30 分钟）
├── IDENTITY.md        # 代理身份
├── SOUL.md            # 代理灵魂
├── TOOLS.md           # 工具描述
└── USER.md            # 用户偏好
```

**安全沙盒**：
- `restrict_to_workspace`: 限制文件/命令访问工作空间目录（默认：true）
- 路径白名单：`allow_read_paths`、`allow_write_paths` 支持 glob 模式
- 危险命令阻止：`rm -rf`、`format`、`dd if=`、fork 炸弹等
- 在主代理、子代理和心跳任务中保持一致

**模型配置**：
- 新的以模型为中心的方法：`model_list` 使用 `vendor/model` 格式
- 基于前缀的自动提供商路由（例如：`zhipu/glm-4.7`、`anthropic/claude-sonnet-4.6`）
- 回退支持以提高弹性
- 跨多个端点的轮询负载均衡
- 向后兼容旧的 `providers` 配置

**Heartbeat 系统**：
- 每 30 分钟读取一次 `HEARTBEAT.md`（可配置）
- 对长时间运行的任务使用 `spawn` 工具（非阻塞子代理）
- 子代理通过 `message` 工具直接通信

**路由优先级**：
1. 对等方绑定（特定渠道中的特定用户）
2. 父对等方绑定（线程父级）
3. 公会绑定（Discord 服务器）
4. 团队绑定（Slack 工作区）
5. 账户绑定（渠道中的机器人账户）
6. 渠道通配符（渠道中的任何用户）
7. 默认代理

## 配置

### 模型列表格式

```json
{
  "model_list": [
    {
      "model_name": "gpt4",
      "model": "openai/gpt-5.2",
      "api_key": "sk-...",
      "request_timeout": 300
    },
    {
      "model_name": "claude",
      "model": "anthropic/claude-sonnet-4.6",
      "api_key": "sk-ant-..."
    }
  ],
  "agents": {
    "defaults": {
      "model": "gpt4"
    }
  }
}
```

### 环境变量

- `PICOCLAW_CONFIG`: 覆盖配置文件路径
- `PICOCLAW_HOME`: 覆盖根目录（默认：`~/.picoclaw`）
- `PICOCLAW_BUILTIN_SKILLS`: 覆盖内置技能目录
- `PICOCLAW_AGENTS_DEFAULTS_RESTRICT_TO_WORKSPACE`: 安全开关
- `PICOCLAW_HEARTBEAT_ENABLED`: 启用/禁用心跳
- `PICOCLAW_HEARTBEAT_INTERVAL`: 心跳间隔（分钟）

## CLI 命令

```bash
picoclaw onboard              # 初始化配置和工作空间
picoclaw agent -m "message"   # 单次对话
picoclaw agent                # 交互式聊天模式
picoclaw gateway              # 启动网关（所有启用的渠道）
picoclaw status               # 显示状态
picoclaw cron list            # 列出定时任务
picoclaw cron add ...         # 添加定时任务
picoclaw auth login           # 使用提供商进行身份验证
picoclaw migrate              # 运行数据库迁移
picoclaw skills list          # 列出可用技能
picoclaw version              # 显示版本信息
```

## 代码组织

### 包结构

- `cmd/picoclaw/`: 主 CLI 入口点
  - `internal/`: 命令实现（agent、gateway、onboard 等）
- `pkg/`: 核心包
  - `agent/`: 代理编排
  - `channels/`: 聊天平台集成
  - `providers/`: LLM 提供商客户端
  - `tools/`: 代理工具/函数
  - `routing/`: 代理路由和会话管理
  - `config/`: 配置加载和验证
  - `session/`: 会话历史存储
  - `skills/`: 技能系统
  - `bus/`: 用于组件间通信的事件总线
  - `heartbeat/`: 定期任务执行
  - `mcp/`: 模型上下文协议集成
  - `auth/`: 身份验证管理
  - `logger/`: 结构化日志
  - `utils/`: 共享工具

### 构建标签

- `whatsapp_native`: 包含原生 WhatsApp 支持（增加二进制文件大小）
- `stdjson`: 使用标准库 JSON 而非更快的替代方案（Makefile 中的默认设置）

### 版本信息

版本信息通过 ldflags 在构建时嵌入：
- `version`: Git 标签或 "dev"
- `gitCommit`: 短提交哈希
- `buildTime`: 构建时间戳
- `goVersion`: Go 编译器版本

参见 `cmd/picoclaw/internal/version.go` 了解版本处理。

## 测试指南

- 将测试放在其测试的代码旁边（`*_test.go`）
- 对多种场景使用表驱动测试
- 模拟外部依赖（参见 `pkg/agent/mock_provider_test.go`）
- 测试错误路径，而不仅仅是正常路径
- 为性能关键代码编写基准测试

## 常见开发任务

### 添加新的 LLM 提供商

1. 确定是 OpenAI 兼容还是需要自定义协议
2. 如果是 OpenAI 兼容：只需添加到 `model_list`，并设置适当的 `api_base`
3. 如果是自定义协议：在 `pkg/providers/` 中创建新包，实现 `LLMProvider` 接口
4. 在提供商工厂中注册

### 添加新的渠道

1. 在 `pkg/channels/<name>/` 中创建包
2. 实现 `Channel` 接口
3. 在网关命令中注册
4. 将配置结构添加到 `pkg/config/config.go`
5. 在 README.md 中记录

### 添加新工具

1. 在 `pkg/tools/` 中创建工具文件
2. 实现 `Tool` 接口（Name、Description、InputSchema、Execute）
3. 在 `pkg/agent/instance.go` 工具注册表中注册
4. 如果访问文件系统，处理工作空间限制

## 重要说明

- 所有文件操作都应遵守工作空间边界
- 使用结构化日志（`pkg/logger`）而非 fmt.Print
- 配置更改应向后兼容
- 内存占用是主要关注点 - 避免大型依赖
- 二进制文件大小很重要 - 对可选功能使用构建标签
- 添加功能时在资源受限的硬件上测试
- 欢迎并鼓励 AI 辅助贡献
