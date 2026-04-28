# Eino Examples 运行指导

本文档旨在帮助你快速配置环境并运行本仓库中的示例代码。

## 前置要求

- **Go 1.24+**（本仓库使用 Go 1.24.7）
- **Git**
- **Docker**（仅 `quickstart/eino_assistant` 需要，用于启动 Redis）

## 环境变量总览

本仓库的示例使用不同的 LLM 提供商。大多数示例默认使用 **OpenAI 兼容 API**，部分示例使用 **火山引擎豆包（Ark）**。你只需配置你打算使用的提供商即可。

### 方式一：OpenAI 兼容 API（推荐新手使用）

大部分示例（ADK、Compose、Flow、Components、QuickStart/chat 等）默认使用 OpenAI 兼容接口。

```bash
export OPENAI_API_KEY="sk-xxx"          # 你的 API Key
export OPENAI_BASE_URL="https://api.openai.com/v1"  # API 地址，可替换为任何兼容 OpenAI 的服务
export OPENAI_MODEL_NAME="gpt-4o"       # 部分示例使用此变量
export OPENAI_MODEL="gpt-4o"            # 部分示例使用此变量
```

> **提示**：`OPENAI_MODEL_NAME` 和 `OPENAI_MODEL` 在不同示例中命名不一致，建议同时设置为相同值。

**使用第三方兼容服务**（如 DeepSeek、智谱、月之暗面等）只需修改 `OPENAI_BASE_URL` 和 `OPENAI_MODEL_NAME` 即可，例如：

```bash
export OPENAI_API_KEY="your-deepseek-key"
export OPENAI_BASE_URL="https://api.deepseek.com"
export OPENAI_MODEL_NAME="deepseek-chat"
export OPENAI_MODEL="deepseek-chat"
```

**使用 Azure OpenAI** 需额外设置：

```bash
export OPENAI_BY_AZURE="true"
```

### 方式二：火山引擎豆包（Ark）

ADK 示例支持通过 `MODEL_TYPE=ark` 切换到豆包模型。`quickstart/eino_assistant` 强制要求 Ark 配置。

```bash
export MODEL_TYPE="ark"                  # 切换 ADK 示例使用 Ark（不设置则默认用 OpenAI）
export ARK_API_KEY="xxx"                 # 火山引擎 API Key
export ARK_MODEL="ep-xxx"               # 模型接入点 ID
export ARK_BASE_URL=""                   # 可选，默认使用火山引擎官方地址
export ARK_REGION=""                     # 可选，区域设置
```

