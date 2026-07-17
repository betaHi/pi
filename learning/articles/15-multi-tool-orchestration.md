# Pi 系列 15｜多工具编排：一个 turn 里多个 tool 怎么跑

本系列其他文章：

placeholder

> 本文主要参考 `packages/agent/src/agent-loop.ts` 和 `packages/agent/src/types.ts`。

工具系统那篇讲的是一个 toolCall 的一生：怎么准备、执行、出错、把结果回传给模型。但模型一个 turn 里，可以一次返回多个 toolCall。这一批工具怎么调度——串行还是并行、并行到什么程度、结果顺序乱不乱——是那篇没展开的部分。这篇补上。

一条 assistant 消息里的多个 toolCall，是这样被取出来的：

```ts
const toolCalls = message.content.filter((c) => c.type === "toolCall");
if (toolCalls.length > 0) {
  // 交给 executeToolCalls 处理这一批
}
```

编排的核心就在 `executeToolCalls` 这一组函数里。

## 一、先决定整批串行还是并行

`executeToolCalls` 拿到一批 toolCall，第一件事是决定整批怎么跑：

```ts
const hasSequentialToolCall = toolCalls.some(
  (tc) => 找到这个 tool，它的 executionMode === "sequential",
);
if (config.toolExecution === "sequential" || hasSequentialToolCall) {
  执行串行版本;
} else {
  执行并行版本;   // 默认走这里
}
```

两个条件，任意一个成立就整批串行：

- 全局配置 `toolExecution` 设成了 `"sequential"`（默认是 `"parallel"`）。
- 这一批里，任何一个工具自己声明了 `executionMode: "sequential"`。

第二个条件值得注意：**单个工具能影响整批**。哪怕其它工具都能并行，只要有一个工具标了 sequential，整批就退化成串行。一个有副作用、不适合和别人并发的工具，会让这一轮所有工具都排队执行。

`executionMode` 是工具可以选择声明的字段。内置工具当前没有声明 sequential；示例 extension `tic-tac-toe.ts` 给自己的工具标了 `executionMode: "sequential"`，用来避免多个工具调用同时改棋盘状态。

<!-- 图1：整批串/并的判断
生图 prompt：
一张横版判断流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左边画一叠「一批 toolCall」用浅灰蓝 #E6EDF2。中间一个菱形判断框「toolExecution === sequential ？ 或 任一工具 executionMode === sequential ？」。
判断框向下两条出口：一条「是 → 整批串行」用珊瑚色 #E76F51；一条「否 → 并行（默认）」用蓝绿色 #2A9D8F。
在「任一工具 sequential」旁画一个小高亮标注：三个工具方块里有一个标红 sequential，箭头指向"整批串行"，标「单个工具能让整批退化成串行」。
底部小字：「默认并行；全局配 sequential、或任一工具声明 sequential，整批串行。」
建议文件名：./pi_15_1.png
-->

## 二、并行到底并在哪一步

这里容易误解的一点是："并行执行多个工具"听起来像是所有工具从一开始就同时执行，源码里的并行版本其实分成两个阶段：

```ts
// 阶段一：for 循环里逐个准备 —— 这一步是串行的
for (const toolCall of toolCalls) {
  发出 tool_execution_start 事件;
  const preparation = await prepareToolCall(...);   // await，一个一个来
  // 把执行部分包成一个函数，先存起来，还不跑
  finalizedCalls.push(async () => {
    const executed = await executePreparedToolCall(preparation, ...);
    return finalize(executed);
  });
}

// 阶段二：所有执行函数一次性并行跑
const results = await Promise.all(finalizedCalls.map((fn) => fn()));
```

把两个阶段分开看：

- **准备阶段是串行的**。`for` 循环里逐个 `await prepareToolCall`。准备这一步会找工具、运行 `prepareArguments`、校验参数，再调用 `beforeToolCall`。所以参数准备和权限检查是一个接一个过的。
- **执行阶段才是并行的**。准备好的工具，各自的执行部分被包成函数存进数组，最后用 `Promise.all` 一次性全跑。真正耗时的动作——跑 bash、读文件、发请求——在这一步并发。

准备阶段还有一个 `immediate` 分支。工具不存在、参数校验失败、`beforeToolCall` 返回 block，都会直接变成错误 tool result。这类结果不会进入执行阶段，但仍会参与后面的排序和 terminate 判断。

