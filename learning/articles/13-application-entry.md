# Pi 系列 13｜应用入口：一个 agent 内核，多种外壳

本系列其他文章：

placeholder

> 本文源码主要在 `core/sdk.ts`、`packages/agent/src/agent.ts`、`modes/`、`packages/web-ui`、`examples/sdk/`。

前面十二篇拆的都是 agent 的内部机制——loop、tool、session、context、skill、extension、memory、自我进化。这一篇反过来看：这些机制怎么被包成一个能用的应用入口，以及怎么用同一个 agent 内核，支撑 CLI、TUI、远程调用、浏览器等多种形态。

pi 的回答是两层入口加多种形态。这一篇顺着这条线走一遍。

## 一、两层入口

pi 有两个层级的入口。

底层是 `new Agent`——一个纯粹的对话引擎。它管的是 transcript、事件流、工具执行、消息队列，方法有 `prompt`、`continue`、`subscribe`、`steer`、`abort` 等，底层跑 agent loop。它不管鉴权、不管模型解析、不管会话持久化、不带内置工具，这些都要使用者自己接。

上层是 `createAgentSession`——在 `new Agent` 外面包了一整套设施。它的内部装配顺序是：

```
建鉴权存储 → 建模型注册表 → 建设置管理 → 建会话管理
  → 建资源加载器并加载
  → 解析模型、限制思考等级
  → 设默认激活工具 read / bash / edit / write
  → new Agent（core 引擎）
  → new AgentSession（包一层设施）
```

有一个设计细节值得点出：传给 `new Agent` 的发请求函数是一个闭包，它内部先从模型注册表取出对应的 key 和 header，再发请求。也就是说，**core 引擎本身不碰鉴权，"用哪个 key、什么 header"是 `createAgentSession` 这层填进去的**。这就是分层——core 只负责发请求，鉴权是外面这层的事。

<!-- 图1：两层入口
生图 prompt：
一张横版嵌套图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画一个大方框标「createAgentSession」用蓝绿色 #2A9D8F 边框，里面围绕一个小方框「new Agent（纯对话引擎）」用琥珀色 #E9B44C。
大方框内、小方框外的环形空间里，均匀放几个小卡片标「鉴权存储」「模型注册表」「会话管理」「资源加载」「内置工具」「skill / extension」，标「createAgentSession 加的设施」。
小方框「new Agent」下方标「只管 transcript / 事件 / 工具执行 / 队列」。
一根箭头从「鉴权存储」指向 new Agent 的"发请求函数"，标「鉴权在这层闭包里填，core 不碰」。
底部小字：「core 是纯引擎；createAgentSession 在外面装配一整套设施。」
-->
![两层入口：new Agent 纯引擎，createAgentSession 包一层设施](./pi_13_1.png)

两层的分工很清楚：

| | new Agent | createAgentSession 额外加的 |
|---|---|---|
| transcript、事件、工具执行、队列 | 有 | — |
| 鉴权、模型解析 | 无 | 有 |
| 会话持久化和恢复 | 无 | 有 |
| 资源加载、内置工具、skill、extension | 无 | 有 |

选哪个，取决于需求。已经有自己的存储、协议、模型路由，只缺一个对话内核，用 `new Agent`。想要 skill、extension、会话、memory 这些地基，用 `createAgentSession`。（本系列实战篇里的 crbuddy 就走过这条路——最初用 `new Agent`，后来因为要 skill、agent、memory 迁到了 `createAgentSession`。）

## 二、一个 session，多种运行形态

同一个 AgentSession，可以跑成不同的形态。pi 内置三种模式，按启动参数派发：

| 模式 | 场景 |
|---|---|
| print | CLI 单发即退（发一个 prompt，输出结果就退出） |
| interactive | 终端交互界面（TUI） |
| rpc | 远程或程序调用 |

其中 rpc 模式最能体现"归一化"的思路。它把 AgentSession 暴露成一个读写标准输入输出的服务：从标准输入读 JSON 命令（prompt、steer、abort、切换模型、压缩、fork 等），订阅 session 的事件、把每个事件序列化成 JSON 写到标准输出。

```
接管标准输出 → 从标准输入按行读 JSON 命令
  → 分发（prompt / steer / abort / set_model / compact / fork ...）
  → session 订阅事件 → 每个事件序列化成 JSON 写标准输出
```

这套基于标准输入输出的 JSON 协议，让任何一个进程只要会读写这套协议，就能驱动同一个 agent。前端是什么无所谓——命令行脚本、另一个程序、一个 web 服务，后面都是同一个 AgentSession。

