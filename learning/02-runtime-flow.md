# 运行链路

## 1. CLI 启动

入口：

- `packages/coding-agent/src/cli.ts`
- `packages/coding-agent/src/main.ts`

`cli.ts` 做三件事：

1. 设置 `process.title` 和 `PI_CODING_AGENT`。
2. 配置 undici dispatcher，避免长流式请求被默认 timeout 中断。
3. 调用 `main(process.argv.slice(2))`。

`main.ts` 负责把命令行参数转换成 runtime：

```text
parseArgs()
  -> createSessionManager()
  -> createAgentSessionServices()
  -> createAgentSessionFromServices()
  -> createAgentSessionRuntime()
  -> run mode: interactive | print | json | rpc
```

## 2. 创建 AgentSession

入口：

- `packages/coding-agent/src/core/sdk.ts`
- `packages/coding-agent/src/core/agent-session-services.ts`
- `packages/coding-agent/src/core/agent-session-runtime.ts`

`createAgentSession()` 是 SDK 级别装配点：

1. 创建或接收 `AuthStorage`、`ModelRegistry`、`SettingsManager`、`SessionManager`、`ResourceLoader`。
2. 从 session 或 settings 恢复 model/thinking level。
3. 创建 `Agent`。
4. 给 `Agent` 注入：
   - `convertToLlm`
   - `streamFn`
   - `onPayload`
   - `onResponse`
   - `transformContext`
   - steering/follow-up mode
5. 创建 `AgentSession`，注册内置工具和扩展工具。

关键点：coding-agent 不直接调用 provider。它把认证、headers、扩展 hook 封到 `Agent.streamFn` 里，最终还是走 `pi-ai` 的 `streamSimple()`。

## 3. 用户 prompt 进入系统

入口：

- `packages/coding-agent/src/core/agent-session.ts`

`AgentSession.prompt(text, options)` 的主要步骤：

1. 如果是 `/` 开头，先尝试扩展命令。
2. 触发扩展 `input` hook，允许 handled 或 transform。
3. 展开 `/skill:name` 和 prompt template。
4. 如果 agent 正在 streaming，根据 `streamingBehavior` 进入 steering 或 follow-up queue。
5. 非 streaming 时验证 model 和 auth。
6. 必要时先做 compaction。
7. 构造 `user` message。
8. 触发 `before_agent_start` hook，允许扩展追加 custom message 或修改 system prompt。
9. 调用 `_runAgentPrompt()`，进入 core agent。

Skill 在这里有两条路径：

- 自动暴露：`ResourceLoader` 加载 skills 后，`buildSystemPrompt()` 只把 skill 的 name、description、location 放进 `<available_skills>`，让模型知道“需要时可以用 read 工具读取 skill 文件”。
- 显式调用：用户输入 `/skill:name args` 时，`AgentSession._expandSkillCommand()` 读取 skill 文件，去掉 frontmatter，包成 `<skill name="..." location="...">...</skill>`，再把用户参数接在后面进入 prompt。

这说明 skill 更像“可发现、可显式展开的 reusable instruction”，不是会自己学习和改写的 memory。

## 4. Agent core loop

入口：

- `packages/agent/src/agent.ts`
- `packages/agent/src/agent-loop.ts`

`Agent.prompt()` 会：

```text
Agent.prompt()
  -> runPromptMessages()
  -> runAgentLoop()
  -> runLoop()
```

`runLoop()` 是关键：

```text
outer loop: follow-up queue
  inner loop: tool calls + steering queue
    turn_start
    inject pending steering messages
    streamAssistantResponse()
    executeToolCalls()
    turn_end
    prepareNextTurn()
    shouldStopAfterTurn()
  drain follow-up messages
agent_end
```

这解释了 pi 的两个队列：

- steering：当前 assistant turn 和 tool calls 完成后，下一次 LLM call 前插入。
- follow-up：agent 没有更多工具和 steering 后再插入。

