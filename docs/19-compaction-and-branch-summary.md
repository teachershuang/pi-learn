# 第 19 讲：压缩与分支摘要

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 不会在上下文将满时改写或删除旧 session entry。它让完整历史继续留在 JSONL 中，另行追加一个摘要节点，再用 `firstKeptEntryId` 指明哪些近期 entry 仍按原文进入模型。这样压缩改变的是“当前 branch 如何投影成模型上下文”，不是历史本身。

分支摘要解决的是另一件事。用户通过 `/tree` 离开当前 branch 时，旧路径上的决策和文件状态可能仍对新路径有用。Pi 找出旧 leaf 与目标节点之间被放弃的那段路径，为它生成摘要，并把摘要接到导航后的新 leaf 上。

```text
上下文压缩
旧摘要（可选） + 待压缩历史 ──LLM──> compaction entry
                                        │ firstKeptEntryId
                                        ▼
模型上下文 = 压缩摘要 + 近期原文 + 此后新增消息

分支导航
旧 leaf ──回溯──> 共同祖先 <──目标节点
   │
   └─ 被放弃 entries ──LLM──> branch summary ──接到导航后的 leaf
```

两条路径共用摘要格式、对话序列化和文件状态提取，但触发者、取材范围、持久化位置与失败语义并不相同。

## 1. 压缩何时发生

### 解决的问题

模型的 context window 有硬上限。Pi 不能等 provider 拒绝请求后才处理，也不能只按 session 文件大小判断，因为历史里还有废弃 branch、标签和扩展状态，它们未必进入当前请求。

压缩有三种原因：

| 原因 | 触发位置 | 输入 | 决策者 | 结果 |
|---|---|---|---|---|
| `manual` | 用户执行 `/compact` | 当前 branch、可选自定义指令 | 用户明确要求；准备阶段仍可判定无内容 | 成功后追加摘要，不自动重试 turn |
| `threshold` | prompt 发送前或一次 agent run 结束后 | 最近有效 usage、尾部消息估算、当前模型窗口 | `shouldCompact()` | run 后压缩可继续 Agent；prompt 前压缩后直接发送用户输入 |
| `overflow` | provider 报告上下文溢出后 | 失败消息、模型窗口与重试状态 | `AgentSession` 的恢复逻辑 | 压缩后可重试被中断的 turn |

`overflow` 的识别和重试属于下一讲。本讲先看三条路径汇合后的压缩机制。

### 阈值不是“已用比例”

默认配置位于 `packages/coding-agent/src/core/compaction/compaction.ts:100-110`：

```ts
export const DEFAULT_COMPACTION_SETTINGS: CompactionSettings = {
	reserveTokens: 16384,
	keepRecentTokens: 20000,
};

export function shouldCompact(
	contextTokens: number,
	contextWindow: number,
	settings: CompactionSettings,
): boolean {
	if (!settings.enabled) return false;
	return contextTokens > contextWindow - settings.reserveTokens;
}
```

`reserveTokens` 是必须留给下一次回复、工具结果和协议开销的空间。100K 窗口、10K reserve 时，89K 不触发，95K 触发；测试见 `packages/coding-agent/test/compaction.test.ts:274-299`。

Pi 优先使用最近一条正常 assistant message 的 `usage.totalTokens`。这比逐条累加 usage 更可靠，因为 provider 返回的是该次请求的整体上下文用量，不是该消息自身的大小。若最近的 assistant message 被中止、报错或 usage 为零，`estimateContextTokens()` 会向前寻找有效 usage，再按字符数估算其后的消息；找不到锚点时才估算全部上下文。入口在 `packages/coding-agent/src/core/compaction/compaction.ts:120-212`。

图片按固定的 4800 字符折算，文本粗略按四字符一个 token。这个估算只服务于提前触发，不试图复刻每家 tokenizer。直接引入各 provider tokenizer会提高局部精度，却会让 coding-agent 核心依赖模型协议和版本。

### 两个自动检查点

`AgentSession` 在两个位置调用 `_checkCompaction()`：

- `packages/coding-agent/src/core/agent-session.ts:1049-1090`：一次 agent run 收尾时检查；若压缩结果要求继续，调用 `agent.continue()`。
- `packages/coding-agent/src/core/agent-session.ts:1185-1190`：发送新 prompt 前检查；此处忽略“是否继续”的返回值，因为新的用户消息马上会启动下一轮。

检查逻辑还会排除旧模型留下的 overflow 标记，以及最新压缩节点之前的 assistant usage，见 `packages/coding-agent/src/core/agent-session.ts:1935-2024`。否则刚压缩完的 session 可能被旧的高 usage 立即再次触发。

