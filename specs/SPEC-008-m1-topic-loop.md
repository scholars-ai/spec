# SPEC-008 · M1 实施计划：选题闭环

- 状态：Accepted
- 日期：2026-08-08
- 依赖：SPEC-003（采集）、SPEC-004（评分）、SPEC-007 §M1
- 前置决策：ADR-004（DB 自托管）、ADR-005（Ollama embedding）

M0 已交付五仓骨架。本 spec 把 SPEC-007 的 M1 里程碑落成可执行清单。

## 1. 目标

打通五步流程的前两步——**采集 → 聚合选题 → 评分 → 人工确认**，全链路运行在 VPS 上。
产出物的判定标准：**每天打开看板，就有带评分和理由的可写选题**。

## 2. 已拍板的技术决策

| 项 | 决定 | 依据 |
|---|---|---|
| 数据库 | VPS 自托管 Postgres 17（业务库 + Langfuse 库同实例），每日 pg_dump 推 COS | ADR-004 |
| Embedding | VPS Ollama + `qwen3-embedding:4b`，MRL 截断前 1024 维并归一化 | ADR-005 |
| 可观测 | Langfuse 自托管，trace 保留 30 天 | ADR-004 |

## 3. 数据流（M1 结束时的运行态）

```
core cron（每小时）──▶ 对每个 enabled source 投递 source_fetch（事务性入队）
                            │
agents SourcingHandler ◀────┘
  feedparser 拉取 → trafilatura 清洗 → content_hash 精确去重
  → Ollama embedding（1024d）→ 语义去重（近 14 天 cos > 0.92 合并）
  → raw_items(status=new)
                            │
core cron（每日 2 次）──▶ topic_scout
                            │
agents TopicScout（自研 loop 首次实战）
  聚类素材 → 每簇提 1–3 个选题角度 → 与既有 topics 向量查重
  → topics(status=candidate)
                            │
core harvester ──▶ 对新 candidate 投递 topic_evaluate
                            │
agents TopicJudge（structured output 首次实战）
  读 rubrics/topic.v1.yaml + DB weight_sets → 维度分 + 理由
  → topic_evaluations + agent_runs（挂 Langfuse trace）
                            │
core harvester ──▶ 状态机 candidate→scored；≥75 推荐 / 60–75 备选 / <60 自动 rejected
                            │
scholar-client 选题看板 ──▶ 人工 approve / reject（scored → approved / rejected）
```

## 4. 交付清单

### scholar-shared（契约先行）
- [ ] `openapi/core.yaml` 扩展：sources CRUD、手动投喂 URL、topic approve/reject
- [ ] 新增 TopicJudge 结构化输出 schema（维度分 + rationale，供 `complete_structured` 校验）
- [ ] codegen 三端并提交生成物

### scholar-core（Go）
- [ ] sources CRUD API + sqlc 查询
- [ ] 手动投喂 API：POST URL → 入 raw_items 流程
- [ ] cron 调度（robfig/cron）：每小时 source_fetch、每日 2 次 topic_scout；调度留痕
- [ ] harvester：轮询结果表推进状态机（唯一写入口原则不破）
- [ ] topic approve/reject API（非法流转返回 409）
- [ ] CI：Go 队列常量与 `schemas/queues.json` 一致性校验（M0 遗留口子）

### scholar-agents（Python）
- [ ] `embed()` 封装：Ollama 调用 + 2560→1024 MRL 截断 + L2 归一化（单元测试覆盖维度与范数）
- [ ] SourcingHandler：拉取/清洗/双重去重；**单源失败隔离**（一个源失败不影响其他源），连续失败计数
- [ ] TopicScout：素材聚类 + 角度生成 + 查重；insights 检索留接口（M3 才有数据）
- [ ] TopicJudge：rubric YAML + 生效权重 → 结构化评分；一票否决逻辑（本 rubric 暂无 veto 维度，但代码路径就位）
- [ ] Langfuse 接入：每 job 一条 trace，评分作为 score 挂载，trace_id 回写 agent_runs
- [ ] 每日 token 预算熔断（env 配置上限，超限停止消费 + 告警）

### scholar-client（Next.js）
- [ ] 引入 shadcn/ui + TanStack Query
- [ ] 选题看板：候选列表（分数排序）、维度分可视化、评分理由展开、approve/reject
- [ ] 信源管理页：增删改、启停、最近抓取状态与连续失败告警
- [ ] 手动投喂入口（高频动作，一键贴 URL）

### scholar-infra + 部署
- [ ] `compose.prod.yaml` 增加 postgres 服务（自建镜像）+ named volume；langfuse 指向同实例独立 database
- [ ] 备份：每日 `pg_dump` → 加密 → 推腾讯云 COS，本地保留 7 份；**恢复流程实测并记录**
- [ ] 磁盘监控告警（> 85%）
- [ ] GHCR 镜像构建 CI（core/agents）+ deploy.sh 首次真实部署
- [ ] nginx 反代（复用现有实例）：client 访问 core API

## 5. 实施顺序

每步结束都可独立验证，避免"全写完再调试"：

1. **infra 先行**：VPS 起自托管 Postgres + goose 迁移 + 备份/监控就位（数据地基）
2. **契约扩展**：shared 改 OpenAPI + 评分 schema → codegen
3. **core CRUD**：sources 管理 + 手动投喂（先把 API 链路走通）
4. **agents embed + SourcingHandler**：先接 2–3 个源本地验证，再扩到 ≥8
5. **core cron**：采集自动化
6. **agents TopicScout → TopicJudge**：**同时接 Langfuse**（调 prompt 全靠看 trace，这步工作量最大）
7. **core harvester**：闭环在后端成立
8. **client 看板 + 信源管理**：有真数据后再做界面
9. **VPS 部署 + 试运行一周** → 逐条打勾 §6

## 6. 验收标准

**功能**
- [ ] ≥8 个信源稳定采集；单源失败隔离，连续失败告警
- [ ] 每天自动产出 ≥10 条选题候选，语义重复率 < 10%
- [ ] 手动投喂 URL → 出现在候选池 < 2 分钟
- [ ] 每条候选具备总分、6 维度分、人可读理由
- [ ] approve/reject 生效，非法状态流转被拒（409）

**质量（M1 最重要的软验收）**
- [ ] 人工抽查 20 条评分，**理由认可率 ≥ 80%**；不认可的能归因为 rubric 定义问题或模型问题
- [ ] 每次评分记录 rubric_version 与权重版本，可回放

**工程**
- [ ] 全链路 Langfuse trace 可查（prompt / 输出 / token / 成本）
- [ ] token 预算熔断手动压测触发一次
- [ ] agents 崩溃重启后 job 不丢（pgmq visibility timeout 实测）
- [ ] 数据库备份产出 + **恢复演练成功**
- [ ] CI：队列名一致性校验生效；GHCR 镜像可部署

## 7. 明确不做（防止范围膨胀）

- 不写文章（Writer 属 M2）
- 不做数据回流与记忆（M3）；TopicScout 的 insights 检索只留接口
- 不做评分权重校准（M4，尚无数据）
- X/Twitter 源不稳则暂时放弃该源，不为其自研爬虫

## 8. 待运营输入

- 首批 ≥8 个信源清单（news / research / tutorial / kol 四类），尤其 kol 类的 X 账号与 Newsletter——该类源出爆款选题密度最高。
