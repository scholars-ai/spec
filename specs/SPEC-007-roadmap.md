# SPEC-007 · 路线图

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-000 ~ 006

节奏假设：业余时间迭代，每个里程碑 2–4 周弹性。**每个里程碑结束时系统都处于"可用"状态**，不搞长期不可用的大重构。

## M0 · 地基（已完成）

- [x] 设计 spec（本仓库）
- [x] 创建 GitHub Organization `scholars-ai`
- [x] 建 6 个仓库：spec / scholar-shared / scholar-core / scholar-agents / scholar-client / scholar-infra，推入初始骨架
- [x] scholar-shared：JSON Schema 首版（实体 + job payload + rubric 结构）+ rubric YAML + codegen 管线（Go/Python/TS）
- [x] scholar-core：Go 骨架（chi + sqlc + oapi-codegen）+ goose 迁移（SPEC-002 全部表）
- [x] scholar-infra：docker-compose（core / agents / langfuse；本地 Postgres 含 pgmq+pgvector 扩展）+ 本地开发 compose
- [x] CI：core（go vet/test/build）、agents（ruff/mypy/pytest）、client（lint/typecheck）、shared（codegen 无 diff 校验）跑通
- 验收：本地 `docker compose up` 起全套，core 健康检查通过，空管道可插入一条手动 topic

## M1 · 选题闭环（已完成，人工确认）

- [x] Sourcing 模块：≥8 信源接入（复用 VPS RSSHub）+ 手动投喂 URL
- [x] 去重（hash + embedding）、TopicScout 聚合选题
- [x] 选题评分 rubric topic@v2 + TopicJudge Agent + Langfuse 接入
- [x] client v1：选题看板（候选/评分/理由/确认与否决）、信源管理
- [x] 部署上 VPS，定时任务跑起来
- 验收：SPEC-003 §5 + SPEC-004 验收标准 M1 条目

## M1.1 · 可靠性与全局可观测性（实现完成，待部署）

- [x] Provider 结构化错误分类；普通 429 可重试，明确 quota/balance code 才永久失败
- [x] whole-job deadline、visibility lease、Retry-After/指数退避、成功回执幂等和死信审计
- [x] 状态流转审计、手动 reason/note、逐维度理由、source fetch 执行记录和 source 软归档
- [x] queue-specific Worker 进程，消除慢队列阻塞其他队列
- [x] OTel Collector + Tempo + Prometheus + Grafana；Langfuse 保留 LLM 细节
- [x] shared `_meta`、W3C Trace Context 和 correlation/job/parent job 传播
- [x] client 与 shared 的跨仓库 TS 生成物漂移 CI
- [x] scholar-infra 中真实 PostgreSQL/pgmq + Fake AI 的跨仓库 E2E
- [x] 本地 E2E 运行验收通过
- [ ] 评估 VPS 资源后部署观测组件并进行 7/30 天保留期调优

正确性优先级固定为：`deadline + visibility lease + idempotency → concurrency`。生产部署不属于本次本地实现授权范围。

## M2 · 写作军团

- [ ] Platform Profile × 3（小红书/知乎/公众号）+ WriterOrchestrator 流水线
- [ ] 文章评分 rubric × 3 + ArticleJudge + 回炉机制
- [ ] client：文章审阅页（diff 编辑、终审通过/拒绝、复制导出 md）
- [ ] publications 手动登记（发了哪、链接、最终稿 diff）
- 验收：SPEC-005 §5 条目；开始真实运营发帖

## M3 · 数据回流与记忆 v1

- [ ] 指标录入（表单 + CSV 导入）+ 24h/72h/7d 快照提醒
- [ ] 表现分 P 计算 + 数据面板（client）
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
