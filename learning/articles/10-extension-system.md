# Pi 系列 10｜Extension 系统：agent 生命周期的介入点

本系列其他文章：

placeholder

> 本文源码主要在 `core/extensions/types.ts`、`runner.ts`、`loader.ts`、`resource-loader.ts`、`examples/extensions/`。

前面几篇里，extension 出现过好几次：Context 篇它能改写整组对话，Tool 篇它能注册工具，Skills 篇它是 slash command 的一种来源。这一篇把它们收到一起看：extension 到底是什么，它能在 agent 运行的哪些时机介入。

一个 extension 是一个工厂函数：

```ts
type ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>;
```

它拿到一个 `pi` 对象，用 `pi.registerTool`、`pi.registerCommand`、`pi.on("context", ...)` 这些方法注册能力、挂监听。core 定义"有哪些时机、每个时机能改什么"，extension 决定"这些时机具体做什么"。这一篇的主线，就是把这些介入点铺开。

## 一、extension 能注册和调用什么

`pi` 对象上的方法分三类。

**注册能力**：

| 方法 | 作用 |
|---|---|
| `registerTool` | 注册一个 LLM 可调用的工具 |
| `registerCommand` | 注册一个 slash 命令 |
| `registerShortcut` | 注册一个键盘快捷键 |
| `registerFlag` / `getFlag` | 注册和读取 CLI flag |
| `registerMessageRenderer` | 自定义某种消息在 TUI 里的渲染 |
| `registerProvider` | 注册一个模型 provider |

**监听事件**：`on(event, handler)`，订阅生命周期事件——下一节详说，这是本文重点。

**运行时动作**：`sendMessage` / `appendEntry`（往 session 加内容）、`setModel` / `setThinkingLevel`、`getActiveTools` / `setActiveTools`、`setSessionName` / `setLabel`、`exec`，以及 `events`（extension 之间通信的总线）。

所以一个 extension 既能"加能力"（工具、命令、provider），也能"在运行中动手"（改消息、换模型、调整工具集）。

<!-- 图1：ExtensionAPI 能力面
生图 prompt：
一张横版三分区图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中间画一个方块标「pi（ExtensionAPI）」。向外分三组：
左组用蓝绿色 #2A9D8F 标「注册能力」，列「registerTool / registerCommand / registerShortcut / registerProvider / registerMessageRenderer」。
上组用琥珀色 #E9B44C 标「监听事件」，列「on(event, handler)」，旁注「~30 个生命周期事件（见下图）」。
右组用靛蓝 #5B7DB1 标「运行时动作」，列「sendMessage / setModel / setActiveTools / appendEntry / events」。
底部小字：「一个 extension 既能加能力，也能在运行中动手。」
-->
![ExtensionAPI 的三类能力：注册、监听、运行时动作](./pi_10_1.png)

## 二、事件生命周期

这是 extension 系统的核心。`pi.on(event, handler)` 能挂在约 30 个事件上，覆盖 agent 从收到输入到发出请求、执行工具、结束会话的全过程。

handler 的签名是 `(event, ctx) => result | void`。**"能改什么"由返回值决定**：返回一个结果，就替换或拦截；返回空，就不改。

按 agent 运行的时序，关键事件如下（每个都在源码里确认了对应的触发方法）：

| 事件 | 时机 | 返回值能做什么 |
|---|---|---|
| `before_agent_start` | 用户提交后、开始处理前 | 临时改本轮的 prompt 或 systemPrompt |
| `context` | 每次 LLM 调用前 | 替换整组 messages |
| `before_provider_request` | 请求发出前 | 替换最终的 provider payload |
| `after_provider_response` | 收到响应后 | 只观察（status、headers） |
| `tool_call` | 模型要调工具时 | 拦截（`{block, reason}`）；改参数靠原地修改 event |
| `tool_result` | 工具返回后 | 改结果内容 |
| `message_end` | 一条消息完成 | 替换这条消息（须同 role） |
| `user_bash` | 用户用 `!` 执行 bash | 改操作或结果 |
| `resources_discover` | 资源发现时 | 追加 skill / prompt / theme 路径 |
| `session_start` | 会话开始 | 初始化 |
| `session_before_switch` / `fork` / `compact` / `tree` | 这些切换或破坏性操作前 | 返回 `{cancel:true}` 否决该操作 |
| `session_shutdown` | 会话关闭 | 清理 |

