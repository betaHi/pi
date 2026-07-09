# Pi 系列 12｜Memory 系统：会话记录之外，还要补哪些能力

本系列其他文章：

placeholder

> 本文主要参考 `session-manager.ts`、`messages.ts`、`extensions/runner.ts`、`agent-session.ts`、`resource-loader.ts` 和 `docs/extensions.md`。

前面讲 skill 和 extension 时，已经看到 pi 能把额外信息接进上下文。到了 memory，问题变成：长期记忆如何从对话里产生，存到哪里，怎样在合适的时候回到模型上下文。

这篇先划清一个边界：pi 会保存会话历史，但长期记忆还需要抽取、存储、检索、去重、作用域和删除策略。当前源码没有专门的长期记忆模块。pi 提供的是可承载 memory 的接口，应用层要补完整策略。

## 一、session 记录和长期记忆的差别

pi 的 session tree 可以恢复一段会话。它记录用户消息、assistant 消息、工具结果、compaction、custom entry 等。这能解决“这次会话如何继续”的问题。

长期记忆要解决的问题不同：

- 哪些信息值得从对话里抽出来。
- 这些信息属于用户、项目、仓库还是当前 session。
- 存储在文件、数据库、向量库或其他系统里。
- 当前任务开始时，怎样找到相关条目。
- 错误或过期的记忆怎样删除或降权。

当前源码里没有专门的 embedding、向量库、记忆抽取器、去重器或长期记忆策略。出现的 `InMemory*` 主要是内存存储或测试用实现。这个事实很重要：session 持久化可以作为输入来源，也可以作为审计记录；长期记忆的策略需要另外设计。

## 二、长期记忆系统的四步

一个实用的 memory 系统通常要做四步。

| 步骤 | 做什么 | pi 提供 | 应用层要补 |
|---|---|---|---|
| 提取 | 从对话、工具结果、用户纠正里挑出值得记的事实 | `turn_end`、`message_end`、`session_shutdown` 等事件时机 | 判断规则、抽取 prompt、人工确认策略 |
| 存储 | 写入能跨会话保留的地方 | session 记录当前会话；`appendEntry` 可存 extension 状态 | DB、文件、向量库、账号/项目作用域 |
| 检索 | 按当前任务找相关记忆 | 当前没有内置检索算法 | query 构造、相似度、过滤、排序、预算控制 |
| 注入 | 把相关记忆放进模型上下文 | `context` hook、custom message、project context、resource discovery | 注入格式、数量、优先级、冲突处理 |

pi 在“事件时机”和“上下文入口”上给了接口。记忆质量主要取决于应用层的提取、存储和检索策略。

## 三、路径一：`context` hook，适合临时相关记忆

`createAgentSession` 会给 core `Agent` 设置 `transformContext`。这个函数内部调用 extension runner 的 `emitContext(messages)`。

执行顺序大致是：

```text
agent 准备 LLM context
  -> transformContext(messages)
  -> runner.emitContext(messages)
  -> 每个 context handler 可返回 { messages }
  -> convertToLlm(...)
```

用它注入记忆，有几个特点：

- 每次 LLM 调用前重新计算。
- 多个 context handler 会串联执行。
- 如果注入内容只存在于返回的 messages 里，它不会自动写回 session tree。
- 适合当前问题相关、用完可以丢弃的记忆。

这条路径适合做检索式记忆：根据当前输入、cwd、工具状态，从外部 store 找出少量相关条目，压缩成一段说明，插进 messages。它的难点在检索质量和 token 预算。

## 四、路径二：custom message，适合进入会话历史的内容

`SessionManager.appendCustomMessageEntry` 的注释写明 custom message participates in LLM context。进入模型上下文的原因在 `messages.ts`：`convertToLlm` 会把 `role: "custom"` 的消息转成 user message。

相关 API 包括：

- `pi.sendMessage(...)`：extension 发送 custom message，可以选择是否触发 turn，也可以用 `steer`、`followUp`、`nextTurn` 投递。
- `sessionManager.appendCustomMessageEntry(...)`：底层持久化 entry。
- `display`：控制 TUI 是否展示。`display: false` 只隐藏 UI 展示，消息仍可能进入 LLM context。

custom message 适合放那些确实要成为会话历史一部分的内容，例如扩展生成的结构化结果、用户明确要求保留的上下文、恢复会话后仍要看到的说明。

它不适合保存 memory 系统内部状态。评分、版本号、外部 store key、抽取游标这类信息，应放到 `appendEntry`。

## 五、路径三：`appendEntry`，适合存内部状态

`appendEntry` 写入的是 `custom` entry。它和 `custom_message` entry 很接近，行为差别很大：

| entry | 是否进 LLM context | 适合存什么 |
|---|---|---|
| `custom_message` | 是 | 给模型看的扩展消息 |
| `custom` | 否 | extension 自己恢复状态 |

memory 系统可以用 `appendEntry` 记录内部状态，例如最后一次抽取时间、外部记忆条目的 id、用户是否确认过某条记忆。这样做有会话留痕，也不会把内部字段塞进模型上下文。

## 六、静态记忆：project context 和资源

`loadProjectContextFiles` 会加载全局 agentDir 下的 `AGENTS.md` / `CLAUDE.md`，以及 cwd 到祖先目录里的同名文件。这些内容进入 `<project_context>`，更像静态上下文。

静态上下文适合放项目规则、团队约定、固定偏好。它不适合频繁写入的动态记忆，因为它没有抽取、去重、冲突处理，也会长期占用 system prompt。

extension 还可以在 `resources_discover` 里返回 skill、prompt、theme 路径。memory 系统可以把稳定规则整理成 prompt 或 skill，再通过资源发现和 reload 纳入运行时。这仍然是应用层策略，pi 处理资源加载和生效。

## 七、一个可落地的 memory 设计

结合上面的接口，可以搭出这样的结构：

```text
提取：extension 监听 turn_end / message_end
  -> 判断用户纠正、偏好、项目事实是否值得记录
  -> 必要时向用户确认

存储：写外部 store
  -> 记录 scope：user / repo / session
  -> 记录来源、时间、置信度、过期策略

检索：新 turn 前按用户输入和 cwd 查询
  -> 关键词、向量或规则混合
  -> 排序、去重、截断

注入：
  -> 当前任务相关的临时记忆：context hook
  -> 要进入会话历史的扩展消息：custom_message
  -> extension 内部状态：custom entry
  -> 稳定规则：AGENTS.md / prompt / skill
```

几个治理问题会直接影响效果：

- 记忆要有作用域，避免把一个项目的规则带到另一个项目。
- 记忆要有来源和删除路径，方便修正错误事实。
- 注入要有 token 预算，避免把 memory 变成另一段很长的 transcript。
- 用户偏好和项目事实建议分开存储。

## 八、和下一篇的关系

memory 关注“把事实或偏好带回上下文”。下一篇自我进化关注“根据反馈调整行为规则”。两者会复用相同接口：session 事件、extension hook、resource loader、custom entry。区别在于，memory 主要做检索和注入；自我进化还要处理评估、验证和版本回退。

pi 给 memory 提供的是接入面。要做出稳定可用的长期记忆，还需要应用层把提取、存储、检索、作用域和治理补齐。

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。文中的 memory 系统设计是基于这些接口的应用层方案。*