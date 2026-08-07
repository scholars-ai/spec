# SPEC-007 · 路线图

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-000 ~ 006

节奏假设：业余时间迭代，每个里程碑 2–4 周弹性。**每个里程碑结束时系统都处于"可用"状态**，不搞长期不可用的大重构。

## M0 · 地基（本周）

- [x] 设计 spec（本仓库）
- [ ] 创建 GitHub Organization `scholars-ai`（网页手动，见下方备注）
- [ ] 建 6 个仓库：spec / scholar-shared / scholar-core / scholar-agents / scholar-console / scholar-infra，推入初始骨架
- [ ] scholar-shared：JSON Schema 首版（实体 + job payload + rubric 结构）+ rubric YAML + codegen 管线（Go/Python/TS）
- [ ] scholar-core：Go 骨架（chi + sqlc + oapi-codegen）+ goose 迁移（SPEC-002 全部表）
- [ ] scholar-infra：docker-compose（core / agents / langfuse；本地 Postgres 含 pgmq+pgvector 扩展）+ 本地开发 compose
- [ ] CI：core（go vet/test/build）、agents（ruff/mypy/pytest）、console（lint/typecheck）、shared（codegen 无 diff 校验）跑通
- 验收：本地 `docker compose up` 起全套，core 健康检查通过，空管道可插入一条手动 topic

## M1 · 选题闭环（人工确认）

- [ ] Sourcing 模块：≥8 信源接入（复用 VPS RSSHub）+ 手动投喂 URL
- [ ] 去重（hash + embedding）、TopicScout 聚合选题
- [ ] 选题评分 rubric topic@v1 + TopicJudge Agent + Langfuse 接入
- [ ] console v1：选题看板（候选/评分/理由/确认与否决）、信源管理
- [ ] 部署上 VPS，定时任务跑起来
- 验收：SPEC-003 §5 + SPEC-004 验收标准 M1 条目

## M2 · 写作军团

- [ ] Platform Profile × 3（小红书/知乎/公众号）+ WriterOrchestrator 流水线
- [ ] 文章评分 rubric × 3 + ArticleJudge + 回炉机制
- [ ] console：文章审阅页（diff 编辑、终审通过/拒绝、复制导出 md）
- [ ] publications 手动登记（发了哪、链接、最终稿 diff）
- 验收：SPEC-005 §5 条目；开始真实运营发帖

## M3 · 数据回流与记忆 v1

- [ ] 指标录入（表单 + CSV 导入）+ 24h/72h/7d 快照提醒
- [ ] 表现分 P 计算 + 数据面板（console）
- [ ] Reflector Agent + insights 库 + 周报
- [ ] 记忆注入 TopicScout / Writer
- [ ] 首份评分校准报告（只报告，不自动调权）
- 验收：SPEC-006 §7 前两条

## M4 · 校准与进化

- [ ] 权重校准流程（人工确认制）落地 ≥2 轮
- [ ] 锚定样例从真实数据更新
- [ ] 配图/封面：Illustrator 子 Agent + COS 接入
- [ ] 评估自动发布可行性（分平台调研，单独立项）
- [ ] 盲测记忆有效性（SPEC-006 §7）
- 验收：SPEC-004 M4 条目；系统进入常态运营 + 持续校准节奏

## 里程碑之外的常态事项

- 每周：看 Langfuse 成本报表；review Reflector 周报
- 每里程碑：回顾 spec 与实现的偏差，修订 spec（spec 是活文档）

## 备注：GitHub Organization 创建

github.com 的个人账号**无法通过 API 创建组织**（该 API 仅 GitHub Enterprise Server 提供），需网页操作：
1. 访问 https://github.com/account/organizations/new ，选 Free plan
2. 组织名填 `scholars-ai`（截至 2026-08-06 该名称未被占用）
3. 创建后 Claude Code 侧执行 `gh auth refresh -h github.com -s repo,admin:org` 刷新 token，即可用 gh 建仓、推代码、配 org secrets
