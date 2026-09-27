---
title: "Hermes Agent 进阶玩法：从安装到自动化的完整上手指南"
date: 2026-06-27T00:30:00+08:00
draft: false
author: "Ning"
description: "给刚安装 Hermes Agent 的新手：配置多模型路由、落地 Skills、跑通 Terminal/Browser/MCP 工具链、写 Cron 自动化、用 Subagents 并行任务。配架构图全景一张图看懂全链路。"
tags: ["Hermes Agent", "AI Agent", "自动化", "MCP", "工具链", "本地优先"]
categories: ["AI技术", "工具指南"]
showToc: true
TocOpen: false
---

![Hermes Agent Architecture](/images/posts/hermes-architecture.html)

> Hermes Agent 是一款**本地优先、配置驱动、技能可插拔**的 AI Agent 框架。它把“模型网关 + 工具执行引擎 + 技能系统 + 持久化记忆”全部跑在你自己的机器上，配一个 YAML 就能把 Telegram、终端、浏览器、MCP 服务器、Cloudflare Workers 全串起来。这篇文章把新手最容易卡的配置、技能、工具、自动化四大块拆开讲，配一张全景架构图帮你建立心智模型。

## 一、核心心智模型：五层架构

打开文章顶部的架构图（或在浏览器打开 `/images/posts/hermes-architecture.html`），你会看到五层：

| 层级 | 职责 | 关键词 |
|------|------|--------|
| **UI Layer** | Telegram / Web UI / CLI | 统一入口，同一会话多端同步 |
| **Gateway & Routing** | 模型路由、Fallback 链、Config/Profile/Session/Memory 管理 | `config.yaml` 是唯一真理来源 |
| **Provider Layer** | NVIDIA / OpenRouter / Anthropic / Cloudflare AI / ZAI / Custom | 统一 OpenAI 兼容接口，支持任意厂商 |
| **Tool Engine** | Terminal / File / Browser / Web Search / Skills / MCP | 全部本地执行，沙箱隔离，支持后台+通知 |
| **Extensions** | Cron Jobs / Delegation / Platform Bridges / CF Workers / MCP Servers | 定时、并行、跨平台、边缘计算 |

> **一句话心法**：`config.yaml` 定义**“用什么模型、怎么路由、开什么工具、加什么技能”**，剩下的 Hermes 全自动帮你接线。

---

## 二、安装与首发配置（5 分钟跑通）

### 1. 一键安装

```bash
curl -fsSL https://hermes-agent.dev/install.sh | bash
# 或手动：下载对应平台的二进制，放入 PATH
```

安装后会在 `~/.hermes/` 生成：
```
~/.hermes/
├── config.yaml          # 主配置（必改）
├── profiles/            # 多工作区隔离
├── skills/              # 技能目录（drop-in 安装）
├── plugins/             # 插件目录
├── cron/                # 定时任务
└── sessions.db          # SQLite + FTS5 全文检索
```

### 2. 编辑 `~/.hermes/config.yaml` —— 只改三处

```yaml
model:
  default: nvidia/nemotron-3-ultra-550b-a55b
  provider: nvidia
  base_url: https://integrate.api.nvidia.com/v1

# 关键：fallback 链（主模型挂了自动切）
fallback_providers:
  - provider: nvidia
    model: nvidia/nemotron-3-ultra-550b-a55b
    base_url: https://integrate.api.nvidia.com/v1
  - provider: openrouter
    model: nvidia/nemotron-3-ultra-550b-a55b:free
    base_url: https://openrouter.ai/api/v1
    api_mode: chat_completions

# 关键：把常用工具集打开
toolsets:
  - hermes-cli
  - terminal
  - file
  - browser
  - web
  - skills
  - cronjob
  - delegation
```

### 3. 填 Key（环境变量优先，不写进 YAML）

```bash
# ~/.bashrc 或 ~/.zshrc
export NVIDIA_API_KEY="nvap...port OPENROUTER_API_KEY="sk-o...port ANTHROPIC_API_KEY=***  # 可选
export CLOUDFLARE_API_TOKEN=***  # 可选，用于 CF Workers 部署
export ZAI_API_KEY=***                # 可选
```

> **安全提示**：`config.yaml` 里只写 `base_url` 和 `model` 名，**绝不写 key**。Key 全靠环境变量注入，进程退出即失效。

### 4. 验证跑通

```bash
hermes chat "你好，给我讲个笑话"   # CLI 快测
hermes tui                         # 交互式 TUI（推荐日常用）
```

看到回复即成功。`hermes tui` 里敲 `/help` 能看到所有内置命令。

---

## 三、技能系统：把“会的事”变成“可复用的包”

