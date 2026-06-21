# Pi 学习观察笔记

> 一个观察一节，按时间倒序往上加（最新的在最前面）。每节按 `00-how-to-learn.md` 的输出物模板：一张图 / 关键类型 / 一条调用链 / 反向验证题 / 可验证结论 / 疑问。
>
> 命名规则：`## NN — 主题（YYYY-MM-DD）`，NN 递增。

> 校正说明：这份文件保留原始观察和修正过程。写正式文章前，以 `articles/` 和源码复核后的结论为准；早期关于 `agent_end.messages`、`willRetry`、`stopReason` 的判断已经被后续源码追踪校正。

---

## 02 — agent runtime 源码追踪：从 SDK 入口到 agent loop（2026-05-30）

### 总图（认路用）

```
hello.mjs
  │
  │ createAgentSession(options)             ← sdk.ts:193
  │   ├─ 拼装 services (auth/model/settings/session/loader)
  │   ├─ new Agent({ streamFn, ... })       ← sdk.ts:320  注入"怎么调模型"
  │   └─ new AgentSession({ agent, ... })   ← sdk.ts:392  SDK 外壳
  │
  │ session.prompt("...")                   ← agent-session.ts:939   外壳层
  │ agent.prompt("...")                     ← agent.ts:324           分发
  │ runAgentLoop([userMsg], ctx, cfg)       ← agent-loop.ts:95       第一段 emit
  │ runLoop(...)                            ← agent-loop.ts:155 ★核心★
  │
  ↓ 核心循环（伪代码，对应 agent-loop.ts:170-266）
┌──────────────────────────────────────────────────────────────┐
│ while (true) {                            // 外层：follow-up │
│   while (hasMoreToolCalls || pending) {   // 内层：turn      │
│     if (!firstTurn) emit turn_start                          │
│     msg = await streamAssistantResponse(...)  ← 调模型       │
│     toolResults = await executeToolCalls(...) ← 跑工具       │
│     emit turn_end { message, toolResults }                   │
│     hasMoreToolCalls = (toolCalls.length > 0)                │
│   }                                                          │
│   followUp = ... ; if (followUp) continue;                   │
│   break;                                                     │
│ }                                                            │
│ emit agent_end                                               │
└──────────────────────────────────────────────────────────────┘
```

### 关键节点 5 处

#### 1. streamFn 注入 —— 三层架构的胶水（`sdk.ts:328`）

```js
agent = new Agent({
  streamFn: async (model, context, options) => {
    const auth = await modelRegistry.getApiKeyAndHeaders(model);
    return streamSimple(model, context, { ...options, apiKey: auth.apiKey, ... });
  },
  ...
});
```

**意义**：`Agent` 自己不知道怎么调 LLM，全靠这个回调。`packages/agent` 因此 provider-无关，`packages/ai` 才是 provider 实现。三层分离的体现。

#### 2. runAgentLoop 头几行 emit —— 对应 out.log 开头（`agent-loop.ts:109-114`）

```js
await emit({ type: "agent_start" });
await emit({ type: "turn_start" });
for (const prompt of prompts) {
  await emit({ type: "message_start", message: prompt });
  await emit({ type: "message_end", message: prompt });
}
```

**意义**：你 out.log 前 4 个事件就是这 4 行 emit 的。

#### 3. runLoop 内循环 —— pi 的 agent loop 本体（`agent-loop.ts:174-218`）

```js
while (hasMoreToolCalls || pendingMessages.length > 0) {
  if (!firstTurn) await emit({ type: "turn_start" });
  firstTurn = false;
  ...
  const message = await streamAssistantResponse(...);   // 第 193 行
  ...
  const toolCalls = message.content.filter(c => c.type === "toolCall");
  let toolResults = [];
  hasMoreToolCalls = false;
  if (toolCalls.length > 0) {
    const batch = await executeToolCalls(...);          // 第 208 行
    toolResults = batch.messages;
    hasMoreToolCalls = !batch.terminate;
  }
  await emit({ type: "turn_end", message, toolResults });
}
```

**意义**：昨天笔记里那句 `while (stopReason === "toolUse")` 的源码原型。判据是 `toolCalls.length > 0`，不是直接看 stopReason 字段。

#### 4. streamAssistantResponse 转事件 —— provider 事件 → pi 事件（`agent-loop.ts:313-340`）

```js
for await (const event of response) {
  switch (event.type) {
    case "start":         emit message_start
    case "text_delta":
    case "toolcall_delta":
    case "thinking_delta":
    case ...:             emit message_update { assistantMessageEvent: event }
    case "done":
    case "error":         emit message_end + return
  }
}
```

**意义**：你 out.log 里 `message_update.assistantMessageEvent.type` 的来历 —— packages/ai 的细粒度事件被原样包进 `assistantMessageEvent` 字段，外面套上 message 完整快照。

#### 5. executeToolCallsSequential —— 工具执行链（`agent-loop.ts:406-438`）

