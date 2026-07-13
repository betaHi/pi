# Pi 系列 13｜自我进化：更新面、验证和回退边界

本系列其他文章：

placeholder

> 本文主要参考 `agent-session.ts`、`resource-loader.ts`、`extensions/runner.ts`、`messages.ts`、`session-manager.ts`，以及 `examples/extensions/prompt-customizer.ts`、`dynamic-tools.ts`、`plan-mode/index.ts`。

上一篇 memory 关注如何把事实和偏好带回上下文。这一篇看另一类问题：应用能否根据反馈调整 agent 的行为规则，例如改 prompt、沉淀 skill、调整工具集。

当前 pi 源码里没有专门的自我进化模块，也没有通用 eval、reward、自动回滚或长期反馈学习器。pi 提供的是几个可更新的运行时表面。如何判断要不要更新、怎样验证、如何回退，需要应用层设计。

## 一、一个可控的更新流程

如果把“自我进化”写成工程流程，通常至少有四步：

| 步骤 | 做什么 | pi 提供 | 应用层要补 |
|---|---|---|---|
| 评估 | 判断当前行为哪里不好 | 事件流、session 历史、工具结果、用户输入 | 指标、反馈识别、人工确认、失败归因 |
| 更新 | 改 prompt、skill、工具集、模型策略 | 多个运行时更新面 | 生成修改方案、写文件、版本化 |
| 验证 | 确认修改有效且副作用可接受 | 可运行命令，extension 可触发流程 | eval 集、项目测试、对比策略 |
| 回退 | 修改效果差时退回旧版本 | session append-only 留痕，reload 可重载旧资源 | 资源版本管理、回退触发、冲突处理 |

pi 覆盖的是“从哪里拿反馈”和“哪些表面可以被更新”。流程能不能稳定运行，取决于应用层的评估、验证和版本管理。

<!-- 图1：自我进化的受控闭环
生图 prompt：
一张横版闭环流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
画一个顺时针循环：评估 → 更新提议 → 试运行 → 验证 → 固化 → 回退。每个节点下方加一行短说明：评估「用户纠正/工具结果/测试」；更新提议「prompt/skill/tool 策略」；试运行「before_agent_start 或 active tools」；验证「eval/测试/确认」；固化「写文件或 settings + reload」；回退「还原旧版本 + reload」。
用蓝绿色标 pi 提供的接入点，用琥珀色标应用层策略。回退箭头回到更新提议前。
底部小字：「自我进化要分试运行和固化；验证与回退由应用层控制。」
建议文件名：./pi_13_1.png
-->

## 二、可更新面一：本轮 system prompt

`before_agent_start` 发生在用户输入展开后、agent loop 前。extension handler 可以返回新的 `systemPrompt`：

```ts
pi.on("before_agent_start", (event) => {
  return { systemPrompt: event.systemPrompt + "\n..." };
});
```

runner 会把多个 extension 的返回值串联起来。`AgentSession.prompt()` 在本轮使用这个 modified prompt；下一轮如果没有 extension 返回 prompt，系统会回到 `_baseSystemPrompt`。

这适合试运行策略。例如用户连续纠正某类输出，extension 可以在下一轮临时加一条更具体的指导语，观察效果。验证通过后，再考虑把规则写入 prompt 文件、skill 或项目上下文。

这里要注意 custom message。`before_agent_start` 也能返回 custom message。custom message 会进入本轮消息流，正常情况下会持久化并参与后续上下文。评估结果、版本号、候选规则这类内部状态不适合放这里。

<!-- 图2：临时 prompt 试运行
生图 prompt：
一张横版流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左侧是「_baseSystemPrompt」基线卡片。中间是「before_agent_start handler」节点，接收用户输入和当前 systemPrompt。右侧分两条：上条「返回 modified systemPrompt」进入「本轮 AgentSession.prompt 使用」；下条「下一轮无返回」回到「_baseSystemPrompt」。
旁边加一个小警示框：「custom message 会进入消息流；内部状态用 appendEntry」。
底部小字：「临时修改适合试运行；验证后再写入 prompt/skill/settings。」
建议文件名：./pi_13_2.png
-->

## 三、可更新面二：资源和 reload

资源层有两个相关方法：

```ts
extendResources(paths): void
reload(): Promise<void>
```

`extendResources` 会把新路径合并进 `lastSkillPaths`、`lastPromptPaths`、`lastThemePaths`，并重新加载对应资源。资源真正进入当前 system prompt 的位置在 `AgentSession.extendResourcesFromExtensions()`：它处理 `resources_discover` 的返回值，调用 `extendResources`，然后重建 system prompt。

`session.reload()` 的范围更大：

- 触发 `session_shutdown`，reason 是 `reload`。
- 重新加载 settings、资源、extensions。
- 重建 ExtensionRunner 和工具注册表。
- 已经绑定 UI 或命令上下文时，再触发 `session_start` 和 `resources_discover`。
- reload 后旧 ctx 会失效。

