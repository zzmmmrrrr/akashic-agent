# akashic-agent 技术学习指南

> 面向面试的知识体系整理，聚焦宏观技术栈与 Agent 领域热门技术。

---

## 一、项目一句话定位

**akashic-agent** 是一个 **多频道（Telegram / QQ）、带长期记忆、支持自主推送与后台闲时任务、支持插件扩展的个人 AI Agent 框架**。

核心能力：
- 被动回复：接收用户消息，经过 5 阶段生命周期管道，产生回复
- 主动推送：定时拉取外部信息源，Agent 自主决定是否推送
- Drift 闲时任务：没有推送内容时，Agent 利用空闲时间自主执行 skill 任务
- 插件系统：声明式注册 tool / lifecycle hook / pre-hook 拦截
- 长期记忆：语义向量检索 + 关键词检索 + RRF 融合，支持记忆沉淀与过期

---

## 二、项目技术栈全景

### 2.1 语言与运行时

| 技术 | 用途 |
|------|------|
| **Python 3.14+** | 主力语言 |
| **asyncio** | 全链路异步（async/await），单进程并发处理多频道多会话 |
| **Pydantic v2** | 配置模型、数据校验、PluginConfig schema |
| **dataclasses** | 轻量数据载体（PhaseFrame、ToolDocument、GatewayResult 等） |

### 2.2 存储层

| 技术 | 用途 |
|------|------|
| **SQLite + SQLAlchemy 2.0** | 会话存储（sessions.db）、observe trace |
| **sqlite-vec** | 向量存储扩展，本地语义检索（无需外部向量数据库） |
| **json_store** | 轻量 JSON 文件存储（proactive 状态、schedule、quota 等） |
| **Markdown 文件** | 长期记忆文件（MEMORY.md、SELF.md、HISTORY.md） |

### 2.3 LLM 对接

| 技术 | 用途 |
|------|------|
| **OpenAI SDK** | 统一的 LLM 调用协议（兼容 OpenAI / DeepSeek / Qwen 等） |
| **多模型分离** | main（推理）、fast/light（memory gate / query rewrite / HyDE）、vl（视觉） |
| **Prompt Caching** | KV-cache 复用，降低长 system prompt 的重复计算成本 |
| **Streaming** | SSE 流式输出，通过 EventBus 分发 StreamDeltaReady 事件 |

### 2.4 通讯层

| 技术 | 用途 |
|------|------|
| **python-telegram-bot** | Telegram Bot 接入 |
| **ncatbot** | QQ Bot 接入 |
| **FastAPI + uvicorn** | Dashboard HTTP API + IPC Server（Unix domain socket） |
| **Rich / Textual** | CLI TUI 客户端 |

### 2.5 网络与内容

| 技术 | 用途 |
|------|------|
| **httpx / curl_cffi** | HTTP 客户端（含 TLS 指纹伪装） |
| **BeautifulSoup4 / lxml / html2text** | 网页抓取与内容提取 |
| **duckduckgo_search** | Web 搜索后端 |
| **yt-dlp** | 视频/音频内容下载 |

### 2.6 定时与后台

| 技术 | 用途 |
|------|------|
| **APScheduler** | 定时任务调度（proactive tick、schedule 工具） |
| **schedule** | 轻量定时表达式 |

---

## 三、Agent 核心技术详解

### 3.1 ReAct 循环（Reasoning + Acting）

**是什么**：Agent 的核心执行模式。模型不是一次生成回答，而是多轮迭代：每轮可以调用工具获取信息，再基于工具结果继续推理，直到产出最终回复。

**在本项目中的实现**：`DefaultReasoner.run()`（`agent/core/passive_turn.py:719`）

```
for iteration in range(max_iterations):
    BeforeStep  →  token 估算 + 注入提示
    LLM 调用    →  得到 tool_calls 或最终回复
    如果有 tool_calls  →  执行工具 → AfterStep → 继续下一轮
    如果没有 tool_calls →  最终回复 → 退出循环
```

