# Pi 系列 10｜Agent 抽象：一段对话的运行对象

本系列其他文章：

placeholder

> 本文主要参考 `packages/agent/src/agent.ts`、`agent-loop.ts`、`types.ts` 和 `packages/coding-agent/src/core/sdk.ts`。

前面第 02 篇讲过 agent loop 怎样把一次 prompt 拆成多个 turn。应用入口篇讲过 `Agent`、`AgentSession`、`AgentSessionRuntime` 三层入口。这篇只看中间这一层：`Agent` 本身是什么。

在 pi 的源码里，`Agent` 首先是一个类。`new Agent(options)` 创建的是当前 JS 运行时里的一个对象。它持有一段对话的状态，提供启动、排队、控制和订阅事件的方法。模型请求、工具调用和事件流由它交给 `runAgentLoop` 处理。

源码注释对它的职责有一句概括：`Agent` owns the current transcript, emits lifecycle events, executes tools, and exposes queueing APIs for steering and follow-up messages。换成中文，就是：`Agent` 持有当前 transcript，发出生命周期事件，执行工具，并提供 steering / follow-up 消息的排队 API。

<!-- 图0：Agent 抽象总览
生图 prompt：
一张横版总览图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画面中心是一个大框「Agent」。大框内分四块：1「state：systemPrompt / messages / model / tools」；2「queues：steeringQueue / followUpQueue」；3「runtime：activeRun / AbortController」；4「hooks：convertToLlm / transformContext / streamFn / tool hooks」。
左侧画「外部调用」进入 Agent，列出 prompt / continue / steer / followUp / abort / subscribe。右侧画「runAgentLoop」节点，内部小循环标「model response → tool calls → tool results → next turn」。Agent 到 runAgentLoop 用箭头连接，标「context snapshot + loop config」。
下方画一条事件流从 runAgentLoop 回到 Agent，再到「subscribers」，标「AgentEvent：message_start / message_update / message_end / tool_execution / turn_end / agent_end」。
右下角画一个外层虚线框「AgentSession」，包住 Agent 但颜色更浅，标「存储 / 鉴权 / ResourceLoader / ExtensionRunner 在外层」。突出 Agent 是一段对话的运行对象，不是完整应用。
底部小字：「Agent 持有 transcript，发出生命周期事件，执行工具，并提供 steering/follow-up 排队 API。」
建议文件名：./pi_10_0.png
-->

## 一、Agent 是类实例，运行在当前进程里

`packages/agent/src/agent.ts` 里有 `export class Agent`。创建 Agent 实例不会启动独立 OS 进程，也不会启动后台服务。它就是当前程序内存里的一个对象。

这带来几个直接结论：

- 一个 Agent 实例通常对应一段独立对话状态。
- 一个进程里可以创建多个 Agent 实例，它们各自持有 messages、model、tools 和队列。
- core Agent 不绑定 Node 文件系统。web-ui 示例在浏览器里直接 `new Agent`，再配浏览器侧的存储、鉴权和工具。

所以讨论 Agent 时，要把两个层次分开：操作系统里的进程是一层；进程里创建的 Agent 对象是另一层。pi CLI、web-ui 或你的应用进程里，都可以创建 Agent 对象。

<!-- 图1：Agent 是进程内对象
生图 prompt：
一张横版示意图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画一个大框「当前 JS 进程 / 浏览器运行时」，里面并排放三个小卡片「Agent A」「Agent B」「Agent C」。每张卡片里列「messages / model / tools / queues」。
大框外避免画服务器或端口；下方标「new Agent(options) 创建的是当前进程里的内存对象」。
右侧加一个小对照：「pi CLI / web-ui / 你的应用」都可以在自己的进程里创建 Agent 对象。
底部小字：「Agent 是一段对话的运行对象，一个进程里可以有多个实例。」
建议文件名：./pi_10_1.png
-->

## 二、Agent state 里有什么

`createMutableAgentState` 创建的是 Agent 暴露出来的 state。主要字段如下：

```ts
state = {
  systemPrompt,
  model,
  thinkingLevel,
  tools,
  messages,
  isStreaming,
  streamingMessage,
  pendingToolCalls,
  errorMessage,
}
```

前五个更接近对话内容：当前 system prompt、模型、thinking level、工具列表和消息历史。后四个更接近运行时状态：是否正在流式输出、当前流式消息、执行中的工具调用和错误信息。

