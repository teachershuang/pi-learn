# 第 18 讲：从会话历史投影模型上下文

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

SessionManager 保存的是完整事件树，provider 接收的却是一条线性的 `Message[]`。两者之间不是一次简单的 `filter(message)`。Pi 先选活动分支和压缩窗口，得到仍含元数据节点的 `SessionEntry[]`；再把有上下文语义的 entry 投影成 `AgentMessage[]`；每次请求前，扩展还可以改写这份临时消息，最后才转换为 provider-neutral 的 `Message[]`。

```text
完整 JSONL 历史树
  │  buildSessionPath：leaf 反向追 parentId
  ▼
活动分支 SessionEntry[]
  │  buildContextEntries：应用最新 compaction 窗口
  ▼
上下文 entries（仍含 custom、model change 等非消息节点）
  │  sessionEntryToContextMessages
  ▼
AgentMessage[]（Agent 的语义消息）
  │  context 扩展：仅改本次请求副本
  │  convertToLlm：扩展角色转为基础消息
  ▼
Message[] + systemPrompt + tools
  │  provider 协议适配
  ▼
HTTP / WebSocket 请求体
```

`packages/coding-agent/docs/session-format.md` 把 `buildSessionContext()` 的结果简称为“给 LLM 的消息列表”。固定源码里的返回类型是 `AgentMessage[]`，还不是 provider 最终收到的 `Message[]`。这个中间层保留了 `custom`、`branchSummary`、`compactionSummary` 和 `bashExecution` 等 coding-agent 角色，供 Agent 与扩展处理。

## 1. 五种历史视图不能混用

### 解决的问题

同一份 session 同时服务于审计、树形导航、TUI 渲染、Agent 恢复和模型请求。如果这些调用都拿 `getEntries()` 的结果，废弃分支、标签和扩展私有状态会进入模型；如果一开始就只保存 LLM messages，分支与恢复信息又会丢失。

Pi 为此保留了五种视图：

| 视图 | 入口 | 包含什么 | 主要使用者 |
|---|---|---|---|
| 完整历史 | `getEntries()` | 文件中全部非 header entries，包含所有分支 | 审计、扩展恢复、session 列表与导出 |
| 活动分支 | `getBranch()` / `buildSessionPath()` | 当前 leaf 到根的祖先链 | tree navigation、设置恢复、压缩准备 |
| 上下文 entries | `buildContextEntries()` | 活动分支应用最新 compaction 后的 entry 窗口 | TUI 重绘、消息投影 |
| Agent 上下文 | `buildSessionContext().messages` | 可解释为 AgentMessage 的 entries | session 恢复、branch 切换、压缩后重建 |
| provider 上下文 | `Context.messages` | 基础 `user/assistant/toolResult` messages | `pi-ai` provider 适配器 |

完整历史和模型上下文的差异是设计结果，不是丢数据。一个旧 branch 可以继续留在 JSONL 中，label 和 `custom` entry 也能供界面或扩展使用，但当前模型请求不需要看到它们。

## 2. 第一道投影：只取活动 branch

### 源码入口

`buildSessionPath()` 位于 `packages/coding-agent/src/core/session-manager.ts:330-356`。它先建立 `id → entry` 索引，再从 leaf 沿 `parentId` 反向查找：

```ts
function buildSessionPath(
	entries: SessionEntry[],
	leafId?: string | null,
	byId?: Map<string, SessionEntry>,
): SessionEntry[] {
	const index = buildEntryIndex(entries, byId);
	let leaf: SessionEntry | undefined;
	if (leafId === null) return [];
	if (leafId) leaf = index.get(leafId);
	leaf ??= entries[entries.length - 1];
	if (!leaf) return [];

	const path: SessionEntry[] = [];
	let current: SessionEntry | undefined = leaf;
	while (current) {
		path.push(current);
		current = current.parentId ? index.get(current.parentId) : undefined;
	}
	path.reverse();
	return path;
}
```

文件物理顺序只负责提供默认 leaf；路径顺序由 `parentId` 决定。下面这份文件有四条消息，leaf 为 D 时只投影 A、B、D：

