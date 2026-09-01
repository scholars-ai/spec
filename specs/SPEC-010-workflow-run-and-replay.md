# SPEC-010 · 批次工作流、漏斗判定与节点级回放

- 状态：Accepted（实现中；运行时与基础回放已落地，完整验收待补齐）
- 日期：2026-08-24
- 依赖：SPEC-001、SPEC-002、SPEC-003、SPEC-004、SPEC-005、SPEC-007；SPEC-008 仅作 M1 历史参考

## 1. 定位

Scholar AI 的内容生产不是一组互相独立的 Agent 调用，而是一条可以观察、解释和反复优化的批次型工作流：

```text
WorkflowRun
  → source_fetch
  → topic_scout
  → topic_evaluate
  → article_write
  → article_evaluate
  → human_review
```

本 spec 定义一次运行的边界、动态漏斗语义、节点输入输出与逐条判定、历史回溯、节点级重跑和任务详情页。

第一阶段采用一个内置的固定工作流 `content_production`。它提供类似 Dify 的运行画布和调试体验，但不引入通用工作流编辑器，也不让工作流层取代 Core 的领域状态机。

## 1A. 当前实现基线（2026-09-01）

以下能力已经在各子仓库中落地，并作为后续开发的起点：

- Core 已实现统一的自动/手动 `WorkflowRun` 创建、六阶段队列编排、动态漏斗屏障、运行事件、节点运行、快照、产物引用和逐条判定关联。
- 自动调度默认每 12 小时运行一次；调度留痕与运行创建在同一事务中，并通过唯一调度窗口防重。
- Replay 已创建不可变父子运行，支持 `full`、`failed_items`、`selected_items` 和 `evaluate_only` 范围；从 `article_write` 回放复用父运行选题输入，从 `article_evaluate` 回放复用已有文章输入。
- 工作流生成的文章版本固化 `correlation_id`；`article_write` replay 必须生成新版本并以子运行 ID 隔离，不能命中或覆盖父运行文章。
- Agents 已记录业务拒绝与技术失败的区别、reason code、分数/阈值、rubric/权重版本、模型和 trace 信息，并支持 replay 的模型与阈值覆盖。
- Client 已提供 Dify 风格运行画布、任务历史列表、时间线、节点产物/判定检查器和 replay 入口；任务列表从 Core 读取每阶段输入/通过/拒绝/失败、产物、耗时、成本和最近失败节点摘要。
- Workflow 快照已支持按运行隔离的可逆归档/恢复 API；归档保留 payload、checksum、血缘和 `storageRef`，恢复只清除归档标记，并已纳入 VPS E2E 验收。

当前跨仓库验收统一在 VPS 执行：`/root/scholars-ai/scholar-infra/e2e/run.sh`。本机 E2E 的环境差异不作为验收阻塞；每次提交推送后先同步 VPS，再以该脚本的结果为准。

尚未完成的内容不改变本文件的目标语义，主要集中在：完整 PostgreSQL/pgmq 集成与 Docker E2E、配置覆盖的版本化校验、父运行与 replay 的完整对比指标、Client 的 replay 范围/配置/对比交互、快照 retention 后台策略与大型对象存储，以及观测缺失告警。

## 2. 已拍板的设计原则

### 2.1 统一运行入口

自动运行和手动运行都必须创建 `WorkflowRun`，并通过同一条完整链路执行：

- 自动运行：scheduler 默认每 12 小时触发一次；
- 手动运行：用户在 Client 触发，使用同一条工作流；
- 调试运行：从已有运行的某个节点创建 replay run。

原有的 source、scout 等独立 API 可以保留用于诊断和人工操作，但不得与生产工作流的自动调度重复投递同一批业务任务。生产路径的阶段推进由 WorkflowRun 的阶段屏障负责。

### 2.2 动态漏斗，不设业务数量配额

每一阶段传递集合，不预设固定数量。一次运行可能出现：

```text
100 raw items → 30 topics → 18 topics passed → 15 articles → 8 articles passed
```

