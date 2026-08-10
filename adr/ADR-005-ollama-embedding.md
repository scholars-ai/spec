# ADR-005 · Embedding 方案：SiliconFlow API（bge-m3）；本机 Ollama 降为备用

- 状态：Accepted（v2，2026-08-10 修订。v1 曾决定用 VPS 自托管 Ollama，**实测不可行**，见 §决策变更）
- 日期：2026-08-08 / 2026-08-10
- 相关：SPEC-003（去重）、SPEC-006（记忆检索）、ADR-004（自托管取向）

## 背景

去重（raw_items/topics）与记忆检索（insights）依赖 embedding。库表契约为 `vector(1024)`。

## 决策

**默认后端：SiliconFlow API，模型 `BAAI/bge-m3`**（OpenAI 兼容协议，原生 1024 维）。
本机 Ollama 路径保留在代码中，通过 `EMBED_BACKEND=ollama` 切换，供离线/调试使用。

配置（环境变量，密钥只存 VPS 部署工作区）：

| 变量 | 默认 | 说明 |
|---|---|---|
| `EMBED_BACKEND` | `siliconflow` | 或 `ollama` |
| `SF_API_KEY` | — | 必需 |
| `SF_EMBED_MODEL` | `BAAI/bge-m3` | 原生 1024 维，中英双语 |
| `EMBED_MAX_CHARS` | `600` | 输入截断上限，见 §性能命门 |

## 决策变更：为什么放弃 v1 的本机 Ollama

v1 的判断"VPS 已有 Ollama 与模型，零成本零部署"在实测中被推翻：

| 实测项 | 结果 |
|---|---|
| `qwen3-embedding:4b` 内存占用 | **4.9 GB RSS**（VPS 总内存 7.5 GB） |
| 实测时 VPS 状态 | 可用内存 **955 MiB**、**swap 已用 4 GB**、load average 5.6 |
| 4b 冷启动 / 首次 4000 字长文本 | 30–37 s / **465 s** |
| 换 `qwen3-embedding:0.6b`（639 MB）后 | 300 字 **57 s**、600 字 **32 s**、1000 字 **58 s**，方差 20–75 s |
| 采集实际表现 | AIHOT 单源 50 条，**10 分钟只处理 14 条**即超时 |

关键结论：**瓶颈不是模型大小，而是 VPS 整体资源竞争**（同机还有 RSSHub + browserless、另一个 Postgres、codex-cron 的写作 agents）。换更小的模型无法解决——0.6b 仍然不稳。

改用 API 后同一份采集任务：

| | 本机 Ollama | SiliconFlow API |
|---|---|---|
| 单次 embedding | 20–75 s | **0.33 s** |
| AIHOg 单源 50 条 | 10 min 处理 14 条（超时） | **14 秒全部完成** |
| 语义质量 | — | 相似句 0.9523 / 不相关 0.3980（区分度良好，0.92 去重阈值合理） |

## 理由

1. **可行性优先**：本机方案实测跑不动，不是优化问题而是能否运行的问题。
2. 成本可忽略：输入截断至 600 字符后，每条约 400 token；每天 50–100 条 ≈ 2–4 万 token，每月 60–120 万 token。该量级下即便付费模型也是每月几毛钱。
3. 维度天然匹配：bge-m3 原生 1024 维 = 库表契约，**无需 MRL 截断**（v1 因 4b 的 2560 维超出 pgvector HNSW 上限 2000 才需要截断）。
4. 中英双语：信源结构是中文资讯 + 英文论文/媒体（arXiv、TechCrunch、The Verge），纯中文模型（如 bge-large-zh）不适配。
5. 释放 VPS 内存给真正必须本机运行的组件（Postgres、core、agents、Langfuse）。

## 性能命门：输入截断至 600 字符

`EMBED_MAX_CHARS=600` 不是随意取值。embedding 在本项目只用于**语义指纹**（同一事件去重、相似检索），标题 + 开头数百字已足够；全文完整保存在 `raw_items.content`，写作时读的是那里。

不截断的代价在 v1 实测中很直接：4000 字符输入的冷调用达 465 秒。API 后端下也同样影响 token 成本与延迟。

## 后果与代价

- **偏离 ADR-004 的"数据主权"取向**：素材原文的前 600 字会发往第三方 API。可接受——采集的素材本就是公开内容，不含隐私或自有资产（prompt、评分数据不经过 embedding 接口）。
- 新增外部依赖与故障面：API 不可用时采集会失败。当前策略是逐条失败隔离（不阻塞整批），后续可加本地 Ollama 降级（接口已隔离，切 `EMBED_BACKEND` 即可）。
- **换模型/换维度必须走"新列 + 回填 + 切换"**，不可原地替换，否则库内既有向量全部失效。当前契约：**1024 维**。
- 密钥管理：`SF_API_KEY` 纳入既有三层防线（.gitignore + gitleaks CI + push protection）。
