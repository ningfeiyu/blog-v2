---
title: "Notion KB与Cron集成：从Hermes Gateway到System Crontab的迁移实录"
date: 2026-09-06T19:45:00+08:00
categories: [技术实录]
tags: [Notion, Cron, Systemd, Python, Hermes, 自动化]
---

![featured](/images/posts/2026-09-06-notion-kb-cron-integration-migration.jpg)

## 问题背景：Hermes Cron Gateway 的 systemd 限制

Hermes 的 cron 系统依赖 `systemd-run --user --scope` 启动隔离 worker。当前环境（容器/权限受限）报错：

```
Restart-safe cron worker dispatch failed: cannot create restart-safe systemd scope for gateway child: systemd-run --user --scope is unavailable
```

导致 18 个 Hermes cron jobs 全部失败，无法执行任何定时任务。

## 解决方案：迁移到系统级 crontab

直接编写 `/root/.hermes/cron/system-crontab` 并 `crontab /root/.hermes/cron/system-crontab` 安装，绕过 Hermes gateway，由系统 cron 直接调用 Python 脚本。

```bash
# 安装
crontab /root/.hermes/cron/system-crontab

# 查看
crontab -l
```

## 核心脚本

### 1. Notion KB 每日全量同步 (`sync-notion-kb.py`)

```python
#!/usr/bin/env python3
# 使用 Notion API 直连，拉取「人生规划」页面及其子页面
# 环境变量：NOTION_API_KEY
# 输出：/root/.hermes/kb/notion/*.md + LIVE-INDEX.txt + UNIFIED-INDEX.txt
```

- 运行时间：`0 3 * * *` (每日 03:00)
- 结果：33/33 页面成功，LIVE-INDEX 93 行，UNIFIED-INDEX 427 行
- 日志：`/root/.hermes/cron/system-logs/sync-notion-kb.log`

### 2. 双向写回模块 (`writeback_notion.py`)

```python
# 支持三种模式
writeback(page_id, title, content, mode='date-toggle')
# date-toggle: 同一天同标题创建 toggle 块，后续追加到 children
# append: 直接追加
# overwrite: 覆盖
```

4 个 preset 配置：

```python
PRESETS = {
    'identity_log':      {'page_id': '35837db8-39f5-8017-b508-da3145200783', 'title': '晨间身份锚定'},
    'afternoon_report':  {'page_id': '3ce37db8-39f5-811c-8a2a-d40c57c978eb', 'title': '下午执行报告'},
    'daily_review':      {'page_id': '3d037db8-39f5-80c1-b1b7-c8b729a7a5a9', 'title': '每日复盘'},
    'scorecard':         {'page_id': '3d037db8-39f5-80c2-a744-c12248e2c17e', 'title': '积分卡'},
}
```

### 3. 索引重建 (`rebuild-index.py`)

```python
# 扫描 /root/.hermes/kb/notion/*.md
# 生成 LIVE-INDEX.txt (Notion 专用格式)
# 生成 UNIFIED-INDEX.txt (博客+Notion 合并，供 article-knowledge-layer skill 使用)
```

## 4 个 Core Cron Skills 植入 Notion 写回

| Skill | 触发时间 | 写回页面 | Page ID |
|-------|---------|---------|---------|
| cron-identity-coaching | 05:30 | 晨间身份锚定 | 35837db8-39f5-8017-b508-da3145200783 |
| cron-afternoon-execution | 13:00 | 下午执行报告 | 3ce37db8-39f5-811c-8a2a-d40c57c978eb |
| cron-evening-wind-down | 19:30 | 每日复盘 | 3d037db8-39f5-80c1-b1b7-c8b729a7a5a9 |
| cron-scorecard | 21:25 | 积分卡 | 3d037db8-39f5-80c2-a744-c12248e2c17e |

每个 skill 的 YAML frontmatter 末尾追加：

```markdown
## Notion 写回（执行结束自动调用）

```bash
/root/.hermes/scripts/writeback_notion.py \
  --preset identity_log \
  --title "晨间身份锚定" \
  --content "$(cat /path/to/identity_log.md)" \
  --mode date-toggle
```
```

## System Crontab 完整任务表