```text
文件顺序：A, B, C, D

A(user) → B(assistant) → C(user)       旧 branch
                      └→ D(user)       当前 leaf

完整历史：A, B, C, D
活动分支：A, B, D
```

`packages/coding-agent/test/session-manager/build-context.test.ts:211-229` 传入同一棵树的两个 leaf，断言末条消息分别是 branch A 和 branch B。`tree-traversal.test.ts:388-407` 则从当前 leaf 验证旧 sibling 不会进入上下文。

### 状态变化

正常 append 会把新 entry 设为 leaf；`branch(id)` 先把 leaf 移到旧节点，下一次 append 产生它的新孩子。`buildSessionContext()` 不修改树，也不移动 leaf，它只是读取当前选择。

树形导航完成后，`AgentSession.navigateTree()` 会重新构建上下文并替换 Agent 的活动消息，见 `packages/coding-agent/src/core/agent-session.ts:2969-3002`：

```ts
if (summaryText) {
	this.sessionManager.branchWithSummary(newLeafId, summaryText, summaryDetails, fromExtension);
} else if (newLeafId === null) {
	this.sessionManager.resetLeaf();
} else {
	this.sessionManager.branch(newLeafId);
}

const sessionContext = this.sessionManager.buildSessionContext();
this.agent.state.messages = sessionContext.messages;
```

替换的是 `agent.state.messages`，完整 JSONL entries 不变。后续 prompt 从新 branch 的消息继续，旧 branch 仍可导航回来。

### 失败路径

`buildSessionPath()` 采取宽松恢复策略：

- `leafId === null` 明确返回空路径；
- 未传 leaf 时使用 entries 最后一项；
- 传入不存在的 leaf id 时也退回最后一项，而不是报错；
- parent 缺失时在断链处停止，不会按 timestamp 猜父节点。

`build-context.test.ts:288-305` 固定了后两项行为。公开 API 的 `branch()` 会先校验目标 id，因此正常导航不会传入不存在的 leaf；宽松 fallback 主要保护直接调用投影函数和不完整历史文件。代价是损坏文件可能“能够恢复”，但恢复出的上下文少于原历史。

## 3. 第二道投影：latest compaction 划定消息窗口

### 解决的问题

活动 branch 仍可能超过模型上下文窗口。Compaction 不删除旧 entries，而是添加一个摘要节点，并用 `firstKeptEntryId` 指明哪些近期节点继续逐条保留。`buildContextEntries()` 负责解释这份声明；摘要怎样生成、cut point 如何选择留到第 19 讲。

实现位于 `session-manager.ts:414-450`：

```ts
const path = buildSessionPath(entries, leafId, byId);
let compaction: CompactionEntry | null = null;
for (const entry of path) {
	if (entry.type === "compaction") compaction = entry;
}

if (!compaction) return path;

const compactionIdx = path.findIndex((entry) => entry.id === compaction.id);
const contextEntries: SessionEntry[] = [compaction];
let foundFirstKept = false;
for (let i = 0; i < compactionIdx; i++) {
	const entry = path[i];
	if (entry.id === compaction.firstKeptEntryId) foundFirstKept = true;
	if (foundFirstKept) contextEntries.push(entry);
}
contextEntries.push(...path.slice(compactionIdx + 1));
return contextEntries;
```

结果顺序不是原路径的子数组，而是：

```text
[最新 compaction 摘要]
+ [firstKeptEntryId 到 compaction 之前的 entries]
+ [compaction 之后的 entries]
```

例如原路径为 `1 → 2 → 3 → 4 → compact(5, keep=3) → 6 → 7`，上下文 entries 是 `5, 3, 4, 6, 7`。1、2 仍在审计历史中，其语义改由 5 的 summary 表达。

只使用活动路径上的最后一个 compaction。另一个 branch 的 compaction 与当前路径无关；同一路径有多次 compaction 时，实现假定最新摘要已经包含此前需要保留的语义，因此旧摘要不再单独加入。`build-context.test.ts:126-177` 覆盖摘要顺序、从首条保留和多次压缩。

