---
title: 自定义 Hermes 技能：打造个性化工作流
description: 通过自定义 Hermes 技能，将重复性任务封装为一键命令，提升个人工作效率与系统一致性。
date: '2026-06-02T12:05:00+08:00'
draft: false
tags:
- Hermes
- 技能
- 自动化
- 工作流
categories:
- 技术工具
showToc: false
---

![Hermes Skills 示例](/images/remote/ext-3ba60f61cee1.jpg)


<img src="/images/posts/QvVI6vAqtg8.jpg" alt="通过自定义 Hermes 技能，将重复性任务封装为一键命令，提升个人工作效率与系统一致性。" loading="lazy" style="max-width:100%;height:auto;">

## 为什么需要自定义技能？
开发者的日常工作中充满重复性任务，如部署博客、格式化代码、备份数据等。通过将这些任务封装为 Hermes **技能（Skills）**，可以实现一键执行，节省时间。

## 技能的基础结构
Hermes 技能由以下部分组成：
- **元数据**：技能名称、描述、触发条件（YAML 前言）。
- **步骤**：具体的执行逻辑，支持工具调用和条件分支。
- **引用文件**：如脚本、模板或配置文件（可选）。

技能存储在 `~/.hermes/skills/` 目录下，每个技能是一个子目录，包含 `SKILL.md` 文件。

## 创建第一个技能：自动部署 Hugo 博客
### 第一步：创建技能目录
```bash
mkdir -p ~/.hermes/skills/deploy-blog
cd ~/.hermes/skills/deploy-blog
```

### 第二步：编写 `SKILL.md`
```markdown
---
name: deploy-blog
description: "一键构建并部署 Hugo 博客到 Vercel"
---
1. 进入博客目录：
   ```bash
   cd /root/hugo-blog
   ```

2. 构建静态文件：
   ```bash
   hugo --minify
   ```

3. 部署到 Vercel：
   ```bash
   npx vercel deploy --prod --yes
   ```
```

### 第三步：测试技能
```bash
hermes run deploy-blog
```

## 进阶技巧
### 1. 动态参数传递
在技能步骤中使用变量：
```bash
hermes run deploy-blog --env "production"
```
并在 `SKILL.md` 中引用：
```bash
npx vercel deploy --prod --env {{env}} --yes
```

### 2. 多步骤工作流
将复杂任务分解为子步骤，Hermes 会按顺序执行：
```markdown
1. 备份数据库
   ```bash
   mysqldump -u user -p db_name > backup.sql
   ```

2. 压缩备份文件
   ```bash
   gzip backup.sql
   ```

3. 上传到云存储
   ```bash
   aws s3 cp backup.sql.gz s3://my-backups/
   ```
```

### 3. 错误处理
在技能中添加错误检查逻辑：
```bash
if [ $? -ne 0 ]; then
  echo "部署失败，请检查日志！"
  exit 1
fi
```

## 常见问题
**Q: 如何调试技能？**
A: 使用 `hermes run --debug <skill-name>` 查看详细日志。

**Q: 技能存储在哪里？**
A: 默认在 `~/.hermes/skills/`，可通过 `hermes skills path` 查看。

## 实战案例：博客图片优化技能
一键压缩博客目录下所有图片并替换原文件：
```bash
#!/bin/bash
find /root/hugo-blog/static/images -type f \( -iname "*.jpg" -o -iname "*.png" \) \
  -exec sh -c 'cwebp -q 80 "$0" -o "${0%.*}.webp" && rm "$0"' {} \;
```
封装为技能后，只需运行 `hermes run optimize-images`。

--- 
> 💡 **提示**：将常用的技能添加到 `~/.bashrc` 或 `~/.zshrc` 中，以便在任意目录下调用。