<!-- 图2：一个 session 多种形态
生图 prompt：
一张横版分叉图，中间一个核心向外接多个外壳，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中间画一个核心方块「AgentSession」用蓝绿色 #2A9D8F。向外接四个外壳方块：
1「print：CLI 单发即退」浅灰蓝 #E6EDF2；
2「interactive：终端 TUI」浅灰蓝；
3「rpc：JSON over 标准输入输出」用琥珀色 #E9B44C 高亮，旁边画双向箭头标「stdin 进命令 / stdout 出事件」；
4「web-ui：浏览器组件」靛蓝 #5B7DB1。
底部小字：「同一个 session，接不同外壳。rpc 的 JSON 协议让任意前端驱动同一个 agent。」
-->
![一个 AgentSession，接 print / interactive / rpc / web 多种外壳](./pi_13_2.png)

## 三、浏览器端的复用

pi 还有一个浏览器端的聊天组件，它复用的是底层的 `new Agent`，不走 `createAgentSession`。它自己带一套浏览器侧的设施——用 IndexedDB 做持久化、浏览器侧的存储管理鉴权和会话，发请求函数走浏览器的方案——替代了 `createAgentSession` 在 Node 端提供的那套。

这正好印证了分层的价值：底层的 `Agent` 是和界面、和运行环境无关的内核，Node 端和浏览器端各自配自己的存储、鉴权、传输，共用同一个对话引擎。同一套核心逻辑，既能在服务器上跑，也能在浏览器里跑。

## 四、最小可跑的应用

一个 pi 应用的骨架非常短。全默认的版本，一行就建好了 session：

```ts
import { createAgentSession } from "@earendil-works/pi-coding-agent";

const { session } = await createAgentSession();   // 全默认：自动发现 skill、extension、工具，从设置里选模型
session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});
await session.prompt("What files are in the current directory?");
session.dispose();
```

要完全控制的话，多几步显式的准备：建鉴权存储、建模型注册表、取模型，再把它们传进 `createAgentSession`。但骨架是一样的——建 session、订阅事件、发 prompt、结束时释放。前面十二篇讲的所有机制，都在 `createAgentSession` 这一行背后装配好了。

<!-- 图3：最小应用骨架
生图 prompt：
一张竖向四步流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
四个步骤从上到下用圆角方块，编号：
1「createAgentSession()」用蓝绿色 #2A9D8F，旁注「所有机制在这行背后装配」；
2「session.subscribe(event => ...)」标「订阅事件流」；
3「await session.prompt(...)」用琥珀色 #E9B44C 标「发一个 prompt」；
4「session.dispose()」标「结束时释放」。
右侧用一个细括号把四步框起来，标「一个 pi 应用的骨架」。
底部小字：「建 session、订阅、prompt、释放——前十二篇的机制都在第一行背后。」
-->
![最小应用骨架：建 session、订阅、prompt、释放](./pi_13_3.png)

## 五、"归一化入口"是什么意思

把散落的工作流收到一个 agent 入口，具体指三件事：

- **一个内核**：AgentSession 或 Agent，承载 loop、tool、session、context、skill、extension 的全部机制；
- **多个外壳**：print、interactive、rpc、web，各自把这个内核接到不同的前端；
- **一套协议**：rpc 基于标准输入输出的 JSON 协议，让任何前端都能驱动同一个 agent。

所以"build 一个 UI bot 或 CLI"不是给每个前端各写一套 agent 逻辑，而是：写好一个 `createAgentSession`（或 `new Agent`），然后选一种形态，或者像 crbuddy 那样自定义一种形态（web 服务加事件流），把这个内核暴露出去。agent 逻辑只有一份，入口形态可以有很多种。

<!-- 图4：归一化入口
生图 prompt：
一张横版汇聚再分发图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左边画多个不同的前端图标（CLI 终端、聊天窗口、机器人 bot、web 页面），用细线全部汇向中间。
中间一个大方块「一份 agent 逻辑：createAgentSession / new Agent」用蓝绿色 #2A9D8F。
方块内部用小字列「loop / tool / session / context / skill / extension」。
右边标「选一种形态或自定义一种（如 crbuddy 的 web 服务 + 事件流）」用琥珀色 #E9B44C。
底部小字：「agent 逻辑只有一份，入口形态可以有很多种。这就是归一化。」
-->
![归一化：多种前端，一份 agent 逻辑，多种入口形态](./pi_13_4.png)

## 系列收尾

这是本系列的最后一篇。回头看整条线：从最小例子看事件流，到 agent loop 与 turn，到 provider 抽象，到工具系统，到 session、context，到 skill、extension，再到 memory、自我进化，最后到应用入口。

这些机制不是孤立的。session 的 append-only 树，被 context 的投影用、被 memory 的持久注入用、被自我进化的状态存储用；extension 的事件表，是 context 改写、memory 注入、自我进化更新共同的接入点；分层的 core 与应用层，让同一个内核能跑在 CLI、TUI、rpc、浏览器上。理解一个 agent harness，理解的就是这些机制怎么彼此咬合，最后收到一个可用的入口上。

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*
