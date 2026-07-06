# Pi 系列 09｜Skills 系统：reusable instruction 怎么按需加载

本系列其他文章：

placeholder

> 本文源码主要在 `core/skills.ts`、`formatSkillsForPrompt`、`resource-loader.ts`。

前面几篇拆的是 agent 内部机制。这一篇看一个更贴近使用的东西：skill——一份可复用的指令文件，让 agent 在特定任务上按预设的方式工作。

进入源码之前，先看一个矛盾。

一份 skill 可能写得很长，几百上千 token。一个项目里可以有几十份 skill。如果把所有 skill 的正文都放进 system prompt，上下文很快被占满，而且当前这个任务大概率只用得上其中一两份。

pi 的处理是：system prompt 里每份 skill 只放三样——名字、描述、文件路径。模型看到"有哪些 skill、各自干什么、文件在哪"，当任务和某份 skill 的描述匹配时，再用 `read` 工具去读那份文件，正文这时才进上下文。

这就是本文的主线：skill 怎么被发现、怎么只把描述交给模型、正文怎么按需加载。

## 一、skill 在磁盘上是什么

一份 skill 是一个 `SKILL.md` 文件，头部是 YAML frontmatter，正文是给模型的指令。frontmatter 有三个字段：

```ts
interface SkillFrontmatter {
  name?: string;                        // 可选，缺省用父目录名
  description?: string;                 // 必填，缺了不加载
  "disable-model-invocation"?: boolean; // true = 模型看不到，只能显式调用
}
```

解析后产出一个 `Skill` 对象，带上名字、描述、文件路径、来源等信息。三个字段里，`description` 是唯一必填的——后面会看到，描述就是模型判断"这份 skill 是否匹配当前任务"的依据，没有描述这份 skill 就没有意义。

<!-- 图1：skill 文件结构
生图 prompt：
一张横版示意图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左边画一个文件图标标「SKILL.md」。文件内部分上下两块：上块用蓝绿色 #2A9D8F 标「frontmatter」，里面三行「name（可选）」「description（必填）」「disable-model-invocation（可选）」；下块用浅灰蓝 #E6EDF2 标「正文：给模型的指令（可能很长）」。
右边用一个箭头指向一个卡片「Skill 对象：name / description / filePath / disableModelInvocation」，标「解析后」。
底部小字：「description 必填——它是模型判断这份 skill 是否匹配任务的依据。」
-->
![skill 文件结构：frontmatter + 正文，解析成 Skill 对象](./pi_09_1.png)

## 二、从哪发现

`loadSkills` 扫三处目录，每处标一个来源：

| 来源 | 目录 |
|---|---|
| user | `<agentDir>/skills/` |
| project | 项目配置目录下的 `skills/` |
| path | 额外显式传入的路径 |

user 是用户级（跨项目通用），project 是项目级（跟着代码库走），path 是显式指定的额外路径。三处的 skill 汇到一起，交给下面的解析。

## 三、目录怎么扫

`loadSkillsFromDir` 的扫描规则有三条：

- 某个目录里有 `SKILL.md`，就把这个目录当作一份 skill，不再往下递归；
- 否则，加载这个目录下直接的 `.md` 文件；
- 递归进子目录，继续找 `SKILL.md`。

第一条是关键：一份 skill 可以带自己的子目录（放引用的文件、脚本等），只要根上有 `SKILL.md`，pi 就认这一份、不把子目录里的文件误当成别的 skill。

## 四、解析一份 skill

`loadSkillFromFile` 做这几步：

1. 读文件，切出 frontmatter；
2. 校验名字和描述——名字只能是小写字母、数字、连字符，不能首尾或连续连字符；描述必填、有长度上限；
3. 名字缺省时，用父目录名兜底；
4. 产出 `Skill` 对象。

这里的校验是宽松的：名字不规范只记一条警告，不阻止加载；唯一会导致这份 skill 被丢弃的，是描述完全为空。设计上倾向"尽量加载"，把不规范当提示而非错误。

<!-- 图2：skill 的发现与解析
生图 prompt：
一张横版流程图，从左到右，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
最左三个文件夹图标竖排，用蓝绿色 #2A9D8F 标「user：agentDir/skills」「project：项目/skills」「path：显式路径」，汇成一条线进入中间方框「loadSkills」。
中间方框后接一个方框「loadSkillsFromDir：目录里有 SKILL.md 就认作一份，不再递归」。
再接一个方框「loadSkillFromFile：切 frontmatter + 校验」，下方用两个分支：一个绿色勾「名字不规范→只警告，仍加载」，一个珊瑚色 #E76F51 叉「description 为空→丢弃」。
最右输出一叠卡片标「Skill 对象列表」。
底部小字：「三处目录发现，宽松校验，只有描述为空才丢弃。」
-->
![skill 的发现与解析：三处目录 → 扫描 → 校验 → Skill 列表](./pi_09_2.png)