**关键设计点**：
- **工具循环保护**：连续 3 次相同签名调用触发 force stop
- **步骤预算**：剩余 ≤5 步时注入 warning hint，最后一步强制收尾
- **空回复重试**：模型只输出 thinking 无 content 时自动 retry
- **Deferred Tool Loading**：非 always-on 工具不暴露 schema，模型按需 `tool_search` 加载，节省 prompt tokens

**面试话术**：
> Agent 的本质是 LLM + Tool-Use Loop。我们的 Reasoner 实现了一个有界 ReAct 循环：每次迭代包括 BeforeStep（token 估算 + 提示注入）、LLM 推理、工具执行、AfterStep（fanout 通知）。有工具循环检测保护、步骤预算控制、上下文超长 retry 策略等工程化考虑。

---

### 3.2 Tool Calling / Function Calling

**是什么**：LLM 不只是输出文本，还能输出结构化的工具调用请求（函数名 + JSON 参数）。Agent 框架解析这些请求，执行真实函数，将结果返回给 LLM 继续推理。

**在本项目中的实现**：`ToolRegistry`（`agent/tools/registry.py`）

```python
# 工具注册
registry.register(
    Tool(name="web_search", description="搜索互联网", parameters={...}),
    risk="read-only",
    always_on=False,
)

# 获取 OpenAI 格式的 tool schemas
schemas = registry.get_schemas(names=visible_names)

# 执行工具
result = await registry.execute("web_search", {"query": "agent 最新论文"})
```

**关键设计点**：
- **ToolRegistry**：统一管理 builtin / MCP / plugin 三种来源的工具
- **ToolMeta**：risk（read-only / write / external-side-effect）、always_on
- **ToolSearch**：模型发现工具的方式——不是一次性塞所有 schema，而是按需搜索加载（类似 Anthropic 的 tool search 机制）
- **ToolExecutor**：pre/post hook 链，插件可拦截/修改参数
- **来源标记**：每个工具有 source_type（builtin/mcp/plugin）+ source_name

**面试话术**：
> Tool Registry 是我们 Agent 的能力层。工具按 risk 分级、按来源标记，支持 deferred loading——不在 prompt 中暴露所有工具的 schema，而是通过 tool_search 让模型按需检索。这相比一次性注入所有工具定义，能大幅节省 system prompt 的 token 开销，同时避免模型「看到太多工具不会选」的问题。

---

### 3.3 MCP (Model Context Protocol)

**是什么**：Anthropic 提出的开放协议，让 LLM 应用通过标准化的 JSON-RPC over stdio/HTTP 连接外部工具服务器。类似于 LSP（Language Server Protocol）之于编辑器。

**在本项目中的实现**：`McpClient`（`agent/mcp/client.py`）

```python
# 启动 stdio MCP server 子进程，完成 JSON-RPC 握手
client = McpClient(
    name="filesystem",
    command=["npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
)
tools = await client.connect()  # initialize → tools/list → 获得工具列表

# 调用远端工具
result = await client.call("read_file", {"path": "/tmp/note.txt"})
```

**完整流程**：
1. 启动 MCP server 子进程（stdio）
2. JSON-RPC 握手：`initialize` → `notifications/initialized`
3. 获取工具列表：`tools/list` → 解析为 `McpToolInfo`
4. 注册到 `ToolRegistry`（标记 source_type="mcp"）
5. Agent 调用时通过 `tools/call` 发送 JSON-RPC 请求

**关键设计点**：
- 支持子进程工作目录推断（从 command 中的绝对路径）
- stderr 后台 drain 防止缓冲区阻塞
- 超时信息包含 stdout/stderr 近期输出，方便排障
- 单源失败不影响其他 source（与 gateway 并行拉取原则一致）

**面试话术**：
> MCP 是 Anthropic 推出的模型上下文协议，本质是标准化的工具调用 JSON-RPC 协议。我们实现了一个 MCP 客户端，通过子进程 stdio 通信完成握手和工具调用。MCP server 的工具会被自动注册到 ToolRegistry，与内置工具无差别使用。这样 Agent 的能力边界可以通过安装 MCP server 动态扩展。

