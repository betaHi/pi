# 源码证据地图

这个文件把前面文档里的关键判断落到具体源码。学习时不要只读结论，要顺着这里的文件和函数去验证。

## 1. CLI 只是装配入口

结论：`pi` 命令本身不直接实现 agent loop，它负责解析参数、确定模式、创建 session/runtime，然后交给具体模式运行。

源码证据：

- `packages/coding-agent/src/main.ts`
  - `main()`：总入口，串起参数解析、session manager、runtime、mode。
  - `resolveAppMode()`：决定 `interactive`、`print`、`json`、`rpc`。
  - `createSessionManager()`：处理 `--session`、`--resume`、`--continue`、`--fork`、`--no-session`。
  - `buildSessionOptions()`：把 CLI model/thinking/tools 参数转成 `CreateAgentSessionOptions`。
  - `createAgentSessionRuntime()` 调用点：创建最终 runtime。
  - `runPrintMode()`、`runRpcMode()`、`InteractiveMode.run()` 调用点：不同运行模式共用同一个 session/runtime。

要验证的问题：

- 为什么 `main.ts` 要先确定 session cwd，再创建 cwd-bound services？
- 为什么 print/json/rpc/interactive 可以共用 `AgentSession`？

## 2. Coding Agent 的核心装配点是 `createAgentSession()`

结论：`packages/coding-agent` 把通用 `Agent` 装配成 coding agent，核心注入点在 SDK 层。

源码证据：

- `packages/coding-agent/src/core/sdk.ts`
  - `createAgentSession()`：创建 `AuthStorage`、`ModelRegistry`、`SettingsManager`、`SessionManager`、`ResourceLoader`。
  - `convertToLlmWithBlockImages()`：把 coding-agent 消息转成 LLM 消息，并按设置过滤图片。
  - `new Agent({ ... })`：注入 `convertToLlm`、`streamFn`、`onPayload`、`onResponse`、`transformContext`、queue mode、transport、thinking budgets。
  - `streamFn`：读取 model auth/header/retry 设置后调用 `streamSimple()`。
  - `transformContext`：把 extension `context` hook 插入每次 LLM 请求前。

要验证的问题：

- coding-agent 为什么不直接调用 provider？
- API key、headers、retry 设置在哪里进入 provider 请求？
- extension 的 `before_provider_request` 和 `after_provider_response` 为什么在 SDK 层处理？

## 3. 用户输入先进入 `AgentSession.prompt()`

结论：`AgentSession` 是应用层运行时，负责命令、模板、skill、队列、鉴权、压缩、扩展 hook，再把消息交给通用 `Agent`。

源码证据：

- `packages/coding-agent/src/core/agent-session.ts`
  - `prompt()`：处理 extension command、input hook、skill/template expansion、streaming queue、model/auth 校验、compaction 前置检查、user message 构造、`before_agent_start` hook。
  - `_runAgentPrompt()`：调用 `this.agent.prompt()`，并在需要 retry/compaction continuation 时调用 `this.agent.continue()`。
  - `_handlePostAgentRun()`：串起 retry 和 compaction 检查。
  - `_handleAgentEvent()`：订阅 core agent event，转给 extensions/UI，并在 `message_end` 时写入 session。
  - `_installAgentToolHooks()`：把 extension `tool_call` 和 `tool_result` hook 接入 agent core。

要验证的问题：

- `AgentSession.prompt()` 和 `Agent.prompt()` 的职责边界是什么？
- streaming 时为什么要用 `steer` 或 `followUp` queue？
- 为什么 session 持久化放在 `message_end` 事件处理里？

## 4. 通用 Agent 只关心 transcript、事件和队列

结论：`packages/agent` 是通用 agent runtime，不知道 CLI、TUI、文件系统，也不知道具体 provider。

源码证据：

