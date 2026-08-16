# SPEC-002 · 数据模型

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-001

所有实体的 JSON Schema 定义在 `scholar-shared/schemas/`（单一事实来源，codegen 出 Go/Python/TS 三端类型），`scholar-core` 的 goose 迁移与之对齐，由 CI 保证一致。下面是核心实体与关系（省略 created_at/updated_at 等审计字段）。

## 1. 实体关系总览

```
Source 1──n RawItem n──1 Topic 1──n TopicEvaluation
                           │
                           1──n Article 1──n ArticleEvaluation
                                  │
                                  1──n Publication 1──n MetricSnapshot
                                  │
Insight n──────────────────────────┘ (反思产出，关联文章/选题)
AgentRun (贯穿所有环节的运行留痕，soft 关联各实体)
```

## 2. 表定义

### sources — 信源

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| name | text | 如 "Anthropic Blog"、"某大佬的推" |
| type | enum | `rss` / `rsshub` / `manual` / `crawler` |
| url | text | 订阅/抓取地址 |
| category | enum | `news` / `research` / `tutorial` / `kol` |
| weight | numeric | 信源质量权重（0–1），被反馈闭环校准 |
| enabled | boolean | |
| fetch_config | jsonb | 抓取频率、选择器等 |

### raw_items — 原始采集条目

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| source_id | fk → sources | |
| title / url / author | text | |
| content | text | 正文或摘要（清洗后的纯文本/md） |
| published_at | timestamptz | |
| content_hash | text unique | 精确去重 |
| embedding | vector(1024) | 语义去重 + 相似检索 |
| status | enum | `new` / `clustered` / `discarded` |

### topics — 选题

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| title | text | 选题标题（Agent 提炼） |
| angle | text | 切入角度描述 |
| summary | text | 选题简介 + 素材要点 |
| raw_item_ids | uuid[] | 支撑素材（可多条聚合成一个选题） |
| target_platforms | text[] | 建议平台 `xiaohongshu`/`zhihu`/`wechat` |
| status | enum | 见状态机 §3 |
| latest_score | numeric | 冗余最近一次总分，便于列表排序 |
| embedding | vector(1024) | 选题查重、与历史爆款相似度 |

### topic_evaluations — 选题评分（一题多次评，保留历史）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| topic_id | fk → topics | |
| rubric_version | text | 对应 `scholar-shared` 里的 rubric 版本号 |
| dimension_scores | jsonb | `{timeliness: 8, audience_fit: 7, ...}` 各维度分 |
| total_score | numeric | 加权总分（0–100） |
| rationale | text | 评审理由（人可读） |
| judge_model | text | 评审用的模型 |
| agent_run_id | fk → agent_runs | 溯源 |
| weight_version | int nullable | 评分时生效的 `weight_sets.version`，用于历史回放 |
| vetoed_dimension | text nullable | 触发一票否决的维度；没有 veto 时为空 |

### articles — 文章

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| topic_id | fk → topics | |
| platform | enum | `xiaohongshu` / `zhihu` / `wechat`（枚举可扩展） |
| version | int | 同一 topic+platform 的重写版本号 |
| format | enum | `markdown`（预留 `html` / `rich_text`） |
| title | text | |
| content_md | text | 正文 Markdown |
| assets | jsonb | 预留：配图/封面 `[{role:'cover', url:...}]` |
| writer_agent | text | 产出它的专家 Agent 标识 + prompt 版本 |
| status | enum | 见状态机 §3 |
| latest_score | numeric | |

### article_evaluations — 文章评分

结构同 topic_evaluations（rubric_version / dimension_scores / total_score / rationale / agent_run_id），外键换成 article_id。**平台不同 rubric 不同**，rubric_version 形如 `article/xiaohongshu@v3`。

### publications — 发布记录

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| article_id | fk → articles | |
| platform | enum | 实际发布平台 |
| platform_post_id | text | 平台侧 ID/链接 |
| published_at | timestamptz | |
| final_content_diff | text | 人工发布前的修改 diff（衡量"人工修改量"指标） |

### metric_snapshots — 数据快照（同一发布多次采样，看增长曲线）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| publication_id | fk → publications | |
| captured_at | timestamptz | |
| metrics | jsonb | `{views, likes, favorites, comments, shares, follows}` 平台字段不齐没关系，jsonb 兜底 |
| source | enum | `manual` / `import` / `api` |

