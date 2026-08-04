# 第 02 讲：消息、上下文与状态不是同一个对象

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

“聊天记录”这个词在 Pi 源码里太含糊。磁盘上保存的是一棵 session entry 树，Agent 内存里保存的是当前活动 transcript，某次模型请求拿到的只是经过分支选择、压缩、扩展和消息转换后的快照，TUI 又有自己的组件状态。这几份数据经常内容相似，但所有者、修改时机和恢复能力完全不同。

先把一条消息的路径展开：

```text
SessionEntry[]                    完整、持久化、带树关系
      │ buildSessionContext()
      ▼
SessionContext.messages          当前分支的 AgentMessage[] 投影
      │ 恢复或替换 Agent transcript
      ▼
AgentState.messages              当前活动 transcript，加运行时状态
      │ createContextSnapshot()
      ▼
AgentContext                     本次低层 run 的顶层数组快照
      │ transformContext
      ▼
AgentMessage[]                   扩展可临时修改的请求视图
      │ convertToLlm
      ▼
pi-ai Context.messages           provider 可接受的 Message[]
      │ provider stream
      ▼
AssistantMessageEvent            增量事件
      │ AgentEvent / AgentSessionEvent
      ├── 更新 AgentState
      ├── 完成消息写成 SessionEntry
      └── 刷新 TUI、JSON 或 RPC 输出
```

这条链上没有一个对象能单独代表“全部会话状态”。

## 1. `Message` 是模型协议层允许发送的消息

### 解决的问题

Anthropic、OpenAI、Google 等 provider 的原始消息格式不同。agent-loop 需要一套与具体 API 无关的输入输出，provider 适配器再把它转成各自协议。

### 源码入口与关键代码

`packages/ai/src/types.ts:382-454` 定义了三种 `Message`，以及真正传给 `streamSimple()` 的 `Context`：

