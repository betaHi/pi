# 实战复盘：给 crbuddy 加第一个领域工具 gerrit_get_cl

> commit e3e8180（feat(chat): gerrit_get_cl custom tool）。
> 把 04 篇学的「工具怎么进系统」用在真实项目上的一次验证。

## 背景

crbuddy 已从 `new Agent` 迁到 `createAgentSession`。这个 commit 是迁移后的第一个回报：给 chat agent 加一个真工具，让它能真去抓 Chromium CL，而不是回「我打不开链接」。

## 改了 5 个文件（git show --stat）

| 文件 | 改动 | 对应 04 篇哪个点 |
|---|---|---|
| `tools.ts`（+81，新增） | 写 `gerrit_get_cl` 的 `ToolDefinition` + 导出 `CRBUDDY_TOOLS` | Q1/Q3：自定义工具的形状（ToolDefinition），SDK customTools 来源 |
| `runtime.ts`（+2） | `customTools: CRBUDDY_TOOLS` 传进 `createAgentSession` | Q3：三来源之一「SDK 直接传入」 |
| `gerrit.ts`（+90） | `getChangeForReview()` 聚合 CL 详情；`get()` 加 429 退避重试 | 工具的真实血肉（execute 里调的领域 client，pi 不管） |
| `prompt.ts`（+18） | system prompt 里告诉模型遇到 CL 就调 `gerrit_get_cl` | Q2：harness「把说明摆好」，引导模型何时调 |
| `main.ts`（−8） | 删掉前端一个 echo-skip hack（迁移后不需要了） | 无关 tool，顺手清理 |

## 核心：一个工具怎么接进去（三步，全对应 04 篇）

```
1. 写 ToolDefinition（tools.ts）
   gerrit_get_cl = { name, label, description, promptSnippet,
                     parameters(TypeBox), execute() }
        ↓
2. 放进 customTools 数组传给 createAgentSession（runtime.ts）
   createAgentSession({ noTools:'builtin', customTools: CRBUDDY_TOOLS })
        ↓ pi 内部 _refreshToolRegistry 把它合进 registry → active → agent.state.tools
3. system prompt 引导模型何时调（prompt.ts）
```

这正是文章 Q1 那句的真实印证：**pi 只给「形状」（ToolDefinition 类型 + TypeBox），血肉（execute 调 gerrit client）全是自己写的。** pi 完全不知道 gerrit 是什么。

## 两个设计决定，正好用上 04 篇的判断

1. **`noTools: 'builtin'`** —— 关掉内置 read/bash/edit/write。
   - 理由（注释原文）：web-facing, no fs access。web 服务不该有文件系统操作。
   - 对应 Q3「准入控制」+ Q1 引申的「能力消除优于运行时防御」：不需要的工具从源头不给，比给了再拦更稳。

2. **领域工具走 `customTools`，不走 extension `registerTool`**
   - crbuddy 用 SDK 嵌入，没有 extension 文件系统，customTools 是最直接的入口。
   - 对应 Q3 三来源里的「SDK 直接传入（_customTools）」。

## 一个值得记的工程细节（超出 pi）

`gerrit.ts` 的 `get()` 加了 **429 退避重试**——Gerrit 对突发请求限流。这是「工具 execute 里要处理真实世界的不可靠」的例子，和 04 篇 Q6（read 截断、bash 杀进程树）同一类：**工具的复杂度不在「调一次 API」，而在处理它的现实问题（限流、超大输出、失控进程）。**

## E2E 验证（commit message 记录）

- 问「review CL 7915991」→ 模型发出 `gerrit_get_cl` tool call，参数正确；
- `tool_end ok:true`；
- 抓取失败时，模型**如实报告**而不是编造 review —— 对应 04 上篇「错误变成结果回传给模型，模型据此决定下一步」。

## 一句话总结

这次实战把 04 篇「工具供给侧」的链路完整走了一遍真实版：**写 ToolDefinition（形状由 pi 给、血肉自己写）→ customTools 传入 → registry 汇总 → 暴露给模型 → prompt 引导**。关掉内置文件工具 + 只给安全领域工具的选择，也印证了「准入控制」这条判断。下一个工具（比如 weather）就是把这套再走一遍。
