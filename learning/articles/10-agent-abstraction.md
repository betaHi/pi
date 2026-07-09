# Pi 系列 10｜Agent 抽象：一段对话的运行对象

本系列其他文章：

placeholder

> 本文主要参考 `packages/agent/src/agent.ts`、`agent-loop.ts`、`types.ts` 和 `packages/coding-agent/src/core/sdk.ts`。

前面第 02 篇讲过 agent loop 怎样把一次 prompt 拆成多个 turn。应用入口篇讲过 `Agent`、`AgentSession`、`AgentSessionRuntime` 三层入口。这里补中间那层最基础的问题：`Agent` 本身是什么。

在 pi 的源码里，`Agent` 首先是一个类。`new Agent(options)` 创建的是当前 JS 运行时里的一个对象。它持有一段对话的状态，提供启动、排队、控制和订阅事件的方法。模型请求、工具调用和事件流由它交给 `runAgentLoop` 处理。

## 一、Agent 是类实例，运行在当前进程里

`packages/agent/src/agent.ts` 里有 `export class Agent`。创建 Agent 实例不会启动独立 OS 进程，也不会启动后台服务。它就是当前程序内存里的一个对象。

这带来几个直接结论：

- 一个 Agent 实例通常对应一段独立对话状态。
- 一个进程里可以创建多个 Agent 实例，它们各自持有 messages、model、tools 和队列。
- core Agent 不绑定 Node 文件系统。web-ui 示例在浏览器里直接 `new Agent`，再配浏览器侧的存储、鉴权和工具。

所以讨论 Agent 时，要把两个层次分开：操作系统里的进程是一层；进程里创建的 Agent 对象是另一层。pi CLI、web-ui 或你的应用进程里，都可以创建 Agent 对象。

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

还有一些控制字段不在 `state` 对象里，比如 `activeRun`、steering 队列和 follow-up 队列。它们是 Agent 实例的私有字段。换句话说，`state` 是 Agent 对外暴露的主要状态，不是这个对象内部所有字段的完整列表。

把创建一个 Agent 时实际持有的东西列全，是这几类：`state`（对话状态，唯一对外暴露的）、两个消息队列（steering 和 follow-up）、事件监听者集合、一组可注入的回调（`convertToLlm`、`transformContext`、`streamFn`、`getApiKey`、`beforeToolCall` 等），以及传输和运行配置（`sessionId`、`transport`、`toolExecution` 等）。所以创建一个 Agent，就是在内存里创建一个把这些打包在一起的对象。它不连数据库、不起服务、不占端口，也不内置应用级鉴权、存储、skill、extension 系统——这些是 AgentSession 或其他外层应用装的。

Context 篇讲过的三件套 `systemPrompt`、`messages`、`tools`，正好来自这里。Agent 每次调用 loop 前，会把这三项打包成 context snapshot。

## 三、Agent 暴露哪些方法

Agent 的方法可以按用途分成四组。

### 启动一轮

- `prompt(input)`：从文本、单条消息或一批消息开始一轮。已有 active run 时会抛错。
- `continue()`：从当前 transcript 继续跑。最后一条不是 user 或 `toolResult` 时会抛错；如果最后是 assistant，它会先检查 steering/follow-up 队列。

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

这里的一个重要约束是：同一个 Agent 同一时刻只跑一个 active run。运行中要追加用户意图，应通过 `steer` 或 `followUp` 排队。

## 四、Agent 怎样连接 loop

`prompt()` 本身不直接写模型和工具循环。它会先把输入标准化，再进入 lifecycle 包装，最后调用 `runAgentLoop`。

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

两个打包动作很关键：

- `createContextSnapshot()` 把 `state.systemPrompt`、`state.messages`、`state.tools` 拷贝成一份 loop context。
- `createLoopConfig()` 把 model、thinking、transport、`convertToLlm`、`transformContext`、tool hooks、队列 drain 函数等传给 loop。

