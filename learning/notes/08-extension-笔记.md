# 08 Extension 系统 · 学习笔记

> README 核心问题：tool、slash command、hook 如何进入 agent 生命周期？extension 能在哪些时机介入？
> 证据来源：`core/extensions/types.ts`、`runner.ts`、`loader.ts`、`resource-loader.ts`、`examples/extensions/*`。
> 一句话：extension 是一个拿到 `ExtensionAPI`（`pi`）的工厂函数，通过 `pi.registerXXX` 注册能力、`pi.on(event)` 挂在 agent 生命周期的 30 个事件点上；core 提供事件，extension 填策略。

---

## 题眼：extension 是什么

Context 篇见过 `transformContext` = extension 的 context hook；Tool 篇见过 extension 能注册工具。把这些串起来：**extension 是 pi 留给应用层的一整套介入点**。

一个 extension 就是个工厂函数：
```ts
type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```
它拿到 `pi`（`ExtensionAPI`），用 `pi.registerTool` / `pi.registerCommand` / `pi.on("context", ...)` 等注册能力和监听事件。core 定义"有哪些时机、能改什么"，extension 决定"这些时机做什么"。

---

## 一、ExtensionAPI 能力面（`types.ts` 的 ExtensionAPI 接口）

`pi.` 上能调的，分三类：

**注册类（registerXXX）**：
| 方法 | 作用 |
|---|---|
| `registerTool(tool)` | 注册 LLM 可调用的工具（配 `defineTool`） |
| `registerCommand(name, opts)` | 注册 slash 命令（见第五节） |
| `registerShortcut(key, opts)` | 注册键盘快捷键 |
| `registerFlag(name, opts)` / `getFlag` | 注册/读 CLI flag |
| `registerMessageRenderer(type, fn)` | 自定义 CustomMessage 的渲染 |
| `registerProvider(name, cfg)` / `unregisterProvider` | 注册模型 provider（含 OAuth/streamSimple） |

**监听类**：`on(event, handler)` —— 订阅生命周期事件（第二节）。

**动作类**：`sendMessage` / `sendUserMessage` / `appendEntry`（往 session 加东西）、`setModel` / `setThinkingLevel`、`getActiveTools` / `setActiveTools`、`setSessionName` / `setLabel`、`exec`、`events`（extension 间通信的 EventBus）。

→ 一个 extension 既能"加能力"（工具/命令/provider），也能"在运行中动手"（改消息/模型/工具集）。

---

## 二、事件生命周期（核心：`pi.on(event)` 能挂哪些点）

`ExtensionEvent` 是个 union，约 30 个事件类型。handler 签名：
```ts
type ExtensionHandler<E,R> = (event: E, ctx: ExtensionContext) => R | void;
```
**"能改什么"由返回值决定**——返回一个 result 就替换/拦截，返回 void/undefined 就不改。分发在 `runner.ts`：通用 `emit()` 遍历 `ext.handlers.get(event.type)`，多数事件另有专用 `emitXxx`。

按 agent 运行时序，关键事件（都已在 runner.ts 确认对应 emit 方法）：

| 事件 | 时机 | 能改什么 |
|---|---|---|
| `before_agent_start` | 用户提交后、loop 前 | 返回 `{message?, systemPrompt?}`——临时改本轮 prompt（Context 篇的 emitBeforeAgentStart） |
| `context` | 每次 LLM 调用前 | 返回 `{messages?}` 替换整组消息（Context 篇的 transformContext） |
| `before_provider_request` | 请求发出前 | 返回值整体替换 provider `payload`（Context 篇讲过） |
| `after_provider_response` | 收到响应头后 | 给 status/headers，无返回（只观察） |
| `tool_call` | 模型要调工具时 | 返回 `{block?, reason?}` 拦截；改参数靠**原地 mutate `event.input`** |
| `tool_result` | 工具返回后 | 返回 `{content?, details?, isError?}` 改结果 |
| `message_end` | 一条消息完成 | 返回 `{message?}` 替换（须同 role） |
| `user_bash` | 用户 `!` 执行 bash | 返回 `{operations?, result?}` |
| `input` | 用户输入 | 返回 `continue / transform / handled` |
| `resources_discover` | 资源发现时 | 返回 `{skillPaths?, promptPaths?, themePaths?}`——**extension 能给 agent 添 skill/prompt/theme** |
| `session_start` | 会话开始 | 观察/初始化 |
| `session_before_switch/fork/compact/tree` | 这些破坏性/切换操作前 | 返回 `{cancel:true}` **否决**该操作 |
| `session_shutdown` | 会话关闭 | 独立 `emitSessionShutdownEvent`，清理 |

还有 `agent_start/end`、`turn_start/end`、`message_start/update`、`tool_execution_start/update/end`、`model_select`、`thinking_level_select` 等观察点。

**这张表是 Extension 篇的核心**：它就是"pi 允许应用层介入 agent 的全部时机"。memory 注入挂 `context`、权限拦截挂 `tool_call`、防误删挂 `session_before_*`、审计挂 `before_provider_request`——都是在这张表里选一个点。

---

## 三、怎么被发现和加载

