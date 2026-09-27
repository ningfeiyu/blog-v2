---
title: "为什么 AI 助手总是记不住你的博客流程？"
date: 2026-06-09T06:00:00+08:00
draft: false
author: "Ning"
description: "从 TOML frontmatter 残留到配图位置错误，复盘 AI 助手在 Hugo 博客工作流中反复踩坑的 7 个原因，以及 Hermes Agent 的 memory + skill 机制如何解决它。"
tags: ["AI", "Hugo", "工作流", "Hermes", "效率工具"]
categories: ["技术"]
---

![AI Memory Workflow](/images/remote/ext-dedc0a8d649b.jpg)

> 如果你已经第三次教 AI 同样的事情，问题可能不在 AI，而在工作流本身。

## 一、那些反复出现的「低级错误」

和 AI 助手协作写博客一段时间后，我开始注意到一个规律：**同样的 bug 会以不同的面貌反复出现**。

第一次是 `+++` 残留在 YAML frontmatter 之后，导致 Hugo 解析失败，文章虽然能访问但首页根本看不到它。第二次是配图位置放错——AI 把 `![featured]` 放在了二级标题下面，而不是文章开头。第三次是日期设成了未来，Hugo 直接跳过不生成。

每次修复完，我都以为「这次总该记住了」。但下一次，类似的问题又以新的形式出现。

这不是 AI 变笨了。这是**流程的脆弱性**在反复暴露。

## 二、7 个反复踩坑的具体场景

### 1. Frontmatter 格式混乱：YAML 与 TOML 混用

Hugo 支持 YAML (`---`)、TOML (`+++`) 和 JSON 三种 frontmatter。但如果一个项目里大部分文章用 YAML，突然插入一篇 TOML，PaperMod 的列表逻辑可能会跳过它——不是报错，就是**安静地不显示**。

### 2. `+++` 残留：最隐蔽的错误

把 TOML 转成 YAML 时，如果前面的 `---` 忘记删掉，或者后面跟着一串 `+++`，Hugo 不会报错。文章能 build，URL 能访问，但首页和分类列表里**永远找不到它**。`hugo list all` 显示正常，但 `index.html` 的 post-entry 列表里没有。

### 3. 配图位置：一放就错

PaperMod 主题的文章结构是：`<header>` 里的 H1 标题 → `<div class="post-content">` 里的正文。配图必须放在 frontmatter 之后、第一个 heading 之前。稍不注意，配图就会跑到二级标题下面，或者 mid-article 的奇怪位置。

### 4. 日期陷阱：未来不生成，过去不显示

- **未来日期**：Hugo 默认不渲染未来日期的文章，没有任何警告，直接跳过。
- **过去日期**：如果 date 设成一年前，文章 URL 可以正常访问，但 PaperMod 按时间倒序排列，它会沉到列表最底部，用户以为「文章没有发布」。

### 5. 本地 build 成功，线上还是旧的

`hugo --minify` 成功了，`public/` 目录下也有新文件，但 `npx vercel deploy --prod` 后刷新网页，内容还是旧的。原因是 Hugo 的 `resources/` 缓存和 `.hugo_build.lock` 锁住了上一次的状态，需要 `rm -rf public/ resources/ .hugo_build.lock` 彻底清理。

### 6. Vercel cloud build vs prebuilt

直接运行 `npx vercel --prod` 让 Vercel 云端构建，可能遇到 Hugo 版本不匹配、环境变量缺失等问题。**正确的流程是本地 `hugo --gc --minify` 生成静态文件，再用 `vercel build --prod && vercel deploy --prebuilt --prod` 推送**，确保线上和本地完全一致。

### 7. 第三方图片的不可控性

Unsplash 的 `source.unsplash.com/featured` 返回过 503，`images.unsplash.com/photo-<ID>` 也有 ID 失效变成 404 的情况。依赖外部图片，意味着你的文章随时可能丢失配图。

## 三、问题根源：没有固化下来的工作流

上面 7 个问题，每一个都曾在某一轮对话中被「解决」。但当我开启下一次对话，AI 只能依赖**系统提示词里的记忆片段**——一段 2000 字左右的摘要。它没有能力「记住」完整的 20 步部署流程，更不可能记得三个月前某篇文章的配图是怎么放的。

**记忆的本质问题是：它是被动注入的，不是主动执行的。**

每次我告诉 AI「记得配图放开头」，AI 会在当前对话中遵守，但下一条对话/第二天/换了一个模型后，这条指令可能就不见了。

## 四、Hermes Agent 的解决路径

### Memory：事实，不是指令

Hermes 的记忆系统让我可以把「这个项目用 YAML frontmatter」「配图放在 frontmatter 和第一个 heading 之间」「Unsplash 用 photo-ID 格式」这类**事实性信息**持久化。每次对话自动注入，AI 不需要「记」，它只需要「看」。

### Skill：可执行的工作流

比记忆更重要的是 **Skill**。Skill 是一个完整的 Markdown 文件，里面不仅包含「 Hugo 博客部署」的全部步骤，还记录了每个坑的触发条件和修复方法。当我说「写一篇新文章」时，AI 会加载 Skill，按步骤执行，而不是从零开始推理。

Skill 相当于给 AI 写了一份**标准作业程序（SOP）**。它不依赖 AI 的记忆，而是依赖**加载时的确定性**。

### Checklist：在出错前拦截

对于最高频的 3-4 个错误点，可以固化成**前置检查清单**（Checklist）：

1. 文章放在 `content/posts/` 下（不是 `blogs/` 或其他子目录）
2. 日期 ≤ 今天
3. Frontmatter 是纯 YAML，没有 `+++` 残留
4. 配图在 `---` 之后、第一个 `#` 或 `##` 之前
5. 本地 `hugo --gc --minify` 通过后再部署
6. 部署前 `rm -rf public/ resources/ .hugo_build.lock`

## 五、给协作工作流设计的建议

如果你也在用 AI 助手管理长期项目，这里有几条实践经验：

**第一，把「正确答案」写进 Skill，不要口头教。**

口头提醒会在 3 轮对话后消失，但 Skill 每次都会被加载。一次写好永久生效。

**第二，区分「事实记忆」和「临时状态」。**

「这个博客用 PaperMod 主题、部署在 Vercel」是事实记忆，应该持久化。「今天这篇文章写 AI 工具对比」是临时状态，不需要进记忆。

**第三，让 AI 在每次操作后「自检」。**

不是写完后直接说「好了」，而是要求 AI 在提交前执行一个验证步骤：检查 frontmatter 格式、grep 是否有 `+++`、curl 线上确认部署结果。自检比事后返工的成本低得多。

**第四，接受「流程版本迭代」，而不是追求完美一次性。**

每次发现一个坑，就把它写进 Skill 的 Pitfalls 章节。Skill 本身就是一个活的文档，会随时间越来越完善。

## 六、写在最后

AI 助手不是记不住，是**没有一种机制让正确的流程被稳定加载**。

当你第三次修复同样的问题时，真正该做的不是吐槽 AI「又忘了」，而是把这个修复步骤写进 Skill 的 Checklist 里。下一次，它会在 AI 动手之前就自动拦截掉这个错误。

**让流程的稳健性不依赖于 AI 的记忆，而依赖于可复用的工作流设计——这才是人机协作的正确打开方式。**
