---
title: "W28 周记：给 Hermes 装上 Harness——但不用新标准，用已有机制"
date: 2026-07-14T01:55:00+08:00
draft: false
description: "读完 Daniel Warfield 的 Agent Harness 文章，对照 Hermes 现有架构，发现它本质上已经是 Harness 实现。改善 3 点：MEMORY.md 加核心 skill 索引、STATE.md 高信号搬家、项目级 AGENTS.md 落地。"
tags: ["周记", "Hermes", "Agent Harness", "架构"]
categories: ["周记"]
series: [周记]
showToc: true
TocOpen: false
---

![featured](/images/posts/weekly-2026-w28.jpg)

## 起因

上周读到 Daniel Warfield 的 [《Agent Harnesses：现代 AI Agent 开发的新标准》](https://www.drpang.ai/agent-harnesses-ai-agent-standard/)（原文 Daniel Warfield，Medium/Substack）。核心观点：

> Harness = 角色 + 上下文 + Skills + References 的结构化打包，让通用 Agent 在具体任务中稳定、可复用、可维护地工作。

三层结构：
- `HARNESS.md` —— 入口地图
- `skills/` —— 能力包
- `references/` —— 环境手册

设计原则：**Progressive Disclosure**（渐进披露）——先读入口，按需加载，别一次性塞满上下文。

---

## 对照 Hermes：它本质上已经是 Harness

读完后我没急着加文件，先把 Hermes 现有架构拆开对照：

| Harness 概念 | Hermes 对应 | 判断 |
|:---|:---|:---|
| `SKILL.md` + scripts/references/assets | `~/.hermes/skills/` 99 个 skill，完整目录结构 | ✅ 一致 |
| Skills 目录 | 已按 category 分子目录（devops/、creative/ 等） | ✅ 更成熟 |
| References | 每个 skill 内 `references/`（如 hugo-blog 有 31 个） | ✅ 已有 |
| Progressive Disclosure | system prompt 只注入 skill name+desc → `skill_view(name)` 加载全文 → `skill_view(name, file_path)` 加载单个 reference | ✅ 三层已实现 |
| 角色约束 | `MEMORY.md` + `USER.md` + system prompt 身份注入 | ⚠️ 隐式实现 |
| 可版本化 | skills/ 有 git + curator 生命周期 | ✅ |
| 子目录导航 | category 分层 + system prompt 缩进列表 | ✅ |

**结论**：Hermes 不需要引入 Harness 标准——它已经是 Harness 思想的另一种实现。区别只是组织方式不同：

```
Harness 标准:  HARNESS.md → skills/ → references/
Hermes 实际:   system prompt → MEMORY.md/USER.md → skills_list → skill_view → skill_view(file_path)
```

---

## 真正的改善：3 点，零新文件格式

既然已有机制，就用好它们。

### 1. MEMORY.md 加核心 skill 索引

**痛点**：99 个 skill 平铺在 system prompt 里，每次 session 开局 agent 都要从头猜「该加载哪几个」。

**改善**：在 MEMORY.md（每 session 自动注入）加一行：

```markdown
Core skills per session: hugo-blog, cloudflare-api-deploy, hermes-agent, cron-reminder-debugging, daily-execute-blocks, systematic-debugging.
```

效果：session 启动即知道最常用的 6 个 skill，不用靠 99 个列表猜。

---

### 2. STATE.md 高信号搬家

**痛点**：按 Fable 5 框架建的 `~/.hermes/STATE.md` 很完整，但 Hermes **没有自动注入机制**——它永远不会被 session 读到。

**改善**：把高信号内容搬进 MEMORY.md（自动注入）：

- cron 双通道已解决、[SILENT] 规则已移除、AGENTS.md 位置
- 约束行更新（如 `delegation.model` 待设置轻量模型）

STATE.md 保留为诊断日志（§3 Open Failures、§4 Lessons Learned），定位正确。

---

### 3. `/root/hugo-blog/AGENTS.md` 项目级 Harness

**痛点**：写博客、部署、跑 cron 时，agent 要么手动 `skill_view`，要么凭记忆找部署命令。

**改善**：在项目根目录放 `AGENTS.md`。Hermes 的 `terminal(workdir=...)` 和 `cron workdir=...` 会**自动加载**该目录下的 `AGENTS.md` / `CLAUDE.md` 到 system prompt。

内容包含：
- 角色定义：NFY 的个人 AI 助手，主域 Hugo 博客发布 + CF 部署
- 部署 SOP（4 步标准化）
- 6 个核心 skill + 5 个关键 reference 索引表
- 写作规则（日期不超前、本地配图、YAML frontmatter）
- 常见坑表（7 项，如 baseURL、TOML/frontmatter、Vercel 反射性触发等）

现在 `terminal(workdir=/root/hugo-blog)` 一跑，上下文全在了。

---

## 关键设计原则：不引入新标准

这 3 件事**没有**新增任何：
- ❌ 新文件格式
- ❌ 新注入机制
- ❌ 新目录结构
- ❌ 新工具

全部利用 Hermes **已有的注入链路**：

| 改善 | 利用的既有机制 |
|:---|:---|
| MEMORY.md 索引 | `memory` 工具 → 每 session 自动注入 |
| STATE.md 搬家 | 同上 |
| AGENTS.md | `workdir` 参数 → 自动读取项目级上下文 |

Harness 标准里的 `HARNESS.md`、`SKILLS.md`、`REFERENCES.md` 文件，在 Hermes 里分别对应 `MEMORY.md`、`system prompt available_skills`、`skill 内部 references`——**不用重复造轮子**。

---

## 部署验证

```bash
cd /root/hugo-blog
hugo --gc --minify
# baseURL ok ✅ | article rendered ✅

CLOUDFLARE_API_TOKEN=*** node /root/.hermes/node/lib/node_modules/wrangler/bin/wrangler.js \
  pages deploy public/ --project-name=ningop-blog --branch=main --commit-dirty=true
# Deployed ✅

curl -sI https://ningop.com/posts/weekly-2026-w28/ | head -3
# HTTP/2 200 ✅
```

---

## 下周关注

1. **`delegation.model` 设置轻量模型** —— 子代理不再继承主模型（z-ai/glm-5.2），落地模型分工降本
2. **cron 任务验收** —— 观察 14 个提醒在双通道下的送达率
3. **AGENTS.md 实战** —— 下次写博客/部署时看 `workdir` 自动注入的效果

---

## 延伸阅读

- [Agent Harnesses 原文](https://www.drpang.ai/agent-harnesses-ai-agent-standard/)（Daniel Warfield / Medium）
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs/)
- [hugo-blog skill references/series-trilogy-workflow.md](/root/.hermes/skills/devops/hugo-blog/references/series-trilogy-workflow.md)
- [daily-execute-blocks skill](/root/.hermes/skills/productivity/daily-execute-blocks/SKILL.md)