没有传 `initialState` 时，Agent 会从空 system prompt、`unknown` model、`thinkingLevel: "off"`、空 tools 和空 messages 开始。这能创建一个对象；要完成真实模型调用，还需要外层提供可用模型、凭据，或传入自定义 `streamFn`。

`tools` 和 `messages` 有 getter/setter，赋值时会拷贝顶层数组。coding-agent 恢复会话时，就是把已有 session messages 放回 `agent.state.messages`。

还有一些控制字段不在 `state` 对象里，比如 `activeRun`、steering 队列和 follow-up 队列。它们是 Agent 实例的私有字段。换句话说，`state` 是 Agent 对外暴露的主要状态，内部还另有队列、监听者和运行控制字段。

创建一个 Agent 时，它主要持有这些内容：`state`（对外暴露的对话状态）、两个消息队列（steering 和 follow-up）、事件监听者集合、一组可注入的回调（`convertToLlm`、`transformContext`、`streamFn`、`getApiKey`、`beforeToolCall` 等），以及传输和运行配置（`sessionId`、`transport`、`toolExecution` 等）。也就是说，Agent 是一个把这些状态和配置收在一起的内存对象。它不连数据库、不起服务、不占端口，也不内置应用级鉴权、存储、skill、extension 系统；这些由 AgentSession 或其他外层应用处理。

Context 篇讲过的三件套 `systemPrompt`、`messages`、`tools`，正好来自这里。Agent 每次调用 loop 前，会把这三项打包成 context snapshot。

<!-- 图2：Agent 实例里实际持有的东西
生图 prompt：
一张横版分层卡片图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中心是一个大卡片「Agent 实例」。内部划分五块：1「state：systemPrompt / model / thinkingLevel / tools / messages / isStreaming / streamingMessage / pendingToolCalls / errorMessage」；2「queues：steeringQueue / followUpQueue」；3「listeners：subscribe 注册的监听者」；4「callbacks：convertToLlm / transformContext / streamFn / beforeToolCall / afterToolCall」；5「runtime config：sessionId / transport / toolExecution」。
在 state 的 systemPrompt/messages/tools 三项旁加括号「Context 篇三件套」。
底部小字：「state 是对外暴露的主要状态；队列、监听者和 activeRun 是实例内部控制字段。」
建议文件名：./pi_10_2.png
-->

## 三、Agent 暴露哪些方法

Agent 的方法可以按用途分成四组。

### 启动一轮

- `prompt(input)`：从文本、单条消息或一批消息开始一轮。已有 active run 时会抛错。
- `continue()`：从当前 transcript 继续跑。最后一条需要是 user 或 `toolResult`；如果最后是 assistant，它会先检查 steering/follow-up 队列。

### 排队

- `steer(message)`：进入 steering 队列。当前 assistant turn 完成后，下一次 LLM 调用前注入。
- `followUp(message)`：进入 follow-up 队列。agent 本来要停下时再作为新消息继续。

两个队列都有 `one-at-a-time` 和 `all` 两种 drain 模式。第 02 篇讲 loop 时提到的 pending messages，就是从这些队列取出来的。

### 控制

- `abort()`：中止当前 active run。
- `waitForIdle()`：等待当前 run 和事件 listener 都结束。
- `reset()`：清空 messages、运行时状态和两个队列。

### 观察

- `subscribe(listener)`：订阅 `AgentEvent`，返回取消订阅函数。
- `state`：访问当前状态。

有一个约束需要注意：同一个 Agent 同一时刻只跑一个 active run。运行中要追加用户意图，应通过 `steer` 或 `followUp` 排队。

也就是说，一个 Agent 同一时刻只对应一个正在运行的 loop。每次 `prompt()` 或 `continue()` 会启动一次 active run，并在这次 run 里调用 `runAgentLoop`。这次 loop 结束后，Agent 实例仍然保留；后面再调用 `prompt()` 或 `continue()`，会再启动下一次 loop。

这里还要区分 Agent 实例和 active run 的生命周期。`new Agent(options)` 创建的是一个内存对象。只要外层应用还持有它，这个对象就还在；外层不再持有引用后，它和普通 JavaScript 对象一样，由 JS 运行时按垃圾回收规则处理。`packages/agent/src/agent.ts` 里没有 `dispose()`、`destroy()`、`shutdown()` 或 `exit()` 方法；core Agent 自己不退出进程。