**磁盘发现**（`loader.ts` 的 `discoverAndLoadExtensions`），按序三处：
1. 项目级 `<cwd>/.pi/extensions/`
2. 全局 `~/.pi/agent/extensions/`
3. 显式配置路径

`discoverExtensionsInDir` 收 `*.ts/*.js` 直接文件 + 子目录（子目录需 package.json 有 `"pi"` 字段或 index.ts）。加载用 **jiti.import**（支持直接 import TS），把 factory 传入包装后的 `pi`；`on()` 把 handler 存进 `extension.handlers` Map。

**内联 factory**（`ResourceLoader.extensionFactories`）：不走磁盘，直接传函数进来，path 记为 `<inline:N>`。

**`noExtensions` 的真实行为**（resource-loader.ts，已核实源码）：
```ts
const extensionPaths = this.noExtensions
  ? cliEnabledExtensions               // noExtensions 时只留 CLI 显式启用的
  : this.mergePaths(cliEnabledExtensions, enabledExtensions);
const extensionsResult = await loadExtensions(extensionPaths, ...);
const inlineExtensions = await this.loadExtensionFactories(...);  // ← 内联 factory 照样加载
extensionsResult.extensions.push(...inlineExtensions.extensions);
```
→ **`noExtensions` 只挡磁盘发现，不挡内联 factory**。这印证了 06 实战篇踩过的点：crbuddy 设 `noExtensions:true` 但 `weatherExtension` 内联 factory 照样注册。

---

## 四、真实例子（examples/extensions/，"能干嘛"的证据）

| extension | hook / 注册 | 干什么 |
|---|---|---|
| `confirm-destructive.ts` | `on("session_before_switch"/"before_fork")` | `ctx.ui.confirm` 后 `return {cancel:true}` 否决销毁性操作 |
| `protected-paths.ts` | `on("tool_call")` | 判 write/edit 命中保护路径 → `return {block:true, reason}` |
| `custom-header.ts` | `on("session_start")` + `registerCommand` | `ctx.ui.setHeader(...)` 改 TUI 头 |
| `provider-payload.ts` | `on("before_provider_request")` + `after_provider_response` | 记录/替换 payload、记 status |
| `hello.ts` | `defineTool` + `registerTool` | 加一个自定义工具 |
| `commands.ts` | `registerCommand` + `pi.getCommands()` | 加一个列命令的 slash 命令 |

这些覆盖了：否决操作、拦截工具、改 UI、审计请求、加工具、加命令——正好对应第二节那张事件表的不同点。

---

## 五、slash command 机制（`registerCommand`）

注册结构 `RegisteredCommand`：
```ts
{ name, description?,
  getArgumentCompletions?(prefix) => AutocompleteItem[],   // 参数自动补全
  handler: (args: string, ctx: ExtensionCommandContext) => Promise<void> }
```
**触发**（`agent-session.ts` 的 `_tryExecuteExtensionCommand`，已核实）：用户输入切 `/` 后首个空格 → `commandName` = 空格前、`args` = 空格后整串 → `runner.getCommand(name)` → `command.handler(args, ctx)`。

- **同名去重**：多个 extension 注册同名命令，去重成 `name:suffix`。
- **handler 参数**：`args`（字符串）+ `ExtensionCommandContext`（比普通 ctx 多 `waitForIdle/newSession/fork/navigateTree/switchSession/reload` 等会话操作）。
- **返回 `Promise<void>`**，无返回值——效果全走 `ctx.ui.*` 和 ctx 方法的副作用。

→ slash command 是"给用户的入口"，和 skill 的 `/skill:name`、prompt template 并列（`SlashCommandSource = "extension" | "prompt" | "skill"`）。

---

## 核心链路

```
磁盘 <cwd>/.pi/extensions/ 或 ~/.pi/agent/extensions/  ─┐
显式配置路径                                          ─┤→ discoverAndLoadExtensions
内联 extensionFactories（noExtensions 也生效）         ─┘   → jiti.import → factory(pi)
                                                          → pi.registerXXX / pi.on(event)
                                                          → handler 存进 extension.handlers Map
运行时：agent 到某个时机 → runner.emitXxx(event)
                          → 遍历 handlers.get(type) → 按返回值 替换/拦截/否决
```

---

## 一句话总结

extension = 拿到 `ExtensionAPI` 的工厂函数：`registerXXX` 加能力（工具/命令/provider/渲染器）、`on(event)` 挂生命周期的 ~30 个事件点、动作类 API 运行中动手。core 定义时机和"能改什么"（由 handler 返回值决定），extension 填策略。它是 pi 把 agent 内部开放给应用层的总接口——memory、权限、审计、UI 定制都从这里接入。

---

## 待深入
- [ ] `registerShortcut` 的 KeyId → 触发链路，未细读（仅确认存在）。
- [ ] `resolveExtensionEntries` 对 package.json `"pi"` manifest 的完整解析规则，未逐行读。
- [ ] `EventBus`（extension 间通信）的实现，未展开。
- [ ] context ctx（ExtensionContext）完整方法面（ui.confirm/select/setHeader 等），只见于 example 用法，未读接口定义。