---

### 3.4 生命周期管道（Lifecycle Phase Pipeline）

**是什么**：把每条消息的处理拆分为 5 个顶层阶段，每个阶段内部是一条模块链。所有模块（内置 + 插件）在链上是平等节点。

**在本项目中的实现**：`Phase[I, O, F]` + `PhaseModule[F]`（`agent/lifecycle/phase.py`）

```
InboundMessage（输入数据，不是 Phase）
  → [Phase 1] BeforeTurn          session 获取 → 记忆检索 → EventBus emit → 插件介入
  → [Phase 2] BeforeReasoning     工具同步 → 构建上下文 → EventBus emit → prompt 预热
  → [Phase 3] Reasoner.run_turn()  ReAct loop
                ├─ BeforeStep（每轮迭代前：token 估算 + 提示注入）
                ├─ LLM 推理 + 工具执行
                └─ AfterStep（每轮迭代后：fanout 通知）
  → [Phase 4] AfterReasoning      解析回复 → 持久化 → 构建出站消息
  → [Phase 5] AfterTurn           TurnCommitted fanout → dispatch
  → OutboundMessage（输出数据，不是 Phase）
```

**关键澄清**：
- **顶层 Phase 是 5 个**，不是 6 个。`InboundMessage` 和 `OutboundMessage` 是管道处理的数据载体（`Phase[I, O, F]` 中的 I 和 O），不是 Phase
- **BeforeStep / AfterStep** 是内嵌在 `Reasoner.run_turn()` 里的子步骤，用同样的 `PhaseModule` 协议实现，但和顶层 Phase 是层级关系，不应并列数进去
- 完整链路：`InboundMessage → [5 个 Phase 的模块链] → OutboundMessage`

**核心抽象三件套**：

| 抽象 | 说明 |
|------|------|
| `PhaseFrame[I, O]` | 数据盘。`input`（只读）、`slots`（模块间 dict 通道）、`output`（最终产出） |
| `PhaseModule[F]` | 单个步骤。接口 `async (frame) → frame`，可选声明 `requires`/`produces` 做启动校验 |
| `Phase[I, O, F]` | 生产线。按序执行 module 列表 |

**代码示例**：
```python
@dataclass
class PhaseFrame(Generic[I, O]):
    input: I
    slots: dict[str, Any] = field(default_factory=dict)
    output: O | None = None

class PhaseModule(Protocol[F]):
    async def run(self, frame: F) -> F: ...

class Phase(Generic[I, O, F]):
    def __init__(self, modules: Sequence[PhaseModule[F]], *, frame_factory: Callable[[I], F]):
        self._modules = list(modules)
        self._frame_factory = frame_factory
        self._validate()  # 检查 requires/produces 链是否闭合

    async def run(self, input: I) -> O:
        frame = self._frame_factory(input)
        for module in self._modules:
            frame = await module.run(frame)
        return frame.output
```

**面试话术**：
> 我们设计了一个泛型 Phase 管道抽象，每个 Phase 内部是一条 PhaseModule 链。内置模块和插件模块在链上是平等节点——都通过 slots dict 读写数据，都满足相同的协议。这样插件可以以 early/late module 的形式插入 BeforeTurn 阶段，也可以挂载其他的生命周期钩子。这种设计的好处是数据流显式（slots 作为模块间的合同）、易于测试（每个 Phase 可独立单测）、易于扩展（新增阶段只需加一条 module）。

---

### 3.5 EventBus（类型化事件总线）

**是什么**：进程内类型化事件总线，支持三种消费模式，是生命周期各模块之间解耦通信的核心。

**在本项目中的实现**：`EventBus`（`bus/event_bus.py`）

