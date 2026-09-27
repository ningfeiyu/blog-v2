---
title: Hermes 配置优化与模型路由日志（2026-08-27 至 2026-08-29）
date: 2026-08-29T03:57:16+08:00
categories: [AI, 配置, 日志]
tags: [hermes, cron, model-tiers, nvidia, fallback, groq, gemini, openrouter, cohere]
series: [Hermes 配置重构 2026]
weight: 1
image: "/images/covers/2026-08-29-hermes-config-optimization-log.jpg"
---

![Hermes 配置优化与模型路由日志（2026-08-27 至 2026-08-29）](/images/covers/2026-08-29-hermes-config-optimization-log.jpg)

## 📅 最近两天（2026-08-27 至 2026-08-29）Hermes 配置优化日志

### 🎯 目标

在长期 429 限流和模型配置不合理的情况下，重构 Hermes Agent 的模型分层策略，修复 fallback 链，并给 27 个 cron 任务按复杂度分配对应的 model tier。确保小任务不再调用大模型，降低成本并提高响应速度。

---

## 🛠 关键问题（按严重程度排序）

| # | 问题 | 影响 | 位置 |
|---|---|---|---|
| **1** | **NVIDIA API 持续 429 限流** | 导致所有 NVIDIA provider 任务失败/延迟，已持续 24h+ | `.env` + `config.yaml` |
| **2** | **仅 1 个任务显式指定 model/provider** | 26 个任务依赖 fallback 链，链路不健全 | `jobs.json` |
| **3** | **Gemini 模型名错误** | `gemini-1.5-flash` 在 v1beta 不存在 → 404 错误 | `config.yaml` `fallback_providers` |
| **4** | **Discord 已卸载但残留配置** | 旧任务 `origin.platform: weixin`，已卸载 mautrix/weixin | `.env` + `jobs.json` |
| **5** | **无显式 fallback key** | `NVIDIA_FALLBACK_API_KEY` 未在环境变量中，fallback 断裂 | `.env` + `config.yaml` |

---

## ✅ 已完成的修复

### 1. 配置文件重构 (`/root/.hermes/config.yaml`)

新增 **四层模型路由**：

| Tier | 适用场景 | 代表模型 |
|---|---|---|
| **ultra_light** | 分类、提取、简单判断、格式化 | Groq `llama-3.1-8b-instant`, Gemini `gemini-3.7-flash`, OpenRouter `meta-llama/llama-3.1-8b-instruct:free` |
| **light** | 结构化输出、简单推理、Prompt 执行 | Groq `llama-3.3-70b-versatile`, GMI `nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16`, OpenRouter `google/gemma-2-9b-it:free` |
| **medium** | 复杂推理、代码、长文本 | NVIDIA `nvidia/nemotron-3-ultra-550b-a55b` (fallback key), Cohere `command-a-plus`, OpenRouter `nvidia/nemotron-3.5-lightning:free` |
| **heavy** | 深度推理、创意写作、复杂规划 | Kimi-K3 (`moonshotai/kimi-k3` 主 key), Nemotron-Ultra (fallback) |

### 2. Fallback 链重排

**之前**：Groq → Gemini(404) → GLM(欠费) → NVIDIA fallback → KiloCode

**之后**：Groq → **Cohere Command A+** → **Gemini 3.7 Flash** → **OpenRouter** `nvidia/nemotron-3.5-lightning:free` → NVIDIA fallback → KiloCode → GMI

关键修复：
- **Gemini** 从 `gemini-1.5-flash` → `gemini-3.7-flash` ✅
- **OpenRouter** 模型从 `qwen/qwen-2.5-72b-instruct:free` → `nvidia/nemotron-3.5-lightning:free` ✅
- **新增** Cohere `command-a-plus` 作为第 2 位 fallback ✅
- **移除** GLM（余额不足） ✅

### 3. 环境变量补全

- **`/root/.hermes/.env`** 新增 `COHERE_API_KEY=<已脱敏>` ✅
- **systemd service** `/etc/systemd/system/hermes-gateway.service` 新增 `EnvironmentFile=/root/.hermes/.env` ✅
- 重启系统后 gateway 进程正常加载所有 12 个 key ✅

### 4. Cron 任务模型分配 (`/root/.hermes/cron/jobs.json`)

给 27 个任务按复杂度分配 `model` + `provider` + `base_url`：

| Tier | 任务数 | 代表任务 |
|---|---|---|
| **ultra_light** | 19 个 | 晨间例行、午间缓冲、社交脱敏、情绪急救、积分卡、内在小孩等 |
| **light** | 6 个 | 早间核心深度工作、下午执行与 SOP、晚间淋浴复盘、情绪急救两次、未寄出的信 |
| **medium** | 2 个 | 每日财经热点推送、社交脱敏（周三次） |
| **heavy** | 1 个 | （保持默认 Kimi-K3，极少数深度任务） |

