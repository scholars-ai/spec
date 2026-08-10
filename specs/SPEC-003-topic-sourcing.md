# SPEC-003 · 选题采集（Sourcing）

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-001, SPEC-002

## 1. 目标

每天自动、稳定地把 AI 领域值得写的素材汇入系统，聚合成**带素材支撑的选题候选**，供评分环节消费。质量优先于数量：宁可 10 条高质量候选，不要 100 条噪音。

## 2. 信源分类与角色

### 2.1 两类角色（M1 实测后新增的核心设计）

素材的**厚薄**直接决定文章质量上限（"垃圾进垃圾出"），而不同信源给到的内容量差两个数量级。因此每个信源标注角色，`sources.fetch_config` 承载：

| 角色 | 含义 | 采集策略 | 对写作的意义 |
|---|---|---|---|
| `material` | 能拿到原文/完整摘要 | RSS 全文字段，或抓原文页 | 可支撑知乎/公众号深度长文 |
| `signal` | 只能拿到二手摘要（几百字） | 只存摘要 | 够判断"值不值得写"，深度素材需 M2 增强环节补 |

`fetch_config` 字段：
- `role`: `material` | `signal`
- `full_text`: `rss_description`（RSS 里已有足够正文）| `fetch_page`（需 trafilatura 抓原文页）

判定依据不是猜测而是实测：M1 开工前逐源验证了正文获取量，结果见 SPEC-008 §3。

### 2.2 类别（沿用 SPEC-002 的 source_category）

| 类别 | 说明 |
|---|---|
| news | 官方动态、产品发布 |
| research | 论文、技术深度文 |
| tutorial | 学习资料、最佳实践 |
| kol | 大佬分享（X 账号、Newsletter） |

- 接入方式优先级：**RSS 原生 > RSSHub 路由 > trafilatura 抓页面 > 二手摘要兜底**。VPS 已有 RSSHub 实例（已配 `TWITTER_AUTH_TOKEN` / `ZHIHU_COOKIES` / `GITHUB_ACCESS_TOKEN`），直接复用。
- 每个信源带 `weight`（初始人工设定 0.5–0.8），后续由反馈闭环校准（SPEC-006：某信源孵化的文章表现持续好 → 权重上调）。
- 支持**手动投喂**：client 上贴一个 URL，抓取正文入库，等同一条 raw_item（很多好选题来自自己刷到的东西，这个入口必须顺手）。

### 2.3 明确拿不到的：微信公众号

实测 RSSHub 的 10 个 `/wechat/*` 路由**全部是第三方镜像的代理**（`ce` 返回 503、`uread` 超时），RSSHub 自身抓不了微信；直接抓 `mp.weixin.qq.com` 页面被反爬拦截（trafilatura 仅提取到 36 字符噪音）。

结论：**不为微信自研爬虫**（SPEC-008 §7）。公众号内容通过 `signal` 型聚合源的摘要覆盖。实际损失有限——重要新闻必被多个网页源覆盖，M2 素材增强可按关键词找到可抓取的替代来源。

## 3. 采集流水线

```
core cron 每小时投递 pgmq job ──▶ agents 侧 fetch(source)
   ──▶ role=material 且 full_text=fetch_page 时抓原文页（trafilatura）
   ──▶ 清洗 ──▶ 精确去重（guid 优先，回退 content_hash）
   ──▶ embedding（Ollama，1024d）──▶ 语义去重（近 14 天相似度 > 0.92 合并）
   ──▶ 入库 status=new
```

- 清洗规则：HTML → 纯文本/Markdown（trafilatura），去广告导航，截断超长正文（保留前 8k token）。
- **去重键优先用 feed 的 `guid`**（实测聚合源 guid 唯一性 50/50），无 guid 时回退 `sha256(正文)`。
- **XML 解析必须用 `defusedxml` 或 feedparser**（第三方 feed 是不可信输入，防 XXE / billion-laughs）。
- 语义去重命中时，**保留素材更厚的那条**（material 优先于 signal），薄的那条作为补充素材记录。

### 3.1 两道入库前的闸门（M1 实测踩坑后新增）

**feed 的条目数与时间范围完全不可信**，必须双重设限，且都要在 embedding 之前生效（否则白烧额度）：

| 闸门 | 默认 | 可覆盖字段 | 实测依据 |
|---|---|---|---|
| 单次条目上限 | 30 条 | `fetch_config.max_items` | arXiv 的 RSS 一次返回当天全部论文：cs.AI **295 篇**、cs.CL **119 篇**。两个源即可一天灌入 400+ 条，淹没为"日均 10 条候选"设计的选题池 |
| 时效性窗口 | 14 天 | `fetch_config.max_age_days` | OpenAI 的 news RSS 返回**全部历史归档**：单次 **1115 条**，回溯数年。首次全量采集因此灌入 1004 条旧闻，占当时全库 61%，其中 60% 早于 30 天 |

时效性窗口是更本质的约束——**选题系统要的是新鲜资讯**，`max_items` 只是防止单源体量失控。
两者顺序：先按 `max_items` 截断条目，再逐条按 `published_at` 过滤，最后才 embedding。

对无 `published_at` 的条目不做时效过滤（宁可多收，交给后续评分环节判断）。

### 3.2 去重 vs 聚类的职责边界

实测发现语义相似度落在 0.85–0.90 区间的条目**无法用阈值区分两种情况**：

| 条目 A | 条目 B | 相似度 | 期望行为 |
|---|---|---|---|
| `Agent Plugins 1.0.0 发布：谷歌、亚马逊、微软等支持…` | `Agent Plugins package your skills, tools, and more` | 0.88 | 同一事件的中英双源，**应合并** |
| `How ChatGPT adoption has expanded` | `From asking to doing: How the world is putting ChatGPT to work` | 0.89 | 两篇不同文章，**不应合并** |

因此明确分工，不试图靠调阈值解决：

- **raw_items 语义去重（0.92）**：只挡"同一条内容重复入库"，保持高阈值（宁可漏合并，不可错杀素材）
- **同一事件的多源合并**：属 §4 TopicScout 的聚类职责——它有 LLM 语义理解，能判断"同一事件的不同角度"与"两个不同话题"的差别

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
