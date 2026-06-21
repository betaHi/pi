# 架构地图

## 包依赖

核心依赖方向是单向的。下面箭头表示“上层依赖下层”：

```text
packages/coding-agent
  -> packages/agent
      -> packages/ai
  -> packages/tui

packages/web-ui
  -> packages/agent
  -> packages/ai
```

更准确地说：

- `packages/agent` 依赖 `packages/ai`，但不依赖 coding-agent。
- `packages/coding-agent` 依赖 `packages/agent`、`packages/ai`、`packages/tui`。
- `packages/tui` 是 UI 基础层，不知道 agent 业务。
- `packages/web-ui` 是另一套 UI 表达，直接复用 agent core 和 ai 层。

## packages/ai

职责：统一不同 LLM provider 的模型、认证、流式事件、工具调用和 token/cost 统计。

关键文件：

- `packages/ai/src/types.ts`：核心类型，包括 `Model`、`Message`、`AssistantMessage`、`ToolCall`、`AssistantMessageEvent`。
- `packages/ai/src/stream.ts`：`stream`、`complete`、`streamSimple`、`completeSimple` 的统一入口。
- `packages/ai/src/api-registry.ts`：provider registry，按 `model.api` 找到对应 stream 函数。
- `packages/ai/src/providers/register-builtins.ts`：内置 provider 的 lazy registration。
- `packages/ai/src/providers/transform-messages.ts`：跨 provider 回放时处理图片、thinking、tool call id、孤立 tool result。
- `packages/ai/src/models.ts`：模型注册表、thinking level、cost 计算。

核心抽象：

- `Model<TApi>`：模型身份、provider、api、baseUrl、能力、价格、上下文窗口。
- `Context`：`systemPrompt`、`messages`、`tools`。
- `AssistantMessageEventStream`：异步事件流，最后产出一个完整 `AssistantMessage`。
- `AssistantMessageEvent`：`start`、`text_delta`、`thinking_delta`、`toolcall_delta`、`done/error`。

## packages/agent

职责：通用 agent runtime，不关心 coding CLI，也不关心具体 provider 细节。

关键文件：

- `packages/agent/src/types.ts`：`AgentMessage`、`AgentTool`、`AgentEvent`、hook、queue mode、tool execution mode。
- `packages/agent/src/agent-loop.ts`：低层 agent loop，负责 LLM turn、tool call 执行、steering/follow-up queue。
- `packages/agent/src/agent.ts`：有状态封装，管理 transcript、queue、abort、listener、runtime state。
- `packages/agent/src/proxy.ts`：代理 streamFn，把远端 proxy 事件重建为本地 `AssistantMessageEventStream`。

核心设计：

- `AgentMessage` 是可扩展消息类型，LLM 边界前通过 `convertToLlm` 过滤或转换。
- `AgentEvent` 是 UI 和上层应用消费的事件协议。
- `AgentTool` 扩展自 `pi-ai` 的 `Tool`，增加 `execute`、`prepareArguments`、`executionMode`。
- 工具执行分 `parallel` 和 `sequential`。如果某个 tool 声明 `executionMode: "sequential"`，整批 tool call 会顺序执行。

## packages/coding-agent

职责：把通用 agent core 变成可用的 coding agent 应用。

关键文件：

- `packages/coding-agent/src/cli.ts`：bin 入口，设置 undici/fetch，再进入 `main()`。
- `packages/coding-agent/src/main.ts`：解析 CLI、创建 session manager、services、runtime，选择 interactive/print/json/rpc 模式。
- `packages/coding-agent/src/core/sdk.ts`：SDK 入口 `createAgentSession()`，装配 `Agent`、model registry、resource loader、tools。
- `packages/coding-agent/src/core/agent-session.ts`：上层 session runtime，处理 prompt、工具、扩展、压缩、重试、bash、tree/fork。
- `packages/coding-agent/src/core/session-manager.ts`：JSONL session tree，负责 append-only persistence、branch、fork、context rebuild。
- `packages/coding-agent/src/core/model-registry.ts`：内置模型、自定义模型、provider override、auth/header 解析。
- `packages/coding-agent/src/core/resource-loader.ts`：扩展、skills、prompts、themes、AGENTS.md/SYSTEM.md 加载。
- `packages/coding-agent/src/core/extensions/runner.ts`：扩展事件分发、命令、工具、provider registration。
- `packages/coding-agent/src/core/tools/*.ts`：内置工具定义与渲染。

## packages/tui

职责：终端 UI 框架。

关键文件：

- `packages/tui/src/tui.ts`：`Component`、`Container`、`TUI`、overlay、focus、render loop。
- `packages/tui/src/components/*`：Text、Box、Markdown、Editor、SelectList 等组件。
- `packages/tui/src/keybindings.ts` 和 `packages/tui/src/keys.ts`：键盘输入和 keybinding。

coding-agent 的 interactive mode 是在这层之上构建的业务 UI。

## packages/web-ui

职责：浏览器 UI 组件。

关键文件：

- `packages/web-ui/src/components/AgentInterface.ts`：Web 版 agent chat，订阅 `AgentEvent` 并更新 message list/streaming container。
- `packages/web-ui/src/ChatPanel.ts`：组合 `agent-interface` 和 artifacts panel。
- `packages/web-ui/src/utils/proxy-utils.ts`：浏览器 CORS proxy 决策。

## 最重要的边界

```text
LLM provider event
  -> pi-ai AssistantMessageEvent
  -> pi-agent-core AgentEvent
  -> coding-agent AgentSessionEvent
  -> TUI/Web UI render
```

```text
model tool call
  -> AgentTool.execute()
  -> ToolResultMessage
  -> next LLM turn
```

```text
session JSONL entries
  -> SessionManager.buildSessionContext()
  -> Agent.state.messages
  -> convertToLlm()
  -> pi-ai Context.messages
```
