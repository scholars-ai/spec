# SPEC-008 · M1 实施计划：选题闭环

- 状态：Superseded（M1 历史实施与验收记录；当前运行基线为 SPEC-010）
- 日期：2026-08-08
- 依赖：SPEC-003（采集）、SPEC-004（评分）、SPEC-007 §M1
- 前置决策：ADR-004（DB 自托管）、ADR-005 v2（embedding 走 SiliconFlow API，Ollama 降为备用）

M0 已交付六仓库骨架。本 spec 把 SPEC-007 的 M1 里程碑落成可执行清单。

> **历史文档声明**：本文件记录 M1 选题闭环在 2026-08 的实施过程、当时的调度配置和验收证据。它不定义当前生产工作流。当前自动/手动运行、12 小时调度、动态漏斗、节点判定、快照和 replay 均以 [SPEC-010](SPEC-010-workflow-run-and-replay.md) 为准；下文带日期的数字和配置不得作为新实现要求。

## 1. 目标（历史 M1）

打通当时五步流程的前两步——**采集 → 聚合选题 → 评分 → 人工确认**，全链路运行在 VPS 上。该阶段目标已完成并被 SPEC-010 的六阶段 `WorkflowRun` 包含。
产出物的判定标准：**每天打开看板，就有带评分和理由的可写选题**。

## 2. 已拍板的技术决策

| 项 | 决定 | 依据 |
|---|---|---|
| 数据库 | VPS 自托管 Postgres 17（业务库 + Langfuse 库同实例），每日 pg_dump 推 COS | ADR-004 |
| Embedding | **SiliconFlow API `BAAI/bge-m3`（原生 1024 维）**，输入截断 600 字符；本机 Ollama 降为 `EMBED_BACKEND=ollama` 备用 | ADR-005 v2（本机方案实测不可行，见该 ADR §决策变更） |
| 可观测 | OTel Collector → Tempo/Prometheus → Grafana；LLM 细节留在 Langfuse | ADR-006；Langfuse 数据与保留见 ADR-004 |

## 3. 数据流（M1 历史运行态）

```
core scheduler tick（每分钟，仅为检查粒度）
  读 DB 调度配置 ──▶ 到期的 source 投递 source_fetch（事务性入队）
                            │
agents SourcingHandler ◀────┘
  feedparser 拉取 → trafilatura 清洗 → content_hash 精确去重
  → embedding（SiliconFlow bge-m3，1024d）→ 语义去重（近 14 天 cos > 0.92）
  → raw_items(status=new)
                            │
core scheduler ──▶ 到期的 topic_scout（M1 历史默认每日 2 次，可配）
                            │
agents TopicScout（自研 loop 首次实战）
  聚类素材 → 每簇提 1–3 个选题角度 → 与既有 topics 向量查重
  → topics(status=candidate)
                            │
core harvester ──▶ 新 candidate **事件驱动**立即投递 topic_evaluate（不等固定时刻）
                            │
agents TopicJudge（structured output 首次实战）
  读当时的 `topic@v1` rubric + DB weight_sets → 维度分 + 理由（历史实现）
  → topic_evaluations + agent_runs（挂 Langfuse trace）
                            │
core harvester ──▶ 状态机 candidate→scored；≥75 推荐 / 60–75 备选 / <60 自动 rejected
                            │
scholar-client 选题看板 ──▶ 人工 approve / reject（scored → approved / rejected）
```

每次 API/调度触发创建 `correlationId`，每条 pgmq 消息拥有独立 `jobId`，下游通过 `parentJobId` 和 W3C `traceparent` 继续同一 Trace。Core/Agents 的安全 Span 进入 Tempo；Scout/Judge 的 prompt、完整输出、token、成本和评分只进入 Langfuse，Tempo 通过 `langfuse.trace_id` 关联。

### 3.1 历史调度配置（已被 SPEC-010 取代）

初版把「每小时采集、每日 2 次 scout」写成固定 cron 表达式；以下是 M1 期间的 DB 配置模型和验收记录，现已被统一 `WorkflowRun` 调度取代：

