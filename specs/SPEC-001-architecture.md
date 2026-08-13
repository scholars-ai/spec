# SPEC-001 · 总体架构与技术选型

- 状态：Accepted（v2，2026-08-07 修订：后端改 Go + Python，见 ADR-001/002/003）
- 日期：2026-08-06 / 2026-08-07
- 依赖：SPEC-000

## 1. 技术选型总览

**三层分离：前端（TS）/ 业务核心（Go）/ Agent 运行时（Python）**，语言按各层的生态主场选择，同时服务于"通过项目学习"的目标（用户已熟 TS，Go/Python 后端工程是新知识）。

| 层 | 选型 | 理由 |
|---|---|---|
| 前端 client | **Next.js 15 (App Router) + TypeScript + Tailwind CSS + shadcn/ui + TanStack Query** | 内部控制台形态；已有 Vercel 生产经验；组件代码自持有 |
| 业务核心 core | **Go 1.23+：chi（路由）+ sqlc（类型安全 SQL）+ goose（迁移）+ oapi-codegen（OpenAPI-first）** | 生产级服务工程的学习主场；单二进制部署、低资源占用（VPS 友好）；全部是社区主流无魔法的工具 |
| Agent 运行时 agents | **Python 3.12：uv（包管理）+ Pydantic + 自研轻量 agent runtime** | AI 生态母语：评分校准的统计分析（pandas/scipy）、Langfuse Python SDK 一等支持；自研 runtime 学习收益最大（ADR-002） |
| 模型接入 | **Provider 抽象层：AnthropicProvider + OpenAICompatProvider 双协议** | OpenAI 协议已是行业通用协议（DeepSeek/Qwen/Kimi/GLM/OpenRouter/Ollama 全兼容），Anthropic 协议有独特能力（caching/thinking）；按任务路由模型，改配置即切换（ADR-002） |
| 数据库 | **PostgreSQL 17 自托管于 VPS + pgvector**（业务库与 Langfuse 库同实例，ADR-004） | 少一个外部依赖；容量不受免费额度限制；与 core/agents 同机低延迟；数据主权在己。代价是备份/监控自负（ADR-004 硬性要求） |
| 任务队列 | **pgmq**（Postgres 扩展，镜像内置） | 跨语言（本质是 SQL）；**入队与业务状态变更同事务**，保证编排正确性；VPS 无需 Redis（ADR-003） |
| 定时调度 | core 内置 cron（robfig/cron）投递 pgmq job | 调度逻辑与编排同处一地，可观测 |
| 契约 | **JSON Schema + OpenAPI 为源，codegen 三端**（详见 §3） | 语言中立，三种语言共享一份契约 |
| Agent 可观测 | **Langfuse 自托管于 VPS** | trace 每次运行的 prompt/成本/评分；trace 保留 30 天防磁盘膨胀（ADR-004） |
| 采集 | **RSSHub（VPS 已有实例，复用）+ Python 侧 feedparser/trafilatura** | 采集与清洗划入 agents 侧的 Python 生态（见 §2 分工） |
| 对象存储 | 预留：腾讯云 COS（配图/封面阶段再接） | 与现有 aicave 基建一致 |
| 部署 | client → **Vercel**；core/agents/Langfuse → **VPS Docker Compose**（复用现有 nginx） | 长任务与常驻 worker 不适合 serverless |
| CI/CD | **GitHub Actions**：各仓库 lint + test + build；镜像推 GHCR | 已建立安全基线（pinned SHA、最小权限、gitleaks、push protection） |

### 明确不选的方案（记录理由，避免反复）

- **全栈 TypeScript（v1 方案）**：用户已熟 TS，学习收益低。被 v2 取代，决策记录见 ADR-001。
- **NestJS/BullMQ/Drizzle（v1 配套）**：随语言切换一并废弃；BullMQ 为 Node 专属，跨语言不可用。
- **Claude Agent SDK / LangGraph**：本项目流程是"确定性 pipeline + 节点上的结构化 LLM 调用"，控制权应在自己手里；自研 runtime 学习收益最大。接口留接缝，未来单点可换（ADR-002）。
- **LiteLLM 直接当 Provider 层**：先自研以吃透双协议差异；接口设计兼容，撑不住时可无缝换入。
- **Monorepo**：维持 Polyrepo（用户要求，练多仓协作）。
- **Java/Spring、Rust**：学习方向与 AI 工程不重叠 / 迭代速度负优化。

## 2. 系统架构

