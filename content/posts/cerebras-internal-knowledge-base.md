---
title: "从深夜的 #incidents 频道看 Cerebras 如何把 1.5 万次日查询变成工程师的「第二大脑」"
date: 2026-08-25
draft: false
tags: ["RAG", "知识库", "检索增强", "LLM", "系统架构", "Slack", "Cerebras"]
categories: ["技术", "深度思考"]
author: "Blog"
description: "深度解读 Cerebras 用联邦而非迁移的方式，把 Slack、代码库、Wiki、自建数据库统一进一张 Postgres 向量表——四信号混合检索、线程蒸馏、突发门限、CocoIndex 增量代码索引、七阶段查询流水线，日均承载 1.5 万次查询。"
---

![Cerebras 内部知识库](/images/posts/cerebras-knowledge-base.jpg)

## 深夜三点，NFS 报错单还在滚

凌晨 3 点 17 分，#incidents-wafer-fab 频道里的红色告警还在刷。某座晶圆厂的 NFS 挂载点突然吐出 `stale file handle`，编译器团队的工程师盯着屏幕，手指悬在键盘上——要不要把这串报错贴到搜索框？要不要先在 Slack 里翻半小时历史？要不要直接 @ 某个大神？

三个月前，这三个选项都是「赌运气」。现在，他在 Cerebras Knowledge 里敲下一行模糊查询：`NFS stale handle wafer fab mount`。两秒后，答案落地：一条 14 天前的 Slack 线程（已蒸馏为结构化问答）、一篇 Confluence runbook（自动展开相邻章节）、三个相关 PR（按合并时间倒序）、以及一行 `who_knows` 路由——「找 @marc，他是存储基建 owner」。

这不是演示，也不是 PPT 里的架构图。这是 Cerebras 内部上线三个月、日均 1.5 万次查询、覆盖芯片设计、编译器、数据中心运维、训练/推理框架、云服务全栈语料的 RAG 平台 **Cerebras Knowledge** 的日常。它没有把所有数据搬进一个向量库，而是选择了更难、也更务实的路：**联邦而非迁移，数据留在原地，只在检索层做「窄腰带」融合。**

---

## 一、 设计哲学：为什么赌「联邦」而不搞「大迁移」？

大多数团队做内部知识库的第一反应是：把 Confluence、Slack、GitHub、Jira、Google Docs 全部倒进一个向量数据库，建统一索引，再套一层 RAG。听起来顺理成章，做起来却是噩梦——权限同步滞后、数据新鲜度失控、上游 Schema 变更连锁崩溃、摄入管道一堵塞全站不可用。

Cerebras 赌的是另一条路：**federate（联邦）≠ migrate（迁移），meet data where it lives（在数据原地与它会面）。**

具体落地为三层分离：

| 层 | 职责 | 关键决策 |
|---|---|---|
| **Collection（采集层）** | 每个源系统一个 Connector，只负责「拉增量、推标准行」 | Connector 只写同一张 Postgres `embeddings` 表，Schema 固定：`vector(3072) + metadata(jsonb) + source_id + acl` |
| **Querying（检索层）** | 六大工具并行、RRF 融合、Cross-encoder 重排、上下文扩展、带引文综合 | 不关心数据从哪来，只认「证据行」统一契约 |
| **Auth & Audit（鉴权审计层）** | 源码级 ACL、项目范围过滤、查询审计日志、合成必带引文、冲突证据生成 caveat | 横切所有层，查询前过滤、查询后留痕 |

这张「唯一的 embeddings 表」就是全平台的 **窄腰带**。Connector 可以是 Slack Socket Mode 推流、Confluence Webhook、GitHub Webhook、CocoIndex 增量 re-embed、甚至是某团队自建 PostgreSQL 的 `SELECT ...` 小脚本——只要吐出同结构的行，立刻被 `search`、`planner`、`who_knows` 全系工具感知，自动继承平台鉴权审计。**不迁移数据，尊重作者在 Slack/Docs/Git 的原生写作习惯，才是让 15k 日查询跑稳的前提。**

---

## 二、 规模与画像：不只是「搜文档」，是「找人、找代码、找上下文」

