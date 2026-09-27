---
title: "Hermes Agent Skill 完全指南：从入门到自定义"
date: 2026-06-18T12:00:00+08:00
draft: false
tags: ["hermes", "skill", "workflow", "教程"]
categories: ["技术"]
description: "深入解析 Hermes Agent 的 Skill 系统，涵盖内置技能使用、自定义技能创建和最佳实践。"
---

![风帆](/images/remote/ext-40d8d0c348e6.jpg)

> Skill 不是魔法，它是经验的复用。

如果你用过 Hermes Agent，一定见过 `skill_view("hugo-blog")` 这样的调用。这就是 **Skill**——Hermes 的核心扩展机制。本文将带你从使用者变成创造者。

---

## 一、Skill 是什么？

简单说：**Skill 是可复用的工作流模板。**

想象你每次写博客都要重复：
1. 创建 `content/posts/xxx.md`
2. 填写 frontmatter
3. 找配图
4. `hugo --minify`
5. 部署

这些步骤完全可以固化成一个 skill，以后只需要说"发篇文章"，Agent 自动走完流程。

---

## 二、Skill 的三种来源

### 1. 内置 Skill（官方维护）

```bash
# 查看所有可用 skill
skills_list()

# 查看特定分类
skills_list(category="devops")

# 加载具体 skill 内容
skill_view("hugo-blog")
skill_view("hugo-blog", file_path="references/deploy.md")
```

内置 skill 存放在 `~/.hermes/skills/` 目录，按分类组织：

```
~/.hermes/skills/
├── devops/
│   ├── hugo-blog/
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   └── sensitive-topic-writing-guide.md
│   │   └── templates/
│   │       └── post-template.md
│   └── astro-vercel-deployment/
├── github/
│   └── codebase-inspection/
│       ├── SKILL.md
│       └── scripts/
│           └── count-lines.py
└── ...
```

### 2. 插件 Skill（第三方扩展）

插件的 skill 用 `插件名:skill名` 格式引用：

```python
skill_view("superpowers:writing-plans")
```

### 3. 自定义 Skill（你自己创建）

这是本文的重点。

---

## 三、创建你的第一个 Skill

### 3.1 最简结构

一个最小 skill 只需要 `SKILL.md`：

```markdown
---
title: "My First Skill"
name: "my-skill"
description: "Does something useful."
category: "productivity"
tags: ["automation"]
version: 1.0
---

# My First Skill

## When to Use
When the user asks for X.

## Steps
1. Do A
2. Do B
3. Done
```

### 3.2 实战：创建一个"写周报"的 Skill

```bash
mkdir -p ~/.hermes/skills/productivity/weekly-report
```

写入 `SKILL.md`：

```markdown
---
title: "Weekly Report Generator"
name: "weekly-report"
description: "Generate professional weekly work reports from bullet points."
category: "productivity"
tags: ["report", "automation"]
version: 1.0
---

# Weekly Report Generator

## When to Use
When the user provides a list of work items and wants a formatted weekly report.

## Input Format
- Bullet points of completed tasks
- Optional: blockers / next week's plan

## Output Format
1. **本周完成**
2. **遇到的问题**
3. **下周计划**

## Example

Input:
- 完成了 API 重构
- 修复了登录 Bug

Output:
> ### 本周完成
> - 完成 API 重构，提升响应速度 30%
> - 修复登录 Bug，解决用户反馈问题
>
> ### 下周计划
> - 优化数据库查询
```

创建：

```python
skill_manage(action="create", name="weekly-report", category="productivity", content="...")
```

---

## 四、Skill 的高级用法

### 4.1 关联文件

Skill 可以附带模板、脚本、参考资料：

```python
# 添加参考文档
skill_manage(
    action="write_file",
    name="my-skill",
    file_path="references/api-reference.md",
    file_content="..."
)

# 添加脚本
skill_manage(
    action="write_file",
    name="my-skill",
    file_path="scripts/deploy.sh",
    file_content="..."
)
```

加载时指定：

```python
skill_view("my-skill")  # 加载 SKILL.md
skill_view("my-skill", file_path="scripts/deploy.sh")  # 加载脚本
```

### 4.2 更新 Skill

```python
# 打补丁
skill_manage(
    action="patch",
    name="my-skill",
    old_string="version: 1.0",
    new_string="version: 1.1"
)

# 或完整重写
skill_manage(
    action="edit",
    name="my-skill",
    content="..."
)
```

### 4.3 删除 Skill

```python
skill_manage(action="delete", name="my-skill")
```

---

## 五、最佳实践

| 原则 | 说明 |
|------|------|
| **单一职责** | 一个 skill 只做一件事 |
| **自包含** | 不依赖外部状态，所有参数通过 frontmatter 或输入提供 |
| **可验证** | 包含明确的"成功/失败"判断标准 |
| **版本管理** | 重大更新时递增 version |

---

## 六、案例分析：context-management

这是我在实际工作中创建的 skill，用于解决 **长任务上下文耗尽** 问题。

**核心设计**：

```markdown
## Execution Protocol

### Step 1: Before Any Action
1. Analyze the request
2. Plan the minimal viable path
3. Decompose into smallest independent subtasks

### Step 2: Single-Step Execution
- Only ONE subtask at a time
- After every tool call, STOP and summarize

### Step 3: Post-Step Summary
- Success: < 300 token summary, discard raw output
- Failure: error + next attempt plan

### Step 4: Context Pruning
If context > 30k tokens:
  1. Pause
  2. Summarize current state (< 200 tokens)
  3. Prune history
  4. Resume
```

**使用时的效果**：
- 写一篇文章，原本可能占用 15k tokens
- 使用这个 skill 后，每步只保留 300 token 总结，总占用 < 5k
- 剩余空间可用于更深度的内容创作

---

## 七、快速参考

| 操作 | 命令 |
|------|------|
| 列出 skill | `skills_list()` |
| 查看 skill | `skill_view("name")` |
| 创建 skill | `skill_manage(action="create", ...)` |
| 更新 skill | `skill_manage(action="patch", ...)` |
| 删除 skill | `skill_manage(action="delete", ...)` |
| 添加文件 | `skill_manage(action="write_file", file_path="...", ...)` |

---

## 结语

Skill 是 Hermes Agent 的"肌肉记忆"。

当你第一次做某件事时，一步步来；当你第十次做时，把它变成 skill；当你第一百次做时，让 skill 自动运行。

从今天开始，把你重复的工作流固化成 skill 吧。