```js
for (const toolCall of toolCalls) {
  await emit({ type: "tool_execution_start", ... });   // 第 407 行
  const prep = await prepareToolCall(...);              // 权限/参数验证
  const executed = await executePreparedToolCall(...);  // 真正跑 bash/read
  const finalized = await finalizeExecutedToolCall(...);
  await emitToolExecutionEnd(finalized, emit);
  const toolResultMessage = createToolResultMessage(finalized);
  await emitToolResultMessage(toolResultMessage, emit);
}
```

**意义**：每个工具一个 `tool_execution_start` → ... → `tool_execution_end` → `message_start/end (toolResult)`。与 out.log 完全对应。还有一个 `executeToolCallsParallel` 在 `:447`，本次没触发。

### 三层架构（确认）

| 层 | 包 | 关键文件 | 职责 |
|---|---|---|---|
| SDK 外壳 | `coding-agent` | `core/sdk.ts`, `core/agent-session.ts` | 装配 services、UI/权限/状态包装 |
| Agent runtime | `agent` | `agent.ts`, `agent-loop.ts` | turn loop、工具调度、事件 emit |
| Provider | `ai` | `providers/*.ts`, `stream.ts` | 把厂商 SSE 转成统一事件 |

### 验证昨天的猜想

- ✅ 更精确地说：从 provider 事件看 `stopReason=toolUse` 通常对应工具调用；源码里的 loop 判据是 assistant message 中的 tool calls 加上工具批次的 `terminate` 信号。
- ✅ `tool_execution_update` 由工具自己 emit（在 `executePreparedToolCall` 内部，工具通过 `onUpdate` 回调控制频率）
- ✅ cwd 通过 `currentContext` 传给工具的 `execute(args, ctx)`

### 新疑问

- `batch.terminate` 什么时候是 true？看 `shouldTerminateToolBatch`
- `transformContext` / `convertToLlm` 分别做什么转换？
- `prepareNextTurn`（`agent-loop.ts:226`）能改 model/thinking —— 是 compaction / 模型切换的钩子？
- 外层 `while(true)` 的 follow-up vs steering 区别？两个 hook 都存在

---

## 01 — Step 0：第一次跑通 SDK，完整事件流（2026-05-30）

### 环境

- `~/pi-hello/hello.mjs`：通过 `registerProvider("anthropic", { baseUrl, apiKey: "dummy" })` 把内置 anthropic provider 的端点改到本地 `betahi-copilot-bridge` (127.0.0.1:4142)
- 模型：`anthropic/claude-opus-4-7`（pi registry 自带，bridge 转发到上游 `claude-opus-4.7-1m-internal`）
- prompt：`"List files in the current directory, then tell me what this project is."`
- 输出导到 `out.log`，**3039 行，完整读完**

### 一张图：整次会话的事件流骨架

```
agent_start
  ├─ turn_start ─────────────────────────────────── Turn 1（user → bash）
  │   ├─ message_start (user)
  │   ├─ message_end   (user)
  │   ├─ message_start (assistant)
  │   │   ├─ message_update { toolcall_start }       模型决定调 bash
  │   │   ├─ message_update { toolcall_delta }×N     参数 JSON 一段段流出
  │   │   │   "{\"com" → ... → "{\"command\":\"ls -la\"}"
  │   │   └─ message_update { toolcall_end }
  │   ├─ message_end   (assistant, stopReason=toolUse)
  │   ├─ tool_execution_start                        本地真正执行 bash
  │   ├─ tool_execution_update ×N                    增量 stdout（bash 才有）
  │   ├─ tool_execution_end
  │   ├─ message_start (toolResult)
  │   ├─ message_end   (toolResult)
  │   └─ turn_end { toolResults: [bash 输出] }
  │
  ├─ turn_start ─────────────────────────────────── Turn 2（read package.json）
  │   ├─ (同样的 assistant→toolcall→execute→toolResult 链路)
  │   └─ turn_end
  │
  ├─ turn_start ─────────────────────────────────── Turn 3（read hello.mjs）
  │   ├─ (同样链路)
  │   └─ turn_end
  │
  ├─ turn_start ─────────────────────────────────── Turn 4（最终回答，纯文本）
  │   ├─ message_start (assistant)
  │   │   ├─ message_update { text_start }
  │   │   ├─ (text_delta 直接打到 stdout，未被 [EVENT] 包装)
  │   │   └─ message_update { text_end, content: "完整 markdown" }
  │   ├─ message_end   (assistant, stopReason=stop)
  │   └─ turn_end { toolResults: [] }                ← 注意空数组
  │
  └─ agent_end { messages: [本次 run 新增的 7 条] }
```

### 关键类型 / 字段

- `event.type`：`agent_start` / `turn_start` / `turn_end` / `message_start` / `message_end` / `message_update` / `tool_execution_start` / `tool_execution_update` / `tool_execution_end` / `agent_end`
- `event.assistantMessageEvent.type`（`message_update` 子事件）：`toolcall_start|delta|end` / `text_start|delta|end` / `thinking_*`
- `message.role`：`user` / `assistant` / `toolResult`
- `message.stopReason`：`toolUse`（要继续转）/ `stop`（终止）
- `turn_end.toolResults`：`[]` 表示这一 turn 没产生工具调用 → 是最后一个 turn
- `agent_end.messages`：本次 agent run 产生的新消息；空会话里看起来像完整历史
- `message.usage` + `message.usage.cost`：每条消息独立带 token 数和美元成本，SDK 层已算好