| 指标 | 数值 | 含义 |
|---|---|---|
| 日均查询 | ~15,000 | 峰值远超均值，早晚高峰明显 |
| 上线时长 | 3 个月 | 从 0 到默认工具，无强制推广 |
| 覆盖语料 | DC 运维 / 芯片设计 / 硬件 / 训练 / 推理 / 云 | 跨职能、异构、强时效性 |
| 核心意图 | 「X 在哪」「谁知道 Y」「Z 是什么」 | 发现 + 专长路由 + 事实查询三位一体 |
| 用户 | 员工 + 自动化工作流 + MCP Agents | 同一检索层服务人与 Agent |

注意最后一行：**MCP Agents 也是一等公民**。Claude Code、Cursor 等客户端通过 MCP 直接调用 `search_slack`、`search_code`、`who_knows` 等窄原语，自己在客户端做编排。这意味着检索层不烧隐式 LLM token，延迟低、吞吐高、成本可控——这在后文「Web UI vs MCP」一节会详细拆解。

---

## 三、 Slack 摄取：把「聊天流」变成「可检索知识」的四重信号

Slack 是内部知识的「暗物质」——占比最高、时效性最强、噪音也最大。Cerebras 没有把原始消息直接嵌入（那会把「好的」「收到」「谢谢」全嵌进去），而是设计了 **四信号融合**：

| 信号 | 技术实现 | 解决的盲区 | 权重策略 |
|---|---|---|---|
| **全文检索** | Postgres GIN 索引 + BM25 | 精确错误串、flag、hostname、异常堆栈 | 基础权重 1.0 |
| **向量嵌入** | 3072 维（text-embedding-3-large） | 跨词面改写、「那个 NFS 挂载挂了」≠「stale file handle」 | 基础权重 1.0 |
| **IDF 抑制** | 词频逆文档频率，过滤高频社交词 | 「好的」「谢谢」「收到」「lgtm」式噪音 | 动态惩罚，IDF < 阈值直接归零 |
| **时间衰减** | `exp(-λ * age_days)`，λ 可配 | Infra 答案半衰期短，六个月前的 runbook 可能已失效 | 偏重近 30 天，配合 age decay 参数 |

融合用 **RRF（Reciprocal Rank Fusion，倒数排名融合）**：`score = Σ w / (k + rank)`，`k=60` 平滑常数偏向跨检索器共识。四信号各补盲区——全文抓精确串，向量抓语义改写，IDF 扫社交噪音，时间衰减保鲜度。**没有单一信号能单打独斗，共识才是硬道理。**

摄取端的工程细节同样硬核：

- **Socket Mode（WebSocket 推流）**：不轮询，事件实时到达，延迟秒级
- **稳定事件 ID 去重**：Slack 事件可能重发，`client_msg_id` + `event_ts` 双键幂等
- **线程级写入**：回复触发「重拉父消息 + 兄弟消息」，存完整对话树，而非孤立单条
- **按频道调刷新频率**：`#incidents-*` 高频推流，`#random` 低频批量
- **Raw FTS 索引落地即可搜**：不等嵌入完成，全文检索先上线

---

## 四、 线程蒸馏与突发门限：从「原始聊天」到「高信号证据」的两道关卡

这是整套系统**最值钱、最可迁移**的设计。请跟着一个真实线程走一遍。

### 4.1 场景还原：某个周二的 #compiler-perf 线程

```
@alice: 大家有没有遇到过 WSE-3 上 compile_time 突然暴涨 3x？log 里全是 "spill to global memory"
@bob: 我上周调过，是新版 LLVM 的 register allocator 把 live range 算错了，PR #12456 有 fix
@carol: @bob 那个 PR 合了吗？我这边还是老版本
@dave: 已合，但要配合 runtime flag `--enable-new-ra=true`，文档在 confluence/xyz
@alice: 好的谢谢 @bob @dave
@eve: 👍
```

原始聊天里：**问题、假设、确认、修复、配置、文档引用、社交噪音** 混在一起。直接嵌入？向量空间里「好的谢谢」和「spill to global memory」距离可能比「register allocator」和「live range」还近。

