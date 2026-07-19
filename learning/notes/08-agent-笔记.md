# 08 Agent 抽象 · 学习笔记

> 核心问题：Agent 到底是什么——一个进程还是一个对象？它管什么状态、暴露哪些方法、和 AgentSession 的边界在哪？为什么 core Agent 保持轻、coding-agent 在外面装设施？
> 证据来源：`packages/agent/src/agent.ts`、`agent-loop.ts`、`packages/coding-agent/src/core/sdk.ts`。
> 一句话：`Agent` 是 `packages/agent` 里的一个类，`new Agent(options)` 造一个内存对象。它持有一段对话的核心状态（messages/systemPrompt/model/tools/thinkingLevel…）和运行控制字段，暴露 prompt/continue/steer/followUp/abort 等方法，底层把状态交给 `runAgentLoop` 跑一轮。鉴权、存储、资源加载这些外围设施由 coding-agent 在外面装配。

---

## 题眼：Agent 是类实例，运行在当前进程里

`agent.ts` 里 `export class Agent`。`new Agent({...})` 创建一个实例，也就是当前 JS 运行时内存里的一个对象。它本身不启动独立进程、线程或常驻服务。

源码注释里对 `Agent` 的职责有一句很直接的定义：`Agent` owns the current transcript, emits lifecycle events, executes tools, and exposes queueing APIs for steering and follow-up messages。对应到中文，就是：`Agent` 持有当前 transcript，发出生命周期事件，执行工具，并提供 steering / follow-up 消息的排队 API。

几个直接推论（都由源码支持）：
- **一个实例通常对应一段独立对话状态**：它自己持有 messages、model、tools，实例之间没有共享 transcript。
- **一个进程可以有多个 Agent 实例**：`new Agent(` 有多个调用点（sdk.ts、web-ui、proxy），没有单例限制。想同时跑三段对话，就 `new` 三个。
- **不绑定 Node 文件系统**：web-ui 在浏览器里也 `new Agent`，说明 core Agent 可以脱离 coding-agent 的 Node 端设施使用。

理解方式（帮理解，非源码术语）：Agent 之于一段对话，类似 `new Map()` 之于一堆键值——一个持有状态、提供方法的普通对象。更准确的说法是“造一个 Agent 对象”。

---

## 一、Agent state 里有哪些字段（`createMutableAgentState`）

一个 Agent 实例暴露出来的 state 主要是这些字段：

```ts
state = {
  systemPrompt,      // 系统提示
  model,             // 当前模型（默认 DEFAULT_MODEL）
  thinkingLevel,     // 思考等级（默认 "off"）
  tools,             // 可用工具（get/set，赋值时拷贝顶层数组）
  messages,          // 对话消息（get/set，赋值时拷贝）
  isStreaming,       // 是否正在流式输出
  streamingMessage,  // 当前流式中的消息
  pendingToolCalls,  // 待执行的工具调用集合
  errorMessage,      // 错误
}
```

要点：
- 这些是对话状态和运行时状态的主要字段。`activeRun`、steering/followUp 队列等控制字段在 Agent 实例私有字段里，不在 `state` 对象里。
- 没传 `initialState` 时，默认是空 system prompt、`unknown` model、`thinkingLevel: "off"`、空 tools、空 messages。这能创建对象，但真实模型调用还要外层补 model/auth 或自定义 `streamFn`。
- `tools` 和 `messages` 有 getter/setter，赋值时拷贝顶层数组——可以运行时替换（Context 篇的 system prompt 重建、Tool 篇的 active tools 就是改这里）。
- **Context 篇的三件套（systemPrompt/messages/tools）就是这里的字段**：`createContextSnapshot` 把它们打包发给模型。现在能看清那三样本来就是 Agent state。
- `get state` 返回当前 state 对象。外部可以通过 `agent.state.messages = [...]` 或 `agent.state.tools = [...]` 替换顶层数组（coding-agent 恢复会话时就这么干）。

**"创建 Agent"实际 new 了什么**（构造函数字段，agent.ts:163-192）：state 只是其中一块，一个实例还持有——
- `_state`：上面那段对话状态（唯一暴露给外部的）。
- `steeringQueue` / `followUpQueue`：两个待注入消息队列（私有）。
- `listeners`：订阅事件的监听者集合（私有）。
- 一组可注入回调（public 字段）：`convertToLlm` / `transformContext` / `streamFn` / `getApiKey` / `onPayload` / `onResponse` / `beforeToolCall` / `afterToolCall` / `prepareNextTurn`。
- 传输/运行配置：`sessionId` / `thinkingBudgets` / `transport` / `maxRetryDelayMs` / `toolExecution`。
- `activeRun`：当前正在跑的那一轮（没跑时 undefined）。

