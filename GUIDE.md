# Eino 框架企业级 AI 应用开发教程

## 课程总目标

本教程说明如何基于 **Eino** 将大模型能力接入 Go 后端，并与 **Gin**、**MySQL**、**对象存储（如阿里云 OSS）** 与**向量数据库**等组件协同工作，形成从网关、业务数据到 AI 编排的完整技术链路。内容涵盖底层机制、分层架构、RAG 与 Agent 等主题，每个知识点均以「解决什么问题」为切入点，并指向仓库中可运行的代码示例，便于对照实践。

---

## 第一章：为什么要用 Eino？

### 1.1 传统 Go 后端遇到了什么新问题？

传统后端聚焦 HTTP 收发、参数校验、MySQL 增删改查、鉴权限流，擅长处理**结构化数据**（学号、姓名、成绩、课程 ID），但面对以下需求无能为力：

- 用户用自然语言提问：「查一下张三的成绩」「帮我批量统计平均分」
- 系统需要自动判断意图、分步执行业务（先查 ID 再改数据）
- 需要理解非结构化文本（简历、文档）并做语义匹配

**AI 后端的增量能力**正是为了解决这些问题：大模型会话、文档解析、向量化、RAG 知识库检索、Agent 工具调度、工作流编排——让后端能「听懂」自然语言并自动执行业务，不再依赖大量 if/else 路由判断。

### 1.2 为什么不直接裸调大模型 API？

直接调用大模型 HTTP 接口看似简单，但在生产环境中会遇到一系列工程难题：

| 问题 | 裸调的困境 | 框架的解法 |
|------|-----------|-----------|
| 文档解析 | 无法直接读取 PDF/Word/OSS 文件，自研清洗成本高 | Loader 组件统一加载与清洗 |
| 会话管理 | 手动拼接历史、截断上下文，高并发下易串扰 | 内置上下文管理与多用户隔离 |
| 工具调度 | 无法自动识别意图、联动数据库，多步流程靠手写 | Agent + Tool 自动路由 |
| 向量检索 | 自行对接多款向量库，兼容性差 | VectorStore 统一封装 |

**Eino 的定位**：覆盖 AI 与后端业务联动的全链路编排需求，让你专注业务逻辑而非基础设施。

### 1.3 Eino 是什么？

- **定位**：CloudWeGo 生态下的开源 AI 应用开发框架，面向 Go，适用于私有化与公有云部署。
- **解决的问题**：屏蔽多厂商 LLM 接口差异，统一封装 RAG、Agent 调度、Graph/Workflow 等能力，降低 AI 工程化接入成本。
- **技术边界**：轻量 AI 编排中间件，**不内置** HTTP 服务、数据库连接池、云存储 SDK——与业务后端解耦，生产环境常与 Gin/Hertz、MySQL、OSS/COS、向量库组合使用。

### 1.4 企业级 AI 系统总架构

前端交互界面 → **OSS/COS**（归档原始文件）→ **Gin 网关**（鉴权、限流、校验、路由）→ **Eino AI 编排层**（文档解析、向量、RAG、Agent、工作流）→ **底层数据与算力**（大模型 + 向量库 + MySQL）。分层解耦，职责单一。

### 1.5 第一个 Eino 入门 Demo

调用大模型完成一次最小请求，确认开发环境、网络与 API 密钥配置正确。建议后续章节前先完成本示例。

> **可运行代码**：`quickstart/chat/openai.go`

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/components/llm/openai"
)

