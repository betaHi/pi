# Pi 系列 16｜MCP 接入：协议之外，应用还要做什么

本系列其他文章：

- [Tool 系统（上）：一个 toolCall 的一生](04-tool-system.md)
- [Tool 系统（下）：工具从哪来、怎么暴露](04-tool-system-2.md)
- [Skills 系统：把可复用指令做成按需内容](09-skills-system.md)
- [Extension 系统：把应用策略接到 agent 生命周期上](11-extension-system.md)
- [多工具编排：一个 turn 里多个 tool 怎么跑](15-multi-tool-orchestration.md)

> 本文主要参考 pi 的 `packages/coding-agent/README.md`、`resource-loader.ts`、`extensions/runner.ts`，`pi-mcp-adapter@2.15.0` 的 `index.ts`、`init.ts`、`server-manager.ts`、`direct-tools.ts`、`proxy-modes.ts`、`metadata-cache.ts`、`lifecycle.ts`、`mcp-output-guard.ts`、`sampling-handler.ts`、`elicitation-handler.ts`，MCP 2025-06-18 specification，以及 Mario Zechner 的 *What if you don't need MCP at all?*。

下文涉及三类结论：MCP 的角色、消息和能力来自官方规范；pi 与 adapter 的当前行为来自对应版本源码；工具暴露方式和 CLI/MCP 选择属于基于这些事实的工程建议，不是源码规定的唯一做法。

MCP 解决一个具体问题：让支持 MCP 的应用用同一套协议连接不同服务。服务可以声明工具、资源和提示模板，也可以请求应用调用模型或收集用户输入；应用负责建立连接并决定是否使用这些能力。

但连接成功不等于接入完成。应用还要决定：哪些工具交给模型，服务何时启动，敏感调用是否需要确认，过大的结果怎样处理，资源和提示模板通过什么入口提供给用户或模型。

这些决定会影响上下文占用、响应时间、安全和模型选择工具的准确率。MCP 规定通信方式，具体的使用方式仍由应用决定。

pi 没有在核心代码中内置 MCP，而是通过扩展接入。本文先说明 MCP 的角色和能力，再看 `pi-mcp-adapter` 怎样把它们转换成普通 Pi 工具，以及这层转换还要处理哪些问题。

## 一、MCP 规定通信方式，应用决定使用方式

MCP 有三个核心角色：

| 角色 | 责任 | 在 pi 中的对应物 |
|---|---|---|
| Host（宿主） | 管理客户端、权限、连接生命周期、用户交互和上下文 | pi 与 MCP 扩展 |
| Client（客户端） | 与一个服务端建立会话，协商能力，收发消息 | 扩展为每个服务端创建的 `Client` |
| Server（服务端） | 提供工具、资源和提示模板 | stdio 子进程或远端 HTTP 服务 |

一个宿主可以管理多个客户端，每个客户端只连接一个服务端。建立连接时不能直接发送 `tools/call`，而要先协商协议版本，交换双方支持的能力和实现信息，然后进入正常通信阶段。

服务端最常提供三类能力，它们的使用方式不同：

- **Tools**：模型可以根据工具定义选择并调用。
- **Resources**：应用决定怎样选择、展示或放入上下文。
- **Prompts**：通常由用户通过命令等入口主动使用。

协议定义了 `tools/list`、`tools/call`、`resources/read`、`prompts/get` 和列表变化通知，但没有要求应用把这些能力全部注册成模型工具。

因此，服务端提供了什么，和模型实际看见什么，是两件事。中间还要由应用做一次选择和转换。

通信也不只从客户端流向服务端。客户端还可以声明 `sampling` 和 `elicitation` 能力，让服务端反向请求应用调用模型或向用户收集输入。是否声明这些能力，以及请求到达后是否允许执行，仍由宿主决定。

<!-- 图1：MCP 角色与控制权
生图 prompt：
一张横版三层架构图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左侧大框「宿主：pi + MCP 扩展」，内部列出「上下文、权限、连接生命周期、用户界面」。中间画三个彼此隔离的「MCP 客户端」，每个客户端各连接一个右侧服务端，强调「一个客户端只连接一个服务端」。
右侧服务端下方分三类能力：「Tools / 模型可调用」「Resources / 应用决定如何使用」「Prompts / 用户主动使用」。
再画两条从服务端返回宿主的虚线箭头：「Sampling / 请求宿主调用模型」「Elicitation / 请求用户输入」，旁边标注「由宿主决定是否声明和确认」。
底部小字：「MCP 规定消息格式，应用决定这些能力怎样提供给用户和模型。」
建议文件名：./pi_16_1.png
-->