```ts
export interface UserMessage {
	role: "user";
	content: string | (TextContent | ImageContent)[];
	timestamp: number;
}

export interface AssistantMessage {
	role: "assistant";
	content: (TextContent | ThinkingContent | ToolCall)[];
	api: Api;
	provider: ProviderId;
	model: string;
	// ...response id、diagnostics 等可选元数据
	usage: Usage;
	stopReason: StopReason;
	errorMessage?: string;
	timestamp: number;
}

export interface ToolResultMessage<TDetails = any> {
	role: "toolResult";
	toolCallId: string;
	toolName: string;
	content: (TextContent | ImageContent)[];
	details?: TDetails;
	isError: boolean;
	timestamp: number;
}

export type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

`Context` 只包含 system prompt、`Message[]` 和工具 schema。它不是一个正在变化的 Agent，也没有 session id、重试次数、当前 leaf 或界面组件。

### 运行流程与状态变化

user message 可以是纯字符串，也可以是文本和图片块。assistant message 除文本外，还能保存 thinking 和 tool call，并记录 provider、模型、用量、停止原因与错误。tool result 用 `toolCallId` 与先前的 tool call 配对。

这些对象是 provider 中立的请求协议。具体适配器仍可能因上游限制再次调整它们，例如重排 tool result、补占位消息或处理 thinking signature；那些变化属于 provider 请求转换，不会反向改写 session entry。

### 失败路径与设计取舍

`StopReason` 把正常结束、长度截断、工具调用、错误和中止放在同一结果类型里。模型失败因此可以成为一条 assistant message，而不必只靠异常传递。

统一类型丢掉了供应商协议中的部分原生形状，换来 agent-loop 的复用。Pi 对必须跨轮保留的供应商数据留了专用字段，例如 `textSignature`、`thinkingSignature`、`thoughtSignature` 和 `responseId`。新增 provider 时，不能简单把所有陌生字段删掉；要判断它们是否参与下一轮请求。

## 2. `AgentMessage` 扩大了应用能够保存和处理的消息集合

### 解决的问题

编码代理除了 user、assistant 和 tool result，还要表示 `!` 命令输出、扩展注入内容、分支摘要和压缩摘要。这些内容可能要显示、持久化或参与上下文，但 provider 不认识这些 role。

### 源码入口与关键代码

`packages/agent/src/types.ts:291-314` 把 `AgentMessage` 定义为基础 `Message` 加上可声明合并的自定义消息。coding-agent 在 `packages/coding-agent/src/core/messages.ts:69-77` 注册自己的四种类型：

```ts
declare module "@earendil-works/pi-agent-core" {
	interface CustomAgentMessages {
		bashExecution: BashExecutionMessage;
		custom: CustomMessage;
		branchSummary: BranchSummaryMessage;
		compactionSummary: CompactionSummaryMessage;
	}
}
```

自定义 role 不能直接进入 `pi-ai Context`。`convertToLlm()` 决定如何降维：

```ts
return messages.map((m): Message | undefined => {
	switch (m.role) {
		case "bashExecution":
			if (m.excludeFromContext) return undefined;
			return {
				role: "user",
				content: [{ type: "text", text: bashExecutionToText(m) }],
				timestamp: m.timestamp,
			};
		case "custom": {
			const content = typeof m.content === "string" ? [{ type: "text" as const, text: m.content }] : m.content;
			return { role: "user", content, timestamp: m.timestamp };
		}
		case "branchSummary":
			return {
				role: "user",
				content: [{ type: "text", text: BRANCH_SUMMARY_PREFIX + m.summary + BRANCH_SUMMARY_SUFFIX }],
				timestamp: m.timestamp,
			};
		case "compactionSummary":
			return {
				role: "user",
				content: [{ type: "text", text: COMPACTION_SUMMARY_PREFIX + m.summary + COMPACTION_SUMMARY_SUFFIX }],
				timestamp: m.timestamp,
			};
		case "user":
		case "assistant":
		case "toolResult":
			return m;
	}
});
```

代码位置：`packages/coding-agent/src/core/messages.ts:148-194`。片段省略了末尾的穷尽检查和 `undefined` 过滤。

### 运行流程与状态变化

`AgentState.messages` 可以同时装基础消息和 coding-agent 自定义消息。进入模型请求前：

- `!!` 产生的 bash message 被过滤；普通 `!` 输出转成 user message；
- custom message 转成 user message，`details` 不发送；
- branch/compaction summary 加上固定说明后转成 user message；
- 三种基础 `Message` 原样保留。

转换只生成请求视图。一个 `role: "custom"` 的 AgentMessage 不会因为被转换成 user message，就在内存或 session 文件里改成 user。

### 失败路径与设计取舍

自定义消息若没有对应的转换逻辑，provider 无法接收。低层 Agent 的默认转换器会直接过滤所有非标准 role；coding-agent 创建 Agent 时注入自己的 `convertToLlm()`，才保留上述语义。

把应用消息和模型消息分开，允许 UI 保存额外信息，也能控制哪些内容发送给模型。代价是新增一种 custom message 时必须同时决定四件事：是否显示、是否持久化、是否进入模型上下文，以及进入时转换成什么。

## 3. session entry 保存历史，而不是直接充当模型上下文

### 解决的问题

持久化需要表达分支、模型切换、thinking level、标签、压缩和扩展状态。把它们全部伪装成聊天消息会丢失结构，也会把不该发送的内部数据暴露给模型。

### 源码入口与关键代码

`packages/coding-agent/src/core/session-manager.ts:46-149` 给每个 entry 加上独立的树节点信息：

```ts
export interface SessionEntryBase {
	type: string;
	id: string;
	parentId: string | null;
	timestamp: string;
}

export interface SessionMessageEntry extends SessionEntryBase {
	type: "message";
	message: AgentMessage;
}

export type SessionEntry =
	| SessionMessageEntry
	| ThinkingLevelChangeEntry
	| ModelChangeEntry
	| CompactionEntry
	| BranchSummaryEntry
	| CustomEntry
	| CustomMessageEntry
	| LabelEntry
	| SessionInfoEntry;
