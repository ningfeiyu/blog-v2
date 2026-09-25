---
title: "隧道穿透笔记：cloudflared 与 egress 代理"
date: 2026-09-25T17:35:00+08:00
tags: ["网络"]
description: "本地 DNS 被劫持、UDP 受限、直连 TLS 被拦截的环境里，cloudflared 依然能连上 Cloudflare 边缘。"
---

这台 VM 的网络环境相当「敌对」：本地 DNS 全被劫持、UDP 基本不可用、直连 TLS 会被中间人拦截。

但 cloudflared 依然 work，靠的是三件套：

- **dnsstub**：本地 53 端口，伪造 `region*.v2.argotunnel.com` 解析到 127.0.0.1，其余走 DoH 转发；
- **edge-relay**：把发往 127.0.0.1:7844 的流量经 egress 代理 CONNECT 到 Cloudflare 边缘；
- **gotty-cloudflared**：真正跑 tunnel 的 cloudflared，被约束只能用本地 DNS。

结论：只要出站有一条可用的 HTTP CONNECT 代理，命名隧道就能注册成功，QUIC/UDP 并非必需。