| 模式 | 语义 | 方法 |
|------|------|------|
| **emit** | 顺序拦截链，handler 返回新事件时替换当前事件（GATE） | `await bus.emit(event)` |
| **fanout** | 并发广播，单个 observer 失败不打断主流程（TAP） | `await bus.fanout(event)` |
| **enqueue** | 后台队列消费，不阻塞主回复（异步后处理） | `bus.enqueue(event)` |

**代码示例**：
```python
bus = EventBus()

# 注册 handler（GATE 模式，可改写事件）
bus.on(BeforeTurnCtx, my_plugin_handler)

# emit：顺序执行，handler 可返回修改后的事件
event = await bus.emit(BeforeTurnCtx(...))

# fanout：并发执行观察者，不修改事件
await bus.fanout(ToolCallStarted(...))

# enqueue：放入后台队列异步消费，不阻塞当前 turn
bus.enqueue(TurnCommitted(...))
```

**完整生命周期事件链路**：
```
TurnStarted → BeforeTurn → BeforeReasoning → BeforeStep(每轮)
  → BeforeToolCall → ToolCallStarted → ToolCallCompleted → AfterToolResult
  → AfterStep(每轮) → AfterReasoning → AfterTurn → TurnCommitted
```

**面试话术**：
> 这是典型的 Observer + Chain of Responsibility 模式。emit 支持拦截改写（GATE 语义，用于安全检查、命令拦截），fanout 支持并发观察（TAP 语义，用于日志、trace、进度通知），enqueue 支持后台异步消费（用于 memory 沉淀等不需要阻塞主回复的操作）。相比简单的 callback 注册，EventBus 的类型化设计让每个 handler 能明确知道它处理的是什么事件类型。

---

### 3.6 插件系统（Plugin System）

**是什么**：基于目录扫描 + 声明式装饰器的插件框架，插件可以通过装饰器注册 tool、lifecycle hook、tool pre-hook、before_turn module。

**在本项目中的实现**：`PluginManager`（`agent/plugins/manager.py`）+ 装饰器（`agent/plugins/decorators.py`）

**插件目录结构**：
```
plugins/
  my_plugin/
    plugin.py          # Plugin 子类（入口）
    manifest.yaml      # 元信息（name/version/desc/author）
    _conf_schema.json  # 配置 schema（含 default 值）
    plugin_config.json # 用户配置覆盖
```

**声明式 API**：
```python
from agent.plugins.base import Plugin

class MyPlugin(Plugin):
    name = "my_plugin"

    async def initialize(self): ...

    # 注册工具
    @tool(name="my_tool", risk="read-only")
    async def handle_my_tool(self, event, query: str) -> str:
        """搜索我的知识库"""
        return f"搜索结果：..."

    # 生命周期钩子（GATE，可改写事件）
    @on_before_turn(priority=10)
    async def on_before_turn(self, event: BeforeTurnCtx):
        # 检查并拦截特定命令
        if event.ctx and "/secret" in event.ctx.get("content", ""):
            event.abort = True
            event.abort_reply = "无权访问"
        return event

    # 工具前置拦截（可修改参数）
    @on_tool_pre(tool_name="shell")
    async def before_shell(self, event: PreToolCtx):
        # 安全检查：拒绝危险命令
        ...
        return event.arguments  # 返回修改后的参数

    # BeforeTurn 阶段插入 module
    def before_turn_modules_early(self):
        return [self._my_module]
```

**四种介入方式**：
| 方式 | 机制 | 用途 |
|------|------|------|
| `@tool` | ToolRegistry 注册 | 扩展 Agent 工具 |
| `@on_*` | EventBus GATE/TAP | 生命周期钩子 |
| `@on_tool_pre` | ToolExecutor pre_hook | 工具调用拦截/改参 |
| `before_turn_modules_early/late()` | PhaseModule 链插入 | 记忆检索前/EventBus 后 |