```

message 自己的 `timestamp` 是消息产生时间；entry 的 ISO `timestamp` 是写入会话树的时间。两者用途不同。

### 运行流程与状态变化

`SessionManager` 保存所有 entry，并用 `id/parentId` 建树。`leafId` 指向当前工作位置，新 entry 成为当前 leaf 的子节点。切回旧节点再追加，不需要删除后来产生的历史，只会形成另一条分支。

entry 是否进入模型上下文由投影函数决定：

| entry | 保存在会话树 | 投影为 `AgentMessage` | 发送给模型 |
| --- | --- | --- | --- |
| `message` | 是 | 原 message | 经过 `convertToLlm()` |
| `custom_message` | 是 | `role: "custom"` | 转成 user message |
| `compaction` | 是 | `compactionSummary` | 转成摘要 user message |
| `branch_summary` | 是 | `branchSummary` | 转成摘要 user message |
| `custom` | 是 | 否 | 否 |
| `model_change` / `thinking_level_change` | 是 | 否 | 不作为消息；恢复配置 |
| `label` / `session_info` | 是 | 否 | 否 |

普通 `custom` entry 用于扩展恢复自己的状态；`custom_message` 才表示扩展希望注入 transcript 的内容。名字接近，但边界相反。

### 失败路径与设计取舍

JSONL 加载会跳过无法解析的行；第一条有效记录若不是 session header，则整份文件视为无效。entry 内容不会在解析时做完整 schema 校验，因此 `sessionEntryToContextMessages()` 会把旧文件、手改文件中 null 或缺失的消息 `content` 归一化为空数组。

宽松加载有利于恢复部分历史，但不能凭空修复树。某个 `parentId` 指向不存在的节点时，活动路径会从断点处开始，丢失更早的上下文。测试明确保留了这种“尽量可用”的行为，没有把文件损坏伪装成完整恢复。

## 4. `buildSessionContext()` 是历史到活动 transcript 的投影

### 解决的问题

完整 session 树可能同时包含多条分支和已经被摘要覆盖的旧消息。恢复 Agent 时不能把整个 JSONL 按文件顺序塞进模型，否则废弃分支也会进入上下文，压缩也失去作用。

### 源码入口与关键代码

`packages/coding-agent/src/core/session-manager.ts:330-465` 先从 leaf 沿 `parentId` 回到根，再处理最新 compaction，最后把选中的 entry 投影为消息：

```ts
function buildSessionPath(
	entries: SessionEntry[],
	leafId?: string | null,
	byId?: Map<string, SessionEntry>,
): SessionEntry[] {
	const index = buildEntryIndex(entries, byId);
	let leaf: SessionEntry | undefined;
	if (leafId === null) {
		return [];
	}
	if (leafId) {
		leaf = index.get(leafId);
	}
	leaf ??= entries[entries.length - 1];
	if (!leaf) {
		return [];
	}

	const path: SessionEntry[] = [];
	let current: SessionEntry | undefined = leaf;
	while (current) {
		path.push(current);
		current = current.parentId ? index.get(current.parentId) : undefined;
	}
	path.reverse();
	return path;
}

export function buildSessionContext(
	entries: SessionEntry[],
	leafId?: string | null,
	byId?: Map<string, SessionEntry>,
): SessionContext {
	const path = buildSessionPath(entries, leafId, byId);
	const { thinkingLevel, model } = getSessionContextSettings(path);
	const messages = buildContextEntries(entries, leafId, byId)
		.flatMap(sessionEntryToContextMessages);
	return { messages, thinkingLevel, model };
}
```

### 运行流程与状态变化

`SessionContext` 这个名字容易造成误会。它不是 `pi-ai Context`，而是会话恢复结果：`AgentMessage[]`、thinking level 和最近模型。它没有 system prompt 和工具。

创建 Agent 时，`createAgentSession()` 把 `sessionManager.buildSessionContext().messages` 赋给 `agent.state.messages`。压缩完成或切换活动树路径后，也会重新投影并替换 Agent transcript。于是：

- session entries 继续保留旧分支和压缩前历史；
- AgentState 只保留当前活动视图；
- provider 在下一步还要接收更窄的 `Message[]`。

### 失败路径与设计取舍

调用者传入不存在的 `leafId` 时，当前实现回退到 entries 中最后一个节点；显式 `leafId: null` 才返回空路径。这个回退提高了旧调用方的容错，但如果上层误传 id，可能悄悄得到另一条上下文。修改导航代码时，需要结合测试决定是否保留该兼容行为。

压缩在这里表现为“发送视图变化”，不是删除历史。最新 compaction entry、它声明保留的消息以及 compaction 之后的 entry 组成活动上下文。具体 cut point 和摘要生成放到第 19 讲。

## 5. `AgentState` 是活的运行时对象，`AgentContext` 是一次 run 的快照

### 解决的问题

模型选择、工具集合和 transcript 会跨 turn 变化；一次已经开始的 run 又需要相对稳定的输入。流式 partial message、执行中的 tool call 和错误提示还必须能被界面即时读取。

### 源码入口与关键代码

`packages/agent/src/types.ts:322-406` 把长期配置、活动 transcript 和短暂运行状态放进 `AgentState`，而 `AgentContext` 只保留低层循环需要的三项：

```ts
export interface AgentState {
	systemPrompt: string;
	model: Model<any>;
	thinkingLevel: ThinkingLevel;
	set tools(tools: AgentTool<any>[]);
	get tools(): AgentTool<any>[];
	set messages(messages: AgentMessage[]);
	get messages(): AgentMessage[];
	readonly isStreaming: boolean;
	readonly streamingMessage?: AgentMessage;
	readonly pendingToolCalls: ReadonlySet<string>;
	readonly errorMessage?: string;
}