loop 返回的事件会经过 `processEvents`。`processEvents` 一边更新 Agent 的内部状态，例如 streaming message、pending tool calls、error message；一边按订阅顺序调用 listener。`agent_end` 是 loop 的最后事件，Agent 真正 idle 要等这个事件的 listener 都处理完，`finishRun()` 再清掉 activeRun。

所以可以这样理解：Agent 负责保存状态和管理生命周期；`runAgentLoop` 负责模型、工具和上下文回填的循环。

## 五、为什么 core Agent 保持轻

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

这里要注意边界：有默认 `streamFn` 不代表一个空配置的 Agent 就能完成真实模型调用。真实调用仍需要可用 model、凭据或自定义 `streamFn`。在应用里，通常由外层提供这些东西。

coding-agent 的 `createAgentSession` 就是外层装配的例子：

- 用自己的 `convertToLlm` 处理 coding-agent 的 custom message、bash execution、summary 等消息类型。
- 用 `streamFn` 闭包从 `ModelRegistry` 取 API key 和 headers，再调用 `streamSimple`。
- 用 `transformContext` 接 extension 的 `context` 事件。
- 在 `AgentSession` 里把 `beforeToolCall` / `afterToolCall` 接到 extension 的 `tool_call` / `tool_result`。

这些接线口可以按用途理解：

| 类别 | 字段 | 作用 |
|---|---|---|
| 上下文转换 | `convertToLlm`、`transformContext` | 把 AgentMessage 转成模型消息，或在转换前改上下文 |
| Provider 请求 | `streamFn`、`getApiKey`、`onPayload`、`onResponse`、`transport`、`sessionId`、`thinkingBudgets` | 控制请求如何发、key 如何取、payload/response 如何观察 |
| 工具执行 | `beforeToolCall`、`afterToolCall`、`toolExecution` | 拦截工具、改工具结果、选择 parallel/sequential 策略 |
| 下一轮准备 | `prepareNextTurn` | turn 结束后、下一次 provider 请求前，替换 context/model/thinkingLevel |
| 队列语义 | `steeringMode`、`followUpMode` | 控制 steering/follow-up 一次 drain 一条还是全部 drain |

这里有个小边界：底层 loop 的 `prepareNextTurn` 会拿到 turn context；`AgentOptions.prepareNextTurn` 这一层当前只接收 signal。也就是说，直接用 `new Agent` 时，这个 hook 更适合做不依赖上一轮细节的下一轮准备。要基于 `message`、`toolResults` 或完整 context 做判断，需要看更低层 loop 或外层 harness 的接法。

core Agent 保持轻，是因为它只关心对话如何跑。存储、资源发现、settings、UI、extension runtime，都在外层包装里处理。

## 六、Agent 和 AgentSession 的边界

把两者放在一起看，边界比较清楚：

| | Agent | AgentSession |
|---|---|---|
| 角色 | 对话运行对象 | Agent 加一整套 coding-agent 设施 |
| 持有 | messages、systemPrompt、model、tools、队列、activeRun | 一个 Agent，再加 SessionManager、ResourceLoader、ExtensionRunner 等 |
| 鉴权 | 可通过 `streamFn` / `getApiKey` 接入，默认没有应用级凭据管理 | `streamFn` 闭包接 ModelRegistry |
| 会话持久化 | 没有 | 有，SessionManager 写 JSONL 树 |
| skill / extension | 没有默认资源系统 | 有，ResourceLoader 和 ExtensionRunner 接入 |
| system prompt 重建 | 由外部设置 `state.systemPrompt` | `_rebuildSystemPrompt` 从工具、skills、project context 等拼装 |

一句话：Agent 管一段对话怎么跑，AgentSession 管这段对话的外围设施。`createAgentSession` 内部先创建 Agent，再用 AgentSession 包一层。

