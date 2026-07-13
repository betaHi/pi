# Pi 系列 14｜应用入口：从 Agent 到运行时

本系列其他文章：

placeholder

> 本文主要参考 `packages/agent/src/agent.ts`、`packages/coding-agent/src/core/sdk.ts`、`agent-session.ts`、`agent-session-runtime.ts`、`main.ts`、`modes/*`、`packages/web-ui` 和 `examples/sdk/*`。

前面十二篇讲了 agent 的内部机制：turn loop、provider、tool、session、context、skill、extension、memory 和自我进化。最后一篇看应用入口。一个应用要怎样使用这些机制，CLI、TUI、RPC、浏览器这些形态又接到哪一层。

pi 的入口可以分成三层：`Agent`、`AgentSession`、`AgentSessionRuntime`。层级越往上，默认设施越多，运行时状态管理也越完整。

## 一、三层入口

| 层级 | 作用 | 适合场景 |
|---|---|---|
| `new Agent` | turn 引擎，负责消息、工具调用、provider stream、队列 | 已有自己的存储、鉴权、UI，只需要 agent loop |
| `createAgentSession` | 装配 Agent、SessionManager、ModelRegistry、ResourceLoader、工具、extension | Node/CLI/服务端应用，希望复用 pi 的完整设施 |
| `createAgentSessionRuntime` | 管理当前 active session，支持 new/resume/fork/import/reload 后重建运行时 | 需要多会话、切换 cwd、内置模式或复杂 UI |

选择入口时，可以先问两个问题：是否需要 pi 的资源、工具、session 和 extension；是否需要替换 active session。只需要对话内核时用 `Agent`。需要 coding-agent 的设施时用 `AgentSession`。需要 new session、resume、fork、reload 后继续保持 UI 和 extension 可用时，用 runtime。

<!-- 图1：三层入口
生图 prompt：
一张横版三层分层图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
从下到上三层：底层「Agent」标「turn loop / messages / tools / model / queues」；中层「AgentSession」标「SessionManager / ModelRegistry / ResourceLoader / tools / extensions」；上层「AgentSessionRuntime」标「active session replacement / new / resume / fork / import / reload」。
每层右侧放适用场景：Agent「已有自己的存储和鉴权」；AgentSession「Node/CLI/服务端复用 pi 设施」；Runtime「多会话和重建运行时」。
底部小字：「层级越往上，默认设施越多，运行时重建能力越完整。」
建议文件名：./pi_14_1.png
-->

## 二、`new Agent`：最小内核

`packages/agent/src/agent.ts` 的 `Agent` 是最底层对话引擎。

它关心这些内容：

- `state.messages`、`systemPrompt`、`model`、`thinkingLevel`、`tools`。
- `prompt()`、`continue()`、`steer()`、`followUp()`、`abort()`、`waitForIdle()`。
- `convertToLlm`、`transformContext`、`streamFn`、`onPayload`、`onResponse`。
- agent event 订阅。

它不带 coding-agent 的默认设施，例如 AuthStorage、ModelRegistry、ResourceLoader、SessionManager、内置工具和 extension runtime。这些要由使用者自己接入。

web-ui 示例就走这层：浏览器端直接 new `Agent`，再把它交给 `pi-chat-panel`。浏览器自己处理 IndexedDB session、API key prompt、artifact tools、JavaScript REPL 和自定义 `convertToLlm`。这说明 core Agent 可以脱离 Node 端设施使用。

## 三、`createAgentSession`：SDK 的常用入口

`createAgentSession(options)` 返回：

```ts
{
  session: AgentSession,
  extensionsResult: LoadExtensionsResult,
  modelFallbackMessage?: string
}
```

默认装配过程大致是：

```text
AuthStorage.create
  -> ModelRegistry.create
  -> SettingsManager.create
  -> SessionManager.create
  -> DefaultResourceLoader.reload()
  -> findInitialModel + clampThinkingLevel
  -> new Agent({ convertToLlm, streamFn, transformContext, hooks })
  -> new AgentSession({ agent, resourceLoader, tools, extensionRunnerRef, ... })
```

