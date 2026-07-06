# 06 Context 构建 · 学习笔记

> README 核心问题：发给模型的输入怎么拼出来、怎么裁剪？为什么不只是 messages？
> 证据来源：`core/system-prompt.ts`、`transformContext`（= `runner.emitContext`）、`core/messages.ts` 的 `convertToLlm`、`before_provider_request`（provider payload hook）。
> 一句话：Session 篇的 `buildSessionContext` 只产出 messages；agent loop 先拿 `{ systemPrompt, messages, tools }` 三件套，再过 messages 改写、LLM 消息翻译，最后 provider 还能在发送前改 payload。

---

## 题眼：发给模型的不只是 messages

`createContextSnapshot()`（agent.ts）打包的是三件套：
```ts
AgentContext = {
  systemPrompt: string,        // 系统提示（拼出来的）
  messages: AgentMessage[],    // 对话（Session 篇投影来的）
  tools: Tool[],               // 工具定义（Tool 篇的 registry）
}
```
messages 只是其中一块。Context 构建 = 把"对话"补齐成"完整工作环境"。

---

## 一、system-prompt.ts：怎么"拼"出 systemPrompt（已读懂）

`buildSystemPrompt(options)`（system-prompt.ts）。有两个分支：`customPrompt` 给了走 custom 分支，否则走默认分支。**默认分支**拼的块，顺序固定：

```
You are an expert coding assistant operating inside pi…   ← 角色定义（硬编码开头）
Available tools:                                          ← ① 工具清单
  - read: <snippet>   - bash: <snippet>
Guidelines:                                               ← ② 准则（动态）
  - <按实际工具生成的条目>
  - Be concise / Show file paths（总是加）
Pi documentation (read only when…):                      ← ③ pi 文档路径
  + appendSystemPrompt                                   ← ④ 追加文本
  + <project_context><project_instructions path=…>      ← ⑤ AGENTS.md / CLAUDE.md 这类项目文件
  + formatSkillsForPrompt(skills)                        ← ⑥ skills 清单（仅当有 read 工具）
  + Current date / Current working directory             ← ⑦ 日期、cwd（每次 new Date 动态）
```

两个"按条件拼"值得记：
- **Available tools 只列"有 snippet 的"**：`visibleTools = tools.filter(name => !!toolSnippets?.[name])`。给了工具但没给一行 snippet，就不进 system prompt 清单（≠ 不能用，只是没在提示里写出来）。
- **Guidelines 随工具动态变**：有 bash 没 grep/find/ls → 加"用 bash 做文件操作"；有 grep/find/ls → 加"优先用它们，比 bash 快、尊重 .gitignore"。→ **system prompt 内容随你开了哪些工具而变。**

custom 分支更短：customPrompt 正文 + appendSystemPrompt + project_context + skills（同样需 read）+ 日期/cwd。

project_context 包法（两个分支一样）：
```
<project_context>
<project_instructions path="<文件路径>">
<文件内容>
</project_instructions>
</project_context>
```
→ **这就是 AGENTS.md / CLAUDE.md 这类 project context 文件进入模型的方式**：被包进 `<project_instructions>` 标签拼进 system prompt。memory 不在这里自动等同；除非某个资源加载/extension hook 明确把 memory 作为 context file 或 message 注入。

skills 只在有 `read` 工具时拼（`customPromptHasRead` / `tools.includes("read")`）——因为 skill 正文要靠 read 工具去读（见 Session 篇前学的 `formatSkillsForPrompt`：只注入 name+description+location，正文按需 read）。

---

## 二、transformContext：extension 改写 context 的钩子（已读懂，已纠错）

> `transformContext` 不是压缩，是 extension 的 context hook。

messages 投影出来后、转成 LLM messages 前过 `transformContext`。coding-agent 里它接到 `runner.emitContext(messages)`（sdk.ts → runner.ts）：

```js
// ExtensionRunner.emitContext（runner.ts）
let currentMessages = structuredClone(messages);   // 深拷贝，不动 Session 树原始 entry
for (const ext of extensions) {
  const handlers = ext.handlers.get("context");     // 找注册了 "context" 的 extension
  for (const handler of handlers) {
    const result = await handler({ type:"context", messages: currentMessages }, ctx);
    if (result?.messages) currentMessages = result.messages;  // 用返回值替换，链式
  }
}
return currentMessages;
```