func main() {
	ctx := context.Background()

	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:  "你的大模型APIKey",
		BaseURL: "你的代理地址",
		Model:   "gpt-3.5-turbo",
	})
	if err != nil {
		log.Fatal("模型初始化失败：", err)
	}

	resp, err := chatModel.Generate(ctx, openai.WithPrompt("你是后端AI助手，简单介绍一下Eino框架"))
	if err != nil {
		log.Fatal("调用模型失败：", err)
	}

	fmt.Println("AI回答：", resp.Content)
}
```

> 更多入门示例见 `quickstart/chat/` 目录，包含流式输出（`stream.go`）、模板对话（`template.go`）等变体。

---

## 第二章：AI 基础概念

### 2.1 Embedding 向量——让机器「读懂」文字

**解决的问题**：机器只能运算数字，无法直接理解自然语言。Embedding 将文本映射为固定维度的浮点向量，语义越相近的文本在向量空间中距离越近，这是智能检索、推荐、问答的底层基石。

简单理解：把「文字」变成「一串数字」，让计算机能够计算两段文字之间的相似度。

### 2.2 Retriever 检索器——从海量文档中精准定位

**解决的问题**：MySQL 的 `LIKE` 只能做字面匹配，无法处理同义词、口语化表达等语义模糊场景（例如「会后端开发」匹配不到「熟练掌握 Go 后端服务搭建」）。

Retriever 在向量库中**毫秒级**召回 Top-N 语义最相关的文档片段，实现真正的「理解式搜索」。

> 检索器示例见 `components/retriever/`，其中 `multiquery/` 演示了多路查询提升召回率的用法。

### 2.3 RAG 检索增强生成——让大模型基于事实回答

**解决的问题**：大模型的训练数据有截止日期，且不了解你的私有业务数据，直接提问容易产生「幻觉」（编造事实）。

RAG 的核心思路：**先查资料，再回答问题**。

1. 私有文档/业务数据 → 向量化后存入向量库
2. 用户提问 → 检索语义相似的资料片段
3. 将资料拼入 Prompt → 大模型基于真实数据生成回答

这样可以有效抑制幻觉、避免泄露未授权数据，并解决公网知识滞后的问题。

> 完整 RAG 链路示例见 `quickstart/eino_assistant/eino/knowledgeindexing/`（入库）与 `quickstart/eino_assistant/eino/einoagent/`（检索问答）。

### 2.4 向量库与 MySQL 各自的职责

两者不是替代关系，而是互补：

| 维度 | MySQL | 向量库（如 Milvus） |
|------|-------|-------------------|
| 存什么 | 结构化字段：学号、姓名、成绩 | 文本片段、浮点向量、元数据 |
| 擅长什么 | 精准查询、事务、关联计算 | 语义相似检索 |
| 不适合 | 超长全文、大文件、向量 | 传统事务、账号体系 |

### 2.5 文本向量化示例

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/components/embedding/openai"
)

func main() {
	ctx := context.Background()

	embedder, err := openai.NewEmbedding(ctx, &openai.EmbeddingConfig{
		APIKey:  "你的APIKey",
		BaseURL: "你的地址",
	})
	if err != nil {
		log.Fatal(err)
	}

	texts := []string{
		"张三GPA成绩优秀",
		"李四多门课程挂科，需要学业预警",
		"今天食堂饭菜很好吃",
	}

	vectors, err := embedder.EmbedTexts(ctx, texts)
	if err != nil {
		log.Fatal(err)
	}

	for i, vec := range vectors {
		fmt.Printf("第%d条文本向量长度：%d\n", i+1, len(vec))
	}
}
```

**注意**：入库片段与用户提问须用**同一厂商、同版本** Embedding 模型，否则维度不一致会导致检索失效。

---

## 第三章：Eino 核心组件精讲

每个组件的介绍围绕三个问题展开：**为什么需要它？它解决什么问题？去哪里看代码？**

### 3.1 Loader 文档加载器

**为什么需要**：AI 系统要处理的数据源五花八门——本地 TXT/PDF/DOCX、公网 URL、OSS 签名链接。如果每种格式都自己写解析逻辑，工作量巨大且难以维护。

**解决的问题**：统一读取各类数据源，输出 Eino 通用 `Document` 结构，后续组件无需关心原始格式差异。只负责拉取、解析、清洗，**不做**向量化和检索。

