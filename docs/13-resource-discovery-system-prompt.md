# 第 13 讲：资源发现与系统提示组装

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 把“启动时找到哪些资源”和“最终给模型什么系统提示”分成两层。`PackageManager` 负责收集路径、标记来源并排定优先级；`ResourceLoader` 读取不同格式的文件、处理同名资源和诊断；`AgentSession` 再把工具说明、系统提示、项目上下文与技能索引组装成模型看到的字符串。

```text
settings / packages / 自动目录 / CLI / extension hook
                         │
                         ▼
              路径 + SourceInfo + 优先级
                         │
                         ▼
                 ResourceLoader.reload()
             ┌───────────┼───────────┐
          skills       prompts      themes
             │             │           │
             └──── 同名资源 first wins ┘
                         │
            SYSTEM / APPEND / AGENTS 上下文
                         │
                         ▼
                 buildSystemPrompt()
                         │
                         ▼
                  Agent.state.systemPrompt
```

这套机制里有两类边界不能混在一起：

- `AGENTS.md`、`CLAUDE.md` 是按目录层级累加的项目上下文，不按资源名互相覆盖；
- skill、prompt template、theme 是具名资源，同名时只保留优先级更高、发现更早的一个，并记录 collision diagnostic。

## 1. 发现层保留路径来源，加载层解释文件内容

### 解决的问题

同一个 skill 可能来自项目 `.pi`、用户目录、包依赖或命令行。只返回一串路径无法回答“为什么加载了这个版本”“另一个版本为什么消失”。Pi 因此让发现结果同时携带来源信息。

`packages/coding-agent/src/core/source-info.ts:1-17` 定义了 `SourceInfo`：

```ts
export interface SourceInfo {
	path: string;
	source: string;
	scope: "user" | "project" | "temporary";
	origin: "package" | "top-level";
	baseDir?: string;
}
```

`path` 是实际文件路径；`scope` 表示用户、项目或本次进程临时来源；`origin` 区分包内资源和直接配置的资源；`source` 保留包声明或顶层来源标签。它们主要服务于诊断和界面展示，不会原样塞进系统提示。

### 来源入口

`packages/coding-agent/src/core/package-manager.ts:2310-2460` 自动检查这些位置：

| 范围 | extensions | skills | prompts | themes |
|---|---|---|---|---|
| 项目 | `.pi/extensions` | `.pi/skills`、祖先目录的 `.agents/skills` | `.pi/prompts` | `.pi/themes` |
| 用户 | agent 目录下的对应子目录 | agent 目录下的 `skills`、`~/.agents/skills` | agent 目录下的 `prompts` | agent 目录下的 `themes` |

项目 `.pi` 和项目侧 `.agents/skills` 只有在项目受信任时才加入结果。用户资源仍可加载。`ResourceLoader.reload()` 在 `packages/coding-agent/src/core/resource-loader.ts:338-493` 先完成信任判断，再用新的信任状态重载 settings 和资源。

扩展还有一个动态入口。扩展处理 `resources_discover` 事件后，可以返回 skill、prompt 和 theme 路径。`packages/coding-agent/src/core/agent-session.ts:2234-2256` 把这些路径标成 `temporary`，交给 `extendResources()` 合并，并立即重建系统提示。扩展不能通过这个事件添加 `AGENTS.md` 或替换 `SYSTEM.md`。

### 失败路径

自动目录不存在时等价于没有资源，不产生错误。命令行或 API 明确给出的附加路径不存在时，`ResourceLoader` 会生成 error diagnostic；单个资源内容无效时通常只产生 warning，其他资源继续加载。启动层或交互界面再决定如何展示这些 diagnostics。

## 2. 优先级先排序，同名处理只认“第一个”

### 排序规则

`packages/coding-agent/src/core/package-manager.ts:175-192` 给路径来源分配固定等级，数值越小越靠前：

| 等级 | 来源 |
|---:|---|
| 0 | 项目 settings 明确配置 |
| 1 | 项目自动发现 |
| 2 | 用户 settings 明确配置 |
| 3 | 用户自动发现 |
| 4 | package 内声明的资源 |

