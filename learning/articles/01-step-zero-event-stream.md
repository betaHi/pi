# Pi 系列 01｜用最小例子看 agent runtime 的事件流

> 系列开篇 · Step 0 · 跑通 pi SDK，把事件流打出来

## 写在前面

Pi 是一个开源的 LLM agent harness，分层清晰，核心抽象齐全（turn、事件流、provider、工具、session、extension），可通过 SDK 与 extension 定制。

这个系列关心的不是“怎么调一次 LLM API”，而是一个 agent harness 如何把模型、工具、状态和扩展组织成可运行、可观察、可恢复的工程系统。N 行 agent demo 能解释 `模型 -> 工具 -> 模型`，但解释不了 provider 差异、工具事件、turn 边界、session 持久化和扩展 hook。

第一篇从最朴素的起点开始：跑通最小例子，把所有事件打印出来。先看到事件流，再回头读源码，才知道哪些抽象是骨架，哪些只是实现细节。

> 工具说明：本系列读 pi 源码用 VS Code（TS 跳转开箱即用）+ `rg` 定位 emit 点，验事件用 `node hello.mjs > out.log 2>&1` 后配合 `grep` / `jq` 反查。

## 先说结论

跑一个最小 pi SDK 例子，把所有事件打到日志里，能直接观察到 pi 的 agent loop 形态：

```
while (模型还在调工具) { 调模型 → 跑工具 }
```

pi 把这条循环里的每一步暴露成带类型的事件，订阅事件就能看到 agent 完整生命周期。

读完这一篇你会看到：

- pi 事件流的事件类型、字段嵌套、turn 边界
- 工具参数 JSON 以流式 delta 形式逐段到达
- 一个具体 prompt 触发的 turn 序列
- 复现以上观察的最小步骤

## 复现步骤

### 1. 准备项目

```bash
mkdir pi-hello && cd pi-hello
npm init -y
npm install @earendil-works/pi-coding-agent
```

### 2. 准备 API key

设置环境变量（任选一种受 pi 支持的 provider，下文用 Anthropic 举例）：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
```

pi 支持的 provider 与对应环境变量见官方 `providers.md`。

### 3. 写入 hello.mjs

```js
// hello.mjs
import {
  AuthStorage, createAgentSession, DefaultResourceLoader,
  getAgentDir, ModelRegistry, SessionManager,
} from "@earendil-works/pi-coding-agent";

const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);
const resourceLoader = new DefaultResourceLoader({
  cwd: process.cwd(),
  agentDir: getAgentDir(),
});
await resourceLoader.reload();

const model = modelRegistry.find("anthropic", "claude-opus-4-5");
if (!model) throw new Error("Model not found");

const { session } = await createAgentSession({
  model, resourceLoader, authStorage, modelRegistry,
  sessionManager: SessionManager.inMemory(),
});

try {
  session.subscribe((event) => {
    if (event.type === "message_update" &&
        event.assistantMessageEvent.type === "text_delta") {
      process.stdout.write(event.assistantMessageEvent.delta);
      return;
    }
    console.log("\n[EVENT]", JSON.stringify(event, null, 2));
  });

  await session.prompt(
    "List files in the current directory, then tell me what this project is."
  );
} finally {
  session.dispose();
}
```

### 4. 跑起来，把输出导到文件

```bash
node hello.mjs > out.log 2>&1
```

本次复现得到的 `out.log` 是 3039 行。下面的观察都来自这份日志。模型选择、prompt 内容、本地目录差异都会影响具体的 turn 数量与事件细节。

## 一张图：事件流骨架

本次复现的实际决策路径：先 `bash ls -la`，然后读 `package.json`，再读 `hello.mjs`，最后用文本回答。对应到事件流：

![pi agent runtime 事件流骨架](./article1.png)

注意：图里的消息数量来自这次空会话复现。一般情况下，`agent_end.messages` 表示本次 run 新增的消息；整段会话历史还要结合已有 session。

## 观察

### 1. agent loop 的形态

伪代码：

```
while (stopReason === "toolUse") {
  runTools();
  callModel();
}
```

模型每次响应都带 `stopReason`。`toolUse` 表示模型还要继续用工具，runtime 把工具执行完，把结果塞回历史，再调一次模型。`stop` 表示模型结束响应，循环退出，emit `agent_end`。

注意这只是循环的"骨架判据"。源码实际不是直接读 `stopReason`，而是先看 assistant message 里有没有 tool calls，再看工具批次是否要求 terminate。真实的 agent loop 还受三件事影响：工具批次的 `shouldTerminateToolBatch`（一组工具跑完后能否直接收尾）、外层的 steering / follow-up 队列（一次会话能否在 stop 之后再起一轮）、用户挂的 `shouldStopAfterTurn` 钩子（每个 turn 末尾给用户一次叫停机会）。

### 2. turn 的作用

一个 turn 由模型一次响应和它触发的工具结果构成。pi 把这段打包成一个边界，外面的代码就有地方挂钩子：

- 每个 turn 跑完，可以决定要不要压缩历史、要不要中断、tool budget 还剩多少
- 想做断点续跑，turn_end 是天然的 checkpoint 点（pi 的 SessionManager 当前其实是在 message_end 时 append，按 turn 切片是设计直觉，具体实现留到 session 那篇验证）
- UI 可以利用 turn 边界把"模型响应 + 工具结果"组织成一组，用户看到的就不是一长串裸事件

类比 git commit：你不会在一行代码改到一半的时候 commit，得改完一个完整的小步骤再提交。turn 就是 agent loop 里的"一个完整的小步骤"——模型说话 + 工具回话，齐了才算一轮。

## 反过来想：没有这些抽象会怎样

理解一个设计比较好的办法是想象它不存在。

**没有 turn 边界？** 没法做 per-turn 的钩子——自动压缩历史、tool budget、abort、UI 折叠全都没地方挂；持久化也没有天然的快照点，崩了不知道从哪接。

**工具参数 JSON 不流式（缓冲完整 JSON 才发）？** 实现更简单，但 UI 看不到"模型正在敲哪条命令"的实时反馈。pi 选流式是 UX 决策。

**`agent_end` 不带 messages 快照？** 上层就只能从一条条 `turn_end` 里自己拼本次 run 的结果。现在 agent_end 直接给本次 run 产生的新消息（在这个空会话的一次性例子里，它看起来就是完整历史），本次增量归档和跨进程回放可以吃这一个事件；整段会话归档还要结合已有历史或 SessionManager。

`turn_end` 与 `agent_end` 大致对应"per-turn 边界"与"本次 run 终态"两层——前者适合作为崩溃恢复的天然切点，后者给出本次 run 的完整增量。这种 per-step 边界 + run 终态的分工在长程任务系统里很常见，LangGraph 的 checkpoint 是 per-step，Temporal 的 history 是 per-activity，都是同样的思路。

好了，开篇结束。


