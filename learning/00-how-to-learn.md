# 00 学习方法补充

> 这份是给"想深入学习 pi 设计，并把它内化成 agent harness 工程能力"的人看的。现在学习路线按工程问题组织，这里写"怎么用"。

---

## 整体判断

当前路线的强项是：

- 01 → 02 → 06 顺序对：先建空间感（架构）→ 再串时间线（运行流）→ 再钉到代码坐标。
- 03 改成工程问题路线：provider、agent loop、context、compaction、tool、session、skills、extension、memory、自我进化、应用入口。
- 强调按问题和调用链读，而不是按文件名读：pi 是分包 monorepo，按文件读会迷失，按工程问题读才能看到设计意图。
- "三件产出"约束（一条调用链 + 3-5 个关键类型 + 一个可验证结论）：区分"读过"和"懂了"的关键。
- 04 逐文件复盘用于验证，不是文章大纲。
- 05 调试 lab 必做：静态读永远只是猜，断点验证才是真懂。

下面 5 条加料，提升学习深度。

---

## 加料一：在 02 之前插一个 step 0 "先跑起来"

哪怕只是 30 行 hello world，先让流式输出在终端跳出来。然后再读 02 ——
脑子里有"它真的在跑"的画面，链路文档才能贴上去。

**理由：** 纯静态读链路文档容易变成"看小说"。跑过一次再读，每个事件名都对应具体的视觉记忆。

推荐 step 0 内容：

```
1. mkdir pi-hello && cd pi-hello && npm init -y
2. npm install @earendil-works/pi-coding-agent
3. 照 pi.dev/docs/latest/sdk 写最小 createAgentSession + subscribe + prompt
4. 用真 ANTHROPIC_API_KEY 跑，亲眼看到:
   - text_delta 一个一个出来
   - 触发 tool 时的 tool_execution_start / end
5. 把每个事件 console.log 一遍，记住它们的字段长什么样
```

跑完这一步再读 02，吸收率翻倍。

---

## 加料二：给每一块加"反向验证题"

03 每个工程问题学完后，强迫自己回答三个问题：

1. 这一块**没有**它，pi 会怎样？
2. 这一块的设计**反过来**会怎样？
   （比如 SessionManager 如果不用 JSONL 用 SQLite 会怎样？）
3. 这一块的设计**和别的 agent 框架的对应模块**有什么不同？
   （Claude Agent SDK / LangGraph / AutoGen / Mastra）

**理由：** 理解一个设计最快的方式是理解它**为什么不是别的样子**。

### 示例问题清单（按工程问题）

**Provider 抽象：**
- 多 provider 怎么把不同的 API 形状统一成同一套 event？
- 流式 token 的"边界"在哪里定义（一个 chunk 是不是一个 token）？
- 如何处理 provider 错误重试？

**Agent loop：**
- 一个 turn 的生命周期是什么？开始/结束信号在哪？
- tool call 失败 / 超时 / 输出过大如何处理？
- queue 是 FIFO 还是有优先级？steer 是怎么插队的？

**Context 与 compaction：**
- LLM 最终看到的 context 由哪些来源合成？
- `transformContext`、`convertToLlm`、provider `Context` 的边界是什么？
- compaction 为什么是 entry，而不是直接删除历史？

**Session / memory / 自我进化：**
- session、memory、自我进化三者边界是什么？
- 哪些事实适合进入长期 memory，哪些只留在 session？
- 自我进化的候选更新如何验证和回滚？

**UI bot / CLI 应用：**
- 最终入口要归一化哪些东西：输入、工具、memory、确认、输出？
- 应该复用 SDK/RPC/Web UI，还是自己写一层？

**TUI / Web UI（可选）：**
- "差量渲染"具体差什么？state diff 还是 layout diff？
- 输入事件如何路由到组件树？

---

## 加料三：05 调试 lab 加"故意搞破坏"

每块都做一次：

- 把某个事件吃掉，看上层会怎样
- 在 tool execute 里抛错，看错误传播路径
- 中断 LLM 流式响应，看 pi 怎么回滚
- 把 session JSONL 文件改坏，看恢复行为

**理由：** 正常路径只让你知道"代码长什么样"，异常路径才让你知道"设计者考虑过什么"。

---

## 加料四：给学习过程加"输出节奏"

按这个频率强迫自己产出：

- **每块学完**：写 1 张图（手画或 ASCII 都行）解释这一块
- **每天学完**：在 fork 里改 1 行代码 + 写注释解释为什么这么改
- **全部学完**：写一篇博文级"pi 设计解读"

**理由：** 能"教给别人"才是真懂的标志。

---

## 加料五：区分"机制" vs "实现细节"

读源码时主动给每个东西分类：