## 二、pi 为什么不内置 MCP

pi 的 README 直接写着 `No MCP`，随后给出两个选择：使用 CLI 配合 README 或 skill，或者通过扩展增加 MCP 支持。

这反映的是核心代码的范围，而不是 pi 完全不能使用 MCP。

一个通用 MCP 服务端可能提供几十个工具。每个工具都有名称、说明和参数结构。如果应用在每次会话开始时把它们全部交给模型，这些定义会持续占用上下文；服务端越多，模型也越难从大量相近工具中选择。

CLI 配合 README 或 skill 的做法更直接：只在任务需要时读取说明，再让模型调用命令。多个命令还可以通过管道、文件和脚本组合，中间结果不一定都要放进模型上下文。

但 MCP 仍有适合的场景。例如已经存在可用的 MCP 服务，需要远端 OAuth，或者需要动态发现工具、资源、提示模板和变化通知。改用 CLI 时，这些能力往往要重新实现。

所以 pi 把选择留在应用层：

```text
已有合适的 CLI
  -> CLI + README/skill

需要现成的 MCP 服务
  -> 安装 Pi package
  -> 扩展注册普通 Pi 工具和命令
  -> 核心 agent loop 不需要理解 MCP
```

这样，MCP 扩展可以单独升级、替换或禁用，连接协议的实现不会进入 agent loop、会话和模型 provider 等核心模块。

## 三、MCP 怎样通过普通扩展进入 pi

安装 `pi-mcp-adapter` 后，pi 仍然按普通 package 加载它：

```text
settings.packages
  -> DefaultPackageManager.resolve()
  -> DefaultResourceLoader.reload()
  -> loadExtensions(extensionPaths)
  -> extension factory(pi)
  -> pi.registerTool / registerCommand / on(event)
```

扩展的工厂函数主要做三件事：

1. 注册一个名为 `mcp` 的代理工具，作为统一入口。
2. 根据配置和工具信息缓存，可选地把部分 MCP 工具直接注册给 Pi。
3. 监听 `session_start` / `session_shutdown`，为当前会话创建和清理 MCP 运行状态；`eager` / `keep-alive` 还可以安排加载期初始化。

`ExtensionRunner` 最后只收集扩展注册的工具定义。对核心代码来说，`mcp` 和其它 Pi 工具没有区别：都是一个带有 `name`、`description`、`parameters` 和 `execute` 的 `AgentTool`。

MCP 协议的处理都在工具的 `execute` 后面。远端结果转换成 Pi `AgentToolResult` 后，后面的步骤继续使用普通工具链路：发送工具结果事件、保存消息，再在下一轮交给模型。

扩展从 MCP 配置中读取服务端定义。每个服务端必须且只能配置一种入口：`command` 启动本地 stdio 子进程，`socket` 连接 Unix socket，`url` 连接 HTTP 服务。`url` 路径会先尝试 **Streamable HTTP**，旧服务端不适用时再尝试 SSE。因此 Streamable HTTP 不是单独的第四种配置字段，而是 HTTP 路径优先使用的传输方式。

远端 HTTP 服务还可以使用 bearer token 或 OAuth。服务端要求 OAuth 时，连接会进入 `needs-auth` 状态，等用户完成认证后再连接。当前版本把持久化 OAuth 凭据保存到操作系统安全凭据库，而不是普通项目配置文件。

这里还要区分“扩展加载”和“服务端启动”。扩展工厂加载时会读取配置和缓存，先注册 `mcp`、缓存中的直接工具和提示模板命令；这时 `lazy` 服务端通常还没有连接。配置为 `eager` 或 `keep-alive` 的服务端可以在加载期安排初始化，以兼容不会发送 `session_start` 的嵌入式宿主。正常收到 `session_start` 后，扩展会先清理旧的加载期或上一个会话状态，再创建当前会话自己的运行状态。

真正建立连接时，`command` 型服务端才会启动 stdio 子进程；`url` 和 `socket` 连接的是已经存在的服务。缓存文件不存在时，当前会话会连接所有已启用的服务端一次来建立工具信息；缓存有效后，`lazy` 和 `lazy-keep-alive` 要等到第一次真实调用才连接。

