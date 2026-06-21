# 调试阅读实验

调试目标不是一次看完整个项目，而是用一个可复现动作追一条调用链。每次只观察一个问题：消息如何进入、事件如何产生、工具如何执行、session 如何落盘、UI 如何更新。

这些 lab 是把源码知识变成工程直觉的地方。它们优先服务学习和文章证据，不要求全部写进文章正文。

## 准备方式

优先使用最小 harness 调试核心逻辑：

- 不依赖真实 provider。
- 不需要跑 dev server。
- 用自定义 `streamFn` 稳定产生 `text`、`toolCall`、`error` 三类事件。
- 先调 `packages/agent` 和 `packages/ai` 边界，再进入 `packages/coding-agent`。

真实 CLI/TUI 调试放在第二阶段，因为它会引入终端输入、渲染、配置、认证和 provider 网络请求，噪声更大。

## Lab 1：一次普通 prompt

目标：看清用户输入如何变成 LLM 请求，再变成 agent 事件。

断点顺序：

1. `packages/coding-agent/src/core/agent-session.ts`
   - `prompt()`
   - `_runAgentPrompt()`
   - `_handleAgentEvent()`
2. `packages/coding-agent/src/core/sdk.ts`
   - `createAgentSession()`
   - `new Agent({ ... })`
   - `convertToLlmWithBlockImages`
   - `streamFn`
3. `packages/agent/src/agent.ts`
   - `prompt()`
   - `runPromptMessages()`
   - `createLoopConfig()`
   - `processEvents()`
4. `packages/agent/src/agent-loop.ts`
   - `runAgentLoop()`
   - `runLoop()`
   - `streamAssistantResponse()`
5. `packages/ai/src/stream.ts`
   - `streamSimple()`
6. `packages/ai/src/api-registry.ts`
   - `getApiProvider()`

观察变量：

- `AgentSession.prompt()` 里的 `expandedText`
- 传入 `Agent.prompt()` 的 `messages`
- `Agent.createContextSnapshot()` 的 `systemPrompt/messages/tools`
- `streamAssistantResponse()` 中的 `messages`、`llmMessages`、`llmContext`
- `Agent.processEvents()` 对 `state.messages`、`streamingMessage`、`isStreaming` 的更新

应该得到的链路：

```text
user input
  -> AgentSession.prompt()
  -> Agent.prompt()
  -> runAgentLoop()
  -> transformContext()
  -> convertToLlm()
  -> streamSimple()
  -> AssistantMessageEvent
  -> AgentEvent
  -> AgentSessionEvent
```

## Lab 2：一次工具调用

目标：看清 model tool call 如何执行，并如何进入下一轮 LLM。

建议先用 fake/custom `streamFn` 返回一个 assistant message，内容包含 `toolCall`。避免真实模型有不稳定输出。

断点顺序：

1. `packages/agent/src/agent-loop.ts`
   - `executeToolCalls()`
   - `executeToolCallsSequential()`
   - `executeToolCallsParallel()`
   - `prepareToolCall()`
   - `executePreparedToolCall()`
   - `finalizeExecutedToolCall()`
   - `createToolResultMessage()`
2. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
   - `wrapToolDefinition()`
3. 任选一个内置工具：
   - `packages/coding-agent/src/core/tools/read.ts`
   - `packages/coding-agent/src/core/tools/bash.ts`
   - `packages/coding-agent/src/core/tools/edit.ts`
   - `packages/coding-agent/src/core/tools/write.ts`
4. `packages/coding-agent/src/core/agent-session.ts`
   - `_installAgentToolHooks()`
   - `_handleAgentEvent()`

观察变量：

- `assistantMessage.content.filter((c) => c.type === "toolCall")`
- `toolCall.arguments`
- `validatedArgs`
- `beforeToolCall` 返回值
- `AgentTool.execute()` 的 result
- `afterToolCall` 返回值
- 生成的 `ToolResultMessage`

应该得到的链路：

```text
assistant toolCall
  -> tool_execution_start
  -> prepareArguments()
  -> validateToolArguments()
  -> beforeToolCall hook
  -> tool.execute()
  -> tool_execution_update
  -> afterToolCall hook
  -> tool_execution_end
  -> ToolResultMessage
  -> next turn
```

重点验证：