## 2. cut point 决定摘要和原文的边界

### 概念模型

`keepRecentTokens` 不是简单截取最后 N 个 token。Pi 从 branch 末尾向前估算 entry 大小，先找到“近期原文大致够多”的位置，再把边界校正到合法 cut point。

合法边界包括 `user`、`assistant`、bash execution、custom、branch summary 和 compaction summary；`toolResult` 不能作为边界。规则位于 `packages/coding-agent/src/core/compaction/compaction.ts:282-435`。

```ts
function isValidCutEntry(entry: SessionEntry): boolean {
	if (entry.type === "message") {
		const role = entry.message.role;
		return role === "user" || role === "assistant";
	}
	return (
		entry.type === "bashExecution" ||
		entry.type === "custom" ||
		entry.type === "branchSummary" ||
		entry.type === "compaction"
	);
}

function isTurnStartEntry(entry: SessionEntry): boolean {
	if (entry.type === "message") return entry.message.role === "user";
	return entry.type === "bashExecution" || entry.type === "custom" ||
		entry.type === "branchSummary" || entry.type === "compaction";
}
```

tool result 必须依附 assistant 发出的 tool call。若从 tool result 中间切开，近期上下文会出现“有结果却没有调用”或相反的协议残片。assistant 可以成为 cut point，是为了处理单个 turn 已经大于近期保留预算的情况；它不等于 turn 起点。

算法的状态变化可以写成：

```text
branch 末尾
  → 向前累计 entry token，达到 keepRecentTokens
  → 在候选位置及其后选择最近的合法 cut point
  → 向前吸收紧邻的、不进入上下文的元数据 entry
  → firstKeptEntryId = cut entry.id
  → cut 之前进入摘要，cut 及之后保留原文
```

向前吸收元数据，是为了防止 model/thinking change、label 等节点孤悬在被压缩区和保留区之间。它们不直接交给 LLM，却会影响 branch 的状态解释。

`findCompactionCutPoint()` 可能返回空：整个 branch 都落在近期预算内，或者没有合法边界。此时手动压缩报告 `Nothing to compact (session too small)`，自动压缩返回 `false`，不会写一个没有缩减效果的摘要节点。

## 3. split turn：一个 turn 自己就放不下

### 解决的问题

一次 turn 可能包含很长的用户输入、多轮 tool call/result 和 assistant 续写。若整个 turn 超过 `keepRecentTokens`，坚持只在 turn 起点切分会让“近期原文”预算失效；直接在 assistant 边界切开，又会让摘要缺少这个 turn 的开头。

当 cut point 不是 turn 起点，且前方能找到所属 turn 的起点时，`findCompactionCutPoint()` 返回 `isSplitTurn: true` 和 `turnStartIndex`。`prepareCompaction()` 随后分成两批材料：

```text
更早历史              当前超长 turn                         近期原文
┌──────────────┐  ┌──────────────────────────┐  ┌────────────────┐
│ history      │  │ turn prefix              │  │ cut ... leaf   │
└──────┬───────┘  └────────────┬─────────────┘  └────────▲───────┘
       │ 常规摘要              │ turn-prefix 摘要                 │
       └──────────────┬────────┘                                │
                      ▼                                         │
            合并后的 compaction summary ── firstKeptEntryId ───┘
```

准备逻辑见 `packages/coding-agent/src/core/compaction/compaction.ts:615-712`，合并逻辑见 `packages/coding-agent/src/core/compaction/compaction.ts:740-826`。常规摘要保留长期目标和进展，prefix 摘要专门说明当前 turn 开头的用户意图、已执行工作和衔接位置。两者分开请求模型，避免旧历史淹没刚被切掉的 turn 前缀。

如果没有更早历史，常规部分写入 `No prior history.`，仍为 prefix 单独生成摘要。任一模型请求报错都会使默认压缩失败，不会追加半成品节点。

## 4. 摘要不是自由发挥的段落

### 结构化输出

默认提示词要求摘要包含 Goal、Instructions、Progress、Key decisions、Next steps 和 Critical context。已存在旧摘要时使用 update 版本：保留仍有效的内容，合并新进展，删除已经失效的信息。提示词在 `packages/coding-agent/src/core/compaction/compaction.ts:441-511`。

这不是可供程序严格解析的 JSON schema。结构的作用是约束模型关注工程状态，持久化字段仍只有一段 `summary` 文本以及独立的 `details`。因此标题遗漏不会破坏 session 格式，却可能降低后续模型恢复任务的质量。

