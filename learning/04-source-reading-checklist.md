# 源码阅读清单

这份清单用于内部验证，不是文章大纲。读源码时按这里补证据；写文章时只抽取能支撑核心机制的少量文件和函数。

## 第一轮：只建立主链路

按顺序读：

1. `packages/coding-agent/src/cli.ts`
2. `packages/coding-agent/src/main.ts`
3. `packages/coding-agent/src/core/sdk.ts`
4. `packages/coding-agent/src/core/agent-session.ts`
5. `packages/agent/src/agent.ts`
6. `packages/agent/src/agent-loop.ts`
7. `packages/ai/src/stream.ts`
8. `packages/ai/src/providers/register-builtins.ts`

读完要能回答：

- `pi` 命令如何变成一个 `AgentSession`？
- `AgentSession.prompt()` 和 `Agent.prompt()` 分别负责什么？
- 一次 LLM response 的 streaming event 如何一路传到 UI？
- tool call 的结果如何回到下一轮 LLM 上下文？

## 第二轮：补齐数据结构

按顺序读：

1. `packages/ai/src/types.ts`
2. `packages/agent/src/types.ts`
3. `packages/coding-agent/src/core/messages.ts`
4. `packages/coding-agent/src/core/session-manager.ts`
5. `packages/coding-agent/src/core/extensions/types.ts`

读完要能回答：

- `Message`、`AgentMessage`、coding-agent custom messages 的关系是什么？
- `AssistantMessageEvent` 和 `AgentEvent` 的关系是什么？
- session entry 和 LLM message 的关系是什么？
- extension event 覆盖了哪些生命周期节点？

## 第三轮：工具执行

按顺序读：

1. `packages/coding-agent/src/core/tools/index.ts`
2. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
3. `packages/coding-agent/src/core/tools/read.ts`
4. `packages/coding-agent/src/core/tools/bash.ts`
5. `packages/coding-agent/src/core/tools/edit.ts`
6. `packages/coding-agent/src/core/tools/write.ts`
7. `packages/agent/src/agent-loop.ts` 的 `executeToolCalls*` 部分

读完要能回答：

- 工具 schema 验证在哪里做？
- tool render 为什么放在 coding-agent，而不是 agent core？
- partial tool update 如何发给 UI？
- parallel tool execution 下，事件顺序和消息顺序哪里不同？

## 第四轮：context 构建、持久化、分支、压缩

按顺序读：

1. `packages/coding-agent/src/core/system-prompt.ts`
2. `packages/coding-agent/src/core/messages.ts`
3. `packages/coding-agent/src/core/session-manager.ts`
4. `packages/coding-agent/src/core/agent-session-runtime.ts`
5. `packages/coding-agent/src/core/agent-session.ts` 的 `compact()`、`navigateTree()`、`getSessionStats()`、`_checkCompaction()`、`_handlePostAgentRun()`。
6. `packages/coding-agent/src/core/compaction/*`
7. `packages/ai/src/utils/overflow.ts`

读完要能回答：

- system prompt、skills、tools、cwd、date 如何进入 LLM context？
- `transformContext`、`convertToLlm`、provider `Context` 的边界是什么？
- session 为什么 append-only？
- `/tree` 为什么可以切 branch 但不删除历史？
- compaction entry 如何影响 `buildSessionContext()`？
- retry/overflow recovery 为什么要调整 agent state？
- context window 不够时，哪些逻辑在 harness，哪些应该留给应用层？

## 第五轮：skills、扩展和资源

按顺序读：

1. `packages/coding-agent/src/core/resource-loader.ts`
2. `packages/coding-agent/src/core/skills.ts`
3. `packages/coding-agent/src/core/system-prompt.ts`
4. `packages/coding-agent/src/core/package-manager.ts`
5. `packages/coding-agent/src/core/extensions/loader.ts`
6. `packages/coding-agent/src/core/extensions/runner.ts`
7. `packages/coding-agent/src/core/agent-session.ts` 的 `_expandSkillCommand()` 和 extension 相关方法。

读完要能回答：

- project/user/CLI/package 资源如何合并？
- skill 如何被发现、校验、放进 system prompt 或通过 `/skill:name` 展开？
- skill、prompt template、extension command 的边界是什么？
- extension command/tool/provider/shortcut 如何注册？
- extension handler 出错如何被隔离？
- reload 后为什么旧 extension context 会失效？