除此之外还有一批观察点：`agent_start` / `agent_end`、`turn_start` / `turn_end`、`message_start` / `update`、`tool_execution_start` / `update` / `end` 等。

这张表值得记住，因为它就是"pi 允许应用层介入 agent 的全部时机"。前面几篇提到的介入，都能在这张表里定位：Context 篇的对话改写是 `context`，请求体改写是 `before_provider_request`，临时改 prompt 是 `before_agent_start`。后面 Memory 和自我进化要用的介入点，也从这张表里选。

<!-- 图2：事件生命周期时间线
生图 prompt：
一张横版时间线图，从左到右，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画一条主时间线，从左「用户提交」到右「会话关闭」。线上按时序放事件节点（小菱形），每个节点下方一行小字说明"能改什么"：
「before_agent_start（改本轮 prompt）」→「context（替换 messages）」→「before_provider_request（替换 payload）」用蓝绿色 #2A9D8F 这三个连成"发请求前"一段；
「tool_call（拦截 block）」→「tool_result（改结果）」用琥珀色 #E9B44C 连成"工具执行"一段；
「session_before_switch/fork/compact（返回 cancel 否决）」用珊瑚色 #E76F51 标"破坏性操作前"；
「session_shutdown（清理）」在最右。
每段上方用细括号标分组名。底部小字：「约 30 个事件点。返回一个结果就替换或拦截，返回空就不改。」
-->
![事件生命周期：agent 从提交到关闭的介入点时间线](./pi_10_2.png)

## 三、怎么被发现和加载

extension 有两种来路。

**磁盘发现**，按顺序找三处：项目级的 `.pi/extensions/`、全局的 `~/.pi/agent/extensions/`、以及显式配置的路径。目录里的 `.ts` / `.js` 文件，或带有效标记的子目录，都会被收进来。加载时用 jiti 直接 import（支持 TypeScript），把工厂函数传入包装好的 `pi`，`on()` 把 handler 存进这个 extension 的 handler 表。

**内联工厂**：不走磁盘，直接把工厂函数传给资源加载器。

这里有个之前实战中确认过的细节：`noExtensions` 这个开关只挡磁盘发现，不挡内联工厂。源码里，`noExtensions` 为真时磁盘那部分被跳过，但紧接着内联工厂照样加载：

```ts
const extensionPaths = this.noExtensions
  ? cliEnabledExtensions               // 只留 CLI 显式启用的
  : this.mergePaths(...);
const extensionsResult = await loadExtensions(extensionPaths, ...);
const inlineExtensions = await this.loadExtensionFactories(...);  // 内联工厂照样加载
extensionsResult.extensions.push(...inlineExtensions.extensions);
```

所以一个应用可以设 `noExtensions: true` 关掉磁盘上的 extension 发现，同时用内联工厂精确控制只加载自己需要的那几个。

<!-- 图3：extension 的三种来路
生图 prompt：
一张横版汇聚图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左边三个来源竖排，汇向中间：
1「项目级 .pi/extensions/」浅灰蓝 #E6EDF2；
2「全局 ~/.pi/agent/extensions/」浅灰蓝；
3「内联 extensionFactories」用蓝绿色 #2A9D8F 高亮。
1 和 2 上画一个开关图标标「noExtensions=true 时跳过」用珊瑚色 #E76F51；3 旁边标「noExtensions 也照样加载」用蓝绿色。
中间方框「jiti import → factory(pi)」，右边输出「handler 存进 extension.handlers 表」。
底部小字：「noExtensions 只挡磁盘发现，不挡内联工厂——应用可以只加载自己需要的那几个。」
-->
![extension 的三种来路：磁盘两处 + 内联工厂，noExtensions 只挡磁盘](./pi_10_3.png)