**机制**（必须吃透，跨项目通用）：
- event-driven agent loop
- tool calling protocol（LLM ↔ runtime ↔ tool）
- context construction / transformation / injection
- context window、compaction、overflow recovery
- session 增量序列化和 branch
- skills / extension lifecycle
- memory extraction / retrieval / injection
- self-evolution feedback / validation / rollback
- steer 的 cancellation 语义

**实现细节**（知道在哪能查到就行）：
- TypeBox vs Zod 的选择
- 内部某个 Map 的 key 命名约定
- TUI 差量算法的具体数据结构

**理由：** 深入学不等于背书。要分清哪些值得吃透，哪些过一遍就行。

---

## 推荐节奏（14 天预算）

| 阶段 | 内容 |
|---|---|
| Day 1 | step 0 跑通 + 读 01（先看到事件流） |
| Day 2 | 读 01-architecture-map + 02-runtime-flow，建立空间和时间坐标 |
| Day 3 | 对照 06-code-evidence-map，把主链路钉到源码 |
| Day 4-6 | 吃 P0：provider、agent loop、context、compaction、tool、session |
| Day 7-8 | 吃 P0：skills、extension、memory、自我进化 |
| Day 9-10 | 做 05-debugging-labs，重点跑异常路径和 context 差异 |
| Day 11 | 写 3 张图：event flow、context flow、session/memory flow |
| Day 12-13 | build 一个最小 UI bot 或 CLI，归一化一个真实工作流 |
| Day 14 | 输出：更新文章大纲 + 总结自己的 harness 设计清单 |

---

## IDE / 工具建议

**用 VS Code 就够。** 不用换专业源码阅读工具。

### 必装插件

| 插件 | 用途 |
|---|---|
| TypeScript and JavaScript Language Features | 内置；F12 跳定义、Shift+F12 找引用 |
| Pretty TypeScript Errors | pi 类型嵌套深，原始报错难读 |
| Error Lens | 错误直接显在行尾 |
| GitLens | 任意行 hover 看历史 commit —— "这段代码为什么这么写"靠它 |

### 必练快捷键（Mac）

| 快捷键 | 操作 | 用途占比 |
|---|---|---|
| F12 | 跳到定义 | 60% |
| ⌥F12 | peek 定义（不离开当前文件） | 20% |
| ⇧F12 | 找所有引用 | 15% |
| ⌘T | workspace 里搜符号 | 找类/函数名 |
| ⌃- / ⌃⇧- | 回到上一个/下一个位置 | 跨包跳转后救命 |

### 必配 2 项

1. **打开 monorepo 根目录**，不是单个包 —— 跨包跳转才能工作
2. **右下角确认用 workspace TS 版本** —— 否则 F12 跳不准

### launch.json 模板

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Run my harness",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["tsx", "${file}"],
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**"],
      "env": { "ANTHROPIC_API_KEY": "${env:ANTHROPIC_API_KEY}" }
    }
  ]
}
```

打开任意 `.ts` 文件按 F5，自动跑。

### 学源码的最佳操作流程

```
⌘T 搜入口符号  →  F12 跳定义  →  ⌥F12 peek 内部调用
   →  ⌃- 回到上一个位置  →  ⇧F12 找谁调用它
   →  GitLens hover 看历史  →  看不懂处 bookmark + // TODO
```

### 隐藏技巧：两个 VS Code 窗口

- **窗口 A**：pi 的源码（只读）
- **窗口 B**：你的学习 harness（`pi-hello/`，写最小例子验证）

A 窗口跳定义做笔记；B 窗口跑代码验证。**左右分屏**。

---

## 别走的弯路

- ❌ 按文件名顺序读源码 —— monorepo 没人这么读
- ❌ 跳过 step 0 直接啃源码 —— 没有运行画面贴不上文档
- ❌ 让 AI 帮你解释 —— 学源码必须先自己读，读不懂再问
- ❌ 不打断点光读静态文本 —— 看 100 遍 event-driven 不如断 1 次点
- ❌ 不写笔记 —— 三天后忘光
- ❌ 一定要读完全部 —— TUI / web-ui 不是你的目标可以跳

---

## 输出物模板

每学完一块，按这个填：

```markdown
## 块名称：XXX

### 一张图
（ASCII 或手绘截图）

### 关键类型 / 函数
- TypeName1 — 干嘛的
- functionName2 — 干嘛的
- ...

### 一条调用链
A → B → C → D

### 反向验证题答案
1. 没有它会怎样：...
2. 反过来设计会怎样：...
3. 跟 X 框架比有何不同：...

### 一个可验证结论
（用断点 / 改代码 / 写 harness 证明的事实）

### 我的疑问
（留给自己回头解决的问题）
```

坚持 10 天填下来，就是一份属于你自己的 pi 解读手册。
