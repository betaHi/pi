# 07 Skills 系统 · 学习笔记

> 核心问题：reusable instruction 怎么被发现、暴露、按需加载？为什么只把描述放进 prompt，正文靠 read 或显式命令再取？
> 证据来源：`packages/coding-agent/src/core/skills.ts`、`package-manager.ts`、`resource-loader.ts`、`system-prompt.ts`、`agent-session.ts`、`docs/skills.md`，以及 `packages/agent/src/harness/skills.ts`。
> 一句话：skill 是带 frontmatter 的 Markdown 能力包。pi 启动时只把 `name`、`description`、`location` 放进 system prompt；真正的说明正文要么由模型按描述用 `read` 加载，要么由用户用 `/skill:name` 强制展开。

---

## 题眼：skill 以索引常驻，正文按需进入上下文

skill 的正文可以很长，也可能带脚本、参考资料和使用流程。如果启动时把所有正文塞进 system prompt，成本高，且多数任务用不上。

pi 的做法是两层：
- 常驻层：system prompt 里只放 skill 清单，包含名字、描述、位置和相对路径解析规则。
- 按需层：模型判断任务匹配时用 `read` 读 `SKILL.md`；用户也可以用 `/skill:name` 把正文直接展开到本轮输入。

这里的重点是把 skill 做成可检索的能力索引。描述写得准，模型才更可能在合适任务里读取正文。

---

## 一、skill 在磁盘上是什么

常见形态是一个目录加一个 `SKILL.md`：

```text
my-skill/
  SKILL.md
  scripts/
  references/
  assets/
```

`SKILL.md` 的 frontmatter 当前实现会读取这些字段：

```ts
interface SkillFrontmatter {
  name?: string;
  description?: string;
  "disable-model-invocation"?: boolean;
  [key: string]: unknown;
}
```

实现上的关键点：
- `description` 必须有，空描述的 skill 不加载。
- `name` 可以缺，缺了用父目录名兜底；这比 Agent Skills 标准更宽松。
- `name` 和 `description` 有长度、字符规则校验，但大多数问题只产生 warning，不阻止加载。
- 未使用的 frontmatter 字段会被忽略。比如 docs 里提到的 `allowed-tools` 当前没有接入工具准入逻辑，不能把它当权限机制。

解析后得到的对象大致是：

```ts
interface Skill {
  name: string;
  description: string;
  filePath: string;
  baseDir: string;
  sourceInfo: SourceInfo;
  disableModelInvocation: boolean;
}
```

---

## 二、从哪里发现

coding-agent 的来源范围比两个固定目录更宽。真实路径来源先由 `DefaultPackageManager` 汇总，再交给 `DefaultResourceLoader` 加载。

主要来源：

| 来源 | 说明 |
|---|---|
| 项目 `.pi/skills/` | 自动发现，支持根目录 `.md` 和递归 `SKILL.md` |
| 全局 `~/.pi/agent/skills/` | 自动发现，规则同 `.pi/skills/` |
| 项目 `.agents/skills/` | 从当前 cwd 往祖先目录找，直到 git 根或文件系统根；只发现递归 `SKILL.md` |
| 全局 `~/.agents/skills/` | 兼容其他 harness；只发现递归 `SKILL.md` |
| settings / CLI | `settings.json` 的 `skills`、`--skill <path>`，可指文件或目录 |
| packages | npm/git/local package 的 `skills/` 或 `pi.skills` manifest |
| extension | `resources_discover` 事件可返回额外 skill 路径 |

`--no-skills`/`noSkills` 会关掉默认/自动发现路径；显式 CLI/SDK 追加路径仍可加载。这能同时支持“默认不要扫”和“我明确指定这个 skill”两种需求。

冲突处理也很重要：同名 skill 只保留第一个，后面的产生 collision diagnostic。package-manager 会按优先级排序，项目配置和项目自动发现优先于用户级，package 资源靠后；CLI/显式路径在 resource loader 里作为显式输入合并。

---

## 三、目录扫描规则

`loadSkillsFromDirInternal` 的核心规则：

1. 如果当前目录有 `SKILL.md`，这个目录就是一个 skill 根，加载后不再递归子目录。
2. 否则在“根目录允许直接文件”的模式下，加载直接子级 `.md` 文件。
3. 继续递归子目录寻找 `SKILL.md`。
4. 跳过点目录、`node_modules`，并读取 `.gitignore`、`.ignore`、`.fdignore`。

