# 从 0 到 1 搭建工业级 Agent：源码剖析、实战与面试指南（学习大纲）

> 基于 `deepseek-harness` `0.2.0-rc.1`（commit `4878cdabd`）源码扫描整理。文中路径均相对仓库根目录，行号对应该 commit，后续版本可能漂移，以函数名为准。

## 如何使用本大纲

**目标**：以 deepseek-harness（下称 DSH）为样本，系统掌握工业级 Agent 的核心模块、设计取舍和选型依据，最后能在面试中讲清楚"为什么这样设计"。

**每章固定六个栏目**：

| 栏目 | 内容 |
|---|---|
| 学习目标 | 本章读完应能回答的问题 |
| 源码地图 | 关键文件、函数与行号，以及配套文档 |
| 设计考量与 Trade-offs | 这样设计的原因、优点和瓶颈 |
| 业界方案对比 | 与 LangGraph / CrewAI / AutoGen / LlamaIndex / OpenAI Agents SDK 等主流方案对照 |
| 面试高频深度题 | 2~3 道高频题及答题要点（逐章深入时展开为完整答案） |
| 动手实验 | 在本仓库中可直接完成的小实验 |

**阅读方法**（DSH 的文档体系很完整，建议按以下顺序读每个包）：

1. 先读包 README 的 `Summary` 和 `Use this package`，了解对外行为；
2. 再读 `src/`，对照源码地图中的函数；
3. 然后读 `tests/*.spec.ts`，测试用例就是行为规格；
4. 最后读 `.agents/notes/implemented/` 下的决策记录（ADR），这是面试中讲"为什么"的最佳素材。

**建议节奏**：12 章约 4 周。第 1、3、6、7 章是面试重点，需精读源码；其余章节读 README 和关键函数即可。

**前置知识**：TypeScript 的 async/await 与 `AbortSignal`；LLM Function Calling 的基本请求结构（`tool_call` / `tool_result` 配对）；第 0 章会补齐 Cordis 插件框架的基础。

## 全局地图

### 模块分层

```text
                ┌──────────── 入口 Profiles ────────────┐
                │ web · headless · sdk · acp · desktop   │
                └───────────────────┬────────────────────┘
                                    │ Cordis Loader：bundle + patch 分层组装插件树
┌───────────────────────────────────▼───────────────────────────────────────┐
│ ctx.agents（dsh-agent：Agent 接口/注册表/agent/* 事件）                       │
│      ▲ AgentFactory                                                        │
│ ctx.agentLoop（dsh-agent-loop：ReactLoopAgent，Turn/Step 驱动器）             │
│      │ assemble              │ prepareCall / stream     │ execute          │
│ ctx.systemPrompt          ctx.llm                    ctx.tools              │
│ （段落/上下文/工具 schema）  （适配器/流协议）          （执行管线/守卫）         │
│      └─────────────┬─────────┴─────────────┬───────────┘                    │
│            ctx.sessions：只追加事件日志 → deriveMessages() 派生模型历史         │
│            ctx.sessionPersistence（JSONL） · ctx.sessionProjections（读模型）  │
└────────────────────────────────────────────────────────────────────────────┘
  挂在事件上的横切插件：compaction · llm-retry · repeat-tool-reminder · timeout-policy
  user-approval · sandbox · hooks · checkpoint-policy · spill-policy · invariants · otel
```

### 一次 Turn 的完整流程（精简版，完整版见 [docs/architecture.md](docs/architecture.md#turn-flow)）

```text
turn/start
 └─ step N
     inbox.claim()                        领取 next-step 输入 + 至多一条排队 prompt
     systemPrompt.assemble()              组装段落、运行时上下文、工具 schema
     agent/pre-step      (waterfall)      压缩 / Hook / Goal / 重复检测复位；可 reject
     step/start
     agent/request       (waterfall)      决定 provider/model/参数
     llm.prepareCall()                    解析路由能力（上下文窗口、prompt 更新方式）
     追加 system/message、user/message → request/header → deriveMessages() → 冻结请求
     llm.stream()  →  assistant/message（成功）| assistant/attempt（失败，仅记录）
     agent/request-error (waterfall)      重试 / 上下文溢出后压缩再重试
     tool/call* → tools/pre-execute → guards → approval → tools/execute → tools/post-execute → tool/result*
     step/end
 └─ 若工具产生了结果或有 steer 输入 → 进入下一个 step；否则触发 agent/turn-stopping（Stop Hook 可强制继续）
turn/end { completed | max-tokens | blocked | aborted | error }
```

## 目录