- parallel 模式下，`tool_execution_end` 可能按完成顺序发出。
- 但最终 `ToolResultMessage` 仍按 assistant 原始 tool call 顺序写入。
- 如果任何 tool 标记 `executionMode: "sequential"`，整批 tool calls 顺序执行。

## Lab 3：session 落盘和恢复

目标：理解 JSONL session tree，而不是把 session 当普通数组。

断点顺序：

1. `packages/coding-agent/src/core/agent-session.ts`
   - `_handleAgentEvent()`
2. `packages/coding-agent/src/core/session-manager.ts`
   - `appendMessage()`
   - `_appendEntry()`
   - `_persist()`
   - `buildSessionContext()`
   - `getBranch()`
   - `branch()`
   - `createBranchedSession()`
3. `packages/coding-agent/src/core/sdk.ts`
   - `createAgentSession()` 中恢复 existing session 的逻辑

同时打开当前 session JSONL 文件，观察：

- header entry：`type: "session"`
- message entry：`type: "message"`
- model entry：`type: "model_change"`
- thinking entry：`type: "thinking_level_change"`
- compaction entry：`type: "compaction"`
- label entry：`type: "label"`
- 每条 entry 的 `id`、`parentId`

应该得到的结构：

```text
session JSONL entries
  -> byId index
  -> leafId
  -> getBranch()
  -> buildSessionContext()
  -> Agent.state.messages
```

重点验证：

- session 是 append-only。
- `/tree` 改的是当前 leaf，不删除旧 entry。
- `/fork` 是把一条 branch path 复制成新 session。
- compaction 不删除历史，只改变后续 context rebuild 的结果。

## Lab 4：context 构建、compaction 和 retry

目标：理解 LLM context 如何构建，以及长上下文和临时 provider 错误如何恢复。

断点顺序：

1. `packages/coding-agent/src/core/agent-session.ts`
   - `_buildSystemPrompt()`
   - `_runAgentPrompt()`
   - `_handlePostAgentRun()`
   - `_checkCompaction()`
   - `_runAutoCompaction()`
   - `compact()`
   - `_isRetryableError()`
   - `_prepareRetry()`
2. `packages/coding-agent/src/core/system-prompt.ts`
   - `buildSystemPrompt()`
3. `packages/coding-agent/src/core/messages.ts`
   - coding-agent message 到 LLM message 的转换路径
4. `packages/agent/src/agent-loop.ts`
   - `streamAssistantResponse()`
5. `packages/coding-agent/src/core/compaction/compaction.ts`
   - `prepareCompaction()`
   - `compact()`
   - `shouldCompact()`
6. `packages/ai/src/utils/overflow.ts`
   - `isContextOverflow()`

观察变量：

- `BuildSystemPromptOptions`
- `context.messages`
- `llmMessages`
- `llmContext`
- `assistantMessage.stopReason`
- `assistantMessage.errorMessage`
- `contextWindow`
- `contextTokens`
- `settingsManager.getCompactionSettings()`
- `_overflowRecoveryAttempted`
- `_retryAttempt`
- agent state 中最后一条 assistant message 是否被移除

关键点：

- overflow recovery 会 compact 后自动 retry 一次。
- threshold auto-compaction 不一定自动 retry，只在队列里还有消息时继续。
- retry 会把 error assistant message 从 `agent.state.messages` 移除，但 session 历史仍保留。
- 这样下一次 LLM 请求不会重复带上失败响应，但用户仍能看到历史。

## Lab 5：TUI 事件渲染

目标：看清 agent event 如何变成终端 UI。

断点顺序：

1. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
   - `init()`
   - `rebindCurrentSession()`
   - session event listener
   - `setupEditorSubmitHandler()`
   - `setupKeyHandlers()`