所以"创建 Agent" = 在内存里 new 一个对象，把【一段对话状态 + 两个消息队列 + 监听者 + 一组可注入回调 + 传输配置】打包在一起。它不连数据库、不起服务、不占端口，也不内置应用级鉴权、存储、skill、extension 系统——这些是 AgentSession 或其他外层应用加的。

---

## 二、Agent 暴露哪些方法

按用途分组（都在 agent.ts）：

### 启动一轮
- **`prompt(input)`**：从文本、单条消息或一批消息开始一轮。如果已有 activeRun，直接抛错——**同一个 Agent 不能并发跑两轮**，要排队得用 steer/followUp。
- **`continue()`**：从当前 transcript 续跑。最后一条必须是 user 或 `toolResult`。若最后是 assistant，会先看 steering 队列、再看 followUp 队列有没有排队消息。

### 排队（不打断当前轮）
- **`steer(message)`**：入 steering 队列——当前 assistant 轮结束后注入。
- **`followUp(message)`**：入 followUp 队列——agent 本会停下时才跑。
- 两个队列都有 `clearXxxQueue` / `hasQueuedMessages` / mode 设置（`one-at-a-time` 或 `all`）。

### 控制
- **`abort()`**：中止当前轮（调 activeRun 的 abortController）。
- **`waitForIdle()`**：等当前轮 + 所有 event listener 结束。
- **`reset()`**：清空 messages、运行时状态、两个队列。

### 观察
- **`subscribe(listener)`**：订阅 `AgentEvent` 事件流，返回取消订阅函数。
- **`get state`**：读当前状态。

【区分】steer vs followUp（源码注释）：steer 是"当前 assistant 轮结束后注入"（插话），followUp 是"agent 本会停下时才跑"（收尾追加）。两者都是队列，区别在注入时机。

### 存活、结束和退出边界

这里要区分 `Agent` 实例和一次 active run。

- **`Agent` 实例的存活时间**：`new Agent(options)` 后，对象会一直存在，直到外层应用不再持有它并由 JS 运行时回收。`packages/agent/src/agent.ts` 里没有 `dispose()`、`destroy()`、`shutdown()` 或 `exit()` 方法。core `Agent` 不负责退出进程。
- **active run 的开始**：`prompt()` 或 `continue()` 会进入 `runWithLifecycle()`。这里会创建 `AbortController`，设置 `activeRun`，并把 `state.isStreaming` 设为 true。
- **active run 的结束**：`runAgentLoop` / `runAgentLoopContinue` 结束时会发出 `agent_end`。`agent_end` 表示 loop 不再发后续事件，但 Agent 还要等这个事件的 awaited listeners 都处理完。
- **真正 idle 的时点**：listener 处理完后，`finishRun()` 会把 `isStreaming` 设回 false，清空 `streamingMessage`，重置 `pendingToolCalls`，resolve `activeRun.promise`，最后把 `activeRun` 设回 undefined。`waitForIdle()` 等的就是这件事；没有 activeRun 时它直接 resolve。
- **`abort()` 的含义**：`abort()` 只调用当前 activeRun 的 `AbortController.abort()`，请求中止这一轮。它不销毁 Agent 实例，也不清空 transcript。
- **`reset()` 的含义**：`reset()` 清空 messages、运行时状态和 steering/followUp 队列，但对象本身仍然存在，之后还可以继续 `prompt()`。
- **进程退出**：print mode、interactive mode、RPC mode 怎么退出，是 coding-agent CLI 或外层应用的职责，不在 core `Agent` 里。

一句话：`Agent` 是一个可长期存在的对话对象；`prompt()` / `continue()` 启动的是一次短生命周期的 active run。run 会结束，Agent 不会因为 run 结束自动销毁。

---

## 三、Agent 怎么连接 runAgentLoop

`prompt` 不直接跑 loop，中间隔一层：