- `packages/agent/src/agent.ts`
  - `Agent`：有状态封装，持有 `state.messages`、`state.tools`、`model`、`thinkingLevel`。
  - `prompt()`：拒绝并发 prompt，标准化输入后进入 `runPromptMessages()`。
  - `continue()`：从已有 transcript 继续，用于 retry/compaction 后续执行。
  - `steer()`、`followUp()`：两类队列入口。
  - `createContextSnapshot()`：把当前 state 拷贝成 loop 输入。
  - `createLoopConfig()`：把 hooks、queue drains、convert/transform、tool execution 配置传给低层 loop。
  - `processEvents()`：把 `AgentEvent` 归约进 `Agent.state`，再通知 listeners。

要验证的问题：

- 为什么 `Agent.prompt()` 不能并发？
- `state.streamingMessage`、`pendingToolCalls`、`errorMessage` 分别在哪些事件里变化？
- `agent_end` 为什么不等于外部所有 listener 已经完成之前的“空闲”状态？

## 5. Agent Loop 的核心是 turn + tool loop

结论：一次 prompt 可能包含多轮 LLM call，因为 assistant 可能连续请求工具；steering 和 follow-up 是两套不同插入时机的队列。

源码证据：

- `packages/agent/src/agent-loop.ts`
  - `runAgentLoop()`：追加新 prompt，发出 `agent_start`、`turn_start`、user `message_start/message_end`。
  - `runAgentLoopContinue()`：不追加新 prompt，直接从当前上下文继续。
  - `runLoop()`：外层处理 follow-up，内层处理 tool calls 和 steering。
  - `streamAssistantResponse()`：在 LLM 边界调用 `transformContext()`、`convertToLlm()`、`streamFn || streamSimple()`，并把 `AssistantMessageEvent` 转成 `AgentEvent`。
  - `executeToolCalls()`：根据配置和 tool `executionMode` 选择顺序或并行。
  - `prepareToolCall()`：查找 tool、`prepareArguments()`、`validateToolArguments()`、`beforeToolCall`。
  - `executePreparedToolCall()`：执行 tool，并把 partial result 变成 `tool_execution_update`。
  - `finalizeExecutedToolCall()`：执行 `afterToolCall`，允许改写结果。
  - `createToolResultMessage()`：把执行结果变成下一轮 LLM 上下文里的 `toolResult` message。

要验证的问题：

- `transformContext` 和 `convertToLlm` 的边界在哪里？
- parallel tool execution 下，为什么完成事件和最终 `ToolResultMessage` 顺序可能不同？
- `shouldStopAfterTurn()` 和 `prepareNextTurn()` 分别适合做什么？

## 6. Context 是运行时投影，不是原始 session

结论：LLM 收到的 context 是 system prompt、session branch、skills、tools、extension context、compaction summary 等多路信息合成后的运行时视图，不等于 JSONL session 原文。

源码证据：

- `packages/coding-agent/src/core/system-prompt.ts`
  - `buildSystemPrompt()`：把基础提示词、工具说明、skills、project context、date、cwd 合成 system prompt。
- `packages/coding-agent/src/core/sdk.ts`
  - `transformContext`：把 extension `context` hook 插入每次 LLM 请求前。
  - `convertToLlmWithBlockImages()`：把 coding-agent 消息转成 provider 可接受的 LLM messages。
- `packages/agent/src/agent-loop.ts`
  - `streamAssistantResponse()`：在 LLM 边界调用 `transformContext()`、`convertToLlm()`，再构造 `pi-ai Context`。
- `packages/coding-agent/src/core/session-manager.ts`
  - `buildSessionContext()`：把 append-only session tree 投影成当前 branch 的 messages。

要验证的问题：

- context 中哪些内容来自 session，哪些来自 resource loader，哪些来自 extension？
- `before_agent_start` 和 `context` hook 的时机差异是什么？
- compaction summary 如何改变后续 context，而不是删除历史？

## 7. LLM 协议统一在 `packages/ai`

结论：provider 差异被压到 `packages/ai`，上层只消费统一的 `AssistantMessageEventStream`。

源码证据：

- `packages/ai/src/types.ts`
  - `Model<TApi>`：模型身份、provider、api、能力、成本、上下文窗口。
  - `Context`：`systemPrompt`、`messages`、`tools`。
  - `AssistantMessageEvent`：start/delta/end/done/error 等流式事件。