生成请求的输入路径位于 `packages/coding-agent/src/core/compaction/compaction.ts:546-609`：

```ts
const llmMessages = await convertToLlm(messages);
const conversationText = serializeConversation(llmMessages);
const userContent = previousSummary
	? `<conversation>\n${conversationText}\n</conversation>\n\n` +
		`<previous-summary>\n${previousSummary}\n</previous-summary>`
	: `<conversation>\n${conversationText}\n</conversation>`;

const response = await streamFn(model, {
	systemPrompt: systemPromptWithInstructions,
	messages: [{ role: "user", content: userContent, timestamp: Date.now() }],
});
```

它先经 `convertToLlm()` 处理 coding-agent 的扩展消息，再序列化为带角色标签的纯文本。tool result 超过 2000 字符时才截断，user 和 assistant 文本不在这一层截断；图片不写进摘要材料。实现位于 `packages/coding-agent/src/core/compaction/utils.ts:78-149`，对应测试为 `packages/coding-agent/test/compaction-serialization.test.ts:5-77`。

压缩摘要的最大输出 token 取 `reserveTokens × 0.8` 与模型输出上限的较小值，split-turn prefix 使用 `reserveTokens × 0.5`。摘要请求优先复用 session 的 `streamFn`，没有时调用基础 `completeSimple()`。前者使自定义 provider 调用路径仍能参与压缩。

## 5. 重复压缩如何推进窗口

第一次压缩后，session 类似：

```text
A ─ B ─ C ─ D ─ K ─ L
            ▲       ▲
            │       └─ 当前 leaf
            └─ compaction(firstKeptEntryId = K)
```

下一次不能只摘要 D 之后的新消息，也不能从 session 根重新摘要。`prepareCompaction()` 找到前一个 compaction 后，把它的 `firstKeptEntryId` 作为新摘要范围的起点，并把旧 `summary` 单独放进 `<previous-summary>`：

```text
旧 summary 作为待更新状态
K、L……中再次落到 cut 之前的原文作为新增材料
新 cut 及之后继续保留原文
```

源码入口为 `packages/coding-agent/src/core/compaction/compaction.ts:615-712`。若旧节点指向的 K 仍完全落在 `keepRecentTokens` 内，没有新增材料可压缩，准备阶段返回空。`packages/coding-agent/test/compaction.test.ts:395-506` 覆盖“无需二次压缩”和“窗口前移后重新摘要旧保留区”两种情况。

`firstKeptEntryId` 因而有两个职责：

1. 上下文投影从哪里恢复原文；
2. 下一次压缩从哪里开始收集新增摘要材料。

它不是“最后一个被删除节点”，也不是数组下标。entry ID 让边界不依赖 JSONL 的物理行号；代价是损坏或缺失 ID 会让恢复窗口退化。第 18 讲已经分析 `buildContextEntries()` 找不到该 ID 时的行为。

## 6. 文件状态为何单独累计

自然语言摘要可能漏掉文件，也可能把“读过”和“修改过”混在一起。Pi 从 assistant tool call 中识别 `read`、`write`、`edit` 的 `args.path`，生成两组状态：

- `readFiles`：读过但未修改的文件；
- `modifiedFiles`：write 或 edit 过的文件，优先级高于 read。

提取与排序逻辑位于 `packages/coding-agent/src/core/compaction/utils.ts:10-76`。生成结果既存入 `details`，也以 `<read-files>`、`<modified-files>` 标签附到摘要文本末尾。这样模型能看到文件状态，程序又能在下一次摘要时合并集合。

这里有清楚的信任边界：

- 常规压缩只继承上一个由 Pi 生成的 compaction details，再扫描本次被摘要消息中的 tool call；
- 分支摘要继承被放弃路径上由 Pi 生成的 branch-summary details，并扫描本次 token 预算实际纳入的普通消息；
- 扩展返回的摘要标记为 `fromHook`，默认实现不会把其任意 `details` 当作可信的 `FileOperations` 继续累计。

相关入口是 `packages/coding-agent/src/core/compaction/compaction.ts:41-69` 与 `packages/coding-agent/src/core/compaction/branch-summarization.ts:189-241`。只识别工具名和 `args.path` 是保守策略：自定义工具即使修改文件，也不会自动进入列表；反过来，工具调用失败与否并未在这里核验，列表表达的是调用意图和已记录操作，不是磁盘审计结果。

## 7. 持久化发生在摘要成功之后

手动路径位于 `packages/coding-agent/src/core/agent-session.ts:1771-1906`，自动路径位于 `packages/coding-agent/src/core/agent-session.ts:2029-2195`。两者都遵循相同提交顺序：