```
prompt(input)
  -> normalizePromptInput（文本/消息 → AgentMessage[]）
  -> runPromptMessages(messages)
     -> runWithLifecycle（建 activeRun + abortController）
        -> runAgentLoop(
             messages,
             this.createContextSnapshot(),   // { systemPrompt, messages, tools } 快照
             this.createLoopConfig(options),  // 把 AgentOptions 透传给 loop
             event => this.processEvents(event),
             signal,
             this.streamFn,
           )
```

两个关键打包：
- **`createContextSnapshot()`**：把 state 的 `{ systemPrompt, messages, tools }` 打包成一份快照给 loop（这就是 Context 篇的三件套来源）。
- **`createLoopConfig(options)`**：把 model、reasoning、convertToLlm、transformContext、beforeToolCall、afterToolCall、getSteeringMessages（drain steering 队列）、getFollowUpMessages 等透传进 loop config。

→ Agent 的职责是**持有状态 + 管生命周期（activeRun/队列/中止）**，真正的"模型↔工具往返"在 `runAgentLoop`（02 篇讲过）。Agent 是状态和入口，loop 是引擎。

---

## 四、AgentOptions 的几类接线口

这些是 `AgentOptions` 字段，构造时存进实例。关键在于哪些有默认值、哪些留给外层接线：

```ts
this.convertToLlm   = options.convertToLlm   ?? defaultConvertToLlm;  // 有默认（简单过滤）
this.streamFn       = options.streamFn       ?? streamSimple;         // 有默认
this.transport      = options.transport      ?? "auto";
this.toolExecution  = options.toolExecution  ?? "parallel";
this.onPayload      = options.onPayload;
this.onResponse     = options.onResponse;
this.prepareNextTurn = options.prepareNextTurn;
// 以下没有默认，不给就是 undefined：
this.transformContext = options.transformContext;   // undefined
this.beforeToolCall   = options.beforeToolCall;      // undefined
this.afterToolCall    = options.afterToolCall;       // undefined
this.getApiKey        = options.getApiKey;           // undefined
```

分两类：
- **core 给了最小默认**：`convertToLlm`（默认 `defaultConvertToLlm`，只做基本过滤）、`streamFn`（默认 `streamSimple`，走标准 provider 请求）。但真实模型调用仍需要可用的 model/auth，或者由调用方传入自定义 `streamFn`。
- **core 默认留空**：`transformContext`、`beforeToolCall`、`afterToolCall`、`getApiKey` 默认 undefined。core 不预设 extension、权限拦截或鉴权。coding-agent 在外层接线：`transformContext` 接 extension 的 `emitContext`，鉴权放进 `streamFn` 闭包，工具拦截通过 `AgentSession` 安装到 `beforeToolCall` / `afterToolCall`。

按用途看，这些接线口分几类：

| 类别 | 字段 | 用途 |
|---|---|---|
| LLM 转换 | `convertToLlm`、`transformContext` | 把 AgentMessage 转成模型消息；在转换前改上下文 |
| Provider 请求 | `streamFn`、`getApiKey`、`onPayload`、`onResponse`、`transport`、`sessionId`、`thinkingBudgets` | 控制如何发请求、如何取 key、如何观察 payload/response |
| 工具执行 | `beforeToolCall`、`afterToolCall`、`toolExecution` | 拦截工具、改工具结果、选择 parallel/sequential 策略 |
| 下一轮准备 | `prepareNextTurn` | turn 结束后、下一次 provider 请求前，替换 context/model/thinkingLevel |
| 队列语义 | `steeringMode`、`followUpMode` | 控制队列一次 drain 一条或 drain 所有队列项 |

细节：底层 `AgentLoopConfig.prepareNextTurn` 会收到 turn context；`AgentOptions.prepareNextTurn` 这一层当前只接收 signal，Agent wrapper 不把 turn context 透给调用方。要基于 `message/toolResults/context` 做复杂决策，需要看更低层 loop 或外层 harness 怎么接。

这就是 core Agent 保持轻的机制：模型调用、context transform、tool hooks 这些能力通过 `AgentOptions` 或实例属性接入；存储、资源、settings 这类外围设施留给外层包装。

---

## 五、Agent 和 AgentSession 的边界

