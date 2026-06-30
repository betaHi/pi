# 05 Session 系统 · 学习笔记

> README 核心问题：对话怎么存、恢复、分支？为什么是 append-only entry 树，而不是 messages 数组？
> 主战场：`session-manager.ts`（1458 行）、`compaction/`（1408 行）、`buildSessionContext`。

---

## 题眼：为什么 session 是树，不是数组

一个判断题立住整篇：

> 如果 session 只是个线性 messages 数组（不是树），下面哪件事做不了？
> - A. 崩溃后从中间恢复
> - B. `/tree` 回到 10 轮前某条消息，从那里重新问（产生新分支），同时不丢原来那条线
> - C. 把对话存到磁盘

**答案：B。**

排除法即可确认：
- A 崩溃恢复 —— 数组也能做（存下来重新读），不需要树。
- C 存磁盘 —— 数组也能（JSON），不需要树。
- B 分支 —— **只有树能做**。

**分支的本质 = 一个节点有多个 children。** 数组每个元素只有一个 next，根本表达不了「从中间岔开两条线、还共享前面」。只有树（每个 entry 带 parentId 指父）能表达。

所以：**pi 把 session 设计成树，唯一刚需理由是支持分支（`/tree`、`/fork`）。** 崩溃恢复、磁盘存储是数组也有的附带好处，不是用树的理由。

```
线性数组:  msg1 → ... → msg5 → msg6 → ... → msg11   只有一条线，从 msg5 重开就得砍掉后面

树:        msg1 → ... → msg5 ─┬─→ msg6  → ... → msg11   原来那条
                              └─→ msg6' → ...           新分支
                              ↑ msg5 有两个 children，共享前 5 条
```

---

## 已摸到的源码骨架（待深入）

**session 文件里存的是 `SessionEntry`，9 种类型**（不只是消息）：
```
SessionMessageEntry       一条消息
ThinkingLevelChangeEntry  改了思考等级
ModelChangeEntry          切换了模型
CompactionEntry           做过一次压缩，带 summary
BranchSummaryEntry        分支摘要
CustomEntry / CustomMessageEntry
LabelEntry                给节点打标签
SessionInfoEntry
```

每个 entry 都有 `SessionEntryBase`：`id` / `parentId` / `timestamp` / `type`。
→ id + parentId = 树。

**关键函数**：
- `buildSessionContext()`（session-manager.ts:314）：把 entry 树「投影」成 LLM 能读的 context。这是 Session 篇和 Context 篇的接缝。
- `getLatestCompactionEntry()`：找最近一次压缩。
- compaction：往树里加一个 CompactionEntry（append-only，不删旧消息）。

**核心认知（待逐一验证）**：
```
磁盘:  append-only entry 树（什么都不删，9 种 entry）
        ↓ buildSessionContext 投影（走当前分支 leaf→root，遇到 compaction 用 summary 替代旧消息）
内存:  发给模型的 messages（这棵树某条分支的、压缩后的视图）
```

---

## 核心机制：buildSessionContext 怎么把树投影成 context（已读懂）

入口 `buildSessionContext(entries, leafId)`，分三步：

### 第一步：定位 leaf（你现在站在树的哪个节点）
- `leafId === null` → 返回空（在第一条之前）。
- `leafId` 有值 → `byId.get(leafId)` 取那个节点。
- `leafId` 没给 → 默认最后一条。
- **leafId 就是"当前在哪个节点"。`/tree` 切分支改的就是它。**

### 第二步：从 leaf 往 root 爬，收集 path（整个机制的心脏）
```js
let current = leaf;
while (current) {
  path.unshift(current);                       // 头插
  current = byId.get(current.parentId);        // 只跳父节点
}
```
- **只走 leaf→root 一条链**，不遍历整棵树。每个节点只有一个 parentId，所以"向上"永远是单线。
- **不需要 BFS/DFS**：BFS/DFS 是"根→叶往下、处理多个 children、要队列/栈"；这里是"叶→根往上、每节点一个父、一个 while 够了"。要遍历整棵树（如 /tree 列所有分支）才需要 BFS。

**为什么 unshift 不是 push**：爬的方向是 leaf→root（新→老），但发给模型要老→新。头插把老的不断挤到前面，自动得到正序。
例（leaf=D，A→B→C→D）：
```
unshift(D) → [D]
unshift(C) → [C,D]
unshift(B) → [B,C,D]
unshift(A) → [A,B,C,D]   ← 正序
```
（用 push 会得到 [D,C,B,A] 倒序，模型读反。等价写法：push + 最后 reverse。pi 选 unshift 一步到位。）

