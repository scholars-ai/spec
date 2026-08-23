# scholars-ai · Spec 仓库

> AI 内容创作 Agent 军团：选题 → 评分 → 写作 → 评分 → 发布 → 数据回流 → 持续进化的自循环系统。

本仓库是 scholars-ai 组织的**唯一事实来源（Source of Truth）**：所有架构决策、模块设计、评分体系、路线图都以 spec 文档形式沉淀在这里。任何仓库的重大改动，先改 spec，再写代码。

## Spec 索引

| 编号 | 文档 | 内容 |
|---|---|---|
| SPEC-000 | [愿景与范围](specs/SPEC-000-vision-and-scope.md) | 项目目标、非目标、成功标准 |
| SPEC-001 | [总体架构与技术选型](specs/SPEC-001-architecture.md) | 技术栈、Polyrepo 划分、服务间通信 |
| SPEC-002 | [数据模型](specs/SPEC-002-data-model.md) | 核心实体、数据库 Schema、状态机 |
| SPEC-003 | [选题采集](specs/SPEC-003-topic-sourcing.md) | 信源接入、去重、选题候选池 |
| SPEC-004 | [评分体系（Evaluation）](specs/SPEC-004-evaluation-system.md) | 选题评分 + 文章评分 + 数据校准闭环 |
| SPEC-005 | [写作 Agent 军团](specs/SPEC-005-writing-agents.md) | 平台专家 Agent、格式规范、扩展机制 |
| SPEC-006 | [记忆与反馈闭环](specs/SPEC-006-memory-and-feedback-loop.md) | 发帖数据回流、反思机制、经验库 |
| SPEC-007 | [路线图](specs/SPEC-007-roadmap.md) | 里程碑 M0–M4、验收标准 |
| SPEC-008 | [M1 实施计划](specs/SPEC-008-m1-topic-loop.md) | 选题闭环的交付清单、实施顺序、验收 |
| SPEC-009 | [M3 实施计划](specs/SPEC-009-m3-data-memory-loop.md) | 数据快照、表现分、Reflector、记忆注入与验收 |
| SPEC-010 | [批次工作流与节点级回放](specs/SPEC-010-workflow-run-and-replay.md) | 动态漏斗、任务回溯、逐条判定、节点重跑与结果对比 |

## 组织仓库规划（Polyrepo）

| 仓库 | 职责 | 技术栈 |
|---|---|---|
| `spec` | 本仓库，spec / ADR / 路线图 | Markdown |
| `scholar-shared` | 语言中立契约：JSON Schema + OpenAPI + rubric YAML，codegen 三端 | JSON Schema / OpenAPI / codegen |
| `scholar-core` | 后端 API + 流水线状态机 + 调度 | Go（chi + sqlc + goose + oapi-codegen）+ PostgreSQL + pgmq |
| `scholar-agents` | Agent 运行时：采集/选题/评审/写作/反思 + ModelProvider 双协议层 | Python 3.12 + 自研 runtime |
| `scholar-client` | 前端控制台：选题看板、文章审阅、数据面板 | Next.js + shadcn/ui |
| `scholar-infra` | 部署编排：Docker Compose、Nginx、备份脚本 | Docker / Shell |

## 工作方式约定

- **Spec 先行**：新功能先提 spec PR（或修订现有 spec），达成一致后再动代码。
- **ADR**：重大技术决策记录在 `adr/` 目录，编号递增，只增不删（被推翻的标记 Superseded）。

| ADR | 决策 |
|---|---|
| [001](adr/ADR-001-go-core-python-agents.md) | 后端改 Go core + Python agents（弃全栈 TS） |
| [002](adr/ADR-002-self-built-runtime-dual-provider.md) | 自研 agent runtime + 双协议 ModelProvider |
| [003](adr/ADR-003-pgmq-over-redis.md) | 任务队列用 pgmq（事务性入队，无 Redis） |
| [004](adr/ADR-004-self-hosted-postgres.md) | Postgres 自托管于 VPS（弃 Supabase），Langfuse 同实例 |
| [005](adr/ADR-005-ollama-embedding.md) | Embedding 走 SiliconFlow bge-m3（1024d）；本机 Ollama 降为备用 |
| [006](adr/ADR-006-opentelemetry-observability.md) | OTel Collector + Tempo + Prometheus + Grafana；Langfuse 保留 LLM 细节 |
- **契约变更**：`scholar-shared` 的任何 breaking change 必须先在 SPEC-002 中体现。