上面的数字只是一次运行的观测结果，不是配置目标。系统不得因为“没有达到固定数量”而判定运行失败，也不得因为预设数量配额静默丢弃阶段输出。

数量减少只能来自可解释的业务或执行结果：去重、质量门控、跳过、技术失败、重试耗尽或人工拒绝。

### 2.3 质量门控与资源保护分离

TopicJudge 和 ArticleJudge 只负责质量门控：

- 所有达到 `topic_pass_score` 的 topic 都进入写作；
- 所有达到 `article_pass_score` 的 article 都进入人工审核。

并发、超时、token、成本和单批大小属于资源保护，不得伪装成质量判定。达到资源保护上限时，系统必须显式记录 `deferred`、`paused_for_capacity` 或拆分 batch，不能静默截断结果。

### 2.4 领域状态机仍由 Core 负责

- Core 仍是 topic/article 状态机的唯一写入口；
- Agents 仍负责采集、选题、评分、写作和文章评估；
- pgmq 仍负责异步 job；
- Langfuse 仍记录 LLM prompt、结构化输出、token、成本和 trace；
- Workflow 层负责运行编排、跨节点关联、快照、事件、产物血缘和回放。

Workflow 层不得复制一套 topic 或 article 的业务状态机。

## 3. 运行时语义

### 3.1 阶段屏障

每个阶段可以 fan-out 成多个 job，但进入下一阶段前必须经过 fan-in barrier。阶段达到以下任一终态后才可继续：

- 全部 job 成功；
- 部分失败且运行策略允许继续；
- 临时错误重试耗尽；
- 阶段超时；
- 被用户取消。

阶段摘要必须记录输入、成功、拒绝、跳过、失败、重试和输出数量。

### 3.2 标准阶段行为

1. `source_fetch`：拉取所有启用信源，写入去重后的新素材；本次运行只把实际新增或明确纳入 backlog 的素材写入输入快照。
2. `topic_scout`：对本次运行的素材集合生成 0 到多个候选 topic，并记录素材到 topic 的血缘。
3. `topic_evaluate`：逐条评估全部候选 topic；通过、拒绝和执行失败必须分别记录。
4. `article_write`：对全部通过 TopicJudge 的 topic 尝试写作；不因固定文章数量提前截断。
5. `article_evaluate`：逐条评估全部生成文章；通过的文章形成最终业务产物候选。
6. `human_review`：接收全部通过 ArticleJudge 的文章，进入文章审阅列表。该节点不是自动发布节点。

### 3.3 运行状态

WorkflowRun 至少支持：

- `queued`
- `running`
- `waiting_human_review`
- `completed`
- `completed_empty`
- `partial_failed`
- `failed`
- `cancelled`

没有最终产物不等于技术失败。`completed_empty` 必须附带漏斗诊断，说明产出在哪一阶段归零。

## 4. 可解释判定

### 4.1 判定记录

每一个被评估的 raw item、topic 或 article 都必须生成结构化 `WorkflowItemDecision`，至少包含：

- `run_id`、`node_run_id`、`item_id`、`item_type`；
- `decision`：`accepted`、`rejected`、`skipped`、`failed`；
- `reason_code`；
- 人类可读的 `reason`；
- 各维度分数和总分（适用时）；
- 实际使用的阈值、权重和 rubric 版本；
- 输入产物引用和证据引用；
- `agent_run_id`、`trace_id`、模型和 prompt 版本；
- 创建时间。

### 4.2 技术失败与业务拒绝

节点整体可以 `succeeded`，同时包含大量 `rejected` 的业务判定。例如 TopicJudge 成功处理 30 个 topic，其中 20 个因受众不明确被拒绝，节点状态仍为成功，阶段摘要为 `accepted=10, rejected=20`。

节点只有在执行本身未完成、重试耗尽、超时或基础设施故障时才标记为 `failed` 或 `partial_failed`。

### 4.3 零产出诊断

任务详情必须能够直接回答“为什么没有最终文章”：

- 没有新增素材；
- Scout 没生成 topic；
- TopicJudge 全部拒绝；
- 写作任务全部失败或跳过；
- ArticleJudge 全部拒绝；
- 资源保护导致延期；
- 其他明确的业务原因。