这里有一个关键分层：传给 core `Agent` 的 `streamFn` 是闭包。每次请求前，它通过 `modelRegistry.getApiKeyAndHeaders(model)` 取 key 和 header，再调用 `streamSimple`。core Agent 不需要知道凭据存在哪里，鉴权和 provider retry headers 由 SDK 层注入。

`AgentSession` 在 `Agent` 外面补上这些能力：

- session 持久化和恢复。
- 内置工具、custom tools、extension tools。
- `_rebuildSystemPrompt`：把 tools、skills、project context、append prompt 合到 system prompt。
- extension events、commands、custom messages。
- compaction、model/thinking 切换、active tools。

最小 SDK 用法很短：

```ts
const { session } = await createAgentSession();
session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});
await session.prompt("What files are in the current directory?");
session.dispose();
```

对于自定义 CLI、服务端 bot 或本地工具，通常可以从 `createAgentSession` 开始。

<!-- 图2：createAgentSession 装配过程
生图 prompt：
一张横版装配流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
主线节点：AuthStorage → ModelRegistry → SettingsManager → SessionManager → DefaultResourceLoader.reload → findInitialModel/clampThinkingLevel → new Agent → new AgentSession。
在 new Agent 下面标注注入项：「convertToLlm / streamFn / transformContext / hooks」。在 streamFn 旁画一个小锁图标，标「从 ModelRegistry 取 key/header」。
右侧输出「session + extensionsResult + modelFallbackMessage」。
底部小字：「Agent 是内核；createAgentSession 把 Node 端设施接上。」
建议文件名：./pi_14_2.png
-->

## 四、`AgentSessionRuntime`：active session 的运行时

现在的 `main.ts` 会先创建 `AgentSessionRuntime`，再把 runtime 交给 print、interactive 或 rpc 模式。

runtime 负责这些事：

- 持有当前 `runtime.session`。
- 提供 `newSession()`、`switchSession()`、`fork()`、`importFromJsonl()`。
- session replacement 后重新创建 cwd-bound services。
- 让模式层重新 bind extension UI、命令上下文和事件订阅。

这一层解决的是“当前 session 会变化”的问题。TUI 里新建会话、RPC 里切换会话、print 模式里执行 reload，都可能让旧 session 和旧 extension ctx 失效。runtime 给内置模式提供了统一的重建流程。

SDK 文档里的建议也很明确：单会话应用可以直接用 `AgentSession`；需要替换 active session 时，用 `AgentSessionRuntime`。

## 五、几种运行形态

### print mode

`runPrintMode(runtime, options)` 用于 CLI 单发或少量消息：

- text mode 输出最终文本。
- json mode 输出 session events。
- 会 bind extension command context，让 extension command、reload、fork 等在 print 环境里有基本行为。

### interactive mode

`InteractiveMode` 是 TUI 外壳：

- 负责输入框、补全、消息渲染、主题、快捷键。
- skill、prompt template、extension command 都会合进补全。
- extension UI 的 `ctx.ui` 在这里有最完整实现。

### rpc mode

`runRpcMode(runtime)` 把 agent 暴露成 JSONL over stdio：

- stdin 每行一个 command。
- stdout 输出 response 和 session events。
- `prompt` 的 response 表示请求已接受、已排队或已被命令处理；后续结果看事件流。
- RPC 支持一部分 extension UI request/response，例如 confirm、select、input、notify、status。

RPC 适合让 IDE、桌面应用、脚本或其他进程通过子进程驱动 pi。

