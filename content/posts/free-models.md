---
title: "免费模型 API 平台大全"
date: 2026-09-06
image: "/images/covers/free-models.jpg"
---

![免费模型 API 平台大全](/images/covers/free-models.jpg)

# 免费模型 API 平台大全

> 本文收录了截至 2026‑09‑06 可公开获取的免费大模型 API 平台，涵盖国内外主要供应商、接口地址、免费额度、支持模型以及使用注意事项。所有信息均来源于公开文档或平台首页，已尽量验证准确性。若平台政策变化，请自行关注官方公告。

---

## 📚 目录

1. [UnoRouter – 190+ 免费模型聚合](#unorouter)
2. [Poixe AI – 免费模型套餐](#poixe)
3. [ChatAnywhere – 免费模型转发](#chatanywhere)
4. [SiliconFlow / SiliconCloud – 国内免费模型](#siliconflow)
5. [Kilo AI Gateway – 多厂商免费模型](#kilo)
6. [OpenRouter – 免费模型列表](#openrouter)
7. [EdgeOne Makers – LLM Gateway 免费模型](#edgeone)
8. [Free‑LLM‑Collection (GitHub)](#github-collection)
9. [其他值得关注的免费平台](#others)

---

## 1. UnoRouter – 190+ 免费模型聚合 <a id="unorouter"></a>

- **网址**: https://unorouter.com/zh-CN/boke/free-models-aggregated
- **简介**: 把 18 家上游免费提供方的 190+ 条模型统一到一个 OpenAI‑compatible 端点，只需要一把密钥即可调用。
- **免费额度**: 每个模型 **≈1 次/分钟**（平台层面统一限制），上游自身也会有每日/每分钟配额。超过配额返回 `429` 并附 `Retry‑After`。
- **支持的上游**: Groq、Gemini、Cerebras、SambaNova、Mistral、Cloudflare Workers AI、GitHub Models、Z.ai、OVHcloud、AI Horde、Pollinations、Cohere、Jina、NVIDIA NIM（Nemotron、Llama、Qwen、gpt‑oss、DeepSeek 等）、SiliconFlow（DeepSeek‑V4、GLM‑5.1、Qwen3.5/3.6、Kimi‑K2.6、MiniMax‑M3）、HuggingFace 路由器、LLM7 等。
- **使用方式**:
  ```bash
  curl https://api.unorouter.com/v1/chat/completions \
    -H "Authorization: Bearer YOUR_KEY" \
    -d '{"model":"gpt-oss-120b","messages":[{"role":"user","content":"你好"}]}'
  ```
- **注意**: 免费层是 **共享池**，高并发或大批量请求容易触发 `429`；建议轮换模型或加入重试逻辑。

---

## 2. Poixe AI – 免费模型套餐 <a id="poixe"></a>

- **网址**: https://poixe.com/products/free
- **提供**: 两个套餐（大杯/小杯），均为 **免费**，每日请求次数分别为 5 次/天 与 50 次/天，单请求 token 上限 4k/2k（输入/输出）。
- **模型**: 包含 Claude、GPT、Gemini、Mistral 等品牌的免费版，具体模型以页面列表为准。
- **调用示例**（Node.js）:
  ```js
  const openai = new OpenAI({apiKey: 'POIXE_API_KEY', baseURL: 'https://api.poixe.com/v1'});
  const res = await openai.chat.completions.create({model: 'claude-3-5-haiku-20241022:free', messages:[{role:'user',content:'你好'}]});
  console.log(res.choices[0].message.content);
  ```
- **适用场景**: 低频实验、个人项目、教育演示。

---

## 3. ChatAnywhere – 免费模型转发 <a id="chatanywhere"></a>

- **网址**: https://chatanywhere.tech/
- **关键特性**:
  - **国内中转**: `https://api.chatanywhere.tech`（国内网络延迟更低），国外使用 `https://api.chatanywhere.org`。
  - **免费额度**: 每天 **10 000点**（约 10 000 token），各模型每日调用次数上限（如 DeepSeek‑V4‑Flash 30 次/天、GPT‑4o‑mini 100 次/天）。
  - **支持模型**: DeepSeek、OpenAI 系列（gpt‑3.5‑turbo、gpt‑4o、gpt‑4o‑mini 等）、Claude、Gemini、MiniMax、Kimi、Qwen 等。
- **使用示例**（cURL）:
  ```bash
  curl https://api.chatanywhere.tech/v1/chat/completions \
    -H "Authorization: Bearer YOUR_KEY" \
    -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"写一段诗"}]}'
  ```
- **注意**: 免费版需要 **GitHub 登录** 获取 token；若超过配额会返回 `429`，请自行实现退避。

---

## 4. SiliconFlow / SiliconCloud – 国内免费模型 <a id="siliconflow"></a>

- **网址**: https://siliconflow.cn （SiliconFlow） / https://siliconcloud.com （SiliconCloud）
- **免费模型**:
  - `deepseek-ai/DeepSeek-R1-0528-Qwen3-8B`
  - `Qwen/Qwen3-8B`
  - `Qwen/Qwen3.5-4B`
  - `THUDM/glm-4-9b-chat`
  - `THUDM/GLM-4-9B-0414`
  - `THUDM/GLM-Z1-9B-0414`
  - `THUDM/GLM-4.1V-9B-Thinking`
- **API 地址**: `https://api.siliconflow.cn/v1`
- **免费额度**: **1000 RPM / 模型**（每分钟 1000 次请求），每日约 1440 RPD（请求次数）。
- **注册方式**: 通过邮箱或企业微信快速获取 `API_KEY`，无需信用卡。
- **调用示例**（Python）:
  ```python
  import openai
  client = openai.OpenAI(base_url='https://api.siliconflow.cn/v1', api_key='YOUR_KEY')
  resp = client.chat.completions.create(model='deepseek-ai/DeepSeek-R1-0528-Qwen3-8B', messages=[{'role':'user','content':'写一篇短文'}])
  print(resp.choices[0].message.content)
  ```
- **适合**: 对中文/中英混合需求、需要更高令牌上限（相比 30 RPM 的国外免费平台），但仍受每日配额限制。

---

## 5. Kilo AI Gateway – 多厂商免费模型 <a id="kilo"></a>

- **网址**: https://kilo.ai/leaderboard#all-models
- **统一入口**: `https://api.kilo.ai/api/gateway`
- **免费模型**（部分示例）:
  - `openrouter/free`
  - `kili-auto/free`
  - `inclusionai/ling-3.0-flash:free`
  - `poolside/laguna-s-2.1:free`
  - `tencent/hy3:free`
  - `nex-agi/nex-n2-pro:free`
  - `nvidia/nemotron-3-ultra-550b-a55b:free`
- **额度**: **200 RPH**（每小时 200 次请求），适合中等频次的实验。
- **使用方式**（curl）:
  ```bash
  curl https://api.kilo.ai/api/gateway/v1/chat/completions \
    -H "Authorization: Bearer KILO_API_KEY" \
    -d '{"model":"inclusionai/ling-3.0-flash:free","messages":[{"role":"user","content":"解释一下量子计算"}]}'
  ```
- **优势**: 多厂商聚合，支持 **OpenAI、Claude、Gemini、Qwen、GLM、DeepSeek** 等，且不强制传入 `base_url`，只需在 `model` 中指明即可。

---

## 6. OpenRouter – 免费模型列表 <a id="openrouter"></a>

- **网址**: https://openrouter.ai
- **免费模型入口**: `https://openrouter.ai/api/v1`
- **部分免费模型**（官方标记 `free`）:
  - `nvidia/nemotron-3-super-120b-a12b:free`
  - `minimax/minimax-m2.7:free`
  - `google/gemma-4-31b-it:free`
  - `nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free`
  - `z-ai/glm-5.2:free`
  - `cohere/north-mini-code:free`
- **额度**: **20 RPM / 200 RPD**（每模型），超过后返回 `429`。免费模型多数为 **实验性**，可能随时下线。
- **适用**: 快速尝鲜、对多供应商模型有统一调用需求的开发者。

---

## 7. EdgeOne Makers – LLM Gateway 免费模型 <a id="edgeone"></a>

- **网址**: https://pages.edgeone.ai/zh/use-cases/llm-gateway
- **核心卖点**: 统一端点、零平台费用、内置免费 Token（约 50 万 token），支持 **8+** 供应商。
- **免费模型**（示例）:
  - `@makers/deepseek-v4-flash`
  - `@makers/minimax-m3`
  - `@makers/gemini-3.7-flash`
  - `@makers/qwen3.5-4b`
- **额度**: 每账号赠送 **50 万 token**，随后按实际使用的上游配额计费（若使用免费上游则仍受上游限制）。
- **调用方式**（Python）:
  ```python
  import openai
  client = openai.OpenAI(base_url='https://gateway.edgeone.ai/v1', api_key='EDGEONE_API_KEY')
  resp = client.chat.completions.create(model='@makers/deepseek-v4-flash', messages=[{'role':'user','content':'写一首诗'}])
  print(resp.choices[0].message.content)
  ```
- **适合**: 需要一次性接入多家免费模型，且不想自行管理密钥的项目。

---

## 8. Free‑LLM‑Collection (GitHub) – 开源免费模型清单 <a id="github-collection"></a>

- **仓库**: https://github.com/for-the-zero/Free-LLM-Collection
- **内容**: Markdown 表格形式列出 **国内外** 近 200 条免费模型 API，包含以下信息：
  - 平台名称 & 官网
  - API 地址
  - 支持模型（示例）
  - 免费额度（RPM / RPD）
  - 备注（是否需要 GitHub 登录、是否有国内 CDN）
- **使用方式**: 直接克隆仓库或下载 `README.md`，按需挑选模型。
- **优势**: 开源、持续更新、社区维护；可自行部署到自己的文档站点（如 Hugo、Hexo）进行二次分发。

---

## 9. 其他值得关注的免费平台 <a id="others"></a>

| 平台 | 网址 | 免费模型 | 额度示例 | 备注 |
|------|------|----------|----------|------|
| **Celebras** | https://celebras.ai | `gpt-oss-120b`、`zai-glm-4.7` 等 | 30 RPM / 900 RPH | 需邮箱注册，免费额度有限 |
| **Mistral** | https://mistral.ai | `mistral-large-2512`（免费试用） | 未公开，通常需绑定支付 | 付费后额度提升 |
| **BigModel（智谱）** | https://bigmodel.cn | `GLM-4.7-Flash`、`GLM-4.6V-Flash` | 免费配额不明，但官方提供试用 token | 国内备案平台
| **ModelScope** | https://modelscope.cn | `deepseek-ai/DeepSeek-V4-Flash`、`ZhipuAI/GLM-5` 等 | 2000 RPD | 需要 `MODEL_SCOPE_API_KEY`，支持多模型路由 |
| **HuggingFace Router** | https://router.huggingface.co/v1 | 数百开源模型（全部免费） | 300 RPH | 需要 HF 账户 token，配额受 HF 计费策略影响 |
| **G4F（免费代理）** | https://g4f.dev/ | 自动路由到多家免费上游 | 未公开 | 适合作为 **一次性实验**，不保证稳定性 |
| **讯飞星火** | https://spark-api-open.xf-yun.com/v1 | `lite`（免费） | 2 QPS | 仅限中文对话，需企业备案 |
| **VolcEngine** | https://developer.volcengine.com/articles/7535310994050908206 | 多款国产模型（如 Qwen2） | 注册即送 2000 万 token | 需实名认证

---

## 📌 小结

- **免费模型并非无限**：大多数平台都会对 **请求频次、每日 token** 设限，超过后会返回 `429`。建议在代码里实现 **指数退避** 或 **轮询不同模型**。
- **安全性**：免费平台的 API Key 往往 **没有付费额度** 的风险，泄露后影响有限，但仍建议使用 **只读权限** 或 **短期 token**（如 GitHub OAuth）。
- **国内外混合使用**：在国内网络环境下，优先选择 **SiliconFlow、Kilo、ChatAnywhere** 等本地节点；在海外需求（如使用 Code Interpreter）可使用 **UnoRouter、OpenRouter** 的免费层。
- **持续关注**：免费政策随时可能变更，建议关注各平台的官方博客或 Discord 群组，以免业务中断。

祝开发顺利，玩转免费大模型！