`.pi/skills` 和 `~/.pi/agent/skills` 使用 pi 模式，根目录 `.md` 会被当作 skill；`.agents/skills` 使用 agents 模式，根目录 `.md` 不加载，只找 `SKILL.md`。这个差异容易漏。

---

## 四、怎么交给模型

`buildSystemPrompt` 在 read 工具可用时追加 skill 清单。没有 read 工具，给出路径也无法让模型读取正文，所以清单会被跳过。

`formatSkillsForPrompt` 会过滤掉 `disableModelInvocation=true` 的 skill，然后生成 XML：

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

这里有三个设计点：
- prompt 里放的是索引，不放正文。
- location 是绝对路径，并且提示相对路径按 skill 目录解析。
- `disable-model-invocation` 只影响“模型自主发现”，不影响用户显式调用。

---

## 五、`/skill:name` 显式调用

交互界面会把 skill 暴露成 `/skill:<name>` slash command，前提是 `enableSkillCommands` 开启，默认开启。执行路径在 `AgentSession._expandSkillCommand`：

1. 识别 `/skill:name args`。
2. 从已加载 skills 里按 name 找到对应文件。
3. 读取 `SKILL.md`，去掉 frontmatter。
4. 包成：

```text
<skill name="..." location="...">
References are relative to ...

正文
</skill>

用户附加参数
```

这里的参数会原样追加，没有单独结构化字段。`disable-model-invocation` 的 skill 不出现在 system prompt 清单里，但仍可以用 `/skill:name` 强制展开。

`packages/agent` 的 harness 层也有同类能力：`formatSkillInvocation` 和 `AgentHarness.skill()` 做的是相同思想，只是没有 coding-agent 的 slash command UI 包装。

---

## 六、边界

pi 的 skill 系统解决的是发现、索引和按需加载，不解决这些问题：

- 不保证模型一定会读匹配的 skill；必要时用 `/skill:name` 强制。
- 不自动执行 skill 里的脚本；脚本只是说明和资源，是否执行仍由模型调用工具完成。
- 不用 `allowed-tools` 做权限控制；工具可用性仍由 session 的工具注册、allowlist 和 active tools 决定。
- 不做 skill 内容安全审查；skill 可以要求模型执行危险动作，使用前要审阅来源。

---

## 核心链路

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
  -> 或用户 /skill:name 强制展开正文
