# SPEC-006 · 记忆与反馈闭环

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-002, SPEC-004, SPEC-005

## 1. 目标

把"发出去的帖子表现如何"变成 Agent 可消费的经验，让选题、写作、评分三个环节按结果论持续进化。这是区分"内容生成工具"和"会成长的 Agent 军团"的分水岭。

## 2. 数据回流

- **录入方式（渐进）**：
  - M3 起步：client 提供每篇发布记录的指标录入表单（30 秒填完一篇），支持批量粘贴/CSV 导入；
  - M4 探索：平台后台数据导出文件的解析导入；有稳定 API/工具的平台走自动化。
- **采样节奏**：发布后 24h / 72h / 7d 三个标准快照点（metric_snapshots），client 到点提醒待录数据。曲线形态本身就是信息（前 24h 冲高 vs 长尾型）。
- **关联**：publication → article → topic → source 链路在数据模型上天然打通，任何指标都能归因到"哪个信源孵化的哪个选题的哪个平台版本"。

## 3. 表现分 P（真实世界的"评分"）

- 平台内归一化：不同平台指标不可比，P 只在平台内计算。
- v1 定义：`P = Σ(指标 × 平台权重)`，再对该平台近 90 天历史做百分位归一（P75 以上为"高表现"，P25 以下为"低表现"）。
- 平台权重反映运营目标（如小红书重收藏+涨粉、知乎重赞同+收藏、公众号重完读代理指标+在看/转发），定义存 `scholar-shared`，版本化，与 rubric 同等待遇。
- 粉丝基数变化的影响：记录发布时账号粉丝数（publications 补充字段），P 可做基数修正（M4 细化）。

## 4. 反思机制（Reflector Agent）

每周定时 job（`memory.reflect`）：

```
输入：本周期新增快照的文章（高表现 + 低表现两组）+ 其文章内容、选题、评分记录
过程：
  1. 逐篇归因：这篇为什么好/差？（角度、标题、结构、时机、平台匹配……）
     ——要求引用具体证据（如"标题含具体数字，24h 收藏率是账号均值 2.3 倍"）
  2. 跨篇归纳：多篇证据支撑的模式才升级为 insight（单篇孤证只记 candidate）
  3. 与既有 insights 对账：
     - 新证据支持既有经验 → confidence 上调、evidence 追加
     - 新证据矛盾 → confidence 下调，跌破阈值则 status=retired（经验会过时，平台风向会变）
输出：insights 表的增/改（kind: topic_lesson / writing_lesson / platform_lesson / source_lesson）
     + 一份人可读的周报（client 展示，你本人也是这个系统的学习者）
```

**关键纪律：insight 必须是可执行的（actionable）**。"内容质量要高"是废话，"小红书标题带具体数字+成本（'3 天/0 基础/白嫖'）的收藏率显著高于抽象标题"才是 insight。Reflector 的 prompt 会强制要求 evidence 字段非空。

## 5. 经验的消费（记忆注入点）

| 注入点 | 消费什么 | 方式 |
|---|---|---|
| TopicScout 提角度 | topic_lesson + source_lesson | 检索 top-k 相关 insight 进 prompt |
| Writer 写作 | writing_lesson + platform_lesson（按平台过滤）+ 高表现范文 | 检索 + few-shot |
| 评分校准（SPEC-004 §4） | 高/低表现 case | 更新锚定样例、调权重 |
| 信源权重 | source_lesson | 定期调整 sources.weight |

检索策略：embedding 相似度 × confidence × 时效衰减 加权取 top-k（k≈5），避免 prompt 塞满陈旧经验。

## 6. 防退化护栏

- 经验只增不删会导致 prompt 膨胀和自我强化偏见：confidence 衰减 + retired 机制强制新陈代谢。
- 避免"过拟合爆款"：Reflector 周报必须区分"可复制的模式"与"蹭上热点的运气"；样本 < 5 的模式一律标 candidate。
- 人工否决权：client 可手动 retire 任何 insight。

## 7. 验收标准（M3–M4）

- [ ] 发帖数据录入 → 出现在归因周报，全链路 ≤ 1 周期
- [ ] insights 库 ≥ 20 条 active 且全部有 evidence
- [ ] 盲测：注入记忆 vs 不注入记忆各写同题文章，注入版评分/人工偏好胜出
- [ ] 出现 ≥1 条被数据推翻而 retired 的经验（证明新陈代谢机制真的在工作）