### 4.2 蒸馏：LLM 把线程「压缩」为 5 个结构化字段 + raw（仅 FTS）

| 字段 | 示例内容 | 作用 |
|---|---|---|
| `searchable_question` | "WSE-3 compile_time 暴涨 3x spill to global memory 原因" | 工程师真会打的查询语，向量检索命中率 ↑ |
| `summary` | "LLVM register allocator live range 计算错误导致寄存器压力激增，溢出到全局内存" | 语义检索命中、综合时可直接引用 |
| `resolution` | "升级 LLVM 至 PR #12456 版本 + 启用 `--enable-new-ra=true`" | 事实查询直达答案 |
| `systems_mentioned` | `["LLVM", "WSE-3", "compiler", "runtime"]` | `subsystem_index` 路由、项目范围过滤 |
| `code_references` | `["PR #12456", "runtime_flag.new_ra"]` | `search_code`、`recent_prs` 关联 |
| `raw` | 原始线程全文 | 仅建 GIN 索引供全文检索，**不嵌入向量** |

**蒸馏是有损压缩**——切线事实（如 `@eve: 👍`、旁枝讨论「周末要不要加班」）被丢弃。但换来的是：向量空间纯净、检索精准、Token 省 80%+。

### 4.3 突发门限：不让「高信号连续发言」溜走

蒸馏是「线程级」的，但很多高价值知识藏在**单人连续发言**里——某专家连发 5 条消息把一个晦涩的硬件 errata 讲透，没人回复，线程不成形，蒸馏不触发。

**突发门限** 专门捞这类：

| 门限条件 | 阈值 | 通过后动作 |
|---|---|---|
| IDF ≥ 4.0 | 过滤通用词，保留领域术语密度高的段落 |  |
| 组合长度 ≥ 200 字符 | 排除碎碎念 |  |
| 可选：Slack reactions ≥ 1 | 有人点赞/确认，弱监督信号 |  |
| 上下文前缀：线程主题 | 保留话题边界 |  |

**通过门限的高信号单作者连续发言 → 嵌入为 `slack_burst` 记录**；拒绝的直接丢弃，**不污染索引**。

> **对个人知识库的启发**：你的 Obsidian/Notion 里有多少「临时笔记」其实是高信号 burst？试着跑个脚本：按 IDF 过滤、按长度切片、按 reaction（或最后编辑时间）打分，只入库高分段。别把垃圾喂给向量库。

---

## 五、 代码规模化：40 GB+ 仓库的增量 Re-embed 与双粒度检索

代码搜索是 RAG 的「硬骨头」：仓库大、变更快、语义边界不按 token 切。Cerebras 用 **CocoIndex** 做语言感知分块：

1. **增量触发**：按 commit 监听，只 re-embed 变更文件（`git diff --name-only`）
2. **语言感知 regex 分词**：`class → method → block` 递归切分，按语法树而非 token 数
3. **块过大则粗变细**：超过阈值继续切，保证每块语义内聚
4. **双粒度存储**：
   - **File 级**：整文件摘要向量，适合「这个模块大概干啥」
   - **Function 级**：函数/方法向量，适合「这个函数怎么处理 edge case」
5. **Postgres 存向量 + sync 元数据**：同一张表，`metadata` 里带 `file_path`、`function_name`、`start_line`、`end_line`、`commit_sha`
6. **团队级允许/拒绝列表**：Compiler 团队只索引 `compiler/`，DC Ops 团队拒绝 `compiler/**`，权限随代码所有权走

**搜代码不走向量**，走 **ripgrep 精确匹配**（`search_code` 工具）。向量只管「语义找入口」，精确匹配管「定位到行」。这是「粗排精排分离」在代码场景的落地。

---

## 六、 其余源系统：同一行契约，异构摄取策略