<!-- 图2：MCP 作为扩展接入
生图 prompt：
一张横版分层流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
上层「Pi 核心」包含 PackageManager → ResourceLoader → ExtensionRunner → Agent loop。中层「pi-mcp-adapter 扩展」包含 factory 加载期注册、session_start/shutdown、McpServerManager。下层画多个「MCP 服务端」。
从 ExtensionRunner 到扩展标注「普通 AgentTool 接口」，从扩展到服务端分三条连接：「command → stdio」「url → Streamable HTTP，必要时 SSE」「socket → Unix socket」。
在 Agent loop 旁加提示：「核心代码不处理 JSON-RPC，只接收 Pi AgentToolResult」。
底部小字：「MCP 的连接和转换逻辑由扩展负责。」
建议文件名：./pi_16_2.png
-->

## 四、先通过代理工具查找，还是直接注册

扩展默认不会把远端工具全部注册给模型，而是只注册一个 `mcp` 代理工具。模型通过这个入口查找和调用 MCP 工具。

模型先搜索，再描述，最后调用：

```ts
mcp({ search: "screenshot" })
mcp({ describe: "chrome_take_screenshot" })
mcp({ tool: "chrome_take_screenshot", args: { format: "png" } })
```

搜索和查看说明可以读取本地缓存。只有真正调用工具时，扩展才需要连接服务端。

第一次安装时有一个例外。缓存文件还不存在，扩展会连接所有已启用的服务端一次，取得工具信息并建立缓存。以后缓存有效时，搜索和查看说明才不需要实时连接。

这样做可以缩短模型开始时收到的工具列表。代价是第一次使用某个 MCP 工具前，模型通常要多调用一两次 `mcp` 来搜索和查看参数。

扩展也支持直接注册工具。配置 `directTools: true` 或指定工具名后，它会把缓存中的 MCP 工具信息转换成普通 Pi 工具：

| MCP 工具信息 | Pi 工具字段 |
|---|---|
| 加上服务端前缀的名称 | `name` |
| 工具说明 | `description` / `promptSnippet` |
| `inputSchema` | `parameters` |
| `tools/call` | `execute` 内的 `client.callTool` |

直接注册可以省掉搜索步骤，适合数量少、经常使用并且定义稳定的工具。代价是这些工具定义会一直出现在模型的工具列表中。

实际使用时可以按工具数量和使用频率选择：

- 少量、常用的工具直接注册。
- 数量多、很少使用的工具通过 `mcp` 查找。
- 用 `includeTools` 和 `excludeTools` 限制当前任务可以看到的工具。
- 服务端提供的全部工具，不必全部交给模型。

这和 skill 的按需加载相似：先给模型一个入口，需要时再读取完整说明。

## 五、一次 MCP 工具调用怎样执行

以通过 `mcp` 代理工具调用为例：

```text
模型调用 Pi 工具 mcp
  -> execute 解析 tool 和 args
  -> executeCall 从缓存中找到服务端和原始工具名
  -> lazyConnect（尚未连接时）
     -> McpServerManager.connect
     -> 为该服务端创建 Client
     -> 选择 stdio / Streamable HTTP（必要时退回 SSE）/ Unix socket
     -> initialize 并协商双方能力
     -> tools/list
     -> 服务端支持时再调用 resources/list 和 prompts/list
     -> 更新内存中的工具信息和磁盘缓存
  -> client.callTool({ name: originalName, arguments: args })
  -> 把 MCP 内容和错误转换成 Pi content/details
  -> 限制过大的输出
  -> 返回 Pi AgentToolResult
  -> agent loop 生成 toolResult 消息
  -> 下一轮交给模型
```

这里有三层责任：

- **agent loop** 只调用一个 Pi 工具，不生成 MCP JSON-RPC。
- **扩展** 管理 MCP 客户端、传输连接、工具信息和结果转换。
- **服务端** 执行实际操作，并返回工具结果。

直接注册的工具只省掉前面的搜索步骤。后半段仍然需要按需连接、调用 `callTool`、转换结果并限制输出大小。

