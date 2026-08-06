# SPEC-001 · 总体架构与技术选型

- 状态：Draft
- 日期：2026-08-06
- 依赖：SPEC-000

## 1. 技术选型总览

**全栈统一 TypeScript**。理由：前端、后端、Agent SDK（Claude Agent SDK 有一等 TS 支持）共用一种语言，`scholar-shared` 里的类型契约可以在三端直接复用；你已有 Next.js/Vercel/Supabase 的生产经验，学习成本集中在 Agent 工程本身。

| 层 | 选型 | 理由 |
|---|---|---|
| 前端 | **Next.js 15 (App Router) + TypeScript + Tailwind CSS + shadcn/ui + TanStack Query** | 你已有 Vercel 部署经验；shadcn/ui 快速搭建后台类界面；TanStack Query 管理服务端状态 |
| 后端 API | **NestJS 11 + Node.js 22** | 生产级框架：模块化、DI、装饰器路由、内置校验（class-validator）/Swagger，适合长期迭代的多模块项目 |
| 数据库 | **PostgreSQL（Supabase 托管）+ pgvector** | 你已在用 Supabase；pgvector 支撑语义去重和记忆检索，免去独立向量库 |
| ORM | **Drizzle ORM** | SQL-first、类型安全、迁移文件可审查，对 pgvector 支持好 |
| 任务队列/调度 | **BullMQ + Redis** | 流水线各步骤异步化、失败重试、定时采集（repeatable jobs）都靠它 |
| Agent 运行时 | **Claude Agent SDK (TypeScript)** | 多 Agent 编排、子 Agent、工具调用、MCP 生态；这是本项目"练兵"的核心 |
| LLM | **Claude API**（写作/评审用 `claude-sonnet-5`，反思/校准等重任务用 `claude-opus-5`） | 按任务分级用模型，控制成本 |
| Agent 可观测 | **Langfuse（自托管在 VPS）** | trace 每次 Agent 运行的 prompt/输出/成本，评分数据也挂在 trace 上，是 Evaluation 体系的地基 |
| 采集 | **RSSHub（VPS 已有实例，复用）+ rss-parser + Playwright（少量需要渲染的源）** | 你 VPS 上已跑着 RSSHub，直接复用 |
| 对象存储 | 预留：腾讯云 COS（配图/封面阶段再接） | 与现有 aicave 基建一致 |
| 部署 | 前端 → **Vercel**；后端/Agent/Redis/Langfuse → **VPS Docker Compose**（Nginx 反代，复用现有 aicave.cn 的 nginx） | 与你现有基建一致，`scholar-infra` 统一管理 |
| CI/CD | **GitHub Actions**：lint + test + build；后端镜像推 GHCR，VPS 拉取部署 | 组织级复用 workflow |

### 明确不选的方案（记录理由，避免反复）

- **Python + LangGraph**：生态也好，但会分裂技术栈；Claude Agent SDK 的 TS 版能力足够，且与你的日常工具链（Claude Code）同源。
- **Monorepo（Turborepo）**：你明确要求 Polyrepo 练习多仓协作；代价是共享代码要走包发布，用 `scholar-shared` + GitHub Packages 解决。
- **微服务化拆分后端**：单用户系统，`scholar-core` 一个服务足够；Agent 运行时单独成服务只是因为它的伸缩与部署节奏（prompt 迭代频繁）和 API 不同。

## 2. 系统架构