### 第三步：沿 path 提取设置 + 找 compaction
遍历 path（不是整棵树，是当前分支），取：最后一次 thinkingLevel、最后一次 model、有没有 compaction。
→ "用哪个模型、思考等级、有没有压缩"都是**相对当前分支**的，切分支可能不一样。

---

## 分支：切 leaf 后旧消息怎么了（已读懂）

树：`A→B→C─┬─D（原来）` / `└─C'→E'（/tree 切过去）`

站到 E'，从 E' 往上爬 `E'→C'→C→B→A`，path = `[A,B,C,C',E']`。此时 D 的三件事：

| 现象 | 原因 |
|---|---|
| 爬不到 D | 向上只走 parentId，E' 的祖先链里没 D（D 在另一个分叉） |
| D 没删 | 树 append-only，切分支只改 leafId，从不删节点（磁盘上 D 还在） |
| 模型看不到 D | 这次 path 没有 D，不进 messages（context 只是当前分支的视图） |

**"删"和"看不到"是两回事**：D 只是这次没投影进 context，磁盘的树里还在，随时 `/tree` 切回去又能投影出来。

**这就是用树不用数组的全部回报**：
- 数组：从 C 重开 = 砍掉 D = 永久丢失。
- 树：从 C 重开 = 加分支 C'→E' + 移 leaf，D 原封不动，随时能回。

对应最早的直觉题 B：「回到历史节点重开、不丢原来那条线」=
- 回到历史节点 = 移 leaf 到 C / 新建 C'
- 不丢原来那条 = D 还在树里（append-only）
- 产生新分支 = C 有了两个 children
- 重新投影 = buildSessionContext 从新 leaf 爬出新 path

### 分支拿来干嘛：pi 把它接在三处（机制有源码，场景为推测）

> 纪律提醒：slash-commands 里的命令描述只说 what，不讲 why。下面「机制」列是源码原文/行为，「场景」列是我从机制反推的合理用途——**不是 pi 文档的用例**。

**① 编辑旧消息重发（强支撑）**
- 机制：`/fork` = `slash-commands.ts:30` 原文 `"Create a new fork from a previous user message"`。编辑某条 user 消息时，`agent-session.ts:2741-2744`：`newLeafId = targetEntry.parentId`（leaf 退到那条消息的父亲）+ `editorText = 原文`（塞回编辑器）。改完一发，新旧两条共一个父，并存。
- 场景（推测）：10 轮前问"写登录函数"，想改成"用 JWT 写"——回去改这一句、重生成，原来 10 轮那条线不删。
- 成色：机制几乎只有这一个用途，场景只是套了层皮，内核是源码的。

**② 一个起点试两种走法（弱支撑）**
- 机制：`/tree` = `slash-commands.ts:32` 原文 `"Navigate session tree (switch branches)"`。能从同一节点 fork 出多条，`/tree` 切换，`buildSessionContext` 按当前 leaf 各投各的。
- 场景（推测）：A/B 对比（激进重构 vs 保守改），选好的一条继续，另一条留底。
- 成色：机制为真（能切多条），但"拿来 A/B"是我加的用途，pi 只给能力没说用法。

**③ 换线不丢上一条的结论（机制真、剧情假）**
- 机制：`branchWithSummary`（`agent-session.ts:2765`）切分支时在新分支起点放一条 `branch_summary`；投影时 `createBranchSummaryMessage`（buildSessionContext:384）把它变成一条消息发给模型。
- 场景（推测）：A 线得出"某库有坑别用"，切到 B 线时用 summary 把这结论带过去。
- 成色：summary 机制为真，"某库有坑"是我编来解释它的故事。

一句话（我的总结，非 pi 原话）：分支的价值 = **"想回到过去改一下，但不想抹掉现在"**。线性对话（改一句、后面全没）做不到，树天生支持。

---

## 压缩（compaction）：往树里加节点，不删旧消息（已读懂）

直觉先行：append-only 的树从不删节点，所以"压缩"不可能是删旧消息。代码证实——压缩 = 往树里 append 一个 `CompactionEntry`（`appendCompaction`，session-manager.ts:911），里面装 summary + `firstKeptEntryId`。旧消息一条没动，只是投影时不发给模型。