技能就是 `~/.hermes/skills/<skill-name>/SKILL.md` + 可选的 `references/` `templates/` `scripts/`。安装方式极简：

```bash
# 方式 1：手动建目录
mkdir -p ~/.hermes/skills/my-skill/{references,templates,scripts}
# 写 SKILL.md，参考已有技能

# 方式 2：用技能管理命令（如果有）
hermes skill install github:user/repo#skill-name
```

### 新手必装的 5 个官方技能

| 技能名 | 用途 | 典型场景 |
|--------|------|----------|
| `hugo-blog` | Hugo 博客全流程 | 写文章、配图、部署 CF Pages |
| `architecture-diagram` | 生成暗黑风架构图 | 画系统图、云拓扑、数据流 |
| `native-mcp` | 原生 MCP 客户端 | 接入任意 MCP Server（stdio/HTTP） |
| `delegation` | 子任务并行 | 大任务拆解给多个 Subagent 同时跑 |
| `cronjob` | 定时 Agent | 每日早报、监控告警、定时部署 |

> **技能加载是按需的**：对话里提到相关关键词，Hermes 会自动建议加载；也可手动 `skill_view(name="xxx")` 拉取。

### 自己写技能的最小模板

```
~/.hermes/skills/我的技能/
├── SKILL.md          # 必须：YAML frontmatter + markdown 说明
├── references/       # 可选：参考文档、API 规范
├── templates/        # 可选：代码/配置模板
├── scripts/          # 可选：可执行脚本
└── assets/           # 可选：图片、二进制
```

`SKILL.md` 头部示例：

```yaml
---
name: my-skill
description: "一句话描述技能能做什么"
version: 1.0.0
author: Your Name
tags: [tag1, tag2]
metadata:
  hermes:
    triggers: ["关键词1", "关键词2"]  # 对话触发自动加载
---
```

后面跟 Markdown 说明：**触发条件、分步操作、避坑指南、验证步骤**。写得越细，Agent 用得越准。

---

## 四、工具链实战：Terminal / Browser / File / MCP

### 1. Terminal Tool —— 本地 Shell 的完全控制权

```python
# 在 Agent 里（或通过 hermes chat 触发）
terminal(command="cd /root/project && npm run build", timeout=300)
# 后台长任务（编译、部署、测试）
terminal(command="./long-build.sh", background=true, notify_on_complete=true)
# 交互式 REPL（python、node、psql）
terminal(command="python3", pty=true)
```

**新手必知三招**：
- `background=true + notify_on_complete=true` = 启动即忘，完成自动通知
- `process(poll)` / `process(wait)` = 查后台进程输出
- `pty=true` = 交互式 CLI（Claude Code、psql、python REPL）

### 2. File Tool —— 比 `cat/grep/sed` 更安全的文件操作

```python
read_file(path="~/project/main.py", offset=1, limit=50)
write_file(path="~/project/new.py", content="...")
patch(path="~/project/main.py", old_string="foo", new_string="bar")
search_files(pattern="TODO", target="content", file_glob="*.py")
```

> **别再用 `sed/awk` 了**——`patch` 模糊匹配、自动语法检查、不破坏缩进。

### 3. Browser Tool —— 真浏览器自动化（Camofox / CDP）

```python
# 导航、点击、提取、截图
browser_navigate(url="https://example.com")
browser_click(selector="button.submit")
browser_extract(selector=".content")
browser_screenshot(path="/tmp/page.png")
```

**避坑**：私有页面要先登录，用 `browser_navigate` 进登录页，再手动或脚本填表单，Cookie 会在会话内保持。

### 4. 原生 MCP Client —— 零配置接入 MCP Server

在 `config.yaml` 里直接声明：

```yaml
mcp:
  servers:
    - name: github
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_PERSONAL_ACCESS_TOKEN: ${GITHUB_TOKEN}
    - name: filesystem
      command: npx
      args: ["-y", "@modelcontextprotocol/server-filesystem", "/root/workspace"]
```

重启 Hermes，`mcp.github.list_repos`、`mcp.filesystem.read_file` 直接变成可用工具，**无需额外进程管理**。

---

## 五、自动化进阶：Cron + Delegation + Subagents

### 1. Cron Job：定时跑 Agent 任务

```bash
# 创建一个每天早 8 点跑的“每日技术早报” Agent
hermes cron create \
  --name "daily-tech-brief" \
  --schedule "0 8 * * *" \
  --prompt "抓取 GitHub Trending + Hacker News 前 10，生成中文摘要发到 Telegram" \
  --skills "web,github,telegram"
```

**两种模式**：
- `no_agent=false`（默认）：完整 Agent 循环，能推理、调工具、写文件
- `no_agent=true`：纯脚本模式，stdout 直接推送，适合监控告警、心跳检测