2. `packages/coding-agent/src/modes/interactive/components/assistant-message.ts`
3. `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
4. `packages/coding-agent/src/modes/interactive/components/footer.ts`
5. `packages/tui/src/tui.ts`
   - `TUI.start()`
   - `TUI.requestRender()`
   - `TUI.render()`
   - `Container.render()`
   - `setFocus()`

观察变量：

- event type
- message component 是否新增
- tool call id 和 pending tool map
- editor 是否被替换或恢复
- footer data 是否更新
- TUI component tree 的 children

重点验证：

- TUI 不直接调用 provider。
- TUI 通过 `AgentSessionEvent` 更新组件。
- 扩展 UI context 可以插入 widget、footer、overlay、自定义 editor。

## Lab 6：Web UI 事件渲染

目标：理解 Web UI 如何复用同一套 agent core。

断点顺序：

1. `packages/web-ui/src/components/AgentInterface.ts`
   - `setupSessionSubscription()`
   - `sendMessage()`
   - `renderMessages()`
2. `packages/web-ui/src/components/StreamingMessageContainer.ts`
3. `packages/web-ui/src/components/MessageList.ts`
4. `packages/web-ui/src/ChatPanel.ts`
   - `setAgent()`
5. `packages/web-ui/src/utils/proxy-utils.ts`
   - `createStreamFn()`
   - `applyProxyIfNeeded()`

重点验证：

- Web UI 订阅的是 `AgentEvent`，不是 `AgentSessionEvent`。
- streaming message 单独渲染，避免稳定消息列表频繁重绘。
- 浏览器环境可能需要 CORS proxy。
- `ChatPanel` 通过 tools 把 artifacts 接入 agent。

## Lab 7：扩展 hook

目标：理解 extension 是如何插入 agent 生命周期的。

断点顺序：

1. `packages/coding-agent/src/core/resource-loader.ts`
   - `reload()`
   - `extendResources()`
2. `packages/coding-agent/src/core/extensions/loader.ts`
   - extension loading
3. `packages/coding-agent/src/core/extensions/runner.ts`
   - `bindCore()`
   - `emit()`
   - `emitInput()`
   - `emitContext()`
   - `emitBeforeAgentStart()`
   - `emitToolCall()`
   - `emitToolResult()`
4. `packages/coding-agent/src/core/agent-session.ts`
   - `_bindExtensionCore()`
   - `_applyExtensionBindings()`
   - `_refreshToolRegistry()`
   - `reload()`

重点验证：

- extension command 在 `AgentSession.prompt()` 早期执行。
- `input` hook 可以 handled 或 transform。
- `context` hook 发生在每次 LLM 请求前。
- `before_agent_start` 发生在用户 prompt 已准备好、agent run 还没开始时。
- `tool_call` 可以 block 工具调用。
- `tool_result` 可以改写工具结果。
- reload/session replacement 后旧 context 会被 invalidate。

## Lab 8：Skill 加载与调用

目标：看清 skill 如何从 markdown 文件变成 system prompt 中的可发现能力，以及 `/skill:name` 如何显式展开成用户消息。

断点顺序：

1. `packages/coding-agent/src/core/resource-loader.ts`
   - `reload()`
   - `updateSkillsFromPaths()`
2. `packages/coding-agent/src/core/skills.ts`
   - `loadSkills()`
   - `loadSkillsFromDir()`
   - `loadSkillFromFile()`
   - `formatSkillsForPrompt()`
3. `packages/coding-agent/src/core/system-prompt.ts`
   - `buildSystemPrompt()`
4. `packages/coding-agent/src/core/agent-session.ts`
   - `_expandSkillCommand()`
   - `prompt()`
   - `parseSkillBlock()`

重点验证：

- skill frontmatter 里的 `name`、`description` 如何进入 `Skill`。
- system prompt 里只出现 skill 的 name/description/location，不直接塞全文。
- `/skill:name args` 会读取 skill 文件，把正文包成 `<skill>` block，再接上用户参数。
- `disable-model-invocation` 的 skill 不进入 `<available_skills>`，但仍可显式调用。

## Lab 9：最小 memory 实验

目标：验证 pi 提供的是 memory 基座，不是长期记忆系统本身。自己实现一个最小闭环：从 session 中提取事实，存入外部记忆，再在下一次请求注入。

建议实验：

1. 用 `AgentSession.subscribe()` 监听 `message_end` 或 `agent_end`。
2. 从本次 run 的消息中人工或用小模型总结一条偏好。
3. 写入一个临时 markdown/json 文件，例如 `/tmp/pi-memory.md`。
4. 写一个 extension `context` hook，在每次 LLM 请求前读取这个文件并追加 custom message。
5. 发起下一次 prompt，观察模型是否使用了这条记忆。

重点验证：

- `InMemorySessionRepo` 只是内存 session storage，不是长期 memory。
- session 负责历史和分支，memory 策略负责抽取、检索、去重、确认和注入。
- memory 不等于自我进化；它只负责让信息在未来请求中可被找回和使用。

## Lab 10：最小自我进化实验

目标：把“自我进化”拆成受控更新循环，而不是让模型随便改自己。实验只做候选更新和人工确认，不自动写长期规则。

建议实验：

1. 选一个固定任务和一个初版 `SKILL.md`。
2. 故意让 agent 在任务里失败一次，记录失败事件、工具结果或用户纠正。
3. 让模型根据失败信息生成一条候选 skill 修改建议。
4. 人工确认后，把建议写入一个新的 skill 版本或 patch 文件。
5. replay 同一任务，比较修改前后的行为。

重点验证：

- 自我进化的核心是 feedback -> candidate update -> validation -> commit/rollback。
- 更新对象可以是 memory、skill、prompt、tool guideline，但不能无边界写入。
- 人工确认、版本化、回滚是自我进化系统的必要护栏。

## Lab 11：UI bot / CLI 归一化入口

目标：把前面学到的 runtime 能力落成一个应用入口。可以先做 CLI，再考虑 UI bot。

建议实验：

1. CLI 版本：写一个 `my-agent <task>`，内部创建 `AgentSession`，订阅事件并归一化输出。
2. 给 CLI 加一个自定义 tool 或 skill，让它支持一个固定工作流。
3. 加一个 memory 读取步骤：启动时读取外部 memory 文件，通过 context hook 注入。
4. UI bot 版本：用 Web UI 或自建简单界面渲染消息、工具状态、memory 命中、确认按钮。

重点验证：

- 应用入口负责归一化输入、输出、工具状态、session、memory 和用户确认。
- SDK/RPC/extension/Web UI 是不同接入方式，不是新的 agent core。
- 最终应用应该证明你能用 pi 组装一个可复用 agent，而不只是读懂源码。

## 最小 harness 建议

建议后续单独写一个临时调试脚本，放在 `/tmp` 或 `learning/scratch/`，核心结构如下：

```ts
import { Agent } from "@earendil-works/pi-agent-core";
import { AssistantMessageEventStream, type Model } from "@earendil-works/pi-ai";

