# 09 Memory 系统 · 学习笔记

> README 核心问题：长期记忆如何提取、存储、检索、注入？pi 提供了什么，应用层还要补什么？
> 证据来源：`session-manager.ts`（`appendCustomMessageEntry`）、`extensions/runner.ts`（`emitContext`）、README 边界说明。
> 一句话：**pi 没有内置长期 memory**（grep 坐实）。它提供的是"能挂记忆的原语"——session 事件、context hook、custom_message 进树、resource loader；提取/存储/检索/去重/注入策略要应用层自己设计。

---

## 题眼：先分清 memory 不是 pi 的功能，是应用层系统

README 开宗明义把 memory 和自我进化划到"应用层系统"，不是 pi 内置。这一篇最重要的一句：**别把 pi 的 session storage 误读成"长期记忆"**。

坐实（grep 全仓）：
- 搜 `embedding` / `vectorstore` / `retriev` / `semantic search` / `long-term memory` —— **没有相关实现**（唯一 `retriev` 命中在 rpc-mode，与记忆无关）。
- 搜 `memory` —— 只有 5 处，都是 `InMemory*`（内存缓存，如 `AuthStorage.inMemory`、`InMemorySessionRepo`），不是长期记忆。

结论：**pi 不提供记忆的提取、向量检索、去重、外部存储**。这些是应用层要补的。

那 pi 提供什么？——**注入点和原语**。下面把"应用层要做的四步（提取/存储/检索/注入）"和"pi 给了哪些原语"对上。

---

## 一、记忆系统的四步，pi 各管到哪

一个长期记忆系统通常四步。逐个看 pi 给了什么、缺什么：

| 步骤 | 做什么 | pi 提供的原语 | 应用层要补 |
|---|---|---|---|
| **提取** | 从对话里挑出"值得记的" | 无 | 全部（判断什么该记、抽成条目） |
| **存储** | 存到能长期保留、可检索的地方 | 无（session JSONL 只是当前会话的） | 外部 store（DB/向量库/文件） |
| **检索** | 按当前任务找相关记忆 | 无 | 全部（query、相似度、排序） |
| **注入** | 把检索到的记忆放进模型上下文 | ✅ 有两条路（见下） | 决定注哪些、注多少 |

→ **pi 只在"注入"这一步给了现成机制，前三步（提取/存储/检索）完全是应用层的事。**

---

## 二、pi 提供的"注入"原语：两条路

### 路一：context hook（瞬态注入）—— `emitContext`

extension 注册 `context` handler，在每次 LLM 调用前改整组 messages（Extension 篇讲过）：
```
runner.emitContext(messages)  → 每个 context handler 依次改写 → 返回新 messages
```
特点：
- **瞬态**：每 turn 现算，注入的内容**不进 session 树**、不留痕。
- 适合"这次检索到的相关记忆，临时插进去"——query 依赖、用完即弃。
- 真实证据：`examples/extensions/plan-mode` 用 `pi.on("context")` 改上下文（虽非 memory，但证明这条路能用来注入）。

### 路二：custom_message 进树（持久注入）—— `appendCustomMessageEntry`

往 session 树里加一个 `custom_message` entry，它会进入 LLM context。源码注释明写：
```ts
// Append a custom message entry (for extensions) that participates in LLM context.
appendCustomMessageEntry<T>(customType, content, display, details?)
```
特点：
- **持久**：进 session 树、留痕、可随会话恢复（Session 篇的 append-only 树）。
- 投影时 `convertToLlm` 把 `custom` 翻成 user 消息（Context 篇的映射表）。
- 适合"记住这条，之后每次投影都带上"——稳定的长期上下文。

### 两条路的选择

| | context hook | custom_message 进树 |
|---|---|---|
| 生命周期 | 瞬态（每 turn 现算） | 持久（进树、可恢复） |
| 留痕 | 不留 | 留 |
| 适合 | 按 query 检索的相关记忆 | 固定要记住的事实/偏好 |

【推断，非 pi 明示】要"这次相关就临时带"用 context hook；要"永久记住"用 custom_message。pi 两条都给了，选哪条是记忆系统的设计决定。

---

## 三、辅助原语：session 事件 + resource loader

除了注入，pi 还有两个能被记忆系统利用的点：

- **session 事件**（Extension 篇的事件表）：`turn_end`、`message_end`、`session_shutdown` 等——记忆系统可以挂这些点做**提取时机**（一轮结束时判断"这轮有什么值得记的"）。pi 给时机，抽取逻辑应用层写。
- **resource loader / 项目文件**：`AGENTS.md` / `CLAUDE.md` 走 `<project_context>` 进 system prompt（Context 篇）——这是一种"静态记忆"（项目级固定约定）。适合放不变的长期事实，但不能动态更新。

---

## 四、一个记忆系统落到 pi 上会长什么样（推断）

【以下是我顺着原语推的设计，非 pi 源码，标清】

```
提取：extension 挂 turn_end/message_end 事件 → 判断本轮有无值得记的 → 抽成条目
存储：写到外部 store（pi 不管）——DB / 向量库 / 文件
检索：新一轮开始，按当前 query 从 store 找相关条目（pi 不管）
注入：
  - 瞬态相关记忆 → context hook（emitContext）临时插进 messages
  - 固定事实/偏好 → appendCustomMessageEntry 进树持久
```

pi 的边界很清楚：**它是"记忆能挂上来的地基"，不是记忆本身**。提取判断、存储介质、检索算法、去重策略——全在应用层。

---

## 一句话总结

pi 不内置长期记忆（无 embedding/检索/外部 store，grep 坐实）。它提供注入原语两条——context hook（瞬态、不留痕）和 custom_message 进树（持久、可恢复）——外加 session 事件（提取时机）和项目文件（静态记忆）。记忆系统的提取、存储、检索、去重要应用层自己设计，pi 只负责"让记忆能进上下文"。

---

## 待深入
- [ ] `examples/` 里有没有一个真做 memory 的 extension 可对照（本轮只确认 plan-mode 用 context hook，非记忆）。
- [ ] `custom_message` 的 `display` 参数对 TUI 渲染的影响（Session 篇提过 display=false 不渲染），与"注入但不打扰用户"的关系。
- [ ] 项目文件（AGENTS.md/CLAUDE.md）能否被 extension 动态改写后触发 system prompt 重建（Context 篇的 _rebuildSystemPrompt ③ extension 追加资源）——这可能是"半动态静态记忆"的路子。