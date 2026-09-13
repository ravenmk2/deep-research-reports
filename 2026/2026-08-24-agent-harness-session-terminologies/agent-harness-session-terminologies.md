# Agent Harness 核心术语调研报告：Conversation、Session、Thread、Run、Turn、Step

> 调研对象：DeepSeek Harness、Pi、Kimi Code、OpenCode、Codex、DeepAgents、Eino、AgentScope（Java）、Spring AI Alibaba 共 9 个开源 Agent Harness / Agent 框架。
> 调研方法：逐一克隆 9 个仓库主分支源码（2026-08 时点），从术语表、子系统设计文档、核心类型定义、事件词汇与官方文档中提取六概念的定义与层级关系，并以官方文档、发布说明和第三方分析做交叉验证。文中所有定义均可追溯到具体文件路径或公开来源。

## 摘要

九个项目对六个概念的处理呈现出清晰的"两大血统"：

- **交互式编程 Agent 血统**（DeepSeek Harness、Pi、Kimi Code、OpenCode、Codex）：以 **Session / Thread** 为持久化对话容器（几乎全部采用 append-only 事件日志或树结构），以 **Turn** 为"一次用户输入的完整处理周期"，以 **Step** 为"Turn 内的一次模型请求及其引发的工具执行"。其中 Codex 用 **Thread** 作持久层术语、把 **Session** 降级为运行时对象，与其余四家相反。
- **图编排框架血统**（DeepAgents、Eino、Spring AI Alibaba）：以 **Thread / Session** 为跨调用的持久化命名空间（检查点或事件日志），以 **Run** 为"图 / Agent 的一次调用"，以 **Step / Super-step** 为图调度单位；**Turn** 只在需要长驻输入循环时才出现（如 Eino 的 TurnLoop）。

**Conversation 在九个项目中几乎全部不是一等持久对象**，而是 Session / Thread 的"内容面"或 UI 投影。**Run** 的语义最分散：可以指一次 Agent Loop 执行（Pi、Kimi Code、OpenCode）、一次工作流脚本执行（DeepSeek Harness）、一次图 / Agent 调用（DeepAgents、Eino、Spring AI Alibaba）。**Step** 存在"Turn 内一次模型请求"与"图超步（super-step）"两种语义；**Turn** 也存在"一次用户输入的完整处理"与"一次模型生成"两种语义，OpenCode 甚至同时使用两者。

---

## 1. 背景：什么是 Agent Harness

"Agent Harness" 是 2025–2026 年间在生成式 AI 软件工程领域流行起来的术语，指包裹语言模型、使其成为能对工作产出的 Agent 的那一层运行时脚手架。业界目前最通行的表述是 **Agent = Model + Harness**——模型提供智能，Harness 提供状态、工具执行、反馈回路与可强制约束等"模型之外的一切"[^31^][^30^]。Martin Fowler 将其进一步窄化为编码 Agent 语境下的"外围脚手架"：通过前馈控制（规范、提示）与反馈传感器（lint、测试）提升 Agent 一次做对的概率并自我纠错[^29^]。微软 Agent Framework 文档给出的定义是：Harness 是"把语言模型变成能完成工作的 Agent 的运行时脚手架，驱动模型与工具调用、管理对话状态与上下文、应用审批策略，并能让 Agent 在多步任务中持续推进"[^23^]。2026 年的一篇 arXiv 论文甚至专门通过概念分析给出了 Agent Harness 的"充分必要条件"式定义，并把 Claude Code、Codex CLI、Aider、Cline、OpenHands、SWE-agent 列为典型实例[^22^]。

正是在这一层"脚手架"里，Conversation、Session、Thread、Run、Turn、Step 这六个词被反复使用，但各项目的所指并不一致。本报告逐项目给出基于源码证据的定义，再做横向对比。理解这六个概念的差异，对于设计跨 Harness 的协议（如 ACP）、做会话数据的跨工具迁移、或评估各架构的上下文工程能力，都是基础性的工作。

## 2. 参照语义：六个概念的"通用含义"

为便于对比，先给出六个概念在 LLM Agent 语境下的通用参照语义（后续各节将说明各项目如何偏离或细化这些语义）：

| 概念 | 通用参照语义 |
|------|--------------|
| **Conversation（对话）** | 用户与 Agent 之间完整的消息往来内容本身，是最"内容导向"的概念 |
| **Session（会话）** | 一段有始有终、可持久化、可恢复的交互上下文；通常是 Conversation 的载体 |
| **Thread（线程/话题线）** | 一条可独立标识、可分叉的对话线索；在图编排框架中是检查点的命名空间 |
| **Run（运行）** | Agent / 图 / 工作流从启动到结束的一次完整执行 |
| **Turn（轮次）** | 对话中的一个来回：一次用户输入及其引发的全部 Agent 处理，直到重新等待输入 |
| **Step（步骤）** | Turn / Run 内的最小执行单位：一次模型请求加上该请求引发的工具执行 |

需要强调的是，这张表只是分析用的参照系，不是任何项目的官方定义。下文将看到，几乎没有两个项目对这六个词的使用是完全一致的：有的项目刻意只保留其中三四个概念（DeepSeek Harness 明确"一个概念一个规范术语"），有的项目则让同一个词同时承担两种语义（OpenCode 的 Turn 与 Step）。

## 3. 交互式编程 Agent 血统

### 3.1 DeepSeek Harness（`dsh`）

