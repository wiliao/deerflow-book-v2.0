# 第九章 · Memory 记忆系统

> **本章目标**：
> 1. 理解 DeerFlow 2.0 Memory 的真实实现：JSON profile/facts + LLM 更新。
> 2. 掌握 `MemoryStorage`、`MemoryMiddleware`、`MemoryUpdater` 和 prompt 注入流程。
> 3. 区分内置 Memory 与企业外部 RAG/知识库，避免把 Memory 误写成向量数据库。

DeerFlow 2.0 的 Memory 系统不是 embedding/vector DB，也不是语义相似度检索系统。它的核心实现是：

1. 用 JSON 文件保存用户画像、历史摘要和 facts。
2. 用 LLM 从对话中提取事实、偏好、纠错和上下文。
3. 按用户和 Agent 维度隔离 memory 文件。
4. 在运行时把格式化后的 memory 注入提示词。
5. 通过异步队列和 debounce 降低更新频率。

这一点很重要：当前源码中没有 embedding 字段、没有 FAISS/Chroma/Milvus 等向量索引，也没有基于 similarity score 的 Memory recall。它更接近一个由 LLM 维护的结构化用户 profile。

## 9.1 设计目标

Memory 的目标不是“召回语义最相近片段”，而是让 Agent 在后续任务里知道稳定、可操作的用户背景。

| 目标 | 实现方式 |
|------|----------|
| 记住用户偏好 | LLM 从对话提取 preference / behavior facts |
| 记住项目上下文 | 写入 workContext、topOfMind、history summaries |
| 记住纠错 | detect correction 后写入 high-confidence correction facts |
| 控制上下文长度 | `format_memory_for_injection()` 按 token budget 注入 |
| 多用户隔离 | storage path 按 `user_id` 和 `agent_name` 解析 |

DeerFlow 选择这种实现，是因为长期个人/项目记忆更需要稳定、可解释和任务相关，而不只是语义相近。

## 9.2 模块结构

```text
backend/packages/harness/deerflow/agents/memory/
├── __init__.py               # Memory 模块导出
├── storage.py                # MemoryStorage / FileMemoryStorage
├── updater.py                # LLM 更新、JSON 解析、facts 增删
├── queue.py                  # 异步更新队列与 debounce
├── prompt.py                 # 更新 prompt、注入格式化
├── message_processing.py     # 过滤消息、检测 correction/reinforcement
└── summarization_hook.py     # 与 SummarizationMiddleware 交互
```

Agent 侧入口：

```text
backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py
```

`MemoryMiddleware` 是 LangChain `AgentMiddleware`。它在 `after_agent()` 中把本轮对话加入 Memory update queue。

## 9.3 JSON 存储结构

`storage.py` 的 `create_empty_memory()` 定义默认结构：

```json
{
  "version": "1.0",
  "lastUpdated": "2026-06-21T00:00:00Z",
  "user": {
    "workContext": {
      "summary": "",
      "updatedAt": ""
    },
    "personalContext": {
      "summary": "",
      "updatedAt": ""
    },
    "topOfMind": {
      "summary": "",
      "updatedAt": ""
    }
  },
  "history": {
    "recentMonths": {
      "summary": "",
      "updatedAt": ""
    },
    "earlierContext": {
      "summary": "",
      "updatedAt": ""
    },
    "longTermBackground": {
      "summary": "",
      "updatedAt": ""
    }
  },
  "facts": []
}
```

`facts` 中每条记录是普通 JSON object：

```json
{
  "id": "fact_ab12cd34",
  "content": "User prefers concise Chinese status updates for repo work.",
  "category": "preference",
  "confidence": 0.95,
  "createdAt": "2026-06-21T00:00:00Z",
  "source": "thread_id"
}
```

没有 embedding。没有向量索引。没有相似度分数。

## 9.4 Storage：文件存储与隔离

`MemoryStorage` 是抽象接口：

```python
class MemoryStorage(abc.ABC):
    @abc.abstractmethod
    def load(self, agent_name: str | None = None, *, user_id: str | None = None) -> dict[str, Any]:
        pass

    @abc.abstractmethod
    def reload(self, agent_name: str | None = None, *, user_id: str | None = None) -> dict[str, Any]:
        pass

    @abc.abstractmethod
    def save(self, memory_data: dict[str, Any], agent_name: str | None = None, *, user_id: str | None = None) -> bool:
        pass
```