```
                        ┌─────────────────────────────┐
                        │  scholar-console (Next.js)  │  Vercel
                        │  选题看板/文章审阅/数据面板     │
                        └──────────────┬──────────────┘
                                       │ REST (OpenAPI)
┌──────────────────────────────────────▼──────────────────────────────────┐
│                        scholar-core (NestJS)          VPS Docker        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────────────┐  │
│  │ Sourcing   │ │ Pipeline   │ │ Metrics    │ │ REST API + Swagger  │  │
│  │ Module     │ │ Orchestr.  │ │ Module     │ │ (console 的 BFF)    │  │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────────────────────┘  │
└────────┼──────────────┼──────────────┼─────────────────────────────────┘
         │              │ BullMQ jobs  │
         ▼              ▼              ▼
   ┌──────────┐  ┌─────────────────────────────┐   ┌───────────────────┐
   │ RSSHub   │  │  scholar-agents (Agent SDK) │──▶│ Langfuse (VPS)    │
   │ (已有)    │  │  TopicScout / TopicJudge    │   │ trace + eval 数据  │
   └──────────┘  │  Writers×N / ArticleJudge   │   └───────────────────┘
                 │  Reflector                  │
                 └──────────────┬──────────────┘
                                ▼
                 ┌─────────────────────────────┐
                 │ PostgreSQL (Supabase)       │
                 │ + pgvector    + Redis(VPS)  │
                 └─────────────────────────────┘
```

## 3. 服务间通信

- **console → core**：REST + OpenAPI。core 用 Swagger 自动生成 OpenAPI spec，console 用 `openapi-typescript` 生成类型化 client，契约漂移在 CI 里报错。
- **core → agents**：**BullMQ 队列**（Redis 作为 broker）。core 只负责编排（往队列扔 job：`topic.evaluate`、`article.write`、`article.evaluate`、`memory.reflect`），agents 是纯 worker，消费 job、跑 Agent、结果写回 DB 并回报 job 状态。好处：写作任务动辄几分钟，天然异步；失败重试、并发控制、优先级全由 BullMQ 管。
- **共享契约**：job payload、评分 Schema、实体类型全部定义在 `scholar-shared`（Zod schema 单一来源，同时导出 TS 类型），core 和 agents 都从包引入，禁止各自手写。

## 4. Polyrepo 划分与依赖方向

```
spec  (无代码依赖，所有仓库的上游)
scholar-shared  ◀── scholar-core
                ◀── scholar-agents
                ◀── scholar-console (仅类型)
scholar-infra   (引用各仓库的镜像/产物，不被依赖)
```

规则：
1. 依赖只能指向 `scholar-shared`，仓库之间不允许互相 import。
2. `scholar-shared` 用 changesets 管版本，发布到 GitHub Packages；breaking change 必须升 major 并同步修订 SPEC-002。
3. 每个仓库自带 CI（lint/test/build）；`scholar-infra` 存放 compose 文件和部署脚本，部署动作在 VPS 上执行 `deploy.sh <service> <version>`。

## 5. 环境与配置

- 环境：`local`（docker compose 起 Postgres/Redis）→ `prod`（VPS + Supabase）。第一阶段不设独立 staging，用 feature flag + 人工卡点控制风险。
- 密钥：`.env` 不入库；VPS 上密钥统一放 `scholar-infra` 的部署工作区（参照 operation-content-platform 的做法）；GitHub Actions 用 org-level secrets。
- LLM 成本护栏：agents 服务内置每日 token 预算（env 配置），超限熔断并告警。

## 6. 可观测性

- **Agent 层**：Langfuse trace（每个 job 一条 trace，子 Agent 是嵌套 span），评分结果作为 Langfuse score 挂 trace。
- **服务层**：NestJS 结构化日志（pino）+ 日志轮转（VPS 已有 logrotate 实践）。
- **业务层**：console 的数据面板直接查 Postgres 聚合。

## 7. 开放问题

- [ ] `scholar-shared` 初期是否可以先用 git submodule/直接 npm install github: 引用，等稳定后再上 GitHub Packages？（M0 决定）
- [ ] Langfuse 自托管 vs 云免费版：VPS 磁盘 66% 使用率，需评估（M1 决定）
- [ ] 公众号排版（md → 微信富文本）用现成工具（如 wechat-mdeditor 类方案）还是自研渲染层？（M2 决定）