同时提供 reason code 的聚合统计，便于判断 rubric、权重、prompt、Agent 或信源质量是否需要调整。

## 5. 快照、血缘与版本

### 5.1 运行快照

每次运行必须固化：

- 工作流定义及版本；
- 启用信源列表；
- 本次使用的 raw item ID 集合；
- 每阶段输入输出集合；
- scheduler 或手动触发参数；
- 节点配置、并发和资源限制；
- rubric、权重、prompt、模型和 Agent 版本。

历史运行只能读，不得因为后续配置变更而改变解释结果。

### 5.2 产物血缘

必须能沿以下路径回溯：

```text
raw_item → topic → topic_evaluation → article → article_evaluation → human_review
```

工作流 API 返回产物摘要和稳定 ID；完整原文、prompt 和模型输出可通过结构化快照或 Langfuse 链接查看。大文本不得只塞进事件 payload。

### 5.3 存储边界

- Postgres 保存运行元数据、结构化判定、事件、产物引用和必要的输入输出快照；
- Langfuse 保存 LLM 级 prompt、输出、token、成本和 trace；
- 超大原文可使用对象存储，但必须在 Postgres 保存不可变引用和校验信息；
- 观测系统不可用不得阻断业务运行，但缺失的观测必须在任务中标记。

## 6. 节点级回放

### 6.1 不修改原运行

任何节点重跑都创建新的 replay `WorkflowRun`，原运行保持不可变。Replay 必须记录：

- `parent_run_id`；
- `replay_from_node`；
- `replay_scope`；
- 输入快照 ID；
- 新的配置快照 ID；
- 创建人、创建时间和重跑原因。

### 6.2 回放模式

第一阶段支持：

- 重跑当前节点及后续节点；
- 仅重跑当前节点用于诊断；
- 只重跑失败项；
- 只重新评估，不重新生成；
- 选择部分 item 重跑。

从 `article_write` 重跑时，默认使用父运行中通过 TopicJudge 的 topic 快照，重新执行 `article_write → article_evaluate → human_review`，不重新采集 RSS，也不重新生成 topic。

从 `article_evaluate` 重跑时，默认复用已有 article，只替换评估配置和判定结果。

### 6.3 配置替换

Replay 默认使用父运行的输入快照，但可以显式替换当前节点的：

- Agent 版本；
- prompt 版本；
- 模型；
- rubric；
- 权重；
- 阈值；
- 并发和资源配置。

“使用当前数据重新解析输入”属于非严格回放，必须由用户显式选择并在任务中标记。

### 6.4 下游失效与新产物

从某节点回放时，该节点及其下游节点的旧结果不得混入新任务。旧结果保留并标记为历史或 superseded，新结果形成新的产物集合并进入新的人工审核队列。

同一 topic 的新写作结果应生成新的 article version 或新的可追踪产物，不得覆盖旧文章内容。

### 6.5 结果对比

父运行与 replay 运行必须可比较：

- 输入集合是否相同；
- 每阶段输出数量变化；
- 通过率变化；
- 分数分布变化；
- 拒绝原因变化；
- token、耗时和成本变化；
- 最终人工审核产物变化。

## 7. Client 任务与画布

### 7.1 任务列表

任务列表是生产和调试入口，至少显示：

- 运行时间和触发方式；
- 工作流版本；
- 运行状态；
- 各阶段输入、通过、拒绝和失败数量；
- 最终产物数；
- 耗时和成本；
- 最近失败节点；
- parent run / replay 标识。

### 7.2 任务详情

点击任务进入 Dify 风格的运行画布，包含：

- 固定工作流图；
- 节点聚合状态和数量；
- 运行时间线；
- 节点输入输出检查器；
- 逐条判定和拒绝原因；
- 产物血缘；
- 配置、prompt、rubric、模型快照；
- Langfuse / OTel 跳转；
- 重跑和对比入口。

画布节点是阶段聚合节点，不为每个 source、topic 或 article 渲染独立节点。节点详情负责展开 item 级执行。

### 7.3 节点操作