`buildContextEntries()` 不会立即删除非消息节点。测试在 `build-context.test.ts:179-194` 构造了保留区间内和 compaction 之后的 `custom` entries，返回 id 包含它们。原因是这份 entry 视图还供 TUI 渲染；是否成为 AgentMessage 由下一步决定。

### 边界与失败结果

如果 `firstKeptEntryId` 不在 compaction 之前的活动路径里，循环始终找不到保留起点，结果只剩 compaction 和它之后的 entries。实现没有报错或回退。这能避免把已摘要的旧历史重复发送，但损坏的引用会静默丢掉本应保留的近期消息。

Compaction 也不会在这里检查 assistant tool call 与 tool result 是否成对。正常压缩的 cut point 负责避开不安全边界；手工编辑 JSONL 或构造错误 compaction 时，投影层不会修复消息协议。

## 4. model 与 thinking level 走旁路，不变成消息

### 解决的问题

模型选择和思考级别影响下一次请求配置，却不是对话正文。若把它们伪装成 user message，模型会把运行参数当自然语言；若只扫描压缩后的 entry 窗口，压缩点之前的最后一次设置又会消失。

`getSessionContextSettings()` 因此扫描完整活动路径，而不是 `buildContextEntries()` 的压缩窗口，见 `session-manager.ts:358-373`：

```ts
let thinkingLevel = "off";
let model: { provider: string; modelId: string } | null = null;

for (const entry of path) {
	if (entry.type === "thinking_level_change") {
		thinkingLevel = entry.thinkingLevel;
	} else if (entry.type === "model_change") {
		model = { provider: entry.provider, modelId: entry.modelId };
	} else if (entry.type === "message" && entry.message.role === "assistant") {
		model = { provider: entry.message.provider, modelId: entry.message.model };
	}
}
```

规则是路径顺序中的最后一次赋值生效：

- thinking level 只由 `thinking_level_change` 更新，缺省是 `off`；
- model 可以来自 `model_change`，也可以从 assistant message 的 provider/model 反推；
- 较晚的 assistant message 会覆盖之前的 model change。

assistant 反推为旧 session 和不完整变更记录提供兜底。`build-context.test.ts:97-123` 特意在 model change 后放入另一模型的 assistant message，最终选择 assistant 实际使用的模型。

`buildSessionContext()` 返回 `{ messages, thinkingLevel, model }`，只有 `messages` 是 `AgentMessage[]`。SDK 在恢复 session 时才尝试应用另外两项，入口在 `packages/coding-agent/src/core/sdk.ts:182-238`：session model 必须仍能从 ModelRuntime 找到且具备认证，否则使用当前可用模型；thinking level 还会按当前模型能力 clamp。

运行时没有完全消费这三个返回值。`navigateTree()` 重建 branch 时只赋值 `agent.state.messages`，没有把 `sessionContext.model` 和 `thinkingLevel` 写回 Agent。因此，同文件内切换历史 branch 会换消息，但保留导航前的当前模型和思考级别。Model/thinking 的历史恢复发生在 `createAgentSession()` 初始化阶段；如果导航后没有追加新 entry，leaf 移动本身也不会落盘，重新 resume 仍以文件最后一条 entry 为 leaf。这个行为可由 `agent-session.ts:2999-3002`、`sdk.ts:182-238` 和 `session-manager.ts:889-908` 直接核对。

## 5. 第三道投影：SessionEntry 变成 AgentMessage

### 转换表

`sessionEntryToContextMessages()` 位于 `session-manager.ts:379-404`。每个入选 entry 返回零条或一条 AgentMessage：

| SessionEntry | AgentMessage 结果 | 保留或丢弃的内容 |
|---|---|---|
| `message` | 原 `entry.message` | 标准消息、bashExecution 等原样保留 |
| `custom_message` | role=`custom` | content、display、details、customType 均保留 |
| `branch_summary` | role=`branchSummary` | summary、fromId、timestamp |
| `compaction` | role=`compactionSummary` | summary、tokensBefore、timestamp |
| `custom` | 无 | 扩展状态不进 Agent 上下文 |
| `model_change` / `thinking_level_change` | 无 | 已通过返回值旁路恢复 |
| `label` / `session_info` | 无 | 只影响导航和界面元数据 |

