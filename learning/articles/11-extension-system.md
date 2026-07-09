# Pi 系列 11｜Extension 系统：把应用策略接到 agent 生命周期上

本系列其他文章：

placeholder

> 本文主要参考 `packages/coding-agent/src/core/extensions/types.ts`、`runner.ts`、`loader.ts`、`resource-loader.ts`、`package-manager.ts`、`agent-session.ts`、`docs/extensions.md` 和 `examples/extensions/*`。

上一篇讲 skill，重点是可复用指令怎样被发现和按需加载。extension 面向的问题更大：应用想在 agent 运行时加工具、加命令、拦工具调用、改上下文、改请求体、定制 UI，该接到哪里。

pi 给出的入口是 extension。一个 extension 是工厂函数，加载时拿到 `ExtensionAPI`，运行时通过事件和上下文进入 agent 生命周期。

```ts
type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

这篇文章先看 extension 能注册什么，再看事件返回值怎样生效，最后看加载、命令和几个边界。

## 一、ExtensionAPI 的能力面

factory 拿到的 `pi` 对象可以做四类事。

第一类是注册能力：

| 方法 | 作用 |
|---|---|
| `registerTool(tool)` | 注册 LLM 可调用工具 |
| `registerCommand(name, options)` | 注册 `/command` slash command |
| `registerShortcut(key, options)` | 注册 TUI 快捷键 |
| `registerFlag(name, options)` / `getFlag` | 注册并读取 extension CLI flag |
| `registerMessageRenderer(type, fn)` | 自定义 custom message 渲染 |
| `registerProvider(name, cfg)` / `unregisterProvider` | 动态注册模型 provider |

第二类是订阅事件：`pi.on(event, handler)`。这是 extension 的主轴。不同事件的返回值有不同含义，后面单独展开。

第三类是运行时动作：`sendMessage`、`sendUserMessage`、`appendEntry`、`setSessionName`、`setLabel`、`setActiveTools`、`setModel`、`setThinkingLevel`、`exec`、`shutdown` 等。它们让 extension 可以在事件、命令或工具执行期间改变 session、工具集、模型和 UI。

第四类是上下文和通信。handler 里的 `ctx` 提供 `ui`、`cwd`、`sessionManager`、`modelRegistry`、`model`、`getContextUsage`、`compact`、`getSystemPrompt` 等。`pi.events` 是 extension 之间共享的 EventBus。

加载期和运行期要分清。factory 阶段适合注册 handler、tool、command、provider 等能力。`sendMessage` 这类动作依赖 runtime bind，通常放在事件 handler、命令 handler 或工具执行函数里调用。`registerProvider` 可以在加载期调用，runner 会先排队，bind 后再注册到 model registry。

## 二、事件和返回值规则

extension 事件覆盖 agent 的几个关键阶段：资源发现、输入处理、构造上下文、请求 provider、执行工具、写入消息、会话切换和关闭。

handler 的签名可以理解为：

```ts
(event, ctx) => result | void
```

返回值怎样生效由具体事件决定。下表列的是写 extension 时最常用的事件：

| 事件 | 时机 | 返回值行为 |
|---|---|---|
| `resources_discover` | `session_start` 后，startup/reload 时 | 返回 `skillPaths` / `promptPaths` / `themePaths`，随后追加资源并重建 system prompt |
| `before_agent_start` | 用户输入展开后，agent loop 前 | 可返回 custom message；可返回新的 `systemPrompt`，多个 handler 串联修改 |
| `context` | 每次 LLM 调用前 | 可返回 `{messages}` 替换当前消息数组；handler 依次串联 |
| `before_provider_request` | provider payload 发出前 | 非 `undefined` 返回值替换整个 payload；handler 依次串联 |
| `after_provider_response` | 响应头到达后 | 观察 status/headers |
| `input` | skill/template 展开前 | `transform` 串联改输入，`handled` 直接消费输入 |
| `tool_call` | 工具执行前 | 可 `{block, reason}` 阻止；改参数要原地修改 `event.input` |
| `tool_result` | 工具执行后 | 可替换 `content`、`details`、`isError` |
| `message_end` | 消息完成后、持久化前 | 可替换 message，role 要保持不变；替换会进入 agent state 和 session 持久化 |
| `user_bash` | 用户 `!` bash 前 | 可替换执行 operations，或直接给 result；第一个有效返回生效 |
| `session_before_switch` | 新建/恢复 session 前 | 可 `{cancel:true}` 否决 |
| `session_before_fork` | fork 前 | 可 `{cancel:true}`，也可 `skipConversationRestore` |
| `session_before_compact` | compaction 前 | 可取消或提供 compaction 结果 |
| `session_before_tree` | session tree 跳转前 | 可取消、提供 summary 或覆盖 summarization 指令 |
| `session_shutdown` | quit/reload/session replacement 前 | 清理资源 |

还有一些观察类事件：`session_start`、`session_compact`、`session_tree`、`agent_start/end`、`turn_start/end`、`message_start/update`、`tool_execution_start/update/end`、`model_select`、`thinking_level_select`。

这张表能解释很多上层功能。权限拦截常放在 `tool_call`，请求审计放在 `before_provider_request`，动态资源放在 `resources_discover`，memory 注入放在 `context`，防误操作放在 `session_before_*`。

## 三、发现和加载

默认加载路径由 `DefaultPackageManager` 和 `DefaultResourceLoader` 组合完成。

主要来源有五类：

- 自动发现：项目 `.pi/extensions/`，全局 `~/.pi/agent/extensions/`。
- settings：`extensions` 数组。
- packages：npm、git、local package 的 `extensions/` 或 `pi.extensions` manifest。
- CLI：`-e` / `--extension` 这类临时来源。
- SDK：`extensionFactories` 内联 factory。

目录发现规则比较克制：

- 直接的 `*.ts` / `*.js` 会加载。
- 子目录如果有 `package.json` 的 `pi.extensions`，按 manifest 加载。
- 子目录没有 manifest，但有 `index.ts` 或 `index.js`，加载 index。
- 不做任意深度递归。复杂扩展包通过 manifest 或 index 组织入口。

加载用 `jiti.import`，TypeScript extension 可以直接运行。factory 可以是 async，pi 会等它完成，再进入 `session_start` 和 `resources_discover` 等阶段。

`noExtensions` 的边界也要分清：它会阻止默认和配置发现的 extension；CLI 显式启用的 extension 仍会加载；SDK 传入的内联 `extensionFactories` 也会加载。这样应用可以关掉用户磁盘上的扩展，同时保留自己传入的内联能力。

## 四、slash command 的触发方式

extension 用 `pi.registerCommand(name, { handler })` 注册命令。用户输入 `/name args` 时，`AgentSession._tryExecuteExtensionCommand` 会先解析命令名，再查 runner 的 resolved commands。

几个细节会影响实际行为：

- extension command 优先于 skill 和 prompt template 展开。找到命令后直接执行，不再发送普通 prompt。
- handler 收到的是整段 `args` 字符串，以及 `ExtensionCommandContext`。
- `ExtensionCommandContext` 比普通 ctx 多 `waitForIdle`、`newSession`、`fork`、`navigateTree`、`switchSession`、`reload`。
- 多个 extension 注册同名命令时，runner 会生成去重后的 invocation name，例如 `foo`、`foo:2`。
- handler 返回 `Promise<void>`，效果通过 ctx、UI、session 操作或 `pi.sendMessage` 体现。

skill 和 prompt template 也会出现在 slash command 补全里。它们走各自的展开路径：skill 在 `_expandSkillCommand`，prompt template 在 `expandPromptTemplate`。这和 extension command 的 handler 机制不同。

## 五、示例能说明什么

`examples/extensions/` 里有不少短例子，可以按介入点理解：

| 示例 | 用到的机制 | 说明 |
|---|---|---|
| `protected-paths.ts` | `tool_call` | 拦截写入受保护路径 |
| `confirm-destructive.ts` | `session_before_switch` / `session_before_fork` | 用户确认后再允许切换或 fork |
| `provider-payload.ts` | `before_provider_request` / `after_provider_response` | 观察或替换 provider payload |
| `prompt-customizer.ts` | `before_agent_start` | 基于 active tools 和 skills 临时改 system prompt |
| `dynamic-tools.ts` | `session_start` + `registerTool` + command | 启动或命令运行时动态注册工具 |
| `plan-mode/index.ts` | `context`、`turn_end`、`appendEntry`、`sendMessage` | 做状态管理和上下文改写 |
| `message-renderer.ts` | `registerMessageRenderer` | 自定义 custom message 的 TUI 展示 |

这些例子覆盖了拦截、观察、动态注册、UI、状态持久化和上下文改写。读 extension 示例时，可以先问：它注册了什么，监听了哪个事件，返回值会怎样被 runner 使用。

## 六、边界

extension 的能力面很宽，使用时也要注意边界。

- extension 运行在本机进程里，有系统权限。安全上主要依赖来源可信和代码审阅。
- event handler 出错会进入 extension error 流，runner 会继续处理后续逻辑。
- reload、new session、resume、fork 之后旧 ctx 会失效。命令里如果切换 session，后续操作应放进 `withSession` 回调，用新的 ctx。
- `display: false` 只影响 custom message 在 UI 中的展示。是否进入 LLM context 由消息类型决定。
- 动态工具注册后，还要经过工具注册表、allowlist 和 active tools，才会成为模型实际可用的工具。

## 七、完整链路

```text
DefaultPackageManager.resolve()
  -> 找到 extension 路径
DefaultResourceLoader.reload()
  -> loadExtensions(paths)
  -> jiti.import(factory)
  -> factory(pi) 注册 handlers/tools/commands/flags/providers
AgentSession._buildRuntime()
  -> new ExtensionRunner(...)
  -> bindCore(actions, contextActions)
运行时
  -> runner.emitXxx(event)
  -> handler 返回值按事件规则修改、拦截、取消或观察
```

extension 把应用层策略接到 AgentSession 生命周期上。工具、命令、资源发现、上下文改写、请求审计、权限拦截、UI 交互，都可以通过这一套接口进入运行时。后面的 memory 和自我进化，会继续复用这些事件点。

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*