| | `Agent`（packages/agent） | `AgentSession`（packages/coding-agent） |
|---|---|---|
| 是什么 | 纯对话引擎（状态 + 生命周期 + loop 入口） | Agent + 一整套设施的包装 |
| 持有 | messages/systemPrompt/model/tools + 队列 | 一个 Agent 实例 + SessionManager/ResourceLoader/ExtensionRunner… |
| 鉴权 | 默认没有应用级凭据管理；可通过 `streamFn` / `getApiKey` 接入 | 有（`streamFn` 闭包接 `ModelRegistry`） |
| 会话持久化 | 无 | 有（SessionManager 存 JSONL 树） |
| 资源（skill/extension） | 无 | 有（ResourceLoader） |
| system prompt 重建 | 无（state.systemPrompt 要外部设） | 有（_rebuildSystemPrompt 从资源现拼） |

边界一句话：**Agent 管"一段对话怎么跑"，AgentSession 管"这段对话的外围设施"**（存哪、用哪个 key、有哪些 skill/extension、system prompt 怎么拼）。`createAgentSession` 内部 `new Agent` 再 `new AgentSession` 包一层——应用入口篇讲过装配顺序。

---

## 六、为什么这么分层

core Agent 轻、coding-agent 装设施，好处（部分是顺着源码结构的判断，非源码明写）：
- **core 可跨环境复用**：Agent 不碰文件系统/进程，web-ui 在浏览器里也能直接 `new Agent`，配自己的浏览器侧存储和鉴权（应用入口篇讲过）。
- **设施可替换**：鉴权、存储、资源加载都是注入点，Node 端一套、浏览器端一套，共用同一个引擎。
- **测试友好**：可以传入假的 `streamFn`、tool 和初始状态，绕开鉴权、资源加载等设施来测 loop 行为。
- **应用可精确控制**：crbuddy 最初只要对话内核，直接 `new Agent`；后来要 skill/memory 才迁到 createAgentSession。

---

## 七、subagent / multi-agent：core 没内置，示例扩展能做

这个问题要分层说，不能简单写成“有”或“没有”。

### core Agent 层

`packages/agent` 的 `Agent` 类内部不会自己创建另一个 Agent，也没有 agent pool、swarm、orchestrator 这类编排结构。`Agent` 的职责仍然是跑一段对话：保存状态、处理队列、调用 `runAgentLoop`。

从这个层面看，core Agent 不提供 subagent 原语，也不提供多 agent 协作框架。

### coding-agent 默认工具层

coding-agent 默认内置工具是 read、bash、edit、write、grep、find、ls 这类本地工具。默认工具集中没有 Claude Code 那种直接派生子 agent 的 Agent/Task 工具。

这说明 subagent 属于扩展层能力，默认 active tools 里没有它。

### extension 示例层

仓库里有一个真实的 subagent 示例扩展：`packages/coding-agent/examples/extensions/subagent/`。它会注册一个名为 `subagent` 的自定义工具，调用时为每个子任务启动独立的 `pi` 子进程，让子任务有隔离的上下文窗口。

它的实现链路是：

```text
extension 加载
  -> pi.registerTool({ name: "subagent", ... })
模型调用 subagent 工具
  -> discoverAgents(ctx.cwd, agentScope)
  -> 找到 agent markdown 定义
  -> 把 agent.systemPrompt 写入临时 prompt 文件
  -> spawn 独立 pi 进程：pi --mode json -p --no-session ...
  -> 读取子进程 stdout 的 JSON 事件
  -> 收集 message_end / tool_result_end / usage / stopReason
  -> 把最终输出和 details 返回给父 agent
```

这里的“子 agent”指独立 `pi` 子进程，而非当前 `Agent` 实例内部 new 出来的对象。它带自己的上下文窗口、模型配置、工具配置和输出流。父 agent 只看到 `subagent` 工具结果。

这个示例支持三种模式。

**single：一个 agent 处理一个 task**

假设父模型调用工具时给了这段参数：

```json
{ "agent": "scout", "task": "Find where authentication is implemented" }
```

工具会先在 agent 定义里找 `scout`。如果 `scout.md` 里写了：

```yaml
---
name: scout
description: Fast codebase recon
tools: read, grep, find, ls, bash
model: claude-haiku-4-5
---
You are a fast codebase scout. Return compact findings with file paths.
```

工具会把这份定义翻译成一次子进程调用：

```text
pi --mode json -p --no-session \
  --model claude-haiku-4-5 \
  --tools read,grep,find,ls,bash \
  --append-system-prompt /tmp/pi-subagent-xxx/prompt-scout.md \
  "Task: Find where authentication is implemented"
```

`prompt-scout.md` 里放的是 `scout.md` 的正文。子进程运行时会自己选模型、构造上下文、调用工具。父进程不进入它的上下文，只读取它 stdout 里的 JSON 事件。

