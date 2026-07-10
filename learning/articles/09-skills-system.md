# Pi 系列 09｜Skills 系统：把可复用指令做成按需内容

本系列其他文章：
placeholder

> 本文主要参考 `packages/coding-agent/src/core/skills.ts`、`package-manager.ts`、`resource-loader.ts`、`system-prompt.ts`、`agent-session.ts` 和 `docs/skills.md`。

前面几篇讲的是 agent 的运行机制。到了 skill，问题换成了另一类：一段可复用的工作方法，怎样交给 agent 使用，又怎样避免长期占用上下文。

skill 可以很长，也可以带脚本、参考资料和示例。一个用户或项目里可能有多份 skill。pi 的处理方式比较直接：启动时只把 skill 的名字、描述和文件位置放进 system prompt；正文留在文件里，任务匹配时再读。用户也可以用 `/skill:name` 把某份 skill 明确展开到本轮输入里。

这篇文章按四个问题展开：skill 文件是什么，pi 从哪里发现它，怎么把它交给模型，显式调用和工具权限的边界在哪里。

## 一、skill 文件是什么

常见的 skill 是一个目录，目录里有 `SKILL.md`：

```text
my-skill/
  SKILL.md
  scripts/
  references/
  assets/
```

`SKILL.md` 头部是 YAML frontmatter，正文是给模型看的使用说明。当前 coding-agent 实现读取的 frontmatter 字段主要是这几个：

```ts
interface SkillFrontmatter {
  name?: string;
  description?: string;
  "disable-model-invocation"?: boolean;
  [key: string]: unknown;
}
```

解析后会得到一个 `Skill` 对象，里面有 `name`、`description`、`filePath`、`baseDir`、`sourceInfo` 和 `disableModelInvocation`。

这里有几个实现边界值得先记住：

- `description` 要有内容。空描述的 skill 会被跳过。
- `name` 可以缺省。缺省时用父目录名兜底。
- 名字和描述会做校验，例如名字长度、字符范围、首尾连字符等；多数问题只产生 warning，加载会继续。
- docs 里提到过 `allowed-tools`，当前源码没有把它接到工具准入逻辑里。工具能不能用，仍看 session 的工具注册、allowlist 和 active tools。

这说明 pi 对 skill 文件的校验偏宽松。真正决定一份 skill 能否进入索引的字段，是描述。

## 二、pi 从哪里发现 skill

旧版理解里容易把 skill 来源简化成 user、project、path 三类。当前 coding-agent 的路径来源更宽一些：先由 `DefaultPackageManager` 汇总，再交给 `DefaultResourceLoader` 加载。

主要来源包括：

| 来源 | 行为 |
|---|---|
| 项目 `.pi/skills/` | 自动发现，支持根目录 `.md` 和递归 `SKILL.md` |
| 全局 `~/.pi/agent/skills/` | 自动发现，规则同 `.pi/skills/` |
| 项目 `.agents/skills/` | 从 cwd 往祖先目录找，直到 git 根或文件系统根；只找递归 `SKILL.md` |
| 全局 `~/.agents/skills/` | 兼容其他 harness；只找递归 `SKILL.md` |
| settings / CLI | `settings.json` 的 `skills` 和 `--skill <path>`，可以指文件或目录 |
| packages | npm、git、local package 的 `skills/` 或 `pi.skills` manifest |
| extension | `resources_discover` 事件可以返回额外 skill 路径 |

`--no-skills` 或 SDK 里的 `noSkills` 会关掉默认和自动发现路径。显式给出的路径仍可以加载，例如 CLI 传入的 `--skill` 或 SDK 追加的路径。这能让“默认不扫描”和“明确使用某一份 skill”同时成立。

同名 skill 的处理也要注意：第一个加载到的 skill 保留，后面的同名项产生 collision diagnostic。package-manager 会先按来源优先级排序，项目配置和项目自动发现靠前，用户级资源随后，package 资源更靠后。

