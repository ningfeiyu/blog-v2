---
title: Hermes 快速上手：CLI 终端的高效对话模式
description: Hermes命令式交互指南：避免低效提问，用具体命令直接触发执行，提高AI助手响应效率。
date: '2026-06-01T11:20:00+08:00'
draft: false
tags:
- Hermes
- CLI
- 命令式交互
- 效率工具
categories:
- 技术工具
showToc: false
---


<img src="/images/posts/4Mw7nkQDByk.jpg" alt="Hermes命令式交互指南：避免低效提问，用具体命令直接触发执行，提高AI助手响应效率。" loading="lazy" style="max-width:100%;height:auto;">

## 核心原则：命令式交互
Hermes 是为**行动**设计的，而非解释或规划。模糊的指令会触发不必要的明确化请求（如"你是想让我部署博客吗？"），而命令式语句则直接触发执行。

**低效示例**（会被打断）：
> "如何部署 Hugo 博客到 Vercel？"

**高效示例**（直接执行）：
> "部署 Hugo 博客到 Vercel，使用默认配置"

### 1.1 直击要害
- **部署**：直接说 `部署 Hugo 博客`，Hermes 会自动调用 `npx vercel deploy --prod --yes`。
- **调试**：`检查 Vercel 构建日志` 会触发 `npx vercel logs --prod`。
- **修改文件**：`在 hugo.toml 添加 Umami 配置` 会调用 `patch` 工具。

## 2 避免的坑
### 2.1 工具明确性
Hermes 会优先选择最合适的工具执行任务，但有时可能猜错（如误用 `web_search` 查找本地文件）。在复杂任务中明确指定工具：

```plaintext
# ❌ 低效
"给博客加个 RSS"

# ✅ 高效
"修改 hugo.toml 启用 RSS，使用 patch 工具"
```

### 2.2 文件路径
始终使用**绝对路径**（如 `/root/hugo-blog/hugo.toml`），避免相对路径的歧义。Hermes 默认以用户的 home 目录（`/root`）为起点，但不会假设项目路径。

```plaintext
# ❌ 可能出错
"读取 hugo.toml"

# ✅ 确定有效
"读取 /root/hugo-blog/hugo.toml"
```

## 3 实战案例
### 3.1 一键部署博客
需要部署博客时，直接发送：
```plaintext
deploy Hugo blog to Vercel with production flag
```
Hermes 会执行：
```bash
cd /root/hugo-blog && npx vercel deploy --prod --yes
```

### 3.2 修改 Hugo 配置
例如启用 Umami 统计：
```plaintext
在 /root/hugo-blog/hugo.toml 添加 Umami 配置：网站 ID 为 "abc123"，脚本 URL 为 "https://umami.example.com/script.js"
```
Hermes 会调用 `patch` 工具精确插入配置项，无需手动编辑。

## 4 总结
Hermes 的高效交互依赖于：**明确的指令 + 具体的路径 + 适当的工具限定**。避免开放式提问，直接表达意图，Hermes 会在 1 秒内开始执行。