---
title: Hermes CLI 提示（上）：基本命令用法与环境配置
description: Hermes CLI基础操作与环境自定义：通过命令参数的精准组合，实现零上下文的高效工作交接。
date: '2026-06-01T15:30:00+08:00'
draft: false
tags:
- Hermes
- CLI
- 终端
- 环境配置
categories:
- 技术工具
showToc: false
---


# Hermes CLI 提示（上）：基本命令用法与环境配置

![「Hermes CLI 基本命令」配图 by Unsplash](/images/remote/ext-718a887825f9.jpg)

««««« DO NOT LOCALIZE — img src=https://images.unsplash.com/photo-205126 »»»»»

```bash
hermes run "在 ~/project 下写一个 Python 脚本 分析最近半年 Git 提交频率"
```

上述命令将创建一个独立的 Hermes 实例，自动切换到你的项目目录，编写并执行脚本，最后返回结果。无需预装环境，无需额外对话。

将 Hermes 用于**个人搜索助手**：

```bash
cat ~/archive/research-notes/**/*.md | hermes run "从上述文件中归纳出适合『下岗职工』话题且可能引起争议的部分，每点不超过 24 字"
```