本质：**给 extension 一个在 AgentMessage 层改写整组 messages 的机会。**
- **链式**：多个 extension 依次处理，前一个输出 = 后一个输入。
- **深拷贝** `structuredClone`：改的是副本，不污染 Session 树。
- **容错**：handler 抛错被 catch + emitError，不打断流水线（其它 extension 照跑）。
- **这是 README 说的"怎么裁剪"的入口**——但裁剪/增删由 extension 决定；**pi core 只提供 hook，自己不裁**。

【我的推断，非此处源码明写】memory 系统（README 08）如果做成 extension，很可能挂在这：注册 `context` handler，把"相关记忆"插进 messages。源码事实只到这里：`emitContext` 是 `AgentMessage[] → AgentMessage[]` 的 context hook；最终 provider 请求前还可能经过 `before_agent_start`（本轮 prompt/messages）和 `before_provider_request`（最终 payload）。

---

## 三、convertToLlm：AgentMessage → LLM Message（已读懂）

核心翻译步骤。pi 内部消息角色多，LLM API 只认几种，这里翻译（messages.ts `convertToLlm`）：

| pi 内部 role | → LLM | 处理 |
|---|---|---|
| user / assistant / toolResult | 原样 | 直接返回 |
| `bashExecution` | user | 转文本；`!!` 前缀（`excludeFromContext`）→ **丢弃，不发模型** |
| `custom`（extension 注入） | user | 内容转 user |
| `branchSummary` | user | 包 `BRANCH_SUMMARY_PREFIX/SUFFIX` |
| `compactionSummary` | user | 包 `COMPACTION_SUMMARY_PREFIX/SUFFIX` |

**关键认知**：Session 篇产出的 `compactionSummary`/`branchSummary` 是抽象 entry，**到这一步才翻成模型能读的 user 文本**（加"这是之前对话的摘要…"前缀让模型知道这是摘要）。Session 篇埋的种子，Context 篇这里发芽。

边界补充：coding-agent 实际传给 agent-core 的不是裸 `convertToLlm`，而是 `convertToLlmWithBlockImages`：先调用 `convertToLlm(messages)`，再按 `blockImages` 设置过滤 image content。再往后，provider 还会把 LLM context 组成各家 API payload。

→ 这也解释了"页面看到的消息"和"发给模型的消息"为什么可能不一样：`excludeFromContext` 的 bash、还没被 convertToLlm 处理的内部角色，渲染层和 LLM 层走的是不同投影。（渲染丢消息的问题，根源八成在"渲染用的消息集合 ≠ convertToLlm 后的集合"，待查 UI 层。）

---

## 四、压缩压的是 messages，不是整个 context（已核）

常见误读："压缩 context"听起来像压整个三件套。实际 compaction **只改 session/messages 这条线，不改 systemPrompt/tools 定义**。两处源码钉死：

- **估算入口只吃 messages**：`estimateContextTokens(messages: AgentMessage[])`（compaction.ts）入参就是 messages；`tokensBefore = estimateContextTokens(buildSessionContext(pathEntries).messages).tokens`。但如果 messages 里有历史 assistant usage，估算会沿用那次 provider usage 的总量口径；入口只吃 messages，不等于结果一定是纯 messages token。
- **动手只动 session/messages**：`prepareCompaction` 产出 `messagesToSummarize: AgentMessage[]` / `turnPrefixMessages: AgentMessage[]`，压缩代码里没有任何改 systemPrompt/tools 的语句。