### 2. Delegation / Subagents：大任务拆解并行跑

```python
# 在主 Agent 里调用
delegate_task(tasks=[
    {"goal": "调研竞品 A 定价", "context": "...", "toolsets": ["web"]},
    {"goal": "调研竞品 B 定价", "context": "...", "toolsets": ["web"]},
    {"goal": "汇总对比表", "context": "前两个结果见上", "toolsets": ["file"]},
])
```

- 每个 task 独立会话、独立终端、独立上下文
- 并发上限默认 3（`delegation.max_concurrent_children`）
- 结果以消息形式回传主会话，**不污染主上下文**

> **适用场景**：多源调研、多文件重构、多环境测试、并行部署。

---

## 六、多 Profile 隔离：一个机器跑多个项目互不干扰

```bash
# 创建新 profile
hermes profile create work
hermes profile create personal

# 切换
hermes profile use work
# 此时 ~/.hermes/profiles/work/ 下有独立的 skills、plugins、cron、memory
```

**典型用法**：
- `work`：公司项目，加内网 MCP、私有 GitHub、Jira 技能
- `personal`：博客、副业、公开 API
- `experiment`：乱跑实验，炸了删掉不心疼

---

## 七、避坑速查表（新手最常踩的 10 个坑）

| 现象 | 原因 | 解决 |
|------|------|------|
| 模型返回 401/429 | Key 没进环境变量、或配额用尽 | `export XXX_API_KEY=*** 重启 shell |
| Fallback 不生效 | `fallback_providers` 缩进错、或缺 `api_mode` | 对照文档 YAML 缩进，补全字段 |
| Terminal 卡住 | 长任务没加 `background=true` | 长任务必加后台+通知 |
| Browser 登录态丢 | 会话隔离、或 Camofox 未持久化 | 同一 `browser` 会话内完成登录+操作 |
| Skill 不自动加载 | `triggers` 关键词没写、或目录名不对 | 目录名 = skill name，`SKILL.md` 里写 triggers |
| Cron 不执行 | 时区不对、或 `no_agent` 模式脚本无输出 | 检查 `cronjob` 文档，加时区、保证有输出 |
| Subagent 返回乱码 | 子任务没传 `context` 里的关键信息 | `context` 里塞完整：路径、报错、约束 |
| Memory 爆满 | 存了太多临时事实 | 只存偏好、环境、纠偏；进度用 `session_search` |
| MCP Server 连不上 | `command`/`args` 路径错、或缺 `npx -y` | 用绝对路径，参数数组逐项写 |
| 配置改了不生效 | 没重启 Hermes / TUI 缓存 | `hermes restart` 或重进 TUI |

---

## 八、从“会用”到“会改”：把工作流沉淀为 Skill

当你重复做第三遍同样的操作时，**立刻写成 Skill**。模板：

```markdown
# ~/.hermes/skills/我的工作流/SKILL.md
---
name: my-workflow
description: "每次发版前：跑测试 → 构建 → 部署 CF Pages → 校验域名"
triggers: ["发版", "部署", "deploy"]
---

## 步骤
1. `terminal: cd /root/project && npm test`
2. `terminal: hugo --gc --minify` (background + notify)
3. `terminal: wrangler pages deploy public/ --project-name=xxx`
4. `web: curl https://xxx.pages.dev/health` 校验

## 避坑
- Hugo 必须清理 `public/` `resources/`
- CF Pages 首次部署需在 Dashboard 绑自定义域
- 部署后 30s 内可能 522，再试一次
```

下次对话里说“帮我发版”，Hermes 就会自动加载这个 Skill、按步骤跑。

---

## 九、资源速查

| 资源 | 地址 |
|------|------|
| 官方文档 | https://hermes-agent.nousresearch.com/docs |
| GitHub | https://github.com/NousResearch/hermes-agent |
| 技能市场（社区） | https://github.com/hermes-skills/awesome |
| 架构图源码 | 本文顶部 `/images/posts/hermes-architecture.html` |
| 配置 Schema | `~/.hermes/config.yaml` 顶部注释 |

---

## 十、结语

Hermes Agent 的核心哲学只有八个字：**本地优先，配置驱动**。

- 没有 SaaS 绑定，数据不出机房
- 没有魔法黑盒，`config.yaml` 全白盒
- 没有 vendor lock-in，技能随时删、改、换

新手建议路径：**装好 → 改配置 → 装 5 个官方技能 → 跑通 Terminal/Browser/MCP → 写第一个 Cron → 把重复工作流做成私有 Skill**。大概两三天，你就能把“手动操作”变成“自动化流水线”，真正把 Agent 当成**可编程的队友**而不是聊天机器人。

下一步想试什么？直接在 Telegram 里喊一声，我陪你跑通。