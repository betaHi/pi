# 07 Skills 系统 · 学习笔记

> README 核心问题：reusable instruction 怎么被发现、暴露、按需加载？为什么只把描述放进 prompt、正文靠 read 取？
> 证据来源：`core/skills.ts`、`formatSkillsForPrompt`、`resource-loader.ts`、`interactive-mode.ts`。
> 一句话：skill = 一个带 frontmatter 的 `SKILL.md`；发现后只把「名字+描述+位置」注入 system prompt，正文按需由模型用 `read` 工具自取（渐进式加载）。

---

## 题眼：为什么只注入描述，不注入正文

一份 skill 可能很长（几百上千 token）。如果把所有 skill 正文都塞进 system prompt，几十个 skill 就把上下文占满了，而且大多数当前任务用不上。

pi 的做法：system prompt 里每个 skill 只放三样——`name`、`description`、`location`（文件路径）。模型看到"有哪些 skill、各自干嘛、文件在哪"，**匹配到任务时才用 `read` 工具去读那份 `SKILL.md`**，正文这时才进上下文。

这是渐进式加载（progressive disclosure）：描述当索引，正文当按需内容。它和 tool 系统同思路——先给模型一个轻量目录，真要用再展开。

---

## 一、skill 在磁盘上是什么

一个 skill = 一个 `SKILL.md` 文件，头部是 YAML frontmatter，正文是给模型的指令。frontmatter 字段（`SkillFrontmatter`）：

```ts
interface SkillFrontmatter {
  name?: string;                        // 可选，缺了用父目录名兜底
  description?: string;                 // 必填，缺了这个 skill 不加载
  "disable-model-invocation"?: boolean; // 可选，true = 模型看不到，只能 /skill:name 显式调
}
```

解析后产出 `Skill` 对象：

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

## 二、从哪发现（三个来源）

`loadSkills` 扫三处，来源标记 `user` / `project` / `path`：

| 来源 | 目录 |
|---|---|
| user | `<agentDir>/skills/` |
| project | `<cwd>/<CONFIG_DIR_NAME>/skills/`（项目级） |
| path | 额外显式传入的路径 `skillPaths` |

`getSource` 判断一个路径归哪类：显式路径若落在 user/project 目录下仍归对应类，否则归 `path`。

---

## 三、目录扫描规则（`loadSkillsFromDir`）

源码注释写明三条：
- 某目录里有 `SKILL.md` → 把这个目录当一个 skill 根，**不再往下递归**；
- 否则，加载根目录下直接的 `.md` 子文件；
- 递归子目录去找 `SKILL.md`。

还支持 ignore 文件过滤。

---

## 四、解析一个 skill（`loadSkillFromFile`）

1. `readFileSync` 读文件 → `parseFrontmatter` 切出 frontmatter；
2. **校验**：`validateDescription`（必填、长度上限）、`validateName`（只能小写字母数字连字符、不能首尾连字符、不能连续 `--`）；
3. **容错**：校验出 warning **不阻止加载**（进 diagnostics）；**唯一硬失败是 description 完全为空** → 返回 null；
4. name 缺省用**父目录名**兜底（`frontmatter.name || parentDirName`）；
5. 产出 `Skill` 对象。

→ 校验宽松：名字不规范只警告，只有描述为空才拒。设计上倾向"尽量加载"。

---

## 五、怎么交给模型（`formatSkillsForPrompt`，核心）

**不把正文塞进上下文**，只在 system prompt 里放一个 XML 清单，每个 skill 三样：

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

关键点：
- **prompt 明说 "Use the read tool to load a skill's file when the task matches its description"** —— 平时不加载正文，匹配时模型自己 `read`。
- **`disableModelInvocation: true` 的 skill 被排除出清单**（`visibleSkills = skills.filter(s => !s.disableModelInvocation)`）——模型看不到，只能显式调。
- skills 清单只在**有 read 工具时**才拼进 system prompt（见 Context 篇：`buildSystemPrompt` 里 `hasRead && skills.length` 判断）——没 read 工具，清单给了也没用。

---

## 六、`/skill:name` 显式调用

`disableModelInvocation` 的 skill 模型看不到，但用户能用 `/skill:name` 显式触发。

- slash command 有三种来源：`SlashCommandSource = "extension" | "prompt" | "skill"`。
- 每个 skill 注册成一个命令：interactive 模式里 `const commandName = skill:${skill.name}`。

→ 两条路进模型：① 模型自主（在清单里、用 read 取正文）② 用户 `/skill:name` 显式（不受 disableModelInvocation 限制）。

---

## 核心链路

```
ResourceLoader.getSkills()
  → loadSkills()                     // 扫 user/project/path 三目录
    → loadSkillsFromDirInternal()    // 发现 SKILL.md（有则不递归）
      → loadSkillFromFile()          // 解析 frontmatter + 校验 → Skill 对象
  → formatSkillsForPrompt(skills)    // 只注入 name+description+location 到 system prompt
  → 模型 read <location> 取正文（渐进式加载） / 或用户 /skill:name 显式调
```

---

## 一句话总结

skill = 带 frontmatter 的 `SKILL.md`；三个目录发现 → 解析校验成 `Skill` → 只把「名字+描述+位置」注入 system prompt，正文按需由模型用 `read` 自取。这是**用描述当索引、正文当按需内容**的渐进式加载，省上下文。

---

## 待深入
- [ ] `parseFrontmatter` 的实现（utils/frontmatter.ts），未细读。
- [ ] skill 正文里引用相对路径的解析规则（prompt 里提到"resolve against skill directory"）——机制点到，未验证实现。
- [ ] agent 包里的 `formatSkillsForSystemPrompt`（core 版）与 coding-agent 版差异：已确认逻辑相同，仅换行 + "用 read 工具"措辞不同；core 版当前链路未被调用。