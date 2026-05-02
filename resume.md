# Akashic Agent — 简历 & 面试准备手册

> 面向 AI Agent / LLM 应用开发岗位的系统性技术文档，涵盖项目架构、关键技术原理与面试问答。

---

## 目录

1. [项目关键技术深度解析](#1-项目关键技术深度解析)
   - [MCP (Model Context Protocol)](#11-mcp-model-context-protocol)
   - [ReAct / Tool-Use Loop](#12-react--tool-use-loop)
   - [RAG 记忆与检索增强生成](#13-rag-记忆与检索增强生成)
   - [Tool Calling 工具调用体系](#14-tool-calling-工具调用体系)
   - [Skills 技能系统](#15-skills-技能系统)
   - [上下文窗口管理](#16-上下文窗口管理)
   - [主动推送与漂移检测](#17-主动推送与漂移检测)
2. [项目架构与消息链路](#2-项目架构与消息链路)
3. [简历项目亮点](#3-简历项目亮点)
4. [面试八股文问答（30题）](#4-面试八股文问答)

---

## 1. 项目关键技术深度解析

### 1.1 MCP (Model Context Protocol)

#### 什么是 MCP
MCP 是 Anthropic 在 2024 年底发布的开放协议，用于标准化 AI 模型与外部工具/数据源之间的通信。它解决了 AI Agent 生态中"N 个模型 × M 个工具 = N×M 种集成方式"的碎片化问题。

**核心架构：**
```
Host (IDE / Agent App)
  └─ MCP Client (协议客户端)
       ├─ stdio transport ── MCP Server A (本地进程)
       ├─ HTTP transport ── MCP Server B (远程服务)
       └─ stdio transport ── MCP Server C (本地进程)
```

**协议分层：**
- **传输层**：stdio (子进程 JSON-RPC) 或 Streamable HTTP (SSE)
- **消息层**：JSON-RPC 2.0 — `initialize`, `tools/list`, `tools/call`, `resources/read`, `prompts/get`
- **能力层**：Tools（工具调用）、Resources（资源读取）、Prompts（提示模板）、Sampling（服务端发起 LLM 请求）

#### 本项目中的 MCP 实现

**三层 MCP 架构：**

| 层级 | 文件 | 职责 |
|------|------|------|
| MCP Client | `agent/mcp/client.py:34-259` | stdio 子进程管理、JSON-RPC 编解码、超时/断连恢复 |
| MCP Registry | `agent/mcp/registry.py:15-131` | 多 server 生命周期管理、工具注册同步、配置持久化 |
| MCP Tool Wrapper | `agent/mcp/tool.py:9-33` | 将远端 MCP 工具包装为本地 `Tool` 子类 |

**连接流程 (`McpClient.connect()`):**
```
1. asyncio.create_subprocess_exec(*command, cwd=cwd)
2. → initialize (协议版本协商)
3. → notifications/initialized
4. → tools/list (获取远端工具列表)
5. → 每个工具注册为 McpToolWrapper(mcp_{server}__{tool_name})
```

**工具名生成规则 (`agent/mcp/tool.py:21-22`)：**
```python
f"mcp_{server_name}__{tool_name}"
# 例: mcp_markitdown__convert_to_markdown
```

**关键工程细节：**
- **cwd 推断** (`client.py:25-31`)：未指定 `cwd` 时从 command 中第一个绝对路径文件推断父目录
- **超时机制** (`client.py:181-198`)：默认 30s 等待超时，支持 per-call 覆盖
- **断连重连** (`client.py:159-189`)：`McpClientPool` 层实现了 call 失败自动重连一次
- **per-server 锁** (`mcp_sources.py:102`)：MCP stdio 不支持并发调用，每个 server 串行化

**MCP 服务器配置 (`~/.akashic/workspace/mcp_servers.json`)：**
```json
{
  "servers": {
    "markitdown": {
      "command": ["uv", "run", "mcp", "run", "main.py"],
      "cwd": "/home/zmr/.akashic/markitdown-mcp"
    }
  }
}
```

**MCP 内容源配置 (`~/.akashic/workspace/proactive_sources.json`)：**
```json
{
  "sources": [{
    "server": "rss-digest-bot",
    "channel": "content",
    "get_tool": "get_proactive_events",
    "ack_tool": "acknowledge_events",
    "poll_tool": "poll_feeds"
  }]
}
```

#### 面试速记要点
- MCP 本质：AI 工具调用的 **USB-C 协议**——一次实现，到处使用
- 传输方式：stdio（子进程）和 HTTP（远程），本项目两者都支持
- 与 Function Calling 的关系：Function Calling 定义工具的 schema 格式，MCP 定义工具的**发现、连接、生命周期管理**协议
- 容错：JSON-RPC 错误码、超时重连、per-server 锁保证 stdio 串行安全

---

### 1.2 ReAct / Tool-Use Loop

#### 什么是 ReAct
ReAct = **Reasoning（推理）+ Acting（行动）**，是 Google 2023 年提出的 LLM Agent 范式。核心思想：让模型在同一个循环中交替进行"思考下一步做什么"和"实际调用工具获取信息"，直到信息足够给出最终回答。

#### 为什么需要 ReAct
传统 LLM：用户提问 → 模型直接回答 → 知识截止、幻觉
ReAct：用户提问 → 模型决定需要什么信息 → 调工具 → 拿到结果 → 再思考 → 可能再调工具 → 最终回答

#### 本项目核心实现

**两层结构：**

```
run_turn() — 外层：上下文裁剪重试层
  └─ run() — 内层：核心 Tool-Use Loop
```

**`DefaultReasoner.run()` — 核心循环 (`agent/core/passive_turn.py:719-1155`)：**

```
for iteration in range(max_iterations):    # 默认 10 轮，可配置到 40
    ① BeforeStep — token 估算 + 插件注入
    ② LLM.chat(messages, tools) — 带工具 schema 调模型
    ③ if tool_calls:
         逐个执行工具 → append_tool_result → continue
    ④ else:
         return 最终回复
```

**四大防护机制：**

| 防护 | 实现位置 | 机制 |
|------|----------|------|
| 步骤预算 | `:755` | `max_iterations` 硬上限（默认 10），超限触发 LLM 收尾总结 |
| 重复调用检测 | `:814-845` | 工具名+参数签名去重，连续 3 次相同签名 → 强制收尾 |
| 空回复重试 | `:1070-1097` | 模型只输出 thinking 无 content → 注入 "请直接回复" 后重试 |
| 工具未解锁引导 | `:858-902` | 模型调了不可见的 deferred 工具 → 返回指引让其先调 `tool_search` |

**签名去重原理 (`agent/tool_runtime.py:45-52`)：**
```python
def tool_call_signature(tool_calls):
    # 排除 task_output/task_stop（它们每次参数不同但不算重复）
    # 对其他工具：name + sorted(json(args)) → hash
    # 同一签名连续 ≥3 次 → 判定为循环
```

#### 面试速记要点
- ReAct = 推理和行动交替进行，是 Agent 引擎的核心
- 三个关键防护：步骤预算（max_iterations）、重复检测（签名去重）、空回复重试
- 外层 `run_turn()` 做上下文裁剪重试：按 (1.0, 0.5, 0.0) 比例降级
- 整个 passive turn 经历 6 个阶段：BeforeTurn → BeforeReasoning → Reasoner.run_turn() → AfterReasoning → AfterTurn → dispatch

---

### 1.3 RAG 记忆与检索增强生成

#### 什么是 RAG
RAG = Retrieval-Augmented Generation。在 LLM 生成回复前，先从外部知识库检索相关信息，注入到上下文中，使模型能基于"已知事实"而非仅靠训练记忆来回答。

**传统 RAG vs Agent 记忆 RAG：**
- 传统 RAG：文档库 → 向量检索 → 注入 prompt
- Agent 记忆 RAG：多类型记忆（偏好/事件/流程/画像）→ 多策略检索（向量/关键词/路由）→ 评分过滤 → 注入 prompt

#### 本项目记忆系统架构

**双层记忆引擎：**

| 组件 | 位置 | 职责 |
|------|------|------|
| `agent/retrieval/` | 检索管线抽象层 | 定义 `MemoryRetrievalPipeline` 协议 |
| `memory2/` | 记忆引擎 v2 | 嵌入、检索、评分、去重、注入全流程 |

**四种记忆类型：**

| 类型 | 含义 | 检索阈值 | 示例 |
|------|------|----------|------|
| `preference` | 用户偏好 | 0.50 | "用户喜欢简洁回答" |
| `event` | 事件记录 | 0.50 | "2026-01-15 用户提到在找工作" |
| `profile` | 用户画像 | 0.50 | "用户是 Python 后端开发" |
| `procedure` | 操作流程 | 0.66 | "部署到服务器时需先跑测试" |

**检索管线 (`agent/retrieval/default_pipeline.py`)：**

```
用户消息
  → QueryRewriter (查询重写/扩展)
  → HyDEEnhancer (假设文档嵌入增强，可选)
  → Embedder (文本转向量)
  → Retriever (向量检索 + 阈值过滤 + 去重)
  → InjectionPlanner (注入规划，控制长度)
  → 结果注入到 Context Builder 的 system prompt 中
```

**关键设计：**
- **HyDE (Hypothetical Document Embeddings)**：用 LLM 先生成"假设的理想答案"，对假设答案做向量检索。原理：假设答案在语义空间与真实记忆更接近，提升召回率
- **SufficiencyChecker**：判断检索结果是否足够回答当前问题，不足时触发补充检索
- **去重判断 (DedupDecider)**：检索到的记忆可能与对话历史重复，需要过滤
- **SQLite + sqlite-vec**：向量存储使用 `sqlite-vec` 扩展，零运维、嵌入式向量数据库

#### 面试速记要点
- RAG 的核心问题不是"怎么检索"，而是"检索什么"和"注入多少"
- HyDE：用 LLM 生成假设答案 → 对假设答案做向量检索 → 召回率更高
- 多类型记忆各有不同的检索阈值和注入配额，避免单一类型记忆挤占上下文
- sqlite-vec：嵌入式向量数据库，无需部署外部服务，适合单机 Agent

---

### 1.4 Tool Calling 工具调用体系

#### 什么是 Tool Calling / Function Calling
LLM 不仅能生成文本，还能输出结构化的函数调用请求。Agent 框架拦截这些请求，执行真实的函数/API，把结果返还给 LLM。这是 Agent 从"聊天机器人"升级为"能行动的智能体"的关键能力。

**与 MCP 的关系：**
- Function Calling = OpenAI 定义的工具调用 schema 格式
- MCP = 工具**发现、连接、生命周期管理**的协议标准
- 两者互补：MCP 负责"有哪些工具可用"，Function Calling 负责"模型如何请求调用工具"

#### 本项目工具调用体系

**工具生命周期：**

```
定义 (Tool ABC)
  → 注册 (ToolRegistry.register)
    → 发现 (tool_search / LRU preload)
      → LLM 请求 (tool_calls in response)
        → Hook 链 (ToolExecutor: pre-hooks → execute → post-hooks)
          → 结果追加 (append_tool_result)
```

**工具基类 (`agent/tools/base.py:26-138`)：**
- 抽象属性：`name`, `description`, `parameters` (JSON Schema)
- 抽象方法：`execute(**kwargs) → str | ToolResult`
- `to_schema()`：转 OpenAI Function Calling 格式

**ToolRegistry (`agent/tools/registry.py:115-297`)：**
- `register(tool, risk, always_on, search_hint)`：按风险等级分类
- `get_schemas(names)`：返回当前可见工具的 OpenAI schema
- `execute(name, arguments)`：执行工具并合并 session 上下文
- `search(query, top_k)`：关键词搜索工具

**ToolExecutor Hook 管道 (`agent/tool_hooks/executor.py`)：**
```
Pre-hooks (可修改参数、拒绝调用)
  → 真实工具执行
    → Post-hooks (观测、追踪、日志)
```

**动态工具发现 (Tool Discovery)：**
- 非 `always_on` 工具默认对 LLM 不可见（节省 token）
- LLM 需先调 `tool_search(query="select:...")` 解锁
- LRU 容量限制：每 session 最多 5 个已解锁工具
- 引用计数 + 自动回收

#### 面试速记要点
- Function Calling 是工具调用的 API 格式，Tool Registry 是工具的注册中心
- Hook 管道 = 责任链模式：pre-hooks 可修改参数/拒绝，post-hooks 做观测
- 动态发现 = 解决 "500 个 tool schema 撑爆上下文" 的问题
- LRU 驱逐 = 保证每 session 工具可见集合不会无限膨胀

---

### 1.5 Skills 技能系统

#### 什么是 Skills
Skills 是比 Tool 更高层次的抽象——一个 Skill 包含**多个工具调用序列**、**特定领域的 prompt 模板**、**验证流程**。类似于"给 Agent 装一个专业 App"。

#### 本项目 Skills

| Skill | 位置 | 用途 |
|-------|------|------|
| `create-proactive-source` | `skills/create-proactive-source/` | 引导创建新的主动推送数据源 |
| `skill-creater` | `skills/skill-creater/` | 创建新 Skill 的元 Skill |
| `summarize` | `skills/summarize/` | 对话摘要 |
| `weather` | `skills/weather/` | 天气查询模板 |
| `feed-manage` | `skills/feed-manage/` | RSS 源管理 |
| `meme-manage` | `skills/meme-manage/` | 表情包管理 |
| `create-drift-skill` | `skills/create-drift-skill/` | 创建漂移检测 Skill |

#### 面试速记要点
- Tool ≈ 一个函数调用，Skill ≈ 一个专业 App
- Skill 封装了 "什么时候用 → 怎么用 → 怎么验证结果" 的完整流程
- 区分：Skill 是给用户的，Plugin（生命周期钩子）是给开发者的

---

### 1.6 上下文窗口管理

#### 为什么重要
LLM 有上下文窗口限制（如 DeepSeek 64K tokens），每轮工具调用都把结果追加到消息列表。不加管理会在几轮内超出窗口。

#### 本项目三层策略

| 层级 | 位置 | 机制 |
|------|------|------|
| 内存窗口 | `agent/config_models.py:105` | `memory_window = 24` —— 只保留最近 24 轮消息 |
| 上下文裁剪 | `passive_turn.py:584-649` | 按 (1.0, 0.5, 0.0) 比例降级重试 |
| 段落禁用 | `passive_turn.py:612` | 超长时可禁用某些 context 段落 |
| Token 估算 | `passive_turn.py:785` | 每轮 BeforeStep 估算 input tokens |

#### 面试速记要点
- 上下文管理 = 记忆窗口 + 降级重试 + 段落裁剪，三层防御
- 关键权衡：保留更多历史（更好的连贯性）vs 节省 token（更低的延迟和成本）

---

### 1.7 主动推送与漂移检测

#### 架构

```
定时 tick (每 60s)
  → DataGateway.fetch() — 从 MCP 源拉取 alerts/context/content
    → Content 评分 (HyDE 兴趣匹配 + 去重)
      → Agent 决策循环
        → DeliveryGate (冷却检查)
          → PushTool → QQ/Telegram
```

**关键配置 (`config.toml:81-101`)：**
- `content_limit = 5`：每次最多推送 5 条
- `delivery_cooldown_hours = 1`：同一用户 1 小时内不重复推送
- `context_prob = 0.03`：3% 概率注入背景上下文

**漂移检测 (Drift Detection)：**
- 检测用户兴趣是否发生变化
- `min_interval_hours = 3`：最小间隔 3 小时
- 漂移时触发一次性对话确认

---

## 2. 项目架构与消息链路

### 2.1 总体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        消息通道层                                │
│  QQ ──→ NcatBot SDK ──→ MessageBus                              │
│  Telegram ──→ python-telegram-bot ──→ MessageBus                │
│  CLI/TUI ──→ Unix Socket ──→ MessageBus                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │ InboundMessage
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     AgentLoop (core.py)                          │
│                                                                  │
│  MessageBus.consume_inbound()                                    │
│    → AgentCore.process()                                        │
│      → PassiveTurnPipeline                                      │
│        ┌─────────────────────────────────────────────────────┐  │
│        │ Phase 1: BeforeTurn                                  │  │
│        │   - session 加载 + 上下文准备                         │  │
│        │   - 记忆检索 & 注入                                   │  │
│        │ Phase 2: BeforeReasoning                             │  │
│        │   - 工具同步 + prompt 预热                            │  │
│        │ Phase 3: Reasoner.run_turn()                         │  │
│        │   - 上下文裁剪重试 (1.0→0.5→0.0)                      │  │
│        │   - Reasoner.run() — 核心 Tool-Use Loop              │  │
│        │ Phase 4: AfterReasoning                              │  │
│        │   - 结果解析 + 持久化                                  │  │
│        │ Phase 5: AfterTurn                                   │  │
│        │   - 记忆更新 + 事件分发                                │  │
│        └─────────────────────────────────────────────────────┘  │
│    → OutboundMessage → MessageBus.publish_outbound()             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ OutboundMessage
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        消息通道层 (出站)                          │
│  QQ ←── NcatBot API ←── BusOutboundPort                          │
│  Telegram ←── Bot.send_message ←── BusOutboundPort               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 中间件协作

#### NapCat + NcatBot
- **NapCat**：QQ Bot 的 Docker 容器化方案，基于 NTQQ 协议
- **docker-compose.yml**：`mlikiowa/napcat-docker:v4.8.119`，host 网络模式
- **NcatBot**：Python SDK (`ncatbot>=4.4.0`)，封装 NapCat 的 HTTP API
- **消息流**：QQ 消息 → NapCat 接收 → NcatBot 回调 → 投递到 asyncio 主循环 → MessageBus

#### RSSHub
- **作用**：通用 RSS 生成器，将任意网站转为 RSS 订阅
- **部署**：`docker/rsshub/docker-compose.yml`，含 Redis 缓存 + 浏览器渲染
- **与 Agent 关系**：Agent 的 RSS MCP Bot 可以直接订阅 RSSHub 生成的 feed

#### Docker 服务拓扑

```
docker-compose (两组)
├── napcat/
│   └── napcat — QQ Bot 容器 (host 网络)
└── rsshub/
    ├── rsshub — RSS 生成服务 (:1200)
    ├── redis — RSSHub 缓存
    ├── browserless — 无头浏览器 (JS 渲染页面的 RSS)
    └── real-browser — 真实浏览器 (反爬页面)
```

### 2.3 技术栈总览

| 层级 | 技术 | 用途 |
|------|------|------|
| LLM 调用 | OpenAI SDK (`openai>=2.0`), Anthropic SDK | 多 Provider 统一接口 |
| Web 框架 | FastAPI + Uvicorn | Dashboard API |
| 异步引擎 | asyncio (Python 原生) | 全异步架构 |
| 消息通道 | NcatBot, python-telegram-bot, Unix Socket | 多端接入 |
| 向量存储 | sqlite-vec | 嵌入式向量检索 |
| 数据库 | SQLAlchemy + SQLite | 结构化数据持久化 |
| 任务调度 | APScheduler | 定时 tick |
| MCP 协议 | mcp[cli] + 自研 McpClient | 工具生态 |
| 前端 TUI | Textual + Rich | CLI 终端界面 |
| 内容抓取 | curl_cffi, beautifulsoup4, lxml | 网页抓取和解析 |
| Docker | docker-compose × 2 | NapCat + RSSHub 容器化 |
| 验证 | Pydantic v2 | 全项目配置和类型验证 |

---

## 3. 简历项目亮点

### 建议写法

> **Akashic Agent — 多端智能助手系统** (个人项目)
>
> 从零构建的全栈 LLM Agent，支持 QQ/Telegram/CLI 多端接入。核心包括：自研 ReAct Tool-Use Loop 引擎、MCP 协议集成（工具热插拔 + 主动推送）、双层记忆 RAG 系统（HyDE 增强 + 多类型记忆）、Docker 容器化的 QQ Bot 方案。
>
> **技术栈：** Python 3.12, asyncio, OpenAI/Anthropic SDK, MCP Protocol, FastAPI, Docker, sqlite-vec, NcatBot, RSSHub, Pydantic v2

**项目亮点（4-5 条）：**

1. **自研 ReAct Agent 引擎**：实现有界多轮 Tool-Use Loop，含步骤预算控制、工具调用签名去重防死循环、空回复自动重试、上下文窗口三级降级裁剪（1.0→0.5→0.0），单轮最多 40 步迭代

2. **完整 MCP 协议集成**：自研 McpClient（stdio JSON-RPC 通信 + 超时重连 + cwd 推断）+ McpServerRegistry（多 server 生命周期 + 工具自动注册）+ McpToolWrapper（远端工具透明代理），支持动态工具发现与 LRU 驱逐

3. **双层记忆 RAG 系统**：HyDE 假设文档嵌入增强召回、四种记忆类型（偏好/事件/画像/流程）独立阈值过滤、QueryRewriter 查询重写、SufficiencyChecker 检索充分性判断；基于 sqlite-vec 的嵌入式向量存储

4. **多端消息通道 + 主动推送**：QQ（NapCat Docker 容器化 + NcatBot SDK）、Telegram 双通道接入；定时 tick 驱动的内容抓取 → AI 评分 → 冷却过滤 → 主动推送管线

5. **全异步 + 可观测**：Python asyncio 全链路非阻塞架构，EventBus 事件驱动，完整的 turn trace（每次对话的 token 消耗、缓存命中率、工具调用链、迭代次数）

---

## 4. 面试八股文问答

### MCP 相关

**Q1: 什么是 MCP？和传统的 Function Calling 有什么区别？**

MCP（Model Context Protocol）是 Anthropic 发布的 AI 工具集成标准协议。它解决的核心问题是碎片化——以前每个 AI 应用都要单独对接每个工具，形成 N×M 的集成矩阵。

**和 Function Calling 的关系：** Function Calling 定义了工具调用的数据格式（name/description/parameters 的 JSON Schema），MCP 在此基础上定义了工具的生命周期管理——如何发现工具（tools/list）、如何建立连接（initialize 握手）、如何处理传输（stdio/HTTP）。一句话：Function Calling 管"调用的格式"，MCP 管"工具的连接和发现"。

**MCP 的核心概念：** Client-Server 架构，Client 通过 stdio 或 HTTP 连接 Server，Server 暴露 Tools（可调用函数）、Resources（可读取数据）、Prompts（预定义提示模板）三种能力。

---

**Q2: MCP 的传输方式有哪些？各有什么优缺点？**

两种传输方式：

| | stdio | Streamable HTTP |
|---|---|---|
| 原理 | 子进程 stdin/stdout JSON-RPC | HTTP + SSE 流 |
| 延迟 | 极低（本地进程） | 较高（网络往返） |
| 部署 | 需本地安装运行环境 | 可远程部署 |
| 并发 | 不支持（串行） | 支持 |

**实际选择：** 本地工具（如 markitdown 文件转换）用 stdio，需要共享的服务（如公司内部的数据库查询）用 HTTP。

**本项目使用：** 全部使用 stdio 传输，通过 per-server asyncio.Lock 保证串行安全。

---

**Q3: 你如何处理 MCP 连接失败和断连？**

三层容错：
1. **连接超时**：`McpClient.connect()` 的 initialize 阶段有 30s 超时
2. **调用超时**：`McpPool.call()` 支持 per-call 超时覆盖，默认 30s
3. **自动重连**：调用失败时 `McpPool` 自动 disconnect + reconnect + retry 一次

工具名包含中文字符的问题也踩过坑——DeepSeek API 要求 tool name 匹配 `^[a-zA-Z0-9_-]+$`，所以 MCP server 注册名必须用纯 ASCII。

---

### Tool Calling / ReAct 相关

**Q4: 描述一下 ReAct 循环的工作流程？**

ReAct 是 Reasoning（推理）+ Acting（行动）的缩写。核心是一个有界循环：

```
for iteration in range(max_iterations):
    response = LLM.chat(messages, tools)
    if response.tool_calls:
        for each tool_call:
            result = execute(tool_call)
            messages.append(result)
        continue    # 让模型基于工具结果继续思考
    else:
        return response.content  # 模型认为信息足够
```

在工程实现上需要四层防护：
- **max_iterations** 上限（否则死循环）
- **重复调用检测**（工具名+参数签名连续相同 → 打断）
- **空回复重试**（模型只有 thinking 没有 content 时重新提示）
- **上下文裁剪**（token 超限时按比例裁减历史）

---

**Q5: 工具调用循环中可能出现什么工程问题？你怎么解决的？**

四个核心问题：

1. **死循环**：模型反复调同一个工具。解决：工具名+参数序列化签名去重，连续 3 次相同即打断，触发 LLM 收尾总结。

2. **上下文溢出**：每轮工具调用都追加消息。解决：memory_window 容量限制 + 外层三级降级（1.0→0.5→0.0 历史比例）+ 段落禁用。

3. **空回复**：支持 thinking 的模型有时只输出推理过程没有正文。解决：检测到 thinking-only 响应时注入 "请直接回复" prompt，去工具（tools=[]）重试一次。

4. **工具 schema 过多**：几百个工具的 schema 全部发给 LLM 会撑爆上下文。解决：动态工具发现——默认只加载高频工具，LLM 需通过 tool_search 按需解锁，LRU 驱逐保证不超过容量。

---

**Q6: 工具调用签名去重是怎么实现的？**

`tool_call_signature()` 对每次工具调用计算签名：排除 `task_output` 和 `task_stop`（它们本身就是"结束"信号），对剩余调用取 `tool_name:json.dumps(args, sort_keys=True)`，多个调用用 `|` 拼接。同一签名连续出现 3 次（`_TOOL_LOOP_REPEAT_LIMIT = 3`）判定为陷入循环，强制中断并让 LLM 总结当前进展。

---

### RAG / 记忆相关

**Q7: RAG 的基本原理是什么？为什么 LLM 需要 RAG？**

RAG = Retrieval-Augmented Generation，在 LLM 生成回复前先从外部知识库检索相关信息注入到 prompt 中。

**为什么需要：** LLM 有两个根本限制——知识截止（训练数据有截止日期）和幻觉（对于不知道的事会编造）。RAG 通过引入外部实时信息来解决这两个问题。

**Agent 场景下的 RAG 更复杂：** 不是简单地搜文档，而是需要检索多种类型的"记忆"——用户偏好、历史事件、个人画像、操作流程。不同类型的记忆需要不同的检索策略和注入方式。

---

**Q8: 什么是 HyDE？你们项目是怎么用的？**

HyDE = Hypothetical Document Embeddings（假设文档嵌入）。传统向量检索是 "用户问题 → 向量 → 与知识库向量匹配"，问题在于用户问题和知识库文档处于不同的语义空间。

HyDE 的做法：先让 LLM 根据用户问题生成一个"假设的理想答案"，然后对这个假设答案做向量检索。假设答案在语义上更接近真实的知识库文档，因此召回率更高。

**代价：** 多了一次 LLM 调用（生成假设答案），增加了延迟。所以项目中 HyDE 是可选的（`hyde.enabled = true`），且配置了 `timeout_ms = 2000` 防止超时影响用户体验。

---

**Q9: 你们项目的记忆系统支持哪些类型？各自有什么特点？**

四种类型：preference（偏好）、event（事件）、profile（画像）、procedure（流程）。各自有不同的检索阈值和注入配额。

**区分理由：** procedure 类记忆（如部署步骤）要求精确匹配，因此阈值设得较高（0.66）。preference 类记忆（如用户喜欢简洁回答）几乎每次都需要检索，阈值较低（0.50）。

**注入配额：** 通过 `inject_max_chars` 和 `inject_line_max` 控制总注入量，避免单类记忆挤占所有上下文空间。

---

**Q10: sqlite-vec 是什么？为什么选择它？**

sqlite-vec 是 SQLite 的向量搜索扩展，将向量存储嵌入到 SQLite 中，支持 ANN（近似最近邻）搜索。选择原因：零运维（无需部署独立的向量数据库如 Pinecone/Milvus），嵌入式部署，单机 Agent 场景完全够用。向量维度和搜索 top_k 都可以配置。

---

### Skills / LangChain / LangGraph 相关

**Q11: Skills 和 Tools 有什么区别？**

Tool 是一个原子操作（查天气、搜网页），输入 → 执行 → 输出。Skill 是一组 Tool + Prompt + 验证流程的封装——比如 "创建主动推送源" 这个 Skill 涉及：检查现有 MCP 服务器 → 确定 channel 类型 → 创建/扩展 MCP server → 注册到配置文件 → 运行验证测试。

**类比：** Tool ≈ Linux 的单个命令（`ls`），Skill ≈ 一个 Bash 脚本（组合了 `ls`、`grep`、`awk` 等多个命令完成一个完整任务）。

---

**Q12: 你用过 LangChain/LangGraph 吗？和自研 Agent 框架比有什么优劣？**

LangChain 的优势是开箱即用的组件（各种 Loader、Splitter、Chain），但它的问题也很明显：过度抽象（一个简单任务要理解 Chain→LLMChain→ConversationChain 多层继承）、调试困难（报错追踪穿过多层内部代码）、序列化复杂。

LangGraph 解决了 LangChain 的部分问题——用状态图（StateGraph）定义 Agent 流程，节点之间通过状态传递，天然支持循环和条件分支，适合复杂的多步骤 Agent。

**为什么自研：** 当理解了 ReAct 循环的本质后，你会发现核心引擎就是 100 行 Python——一个 for 循环 + LLM 调用 + 工具执行。自研的优势：完全可控（每个错误都能定位到具体行）、无依赖版本冲突（只依赖 openai sdk）、深入理解底层（这对 Agent 开发岗位面试是巨大加分）。

---

**Q13: 你们项目中有没有类似 LangGraph 的状态图概念？**

有，但更轻量。PassiveTurnPipeline 的 6 个 Phase（BeforeTurn → BeforeReasoning → Reasoner → AfterReasoning → AfterTurn → Dispatch）本质上就是一个线性 Pipeline，每个 Phase 是一个可插拔的模块链。BeforeStep/AfterStep 生命周期钩子则实现了循环内的状态管理。

这种方式比 LangGraph 更直观——没有图编译、没有 checkpoint、没有序列化，就是 Python 异步函数调用链。

---

### 记忆存储相关

**Q14: 你们的记忆如何持久化？跨 session 怎么恢复？**

记忆分两层存储：
- **结构化记忆**（类型、标签、时间戳等）：SQLAlchemy ORM → SQLite 数据库
- **向量嵌入**：sqlite-vec 扩展 → 存储在同一个 SQLite 文件中
- **ACK 状态**：JSON 文件（`~/.akashic/workspace/rss_digest_ack.json`）

跨 session 恢复：Agent 启动时 `Retriever` 自动加载向量索引，结构化记忆通过 SQLAlchemy session 按需查询。Session 本身也有独立的持久化——`SessionManager.save_async()` 将对话历史序列化到 `~/.akashic/workspace/sessions/`。

---

**Q15: 如何处理记忆和对话历史的上下文竞争？**

这是 RAG Agent 的核心难题——同时有对话历史（需要连贯性）和检索记忆（需要准确性），但上下文窗口有限。项目采用 "固定窗口（24 轮历史）+ 记忆注入上限（6000 字符）" 策略：先裁剪对话历史到 memory_window，再从剩余空间分配记忆注入。如果上下文仍然超限，外层的 trim plan 进一步降级（关闭某些段落、缩小历史窗口）。

---

### Docker / NapCat 相关

**Q16: Docker 在你的项目中扮演什么角色？**

两组 Docker Compose：
1. **NapCat (QQ Bot 容器)**：`mlikiowa/napcat-docker:v4.8.119`，使用 host 网络模式，挂载 QQ 配置和日志目录
2. **RSSHub (RSS 生成服务)**：含 4 个容器——rsshub（主服务）、redis（缓存）、browserless（无头浏览器渲染 JS 页面）、real-browser（真实浏览器处理反爬页面）

选型原因：NapCat 的运行环境依赖复杂（特定版本 Node.js、系统库），容器化一劳永逸；RSSHub 的大量插件需要浏览器环境，容器化隔离资源。

---

**Q17: NapCat 是什么？和 NcatBot 的关系？**

NapCat 是基于 NTQQ 协议的 QQ Bot 实现，本质上是一个无头 QQ 客户端，Docker 容器化运行。协议风险是核心注意事项——NTQQ 不是官方 API，有封号风险，所以项目使用小号。

NcatBot (`ncatbot>=4.4.0`) 是 NapCat 的 Python SDK，封装了消息收发、事件回调等 API。

**消息桥接要点：**
- NcatBot 的事件回调运行在独立线程，通过 `asyncio.run_coroutine_threadsafe()` 投递到主事件循环
- NcatBot 的 `run_backend()` 是同步阻塞调用，通过 `loop.run_in_executor()` 包裹执行
- 出站消息通过 MessageBus → BusOutboundPort → NcatBot API 发回 QQ

---

### 全栈 / 架构相关

**Q18: 描述你们的消息总线设计？**

MessageBus 是项目的消息中枢，基于 asyncio.Queue 实现生产者-消费者模式：

```
Channel (QQ/Telegram/CLI) → MessageBus.consume_inbound() → AgentLoop
AgentLoop → MessageBus.publish_outbound() → BusOutboundPort → Channel
```

关键设计：
- **解耦**：Channel 和 Agent 互不感知，通过 MessageBus 通信
- **单消费者**：inbound 队列只有一个消费者（AgentLoop），保证消息处理顺序
- **中断支持**：AgentLoop 持有 `_active_tasks` 字典，外部可通过取消 task 实现中断

---

**Q19: 如果让你优化这个项目，你会优先做什么？**

三个优先级排序：

1. **可观测性**：当前日志分散（print + logging），接入 OpenTelemetry trace，让每个 turn 的工具调用链、token 消耗、检索命中有可视化追踪
2. **工具热更新**：当前修改 MCP 配置需重启 Agent，可以加文件监听（watchfiles）实现热加载
3. **多 Agent 协作**：当前是单 Agent 模式，可以引入 Peer Agent 机制（项目已有 `PeerAgentConfig`），实现子任务委派和结果聚合

---

### 主动推送相关

**Q20: 主动推送和被动回复的核心区别是什么？怎么保证推送内容的相关性？**

**被动回复：** 用户主动发消息 → Agent 回复。**主动推送：** Agent 定时拉取内容 → 判断是否值得推送 → 主动发消息。

**相关性保证：** HyDE 兴趣评分（推送内容和用户历史兴趣的向量相似度）+ 去重机制（URL 去重 + 已推送记录）+ 推送冷却（同一用户 1 小时内不再推送）+ 用户可 ACK（确认已读，类似 "不再推送此类内容" 的隐式反馈）。

**工程难点：** MCP 数据源的可靠性（网络超时、格式错误、ACK 持久化），推送频率和用户打扰的平衡。

---

### 工程能力 / 问题排查相关

**Q21: 你在开发过程中遇到的最棘手的问题是什么？怎么解决的？**

最棘手的是 MCP 工具名含中文导致 DeepSeek API 400 错误。工具名生成规则 `mcp_{server_name}__{tool_name}` 中 server_name 是用户配置的，用户写了中文名 "高德地图Amap"。DeepSeek 的工具名校验正则是 `^[a-zA-Z0-9_-]+$`，中文直接 400。

排查过程：日志里没有直接提示 "工具名非法"，只有 400 Bad Request，需要对比 Amap 工具加载前后的 LLM 请求参数才能定位。解决：一是修复配置（server 名改为纯 ASCII），二是在 McpServerRegistry 中加工具名校验（虽然最终没加，但意识到了防御性编程的重要性）。

**另一个教训：** MCP market 上的第三方 MCP 服务器质量参差不齐——Amap 需要 API key，MarkItDown 远程版有 outputSchema bug。最终都选择了自己写本地 MCP server（30-100 行代码），完全可控。

---

**Q22: 你如何调试 Agent 的行为异常？**

三个层次：日志（结构化 logging 到 stdout + 文件）、turn trace（每次对话记录完整调用链，包括 token 消耗和缓存命中率）、dashboard API（运行时可查询 session 状态和工具列表）。

具体流程：先看 LLM 调用日志确认模型输出（是工具调用还是直接回复），再看工具执行日志确认结果（是成功还是错误），再看 AfterStep 日志确认本轮是否终止。

---

### 系统设计相关

**Q23: 如果要你的 Agent 支持 10000 个并发用户，你会怎么改造？**

瓶颈分析：当前单进程 AgentLoop 处理所有消息，TurnScheduler 控制并发（通过 `_active_tasks` 限制）。单进程模式对 10 人以下完全够用，10000 并发需要重构：

1. **Session 分片**：按 `session_key` hash 分片到多个 Worker 进程，每个 Worker 独立运行 AgentLoop
2. **消息队列**：引入 Redis/Kafka 替代内存 asyncio.Queue
3. **MCP 连接池**：当前每个 MCP server 一个 stdio 连接，多进程下需要 MCP proxy（让多个 Worker 通过 HTTP 共享 MCP 连接）
4. **记忆分离**：SQLite → PostgreSQL，sqlite-vec → pgvector
5. **LLM 限流**：单 API key 有 RPM（每分钟请求数）限制，需要多 key 轮询 + 令牌桶算法

---

**Q24: 上下文窗口只有 128K，如何处理需要阅读大量文档的任务？**

组合策略：
1. **增量读取**：先用 `web_fetch` 获取前 N 字符，判断是否需要继续
2. **分块摘要**：长文档分块 → 逐块 LLM 摘要 → 合并摘要再做决策
3. **工具卸载**：MarkItDown 把 HTML 转 Markdown（减少标签噪音），`web_search` 做结构化搜索
4. **中间结果落盘**：关键中间结果写入临时文件，后续步骤通过文件路径引用而非全文传递

---

### 综合项目理解

**Q25: 从头到尾描述一次被动对话的完整数据流。**

```
1. 用户在 QQ 发消息 "帮我查天气"
2. NapCat 接收到消息 → NcatBot 回调 → run_coroutine_threadsafe → MessageBus
3. AgentLoop.consume_inbound() → InboundMessage(channel=qq, chat_id=3076624956)
4. AgentLoop._process():
   a. 检查中断态 (resume 逻辑)
   b. 加载 session（从 SessionManager）
   c. 执行记忆检索（DefaultMemoryRetrievalPipeline）
   d. ContextBuilder.render() 组装 prompt
   e. PassiveTurnPipeline.run():
      - BeforeTurn: context prep + 记忆注入
      - BeforeReasoning: 工具同步
      - Reasoner.run_turn(): 上下文裁剪 → Reasoner.run():
        for iteration in range(40):
          BeforeStep (token 估算)
          LLM.chat(messages, tools) → response
          if tool_calls: 执行工具 → append_tool_result → continue
          else: 返回 reply
      - AfterReasoning: 解析 + 持久化
      - AfterTurn: 记忆更新 + 事件发布
   f. OutboundMessage → MessageBus.publish_outbound()
5. BusOutboundPort → NcatBot API → QQ 群/私聊
```

整个过程异步非阻塞，从接收消息到发出回复通常在 10-30 秒内完成（取决于工具调用次数）。

---

### 快速学习路径

如果你需要面试前快速掌握这个项目：

1. **第一优先级（必读）**：`agent/core/passive_turn.py:719-1155`（ReAct 核心循环）→ 理解 Agent 引擎怎么转的
2. **第二优先级**：`agent/mcp/client.py` + `agent/mcp/tool.py` → 理解 MCP 协议怎么接的
3. **第三优先级**：`agent/retrieval/default_pipeline.py` + `memory2/` 目录 → 理解记忆怎么存的怎么取的
4. **第四优先级**：`agent/looping/core.py:634-717`（`_process` 方法）→ 理解完整的消息处理链
5. **辅助理解**：`bootstrap/app.py`（启动和依赖注入）→ 理解所有组件怎么拼在一起的

---

*本文档基于 Akashic Agent 实际代码分析生成，覆盖面试中大概率被问到的核心技术点。建议结合源码阅读使用。*