**插件注册链路**：
```
Plugin.__init_subclass__()    → plugin_registry 注册类
PluginManager.load_all()      → discover() → _load_one()
  → _import_plugin()          → exec_module（触发 __init_subclass__）
  → 实例化 Plugin + 读取 manifest
  → _register_tools()         → 动态创建 Tool 子类 → ToolRegistry.register()
  → _bind_handlers()          → EventBus.on(ctx_type, bound_handler)
  → _bind_tool_hooks()        → ToolExecutor.add_hooks()
  → _collect_before_turn_modules()
  → await plugin.initialize() → 异步初始化
```

**面试话术**：
> 我们的插件系统采用声明式装饰器 API，类似 VS Code 的贡献点模式。插件可以通过 `@tool` 注册新工具、通过 `@on_before_turn` 等装饰器钩入生命周期事件、通过 `@on_tool_pre` 拦截工具调用。插件加载采用 importlib 文件路径加载 + `__init_subclass__` 自动注册，支持热发现。关键是内置模块和插件模块在 Phase 管道上是平等节点——都满足 PhaseModule 协议，都通过 slots 读写数据。

---

### 3.7 长期记忆系统（Memory v2）

**是什么**：存储用户偏好、操作流程、历史事件、个人画像，在对话时检索相关记忆注入 prompt。

**在本项目中的实现**：`memory2/` 模块

**完整链路**：

```
1. 记忆写入
   用户对话 → 每轮后 post_response_worker 检测失效 → supersede 旧记忆
   → Consolidation 批量阶段抽取新记忆 → memorizer → store

2. 记忆检索（检索时增强）
   Query → QueryRewriter (light model 改写)
        → HyDE (可选，Hypothetical Document Embedding，生成假设答案再检索)
        → Embedder (text-embedding-v3)
        → 多路检索：
           ├─ Vector Search (sqlite-vec 余弦相似度 + 热度加权)
           ├─ 多 query lane（主 query + aux_queries）
           └─ Keyword Search (CJK 分词)
        → RRF 融合排序
        → SufficiencyChecker (检查是否足够)
        → build_injection_block (格式化 + 字符预算控制)

3. 注入类型
   - 强制约束（procedure + tool_requirement）
   - 流程规范（preference）
   - 相关历史（event）
   - 用户画像（profile）
```

**核心技术点**：

**HyDE（Hypothetical Document Embedding）**：
用轻量模型先生成假设答案，再对假设答案做 embedding 检索。比直接对 query 做 embedding 更精准（query 和 document 可能用词不同）。

**RRF（Reciprocal Rank Fusion）**：
```python
rrf_score = 1/(k + vec_rank) + keyword_weight/(k + keyword_rank)
```
将向量检索和关键词检索的结果融合排序，取各路上的前 top_n。

**热度加权**：
```python
hotness_alpha=0.20, hotness_half_life_days=14.0
# 近期频繁访问的记忆有更高权重
```

**面试话术**：
> 我们的记忆系统是典型的 RAG 模式，但检索侧做了多层增强：QueryRewriter 改写用户 query、可选的 HyDE 生成假设文档再检索、向量 + 关键词双路检索、RRF 融合排序。存储侧用 sqlite-vec 做本地向量存储，避免了外部向量数据库的运维成本。记忆还带有热度衰减权重，最近常访问的记忆排序更靠前。

---

### 3.8 Proactive 主动推送系统

**是什么**：Agent 不只是被动回复，还能定时拉取外部信息源（RSS、API、网页），由 Agent 自主决策是否需要推送给用户。

**在本项目中的实现**：`proactive_v2/` 模块

**核心链路**（`agent_tick.py`）：
```
每个 tick（定时触发）
  ├─ Pre-gate：冷却检查 / 用户在线检查 / busy 检查
  ├─ DataGateway.run()：三路并行预取
  │    ├─ alerts（实时告警，完整内容直传 agent）
  │    ├─ context（上下文条目，直传 agent）
  │    └─ content（feed 内容，并行 web_fetch 存入 content_store）
  └─ Agent Loop（max 20 步）
       工具：recall_memory / get_content / web_fetch
            mark_interesting / mark_not_interesting / send_message
```

