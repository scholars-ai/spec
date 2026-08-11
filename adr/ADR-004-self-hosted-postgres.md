# ADR-004 · 数据库自托管于 VPS，放弃 Supabase 托管

- 状态：Accepted
- 日期：2026-08-08
- 相关：ADR-003（pgmq）、SPEC-001

## 背景

SPEC-001 v1/v2 选 Supabase 托管 Postgres，理由是零运维 + 用户已有使用经验。M1 开工前拍板可观测方案时，用户选择 **Langfuse 自托管**，这改变了前提：

- Langfuse 自身需要一个 Postgres 存 trace（每次 LLM 调用的完整 prompt/输出，增量大）；
- Supabase 免费版 500MB 容量，装不下 trace 数据；
- 既然 VPS 上必须为 Langfuse 起 Postgres，再额外依赖一个云端库就是重复组件。

同时 M0 已产出可用镜像 `scholar-infra/postgres/Dockerfile`（pgvector 0.8.6 + pgmq 1.12，pg17），原为本地开发准备，可直接用于生产。

## 决策

**业务库与 Langfuse 库共用 VPS 上一个自托管 Postgres 实例**（同实例、不同 database/schema），不再使用 Supabase。

配套硬性要求（缺一不可，否则自托管风险不可接受）：

1. **每日备份**：`pg_dump` 定时任务，产物加密，本地保留最近 7 份；恢复流程需实测一次并记录在 scholar-infra README。
2. **磁盘监控告警**：VPS 磁盘使用率 > 85% 告警。
3. **Langfuse trace 保留期设上限**（初始 30 天），防止 trace 无限增长吃满磁盘。
4. 数据卷用 named volume 并纳入备份范围；compose 重建不得丢数据。

### 离机副本：已落地（2026-08-11）

先前曾决定推迟到 M3（理由：个人项目、raw_items 可重新采集、代码已在 GitHub）。用户随后提供了专用 COS 桶，故提前完成——这是纯收益，无需等到数据不可重建时才补。

配置：

| 项 | 值 |
|---|---|
| 桶 | `scholars-backups-1400089319`，**私有读写** |
| 地域 | `ap-singapore` —— **与 VPS 同地域**（实测 VPS 在 ap-singapore-1），走内网、不计公网流量费 |
| 凭证 | 专用子账号 `scholars-ai-backup`，**只授对象级权限**，与 aicave 生产资源的凭证完全隔离 |
| 远端路径 | `db/` 前缀，保留最近 7 份（与本地同策略） |

**灾难恢复已端到端实测**（2026-08-11）：删空全部本地副本 → 从 COS 下载 → 解密 → 恢复到临时库，校验 pgvector/pgmq 扩展、11 个枚举、6 个队列、3 个 HNSW 索引、21 个信源、274 条 raw_items（embedding 全部完好且余弦计算正常）、goose 版本，全部还原。
`restore.sh` 支持 `cos:<key>` 形式直接从远端恢复，因为灾难场景下本地副本可能一并丢失。

### 为什么不用 coscli，改用官方 Python SDK

最小权限的子账号策略与 coscli 的行为不兼容，实测踩到三处，都因此改用 SDK 并留下记录：

| 现象 | 原因 |
|---|---|
| `coscli ls` / 下载前失败（403） | coscli 在下载前做桶级 `HEAD`，需要 `cos:HeadBucket`；子账号只有对象级权限 |
| SDK `upload_file` 大文件失败（416B 成功、1.25MB 失败） | 超过阈值自动改走分片上传，需要 `ListMultipartUploads` 等权限 → 改用 `put_object` 单请求 PUT（上限 5GB，远超备份体积） |
| SDK `list_objects` 带 `Prefix` 失败、不带则正常 | 该策略下前缀化 GetBucket 被拒 → 改为列举后在客户端过滤（对象数少，无性能影响） |

取舍：**保持更小的权限面，让工具适配权限**，而不是为了工具方便去放宽云端授权。
上传后回读 `Content-Length` 校验——只写不验的备份等于没备份。
离机上传失败**只告警不使本次备份失败**（本地副本此时已就绪且已验证完整性）。

## 理由

1. 少一个外部依赖：pgmq/pgvector 由自建镜像内置，业务库与队列、向量、Langfuse 同处一地。
2. 容量与成本：不受免费额度限制；VPS 已付费，边际成本为 0。
3. 延迟：core/agents 与数据库同机，省去公网往返（写作/评分任务读写频繁）。
4. 数据主权：内容资产、prompt trace 全在自己手里。
5. 学习收益：数据库运维（备份/恢复/监控）本身属于用户想练的生产级服务工程。

## 后果与代价

- **运维责任转移到自己**：备份、升级、故障恢复都要自己做——由上述四条硬性要求兜底。
- 单点：VPS 挂了业务全停（单人内容系统可接受；备份保证数据不丢）。
- 磁盘压力上升：需持续关注（trace 保留期 + 告警）。
- 若未来需要多地部署或托管化，Postgres 标准协议保证可迁回任何托管服务（无锁定）。