核心分派代码如下：

```ts
if (entry.type === "message") {
	const message = entry.message;
	if (
		(message.role === "user" || message.role === "assistant" || message.role === "toolResult") &&
		message.content == null
	) {
		return [{ ...message, content: [] }];
	}
	return [message];
}
if (entry.type === "custom_message") {
	return [createCustomMessage(entry.customType, entry.content ?? [], entry.display, entry.details, entry.timestamp)];
}
if (entry.type === "branch_summary" && entry.summary) {
	return [createBranchSummaryMessage(entry.summary, entry.fromId, entry.timestamp)];
}
if (entry.type === "compaction") {
	return [createCompactionSummaryMessage(entry.summary, entry.tokensBefore, entry.timestamp)];
}
return [];
```

Session 文件采用宽松 JSON 解析，没有运行时 schema 校验。投影时只为 user、assistant、toolResult 和 custom message 的 `null`/缺失 content 补空数组，防止常见旧数据在后续转换中崩溃。`packages/coding-agent/test/suite/lax-message-content.test.ts:105-160` 固定了这项兼容行为。其他字段损坏仍可能继续传到 Agent 或 provider 适配器。

## 6. tool result 为什么能保持调用顺序

### 写入与投影

Tool result 没有专用 SessionEntry 类型，它是 `message` entry 内的一种 `AgentMessage`。Agent loop 先完成 assistant tool-use message，再执行工具并发出 toolResult 的 `message_end`。`AgentSession._handleAgentEvent()` 在收到 `message_end` 后顺序持久化，入口在 `packages/coding-agent/src/core/agent-session.ts:603-622`：

```ts
if (event.type === "message_end") {
	if (event.message.role === "custom") {
		this.sessionManager.appendCustomMessageEntry(
			event.message.customType,
			event.message.content,
			event.message.display,
			event.message.details,
		);
	} else if (
		event.message.role === "user" ||
		event.message.role === "assistant" ||
		event.message.role === "toolResult"
	) {
		this.sessionManager.appendMessage(event.message);
	}
}
```

因此正常工具轮次的 branch 顺序是：

```text
user
→ assistant（含 toolCall）
→ toolResult 1
→ toolResult 2
→ assistant（继续回答或再次调用工具）
```

`packages/coding-agent/test/suite/regressions/1717-2113-agent-session-event-settlement.test.ts:29-65` 让 assistant 的扩展 `message_end` handler 主动等待，再验证两条 toolResult 仍排在 tool-use assistant 之后。投影阶段对 toolResult 不改角色、不改 `toolCallId`，`convertToLlm()` 也直接透传。

错误工具结果同样保留：`isError` 和错误正文是模型决定是否修正调用所需的上下文。SessionManager 不会因为 `isError=true` 把它从审计历史或活动消息中删除。

压缩或损坏的 parent 链若让 toolResult 与对应 toolCall 分离，当前投影函数不会检测。最终是否接受取决于 provider 协议适配和服务端约束。

## 7. custom、summary 与 bash 在第二次转换时才落到 user

`AgentMessage[]` 允许 coding-agent 保留基础 LLM 协议没有的语义角色。真正请求模型前，`packages/coding-agent/src/core/messages.ts:148-194` 的 `convertToLlm()` 再做一次转换：

```ts
switch (m.role) {
	case "bashExecution":
		if (m.excludeFromContext) return undefined;
		return {
			role: "user",
			content: [{ type: "text", text: bashExecutionToText(m) }],
			timestamp: m.timestamp,
		};
	case "custom":
		return {
			role: "user",
			content: typeof m.content === "string" ? [{ type: "text", text: m.content }] : m.content,
			timestamp: m.timestamp,
		};
	case "branchSummary":
		return { role: "user", content: [{ type: "text", text: BRANCH_SUMMARY_PREFIX + m.summary + BRANCH_SUMMARY_SUFFIX }], timestamp: m.timestamp };
	case "compactionSummary":
		return { role: "user", content: [{ type: "text", text: COMPACTION_SUMMARY_PREFIX + m.summary + COMPACTION_SUMMARY_SUFFIX }], timestamp: m.timestamp };
}
```

