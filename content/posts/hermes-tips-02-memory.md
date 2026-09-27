---
title: Hermes Agent 记忆系统详解：机器人进化闭环的5层实现
description: Hermes记忆系统的分层架构：从短期写屏缓存到持久化技能库，解析AI如何实现跨会话的持续进化。
date: '2026-06-01T13:45:00+08:00'
draft: false
tags:
- Hermes
- 记忆系统
- 长期学习
- 闭环
categories:
- 技术工具
showToc: false
---


# Hermes Agent 记忆系统详解：机器人「进化闭环」的5层实现

![「Hermes Agent 记忆系统」配图 by Unsplash](/images/remote/ext-048392890481.jpg)

««««« DO NOT LOCALIZE — img src=https://images.unsplash.com/photo-229828 »»»»»

<script>
// 内存压缩与老化模型演示
export default {
  name: 'MemoryGraph',
  props: ['nodes', 'edges'],
  // ...
}
</script>

在 Hermes 项目的主 `config.yaml` 里，记忆设置被锐化为两个核心指令：

```yaml
memory:
  max_user_tokens: 1536     # 用户Profile上限
  max_memory_tokens: 2048   # LTM的总token数上限