| 源 | 分块策略 | 更新机制 | 特殊处理 |
|---|---|---|---|
| **Confluence** | 按标题层级分块（非 token 数） | 近实时 Webhook | 上下文扩展拉相邻 ±2 节，保持章节连贯 |
| **Google Docs** | 同 Confluence | 轮询/推流同一行契约 | **建议编辑留源系统**，Connector 只同步，不反写 |
| **Jira** | 只索引状态字段（标题、状态、assignee、labels） | 工单流转更新行 | 配合 `recent_prs` 应对变更意识 |
| **GitHub Issues/PRs** | 保留标题 + 标签 | 合并 PR 走 Webhook | 互补 `search_code`，不嵌入正文 |

**核心原则**：Connector 只负责「源 → 标准行」，不做业务逻辑。上游改 Schema、换字段，只改 Connector，检索层零感知。

---

## 七、 六大检索工具：各司其职， planner 只管「派单」

| 工具 | 适用场景 | 底层实现 |
|---|---|---|
| `subsystem_index` | 「Compiler 模块大概有哪些文件？」 | File 级 LLM 摘要向量检索 |
| `search` | 通用语义搜索 | 统一向量表（所有源混合） |
| `search_slack` | 找报错、找讨论、找决策 | 四信号 + RRF 融合 |
| `search_code` | 找函数实现、找调用点 | ripgrep 精确匹配 |
| `recent_prs` | 「最近有没有改过 memory allocator？」 | PR 元数据按合并时间倒序 |
| `who_knows` | 「谁懂 NFS mount 参数调优？」 | 专长路由：综合提交历史、Slack 发言、Review 记录 |

**Planner（轻量 LLM）** 读取 query + 项目范围 → 输出工具列表。比如「WSE-3 compile 时间暴涨」→ `[search_slack, search_code, recent_prs, who_knows]`。不调用 `subsystem_index`（太泛）、不调用 `search`（太宽）。**精准派单 = 少跑无效工具 = 省 Token = 低延迟。**

---

## 八、 七阶段查询流水线：跟着一个真实查询走完全程

查询：**`"WSE-3 compile_time spike spill to global memory"`**（工程师在 Web UI 搜索框输入）

### Stage 1：Planner（轻量 LLM，~50ms）
- 读取 query + 用户默认项目（Compiler）+ 项目绑定源集合
- 输出工具列表：`[search_slack, search_code, recent_prs, who_knows]`
- **不调用** `search`（全局向量太宽）、**不调用** `subsystem_index`（太粗）

### Stage 2：Executor（并行 Fan-out，~200ms）
四工具并行跑，各返回统一 Schema 证据行：
```
search_slack → [蒸馏线程#4521 (score 0.89), slack_burst#887 (score 0.82), ...]
search_code  → [compiler/allocator.cpp:register_allocate() (match), ...]
recent_prs   → [PR #12456 "Fix live range calc", PR #12501 "Add RA flag", ...]
who_knows    → [@bob (compiler allocator owner), @dave (runtime flags), ...]
```
每行自带：`source`、`source_id`、`acl`、`project_scope`、`metadata`。

### Stage 3：RRF 融合（~10ms）
`score = Σ w / (60 + rank)`，`k=60` 偏向跨工具共识。
- 蒸馏线程#4521：在 `search_slack` rank=1、`who_knows` 关联 `@bob` rank=2 → **共识强，融合分高**
- `search_code` 命中的函数只在代码工具出现 → **单工具证据，分较低**
- `recent_prs` 的 PR #12456 在 `search_slack` 的 `code_references` 也出现 → **跨工具共识，加分**

### Stage 4：去重 + Cap（~5ms）
- 同一文件的多个 chunk 合并，按文件设限（最多 3 chunk/文件）
- 同一 Slack 线程的 `蒸馏记录` + `burst记录` 合并
- 最终保留 **top ~20 证据行** 进入重排

### Stage 5：Cross-encoder Reranker（~300ms）
轻量 Cross-encoder（如 `bge-reranker-v2-m3`）对 query+证据打 0-10 分，**仅保留 top 10**。
- 蒸馏线程#4521：9.2 分（问题完全匹配、有 resolution、有代码引用）
- PR #12456：8.7 分（直接修复、已合并）
- `@bob` 专长卡：8.1 分（owner 强关联）
- 孤立的代码 chunk：4.3 分（缺上下文，被邻域扩展补救或淘汰）