- `packages/ai/src/utils/event-stream.ts`
  - `AssistantMessageEventStream`：异步事件流，最终返回完整 `AssistantMessage`。
- `packages/ai/src/stream.ts`
  - `stream()`、`complete()`：provider 原始 options 入口。
  - `streamSimple()`、`completeSimple()`：统一简单 options 入口。
  - `resolveApiProvider()`：按 `model.api` 找 provider。
- `packages/ai/src/api-registry.ts`
  - `registerApiProvider()`：注册 provider。
  - `getApiProvider()`：按 api 获取 provider。
  - `wrapStream()`、`wrapStreamSimple()`：运行时校验 `model.api` 和 provider api 匹配。
- `packages/ai/src/providers/register-builtins.ts`
  - 内置 provider 的注册入口。

要验证的问题：

- `Model.provider` 和 `Model.api` 为什么不是同一个字段？
- `streamSimple()` 为什么只需要 `model + context + options`？
- provider lazy registration 后，上层为什么仍然只看到统一事件？

## 8. 工具系统分两层

结论：coding-agent 用 `ToolDefinition` 描述工具及渲染/扩展上下文，agent core 只需要可执行的 `AgentTool`。

源码证据：

- `packages/coding-agent/src/core/extensions/types.ts`
  - `ToolDefinition`：extension/custom tool 面向 coding-agent 的定义。
  - `registerTool()`：extension 注册工具的 API。
- `packages/coding-agent/src/core/tools/index.ts`
  - `createAllToolDefinitions()`：创建内置工具集合。
  - `withFileMutationQueue()`：保护文件写入类工具的顺序。
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
  - `wrapToolDefinition()`：把 `ToolDefinition` 转成 agent core 的 `AgentTool`。
  - `createToolDefinitionFromAgentTool()`：把 plain `AgentTool` 反向合成最小 `ToolDefinition`。
- `packages/coding-agent/src/core/tools/read.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/edit.ts`
- `packages/coding-agent/src/core/tools/write.ts`

要验证的问题：

- schema validation 为什么在 agent loop 中做？
- tool execute 和 tool render 为什么分层？
- `edit`、`write` 为什么需要 mutation queue？

## 9. Session 是 append-only tree，不是普通数组

结论：session 文件是 JSONL append-only tree。`leafId` 表示当前分支，`buildSessionContext()` 才负责把树解析成 LLM 上下文。

源码证据：

- `packages/coding-agent/src/core/session-manager.ts`
  - `SessionEntryBase`：所有 entry 都有 `id`、`parentId`、`timestamp`。
  - `SessionEntry`：message、model change、thinking change、compaction、branch summary、custom、label、session info。
  - `buildSessionContext()`：从 leaf 走到 root，处理 compaction/custom message/branch summary，生成 LLM messages。
  - `SessionManager._appendEntry()`：追加 entry 并更新 `leafId`。
  - `appendMessage()`、`appendCompaction()`、`appendCustomMessageEntry()`：不同 entry 的追加入口。
  - `getBranch()`：从指定 entry 回溯到 root。
  - `branch()`：移动 leaf，不删除历史。
  - `createBranchedSession()`：抽取一条 branch path 生成新 session。

要验证的问题：

- `/tree` 为什么能回到历史节点后继续产生新分支？
- compaction 为什么是 entry，而不是直接删除旧 message？
- custom entry 和 custom message entry 为什么一个不进 LLM，一个进 LLM？

## 10. Compaction 和 retry 都在 AgentSession 层

结论：长上下文恢复、overflow recovery、临时 provider retry 都属于 coding-agent 应用层策略，不属于通用 agent loop。

源码证据：

- `packages/coding-agent/src/core/agent-session.ts`
  - `_handlePostAgentRun()`：每次 agent run 后判断 retry 或 compaction。
  - `_checkCompaction()`：根据 assistant stop/error 和上下文使用情况决定是否 compact。
  - `_runAutoCompaction()`：执行 overflow/threshold 自动压缩。
  - `compact()`：手动压缩入口。
  - `_isRetryableError()`、`_prepareRetry()`：自动重试判断和状态准备。