**去重机制**：
| 场景 | TTL |
|------|-----|
| 已引用内容/告警 | 168h（7天） |
| interesting 未引用 | 24h |
| mark_not_interesting | 720h（30天） |

**面试话术**：
> Proactive 系统让 Agent 具备「主动意识」。每个 tick 中，DataGateway 先并行预取所有数据源（三路：alerts / context / content），形成一份静态输入快照。然后 Agent Loop 启动，agent 可以调用 mark_interesting / mark_not_interesting 标记内容，也可以直接 send_message 推送。去重按 168h/24h/720h 分级 TTL。单源失败不影响其他源。

---

### 3.9 Drift 闲时自主任务

**是什么**：Proactive gateway 没有可推送内容时，Agent 利用空闲时间自主执行有意义的事情。

**在本项目中的实现**：`DriftRunner`（`proactive_v2/drift_runner.py`）

**核心链路**：
```
Gateway 无内容 → DriftRunner.run(ctx, llm_fn)
  1. scan_skills() → 扫描 workspace/drift/skills/ 下的 SKILL.md
  2. 过滤 requires_mcp 未满足的 skill
  3. 构建 system prompt（注入长期记忆 + RECENT_CONTEXT + skill 列表 + 最近运行记录）
  4. Tool Loop（max 20 步）
       工具：read_file / write_file / edit_file
            recall_memory / web_fetch / web_search
            fetch_messages / search_messages / shell
            send_message（最多一次）/ finish_drift
            mount_server（可挂载 MCP server）
  5. 强制落地机制：
      step N-3 → 注入警告提示
      step N-2 → 限制 schema 为 write/edit，强制写文件
      step N-1 → 强制调用 finish_drift
```

**核心约束**：
- 每次进入重新比较所有 skill，不默认继续上次的
- `send_message` 成功后只允许写文件和 finish_drift
- 发出的消息要像自然聊天，不像在汇报内部执行流程

**面试话术**：
> Drift 是 Agent 在空闲时间的自主行为引擎。与被动回复和主动推送不同，Drift 让 agent 在没有外部事件时也能产生价值——比如整理笔记、补充记忆、搜索感兴趣的话题。关键是约束设计：send_message 限一次、最后三步强制落地、消息风格要求自然。这体现了 Agent 自主性的三层模型：被动、主动、自驱。

---

### 3.10 SubAgent（受管子代理）

**是什么**：将长时间、独立的任务 spawn 到后台子 Agent 执行，完成/取消后结果回灌到当前会话。

**在本项目中的实现**：`SubAgent`（`agent/subagent.py`）+ `spawn` 工具

```python
agent = SubAgent(
    provider=provider,
    model="deepseek-chat",
    tools=[WebSearchTool(), WebFetchTool()],
    system_prompt="你是后台研究助手...",
    max_iterations=30,
)
result = await agent.run("调研最新的 agent 相关论文")
```

**关键设计点**：
- 独立工具集，不共享主 Agent 的工具
- 有界循环（max_iterations + tool loop 循环检测 + 强制收尾）
- 长工具结果截断（>100K 字符时截断）
- 旧轮次 tool result 替换为占位符（防止长对话撑爆上下文）
- `mandatory_exit_tools`：预算耗尽时强制调用收尾工具
- 与主 AgentLoop 的区别：无 session/memory 写入、无对话历史持久化

**面试话术**：
> SubAgent 是典型的 multi-agent delegation 模式。主 Agent 通过 spawn 工具将独立子任务委派给后台 SubAgent，SubAgent 有自己的工具集和 LLM 循环。完成或被取消后，结果通过 SpawnCompletionItem 事件回灌到主 Agent 的会话中。核心工程挑战是上下文管理——长工具结果截断、旧轮次结果占位符替换、循环检测、强制收尾机制。

---

### 3.11 Prompt 工程（System Prompt Assembly）

**是什么**：将 system prompt 拆为多个可缓存 section，结合动态 context frame 注入记忆和上下文。