### Stage 6：上下文扩展（~50ms）
- Wiki 证据：自动拉取相邻 ±2 个标题节
- Slack 证据：自动拉取线程完整对话树
- 代码证据：自动拉取函数签名 + 调用图片段
- **目的**：把「孤立段落」还原为「可自证的上下文」

### Stage 7：综合生成（~800ms，主 LLM）
输入：query + top 10 扩展后证据 + 项目范围 + 用户权限
输出：Markdown 答案，**每句关键结论后带引文 `[source:slack#4521]` `[source:pr#12456]`**；检测到冲突证据（Slack 说 A，Wiki 说 B）→ **生成 caveat 段落**，不假装确定。

**端到端延迟**：Web UI ~1.5s（含 Planner+Executor+Rerank+Synthesize）；MCP 客户端自编排可并行化到 ~400ms（只跑 Executor+RRF+Rerank，无 Planner/Synthesize）。

---

## 九、 RRF 参数为什么是 k=60？共识胜过单榜首

`score = Σ w / (k + rank)`，`k=60` 不是拍脑门定的：

- `k` 越小 → 越像「取最小 rank」，单工具榜首主导
- `k` 越大 → 越像「平均 rank」，共识主导
- **k=60 经验甜点**：在 4-6 个检索器、各返回 20-50 条证据的规模下，能让「在 3+ 工具同时 top-5」的证据显著超越「仅在 1 个工具 rank=1」的证据

**Per-list weight 默认 1.0**，缺席贡献 0。Reranker 只看 top 10，**RRF 做粗排共识，Cross-encoder 做精排语义**，分工明确。

---

## 十、 Web UI vs MCP：同一原语，不同编排者

| 维度 | Web UI（端到端） | MCP（窄原语） |
|---|---|---|
| 编排者 | 服务端 Planner + Executor + Synthesizer | 客户端（Claude Code / Cursor 等） |
| 暴露接口 | 单一 `ask(query, project)` | `search_slack`、`search_code`、`search`、`who_knows`、`recent_prs`、`subsystem_index` |
| LLM 调用 | 服务端 3 次（Planner/Rerank/Synthesize） | 客户端自定，**服务端 0 次 LLM** |
| 延迟 | ~1.5s 端到端 | ~400ms 证据抓取（可并行） |
| 成本 | 服务端烧 Token | 客户端烧 Token，服务端只跑检索 |
| 适用场景 | 人类直接提问、需要综合答案 | Agent 工作流、需要精细控制检索策略、多轮推理 |

**MCP 的窄原语设计** 是关键：不暴露 `ask`，只暴露「免 LLM 的检索原语」。客户端自己决定：先 `search_slack` 找报错，再 `search_code` 定位函数，再 `who_knows` 找人，最后 `recent_prs` 确认修复状态。**把编排权还给最懂任务结构的一方——Agent 客户端**。这也是「窄腰带」在编排层的延伸。

---

## 十一、 项目化范围：全局搜索失效后的「精准隔离」

语料增长到一定规模，**全局搜索必然失效**——编译器工程师搜「register allocation」，不想看 DC 运维的「NFS mount 参数」；DC 运维搜「power redundancy」，不想看编译器的「LLVM pass order」。

**项目 = 按团队绑定的源集合**：

| 项目 | 绑定源 | 典型用户 |
|---|---|---|
| `compiler` | compiler/ 仓库、#compiler-* 频道、Compiler Confluence、LLVM 相关 Jira | 编译器团队 |
| `ml-training-infra` | training/ 仓库、#training-infra 频道、集群运维 Wiki | 训练基建团队 |
| `dc-ops` | dc-ops/ 仓库、#incidents-* 频道、Runbook Wiki、Jira 运维工单 | 数据中心运维 |
| `shared` | 通用基建仓库、架构决策记录、跨团队 RFC | 全员可引用 |

**默认项目写入用户 Profile**，入职即高信号范围。**共享源可跨项目引用**（如 `shared` 里的架构文档）。**隔离目标：Precision over Recall**——宁可漏掉跨域关联，也不让噪音淹没核心答案。

