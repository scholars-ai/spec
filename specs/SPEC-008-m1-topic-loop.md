# SPEC-008 · M1 实施计划：选题闭环

- 状态：Accepted
- 日期：2026-08-08
- 依赖：SPEC-003（采集）、SPEC-004（评分）、SPEC-007 §M1
- 前置决策：ADR-004（DB 自托管）、ADR-005 v2（embedding 走 SiliconFlow API，Ollama 降为备用）

M0 已交付五仓骨架。本 spec 把 SPEC-007 的 M1 里程碑落成可执行清单。

## 1. 目标

打通五步流程的前两步——**采集 → 聚合选题 → 评分 → 人工确认**，全链路运行在 VPS 上。
产出物的判定标准：**每天打开看板，就有带评分和理由的可写选题**。

## 2. 已拍板的技术决策

| 项 | 决定 | 依据 |
|---|---|---|
| 数据库 | VPS 自托管 Postgres 17（业务库 + Langfuse 库同实例），每日 pg_dump 推 COS | ADR-004 |
| Embedding | **SiliconFlow API `BAAI/bge-m3`（原生 1024 维）**，输入截断 600 字符；本机 Ollama 降为 `EMBED_BACKEND=ollama` 备用 | ADR-005 v2（本机方案实测不可行，见该 ADR §决策变更） |
| 可观测 | Langfuse 自托管，trace 保留 30 天 | ADR-004 |

## 3. 数据流（M1 结束时的运行态）

```
core scheduler tick（每分钟，仅为检查粒度）
  读 DB 调度配置 ──▶ 到期的 source 投递 source_fetch（事务性入队）
                            │
agents SourcingHandler ◀────┘
  feedparser 拉取 → trafilatura 清洗 → content_hash 精确去重
  → embedding（SiliconFlow bge-m3，1024d）→ 语义去重（近 14 天 cos > 0.92）
  → raw_items(status=new)
                            │
core scheduler ──▶ 到期的 topic_scout（默认每日 2 次，可配）
                            │
agents TopicScout（自研 loop 首次实战）
  聚类素材 → 每簇提 1–3 个选题角度 → 与既有 topics 向量查重
  → topics(status=candidate)
                            │
core harvester ──▶ 新 candidate **事件驱动**立即投递 topic_evaluate（不等固定时刻）
                            │
agents TopicJudge（structured output 首次实战）
  读 rubrics/topic.v1.yaml + DB weight_sets → 维度分 + 理由
  → topic_evaluations + agent_runs（挂 Langfuse trace）
                            │
core harvester ──▶ 状态机 candidate→scored；≥75 推荐 / 60–75 备选 / <60 自动 rejected
                            │
scholar-client 选题看板 ──▶ 人工 approve / reject（scored → approved / rejected）
```

### 3.1 调度是配置，不是硬编码（2026-08-11 修订）

初版把「每小时采集、每日 2 次 scout」写成固定 cron 表达式。这是**默认运行策略，不是业务规则**，因此改为存 DB、由 client 修改：

| job | 调度方式 | 默认值 | 可配粒度 |
|---|---|---|---|
| `source_fetch` | interval | 每 60 分钟 | **全局默认 + 每个 source 单独覆盖**；可暂停；可手动立即触发 |
| `topic_scout` | daily times | 08:00 / 20:00（Asia/Shanghai） | 执行时刻、时区、启停、`min_new_items`（新素材不足则跳过） |
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
- [x] **调度设置页**：全局采集间隔、scout 执行时刻/时区/min_new_items、evaluate 启停与并发。用表单生成 cron，不让用户手写表达式
- [x] 手动投喂入口（高频动作，一键贴 URL）

### scholar-infra + 部署
- [x] `compose.prod.yaml` 增加 postgres 服务（自建镜像）+ named volume；langfuse 指向同实例独立 database
- [x] 备份：每日 `pg_dump` → 加密 → 本地保留 7 份 → COS 离机副本；上传后回读校验，COS 下载恢复演练已完成
- [x] 磁盘监控告警（> 85%），cron 每 6h
- [x] GHCR 镜像构建 CI（core/agents）+ deploy.sh 已具备版本部署路径；core/agents/Langfuse/Postgres 已在 VPS 运行
- [ ] nginx 反代（复用现有实例）：client 访问 core API

## 5. 实施顺序

每步结束都可独立验证，避免"全写完再调试"：

1. **infra 先行**：VPS 起自托管 Postgres + goose 迁移 + 备份/监控就位（数据地基）
2. **契约扩展**：shared 改 OpenAPI + 评分 schema → codegen
3. **core CRUD**：sources 管理 + 手动投喂（先把 API 链路走通）
4. **agents embed + SourcingHandler**：先接 2–3 个源本地验证，再扩到 ≥8
5. **core cron**：采集自动化
6. **agents TopicScout → TopicJudge**：**同时接 Langfuse**（调 prompt 全靠看 trace，这步工作量最大）
7. **core harvester**：闭环在后端成立
8. **client 看板 + 信源管理**：有真数据后再做界面
9. **VPS 部署 + 试运行一周** → 逐条打勾 §6

