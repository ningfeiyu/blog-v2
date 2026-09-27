---
title: "AI 基础设施免费工具全图谱：从 AI Gateway 到模型路由的 20 款核心服务"
date: 2026-06-06T23:00:00+08:00
draft: false
categories: ["技术"]
tags: ["AI Gateway", "LLM", "API", "基础设施", "开源", "免费", "模型路由", "代理"]
description: "深度梳理2020年代AI开发者基础设施层的免费工具，聚焦AI Gateway、LLM路由代理、模型聚合、成本优化等核心环节。"
---

![AI Gateway Tools](/images/remote/ext-2c9198c7a5fe.jpg)

> *"当所有人都在聊 Prompt Engineering 的时候，真正的工程师在优化 Gateway 层的延迟和成本。"*

## 为什么 AI Gateway 如此重要

2026年的AI应用开发已经分层：

- **上层**：Prompt 工程、RAG、Agent 编排
- **中层**：模型选择、参数调优
- **底层**：AI Gateway —— 路由、重试、缓存、限流、成本监控

**AI Gateway 是生产环境的咽喉。** 没有它，你的应用只是个玩具。

---

## 一、AI Gateway 核心工具

### 1. **Kong AI Gateway** | 🌟🌟🌟🌟🌟

- **定位**：企业级 API Gateway + AI 专用插件
- **免费点**：开源版完全免费
- **AI加持**：原生支持 LLM 路由、token 计费、语义缓存
- **杀手锏**：《 fortune 500 企业的选择，插件生态极其丰富
- **适用场景**：已有 Kong 基础设施的团队扩展 AI 能力
- **官网**：[konghq.com](https://konghq.com)

```yaml
# Kong AI 插件配置示例
plugins:
  - name: ai-proxy
    config:
      route_type: "llm/v1/chat"
      model_provider: "openai"
      auth_header_name: "Authorization"
```

> "如果你已经在用 Kong，AI 能力就是一行配置的事。"

---

### 2. **LiteLLM** | 🌟🌟🌟🌟🌟

- **定位**：LLM 统一接口层 + 代理服务器
- **免费点**：开源核心完全免费
- **杀手锏**：一行代码切换 100+ 模型提供商，统一 OpenAI 格式
- **核心功能**：
  - 多模型负载均衡与故障转移
  - 请求/响应记录与审计
  - 虚拟 key 管理与精细化权限
  - Spend 追踪与预算控制
- **官网**：[litellm.ai](https://litellm.ai)

```python
import litellm

# 统一的调用方式，底层自动路由
response = litellm.completion(
    model="gpt-4",  # 或 "claude-3-5-sonnet", "gemini-1.5-pro"
    messages=[{"role": "user", "content": "Hello"}]
)
```

---

### 3. **Helicone** | 🌟🌟🌟🌟

- **定位**：LLM 可观测性 + 代理
- **免费点**：每月 10万 请求免费
- **杀手锏**：一行代码接入全链路追踪
- **核心能力**：
  - 请求/响应完整记录
  - Cost/token 实时分析
  - A/B 测试与 Prompt 版本管理
  - 缓存与重试策略
- **官网**：[helicone.ai](https://helicone.ai)

```python
# 只需修改 base_url
openai.api_base = "https://oai.helicone.ai/v1"
```

---

### 4. **AI Gateway (Cloudflare)** | 🌟🌟🌟🌟🌟

- **定位**：边缘 AI 推理网关
- **免费点**：Cloudflare Workers 免费额度内
- **杀手锏**：全球 300+ 边缘节点，推理延迟 < 50ms
- **核心功能**：
  - 统一 REST/gRPC 接口
  - 模型编排与提示词模板
  - 边缘缓存与流式响应
  - 与 Workers AI 深度集成
- **官网**：[developers.cloudflare.com/ai-gateway](https://developers.cloudflare.com/ai-gateway)

```bash
# Cloudflare Workers AI 调用示例
curl https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/ai/run/@cf/meta/llama-3-8b-instruct \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"messages": [{"role": "user", "content": "Hello"}]}'
```

---

### 5. **Portkey AI** | 🌟🌟🌟🌟

- **定位**：LLM 生产化网关
- **免费点**：每月 1万 请求免费
- **杀手锏**：Guardrails（安全检查）、Fallback 链、Prompt 管理
- **核心能力**：
  - 输入/输出内容安全过滤
  - 多模型 Fallback 与重试
  - 语义缓存降低 API 成本
  - Prompt 版本与 A/B 测试
- **官网**：[portkey.ai](https://portkey.ai)

---

## 二、模型路由与聚合

### 6. **OpenRouter** | 🌟🌟🌟🌟🌟

- **定位**：LLM 路由市场
- **免费点**：大量模型有免费层
- **杀手锏**：统一接口访问 200+ 模型，自动选择最优路径
- **特色**：模型按价格/质量排序，自由选择
- **官网**：[openrouter.ai](https://openrouter.ai)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="$OPENROUTER_API_KEY"
)

# 自动路由到最优模型
response = client.chat.completions.create(
    model="openrouter/auto",  # 自动选择
    messages=[{"role": "user", "content": "Hello"}]
)
```

---

### 7. **One API** | 🌟🌟🌟🌟

- **定位**：国产 LLM API 聚合与转发
- **免费点**：开源免费
- **杀手锏**：把各种国内大模型（通义千问、文心一言、智谱）转成 OpenAI 格式
- **适用场景**：国内部署，需要统一管理多家模型
- **GitHub**：`songquanpeng/one-api`

---

### 8. **AI SDK (Vercel)** | 🌟🌟🌟🌟🌟

- **定位**：前端友好的 AI 开发套件
- **免费点**：完全开源免费
- **杀手锏**：前端流式 UI + 后端 AI 推理的最丝滑体验
- **核心能力**：
  - `useChat()` 一键流式对话
  - `streamText()` 文本生成
  - 多提供商切换（OpenAI、Anthropic、Google等）
  - Tool/Function calling 支持
- **官网**：[sdk.vercel.ai](https://sdk.vercel.ai)

```tsx
import { useChat } from 'ai/react'

function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat()
  
  return (
    <form onSubmit={handleSubmit}>
      {messages.map(m => (
        <div key={m.id}>{m.role}: {m.content}</div>
      ))}
      <input value={input} onChange={handleInputChange} />
    </form>
  )
}
```

---

## 三、AI 基础设施与中间件

### 9. **Langfuse** | 🌟🌟🌟🌟🌟

- **定位**：开源 LLM 可观测性平台
- **免费点**：开源版完全免费
- **杀手锏**：自托管，数据完全可控
- **核心能力**：
  - Trace 追踪每个请求的完整链路
  - 成本分析（按模型、按用户、按会话）
  - 评估与打分（人工/自动）
  - Prompt 版本管理
- **官网**：[langfuse.com](https://langfuse.com)

---

### 10. **LangSmith** | 🌟🌟🌟🌟

- **定位**：LangChain 官方可观测性平台
- **免费点**：每月 十万 trace 免费
- **杀手锏**：与 LangChain 深度集成
- **官网**：[smith.langchain.com](https://smith.langchain.com)

---

### 11. **Braintrust** | 🌟🌟🌟🌟

- **定位**：AI 应用评估平台
- **免费点**： generous free tier
- **杀手锏**：用 "evals" 评估 AI 输出质量
- **核心能力**：
  - 数据集管理
  - 自动评分（规则/LLM judge）
  - 回归测试
  - A/B 对比
- **官网**：[braintrust.dev](https://braintrust.dev)

---

## 四、向量数据库与检索

### 12. **Pinecone** | 🌟🌟🌟🌟

- **定位**：托管向量数据库
- **免费点**：每月 10万 向量操作
- **杀手锏**：零运维，毫秒级向量检索
- **官网**：[pinecone.io](https://pinecone.io)

---

### 13. **Weaviate** | 🌟🌟🌟🌟🌟

- **定位**：开源向量搜索引擎
- **免费点**：开源版完全免费
- **杀手锏**：GraphQL 接口，模块可扩展，支持混合检索
- **官网**：[weaviate.io](https://weaviate.io)

```python
import weaviate

client = weaviate.Client("https://ningop-weaviate.example.com")

# 混合检索（向量 + 关键词）
result = client.query.get("Article", ["title", "content"])\
    .with_hybrid(query="AI Gateway", alpha=0.5)\
    .do()
```

---

### 14. **Chroma** | 🌟🌟🌟🌟

- **定位**：轻量级本地向量数据库
- **免费点**：完全开源免费
- **杀手锏**：`pip install chromadb`，5分钟上手
- **适用场景**：原型开发、本地 RAG
- **官网**：[tr-llm-try-chroma-try](https://trychroma.com)

---

## 五、AI 代理与编排

### 15. **LangChain** | 🌟🌟🌟🌟🌟

- **定位**：LLM 应用开发框架
- **免费点**：完全开源
- **杀手锏**：链式调用、工具集成、记忆管理
- **官网**：[langchain.com](https://langchain.com)

---

### 16. **AutoGen / AgentScope** | 🌟🌟🌟🌟

- **定位**：多智能体对话框架
- **免费点**：开源
- **杀手锏**：多个 AI Agent 协作完成任务
- **官网**：微软 AutoGen

---

## 六、全方位能力矩阵

| 工具 | 类型 | 核心能力 | 免费层 | 生产就绪 |
|------|------|---------|--------|---------|
| **Kong AI** | Gateway | 企业级路由/安全 | ⭐⭐⭐⭐⭐ | ✅ 极高 |
| **LiteLLM** | 代理/路由 | 100+模型统一接口 | ⭐⭐⭐⭐⭐ | ✅ 高 |
| **Helicone** | 可观测性 | 全链路追踪 | ⭐⭐⭐⭐ | ✅ 高 |
| **Cloudflare AI Gateway** | 边缘网关 | 全球边缘推理 | ⭐⭐⭐⭐⭐ | ✅ 极高 |
| **Portkey** | Gateway | Guardrails/缓存 | ⭐⭐⭐⭐ | ✅ 高 |
| **OpenRouter** | 模型市场 | 自动路由200+模型 | ⭐⭐⭐⭐⭐ | ✅ 中 |
| **One API** | 聚合转发 | 国产模型转换 | ⭐⭐⭐⭐⭐ | ✅ 中 |
| **Vercel AI SDK** | 前端套件 | 流式UI/多提供商 | ⭐⭐⭐⭐⭐ | ✅ 高 |
| **Langfuse** | 可观测性 | 自托管Trace | ⭐⭐⭐⭐⭐ | ✅ 高 |
| **LangSmith** | 可观测性 | LangChain集成 | ⭐⭐⭐⭐ | ✅ 高 |
| **Braintrust** | 评估平台 | AI输出质量评估 | ⭐⭐⭐⭐ | ✅ 中 |
| **Pinecone** | 向量DB | 托管检索 | ⭐⭐⭐⭐ | ✅ 极高 |
| **Weaviate** | 向量DB | 混合检索 | ⭐⭐⭐⭐⭐ | ✅ 高 |
| **Chroma** | 向量DB | 轻量本地 | ⭐⭐⭐⭐⭐ | ✅ 中 |
| **LangChain** | 编排框架 | 链式/工具/记忆 | ⭐⭐⭐⭐⭐ | ✅ 高 |

---

## 七、生产架构推荐

### 标准三层架构

```
┌─────────────────────────────────────────────┐
│                  前端层                       │
│        Next.js + Vercel AI SDK               │
│              useChat()                       │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│              AI Gateway 层                   │
│  ┌─────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ LiteLLM│  │ Helicone│  │Cloudflare AI│  │
│  │(路由切换)│  │(追踪记录)│  │(边缘缓存)   │  │
│  └────┬────┘  └────┬─────┘  └──────┬──────┘  │
│       └─────────────┴───────────────┘          │
│                         统一 OpenAI 格式        │
└────────────────────┬───────────────────────────┘
                    │
┌───────────────────▼───────────────────────────┐
│               模型层                           │
│  ┌──────────┐ ┌─────────┐ ┌──────────────┐  │
│  │ OpenAI   │ │Anthropic│ │ Azure / GCP  │  │
│  │ GPT-4o   │ │ Claude  │ │  Vertex AI   │  │
│  └──────────┘ └─────────┘ └──────────────┘  │
│  ┌──────────┐ ┌─────────┐ ┌──────────────┐  │
│  │ 国内模型  │ │本地化部署│ │  Workers AI  │  │
│  │ 通义千问  │ │ Ollama  │ │   (免费)     │  │
│  └──────────┘ └─────────┘ └──────────────┘  │
└───────────────────────────────────────────────┘
```

---

## 八、成本优化的三个底牌

1. **语义缓存**：Helicone/Portkey 的缓存可将重复请求成本降低 90%

2. **模型降级**：非关键任务用 `gpt-3.5-turbo` 或免费层模型（Claude Haiku、Gemini Flash）

3. **批处理聚合**：LiteLLM 的 batch 模式减少 API 调用次数

---

> *"2026年的AI开发，拼的不是谁会用 ChatGPT，而是谁能把十几家模型提供商、几十种工具链、数百万次 API 调用，优雅地编织成一个可靠的生产系统。AI Gateway 是这个系统的基石。"*