## 第六轮：应用层 memory

按顺序读：

1. `packages/coding-agent/docs/sdk.md`
2. `packages/coding-agent/examples/sdk/*`
3. `packages/coding-agent/src/core/agent-session.ts` 的 `subscribe()`、`prompt()`、`followUp()`、`steer()`。
4. `packages/coding-agent/src/core/extensions/runner.ts` 的 `emitContext()`。
5. `packages/agent/src/harness/session/memory-repo.ts`

读完要能回答：

- pi 的 session storage 和长期 memory 有什么区别？
- 应用层如何从事件中抽取记忆，再通过 context hook 注入？
- 记忆的存储、检索、去重、过期、优先级分别在哪里处理？

## 第七轮：自我进化机制

按顺序读：

1. `packages/coding-agent/docs/extensions.md`
2. `packages/coding-agent/src/core/extensions/runner.ts`
3. `packages/coding-agent/src/core/agent-session.ts` 的 event、retry、compaction 路径。
4. `packages/coding-agent/src/core/skills.ts`
5. `packages/coding-agent/src/core/session-manager.ts`

读完要能回答：

- 自我进化和 memory 的边界是什么？
- 哪些反馈信号可以生成候选更新？
- 候选更新如何验证、确认、落盘和回滚？
- 哪些内容绝不应该自动写入长期上下文？

## 第八轮：UI 和 pi 应用入口

TUI：

1. `packages/tui/src/tui.ts`
2. `packages/tui/src/components/editor.ts`
3. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`
4. `packages/coding-agent/src/modes/interactive/components/*`

Web UI：

1. `packages/web-ui/src/components/AgentInterface.ts`
2. `packages/web-ui/src/ChatPanel.ts`
3. `packages/web-ui/src/components/Messages.ts`
4. `packages/web-ui/src/tools/*`

读完要能回答：

- terminal UI 的组件树如何渲染？
- interactive mode 如何处理输入、slash command、bash command、queue？
- Web UI 复用了哪些 core 能力，哪些逻辑是浏览器特有的？
- 如果 build 一个 UI bot 或 CLI，应该复用 SDK、RPC、Web UI 还是 extension？
- 如何把输入、工具、memory 命中、确认动作和输出格式归一化到一个入口？

## 推荐源码笔记格式

每读一个文件，记录四项：

```text
文件:
职责:
核心类型/函数:
上游调用者:
下游依赖:
```

示例：

```text
文件: packages/agent/src/agent-loop.ts
职责: 执行 agent turn、LLM streaming、tool call、steering/follow-up queue。
核心类型/函数: runAgentLoop, runLoop, streamAssistantResponse, executeToolCalls。
上游调用者: Agent.runPromptMessages, Agent.runContinuation。
下游依赖: streamSimple/streamFn, validateToolArguments, AgentTool.execute。
```

## 当前已重点阅读过的文件

- `packages/ai/src/types.ts`
- `packages/ai/src/stream.ts`
- `packages/ai/src/api-registry.ts`
- `packages/ai/src/models.ts`
- `packages/ai/src/utils/event-stream.ts`
- `packages/ai/src/providers/register-builtins.ts`
- `packages/ai/src/providers/simple-options.ts`
- `packages/ai/src/providers/transform-messages.ts`
- `packages/agent/src/types.ts`
- `packages/agent/src/agent.ts`
- `packages/agent/src/agent-loop.ts`
- `packages/agent/src/proxy.ts`
- `packages/coding-agent/src/cli.ts`
- `packages/coding-agent/src/main.ts`
- `packages/coding-agent/src/core/sdk.ts`
- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/core/agent-session-runtime.ts`
- `packages/coding-agent/src/core/agent-session-services.ts`
- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/messages.ts`
- `packages/coding-agent/src/core/system-prompt.ts`
- `packages/coding-agent/src/core/model-registry.ts`
- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/extensions/types.ts`
- `packages/coding-agent/src/core/tools/index.ts`
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
- `packages/coding-agent/src/core/tools/read.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/edit.ts`
- `packages/coding-agent/src/core/tools/write.ts`
- `packages/tui/src/index.ts`
- `packages/web-ui/src/index.ts`
- `packages/web-ui/src/components/AgentInterface.ts`
- `packages/web-ui/src/ChatPanel.ts`
- `packages/web-ui/src/utils/proxy-utils.ts`