**生产建议**：优先 URLLoader + OSS 地址，避免本地磁盘 IO 和多节点不同步。

> 文档解析示例见 `components/document/parser/`，包含自定义解析器（`customparser/`）、扩展解析器（`extparser/`）等实现。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/components/document/loader"
)

func main() {
	ctx := context.Background()

	l := loader.NewURLLoader("https://example.com/student-rule.txt")
	doc, err := l.Load(ctx)
	if err != nil {
		log.Fatal("加载文档失败：", err)
	}

	fmt.Println("文档内容预览：", string(doc.Content[:100]))
}
```

### 3.2 Splitter 文档切分器

**为什么需要**：大模型有 Token 上限（如 4K/8K/128K），一份完整文档动辄数万字，直接送入会被截断或超限。同时，过长的文本语义混杂，检索精度也会下降。

**解决的问题**：将长文档按策略切分为短片段，每段语义更纯粹，检索命中率更高。`ChunkOverlap` 参数确保相邻片段有重叠，避免关键句被截断。

```go
package main

import (
	"context"
	"fmt"

	"github.com/cloudwego/eino/components/document"
	"github.com/cloudwego/eino/components/document/splitter"
)

func main() {
	ctx := context.Background()

	longText := `学生信息管理系统包含：学生新增、学生删除、成绩录入、成绩查询、学业预警、GPA统计等全部功能。所有操作必须校验权限，必须真实读取数据库数据。`

	doc := &document.Document{Content: []byte(longText)}

	sp := splitter.NewRecursiveSplitter(
		splitter.WithChunkSize(60),
		splitter.WithChunkOverlap(10),
	)
	chunks, _ := sp.Split(ctx, doc)
	for i, c := range chunks {
		fmt.Printf("分片%d：%s\n", i, string(c.Content))
	}
}
```

### 3.3 Embedding 向量生成组件

**为什么需要**：不同厂商（OpenAI、通义千问、本地模型等）的 Embedding 接口各异，直接对接每家的 SDK 会让代码充满适配逻辑。

**解决的问题**：统一封装多厂商 Embedding，将切分后的文本与实时提问转为**同维度**向量。入库与检索必须使用同一模型版本，否则维度不一致会导致检索失效。

> 模型调用示例见 `components/model/`，包含 A/B 测试（`abtest/`）和 HTTP 传输（`httptransport/`）等进阶用法。

### 3.4 VectorStore 向量库组件

**为什么需要**：向量化后的数据需要一个专门的存储引擎来做高效的相似度检索，MySQL 的 B+Tree 索引无法胜任这类计算。

**解决的问题**：统一对接多款向量库（Milvus、Pinecone 等），存储**向量 + 原文片段 + 元数据**（用户 ID、标签、时间等）。检索时携带用户 ID、权限等过滤条件，防止越权泄露。

### 3.5 LLM 大模型调用组件

**为什么需要**：不同厂商的模型接口格式、参数命名、错误处理各不相同，业务代码中到处散落厂商特定的 SDK 调用会导致耦合严重、切换成本高。

**解决的问题**：统一封装多厂商接口，一套代码即可对接 OpenAI、Azure、通义千问等，支撑总结、抽取、问答、打分、报告生成等场景。

> 基础调用见 `quickstart/chat/`，Prompt 模板见 `components/prompt/chat_prompt/`。

### 3.6 Tool 工具组件

**为什么需要**：AI 不能直接操作数据库或调用业务 API，需要一种标准化的方式让 Agent「伸出手」触达业务系统。

**解决的问题**：将 MySQL 查询、统计、校验等业务逻辑封装为 Eino 标准工具，Agent 自动解析用户意图、补充参数、串并联调用，大幅减少手写路由代码。

> 工具定义示例见 `components/tool/`，MCP 工具对接见 `components/tool/mcptool/`。

### 3.7 ReAct Agent 智能调度

**为什么需要**：面对「帮我查一下张三的成绩，如果挂科了就发预警」这类多步请求，传统做法需要手写大量 if/else 来判断意图和编排步骤。

**解决的问题**：Agent 内置「推理-行动」循环，自动选择工具、规划多步、串行/并行执行，将复杂的意图路由交给模型推理完成。

> ReAct Agent 示例见 `flow/agent/react/`，含动态选项（`dynamic_option_example/`）、记忆管理（`memory_example/`）等进阶用法。Graph 方式构建的 Agent 见 `compose/graph/tool_call_agent/`。

### 3.8 Graph / Workflow 工作流

**为什么需要**：复杂业务流程（简历归档、面试闭环、学业预警等）涉及多个步骤串联，如果用普通函数调用，流程变更时牵一发动全身，且难以监控和复用。

**解决的问题**：将 Loader → Splitter → Embedding → 入库 → 打分 → 汇总等步骤编排为**可复用、可监控、可视化**的流水线，长链路业务建议用 Workflow 固化。

> Workflow 示例见 `compose/workflow/`（从简单到字段映射共 6 个渐进示例），Graph 示例见 `compose/graph/`（含状态管理、异步节点、中断恢复等）。

---

## 第四章：AI Agent 与 Tool 自动路由

### 4.1 要解决什么问题？

以教务场景为例，用户可能这样提问：

- 「查一下 2025 级所有学生成绩」→ 应走 MySQL 查询
- 「帮我分析一下全班平均分」→ 应走统计工具
- 「删除一个学生信息」→ 应先查 ID，再删除

传统写法需要大量 if/else 分支来维护意图识别，每新增一种问法都要改代码，扩展与测试成本高。

**Eino 的解法**：将业务操作封装为标准 Tool，由 Agent 结合工具描述与系统提示自动选择调用路径——你只需要告诉 Agent「有哪些工具可用」，剩下的意图识别和参数组装由模型推理完成。

### 4.2 Tool + Agent 完整示例

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/agent/react"
	"github.com/cloudwego/eino/components/llm/openai"
	"github.com/cloudwego/eino/schema"
)

// 模拟数据库工具：查询学生信息
func queryStudentTool(ctx context.Context, args map[string]any) (string, error) {
	name, ok := args["name"].(string)
	if !ok {
		return "参数错误", nil
	}
	// 这里真实项目换成 MySQL SQL 查询
	return fmt.Sprintf("【数据库查询结果】学生%s，学号2025001，GPA3.8，无挂科", name), nil
}

func main() {
	ctx := context.Background()

	// 1. 初始化大模型
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:  "你的APIKey",
		BaseURL: "你的地址",
		Model:   "gpt-3.5-turbo",
	})
	if err != nil {
		log.Fatal(err)
	}

	// 2. 注册工具：查询学生
	tools := []*schema.ToolInfo{
		{
			Name:        "query_student",
			Description: "根据学生姓名查询学生全部档案信息，需要传入学生姓名",
			Params: map[string]*schema.ParameterInfo{
				"name": {Type: "string", Description: "学生姓名"},
			},
			Function: queryStudentTool,
		},
	}

	// 3. 创建智能Agent
	agent, err := react.NewAgent(ctx, &react.AgentConfig{
		ToolCallingModel: chatModel,
		ToolsConfig:      tools,
		MessageModifier: func(ctx context.Context, input []*schema.Message) []*schema.Message {
			sys := schema.SystemMessage(`你是大学生信息管理系统AI助手。