| job | 调度方式 | 默认值 | 可配粒度 |
|---|---|---|---|
| `source_fetch` | interval | 每 60 分钟（历史值） | **全局默认 + 每个 source 单独覆盖**；可暂停；可手动立即触发 |
| `topic_scout` | daily times | 08:00 / 20:00（历史值） | 执行时刻、时区、启停、`min_new_items`（新素材不足则跳过） |
| `topic_evaluate` | **event-driven** | candidate 产生即投递 | 启停、并发上限 |

三条设计纪律：

1. **唯一写死的是 scheduler 的 tick 粒度（1 分钟）**——它只是内部检查频率，不是业务策略。所有业务频率来自 DB。
2. **`topic_evaluate` 不设固定时刻**。评分是对「新 candidate 出现」的响应；固定每日跑两次会让候选白等半天。原初版把它和 scout 混为一谈是设计错误。
3. **按 source 差异化采集**：arXiv 每 6 小时、活跃博客每 30 分钟、不稳定源直接暂停——统一每小时既浪费请求也可能触发上游限流（RSSHub / X 尤其敏感）。

调度正确性要求：

- 同一 source 同一时间窗口不得重复投递（`next_run_at` + 唯一约束）
- 多 core 实例并存时不重复调度（advisory lock）
- 配置变更 ≤ 1 个 tick 生效
- source 暂停后不再产生新 job，已领取的 job 正常跑完
- 单条配置非法不得导致整个 scheduler 崩溃（跳过并告警）
- 每次调度留痕：schedule key、触发时间、job id

### 3.2 环境变量与 DB 配置的边界

`DEFAULT_*` 环境变量**只用于首次 seed**（DB 无配置时的初始值）。一旦用户在 client 改过，运行时真相只在 DB——不再回读环境变量，否则重启会静默覆盖用户设置。

当前替代规则：scheduler 默认每 12 小时创建一次 `WorkflowRun`，自动和手动入口统一执行 `source_fetch → topic_scout → topic_evaluate → article_write → article_evaluate → human_review`；节点推进和 fan-in/fan-out 由 SPEC-010 定义。

## 4. 交付清单

### scholar-shared（契约先行）
- [x] `openapi/core.yaml` 扩展：sources CRUD、手动投喂 URL、topic approve/reject、**调度设置 API**
- [x] 新增 TopicJudge 结构化输出 schema（维度分 + rationale，供 `complete_structured` 校验）
- [x] `SchedulerSettings` / `SourceFetchConfig` schema（§3.1 的配置结构）
- [x] codegen 三端并提交生成物

### scholar-core（Go）
- [x] sources CRUD API + sqlc 查询
- [x] 手动投喂 API：POST URL → 入 raw_items 流程
- [x] **动态 scheduler**：1 分钟 tick 读 DB 配置；source 级 interval 覆盖；scout 按配置时刻；调度留痕 + 防重投
- [x] **调度设置 API**（GET/PATCH）+ 首次 seed 默认值
- [x] harvester：轮询结果表推进状态机（唯一写入口原则不破）；**candidate 出现即投递 topic_evaluate**
- [x] topic approve/reject API（非法流转返回 409）
- [x] CI：Go 队列常量与 `schemas/queues.json` 一致性校验（M0 遗留口子）

### scholar-agents（Python）
- [x] `embed()` 封装：双后端（SiliconFlow API / 本机 Ollama），维度契约校验 + L2 归一化（13 项单测）
- [x] SourcingHandler：拉取/清洗/双重去重；**条目级失败隔离**（一条脏数据不毁整批）+ 源级失败隔离
- [x] TopicScout：素材聚类 + 角度生成 + 查重；insights 检索留接口（M3 才有数据）
- [x] TopicJudge：rubric YAML + 生效权重 → 结构化评分；一票否决逻辑（本 rubric 暂无 veto 维度，但代码路径就位）
- [x] Langfuse 接入：每 job 一条 trace，评分作为 score 挂载，trace_id 回写 agent_runs
- [x] 每个 LLM 调用通过 Langfuse trace 记录 prompt、输出、输入/输出 token 和成本；不在应用层重复实现每日 token 预算，供应商 API key 负责额度限制