```text
准备 CompactionPreparation
  → 发出 session_before_compact，允许取消或接管
  → 默认生成摘要，或采用扩展结果
  → 再次检查 AbortSignal
  → SessionManager.appendCompaction(...)
  → buildSessionContext() 重建 agent.state.messages
  → 发出 session_compact / compaction_end
```

摘要生成期间，旧 entries 和 leaf 都不变。只有完整结果拿到且没有 abort，才追加 compaction entry。追加之后又从 SessionManager 重建内存消息，而不是直接拼接摘要，这保证运行态和持久化投影使用同一套规则。

`AgentSession.isCompacting` 同时观察手动压缩、自动压缩和分支摘要的三个 AbortController，见 `packages/coding-agent/src/core/agent-session.ts:931-937`。界面因此可以用一个状态阻止重入，而 `abortCompaction()` 和 branch-summary abort 仍有各自的控制器。

失败结果有所区分：

| 失败点 | 手动压缩 | 自动压缩 | 持久化变化 |
|---|---|---|---|
| 没有模型或认证 | 抛出错误 | 发出失败事件并返回 `false` | 无 |
| 没有可压缩材料 | 抛出“session too small”或“already compacted” | 返回 `false` | 无 |
| 扩展取消 | 抛出 cancelled | 发出 aborted 结束事件并返回 `false` | 无 |
| 默认摘要调用失败 | 抛出错误 | 记录失败并返回 `false` | 无 compaction entry |
| append 后事件 handler 失败 | extension runner 隔离 handler 错误 | 同左 | compaction 已保存 |

## 8. 分支摘要从哪里取材

### 找出被放弃的路径

`collectEntriesForBranchSummary()` 接收旧 leaf 和目标节点。它构造目标节点到根的路径，找出与旧 branch 最深的共同祖先，再从旧 leaf 向上收集到共同祖先之前，最后反转为时间顺序。实现位于 `packages/coding-agent/src/core/compaction/branch-summarization.ts:102-140`。

```text
root ─ A ─ B ─ C ─ D       old leaf = D
          └─ X ─ Y         target = Y

common ancestor = A
entriesToSummarize = B, C, D
```

共同祖先不需要进入摘要，因为它本来就在目标路径中。被放弃路径里的 compaction 和 branch summary 可以继续作为摘要材料；tool result entry 不单独序列化，因为 assistant tool call 已说明调用，且完整结果可能很大。

### token 预算与嵌套摘要

分支摘要预算是模型 context window 减去 reserve。`prepareBranchEntries()` 先遍历所有被放弃 entries，累计可信的旧 branch-summary 文件状态；再从最新 entry 向前装入消息，直到预算耗尽。若越界项本身是 compaction 或 branch summary，且当前预算使用量低于 90%，仍允许纳入该摘要后停止。入口位于 `packages/coding-agent/src/core/compaction/branch-summarization.ts:189-241`。

这个例外偏向保留已有的高密度摘要。普通长消息严格受预算约束，旧摘要则可能包含一整段已经压缩过的决策链。代价是预算是基于字符估算的软边界，最终 provider token 数仍可能有偏差。

默认 branch-summary 输出上限固定为 2048 token，提示词采用与压缩相同的工程状态章节，并额外强调分支中采用过的方案和未完成事项。实现位于 `packages/coding-agent/src/core/compaction/branch-summarization.ts:287-371`。

## 9. `/tree` 导航的状态提交

树导航入口位于 `packages/coding-agent/src/core/agent-session.ts:2832-3018`。目标 entry 的类型会改变落点：

- 目标是 user 或 custom entry：把其内容放回编辑器，新 leaf 设为目标的父节点；这样下一次提交可以替换原分支输入。
- 目标是 assistant 等其他 entry：新 leaf 就是目标自身。

若用户要求摘要，Pi 调用 `branchWithSummary(newLeafId, ...)`，把 branch-summary entry 接到上述新 leaf 后，并使它成为当前 leaf。没有摘要时只切换 leaf，不新增 entry。随后重新执行 `buildSessionContext()`，让 Agent 看到目标路径和新摘要。

这也说明 branch summary 的实际位置：它挂在导航后的路径上，用于新分支后续上下文，并非追加到已经离开的旧 leaf 后。`packages/coding-agent/test/agent-session-tree-navigation.test.ts:29-169` 分别验证 user 目标和 assistant 目标的父子关系。官方 `packages/coding-agent/docs/compaction.md` 的示意图可被理解为“总结旧 branch”，但持久化位置应以实现和测试为准。