`toResolvedPaths()` 在 `packages/coding-agent/src/core/package-manager.ts:2527-2547` 先按这个等级排序，再按规范化后的绝对路径去重。相同包同时出现在项目和用户配置时，`dedupePackages()` 让项目声明优先，入口在 `packages/coding-agent/src/core/package-manager.ts:1697-1730`。

`ResourceLoader` 随后按数组顺序读取。skill、prompt、theme 的名字若已出现，旧值是 winner，新值成为 loser：

```ts
const existing = seen.get(t.name);
if (existing) {
	diagnostics.push({
		type: "collision",
		message: `theme name collision \"${t.name}\"`,
		resourceType: "theme",
		winnerPath: existing.sourcePath ?? "<builtin>",
		loserPath: t.sourcePath ?? "<builtin>",
	});
	continue;
}
```

代码位于 `packages/coding-agent/src/core/resource-loader.ts:943-960`。collision 不会让整次加载失败，也不会合并两个资源；运行时只留下 winner。回归测试 `packages/coding-agent/test/suite/regressions/2781-skill-collision-precedence.test.ts:16-120` 固定了“项目 skill 胜过用户 skill，用户 skill 胜过包内 skill”的结果和诊断内容。

### CLI 不是一条笼统的最高优先级

显式 extension 路径和由 `--extension` 带入的 package resources 会放到对应列表前面。直接追加的 `--skill`、`--prompt-template`、`--theme` 路径则在已解析资源之后合并。因而判断碰撞结果时要看具体参数落入哪条路径，不能只用“CLI 总是覆盖配置”概括。

同一实际路径会先按规范路径去重；两个不同文件声明同名资源才形成名称碰撞。这个区分避免同一个文件通过多条发现路径重复报警，同时保留真正的来源冲突。

## 3. `AGENTS.md` 是目录上下文链，不是具名资源

### 搜索与顺序

`packages/coding-agent/src/core/resource-loader.ts:66-119` 从两部分加载上下文：

1. 先检查用户 agent 目录；
2. 再从文件系统根目录走到当前 cwd，逐级检查每个目录。

每个目录的候选顺序固定为 `AGENTS.md`、`AGENTS.MD`、`CLAUDE.md`、`CLAUDE.MD`，只读取该目录中第一个存在且可读的候选。不同目录的文件不会互相覆盖，最终顺序是用户全局上下文、较远祖先、较近祖先、当前目录。

```ts
const candidates = ["AGENTS.md", "AGENTS.MD", "CLAUDE.md", "CLAUDE.MD"];
for (const candidate of candidates) {
	const candidatePath = join(dir, candidate);
	try {
		const content = readFileSync(candidatePath, "utf-8");
		return { path: candidatePath, content };
	} catch {
		// Try next candidate
	}
}
```

这意味着同一目录同时存在 `AGENTS.md` 和 `CLAUDE.md` 时，后者不会加载；父目录和子目录各有一个时，两者都会进入提示。这里只做精确路径去重，不解析规则覆盖关系，冲突指令最终由模型面对。

### 信任边界

`--no-context-files` 会让这条链整体为空。项目未受信任时，`.pi` 内的 extension、skill、prompt、theme、`SYSTEM.md` 被挡住，但项目路径上的 `AGENTS.md` 仍会加载。这个行为由 `packages/coding-agent/test/resource-loader.test.ts:348-434` 直接覆盖。

因此，项目信任不能被理解为“禁止读取项目中的所有上下文”。当前实现的信任门针对可执行扩展和 `.pi` 资源；普通项目上下文文件属于另一条发现链。

## 4. skill 先向模型公开索引，需要时再读正文

### 目录与文件格式

skill 的核心入口是 `packages/coding-agent/src/core/skills.ts:1-486`。一个目录根部存在 `SKILL.md` 时，该目录视为一个 skill，扫描不再深入；否则会接受根目录直属的 Markdown 文件，并递归寻找子目录中的 `SKILL.md`。扫描跳过隐藏目录和 `node_modules`，遵守 `.gitignore`、`.ignore`、`.fdignore`，也会处理符号链接。

frontmatter 至少要提供非空 `description`。`name` 缺失时使用父目录名；名称规则要求小写字母、数字和连字符，总长不超过 64。名称不规范会报警但仍可加载，缺少 description 则产生 warning 并排除该 skill。

### 系统提示只放元数据

`formatSkillsForPrompt()` 在 `packages/coding-agent/src/core/skills.ts:330-374` 只生成名称、说明和文件位置：

```ts
const visibleSkills = skills.filter((skill) => !skill.disableModelInvocation);
return `<available_skills>
${visibleSkills
	.map(
		(skill) => `<skill>