### 一条调用链

```
session.prompt()
  → agent.run()
    → loop:
        turn_start
        model.stream()                          ← 发请求，流式解析
          → 拼出 toolCall (或 text)
          → message_end(stopReason)
        if stopReason === "toolUse":
            tool.execute()                      ← 本地跑工具
            push toolResult message
            turn_end({toolResults: [...]})
            continue                            ← 进下一 turn
        else:
            turn_end({toolResults: []})
            break
      agent_end({messages})
```

### 7 个观察

1. **流式不只在文本，工具参数 JSON 也是流式的**。`toolcall_delta` 一段段拼出 `{"command":"ls -la"}`。pi 原样暴露 anthropic-messages 的 input streaming，不缓冲到完整 JSON 才发。
2. **`message_update` 是包裹器**，真正的语义在嵌套的 `assistantMessageEvent.type`。pi 事件流的核心一层。
3. **turn = 模型一次响应 + 它触发的工具结果**。模型继续调工具 → 新 turn；模型 `stop` → 当前 turn 是最后一个，`toolResults: []`。
4. **`stopReason` 是理解循环的好入口，但不是源码里的直接判据**。源码实际看 assistant message 里有没有 tool calls，再看工具批次是否要求 terminate。
5. **`tool_execution_update` 不是每个工具都有**。bash 在跑 `ls -la` 时 emit 了增量 stdout，read 工具只在 `tool_execution_end` 一次性给出完整内容 —— 决定权在工具自己。
6. **`agent_end` 是本次 run 的终点**，携带本次 run 产生的新消息。整段会话归档还要结合已有历史或 SessionManager。
7. **`text_delta` 出现在最终回答 turn**。前 3 个 turn 模型只产 `toolcall`，没产 `text`；最后一个 turn 才进入 `text_start` → `text_delta`×N → `text_end`，markdown 一段段流出来。

### 反向验证题

1. **没有 turn 边界会怎样？** —— 没法做 per-turn 的 hook（自动 compaction、tool budget、abort、UI 进度分组），也没法把"一次工具往返"作为最小持久化单元。
2. **如果工具参数不流式（缓冲完整 JSON）会怎样？** —— UI 看不到"模型正在敲哪条命令"的实时反馈，但实现简单。pi 选流式是 UX 决策。
3. **如果 `agent_end` 不带 `messages` 快照会怎样？** —— 上层要自己从 `turn_end` 拼本次 run 的增量；现在 `agent_end` 直接给本次 run 的完整增量。
4. **跟 Claude Agent SDK / LangGraph 比？** —— 待补：读完 02 后回填。

### 可验证结论

**pi 的 agent loop 是 4 个 turn 完成这个 prompt：bash(ls) → read(package.json) → read(hello.mjs) → text 回答。每个工具往返一个 turn，最后一个 turn 不带工具（`toolResults: []`），然后 `agent_end`。**

证据（来自 out.log 3039 行）：
- 4 个 `turn_start` / `turn_end`
- 前 3 个 turn 的 assistant message `stopReason: "toolUse"`，turn_end 都带 1 个 toolResult
- 第 4 个 turn 的 assistant message `stopReason: "stop"`，`turn_end.toolResults: []`
- 紧接着 `agent_end`，messages 数组长度 7（user + 3 × {assistant+toolResult} + 1 × assistant text）

### 我的疑问

1. `responseId` 每个 turn 都不一样（`msg_vrtx_01P5...` / `msg_vrtx_01DM...` / ...）—— 这是 anthropic 上游每次请求新生成的，pi 内部没用它做任何串联。但既然带出来了，是不是为 session resume / 调试用？
2. `tool_execution_update` 的频率和分块策略：bash 这次只 emit 了 1 次 update（一次性把 `ls -la` 全部输出给出），不是真正的 streaming。是因为命令太快没时间分块，还是 bash 工具本来就只 emit 一次？要在长跑命令上再验证。
3. `cacheRead: 2109` 在第一次请求就有值 —— 是 anthropic provider 默认在 system prompt / tool definitions 上加了 `cache_control`，bridge 转发后上游命中之前别的 session 的缓存？
4. `cwd` 没出现在事件里，pi 怎么知道在 `/Users/z/pi-hello` 跑 `ls`？答案大概率：`createAgentSession({ resourceLoader: new DefaultResourceLoader({ cwd: process.cwd() }) })` 时绑定，bash 工具从 ResourceLoader / services 取。要在 02 / 06 文档里核实。
5. 早期日志里看到的 `willRetry` 需要重新核对事件来源；源码里它属于 compaction 相关事件，不是 core `agent_end` 字段。