业务规则：
1. 查询学生必须调用query_student工具
2. 需要先查ID才能操作数据时，必须先调用查询工具
3. 回答简洁、专业、友好，只用中文`)
			return append([]*schema.Message{sys}, input...)
		},
		MaxStep: 10,
	})
	if err != nil {
		log.Fatal("创建Agent失败：", err)
	}

	// 4. 测试自然语言提问
	resp, err := agent.Run(ctx, []*schema.Message{
		schema.UserMessage("帮我查一下张三的全部学生信息"),
	})
	if err != nil {
		log.Fatal("Agent运行失败：", err)
	}

	// 5. 输出最终智能回答
	fmt.Println("AI Agent最终回答：", resp.Content)
}
```

### 4.3 运行机制

Agent 的工作循环：解析自然语言 → 匹配最合适的工具 → 拼装参数 → 调用业务函数 → 将结果组织为自然语言输出。多步任务会自动循环执行，直到完成目标。

> 更完整的 Agent 示例见 `flow/agent/react/`，多 Agent 协作见 `adk/multiagent/`，可与第七章「意图识别方案对比」对照阅读。

---

## 第五章：文件上传与 AI 处理链路

### 5.1 为什么文件不直接传给后端？

高并发上传会打满应用服务器的带宽与磁盘 IO，引发超时甚至雪崩。更合理的做法是：前端直连 **OSS/COS** 完成分片上传、断点续传，**仅将文件 URL** 回传后端。后端把 URL 交给 Eino 的 Loader 远程拉取并解析，不经后端中转大文件。