**"要不要压缩"的度量，主路不是 estimateContextTokens**（agent-session.ts:1800-1818）。判断压缩时分两路：
- **正常成功回复**：`contextTokens = calculateContextTokens(assistantMessage.usage)` —— 用真实 usage。而 `calculateContextTokens = usage.totalTokens || input+output+cacheRead+cacheWrite`（compaction.ts:136）。其中 `usage.input` 直接取 API 的 `input_tokens`（anthropic.ts:510），`cacheRead/cacheWrite` 取 API 的缓存 input token 计数；按 API 请求定义，这组 input-side usage 覆盖整次请求输入 = system prompt + tools + messages 全部。所以主路的输入侧 usage 是**含 system+tools 的全量输入口径**，`contextTokens` 则是 usage 总量口径，不是纯 messages 估算。
- **本轮出错/无 usage**：才退回 `estimateContextTokens(messages)` 作兜底。它会用历史 usage 锚点 + trailing messages 估算；完全没有历史 usage 时，才全部 chars/4 估。

「压缩只看 messages」对 `estimateContextTokens` 的入口成立、对"实际触发判断"不成立——正常路看的是 provider 返回的真实 usage，其中输入侧 usage 已覆盖 system+tools+messages。**压缩动手的对象**仍只有 messages（systemPrompt/tools 不被改）。度量看真实请求用量、改动只碰 messages，是两回事。

再精确一点：压的不是整个 messages，是从 session branch entries 找 cut point，把**靠前的旧前缀段**压成 summary；`firstKeptEntryId` 之后的 kept entries 仍由 `buildSessionContext` 投影回来。split turn 时，还会把同一 turn 的前缀单独总结进 summary。

【我的解释，非源码明写】为什么只压 messages：systemPrompt（身份/规则/AGENTS.md / CLAUDE.md）和 tools（工具定义）**必须每轮常驻、且大小基本恒定**，压了模型就忘了自己是谁 / 有哪些工具；messages 是**唯一随对话增长**的部分，token 膨胀几乎全是它。所以压缩精准打击"唯一会膨胀的那块"。

---

## 五、system prompt 何时重建（`_rebuildSystemPrompt`，已读透）

`buildSystemPrompt` 是"怎么拼"；这一环是"**何时重拼**"。

**两个字段，别混**：

| 字段 | 是什么 | 谁改 |
|---|---|---|
| `_baseSystemPrompt` | **基线**，重建后存这（默认 `""`） | `_rebuildSystemPrompt` 的结果 |
| `agent.state.systemPrompt` | **这轮实际发给模型的** | 通常=基线；每轮可被 extension 临时覆盖 |

**重建基线 `_rebuildSystemPrompt` 的三个触发**（grep 全）：

| 触发 | 位置 / 入口 | 说明 |
|---|---|---|
| ① 开局初始构建 | 初始化调 `setActiveToolsByName(初始工具集)` → 内部触发 | **开局 prompt 搭在"设初始工具集"上，不是单独初始化** |
| ② 工具集变化 | `setActiveToolsByName`（注释：rebuild to reflect new tool set，下一轮生效） | Available tools/Guidelines 按工具生成，工具变要重拼 |
| ③ extension 追加资源 | `extendResourcesFromExtensions(reason)`（"startup"/"reload"） | ext 带来新 skills/prompts/themes，ResourceLoader 更新后重建 |

**重建时做什么**（`_rebuildSystemPrompt`）：从 `resourceLoader` 当前已加载的资源重新组装 `_baseSystemPromptOptions`，再 `buildSystemPrompt`。这不是每次都重新读磁盘；真正重新加载发生在 `resourceLoader.reload()` 或 `extendResources()`：
```
skills       = resourceLoader.getSkills().skills
contextFiles = resourceLoader.getAgentsFiles().agentsFiles   // AGENTS.md / CLAUDE.md
customPrompt = resourceLoader.getSystemPrompt()
appendPrompt = resourceLoader.getAppendSystemPrompt()
+ 按 validToolNames 组装 toolSnippets / promptGuidelines
→ 存 _baseSystemPromptOptions → buildSystemPrompt(...)
```
（options 存下来是给第④处用的。）

**④ 每轮临时覆盖 —— `emitBeforeAgentStart`（1071）**：
每次发 prompt 前调 `emitBeforeAgentStart(text, images, _baseSystemPrompt, _baseSystemPromptOptions)`：
```ts
if (result?.systemPrompt) agent.state.systemPrompt = result.systemPrompt; // ext 改了→用改后
else                      agent.state.systemPrompt = _baseSystemPrompt;    // 没改→还原基线
```
→ **这就是为什么要两个字段**：extension 能"这一轮"临时改 prompt，但不污染基线；下一轮没改就 else 还原成 `_baseSystemPrompt`。基线=稳定的底，state=这轮实际发的。