### scholar-client（Next.js）
- [x] 选题看板：候选列表（分数排序）、维度分可视化、评分理由展开、approve/reject
- [x] 信源管理页：增删改、启停、**单独采集频率覆盖**、手动立即采集、最近抓取状态与连续失败告警
- [x] **调度设置页**：全局采集间隔、scout 执行时刻/时区/min_new_items、evaluate 启停与并发。用表单生成调度配置，不让用户手写表达式
- [x] 手动投喂入口（高频动作，一键贴 URL）

### scholar-infra + 部署
- [x] `compose.prod.yaml` 增加 postgres 服务（自建镜像）+ named volume；langfuse 指向同实例独立 database
- [x] 备份：每日 `pg_dump` → 加密 → 本地保留 7 份 → COS 离机副本；上传后回读校验，COS 下载恢复演练已完成
- [x] 磁盘监控告警（> 85%），每 6h 定时检查
- [x] GHCR 镜像构建 CI（core/agents）+ deploy.sh 已具备版本部署路径；core/agents/Langfuse/Postgres 已在 VPS 运行
- [ ] nginx 反代（复用现有实例）：client 访问 core API

## 5. 实施顺序

每步结束都可独立验证，避免"全写完再调试"：

1. **infra 先行**：VPS 起自托管 Postgres + goose 迁移 + 备份/监控就位（数据地基）
2. **契约扩展**：shared 改 OpenAPI + 评分 schema → codegen
3. **core CRUD**：sources 管理 + 手动投喂（先把 API 链路走通）
4. **agents embed + SourcingHandler**：先接 2–3 个源本地验证，再扩到 ≥8
5. **core scheduler**：采集自动化（历史 M1 步骤）
6. **agents TopicScout → TopicJudge**：**同时接 Langfuse**（调 prompt 全靠看 trace，这步工作量最大）
7. **core harvester**：闭环在后端成立
8. **client 看板 + 信源管理**：有真数据后再做界面
9. **VPS 部署 + 试运行一周** → 逐条打勾 §6

## 6. 验收标准（历史 M1；当前验收以 SPEC-010 §9 为准）

**功能**
- [x] ≥8 个信源接入并完成真实采集；单源失败隔离、连续失败状态可观测
- [x] 历史目标：每天自动产出 ≥10 条选题候选，语义重复率 < 10%（该数量不构成当前固定配额）
- [x] 手动投喂 URL → 出现在候选池 < 2 分钟（2026-08-14 真实验收：新 URL 入库后定向 Scout 生成 3 条候选约 31 秒）
- [x] 每条候选具备总分、6 维度分、人可读理由
- [x] approve/reject 生效，非法状态流转被拒（409）

**质量（M1 最重要的软验收）**
- [x] 人工抽查 20 条评分，**理由认可率 ≥ 80%**；2026-08-14 v2 人工结论为认可 `10/20`、部分认可 `8/20`、不认可 `2/20`，按认可和部分认可均计入的口径，有效认可 `18/20 = 90%`
- [x] 每次评分记录 rubric_version 与权重版本，可回放

**工程**
- [x] 全链路 Langfuse trace 可查（prompt / 输出 / token / 成本）
- [x] quota/余额/无效 key 等永久错误不重复重试；临时错误的 job 重试次数有限且可观测
- [x] agents 崩溃重启后 job 不丢（pgmq visibility timeout 实测）
- [x] job 有 whole-job deadline，visibility lease 覆盖 deadline + grace；成功回执防止 delete 前崩溃导致重复执行业务
- [x] 永久失败/重试耗尽写入 `job_failures` 并 archive；状态流转写入 `state_transition_events`
- [x] source_fetch/topic_scout/topic_evaluate 使用独立 Worker 进程，可分别扩容和隔离慢任务
- [x] Core/Agents 通过 OTLP 上报 Trace/Metrics；观测组件不可用不影响业务
- [x] 跨仓库 E2E：两条 M1 路径、Tempo correlation、Langfuse link 和 Collector 故障隔离全部运行通过
- [x] 数据库备份产出 + **恢复演练成功**（本地加密副本与 COS 下载副本均验证）
- [x] CI：队列名一致性校验生效；GHCR 镜像构建通过，VPS 生产服务已部署