默认实现是 `FileMemoryStorage`：

| 能力 | 实现 |
|------|------|
| 文件格式 | JSON |
| 读写方式 | `json.load` / `json.dump(..., ensure_ascii=False)` |
| 原子写入 | 先写临时文件，再 `replace()` |
| 缓存 | `(user_id, agent_name)` + 文件 mtime |
| 并发保护 | `threading.Lock` 保护 cache |
| 路径安全 | `agent_name` 必须匹配 `AGENT_NAME_PATTERN` |

路径选择逻辑：

```text
user_id + agent_name -> user_agent_memory_file(user_id, agent_name)
user_id only         -> user_memory_file(user_id)
agent_name only      -> agent_memory_file(agent_name)
legacy global        -> memory_file 或 memory.storage_path
```

这和 DeerFlow 2.0 的多用户架构一致：Memory 不再只是全局文件，也能按用户与 Agent 作用域隔离。

## 9.5 MemoryMiddleware：何时更新

`MemoryMiddleware` 不在模型调用前做向量检索。它在 `after_agent()` 做一件事：把有意义的对话放进更新队列。

```text
Agent finishes
  |
  v
MemoryMiddleware.after_agent()
  |
  +-- check memory.enabled
  +-- resolve thread_id
  +-- read state["messages"]
  +-- filter to user inputs and final assistant responses
  +-- detect correction / reinforcement
  +-- capture effective user_id
  +-- queue.add(...)
```

它只保留用户输入和最终 assistant 回复，忽略 tool calls。这样做是为了避免把工具噪声、临时错误和大段中间过程直接写入长期记忆。

## 9.6 Queue：异步与 Debounce

Memory 更新由 `queue.py` 管理。设计目标：

1. 不阻塞 Agent 主流程。
2. 多轮对话可以 debounce 合并。
3. update 时携带 `thread_id`、`agent_name`、`user_id`、correction/reinforcement 信号。
4. 用户上下文在 enqueue 时捕获，因为 timer 线程不会自动继承 request ContextVar。

这解释了为什么 Memory 是“最终一致”的：一次 Agent 结束后，记忆更新可能稍后发生，而不是同步写入。

## 9.7 Updater：LLM 生成 JSON 更新

`MemoryUpdater` 的核心流程：

```text
load current memory
  |
  v
format conversation for update
  |
  v
build MEMORY_UPDATE_PROMPT
  |
  v
model.invoke(..., run_name="memory_agent")
  |
  v
parse first valid JSON object
  |
  v
apply updates
  |
  v
strip upload event mentions
  |
  v
save JSON memory
```

LLM 输出格式包含四部分：

```json
{
  "user": {
    "workContext": {
      "shouldUpdate": true,
      "summary": "..."
    },
    "personalContext": {
      "shouldUpdate": false,
      "summary": ""
    },
    "topOfMind": {
      "shouldUpdate": true,
      "summary": "..."
    }
  },
  "history": {
    "recentMonths": {
      "shouldUpdate": true,
      "summary": "..."
    },
    "earlierContext": {
      "shouldUpdate": false,
      "summary": ""
    },
    "longTermBackground": {
      "shouldUpdate": false,
      "summary": ""
    }
  },
  "newFacts": [
    {
      "content": "...",
      "category": "preference",
      "confidence": 0.95
    }
  ],
  "factsToRemove": ["fact_ab12cd34"]
}
```

`updater.py` 会做严格归一化：

| 输入 | 处理 |
|------|------|
| 非 JSON 或缺少必要 top-level key | 解析失败，本次更新放弃 |
| malformed `newFacts` 且同时有 `factsToRemove` | 视为 unsafe partial update，拒绝 |
| fact content 为空 | 丢弃 |
| confidence 非有限数、<0 或 >1 | 丢弃或报错 |
| 重复 fact content | 去重 |
| fact 数超过 `max_facts` | 按 confidence 降序保留 |

## 9.8 Fact 分类

官方 prompt 使用这些 category：

| category | 含义 |
|----------|------|
| `preference` | 工具、风格、方法偏好 |
| `knowledge` | 用户掌握的技术或领域知识 |
| `context` | 背景事实，如项目、职位、语言 |
| `behavior` | 工作模式、沟通习惯、问题处理方式 |
| `goal` | 明确目标、学习方向、项目计划 |
| `correction` | 用户明确纠正过的错误和正确做法 |

