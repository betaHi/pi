# 工程问题学习路线

内部学习也按工程问题组织，不按包名或文件名组织。每块先回答“这个问题为什么存在”，再追源码和实验。

优先级说明：

- P0：必须吃透，会直接进入文章主线，也是 agent harness 工程能力的核心。
- P1：需要理解，作为文章证据或扩展学习，不必展开成独立文章。
- P2：知道边界和查找位置即可，除非后续真的要改对应模块。

## Block 1：不同 provider 的流式事件怎么统一？（P0）

目标：理解 pi 如何把 Anthropic / OpenAI / Gemini 等不同响应格式统一成一种 `AssistantMessageEvent` 协议。

先读：

- `packages/ai/src/types.ts`
- `packages/ai/src/utils/event-stream.ts`
- `packages/ai/src/stream.ts`
- `packages/ai/src/api-registry.ts`
- `packages/ai/src/providers/register-builtins.ts`
- `packages/ai/src/providers/*`

重点问题：

- `AssistantMessageEvent` 为什么同时有 partial update 和 final message？
- `Model.api` 和 `Model.provider` 分别解决什么问题？
- provider lazy load 出错时为什么也要通过 stream 协议返回 error message？
- `stream()` 和 `streamSimple()` 的边界是什么？

练习：

- 用 `providers/faux.ts` 或最小自定义 stream function，模拟 text/tool call/error 三种事件。
- 画出一个 `AssistantMessage` 从空 content 到 final content 的变化过程。

## Block 2：模型和工具的主循环怎么跑？（P0）

目标：掌握 agent loop、turn、steering/follow-up queue，以及模型和工具如何往返。

先读：

- `packages/agent/src/types.ts`
- `packages/agent/src/agent-loop.ts`
- `packages/agent/src/agent.ts`

重点问题：

- 一次 prompt 为什么可能产生多个 turn？
- `stopReason` 和源码里的 tool calls 判据是什么关系？
- steering queue 和 follow-up queue 的插入点有什么不同？
- `prepareNextTurn()` 和 `shouldStopAfterTurn()` 分别适合做什么？
- parallel tool execution 下，事件顺序和消息顺序为什么可能不同？

练习：

- 写一个固定文本 `streamFn`，用 `Agent.prompt()` 跑通。
- 写一个会触发 tool call 的 fake `streamFn`，观察 `turn_start` / `turn_end`。
- 改 queue mode，比较 `all` 和 `one-at-a-time` 的上下文变化。

## Block 3：Context 如何构建、转换和注入？（P0）

目标：理解 LLM 看到的 context 不是 session 文件本身，而是运行时构造出来的输入视图。

先读：

- `packages/coding-agent/src/core/system-prompt.ts`
- `packages/coding-agent/src/core/sdk.ts`
- `packages/coding-agent/src/core/messages.ts`
- `packages/coding-agent/src/core/agent-session.ts`
- `packages/agent/src/agent-loop.ts` 的 `streamAssistantResponse()`
- `packages/coding-agent/src/core/extensions/runner.ts` 的 `emitContext()`

重点问题：

- system prompt、AGENTS.md、skills、tools、cwd、date 分别什么时候进入 context？
- `transformContext` 和 `convertToLlm` 的边界在哪里？
- extension `context` hook 和 `before_agent_start` hook 的时机差异是什么？
- 图片、custom message、compaction summary、tool result 如何变成 provider 可接受的消息？
- context 是 session 的投影，还是 session 本身？

练习：

- 打断点看一次 `AgentMessage[] -> llmMessages -> pi-ai Context`。
- 写一个 `context` hook，给每次 LLM 请求追加一条 custom message。
- 对比有无 skill / AGENTS.md / compaction summary 时的 context 差异。

## Block 4：上下文窗口不够时怎么办？（P0）

目标：理解 compaction、overflow recovery、retry 和 context budget。这里是长程 agent 工程的核心问题之一。

先读：

- `packages/coding-agent/src/core/agent-session.ts` 的 `_checkCompaction()`、`_runAutoCompaction()`、`compact()`、`_handlePostAgentRun()`、`_prepareRetry()`。
- `packages/coding-agent/src/core/compaction/*`
- `packages/ai/src/utils/overflow.ts`
- `packages/coding-agent/src/core/session-manager.ts` 的 compaction entry / `buildSessionContext()`。

重点问题：