<!-- 图1：skill 来源汇总
生图 prompt：
一张横版流程图，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左侧分三组来源：第一组「自动发现」包含「项目 .pi/skills」「全局 ~/.pi/agent/skills」「.agents/skills」；第二组「显式配置」包含「settings skills」「CLI --skill」；第三组「扩展和包」包含「package pi.skills」「resources_discover」。三组用不同浅色底但不花哨。
中间箭头汇入「DefaultPackageManager.resolve」方框，再流向「DefaultResourceLoader.reload」方框。
右侧输出一叠「Skill 对象」，旁边标注「同名：first wins + collision diagnostic」。
底部小字：「skill 来源先汇总，再交给 resource loader 加载。不同来源决定优先级和 sourceInfo。」
建议文件名：./pi_09_1.png
-->

## 三、目录扫描规则

目录扫描的核心逻辑在 `loadSkillsFromDirInternal`。

规则可以概括为四条：

1. 当前目录如果有 `SKILL.md`，这个目录就是一份 skill 的根目录，加载后停止继续递归。
2. 如果当前目录没有 `SKILL.md`，且处在允许根目录直接文件的模式，就加载直接子级 `.md` 文件。
3. 继续递归子目录寻找 `SKILL.md`。
4. 跳过点目录、`node_modules`，并读取 `.gitignore`、`.ignore`、`.fdignore`。

第一条很重要。一份 skill 可以带自己的 `scripts/`、`references/`、`assets/`。根目录出现 `SKILL.md` 后，子目录里的 Markdown 不会被误当成另一份 skill。

`.pi/skills` 和 `~/.pi/agent/skills` 使用 pi 模式，根目录 `.md` 可以作为 skill。`.agents/skills` 使用 agents 模式，根目录 `.md` 不加载，只找 `SKILL.md`。这处差异来自兼容不同 harness 的目录约定。

## 四、交给模型的只有索引

`buildSystemPrompt` 拼 system prompt 时，会检查当前工具列表里有没有 `read`。有 `read` 时，才会追加 skill 清单。原因很直接：正文要靠文件读取工具拿到；没有 `read`，给模型文件路径也没有执行路径。

`formatSkillsForPrompt` 会过滤掉 `disable-model-invocation: true` 的 skill，然后生成一段 XML 清单，大致是：

```text
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory ...

<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>/absolute/path/SKILL.md</location>
  </skill>
</available_skills>
```

这段清单只承担索引作用：

- `name` 用来标识 skill。
- `description` 用来说明什么时候适合用。
- `location` 给出 `SKILL.md` 的完整文件路径。
- 额外提示说明相对路径要按 skill 目录解析。

正文留在文件里。模型看到清单后，如果当前任务和某个描述匹配，再用 `read` 读取对应文件。这样可以让多份 skill 同时可见，又不把所有正文长期放进 system prompt。

<!-- 图2：只交索引，正文按需读
生图 prompt：
一张横版对比图，左右两栏，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
左栏标题「常驻：system prompt 里」：画一份清单卡片，列三份 skill，每份只有三行小字「name / description / location」，用蓝绿色 #2A9D8F；卡片旁标「只放索引，比正文轻得多」。
右栏标题「按需：任务匹配时」：画一个模型图标，从其中一份 skill 的 location 用箭头指向一个展开的文件图标「SKILL.md 正文」，用琥珀色 #E9B44C，标「模型用 read 工具读取，正文这一刻才进上下文」。三份里另外两份保持折叠，标「不匹配就不读」。
中间用一条竖虚线分隔两栏，虚线上标「read 工具是这条边界」。
底部小字：「常驻的是索引，按需的是正文。描述写得准，模型才更可能在合适任务里读取。」
建议文件名：./pi_09_2.png
-->

这套设计对 description 的质量要求很高。描述过宽，模型容易误用；描述过窄，模型可能错过。写 skill 时，description 应该说明能力范围和触发场景，而不只写一句泛泛的介绍。

## 五、用户显式调用 `/skill:name`

除了模型按描述自主读取，用户也可以显式调用 skill。

交互界面会把 skill 暴露成 `/skill:<name>`，前提是 `enableSkillCommands` 开启，默认开启。真正执行的代码在 `AgentSession._expandSkillCommand`：

1. 识别 `/skill:name args`。
2. 从已加载 skill 列表里按 name 找文件。
3. 读取 `SKILL.md`，去掉 frontmatter。
4. 把正文包成一个 `<skill>` 块，再把用户参数接在后面。

展开后的形态类似：

