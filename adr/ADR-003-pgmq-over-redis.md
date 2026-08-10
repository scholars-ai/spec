# ADR-003 · 任务队列用 pgmq 取代 BullMQ/Redis

- 状态：Accepted
- 日期：2026-08-07

## 背景

v1 选 BullMQ + Redis 做 core → agents 的任务队列。ADR-001 语言切换后 BullMQ（Node 专属）不可用，需要 Go 生产者 + Python 消费者都支持的队列。

## 决策

使用 **pgmq**（Postgres 扩展，亦是 Supabase Queues 的底层实现）作为任务队列；core 内置 cron（robfig/cron）负责定时投递；不引入 Redis。

## 理由

1. **跨语言天然支持**：pgmq 本质是 SQL 函数调用，Go（sqlc/pgx）和 Python（psycopg）零障碍。
2. **事务性入队**：业务状态变更与 job 投递在同一个数据库事务中提交（如"topic → in_writing"与 `article.write` 入队原子化），Redis 队列做不到这种正确性，需要 outbox 模式弥补。
3. **少一个运维组件**：VPS 磁盘紧张、单人运维，每个中间件都是持续成本；pgmq 是 Postgres 扩展，随库而来（自托管镜像已内置，ADR-004）。
4. 吞吐量层面：单人内容系统的 job 量（每天几十上百）离 Postgres 队列的性能边界差几个数量级。

## 后果与代价

- 无 BullMQ 自带的 dashboard——client 的运维页直接查 pgmq 表补齐（本来也要做 agent_runs 展示）。
- 消费语义为 visibility timeout 模型，重试/死信需按 pgmq 惯例自建薄封装（约几十行 SQL/代码）。
- 若未来吞吐暴涨（多租户化），迁移到专业队列需要重写队列薄层——接口已隔离，风险可控。