一次 active run 是短生命周期的。`prompt()` 或 `continue()` 进入 `runWithLifecycle()` 后，会创建 `AbortController`，设置 `activeRun`，并把 `state.isStreaming` 设为 true。loop 结束时会发出 `agent_end`。`agent_end` 表示 loop 不再发后续事件，但 Agent 要等这个事件的 listener 都处理完，才会在 `finishRun()` 里清掉运行时状态：`isStreaming` 变回 false，`streamingMessage` 清空，`pendingToolCalls` 重置，`activeRun` 变回 undefined。

因此，run 结束不等于 Agent 销毁。run 结束后，messages、tools、systemPrompt、listeners 和队列仍然属于这个 Agent 实例。`waitForIdle()` 等的是当前 run 和 listener 处理完成；没有 active run 时会直接 resolve。`abort()` 只是中止当前 run，不清空 transcript，也不销毁 Agent。`reset()` 会清空 messages、运行时状态和队列，但对象本身仍然存在。进程退出由 coding-agent 的运行模式或外层应用决定，不属于 core Agent 的职责。

## 四、Agent 怎样连接 loop

`prompt()` 本身不直接处理模型和工具循环。它会先把输入标准化，再进入 lifecycle 包装，最后调用 `runAgentLoop`。

链路大致是：

```text
agent.prompt(input)
  -> normalizePromptInput(input)
  -> runPromptMessages(messages)
  -> runWithLifecycle(activeRun + AbortController)
  -> runAgentLoop(
       messages,
       createContextSnapshot(),
       createLoopConfig(options),
       event => processEvents(event),
       signal,
       streamFn,
     )
```

需要看清楚两个动作：

- `createContextSnapshot()` 把 `state.systemPrompt`、`state.messages`、`state.tools` 拷贝成一份 loop context。
- `createLoopConfig()` 把 model、thinking、transport、`convertToLlm`、`transformContext`、tool hooks、队列 drain 函数等传给 loop。

loop 返回的事件会经过 `processEvents`。`processEvents` 一边更新 Agent 的内部状态，例如 streaming message、pending tool calls、error message；一边按订阅顺序调用 listener。`agent_end` 是 loop 的最后事件。Agent 要等这个事件的 listener 都处理完，才算真正 idle；之后 `finishRun()` 会清掉 activeRun。

可以概括为：Agent 负责保存状态和管理生命周期；`runAgentLoop` 负责模型、工具和上下文回填的循环。

<!-- 图3：Agent 连接 runAgentLoop
生图 prompt：
一张横版流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
主线节点从左到右：agent.prompt(input) → normalizePromptInput → runWithLifecycle(activeRun + AbortController) → createContextSnapshot(systemPrompt/messages/tools) + createLoopConfig(options) → runAgentLoop → processEvents → listeners。
在 runAgentLoop 下方画一个小循环「模型响应 → 工具调用 → toolResult 回填 → 下一 turn」。
在 processEvents 旁标「更新 streamingMessage / pendingToolCalls / errorMessage，并按顺序通知 listener」。
底部小字：「Agent 管状态和生命周期；loop 管模型、工具和上下文回填。」
建议文件名：./pi_10_3.png
-->

## 五、为什么 core Agent 只保留基础能力

Agent 构造函数接收一组 `AgentOptions`。其中一部分有默认值，一部分默认留空：

```ts
this.convertToLlm = options.convertToLlm ?? defaultConvertToLlm;
this.streamFn = options.streamFn ?? streamSimple;
this.transformContext = options.transformContext;
this.beforeToolCall = options.beforeToolCall;
this.afterToolCall = options.afterToolCall;
this.getApiKey = options.getApiKey;
this.prepareNextTurn = options.prepareNextTurn;
this.toolExecution = options.toolExecution ?? "parallel";
```

`defaultConvertToLlm` 只做基础过滤，把 user、assistant、toolResult 这几类消息交给模型。默认 `streamFn` 是 `streamSimple`，走标准 provider 请求。

需要注意：有默认 `streamFn` 不代表一个空配置的 Agent 就能完成真实模型调用。真实调用仍需要可用 model、凭据或自定义 `streamFn`。在应用里，通常由外层提供这些输入。

coding-agent 的 `createAgentSession` 就是外层装配的例子：

- 用自己的 `convertToLlm` 处理 coding-agent 的 custom message、bash execution、summary 等消息类型。
- 用 `streamFn` 闭包从 `ModelRegistry` 取 API key 和 headers，再调用 `streamSimple`。
- 用 `transformContext` 接 extension 的 `context` 事件。
- 在 `AgentSession` 里把 `beforeToolCall` / `afterToolCall` 接到 extension 的 `tool_call` / `tool_result`。