### 5.2 为什么不直接把 OSS 链接丢给公网大模型？

- **合规**：涉密/隐私数据外传第三方模型可能触碰红线
- **环境**：私有化、政务内网可能无外网，必须本地解析
- **可控性**：厂商解析黑盒，难以定制清洗规则

### 5.3 推荐的工程落地范式

**OSS 加密 URL** → Eino **URLLoader** 拉取与清洗 → **Splitter** 切分 → **Embedding** → **VectorStore** 带元数据入库。全链路可监控、可回溯，每个环节对应第三章的组件。

---

## 第六章：RAG 全链路实践

### 6.1 入库链路——把文档变成可检索的知识

**OSS URL** → Loader 解析 → Splitter 分片 → Embedding 批量向量化 → 向量库写入（绑定用户/业务元数据）。此过程异步执行，**不阻塞**前端主流程。

### 6.2 查询链路——用户提问到智能回答

用户提问 → Embedding → 向量库检索（**带用户 ID 等权限过滤**）→ 召回相关片段 → 拼入 Prompt → LLM 基于私域数据作答 → 前端展示。

### 6.3 组件对应关系

入库和查询链路的每个环节都对应第三章的 Eino 组件（Loader → Splitter → Embedding → VectorStore → LLM），排查问题时可按链路分段定位。

> RAG 完整实现见 `quickstart/eino_assistant/`，其中 `eino/knowledgeindexing/` 为入库链路，`eino/einoagent/` 为查询链路。分步教程见 `quickstart/chatwitheino/cmd/`（ch01-ch09 渐进式演示）。

---

## 第七章：多业务意图自动分发

### 7.1 为什么需要意图分发？

同一个系统中往往并存两类查询：

- **私有资料问答**（如「帮我看看张三的简历」）：必须走向量 RAG，且按用户 ID 过滤，防止越权
- **公共业务查询**（如「查一下在招岗位」）：只需 MySQL 工具，不必消耗向量算力

如果所有请求都走同一条链路，要么浪费资源，要么答非所问。需要一种机制自动识别意图并路由到正确链路。

### 7.2 四种方案对比

| 方案 | 优点 | 局限 | 适用场景 |
|------|------|------|---------|
| 关键词/正则 | 实现快 | 覆盖面窄，难处理口语化 | 极简原型 |
| Prompt 意图分类 | 灵活 | 消耗 Token | 中小型项目 |
| 离线小模型分类 | 快、省 Token | 需训练维护 | 政务内网 |
| **ReAct Agent + Tool** | 规则少，与 Eino 一致 | 依赖模型推理质量 | 多数企业项目 |

### 7.3 生产落地要点

- 用 **MessageModifier** 统一注入系统规则，减少对业务接口的侵入
- **增删改**前先调用查询工具校验 ID 与权限
- Agent 结合工具描述与系统规则自主选择工具与步骤——这就是为什么可以少写 if/else（可与第四章示例对照）