投影在 `buildSessionContext` 第三步（370 行往后）。先有个小工具把 entry 翻成 message：
```js
const appendMessage = (entry) => {
  if (entry.type === "message") {
    messages.push(entry.message);
  } else if (entry.type === "custom_message") {
    messages.push(createCustomMessage(entry.customType, entry.content, entry.display, entry.details, entry.timestamp));
  } else if (entry.type === "branch_summary" && entry.summary) {
    messages.push(createBranchSummaryMessage(entry.summary, entry.fromId, entry.timestamp));
  }
};
```
然后分两条路：

没压缩 —— 整条 path 原样投影：
```js
for (const entry of path) {
  appendMessage(entry);
}
```

有压缩 —— summary 替代被压缩的旧消息（三段）：
```js
if (compaction) {
  // ① 先放 summary（代替被压缩那段）
  messages.push(createCompactionSummaryMessage(compaction.summary, compaction.tokensBefore, compaction.timestamp));

  const compactionIdx = path.findIndex((e) => e.type === "compaction" && e.id === compaction.id);

  // ② 从 firstKeptEntryId 起，放「保留的」旧消息（压缩点前但指定保留的）
  let foundFirstKept = false;
  for (let i = 0; i < compactionIdx; i++) {
    const entry = path[i];
    if (entry.id === compaction.firstKeptEntryId) foundFirstKept = true;
    if (foundFirstKept) appendMessage(entry);
  }

  // ③ 压缩点之后的消息，全放
  for (let i = compactionIdx + 1; i < path.length; i++) {
    appendMessage(path[i]);
  }
}
```

投影结果 = `[summary] + [firstKeptEntryId 之后的旧消息] + [压缩点之后的新消息]`。

| | 树里（磁盘） | 投影后（发给模型） |
|---|---|---|
| 压缩点前、没保留的旧消息 | 还在 | 不投影，被 summary 代替 |
| CompactionEntry | 在 | 变成一条 summary message |
| firstKeptEntryId 之后的消息 | 在 | 投影（保留原文） |
| 压缩点之后的消息 | 在 | 投影 |

**`firstKeptEntryId` 的意义**：压缩不是"之前全砍"，而是留最近几条**原文**。summary 是模型概括、会丢细节，保留近处原文让模型"远处有概括、近处有原文"。

**压缩和分支是同一招**：
- 分支：切 leaf → 投影走另一条 path → 旧分支"看不到"但没删。
- 压缩：加 CompactionEntry → 投影用 summary 替代 → 旧消息"看不到"但没删。
两者都是"磁盘 append-only（只增不删）+ 投影是视图（决定模型看到什么）"。这也是 `buildSessionContext` 叫"build"而不是"read"：它按 树 + leaf + compaction 算出一个视图，不是直接读消息。

---

## 多叉树 + 建树 + 两套遍历（已读懂）

### 是多叉树吗——是，但是「隐式多叉」
- `SessionTreeNode.children: SessionTreeNode[]`（154）是数组；`getTree` 里 `parent.children.push(node)`（1132）往同一个 parent 挂多个 child；`getChildren(parentId)`（1021）收所有 `parentId===入参` 的 entry。三处坐实"一个父能有多个子"。
- 但**磁盘只存 `parentId`**（向上单指针，46）。children（向下）不持久化，是 `getTree` 读时扫一遍 entries 现算的。
- → 存储侧只有"我爸是谁"；树形（向下的 children）运行时重建。

### 建树：`_appendEntry`（858），三行
```js
private _appendEntry(entry) {
  this.fileEntries.push(entry);    // 进内存列表
  this.byId.set(entry.id, entry);  // 进 id→entry 索引
  this.leafId = entry.id;          // leaf 前移到新节点
  this._persist(entry);            // append 写 JSONL
}
```
所有 `appendMessage`/`appendCompaction`/… 长一个样：造 entry 时 `parentId: this.leafId`，再 `_appendEntry`。规则一句话：**新节点的父亲永远是当前 leaf，然后 leaf 前移到自己。**
```js
appendMessage(message) {
  const entry = {
    type: "message",
    id: generateId(this.byId),
    parentId: this.leafId,        // ← 父 = 当前 leaf
    timestamp: new Date().toISOString(),
    message,
  };
  this._appendEntry(entry);       // ← 里面把 leaf 前移到 entry.id
  return entry.id;
}
```

