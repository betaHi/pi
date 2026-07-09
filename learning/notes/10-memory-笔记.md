# 10 Memory 系统 · 学习笔记

> 核心问题：长期记忆如何提取、存储、检索、注入？pi 内置了什么，应用层还要补什么？
> 证据来源：`session-manager.ts`、`messages.ts`、`extensions/runner.ts`、`agent-session.ts`、`resource-loader.ts`、`docs/extensions.md`。
> 一句话：pi 没有内置长期记忆系统。它提供 session、extension event、context hook、custom message、project context 等原语；记忆的抽取、存储、检索、去重和治理要由应用层实现。

---

## 题眼：session 持久化和长期记忆是两层问题

pi 会把会话保存成 append-only session tree，这能恢复对话历史。长期 memory 还需要额外的抽取、检索和治理策略。

长期 memory 至少要回答几个问题：什么值得记、存在哪里、怎么检索、何时注入、过期和冲突怎么处理。当前源码没有专门的 embedding、向量库、记忆抽取器、去重器或长期记忆策略模块。出现的 `InMemory*` 主要是内存存储/测试用实现，和“用户长期记忆”属于不同层次。

所以这一篇重点看 pi 给了哪些能承载 memory 的接口。

---

## 一、长期记忆系统需要的四步

| 步骤 | 做什么 | pi 提供 | 应用层要补 |
|---|---|---|---|
| 提取 | 从对话、工具结果、用户纠正里挑出值得记的事实 | 事件时机，如 `turn_end`、`message_end`、`session_shutdown` | 判断规则、抽取 prompt、人工确认策略 |
| 存储 | 写入能跨会话保留的地方 | session 只能存当前会话树；`appendEntry` 可存 extension 状态 | DB、文件、向量库、账号/项目作用域 |
| 检索 | 按当前任务找相关记忆 | 无内置检索算法 | query 构造、相似度、过滤、排序、预算控制 |
| 注入 | 把相关记忆放进模型上下文 | `context` hook、custom message、project context、resource discovery | 注入格式、数量、优先级、冲突处理 |

pi 在“注入”和“事件时机”上给了现成接口；长期记忆的核心策略仍在应用层。

---

## 二、注入路径一：`context` hook，适合瞬态检索结果

`createAgentSession` 给 core `Agent` 设置了 `transformContext`，内部调用 extension runner 的 `emitContext(messages)`。

执行方式：

```text
agent 准备 LLM context
  -> transformContext(messages)
  -> runner.emitContext(messages)
  -> 每个 context handler 可返回 { messages }
  -> convertToLlm(...)
```

特点：
- 每次 LLM 调用前重新计算。
- handler 串联执行，后一个看到前一个修改后的 messages。
- 注入内容如果只存在于返回的 messages 里，不会自动写回 session tree。
- 适合“根据当前问题临时检索几条相关记忆”。

这条路的好处是干净：相关才注入，用完不留在长期 transcript 里。缺点是应用层每次都要做检索和预算控制。

---

## 三、注入路径二：custom message，适合要进入会话历史的内容

`SessionManager.appendCustomMessageEntry` 的注释写得很明确：custom message participates in LLM context。

进入 LLM 的原因在 `messages.ts`：`convertToLlm` 会把 `role: "custom"` 的消息转成 user message。

相关 API：
- `pi.sendMessage(...)`：extension 发送 custom message，可选择是否触发 turn，或用 `steer` / `followUp` / `nextTurn` 投递。
- `sessionManager.appendCustomMessageEntry(...)`：底层持久化 entry。
- `display`：控制 TUI 是否展示。`display: false` 只是隐藏 UI，不表示不进 LLM context。

适合场景：
- 用户明确要求“以后这段上下文也算当前会话的一部分”。
- extension 要把某个结构化结果作为 conversation history 保存。
- 需要导出 HTML 或恢复会话时还能看到这条注入。

不适合场景：
- 内部评分、版本号、索引游标等 memory 系统状态。这些应放 `appendEntry`，不要放 custom message。

---

## 四、状态路径：`appendEntry`，不进 LLM context

`appendEntry` 最终写的是 `custom` entry；`custom_message` entry 是另一类会进入上下文的消息。两者差一词，语义不同：

| entry | 是否进 LLM context | 用途 |
|---|---|---|
| `custom_message` | 是 | 给模型看的扩展消息 |
| `custom` | 否 | 给 extension 自己恢复状态 |

memory 系统可以用 `appendEntry` 记录内部状态，例如：最后一次抽取时间、外部 store 的 key、用户是否确认过某条记忆。这样有留痕，但不污染模型上下文。

---

## 五、静态记忆：project context 和资源

`loadProjectContextFiles` 会加载全局 agentDir 下的 `AGENTS.md`/`CLAUDE.md`，以及 cwd 到祖先目录里的同名文件。它们进入 `<project_context>`，属于静态上下文。

这适合放项目规则、团队约定、固定偏好。不适合频繁写入的动态 memory，因为它没有抽取、去重、冲突处理，也会长期占据 system prompt。

extension 还可以在 `resources_discover` 返回 skill/prompt/theme 路径。对 memory 来说，这可以做“静态化后的记忆包”：例如把稳定规则整理成 prompt 或 skill，再通过 reload 纳入资源。这仍然属于应用层策略。

---

## 六、一个可落地的 memory 设计

下面是基于 pi 原语可以搭出来的结构，属于应用层设计。

```text
提取：extension 监听 turn_end/message_end
  -> 判断用户纠正、偏好、项目事实是否值得记
  -> 必要时向用户确认

存储：写外部 store
  -> 记录 scope：user / repo / session
  -> 记录来源、时间、置信度、过期策略

检索：新 turn 前按用户输入和 cwd 查询
  -> 关键词/向量/规则混合
  -> 排序、去重、截断

注入：
  -> 任务相关、临时记忆：context hook
  -> 会话内必须保留的扩展消息：custom_message
  -> extension 内部状态：custom entry
  -> 稳定规则：AGENTS.md / prompt / skill
```

关键治理问题：
- 记忆要有作用域，避免把一个项目的规则带到另一个项目。
- 记忆要有来源和可删除路径，避免错误事实长期残留。
- 注入要有 token 预算，不能把 memory 做成第二个无限 transcript。
- 用户偏好和项目事实要分开，不要混在同一个 store 里。

---

## 一句话总结

pi 给 memory 提供的是接入点。瞬态相关记忆用 `context` hook，持久会话消息用 `custom_message`，extension 内部状态用 `custom` entry，稳定项目规则用 project context 或资源。抽取、检索、去重、作用域和治理都要应用层自己实现。