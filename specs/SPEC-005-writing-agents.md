# SPEC-005 · 写作 Agent 军团

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-002, SPEC-004

## 1. 目标与形态

一个 topic 被 approved 后，按 target_platforms 分派给对应的**平台专家 Agent**并行写作，各自产出适配该平台的 Markdown 成品。专家 Agent 是**同构不同魂**：共享同一套写作流水线骨架，差异全部收敛在"平台档案（Platform Profile）"里——这是保证"后续新增平台专家 Agent"低成本的关键设计。

## 2. Agent 编排（自研 runtime，ADR-002）

```
article.write job (topic_id, platform)
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
- 回炉：评分不达标时，评审意见作为额外输入重跑 ③④（不重跑大纲，除非结构维度不达标）。

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

- [ ] 一个 approved topic 10 分钟内并行产出 3 平台文章并完成评分
- [ ] 三平台文章风格肉眼可辨（盲测能分出哪篇是给哪个平台的）
- [ ] Formatter 拦截率可观测：硬性约束violations 在正式评分前被修复
- [ ] 人工修改量（publications.final_content_diff）平均 < 30%
- [ ] 演练"新增平台"：按 §3 流程加一个假想平台，全程不改流水线代码

## 6. 开放问题

- [ ] 三平台同题写作是否共享一次素材消化（省 token）再分叉，还是完全独立（隔离性好）——M2 实测定
- [ ] SelfCritic 与正式 Judge 的 rubric 是否同源（同源省维护，但可能让自查"应试化"）
