# SPEC-005 · 写作 Agent 军团

- 状态：Implemented（工程闭环完成；真实运营质量指标持续验收）
- 日期：2026-08-06
- 依赖：SPEC-002, SPEC-004

## 1. 目标与形态

一个 topic 被 approved 后，按 target_platforms 分派给对应的**平台专家 Agent**并行写作，各自产出适配该平台的 Markdown 成品。专家 Agent 是**同构不同魂**：共享同一套写作流水线骨架，差异全部收敛在"平台档案（Platform Profile）"里——这是保证"后续新增平台专家 Agent"低成本的关键设计。

## 2. Agent 编排（自研 runtime，ADR-002）

```
article.write job (topic_id, platform, rewrite?)
   │
   ▼
WriterOrchestrator（每平台一个实例，注入对应 Platform Profile）
   ├── ① ContextBuilder：拉取素材（raw_items 原文）、选题角度、
   │      检索相关 insights（该平台的 writing_lesson / platform_lesson）
   │      + 该平台历史高分文章 top-3 作为风格参照（few-shot）
   ├── ② Outliner 子 Agent：产出大纲（钩子、结构、核心论点）
   ├── ③ Drafter 子 Agent：按大纲成文（平台风格约束在 system prompt）
   ├── ④ SelfCritic 子 Agent：对照平台 rubric 自查一轮并修订
   │      （廉价的预评分，减少正式评分回炉率）
   └── ⑤ Formatter：确定性代码（非 LLM）做格式收尾：
          字数校验、md lint、敏感词扫描、平台硬性约束检查
   ▼
产出 Article(draft) → core 触发 article.evaluate（SPEC-004 的独立 Judge）
```

- 模型分配：Outliner/Drafter 用 `claude-sonnet-5`；SelfCritic 可用更便宜档位。prompt 版本号记入 articles.writer_agent。
- 回炉：评分不达标时，core 原子执行旧版本 `scored → rewrite_queued` 并投递下一版本任务；Writer 新建 Article 行并写入 `previous_article_id`，不覆盖旧稿。
- `rewrite.evaluationFeedback` 组合总体理由、每维分数/理由和 veto 维度；默认只重跑 ③④，`structure < 6` 时设置 `redoOutline=true` 重跑 ②。
- 版本上限为 v3：v1 最多回炉到 v2、v3；v3 仍不通过时不再投递写作任务，交人工介入。

## 3. Platform Profile（平台档案）

每个平台一份声明式配置（`scholar-shared/` 内 YAML + prompt 片段，结构由 JSON Schema 约束），包含：

| 组成 | 内容 |
|---|---|
| voice | 人设与语气（如小红书：闺蜜感分享者；知乎：一线从业者；公众号：深度观察者） |
| structure_template | 推荐结构（钩子模式、段落长度、结尾 CTA 形态） |
| hard_constraints | 硬性约束：字数区间、标题字数、话题标签数量、图片位数（预留） |
| style_rules | 风格细则：emoji 密度、口语化程度、专业术语处理方式 |
| rubric_ref | 对应的文章评分 rubric 版本（SPEC-004） |
| exemplars | 高分范文引用（初始人工挑选，后续从高表现文章自动补充） |

首批三个平台的关键差异（v1，细则在实现时打磨）：

- **小红书**：600–1200 字，强钩子标题 ≤20 字，高密度分段 + emoji，结尾引导收藏/评论，附 3–5 个话题标签。
- **知乎**：1500–4000 字，问题导向开头，重论证与引用，可有代码/公式，克制使用 emoji。
- **公众号**：1200–3000 字，重叙事与金句，小标题分节，开头 100 字决定留存，结尾引导在看/转发。

**新增平台 = 新增一份 Platform Profile + 一份平台 rubric + 数据枚举扩一个值**，不改流水线代码。这是扩展性的验收标准。

## 4. 格式与资产（预留设计）

- 第一阶段统一输出 **Markdown**（articles.format = markdown）。
- 渲染层抽象：`Renderer` 接口 `render(article, targetFormat)`，第一阶段只有 `MarkdownRenderer`（原样输出）；后续加 `WechatRichTextRenderer`（md → 公众号排版 HTML）等，不影响写作层。
- 配图/封面：articles.assets 字段已预留（SPEC-002）。M3+ 增加 `Illustrator` 子 Agent：产出配图 brief → 文生图 → 存 COS → 回填 assets。写作时 Drafter 已被要求在 md 中留 `<!-- image: 描述 -->` 占位注释，届时无缝接上。

## 5. 验收标准（M2）

- [x] 一个 approved topic 10 分钟内并行产出 3 平台文章并完成评分；跨仓库 Fake AI E2E 对批准时间到三平台 `pending_review` 的墙钟时间做 ≤10 分钟断言
- [ ] 三平台文章风格肉眼可辨（盲测能分出哪篇是给哪个平台的）
- [x] Formatter 拦截率可观测：`scholar_agent_formatter_violations_total{platform,stage}` 区分首次拦截与修复后仍失败，`scholar-infra/scripts/m2-audit.sh` 可直接查询
- [ ] 人工修改量（publications.final_content_diff）平均 < 30%
- [x] 新增平台扩展性回归：Shared 校验要求 Platform enum、Profile、rubric 一一覆盖；Agents 测试参数化遍历全部 Platform，WriterOrchestrator 内无平台分支。扩 enum/profile/rubric 与 DB enum 后无需修改流水线代码，仓库不长期保留污染生产枚举的假想平台

前两项未完成项是**真实运营质量验收**，不是缺失的工程功能：盲测必须使用真实模型生成的同题三平台文章；人工修改量必须由真实 Publication 样本计算，不能用 Fake AI 或伪造发布记录冒充。

### 5.1 2026-08-16 工程验收记录

- `scholar-shared` OpenAPI 提供 Article 列表/详情/评分历史、终审通过/拒绝与 Publication 登记契约。
- Article 原稿保持不可变；浏览器编辑人工终稿，Core 按“标题 + Markdown 正文”生成行级 diff，并用 Unicode 字符级 Levenshtein 距离计算 `edit_ratio`。
- 首次 Publication 登记与 `approved → published` 状态流转、结构化审计处于同一数据库事务；同一平台帖子 ID/链接重复登记返回 409。
- Client 提供版本链切换、Agent 原稿/人工终稿双栏、diff 预览、评分理由、复制/下载 Markdown、终审与发布历史。
- 跨仓库 E2E 已覆盖 v2 回炉、终审通过/拒绝、发布登记、修改比例、状态审计、防重与观测故障隔离。

## 6. 已决策问题

- [x] M2 保持三平台独立 ContextBuilder/Writer 执行：当前低流量阶段优先故障隔离和可回放性；素材共享作为真实成本数据证明必要后再优化。
- [x] SelfCritic 读取 Platform Profile 与硬约束，正式 ArticleJudge 独立读取版本化 rubric；两者共享平台目标但不共享完整评分 prompt，避免 SelfCritic 只针对 Judge 应试。