这个分层让同一个 Agent 可以被不同环境复用。Node CLI 可以用 AgentSession 装资源和鉴权；浏览器可以直接用 Agent，再配自己的 IndexedDB、API key prompt 和 sandbox tools；测试也可以传入假的 `streamFn` 和工具，单独验证 loop 行为。

## 七、subagent 和 multi-agent 放在哪一层

理解了 Agent 的职责边界，一个自然的问题是：pi 支不支持 subagent 或 multi-agent。

答案要分层看。

在 core Agent 层，`Agent` 类内部不会创建另一个 Agent，也没有 agent pool、swarm、orchestrator 这类编排结构。`packages/agent` 处理的是一段对话的状态、队列、事件和 loop。

在 coding-agent 默认工具层，内置工具是 read、bash、edit、write、grep、find、ls 这类本地工具。默认 active tools 里没有 Claude Code 那种直接派生子 agent 的 Agent/Task 工具。

但仓库里有一个真实的 subagent 示例扩展：`packages/coding-agent/examples/extensions/subagent/`。这个 extension 注册 `subagent` 自定义工具。工具执行时，它会为子任务启动独立的 `pi` 子进程，让子任务有自己的上下文窗口。

实现链路大致是这样：extension 加载时注册 `subagent` 工具；模型调用这个工具时，工具先按 scope 发现 agent 定义，再把选中的 agent system prompt 写入临时文件，最后启动独立 `pi --mode json -p --no-session ...` 子进程。父进程读取子进程 stdout 里的 JSON 事件，收集 assistant 消息、工具结果、usage、stopReason，再把最终输出作为 `subagent` 工具结果返回给父 agent。

这里的“子 agent”不是当前 `Agent` 实例内部创建的对象。它是一个独立的 `pi` 子进程，有自己的上下文窗口、模型参数、工具配置和输出流。父 agent 不直接共享它的 transcript，只收到工具结果和 details。

这个示例支持 single、parallel、chain 三种模式。single 是一个 agent 做一个任务；parallel 是多个 agent 并行处理多个任务，代码里限制最多 8 个任务、4 个并发；chain 是按顺序跑多个 agent，后一步可以引用前一步输出。parallel 返回给父模型的每个任务输出有 50 KB 上限，完整结果保留在 tool details。

它还定义了自己的 agent 配置文件。agent 是 markdown 文件，frontmatter 里有 `name`、`description`，可选 `tools` 和 `model`，正文作为子进程的附加 system prompt。用户级 agent 放在 `~/.pi/agent/agents/*.md`，项目级 agent 放在 `.pi/agents/*.md`。项目级 agent 涉及仓库控制的 prompt，示例里默认不加载，需要显式 scope，并在交互模式下确认。示例自带 `scout`、`planner`、`reviewer`、`worker` 几个 agent 定义，它们是配置文件，不是 core 里的新类。

所以更准确的说法是：pi core 没有把 subagent/multi-agent 做成内置运行模型；coding-agent 默认也没有启用 agent 工具；但 extension 机制可以实现这类编排，仓库里的 subagent 示例已经证明了这一点。

这个分层也解释了原因。core Agent 只处理一段对话。跨 agent 调度、子进程管理、agent 定义发现、安全确认、并行结果汇总、输出截断和 UI 渲染，都需要 CLI、extension、UI 和资源体系。放在 extension 层，和 pi 现有的职责边界更一致。

## 八、核心链路

把 Agent 的主路径压成一张文字图：

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

Agent 不是一个外部服务，它是对话运行对象。它把一段 transcript、当前模型、工具、队列和生命周期控制收在一起，再把实际 turn loop 交给 `runAgentLoop`。理解这一层之后，前面的 loop、context、tool，后面的 AgentSession 和应用入口，就能连成一条线。

---

*本文基于对 `@earendil-works/pi-agent-core` 和 `@earendil-works/pi-coding-agent` 源码的阅读。*