<!-- 图3：运行形态
生图 prompt：
一张横版中心辐射图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中心方块「AgentSessionRuntime」。向外四个方向连接：「print mode：CLI 单发/JSON events」「interactive mode：TUI 输入、补全、渲染」「rpc mode：stdin 命令 / stdout events」「web-ui：core Agent + 浏览器存储/工具」。
rpc 方向画双向箭头标「JSONL over stdio」。web-ui 方向用虚线标「不走 coding-agent AgentSession」。
底部小字：「同一套 agent 能力，可以接不同外壳；web-ui 直接复用 core Agent。」
建议文件名：./pi_14_3.png
-->

### web-ui

`packages/web-ui` 走的是另一条路线。它直接使用 core `Agent`，再通过 lit web component `pi-chat-panel` 展示。

浏览器端会替代 Node 端设施：IndexedDB 存 session，浏览器 UI 处理 API key，工具由组件和 sandbox 提供。这个例子说明底层 `Agent` 可以跨运行环境复用，但每个环境要自己补存储、鉴权和传输。

## 六、选择入口

可以按需求做一个简单判断：

| 需求 | 入口 |
|---|---|
| 只要 agent loop，其他设施已有 | `new Agent` |
| 要 pi 的 tools、skills、extensions、sessions | `createAgentSession` |
| 要多会话、fork、resume、reload 后重绑 | `createAgentSessionRuntime` |
| 要外部进程控制 | RPC mode 或直接用 SDK |
| 要浏览器组件 | `packages/web-ui` 的 `ChatPanel` + core `Agent` |

如果目标是做一个 UI bot 或 CLI，较稳的起点通常是 `createAgentSession`。后续出现多会话、切 cwd、fork、reload 后重绑这些需求，再上 runtime。

<!-- 图4：入口选择决策图
生图 prompt：
一张竖向决策流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
顶部问题「你只需要 agent loop 吗？」是则到「new Agent」。否则到第二个问题「需要 pi 的 tools/skills/extensions/sessions 吗？」是则到「createAgentSession」。再到第三个问题「需要 new/resume/fork/reload 后重绑 active session 吗？」是则到「createAgentSessionRuntime」。旁边另有分支「浏览器组件」到「web-ui ChatPanel + core Agent」，「外部进程控制」到「RPC mode」。
底部小字：「先选最小够用层；需要更多设施再往上走。」
建议文件名：./pi_14_4.png
-->

## 七、入口归一化的含义

入口归一化的目的，是让不同前端尽量复用同一套 agent 能力：

- 同一套 turn loop 和工具调用语义。
- 同一套 session/history/context 投影。
- 同一套 skill、extension、resource 机制。
- 多个外壳负责输入输出和 UI 差异。

这样，CLI、TUI、RPC、bot、web UI 可以共享行为。前端更换时，主要改 transport 和交互层；agent 逻辑尽量保持一份。

## 系列收尾

回看整组文章，pi 的结构可以按一条线理解：

1. agent loop 处理 turn。
2. provider 抽象负责模型流式输出。
3. tool 系统把模型动作接到本地能力。
4. session 保存历史并支持分支、压缩和恢复。
5. context 构造把 session、工具、项目规则转成模型输入。
6. skill 把可复用指令做成按需资源。
7. extension 把应用策略接入生命周期。
8. memory 和自我进化复用这些接口，形成应用层系统。
9. 应用入口把这些机制包装成 CLI、TUI、RPC、web 或自定义 bot。

理解 agent harness，关键在于看清这些层之间的数据流：消息如何进入 loop，工具结果如何回到 session，session 如何投影成 context，extension 在哪些点介入，最后哪一层把它暴露给用户。

<!-- 图5：系列机制地图
生图 prompt：
一张横版机制地图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
从左到右画九个节点：agent loop → provider → tool → session → context → skill → extension → memory/self-evolution → application entry。节点之间用箭头连接。
在 session 到 context 之间标「投影」，tool 到 session 标「toolResult 回填」，extension 向 context/tool/session 三处画细线标「hook」。application entry 向 print/TUI/RPC/web 四个外壳发散。
底部小字：「理解 harness，就是理解这些机制怎样把一次对话变成可用的应用入口。」
建议文件名：./pi_14_5.png
-->

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*