- `packages/coding-agent/src/core/compaction/compaction.ts`
  - `prepareCompaction()`、`compact()`、`shouldCompact()`。
- `packages/ai/src/utils/overflow.ts`
  - `isContextOverflow()`。

要验证的问题：

- overflow recovery 和 threshold auto-compaction 的触发条件有什么不同？
- retry 为什么要调整 agent state，但不删除 session 历史？
- compaction 后 `buildSessionContext()` 如何改变后续 LLM 输入？

## 11. 扩展系统覆盖 agent 生命周期

结论：extension 不是简单插件菜单，而是可以参与输入、上下文、provider 请求、agent 生命周期、tool hook、session 操作和 UI。

源码证据：

- `packages/coding-agent/src/core/resource-loader.ts`
  - 负责加载 extensions、skills、prompt templates、themes、context files、system prompt。
- `packages/coding-agent/src/core/extensions/loader.ts`
  - `registerTool()`、`registerCommand()` 等 extension API 的加载期实现。
- `packages/coding-agent/src/core/extensions/runner.ts`
  - `bindCore()`：把 session/runtime 能力绑定给 extension runtime。
  - `bindCommandContext()`：绑定命令上下文。
  - `createContext()`、`createCommandContext()`：生成 extension handler 上下文。
  - `emit()`：通用事件分发。
  - `emitInput()`、`emitContext()`、`emitBeforeAgentStart()`、`emitToolCall()`、`emitToolResult()`、`emitBeforeProviderRequest()`：关键专用 hook。
  - `invalidate()`：session replacement 或 reload 后让旧 context 失效。
- `packages/coding-agent/src/core/agent-session.ts`
  - `_bindExtensionCore()`、`_refreshToolRegistry()`、`_buildRuntime()`、`bindExtensions()`。

要验证的问题：

- `input`、`before_agent_start`、`context` 三个 hook 的时机区别是什么？
- extension tool 如何进入 `agent.state.tools`？
- 为什么 reload/session replacement 后旧 extension context 必须失效？

## 12. Skills 是可复用指令，不是自动记忆

结论：skill 是一种可发现、可显式调用的 instruction 资源。pi 会加载、列出、展开 skill，但不会自动从 session 中学习并改写 skill。

源码证据：

- `packages/coding-agent/src/core/skills.ts`
  - `loadSkills()`：从 user/project/显式路径加载 skills。
  - `loadSkillFromFile()`：解析 frontmatter，校验 name/description，生成 `Skill`。
  - `formatSkillsForPrompt()`：把可模型调用的 skills 格式化成 `<available_skills>`，只包含 name/description/location。
- `packages/coding-agent/src/core/system-prompt.ts`
  - `buildSystemPrompt()`：在 read tool 可用时，把 skills section 放入 system prompt。
- `packages/coding-agent/src/core/agent-session.ts`
  - `_expandSkillCommand()`：把 `/skill:name args` 展开成完整 `<skill>` block。
  - `parseSkillBlock()`：让 UI/export 能识别 skill invocation。
- `packages/coding-agent/src/core/resource-loader.ts`
  - `updateSkillsFromPaths()`：把资源路径解析成当前 session 可见 skills。

要验证的问题：

- skill 为什么默认只暴露描述和位置，而不是直接塞全文？
- `disable-model-invocation` 如何让 skill 只能被 `/skill:name` 显式调用？
- skill、prompt template、extension command 分别解决什么问题？

## 13. Memory 是应用层系统

结论：pi 的 session、skills、context hook、extension 能支撑 memory 系统，但 pi core 不负责自动学习用户偏好、长期检索和记忆更新策略。Memory 需要应用层自己设计提取、存储、检索、注入和确认。

源码证据：

- `packages/coding-agent/src/core/session-manager.ts`
  - session 保存的是 append-only tree 和 context rebuild，不是语义记忆库。
- `packages/agent/src/harness/session/memory-repo.ts`
  - `InMemorySessionRepo` 是内存 session storage，不是长期 memory 系统。
