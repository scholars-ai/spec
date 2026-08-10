# ADR-005 · Embedding 用 VPS 自托管 Ollama（qwen3-embedding:4b，MRL 截断至 1024 维）

- 状态：Accepted
- 日期：2026-08-08
- 相关：SPEC-003（去重）、SPEC-006（记忆检索）

## 背景

去重（raw_items/topics）与记忆检索（insights）都依赖 embedding。SPEC-001 遗留开放问题：自托管 vs API。
VPS 现状核实：Ollama 服务 active，已装 `qwen3-embedding:4b`（2.5GB，7 周前拉取）。

## 决策

使用 VPS 上已有的 Ollama + `qwen3-embedding:4b`，通过 **MRL 截断取前 1024 维并 L2 归一化**，匹配库表 `vector(1024)`。

## 理由

1. 零新增成本、零新增部署：服务与模型都已就绪。
2. 数据不出本机：素材原文不发往第三方。
3. 与 ADR-004 的自托管方向一致，链路全在 VPS 内网。
4. 模型质量足够：Qwen3-Embedding 系列在中英文检索任务上表现优秀，且原生支持中文（我们的素材中英混合）。

## 维度选择说明

- 该模型原生输出 **2560 维**，超过 pgvector HNSW 索引上限（2000 维）。
- Qwen3-Embedding 基于 MRL（Matryoshka Representation Learning）训练，**截断前 N 维再归一化是官方支持的降维方式**，质量损失很小。
- 取 1024 维：与既有迁移一致，索引可建，存储与检索开销更低。
- 截断与归一化在 agents 的 embed 封装内统一处理，调用方不感知。

## 后果与代价

- Ollama 与业务服务共享 VPS 资源（CPU/内存）；批量 embedding 需限并发，避免影响 core/agents。
- 更换 embedding 模型将使既有向量失效：**必须走"新列 + 回填 + 切换"流程**，不可原地替换（M1 起在 scholar-agents README 记录当前模型与维度）。
- 若未来需要更高检索质量，接口已隔离（embed 封装单点），可换 API 方案。