<!-- 图3：一次代理工具调用
生图 prompt：
一张横版时序图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
四条泳道：模型、Pi agent loop、MCP 扩展、MCP 服务端。
时序：模型 → agent loop「调用 mcp 工具」；agent loop → 扩展「execute(tool,args)」；扩展先查本地工具信息，若未连接则向服务端发送 initialize / initialized，再发送 tools/list；随后扩展 → 服务端「tools/call」；服务端 → 扩展「content / isError」；扩展内画「结果转换 + 输出限制」；扩展 → agent loop「Pi AgentToolResult」；agent loop → 模型「toolResult 消息」。
在本地工具信息旁标注：「缓存有效时，搜索和查看说明不需要连接」。
底部小字：「模型调用的是 Pi 工具，MCP 调用由扩展完成。」
建议文件名：./pi_16_3.png
-->

## 六、缓存和连接各有一套状态

默认的 `lazy` 模式容易被理解成“启动时没有任何工具信息”。实际实现把工具信息和实时连接分开管理。

扩展把工具、资源、提示模板和服务端说明缓存到磁盘，并根据服务端配置的摘要值和缓存时间判断是否仍然有效。缓存有效时：

- `search`、`list`、`describe` 可以在服务端未连接时工作。
- 直接注册的工具可以从缓存恢复，不必启动所有服务端。
- 真正调用才触发 `lazyConnect`。

第一次运行是例外。缓存文件不存在时，`initializeMcp()` 会连接所有已启用的服务端一次，取得并保存工具信息。缓存建立后，启动时只主动连接 `eager` 和 `keep-alive` 服务端；`lazy` 服务端等到真正调用时再连接。

空闲连接关闭后，工具信息仍然保留。下一次可以先搜索，再重新连接。

```text
工具信息缓存：用于搜索和查看参数，可以跨会话保留
实时连接：占用进程或网络资源，按生命周期管理
```

扩展提供四种连接模式：

| 模式 | 首次连接 | 空闲处理 | 断线处理 |
|---|---|---|---|
| `lazy` | 首次使用 | 默认关闭 | 下次使用重连 |
| `eager` | 会话启动 | 默认保留 | 不自动保持连接 |
| `keep-alive` | 会话启动 | 保留 | 定期检查并重连 |
| `lazy-keep-alive` | 首次使用 | 保留 | 首次连接后重连 |

会话关闭时，扩展先中止当前运行状态，刷新工具信息缓存，再执行 `gracefulShutdown -> closeAll()`。先发出中止信号，再等待连接清理，可以避免旧会话中的异步任务继续使用已经失效的上下文。

连接管理还要处理同一服务端的并发连接、取消、超时、断线、重连、OAuth 和会话切换。源码分别为这些情况设置了连接去重、中止信号、超时、重连和关闭保护。

服务端如果声明支持列表变化通知，tools、resources 或 prompts 发生变化后，adapter 会更新当前连接中的列表，重建内存信息，写回磁盘缓存，再同步提示模板命令、代理工具和直接注册的工具。服务端没有声明这项能力时，adapter 不能自动知道远端列表已经改变，需要通过重新连接等操作刷新。

## 七、资源和提示模板怎样接入 pi

MCP 的几类能力不会按同一种方式进入 Pi。

扩展默认把 resource 转成名为 `read_<resource>` 的工具。执行时调用 `resources/read`，而不是 `tools/call`。这样资源可以复用 Pi 原有的工具调用流程。这是扩展选择的接入方式，不是 MCP 协议的强制要求。

prompt 则注册成 `/mcp__<server>__<prompt>` 命令。用户触发命令后，扩展调用 `prompts/get`，再把返回的消息交给 Pi。这样，提示模板仍然由用户主动选择，而不是交给模型自动调用。

工具、资源和提示模板即使来自同一个服务端，也不必使用同一种入口。应用要根据谁来触发、内容怎样进入上下文，分别选择接入方式。

### 服务端也可以反向发起请求

MCP 还允许服务端请求客户端做两类事情：

- **Sampling**：请求宿主调用一个模型。adapter 会从服务端给出的模型偏好、当前模型和可用模型中选择，并通过 pi-ai 发起调用。默认在发送请求前和把结果返回服务端前分别确认一次。
- **Elicitation**：请求用户填写表单或打开网页。adapter 使用 Pi 的选择框和输入框收集内容，校验后再让用户确认提交；打开网页也需要单独确认。

这两项能力不是无条件开放。Sampling 在没有交互界面时默认不可用，除非明确配置自动批准，而且当前实现只支持文本，不支持让服务端带入上下文或工具。Elicitation 依赖 Pi 的交互界面，URL 模式只在 TUI 中提供。