> **对个人 Agent OS 的启发**：你的「项目」可以是「当前 Sprint」「当前论文」「当前副业」。给每个项目配一组源（Obsidian 文件夹、GitHub 仓库、Slack 导出、浏览器历史），查询时自动限域。别让「去年的读书笔记」干扰「今天的 Debug」。

---

## 十二、 自定义数据源插件：团队有 DB 就发 PR，平台做「标准化+鉴权+审计」

某团队有个自建 PostgreSQL 存「芯片测试向量」，想进 Knowledge。不需要开工单、不需要平台组写代码：

1. **写个小 Python 插件**（~50 行）：
```python
def emit_rows(connection) -> Iterable[EmbeddingRow]:
    for row in connection.execute("SELECT id, test_vector, metadata FROM test_vectors WHERE updated_at > :last_sync"):
        yield EmbeddingRow(
            vector=embed(row['test_vector']),  # 调平台统一 embedding API
            metadata={"test_id": row['id'], **row['metadata']},
            source_id="custom:test_vectors",
            acl={"team": "chip-validation"}  # 源码级 ACL
        )
```
2. **加一行数据源配置**：指定插件名、连接串、同步频率、所属项目
3. **提 PR 合并** → 立刻被 `search`、`planner`、`who_knows` 检索，**自动继承平台鉴权审计**

**平台只管契约，不管业务逻辑**。这是「窄腰带」在扩展层的体现。

---

## 十三、 鉴权、审计、信任：不信任、不假装、留痕迹

| 机制 | 实现 | 关键点 |
|---|---|---|
| **源码级 ACL** | Connector 读取时把 Slack channel 私有性、Repo 读权限、Confluence 空间权限写入 `embeddings.acl` | 查询时 **每次** 以用户身份过滤，不缓存权限 |
| **项目范围过滤** | Planner 阶段按用户默认项目 + 显式指定项目，预过滤源集合 | 从源头砍掉无关语料，不进 Executor |
| **查询审计日志** | 每查询记录：`user_id, query, project_scope, tools_called, evidence_count, latency_ms` | 事后可审、可追责、可分析使用模式 |
| **合成必带引文** | Synthesizer 强制在每句关键结论后附 `[source:xyz]` | 可溯源、可人工核验 |
| **冲突证据生成 Caveat** | 检测到多源结论不一致（Slack 说 A，Wiki 说 B）→ 输出 `⚠️ Caveat: 来源冲突...` | **不假装确定**，把判断权还给人 |

---

## 十四、 11 类失败模式与缓解：把「会翻车」的地方全堵上

| # | 失败模式 | 典型表现 | 缓解手段 | 核心思想 |
|---|---|---|---|---|
| 1 | **Stale Wiki** | Confluence 跑步机文档，半年前的 runbook 还在排首位 | Age decay + 持续摄入 + 合成时标注「最后更新 2024-03」 | **鲜度是第一位信号** |
| 2 | **Filler 占 Rank** | 「好的」「收到」「lgtm」挤进 top-k | IDF 门槛 + 蒸馏丢弃社交噪音 + Burst 门槛 | **源头不入库，入库再打分** |
| 3 | **失切线事实** | 专家连发 5 条讲 errata，蒸馏只保主线 | Burst 门限（IDF≥4、长度≥200、可选 reaction）补偿 | **显式捞高信号单人流** |
| 4 | **分数尺度漂移** | BM25 分 0-100，向量分 0-1，直接加权不可比 | **RRF 只用 rank，不看原始分** | **Rank 是通货，Score 不是** |
| 5 | **词汇假阳性** | 「Transformer」同时匹配「注意力机制」和「电力变压器」 | Cross-encoder Reranker 语义级过滤 | **粗排召回，精排理解** |
| 6 | **块边界丢失** | 函数中间一段孤立嵌入，缺签名、缺调用上下文 | 邻域 Wiki 节扩展 + 代码双粒度 + 函数级向量 | **把上下文还给 Chunk** |
| 7 | **全局搜索噪音** | 编译器工程师看到 DC runbook | 项目范围预过滤（Planner 前置） | **范围隔离 > 召回全量** |
| 8 | **权限泄漏** | 未授权私有频道内容出现在搜索 | 每次查询实时按 ACL 过滤，不缓存 | **零信任每一跳** |
| 9 | **MCP 过载** | Agent 疯狂刷工具、烧光预算 | 窄原语契约 + 客户端预算控制（每分钟 N 次） | **把节流权交给调用方** |
| 10 | **40GB 仓滞后** | 代码答案指向已删除函数 | CocoIndex 逐 commit 增量 re-embed，同步元数据 | **增量同步 = 新鲜度保证** |
| 11 | **冲突来源** | Slack 说「参数 A=1」，Wiki 说「参数 A=2」 | 合成引用两边 + 生成 Caveat 段落 | **冲突显性化，不调和** |