> **获取方式**：访问 [火山引擎方舟控制台](https://console.volcengine.com/ark/region:ark+cn-beijing/model)，选择模型后点击「推理」创建接入点，`ep-xxx` 即为模型 ID。

### 方式三：Ollama（本地模型）

部分示例支持 Ollama，适合无 API Key 的情况。

```bash
export OLLAMA_BASE_URL="http://localhost:11434"
export OLLAMA_MODEL="llama2"
export OLLAMA_MODEL_NAME="llama2"
```

`quickstart/chat` 中使用 Ollama 还需设置：

```bash
export EINO_CHAT_PROVIDER="ollama"
```

### 方式四：DeepSeek（独立配置）

`flow/agent/multiagent/plan_execute` 示例使用独立的 DeepSeek 配置：

```bash
export DEEPSEEK_API_KEY="xxx"
export DEEPSEEK_BASE_URL="https://api.deepseek.com"
export DEEPSEEK_MODEL_NAME="deepseek-chat"
```

---

## 可观测（可选）

以下环境变量用于链路追踪和可观测，不影响示例运行，可按需配置。

### CozeLoop

大量示例中集成了 CozeLoop 追踪，不配置不影响基本功能：

```bash
export COZELOOP_API_TOKEN="xxx"
export COZELOOP_WORKSPACE_ID="xxx"
```

### APMPlus（火山引擎）

```bash
export APMPLUS_APP_KEY="xxx"
export APMPLUS_REGION="xxx"       # 可选
```

### Langfuse

`flow/agent/manus` 和 `quickstart/eino_assistant` 支持 Langfuse：

```bash
export LANGFUSE_PUBLIC_KEY="xxx"
export LANGFUSE_SECRET_KEY="xxx"
export LANGFUSE_HOST="xxx"        # 可选，默认为 Langfuse Cloud
```

---

## 各示例运行方法

### 1. QuickStart/chat — 最简单的入门示例

```bash
# 设置环境变量后
cd quickstart/chat
go run .
```

### 2. ADK — Agent 开发套件

ADK 系列示例共用 `adk/common/model/chat_model.go` 创建模型，通过 `MODEL_TYPE` 切换提供商。

```bash
# 使用 OpenAI 兼容 API（默认）
export OPENAI_API_KEY="sk-xxx"
export OPENAI_MODEL="gpt-4o"
export OPENAI_BASE_URL="https://api.openai.com/v1"

# 或者使用 Ark
export MODEL_TYPE="ark"
export ARK_API_KEY="xxx"
export ARK_MODEL="ep-xxx"

# 运行 Hello World 示例
cd adk/helloworld
go run .
```

### 3. Compose — 编排示例

```bash
export OPENAI_API_KEY="sk-xxx"
export OPENAI_MODEL_NAME="gpt-4o"
export OPENAI_BASE_URL="https://api.openai.com/v1"

# 运行 Chain 示例
cd compose/chain
go run .

# 运行 Graph 示例
cd compose/graph/simple
go run .
```

### 4. QuickStart/eino_assistant — 完整 RAG 应用

此示例**强制要求**火山引擎 Ark 配置，并需要 Redis。

```bash
# 1. 启动 Redis（在 quickstart/eino_assistant 目录下）
cd quickstart/eino_assistant
docker-compose up -d

# 2. 设置必需环境变量
export ARK_API_KEY="xxx"
export ARK_CHAT_MODEL="ep-xxx"          # 推荐 Doubao-pro-4k
export ARK_EMBEDDING_MODEL="ep-xxx"     # 推荐 Doubao-embedding-large

# 3. 启动 Agent 服务
go run cmd/einoagent/main.go

# 4. 访问 http://127.0.0.1:8080/
```

### 5. Flow/Agent — 高级 Agent 示例

```bash
# ReAct Agent
cd flow/agent/react
export OPENAI_API_KEY="sk-xxx"
export OPENAI_MODEL_NAME="gpt-4o"
export OPENAI_BASE_URL="https://api.openai.com/v1"
go run .

# Manus Agent（独立 go.mod）
cd flow/agent/manus
go run .
```

### 6. DevOps — 调试与可视化

这些示例不需要真实 LLM 调用，主要展示调试和可视化功能：

```bash
cd devops/visualize
go run .
```

---

## 常见问题

### Q: `OPENAI_MODEL` 和 `OPENAI_MODEL_NAME` 有什么区别？

仓库中不同示例使用了不同的变量名，建议同时设置为相同值。`OPENAI_MODEL` 主要用于 ADK 和 flow 示例，`OPENAI_MODEL_NAME` 主要用于 compose 和 quickstart 示例。

### Q: 没有 OpenAI Key 怎么办？

有以下几种替代方案：
1. **使用国内兼容服务**：DeepSeek、智谱 AI、月之暗面等都提供 OpenAI 兼容 API，只需修改 `OPENAI_BASE_URL` 和模型名称
2. **使用火山引擎豆包**：设置 `MODEL_TYPE=ark` 并配置 `ARK_*` 相关变量
3. **使用 Ollama 本地模型**：安装 Ollama 后无需任何 API Key（注意：部分示例需要 Function Calling 能力，本地小模型可能不支持）

### Q: 示例运行报 `dial tcp: lookup api.openai.com: no such host` 怎么办？

国内网络可能无法直接访问 OpenAI API，建议：
1. 使用代理，或设置 `OPENAI_BASE_URL` 为代理地址
2. 切换为国内兼容的 API 服务

### Q: 部分示例需要额外的基础设施？

| 示例 | 依赖 |
|------|------|
| `quickstart/eino_assistant` | Redis（通过 docker-compose 启动） |
| `components/retriever/*` | VikingDB（需要 `VIKING_DB_*` 环境变量） |
| `adk/multiagent/deep` | Python 环境（`EXCEL_AGENT_PYTHON_EXECUTABLE_PATH`） |

---

## 快速上手推荐路径

如果你是第一次接触 Eino 框架，建议按以下顺序学习：

1. **`quickstart/chat`** — 理解基本的 LLM 调用
2. **`compose/chain`** — 学习 Prompt + Model 的链式编排
3. **`compose/graph/simple`** — 学习图编排
4. **`adk/helloworld`** — 使用 ADK 创建第一个 Agent
5. **`adk/intro/chatmodel`** — 了解 ChatModelAgent 和中断机制
6. **`flow/agent/react`** — 体验 ReAct Agent
7. **`quickstart/eino_assistant`** — 运行完整的 RAG 应用

---

## 环境变量速查表

| 变量名 | 用途 | 适用示例 |
|--------|------|---------|
| `OPENAI_API_KEY` | OpenAI 兼容 API Key | 大部分示例 |
| `OPENAI_BASE_URL` | API 地址 | 大部分示例 |
| `OPENAI_MODEL` / `OPENAI_MODEL_NAME` | 模型名称 | 大部分示例 |
| `OPENAI_BY_AZURE` | 启用 Azure OpenAI | 需要 Azure 的示例 |
| `MODEL_TYPE` | 模型提供商切换（`ark`） | ADK 示例 |
| `ARK_API_KEY` | 火山引擎 API Key | Ark 相关示例 |
| `ARK_MODEL` / `ARK_MODEL_NAME` | 豆包模型 ID | Ark 相关示例 |
| `ARK_BASE_URL` | Ark API 地址 | Ark 相关示例 |
| `ARK_CHAT_MODEL` | 豆包对话模型 | eino_assistant |
| `ARK_EMBEDDING_MODEL` | 豆包 Embedding 模型 | eino_assistant |
| `DEEPSEEK_API_KEY` | DeepSeek API Key | plan_execute |
| `DEEPSEEK_BASE_URL` | DeepSeek API 地址 | plan_execute |
| `DEEPSEEK_MODEL_NAME` | DeepSeek 模型名 | plan_execute |
| `OLLAMA_BASE_URL` | Ollama 地址 | Ollama 相关示例 |
| `OLLAMA_MODEL` / `OLLAMA_MODEL_NAME` | Ollama 模型名 | Ollama 相关示例 |
| `EINO_CHAT_PROVIDER` | 切换 chat 提供商 | quickstart/chat |
| `COZELOOP_API_TOKEN` | CozeLoop 追踪 Token | 可选，多数示例 |
| `COZELOOP_WORKSPACE_ID` | CozeLoop 工作区 ID | 可选，多数示例 |
| `APMPLUS_APP_KEY` | APMPlus Key | 可选，eino_assistant |
| `REDIS_ADDR` | Redis 地址 | eino_assistant |