导航是一段延迟提交事务：收集 entries、调用扩展和生成摘要时不改 leaf；摘要成功后才 branch；abort 或生成错误都保留原 leaf。回归测试 `packages/coding-agent/test/suite/regressions/3688-tree-cancel-compacting.test.ts:14-34` 还验证扩展取消后 `isCompacting` 会恢复为 `false`。

## 10. 扩展可以取消、改提示词或完全接管

### 压缩扩展

`session_before_compact` 收到 `preparation`、当前 branch、触发原因、是否准备重试和 AbortSignal。返回值有三种：

```ts
export interface SessionBeforeCompactResult {
	cancel?: boolean;
	compaction?: CompactionResult;
}
```

返回 `cancel` 会阻止提交；返回完整 `compaction` 会跳过默认 LLM 摘要；不返回则继续默认实现。类型位于 `packages/coding-agent/src/core/extensions/types.ts:1096-1099`。

### tree 扩展

`session_before_tree` 无论用户是否勾选 summarize 都会触发，所以扩展可以把导航本身视为 guard。它可以取消、直接给出 summary、覆盖自定义指令的追加/替换方式，或设置摘要 label。只有 `userWantsSummary` 为真时，扩展 summary 才会被采用；导航请求本身没有摘要时，hook 不能暗中强制写入摘要。入口见 `packages/coding-agent/src/core/agent-session.ts:2875-2965`。

### 多 handler 决策

ExtensionRunner 按扩展和注册顺序串行执行 handler。每个非空结果覆盖前一个结果；一旦出现 `cancel` 就立即返回。handler 抛错只产生 extension error，后面的 handler 与默认流程仍继续。核心循环位于 `packages/coding-agent/src/core/extensions/runner.ts:759-786`。

扩展提供的摘要和 details 被视为接管结果，AgentSession 不重新校验摘要质量、token 计数或 `firstKeptEntryId` 的语义。`packages/coding-agent/test/compaction-extensions.test.ts:125-330` 验证取消、自定义结果、事件时序、异常隔离和多 handler 覆盖。接管能力适合审计摘要、私有状态压缩或确定性摘要器，也把结构正确性的责任交给扩展作者。

## 11. 两种摘要的边界对照

| 维度 | compaction | branch summary |
|---|---|---|
| 解决的问题 | 当前 branch 太长 | 离开 branch 时保留旧路径信息 |
| 触发 | 手动、阈值、overflow | `/tree` navigation |
| 取材 | 上次保留边界到本次 cut 之前 | 旧 leaf 到共同祖先之前 |
| 原文保留 | `firstKeptEntryId` 之后继续原样投影 | 目标 branch 原路径原样保留 |
| 持久化节点 | `compaction` | `branchSummary` |
| 旧摘要处理 | 作为 `<previous-summary>` 更新 | 旧 compaction/branch summary 可作为普通摘要材料 |
| 默认输出预算 | reserve 的 80%，受模型上限约束 | 固定 2048 token |
| 失败语义 | 手动抛错；自动返回失败 | 返回 aborted/error，导航不提交 |
| 扩展入口 | `session_before_compact` | `session_before_tree` |

两者都保持 append-only session。摘要是新的事实记录，不是覆盖旧事实；模型上下文只选择其中一条投影视图。

## 12. 一条完整状态流

```text
一次 assistant message 完成
  → 持久化 message
  → _checkCompaction()
       选择有效 usage 或估算 trailing messages
       contextTokens > contextWindow - reserveTokens ?
  → prepareCompaction()
       找上一 compaction 与 firstKeptEntryId
       从末尾按 keepRecentTokens 找 cut point
       判断是否 split turn
       收集待摘要消息和累计文件状态
  → session_before_compact
       cancel / 扩展结果 / 默认生成
  → appendCompaction(summary, firstKeptEntryId, details)
  → buildSessionContext()：摘要 + 近期原文 + 后续消息
  → 必要时 agent.continue()

/tree 选择历史节点
  → old leaf 与 target 求共同祖先
  → 收集被放弃 entries
  → session_before_tree
       cancel / 改指令 / 扩展摘要 / 默认生成
  → 根据 target 类型决定 newLeafId
  → branchWithSummary(newLeafId, summary)
  → 重建 Agent 上下文
```

定位压缩问题时，应分别检查触发所用 token、cut point、`firstKeptEntryId`、摘要 entry 的父节点和扩展事件结果。只看摘要文本，无法判断近期原文是否切对；只看 session 文件末尾，也无法判断当前 leaf 投影的是哪条路径。