**多叉怎么来的**：`branch(branchFromId)`（1162）只干一件事——把 leaf 倒回老节点：
```js
branch(branchFromId) {
  if (!this.byId.has(branchFromId)) throw new Error(`Entry ${branchFromId} not found`);
  this.leafId = branchFromId;     // ← 只改指针，不删不改任何 entry
}
```
下次 append 的 `parentId` 就指向老节点，那个老节点于是有了第二个 child：
```
append A→B→C→D       C 的 child 只有 D
branch(C)            leaf 倒回 C（D 没动）
append E             E.parentId = C → C 现在有 D、E 两个 child
```
多叉不是设计了一个"分叉 API"，是 append-only（只设 parentId）+ 可回退的 leaf 指针自然长出来的。

### 两套遍历，方向相反
| 遍历 | 方法 | 方向 | 结构 | 用途 |
|---|---|---|---|---|
| 单链 | `buildSessionContext`(348)、`getBranch`(1073) | 叶→根（向上） | while + parentId | 投影当前分支、发给模型 |
| 全树 | `getTree`(1112) | 根→叶（向下） | nodeMap + stack | `/tree` 列所有分支 |

单链（投影/getBranch 同款）——只走一条祖先链，不碰别的分支，所以**不用 BFS**：
```js
let current = leaf;
while (current) {
  path.unshift(current);
  current = current.parentId ? byId.get(current.parentId) : undefined;
}
```

全树（`getTree`）——才需要处理"一个父多个子"：先建 nodeMap，扫一遍按 parentId 挂 children，再用 stack 迭代排序（注释写明"避免深树爆栈"）：
```js
// 建节点
for (const entry of entries) nodeMap.set(entry.id, { entry, children: [], ... });
// 挂 children
for (const entry of entries) {
  const node = nodeMap.get(entry.id);
  if (entry.parentId === null || entry.parentId === entry.id) roots.push(node);
  else {
    const parent = nodeMap.get(entry.parentId);
    parent ? parent.children.push(node) : roots.push(node); // 找不到父 = 孤儿，当 root
  }
}
// stack 迭代排序（DFS 式下行，不是 BFS）
const stack = [...roots];
while (stack.length > 0) {
  const node = stack.pop();
  node.children.sort((a, b) => 时间升序);
  stack.push(...node.children);
}
```

一句话：**向上爬一条链 = 当前对话（投影）；向下铺整棵树 = 所有分支（`/tree`）。** 同一棵树，两个方向，两个方法。

---

## 落到磁盘：JSONL 文件 + 攒批落盘 + 读回重建（已读懂）

### 1. 磁盘形式：一个 session = 一个 `.jsonl` 文件，一行一个 entry
文件名（`newSession`，786）：`{时间戳}_{sessionId}.jsonl`。JSONL = 每行一条独立 JSON，不是整文件一个大 JSON。`loadEntriesFromFile`（442）就是 `content.trim().split("\n")` 逐行 `JSON.parse`。

> 下面这段是**演示**：字段名（type/id/parentId/timestamp/message/summary/firstKeptEntryId）取自源码，**值（abc123/e1/时间戳）是占位编的**，别当实测。version 真实常量是 `CURRENT_SESSION_VERSION = 3`（第 27 行），不是写死数字。
```
{"type":"session","version":<CURRENT_SESSION_VERSION=3>,"id":"<sessionId>","cwd":"/...","timestamp":"..."}   ← 第一行永远是 header
{"type":"message","id":"e1","parentId":null,...,"message":{...}}
{"type":"message","id":"e2","parentId":"e1",...,"message":{...}}
{"type":"compaction","id":"e3","parentId":"e2","summary":"...","firstKeptEntryId":"e1",...}
```
- 第一行固定是 `session` header；不是 `session` 就当无效文件丢弃（`loadEntriesFromFile` 457 校验）。
- **树关系编码在每行的 `parentId` 字段里**。文件本身是平的（flat），树是读出来重建的。

### 2. 为什么 JSONL 不是一个大 JSON：为了能"追加一行"
大 JSON 数组每加一条都要读出整体、push、整体重写。JSONL 只在文件尾 `appendFileSync` 追加一行，不碰前面。
→ **append-only 的树，落盘就是 append-only 的文件——同一个选择的两面。**