```

---

## 一句话总结

skill 是“描述常驻、正文按需”的能力包。pi 负责发现来源、校验、去重、把索引放进 system prompt，并提供 `/skill:name` 强制展开；skill 的选择质量、安全性和具体执行仍要靠描述、用户判断和工具准入来保证。

---

## Q & A

### Q：skill metadata 在 什么时候会被加载进system prompt，用户发消息时候吗

不是“用户一发消息才加载 skill metadata”。

更准确是：

1. 启动/创建 session runtime 时，先创建 `DefaultResourceLoader` 并执行 `resourceLoader.reload()`，这一步会发现并解析 skills metadata：`name`、`description`、`filePath`、`disable-model-invocation` 等。见 [packages/coding-agent/src/core/agent-session-services.ts](packages/coding-agent/src/core/agent-session-services.ts#L137-L143) 和 [packages/coding-agent/src/core/resource-loader.ts](packages/coding-agent/src/core/resource-loader.ts#L320-L390)。

2. 构造 `AgentSession` 时会 `_buildRuntime()`，里面最终 `_rebuildSystemPrompt()`，从 `resourceLoader.getSkills().skills` 取出已加载的 skills，然后调用 `buildSystemPrompt()` 拼进 base system prompt。见 [packages/coding-agent/src/core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts#L855-L888)。

3. 用户发消息时，通常不会重新扫描 skill 文件；`prompt()` 只是把当前 `_baseSystemPrompt` 应用到本轮，除非 extension 的 `before_agent_start` 修改 system prompt。然后底层 `agentLoop` 在真正请求模型前，把 `context.systemPrompt` 放进 LLM context。见 [packages/coding-agent/src/core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts#L1031-L1088) 和 [packages/agent/src/agent-loop.ts](packages/agent/src/agent-loop.ts#L288-L302)。

另外，system prompt 里放的不是 skill 全文，而是可见 skill 的索引块：`name`、`description`、`location`。全文只有两种常见路径会进上下文：模型看到 location 后用 `read` 工具读取，或者用户显式输入 `/skill:name ...`，这会把 skill 文件内容展开成用户消息块。相关格式在 [packages/coding-agent/src/core/skills.ts](packages/coding-agent/src/core/skills.ts#L336-L367)，显式 `/skill:` 展开在 [packages/coding-agent/src/core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts#L1118-L1155)。

所以一句话：skill metadata 在 session 初始化/reload/资源扩展或工具集变化导致重建 system prompt 时加载并拼入；用户发消息时只是使用这个已经构建好的 system prompt 发送给模型。

### Q：考虑一个实际问题，如果skill很多，怎么才能找到我们想要的

短答案：**skill 多了以后，不能只靠 system prompt 里的一长串 `<available_skills>` 让模型直接选择**。那只是最基础的索引方式，适合几十个以内；再多就会遇到：

- prompt 变长，常驻成本上升。
- 多个 description 相似，模型容易选错。
- 任务描述不够明确时，模型可能漏选。
- 新增/修改 skill 后，如果没有 reload 或重建 system prompt，当前会话不一定看到。

可以按规模分三层处理。数量只是粗略提示，更重要的是看模型还能不能稳定地从清单里选对：

| 层级 | 粗略数量 | 更重要的判断标准 |
|---|---:|---|
| 少量 skill | 1-15 个左右 | skill 之间边界清楚，description 不相似，模型基本能直接选对 |
| 中量 skill | 15-60 个左右 | 清单还能放进 prompt，但开始出现相似 skill、跨项目无关 skill、低频流程 skill |
| 大量 skill | 60-100+ 个 | 清单占用较多上下文，模型选择不稳定，很多 skill 与当前任务无关，需要先检索/过滤 |

更实用的划分方式是看这些信号：

1. **看 prompt 成本**

  如果 `<available_skills>` 已经占了明显上下文，或者每轮都带着很多当前任务用不上的 metadata，就进入中量/大量问题了。

2. **看相似度**

  数量不多但 description 很像，也应该按中量处理。比如：

  - `release-checklist`
  - `release-notes`
  - `release-versioning`
  - `release-publish`

  这类 skill 只有十几个，也可能让模型选错。

3. **看作用域混杂程度**

  如果全局 skill、项目 skill、package skill、extension skill 混在一起，当前项目只会用其中一部分，就不能只看总数。即使 30 个也应该做 scope 控制。

4. **看选择失败率**

  如果经常出现这些情况，就说明已经超过“少量”：

  - 模型没有读应该读的 skill。
  - 模型读了不相关的 skill。
  - 用户经常需要纠正“不是这个 skill”。
  - 你开始依赖 `/skill:name` 才能保证正确。

5. **看更新频率**

  如果 skill 经常新增、删除、改名，或者来自 extension/package 动态发现，就更适合 retrieval 或 `search_skills`，因为静态 system prompt 清单容易过期。

**第一层：少量 skill**

少量 skill 时，当前 pi 的基础机制就够用：启动/reload 时加载所有可见 skill metadata，然后把 `name + description + location` 放进 system prompt。模型看到清单后，按 description 判断是否需要 `read` 对应的 `SKILL.md`。

这一层的重点是：

- skill 数量少，prompt 成本可控。
- 每个 skill 的 description 写清楚即可。
- 用户知道要用哪份时，仍然可以 `/skill:name` 点名。

**第二层：中量 skill**

中量 skill 的意思是：数量已经多到会出现干扰，但还没有多到必须上检索系统。这个阶段的目标不是“搜索所有 skill”，而是先管理好哪些 skill 会被模型看到、什么时候被看到、用户怎么点名。

要让它“找得到”，现在能做的是：

1. **控制可见范围**

   不要把所有全局 skill 都默认暴露给每个项目。项目 `.pi/skills` 放项目相关的，全局 skill 放真正通用的。少用大而全的全局 skill 仓库。

2. **写好 `description`**

   description 不是介绍文案，而是检索触发条件。应该写：

   ```text
   Use when analyzing pi agent session lifecycle, resource loading, or system prompt construction.
   ```

   而不是：

   ```text
   Helps understand pi.
   ```

   好 description 要有领域、任务、触发词、边界。最好还能包含“不适用场景”。

3. **用 `disable-model-invocation` 管理低频 skill**

   有些 skill 不该让模型自己发现，比如危险操作、少见流程、非常具体的发布流程。可以设 `disable-model-invocation: true`，让它不出现在 system prompt，只能用户 `/skill:name` 点名调用。

4. **命名要可搜索**

   `skill name` 应该像命令名，而不是文章标题。比如：

   ```text
   agent-session-lifecycle
   skill-system-analysis
   release-checklist
   provider-integration
   ```

   这样用户用 `/skill:` 补全或搜索时也容易找到。

5. **显式调用是兜底**

   如果你知道要用哪份 skill，最好直接 `/skill:name`。模型自主选择适合“我不确定是否需要 skill”的情况；用户点名适合“我明确知道这轮要按某套流程来”。

这一层可以理解成：先通过 scope 减少无关 skill，通过 description 提高模型判断质量，通过 `disable-model-invocation` 隐藏不该自主触发的 skill，再用 `/skill:name` 做人工兜底。

**第三层：大量 skill**

大量 skill 的意思是：skill 已经多到不适合全部放进 `<available_skills>`。这时继续扩大 system prompt 清单会降低选择稳定性，也会增加每轮请求的固定上下文成本。

大量 skill 有两种更合适的做法。

**做法 A：skill retrieval layer**

这是“模型请求前”的自动检索层。它不让模型先阅读完整 skill 清单，而是在构造本轮上下文前，先由 harness 根据用户消息和运行环境筛出少量候选。

流程可以是：

```text
User message
  -> skill retriever
       - BM25 / embedding / tag filter
       - 根据 cwd、package、语言、任务类型过滤
       - 返回 top-k skill metadata
  -> system prompt 只注入 top-k candidates
  -> 模型决定 read 哪个 SKILL.md