DeepSeek Harness 是 DeepSeek 开源的 Agent Harness（命令为 `dsh`），采用"一切皆插件"的架构，构建在 Cordis 之上，目前处于开发者预览阶段（见仓库 `README.md`）。它是九个项目中术语治理最严格的一个：仓库维护了一份正式术语表 `docs/glossary.md`，并声明"领域词汇每个概念只用一个规范术语"（"Domain vocabulary … uses one canonical term per concept"）。

其概念体系以 **Session** 为核心。在 `docs/subsystems/session.md` 中，Session 被定义为"一个 append-only 的类型化 `SessionEvent` 日志，是 Agent 完整交互历史的唯一事实来源（single source of truth）；LLM 消息历史是从日志**派生**的，从不单独存储；重放（replay）就是从同样的事件重新派生"。事件词汇包括 `turn/start`、`turn/end`、`step/start`、`step/end`、`user/message`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result` 等，且可通过声明合并（declaration merging）由插件扩展。

**Turn 与 Step 的定义**直接写在术语表的 "loop hierarchy" 条目里：

- **Turn**——"会话中对已准入输入的一次排空（one drain of admitted input in a session），在模型及其工具停止、或某个终止性策略介入时结束"。
- **Step**——"一次模型请求，加上其响应所引起的工具执行；一个 Turn 包含零个或多个 Step"。
- **Round**（额外概念）——"包含一个 Turn 的外层策略迭代"，例如 Goal Round 或 Ralph Round；轮次计数器属于该策略，不统计会话中的每一个 Turn。

**Run** 在 DeepSeek Harness 中不属于核心 Agent Loop，而属于工作流子系统：`docs/subsystems/workflow.md` 定义了"一次工作流运行（workflow run）"——调用方提交一段模型编写的编排脚本，引擎为其创建 worker，返回 `WorkflowRun` 句柄（可等待 `result`、可 `cancel`、必须 `dispose`），其终止原因 `stopReason` 是封闭枚举 `completed | cancelled | error`。Ralph Loop 则被定义为"朝一个不可变目标进行的一次前台全新 Agent 工作流运行（one foreground fresh-agent workflow run）"。

**Conversation 与 Thread**：DeepSeek Harness 没有 Thread 概念（"thread" 仅出现在 worker_threads 引擎实现细节中）。Conversation 也不是持久化领域对象：术语表在解释 Goal 时特意澄清"Goal 是状态，不是调度器，也不是另一个对话（conversation）；会话日志仍是其事实来源"；在 UI 层，`dsh-client-ui-conversation` 包提供了 "Conversation Node 引擎"，把 Session 事件折叠成 Chat 节点用于展示（见 `docs/cookbook/adding-a-conversation-node.md` 与 `docs/subsystems/workflow.md` 末节）。也就是说，**Conversation 在 dsh 中是 Session 日志在 UI 上的投影**。

综上，DeepSeek Harness 的层级为：`Session ⊃ (Round/Goal) ⊃ Turn ⊃ Step`，`Run` 独立存在于 Workflow 子系统。

### 3.2 Pi（`@earendil-works/pi`）

Pi 是 earendil-works 开源的 Agent Harness 项目，包含交互式编码 Agent CLI（`pi-coding-agent`）、Agent 运行时（`pi-agent-core`）与统一多供应商 LLM API（`pi-ai`）（见仓库 `README.md`）。

**Session 即持久化的对话。** `packages/coding-agent/docs/sessions.md` 开篇即说："Pi 把对话保存为会话（Pi saves conversations as sessions），以便你继续工作、从较早的轮次分叉、重访之前的路径"。每个 Session 是一个 JSONL 文件，按工作目录组织存放于 `~/.pi/agent/sessions/`；文件格式（`session-format.md`）规定 Session 条目通过 `id`/`parentId` 构成**树结构**，从而支持原地分叉（branching）而无需新建文件；格式已有三个版本：v1 线性序列、v2 树结构、v3 重命名 `hookMessage` 为 `custom`。在新版 harness 会话层（`packages/agent/src/harness/session/`）中，Session 进一步被建模为 **Entry**（`message`、`model_change`、`compaction`、`branch_summary` 等）与 **Record**（`operation_started`、`abort_requested` 等）两类记录，并提供 **Lane** 这一概念：Lane 是 Session 树中一个命名的、可移动的叶子指针，相当于分支指针（`session.ts` 的 `createLane`/`moveLane`）。

**Run 是一次 Agent Loop 调用。** 事件类型注释（`packages/agent/src/types.ts`）写明："`agent_end` 是一次运行（run）发出的最后一个事件"；`ShouldStopAfterTurnContext` 的注释进一步区分了"提示运行（prompt runs）包含初始提示消息；续接运行（continuation runs）不包含既有上下文消息"。在持久层，一次 Run 被记录为 `operation_started{ kind: "run", originalPrompt, initialMessages }`（`harness/session/types.ts`）。

**Turn 的定义**同样来自 `types.ts` 的事件注释："Turn 生命周期——**一个 Turn 是一次助手响应加上任何工具调用/结果**（a turn is one assistant response + any tool calls/results）"，对应 `turn_start` / `turn_end{ message, toolResults }` 事件。这意味着在 Pi 中一个 Run 通常包含多个 Turn（每次模型响应+工具执行为一个 Turn），Turn 的粒度明显小于"一次用户输入"。

**Step 与 Thread** 不是 Pi 的一等概念：`step` 仅在 harness reducer 中作为区分 `assistant` / `compaction` / `branch_summary` 记录序列的字段出现；`thread` 只在一致性测试中作为示例 Lane 名出现。Pi 的层级为：`Session（树，含 Lane）⊃ Run ⊃ Turn`。

### 3.3 Kimi Code（`kimi-code`）

Kimi Code CLI 是月之暗面（Moonshot AI）开源的终端 AI 编码 Agent（见仓库 `README.md`）。其文档明确写道："Kimi Code CLI 把每一段对话持久化为一个'会话'（session）——存储消息历史与元数据，以便关闭终端后从上次的位置继续"（`docs/en/guides/sessions.md`）。

**Session 的存储结构**为 `~/.kimi-code/sessions/<workDirKey>/<sessionId>/`，其中 `state.json` 保存会话元数据（标题、创建时间等），`agents/*/wire.jsonl` 是 **Agent 事件流**（wire 日志），用于会话恢复与重放，并携带请求轨迹（发给模型的工具 schema、请求参数、MCP 工具列表）供调试；子 Agent 各有独立的 wire 日志。这与 DeepSeek Harness 的事件溯源思路高度一致。

**Turn 是核心调度单位。** 在 `packages/agent-core-v2/src/agent/loop/` 中，`Turn` 类型（`loop.ts`）具有 `queued | running | completed | failed | cancelled` 状态机；事件体系（`turnEvents.ts`）定义了 `turn.started`（携带 `agentId`、`turnId`、提示来源 `PromptOrigin`）、`turn.ended`（`TurnEndReason = completed | cancelled | failed | blocked`）以及 `turn.prompt` / `turn.steer` / `turn.cancel` 操作（`turnOps.ts`）。用户文档印证了这一语义：流式输出期间按 `Esc`/`Ctrl-C` 是"打断当前 Turn"，`Ctrl-S` 可"向正在运行的 Turn 注入内容"，多个 Skill 与提示词"作为单个 Turn 运行（一次 `/undo` 撤销整次提交）"（`docs/en/guides/interaction.md`）。Kimi Code 还提供 Goal 模式："在每一轮（turn）之后检查目标是否完成、受阻、暂停或仍活跃"（`docs/en/guides/goals.md`）。

**Step 是 Turn 内的循环迭代。** `turn.step.started` / `turn.step.completed` / `turn.step.interrupted` 事件携带 `turnId + step + stepId` 与 Token 用量；`BeforeStepContext` 含 `firstStepOfTurn` 标记；Turn 内 Step 数受 `loop_control.max_steps_per_turn` 配置上限约束（`loop.ts` 的 `createMaxStepsExceededError`）。**Run** 则与 Turn 的执行几乎同义：循环服务以 `LoopRunOptions{ turnId }` 启动一次运行，返回 `LoopRunResult{ type: completed | failed | cancelled, steps }`，且类型别名 `TurnResult = LoopRunResult`——即"Run 一次 Loop"就是"执行一个 Turn"。

**Thread 与 Conversation**：Kimi Code 没有 Thread 概念；Conversation 仅作为非正式用语出现（"把对话持久化为会话"、"Shell 模式的输出写入对话上下文，Agent 在后续轮次可见"）。层级为：`Session ⊃ Turn ≈ Run ⊃ Step`。

### 3.4 OpenCode（`anomalyco/opencode`）

OpenCode 是 SST 团队背景的开源终端 AI 编码 Agent，将会话数据存储在本地 SQLite 数据库（`~/.local/share/opencode/`）中[^1^]。

**Session 是一等持久对象。** 核心 schema（`packages/core/src/session/`）中，Session 表包含 `id`、`project_id`、`parent_id`、`title`、`agent`、`model`、累计 token/成本、`revert` 指针与时间戳；`parent_id` 非空即表示这是子 Agent 会话（第三方 schema 分析文档对此有详细印证：`parent_id IS NULL` 为顶层会话，否则为子会话[^3^]）。会话之下是 **Message**（`user` / `assistant` 两种角色）与 **Part**（`text`、`tool`、`reasoning`、`step-start` 等类型）。OpenCode 官方 CLI 提供 `opencode export [sessionID]` 导出会话数据、`opencode -c` 继续上次会话、`-s <id>` 恢复指定会话[^14^]。

**Turn 在 OpenCode 中有两个层面的含义。** 其一在压缩（compaction）逻辑中：`packages/opencode/src/session/compaction.ts` 定义了内部类型 `Turn = { start, end, id: MessageID }`，即以一条用户消息锚定、延伸到下一条用户消息之前的一段消息区间——这是"一次用户输入及其引发的全部处理"的经典 Turn 语义。其二是消息层面的隐含语义：助手消息的 `finish: "stop"` 与 `time.completed` 标志一个助手 Turn 的结束，第三方文档直接把 `message` 表称为"turns 表"[^3^]。

**Step 则直接继承 Vercel AI SDK 的语义**：处理管线（`processor.ts`）把模型流中的 `step-start` / `step-finish` 事件落盘为同名 Part——AI SDK 中一个 "step" 就是多步工具调用循环中的一次 LLM 生成。OpenCode 的 Agent 配置提供 `steps` 上限（`docs` 中 "Max steps" 一节），触顶后的行为是"该 Agent 的最大步数已达，工具被禁用，直到下一次用户输入"（`runner/max-steps.ts`），这句话本身就精确划出了 Turn（两次用户输入之间）与 Step（其间的模型调用次数）的边界。

**Run 由 SessionRunner 定义**（`packages/core/src/session/runner/index.ts`）："从已记录的 Session 历史运行一次本地续接（Runs one local continuation from already-recorded Session history）"；其接口注释说明"显式运行即使在没有符合条件的工作时也执行一次供应商尝试"。上层的 `SessionExecution` 服务（`execution.ts`）提供 `resume`（空闲时启动执行、否则加入活跃执行）、`wake`（登记新记录的工作，重复唤醒可合并）、`interrupt`（中断活跃工作）。**Thread 与 Conversation** 均非一等概念（第三方集成会把 OpenCode 会话映射为 Discord 的 thread，那是外部语义）。层级为：`Session ⊃ Turn（消息区间）⊃ Step（AI SDK 步）`，Run 是驱动 Session 前进的执行动作。

### 3.5 Codex（`openai/codex`）

Codex 是 OpenAI 开源的编码 Agent（Rust 实现，`codex-rs/`）。它是五个 CLI Harness 中唯一用 **Thread** 作持久层顶级术语的项目，这一选择在 app-server v2 协议中体现得最清楚：`Thread` 结构（`app-server-protocol/src/protocol/v2/thread_data.rs`）包含 `id`（UUIDv7）、`session_id`（"属于同一 session tree 的 thread 共享的会话 id"）、`forked_from_id`（由分叉创建时的源 thread）与 `parent_thread_id`（仅当该 thread 是子 Agent 时设置）。客户端通过 JSON-RPC 方法 `thread/start` 创建线程、`turn/start` 开始一轮；非临时线程的 rollout 文件（持久化日志）在**首次 `turn/start` 时才惰性创建**，而不是在 `thread/start` 时创建[^25^]。在 v1 协议中，`conversation_id` 直接就是 `ThreadId` 的别名（`protocol/v1.rs`）——**Conversation 是 Thread 的旧称**。

**Session 在 Codex 中是运行时对象而非持久概念。** `core/src/session/session.rs` 的注释写道："Session 是一个已初始化的模型 Agent 的上下文；一个 Session 同一时间至多运行一个任务，且可被用户输入打断"。`CodexThread`（`codex_thread.rs`）则是把 `Arc<Session>`、IO、会话来源与 rollout 路径捆在一起的句柄。也就是说：**Thread 是持久身份与历史，Session 是其内存中的运行态**——这与其余四家的用法正好相反，是术语对比中最值得注意的错位。

**Turn 与 Step 的边界**由两个 Context 类型精确刻画。`TurnContext`（`session/turn_context.rs`）的注释是"线程中单个轮次所需的上下文（The context needed for a single turn of the thread）"，持有轮次级配置（模型、审批策略、沙箱等）；`run_turn` 函数（`session/turn.rs`）的文档注释描述了 Turn 的完整语义："接收轮次输入并运行一个循环：在每次采样请求中，模型要么返回函数调用、要么返回助手消息；若模型请求函数调用，我们执行它并把输出在下一次采样请求中送回模型；若模型只发来助手消息，我们把它记入对话历史并认为本轮完成"。`StepContext`（`session/step_context.rs`）的注释则是"可能在两次模型采样请求之间改变的请求级状态（Request-scoped state that may change between model sampling requests）"——**Step = Turn 内的一次模型采样请求**。代码中 `run_turn` 显式"拥有用于播种上下文并发起第一次采样请求的 step"（`turn.rs` 第 206 行注释）。

**Run** 在 Codex 中不是正式术语（`run_turn` 只是函数名；`codex exec` 文档把非交互执行称为一次运行）。层级为：`Session tree ⊃ Thread ⊃ Turn ⊃ Step（采样请求）`。

## 4. 图编排框架血统

### 4.1 DeepAgents（`langchain-ai/deepagents`）

Deep Agents 是 LangChain 出品的"开箱即用的 Agent Harness"（自称 "the batteries-included agent harness"），构建在 LangGraph 之上，获得其流式、持久化与检查点能力（见 `README.md`）。它的术语基本**整体继承 LangGraph**，自身不再另造概念。

**Thread 是持久化对话的命名空间。** LangGraph 的持久化层围绕 Thread 构建：Thread 代表一次对话或一个任务，通过 `thread_id` 隔离不同用户与任务；每次图执行到新的节点，检查点（checkpoint）就被写入存储；恢复 Thread 时，Agent 从最近的检查点重建状态[^13^]。LangGraph 官方持久化文档的表述是：thread 是"checkpointer 保存的每个 checkpoint 所归属的唯一标识，包含多次 run 累积的状态序列"[^18^]。DeepAgents 的 README 直接使用这套词汇："上下文管理——总结超长 thread（summarize long threads）"、"持久记忆——可插拔的 state 与 store 后端，实现跨 session 召回（cross-session recall）"。

**Run 是图的一次调用。** `create_deep_agent` 的参数文档（`libs/deepagents/deepagents/graph.py`）写明：`checkpointer` 用于"在多次运行之间持久化 Agent 状态（persisting agent state between runs)"；`context_schema` 是"定义不可变的**运行级**上下文（immutable run-scoped context）的 schema 类"。这与 LangGraph Platform 的概念一致：Thread 累积多次 Run 的状态。

**Step / Super-step**：DeepAgents 代码注释（`backends/state.py`）提到"在每个 agent step 之后设置检查点"以及"在单个超步（a single superstep）内"——Super-step 是 LangGraph Pregel 式执行模型的调度单位，一次超步内所有就绪节点并行执行；LangSmith 文档中创建 Thread 时也可传入 `supersteps` 序列来预置状态[^15^]。**Session 与 Conversation** 不是核心库概念，只出现在 CLI 层面（`deepagents-code` "启动交互会话"，`main.py`）。层级为：`Thread ⊃ Run ⊃ (super-)step`。

### 4.2 Eino（`cloudwego/eino`）

Eino 是 CloudWeGo（字节跳动开源）的 Go 语言 LLM 应用开发框架，借鉴 LangChain 与 Google ADK，提供组件、编排（compose）与 Agent 开发套件（ADK）（见 `README.md`）。作为框架而非完整产品，它的会话概念主要由 ADK 的 Runner 体系给出。

**Run 是 ADK 的核心执行单位。** `Runner`（`adk/runner.go`）是"执行 Agent、管理多 Agent 协调、上下文传递与中断/恢复的核心引擎"，`runner.Query(ctx, "...")` 或 `runner.Run(ctx, messages)` 返回事件迭代器[^10^]。运行内有明确的作用域区分：`SetRunLocalValue` 设置的值"限定于本次 agent Run() 调用，不跨 Run 或 Agent 实例共享"，而 `WithSessionValues` 设置"Agent 运行的会话级值（session-scoped values）"[^28^]。运行状态（`runSession`，`adk/runctx.go`）保存该次运行的 Values 与事件序列，可通过 gob 序列化持久化。

**Session 在 v0.10 被提升为 Runner 管理的一等概念。** 官方发布讨论写道："v0.10 通过 `SessionID`、`SessionStore` 和 `SessionConfig` 增加了 Runner 管理的会话持久化。启用会话模式后，Runner 把对话记录为 append-only 的会话事件日志（session event log）；新的轮次可以加载先前的会话事件、重建面向模型的消息与工具上下文，然后追加最新输入再运行 Agent"。并且特意划清边界："检查点与会话持久化保持分离——会话事件跨轮次重建对话/会话状态；检查点恢复被中断的执行状态"[^21^]。

**Turn 由 TurnLoop 给出。** TurnLoop 是"面向随时间接收外部输入的长驻 Agent 的 push 式运行时"；"配置会话后，每个轮次（turn）都可以基于重建的会话历史运行；配置检查后，被中断的轮次仍可从中断点恢复"；其职责划分是"TurnLoop 管理输入缓冲与轮次调度，session 管理持久对话状态，checkpoint 管理被中断的执行状态"[^21^]。`TurnLoopConfig` 的 `GenInput` 回调"每轮调用一次，决定消费哪些推送项"（`adk/turn_loop.go`）。

**Step 有两层**：ADK 的 `RunStep` 表示"Agent 执行路径中的一步（a step in the agent execution path）"（主要用于 agent transfer 与 workflow agent）[^28^]；compose 图编排层则采用与 LangGraph 相同的 Pregel 模型，存在 **super step** 概念（`compose/graph_compile_options.go`、`compose/types.go` 中"前一个已完成的超步"）。**Thread 与 Conversation** 不是 Eino 的一等概念。层级为：`Session ⊃ Turn ≈ Run ⊃ RunStep / super-step`。

### 4.3 Spring AI Alibaba（`alibaba/spring-ai-alibaba`）

Spring AI Alibaba 是阿里巴巴的 Agentic 应用框架，采用双层架构：**Graph Core** 作为底层运行时（提供持久化、工作流编排、流式能力，本质上是 LangGraph 的 Java 移植），**Agent Framework** 在其上提供带上下文工程与 Human-in-the-Loop 支持的 ReactAgent 及多 Agent 工作流（`SequentialAgent`、`ParallelAgent`、`RoutingAgent`、`LoopAgent`）（见 `README.md`）。

**Thread 继承 LangGraph 语义。** `RunnableConfig`（`spring-ai-alibaba-graph-core/.../RunnableConfig.java`）持有 `threadId` 与 `checkPointId`；检查点体系（`checkpoint/Checkpoint.java` 及各 Saver 实现）以 thread 为命名空间保存图状态——例如 `MongoSaver` 中的逻辑"活跃 thread 已存在则返回其 thread_id""用 thread_id 作为检查点存储的键"。社区示例也展示了标准用法：为 ReactAgent 配置 `MemorySaver`，用 UUID 生成 `threadId` 传入 `RunnableConfig`，即可实现跨调用的对话记忆与检查点列举[^9^][^16^]。

**Run 是图的一次调用。** `RunnableConfig` 的注释写明："metadata 在执行期间不可变，用于为**某一次运行（a specific run）**提供环境信息"——这与 LangGraph 的 `RunnableConfig` 语义一致：每次 `invoke`/`stream` 传入一次配置，threadId 决定状态归属，run 本身无独立持久身份。

**Step / iteration**：Agent Framework 的 ReactAgent 以 `AgentLlmNode`（LLM 调用）与 `AgentToolNode`（工具执行）构成 ReAct 循环，`AgentLlmNode` 内维护模型迭代计数器（`_MODEL_ITERATION_` key，"Check and manage iteration counter"）；图执行引擎（`GraphRunner`）则以 step 为单位推进节点执行（"expand step continuations"）。**Session 与 Turn** 不是框架的一等领域概念（`ShellSessionManager` 只是 shell 工具的实现细节；会话记忆依赖 Spring AI 生态的 ChatMemory/`conversationId` 抽象，属于上游 Spring AI 的词汇）；**Conversation** 仅作为"对话历史"的非正式用语出现在 SummarizationHook 等组件中。层级为：`Thread ⊃ Run ⊃ iteration/step`。

## 5. AgentScope（`agentscope-ai/agentscope-java`）：介于两血统之间

AgentScope Java 是阿里巴巴的 Agent 框架，其 `agentscope-harness` 模块在核心 ReActAgent 之上提供了完整的 Harness 能力（工作区、记忆、沙箱、子代理、网关等），因此它同时具有框架血统与产品血统的特征。

**Session 是跨运行持久化的一等抽象。** 核心概念文档（`docs/v1/en/docs/quickstart/key-concepts.md`）写道："**Session 提供跨运行的持久存储**（persistent storage across runs）"，框架通过 `StateModule` 接口把"初始化"与"状态"分离（`saveTo` / `loadFrom` / `loadIfExists`），并用 `SessionManager.forSessionId("user123")` 统一管理保存与恢复。Harness 层的会话文档（`docs/v1/en/docs/harness/session.md`）描述了更精细的**双轨持久化**：每次 `agent.call()` 结束后，StateModule 快照（`Memory`、`ToolExecutionContext` 等可序列化状态，默认走 `WorkspaceSession`）与 **Conversation JSONL**（LLM 上下文 + 完整历史，走 `SessionTree`）沿两条并行独立的路径落盘；`SessionTree` 把 JSONL 组织为 `id/parentId` 树，其中 `<sessionId>.jsonl` 是压缩后的 LLM 可见上下文，`<sessionId>.log.jsonl` 是**永不压缩**的完整对话日志。这套设计与 DeepSeek Harness 的事件日志、Pi 的会话树在思想上同构。

**Turn 在网关层有明确的并发语义。** `SessionTurnGate` 接口（`agentscope-harness/.../gateway/SessionTurnGate.java`）的注释定义："针对每个 key 的网关轮次互斥（Per-key mutual exclusion for gateway turns）——即用户/渠道入站运行与子代理通知"；`acquire(key)` 获取轮次槽位并返回必须关闭的 `TurnLease`，`isRunning(key)` 用于跳过已活跃的会话。也就是说，**一个 Turn ≈ 一次由入站消息触发的 Agent 运行**，且同一会话的 Turn 被强制串行化。文档中 "each turn" 的另一处用法指每次模型调用周期（"工作区上下文注入每轮把 persona 与知识重新喂给模型"，`harness/overview.md`）。

**Step 是 ReAct 循环的语义单位。** 关键概念文档中的 ReAct 架构图把 Agent 循环画为"1. Reasoning（Memory→Formatter→Model）→ 需要工具？→ 2. Acting（Toolkit 执行工具、结果存入 Memory）→ 回到第 1 步"；源码注释中大量使用 "reasoning step"（一次 LLM 调用）与 "acting step"（一轮工具执行），循环上限由 `maxIters` 控制（`ReActAgent.java`）。**Run** 则与一次 `agent.call()` 执行同义（Session "across runs" 的表述）。**Thread** 无一等概念；**Conversation** 指 Memory 中的对话历史与 SessionTree 落盘的 JSONL 对话文件。层级为：`Session ⊃ Turn/Run（一次 call）⊃ ReAct reasoning/acting step`。

## 6. 横向对比

### 6.1 六概念 × 九项目总表

下表汇总各项目对六个概念的使用情况。"—"表示该项目无一等概念（至多作为非正式用语出现）。

| 项目 | Conversation | Session | Thread | Run | Turn | Step |
|------|--------------|---------|--------|-----|------|------|
| **DeepSeek Harness** | 非持久对象；Session 日志的 UI 投影（Conversation Node / Chat 节点） | **核心**：append-only `SessionEvent` 日志，交互历史唯一事实来源，消息历史由此派生 | — | 工作流子系统：一次工作流脚本执行（`WorkflowRun`，completed/cancelled/error） | 一次"已准入输入的排空"，模型与工具停止或终止策略介入时结束 | 一次模型请求 + 其响应引发的工具执行；Turn 含 0..n Step |
| **Pi** | 非正式（"把对话保存为会话"） | **核心**：JSONL 树（id/parentId），含 Entry/Record 与 Lane 分支指针 | — | 一次 Agent Loop 调用（`agent_start`→`agent_end`；prompt run / continuation run） | 一次助手响应 + 其工具调用/结果（`turn_start`/`turn_end`） | —（仅 reducer 内部字段） |
| **Kimi Code** | 非正式 | **核心**：state.json + 每 Agent `wire.jsonl` 事件流，可恢复重放 | — | 一次 Loop 运行 ≈ 执行一个 Turn（`TurnResult = LoopRunResult`） | 一次提示提交的循环执行（turn.started/ended；可 steer/cancel） | Turn 内一次迭代（turn.step.* 事件；max_steps_per_turn 上限） |
| **OpenCode** | 非正式 | **核心**：SQLite session 表（parent_id 支持子代理会话）；Message + Part | — | SessionRunner 的一次"本地续接"（从已记录历史驱动 Session 前进） | compaction 中以 user 消息锚定的消息区间；助手消息 finish="stop" 标志 Turn 结束 | AI SDK 语义：assistant 消息内 step-start/step-finish Part；Agent 可配 steps 上限 |
| **Codex** | v1 协议旧称（conversation_id = ThreadId） | **运行时对象**："已初始化模型 Agent 的上下文"，同一时刻至多一个任务 | **持久层核心**：ThreadId + rollout 文件；session_id 标识 session tree；支持 fork 与子代理 | 非正式（run_turn / codex exec） | run_turn：循环采样直至模型只回助手消息；TurnContext 持轮次级配置 | StepContext："两次模型采样请求之间可变的请求级状态"= 一次采样请求 |
| **DeepAgents** | 非正式 | CLI 层交互会话；非核心库概念 | **继承 LangGraph**：thread_id 标识的对话/任务，检查点的归属命名空间 | 图的一次调用；checkpointer "在多次 run 之间持久化状态" | — | agent step / LangGraph super-step（Pregel 调度单位） |
| **Eino** | —（对话历史是运行输入/会话事件） | v0.10 起 Runner 管理：append-only 会话事件日志，跨 Turn 重建对话状态 | — | 一次 Agent 执行（Runner.Run/Query，事件迭代器；RunLocalValue 作用域） | TurnLoop 的调度单位：每轮基于重建的会话历史运行 | RunStep（执行路径一步）；compose 层 super step |
| **AgentScope (Java)** | Memory 对话历史 + SessionTree 双 JSONL 落盘 | **核心**：跨运行的持久存储；StateModule 快照 + Conversation JSONL 双轨 | — | 一次 agent.call() 执行 | 网关层每会话互斥的入站运行（SessionTurnGate/TurnLease） | ReAct 循环的 reasoning step / acting step；maxIters 上限 |
| **Spring AI Alibaba** | 非正式（conversation history） | —（依赖 Spring AI ChatMemory/conversationId 生态） | **继承 LangGraph**：RunnableConfig.threadId + Checkpoint（Memory/Mongo/Postgres Saver） | 图的一次调用（"为某一次 run 提供的不可变元数据"） | — | ReactAgent 的模型 iteration；图执行 step |

### 6.2 层级关系对比

各项目的"持久容器 → 执行单位 → 最小步进"链条如下：

| 项目 | 层级链 |
|------|--------|
| DeepSeek Harness | Session ⊃（Goal/Round）⊃ Turn ⊃ Step；Run 属工作流子系统 |
| Pi | Session（树/Lane）⊃ Run ⊃ Turn |
| Kimi Code | Session ⊃ Turn ≈ Run ⊃ Step |
| OpenCode | Session ⊃ Turn（消息区间）⊃ Step（AI SDK 步） |
| Codex | Session tree ⊃ Thread ⊃ Turn ⊃ Step（采样请求） |
| DeepAgents | Thread ⊃ Run ⊃ (super-)step |
| Eino | Session ⊃ Turn ≈ Run ⊃ RunStep / super-step |
| AgentScope | Session ⊃ Turn ≈ Run（call）⊃ ReAct step |
| Spring AI Alibaba | Thread ⊃ Run ⊃ iteration/step |

### 6.3 关键差异分析

**（1）持久层的命名分裂：Session 派 vs Thread 派。** 五个 CLI Harness 中四家以 Session 为持久对话容器，Codex 却选择 Thread 并把 Session 降格为运行时对象；两个 LangGraph 血统框架（DeepAgents、Spring AI Alibaba）同样使用 Thread，因为 LangGraph 的持久化词汇就是"thread_id + checkpoint"[^18^][^13^]。值得注意的是，即便都用 Session，实现也高度趋同：DeepSeek Harness 的 `SessionEvent` 日志、Pi 的 JSONL 会话树、Kimi Code 的 `wire.jsonl`、AgentScope 的双 JSONL 轨道，本质都是 **append-only 事件日志 + 派生消息历史 + 树形分叉**，事件溯源（event sourcing）已成为这一层事实上的标准模式。OpenCode 是唯一采用关系型（SQLite）而非日志文件方案的，但其 message/part 模型与日志模型在信息内容上等价[^3^]。

**（2）Turn 的两种语义。** "一次用户输入的完整处理"（DeepSeek Harness、Kimi Code、OpenCode 的 compaction Turn、Codex 的 run_turn、AgentScope 的网关 Turn、Eino 的 TurnLoop）与"一次模型生成加工具调用"（Pi）是两种不同的粒度：前者可包含多次模型请求，后者恰等于前者的子单位。OpenCode 内部其实同时存在两种：compaction 的 `Turn{start,end,id}` 是前者，而 AI SDK 的 step 又把后者命名为 step——同一事件流在不同子系统被切成了两种粒度。Pi 是九家中唯一把 Turn 定义在"一次助手响应"粒度上的，因此它的层级是 `Run ⊃ Turn` 而非常见的 `Turn ⊃ Step`。

**（3）Step 的两种语义。** CLI 血统里 Step 几乎都是"一次模型请求 + 其引发的工具执行"（DeepSeek Harness 的定义最为精炼；Codex 用 StepContext 把同一语义落到"采样请求"上；Kimi Code、OpenCode、AgentScope 同义）。图编排血统里 Step 则指图调度单位：LangGraph/Eino compose 的 super-step（一批就绪节点并行执行）或执行路径中的一步（Eino `RunStep`）[^28^]。两种语义在"ReAct 循环一次迭代"这一点上是相容的，分歧只在并发模型（单 Agent 顺序循环 vs 图并行调度）。

**（4）Run 的语义谱系最宽。** 从"驱动一次 Agent Loop"（Pi 的 agent_start→agent_end、Kimi Code 的 LoopRun、OpenCode 的 SessionRunner run、AgentScope 的一次 call）到"一次独立工作流执行"（DeepSeek Harness 的 WorkflowRun），再到"图的一次调用"（DeepAgents、Eino、Spring AI Alibaba）。其共同内核是：**Run 是从启动到终止的一次完整执行，是把持久状态向前推进的原子动作**；差别在于被驱动的主体（Agent Loop / 工作流脚本 / 图）和是否与 Turn 重合。LangGraph 血统明确区分"Thread 累积多次 Run 的状态"[^18^]；Eino v0.10 则明确区分"会话事件跨 Turn 重建对话状态，检查点恢复被中断的执行状态"[^21^]——这两组划分是理解框架血统术语的钥匙。

**（5）Conversation 的"降级"是普遍共识。** 九个项目无一例外地不把 Conversation 作为持久化一等公民：它要么是旧称（Codex v1 的 conversation_id），要么是 Session 的内容面（Kimi Code、Pi 的"把对话保存为会话"），要么是 UI 投影（DeepSeek Harness 的 Conversation Node），要么只是文档里的非正式用语（DeepAgents、Spring AI Alibaba）。工程上的原因不难理解：Harness 需要持久化的是"可重建对话的事件/状态"，而不是"对话内容"本身——内容是派生物。

## 7. 结论

若以"哪个项目用什么词、指什么义"来回答调研问题，第 6.1 节的总表即是直接答案；若进一步归纳规律，可以得出三点结论。其一，术语选择高度由**架构血统**决定：交互式 CLI Harness 收敛于 `Session ⊃ Turn ⊃ Step` 的事件溯源模型，图编排框架收敛于 `Thread ⊃ Run ⊃ super-step` 的检查点模型，而 AgentScope、Eino v0.10 这类"框架长出 Harness 能力"的项目出现了两套词汇的合流（Session 事件日志 + TurnLoop + checkpoint 并存）[^21^]。其二，**Turn 与 Step 是相对最稳定的两个概念**——绝大多数项目认同"Turn = 一次输入的完整处理、Step = 其内一次模型请求加工具执行"，设计新系统时沿用这一语义最不易引起误解；需要避免的坑是 Pi 式的小粒度 Turn 与 OpenCode 式的双粒度并存。其三，**Conversation 应被视为视图而非存储**，持久层应选择 Session（交互产品语境）或 Thread（图/平台语境），并明确 Session 与"运行时对象"的边界——Codex 的反向用法（Thread 持久、Session 运行）虽自洽，但与多数生态相反，跨项目协作时需要显式映射（事实上 ACP 等协议正是用 session 一词指代这一层）。

对于需要在这九个生态之间做互操作（会话导入导出、统一观测、协议适配）的工程师，建议以"持久容器（Session/Thread）— 执行（Run）— 输入轮次（Turn）— 模型步进（Step）"四层作为规范模型，再按第 6.1 节总表逐项映射；其中 Pi 需把 Turn 下调一级、Codex 需把 Thread/Session 对调、图编排项目需把 super-step 与 Turn 区分开来。

## 参考来源

**一手源码证据**（各仓库主分支，2026-08 克隆；路径相对仓库根）：

- DeepSeek Harness：`README.md`、`docs/glossary.md`、`docs/subsystems/session.md`、`docs/subsystems/workflow.md`
- Pi：`README.md`、`packages/coding-agent/docs/sessions.md`、`packages/coding-agent/docs/session-format.md`、`packages/agent/src/types.ts`、`packages/agent/src/agent-loop.ts`、`packages/agent/src/harness/session/types.ts`、`packages/agent/src/harness/session/session.ts`
- Kimi Code：`README.md`、`docs/en/guides/sessions.md`、`docs/en/guides/interaction.md`、`docs/en/guides/goals.md`、`packages/agent-core-v2/src/agent/loop/loop.ts`、`.../turnEvents.ts`、`.../turnOps.ts`、`.../loopService.ts`
- OpenCode：`packages/core/src/session/`（`info.ts`、`execution.ts`、`runner/index.ts`、`runner/max-steps.ts`）、`packages/opencode/src/session/`（`compaction.ts`、`processor.ts`、`message-v2.ts`）
- Codex：`codex-rs/core/src/session/session.rs`、`.../turn.rs`、`.../turn_context.rs`、`.../step_context.rs`、`codex-rs/core/src/codex_thread.rs`、`codex-rs/app-server-protocol/src/protocol/v2/thread_data.rs`、`.../protocol/v1.rs`
- DeepAgents：`README.md`、`libs/deepagents/deepagents/graph.py`、`libs/deepagents/deepagents/backends/state.py`
- Eino：`README.md`、`adk/runner.go`、`adk/runctx.go`、`adk/turn_loop.go`、`compose/graph_compile_options.go`、`compose/types.go`
- AgentScope：`docs/v1/en/docs/quickstart/key-concepts.md`、`docs/v1/en/docs/harness/session.md`、`docs/v1/en/docs/harness/overview.md`、`agentscope-harness/src/main/java/io/agentscope/harness/agent/gateway/SessionTurnGate.java`、`agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java`
- Spring AI Alibaba：`README.md`、`spring-ai-alibaba-graph-core/src/main/java/com/alibaba/cloud/ai/graph/RunnableConfig.java`、`.../checkpoint/Checkpoint.java`、`.../checkpoint/savers/mongo/MongoSaver.java`、`spring-ai-alibaba-agent-framework/.../agent/node/AgentLlmNode.java`

**二手与官方文档**：

[^1^]: https://github.com/erayendes/mimir/issues/22
[^3^]: https://github.com/neochoon/agenthud/blob/main/docs/schemas/opencode-session.md
[^9^]: https://github.com/alibaba/spring-ai-alibaba/issues/1316
[^10^]: https://github.com/cloudwego/eino-ext/blob/main/skills/eino-agent/reference/runner-and-events.md
[^13^]: https://fast.io/resources/langgraph-persistence/
[^14^]: https://opencode.ai/docs/cli/
[^15^]: https://docs.langchain.com/langsmith/use-threads
[^16^]: https://blog.csdn.net/DavidSoCool/article/details/160857790
[^18^]: https://rudaks.tistory.com/entry/LangGraph-%EB%B2%88%EC%97%AD-Persistence （LangGraph Persistence 官方文档译文）
[^21^]: https://github.com/cloudwego/eino/discussions/1159
[^22^]: https://arxiv.org/html/2606.10106v1
[^23^]: https://learn.microsoft.com/en-us/agent-framework/concepts/harness
[^25^]: https://github.com/openai/codex/issues/31158
[^28^]: https://pkg.go.dev/github.com/cloudwego/eino/adk
[^29^]: https://martinfowler.com/articles/harness-engineering.html
[^30^]: https://www.databricks.com/blog/ai-harness
[^31^]: https://www.langchain.com/blog/the-anatomy-of-an-agent-harness