用户在节点详情中可以：

- 查看输入；
- 查看输出；
- 查看失败和拒绝原因；
- 查看配置快照；
- 仅重跑当前节点；
- 从当前节点重跑后续链路；
- 只重跑失败项；
- 只重新评估；
- 查看历史 replay 和差异。

## 8. Polyrepo 交付边界

### scholar-shared

- 扩展 WorkflowRun、WorkflowDefinition、WorkflowNodeRun、WorkflowItemDecision、快照和 replay 契约；
- 定义节点状态、判定状态、reason code 和聚合统计 schema；
- 定义工作流详情、回放、对比 API；
- codegen 三端生成物。

### scholar-core

- WorkflowRun 创建、阶段屏障、父子 replay 关系和不可变快照；
- node run、item decision、事件和产物血缘落库；
- 统一自动/手动触发，自动调度默认 12 小时；
- 从任意节点创建 replay run，并校验输入快照和下游失效；
- 仍由 Core 负责 topic/article 状态机写入和任务幂等；
- 提供任务列表、详情、节点输入输出、判定、重跑和对比 API。

### scholar-agents

- 每个 job 回传结构化执行摘要、输入输出引用、逐条判定和 reason code；
- 固化 Agent、prompt、模型、rubric 和权重版本；
- 支持使用父运行输入快照执行 replay；
- 不在 Agent 侧自行修改 WorkflowRun 状态。

### scholar-client

- 任务列表；
- Dify 风格运行画布；
- 节点输入输出和漏斗统计；
- item 级拒绝原因和 trace 链接；
- 节点级重跑、失败项重跑、只重评估；
- 父运行与 replay 运行对比。

### scholar-infra

- 12 小时 scheduler 配置和运行限制；
- 大型输入输出快照的持久化策略；
- 快照保留、归档和恢复策略；
- 观测系统故障时的缺失标记和告警。

## 9. 验收标准

### 功能

- [ ] 自动运行默认每 12 小时创建一次完整 WorkflowRun。
- [ ] 手动运行与自动运行使用同一条完整链路。
- [ ] 每阶段支持 fan-out/fan-in，并能处理动态数量。
- [ ] 任何阶段不因固定业务配额静默丢弃结果。
- [ ] 最终通过 ArticleJudge 的全部文章进入人工审核列表。
- [ ] 任务详情可以从 raw item 回溯到 topic、article、评估和人工审核。

### 可解释性

- [ ] 每个被评估 item 都有通过、拒绝、跳过或失败判定。
- [ ] 每条拒绝记录包含 reason code、可读理由、分数、阈值和版本信息。
- [ ] 技术失败与业务拒绝在节点状态和统计中分离。
- [ ] 零产出任务可以显示漏斗在哪一阶段归零及原因聚合。
- [ ] 任务历史不受后续 prompt、rubric 或配置修改影响。

### 回放

- [ ] 从任意已完成节点创建 replay run，原运行不变。
- [ ] 从 article_write 回放不会重新采集 RSS 或生成 topic。
- [ ] 支持只重跑失败项和只重新评估。
- [ ] Replay 可以替换当前节点的 Agent、prompt、模型、rubric 或阈值。
- [ ] 回放的旧下游结果不会混入新产物集合。
- [ ] 父运行与 replay 运行可以比较通过率、拒绝原因、耗时和成本。

### 工程

- [ ] 重复提交同一 replay 请求不会产生重复业务执行。
- [ ] worker 重启、临时错误和观测系统故障不会破坏运行快照与判定记录。
- [ ] 大型输入输出有持久化引用和校验，不依赖单条事件 payload。
- [ ] 所有跨仓库契约先在 scholar-shared 更新，再生成 Core/Agents/Client 类型。

## 10. 明确不做

- 不做用户自由拖拽的通用 Dify 工作流编辑器；
- 不做任意第三方节点或脚本沙箱；
- 不自动修改 rubric、权重或信源质量；
- 不因成本控制自动删除或静默丢弃阶段产物；
- 不覆盖历史运行、旧文章版本或旧评估结果；
- 不把人工审核自动升级为自动发布。