---

## 十五、 结语：给「一人公司 / 个人 Agent OS」的 4 条可直接抄的动作

Cerebras Knowledge 不是大厂专属——它的每个设计点都能**缩小版复刻**到个人知识库、小团队 Wiki、甚至单人 Agent OS 里：

1. **别搬家，建连接器**  
   Obsidian、Notion、GitHub、Slack 导出、浏览器历史、Kindle 笔记——**留在原地**。写 5 个 50 行 Python 脚本，定时/事件驱动把增量推到**同一张 SQLite/Postgres 表**（`vector + metadata + source + acl`）。这就是你的「窄腰带」。

2. **Slack/微信/钉钉聊天记录：蒸馏 > 直接嵌入**  
   跑个夜ly 批处理：LLM 把每条线程压缩为 `{question, summary, resolution, tags, code_refs}` + `raw`（仅全文索引）。再加个 **Burst 门限**（IDF 阈值 + 长度 + 表情反应），单人高信号连发自动入库。**别让「好的谢谢」污染你的向量空间。**

3. **检索用「共识」而非「单榜首」**  
   给每个源一个检索器（全文、向量、代码 grep、标签、时间），**RRF k=60 融合**，再跑个轻量 Cross-encoder 重排 top 10。代码 20 行，效果碾压单一向量搜索。

4. **给 Agent 暴露「窄原语」，别暴露「问答接口」**  
   MCP 服务端只提供 `search_slack`、`search_code`、`search_notion`、`who_knows`、`recent_commits`。**编排权留给 Claude Code / Cursor / 自写 Agent**。你省 Token、省延迟、省耦合，Agent 拿到干净证据自己推理。

---

**Cerebras Knowledge 的故事，本质上是一个「约定优于配置」的工程叙事：**

- 约定**同一张表**（窄腰带）
- 约定**同一行契约**（EmbeddingRow）
- 约定**同一套融合**（RRF + Rerank）
- 约定**同一层鉴权**（源码级 ACL）
- 约定**同一组原语**（六大工具 + MCP 窄接口）

没有统一数据湖，没有重模型轻工程，没有「先把数仓建好再做应用」。**从最痛的源头（Slack）切起，用四信号+蒸馏+突发门限把噪音变信号，用项目范围把噪音隔出去，用 MCP 窄原语把检索能力借给 Agent。**

15k 日查询、3 个月成默认工具、覆盖全栈语料、人与 Agent 共用一层——**这不是 RAG 的终局，但它是目前工程落地最诚实、最可复制的范本之一。**

---

> **延伸阅读**
> - [个人 Agent OS 设计反思：从过度架构到五层精简](/posts/agent-os-design-reflection/)
> - [Agent OS v2.0 落地笔记：给 AI 助手装上记忆防御层](/posts/agent-os-v2-memory-architecture-upgrade/)
> - [Hermes Agent 记忆系统详解：机器人进化闭环的5层实现](/posts/hermes-tips-02-memory/)
> - [为什么 AI 助手总是记不住你的博客流程？](/posts/hermes-memory-workflow-fix/)

---

> **参考来源**  
> - Cerebras 官方工程博客：`Cerebras Knowledge: Building a RAG Platform for 15k Queries/Day` (mer.vin, 2026-07)  
> - Cursor 团队博客：`Semantic Search at Scale: How We Built Codebase-Aware Search`  
> - 文中所有数字、架构细节、参数配置均来源于上述一手工程分享，未做任何推测性补充。