## 5. LLM 流式响应

入口：

- `packages/agent/src/agent-loop.ts`
- `packages/ai/src/stream.ts`
- `packages/ai/src/api-registry.ts`
- `packages/ai/src/providers/register-builtins.ts`

`streamAssistantResponse()` 做边界转换：

```text
AgentMessage[]
  -> transformContext()
  -> convertToLlm()
  -> pi-ai Context
  -> streamSimple(model, context, options)
  -> AssistantMessageEventStream
```

provider 事件会变成 agent 事件：

```text
pi-ai start
  -> AgentEvent message_start
pi-ai text_delta / thinking_delta / toolcall_delta
  -> AgentEvent message_update
pi-ai done/error
  -> AgentEvent message_end
```

`Agent.processEvents()` 同时更新运行态：

- `isStreaming`
- `streamingMessage`
- `pendingToolCalls`
- `errorMessage`
- `state.messages`

## 6. 工具调用

入口：

- `packages/agent/src/agent-loop.ts`
- `packages/coding-agent/src/core/tools/*.ts`
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`

工具调用链：

```text
assistant message contains ToolCall[]
  -> executeToolCalls()
  -> prepareToolCall()
      find tool
      prepareArguments()
      validateToolArguments()
      beforeToolCall hook
  -> executePreparedToolCall()
      tool.execute()
      tool_execution_update events
  -> finalizeExecutedToolCall()
      afterToolCall hook
  -> ToolResultMessage
  -> next LLM turn
```

内置工具来自 `ToolDefinition`，再通过 `wrapToolDefinition()` 变成 agent core 需要的 `AgentTool`。

默认 coding tools：

- `read`：读文本和图片，带截断、图片 resize、渲染逻辑。
- `bash`：执行 shell，流式输出，截断后写临时完整输出。
- `edit`：精确文本替换，支持一批非重叠 edits，生成 diff。
- `write`：创建或覆盖文件，自动创建目录。

另外还有只读辅助工具：

- `grep`
- `find`
- `ls`

## 7. 会话持久化

入口：

- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/session-manager.ts`

`AgentSession` 订阅 `AgentEvent`。每次 `message_end`：

```text
AgentEvent message_end
  -> AgentSession._handleAgentEvent()
  -> SessionManager.appendMessage() or appendCustomMessageEntry()
  -> JSONL append
```

session 文件不是纯线性数组，而是 append-only tree：

- 每个 entry 有 `id` 和 `parentId`。
- 当前分支由 `leafId` 决定。
- `/tree` 是移动 leaf，不删除旧历史。
- `/fork` 是抽取某条 branch path 到新 session。
- compaction 是单独 entry，`buildSessionContext()` 会把 summary 和保留消息重建成 LLM 上下文。

## 8. UI 更新

TUI：

- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
- `packages/tui/src/tui.ts`

Web UI：

- `packages/web-ui/src/components/AgentInterface.ts`

两者都不是直接轮询模型，而是订阅事件：

```text
AgentEvent / AgentSessionEvent
  -> message component
  -> tool execution component
  -> queue/status/footer
  -> render
```

TUI 还额外管理：

- editor
- slash commands
- autocomplete
- keybindings
- overlay/selectors
- extension UI context
- terminal title/footer/status

## 9. 扩展系统

入口：

- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/extensions/loader.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/extensions/types.ts`

扩展可以参与多个阶段：

- 资源发现：`resources_discover`
- 输入处理：`input`
- context 变换：`context`
- provider 请求前后：`before_provider_request`、`after_provider_response`
- agent 生命周期：`agent_start`、`turn_start`、`message_end` 等
- tool hook：`tool_call`、`tool_result`
- session hook：`session_before_switch`、`session_before_compact`、`session_before_tree`
- UI、commands、shortcuts、tools、provider registration

这也是 pi 的核心设计：默认功能保持小，但扩展点覆盖 agent 生命周期。