这些选项可以按用途分成几类：

| 类别 | 字段 | 作用 |
|---|---|---|
| 上下文转换 | `convertToLlm`、`transformContext` | 把 AgentMessage 转成模型消息，或在转换前改上下文 |
| Provider 请求 | `streamFn`、`getApiKey`、`onPayload`、`onResponse`、`transport`、`sessionId`、`thinkingBudgets` | 控制请求如何发、key 如何取、payload/response 如何观察 |
| 工具执行 | `beforeToolCall`、`afterToolCall`、`toolExecution` | 拦截工具、改工具结果、选择 parallel/sequential 策略 |
| 下一轮准备 | `prepareNextTurn` | turn 结束后、下一次 provider 请求前，替换 context/model/thinkingLevel |
| 队列语义 | `steeringMode`、`followUpMode` | 控制 steering/follow-up 一次 drain 一条或 drain 所有队列项 |

还需要注意：底层 loop 的 `prepareNextTurn` 会拿到 turn context；`AgentOptions.prepareNextTurn` 这一层当前只接收 signal。也就是说，直接用 `new Agent` 时，这个 hook 更适合做不依赖上一轮细节的下一轮准备。要基于 `message`、`toolResults` 或完整 context 做判断，需要使用更低层 loop 或外层 harness 的接法。

core Agent 只保留基础能力，是因为它只关心对话如何跑。存储、资源发现、settings、UI、extension runtime，都在外层包装里处理。

## 六、Agent 和 AgentSession 的分工

把两者放在一起看，分工比较清楚：

| | Agent | AgentSession |
|---|---|---|
| 角色 | 对话运行对象 | Agent 加一整套 coding-agent 设施 |
| 持有 | messages、systemPrompt、model、tools、队列、activeRun | 一个 Agent，再加 SessionManager、ResourceLoader、ExtensionRunner 等 |
| 鉴权 | 可通过 `streamFn` / `getApiKey` 接入，默认没有应用级凭据管理 | `streamFn` 闭包接 ModelRegistry |
| 会话持久化 | 没有 | 有，SessionManager 写 JSONL 树 |
| skill / extension | 没有默认资源系统 | 有，ResourceLoader 和 ExtensionRunner 接入 |
| system prompt 重建 | 由外部设置 `state.systemPrompt` | `_rebuildSystemPrompt` 从工具、skills、project context 等拼装 |

一句话：Agent 管一段对话怎么跑，AgentSession 管这段对话的外围设施。`createAgentSession` 内部先创建 Agent，再用 AgentSession 包一层。

这种分层让同一个 Agent 可以被不同环境复用。Node CLI 可以用 AgentSession 装资源和鉴权；浏览器可以直接用 Agent，再配自己的 IndexedDB、API key prompt 和 sandbox tools；测试也可以传入假的 `streamFn` 和工具，单独验证 loop 行为。

<!-- 图4：Agent 与 AgentSession 边界
生图 prompt：
一张横版嵌套图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
内层方框「Agent」列：messages、systemPrompt、model、tools、queues、activeRun、runAgentLoop 入口。
外层方框「AgentSession」包住 Agent，列：SessionManager、ModelRegistry、ResourceLoader、ExtensionRunner、内置工具、_rebuildSystemPrompt、compaction。
左侧画「web-ui」箭头直接连到 Agent，标「浏览器自己补存储/鉴权/工具」。右侧画「coding-agent CLI」箭头连到 AgentSession，标「Node 端设施齐全」。
底部小字：「Agent 管对话怎么跑；AgentSession 管这段对话的外围设施。」
建议文件名：./pi_10_4.png
-->

## 七、subagent 和 multi-agent 放在哪一层

理解了 Agent 负责什么，接下来会遇到一个问题：pi 支不支持 subagent 或 multi-agent。

答案要分层看。

在 core Agent 层，`Agent` 类内部不会创建另一个 Agent，也没有 agent pool、swarm、orchestrator 这类编排结构。`packages/agent` 处理的是一段对话的状态、队列、事件和 loop。

在 coding-agent 默认工具层，内置工具是 read、bash、edit、write、grep、find、ls 这类本地工具。默认 active tools 里没有 Claude Code 那种直接派生子 agent 的 Agent/Task 工具。