父进程读取子进程 JSON 事件，收集 assistant 消息、工具结果、usage、stopReason。子进程成功时，返回最后一条 assistant 文本；失败时，返回 errorMessage、stderr 或最后输出。

**parallel：多个 agent 并行处理多个 task**

parallel 的输入像这样：

```json
{
  "tasks": [
    { "agent": "scout", "task": "Find model resolution code" },
    { "agent": "scout", "task": "Find provider registration code" },
    { "agent": "reviewer", "task": "Review auth-related code paths" }
  ]
}
```

每个 task 都会走一次 single 的 `runSingleAgent` 路径，也就是各自启动一个独立 `pi` 子进程。代码先检查任务数，最多 8 个；真正执行时通过 `mapWithConcurrencyLimit` 控制并发，最多 4 个子进程同时跑。

运行期间，父工具维护一个 `allResults` 数组。每个子进程有更新时，父工具会把当前完成数和运行数通过 `onUpdate` 发回 UI，例如“2/3 done, 1 running”。所有任务结束后，它把每个 task 的结果整理成小结返回给父模型。为了避免一次并行输出塞太多上下文，每个 task 返回给父模型的文本最多 50 KB；完整消息、stderr、usage 等放在 tool details 里。

**chain：多个 agent 按顺序接力**

chain 的输入像这样：

```json
{
  "chain": [
    { "agent": "scout", "task": "Find the read tool implementation" },
    { "agent": "planner", "task": "Use these findings to propose a refactor plan:\n{previous}" },
    { "agent": "worker", "task": "Implement the plan:\n{previous}" }
  ]
}
```

chain 一次只跑一个子进程。第一步 `scout` 完成后，代码取它的最终输出，存进 `previousOutput`。第二步 `planner` 的 task 里有 `{previous}`，执行前会替换成 scout 的输出。第三步同理，会拿到 planner 的输出。

如果某一步失败，chain 会停止，返回“停在哪一步、哪个 agent 失败、失败输出是什么”，并把已经完成的步骤放进 details。所有步骤成功时，返回最后一步的最终输出。

它还定义了自己的 agent 配置格式：markdown frontmatter 里有 `name`、`description`，可选 `tools` 和 `model`，正文作为子进程的附加 system prompt。`~/.pi/agent/agents/*.md` 是用户级 agent，`.pi/agents/*.md` 是项目级 agent；默认只加载用户级。项目级 agent 需要设置 `agentScope: "project"` 或 `"both"`，交互模式下还会提示确认。

示例自带的 agent 有 `scout`、`planner`、`reviewer`、`worker`。它们是几份 markdown 配置，不属于 core 里的新类。

所以更准确的结论是：**pi core 没有内置 subagent/multi-agent 编排；coding-agent 默认也不带 agent 工具；但 pi 提供 extension 和 tool 机制，仓库里已经有 subagent 示例扩展，说明这类能力可以放在应用层或扩展层实现。**

为什么这样分层，事实上能从源码结构看出来：core Agent 只处理一段对话的运行，跨 agent 调度、子进程管理、agent 定义发现、安全确认、并行输出截断和 UI 渲染，都需要 coding-agent 的 CLI、extension、UI 和资源体系。把它放在 extension 层，比放进 core Agent 更符合现有边界。

---

## 核心链路

```text
new Agent(options)
  -> createMutableAgentState(options.initialState)   // 建 state
  -> 存 options 各字段（有默认的填默认，无默认的留 undefined）
agent.prompt(input)
  -> normalizePromptInput -> runPromptMessages
  -> runWithLifecycle（activeRun + abortController）
  -> runAgentLoop(messages, contextSnapshot{sys,msgs,tools}, loopConfig{options 透传}, ...)
  -> 事件经 processEvents 广播给 subscribe 的 listener
```

---

## 一句话总结

Agent 是 packages/agent 里的一个类，一个实例就是内存里的一段对话运行对象：持有 messages/systemPrompt/model/tools 等状态，维护 activeRun 和两个消息队列，暴露 prompt/continue/steer/followUp/abort 等方法，把状态交给 runAgentLoop 跑一轮。它把模型调用、上下文变换、工具 hook 做成可接线的入口；存储、资源、鉴权和 UI 设施由 AgentSession 或其他外层应用装配。Agent 管对话怎么跑，AgentSession 管对话的外围设施。