具体边界如下：

- 入选活动上下文后，`custom_message.display=false` 只让 TUI 隐藏该消息，模型仍会收到其 content；
- custom 的 `customType`、`display` 和 `details` 不进入基础 Message，只剩 user content 与 timestamp；
- branch summary 和 compaction summary 都包装成带固定英文前后缀的 user text；
- 单叹号 bash 结果转换为 user text，带 `excludeFromContext` 的双叹号结果在这里过滤；
- 标准 user、assistant、toolResult 原样通过。

`packages/coding-agent/test/suite/agent-session-bash-persistence.test.ts:135-175` 构造 custom → user → assistant tool call → toolResult → assistant，断言 session entries 与 Agent 角色顺序一致。它说明 custom message 是可持久化的上下文消息，不是普通 `custom` 状态 entry。

这种两层类型的好处是 TUI 和扩展能识别“这是压缩摘要”或“这是扩展注入”，而 provider 只面对三种基础角色。代价是审计文件中的 role 与线上请求 role 不总是一致；排查 prompt 时只看 JSONL 仍不够，还要考虑 `convertToLlm()` 的包装和过滤。

## 8. 每次模型调用前还有一次临时投影

### 新 prompt 先进入运行时消息

恢复 session 时，SDK 把 `buildSessionContext().messages` 赋给 `agent.state.messages`，见 `packages/coding-agent/src/core/sdk.ts:357-369`。新一轮 prompt 不是重新扫描 JSONL：Agent 先复制当前状态，再把本轮 prompts 追加到副本。`packages/agent/src/agent-loop.ts:95-116` 同时发出 prompt 的 `message_end`，AgentSession 因而会在模型请求前把 user/custom prompt 追加到 session。

```ts
const currentContext: AgentContext = {
	...context,
	messages: [...context.messages, ...prompts],
};

await emit({ type: "agent_start" });
await emit({ type: "turn_start" });
for (const prompt of prompts) {
	await emit({ type: "message_start", message: prompt });
	await emit({ type: "message_end", message: prompt });
}
```

请求发出前有两份同步状态：当前 Agent 上下文已含新 prompt，SessionManager 也已记录它。若 provider 请求随后失败，user message 仍在历史中。`Agent.handleRunFailure()` 还会生成 stopReason 为 `error` 或 `aborted` 的 assistant message，按正常 `message_end` 路径持久化，见 `packages/agent/src/agent.ts:494-509`。

### context 扩展只改本次请求

每次 `streamAssistantResponse()` 都按固定顺序处理消息，入口在 `agent-loop.ts:281-302`：

```ts
let messages = context.messages;
if (config.transformContext) {
	messages = await config.transformContext(messages, signal);
}

const llmMessages = await config.convertToLlm(messages);
const llmContext: Context = {
	systemPrompt: context.systemPrompt,
	messages: llmMessages,
	tools: context.tools,
};
```

coding-agent 把 `transformContext` 绑定到扩展的 `context` event，见 `packages/coding-agent/src/core/sdk.ts:344-349`。`ExtensionRunner.emitContext()` 在 `packages/coding-agent/src/core/extensions/runner.ts:937-967` 对输入执行 `structuredClone`，再按扩展和 handler 注册顺序串行传递修改结果。

这份结果只存在于本次 provider 调用的局部变量中，不会回写 `agent.state.messages`，也不会追加 JSONL。`packages/coding-agent/test/suite/agent-session-model-extension.test.ts:204-240` 把 user content 从 `original` 改成 `rewritten`，provider 观察到后者，Agent 内仍保存前者。Session 记录用户原话，Agent state 保存语义历史，线上请求还可能经过扩展改写。

某个 context handler 抛错时，runner 发出 extension error，保留该 handler 之前的 `currentMessages`，继续执行后续 handler。实现不会让单个扩展异常中止模型调用。低层 `AgentLoopConfig` 对任意 `transformContext` 和 `convertToLlm` 都规定“不应抛错”；coding-agent 的 runner 实现了 context handler 级隔离，但嵌入式 SDK 若注入自己的转换函数，仍要自己遵守这项契约，见 `packages/agent/src/types.ts:140-191`。