但仓库里有一个可运行的 subagent 示例扩展：`packages/coding-agent/examples/extensions/subagent/`。该 extension 注册 `subagent` 自定义工具。工具执行时，它会为子任务启动独立的 `pi` 子进程，让子任务有自己的上下文窗口。

实现链路大致是这样：extension 加载时注册 `subagent` 工具；模型调用这个工具时，工具先按 scope 发现 agent 定义，再把选中的 agent system prompt 写入临时文件，最后启动独立 `pi --mode json -p --no-session ...` 子进程。父进程读取子进程 stdout 里的 JSON 事件，收集 assistant 消息、工具结果、usage、stopReason，再把最终输出作为 `subagent` 工具结果返回给父 agent。

文中的“子 agent”指独立的 `pi` 子进程，有自己的上下文窗口、模型参数、工具配置和输出流。父 agent 不直接共享它的 transcript，只收到工具结果和 details。

该示例支持 single、parallel、chain 三种模式。三种模式共用同一个 `runSingleAgent` 执行单元，差别在父工具怎样组织多个子进程。

### single：一个 agent 处理一个 task

single 可以用一个具体例子理解。父模型调用 `subagent` 工具时，参数可能是：

```json
{ "agent": "scout", "task": "Find where authentication is implemented" }
```

工具会去 agent 定义目录里找 `scout.md`。假设这份文件是：

```markdown
---
name: scout
description: Fast codebase recon
tools: read, grep, find, ls, bash
model: claude-haiku-4-5
---
You are a fast codebase scout. Return compact findings with file paths.
```

工具会把它翻译成一次子进程调用：

```text
pi --mode json -p --no-session \
  --model claude-haiku-4-5 \
  --tools read,grep,find,ls,bash \
  --append-system-prompt /tmp/pi-subagent-xxx/prompt-scout.md \
  "Task: Find where authentication is implemented"
```

临时 prompt 文件里放的是 `scout.md` 的正文。子进程运行时会自己构造上下文、调用工具、输出 JSON 事件。父进程不共享它的 transcript，只读取这些 JSON 事件并整理结果。

父进程读取子进程 stdout 里的 JSON 事件，收集 assistant 消息、工具结果、usage 和 stopReason。子进程成功时，父工具返回最后一条 assistant 文本；失败时，返回 errorMessage、stderr 或最后输出。

<!-- 图5：subagent single 模式
生图 prompt：
一张横版流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
节点依次是：父模型调用 `subagent(agent, task)` → subagent 工具 → 发现 agent markdown → 写临时 system prompt → 启动 `pi --mode json -p --no-session` 子进程 → 读取 JSON 事件 → 返回最终文本给父模型。
用单条粗箭头强调「一个 task → 一个子进程 → 一个结果」。
底部小字：「single 模式把一次委派变成一次独立 pi 子进程调用。」
建议文件名：./pi_10_5.png
-->

### parallel：多个 agent 并行处理多个 task

parallel 的输入是一组任务，例如：

```json
{
  "tasks": [
    { "agent": "scout", "task": "Find model resolution code" },
    { "agent": "scout", "task": "Find provider registration code" },
    { "agent": "reviewer", "task": "Review auth-related code paths" }
  ]
}
```

每个 task 都会走 single 的执行路径，各自启动一个独立 `pi` 子进程。代码先检查任务数，最多 8 个；执行时用 `mapWithConcurrencyLimit` 控制并发，最多 4 个子进程同时运行。

父工具会维护一个 `allResults` 数组。每个子进程有新消息时，对应位置会被更新，并通过 `onUpdate` 把进度发给 UI，例如“2/3 done, 1 running”。所有任务结束后，父工具把每个 task 的输出整理成小结返回给父模型。为了控制返回给父模型的上下文量，每个 task 的可见输出最多 50 KB；完整消息、stderr、usage 等保存在 tool details。

<!-- 图6：subagent parallel 模式
生图 prompt：
一张横版分叉汇总图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左边是父 agent，中间是 `subagent` 工具，右边并排 3 到 4 个 `pi` 子进程小框，标「最多 4 个并发 / 最多 8 个任务」。
每个子进程箭头回到一个 `allResults` 汇总表，汇总表再输出「父模型可见小结」。旁边加标注：「每个 task 返回给模型最多 50 KB；完整消息、stderr、usage 在 details」。
底部小字：「parallel 模式并行跑多个独立子进程，父工具负责进度和结果汇总。」
建议文件名：./pi_10_6.png
-->

### chain：多个 agent 按顺序接力

chain 的输入是一条步骤列表，例如：