**在本项目中的实现**：`PromptAssembler`（`agent/prompting/assembler.py`）

```
System Prompt 组装：
  ├─ 【静态】sections（persona / skills / rules / tools）→ 可缓存
  ├─ 【动态】context frame（伪装成 user message）
  │    ├─ active_skills
  │    ├─ recent_context
  │    └─ retrieved_memory
  └─ 【动态】turn injection（deferred tools hint / tool state）
```

**关键设计点**：
- Context frame 伪装成 `<system-reminder>` 包裹的 user message
- 静态 sections 可通过 KV-cache 复用（需模型支持 prompt caching）
- 动态 sections 和 context frame 标记 `is_static=False`
- `SectionCache`：按 scope + section_name + signature 缓存 prompt 片段

**面试话术**：
> Prompt 组装的核心思想是将静态内容和动态内容分离。静态部分（persona、skill 描述、工具规则）内容稳定，可通过 Anthropic/DeepSeek 的 prompt caching 机制复用 KV-cache。动态部分（检索记忆、近期上下文）打包成 system-reminder 伪装消息注入。这样在长对话中每次 turn 只需要重新计算动态部分的 tokens。

---

### 3.12 Streaming + Event-Driven 输出

**是什么**：LLM 流式输出不再直接写入 channel，而是通过 EventBus 发射 `StreamDeltaReady` 事件，各频道自订阅消费。

**在本项目中的实现**（`agent/looping/core.py:206-233`）：

```python
async def _push(delta: StreamDelta):
    # 1. 拼装 Delta
    # 2. 更新 TurnInterruptState（中断续跑用）
    # 3. 发射 StreamDeltaReady 事件
    await self._event_bus.observe(
        StreamDeltaReady(
            session_key=session_key,
            content_delta=content_delta,
            thinking_delta=thinking_delta,
        )
    )
```

**优势**：
- 频道解耦：新增频道只需订阅事件，不修改 Agent 核心代码
- 可观察性：streaming 成为可被其他插件监听的事件

**面试话术**：
> Streaming 输出从传统的 callback 模式重构为事件驱动模式。Agent 将每个 delta 作为 StreamDeltaReady 事件发射到 EventBus，各频道 adapter 自行订阅消费。这样频道层和 Agent 核心完全解耦，新增通讯渠道不需要改动 Agent 代码。

---

## 四、项目技能点对照表（面试复习清单）

| 技能领域 | 对应技术 | 项目中的体现 |
|----------|---------|-------------|
| **Agent 架构** | ReAct Loop、Multi-Agent | Reasoner (tool loop) + SubAgent (spawn) |
| **Function Calling** | Tool Use / OpenAI tool format | ToolRegistry + ToolSearch deferred loading |
| **MCP 协议** | Model Context Protocol (JSON-RPC over stdio) | McpClient 完整实现 |
| **RAG** | 检索增强生成 | Memory v2: embedding + keyword + RRF |
| **向量搜索** | sqlite-vec 本地向量数据库 | vector_search / vector_search_batch |
| **HyDE** | Hypothetical Document Embedding | HyDEEnhancer (light model 生成假设答案) |
| **Prompt Engineering** | Prompt Caching + Context Frame | PromptAssembler + SectionCache |
| **插件系统** | Plugin Architecture / 装饰器 | PluginManager + @tool / @on_before_turn / @on_tool_pre |
| **事件驱动** | EventBus / Observer Pattern | emit（GATE）/ fanout（TAP）/ enqueue（后台） |
| **管道架构** | Chain of Responsibility | Phase + PhaseModule（lifecycle pipeline） |
| **流式输出** | SSE / Streaming | event-driven streaming via EventBus |
| **长期记忆** | Memory Consolidation | post_response_worker + ConsolidationService |
| **自主 Agent** | Proactive / Autonomous | Proactive (data gateway + agent loop) + Drift |
| **异步编程** | Python asyncio | 全链路 async/await |
| **并发控制** | 多会话管理 + busy 锁 | AgentLoop._active_tasks + ProcessingState |
| **错误处理** | Retry + Graceful Degradation | trim plan retry / safety retry / timeout fallback |
| **协议设计** | JSON-RPC 2.0 | MCP client / server 交互 |

