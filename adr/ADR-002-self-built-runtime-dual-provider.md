# ADR-002 · 自研轻量 Agent runtime + 双协议 ModelProvider 抽象

- 状态：Accepted
- 日期：2026-08-07

## 背景

Agent 实现有三条路：自研轻量 runtime（Anthropic/OpenAI SDK 直调 + 自写 loop）、Claude Agent SDK、LangGraph 等框架。同时用户明确要求：模型层要能**无缝切换 Anthropic 协议与 OpenAI 协议**（接入 DeepSeek/Qwen/Kimi/GLM 等 OpenAI 兼容模型）。

关键判断：本项目五步流程是**确定性工作流 + 节点上的结构化 LLM 调用**，而非开放式自主 Agent。真正需要"loop + 工具调用"的只有 TopicScout 和 Reflector。

## 决策

1. **自研轻量 runtime**（Python）：agent loop、tool dispatch、结构化输出（JSON Schema 约束 + 校验失败重试）、成本记账、重试。
2. **ModelProvider 抽象层**：统一 `complete(ChatRequest) -> ChatResponse` 接口，两个实现：
   - `AnthropicProvider`：anthropic 官方 SDK（原生 caching/thinking/tool_use block）
   - `OpenAICompatProvider`：openai 官方 SDK + base_url 覆盖，一个 adapter 通吃所有 OpenAI 兼容端点（DeepSeek/Qwen/Kimi/GLM/OpenRouter/Ollama）
3. **按任务路由模型**：`model_routing.yaml` 配置 task → provider/model，切换零代码改动。
4. 协议差异归一策略：工具调用/工具结果/system/结构化输出/usage 的两协议差异全部在 adapter 内消化，runtime 之上不感知协议。

## 理由

- 学习收益最大：吃透 agent loop 与双协议差异，正是"agent model 层"的核心知识；框架用户与框架作者的理解深度差一个量级。
- 控制权：pipeline 的流程控制权必须在自己手里，Claude Agent SDK 为开放式自主 agent 设计（文件系统/bash 优先），与场景错配。
- 多模型是硬需求：国产模型走 OpenAI 协议是事实标准，单绑 Claude Agent SDK 不满足。
- 工作量可控：Provider 层约 400–600 行，loop 两三百行。

## 后果与代价

- subagent 编排、MCP 接入等需自建（当前场景不需要；未来需要时单点引入）。
- 留接缝：Provider 接口设计与 LiteLLM 形态兼容，自研维护不动时可换入；某个 Agent 未来需要开放式自主性时，可单点换 Claude Agent SDK 而不动全局。
- 结构化输出的跨协议一致性（Anthropic 强制 tool call vs OpenAI json_schema）需要充分测试覆盖。