**调度可配（§3.1）**
- [x] client 改采集间隔后 ≤ 1 个 tick 生效，且**重启 core 不被环境变量覆盖**
- [x] 单个 source 设为 6 小时 / 暂停，各自按自身配置执行，不受全局默认影响
- [x] scout 改执行时刻后按新时刻触发；`min_new_items` 不满足时跳过并留痕
- [x] candidate 产生到 `topic_evaluate` 入队 < 1 分钟（事件驱动，非固定时刻）
- [x] 非法配置（时间格式/间隔越界/重复时刻）被 API 拒绝，scheduler 不崩

## 7. M1 阶段明确不做（历史范围，不代表当前范围）

- 不写文章（Writer 属 M2）
- 不做数据回流与记忆（M3）；TopicScout 的 insights 检索只留接口
- 不做评分权重校准（M4，尚无数据）
- X/Twitter 源不稳则暂时放弃该源，不为其自研爬虫

## 9. 首批信源清单（2026-08-10 逐源实测）

起点是用户指定的聚合源 `aihot.virxact.com`。分析其 50 条的 `author` 字段发现它标注了全部 33 个上游源及类型，据此**绕过聚合层直连上游拿原文**，聚合源退为信号兜底。

### A 层 · material（直连，拿原文）

| 源 | 接入 | 实测正文量 | 类别 | full_text |
|---|---|---|---|---|
| MarkTechPost | 原生 RSS | **28 KB 全文** | tutorial | rss_description |
| Interconnects（Nathan Lambert） | 原生 RSS | **23 KB 全文** | tutorial | rss_description |
| GitHub Blog | 原生 RSS | **21 KB 全文** | news | rss_description |
| NVIDIA Blog | 原生 RSS | **12 KB 全文** | news | rss_description |
| IT之家 | RSSHub `/ithome/it` | 2.3 KB | news | rss_description |
| arXiv cs.CL / cs.AI | 原生 RSS | 1.6 KB（完整摘要） | research | rss_description |
| The Verge AI | 原生 RSS | 1.6 KB | news | rss_description |
| Ars Technica AI | 原生 RSS | 1.5 KB | news | rss_description |
| The Decoder | 原生 RSS | 0.9 KB | news | rss_description |
| Apple ML Research | 原生 RSS | 0.6 KB | research | rss_description |
| Google Developers Blog | 原生 RSS | 0.6 KB | tutorial | rss_description |
| Hacker News frontpage | `hnrss.org/frontpage` | 0.3 KB | news | fetch_page |
| TechCrunch AI | 原生 RSS 0.14 KB | 抓页面得 1 KB | news | fetch_page |
| OpenAI news | 原生 RSS 0.15 KB | 需抓页面 | news | fetch_page |
| **8 个 X 账号** ：@OpenBMB(1.6KB) / @rohanpaul_ai(1.0KB) / @bcherny(0.8KB) / @krea_ai / @AISafetyMemes / @elonmusk / @suno / @OpenAI | RSSHub `/twitter/user/*`（已配 `TWITTER_AUTH_TOKEN`） | 推文文本即原文 | kol | rss_description |

### B 层 · signal（拿不到原文，摘要兜底）

| 源 | 原因 |
|---|---|
| `aihot.virxact.com/feed.xml`（日均约 14 条，自带 category，guid 唯一 50/50） | 二手摘要约 400 字；覆盖 7 个公众号（千问APP、数字生命卡兹克、卡尔的AI沃茨、蚂蚁百灵、小红书技术、面壁智能、火山引擎）+ 下述失败项 + 跨源发现安全网 |
| @ClaudeDevs、@runwayml | RSSHub 返回 200 但 0 条（推测近期以转推/回复为主被默认过滤），清缓存重试无效 |
| Anthropic Newsroom、LangChain Blog | 查无可用 RSS（`/rss.xml`、`/news/rss.xml`、`/engineering/rss.xml` 均 404；LangChain 返回非法 XML） |

