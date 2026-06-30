# Pi 学习与文章路线

这个目录有两个目标：先把 pi 内化成 agent harness 工程能力，再把关键机制写成一组文章。不要把目标变成“读完所有源码”。源码细节服务于判断力：一个可用的 agent harness 为什么要有事件流、turn、provider 抽象、工具系统、session、skills、extension，以及怎样在这些基础上做自己的 pi 应用。

先区分两个边界：memory 和自我进化不是一回事。Memory 关注“记忆提取 -> 归档 -> 检索 -> 注入上下文”；自我进化关注“从反馈中评估表现 -> 更新规则/skills/prompts/tools -> 验证效果 -> 控制回滚”。pi 提供的是 session、skills、context hook、extension、资源加载这些基座；长期 memory 和自我进化策略要作为应用层系统自己设计。

## 目录分工

- `articles/`：对外文章，只讲关键机制、运行证据和工程启发。
- `00-how-to-learn.md`：学习方法，提醒哪些机制值得吃透，哪些只是实现细节。
- `01-architecture-map.md`：空间地图，回答各包边界和依赖方向。
- `02-runtime-flow.md`：时间线地图，回答一次 prompt 如何穿过 runtime。
- `03-learning-blocks.md`：分块学习路线，按 P0/P1/P2 控制深度。
- `04-source-reading-checklist.md`：源码复盘清单，给深入验证用，不是文章大纲。
- `05-debugging-labs.md`：调试实验，用最小 harness 验证机制。
- `06-code-evidence-map.md`：源码证据库，给文章和笔记引用。
- `notes/`：原始观察，可以保留修正痕迹；写文章前必须核对最新源码结论。

## 第一轮学习顺序

第一轮只追主链路，不追所有文件：

1. 读 [articles/01-step-zero-event-stream.md](articles/01-step-zero-event-stream.md)：先跑起来，看见事件流和 turn。
2. 读 [01-architecture-map.md](01-architecture-map.md)：建立包边界和依赖方向。
3. 读 [02-runtime-flow.md](02-runtime-flow.md)：把一次 prompt 串成时间线。
4. 对照 [06-code-evidence-map.md](06-code-evidence-map.md)：给每个结论找源码锚点。
5. 按 [03-learning-blocks.md](03-learning-blocks.md) 的 P0 块深入：provider、agent loop、context、compaction、工具、session、skills/extension、memory、自我进化。
6. 用 [05-debugging-labs.md](05-debugging-labs.md) 做最小实验，验证一个结论后再继续。
7. 最后做一个小 pi 应用：build 一个 UI bot 或 CLI，把分散的工作流归一化成同一个 agent 入口。

每次学习只要求输出三样东西：

- 一条调用链。
- 三到五个关键类型或函数。
- 一个能用断点、日志或最小 harness 验证的结论。

## 文章路线

这里是唯一维护的文章路线。对外系列按“工程问题”组织，不按包名组织。01 同时承担动机和 step 0，不再单独发一篇“为什么选择 pi”。其它学习文档只提供证据、实验和笔记模板，不再重复维护路线表。

| # | 文章主题 | 核心问题 | 主要证据来源 |
|---|---|---|---|
| 01 | 用最小例子看 pi runtime 的事件流 | 为什么要看 harness？agent runtime 第一眼长什么样？ | `articles/01`、`02-runtime-flow` |
| 02 | Agent loop 与 turn | 模型和工具如何往返？一次 prompt 为什么会拆成多个 turn？ | `agent-loop.ts`、`06-code-evidence-map` |
| 03 | Provider 抽象 | 不同 LLM API 如何统一成一套事件协议？ | `packages/ai`、provider 事件 |
| 04 | Tool 系统 | 从 tool call 到本地执行、结果回传，中间有哪些工程边界？ | `agent-loop.ts`、`core/tools/*` |
| 05 | Session 系统 | 对话怎么存、恢复、分支？为什么是 append-only entry 树而不是 messages 数组？ | `session-manager.ts`、compaction、`buildSessionContext` |
| 06 | Context 构建 | 发给模型的输入怎么拼出来、怎么裁剪？为什么不只是 messages？ | `system-prompt.ts`、`transformContext`、`convertToLlm` |
| 07 | Skills 与 Extension | reusable instruction、slash command、hook、tool 如何进入生命周期？ | `skills.ts`、`resource-loader.ts`、`extensions/*` |
| 08 | Memory 系统 | 长期记忆如何提取、存储、检索、注入？pi 提供了什么，应用层还要补什么？ | session events、context hook、external store |
| 09 | 自我进化 | agent 如何从反馈中更新规则、skills 或工具策略，并避免污染长期上下文？ | memory、skills、extension hooks、eval loop |
| 10 | Pi 应用与归一化入口 | 怎么 build 一个 UI bot 或 CLI，把工作流归一化到一个 agent 入口？ | SDK/RPC/Web UI/examples、前 9 篇总结 |

每篇文章固定控制在一个问题、一张图、一条调用链、三到五个源码证据、一个工程启发。能帮助读者成为 agent 工程师的机制要讲透；API 细节和文件清单放回学习材料。内部学习路线也按工程问题组织，避免重新退回“按包读源码”。

## 总体判断

pi 不是一个单文件 agent demo，而是分层 agent harness：

- `packages/ai`：统一多 provider LLM API，把不同厂商转成同一套 `AssistantMessageEvent` 流。
- `packages/agent`：通用 agent runtime，负责 prompt、turn、tool call、queue、event stream 和状态。
- `packages/coding-agent`：面向代码任务的完整应用层，负责 CLI、TUI、工具、skills、会话、压缩、模型选择、扩展、资源加载。
- `packages/tui`：终端 UI 基础库，提供组件树、焦点、输入、差量渲染和常用控件。
- `packages/web-ui`：浏览器聊天组件，复用 `pi-agent-core` 和 `pi-ai`，但走 Web Component/Lit UI。

## 取舍原则

每个模块都按同一模式学习：

1. 先抓机制：它解决什么 agent 工程问题。
2. 再看类型：边界数据结构是什么。
3. 再追调用链：机制如何被串起来。
4. 最后写最小实验验证，不为读源码而读源码。

Memory 按长期上下文系统学习：先看 pi 的 session 和 skills 如何保存、暴露和注入信息，再设计自己的记忆提取、检索、去重和注入策略。不要把 `InMemorySessionRepo` 误读成“长期记忆”；它只是内存 session storage。

自我进化按评估和变更控制学习：它不是“让模型随便改自己”，而是让 agent 基于反馈提出更新，经过验证后再写入 memory、skills、prompts 或工具策略。重点是边界、确认、回滚和可观测性。

最终应用目标是 build 一个 UI bot 或 CLI。它不是再讲一个源码模块，而是把前面的 event、tool、session、skills、memory、extension 归一化成一个可用入口。

当前目录只放学习材料，不修改项目运行逻辑。