```text
<skill name="..." location="...">
References are relative to ...

正文
</skill>

用户附加参数
```

`disable-model-invocation` 的作用也在这里体现。它会让 skill 从 system prompt 清单里消失，模型无法通过描述自主发现；用户仍可以用 `/skill:name` 展开。这适合那些只希望在明确场景中使用的 skill。

`packages/agent` 的 harness 层也有同类能力：`formatSkillInvocation` 和 `AgentHarness.skill()` 会直接把某份 skill 包成输入。coding-agent 只是把这件事接到了 slash command 和交互补全上。

<!-- 图3：一份 skill 的两条进入路径
生图 prompt：
一张横版图，一个 skill 分出两条路径，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
中间画一个文件图标「SKILL.md」用浅灰蓝 #E6EDF2。向右分出两条路径：
上路用蓝绿色 #2A9D8F 标「模型自主」：skill 出现在 system prompt 清单里 → 模型按描述判断匹配 → 用 read 读正文。路径旁标一个开关「disable-model-invocation: true 时，这条路关闭（skill 从清单里消失）」用珊瑚色 #E76F51。
下路用琥珀色 #E9B44C 标「用户显式」：用户输入 /skill:name → _expandSkillCommand 读文件去 frontmatter → 正文包成 <skill> 块接在输入里。路径旁标「只要 skill 已加载且 skill 命令开启，就可点名展开」。
底部小字：「模型按描述自主读，用户也可以点名展开。disable-model-invocation 只影响模型自主发现路径。」
建议文件名：./pi_09_3.png
-->

## 六、边界

skill 系统处理的是发现、索引和按需加载。几个边界需要分清：

- 模型看到了 skill 清单，也可能没有读取对应文件。需要确保使用时，可以用 `/skill:name`。
- skill 里的脚本和资料只是说明资源。实际执行仍要经过模型调用工具。
- `allowed-tools` 当前没有形成权限边界。
- skill 内容来自本地文件或包，使用前应审阅来源。它可以指导模型执行有副作用的命令。

## 七、完整链路

把上面的过程串起来：

```text
DefaultPackageManager.resolve()
  -> 汇总 packages / settings / 自动发现 / CLI 资源
DefaultResourceLoader.reload()
  -> loadSkills(skillPaths, includeDefaults=false)
  -> loadSkillFromFile(frontmatter 校验 + SourceInfo)
AgentSession._rebuildSystemPrompt()
  -> buildSystemPrompt(... skills ...)
  -> formatSkillsForPrompt(name + description + location)
运行时
  -> 模型按描述 read SKILL.md
  -> 用户 /skill:name 强制展开正文
```

skill 系统的价值在于把可复用指令整理成可发现的资源。常驻上下文里只放索引，细节留在文件里；需要时再读，用户也可以明确点名。这种拆法让 agent 可以带着一组可复用工作方法运行，同时把上下文成本控制在可管理范围内。

<!-- 图4：skill 完整链路
生图 prompt：
一张横版端到端流程图，从左到右一条主线，现代技术插画风格，温白背景 #F7F3EA，深蓝灰细线描边 #263238，无强渐变无厚重阴影，中文标签清晰。
主线分三段，用细分隔或背景色块区分：
第一段「发现与加载」用浅灰蓝 #E6EDF2：节点「DefaultPackageManager.resolve（汇总 packages/settings/自动发现/CLI）」→「DefaultResourceLoader.reload → loadSkills → loadSkillFromFile（frontmatter 校验）」，产出一叠「Skill 对象」。
第二段「注入索引」用蓝绿色 #2A9D8F：节点「_rebuildSystemPrompt → buildSystemPrompt → formatSkillsForPrompt」，标「只放 name + description + location」，箭头汇入一个「system prompt」方块。
第三段「运行时」从 system prompt 分两条出去：上条用琥珀色 #E9B44C「模型按描述 read SKILL.md」，下条用靛蓝 #5B7DB1「用户 /skill:name 展开正文」。
底部小字：「发现汇总 → 解析成 Skill → 只注入索引 → 运行时按需读或点名展开。」
建议文件名：./pi_09_4.png
-->

---

*本文基于对 `@earendil-works/pi-coding-agent` 源码的阅读。*
