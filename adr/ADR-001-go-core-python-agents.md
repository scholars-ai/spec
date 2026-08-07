# ADR-001 · 后端从全栈 TypeScript 改为 Go core + Python agents

- 状态：Accepted
- 日期：2026-08-07

## 背景

SPEC-001 v1 选择全栈 TypeScript（NestJS + Claude Agent SDK TS + Next.js），核心理由是契约复用和用户已有 TS 经验。用户 review 后提出：TS 已经很熟，项目的双重目标之一是学习成长，希望换后端语言。

## 决策

- `scholar-core`（业务核心/API/编排）：**Go**（chi + sqlc + goose + oapi-codegen）
- `scholar-agents`（Agent 运行时）：**Python 3.12**（uv + Pydantic + 自研 runtime）
- `scholar-console`（前端）：维持 Next.js/TS
- 契约层改为语言中立：JSON Schema + OpenAPI 为源，codegen 出 Go/Python/TS 三端代码

## 理由

1. 学习收益：Go 的生产级服务工程 + Python 的后端工程化，都是用户的新知识区；候选中"Python 全包"学习广度不如此方案，"Go 全包"在 LLM/统计分析生态上逆风。
2. 各用其长：core 不碰 LLM，Go 的并发/部署/资源占用优势正中；agents 全是 LLM 与数据处理，Python 是母语（pandas/scipy 做评分校准、Langfuse SDK、trafilatura 清洗）。
3. 采集职责随之从 core 移到 agents（正文提取/embedding 是 Python 强项）。

## 后果与代价

- 需同时维护两套语言工程链（CI、依赖、镜像）——用户接受，视为学习内容的一部分。
- 契约不能再用 TS 包直接 import，改为 codegen 流程（复杂度上升，但换来语言中立）。
- v1 已写的 scholar-shared TS 实现废弃，rubric 定义与契约测试逻辑移植到新结构（rubric → YAML，逻辑 → Python）。
- BullMQ（Node 专属）不可用 → ADR-003。