这部分说明 MCP 是双向协议：应用不只要控制自己调用哪些远端工具，也要控制服务端可以反向请求哪些本地能力。

## 八、结果转换也是安全和上下文边界

MCP 工具结果可以包含文本、图片、音频、资源链接、内嵌资源和结构化内容。Pi 的工具结果没有完全相同的类型集合，因此扩展需要做转换：

- 文本和图片保留为 Pi content block。
- 内嵌资源和资源链接转成带 URI 的文本。
- 音频只保留一条说明其类型的文本。
- `content` 为空时，把结构化内容序列化成 JSON 文本。

错误也要转换。`isError: true`、连接错误、OAuth 失败、取消和会话失效，最终都要变成 Pi 能识别和保存的工具结果状态。

还要处理输出大小。远端工具可能一次返回几万行文本。如果全部放进上下文，模型输入和会话文件都会迅速增大。扩展默认限制正文和原始详情的大小；超过限制时只返回开头的预览，并把完整结果写到临时文件，模型可以再用 `read` 或 `grep` 查看需要的部分。这些临时文件不会自动删除，其中可能包含敏感数据。

这个限制只解决大小问题，不保证内容可靠。MCP 服务端返回的内容和附加标记可能过时、错误，也可能包含提示词注入。应用还要负责：

- 敏感调用确认和参数展示。
- 请求超时、取消和调用记录。
- 服务端和工具的允许列表。
- 凭据不进入日志和模型上下文。
- 结果必要时做结构校验和来源复核。

这里要区分已有实现和应用责任。`pi-mcp-adapter@2.15.0` 会确认 sampling 和 elicitation 这类反向请求，但普通 MCP 工具调用没有统一的调用前确认步骤。危险工具仍需要通过工具过滤、额外的 pi 扩展或隔离环境限制。

stdio 服务端也是本机进程。客户端和服务端分开，并不表示服务端运行在沙箱中。按需连接也不会减少服务端连接后拥有的文件和网络权限。

<!-- 图4：MCP 扩展需要处理的四类问题
生图 prompt：
一张横版四步处理图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
从左到右：MCP 服务端能力 →「工具选择：代理工具 / 直接注册 / 过滤」→「连接管理：缓存 / 按需连接 / 超时」→「调用控制：认证 / 取消 / 反向请求确认」→「结果处理：类型转换 / 大小限制」→ Pi 模型上下文。
每一步下方放一句短说明：「决定模型看见什么」「决定何时建立连接」「决定调用能否执行」「决定结果怎样进入上下文」。
底部小字：「MCP 扩展除了转发请求，还要管理工具、连接、权限和结果。」
建议文件名：./pi_16_4.png
-->

## 九、选型建议：如何选择 CLI 和 MCP

选择 CLI 还是 MCP，可以从现有工具和接入要求出发。

| 问题 | 更偏 CLI + skill | 更偏 MCP |
|---|---|---|
| 已有能力 | 已有稳定 CLI | 已有可用的 MCP 服务端 |
| 跨应用复用 | 不重要 | 需要在多个支持 MCP 的应用中复用 |
| 运行时发现 | 很少需要 | 工具、资源或提示模板会变化 |
| 数据组合 | 依赖管道、文件和脚本 | 主要通过协议请求完成 |
| 远端授权 | 简单 token 或环境变量 | 需要 OAuth 和远端连接 |
| 工具数量 | 用 README 按需说明 | 用代理、过滤和缓存控制工具列表 |
| 用户形态 | 技术用户可操作 shell | 需要界面、确认操作和可用工具列表 |

即使选择 MCP，也还有第二层选择：

- 少量稳定工具，可以直接注册。
- 工具很多的服务端，可以先通过 `mcp` 搜索。
- 复杂的多步数据处理，可以让代码或脚本先组合结果，再把必要部分交给模型。
- 权限较高的服务端，要单独处理授权、凭据和进程隔离。

## 十、完整链路