**完整图景（四处会影响 system prompt）**：
```
① 开局   setActiveToolsByName(初始工具)  → _rebuildSystemPrompt → 建基线
② 工具变 setActiveToolsByName(新工具)    → _rebuildSystemPrompt → 重建基线
③ ext资源 extendResourcesFromExtensions → _rebuildSystemPrompt → 重建基线
④ 每轮   emitBeforeAgentStart → ext 可临时改 state.systemPrompt，没改则还原基线
```
①②③ 写 `_baseSystemPrompt` + `state`；④ 只动 `state`，基线不碰。

---

## 六、context 用量 ↔ 压缩接缝（`estimateContextTokens`，已核）

`estimateContextTokens` 的算法：真实用量打底 + 增量估算（聪明处）：

- **有历史请求**：`tokens = usageTokens + trailingTokens` = 上次 assistant 回复时 API 报的**真实 token**（`usageTokens`）+ 那之后新增消息的**估算**（`trailingTokens`，chars/4）。
- **第一轮无历史**：全部 chars/4 估。

为什么这么设计：chars/4 是粗估、不准；但成功的 assistant 回复通常带 provider `usage`。以**最近一次真实用量为锚点**，只对"那之后新增的几条"用粗估——**误差被限制在最近几条内**，不让整段历史用不准的估算累积。

接缝（Context ↔ Compaction 是一根链的两端）：
```
正常 assistant usage → calculateContextTokens(usage)   // input-side usage = system+tools+messages 全量；contextTokens 取 usage 总量
错误/无本轮 usage 等场景 → estimateContextTokens(messages) // 历史 usage 锚点 + trailing 估算；无 usage 才全量 chars/4
             → shouldCompact: contextTokens > contextWindow - reserveTokens ? 触发
             → compaction 压 session/messages 旧前缀
```
context 负责"拼"，usage / estimateContextTokens 负责"量"，compaction 负责"到量了就砍"。注意"量"的口径：正常路的输入侧 usage 是**含 system+tools 的全量输入**，正常路的 `contextTokens` 是 usage 总量；`estimateContextTokens` 的函数入口只吃 messages，但有历史 usage 时也会继承 usage 的全量口径。

**防重复触发**（agent-session.ts:1764-1770 + 1804-1814）：走估算兜底路时，会检查"锚点 usage 是不是压缩之前的旧值"——若 `usageMsg.timestamp <= compactionEntry.timestamp`（即这个 usage 反映的是压缩前那个更大的 context），`return false` 不触发。外层还先跳过 `assistantMessage.timestamp <= compactionEntry.timestamp` 的旧 assistant。两层一起防止"一次压缩刚结束，又被压缩前的旧 usage / 旧 assistant 误判、立刻再压一次"。

---

## 完整流水线（一条调用链）

```
buildSessionContext()              // Session 篇：树 → messages
        ↓
createContextSnapshot()            // 打包 { systemPrompt, messages, tools }
        ↓  systemPrompt 来自 buildSystemPrompt（角色/工具/准则/AGENTS.md/CLAUDE.md/skills/日期）
        ↓  tools 来自 registry（Tool 篇）
runAgentLoop(...)
        ↓
transformContext(messages)         // 加工①：extension 改写（emitContext，memory 注入/裁剪入口）
        ↓
convertToLlm(messages)             // 加工②：AgentMessage → LLM Message（摘要翻成文本）
        ↓
LLM Context { systemPrompt, messages, tools }
        ↓
provider 组装各家 API payload
        ↓
before_provider_request(payload)   // 加工③：extension 可 inspect/replace 最终 payload
        ↓
provider 发出请求（03 篇）
```

为什么不只是 messages：模型要知道"我是谁(systemPrompt)、项目规矩(AGENTS.md / CLAUDE.md)、能用啥工具(tools)、有啥技能(skills)"；对话里的特殊消息(bash 输出/压缩摘要)还要翻成它能读的格式(convertToLlm)。messages 只是一块；provider payload 是更靠后的发送层。