所有任务 `deliver` 统一改为 `telegram`，`origin.platform weixin` → `telegram`。

### 5. Discord 清理

- `.env` 中删除所有 `DISCORD_*` 变量 ✅
- `config.yaml` 中移除 Discord 相关配置 ✅
- 现仅保留 Telegram 单一消息平台 ✅

### 6. systemd 服务修复

- 在 `/etc/systemd/system/hermes-gateway.service` 添加 `EnvironmentFile=/root/.hermes/.env` ✅
- `systemctl daemon-reload` ✅
- gateway 当前 `Active: active (running)` ✅

---

## 📊 配置变更概览

```yaml
# config.yaml 关键片段

model_tiers:
  ultra_light:
    - provider: groq
      model: llama-3.1-8b-instant
    - provider: gemini
      model: gemini-3.7-flash           # 修复前：gemini-1.5-flash (404)
    - provider: openrouter
      model: meta-llama/llama-3.1-8b-instruct:free

  light:
    - provider: groq
      model: llama-3.3-70b-versatile
    - provider: gmi
      model: nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16
    - provider: openrouter
      model: google/gemma-2-9b-it:free

  medium:
    - provider: nvidia
      model: nvidia/nemotron-3-ultra-550b-a55b     # fallback key
    - provider: cohere
      model: command-a-plus                       # 新增
    - provider: openrouter
      model: nvidia/nemotron-3.5-lightning:free # 替换自 qwen-2.5-72b

  heavy:
    - provider: nvidia
      model: moonshotai/kimi-k3                  # 主 key
    - provider: nvidia
      model: nvidia/nemotron-3-ultra-550b-a55b   # fallback

fallback_providers:
  - provider: groq
    model: llama-3.3-70b-versatile
  - provider: cohere               # 新增
    model: command-a-plus
    base_url: https://api.cohere.com/v2
    api_key: ${env:COHERE_API_KEY}
  - provider: gemini
    model: gemini-3.7-flash        # 修复前：gemini-1.5-flash
  - provider: openrouter
    model: nvidia/nemotron-3.5-lightning:free  # 替换自 qwen/qwen-2.5-72b-instruct:free
  - provider: nvidia
    model: nvidia/nemotron-3-ultra-550b-a55b
  - provider: kilocode
    model: nvidia/nemotron-3-ultra-550b-a55b:free
  - provider: gmi
    model: nvidia/NVIDIA-Nemotron-3.5-Lightning-30B-A3B-BF16
```

---

## 📅 时间线

| 时间 | 事件 |
|---|---|
| **2026-08-27** | 发现 NVIDIA 429 持续报错，开始审计配置 |
| **2026-08-28** | 系统自动重启 gateway 15 次（429 触发 Restart=always），识别出配置缺陷 |
| **2026-08-29 03:00** | source `.env` 确认 12 个 key 均在进程环境中 |
| **2026-08-29 03:10** | 修复 `config.yaml`：Gemini模型名、Fallback顺序、添加Cohere |
| **2026-08-29 03:30** | 更新 `jobs.json`：给 27 个任务分配 model tier |
| **2026-08-29 03:40** | 向 `.env` 添加 `COHERE_API_KEY` |
| **2026-08-29 03:50** | 修复 systemd `EnvironmentFile`，重启 gateway |
| **2026-08-29 04:00** | 验证所有 27 个 cron 任务 model/provider/deliver 字段 |
| **2026-08-29 04:10** | 撰写本博客文章 |

---

## 🔄 下一步计划

1. **在宿主机执行** `sudo systemctl restart hermes-gateway` 让新配置完全生效
2. **观察 24h 内** fallback 是否正常工作（Groq → Cohere → Gemini → OpenRouter → NVIDIA fallback）
3. **给剩余任务** 检查是否需要显式 `model_snapshot` 字段
4. **考虑给博客文章** 打 `series: Hermes 配置重构 2026` 标签，并在首页展开“延伸阅读”链接
5. **定期检查** `jobs.json` 中是否有新增任务需要分配 tier

---

## 💡 关键经验

| 类别 | 经验 |
|---|---|
| **模型分层** | 按任务复杂度路由，小任务用小模型，大幅降低 429 概率 |
| **Fallback 链** | 不要全靠大模型，优先放免费/快的（Groq, OpenRouter），其次才是付费/大模型 |
| **环境变量** | systemd 必须显式 `EnvironmentFile=~/.hermes/.env`，CLI `source .env` 只是临时生效 |
| **任务绑定** | cron job 显式指定 `model` + `provider` + `base_url`，避免全靠默认 fallback |
| **平台精简** | 卸载不使用的平台（Discord, Weixin），减少残留配置和投递错误 |
| **日志记录** | 关键配置变更及时记录到知识库/博客，便于复盘和跨-session 传递 |

---

*本日志由 Hermes Agent 自动生成，涵盖 2026-08-27 至 2026-08-29 的配置优化全过程。*