## 四、真实的 extension 长什么样

pi 自带的示例覆盖了各种介入点，每个都很短：

| 示例 | 介入点 | 做什么 |
|---|---|---|
| confirm-destructive | `session_before_switch` / `fork` | 弹确认框，用户取消就返回 `{cancel:true}` 否决 |
| protected-paths | `tool_call` | 判断 write / edit 是否命中受保护路径，命中就 `{block:true}` |
| custom-header | `session_start` + `registerCommand` | 改 TUI 顶部的显示 |
| provider-payload | `before_provider_request` | 记录或替换发出的请求体 |
| hello | `registerTool` | 加一个自定义工具 |
| commands | `registerCommand` | 加一个列出所有命令的 slash 命令 |

这几个例子分别落在第二节那张表的不同点上：否决操作、拦截工具、改界面、审计请求、加工具、加命令。把它们放一起看，就能感觉到 extension 系统的覆盖面——从模型请求到工具执行到会话操作到界面，几乎每个环节都开了口子。

## 五、slash command 怎么触发

extension 用 `registerCommand` 注册的命令，结构是名字、描述、参数补全函数、handler。

触发过程：用户输入以 `/` 开头，pi 切出第一个空格——空格前是命令名，空格后整串是参数——找到对应命令，调它的 handler，把参数字符串和一个上下文对象传进去。多个 extension 注册了同名命令时，会自动加后缀去重。

handler 的返回值是空的，它的效果全部通过上下文对象的方法产生副作用——改界面、开新会话、fork、切换会话等。这个上下文比普通事件的上下文多了一批会话级操作。

slash command 是"给用户的入口"，和 skill 的 `/skill:name`、prompt 模板并列，同属 pi 的三种命令来源。

## 完整链路

```
磁盘 .pi/extensions/、~/.pi/agent/extensions/、显式路径 ─┐
内联 extensionFactories（noExtensions 也生效）          ─┤→ 加载
                                                        ─┘   → factory(pi)
                                                            → pi.registerXXX 注册能力
                                                            → pi.on(event) 挂监听
                                                            → handler 存进 handlers 表
运行时：agent 到某个时机 → runner 触发该事件
                          → 遍历对应 handler → 按返回值 替换 / 拦截 / 否决
```

回过头看，extension 是 pi 把 agent 内部开放给应用层的总接口。前面几篇讲的机制——loop、tool、session、context——都是 agent 的内部构造；extension 则是在这些构造的关键节点上留出的介入点。想加工具、拦请求、改上下文、否决危险操作、定制界面，都从这里接入。后面两篇的 Memory 和自我进化，本质上也是在这张事件表上选点，接上应用层自己的逻辑。

<!-- 图4：extension 作为总接口
生图 prompt：
一张横版结构图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中间画一个大的圆角方框标「agent 内部：loop / tool / session / context」，浅灰蓝 #E6EDF2。方框边缘均匀分布若干个小接口点（用蓝绿色 #2A9D8F 小圆），每个接口点旁一行小字：「context」「tool_call」「before_provider_request」「session_before_fork」「resources_discover」等。
方框外围一圈用琥珀色 #E9B44C 标「extension」，用细线连到这些接口点。
右下角用靛蓝 #5B7DB1 标注三个应用「memory」「权限控制」「自我进化」，各用一条线接到其中一个接口点，标「都从这张事件表选点接入」。
底部小字：「extension 是 pi 把 agent 内部开放给应用层的总接口。」
-->
![extension 是 agent 内部开放给应用层的总接口](./pi_10_4.png)

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*