export interface AgentContext {
	systemPrompt: string;
	messages: AgentMessage[];
	tools?: AgentTool<any>[];
}
```

`Agent.createContextSnapshot()` 对 messages 和 tools 做顶层数组复制。`createMutableAgentState()` 在外部重新赋值数组时也会复制数组，但不会深拷贝每个 message 和 tool 对象。

### 运行流程与状态变化

run 开始前，Agent 从 state 创建 `AgentContext`。循环追加本轮 user、assistant 和 tool result 时，先更新这次 run 的 context，再通过事件归约回 AgentState。运行中 `streamingMessage` 指向 partial assistant；`message_end` 后它被清空，final message 进入 `state.messages`。

thinking level 不进入 `AgentContext`，而是进入 `AgentLoopConfig`；session id、传输方式、队列和 abort controller 也有各自所有者。判断状态位置要看谁负责修改它，不能因为都影响下一次请求就塞进 `Context`。

Agent 与 AgentSession 甚至有两个不同边界的 `isStreaming`：

- `AgentState.isStreaming` 覆盖当前一次低层 prompt 或 continue；
- `AgentSession.isStreaming` 来自 `_isAgentRunActive`，跨越自动重试、压缩和后续 continue，直到 `agent_settled`。

### 失败路径与设计取舍

顶层数组复制可以防止调用者在赋值后通过原数组增删 transcript，却没有提供不可变消息树。源码会在少数扩展替换场景中原地更新完成消息，以保持 AgentState、事件和随后持久化对象一致。把 `AgentState` 当成 immutable snapshot 会读错这部分代码。

每次 run 使用快照，避免中途替换整个 state 数组直接改变当前迭代；AgentSession 的 `prepareNextTurn` 又能在工具轮之间显式刷新 system prompt、工具、模型和 thinking level。它在稳定性和运行中配置更新之间留了一个明确的同步点。

## 6. context hook 改请求视图，不改历史

### 解决的问题

扩展可能要在每次模型调用前删减消息、插入临时说明或按外部状态重写上下文。若这些临时变化自动持久化，会话审计历史会被请求策略污染。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:288-302` 固定了转换顺序：

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

coding-agent 注入的 `transformContext` 调用 `ExtensionRunner.emitContext()`。`packages/coding-agent/src/core/extensions/runner.ts:937-966` 先 `structuredClone(messages)`，再按扩展加载顺序串行传递修改结果。

### 运行流程与状态变化

某个 context handler 返回新的 `messages` 后，下一个 handler 看到修改后的结果；所有 handler 结束后才执行 `convertToLlm()`。原来的 `AgentState.messages` 和 session entries 不变。下一次模型调用会重新执行整套 hook，不会默认继承上次临时结果。

这也解释了压缩和 context hook 的区别：压缩写入 session entry，并重新构建 AgentState；context hook 只影响眼前这次 provider request。

### 失败路径与设计取舍

context handler 抛错时，ExtensionRunner 报告 extension error，然后继续处理剩余 handler，当前消息视图仍可发送。这里选择的是扩展故障降级，而不是阻断模型调用。

handler 之间串行执行，结果确定，却让后加载的扩展依赖前面扩展的输出。扩展顺序因此是行为的一部分。若改成并行，就必须另行定义冲突合并规则。

## 7. 事件描述变化，不是另一份 transcript

### 解决的问题

provider 按 token 和内容块返回增量，Agent、会话和界面需要在不同粒度上消费。把每个 delta 都追加成 message，会让 transcript 膨胀，也无法区分 partial 与 final。

### 源码入口与关键代码

`packages/ai/src/types.ts:456-476` 定义 provider 中立的 `AssistantMessageEvent`：`start` 建立 partial message，text/thinking/toolcall 事件更新内容块，`done` 或 `error` 带最终 assistant message 终止流。

Agent 再把 provider 事件翻译成较粗的生命周期事件。`packages/agent/src/types.ts:415-430` 中：