```

这一层可以做几件事：

- 建索引：把 skill 的 `name`、`description`、`location`、`sourceInfo`、可选 tags、可选正文摘要放进索引。
- 取查询：用用户最新消息、最近几轮上下文、cwd、当前 package、文件语言、命令来源构造 query。
- 先过滤：按项目/全局 scope、package、语言、任务类型、`disable-model-invocation` 做硬过滤。
- 再排序：用 BM25、embedding similarity、关键词命中、来源优先级、最近使用记录综合打分。
- 控制数量：只返回 top-k，比如 3 到 8 个，避免候选列表再次膨胀。
- 给理由：每个候选最好带 `score` 和 `reason`，让模型知道为什么它被选出来。

它的优点：

- 不需要把所有 skill metadata 常驻进 system prompt。
- 候选更贴近当前项目和当前任务。
- 选择发生在模型外部，更容易测试和调参。
- 可以缓存索引，只在 reload、skill 文件变化或配置变化时更新。

它的风险：

- retriever 如果召回失败，模型就看不到正确 skill。
- scoring 规则会影响模型行为，需要观察误召回和漏召回。
- top-k 太小会漏，太大又回到 prompt 膨胀问题。

所以 retrieval layer 适合做默认路径，但最好保留 `/skill:name` 和 `search_skills` 作为备用路径。

**做法 B：`search_skills(query)` 工具**

这是“模型请求中”的主动搜索工具。system prompt 不再列几百个 skill，只告诉模型：需要专门流程时，可以先调用 `search_skills` 找候选。

工具返回可以是：

```text
search_skills(query) -> [{ name, description, location, score }]
```

更完整一点，结果可以包含：

- `name`：用于用户或模型识别 skill。
- `description`：用于判断是否匹配当前任务。
- `location`：给模型后续用 `read` 读取 `SKILL.md`。
- `score`：检索分数。
- `reason`：为什么命中，比如关键词、tag、项目 scope、最近使用。
- `sourceInfo`：来自项目、用户全局、package 还是 extension。

运行流程是：

```text
User message
  -> model sees: use search_skills when specialized instructions may help
  -> model calls search_skills("...")
  -> tool returns candidate skills
  -> model chooses one or more locations
  -> model uses read to load SKILL.md
  -> model follows the skill content