## 五、怎么交给模型

这是 skill 系统的核心：`formatSkillsForPrompt` 不把正文放进上下文，只在 system prompt 里放一份清单，每份 skill 三样信息，用 XML 包起来：

```
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>/绝对路径/SKILL.md</location>
  </skill>
</available_skills>
```

三个要点：

- **清单里有一句明确指示**："当任务和描述匹配时，用 read 工具加载这份 skill 文件。" 平时不加载正文，需要时模型自己去读。
- **`disable-model-invocation: true` 的 skill 被排除出清单**。模型看不到它，只能由用户显式调用（下一节）。
- **这份清单只在有 `read` 工具时才拼进 system prompt**（Context 篇讲过 `buildSystemPrompt` 里的判断）。原因很直接：正文要靠 read 工具去读，没有 read 工具，给了清单也用不上。

<!-- 图3：只注入描述，正文按需读
生图 prompt：
一张横版对比图，左右两栏，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左栏标题「system prompt 里」：画一份清单卡片，列三份 skill，每份只有三行小字「name / description / location」，用蓝绿色 #2A9D8F；卡片旁标「只有描述，占用很小」。
右栏标题「任务匹配时」：画一个模型图标，从其中一份 skill 的 location 用箭头指向一个展开的文件图标「SKILL.md 正文」，用琥珀色 #E9B44C，标「模型用 read 工具读取，正文这时才进上下文」。
中间用一条虚线分隔两栏，虚线上标「渐进式加载」。
底部小字：「描述当索引，正文当按需内容。几十份 skill 也只占很小的固定空间。」
-->
![只把描述放进 prompt，正文匹配时才用 read 读取](./pi_09_3.png)

这套做法有个名字：渐进式加载（progressive disclosure）。描述当索引，正文当按需内容。它和工具系统同一个思路——先给模型一份轻量目录，真要用时再展开具体内容。几十份 skill 在 system prompt 里也只占很小的固定空间。

## 六、用户显式调用

标了 `disable-model-invocation` 的 skill，模型在清单里看不到，但用户可以用 `/skill:name` 显式触发。

pi 的 slash command 有三种来源：extension、prompt、skill。每份 skill 会注册成一个命令，命令名是 `skill:<skill 名>`。所以一份 skill 有两条进入模型的路径：

- 模型自主：skill 在清单里，模型根据描述判断要不要用，用 read 读正文；
- 用户显式：`/skill:name` 直接触发，不受 `disable-model-invocation` 限制。

`disable-model-invocation` 的意义就在这里：有些 skill 不希望模型自己乱用（比如有副作用的、或只在特定场合用的），就把它从清单里拿掉，只留显式调用这一条路。

## 完整链路

```
ResourceLoader.getSkills()
        ↓
loadSkills()                     // 扫 user / project / path 三处目录
        ↓
loadSkillsFromDir()              // 有 SKILL.md 就认作一份，不再递归
        ↓
loadSkillFromFile()              // 切 frontmatter + 校验 → Skill 对象
        ↓
formatSkillsForPrompt()          // 只把 name + description + location 放进 system prompt
        ↓
模型 read <location> 读正文（任务匹配时）  或  用户 /skill:name 显式调用
```

回到开头那个矛盾——skill 可能很长、数量可能很多，但 system prompt 不能被占满。pi 的解法是把"有哪些 skill"和"skill 的正文"分开：前者是一份轻量清单，一直在 system prompt 里；后者是文件，模型按需去读。描述负责让模型知道"有这么个东西、什么时候用"，正文负责在真正用到时提供细节。

<!-- 图4：skill 完整链路
生图 prompt：
一张横版端到端流程图，从左到右一条主线，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
主线节点依次（圆角方块）：1「loadSkills」标"扫三处目录"；2「loadSkillsFromDir」标"发现 SKILL.md"；3「loadSkillFromFile」标"解析+校验"；4「formatSkillsForPrompt」用蓝绿色 #2A9D8F 标"只注入 name+description+location"；5「system prompt」。
从节点 5 引出两条分支：上分支用琥珀色 #E9B44C「模型 read location 读正文」标"任务匹配时"；下分支用靛蓝 #5B7DB1「用户 /skill:name」标"显式调用，不受 disable-model-invocation 限制"。
底部小字：「有哪些 skill」是常驻清单，「skill 正文」按需加载——两者分开是这套设计的核心。
-->
![skill 完整链路：发现 → 解析 → 注入描述 → 按需读取或显式调用](./pi_09_4.png)

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*