注：`/feed/full.xml` 经逐条比对**并非全文**（48/50 条 description 与摘要版完全一致，仅 9 条推特来源带 `content:encoded` 且 ≤808B），故用 `feed.xml`。

## 10. 待运营输入

- 补充你自己订阅的高价值源，尤其 **kol 类**（X 账号、Newsletter）——该类源出爆款选题密度最高，也是最难由系统替你猜的部分。

## 11. 生产验收记录（历史快照，仅用于追溯）

- **VPS 全链路**：手动投喂 `https://www.python.org/downloads/release/python-3130/` 后，`raw_items` 入库并被定向 Scout 处理；约 31 秒内生成 3 条 candidate，随后 3 条均完成 TopicJudge，状态均为 `scored`，总分为 `91.76 / 81.18 / 81.18`。
- **自动调度**：2026-08-14 05:12（Asia/Shanghai）由 core scheduler 按 DB 配置自动投递 `topic_scout`，5 条素材批次约 50 秒完成，随后 8 条 Judge 任务全部成功；为满足“每天至少 10 条”并保持单 job 输入边界，生产调度使用 `08:00/12:00/16:00/20:00` 四个窗口，普通定时 Scout 单 job 默认最多处理 20 条素材，手动定向投喂不受该限制。
- **语义重复检查**：截至 2026-08-14 05:28，生产库 25 条 topic 在系统查重阈值 `cosine >= 0.92` 下重复率为 `0%`（最高 pair similarity 约 `0.8605`）。该结果是当前样本快照，不替代自然日连续观察。
- **业务留痕**：本次 3 条 Judge 记录均包含 `rubric_version=topic@v1`、`weight_version=1`、`vetoed_dimension=null`、`agent_run_id`；Scout/Judge 的 `agent_runs` 均包含模型、prompt version、输入/输出 token 和 trace ID。
- **版本一致性**：2026-08-14 05:32 的 Node.js 真实验收确认，Judge 的 `agent_runs.prompt_version=topic-judge@v1` 与 Langfuse generation 的 prompt version 一致；rubric 版本仍独立记录为 `rubric_version=topic@v1`。
- **Langfuse 对账**：本次验收的 1 条 Scout trace 与 3 条 Judge trace 均存在 generation；Judge 的 3 条 trace 均存在 `topic_total_score` score。生产 Langfuse observation 能查到 prompt、结构化输出、模型与 token usage；成本字段在供应商未返回价格或 Langfuse 未配置对应价格时为空。
- **当前版本回归与全量对账（2026-08-14 06:10 Asia/Shanghai）**：agents `1f86f57` 部署后，重新投喂 `https://nodejs.org/en/blog/release/v24.16.0`，3 条 raw item 生成 3 条 candidate，3 条均进入 `scored`，每条 Judge 的 `agent_runs.prompt_version=topic-judge@v1`、`rubric_version=topic@v1`、`weight_version=1`、`vetoed_dimension=null`，且记录输入/输出 token。对应 3 个 Langfuse trace 均存在 generation（模型 `deepseek-ai/DeepSeek-V3`，token 分别为 `8777 / 8742 / 8618`）和 `topic_total_score` score。历史缺失的 9 条 Judge score 已按数据库评分回填；当前生产库 46/46 个成功 Judge 均有 `agent_runs`、token 和 `topic_total_score`。
- **观测可靠性修复**：`scholar-agents:d483d2c` 为 Langfuse ingestion 增加固定 3 次有限重试，且 `4xx` 永久错误不重试；agents 全量测试 `101 passed`、ruff 和 mypy 通过，并已在 VPS 本地构建部署。该重试只保护 Langfuse 观测写入，不改变业务 job 的临时错误/永久错误重试策略。
- **当前生产快照（2026-08-14 06:13 Asia/Shanghai）**：47 条 topic、47 条已评分；1035 组向量 pair 中 `cosine >= 0.92` 的重复 pair 为 0，最高相似度 `0.903497`；source_fetch、topic_scout、topic_evaluate 三个队列均为 0 积压。该快照仍不替代自然日连续观察。
- **当前版本自动批次验收（2026-08-14 09:00 Asia/Shanghai）**：生产镜像 `scholar-core:795a619` / `scholar-agents:e2b5b51` 按 DB scheduler 自动投递 `topic_scout`（`planned_at=2026-08-14 01:00:00 UTC`），产生 12 条候选；12 条均成功 Judge，均有输入/输出 token、`topic@v1`、`weight_version=1`、`vetoed_dimension=null`。该批次 1 个 Scout trace 与 12 个 Judge trace 均有 Langfuse generation，12 个 Judge trace 均有 `topic_total_score`，缺失数均为 0。
- **历史观测边界（2026-08-14）**：全库审计仍能看到 11 个旧 Scout `agent_runs` 的 trace 缺少 generation；它们集中在旧镜像/旧 Langfuse 状态窗口（2026-08-13 19:50–20:00 UTC），无法从业务库重建原始输入/输出，未伪造历史 observation。当前部署版本从 `2026-08-14 00:54 UTC` 起的 1 个 Scout + 12 个 Judge trace parity 均为 0 缺失。
- **历史观测补评（2026-08-14 06:21 Asia/Shanghai）**：发现 9 条早期成功 Judge 只有 generation 缺失，无法从数据库重建原始 LLM 输入/输出；未伪造历史 observation，而是使用当前 agents 版本重新执行这 9 个 topic，保留原评价历史。补评任务 `msg_id=51–59` 全部成功，当前每个 topic 的最新评价均有 generation、输入/输出 token 和 `topic_total_score`：`46/46` 最新评价完整可回放，三个队列仍为 0 积压。
- **永久 source 错误修复（2026-08-13）**：`scholar-agents:6cf7347` 将“source 不存在 / source 没有 URL”归类为永久错误，不再进行最多 3 次 worker 重试；对应回归测试通过，agents 全量测试 `103 passed`、ruff 和 mypy 通过，并已在 VPS 部署。时间校准期间遗留的两个历史 source_fetch 消息已清理，当前 source_fetch 队列为 0。
- **生产时间校准（2026-08-13 06:43 Asia/Shanghai）**：发现 VPS 时钟比本次验收基准提前 24 小时，导致旧 `source_health.next_run_at` 与 pgmq visibility 时间落在未来；已停用错误时间同步服务、将系统时间校准到 `2026-08-13`，并将受影响的 22 个 `source_health` 调度时间回拨 24 小时。校准后 core/agents/Postgres/Langfuse 均健康，下一次 source interval 按校准后的 DB 时间正常计算。自然日统计从本时间点重新开始，不使用校准前的日期快照作为完整自然日证据。
- **备份恢复**：生产备份 `scholar` 与 `langfuse` 均完成加密完整性校验，并成功上传 COS、回读校验；从 COS 下载后恢复到临时库，业务库核对出 `17` 条 topics、`16` 条 topic_evaluations，`pgmq`/`vector` 扩展存在，Langfuse 临时库恢复出 `72` 条 observations。
- **初轮人工抽查结果（2026-08-14，topic@v1）**：20 条人工抽查中认可 `13/20`、部分认可 `0/20`、不认可 `7/20`，认可率 `65%`，未达到 M1 要求的 `≥80%`。不认可样本主要涉及受众认知门槛、主题抽象、主题重复和中文社区定位不匹配；该结果作为 v1 基线保留，不自动替代 v2 人工验收。
- **topic@v2 生产重评与人工验收（2026-08-14）**：生产 agents 已部署 `topic-v2-retry-local`，使用 `topic@v2` / `topic-judge@v2` 重评同一批 20 条样本；20/20 成功、20/20 记录输入/输出 token、20/20 有 Langfuse generation、20/20 有 `topic_total_score`，`topic_evaluate` 队列为 0。人工结论为认可 `10/20`、部分认可 `8/20`、不认可 `2/20`；按认可和部分认可均计入的口径，有效认可率为 `90%`，达到 M1 的 `≥80%` 门槛。完整材料见 `spec/evidence/M1-topic-review-20.md`。
- **仍需完成**：确认 Scholar 域名后配置 nginx/client 公网部署。人工质量结论不能由自动规则替代；公网入口也不能在未确认域名和 API 安全方案时擅自暴露写接口。
