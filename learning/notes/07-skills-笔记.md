# 07 Skills 系统 · 学习笔记

> 核心问题：reusable instruction 怎么被发现、暴露、按需加载？为什么只把描述放进 prompt，正文靠 read 或显式命令再取？
> 证据来源：`packages/coding-agent/src/core/skills.ts`、`package-manager.ts`、`resource-loader.ts`、`system-prompt.ts`、`agent-session.ts`、`docs/skills.md`，以及 `packages/agent/src/harness/skills.ts`。
> 一句话：skill 是带 frontmatter 的 Markdown 能力包。pi 启动时只把 `name`、`description`、`location` 放进 system prompt；真正的说明正文要么由模型按描述用 `read` 加载，要么由用户用 `/skill:name` 强制展开。

---

## 题眼：skill 是索引，不是常驻上下文

skill 的正文可以很长，也可能带脚本、参考资料和使用流程。如果启动时把所有正文塞进 system prompt，成本高，且多数任务用不上。

pi 的做法是两层：
- 常驻层：system prompt 里只放 skill 清单，包含名字、描述、位置和相对路径解析规则。
- 按需层：模型判断任务匹配时用 `read` 读 `SKILL.md`；用户也可以用 `/skill:name` 把正文直接展开到本轮输入。

这不是“自动执行 skill”，而是把 skill 做成可检索的能力索引。描述写得准，模型才更可能在合适任务里读取正文。

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

coding-agent 不是只扫两个固定目录。真实路径来源先由 `DefaultPackageManager` 汇总，再交给 `DefaultResourceLoader` 加载。

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

`--no-skills`/`noSkills` 不是绝对屏蔽一切：默认/自动发现会被关掉，但显式 CLI/SDK 追加路径仍可加载。这是为了让“默认不要扫”和“我明确指定这个 skill”同时成立。

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

`buildSystemPrompt` 只在 read 工具可用时追加 skill 清单。没有 read 工具，给出路径也无法让模型读取正文，所以清单会被跳过。

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

这里的参数是原样追加，不是单独结构化字段。`disable-model-invocation` 的 skill 不出现在 system prompt 清单里，但仍可以用 `/skill:name` 强制展开。

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