## 6. 验收标准

**功能**
- [x] ≥8 个信源接入并完成真实采集；单源失败隔离、连续失败状态可观测
- [ ] 每天自动产出 ≥10 条选题候选，语义重复率 < 10%（已完成一次受控批量验收：10 条 Scout/Judge 任务，仍需按自然日持续运行验证）
- [x] 手动投喂 URL → 出现在候选池 < 2 分钟（2026-08-14 真实验收：新 URL 入库后定向 Scout 生成 3 条候选约 31 秒）
- [x] 每条候选具备总分、6 维度分、人可读理由
- [x] approve/reject 生效，非法状态流转被拒（409）

**质量（M1 最重要的软验收）**
- [ ] 人工抽查 20 条评分，**理由认可率 ≥ 80%**；不认可的能归因为 rubric 定义问题或模型问题（需要人工参与）
- [x] 每次评分记录 rubric_version 与权重版本，可回放

**工程**
- [x] 全链路 Langfuse trace 可查（prompt / 输出 / token / 成本）
- [x] quota/余额/无效 key 等永久错误不重复重试；临时错误的 job 重试次数有限且可观测
- [x] agents 崩溃重启后 job 不丢（pgmq visibility timeout 实测）
- [x] 数据库备份产出 + **恢复演练成功**（本地加密副本与 COS 下载副本均验证）
- [x] CI：队列名一致性校验生效；GHCR 镜像构建通过，VPS 生产服务已部署

**调度可配（§3.1）**
- [x] client 改采集间隔后 ≤ 1 个 tick 生效，且**重启 core 不被环境变量覆盖**
- [x] 单个 source 设为 6 小时 / 暂停，各自按自身配置执行，不受全局默认影响
- [x] scout 改执行时刻后按新时刻触发；`min_new_items` 不满足时跳过并留痕
- [x] candidate 产生到 `topic_evaluate` 入队 < 1 分钟（事件驱动，非固定时刻）
- [x] 非法配置（时间格式/间隔越界/重复时刻）被 API 拒绝，scheduler 不崩

## 7. 明确不做（防止范围膨胀）

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

## 11. 生产验收记录

- **VPS 全链路**：手动投喂 `https://www.python.org/downloads/release/python-3130/` 后，`raw_items` 入库并被定向 Scout 处理；约 31 秒内生成 3 条 candidate，随后 3 条均完成 TopicJudge，状态均为 `scored`，总分为 `91.76 / 81.18 / 81.18`。
- **自动调度**：2026-08-14 05:12（Asia/Shanghai）由 core scheduler 按 DB 配置自动投递 `topic_scout`，5 条素材批次约 50 秒完成，随后 8 条 Judge 任务全部成功；当前调度已恢复默认 `08:00/20:00`，普通定时 Scout 单 job 默认最多处理 5 条素材。
- **语义重复检查**：截至 2026-08-14 05:28，生产库 25 条 topic 在系统查重阈值 `cosine >= 0.92` 下重复率为 `0%`（最高 pair similarity 约 `0.8605`）。该结果是当前样本快照，不替代自然日连续观察。
- **业务留痕**：本次 3 条 Judge 记录均包含 `rubric_version=topic@v1`、`weight_version=1`、`vetoed_dimension=null`、`agent_run_id`；Scout/Judge 的 `agent_runs` 均包含模型、prompt version、输入/输出 token 和 trace ID。
- **版本一致性**：2026-08-14 05:32 的 Node.js 真实验收确认，Judge 的 `agent_runs.prompt_version=topic-judge@v1` 与 Langfuse generation 的 prompt version 一致；rubric 版本仍独立记录为 `rubric_version=topic@v1`。
- **Langfuse 对账**：本次验收的 1 条 Scout trace 与 3 条 Judge trace 均存在 generation；Judge 的 3 条 trace 均存在 `topic_total_score` score。生产 Langfuse observation 能查到 prompt、结构化输出、模型与 token usage；成本字段在供应商未返回价格或 Langfuse 未配置对应价格时为空。
- **备份恢复**：生产备份 `scholar` 与 `langfuse` 均完成加密完整性校验，并成功上传 COS、回读校验；从 COS 下载后恢复到临时库，业务库核对出 `17` 条 topics、`16` 条 topic_evaluations，`pgmq`/`vector` 扩展存在，Langfuse 临时库恢复出 `72` 条 observations。
- **仍需完成**：自然日连续运行观察（每天自动 ≥10 条且语义重复率 <10%）、人工抽查 20 条评分理由（认可率 ≥80%），以及确认 Scholar 域名后配置 nginx/client 公网部署。这三项不能用单次自动触发、受控投喂或自动检查替代。
