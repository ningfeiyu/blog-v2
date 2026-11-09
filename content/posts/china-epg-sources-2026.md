---
title: 中国大陆电视 EPG（电子节目单）推荐：目前较常用的优质源
description: 2026 年信息，免费公开 XMLTV / DIYP 接口为主：epg.pw、laobaiepg、51zmt、ERW、112114 及镜像，附使用建议。
date: '2026-11-09T11:00:00+08:00'
draft: false
tags:
- EPG
- IPTV
categories:
- 工具
showToc: false
image: "/images/covers/china-epg-sources-2026.jpg"
---

![中国大陆电视 EPG（电子节目单）推荐：目前较常用的优质源](/images/covers/china-epg-sources-2026.jpg)

看电视直播，节目单是刚需。下面是目前较常用的几个免费公开 EPG 源，以 XMLTV / DIYP 接口为主，2026 年信息。

## 1. epg.pw（强烈推荐，覆盖全、更新稳定）

- 中国专用：`https://epg.pw/xmltv/epg_CN.xml`，或压缩版：`https://epg.pw/xmltv/epg_CN.xml.gz`
- 完整全球：`https://epg.pw/xmltv/epg.xml`（或 `.gz`）
- Lite 轻量版也有。

数据相对干净，时区已校正为 +0800，适合大多数播放器（TiviMate、IPTVnator、APTV、Kodi 等）。

## 2. laobaiepg（多源聚合，推荐优先试）

- 完整：`https://laobaiepg.laobaitv.net/guide.xml.gz`（推荐压缩版）
- 仅大陆频道：`https://laobaiepg.laobaitv.net/guide_mainland.xml`
- 港澳台：`https://laobaiepg.laobaitv.net/guide_hktw.xml`

聚合了 epg.pw、CNTV 官方、电视猫、看看新闻等多个源，自动换源，覆盖央视 + 卫视 + 地方台较好，更新及时。

## 3. 老张的 EPG（51zmt，国内老牌稳定）

- 总表：`http://epg.51zmt.top:8000/e.xml`（或 `.gz`）
- 央视 + 卫视：`http://epg.51zmt.top:8000/cc.xml`
- 地方 / 数字：`http://epg.51zmt.top:8000/difang.xml`

国内访问友好，长期维护。国内用户可用 Gitee 镜像（更稳）：`https://gitee.com/taksssss/tv/raw/main/epg/51zmt.xml.gz`（还有 51zmte1 / 51zmte2 等变体）。

## 4. ERW（e.erw.cc，支持多天回看）

- 当天：`https://e.erw.cc/e.xml`（或 `.gz`）
- 七天回看（含当天）：`https://e.erw.cc/all.xml`（或 `.gz`）
- 央卫精简：`https://e.erw.cc/cc.xml` / `allcc.xml`

适合需要回看功能的场景。Gitee 镜像也有：`https://gitee.com/taksssss/tv/raw/main/epg/erw.xml.gz`

## 5. 112114 + 镜像（DIYP / TVBox 友好）

- 原站：`https://epg.112114.xyz/pp.xml`（或 DIYP 接口直接填 `https://epg.112114.xyz`）
- 可靠镜像（推荐）：
  - fanmingming：`https://live.fanmingming.cn/e.xml` 或 `https://live.fanmingming.com/e.xml`
  - Gitee：`https://gitee.com/taksssss/tv/raw/main/epg/112114.xml.gz`

原站有时不稳定或被墙，建议优先用镜像。

## 使用建议

- **优先顺序**：先试 **laobaiepg 大陆版** 或 **epg.pw CN**，再备 51zmt / ERW。
- 播放器里填 EPG 地址后，频道名尽量匹配（CCTV1、湖南卫视等标准名称），匹配率才高。
- 国内网络建议优先用 **Gitee 镜像** 或带 `.gz` 的压缩地址，加载更快更稳。
- 这些都是公开免费源，实际准确率和覆盖会随上游变化，建议多备几个轮换。
