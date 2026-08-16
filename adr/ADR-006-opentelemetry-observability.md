# ADR-006 · OpenTelemetry 全局可观测性，Langfuse 保留 LLM 细节

- 状态：Accepted
- 日期：2026-08-16
- 决策者：Scholars AI

## 背景

Langfuse 能解释一次 LLM 调用使用了什么 prompt、模型、token 和评分，但不能完整回答 Core API、Scheduler、pgmq 排队、Worker、数据库和 Harvester 之间的端到端耗时与失败位置。只依赖结构化日志也难以按一个任务下钻跨服务调用链。

系统需要同时具备两种视角：

- Prometheus 的低基数整体健康与告警；
- Tempo 的单任务 Trace 瀑布图，并从 LLM Span 跳转到 Langfuse。

## 决策

采用以下链路：

```text
scholar-core / scholar-agents
        │ OTLP traces + metrics
        ▼
OpenTelemetry Collector
        ├── traces ──▶ Tempo
        └── metrics ─▶ Prometheus
                         │
Tempo ───────────────────┤
                         ▼
                      Grafana

LLM Span -- langfuse.trace_id --> Langfuse
```

### 职责边界

- OpenTelemetry：跨服务上下文传播、Span 和 Metrics 埋点；
- Collector：统一接收、批量、限流、重试和转发；
- Tempo：保存任务级 Trace，不保存 prompt、输出或正文；
- Prometheus：保存低基数聚合指标；
- Grafana：Dashboard、Trace 查询和告警；
- Langfuse：保存 LLM prompt、输出、token、模型、成本和评分。

### 跨队列上下文

所有 job payload 可选携带 `_meta`：

```text
jobId / correlationId / parentJobId
traceparent / tracestate / baggage
enqueuedAt / triggerType
```

旧消息没有 `_meta` 时，Consumer 使用 `queue + msg_id` 生成稳定 job ID，并创建新的 root trace，保证向后兼容。

### 数据与基数纪律

- `job.id`、`correlation.id`、实体 ID 和 `langfuse.trace_id` 只进入 Trace attributes，不成为 Prometheus label；
- Tempo 禁止记录 prompt、LLM 完整输出、正文、URL、API key、数据库 DSN 和隐私数据；
- Metrics label 只使用 queue、job type、status、provider、model、状态边和错误类型等有限集合；
- 观测初始化、查询或导出失败不得让 API 或 job 失败。

## 后果

收益：

- 可以从一次 API/调度触发沿 correlation ID 追踪到全部下游 job；
- 可以区分排队、抓取、Embedding、LLM、数据库和 Harvester 耗时；
- 可用统一 Dashboard 观察积压、失败、重试、死信和服务延迟；
- LLM 细节继续只存一份，避免 Tempo 与 Langfuse 重复和泄漏。

代价：

- 增加 Collector、Tempo、Prometheus、Grafana 四个常驻组件及存储开销；
- 埋点字段、采样、保留期和 Dashboard 需要随真实流量持续维护；
- Polyrepo 的 Trace Context 契约必须兼容升级。

## 被否决的方案

- 只用 Langfuse：无法覆盖 Core、队列、数据库和状态机；
- 只用日志：聚合健康和单任务瀑布图都较弱；
- 把 prompt/output 复制进 Tempo：重复存储且扩大敏感数据面；
- 第一版同时部署 Loki：当前日志量和排障需求不足以抵消额外运维成本。