```
                        ┌─────────────────────────────┐
                        │  scholar-client (Next.js)  │  Vercel
                        │  选题看板/文章审阅/数据面板     │
                        └──────────────┬──────────────┘
                                       │ REST (OpenAPI，类型化 client 由 codegen 生成)
┌──────────────────────────────────────▼──────────────────────────────────┐
│                     scholar-core (Go)                 VPS Docker        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌─────────────────────┐  │
│  │ Pipeline   │ │ Metrics    │ │ Cron       │ │ REST API            │  │
│  │ Orchestr.  │ │ Module     │ │ Scheduler  │ │ (oapi-codegen)      │  │
│  │ (状态机唯一 │ └────────────┘ └────────────┘ └─────────────────────┘  │
│  │  写入口)    │                                                        │
│  └─────┬──────┘                                                        │
└────────┼───────────────────────────────────────────────────────────────┘
         │ pgmq（与状态变更同事务入队）
         ▼
┌─────────────────────────────────────────────┐    ┌───────────────────┐
│  scholar-agents (Python)      VPS Docker    │───▶│ Langfuse (VPS)    │
│  ┌───────────────────────────────────────┐  │    └───────────────────┘
│  │ 自研 agent runtime                     │  │
│  │  Sourcing / TopicScout / TopicJudge   │  │    ┌───────────────────┐
│  │  Writers×N / ArticleJudge / Reflector │  │◀───│ RSSHub (VPS 已有)  │
│  ├───────────────────────────────────────┤  │    └───────────────────┘
│  │ ModelProvider 层                       │  │
│  │  AnthropicProvider │ OpenAICompatProv. │  │
│  └───────────────────────────────────────┘  │
└──────────────────────┬──────────────────────┘
                       ▼
        ┌─────────────────────────────┐
        │ PostgreSQL 17 (VPS 自托管)   │
        │  业务表 + pgvector + pgmq    │
        └─────────────────────────────┘
```

分工原则：
- **core（Go）**：API、Pipeline 状态机（唯一写入口）、调度、指标聚合。不碰 LLM。
- **agents（Python）**：一切与 LLM 和内容处理相关的工作——采集清洗（feedparser/trafilatura/embedding）、六类 Agent、Provider 层、Langfuse 上报。纯 worker：消费 pgmq job → 干活 → 写结果表 → core 收割结果推进状态机。
- 采集划入 agents 侧（v1 曾划给 core）：正文提取/清洗/embedding 全是 Python 生态强项，且它们与 TopicScout 同处一个数据流。

## 3. 契约层（scholar-shared）

语言中立，**schema 为源，代码皆生成物**：

```
scholar-shared/
  schemas/            # JSON Schema：实体、枚举、job payload、rubric 结构（单一事实来源）
  openapi/core.yaml   # core 的 REST API 契约
  gen/
    go/               # quicktype/oapi-codegen 产物 → core 引用（go mod）
    python/           # datamodel-code-generator 产物（Pydantic v2）→ agents 引用
    ts/               # openapi-typescript + json-schema-to-typescript 产物 → client 引用
  rubrics/            # rubric 定义（YAML，版本化，如 topic.v1.yaml）——数据而非代码，三语言可读
```

- 生成物提交入库，CI 校验"重新生成后无 diff"，防手改漂移。
- rubric 从 v1 的"TS 代码"改为 **YAML 数据文件**（结构由 `schemas/rubric.schema.json` 约束）：Python 评分逻辑读取，Go/TS 只读展示。加权计算、一票否决等**逻辑**在 agents 侧实现并测试。
- 队列名与 payload schema 的映射表也在 schemas 中定义，Go 入队与 Python 消费共用。

## 4. Polyrepo 划分与依赖方向

```
spec  (无代码依赖，所有仓库的上游)
scholar-shared  ◀── scholar-core     (Go：引用 gen/go)
                ◀── scholar-agents   (Python：引用 gen/python)
                ◀── scholar-client  (TS：引用 gen/ts)
scholar-infra   (引用各仓库镜像/产物，不被依赖)
```

规则不变：依赖只指向 shared；仓库间不互相 import；breaking change 先改 spec 再改 schema。

## 5. 环境与配置

- 环境：`local`（docker compose 起 Postgres——含 pgmq/pgvector 扩展——即可全栈本地跑）→ `prod`（VPS，同一镜像自托管 + 每日 pg_dump 备份推 COS，ADR-004）。
- 密钥纪律（硬性）：见各仓库 .gitignore + gitleaks CI + GitHub push protection 三层防线；密钥只存 VPS 部署工作区 / GitHub Actions secrets / 本地 .env。
- 模型路由与用量：agents 的 `model_routing.yaml` 按任务配置 provider/model；每个 LLM 调用通过 Langfuse 和 `agent_runs` 记录 token/成本，供应商 API key 负责额度限制。

## 6. 可观测性

- **Agent 层**：Langfuse trace（每个 job 一条 trace，子步骤嵌套 span），评分作为 score 挂 trace。
- **服务层**：core 用 slog 结构化日志；agents 用 structlog；VPS logrotate（已有实践）。
- **业务层**：client 数据面板直查 Postgres 聚合（core 提供 API）。

## 7. 开放问题

- [ ] gen/go 以 go module 方式被 core 引用：直接 `require github.com/scholars-ai/scholar-shared/gen/go` 还是 go.work？（M0 实操定）
- [x] Langfuse：自托管于 VPS，与业务库共用 Postgres 实例，trace 保留 30 天（ADR-004）
- [ ] 公众号排版（md → 微信富文本）方案（M2 决定）
- [x] embedding：VPS Ollama + qwen3-embedding:4b，MRL 截断 1024 维（ADR-005）