---

## 五、建议学习路线

### 第一阶段：理解 Agent 基础（1-2 天）
1. 阅读 `main.py` 理解启动流程和数据流
2. 阅读 `agent/core/passive_turn.py` 理解 5 阶段生命周期管道
3. 阅读 `agent/lifecycle/phase.py` 理解 Phase 抽象
4. 运行起来，发一条消息，跟随日志走通完整流程

### 第二阶段：掌握核心技术栈（3-5 天）
1. **Tool Calling**：`agent/tools/registry.py` + `agent/tools/tool_search.py`
2. **MCP**：`agent/mcp/client.py`，理解 JSON-RPC 握手和工具调用
3. **EventBus**：`bus/event_bus.py`，理解 emit/fanout/enqueue 三种模式
4. **Memory**：`memory2/retriever.py`，理解 vector + keyword + RRF 检索链
5. **Plugin**：`agent/plugins/manager.py` + `agent/plugins/decorators.py`

### 第三阶段：理解高级特性（3-5 天）
1. **Proactive**：`proactive_v2/gateway.py` + `proactive_v2/agent_tick.py`
2. **Drift**：`proactive_v2/drift_runner.py` + `proactive_v2/drift_tools.py`
3. **SubAgent**：`agent/subagent.py` + `agent/background/subagent_manager.py`
4. **Prompt Assembly**：`agent/prompting/assembler.py` + `agent/context.py`

### 第四阶段：面试准备（2-3 天）
1. 能画出 Agent 的完整架构图（生命周期管道 + ReAct 循环）
2. 能说清楚每个核心技术的 **定义 → 作用 → 在本项目中的实现 → 面试话术**
3. 准备 3-5 个有深度的问题（比如「如果让你重新设计插件系统，你会怎么改进？」）
4. 准备 2-3 个自己踩过的坑或做的改进（可以从 git log 中找最近提交）

---

## 六、高频面试问题速答

**Q: 你们的 Agent 是怎么工作的？**
> 三条链路。被动回复走 5 阶段生命周期管道（BeforeTurn → BeforeReasoning → Reasoner 含内部 BeforeStep/AfterStep → AfterReasoning → AfterTurn），核心是 ReAct 循环（LLM 推理 + 工具调用迭代）。主动推送走定时 tick，DataGateway 先并行预取数据源，Agent 自主决策是否推送。Drift 模式在没有推送内容时利用空闲时间自主执行 skill 任务。

**Q: 工具系统是怎么设计的？**
> 工具按来源分 builtin/MCP/plugin 三类，统一注册到 ToolRegistry。采用 deferred loading——非 always-on 工具不暴露 schema，模型通过 tool_search 按需检索加载。工具执行走 ToolExecutor 的 pre/post hook 链，插件可拦截改参。

**Q: MCP 协议怎么实现的？**
> 启动 MCP server 子进程走 stdio JSON-RPC 2.0 通信。先 initialize 握手，再 tools/list 获取工具列表注册到 ToolRegistry。调用时走 tools/call。支持子进程工作目录推断、超时排障、stderr 后台 drain。

**Q: 记忆系统用的什么方案？**
> 本地 sqlite-vec 做向量存储。检索时 QueryRewriter 改写 query + 可选 HyDE 增强 + 向量/关键词双路检索 + RRF 融合排序。记忆有热度衰减权重，有分级阈值和字符预算控制。

**Q: 你们怎么做上下文管理？**
> Prompt 分静态/动态两部分，静态部分可走 prompt caching 复用 KV-cache。Context frame 伪装成 system-reminder 注入动态内容。上下文超长时有 trim plan retry 策略（逐步裁剪重试）。旧轮次 tool result 有占位符替换防止上下文膨胀。
