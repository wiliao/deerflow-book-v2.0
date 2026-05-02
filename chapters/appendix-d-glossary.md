# 附录 D · 术语表

> 本术语表汇总 DeerFlow 文档中的核心概念与缩写，按字母顺序排列，方便快速查阅。

---

## A

**Agent（代理）**
：具备感知、决策和执行能力的 AI 实体。DeerFlow 中 Agent 分为 Lead Agent（主代理）和 Sub-Agent（子代理），通过 LangGraph 的状态图进行编排。
> 相关章节：第 2 章、第 5 章、第 7 章

---

## C

**Checkpoint（检查点）**
：LangGraph 提供的持久化机制，用于保存和恢复 Agent 执行状态。DeerFlow 利用 checkpoint 实现长时任务的断点续传和错误恢复。
> 相关章节：第 2 章、第 9 章

**Context（上下文）**
：LLM 在单次推理中能够接收的全部输入信息，包括系统提示、用户消息、工具结果和历史记录。DeerFlow 通过 Context Engineering 技术对上下文进行分层管理。
> 相关章节：第 10 章

---

## E

**Ed25519**
：一种现代椭圆曲线数字签名算法，具有速度快、密钥短、安全性高的特点。DeerFlow 在审计日志签名和 MCP 认证中可选使用 Ed25519。
> 相关章节：第 11 章、第 14 章

**Edge（边）**
：LangGraph 状态图中的连接元素，定义了状态节点之间的流转方向。Edge 可以是固定的（`add_edge`）或条件路由的（`add_conditional_edges`）。
> 相关章节：第 2 章、第 5 章

---

## G

**Graph（图）**
：LangGraph 中的核心抽象，由节点（Node）和边（Edge）组成的有向图，用于描述 Agent 的执行流程和状态转换。
> 相关章节：第 2 章、第 5 章

---

## L

**LLM（大语言模型）**
：Large Language Model 的缩写，DeerFlow 支持多种 LLM 后端（OpenAI、Claude、Gemini 等），通过统一接口进行调用和切换。
> 相关章节：第 1 章、第 5 章

---

## M

**MCP（Model Context Protocol）**
：由 Anthropic 推出的开放协议，用于标准化 AI 模型与外部工具、数据源的连接。DeerFlow 原生支持 MCP Server 集成。
> 相关章节：第 11 章

**Memory（记忆）**
：Agent 在跨会话中保留的信息，用于维持长期上下文连续性。DeerFlow 的记忆系统分为 Working Memory、Semantic Memory 和 Episodic Memory 三层。
> 相关章节：第 2 章、第 9 章

**Middleware（中间件）**
：插入在 Agent 执行流程中的可扩展组件，用于实现安全检查、上下文压缩、审计日志等横切关注点。DeerFlow 内置 18 个中间件。
> 相关章节：第 3 章、第 5 章

---

## N

**Node（节点）**
：LangGraph 状态图中的执行单元，每个节点对应一个处理函数（如规划、执行、审查）。节点接收当前状态并返回更新后的状态。
> 相关章节：第 2 章、第 5 章

---

## P

**pgvector**
：PostgreSQL 的扩展插件，为数据库添加向量相似度搜索能力。DeerFlow 使用 pgvector 作为语义记忆的向量存储后端。
> 相关章节：第 9 章

**Prompt（提示词）**
：发送给 LLM 的输入文本，包含系统指令、用户问题和上下文信息。DeerFlow 的 Skill 系统通过 SKILL.md 文件管理可复用的 Prompt 模板。
> 相关章节：第 2 章、第 6 章、第 12 章

---

## S

**Sandbox（沙箱）**
：隔离的执行环境，用于安全运行 AI Agent 生成的代码。DeerFlow 支持 Local Sandbox、Docker Sandbox 和 K8s Provisioner 三种实现。
> 相关章节：第 2 章、第 8 章

**SSE（Server-Sent Events）**
：一种服务器向客户端单向推送实时数据的标准技术。DeerFlow 的 Gateway API 使用 SSE 向客户端流式传输 Agent 执行事件。
> 相关章节：第 3 章、第 11 章

**Skill（技能）**
：DeerFlow 中可复用的能力单元，包含 Prompt 模板、工具定义和实现代码。Skill 通过 SKILL.md 文件描述，支持动态加载和渐进式加载。
> 相关章节：第 2 章、第 6 章、第 12 章

**StateGraph（状态图）**
：LangGraph 的核心抽象，一种特殊的有向图，其中每个节点接收并返回状态对象。DeerFlow 的 Agent 编排基于 StateGraph 实现。
> 相关章节：第 2 章、第 5 章

**Sub-Agent（子代理）**
：由 Lead Agent 委派执行特定任务的子级 Agent，拥有独立的工具集和配置。Sub-Agent 通过协作机制实现复杂任务的分治处理。
> 相关章节：第 2 章、第 7 章

---

## T

**Thread（线程/会话）**
：DeerFlow 中的对话上下文容器，每个用户会话对应一个 Thread，包含消息历史、中间状态和记忆引用。Thread 通过 `thread_id` 唯一标识。
> 相关章节：第 3 章、第 5 章

**Tool（工具）**
：Agent 可调用的外部能力接口，如代码执行、文件操作、网络请求等。Tool 是 Skill 的底层实现，一个 Skill 可以暴露多个 Tool。
> 相关章节：第 2 章、第 6 章

---

## W

**Workflow（工作流）**
：由状态图编译而成的可执行对象，定义了 Agent 的完整处理流程。DeerFlow 通过 `workflow.compile()` 将 StateGraph 转换为可运行的工作流。
> 相关章节：第 2 章、第 5 章
