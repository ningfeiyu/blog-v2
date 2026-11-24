---
title: 工作日志：152 篇草稿、89 条密钥与九次重置
description: 11 月 21-24 日工作小结：微信公众号 152 篇进草稿箱、Infisical 导入 89 条密钥、LLM 免费额度逐个实测、d1/d2 卸载 TrustTunnel；本机 VM 三天重置九次。
date: '2026-11-24T11:59:39+08:00'
draft: false
tags:
- 工作日志
- 复盘
categories:
- 工作日志
showToc: false
image: "/images/covers/worklog-2026-11-24-wechat-sync.jpg"
---

![工作日志：152 篇草稿、89 条密钥与九次重置](/images/covers/worklog-2026-11-24-wechat-sync.jpg)

11 月 21 日到 24 日，四天干了四件大事，外加一台总重置的虚拟机。按事记，不按天记。

## 微信公众号：152 篇进了草稿箱

11 月 23 日晚，博客 152 篇文章全部同步进微信公众号草稿箱，只建草稿、不群发。调接口那步放在固定 IP 的 g1 上跑，公众号白名单只加这一个 IP；文章从 GitHub 拉。

踩了三个坑，都是微信的硬限制：

- 标题限 64 字节、摘要限 120 字节，按字节算不是按字算，超了就报 45003/45004。
- 正文相对链接报 45166，全库 312 处改成绝对链接。
- JSON 必须 raw UTF-8 发，\u 转义会被微信原样存成垃圾文本。

4 篇失败也修好了：hermes 那篇正文嵌了一个 .html 交互图，微信当图片拒了；两篇 SVG 含裸 `<` 与 `&`，转图时转义；snell 是状态记错了，移除后重推。

作者名正在从宁飞改成 Nfy，141/152 已改，剩 11 篇被腾讯 WAF 拦了（HTTP 501），短时间改太多触发的限流，11 月 25 日早上再试。

## 密钥搬家：89 条进了 Infisical，24 条没了

同一天，把 .env 里的密钥倒进 Infisical（项目 homelab，环境 prod），89 条、10 个夹，逐项核对无缺。Groq 按令两边清净。

但 16:40 复查发现 24 条没了：/nimcli 从 28 剩 9，/ccswitch 5 条全空。dev、staging、根目录都没有，不是挪走。原因未明，待我自己确认是不是在网页上删的。本地 .env 里值都在，能一条命令导回。

自动轮换问过了：Infisical 自带的只覆盖数据库、AWS、Stripe、OpenAI 等固定目标，Cloudflare、GitHub、NIM 这些不在列表里；自动轮换是 Pro 功能，20 美元/身份/月；自托管最低 2 核 4G，g1 不够。还没拍板。

## LLM 免费额度：逐个实测了一遍

NVIDIA 的模型目录大换血：kimi-k3、glm-5.3、deepseek-v4.1-flash 都 410 Gone 了。现存能用的是 nemotron 三个（lightning、super-vl、bolt）。writekit 实测：链首 step-5 活，备 2 nemotron-super-vl 活，备 1 minimax 60 秒无首字暂时用不了。中文长文走 step-5 → nemotron-super-vl。

能用的：Google gemini-3.5-flash-lite（500/天）、OpenRouter 免费模型（50/天）、Kilo Code（200/小时）、GLM 直连 glm-4-flash。暂时不行：AMD 两次 429，OpenCode Zen 密钥过期。Groq 已停用。

## d1/d2：TrustTunnel 卸载干净

11 月 23 日上午，按令把 d1、d2 的 TrustTunnel 彻底卸载：服务、二进制、持久盘副本、恢复脚本引用、g1 转发、DNS 记录全清，备份在各自持久盘。q1、q2、vn1 的不动。外网验收 tt-d1、tt-d2 均回 000，SSH 通道与 cloudflared、gotty 不动。

教训：卸载脚本里 `pkill -f` 匹配到脚本自身，把自己杀了，后半段没跑但退码显示成功。改用 `pkill -x` 精确进程名，逐项复核。

## 虚拟机：三天重置九次

11 月 21 日三次、22 日四次、23 日两次。本机 VM 根是 overlay，重置后看门狗和 bootstrap 自动拉回 10 个服务，公网三站验 200。开机自恢复仍未补，下次重置仍会断。

k1、k2 在 11 月 22 日按令重启并验活。但 k1 的甲骨文保活容器同日 15:00 以 255 退出、日志为空，启动脚本没带 --restart，修法已报，等一句话才动手。

## 其他

- 谢霆锋合辑 246 个 .lrc，手机上 GBK 转 UTF-8，0 失败。
- UISP Console：大陆官网无 UISP 系列，美区 $99 只发美国，B&H $149 支持直邮中国。未下单。
- 羊毛盯梢每天 8 轮，0 候选静默；最高分仍是 MobiOffice 79 分，未达 90 分线。

四天里最值钱的两件事都做完了：152 篇进了草稿箱，89 条密钥进了保险库。剩下的尾巴——11 篇作者、24 条密钥、k1 保活容器——都在等一个冷却、一个确认、一句话。