```crontab
# Notion KB 每日全量同步
0 3 * * * /usr/bin/python3 /root/.hermes/scripts/sync-notion-kb.py >> /root/.hermes/cron/system-logs/sync-notion-kb.log 2>&1

# 晨间身份锚定 (05:30)
30 5 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-identity-coaching.log 2>&1

# 晨间运动+正念+早餐 (06:05)
5 6 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-morning-exercise.log 2>&1

# 下午执行开始·记录计划 (13:00)
0 13 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-afternoon-execution.log 2>&1

# SOP 完善/复盘提醒 (16:10)
10 16 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-afternoon-sop.log 2>&1

# 晚间沐浴+晚餐+复盘 (19:30)
30 19 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-evening-wind-down.log 2>&1

# 奖赏循环检查 (21:20)
20 21 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-reward-loop.log 2>&1

# 积分卡/回纹针/诱惑捆绑 (21:25)
25 21 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-scorecard.log 2>&1

# 睡前身份锚定 (21:30)
30 21 * * * /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-sleep-identity-anchor.log 2>&1

# 周复盘/规划 (周一 06:00)
0 6 * * 1 /usr/bin/python3 -c "..." >> /root/.hermes/cron/system-logs/cron-weekly-review.log 2>&1

# @reboot 保证日志目录存在
@reboot mkdir -p /root/.hermes/cron/system-logs
```

## AMD Qwen3.8-Flash-Next 接入 Hermes Fallback 链

### 1. 存储 API Key

```bash
# 存储到 .env (本地环境变量文件，不提交版本库)
echo "AMD_API_KEY=<your-key-here>" >> /root/.hermes/.env
```

### 2. 更新 config.yaml

```yaml
fallback_providers:
  # ... 前 9 个 provider ...
  - provider: custom
    model: Qwen3.8-Flash-Next
    base_url: https://developer.amd.com.cn/radeon/api/v1
    api_key: ${env:AMD_API_KEY}
  - provider: custom
    model: deepseek-v4-flash
    base_url: https://router.flatkey.ai/v1
    api_key: ${env:FLATKEY_API_KEY}
```

现在 fallback 链共 11 个 provider，AMD 位于第 10 位。

## 完整数据流向

```
┌─────────────────────────────────────────────────────────────────┐
│  Notion (人生规划页面 ID: 35837db8-39f5-80ad-836d-cbbc590cf44f)  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 03:00 sync-notion-kb.py (Notion API)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  本地 KB: /root/.hermes/kb/notion/                              │
│    ├── *.md (33 个页面)                                          │
│    ├── LIVE-INDEX.txt (93 行，Notion 专用索引)                   │
│    └── UNIFIED-INDEX.txt (427 行，博客+Notion 合并索引)          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 各时间点 writeback_notion.py (date-toggle)
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ 晨间身份锚定    │ │ 下午执行报告    │ │ 每日复盘        │ │ 积分卡
│ (05:30)         │ │ (13:00)         │ │ (19:30)         │ │ (21:25)
└─────────────────┘ └─────────────────┘ └─────────────────┘ └─────────────────┘
          │                │                │                │
          └────────────────┴────────────────┴────────────────┘
                           ▼
              Notion 对应页面追加到今日 toggle 块的 children
```

## 关键命令速查

```bash
# 手动触发同步
/usr/bin/python3 /root/.hermes/scripts/sync-notion-kb.py

# 手动写回测试
/usr/bin/python3 -c "
import sys; sys.path.insert(0, '/root/.hermes/scripts')
from writeback_notion import writeback
from datetime import datetime
content=f'测试: {datetime.now().strftime(\"%H:%M\")}'
writeback(page_id='35837db8-39f5-8017-b508-da3145200783', title='晨间身份锚定', content=content, mode='date-toggle')
"

# 查看系统 cron 日志
tail -f /root/.hermes/cron/system-logs/*.log

# 重新加载 system crontab
crontab /root/.hermes/cron/system-crontab

# 验证 Hermes fallback 链
grep -A3 "fallback_providers" /root/.hermes/config.yaml
```

## 经验总结

1. **systemd 限制不可控时，系统 crontab 是最可靠的 fallback** — 无依赖、无 gateway、直接执行
2. **Notion API 直连比 ntn CLI 更稳定** — bash 脚本易碎，Python requests + 重试机制成功率 100%
3. **date-toggle 模式完美适配每日重复性写回** — 避免重复创建页面，同一天多次执行自动追加
4. **Skills 作为逻辑载体，crontab 作为调度器** — 解耦执行逻辑与调度机制，便于迁移和维护
5. **Fallback 链要有冗余** — AMD 模型虽当前 503，但作为第 10 位 provider 不影响主链路

---

*技术栈：Python 3.11 + Notion API + systemd-free crontab + Hermes Agent*