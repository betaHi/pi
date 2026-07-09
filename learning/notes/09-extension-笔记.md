# 09 Extension 系统 · 学习笔记

> 核心问题：tool、slash command、hook 如何进入 agent 生命周期？extension 能在哪些时机介入？
> 证据来源：`core/extensions/types.ts`、`runner.ts`、`loader.ts`、`resource-loader.ts`、`package-manager.ts`、`agent-session.ts`、`docs/extensions.md`、`examples/extensions/*`。
> 一句话：extension 是一个拿到 `ExtensionAPI` 的工厂函数。它通过 `pi.registerXXX` 注册能力，通过 `pi.on(event)` 订阅生命周期事件；core 提供时机和边界，extension 提供策略。

---

## 题眼：extension 是应用层介入点

前面几篇看到过 context hook、tool hook、custom message、resource discovery。它们都可以放进 extension 体系里理解。

一个 extension 的基本形态：

```ts
type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

factory 加载时拿到 `pi`，通常做三件事：
- 注册能力：tool、command、shortcut、flag、provider、message renderer。
- 订阅事件：`context`、`tool_call`、`before_agent_start`、`session_before_*` 等。
- 在事件或命令运行期调用动作：`sendMessage`、`appendEntry`、`setActiveTools`、`setModel`、`exec` 等。

注意加载期和运行期不同：factory 阶段适合注册能力；`sendMessage` 这类动作依赖 runtime bind，通常应在事件 handler、命令 handler 或工具执行时调用。`registerProvider` 例外，它在加载期会先排队，bind 后再注册到 model registry。

---

## 一、ExtensionAPI 能力面

`ExtensionAPI` 可以分成四类。

### 注册类

| 方法 | 作用 |
|---|---|
| `registerTool(tool)` | 注册 LLM 可调用工具 |
| `registerCommand(name, options)` | 注册 `/command` slash command |
| `registerShortcut(key, options)` | 注册 TUI 快捷键 |
| `registerFlag(name, options)` / `getFlag` | 注册并读取 extension CLI flag |
| `registerMessageRenderer(type, fn)` | 自定义 custom message 渲染 |
| `registerProvider(name, cfg)` / `unregisterProvider` | 动态注册模型 provider |

### 事件类

`pi.on(event, handler)` 订阅二十多个事件。handler 返回值是否生效，取决于事件类型。

### 动作类

`sendMessage`、`sendUserMessage`、`appendEntry`、`setSessionName`、`setLabel`、`setActiveTools`、`setModel`、`setThinkingLevel`、`exec`、`shutdown` 等。这些让 extension 能在运行时改变 session、工具集、模型或 UI。

### 上下文和通信

handler 的 `ctx` 提供 `ui`、`cwd`、`sessionManager`、`modelRegistry`、`model`、`getContextUsage`、`compact`、`getSystemPrompt` 等。`pi.events` 是 extension 间共享的 EventBus。

---

## 二、事件生命周期和返回值规则

不同事件的返回值规则不同，要看 runner 怎么处理。

| 事件 | 时机 | 返回值行为 |
|---|---|---|
| `resources_discover` | `session_start` 后、startup/reload 时 | 返回 `skillPaths` / `promptPaths` / `themePaths`，随后 resource loader 追加并重建 system prompt |
| `before_agent_start` | 用户输入展开后、agent loop 前 | 可返回 custom message；可返回新的 `systemPrompt`，多 extension 串联修改 |
| `context` | 每次 LLM 调用前 | 可返回 `{messages}` 替换当前消息数组；handler 依次串联 |
| `before_provider_request` | provider payload 发出前 | 非 `undefined` 返回值替换整个 payload；handler 依次串联 |
| `after_provider_response` | 响应头到达后 | 只观察 status/headers |
| `input` | skill/template 展开前 | `transform` 串联改输入，`handled` 直接消费输入 |
| `tool_call` | 工具执行前 | 可 `{block, reason}` 阻止；要改参数需原地修改 `event.input` |
| `tool_result` | 工具执行后 | 可替换 `content`、`details`、`isError` |
| `message_end` | 消息完成后、持久化前 | 可替换 message，但 role 必须不变；替换会同步进 agent state 和 session 持久化 |
| `user_bash` | 用户 `!` bash 前 | 可替换执行 operations，或直接给 result；第一个有效返回生效 |
| `session_before_switch` | 新建/恢复 session 前 | 可 `{cancel:true}` 否决 |
| `session_before_fork` | fork 前 | 可 `{cancel:true}`，也可 `skipConversationRestore` |
| `session_before_compact` | compaction 前 | 可取消或提供 compaction 结果 |
| `session_before_tree` | session tree 跳转前 | 可取消、提供 summary 或覆盖 summarization 指令 |
| `session_shutdown` | quit/reload/session replacement 前 | 清理资源 |

还有 `session_start`、`session_compact`、`session_tree`、`agent_start/end`、`turn_start/end`、`message_start/update`、`tool_execution_start/update/end`、`model_select`、`thinking_level_select` 等观察点。

这张表是 extension 系统的核心。权限拦截用 `tool_call`，请求审计用 `before_provider_request`，记忆注入用 `context`，动态资源用 `resources_discover`，防误操作用 `session_before_*`。

---

## 三、发现和加载

coding-agent 的默认加载路径由 `DefaultPackageManager` 和 `DefaultResourceLoader` 组合完成。

主要来源：
- 自动发现：项目 `.pi/extensions/`、全局 `~/.pi/agent/extensions/`。
- settings：`extensions` 数组。
- packages：npm/git/local package 的 `extensions/` 或 `pi.extensions` manifest。
- CLI：`-e` / `--extension` 这类临时来源。
- SDK：`extensionFactories` 内联 factory。

extension 目录发现规则：
- 直接的 `*.ts` / `*.js` 会加载。
- 子目录如果有 `package.json` 的 `pi.extensions`，按 manifest 加载。
- 否则子目录有 `index.ts` 或 `index.js` 时加载 index。
- 不做任意深度递归；复杂包要用 manifest 或 index。

加载使用 `jiti.import`，所以 TypeScript extension 可以直接运行。factory 可以是 async，pi 会等待它完成后再进入 `session_start`、`resources_discover` 等阶段。

`noExtensions` 的边界要写清楚：它会阻止默认/配置发现的 extension，但 CLI 显式启用的 extension 仍会加载，SDK 传入的内联 `extensionFactories` 也仍会加载。

---

## 四、slash command 机制

extension 用 `pi.registerCommand(name, { handler })` 注册命令。用户输入 `/name args` 时，`AgentSession._tryExecuteExtensionCommand` 先解析命令名，再查 runner 的 resolved commands。

关键点：
- extension command 优先于 skill/template 展开；找到后立即执行，不再发送普通 prompt。
- handler 参数是整段 `args` 字符串和 `ExtensionCommandContext`。
- `ExtensionCommandContext` 比普通 ctx 多 `waitForIdle`、`newSession`、`fork`、`navigateTree`、`switchSession`、`reload`。
- 同名命令会生成去重后的 invocation name，例如 `foo`、`foo:2`。
- handler 返回 `Promise<void>`，效果通过 ctx、UI、session 操作或 `pi.sendMessage` 体现。

skills 和 prompt templates 也会出现在 slash command 补全里，但它们走各自的执行路径：skill 在 `_expandSkillCommand`，prompt template 在 `expandPromptTemplate`。

---

## 五、几个真实用法

| extension | 用到的机制 | 说明 |
|---|---|---|
| `protected-paths.ts` | `tool_call` | 拦截写入受保护路径 |
| `confirm-destructive.ts` | `session_before_switch` / `session_before_fork` | 用户确认后再允许切换或 fork |
| `provider-payload.ts` | `before_provider_request` / `after_provider_response` | 观察或替换 provider payload |
| `prompt-customizer.ts` | `before_agent_start` | 基于 active tools 和 skills 临时改 system prompt |
| `dynamic-tools.ts` | `session_start` + `registerTool` + command | 启动或命令运行时动态注册工具 |
| `plan-mode/index.ts` | `context`、`turn_end`、`appendEntry`、`sendMessage` | 做状态管理和上下文改写 |
| `message-renderer.ts` | `registerMessageRenderer` | 自定义 custom message 的 TUI 展示 |

这些例子覆盖了 extension 的几个主要方向：拦截、观察、动态注册、UI、状态持久化、上下文改写。

---

## 六、边界

- extension 运行在本机进程里，有系统权限；安全边界主要靠用户信任和代码审阅。
- event handler 出错会进入 extension error 流，不应该让整个 agent 崩掉。
- reload/session replacement 后旧 ctx 会失效；命令里如果切换 session，要把后续操作放进 `withSession` 回调，用新的 ctx。
- `display: false` 只影响 custom message 是否展示，不表示不进上下文；是否进上下文取决于消息类型。
- 动态工具注册后还要经过工具注册表、allowlist、active tools，才能真正出现在模型可用工具里。

---

## 核心链路

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

---

## 一句话总结

extension 是 pi 暴露应用层策略的总接口。它把工具、命令、资源发现、上下文改写、请求审计、权限拦截和 UI 交互挂到同一个 AgentSession 生命周期上。