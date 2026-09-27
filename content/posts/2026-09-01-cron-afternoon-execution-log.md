---
title: "下午执行块复盘：90min深度任务 + 流程化执行 + SOP完善"
date: 2026-09-01T13:00:00+08:00
categories: [周记]
tags: [cron, workflow, sop, hugo]
series: [AI时代个人重构指南]
showToc: true
TocOpen: true
weight: 1
---

## 今天的执行回顾

### 13:00-14:30 核心深度任务：写作

今天按 `cron-afternoon-execution` 技能执行了下午执行块：

- **13:00-14:30**（90min）：完成了一篇博客文章的撰写与发布流程
- **14:30-14:40**：站立休息、喝水、远眺
- **14:40-16:00**：流程化执行——构建、部署、验证
- **16:00-16:10**：第二次 10min 休息
- **16:10-17:00**：SOP 完善与复盘，更新了对应文档

### 14:40-16:00 流程化执行：构建与部署

按 Hugo 部署 SOP 执行了完整流程：

1. `hugo --gc --minify` — 本地构建无错误
2. 验证 `baseURL = "https://ningop.com"` — 未检测到 `localhost` 字样
3. `wrangler pages deploy public/ --project-name=ningop-blog --branch=main --commit-dirty=true` — 部署成功
4. 多站点验证：
   - 部署 URL：HTTP/2 200 ✓
   - 别名 URL：HTTP/2 200（首次 522 已过，edge propagation 完成）
   - 自定义域 `ningop.com`：HTTP/2 200 ✓

### 16:10-17:00 SOP 完善/复盘

将今天跑通的流程写成/update 了 SOP 文档。关键点记录：

- **原子部署**：一次性完成 `hugo --gc --minify` + `wrangler pages deploy`，避免中间状态泄漏自定义域/CFG baseURL 不匹配
- **baseURL 检查**：部署前 `grep -rq 'localhost' public/` 必须 abort；确认 `hugo.toml baseURL = "https://ningop.com"`
- **部署后验证**：优先验证 deployment URL（即时 200），再等 10-30s 验证自定义域（可能出现单次 522，属正常边缘传播）
- **future date 陷阱**：article `date` 必须 ≤ build机器当下的北京时间；若使用 `date: YYYY-MM-DD` 纯日期更稳健
- **frontmatter 统一**：全站统一 YAML `---` 格式，禁止 `+++` TOML残留，否则 PaperMod 列表会静默跳过
- **hero 图位置**：`![featured](...)` 必须紧接在 YAML `---` 闭合后、第一个 `##` 之前；错误位置会在文章内部渲染而非页头
- **日期归档问题**：`date` 过老（如 2025-06）会沉到首页最底部 PaperMod 翻页之外；如需保持页首可见，建议 `date` 设为今日或适当适当 `weight` 微调

### 检查清单 & 下一步

| 检查项 | 状态 | 备注 |
|--------|------|------|
| 文章路径 `content/posts/` | ✅ | 非 `blogs/` |
| `date` 不超前 | ✅ | 2026-09-01 北京时间 |
| Frontmatter YAML `---` | ✅ | 无 `+++`残留 |
| Hero 图在 `---` 与第一个 `##` 之间 | ✅ | 已验证渲染位置 |
| `baseURL = "https://ningop.com"` | ✅ | 部署前确认 |
| `wrangler pages deploy` 成功 | ✅ | 项目 `ningop-blog` |
| 三站点验证通过 | ✅ | ningop.com / www.ningop.com / hugo.ningop.com |

**下一步**：继续下一篇系列文章的写作，或根据需要在 `content/posts/` 下创建新草稿。下一次 `cron-afternoon-execution` 块将在明天 13:00 触发。

[延伸阅读]: /posts/2026-08-31-ai-era-personal-reconstruction-log.md / posts/2026-08-29-ai-arbitrage-whitepaper-series.md