```text
Pi 发现 package
  -> 加载 MCP 扩展工厂函数
  -> 注册 mcp 代理工具
  -> 可选：从工具信息缓存中直接注册部分工具
  -> 如果存在 eager/keep-alive：可安排加载期初始化
  -> session_start：读取配置和缓存，设置连接方式
     -> 先停止旧的加载期或上一个会话状态
     -> 创建当前会话自己的运行状态
     -> 没有缓存：连接已启用的服务端，建立工具信息缓存
     -> 缓存有效：只按 eager/keep-alive 配置主动连接

模型需要 MCP 能力
  -> 通过 mcp 搜索和查看说明（缓存有效时不连接）
  -> 调用工具
  -> lazyConnect
  -> initialize 并协商能力
  -> 读取 tools/resources/prompts 列表
  -> tools/call 或 resources/read
  -> MCP 结果转换成 Pi AgentToolResult
  -> 限制过大的输出
  -> 通过普通 Pi toolResult 链路交给模型

服务端可选地反向请求
  -> sampling/createMessage：确认后调用模型并返回结果
  -> elicitation/create：确认后收集用户输入并返回结果

session_shutdown
  -> 中止仍在运行的任务
  -> 刷新工具信息缓存
  -> 关闭客户端、传输连接和子进程
```

<!-- 图5：MCP 接入的完整链路
生图 prompt：
一张横版端到端流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，Pi 核心使用浅灰蓝 #E6EDF2，MCP 扩展使用蓝绿色 #2A9D8F，MCP 服务端和关闭节点使用珊瑚色 #E76F51，无强渐变，无厚重阴影，中文标签清晰。
整张图从左到右分成四个阶段，并用细竖线分隔：「加载」「发现与连接」「调用与返回」「会话关闭」。
第一阶段「加载」：Pi 发现 package → 加载 MCP 扩展 → 注册 mcp 代理工具 → 可选注册直接工具和 MCP prompt 命令。
在第一阶段和第二阶段之间加一个小提示：「注册工具不等于服务端已启动」。若存在 eager / keep-alive 服务端，可画一条加载期初始化支线；随后 session_start 停止旧状态并创建当前会话状态。
第二阶段「发现与连接」：session_start → 读取配置和工具信息缓存。这里画一个清楚的分支判断「缓存是否存在？」：没有缓存 → 连接已启用的服务端 → initialize / capabilities → tools/list、resources/list、prompts/list → 写入缓存；缓存有效 → lazy 服务端保持未连接，eager / keep-alive 服务端主动连接。
第三阶段「调用与返回」：模型通过 mcp 搜索或直接调用工具 → Pi AgentTool.execute → MCP 扩展定位服务端和原始工具名 → 必要时 lazyConnect → tools/call 或 resources/read → MCP 服务端返回 content / isError → 扩展进行类型转换和输出大小限制 → 返回 Pi AgentToolResult → agent loop 生成 toolResult → 下一轮交给模型。
在第三阶段下方增加一条可选的反向支线：「MCP 服务端 → sampling/createMessage 或 elicitation/create → MCP 扩展检查已声明能力 → 模型调用确认或用户输入确认 → 将结果返回服务端」。用虚线表示它不属于每次工具调用都会发生的主流程。
第四阶段「会话关闭」：session_shutdown → 中止仍在运行的任务 → 刷新工具信息缓存 → closeAll → 关闭客户端、传输连接和 stdio 子进程。
图中使用三条水平泳道：「Pi 核心」「MCP 扩展」「MCP 服务端」，每个节点放在实际负责它的泳道中。用单向箭头表示主流程，用虚线箭头表示缓存读取和写入。右下角放一句小字：「Pi 核心只执行普通工具；MCP 扩展负责协议连接、工具转换和资源清理。」
节点文字保持短句，不展示大段代码，不使用装饰性人物或营销插画，重点是层次、分支和时序清楚。
建议文件名：./pi_16_5_full_flow.png
-->

MCP 解决了客户端和服务端怎样用统一协议交换能力。应用接入 MCP 后，还要决定给模型哪些工具，怎样管理连接，哪些正向或反向请求需要确认，以及远端结果怎样进入上下文。

pi 把 MCP 留在扩展层：核心代码只保留稳定的工具接口，扩展负责协议连接和结果转换。默认的代理工具减少初始工具数量，工具信息缓存让搜索不必总是连接服务端，按需连接减少长期占用，输出限制避免单次结果过多占用上下文。

从这个实现可以得到一个直接的判断：外部服务提供的全部能力，不必全部交给当前模型。接入协议之后，应用仍然要根据任务选择工具和上下文。

---

*本文基于对 `@earendil-works/pi-coding-agent`、`pi-mcp-adapter@2.15.0` 和 MCP specification 的阅读。*
