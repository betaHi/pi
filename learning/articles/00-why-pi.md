# 素材稿：为什么用 pi 来理解 agent harness

> 状态：这篇不再作为独立发布文章。核心动机已经并入 01；这里保留为素材池，后续写 01 和 07 时抽取使用。

## 一句话

**这个系列要回答的问题**：一个"够格"的 LLM agent 框架到底长什么样？

我用一个开源项目 [pi](https://pi.dev)（@earendil-works/pi-coding-agent）做参考实现，**从事件流、agent loop、provider 抽离、工具系统、session、extension 一路拆到底**，最后回答一个收敛问题：

> 如果你要从零写一个 agent harness，必须有哪些东西，哪些可以省，每个组件该长什么样？

## 我先承认一个偏见

大多数人聊 LLM agent 停在这一层：

```python
while True:
    response = llm.call(messages)
    if response.has_tool_call:
        result = run_tool(response.tool_call)
        messages.append(result)
    else:
        return response.text
```

这没错。但**它解释不了**：

- Claude Code 为什么能崩了之后从中间接着跑？
- Cursor 怎么知道在哪个 turn 该自动压缩历史？
- 多个工具同时跑时，事件流怎么不乱？
- 切换 OpenAI / Anthropic / Gemini 时上层代码为什么不用改？
- 用户按 ESC 中断时，正在流式吐字的模型怎么干净停下？
- 你的 wrapper 怎么让别人写插件加新工具，而不需要 fork？

这些问题的答案不在 LLM API 文档里，**在 harness 里**。

而绝大多数"agent 教程"和"用 N 行代码写一个 agent"都跳过了这层。

## 什么是 agent harness

我用这个词指：**包裹 LLM API、提供 agent 生命周期管理的整套基础设施**。最少包含：

1. **事件协议**：把 LLM 流式响应、工具调用、状态变化抽象成统一的事件
2. **agent loop**：模型 → 工具 → 模型的主循环
3. **turn 抽象**：给 agent loop 画一个"原子单元"边界
4. **provider 抽象**：抹平不同 LLM API 的差异
5. **工具系统**：注册、调度、权限、流式输出
6. **session**：状态持久化、崩溃恢复
7. **extension**：让生态能注入工具、provider、hook

一个简单 wrapper 通常只覆盖前几项。一个完整 harness 还要处理 provider、工具、session、extension 等工程问题。

Claude Code、Cursor agent、Devin、Replit Agent 这类产品背后都有各自的 harness。完整产品级 harness 很少开源，这也是 pi 适合作为学习样本的原因。

## 为什么是 pi

我筛了几个候选：

| 候选 | 问题 |
|---|---|
| Claude Code | 闭源 |
| Cursor / Devin | 闭源 |
| LangGraph | 抽象层太多，看 graph 反推不出 agent 设计意图 |
| AutoGen | 偏多 agent 编排，单 agent harness 不够典型 |
| Mastra | 年轻，社区在快速迭代，作为"稳定参考"风险大 |
| **pi** | ✅ 单 monorepo、多包分层、源码可读、覆盖完整 harness 形态 |

pi 不一定是"最好的"，但作为**学习对象**它有几个无可替代的特点：

1. **完整**：从 CLI 到 TUI 到 SDK 到 extension 全套，不是 demo
2. **分层干净**：`ai` / `agent` / `coding-agent` / `tui` / `web-ui` 五个包，职责一眼看出
3. **可读**：TypeScript、注释、文档齐全
4. **不是玩具 demo**：覆盖真实 coding agent 需要面对的模型、工具、会话和扩展问题
5. **概念稳定**：核心抽象（event、turn、tool、session）和主流 coding agent 产品要解决的问题高度相似

学懂 pi，你看 Claude Code / Cursor 这类产品时，会更容易推断它们背后的工程分层。

## 这个系列怎么读

每篇围绕一个核心问题，论点 → 证据 → 设计动机 → 反向验证。证据来自三种：

- **跑出来的事件流**（你能复现）
- **源码（带文件 + 行号）**（你能验证）
- **跟其他框架的对比**（你能选型）

不会有"完整 API 文档"那种东西 —— 用 pi 的话直接看官方文档。这个系列是**设计解读**。

### 当前文章路线

文章路线只在 [learning/README.md](../README.md) 维护。这里不再复制表格，避免素材稿和主路线漂移。

## 读完你能得到什么

- 看 Claude Code / Cursor 这类产品时，你能猜到内部结构
- 评估 agent 框架时，你有自己的判据清单（不被 marketing 牵着走）
- 写自己的 wrapper / harness 时，知道哪些必须做、哪些可以偷懒
- 跟人讨论 agent 时，不再停留在"调 API + 加工具"那一层

## 下一篇

> 用最小例子看 pi runtime 的事件流

跑一个最小 SDK 例子，把所有事件 `console.log` 一遍，10 秒看完一个 LLM agent 的完整生命周期。

---

**关于作者**：这里后续如果要发公开文章，再补个人背景；当前先保留为素材。