执行之后还有一个对称的收尾步骤。准备阶段有 `beforeToolCall`（执行前，可以拦截或改参数），执行完成后则有 `afterToolCall`（执行后，可以改结果——替换内容、改 `isError`、改 `terminate`）。两个 hook 一前一后，正好对应扩展系统里的 `tool_call` 和 `tool_result` 两个事件。`afterToolCall` 自己也包在 `try/catch` 里，它出错也会变成一个错误结果，不会中断整批。

为什么这样拆（这一段是笔者的理解，源码没有直接解释）：拦截、参数修正和校验放在串行阶段，比较容易保持决策顺序；真正花时间的是执行，把执行并行就能减少等待。只看"并行执行工具"这句话，不容易看出准备和执行其实是分开的。

<!-- 图2：准备串行、执行并行
生图 prompt：
一张横版两阶段图，从左到右，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左半「阶段一：准备（串行）」用浅灰蓝 #E6EDF2 背景：三个工具的 prepare 方块竖排、用一条从上到下的箭头串起来，标「prepareToolCall 逐个 await」，其中每个 prepare 方块里标一行小字「含 beforeToolCall 权限拦截」用珊瑚色 #E76F51。
右半「阶段二：执行（并行）」用蓝绿色 #2A9D8F 背景：三个 execute 方块横向并排、用一个大括号 Promise.all 同时括住，标「一次性并发：bash / read / 请求」。
中间用一条竖线分隔两阶段。
底部小字：「权限检查一个一个过，保证拦截不竞态；真正的执行才并行，省等待时间。」
建议文件名：./pi_15_2.png
-->

## 三、事件顺序和结果顺序

工具并行执行时，谁先跑完是不确定的。要分清两个顺序：执行事件的顺序，以及回传给模型的 toolResult 消息顺序。

先看 toolResult 消息。回传给模型的结果顺序是确定的：

```ts
const results = await Promise.all(
  finalizedCalls.map((fn) => fn()),   // map 保留数组原来的顺序
);
```

`Promise.all` 配合 `map`，保留了 toolCall 的原始顺序。也就是说，哪个工具先执行完不重要，回传给模型的 toolResult 顺序，始终和模型当初返回的 toolCall 顺序一致。模型看到的结果次序是稳定的，不受执行快慢影响。

事件顺序更细：

- `tool_execution_start` 在 prepare 阶段逐个发，顺序跟原始 toolCall 一致。
- `tool_execution_update` 来自工具执行过程，并行时可能交错。
- `tool_execution_end` 在并行分支里由每个执行任务自己发，谁先完成谁先发。
- `message_start/message_end` 的 toolResult 消息在最终循环里发，顺序回到原始 toolCall 顺序。

这一点对 UI 和模型都重要：UI 可以用执行事件显示实时进度；模型收到的 toolResult 消息仍保持稳定对应关系。

<!-- 图3：执行事件顺序和 toolResult 消息顺序
生图 prompt：
一张横版双轨时间线图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
上轨标题「执行事件（给 UI 看）」：tool_execution_start 按 A→B→C 顺序出现；tool_execution_update 画成交错小波纹；tool_execution_end 按完成顺序例如 B→A→C 出现。
下轨标题「toolResult 消息（给模型看）」：message_start/message_end 按原始 toolCall 顺序 A→B→C 出现。
中间标注「Promise.all 执行可交错；orderedFinalizedCalls 回传保持原始顺序」。
底部小字：「实时事件可以反映完成先后，模型看到的结果仍按请求顺序对应。」
建议文件名：./pi_15_3.png
-->

## 四、并行没有并发上限

值得说明的是，core 这一层当前没有显式并发池、信号量或分批逻辑。`Promise.all` 会把这一批已经 prepare 完的执行同时启动。模型一个 turn 返回十个可执行 toolCall，就会尝试同时执行十个。

这里要分清层次。上面说的是 pi core 的工具编排。subagent 示例扩展自己设了最多 8 个任务、4 个并发的限制，那是扩展层为子进程做的策略。