const model = {
  id: "debug-model",
  name: "debug-model",
  api: "debug-api",
  provider: "debug-provider",
  baseUrl: "",
  reasoning: false,
  input: ["text"],
  cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
  contextWindow: 128000,
  maxTokens: 4096,
} satisfies Model<any>;

const streamFn = () => {
  const stream = new AssistantMessageEventStream();
  queueMicrotask(() => {
    const partial = {
      role: "assistant" as const,
      content: [{ type: "text" as const, text: "" }],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: {
        input: 1,
        output: 1,
        cacheRead: 0,
        cacheWrite: 0,
        totalTokens: 2,
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
      },
      stopReason: "stop" as const,
      timestamp: Date.now(),
    };
    stream.push({ type: "start", partial });
    partial.content[0].text = "debug response";
    stream.push({ type: "text_delta", contentIndex: 0, delta: "debug response", partial });
    stream.push({ type: "done", reason: "stop", message: partial });
    stream.end(partial);
  });
  return stream;
};

const agent = new Agent({
  initialState: { model },
  streamFn,
});

agent.subscribe((event) => {
  console.log(event.type);
});

await agent.prompt("hello");
```

后续可以把这个 harness 改成三种版本：

- text-only：只验证普通 streaming。
- tool-call：验证 tool call 和 tool result。
- error：验证 retry/compaction 前置条件。

## 调试记录模板

每次调试用固定模板记录，避免只看断点不沉淀：

```text
实验:
入口动作:
断点:
关键变量:
事件顺序:
状态变化:
session entry:
结论:
疑问:
```

示例：

```text
实验: 普通 prompt
入口动作: AgentSession.prompt("hello")
断点: AgentSession.prompt -> Agent.prompt -> runLoop -> streamAssistantResponse
关键变量: messages, llmMessages, partialMessage
事件顺序: agent_start -> turn_start -> message_start(user) -> message_end(user) -> message_start(assistant) -> message_update -> message_end -> turn_end -> agent_end
状态变化: state.messages 增加 user 和 assistant；streamingMessage 在 message_end 清空
session entry: message(user), message(assistant)
结论: AgentSession 负责应用层前处理和持久化，Agent core 负责 turn loop
疑问: transformContext 在 extension context hook 中会不会影响 session 原始消息
```