- **第一部分　基础循环**
  - [第 0 章　全景：微内核与全插件架构](#ch0)
  - [第 1 章　Agent Loop：Turn/Step 状态机](#ch1)
  - [第 2 章　LLM 接入层：适配器、流式协议与重试](#ch2)
- **第二部分　工具调用**
  - [第 3 章　工具定义与执行管线](#ch3)
  - [第 4 章　工具生态：MCP、PTC、Skills、Hooks](#ch4)
- **第三部分　状态与上下文**
  - [第 5 章　Prompt 组装与上下文工程](#ch5)
  - [第 6 章　事件溯源的 Session：状态、持久化与崩溃恢复](#ch6)
  - [第 7 章　长上下文治理：Token 计量、压缩与溢出恢复](#ch7)
- **第四部分　多 Agent 与编排**
  - [第 8 章　Subagent、Workflow、Goal 与 Agent Team](#ch8)
- **第五部分　安全、稳定性与评测**
  - [第 9 章　安全与人机协同：沙箱、审批、注入防御](#ch9)
  - [第 10 章　稳定性工程：死循环、超时、取消与不变量](#ch10)
  - [第 11 章　评测与可观测性：Mock、录制回放、真实 API 与遥测](#ch11)
- **第六部分　落地**
  - [第 12 章　部署形态与 Capstone：从 0 到 1 复刻 mini-harness](#ch12)
- [附录 A　高频考点 → 章节 → 本项目答案锚点](#appendix-a)
- [附录 B　术语表](#appendix-b)
- [附录 C　必读决策记录（ADR）](#appendix-c)

---

<a id="ch0"></a>
## 第 0 章　全景：微内核与全插件架构

### 学习目标

- 理解 DSH 的核心理念"一切皆插件"：连 Agent Loop 本身都是可替换插件。
- 掌握 Cordis 的五个核心概念：Service、`inject` 依赖声明、类型化事件、五种分发模式、可逆的注册副作用（effect）。
- 看懂 Profile → Bundle → Patch 的分层组装，以及"能力缝（Capability Seam）"的三角色划分。

### 源码地图

| 主题 | 位置 | 重点 |
|---|---|---|
| Cordis 入门 | [docs/cordis-primer.md](docs/cordis-primer.md) | 五个概念；`emit`/`waterfall`/`parallel`/`serial`/`bail` 五种分发模式；waterfall 必须调用 `next()` |
| 架构总览 | [docs/architecture.md](docs/architecture.md) | Turn flow；"Where new behavior goes"表（新功能应挂在哪个扩展点） |
| 默认组合 | [packages/bundle/base/cordis.patch.yml](packages/bundle/base/cordis.patch.yml) | 生产 Agent 由哪些插件拼成 |
| 能力缝 | [docs/capability-seams.md](docs/capability-seams.md) | Service Definition / Provider / Consumer 三角色 |
| 事件地图 | [docs/event-producer-consumer.md](docs/event-producer-consumer.md) | 每个事件的生产者与消费者 |
| 插件入口范例 | [packages/core/agent-loop/src/index.ts](packages/core/agent-loop/src/index.ts) | `AgentLoop extends Service`、`static Config` schema、`AgentFactory` 注册 |
| 作用域 | [packages/core/scope/src/index.ts](packages/core/scope/src/index.ts) | 每个 Agent 独立的注册作用域（`createScope` / `scopeTarget`） |

### 设计考量与 Trade-offs

- **为什么这样设计**：压缩、重试、审批、Hook、Goal 续跑等 Agent 行为，本质上都是"在某个时机插入一段逻辑"。DSH 把这些时机公开为类型化事件（waterfall 可包裹、改写或短路），核心驱动器因此只有约 700 行（[agent.ts](packages/core/agent-loop/src/agent.ts)），其余能力全部通过插件挂载。
- **优点**：
  - 可替换：更换 LLM 适配器、持久化后端或沙箱实现都不需要改动核心代码；
  - 可热重载：每次注册都返回 disposer，卸载插件即可撤销其全部注册；
  - 配置即组合：`cordis.yml` 的 patch 按行 id 覆盖，同一套代码能组装出 web、headless、sdk 等多种产品形态。
- **瓶颈**：
  - 控制流分散在大量监听器中，调试时必须对照事件地图；
  - waterfall 监听器忘记调用 `next()` 会静默短路后续逻辑；
  - Cordis 是小众框架，上手成本高；
  - 类型声明合并会增加编译期复杂度。

### 业界方案对比

| 维度 | DSH | LangChain / LangGraph | AutoGen v0.4 | OpenAI Agents SDK |
|---|---|---|---|---|
| 扩展单元 | 插件 + 类型化事件 | Runnable 组合 / 图节点 | Actor 运行时 + 消息 | Tool、Guardrail、Hook |
| 核心循环 | 固定 ReAct 驱动器，可整体替换 | 由用户定义图结构 | 由 agent 间的消息驱动 | 固定 Runner 循环 |
| 组装方式 | YAML 分层 patch | 代码 | 代码 | 代码 |
| 拦截能力 | waterfall 可改写或短路 | callbacks 只能观察，改写需修改图结构 | 消息拦截 | input/output guardrails |

### 面试高频深度题

1. **从零设计一个 Agent 框架，哪些东西放进内核，哪些做成插件？**
   答题要点：内核只保留事件日志、驱动循环和注册表；策略性逻辑（压缩、重试、审批）全部外挂。按语义选择事件分发模式：观察用 `emit`，包裹或决策用 `waterfall`，有序副作用用 `serial`。
2. **插件式架构如何保证热卸载时不泄漏资源？**
   答题要点：注册即副作用（`ctx.effect()` 返回 disposer）；按 Agent 划分 scope；为每个注册表编写 HMR 安全测试。
3. **waterfall 中间件与 LangGraph 条件边在"拦截/改写"能力上有何区别？**

### 动手实验

运行 `pnpm dsh --profile headless --dump-config`，在输出中找到 `agent-loop`、`compaction-basic`、`llm-retry`、`repeat-tool-reminder` 这几行，再画出它们各自监听了哪些事件。

---

<a id="ch1"></a>
## 第 1 章　Agent Loop：Turn/Step 状态机

### 学习目标

- 读懂 `ReactLoopAgent` 的 Phase 状态机（`idle` / `maintenance` / `running`），以及 `turn()` 与 `step()` 两层循环。
- 区分三种输入语义：`followup`（下一个 turn）、`steer`（下一个 step，并唤醒驱动器）、`inject`（下一个 step，不唤醒驱动器）。
- 理解协作式取消，以及取消后如何保持历史一致。

### 源码地图

- [packages/core/agent-loop/src/agent.ts](packages/core/agent-loop/src/agent.ts)
  - `ReactLoopAgent`（L98）：`Phase` 联合类型，保证同一 Agent 只有一个驱动器
  - `send / followup / steer / inject`（L154）：三种输入 = 目标队列 × 是否唤醒
  - `cancel`（L175）：`AbortController.abort(cause)`；`keepInbox` 决定是否保留待处理输入
  - `wakeDriver`（L214）/ `kick`（L252）：`while (await this.turn()) {}`，单一驱动器，错误在驱动器边界隔离
  - `preStep`（L267）：`inbox.claim` → `systemPrompt.assemble` → 投影运行时上下文 → `agent/pre-step`
  - `turn`（L296）：`turn/start … turn/end`；5 种结束原因；`max-tokens` 一旦出现就保持到 turn 结束；决定是否进入下一个 step
  - `step`（L398）：一次模型调用 + 工具执行；内部 `while (true)` 是重试循环；取消时保存已流出的部分输出（`interrupted: true`）
  - `prepareRequest`（L547）/ `buildRequest`（L599）：见第 2、5 章
- [packages/core/agent-loop/src/inbox.ts](packages/core/agent-loop/src/inbox.ts)：`ReactLoopInbox.claim`（L109）；输入队列本身也是事件溯源的（`agent/inbox/spliced`）
- [packages/core/agent/src/runtime-types.ts](packages/core/agent/src/runtime-types.ts)：`agent/*` 事件声明（`pre-step`、`request`、`request-error`、`turn-stopping`……）
- [packages/core/agent/src/types.ts](packages/core/agent/src/types.ts)：公开的 `Agent` 接口
- 文档：[docs/agent-lifecycle.md](docs/agent-lifecycle.md)（时序图）、[docs/subsystems/core.md](docs/subsystems/core.md)、[agent-loop README](packages/core/agent-loop/README.md)
- 测试：[loop.spec.ts](packages/core/agent-loop/tests/loop.spec.ts)、[cancel.spec.ts](packages/core/agent-loop/tests/cancel.spec.ts)、[interception.spec.ts](packages/core/agent-loop/tests/interception.spec.ts)

### 设计考量与 Trade-offs

- **固定 ReAct 驱动器 + 事件拦截点，而不是通用状态图**：
  - 优点：行为可预测，每一步都可重放，新行为通过事件扩展；
  - 瓶颈：Plan-and-Execute、树搜索等非 ReAct 拓扑只能在循环外编排（第 8 章的 workflow / subagent），或者整体替换 `Agent` 实现。
- **Turn 与 Step 两级结构**：
  - Turn 指一次用户请求，直到"没有欠账"为止；Step 指一次模型请求加上它发起的工具调用；
  - 用户中途插话（steer）会进入 next-step 队列，在 step 边界被领取，不会打断正在执行的工具。
- **协作式取消**：
  - `AbortSignal` 贯穿请求和工具执行；
  - 被取消的流会把用户已看到的前缀保存为 `interrupted` 消息，下一次请求里模型能看到自己说过的话；
  - 尚未派发的工具调用会补上合成的错误结果（`ABORTED_BEFORE_DISPATCH`），保证 `tool_call` / `tool_result` 始终配对。
- **没有 `maxSteps` 硬上限**：`AgentLoop.Config` 只有 `maxParallelToolCalls` 和 `agents`。终止依靠模型自行停止、用户取消和各类守卫（第 10 章），这是一个值得在面试中讨论的取舍。
- **`max-tokens` 直接结束 turn**，不会自动续写，避免输出在截断处被悄悄拼接。

### 业界方案对比

| 维度 | DSH | LangGraph | CrewAI | AutoGen | OpenAI Agents SDK |
|---|---|---|---|---|---|
| 循环形态 | 固定 ReAct 驱动器 + 事件钩子 | StateGraph（节点/边/条件边）；`create_react_agent` 是预制图 | Agent 内部 ReAct 循环 + Crew Process（sequential / hierarchical） | 多 agent 对话轮转；v0.4 基于事件驱动 actor | Runner 循环 + handoff |
| 中途插话 | steer 进入 next-step 队列 | `interrupt()` 或更新 state | 基本不支持 | 发送新消息 | 需自行实现 |
| 终止条件 | 无工具调用 / cancel / pre-step reject / Stop Hook | 到达 END / `recursion_limit` | `max_iter` | termination condition / `max_turns` | final output / `max_turns` |
| 历史一致性 | 日志派生 + 强制配对 | 由 state reducer 决定 | 框架内部管理 | 消息列表 | 由 Runner 管理 |

### 面试高频深度题

1. **写一个 ReAct 循环只要几十行，工业级 Loop 还要多处理什么？**
   答题要点：中断与取消后的一致性、并发输入、流式部分输出、重试、崩溃恢复、工具调用配对完整性、可观测性、驱动器唯一性。
2. **Agent 正在执行工具时用户发来新消息，应该怎么处理？**
   答题要点：区分 followup / steer / inject 三种语义；新消息进入 next-step 队列，在 step 边界领取；如需立即中止，调用 cancel 并选择是否保留队列。
3. **取消后为什么要给没执行的 `tool_call` 补一个错误结果？**
   答题要点：主流 API 要求 `tool_call` 与 `tool_result` 配对，否则下一次请求直接报 400；同时要明确标记"未执行"，避免之后误重试不确定的操作。

### 动手实验

用 [agent-loop-testkit](packages/test-support/agent-loop-testkit/README.md) 配合 mock adapter 写一个测试：第一步模型返回 tool call，第二步返回纯文本，断言 session 事件序列为 `turn/start, step/start, …, tool/call, tool/result, step/end, step/start, …, turn/end`；再在工具执行期间调用 `cancel`，观察产生的 `ABORTED_BEFORE_DISPATCH` 结果。

---

<a id="ch2"></a>
## 第 2 章　LLM 接入层：适配器、流式协议与重试

### 学习目标

- 理解适配器 seam 的两阶段调用：`prepareCall`（解析路由和能力）→ `stream`（绑定并发起调用）。
- 掌握统一的消息与内容块模型、流块组装方式，以及失败尝试如何记录。
- 能设计有界的重试策略，说清楚哪些错误可重试、哪些不可以。

### 源码地图

- [packages/llm/llm/src/index.ts](packages/llm/llm/src/index.ts)
  - `LlmAdapter`（L208）：适配器抽象（`providerInfo`、`providerRetryPolicy`、`listModels`、`prepareCall`）
  - `LlmRuntime.registerAdapter`（L396）、`prepareCall`（L936）、`stream`（L1135）；`llm/stream` 是 waterfall 中间件链
- [packages/llm/llm/src/types.ts](packages/llm/llm/src/types.ts)：`ContentBlockMap`（`text` / `reasoning` / `image` / `file` / `tool-call` / `tool-addition` / `tool-removal`）、`StreamChunk`、`GenerateOptions`
- [packages/llm/llm/src/assembler.ts](packages/llm/llm/src/assembler.ts)：`BlockAssembler`，把流块组装为内容块
- [packages/core/agent-loop/src/assistant-stream.ts](packages/core/agent-loop/src/assistant-stream.ts)：`AssistantStreamAttempt`，负责实时帧、落盘结算和 `interruptedBlocks`
- [packages/llm/llm/src/error.ts](packages/llm/llm/src/error.ts)：`LlmError` 错误码（`RATE_LIMIT` / `SERVER` / `TIMEOUT` / `TRANSPORT` / `CONTEXT_WINDOW_EXCEEDED` 等）
- [packages/llm/llm/src/retry-policy.ts](packages/llm/llm/src/retry-policy.ts)：重试策略归 provider 所有；默认 5 次，初始延迟 500ms，上限 10s，jitter 0.1
- [packages/llm/llm-retry/src/index.ts](packages/llm/llm-retry/src/index.ts)：重试执行器，监听 `agent/request-error`（L243）；指数退避 + 抖动；等待可取消；**先写入持久化记录再开始等待**
- [packages/llm/llm-deepseek/src/](packages/llm/llm-deepseek/src/)：DeepSeek 适配器（`sse.ts` 解析 SSE，`serialize.ts` 序列化请求，`translate.ts` 转换 usage，含 cache read/write tokens）
- [packages/llm/llm-pi-ai](packages/llm/llm-pi-ai/README.md)：多 provider 及 OpenAI 兼容网关
- 文档：[docs/subsystems/llm-streaming.md](docs/subsystems/llm-streaming.md)、[docs/cookbook/adding-an-llm-adapter.md](docs/cookbook/adding-an-llm-adapter.md)

### 设计考量与 Trade-offs

- **两阶段调用**：`prepareCall` 先解析出路由能力（`contextWindow`、`systemPromptUpdate`、`toolUpdate`），Loop 据此决定 system prompt 以何种方式进入历史（第 5 章），然后再用同一个已绑定的适配器发起调用。
- **重试策略与执行器解耦**：策略由各 provider 配置；执行器 `llm-retry` 是独立插件。每次重试先写持久化事件再等待，因此可审计，崩溃后也能恢复。默认只重试瞬时错误码。
- **失败尝试只记录、不入历史**：失败的流被记录为 `assistant/attempt`，不会进入模型历史，便于诊断，又不会污染上下文。
- **瓶颈**：
  - 自有消息和流协议意味着每接入一家 provider 都要写一层翻译；
  - reasoning 块、`replayState` 等 provider 私有状态必须无损保存，才能保证重放一致。

### 业界方案对比

| 维度 | DSH | LiteLLM | LangChain `BaseChatModel` | Vercel AI SDK |
|---|---|---|---|---|
| 统一方式 | 自有 Message / StreamChunk + 适配器 | 统一成 OpenAI 格式的代理层 | 统一的 Chat 接口 | `streamText` + 各 provider 包 |
| 重试 | provider 自带策略 + 可持久化的执行器 | 内置 retries / fallbacks | `with_retry` / `with_fallbacks` | `maxRetries` |
| 能力协商 | `prepareCall` 返回路由能力 | 模型信息表 | `bind_tools` 等 | provider options |

### 面试高频深度题

1. **流式输出到一半网络断了，怎么处理？**
   答题要点：用户取消时，把已流出的内容保存为 `interrupted` 消息；出错时记录 attempt，并按策略重试整个请求。此时工具还没执行，重新请求不会重复产生副作用。
2. **如何设计 LLM 调用的重试策略？哪些错误可以重试？**
   答题要点：可重试的有 429、5xx、超时、传输层错误和空响应；上下文溢出不能原样重试，要先压缩再重试（第 7 章）；使用指数退避 + jitter；尊重 `retry-after`；取消优先于重试；设置次数上限。
3. **多模型路由和降级怎么做？**
   答题要点：在 `agent/request` waterfall 中改写 provider/model，由 `prepareCall` 校验；与 LiteLLM fallbacks 对比；注意切换模型后 prompt cache 会失效。

### 动手实验

用 [llm-mock-server](packages/test-support/llm-mock-server/README.md) 编排"429 → 500 → 成功"三次响应，观察 `llm-retry` 写入的事件和退避间隔。

---

<a id="ch3"></a>
## 第 3 章　工具定义与执行管线

### 学习目标

- 掌握 `ToolDefinition` 的完整约定：参数和输出 schema、并发安全声明、超时、内容投影、UI 呈现。
- 读懂 `executeToolCalls`：滚动并行池 + 独占屏障 + 按模型顺序提交结果。
- 画出工具执行管线：`pre-execute` → `guards` → `approval` → `execute` → `post-execute` → `finalizeContent` → `tools/result`。

### 源码地图

- [packages/core/tools/src/index.ts](packages/core/tools/src/index.ts)
  - `ToolDefinition`（L223）：`output` schema、`execute`、`isConcurrencySafe`、`timeoutMs`、`projectContent` / `finalizeContent`、`presentCall` / `presentResult`
  - `PreToolDecision` / `PostToolDecision`（L607 / L617）：`allow` / `deny` / `cancel` / `ask`；`accept` / `block`，可附带 `additionalContexts`
  - `ToolRuntime.register`（L1063）、`restrict`（L1097）、`guard`（L1136）、`executionMode`（L1303）、`execute`（L1369）
- [packages/core/tools/src/schema.ts](packages/core/tools/src/schema.ts)：`defineTool`（L554），负责参数校验与输出声明
- [packages/core/agent-loop/src/tool-calls.ts](packages/core/agent-loop/src/tool-calls.ts)
  - `executeToolCalls`（L60）：按执行模式分组
  - `parseArguments`（L105）：非法 JSON 保留为原始字符串
  - `runGroup`（L122）：有界滚动池；已完成的结果按模型顺序连续提交
  - `appendSkippedToolCall`（L250）：取消后补齐的合成结果
- 策略插件：[timeout-policy](packages/guard/timeout-policy/src/index.ts)（包裹 `tools/execute`）、[spill-policy](packages/spill/spill-policy/README.md)（大输出落盘，头尾内联）、[fs-observation-policy](packages/fs/fs-observation-policy/README.md)（先读后写）
- 工具实例：[tool-fs](packages/fs/tool-fs/README.md)、[tool-bash](packages/shell/tool-bash/README.md)、[tool-todo](packages/todo/tool-todo/README.md)
- 文档：[docs/tool-execution-pipeline.md](docs/tool-execution-pipeline.md)（Mermaid 流程图）、[docs/cookbook/adding-a-tool.md](docs/cookbook/adding-a-tool.md)、[docs/tool-catalog.md](docs/tool-catalog.md)

### 设计考量与 Trade-offs

- **工具错误是结果，不是异常**：
  - 管线任一环节抛出的错误都会被规范化为 `isError` 结果回灌给模型，让模型自我修正；
  - 只有调度器内部故障才会终止当前 step；此时 Loop 会为每个没有结果的调用补上 `TOOL_OUTCOME_UNKNOWN` 或 `TOOL_NOT_STARTED`。
- **参数非法不崩溃**：非法 JSON 按原始字符串传给工具，由工具校验后回报错误。
- **并行但结果有序**：
  - 工具通过 `isConcurrencySafe(args)` 主动声明可并行，默认独占；
  - 并行调用在有界池（`maxParallelToolCalls` 默认 10）中执行，结果按模型给出的顺序提交，因此历史确定、可重放；
  - 独占调用形成屏障，用于写操作。
- **管线分层**：
  - `pre-execute` 承载 Hook、权限和沙箱策略；**不允许改写参数**，因为参数已经记录在日志中并展示给用户；
  - guards 是单调的，只能拒绝；
  - `execute` 用包裹式实现超时、重试和指标；
  - `post-execute` 可以接受、替换或阻断结果，并能为下一步附加上下文。
- **输出必须是无损 JSON**：保证结果可持久化、可重放。
- **瓶颈**：管线层数多，理解成本高；同进程工具无法被强制终止，超时只能协作式实现（第 10 章）。

### 业界方案对比

| 维度 | DSH | OpenAI / Anthropic 原生 | LangGraph `ToolNode` | CrewAI Tool |
|---|---|---|---|---|
| 并行 | 工具主动声明 + 有界池 + 顺序提交 | API 层 `parallel_tool_calls` 只决定模型能否一次返回多个调用 | 默认并发执行同一消息中的多个调用 | 顺序执行为主 |
| 错误处理 | 统一规范化为 `isError` 结果 | 由调用方决定 | `handle_tool_errors` | 异常 + 重试 |
| 策略拦截 | pre/post waterfall + guard + 审批 | 无 | 通过图节点或 `interrupt` 实现 | 回调 |

### 面试高频深度题

1. **模型一次返回多个工具调用，如何安全地并行执行？**
   答题要点：工具声明并发安全，默认串行；限制并发数；写操作作为屏障；结果按模型顺序提交；取消时先等已启动的调用结束，再为未启动的补齐结果。
2. **工具报错时，应该抛异常中断，还是把错误回灌给模型？**
   答题要点：分类处理。可恢复的错误回灌为 `isError`；基础设施故障终止 step 并补齐结果；超时返回结构化的 `TOOL_TIMEOUT`。
3. **工具输出过大（例如 `cat` 一个 10MB 日志）怎么办？**
   答题要点：spill 策略按 token 预算内联头尾内容，附上完整文件路径；压缩阶段用 pruner 裁剪旧结果；原文保留在日志中。

### 动手实验

用 `defineTool` 写一个 `word_count` 工具，声明 `isConcurrencySafe: () => true` 和 `timeoutMs`；再写一个 `tools/pre-execute` 监听器，对特定参数返回 `deny`，观察模型拿到的错误文本。

---

<a id="ch4"></a>
## 第 4 章　工具生态：MCP、PTC、Skills、Hooks

### 学习目标

- 理解 MCP 工具如何被桥接成原生工具，从而复用整条执行管线。
- 理解 PTC（Programmatic Tool Calling，"代码即行动"）：模型写一段程序批量调用工具。
- 理解 Skills 的渐进披露、Hooks 如何兼容 Claude Code，以及工具很多时如何控制可见性。

### 源码地图

- **MCP**：
  - [packages/mcp/mcp-client/src/tools.ts](packages/mcp/mcp-client/src/tools.ts)：`listTools` 获取工具后用 `ctx.tools.register` 注册，命名为 `mcp__<server>__<tool>`
  - 同目录的 `connection.ts` / `transport.ts`：连接与传输
  - [mcp-resources](packages/mcp/mcp-resources/README.md)：资源发现与读取
  - 文档 [docs/subsystems/mcp.md](docs/subsystems/mcp.md)
- **PTC**：
  - [packages/core/tools/src/ptc.ts](packages/core/tools/src/ptc.ts)：`run_code` 传输
  - [ts-types.ts](packages/core/tools/src/ts-types.ts) / [py-types.ts](packages/core/tools/src/py-types.ts)：从 JSON Schema 生成 SDK 类型
  - [ptc-runtime-node](packages/ptc-runtime/ptc-runtime-node/README.md)：在子进程沙箱中执行程序
  - 文档 [docs/subsystems/ptc-runtime.md](docs/subsystems/ptc-runtime.md)
- **Skills**：[skill](packages/skill/skill/README.md)（目录）、[skill-filesystem](packages/skill/skill-filesystem/README.md)、[tool-skill](packages/skill/tool-skill/README.md)（按需加载全文）；文档 [docs/subsystems/skills.md](docs/subsystems/skills.md)
- **Hooks**：[packages/hooks/hooks-claude-code/src/index.ts](packages/hooks/hooks-claude-code/src/index.ts) 的事件映射：
  - `UserPromptSubmit` → `agent/pre-step`
  - `PreToolUse` → `tools/pre-execute`
  - `PostToolUse` → `tools/post-execute`
  - `Stop` → `agent/turn-stopping`（阻断型 Stop Hook 会强制继续执行）
- **Web**：[tool-web](packages/web/tool-web/README.md) 把外部内容标注为不可信，并剔除 HTML 中的隐藏内容
- **可见性控制**：`ToolSchema.deferLoading`、`ctx.tools.restrict`、system-prompt 的 `toolOrder`、按 Agent 组合能力的 [agent-preset](packages/preset/agent-preset/README.md)

### 设计考量与 Trade-offs

- **MCP 工具与原生工具同等对待**：审批、沙箱、超时、Hook 全部适用，模型感知不到来源差异。代价是进程外调用的延迟、MCP 服务器崩溃后的恢复，以及命名冲突（需要命名空间和名称规范化）。
- **PTC**：
  - 模型写代码批量调用工具，中间结果不进入上下文，只有最终结果进入历史，从而节省 token 和往返次数；
  - 每个子调用依然走完整管线，并记录 `tool/ptc-dispatch` 以便重放；
  - 代价：需要代码执行沙箱，调试更难，且要求模型具备写代码能力。
- **Skills 渐进披露**：名称和描述常驻 prompt，正文按需加载，控制 prompt 体积。
- **Hooks 兼容 Claude Code 配置格式**：可以直接复用现有 Hook 生态。

### 业界方案对比：本地直接调用 vs MCP vs 代码即行动

| 维度 | 本地函数调用 | MCP | 代码即行动（smolagents CodeAgent / CodeAct / DSH PTC） |
|---|---|---|---|
| 延迟 | 最低（同进程） | 多一层 JSON-RPC（stdio / HTTP） | 一次调用完成多步 |
| 类型与校验 | 强类型 | JSON Schema | 生成 SDK 类型 |
| 生态与复用 | 各框架私有 | 跨语言、跨客户端标准，支持动态发现 | 依赖执行环境 |
| 安全 | 同进程权限 | 独立进程，权限取决于服务器实现 | 需要沙箱，风险最高 |
| 上下文消耗 | 每次调用都进上下文 | 同左 | 只有最终结果进上下文 |

### 面试高频深度题

1. **MCP 与普通 Function Calling 是什么关系？什么时候该用 MCP？**
   答题要点：Function Calling 是模型接口层的协议，MCP 是工具提供方与 Agent 宿主之间的协议，两者是不同层次，可以组合使用；需要跨团队复用、动态发现、多客户端共享工具时用 MCP。
2. **工具有上百个时，如何避免 prompt 膨胀和选错工具？**
   答题要点：命名空间；按 Agent 限制可见工具；延迟加载；Skills 渐进披露；用子 Agent 分工。
3. **让模型写代码来调用工具（PTC / CodeAct），收益和风险分别是什么？**

### 动手实验

配置一个 stdio MCP server（例如官方的 filesystem server），确认带 `mcp__` 前缀的工具出现在 `request/header` 事件的 tools 中；再为它配置一条 `PreToolUse` Hook，拦截其中一个工具。

---

<a id="ch5"></a>
## 第 5 章　Prompt 组装与上下文工程

### 学习目标

- 读懂 `SystemPrompt.assemble`：段落排序、作用域覆盖、变量插值、工具排序。
- 理解两类内容的分离：稳定的 system prompt，和作为 user 角色快照追加的运行时上下文。
- 理解"system prompt 作为历史中的一个节点"的设计，以及它与 prompt cache 的关系。

### 源码地图

- [packages/core/system-prompt/src/index.ts](packages/core/system-prompt/src/index.ts)
  - `SECTION_ORDERS`：集中分配各段落的顺序值（身份 → persona 前缀 → 策略 → 各工具说明 → MCP → 结构化输出 → persona 后缀）
  - `PromptSection`：`order`、`text`（可以是函数）、`interpolate`、`complete`（整段替换）
  - `section` / `context` / `tools` / `variable` 四类注册
  - `assemble`（L558）：合并作用域层、排序工具（`toolOrder`）、触发 `system-prompt/assemble` waterfall
  - `renderPrompt`（L279）：严格的 `{{variable}}` 插值，未知变量直接抛错
  - `renderContextSections`（L318）
- [packages/core/agent-loop/src/runtime-context.ts](packages/core/agent-loop/src/runtime-context.ts)：`RuntimeContextProjection`（运行时上下文快照）和 `SystemPromptProjection`（system prompt 节点调和）
- [packages/core/agent-loop/src/agent.ts](packages/core/agent-loop/src/agent.ts) 的 `buildRequest`（L599）：
  - `request/header` 记录 config 和 tools，不含 prompt
  - `request/context` 记录路由能力
  - 工具增删通过 `developer/message` 增量通知模型
- 上下文插件：[agent-instructions](packages/context/agent-instructions/README.md)（AGENTS.md 链 + 字节预算）、[time-context](packages/context/time-context/README.md)、[file-reference](packages/context/file-reference/README.md)
- Persona：[persona](packages/preset/persona/README.md)
- 文档：[docs/subsystems/system-prompt.md](docs/subsystems/system-prompt.md)
- 快照示例：[snapshots/web/minimal-preset/system-prompt.expected.md](snapshots/web/minimal-preset/system-prompt.expected.md)

### 设计考量与 Trade-offs

- **确定性**：段落按数字顺序加名称排序，工具按配置或字典序排序，保证每台机器输出一致。这既有利于 prompt cache 命中，也方便快照测试。
- **稳定与易变分离**：沙箱策略、审批策略、当前时间等易变信息，以 user 角色的上下文快照追加在历史后部，避免改动 system 前缀而让 KV cache 失效。
- **system prompt 是日志中的节点**：
  - 路由支持时，prompt 的变更以"历史内追加"的方式进入；
  - 路由不支持或开始新的请求序列时，把 prompt 合并回头部节点；
  - 工具增删用 developer message 增量通知，不重写历史。
- **工具说明归属于工具包**：每个工具包自己贡献使用说明段落，工具和它的说明一起增减。
- **瓶颈**：
  - series / generation / consolidation 规则复杂，是本项目最难读的部分之一；
  - 插值失败时直接抛错（fail loud），要求配置严格正确。

### 业界方案对比

| 维度 | DSH | LangChain `ChatPromptTemplate` | DSPy | Claude Code / Cursor |
|---|---|---|---|---|
| 组织方式 | 插件贡献的有序段落 + 变量 | 模板 + `MessagesPlaceholder` | Signature + 优化器自动生成 | CLAUDE.md / rules 文件 |
| 动态上下文 | user 角色快照，可记录可重放 | 模板变量 | 由优化器决定 | 注入到消息中 |
| 缓存友好 | 显式设计 | 由使用者自己保证 | 不关注 | 显式设计 |

### 面试高频深度题

1. **如何设计 prompt 结构，最大化 prompt cache 命中率？**
   答题要点：前缀稳定；排序确定；易变信息后置；工具变化用增量通知；压缩时的摘要请求复用会话前缀。
2. **项目级指令（AGENTS.md / CLAUDE.md）太长怎么办？**
   答题要点：设字节预算，由远及近省略，最具体的文件放在最后截断；子目录的指令按需发现。
3. **哪些信息放 system prompt，哪些放 user message，哪些放在工具结果里？**

### 动手实验

在 headless profile 中配置 `personaPrefix` 和 `toolOrder`，对比两次运行 `request/header` 与 `system/message` 事件的差异。

---

<a id="ch6"></a>
## 第 6 章　事件溯源的 Session：状态、持久化与崩溃恢复

### 学习目标

- 理解"模型可见 ⟺ 已记录日志"这一核心不变量。
- 掌握只追加日志、surface（模型可见节点序列）、`deriveMessages` 和 projection 读模型之间的关系。
- 能讲清楚断点续跑、fork，以及崩溃时执行到一半的工具该怎么处理。

### 源码地图

- [packages/core/session/src/types.ts](packages/core/session/src/types.ts)：`SessionEventMap`，包括：
  - 结构事件：`turn/*`、`step/*`
  - 消息事件：`user/message`、`system/message`、`developer/message`、`assistant/message`、`assistant/attempt`
  - 工具事件：`tool/call`、`tool/result`
  - 请求事件：`request/header`、`request/context`
- [packages/core/session/src/index.ts](packages/core/session/src/index.ts)：`Session`（L446）、`append`（L722）、`deriveMessages`（L860）
- [packages/core/session/src/surface.ts](packages/core/session/src/surface.ts)：`surfaceOp` 的 `append` / `replace`，`foldSurface`（L600），`SurfaceManager`（L618）
- [packages/core/session/src/repair.ts](packages/core/session/src/repair.ts)：`openTurnClosers`（L53）补齐被中断的 turn；`ToolCallRecovery`（L105）补齐工具结果
- [session-projection](packages/session/session-projection/README.md)：`ctx.sessionProjections`，对事件做增量 fold 得到读模型
- [session-persistence-jsonl](packages/session/session-persistence-jsonl/README.md)：JSONL + zstd 存储；[session-format-catalog](packages/session/session-format-catalog/README.md) 管理格式代际迁移
- [session-checkpoint-policy](packages/session/session-checkpoint-policy/src/index.ts)：在请求前和工具执行前做持久化 checkpoint，失败则拒绝继续（fail-closed）
- [packages/core/agent-loop/src/invariant.ts](packages/core/agent-loop/src/invariant.ts)：运行时校验"请求可以由日志重建"
- 文档：[docs/subsystems/session.md](docs/subsystems/session.md)、[persistence.md](docs/subsystems/persistence.md)、[session-projection.md](docs/subsystems/session-projection.md)、[docs/session-format-status.md](docs/session-format-status.md)

### 设计考量与 Trade-offs

- **模型可见 ⟺ 已记录日志**：任何进入模型请求的内容都必须能从日志重建，由运行时不变量强制校验。这带来可重放、可审计、可 fork、可调试四个能力。
- **历史是派生出来的**：历史不单独存储，只有一个真相源。压缩在 surface 上执行 `replace`，而不是删除原始事件，原文始终留在日志中。
- **状态即投影**：inbox、goal、todo、plan 等状态都是对日志事件的 projection，崩溃恢复只需重放日志。
- **副作用之前先持久化**：checkpoint 发生在模型请求和工具执行之前；崩溃时执行到一半的工具标记为结果未知（unknown outcome），**不自动重试**，以保护非幂等操作。
- **格式演进只增不改**：每个版本迁移由一个相邻迁移包负责，已提交的历史代际永不覆盖。
- **瓶颈**：
  - 日志只增不减，长 session 打开时需要重放（项目用 [benchmarks/session-open](benchmarks/session-open) 性能门禁和 projection cache 缓解）；
  - 事件 schema 的演进成本高，每个新事件都需要严格评审。

### 业界方案对比

| 维度 | DSH 事件溯源 | LangGraph checkpointer | OpenAI Assistants / Responses | Letta（MemGPT） |
|---|---|---|---|---|
| 存储内容 | 每个事实一条事件，状态由此派生 | 每个 super-step 存一次 state 快照 | 服务端托管的 thread | 分层记忆（核心 / 归档） |
| 回溯与分叉 | 按 turn 边界 fork，原文不丢 | 按 checkpoint 回溯（time travel） | 受限 | 记忆编辑 |
| 恢复 | 重放日志 + 补齐中断 | 从最近快照继续 | 服务端负责 | 服务端负责 |
| 代价 | 重放开销、schema 演进 | 快照体积 | 厂商锁定 | 系统复杂 |

### 面试高频深度题

1. **Agent 执行到一半进程崩溃，如何断点续跑？崩溃时正在执行的工具怎么办？**
   答题要点：重放日志；为未闭合的 turn 补上结束事件；已记录调用但没有结果的工具标记为 unknown outcome，交给模型或人工判断，而不是自动重试。
2. **事件溯源和状态快照各适合什么场景？**
3. **如何实现会话 fork，也就是从某一轮分叉出新对话？**
   答题要点：只在 turn 边界切分；复制前缀事件作为子会话的 seed；记录血缘元数据（`parentSession`、`seedLength`）。

### 动手实验

用 headless profile 跑一个带工具调用的任务，打开生成的 JSONL 日志，逐条对照 turn / step / tool 事件；然后用 [tool-session-query](packages/session-query/tool-session-query/README.md) 搜索该会话。

---

<a id="ch7"></a>
## 第 7 章　长上下文治理：Token 计量、压缩与溢出恢复

### 学习目标

- 掌握三级递进策略：无 LLM 的工具结果裁剪 → LLM 摘要 → provider 报溢出后的被动恢复。
- 理解压缩为什么必须保持 `tool_call` / `tool_result` 配对，以及压缩锁与崩溃检测。
- 理解摘要请求复用会话前缀以命中 KV cache 的技巧。

### 源码地图

- [token-meter](packages/llm/token-meter/README.md)：`estimate.ts`（启发式估算）、`usage-projection.ts`（复用 provider 返回的 usage）、`contextPressure` / `contextBreakdown` 两个 projection
- [packages/compaction/compaction/src/](packages/compaction/compaction/src/)：`CompactionEngine` 接口；`tool-pairing.ts` 中的 `toolPairingBalancedBefore` / `toolPairingBalancedAfter`
- [packages/compaction/compaction-basic/src/index.ts](packages/compaction/compaction-basic/src/index.ts)
  - 在 `agent/pre-step` 上做压力触发的压缩（L158）
  - 在 `agent/request-error` 上做溢出恢复（L190）：只处理 `CONTEXT_WINDOW_EXCEEDED`，受 `maxOverflowRetries` 限制，**只有 surface 确实被替换过才重试**
  - `summarize`（L247）：子类唯一的定制点
- [compaction-basic/src/config.ts](packages/compaction/compaction-basic/src/config.ts)：`thresholdRatio` 默认 0.8，`retainRatio` 默认 0.16，另有 `headroomTokens`，支持按模型覆盖
- [compaction-basic/src/region.ts](packages/compaction/compaction-basic/src/region.ts)：选择压缩区间，保持工具配对，但不要求整轮
- [compaction-basic/src/summarizer.ts](packages/compaction/compaction-basic/src/summarizer.ts)：摘要调用复用会话的 system、tools 和 messages 前缀
- [compaction-tool-result-pruner](packages/compaction/compaction-tool-result-pruner/README.md)：无模型调用的头尾裁剪，先于摘要执行
- [compaction-image-offload](packages/compaction/compaction-image-offload/README.md)：卸载图片；[command-compact](packages/compaction/command-compact/src/index.ts)：`/compact` 手动压缩
- 文档：[docs/subsystems/compaction.md](docs/subsystems/compaction.md)、[docs/subsystems/token-meter.md](docs/subsystems/token-meter.md)

### 设计考量与 Trade-offs

- **主动 + 被动双保险**：
  - 主动：每个 step 之前检查压力，超过窗口的 80% 就压缩；
  - 被动：provider 报上下文溢出后，压缩并重试，同时限制重试次数。
- **由便宜到昂贵逐级尝试**：先做不调用 LLM 的工具结果裁剪；如果裁剪后压力已解除，就跳过摘要。
- **可审计的替换**：
  - 摘要以一条 `user/message` 通过 surface `replace` 替换一段区间，原始事件保留在日志中；
  - `compaction/start` / `compaction/end` 构成一把锁，崩溃后会留下可检测的孤儿锁。
- **配对保护**：区间边界不能拆开 `tool_call` 与 `tool_result`；允许压缩同一个超长 turn 中较早的 step。
- **计量不依赖 tokenizer**：用启发式估算，当请求信封完全一致时复用 provider 真实 usage。
- **瓶颈**：摘要有损，且本身消耗 token 和时间；启发式估算有误差；阈值需要按模型调优。

### 业界方案对比

| 策略 | 代表 | 优点 | 缺点 |
|---|---|---|---|
| 截断 / 滑动窗口 | LangGraph `trim_messages` | 简单、零成本 | 丢失早期约束 |
| 摘要缓冲 | LangChain `ConversationSummaryBufferMemory`、Claude Code 自动压缩 | 保留语义 | 有损、需要额外调用 |
| 向量检索记忆 | RAG memory、LlamaIndex memory | 可扩展到很长的历史 | 召回不稳定 |
| 分层记忆 | MemGPT / Letta | 模型自主管理记忆 | 复杂度高 |
| DSH | 裁剪 → 摘要 → 溢出恢复，原文留在日志，可用 session-query 回查 | 可审计，成本分级 | 实现复杂 |

### 面试高频深度题

1. **长上下文溢出你会怎么处理？请设计一个完整方案。**
   答题要点：预估 token → 阈值触发 → 先裁剪工具输出 → 摘要旧区间并保留尾部 → provider 报溢出时兜底重试 → 保证配对 → 原文可追溯 → 设置重试上限。
2. **压缩时为什么不能把 `tool_call` 和 `tool_result` 拆开？**
3. **摘要丢失了关键信息（例如用户早期提出的约束）怎么办？**
   答题要点：保留逐字的尾部；使用结构化摘要；把 goal、todo、plan 作为独立于对话的持久状态；把文件当外部记忆；需要时回查原始日志。

### 动手实验

把 `thresholdRatio` 调到 0.1，跑一个多轮任务，观察 `compaction/start`、`compaction/summary`、`compaction/end` 事件，以及替换后 `deriveMessages()` 的输出。

---

<a id="ch8"></a>
## 第 8 章　Subagent、Workflow、Goal 与 Agent Team

### 学习目标

- 理解"单 Agent + 委派"模式：子 Agent 以工具调用的形式出现，父 Agent 只看到最终结果。
- 区分几种编排层次：spawn（全新上下文）与 fork（继承已完成的轮次）；one-shot 与 continuable；脚本编排（workflow）；长期目标（goal）；团队协作（agent team）。
- 掌握防止递归委派和成本失控的手段。

### 源码地图

- [packages/subagent/subagent/src/](packages/subagent/subagent/src/)：provider 注册表；`depth.ts`（`delegationDepthOf`、`assertSubagentMaxDepth`）；`continuation.ts`、`control.ts`、`lifecycle.ts`、`run-settlement.ts`
- [tool-subagent](packages/subagent/tool-subagent/README.md)：`subagent` 工具（one-shot / continuable、`maxDepth`、persona、工具过滤）
- 后端：
  - 同进程：[spawn-in-process](packages/subagent/subagent-spawn-in-process/README.md)、[fork-in-process](packages/subagent/subagent-fork-in-process/README.md)
  - 异构外部 Agent：[claude-code](packages/subagent/subagent-claude-code/README.md)、[codex](packages/subagent/subagent-codex/README.md)、[acp](packages/subagent/subagent-acp/README.md)
- [tool-subagent-control](packages/subagent/tool-subagent-control/README.md)：`send_message` / `interrupt_agent` / `list_agents`
- [workflow](packages/workflow/workflow/README.md) + [tool-workflow](packages/workflow/tool-workflow/README.md)：用 JS 脚本编排，可用 `agent()`、`parallel()`、`pipeline()`、`phase()`、`log()`
- [tool-ralph](packages/workflow/tool-ralph/README.md)：多轮迭代，每轮启动一个全新的 Agent
- [goal](packages/goal/goal/README.md) + [goal-round-driver](packages/goal/goal-round-driver/README.md)：长期目标自动续跑，默认上限 256 轮
- [plan-mode](packages/plan/plan-mode/README.md)：先规划，人工审批后再执行；[tool-todo](packages/todo/tool-todo/README.md)：`todo_write`
- [experimental/agent-team](packages/experimental/agent-team/README.md)：Lead + teammates + 邮箱 + 任务板
- 文档：[docs/subsystems/subagent.md](docs/subsystems/subagent.md)、[workflow.md](docs/subsystems/workflow.md)、[goal.md](docs/subsystems/goal.md)、[agent-team.md](docs/subsystems/agent-team.md)

### 设计考量与 Trade-offs

- **委派优先于对等对话**：子 Agent 的中间过程不进入父上下文，节省 token，也实现了隔离；是 fork 还是 spawn，取决于子任务是否需要父 Agent 的上下文。
- **确定性控制流交给代码**：workflow 用脚本表达固定的分支和并行结构，而不是让 LLM 每次重新决定。
- **边界控制**：深度限制（默认 1）、工具过滤和 persona 共同防止递归爆炸与权限扩散；取消以 `parent` cause 向下传播。
- **异构子 Agent**：Claude Code、Codex 等外部 Agent 通过同一个 provider 接口接入。
- **瓶颈**：
  - 子 Agent 的报告不经独立验证（ralph README 明确说明了这一点）；
  - 多个 Agent 共享工作区时可能写冲突；
  - 成本成倍增加，协调逻辑复杂。

### 业界方案对比

| 模式 | 代表 | 控制权 | 上下文 | 适用场景 |
|---|---|---|---|---|
| Agent 作为工具（委派） | DSH subagent、OpenAI Agents SDK agents-as-tools | 父 Agent 保持控制 | 隔离 | 子任务边界清晰 |
| Handoff（移交） | OpenAI Agents SDK / Swarm | 控制权转移给另一个 Agent | 共享 | 客服分流 |
| 对话式群聊 | AutoGen GroupChat | 由管理者选择下一个发言者 | 共享 | 讨论、辩论 |
| 角色 + 流程 | CrewAI Crew / Process | 流程驱动 | 按任务传递 | 流水线式内容生产 |
| 图编排 | LangGraph supervisor / swarm / `Send` map-reduce | 图结构驱动 | state 共享 | 复杂拓扑 |
| 脚本编排 | DSH workflow | 代码驱动 | 隔离 | 可复现的批量任务 |

### 面试高频深度题

1. **什么时候该用 Multi-Agent，什么时候单 Agent 就够了？**
   答题要点：上下文隔离的需求、可并行度、能力或权限差异、成本与可调试性的权衡；先把单 Agent 做好，再考虑拆分。
2. **子 Agent 的上下文如何隔离和传递？结果如何回收？**
3. **如何防止 Agent 无限递归委派，以及成本失控？**
   答题要点：限制最大深度；限制轮数；设置 token 预算；取消向下传播；按角色过滤工具。

### 动手实验

分别配置 `spawn` 和 `fork` 两个 `tool-subagent` 实例，比较子 session 的 seed；再写一个 workflow 脚本，用 `parallel` 同时启动 3 个子 Agent 并汇总结果。

---

<a id="ch9"></a>
## 第 9 章　安全与人机协同：沙箱、审批、注入防御

### 学习目标

- 理解纵深防御链：策略 → 单调守卫 → 人工审批 → 操作系统级沙箱。
- 掌握 fail-closed 原则：沙箱不可用时报错，而不是在无沙箱的环境中直接执行。
- 能讲清楚防幻觉覆盖和防 prompt injection 的工程手段。

### 源码地图

- [sandbox](packages/sandbox/sandbox/README.md)：三种模式 `read-only` / `workspace-write` / `danger-full-access`；无法强制执行时返回 `SANDBOX_UNAVAILABLE`；escalation 允许模型申请放宽一级，但必须经人工批准
- [sandbox-local](packages/sandbox/sandbox-local/README.md)：bwrap / Landlock / Seatbelt / Windows ACL 四种后端；[sandbox-policy](packages/sandbox/sandbox-policy/README.md)
- [user-approval](packages/interaction/user-approval/src/index.ts)：`ApprovalService`（L150）；两种策略 `ask` / `never`；没有应答者时返回 `unavailable` 并拒绝；审批记录写入审计日志
- [permission-presets](packages/interaction/permission-presets/README.md)：把沙箱与审批组合成命名的权限模式
- [experimental/auto-review](packages/experimental/auto-review/README.md)：用 LLM 自动审查待执行的动作（LLM-as-judge 守门）
- [fs-observation-policy](packages/fs/fs-observation-policy/README.md)：必须先读才能写，读取后文件被外部修改则拒绝写入
- [tool-web](packages/web/tool-web/README.md)：外部内容标注为不可信；[tool-ask-user](packages/interaction/tool-ask-user/README.md)：模型主动向用户提问
- 文档：[docs/subsystems/sandbox.md](docs/subsystems/sandbox.md)、[approval.md](docs/subsystems/approval.md)、[permission-presets.md](docs/subsystems/permission-presets.md)、[SAFETY.md](SAFETY.md)、[docs/defensive-patterns.md](docs/defensive-patterns.md)

### 设计考量与 Trade-offs

- **单调守卫**：guard 只能拒绝，不能被后续监听器覆盖为允许，因此策略叠加是安全的。
- **一次一批**：审批结果为 `allowed-once`，只对当前这一次调用有效，不会隐式扩大授权范围。
- **同机沙箱 vs 容器 / 微虚拟机**：同机沙箱更轻量，但与宿主共享内核和文件系统（README 明确说明）；需要强隔离时应使用容器或远程执行器，DSH 的 SSH 系列 provider 支持整体迁移到远端执行。
- **结构化的幻觉拦截**：通过 schema 校验、先读后写和错误回灌等结构手段拦截错误操作，而不依赖对内容的语义判断。
- **瓶颈**：
  - 频繁审批容易让用户疲劳；
  - auto-review 本身可能误判，而且消耗 token；
  - plan mode 并不强制限制工具使用，需要配合沙箱才能真正生效。

### 业界方案对比

| 维度 | DSH | LangGraph | OpenAI Agents SDK | NeMo Guardrails | Claude Code |
|---|---|---|---|---|---|
| 人机协同 | `ask` 审批 + `ask_user_question` + plan 审批 | `interrupt()` + `Command(resume=…)` | 工具审批或自行实现（视版本） | 不负责 | 权限模式 + 逐次询问 |
| 执行隔离 | OS 级沙箱，支持 escalation | 无（依赖外部） | 托管工具 | 不负责 | OS 沙箱 |
| 内容护栏 | 标注不可信内容、Hook、auto-review | 自定义节点 | input / output guardrails | Colang 规则 | Hooks |

### 面试高频深度题

1. **如何防止 Agent 执行危险命令（例如 `rm -rf`，或泄露密钥）？**
   答题要点：分层防御；默认最小权限；fail-closed；人工审批；沙箱限制网络和文件系统；环境变量隔离；审计日志。
2. **如何拦截幻觉导致的错误操作（例如 Agent 编造内容覆盖了原文件）？**
   答题要点：强制先读后写和陈旧检测；schema 校验；工具错误回灌；执行后验证真实世界状态，而不是相信模型自述。
3. **Prompt injection（网页或工具输出中夹带恶意指令）怎么防？**
   答题要点：把不可信来源标注出来；剥离隐藏内容；把数据和指令分离；高危动作需要审批；沙箱最小权限；保留人工确认环节。

### 动手实验

在 `read-only` 沙箱下让 Agent 写一个文件，观察沙箱拒绝和 escalation 审批流程；再开启 `fs-observation-policy`，让 Agent 在没有先读的情况下编辑文件。

---

<a id="ch10"></a>
## 第 10 章　稳定性工程：死循环、超时、取消与不变量

### 学习目标

- 掌握死循环检测与跳出的多层手段，以及"提醒"与"硬中断"之间的取舍。
- 理解协作式超时为什么不能简单用 `Promise.race`。
- 理解运行时不变量与防御式编程的工程实践。

### 源码地图

- **重复调用检测**：[packages/guard/repeat-tool-reminder/src/index.ts](packages/guard/repeat-tool-reminder/src/index.ts)
  - 在 `tools/post-execute` 中统计同一工具、同一参数的连续调用次数；参数先深度排序键再序列化；被拒绝的调用同样计数
  - 达到阈值（默认 `[3, 5, 8]`）时，通过 `additionalContexts` 注入提醒，从温和提醒逐级升级为详细提醒
  - **只提醒，不否决调用**
  - 在 `agent/pre-step` 中，只要出现用户消息就重置计数
- **上层边界**：goal 默认 256 轮上限、ralph 的轮数限制、subagent 的 `maxDepth`；注意 Agent Loop 本身**没有** `maxSteps`
- **超时**：[timeout-policy](packages/guard/timeout-policy/src/index.ts) 包裹 `tools/execute`，超时返回 `TOOL_TIMEOUT`，属于协作式超时；截止时间库见 [packages/util/timeout](packages/util/timeout/README.md)
- **取消**：`AgentCancelCause` 取值为 `user` / `parent` / `disposed` / `hook`；[agent.ts](packages/core/agent-loop/src/agent.ts) 中的 `abortedCancelCause` 负责复制取消原因
- **失败 step 的补齐**：`ToolCallRecovery` 产生 `TOOL_OUTCOME_UNKNOWN` / `TOOL_NOT_STARTED`
- **错误隔离**：插件失败只结束当前 turn，不会让循环本身崩溃（见 `kick()` 的 catch 边界）
- **运行时不变量**：[invariants](packages/runtime-diagnostics/invariants/README.md)，以及各核心包的 `src/invariant.ts`（agent-loop、tools、session、system-prompt）
- **后台任务**：[jobs](packages/jobs/jobs/README.md) 把长任务放到后台执行，避免阻塞 turn
- 文档：[docs/defensive-patterns.md](docs/defensive-patterns.md)（dispose 必须等到真正静止、回调异常在分发器内隔离等）、[docs/postmortem/](docs/postmortem/)（真实事故复盘）

### 设计考量与 Trade-offs

- **提醒而不硬停**：
  - 好处：模型在合理重复（例如轮询构建状态）时保留自由；不引入魔法数字 `maxSteps`；
  - 代价：依赖模型遵从提醒，最终兜底靠用户取消或上层轮数上限；
  - 对照：LangGraph 用 `recursion_limit`（默认 25）直接报错，CrewAI 用 `max_iter`，OpenAI Agents SDK 用 `max_turns`；
  - 如果业务需要硬上限，在 DSH 中写一个插件，在 `agent/pre-step` 返回 `reject` 即可，这是很好的面试延伸点。
- **协作式超时**：超时后不抛弃仍在运行的 promise，避免残留后台任务；前提是工具必须响应 `signal`。
- **结果不确定就不重试**：保护非幂等操作；有界重试只针对模型请求，不针对工具。
- **运行时不变量**：在运行时断言关键关系（例如请求被冻结、请求可重建），在开发阶段尽早暴露问题。

### 业界方案对比（死循环）

| 手段 | 代表 | 特点 |
|---|---|---|
| 步数硬上限 | LangGraph `recursion_limit`、OpenAI SDK `max_turns`、CrewAI `max_iter` | 简单可靠，但可能误杀长任务 |
| 精确重复检测 + 提醒 | DSH repeat-tool-reminder | 不误杀，但依赖模型遵从 |
| 语义重复或进展检测 | 自研（embedding 相似度、状态哈希） | 更准确，但成本更高 |
| 预算上限 | token / 金额 / 墙钟时间 | 与业务成本直接挂钩 |
| 升级到人工 | HITL | 最稳妥，但需要人参与 |

### 面试高频深度题

1. **Agent 陷入工具调用死循环，怎么检测和跳出？**
   答题要点：分层回答。检测手段有精确重复、语义重复、无进展检测；干预手段依次是注入提醒、更换策略、升级到人工、硬停；结合本项目说明"提醒 vs 硬停"的取舍，以及如何用 `agent/pre-step` reject 实现硬停。
2. **工具执行超时如何实现？为什么不能简单用 `Promise.race`？**
   答题要点：`race` 返回后原 promise 仍在运行，会产生幽灵副作用和资源泄漏；正确做法是 `AbortSignal` 协作取消，外加进程级 kill（子进程场景）。
3. **如何保证 Agent 系统中的非幂等操作不被重复执行？**
   答题要点：副作用之前先 checkpoint；结果未知时不自动重试；使用幂等键；让模型或人工决定是否再执行。

### 动手实验

用 mock 模型让 Agent 连续 5 次调用同一工具，观察注入的 `repeat-tool-reminder` 消息；然后写一个 `agent/pre-step` 插件，实现 `maxSteps` 硬上限并编写测试。

---

<a id="ch11"></a>
## 第 11 章　评测与可观测性：Mock、录制回放、真实 API 与遥测

### 学习目标

- 理解多层测试体系：单元测试 + 覆盖率门禁、Mock Server、录制回放快照、真实 API e2e、性能基准。
- 掌握"验证世界状态，而不是 Agent 自述"的评测原则。
- 了解线上可观测性：OTel、token 用量、会话检索。

### 源码地图

- [docs/testing.md](docs/testing.md)：测试分层；"Prefer the real implementation over a mock"；"Verify the world, not the self-report"
- [llm-mock-server](packages/test-support/llm-mock-server/README.md)：可脚本化的 HTTP/SSE 响应，覆盖 429、500、卡顿、畸形 chunk、tool call，支持带种子的随机
- [agent-loop-testkit](packages/test-support/agent-loop-testkit/README.md)：用真实 Loop 驱动器做测试
- [llm-replay](packages/test-support/llm-replay/README.md)：回放录制好的 session，无需 API key
- [session-snapshot](packages/test-support/session-snapshot/README.md) 与 [snapshots/](snapshots/)（session / sdk / acp / web 四类场景）：通过 `pnpm run test:snapshot` 运行，`test:snapshot:record` 重新录制
- 真实 API：`pnpm run test:e2e`，没有 key 时自动跳过
- 性能基准：[benchmarks/](benchmarks/)（session-open、conversation-fold、agent-continuation、long-session-browser）
- 可观测性：
  - [otel](packages/telemetry/otel/README.md)、[session-telemetry-otel](packages/session/session-telemetry-otel/README.md)
  - [token-meter](packages/llm/token-meter/README.md)：turn 级用量，含 cache 读写 token
  - [session-query](packages/session-query/session-query/README.md) 与 [session-query-sqlite](packages/session-query/session-query-sqlite/README.md)：全文检索
  - [session-stats](packages/session/session-stats/README.md)
- 反馈：[message-feedback](packages/feedback/message-feedback/README.md)

### 设计考量与 Trade-offs

- **回放做回归，真实 API 做验收**：录制 → 回放的确定性测试作为回归基线，真实 API e2e 作为最终验收；只 mock 昂贵或非确定的边界（LLM、网络、时钟），下游一律用真实实现。
- **验证世界**：断言时重新读取文件、重新执行命令，确认未触碰的文件逐字节不变，防止 Agent "作弊"通过测试。
- **快照绑定模型可见输出**：prompt 的任何变化都会反映在快照 diff 中，可以直接审查。
- **瓶颈**：
  - prompt 一改，快照就要重新录制；
  - 仓库没有内置"任务成功率"类的离线评测集（例如 SWE-bench），需要自建；
  - LLM 的非确定性让在线指标波动较大。

### 业界方案对比

| 维度 | DSH | LangSmith / Langfuse | OpenAI Evals / Ragas | 学术基准 |
|---|---|---|---|---|
| 追踪 | 事件日志 + OTel | Trace UI | — | — |
| 回归 | 录制回放快照 | Dataset + Evaluator | Eval 集 | SWE-bench、τ-bench、AgentBench |
| 评判方式 | 验证世界状态 | LLM-as-judge / 规则 | 指标 / LLM 评分 | 执行结果 |

### 面试高频深度题

1. **如何评测一个 Agent？离线和在线分别看哪些指标？**
   答题要点：任务成功率、步数和轮数、token 与成本、工具错误率、延迟、人工介入率；区分轨迹评估和结果评估；LLM-as-judge 需要先做校准。
2. **LLM 输出不确定，回归测试怎么做？**
   答题要点：录制回放；Mock Server；断言结构而不是原文；验证世界状态；多次采样做统计。
3. **线上 Agent 出了问题，如何定位？**
   答题要点：根据日志重建当时的请求；看 OTel trace；看 token 用量和 cache 命中率；用会话检索找到出问题的轮次；查看失败尝试（`assistant/attempt`）。

### 动手实验

用 `pnpm run test:snapshot -t <name>` 跑一个快照场景，阅读对应的 `session.jsonl` 和期望输出；再用 llm-mock-server 写一个带 tool call 的 e2e 测试，并通过重新读取文件来断言结果。

---

<a id="ch12"></a>
## 第 12 章　部署形态与 Capstone：从 0 到 1 复刻 mini-harness

### 学习目标

- 了解同一棵插件树如何服务 Web、CLI、SDK、ACP、Webhook、桌面等多种形态。
- 通过 Capstone 项目把前 11 章串起来，形成可以在面试中讲述的项目经历。

### 源码地图

- Profiles：[bundle/](packages/bundle/README.md)（base、headless、web-app、sdk-app、acp-app、sdk-minimal）；[app-boot](packages/boot/app-boot/README.md)
- SDK：[packages/sdk](packages/sdk/README.md)（JSON-RPC 协议、server、client），Python SDK 见 [python/](python/README.md)
- 协议与服务：[acp](packages/acp/README.md)（Agent Client Protocol）、[api](packages/api/README.md)（BFF）、[webhook](packages/webhook/README.md)（外部事件触发会话）、[schedule](packages/schedule/README.md)
- 最小组合参考：[sdk-minimal](packages/bundle/sdk-minimal/README.md)

### 设计考量与 Trade-offs

- **一套内核，多种前端**：所有形态共用同一个插件树，差异只体现在 bundle 层。
- **进程外 SDK**：通过 JSON-RPC 与宿主通信，Agent 崩溃不会拖垮宿主进程。
- **统一启动入口**：profile 是唯一的启动方式，避免出现绕过配置的旁路。

### 业界方案对比

LangServe / LangGraph Platform、Agent Protocol、A2A（Agent-to-Agent）、以 MCP server 形式对外暴露能力、OpenAI Assistants 托管方案。

### Capstone：mini-harness 分阶段路线

| 阶段 | 实现内容 | 对照 DSH 模块 |
|---|---|---|
| 1 | 只追加事件日志 + `deriveMessages` | `core/session` |
| 2 | LLM 适配器 + 流式组装 + 错误码 | `llm/llm`、`llm-deepseek` |
| 3 | Turn/Step 循环 + 取消 + 部分输出保存 | `core/agent-loop` |
| 4 | 工具注册表 + pre/post 钩子 + 并行池 + 顺序提交 | `core/tools`、`tool-calls.ts` |
| 5 | Prompt 分段组装 + 运行时上下文 | `core/system-prompt` |
| 6 | 阈值压缩 + 摘要 + 配对保护 | `compaction-basic` |
| 7 | 有界重试 + 重复调用提醒 + 工具超时 | `llm-retry`、`guard/*` |
| 8 | MCP 桥接 + 子 Agent 工具 | `mcp-client`、`subagent` |
| 9 | 录制回放测试 + 真实 API 冒烟测试 | `test-support/*` |

### 面试高频深度题

1. **如何把 Agent 做成多租户服务？会话之间的并发与隔离怎么做？**
   答题要点：每个会话独占一个写句柄（例如 JSONL 持久化的独占写所有权）；按会话划分 scope；按租户配置沙箱和凭据；取消和资源回收以会话为单位。
2. **讲一个你做过的 Agent 项目的架构。**
   答题要点：以本大纲的分层作为答题框架，依次讲 Loop → 工具 → 状态 → 上下文 → 安全 → 评测，每层讲一个取舍。

---

<a id="appendix-a"></a>
## 附录 A　高频考点 → 章节 → 本项目答案锚点

| 高频考点 | 章节 | DSH 中的答案锚点 |
|---|---|---|
| 长上下文溢出处理 | 7 | `compaction-basic`：pre-step 压力触发 + request-error 溢出恢复；`tool-result-pruner` |
| 幻觉拦截 | 3、9 | `isError` 回灌；`fs-observation-policy` 先读后写；schema 校验；验证世界状态 |
| 工具调用死循环跳出 | 10 | `repeat-tool-reminder`；goal 轮数上限；subagent 深度限制；`agent/pre-step` reject |
| 并行工具调用 | 3 | `tool-calls.ts` 的 `runGroup`：有界池 + 按顺序提交 + 独占屏障 |
| 流式中断与取消 | 1、2 | `interrupted` 消息；`ABORTED_BEFORE_DISPATCH` |
| 崩溃恢复与断点续跑 | 6 | `session-checkpoint-policy`；`ToolCallRecovery`；`openTurnClosers` |
| LLM 重试 | 2 | `retry-policy.ts` + `llm-retry`：有界、指数退避加抖动、先持久化再等待 |
| Prompt cache 优化 | 5、7 | 确定排序；运行时上下文后置；system prompt 节点化；摘要复用前缀 |
| MCP vs Function Calling | 4 | `mcp-client` 桥接为原生工具 |
| Multi-Agent 选型 | 8 | subagent 的 spawn 与 fork；workflow；agent-team |
| HITL 与安全 | 9 | `user-approval`；sandbox；permission-presets；auto-review |
| Agent 评测 | 11 | 录制回放；真实 API e2e；验证世界状态；基准测试 |
| 状态管理选型 | 6 | 事件溯源 vs LangGraph checkpointer |
| 可扩展架构 | 0 | Cordis 插件、类型化事件、能力缝 |

<a id="appendix-b"></a>
## 附录 B　术语表

| 术语 | 含义 |
|---|---|
| Turn | 一次用户请求到"没有欠账"为止的完整过程，以 `turn/start` 和 `turn/end` 界定 |
| Step | 一次模型请求加上它发起的全部工具调用，以 `step/start` 和 `step/end` 界定 |
| Inbox（next-turn / next-step） | Agent 的两个输入队列：前者每个 turn 领取一条，后者在每个 step 边界全部领取 |
| Surface | 日志中对模型可见的节点序列；`deriveMessages()` 从它派生模型历史 |
| surfaceOp `replace` | 用一个新节点替换 surface 上的一段区间，压缩就是通过它实现的 |
| Request series | 共享同一请求前缀的一组请求；新序列会重新合并 system prompt |
| Projection | 对事件日志增量 fold 得到的读模型，例如 inbox、goal、token 用量 |
| Capability seam | 可替换的能力，由 Service Definition、Provider、Consumer 三个角色组成 |
| Waterfall | 可包裹、改写或短路的中间件式事件，监听器必须调用 `next()` 才会继续向下传递 |
| Profile / Bundle / Patch | 启动配置 / 可分发的配置层 / 按行 id 覆盖的配置补丁 |
| PTC | Programmatic Tool Calling：模型通过 `run_code` 写程序批量调用工具 |
| Spill | 把超大的工具输出写入文件，上下文中只保留头尾和文件路径 |
| Invariant | 包自带的运行时一致性断言 |

<a id="appendix-c"></a>
## 附录 C　必读决策记录（ADR）

这些记录直接回答"为什么这样设计"，建议在对应章节精读：

| 章节 | ADR |
|---|---|
| 0 | [microkernel-event-taxonomy](.agents/notes/implemented/architecture/2026-06-11-microkernel-event-taxonomy.md)、[capability-seams](.agents/notes/implemented/architecture/2026-06-13-capability-seams.md) |
| 1 | [explicit-turn-cancellation](.agents/notes/implemented/architecture/2026-07-16-explicit-turn-cancellation.md) |
| 2 | [twin-llm-adapters](.agents/notes/implemented/architecture/2026-06-13-twin-llm-adapters.md)、[bounded-llm-request-recovery](.agents/notes/implemented/architecture/2026-06-21-bounded-llm-request-recovery.md) |
| 3 | [parallel-tool-call-execution](.agents/notes/implemented/feature/2026-07-10-parallel-tool-call-execution.md)、[tool-output-spill-files](.agents/notes/implemented/architecture/2026-07-08-tool-output-spill-files.md) |
| 4 | [mcp-client-plugin](.agents/notes/implemented/feature/2026-07-07-mcp-client-plugin.md)、[ptc](.agents/notes/implemented/feature/2026-06-15-ptc.md)、[creator-skills-progressive-disclosure](.agents/notes/implemented/architecture/2026-09-21-creator-skills-progressive-disclosure.md) |
| 5 | [system-prompt-as-surface-node](.agents/notes/implemented/architecture/2026-09-02-system-prompt-as-surface-node.md)、[prompt-variables-and-tool-guidance-ownership](.agents/notes/implemented/architecture/2026-07-05-prompt-variables-and-tool-guidance-ownership.md) |
| 6 | [event-sourced-sessions](.agents/notes/implemented/architecture/2026-06-11-event-sourced-sessions.md)、[session-surface](.agents/notes/implemented/architecture/2026-06-18-session-surface.md)、[reconstructable-requests](.agents/notes/implemented/architecture/2026-07-05-reconstructable-requests.md)、[failed-step-tool-results](.agents/notes/implemented/bug-fix/2026-09-19-failed-step-tool-results.md) |
| 7 | [compaction-capability-seam](.agents/notes/implemented/feature/2026-06-18-compaction-capability-seam.md) |
| 8 | [subagent-capability-seam](.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md)、[subagent-persona-tool-filter-and-depth](.agents/notes/implemented/feature/2026-07-12-subagent-persona-tool-filter-and-depth.md)、[persisted-same-session-goal-domain](.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md) |
| 9 | [sandbox](.agents/notes/implemented/feature/2026-07-06-sandbox.md)、[approval-seam](.agents/notes/implemented/feature/2026-07-06-approval-seam.md) |
| 10 | [tool-call-timeout-policy](.agents/notes/implemented/architecture/2026-07-07-tool-call-timeout-policy.md)、[dev-invariants-over-deep-readonly](.agents/notes/implemented/architecture/2026-06-11-dev-invariants-over-deep-readonly.md)、[repeat-tool-guard](.agents/notes/archived/feature/2026-07-08-repeat-tool-guard.md)（已归档，仅作历史背景） |
| 11 | [real-api-e2e-ci](.agents/notes/implemented/testing/2026-06-19-real-api-e2e-ci.md) |

---

## 下一步

确认目录结构后，我们按章节逐一深入。每章的深入讲解包括：

1. 关键函数逐段精读，配合时序图；
2. 每道面试题的完整参考答案；
3. 动手实验的具体步骤和预期结果；
4. 可以写进简历的项目描述要点。
