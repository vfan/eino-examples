# Eino AI Agent 后端开发实战

## 第一章：为什么企业后端必须学 Eino + AI Agent

### 1.1 传统后端 vs AI 后端核心区别

**传统 Go 后端**

- 只写接口、写 SQL、操作 MySQL、做鉴权、做分页
- 所有业务判断全靠 if/else 硬编码
- 只能处理结构化数据：学号、姓名、成绩、课程 ID
- 无法理解自然语言、无法自动判断用户意图

**AI Agent 后端（新增能力）**

- 能听懂自然语言：查一下张三的成绩、帮我批量统计平均分
- 自动判断意图、自动分步执行业务
- 自动调用数据库工具、自动查询、自动修改
- 不用写大量 if/else 路由判断

### 1.2 为什么不直接裸调大模型，非要用 Eino？

- 裸调用大模型不能直接读数据库
- 裸调用不能分步执行业务（先查 ID、再改数据）
- 裸调用无法管理多轮会话上下文
- 裸调用没有工具调度能力，不能对接 MySQL 业务接口
- Eino 专门解决：AI + 后端业务联动的全链路编排问题

### 1.3 本章实战：第一个 Eino 入门 Demo

功能：直接调用大模型，测试环境是否可用，零基础必跑。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/eino/components/llm/openai"
)

func main() {
	// 1. 全局上下文
	ctx := context.Background()

	// 2. 初始化大模型客户端
	chatModel, err := openai.NewChatModel(ctx, &openai.ChatModelConfig{
		APIKey:  "你的大模型APIKey",
		BaseURL: "你的代理地址",
		Model:   "gpt-3.5-turbo",
	})
	if err != nil {
		log.Fatal("模型初始化失败：", err)
	}

	// 3. 简单提问
	resp, err := chatModel.Generate(ctx, openai.WithPrompt("你是后端AI助手，简单介绍一下Eino框架"))
	if err != nil {
		log.Fatal("调用模型失败：", err)
	}

	// 4. 输出结果
	fmt.Println("AI回答：", resp.Content)
}
```

---

## 第二章：AI 三大核心基础理论

### 2.1 什么是 Embedding 向量

一句话：把文字变成一串数字数组。

- 文字人类看得懂，机器看不懂
- 向量是机器能计算的数字
- 意思越相近，向量距离越近
- 所有智能问答、智能检索，底层全靠向量支撑

### 2.2 什么是 RAG 检索增强生成

RAG = 先查资料，再回答问题

1. 把私有文档、学生数据、成绩数据转成向量存入向量库
2. 用户提问 → 检索相似资料
3. 把资料送给大模型 → 大模型基于真实数据回答

作用：防止 AI 胡说八道、保护内部隐私数据。

### 2.3 MySQL 和 向量库 分工区别

- **MySQL**：存学号、姓名、成绩、课程、账号密码，精准查数据
- **向量库**：存长文本、文档、语义向量，做模糊语义匹配

### 2.4 本章代码：文本向量化小实验

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

	// 初始化向量模型
	embedder, err := openai.NewEmbedding(ctx, &openai.EmbeddingConfig{
		APIKey:  "你的APIKey",
		BaseURL: "你的地址",
	})
	if err != nil {
		log.Fatal(err)
	}

	// 普通自然语言文本
	texts := []string{
		"张三GPA成绩优秀",
		"李四多门课程挂科，需要学业预警",
		"今天食堂饭菜很好吃",
	}

	// 向量化
	vectors, err := embedder.EmbedTexts(ctx, texts)
	if err != nil {
		log.Fatal(err)
	}

	// 打印向量维度
	for i, vec := range vectors {
		fmt.Printf("第%d条文本向量长度：%d\n", i+1, len(vec))
	}
}
```

---

## 第三章：Eino 核心组件全套精讲（带代码）

### 3.1 Loader 文档加载器

作用：读取 PDF、TXT、OSS 云端文件，输出标准文档对象。

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

	// 线上公开文档地址
	url := "https://example.com/student-rule.txt"

	// 初始化加载器
	l := loader.NewURLLoader(url)

	// 拉取文档
	doc, err := l.Load(ctx)
	if err != nil {
		log.Fatal("加载文档失败：", err)
	}

	fmt.Println("文档内容预览：", string(doc.Content[:100]))
}
```

### 3.2 Splitter 文档切分器

原理：大模型有长度限制，长文本必须切成小段才能处理。

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

	// 切分：每段60字符，重叠10字符
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

---

## 第四章：重点核心：AI Agent + Tool 自动路由（本节课核心考点）

### 4.1 业务痛点

学生提问：

- 查一下 2025 级所有学生成绩 → 走 MySQL 查询
- 帮我分析一下全班平均分 → 走统计工具
- 删除一个学生信息 → 先查 ID，再删除

传统写法：写一堆 if/else 判断意图，维护爆炸。

**Eino 方案：不用 if/else，Agent 自动判断、自动选工具。**

### 4.2 Tool 工具封装标准

把 MySQL 增删改查，全部封装成 Eino 标准工具，Agent 自动调用。

### 4.3 完整可运行：学生信息查询 Tool + Agent 代码

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

### 4.4 核心原理讲解

- Agent 看懂自然语言：查张三
- Agent 自动匹配工具：query_student
- Agent 自动拼装参数 → 自动调用数据库函数
- 拿到结果 → 整理成自然语言返回前端
- 全程没有写一句 if/else 意图判断

---

## 第五章：企业四层架构 + 线上避坑清单

### 5.1 四层标准架构

1. **前端层**：聊天、上传文件
2. **网关层**：Gin 接口、鉴权、限流
3. **AI 编排层**：Eino 所有组件、Agent、RAG
4. **数据层**：MySQL + 向量库 + OSS 云存储

### 5.2 线上六大高危坑点

- 不要把长文本直接存 MySQL
- 不要让前端直接传文件到后端
- 不要所有问题都走向量检索，浪费钱
- 必须用 Agent 自动路由，不要手写意图判断
- 多步业务必须用 Workflow 编排
- 向量检索必须带用户 ID 过滤，防止数据泄露

---

## 第六章：课堂总结 + 学生必背考点

- **Eino 作用**：AI 业务全流程编排框架
- **向量作用**：让机器看懂语义
- **RAG 作用**：私有数据问答，防幻觉
- **Agent 作用**：自动路由、自动调用数据库工具
- **核心代码**：Agent + MessageModifier + 业务 Tool 组合开发