- threshold compaction 和 overflow recovery 有什么不同？
- compaction 为什么写成 session entry，而不是直接删除旧消息？
- retry 为什么要调整 agent state，但不删除 session 历史？
- compaction 后下一次 LLM 请求的 context 如何变化？
- context budget 应该在 harness 层解决，还是交给应用层？

练习：

- 构造一个 assistant overflow error，推演 `_checkCompaction()` 分支。
- 比较 manual compaction、threshold auto-compaction、overflow recovery 的事件差异。
- 手写一个带 compaction entry 的 session，推演 `buildSessionContext()` 输出。

## Block 5：工具调用怎么安全执行和回传？（P0）

目标：掌握 tool definition、schema、执行、partial update、结果消息、渲染和文件 mutation queue。

先读：

- `packages/agent/src/agent-loop.ts` 的 `executeToolCalls*` 部分
- `packages/coding-agent/src/core/tools/index.ts`
- `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
- `packages/coding-agent/src/core/tools/read.ts`
- `packages/coding-agent/src/core/tools/bash.ts`
- `packages/coding-agent/src/core/tools/edit.ts`
- `packages/coding-agent/src/core/tools/write.ts`

重点问题：

- `ToolDefinition` 和 `AgentTool` 的边界是什么？
- schema validation 在 agent loop 哪一步发生？
- `beforeToolCall` / `afterToolCall` 分别适合做什么？
- `read`、`bash`、`edit`、`write` 各自解决什么工程问题？
- `write/edit` 为什么要走 `withFileMutationQueue()`？

练习：

- 给一个自定义 `ReadOperations`，模拟远端文件读取。
- 构造一个多 edit patch，看 diff 如何生成。
- 写一个自定义 tool，观察 tool execution events 和 tool result message。

## Block 6：长程状态为什么不能只是 messages 数组？（P0）

目标：理解 session 是 append-only tree，负责历史、分支、恢复和 context rebuild 的基础输入。

先读：

- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/messages.ts`
- `packages/coding-agent/src/core/agent-session.ts` 中 tree/fork/session stats 相关方法

重点问题：

- entry 的 `id`、`parentId`、`leafId` 如何表达分支？
- `buildSessionContext()` 如何从 tree 生成 LLM messages？
- custom entry 和 custom message entry 有什么区别？
- label/session_info 为什么也是 append-only entry？
- `/tree` 和 `/fork` 的差异是什么？

练习：

- 手写一个很小的 JSONL session，推演 `buildSessionContext()` 输出。
- 找一个真实 session 文件，画出它的 branch path。
- 跟踪 `/tree` 选择历史 user message 后，editor text 如何产生。

## Block 7：Skills 如何承载可复用工作流？（P0）

目标：掌握 reusable instruction 如何被发现、暴露给模型、显式调用，并理解它和 prompt template / extension command 的边界。

先读：

- `packages/coding-agent/src/core/skills.ts`
- `packages/coding-agent/src/core/system-prompt.ts`
- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/agent-session.ts` 的 `_expandSkillCommand()`、`parseSkillBlock()`

重点问题：

- skill frontmatter 如何变成 `Skill`？
- `disable-model-invocation` 改变什么？
- system prompt 里为什么只列 skill name / description / location，而不是直接塞完整 skill 内容？
- `/skill:name args` 如何展开成 `<skill ...>` block，再作为用户消息进入 prompt？
- skill、prompt template、extension command 的边界是什么？

练习：

- 写一个最小 `SKILL.md`，验证它如何出现在 system prompt 的 `<available_skills>` 里。
- 调用 `/skill:name 参数`，观察展开后的 user message 和 UI/export 的 skill block 渲染。

## Block 8：Extension 如何开放生命周期？（P0）

目标：理解 pi 如何让外部代码接管输入、context、provider 请求、工具、session 和 UI，而不是只能注册一个菜单项。

先读：

- `packages/coding-agent/src/core/extensions/types.ts`
- `packages/coding-agent/src/core/extensions/loader.ts`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/resource-loader.ts`
- `packages/coding-agent/src/core/agent-session.ts` 的 `_bindExtensionCore()`、`_refreshToolRegistry()`、`bindExtensions()`。

重点问题：

- `input`、`before_agent_start`、`context` 三个 hook 的时机区别是什么？
- extension command 和 slash command 如何接上？
- extension tool 如何进入 active tools？
- provider registration 如何接入模型调用？
- stale extension context 为什么要 invalidation？