```

它的优点：

- system prompt 更轻，不需要常驻大清单。
- 模型可以按自己的理解改写 query，必要时多搜一次。
- 对开放问题更友好，比如用户只说“帮我处理发布流程”。
- skill 集合很动态时，工具可以实时查最新索引。

它的风险：

- 多一次工具调用，延迟和 token 成本会增加。
- 模型可能忘记搜索，所以 system prompt 要明确什么时候调用。
- 工具结果必须短，不应把 skill 正文直接返回；正文仍应由 `read` 按需读取。
- 查询词如果太泛，需要工具支持 tag/filter 或返回可解释 reason。

两个方案可以组合：

- 默认先用 retrieval layer 自动注入 top-k。
- 如果 top-k 不够，模型再调用 `search_skills(query)`。
- 用户明确知道要用哪份时，仍然直接 `/skill:name`。
- 执行顺序是：自动召回优先，模型主动搜索作为补充，用户点名作为最后的明确选择。

所以总结一下：**少量 skill 靠 system prompt 索引；中量 skill 靠 scope、description、disable-model-invocation 和 `/skill:name` 管理可见性；大量 skill 才进入 retrieval layer 或 `search_skills(query)`，只给模型本轮相关的候选。**

### Q：如果 skill 太多，已经超过上下文额度，该怎么办

这种情况不能只靠压缩历史消息解决。原因是 skill 清单在 system prompt 里；如果 `<available_skills>` 本身已经过大，每轮请求都会携带这段内容。即使历史消息被压缩，system prompt 里的 skill metadata 仍然会占用上下文。

处理原则是：**全量 skill 可以加载到系统内部，但不要全量暴露给模型**。也就是把“发现并加载所有 skill”和“本轮让模型看到哪些 skill”分开。

当前 pi 里可以先做这些事：

1. **减少默认可见 skill**

  - 项目相关 skill 放在项目 `.pi/skills/`。
  - 全局目录只保留真正通用的 skill。
  - 低频、危险、强流程化 skill 设置 `disable-model-invocation: true`。
  - 临时任务可以用 `--no-skills` 或 SDK 的 `noSkills` 关闭默认发现，再显式传入需要的 `--skill <path>` 或 SDK skill 路径。

2. **用 `/skill:name` 点名调用**

  如果用户已经知道要用哪份 skill，就不要依赖完整清单。直接 `/skill:name` 展开正文，这样可以绕开“模型必须先从大清单里找到它”的问题。

3. **按项目和来源拆分 skill**

  不同项目、package、extension 的 skill 不一定都要进入同一个会话。优先让当前 cwd 只暴露当前项目常用的 skill。

4. **修改后 reload**

  skill metadata 和 system prompt 有缓存。移动、禁用、改名、改 description 后，需要 reload 或重建 system prompt 才会影响当前会话。

更长期的设计应该加预算控制：

1. **内部保留全量索引**

  所有 skill metadata 可以进入内存索引、SQLite、BM25 索引或 embedding 索引，但不直接全部写进 system prompt。

2. **给 skill prompt 设置 token budget**

  例如限制 skill 清单最多占上下文的 5%-10%，或者设置一个固定上限。超过预算时，只注入 top-k 候选。

3. **先检索，再注入**

  每轮根据用户消息、cwd、当前文件、package、任务类型检索候选，只把最相关的几个 skill metadata 放进 system prompt。

4. **提供 `search_skills(query)`**

  如果自动注入的候选不够，模型可以再调用工具搜索。工具结果也要限制数量和长度，只返回 `name`、`description`、`location`、`score`、`reason` 这类 metadata。

5. **保留显式调用路径**

  `/skill:name` 仍然重要。它是用户明确知道要用哪份 skill 时的直接入口。

超过预算时的降级规则可以是：

- 优先减少候选数量，而不是截断 XML 或 Markdown 结构。
- 优先保留 `name`、短 `description`、`location`。
- `reason`、长描述、正文摘要可以在预算不足时省略。
- 如果没有候选能安全放入预算，就只告诉模型使用 `search_skills(query)`。
- 不要把 skill 正文放进 system prompt；正文仍然按需 `read`。

所以答案是：**skill 太多并且超过上下文额度时，不应该继续扩大 `<available_skills>`。应该限制 system prompt 里的 skill 预算，把全量 skill 放到外部索引，通过 retrieval layer、`search_skills(query)` 和 `/skill:name` 分层取用。**
