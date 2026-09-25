---
title: "第一篇：这个博客是怎么上线的"
date: 2026-09-25T17:30:00+08:00
tags: ["折腾"]
description: "一行 Python 内置 HTTP 服务 + Cloudflare Tunnel + 一条 API 调用的 DNS 记录。"
---

整个链路只有三步：

1. VM 本机跑一个静态文件服务，监听 `127.0.0.1:18080`，连 Python 都是系统自带的。
2. 用 Cloudflare API 给已有的 `muse` 隧道加一条 ingress 规则：`blog.ning.indevs.in → http://localhost:18080`。
3. 再用 API 加一条 DNS CNAME：`blog.ning.indevs.in → <tunnel-id>.cfargotunnel.com`（proxied）。

访问者 → Cloudflare 边缘 → 隧道 → 本机 18080 端口，全程 TLS，VM 本身不需要任何公网端口。

根文件系统被重置也不怕：systemd unit 模板存在 `/home/hatch/systemd-units/`，bootstrap 脚本一键恢复。

## 升级：换成 Hugo

现在博客改用 [Hugo](https://gohugo.io/) 构建：源码是 Markdown，`hugo` 一条命令生成纯静态页面，主题是手写的极简模板，零外部依赖。以后写文章只需要在 `content/posts/` 里丢一个 `.md` 文件，再跑一次构建。