> 路由检索器示例见 `components/retriever/router/`，演示了按条件分发到不同检索后端的实现。

---

## 第八章：Eino + Gin + MySQL + OSS 分层整合

### 8.1 为什么要分层？

将 AI 能力直接写进 Gin Handler 会导致职责混乱：接口层夹杂向量计算，AI 层散落鉴权逻辑，任何一处变更都可能引发连锁问题。分层的目的是**让每一层只做自己擅长的事**。

### 8.2 四层标准化架构

| 层级 | 职责 | 不做什么 |
|------|------|---------|
| **前端交互层** | 聊天、上传、展示 | 不承载业务逻辑与 AI 编排 |
| **网关层（Gin）** | 鉴权、限流、校验、路由 | 不写向量与大模型逻辑 |
| **AI 编排层（Eino）** | 文档解析、向量、RAG、Agent、Workflow | 不暴露 HTTP、不替代账号体系 |
| **数据底座层** | MySQL（结构化）+ 向量库（语义）+ OSS（文件） | 各司其职，互不替代 |

> 完整的分层整合示例见 `flow/agent/deer-go/`，展示了 Eino 与 Hertz 框架的实际集成方式。`quickstart/chatwitheino/` 则是一个包含前后端的完整 Chat 应用。

---

## 第九章：典型业务场景全流程

以下四个场景串联前面所有章节的知识点，展示 Eino 各组件如何在真实业务中协同工作。

### 9.1 用户投递简历（第五、六章）

前端上传 OSS → Gin 写 MySQL 投递记录与文件 URL → **异步** Eino 链路完成解析、切分、向量化、入库，主流程不阻塞。

### 9.2 HR 查询候选人技能（第四、七章）

Agent 识别意图 → MySQL 工具核验身份与用户 ID → **带 ID 过滤**检索向量库 → LLM 凝练回答。

### 9.3 查询在招岗位（第七章）

意图为公共数据 → **仅 MySQL 工具**，不走向量，响应更快、成本更低。

### 9.4 AI 面试、打分、报告（第三章 3.8）

用 **Graph/Workflow** 固化：读简历 → 出题 → 多轮对话 → 打分 → 生成报告，流程标准化、可追溯。

> Workflow 编排示例见 `compose/workflow/`，Graph 编排见 `compose/graph/`。多 Agent 协作的复杂场景见 `adk/multiagent/`。

---

## 第十章：常见风险与避坑指南

### 10.1 生产环境常见风险

| 风险类型 | 典型问题 | 正确做法 | 对应章节 |
|---------|---------|---------|---------|
| 业务 | 超长全文塞 MySQL 大字段 | 走 OSS + 向量分层存储 | 第五章 |
| 架构 | 前端直传大文件到应用服务器 | 优先直传对象存储 | 第五章 |
| 算力 | 简单问题全部走 RAG | 做意图/路由分层 | 第七章 |
| 智能 | 不做场景分流，盲检索向量 | Agent + Tool 自动路由 | 第四、七章 |
| 运维 | 复杂 AI 流程未 Workflow 化 | 用 Graph/Workflow 固化 | 第三章 3.8 |
| 安全 | 向量检索不带权限过滤 | 检索时携带用户 ID 过滤 | 第六章 |

### 10.2 核心概念速览

| 概念 | 一句话定义 | 解决的问题 |
|------|-----------|-----------|
| **Eino** | AI 链路编排框架 | 串联 LLM、RAG、工具与工作流 |
| **Embedding** | 文本 → 数值向量 | 让机器能计算语义相似度 |
| **RAG** | 先检索再生成 | 缓解幻觉、支持私域数据问答 |
| **Agent + Tool** | 模型驱动的工具调用 | 自然语言 → 业务操作，减少 if/else |
| **Workflow** | 可编排的流水线 | 复杂流程标准化、可监控、可复用 |
