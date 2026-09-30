# 智能体开发 · 30 天面试课程表

每天一个主题，按 `progress.md` 里的 `next_day` 取。30 天后进入下一轮，按反馈和薄弱点加深。
这份表你可以随时改：调整顺序、替换主题、在某行后面加备注，下次运行都会按新内容来。

## 权威来源

内容以 LangChain 和 Claude（Anthropic）的官方资料为准，每篇笔记末尾都会列出引用链接。
- LangChain 官方：docs.langchain.com、python.langchain.com、langchain-ai.github.io（LangGraph）、blog.langchain.com、github.com/langchain-ai
- Claude / Anthropic 官方：docs.claude.com、docs.anthropic.com、anthropic.com/engineering、anthropic.com/research、github.com/anthropics
- MCP 官方规范：modelcontextprotocol.io（由 Anthropic 发起）

这两家覆盖不到的主题（如 A2A、AutoGen、CrewAI、DPO），使用该项目自己的官方文档或原始论文，并在笔记里标注「非 LangChain/Claude 来源」。

| Day | 主题 | 重点 |
|---|---|---|
| 1 | LLM 基础 | Transformer 直觉、token、上下文窗口、temperature/top-p、幻觉成因 |
| 2 | Prompt 工程 | system prompt、few-shot、CoT、结构化输出、提示词迭代方法 |
| 3 | 什么是 Agent | Agent 定义、Agent vs Workflow、Agent loop、何时不该用 Agent |
| 4 | Function Calling / Tool Use | 调用流程、JSON Schema、并行调用、模型如何选工具 |
| 5 | 工具设计 | 工具描述写法、粒度、错误返回、幂等、权限边界 |
| 6 | ReAct | Thought-Action-Observation、实现一个最小 ReAct 循环、常见失败 |
| 7 | 规划 | Plan-and-Execute、ReWOO、任务分解、重新规划 |
| 8 | 反思与自我纠错 | Reflexion、self-critique、验证器、重试策略 |
| 9 | 记忆机制 | 短期/长期记忆、摘要压缩、向量记忆、记忆写入与遗忘 |
| 10 | RAG 基础 | 切分策略、embedding、向量库、检索评估 |
| 11 | RAG 进阶 | 混合检索、重排、查询改写、HyDE、GraphRAG |
| 12 | Agentic RAG | 检索作为工具、多跳检索、何时检索 |
| 13 | 上下文工程 | 上下文预算、prompt caching、上下文污染与隔离 |
| 14 | MCP 协议 | 架构、transport、tools/resources/prompts、写一个 MCP server |
| 15 | Agent 间通信 | A2A 等协议、消息格式、能力发现、与 MCP 的区别 |
| 16 | 多智能体架构 | orchestrator-worker、handoff、并行子 Agent、辩论、适用场景 |
| 17 | LangGraph | 状态图、节点与边、条件路由、checkpoint、人工中断 |
| 18 | 框架对比 | AutoGen、CrewAI、OpenAI Agents SDK、Claude Agent SDK 等选型 |
| 19 | Code Agent 与 Computer Use | 代码执行沙箱、浏览器/桌面 Agent、GUI 操作 |
| 20 | 结构化输出与校验 | JSON mode、schema 约束、Pydantic 校验、失败修复 |
| 21 | 评测 | 评测集构建、LLM-as-judge、轨迹评测、回归测试 |
| 22 | 可观测性 | tracing、span、日志、Langfuse/LangSmith、线上排障 |
| 23 | 安全 | prompt injection、间接注入、工具权限、沙箱隔离、数据泄露 |
| 24 | 护栏与人在回路 | 输入输出过滤、审批节点、可逆性设计 |
| 25 | 成本与延迟优化 | 模型路由、缓存、并行、流式输出、token 预算 |
| 26 | 部署与生产化 | 异步任务、重试与超时、状态持久化、长任务、扩缩容 |
| 27 | 模型适配 | Prompt vs RAG vs 微调、SFT/DPO/RLHF 基础、何时微调 |
| 28 | 端侧与移动端 Agent | Android 上的 Agent、App 能力作为工具、车载语音 Agent、端云协同 |
| 29 | 系统设计题（一） | 设计一个企业客服 Agent：需求、架构、工具、评测、上线 |
| 30 | 系统设计题（二）+ 总复习 | 设计一个 Coding / 深度研究 Agent；30 天知识串联 |