还有一种“多”也要分开看：可用工具很多。可用工具多，不等于一个 turn 里一定会调用很多工具。前者主要影响模型选工具的难度和上下文成本；后者才进入 `executeToolCalls` 的调度逻辑。pi core 不会自动给大量工具分组、按任务裁剪工具列表，或者限制模型一次能发几个 toolCall。这些通常要在 app 或 extension 层处理：只暴露当前场景需要的工具，给危险工具加权限检查，必要时做队列、限流或最大任务数。

## 五、toolResult 能保证什么

toolResult 不是模型直接生成的，而是工具 `execute` 返回的结果，或者运行时把错误转成的错误结果。pi core 能保证的是这条链路：工具名会被查找，参数会经过准备和校验，执行异常会变成 `isError: true` 的结果，结果会带着 `toolCallId`、`toolName`、`content`、`details` 回传，并且顺序和原始 toolCall 对上。

但这不等于 pi core 能保证结果内容在语义上一定真实。工具实现可能有 bug，外部命令或 API 可能返回旧数据，`afterToolCall` 也可以改写结果。core 保证的是“这是这次工具调用产出的结果”，不是“这个结果一定是事实”。如果上层需要更强的可靠性，就要靠工具自己返回结构化证据、清楚标记错误和来源，必要时让另一个工具或检查步骤复核，而不是让模型补写工具没有提供的信息。

## 六、什么时候整批结束

一批工具跑完，loop 要不要停下，由一个条件决定：

```ts
function shouldTerminateToolBatch(finalizedCalls) {
  return 这批结果全都标了 terminate === true;
}
```

只有当所有 toolResult 都标了 `terminate` 时，loop 才终止。只要有一个工具的结果没标终止，loop 就继续。

需要区分的是，单个工具执行出错，会被捕获成一个错误结果回传给模型。一批工具里某个失败了，其它照常执行、结果照常回传。loop 是否终止看 terminate 标记。

这里有一个容易忽略的机制问题。并行用的是 `Promise.all`，而 `Promise.all` 只要有一个 promise 抛出（reject），整个就会失败。如果工具执行抛出的异常直接进入 `Promise.all`，这一批就会整体失败。但这里没有这样做，原因在执行那一步的写法：

```ts
try {
  const result = await prepared.tool.execute(...);
  return { result, isError: false };
} catch (error) {
  return { result: createErrorToolResult(...), isError: true };  // 异常在这里就被接住
}
```

工具执行被包在 `try/catch` 里，异常在**进入 `Promise.all` 之前**就被捕获、转成一个"错误结果"（`isError: true`）。也就是说，交给 `Promise.all` 的每个任务都会正常完成（resolve），没有一个会 reject。一个工具执行失败，只是它自己的结果变成错误结果，其它工具不受影响。所以"出错不影响其它"不是偶然，而是靠"异常在 catch 里转成结果"这个写法保证的。

## 完整链路

把这批工具的处理串起来：

```text
assistant 消息里有多个 toolCall
  -> executeToolCalls：判断整批串行还是并行
     （toolExecution 为 sequential，或任一工具声明 sequential -> 串行）
  -> 并行分支：
      阶段一：for 循环逐个 prepareToolCall（找工具/参数校验/beforeToolCall）——串行
      immediate 分支：工具不存在/参数错误/被 block，直接生成错误结果
      阶段二：Promise.all 跑所有执行——并行，core 当前无显式并发限制
    -> execution update/end 事件可能按完成时机交错
    -> toolResult 消息按模型原始 toolCall 顺序逐个回传
    -> toolResult 是工具执行产物，但语义可靠性取决于工具和上层校验
    -> shouldTerminateToolBatch：所有结果 terminate 才终止 loop
```

回到开头的问题——一个 turn 里多个工具怎么跑。答案由几条规则叠在一起：默认并行，但任一工具可以让整批变串行；并行发生在执行阶段，准备阶段串行；core 当前没有显式并发限制；执行事件可能交错，但 toolResult 消息按原始顺序回传；一批里出错不影响其它，是否停下看终止标记。工具很多和结果可信度这两件事，更多由上层处理：core 负责调度和传递，工具实现与 app/extension 层负责控制规模、权限和结果证据。

这个设计可以概括为：能并行的地方并行以减少等待，涉及决策和顺序的地方保持串行或有序。准备串行、执行并行，是其中比较有代表性的一处。

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。文中"为什么准备串行、执行并行"部分是顺着机制的判断，非源码明写。*
