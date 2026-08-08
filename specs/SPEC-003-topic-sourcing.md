# SPEC-003 · 选题采集（Sourcing）

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-001, SPEC-002

## 1. 目标

每天自动、稳定地把 AI 领域值得写的素材汇入系统，聚合成**带素材支撑的选题候选**，供评分环节消费。质量优先于数量：宁可 10 条高质量候选，不要 100 条噪音。

## 2. 信源分类与首批清单

| 类别 | 说明 | 首批示例（M1 落地时确认） |
|---|---|---|
| news | 官方动态、产品发布 | Anthropic/OpenAI/DeepMind 官方 blog、Hacker News AI 关键词 |
| research | 论文、技术深度文 | arXiv cs.CL/cs.AI 精选、Papers with Code trending |
| tutorial | 学习资料、最佳实践 | 各家官方 docs 更新、优质工程博客 |
| kol | 大佬分享 | Twitter/X 大佬列表（走 RSSHub 路由）、知名 Newsletter |

- 接入方式优先级：**RSS 原生 > RSSHub 路由 >（少数）Playwright 抓取**。VPS 上已有 RSSHub 实例，直接复用。
- 每个信源带 `weight`（初始人工设定 0.5–0.8），后续由反馈闭环校准（SPEC-006：某信源孵化的文章表现持续好 → 权重上调）。
- 支持**手动投喂**：client 上贴一个 URL，core 抓取正文入库，等同一条 raw_item（很多好选题来自你自己刷到的东西，这个入口必须顺手）。

## 3. 采集流水线

```
core cron 每小时投递 pgmq job ──▶ agents 侧 fetch(source) ──▶ 清洗/正文提取 ──▶ 精确去重(content_hash)
   ──▶ embedding ──▶ 语义去重(与近14天 raw_items 相似度 > 0.92 则合并) ──▶ 入库 status=new
```

清洗规则：HTML → Markdown（turndown），去广告/导航噪音，截断超长正文（保留前 8k token + 摘要）。

## 4. 选题聚合（TopicScout Agent）

采集只产生"素材"，**选题是素材之上的创作视角**，由 `scholar-agents` 的 TopicScout 完成：

- 触发：每天定时 2 次（早/晚），消费 `status=new` 的 raw_items。
- 职责：
  1. 聚类相关素材（同一事件的多篇报道合并）；
  2. 对每簇素材提出 1–3 个**选题角度**（同一事件对小红书和知乎的写法角度可能完全不同）；
  3. 输出 Topic：标题、切入角度、素材要点、建议平台；
  4. 与现有 topics 查重（embedding 相似度），撞车则丢弃或标注为已有选题的补充素材。
- 检索增强：TopicScout 生成角度前，检索 insights 表中 `topic_lesson` 类经验（"什么样的角度在什么平台容易爆"），作为 prompt 上下文。

## 5. 验收标准（M1）

- [ ] ≥8 个信源稳定采集，单源失败不影响整体（隔离重试 + 连续失败告警）
- [ ] 每天产出 ≥10 条选题候选，语义重复率 < 10%
- [ ] 手动投喂 URL 到出现在候选池 < 2 分钟
- [ ] client 可管理信源（增删改、启停、看最近抓取状态）

## 6. 开放问题

- [ ] X/Twitter 抓取稳定性（RSSHub 路由经常失效），是否需要备用方案（M1 观察）
- [ ] 英文素材是否在采集阶段就翻译摘要，还是留给写作 Agent（倾向后者，保留原文信息量）