```json
{
  "chain": [
    { "agent": "scout", "task": "Find the read tool implementation" },
    { "agent": "planner", "task": "Use these findings to propose a refactor plan:\n{previous}" },
    { "agent": "worker", "task": "Implement the plan:\n{previous}" }
  ]
}
```

它一次只跑一个子进程。第一步 `scout` 完成后，代码取它的最终输出，存进 `previousOutput`。第二步 `planner` 的 task 里有 `{previous}`，执行前会替换成 scout 的输出。第三步同理，会拿到 planner 的输出。

如果某一步失败，chain 会停止，返回失败发生在哪一步、哪个 agent 失败、失败输出是什么，并把已经完成的步骤放进 details。所有步骤成功时，父工具返回最后一步的最终输出。

<!-- 图7：subagent chain 模式
生图 prompt：
一张横版线性接力图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画三个步骤：Step 1 `scout`，Step 2 `planner`，Step 3 `worker`。每一步下面都有一个独立 `pi` 子进程小框。
从 scout 输出箭头进入 planner 的 `{previous}`，从 planner 输出箭头进入 worker 的 `{previous}`。在线路旁标「成功：输出传给下一步」。另画一条红色细分支「失败：chain 停止，返回已完成 details」。
底部小字：「chain 模式一次只跑一个子进程，用 {previous} 把上一步输出交给下一步。」
建议文件名：./pi_10_7.png
-->

它还定义了自己的 agent 配置文件。agent 是 markdown 文件，frontmatter 里有 `name`、`description`，可选 `tools` 和 `model`，正文作为子进程的附加 system prompt。用户级 agent 放在 `~/.pi/agent/agents/*.md`，项目级 agent 放在 `.pi/agents/*.md`。项目级 agent 涉及仓库控制的 prompt，示例里默认不加载，需要显式 scope，并在交互模式下确认。示例自带 `scout`、`planner`、`reviewer`、`worker` 几个 agent 定义，它们是配置文件，不属于 core 里的新类。

更具体地说：pi core 没有把 subagent/multi-agent 做成内置运行模型；coding-agent 默认也没有启用 agent 工具；但 extension 机制可以实现这类编排，仓库里的 subagent 示例说明这条路可行。

这样放也有原因。core Agent 只处理一段对话。跨 agent 调度、子进程管理、agent 定义发现、安全确认、并行结果汇总、输出截断和 UI 渲染，都需要 CLI、extension、UI 和资源体系。放在 extension 层，和 pi 现有的分工更一致。

## 八、主链路

把 Agent 的主路径整理成一张文字图：

```text
new Agent(options)
  -> createMutableAgentState(options.initialState)
  -> 保存 convertToLlm / streamFn / hooks / queue mode 等配置

agent.prompt(input)
  -> normalizePromptInput
  -> runWithLifecycle(activeRun + AbortController)
  -> runAgentLoop(messages, contextSnapshot, loopConfig, emit, signal, streamFn)
  -> processEvents 更新 state 并广播给 listener
  -> finishRun 清理 activeRun 和运行时状态
```

<!-- 图8：Agent 主链路
生图 prompt：
一张横版主链路流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
从左到右画主线：`new Agent(options)` → `createMutableAgentState` → 保存 `convertToLlm / streamFn / hooks / queue mode` → `agent.prompt(input)` 或 `agent.continue()` → `runWithLifecycle(activeRun + AbortController)` → `runAgentLoop(contextSnapshot + loopConfig)` → `processEvents 更新 state 并通知 listener` → `finishRun 清理 activeRun`。
在 `runAgentLoop` 节点下方画一个小循环：「assistant message → tool calls → tool results → next turn」。在 `finishRun` 后画一个回到「Agent 实例仍存在」的小箭头，标注「后续还可以再次 prompt / continue」。
右侧用小分支表示生命周期边界：「agent_end：loop 不再发事件」「waitForIdle：等待 listener 完成」「abort：只中止当前 run」「reset：清空 transcript 和队列，不销毁对象」。
底部小字：「Agent 对象可以长期存在；active run 是一次短生命周期的执行。」
建议文件名：./pi_10_8.png
-->

Agent 是对话运行对象。它把一段 transcript、当前模型、工具、队列和生命周期控制收在一起。每次启动 active run 时，它再把实际 turn loop 交给 `runAgentLoop`。理解这一层之后，前面的 loop、context、tool，后面的 AgentSession 和应用入口，就能对应起来。