```ts
export type AgentEvent =
	| { type: "agent_start" }
	| { type: "agent_end"; messages: AgentMessage[] }
	| { type: "turn_start" }
	| { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
	| { type: "message_start"; message: AgentMessage }
	| {
			type: "message_update";
			message: AgentMessage;
			assistantMessageEvent: AssistantMessageEvent;
	  }
	| { type: "message_end"; message: AgentMessage }
	| { type: "tool_execution_start"; toolCallId: string; toolName: string; args: any }
	| { type: "tool_execution_update"; toolCallId: string; toolName: string; args: any; partialResult: any }
	| { type: "tool_execution_end"; toolCallId: string; toolName: string; result: any; isError: boolean };
```

### 运行流程与状态变化

`message_update` 携带当前 partial assistant 的快照和原始统一流事件，TUI 用前者重绘整条消息，也能用后者判断本次变化属于 text、thinking 还是 tool call。`message_end` 才把 final message 加入 AgentState，并触发 SessionManager 持久化。

`AgentSessionEvent` 在 AgentEvent 之上增加 queue、compaction、retry、session info 和 `agent_settled`。这些事件描述应用级进展，不会自动成为模型消息。TUI 收到事件后创建或更新组件；组件树仍是界面派生状态，不属于 AgentState。

### 失败路径与设计取舍

进程在 `message_update` 与 `message_end` 之间被强制结束时，界面上可能已经出现 partial 文本，但 session 中没有对应 final assistant message。Pi 没有把每个 delta 写进 JSONL，减少了写放大，也放弃了 token 级崩溃恢复。

按默认路径创建的持久化 session 还有一个更窄的窗口：`SessionManager` 在出现第一条 assistant message 前不会把初始 entries 刷入新文件。这样不会为未得到响应的输入留下大量空会话；如果进程在首个 assistant message 前退出，初始 user message 也不会落盘。已有 assistant 的 session 后续采用追加写入。

## 8. 测试如何约束对象之间的转换

`packages/agent/test/agent-loop.test.ts:131-237` 构造自定义 notification message，并验证 `convertToLlm()` 可以过滤它；另一组输入让 `transformContext()` 只保留最后两条消息，确认 `convertToLlm()` 接到的正是修改后的结果。这锁定了“先临时改 AgentMessage，再转换为 Message”的顺序。

`packages/agent/test/agent.test.ts:409-445` 验证 tools 和 messages 赋值时复制顶层数组，但随后仍允许通过 `agent.state.messages.push()` 修改活动 transcript。测试没有声称 message 对象深度不可变。

`packages/coding-agent/test/session-manager/build-context.test.ts:69-304` 覆盖了空会话、模型与 thinking level 恢复、压缩投影、目标 leaf 分支、断裂 parent 和不存在 leaf。它还确认 `custom` entry 可以留在活动 entry 路径中，却不会出现在 `SessionContext.messages`。

`packages/coding-agent/test/suite/lax-message-content.test.ts:105-160` 构造 null 或缺失 content 的 session message 和 custom message entry，确认加载投影统一得到空数组。容错发生在持久化历史到 AgentMessage 的边界，不会修改原 JSONL。

## QA

### `SessionContext` 和 `pi-ai Context` 为什么同名却不同？

它们位于不同层。`SessionContext` 是 session tree 的恢复投影，包含 `AgentMessage[]`、模型和 thinking level；`pi-ai Context` 是一次 provider 请求，包含 system prompt、标准 `Message[]` 和工具 schema。中间还隔着 AgentState、AgentContext、context hook 和 `convertToLlm()`。

### `AgentState.messages` 是完整会话历史吗？

不是。它是当前活动 transcript。分支切换只装入目标路径，压缩后会装入摘要和保留消息；完整的 append-only 历史仍在 SessionManager entries 和 JSONL 文件中。

### 扩展在 context hook 删除一条消息，会把它从会话里删掉吗？

不会。ExtensionRunner 从 `structuredClone(messages)` 开始，只把结果交给本次 provider request。AgentState 和 session entry 都不变。要永久改变当前活动视图，需要使用会话树、压缩或显式消息写入机制。

### `custom` entry 和 `custom_message` entry 有什么区别？

`custom` 保存扩展状态，不进入模型上下文；`custom_message` 会恢复成 `role: "custom"` 的 AgentMessage，再由 coding-agent 转成 user message 发给模型。`display` 只控制 TUI 呈现，不能用来判断是否发送给模型。

### 为什么不直接把界面显示的内容重新发送给模型？

界面可能隐藏 thinking、折叠工具结果、添加状态指示器，或用 renderer 展示 `details`。这些都是派生视图。模型输入必须从 session 和 Agent 的结构化消息生成，否则 UI 设置会意外改变推理上下文。