### 图片策略在基础消息转换后执行

SDK 给 Agent 的不是裸 `convertToLlm`，而是 `convertToLlmWithBlockImages`。当 `blockImages` 开启时，它把 user 和 toolResult content 中的 image 替换成 `Image reading is disabled.`，并合并连续占位文本，入口在 `sdk.ts:250-285`。

custom message 先变成 user message，所以其中的图片也受这项策略控制。原始 session entry 和 AgentMessage 不会被改写；只有当前 provider 请求看见占位文本。

## 9. SessionContext 还不是完整的模型 Context

`buildSessionContext()` 的返回值只有历史消息、模型提示和 thinking level。真正交给 `pi-ai` 的 `Context` 还包含：

- `systemPrompt`：由当前 cwd 的资源、活动工具和配置重新生成；
- `tools`：当前 Agent 的工具定义与 JSON Schema；
- `messages`：本课追踪的投影结果，经 request-time 扩展和基础角色转换处理。

system prompt 和 tools 不在 JSONL message 投影中。Resume 同一 session 时，如果项目 `AGENTS.md`、skills、扩展或工具配置变了，历史 messages 可以相同，实际请求仍会不同。Session 文件是对话与会话事件的审计记录，不是 provider 请求体快照。

Provider 适配器还会执行协议级消息转换，例如合并角色、处理 thinking、编码 tool call、应用缓存标记。那些规则属于第 10 讲的协议层；`buildSessionContext()` 不知道 Anthropic、OpenAI 或 Google 的具体请求格式。

## 10. 可观察的失败边界

投影链上的失败结果可以按层定位：

| 位置 | 输入问题 | 当前处理 | 留下的结果 |
|---|---|---|---|
| branch 选择 | leaf 不存在 | 退回物理最后 entry | 可能继续了错误 branch，不抛错 |
| parent 追踪 | 父 entry 缺失 | 在 orphan 处停止 | 更早历史不进上下文 |
| compaction 窗口 | `firstKeptEntryId` 缺失 | 只保留摘要与压缩后 entries | 近期保留区静默丢失 |
| entry 转 AgentMessage | 标准消息 content 为 null | 替换成 `[]` | role 和其他字段保留 |
| entry 转 AgentMessage | 其他字段格式错误 | 通常原样通过 | 可能在后续转换或 provider 失败 |
| context 扩展 | handler 抛错 | 记录 extension error，继续 | 使用抛错前的消息版本 |
| convertToLlm | 自定义转换抛错 | 中断当前低层 turn | Agent wrapper 生成 error assistant；coding-agent 会将其持久化 |
| provider 协议 | tool call/result 不配对等 | 由适配器或服务端决定 | provider error，session 原历史仍在 |

Pi 在 session 层偏向“保住可读部分”，在请求层要求转换函数提供安全结果。两者适合处理的故障不同：JSONL 损坏可以局部恢复，provider 请求却必须满足严格协议。

## 11. 一条完整状态流

```text
resume / tree navigation / compaction 完成
  → SessionManager.buildSessionContext()
       完整 entries 不变
       active branch = leaf 的祖先链
       context entries = latest compaction + kept + after
       messages = 可投影 entries 的 AgentMessage[]
       model/thinking = 完整 branch 上最后一次设置
  → agent.state.messages = messages

新 prompt
  → Agent 创建 state snapshot，并追加 prompt
  → message_end 让 SessionManager 先持久化 prompt
  → 每次模型调用：
       context extension 临时改 AgentMessage[]
       convertToLlm 转为 user/assistant/toolResult
       blockImages 按当前设置替换图片
       systemPrompt + messages + tools 组成 pi-ai Context
       provider adapter 生成具体协议请求
  → assistant / toolResult 的 message_end 顺序追加 session
```

调试“模型为什么看到了这段内容”时，至少要分别检查活动 leaf、最新 compaction、entry-to-message 转换和 context 扩展。只查看 JSONL 文件末尾，无法还原真正发送给 provider 的消息。