<name>${escapeXml(skill.name)}</name>
<description>${escapeXml(skill.description)}</description>
<location>${escapeXml(skill.filePath)}</location>
</skill>`,
	)
	.join("\n")}
</available_skills>`;
```

正文不会在启动时全部占用上下文。模型先根据 description 决定是否需要，再用 `read` 读取对应文件。正因为这条工作流依赖 `read`，`buildSystemPrompt()` 只有在活动工具包含 `read` 时才加入 skill 索引。

`disable-model-invocation: true` 只把 skill 从模型可见索引中移除，不会从资源列表删除。用户仍可显式执行 `/skill:name`。`packages/coding-agent/src/core/agent-session.ts:1284-1315` 在这时读取完整文件、去掉 frontmatter、附上 skill 基目录并展开参数；读取失败会发出 extension error，原始输入不会被悄悄吞掉。

### 设计取舍

只注入索引显著减少常驻上下文，也让 skill 内容可以在文件修改后按需读取。代价是模型必须拥有读取工具并做一次额外工具调用；仅有名称而 description 含糊时，发现质量也会下降。

## 5. prompt template 在用户提交前做一次文本展开

### 加载与命名

`packages/coding-agent/src/core/prompt-templates.ts:1-270` 把 Markdown 文件名作为命令名。description 优先取 frontmatter；没有时取正文第一行并截断到 60 个字符。`argument-hint` 只用于提示参数形式，正文才是展开结果。

目录扫描只读取该目录直属的 `.md` 文件，不递归。文件不可读或 frontmatter 无法解析时，该模板被跳过；同名模板仍遵循 first wins，并产生 collision diagnostic。

### 展开流程

只有输入以 `/模板名` 开头时才匹配。参数解析支持引号，替换语法包括 `$1`、`$@`、`$ARGUMENTS`、`${N:-default}`、`${@:N}` 和 `${@:N:L}`。替换只做一轮，不递归解释替换结果中的占位符。

状态变化发生在模型调用之前：用户输入字符串变成模板正文，模板参数被填入，随后这段普通文本才进入会话消息。模板本身不进入系统提示，也不形成持久化的独立消息类型。参数解析和替换边界由 `packages/coding-agent/test/prompt-templates.test.ts` 覆盖。

这一实现简单、可预测，不需要模板执行环境；相应地，它没有条件分支、循环或脚本能力。需要动态行为时应由 extension command 或 tool 承担，而不是把 prompt template 当作程序。

## 6. theme 是界面资源，不参与模型上下文

`ResourceLoader` 对 theme 路径的处理位于 `packages/coding-agent/src/core/resource-loader.ts:805-889`。目录只扫描直属 JSON 文件，单个 JSON 路径和指向 JSON 的符号链接也可加载。`loadThemeFromPath()` 在 `packages/coding-agent/src/modes/interactive/theme/theme.ts:623-627` 读取 JSON、校验结构和必需颜色，再生成 `Theme`。

内置 theme、旧式自定义 theme 和资源加载器注册的 theme 最终汇入可选主题列表。列表按名字去重，内置项先加入，见 `packages/coding-agent/src/modes/interactive/theme/theme.ts:462-488`。资源加载阶段内部的同名自定义 theme 仍由来源顺序决定 winner。

文件不存在、类型不是 JSON、JSON 解析失败、主题名或颜色结构非法都会形成 diagnostic；其余主题继续可用。theme 只影响交互界面的颜色和渲染，不会写入 `Agent.state.systemPrompt`。

## 7. `SYSTEM.md` 替换基础提示，`APPEND_SYSTEM.md` 追加内容

### 来源选择

`ResourceLoader` 对两个文件使用不同槽位，而不是把它们当 prompt template：

- `SYSTEM.md`：受信任项目的 `.pi/SYSTEM.md` 优先，否则使用用户 agent 目录下的 `SYSTEM.md`；
- `APPEND_SYSTEM.md`：受信任项目的 `.pi/APPEND_SYSTEM.md` 优先，否则使用用户文件；
- 显式 `systemPrompt` 或 `appendSystemPrompt` 输入优先于自动发现，输入可以是文件路径，也可以是字面文本。

自动发现代码位于 `packages/coding-agent/src/core/resource-loader.ts:965-990`。显式路径读取失败时会记录 warning，然后把原输入当作字面提示继续使用，入口在 `packages/coding-agent/src/core/resource-loader.ts:50-64`。这条回退能兼容直接传文本，但路径拼错也可能变成系统提示正文，诊断不能忽略。

### “替换”不等于清空其他上下文

`SYSTEM.md` 替换 Pi 内置的基础提示；它不会替换 `APPEND_SYSTEM.md`、`AGENTS.md` 上下文、skill 索引或 cwd。`APPEND_SYSTEM.md` 则接在基础提示之后。两者的职责分别是定义 agent 基础身份和附加约束。

## 8. 最终系统提示按固定顺序组装

`AgentSession._rebuildSystemPrompt()` 在 `packages/coding-agent/src/core/agent-session.ts:1009-1042` 收集当前活动工具、工具提示片段、工具指南、资源提示、skills 和上下文文件，再调用 `buildSystemPrompt()`。

`packages/coding-agent/src/core/system-prompt.ts:1-162` 的最终顺序是：

1. 自定义 `SYSTEM`，或 Pi 内置基础提示；
2. `APPEND_SYSTEM`；
3. 全局与项目 `AGENTS.md` / `CLAUDE.md` 内容；
4. 活动工具含 `read` 时的 skill 索引；
5. 当前工作目录。

使用内置基础提示时，只有实际活动的工具会获得对应说明。扩展工具若没有提供 prompt snippet，不会凭工具名自动生成说明；各工具的 guidelines 会去空白、去重后并入通用指南。测试 `packages/coding-agent/test/system-prompt.test.ts:1-112` 覆盖了无工具、工具片段和指南去重等边界。

项目上下文写入提示时带文件路径包装，使模型能区分规则来自哪一级文件。skill 同样带实际位置，便于模型按需读取。这些路径是运行时信息；公开笔记或可提交配置不应复制本机绝对路径。

## 9. reload 重建资源和运行时，不新建会话历史

### 调用链与状态变化

交互模式触发 reload 后，`packages/coding-agent/src/core/agent-session.ts:2577-2600` 依次执行：

```ts
const previousFlagValues = this._extensionRunner.getFlagValues();
await emitSessionShutdownEvent(this._extensionRunner, {
	type: "session_shutdown",
	reason: "reload",
});
await this.settingsManager.reload();
this.syncQueueModesFromSettings();
resetApiProviders();
await this._resourceLoader.reload();
this._buildRuntime({
	activeToolNames: this.getActiveToolNames(),
	flagValues: previousFlagValues,
	includeAllExtensionTools: true,
});
```

随后在已经绑定交互上下文时，新的 extension runtime 收到 `session_start(reason: "reload")`，扩展重新执行 `resources_discover`。资源扩展完成后，`_rebuildSystemPrompt()` 的结果写回 `agent.state.systemPrompt`。

reload 保留同一个 `SessionManager` 和现有消息历史，但替换 extension runner、刷新工具注册表、资源缓存与基础系统提示。扩展 flag 值被显式带到新 runtime；settings 中的 queue mode 和 provider 状态也会重新同步。回归测试 `packages/coding-agent/test/suite/regressions/2753-reload-stale-resource-settings.test.ts:24-104` 先修改 settings 禁用 prompt，再调用 reload，确认旧资源不会残留。

### 失败边界

资源加载器尽量把单项问题降为 diagnostic，因此某个 theme 或 skill 损坏通常不会阻止其他资源。扩展是可执行代码，加载失败或 reload 事件失败可能中断对应阶段。旧 extension context 属于旧 runner，reload 后继续持有它会产生过期上下文问题；需要长期生效的扩展状态应放在 session 或外部持久层，而不是假设旧 runtime 对象仍有效。

## 10. 失败诊断要区分“路径”“内容”和“碰撞”

| 失败点 | 决策位置 | 结果 |
|---|---|---|
| 明确附加路径不存在 | `ResourceLoader.reload()` | error diagnostic，该资源缺席 |
| 自动发现目录不存在 | `PackageManager` | 视为无资源，不报警 |
| skill 缺 description | skill parser | warning，不加载该 skill |
| skill 名不规范 | skill parser | warning，当前实现仍加载 |
| prompt 文件不可读或不可解析 | prompt loader | 跳过该模板 |
| theme JSON 或颜色结构非法 | theme loader | warning，该 theme 缺席 |
| 同名 skill/prompt/theme | 各类型去重器 | collision diagnostic，只保留 winner |
| 显式 SYSTEM 路径读取失败 | prompt input resolver | warning，并把输入当字面文本 |
| 项目未受信任 | trust + discovery | 不加载项目 `.pi` 资源；项目上下文链仍存在 |
| reload 后旧扩展上下文继续使用 | extension runtime | 上下文过期，不能代表新 runtime |

把 diagnostic 与最终有效资源同时保留，是这套设计的重要取舍：配置中的局部错误不会把整个 agent 变成不可用，但“进程能启动”也不代表期望资源已经生效。排查时应查看 winner、loser、来源和 scope，而不是只看目录里有没有文件。

## 11. 常见边界问答

### 子目录的 `AGENTS.md` 会覆盖父目录吗？

不会在加载阶段覆盖。父子文件按从远到近的顺序都进入系统提示。Pi 不解析两份自然语言规则的语义优先级；模型如何处理矛盾指令是提示层行为。

### 项目不受信任时，项目规则是否完全消失？

不会。项目 `.pi` 资源和项目侧 `.agents/skills` 被信任门拦截，但 cwd 祖先链上的 `AGENTS.md` / `CLAUDE.md` 仍会读取。这是当前源码和测试确认的边界。

### 自定义 `SYSTEM.md` 能否彻底控制系统提示？

它能替换 Pi 内置基础提示，不能自动移除追加提示、项目上下文、skill 索引和 cwd。若要求只保留一段自定义文本，还需要显式关闭上下文文件和 skills，并留意 cwd 仍由 builder 添加。

### skill 正文为什么不在启动时全部加载？

系统提示只放 skill 的名称、description 和 location。这样常驻 token 成本与 skill 数量近似按元数据增长，而不是按全部正文增长。代价是模型需要 `read` 并主动选择正确 skill。

### 修改资源文件后，当前对话会丢失吗？

reload 会重载 settings 和资源、重建 extension runtime、工具注册与系统提示，不会创建新的 `SessionManager`，已有消息仍在当前会话中。新系统提示只影响 reload 之后的模型调用，不会改写已持久化的历史消息。

### 为什么同名资源不做深度合并？

skill 和 prompt 的正文缺少可靠的结构化合并语义，theme 合并还可能形成缺色或变量引用不一致。first wins 加 collision diagnostic 让最终结果可追踪，也把组合需求留给明确的新资源，而不是在加载器里隐式拼接。
