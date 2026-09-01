# SPEC-009 · M3 数据回流与记忆 v1 实施计划

- 状态：Implemented（工程闭环完成；真实运营样本持续验收；生产运行编排以 SPEC-010 为准）
- 日期：2026-08-16
- 依赖：SPEC-002、SPEC-004、SPEC-005、SPEC-006

## 1. 本里程碑的边界

本文件是 M3 的实施与验收记录，不替代当前内容生产工作流。它消费 `human_review` 之后人工发布形成的 Publication，属于工作流之外的反馈闭环；不得把其中的周期调度理解为六阶段生产链路的调度。

M3 把一条真实发布记录变成可回放的数据闭环：

```text
Publication
  → 24h / 72h / 7d MetricSnapshot
  → 平台内同窗口 Performance P 百分位
  → weekly memory.reflect
  → WeeklyReport + actionable Insights
  → TopicScout / Writer 检索注入
```

M3 只产出校准建议，不自动修改 rubric、权重或信源权重。任何自动调权属于 M4，
必须经过人工确认。配图、富文本和自动发布也不在本里程碑内。

## 2. 数据与正确性决策

### 2.1 标准快照窗口

- 标准窗口固定为 `h24`、`h72`、`d7`，补录或特殊观察使用 `custom`。
- 同一 Publication 的同一标准窗口最多一条快照；`custom` 可有多条。
- 快照是累计指标，不覆盖历史。录入来源记录为 `manual` / `import` / `api`。
- 提醒由 `published_at + 窗口` 确定；到期且尚无该窗口快照即为待录。

### 2.2 表现分 P

- 原始分严格使用 `performance-weights.v1.yaml` 对应平台权重计算：
  `raw = Σ(metric × weight)`。
- 百分位只与“同平台、同标准窗口、近 90 天”的样本比较，禁止把 24h 和 7d
  快照混在一起；`custom` 不参与百分位。
- 每条快照固化原始分、百分位和权重版本，保证周报可回放。新增样本后 Core 会
  重算同平台同窗口近 90 天百分位；历史原始分不变。
- P75 及以上为 high，P25 及以下为 low，中间为 normal。样本不足 5 条时周报
  明示冷启动，禁止据此调权。

### 2.3 CSV 导入

Client 负责解析 CSV 和展示预检查，Core 接收类型化批量 JSON 并在一个事务中写入。
任何一行非法则整批失败，避免“看似成功但只导入一半”。模板列：

```text
publicationId,snapshotWindow,capturedAt,views,likes,favorites,comments,shares,follows
```

### 2.4 Reflector 与记忆新陈代谢

- 每周一 09:00（Asia/Shanghai）默认反思上一自然周，也可从 Client 手动触发。
- 统计、表现分、相关性和 high/low case 由确定性代码计算；LLM 只做有证据的归因
  与可执行经验提炼。
- 单篇或证据 publication 少于 5 条的模式只能是 `candidate`；达到 5 条独立证据且
  confidence ≥ 0.65 才可成为 `active`。
- 支持既有 insight：追加证据与提高 confidence；反证则降低 confidence，低于 0.35
  退役。人工可随时 retire，Reflector 不得自动复活人工退役项。
- Insight 必须至少关联一个 Article 或 Publication，并包含非空数据说明。

### 2.5 记忆检索

- TopicScout：`topic_lesson + source_lesson`。
- Writer：`writing_lesson + platform_lesson`，按平台过滤。
- 排序：embedding cosine similarity × confidence × 180 天时效衰减，top-k=5。
- 没有 embedding 或冷启动时，以 confidence × 时效作为降级排序；不得因记忆服务
  无数据阻断采集或写作。

## 3. 交付清单

### scholar-shared

- [x] 标准快照窗口、批量录入、表现面板、Insight、WeeklyReport API 契约
- [x] `memory_reflect` 调度契约及三端 codegen
- [x] 表现权重与平台枚举完整性校验

### scholar-core

- [x] M3 migration：快照评分字段、performance weight sets、weekly reports
- [x] 单条/批量指标写入、同窗口防重、百分位重算
- [x] 24h/72h/7d 待录提醒与平台面板
- [x] Insight 查询/人工退役、WeeklyReport 查询、手动反思入队
- [x] 每周反思事务性调度

### scholar-agents

- [x] Reflector：确定性统计输入 + 结构化归因输出 + evidence 纪律
- [x] WeeklyReport / Insight 结果落库与幂等
- [x] TopicScout / Writer 的相关记忆注入

### scholar-client

- [x] Publication 待录列表与 30 秒单篇录入表单
- [x] CSV 模板、预检查和原子批量导入
- [x] 平台表现面板、high/low case 与窗口覆盖
- [x] 周报、校准相关性、case 与 Insights 管理

### scholar-infra

- [x] queue-specific `memory_reflect` worker
- [x] Fake AI 跨仓库 E2E：发布 → 三窗口快照 → P → 反思 → 周报/Insight → 注入可观察
- [x] M3 审计脚本和生产部署说明

## 4. 工程验收

- [x] 单条与 CSV 批量录入都能在 API、DB、Client 中回读，重复标准窗口返回 409。
- [x] 同平台同窗口的 P 百分位可由固定样本确定性复算，跨平台/跨窗口互不污染。
- [x] 到期提醒只列缺失窗口，补录后立即消失。
- [x] 一次 `memory.reflect` 产生一份幂等周报；所有 insight 都有结构化 evidence。
- [x] 周报包含 topic/article 总分及维度分与 P 的相关性；样本不足明确标记冷启动。
- [x] TopicScout 和 Writer 在有 active insight 时收到 top-k 记忆，无记忆时行为不退化。
- [x] Shared/Core/Agents/Client 单测与跨仓库 E2E 全部通过。

工程验收记录（2026-08-16）：Fake AI + 真实 PostgreSQL/pgmq E2E 创建同平台同窗口
三条样本，表现分百分位确定为 `0/50/100`；标准提醒在补录后消失；
`memory_reflect` 生成冷启动周报、相关性数据和 evidence 非空的 candidate insight，
人工退役写入 `manual_status_override`。该记录只证明工程正确性，不替代真实运营数据。

生产部署记录（2026-08-17）：VPS 数据库迁移到 goose v9，部署 Core `8f3be0b`、
Client `06157f2`、Agents `3f7fb3f` 和独立 `memory_reflect` worker。写作、文章评分与
Reflector 路由到 Vtrix OpenAI-compatible `gpt-5.6-sol`；生产探针分别验证了
`chat/completions`、JSON Schema 结构化输出和 tool call。空数据触发真实
`memory.reflect` 后生成 `sample_count=0`、`coldStart=true` 的周报，队列清空且无
未归档失败。当前生产库没有 Publication，因此没有生成或伪造 Insight；真实运营验收
仍保持未完成。

## 5. 真实运营验收

- [ ] 发帖数据录入后在下一份归因周报出现，全链路不超过一个周周期。
- [ ] `active` insights ≥20，且每条 evidence 非空。
- [ ] 产出首份真实评分校准报告；只给建议，不自动调权。

真实运营验收依赖真实 Publication 与平台后台数据，Fake AI 只证明工程回归，不能替代
业务样本或据此宣称记忆有效。