### insights — 反思产出（Agent 记忆的结构化载体）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| kind | enum | `topic_lesson` / `writing_lesson` / `platform_lesson` / `source_lesson` |
| platform | enum nullable | 平台相关经验才填 |
| content | text | 经验条目（一条一个可执行的教训） |
| evidence | jsonb | 支撑证据：关联的 article_ids/publication_ids + 数据摘要 |
| confidence | numeric | 反思 Agent 给出的置信度，随证据增加而调整 |
| status | enum | `active` / `retired`（被后续数据推翻则退役） |
| embedding | vector(1024) | 写作时按选题语义检索相关经验 |

### agent_runs — Agent 运行留痕

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid pk | |
| job_type | text | `topic.evaluate` / `article.write` / ... |
| entity_type / entity_id | text / uuid | 软关联业务实体 |
| langfuse_trace_id | text | 跳转 Langfuse 看完整 trace |
| model / prompt_version | text | |
| tokens_in / tokens_out / cost_usd | numeric | 成本核算 |
| status | enum | `running` / `succeeded` / `failed` |
| correlation_id | uuid nullable | 与 Core/pgmq/Tempo 的一次端到端业务链关联 |

### 运行可靠性与审计表

| 表 | 关键字段 | 作用 |
|---|---|---|
| `state_transition_events` | entity、from/to、actor、trigger、reason、correlation_id、metadata | 每次状态变化的结构化不可变审计；状态更新与事件插入同事务 |
| `job_failures` | queue、msg_id、job_id、correlation_id、payload、read_count、error_type、retryable、archived | 永久错误或重试耗尽后的死信证据 |
| `job_receipts` | job_id、queue、msg_id、correlation_id、completed_at | 成功 job 的持久化幂等回执；处理完成后删除消息前崩溃也不会重复执行业务 |
| `source_fetch_runs` | source_id、job_id、correlation_id、attempt、ok、stats/error、started/finished_at | 每次采集执行结果，区分“已调度”和“真实执行成功” |

### Correlation 与补充字段

- `raw_items.correlation_id`、`topics.correlation_id`、`agent_runs.correlation_id` 保存一次端到端链路标识；
- `raw_items.ingest_note` 保存手动投喂备注，供后续 Scout/运营审计使用；
- `topic_evaluations.dimension_reasons` 保存每个评分维度的理由，不再只保留数值；
- `sources.archived_at` 实现软归档，保留历史素材的外键和审计链；
- 这些字段均允许历史数据为空，队列 `_meta` 也可选，确保旧数据和旧消息兼容。

## 3. 状态机

### Topic

```
candidate ──评分──▶ scored ──人工确认──▶ approved ──分派写作──▶ in_writing ──▶ written
    │                  │
    └──────────────────┴──▶ rejected（人工否决或低分自动淘汰，保留数据用于校准）
```

### Article

```
单个不可变版本：
draft ──评分──▶ scored ──┬─ 通过 ──▶ pending_review ──人工终审──▶ approved ──▶ published
                        ├─ 未通过且 version < 3 ──▶ rewrite_queued（旧版本终态）
                        └─ 未通过且 version = 3 ──▶ pending_review（标记需人工介入）

版本链：v1(rewrite_queued) ── previous_article_id ──▶ v2(draft) ──▶ 最多 v3
人工终审拒绝：pending_review ──▶ rejected
```

Article 状态属于**某一个版本行**，回炉绝不把旧行改回 draft，而是新建下一版本并用
`previous_article_id` 串成线性链。状态流转和回炉任务投递只允许由 core 的 Pipeline
Orchestrator 执行（单一写入口），agents 只追加 Article / ArticleEvaluation 结果。

## 4. 索引与约束要点

- `raw_items.content_hash` unique；embedding 建 HNSW 索引（pgvector）。
- `articles` 上 `(topic_id, platform, version)` unique。
- `articles.previous_article_id` 非空时 unique，保证一个旧版本至多产生一个直接后继版本。
- `metric_snapshots` 上 `(publication_id, captured_at)` unique。
- `job_receipts.job_id` primary key，作为跨重投的持久化幂等键。
- `job_failures(queue, msg_id)` unique，避免同一死信重复落库。
- correlation 字段使用普通/部分索引供运维下钻，但不得成为 Prometheus label。
- 所有 enum 用 Postgres enum + `scholar-shared` 的 JSON Schema enum 双向对齐，新增平台只需扩枚举，不动表结构。