### 3. 落盘时机：不是每条立刻写，等 assistant 回话才刷
`_persist`（838）有"攒一批"的门：
```js
const hasAssistant = this.fileEntries.some(e => e.type === "message" && e.message.role === "assistant");
if (!hasAssistant) { this.flushed = false; return; }   // 还没 assistant 回复 → 先不写，只标"待刷"
if (!this.flushed) {                                    // 首刷：把积攒的所有行补写
  for (const e of this.fileEntries) appendFileSync(file, JSON.stringify(e)+"\n");
  this.flushed = true;
} else {                                                // 之后：每条直接追加一行
  appendFileSync(file, JSON.stringify(entry)+"\n");
}
```
**为什么等 assistant**：一回合里 user 消息/thinking/model_change 先到，assistant 最后到。等它真回话才整批落盘——模型没答就中断（Ctrl-C/报错）不会在磁盘留半截无回复对话。
- 日常写入永远 append。`_rewriteFile`（812）整体重写只在特殊情况：版本迁移（755）、修空文件（746）、fork（1271）。

### 4. 读回成树：`setSessionFile` → `_buildIndex`，一次线性扫描
```js
// setSessionFile(735)
this.fileEntries = loadEntriesFromFile(file);  // ① 逐行 parse 成平数组
// ②（可选）migrateToCurrentVersion
this._buildIndex();                            // ③ 建索引 + 定位 leaf
```
`_buildIndex`（791）核心：
```js
for (const entry of this.fileEntries) {
  if (entry.type === "session") continue;  // 跳 header
  this.byId.set(entry.id, entry);          // 建 id→entry 索引
  this.leafId = entry.id;                  // leaf 一路前移 → 结束时 = 最后一行
}
```
**关键**：读回时**不构造 children、不拼父子指针**，只做两件事：建 `byId`、把 leaf 设成文件最后一行。树形（向上爬 parentId / 向下铺 children）是用到时才现算（buildSessionContext / getTree）。
→ "从文件变回树"准确说是：**文件 → 平数组 → byId 索引**，树是索引之上的视图，不是真建出来的树对象。

### 5. 崩溃恢复 = 正常打开，没有专门"恢复"代码
进程没了文件还在（逐行 append 落盘）。重启 `open`（1317）→ `loadEntriesFromFile` → `_buildIndex`，leaf 落最后一行，接着聊。**磁盘格式和内存重建是同一套，所以"打开"本身就是"恢复"。**

---

## `/fork`：把一条链拷成一棵全新独立的树（已读懂）

`createBranchedSession`（1207）：
```js
const path = this.getBranch(leafId);          // ① 叶→根爬出这一条链（线性）
const header = { type:"session", id:newId, ..., parentSession: previousSessionFile };  // ② 新 header 记出处
this.fileEntries = [header, ...pathWithoutLabels, ...labelEntries];   // ③ 只装这一条链
this._buildIndex();                            // ④ 重建索引
// ⑤ 有 assistant 才立刻写盘，否则等第一次回复
```

**fork 出来是不是新树：是，独立的新树。** 四条证据：
| 判断 | 证据 |
|---|---|
| 新身份 | `newSessionId = createSessionId()`(1217) + 新文件名(1220) |
| 新根 | `fileEntries = [header, ...]`(1259)，新 header 就是新树 root |
| 只带一条链 | `getBranch(leafId)`(1209) 只爬单链，原树其它分支不复制 |
| 认祖不共享 | 新 header `parentSession` 指原文件(1228)，只记出处，两文件各存各的 |

**但出生是"退化树"（直线，无分叉）**：`getBranch` 抽的是线性链，新树刚 fork 完只有主干、零分叉。
```
原树:  A→B→C─┬─D            fork E 这条 →  新树:  A'→B'→C'→E'（直线，没 D，没分叉）
              └─E'  抽这条                       内容是复制的，原树 D 不跟过来；原文件一字不动
```
**还能再长成多叉吗：能。** fork 返回完整 SessionManager，`branch()`/`leafId` 全在。新树里 `/tree` 切回 C' 重开 → C' 长第二个 child → 新树自己也分叉。

三个操作对比（记牢）：
- **`/tree`**：同树换 leaf —— 还是那棵树，站到别的分支。
- **`branch()`**：同树从老节点重开 —— 老节点加新 child，树多一个分叉。
- **`/fork`**：把一条链拷出去新建 —— 从此两棵树，各自独立生长。

---

## 待深入（下次从这里继续）
- [x] compaction —— 见「压缩」节。
- [x] append-only / `_appendEntry` —— 见「建树」节。
- [x] `/fork` —— 见「`/fork`：拷成新树」节。
- [x] 崩溃恢复 / JSONL 持久化 —— 见「落到磁盘」节。
- Session 篇源码侧到此通了，可收口成文。