练习：

- 写一个只注册 command 的最小 extension。
- 写一个 `context` hook，在每次 LLM 请求前追加 custom message。
- 写一个 tool_result hook，给结果 details 加字段。

## Block 9：Memory 系统如何设计？（P0）

目标：把 session、skills、context hook 组合成长期记忆系统。Memory 关注提取、归档、检索、注入，不等于自我进化。

先读：

- `packages/coding-agent/src/core/agent-session.ts` 的 `subscribe()`、`prompt()`、`followUp()`、`steer()`。
- `packages/coding-agent/src/core/session-manager.ts`
- `packages/coding-agent/src/core/extensions/runner.ts` 的 `emitContext()`。
- `packages/coding-agent/src/core/skills.ts`
- `packages/agent/src/harness/session/memory-repo.ts`

重点问题：

- pi 的 session storage 和长期 memory 有什么区别？
- 从哪些事件抽取记忆：`message_end`、`turn_end`、`agent_end`，分别适合什么？
- 记忆写到哪里：外部 markdown/json、向量库、skill、extension 自己的资源？
- 检索到的记忆应该通过 system prompt、custom message、context hook 还是 skill 注入？
- 如何做去重、过期、优先级和人工确认？

练习：

- 做一个最小 memory loop：从一次 session 摘出一条偏好，写入外部 markdown/json，再用 context hook 注入下一次请求。
- 比较“把偏好写成 skill”和“把偏好写进外部 memory store”的差异。
- 设计一条 memory schema：事实、偏好、项目约定、任务状态分别怎么存。

## Block 10：自我进化如何受控发生？（P0）

目标：理解自我进化是受控更新循环，不是让模型随便改自己。它关注反馈、评估、候选变更、验证、落盘和回滚。

先读：

- `packages/coding-agent/docs/extensions.md`
- `packages/coding-agent/src/core/extensions/runner.ts`
- `packages/coding-agent/src/core/agent-session.ts` 的 event、retry、compaction 路径。
- `packages/coding-agent/src/core/skills.ts`
- `packages/coding-agent/src/core/session-manager.ts`

重点问题：

- 自我进化更新的对象是什么：memory、skill、prompt template、tool guideline、模型选择策略？
- 哪些反馈信号可以触发更新：用户纠正、失败工具调用、重复任务、测试结果、人工评分？
- 候选更新如何验证：最小 replay、测试、对比前后输出、人工确认？
- 如何避免污染长期上下文：权限、审批、版本、回滚、作用域。
- 自我进化和普通 memory 的边界在哪里？

练习：

- 设计一个“失败后更新 skill”的流程：失败事件 -> 总结原因 -> 生成 patch -> 人工确认 -> 更新 `SKILL.md`。
- 设计一个“偏好升级”的流程：多次相同偏好 -> 候选长期记忆 -> 确认 -> 注入下一轮。
- 写一份自我进化 guardrail 清单：哪些内容绝不自动写入。

## Block 11：UI bot / CLI 如何归一化工作流？（P0）

目标：从“读 pi”走到“用 pi 做自己的 agent 应用”。最终形态是 build 一个 UI bot 或 CLI，把多个工作流归一化到同一个 agent 入口。

先读：

- `packages/coding-agent/docs/sdk.md`
- `packages/coding-agent/examples/sdk/*`
- `packages/coding-agent/src/core/sdk.ts`
- `packages/coding-agent/src/core/agent-session.ts`
- `packages/coding-agent/src/modes/rpc/*`
- `packages/web-ui/src/components/AgentInterface.ts`
- `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

重点问题：

- 用 SDK 嵌入 pi、用 CLI/RPC 驱动 pi、写 extension 扩展 pi，三种方式分别适合什么？
- UI bot 和 CLI 分别需要归一化什么：输入、上下文、工具、输出格式、session、memory、用户确认？
- 应用层如何把 memory 和自我进化接到同一个入口里，但不污染 core harness？
- 什么时候应该直接复用 Web UI，什么时候应该写自己的 UI？

练习：

- 做一个最小 SDK 应用：固定 prompt + 自定义 tool + 事件日志。
- 做一个最小 CLI：`my-agent <task>`，内部创建 `AgentSession`，统一事件输出和 session 保存。
- 做一个最小 UI bot：订阅 session events，渲染消息、工具状态、memory 命中和确认按钮。