- `packages/coding-agent/src/core/extensions/runner.ts`
  - `emitContext()` 可以在每次 LLM 请求前注入外部检索到的上下文。
- `packages/coding-agent/src/core/skills.ts`
  - skills 可以承载人工整理后的可复用知识或流程。

要验证的问题：

- 如果要做长期 memory，应该从哪些事件抽取信息？
- 记忆应该写入 session、skill、外部 store，还是 extension 自己维护的资源？
- 记忆如何去重、过期、分作用域、排序和注入？

## 14. 自我进化是受控更新循环

结论：自我进化不是 memory 的同义词。它关注从反馈中生成候选变更，并经过验证、确认、版本化和回滚后，更新 memory、skills、prompt 或工具策略。

源码证据：

- `packages/coding-agent/src/core/extensions/runner.ts`
  - extension hooks 可以观察输入、上下文、工具结果和 agent 生命周期，适合采集反馈信号。
- `packages/coding-agent/src/core/agent-session.ts`
  - retry、compaction、event stream 提供可观察的失败和恢复信号。
- `packages/coding-agent/src/core/skills.ts`
  - skills 可以承载人工确认后的新流程或新规则。
- `packages/coding-agent/src/core/session-manager.ts`
  - session 历史可作为 replay 和对比的输入，但不负责自动更新策略。

要验证的问题：

- 哪些反馈可以触发候选更新：用户纠正、失败工具调用、重复任务、测试结果、人工评分？
- 候选更新如何验证：replay、测试、对比输出、人工确认？
- 更新对象如何版本化和回滚？
- 哪些内容绝不允许自动写入长期上下文？

## 15. TUI/Web UI 都消费事件，不直接跑模型

结论：UI 层不应该直接理解 provider。TUI 消费 `AgentSessionEvent`，Web UI 消费 core `AgentEvent` 或 session 事件包装。

源码证据：

- `packages/tui/src/tui.ts`
  - `Component.render(width)`：组件渲染模型。
  - `Container.addChild()`、`Container.render()`：组件树。
  - `TUI.start()`、`TUI.requestRender()`、`TUI.render()`、`TUI.setFocus()`：输入、焦点、差量渲染。
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
  - `InteractiveMode`：绑定 session event、editor、slash command、key handlers、footer/status、extension UI context。
- `packages/coding-agent/src/modes/interactive/components/assistant-message.ts`
- `packages/coding-agent/src/modes/interactive/components/tool-execution.ts`
- `packages/web-ui/src/components/AgentInterface.ts`
  - `setupSessionSubscription()`：订阅 agent/session。
  - `sendMessage()`：把用户输入发给 agent。
  - `renderMessages()`：渲染稳定消息列表。
- `packages/web-ui/src/components/StreamingMessageContainer.ts`
- `packages/web-ui/src/ChatPanel.ts`
  - `setAgent()`：把 agent 接入聊天面板。
- `packages/web-ui/src/utils/proxy-utils.ts`
  - `createStreamFn()`、`applyProxyIfNeeded()`：浏览器侧 provider/proxy 处理。

要验证的问题：

- TUI 为什么只关心事件和组件状态，不关心 provider？
- streaming message 为什么适合单独渲染？
- Web UI 和 TUI 复用的是哪些 core 能力？

## 第一轮学习建议

先用这条最短路径读代码：

```text
packages/coding-agent/src/main.ts
  -> packages/coding-agent/src/core/sdk.ts
  -> packages/coding-agent/src/core/agent-session.ts
  -> packages/agent/src/agent.ts
  -> packages/agent/src/agent-loop.ts
  -> packages/ai/src/stream.ts
  -> packages/ai/src/api-registry.ts
```

读完只要求能讲清一件事：

```text
用户输入
  -> AgentSession.prompt()
  -> Agent.prompt()
  -> runAgentLoop()
  -> streamAssistantResponse()
  -> streamSimple()
  -> provider
  -> AssistantMessageEvent
  -> AgentEvent
  -> AgentSessionEvent/UI/session JSONL
```

第二轮再补 context 构建、context window/compaction、工具、session tree、skills/extensions、memory、自我进化和 UI。