`correction` 是 2.0 Memory 里很实际的设计：当系统检测到用户纠错时，会提示 LLM 特别关注“哪里错了、正确做法是什么”，并用较高 confidence 记录。

## 9.9 上传文件事件不会长期记忆

`updater.py` 有 `_strip_upload_mentions_from_memory()`，会删除“用户上传了某文件”这类记忆。

原因是上传文件通常是 session-scoped。把“用户上传了 `/mnt/user-data/uploads/foo.pdf`”写入长期记忆，会导致未来会话里 Agent 试图访问已经不存在的文件。

## 9.10 Prompt 注入

Memory 的使用点不是 similarity recall，而是格式化注入。

`prompt.py` 的 `format_memory_for_injection()` 会：

1. 读取 user/history summaries。
2. 按 confidence 对 facts 排序。
3. 在 token budget 内尽量加入高置信 facts。
4. 输出可放入提示词的文本。

这使 Memory 的行为可解释：模型看到的是明确的 profile 和 facts，而不是从向量库召回的一堆片段。

## 9.11 配置要点

主配置在 `config.yaml` 的 `memory` block：

```yaml
memory:
  enabled: true
  storage_path: memory.json
  debounce_seconds: 30
  model_name: null
  max_facts: 100
  fact_confidence_threshold: 0.7
  injection_enabled: true
  max_injection_tokens: 2000
  token_counting: tiktoken
```

注意：

1. `storage_path` 是 legacy/global 场景；多用户路径会优先走 `get_paths().user_memory_file(user_id)` 等方法。
2. `token_counting` 控制注入时 token 估算，不代表 embedding。
3. 企业知识库 RAG 应作为 MCP/custom tool 接入，不要误写成内置 Memory 的向量索引。

## 9.12 API 与 SDK 操作

DeerFlow 暴露了 Memory 管理能力，典型操作包括：

```python
client.get_memory()
client.export_memory()
client.import_memory(data)
client.reload_memory()
client.clear_memory()
client.create_memory_fact("内容", category="context")
client.update_memory_fact(fact_id, content="新内容")
client.delete_memory_fact(fact_id)
```

这些操作本质上都是读取或修改 JSON memory 数据。`create_memory_fact()` 会直接追加 fact 并保存；`update_memory_fact()` 和 `delete_memory_fact()` 按 fact id 修改。

## 9.13 企业扩展建议

企业知识库、向量数据库、RAG 可以接入 DeerFlow，但它们不是 DeerFlow 内置 Memory 的实现。推荐区分两层：

| 层级 | 用途 | 推荐接入方式 |
|------|------|--------------|
| DeerFlow Memory | 用户偏好、纠错、长期 profile | 保持 JSON/LLM 更新模型 |
| 企业知识库 | 文档、制度、FAQ、代码库检索 | MCP server、RAG service、custom tool |
| 项目审计 | 决策、审批、工具调用记录 | run_events、外部审计仓库 |

如果要把企业知识库结果注入 Agent，不建议改造 Memory 为向量库。更稳妥的做法是：

1. 建一个带权限过滤的知识库 MCP server。
2. 在 Gateway/MCP interceptor 中注入 user、tenant、request id。
3. 由知识库服务端做文档级权限判断。
4. 让 Agent 在需要时调用检索工具。

这样可以保持 Memory 的简单可靠，同时把大规模检索交给更合适的系统。

## 9.14 小结

DeerFlow 2.0 Memory 的核心要点：

| 组件 | 功能 |
|------|------|
| `FileMemoryStorage` | JSON 文件读写、cache、原子保存 |
| `MemoryMiddleware` | Agent 结束后排队更新 |
| `MemoryUpdateQueue` | debounce、异步更新、携带 user/thread/agent 上下文 |
| `MemoryUpdater` | LLM 生成 JSON 更新、合并 summaries 和 facts |
| `format_memory_for_injection()` | 按 token budget 注入 profile/facts |

需要避免的误解：

- 不要把 DeerFlow Memory 写成 embedding/vector DB。
- 不要把 semantic similarity 当作 Memory 召回机制。
- 不要把企业 RAG 当作内置 Memory。
- 不要把临时上传文件事件写入长期记忆。