因此，规则和技能的固化通常可以走两条路：修改磁盘上的 prompt/skill/settings 后 reload；或者 extension 在 `resources_discover` 里提供新资源路径，再由 session 重建 prompt。

<!-- 图3：资源更新与 reload
生图 prompt：
一张横版双路径流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
路径 A：「修改磁盘 prompt/skill/settings」→「session.reload」→「重新加载 settings/resources/extensions」→「重建 ExtensionRunner 和工具注册表」→「重建 system prompt」。
路径 B：「resources_discover 返回新 skill/prompt/theme 路径」→「extendResources」→「加载新增资源」→「_rebuildSystemPrompt」。
在 reload 节点旁标「旧 ctx 失效」。在 extendResources 节点旁标「追加路径并重新加载对应资源」。
底部小字：「试运行用临时 prompt；固化通常要写资源并 reload。」
建议文件名：./pi_13_3.png
-->

## 四、可更新面三：工具策略

工具可用性有几层：

- 工具定义来自内置工具、SDK custom tools、extension `registerTool`。
- `tools` allowlist 和 `noTools` 决定哪些工具能进入注册表。
- active tools 决定当前模型实际可见的工具。

`setActiveToolsByName(toolNames)` 会保留注册表里存在的工具，更新 `agent.state.tools`，并用新的工具列表重建 base system prompt。变化在下一次 agent turn 生效。

extension 还能动态 `registerTool`。`dynamic-tools.ts` 示例演示了 session_start 或命令运行期注册新工具。注册后还要经过工具注册表刷新、allowlist 和 active tools，模型才会看到它。

这给工具策略调整留下了空间。比如某个项目默认不希望模型写文件，可以把写入工具移出 active tools；某类任务需要专门工具，可以由 extension 在适当时机注册并启用。

## 五、状态和留痕

更新系统需要记录“为什么改、改了什么、验证结果如何”。pi 里有两类 entry 容易混淆：

| API / entry | 是否进 LLM context | 适合存什么 |
|---|---|---|
| `appendEntry` / `custom` | 否 | 版本号、评估结果、外部资源 key、内部状态 |
| `sendMessage` / `custom_message` | 是 | 需要模型看到的扩展消息 |

内部状态建议放 `appendEntry`。它会留在 session tree 里，方便审计和恢复；`convertToLlm` 不会把它投影给模型。

session tree 的 append-only 特性有利于回看历史：可以看到哪一轮触发了更新、extension 存了什么状态、何时 reload。回退仍要由应用层完成，例如通过 git、文件备份或外部 store 的版本字段还原资源，再 reload。

## 六、一个可落地的设计

下面是一种基于 pi 原语的应用层设计：

```text
1. 收集反馈
  extension 监听 turn_end / message_end / tool_result
  -> 收集用户纠正、失败工具、测试结果、重复操作

2. 生成提议
  -> 候选修改：prompt 规则、skill 内容、工具 active set
  -> appendEntry 记录提议和理由

3. 试运行
  -> before_agent_start 临时修改 system prompt
  -> 或临时 setActiveTools

4. 验证
  -> 跑固定 eval、项目测试或用户确认
  -> appendEntry 记录结果

5. 固化
  -> 修改 skill / prompt / settings
  -> session.reload()

6. 回退
  -> 还原旧文件或旧设置
  -> reload
```

这个流程的关键是把试运行和固化分开。临时 prompt 和 active tools 适合试；磁盘上的 skill、prompt、settings 更适合在验证后修改。

## 七、和 memory 的关系

memory 和自我进化会用到同一批接口：session 事件、extension hook、resource loader、custom entry。它们解决的问题不同。

| | memory | 自我进化 |
|---|---|---|
| 关注点 | 把事实或偏好带回上下文 | 改变之后的行为规则 |
| 常用接口 | `context` hook、`custom_message`、project context | `before_agent_start`、`resources_discover`、`setActiveToolsByName`、`appendEntry` |
| 主要难点 | 检索、去重、作用域、注入预算 | 评估、验证、版本管理、回退 |

一个简单判断是：给模型看的事实走 memory 注入；给系统自己用的学习状态走 `appendEntry`；验证后的稳定行为再沉淀成 prompt、skill 或工具策略。

pi 提供了可更新面和事件入口。一个可靠的自我改进流程，还需要明确反馈来源、验证标准和回退方式。

<!-- 图4：memory 与自我进化共用接口
生图 prompt：
一张横版对比图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
底部画一条共享底座「session events / extension hook / resource loader / custom entry」。
上方左右两栏：左栏「memory」标「把事实或偏好带回上下文」，列 context hook、custom_message、project context；右栏「自我进化」标「调整之后的行为规则」，列 before_agent_start、resources_discover/reload、setActiveToolsByName、appendEntry。
中间用细线说明两者共享接口，但目标不同。
底部小字：「同一批接口，可以支撑不同应用层系统。」
建议文件名：./pi_13_4.png
-->

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。文中的自我进化设计是基于这些接口的应用层方案。*