# 第 05 讲：运行时状态、中止与 settlement

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

`agent_end` 看起来像结束事件，源码却没有在发出它时立刻把 Agent 标为空闲。事件订阅器可能还在持久化消息，coding-agent 也可能随后自动重试、压缩上下文，或处理扩展刚排入的 follow-up。Pi 因此有三层结束边界：

```text
agent_end
  一个低层 run 不会再产生新事件
  但 agent_end 订阅器仍可能执行

Agent idle
  当前 run 的所有被 await 订阅器完成
  activeRun 被清除，Agent.waitForIdle() 返回

AgentSession settled
  自动重试、自动压缩和排队续跑全部结束
  agent_settled 发出，AgentSession.waitForIdle() 返回
```

这三个边界决定了什么时候可以安全发起新 prompt、什么时候可以改 session 树，以及 abort 后要等待哪个对象。

## 1. `Agent` 持有活动 transcript 和瞬时运行态

### 解决的问题

低层 `agentLoop()` 接收上下文快照并产生事件，本身不适合长期持有 UI 所需状态。调用者还需要知道当前模型、完整 transcript、正在流式生成的 assistant message、运行中的工具和最近错误。

### 源码入口与关键代码

公开状态定义在 `packages/agent/src/types.ts:315-346`：

```ts
/** Public agent state. */
export interface AgentState {
	/** System prompt sent with each model request. */
	systemPrompt: string;
	/** Active model used for future turns. */
	model: Model<any>;
	/** Requested reasoning level for future turns. */
	thinkingLevel: ThinkingLevel;
	/** Available tools. Assigning a new array copies the top-level array. */
	set tools(tools: AgentTool<any>[]);
	get tools(): AgentTool<any>[];
	/** Conversation transcript. Assigning a new array copies the top-level array. */
	set messages(messages: AgentMessage[]);
	get messages(): AgentMessage[];
	/**
	 * True while the agent is processing a prompt or continuation.
	 * This remains true until awaited `agent_end` listeners settle.
	 */
	readonly isStreaming: boolean;
	/** Partial assistant message for the current streamed response, if any. */
	readonly streamingMessage?: AgentMessage;
	/** Tool call ids currently executing. */
	readonly pendingToolCalls: ReadonlySet<string>;
	/** Error message from the most recent failed or aborted assistant turn, if any. */
	readonly errorMessage?: string;
}
```

内部初始化位于 `packages/agent/src/agent.ts:67-93`：

```ts
function createMutableAgentState(
	initialState?: Partial<Omit<AgentState, "pendingToolCalls" | "isStreaming" | "streamingMessage" | "errorMessage">>,
): MutableAgentState {
	let tools = initialState?.tools?.slice() ?? [];
	let messages = initialState?.messages?.slice() ?? [];

	return {
		systemPrompt: initialState?.systemPrompt ?? "",
		model: initialState?.model ?? DEFAULT_MODEL,
		thinkingLevel: initialState?.thinkingLevel ?? "off",
		get tools() {
			return tools;
		},
		set tools(nextTools: AgentTool<any>[]) {
			tools = nextTools.slice();
		},
		get messages() {
			return messages;
		},
		set messages(nextMessages: AgentMessage[]) {
			messages = nextMessages.slice();
		},
		isStreaming: false,
		streamingMessage: undefined,
		pendingToolCalls: new Set<string>(),
		errorMessage: undefined,
	};
}
```

### 运行流程与状态变化

状态可以分成两组：

| 状态 | 生命周期 | 修改者 |
| --- | --- | --- |
| system prompt、model、thinking level、tools | 跨 run 保留 | 调用者、session 恢复、扩展刷新 |
| messages | 跨 run 增长或被整体替换 | `message_end` 事件、分支恢复、压缩 |
| isStreaming、streamingMessage、pendingToolCalls | 当前 run | `runWithLifecycle()` 与事件归约器 |
| errorMessage | 最近一次失败 turn | run 开始时清空，`turn_end` 时写入 |

给 `state.tools` 或 `state.messages` 赋新数组时，Agent 复制顶层数组，避免调用者之后替换或追加原数组而悄悄改动状态。但 getter 返回的是内部数组本身，`state.messages.push()` 会直接修改当前 transcript。测试 `packages/agent/test/agent.test.ts:409-445` 明确保留了这种行为。

每次进入低层循环前，`createContextSnapshot()` 又复制 messages 和 tools 的顶层数组。run 内可以增长自己的上下文，不会因为数组 `push()` 直接改写 AgentState；最终消息仍要通过事件归约进入长期状态。

### 失败路径与设计取舍

这里只做浅复制。消息对象、工具对象及其嵌套 content 仍共享引用。扩展在 `message_end` 阶段原地修改消息时，AgentState 和随后事件可能看到同一个对象的变化。

`state` 是有意暴露的可变运行对象，不是不可变快照。它方便分支恢复和模型切换，也要求宿主在 Agent 活动期间避免随意重写 transcript。并发 prompt 被禁止，但直接修改 `state.messages` 没有锁或运行时防护。

`reset()` 也不是中止操作。`packages/agent/src/agent.ts:323-331` 会清空 transcript、瞬时状态和队列，却不会 abort 或移除 activeRun。若在运行中调用，后续事件仍可能重新写入消息和运行态。安全顺序是先 abort 并等待 idle，再 reset。

## 2. 事件先归约状态，再通知订阅者

### 解决的问题

收到事件的 UI、持久化器和扩展通常会立即读取 `agent.state`。如果先调用订阅者再更新状态，它们会在同一个事件里看到旧值。

### 源码入口与关键代码

`packages/agent/src/agent.ts:527-573` 把事件归约和通知放在同一个串行函数中：

```ts
private async processEvents(event: AgentEvent): Promise<void> {
	switch (event.type) {
		case "message_start":
			this._state.streamingMessage = event.message;
			break;
		case "message_update":
			this._state.streamingMessage = event.message;
			break;
		case "message_end":
			this._state.streamingMessage = undefined;
			this._state.messages.push(event.message);
			break;
		case "tool_execution_start": {
			const pendingToolCalls = new Set(this._state.pendingToolCalls);
			pendingToolCalls.add(event.toolCallId);
			this._state.pendingToolCalls = pendingToolCalls;
			break;
		}
		case "tool_execution_end": {
			const pendingToolCalls = new Set(this._state.pendingToolCalls);
			pendingToolCalls.delete(event.toolCallId);
			this._state.pendingToolCalls = pendingToolCalls;
			break;
		}
		// ...turn_end 与 agent_end
	}

	const signal = this.activeRun?.abortController.signal;
	if (!signal) {
		throw new Error("Agent listener invoked outside active run");
	}
	for (const listener of this.listeners) {
		await listener(event, signal);
	}
}
```

### 运行流程与状态变化

状态变化发生在订阅者之前：

- `message_start/update` 让 `streamingMessage` 指向当前 assistant 部分消息；
- `message_end` 清掉 partial，并把最终 user、assistant 或 tool result 追加到 transcript；
- 工具 start/end 用复制后的 Set 加减 id，便于基于引用变化触发 UI 更新；
- `turn_end` 若携带 assistant `errorMessage`，更新最近错误；
- `agent_end` 清空 `streamingMessage`，但不关闭 activeRun。

订阅者按注册顺序逐个 await。后注册的订阅者不仅晚收到事件，还必须等前一个订阅者的异步工作完成。`packages/agent/test/agent.test.ts:157-227` 用 barrier 验证：`agent_end` 或 `message_end` 订阅器未完成时，`prompt()` 和 `waitForIdle()` 都不会返回，`isStreaming` 仍为 true。

### 失败路径与设计取舍

订阅者拿到当前 run 的 AbortSignal。abort 后，正在等待的订阅者能看到 `signal.aborted === true`，但 Agent 不会强行取消一个不理会 signal 的 promise。

串行 await 给事件顺序和持久化提供了简单保证，慢订阅者也会给模型循环施加背压。这里没有隔离：订阅者抛错会回到 `runWithLifecycle()` 的 catch。若异常发生在正常事件中，Agent 会尝试生成失败生命周期事件；若失败事件的订阅者仍然抛错，异常可能继续向外传播。订阅器应自行处理内部故障。

## 3. `activeRun` 同时承担并发锁、中止句柄和 idle barrier

### 解决的问题

同一个 AgentState 不能被两个 prompt 循环同时写入。中止、等待完成和拒绝并发调用又必须指向同一次运行，分别维护三套标志容易失去同步。

### 源码入口与关键代码

`packages/agent/src/agent.ts:159-175` 把三个职责放进 `ActiveRun`，Agent 同时持有监听器和两类队列：

```ts
type ActiveRun = {
	promise: Promise<void>;
	resolve: () => void;
	abortController: AbortController;
};

/**
 * Stateful wrapper around the low-level agent loop.
 *
 * `Agent` owns the current transcript, emits lifecycle events, executes tools,
 * and exposes queueing APIs for steering and follow-up messages.
 */
export class Agent {
	private _state: MutableAgentState;
	private readonly listeners = new Set<(event: AgentEvent, signal: AbortSignal) => Promise<void> | void>();
	private readonly steeringQueue: PendingMessageQueue;
	private readonly followUpQueue: PendingMessageQueue;
	// ...其他配置字段与方法
}
```

片段在字段声明处截断了类定义，未改变 `ActiveRun` 与队列的所有权。

`packages/agent/src/agent.ts:304-320` 对外暴露 signal、abort 和等待：

```ts
/** Active abort signal for the current run, if any. */
get signal(): AbortSignal | undefined {
	return this.activeRun?.abortController.signal;
}

/** Abort the current run, if one is active. */
abort(): void {
	this.activeRun?.abortController.abort();
}

/**
 * Resolve when the current run and all awaited event listeners have finished.
 * This resolves after `agent_end` listeners settle.
 */
waitForIdle(): Promise<void> {
	return this.activeRun?.promise ?? Promise.resolve();
}
```

### 运行流程与状态变化

run 开始时创建一个 AbortController 和一条由 Agent 自己 resolve 的 promise，并立即保存到 `activeRun`。因此：

- `prompt()` 与 `continue()` 只需检查 `activeRun` 是否存在，就能拒绝第二个执行循环；
- provider、工具、hook 和订阅者共享同一个 signal；
- 任意调用者取得 `waitForIdle()` 时，拿到的都是这次 run 的 settlement promise。

空闲状态调用 `abort()` 是 no-op，调用 `waitForIdle()` 立即返回。测试 `packages/agent/test/agent.test.ts:468-472` 固定了空闲 abort 不抛错。

### 失败路径与设计取舍

`abort()` 只把 signal 标记为 aborted，不等待任何工作结束，也不直接清除 `activeRun`。调用者若要确认资源已经停止，应随后 `await agent.waitForIdle()`。coding-agent 的 `AgentSession.abort()` 就是“中止重试、调用 Agent.abort、等待 session idle”三步组合。

settlement promise 没有携带成功、错误或 aborted 结果。结果位于 transcript 最后一条 assistant message和 `state.errorMessage` 中。这使 `waitForIdle()` 只表达生命周期屏障，调用者不能把“promise 正常返回”误解为模型成功完成。

## 4. `runWithLifecycle()` 决定何时真正空闲

### 解决的问题

正常响应、provider 抛错、hook 抛错和 abort 都必须清理同一批运行态。清理过早会让第二个 prompt 与事件订阅器并发，清理过晚则可能让 idle 永远不返回。

### 源码入口与关键代码

`packages/agent/src/agent.ts:469-518` 包住每次 prompt 或 continuation：

```ts
private async runWithLifecycle(executor: (signal: AbortSignal) => Promise<void>): Promise<void> {
	if (this.activeRun) {
		throw new Error("Agent is already processing.");
	}

	const abortController = new AbortController();
	let resolvePromise = () => {};
	const promise = new Promise<void>((resolve) => {
		resolvePromise = resolve;
	});
	this.activeRun = { promise, resolve: resolvePromise, abortController };

	this._state.isStreaming = true;
	this._state.streamingMessage = undefined;
	this._state.errorMessage = undefined;

	try {
		await executor(abortController.signal);
	} catch (error) {
		await this.handleRunFailure(error, abortController.signal.aborted);
	} finally {
		this.finishRun();
	}
}

private finishRun(): void {
	this._state.isStreaming = false;
	this._state.streamingMessage = undefined;
	this._state.pendingToolCalls = new Set<string>();
	this.activeRun?.resolve();
	this.activeRun = undefined;
}
```

### 运行流程与状态变化

run 的关键时序是：

```text
创建 activeRun
→ isStreaming = true，清空上次 error
→ 执行 agentLoop / agentLoopContinue
→ agent_end 事件归约状态
→ await 所有 agent_end 订阅者
→ executor 返回
→ finally: finishRun()
→ isStreaming = false
→ resolve waitForIdle promise
→ activeRun = undefined
```

这解释了 `agent_end` 回调中看到 `state.isStreaming === true` 的原因。此时循环事件已经封闭，运行资源尚未释放。新 `prompt()` 仍会因为 activeRun 存在而被拒绝。

`finishRun()` 还会兜底清空 `pendingToolCalls`。正常工具 end 事件应逐项移除 id；若异常打断了事件链，finally 仍能把 UI 的运行中标志收干净。

### 失败路径与设计取舍

`finishRun()` 先 resolve promise，再把 `activeRun` 设为 undefined。Promise reaction 进入微任务队列，当前同步栈会先完成赋值，因此 `await waitForIdle()` 返回时 activeRun 已清除。

Agent 使用 `isStreaming` 表示整次 run 活动期。provider 传 token、工具执行和异步 `agent_end` 订阅器期间，它都保持 true。这个名字容易被误读成纯网络流状态，实际更接近 `isRunActive`。

## 5. 抛出的运行异常会被补成完整事件序列

### 解决的问题

provider stream 函数可能在产生 assistant message 前就抛错，context transform 或循环回调也可能违反“不抛错”契约。如果只让 promise reject，UI 无法收到闭合事件，session 也没有可持久化的失败消息。

### 源码入口与关键代码

`packages/agent/src/agent.ts:494-509` 构造一个合成 assistant message：

```ts
private async handleRunFailure(error: unknown, aborted: boolean): Promise<void> {
	const failureMessage = {
		role: "assistant",
		content: [{ type: "text", text: "" }],
		api: this._state.model.api,
		provider: this._state.model.provider,
		model: this._state.model.id,
		usage: EMPTY_USAGE,
		stopReason: aborted ? "aborted" : "error",
		errorMessage: error instanceof Error ? error.message : String(error),
		timestamp: Date.now(),
	} satisfies AgentMessage;
	await this.processEvents({ type: "message_start", message: failureMessage });
	await this.processEvents({ type: "message_end", message: failureMessage });
	await this.processEvents({ type: "turn_end", message: failureMessage, toolResults: [] });
	await this.processEvents({ type: "agent_end", messages: [failureMessage] });
}
```

### 运行流程与状态变化

若 `streamFn()` 直接抛出 `provider exploded`，此前用户 prompt 事件已经写入状态。catch 随后补发：

```text
message_start(failure assistant)
message_end(failure assistant)   transcript 写入失败消息
turn_end(toolResults = [])       errorMessage 写入 AgentState
agent_end                        正常封闭 run
```

测试 `packages/agent/test/agent.test.ts:126-155` 验证了完整事件顺序和最终状态。`agent.prompt()` 在该路径上正常 resolve；失败事实保存在 assistant `stopReason/errorMessage`，而不是靠外层 promise rejection 表达。

### 失败路径与设计取舍

catch 根据 AbortController 当前是否 aborted 选择 `stopReason`。因此底层抛出同一种异常时，已经收到 abort 的 run 会记录为 aborted，否则记录为 error。

低层循环也可能从 provider 正常得到一个 `stopReason: "error"` 或 `"aborted"` 的最终 assistant message。那条消息由 `runLoop()` 自己发出 `turn_end → agent_end`，不会再经过合成失败路径。两种来源最后形成相似事件序列，但 usage、content 和 provider 元数据可能不同。

## 6. abort 是协作式中止，不是回滚

### 解决的问题

模型请求、工具和异步 hook 都可能在 run 中持有资源。JavaScript 无法安全地从外部强制终止任意 promise，Pi 只能传播统一的 AbortSignal，让各层在自己的安全点响应。

### 运行流程与状态变化

`agent.abort()` 调用当前 AbortController。signal 被传给：

- provider 的 stream 选项；
- 工具 `execute()`；
- before/after tool hook；
- context transform 与 next-turn hook；
- 每个 Agent 事件订阅者。

provider 若响应中止，会完成一条 `stopReason: "aborted"` 的 assistant message。coding-agent 的测试 `packages/coding-agent/test/suite/agent-session-retry-events.test.ts:337-362` 在收到流式 update 后中止，最终仍收到一次 `agent_end` 和一次 `agent_settled`，aborted assistant 也留在 session transcript。

### 失败路径与设计取舍

abort 不会删除已经完成的 user message、assistant partial 或 tool result，也不撤销已经产生的文件和进程副作用。工具是否能及时停下，取决于实现是否检查 signal，以及正在进行的底层操作是否支持取消。

调用 `Agent.abort()` 后立刻读状态，`isStreaming` 仍可能为 true。可靠的关闭写法是先执行 `agent.abort()`，再 `await agent.waitForIdle()`。

在 coding-agent 层应使用 `await session.abort()`，它还会停止自动重试，并等待 session 级 settlement。它不会调用 `abortCompaction()`；若要取消独立的压缩操作，需要显式使用对应 API。直接中止 `session.agent` 只覆盖当前低层 run，无法单独表达“后续自动流程也不要继续”。

## 7. 队列状态和 transcript 不是同一个集合

### 解决的问题

用户在 run 期间输入 steering 或 follow-up 时，消息必须先排队，等循环到达安全注入点再成为 transcript 的一部分。若入队时就追加 messages，模型可能提前看到尚未正式交付的输入，UI 也无法区分 pending 与 delivered。

### 源码入口与关键代码

`packages/agent/src/agent.ts:123-156` 的队列只保存待交付消息：

```ts
class PendingMessageQueue {
	private messages: AgentMessage[] = [];
	public mode: QueueMode;

	constructor(mode: QueueMode) {
		this.mode = mode;
	}

	enqueue(message: AgentMessage): void {
		this.messages.push(message);
	}

	hasItems(): boolean {
		return this.messages.length > 0;
	}

	drain(): AgentMessage[] {
		if (this.mode === "all") {
			const drained = this.messages.slice();
			this.messages = [];
			return drained;
		}

		const first = this.messages[0];
		if (!first) {
			return [];
		}
		this.messages = this.messages.slice(1);
		return [first];
	}

	clear(): void {
		this.messages = [];
	}
}
```

### 运行流程与状态变化

`steer()` 和 `followUp()` 只 enqueue，不触发消息事件，也不修改 `state.messages`。测试 `packages/agent/test/agent.test.ts:448-466` 直接验证了这一点。

循环 drain 消息后才发出 `message_start/message_end`，此时事件归约器把它追加进 transcript。coding-agent 另有 `_steeringMessages` 和 `_followUpMessages` 字符串数组供 UI 展示；收到对应 user `message_start` 时先移除显示队列，再把事件通知界面。测试 `packages/coding-agent/test/suite/agent-session-queue.test.ts:356-384` 观察到 queued message 的 start 事件到达时，`pendingMessageCount` 已经归零。

### 失败路径与设计取舍

`all` 模式一次 drain 全部，`one-at-a-time` 每个检查点取最老一条。drain 是破坏性读取：消息一旦交给循环，就不再属于 pending，即使之后 provider 请求失败，它也已通过 message events 进入 transcript。

Agent 的队列不公开消息列表，只公开 `hasQueuedMessages()` 和清空方法；coding-agent 为 UI 另外保留文本副本。这造成两份队列状态，需要在消息真正交付时同步移除。文本匹配也意味着内容相同的消息按第一次匹配项移除，UI 层不持有 core queue 的对象标识。

## 8. `Agent.waitForIdle()` 只等待一个低层 run

### 解决的问题

Agent core 不知道自动重试、自动压缩和 extension command。它只能判断自己当前的 `activeRun` 是否完整结束。

### 运行流程与状态变化

`Agent.waitForIdle()` 在调用瞬间捕获当前 activeRun 的 promise。若没有 activeRun，它立即 resolve；若 run 存在，就等该 run 的事件订阅器和 finally 清理完成。

这不是“等待未来所有工作”的永久条件变量。某个 `agent_end` 订阅器可以排入消息，但真正的下一次 `agent.continue()` 由更上层稍后发起。只拿第一次 run 的 promise，不能自动包含一个尚未创建的后续 run。

### 失败路径与设计取舍

如果应用只使用独立 `Agent`，这个边界正合适：core 不擅自推断队列是否必须续跑。coding-agent 则需要更宽的等待语义，因此维护自己的 `_isAgentRunActive` 和 idle promise。

`packages/coding-agent/test/suite/agent-session-queue.test.ts:423-443` 中，扩展在 `agent_end` 排入 follow-up。AgentSession 等待并启动 continuation，最终 transcript 含有新消息。调用方若只围绕单个 core run 组织业务，就必须自行决定是否消费结束阶段新入队的工作。

## 9. `AgentSession` 把多个 run 收束成一次 settled

### 解决的问题

一次 coding-agent prompt 可能经历多个低层 Agent run。每次自动重试都有自己的 `agent_start/agent_end`；只监听最后一个低层事件很难判断整项工作是否完成。

### 源码入口与关键代码

`packages/coding-agent/src/core/agent-session.ts:1049-1090` 在 run 结束后检查续跑条件：

```ts
private async _runAgentPrompt(messages: AgentMessage | AgentMessage[]): Promise<void> {
	this._isAgentRunActive = true;
	try {
		await this.agent.prompt(messages);
		while (await this._handlePostAgentRun()) {
			await this.agent.continue();
		}
	} finally {
		this._systemPromptOverride = undefined;
		this._flushPendingBashMessages();
		await this._emitAgentSettled();
	}
}

private async _handlePostAgentRun(): Promise<boolean> {
	const msg = this._lastAssistantMessage;
	this._lastAssistantMessage = undefined;
	if (!msg) {
		return false;
	}
	if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) return true;
	// ...失败重试状态收尾
	if (await this._checkCompaction(msg)) return true;
	return this.agent.hasQueuedMessages();
}
```

session 自己的结束屏障位于 `packages/coding-agent/src/core/agent-session.ts:541-567`：

```ts
private _getIdleWaitPromise(): Promise<void> {
	if (!this._idleWaitPromise) {
		this._idleWaitPromise = new Promise((resolve) => {
			this._resolveIdleWait = resolve;
		});
	}
	return this._idleWaitPromise;
}

private _resolveIdleWaitIfIdle(): void {
	if (this._isAgentRunActive || !this._resolveIdleWait) {
		return;
	}
	const resolve = this._resolveIdleWait;
	this._idleWaitPromise = undefined;
	this._resolveIdleWait = undefined;
	resolve();
}

private async _emitAgentSettled(): Promise<void> {
	this._isAgentRunActive = false;
	try {
		await this._extensionRunner.emit({ type: "agent_settled" });
		this._emit({ type: "agent_settled" });
	} finally {
		this._resolveIdleWaitIfIdle();
	}
}
```

### 运行流程与状态变化

`_isAgentRunActive` 的范围覆盖初次 prompt、后续 retry、compaction continuation 和结束阶段排队消息。直到 `_handlePostAgentRun()` 返回 false，session 才将它设为 false，先 await 扩展 `agent_settled`，再通知同步的公开 listener，最后 resolve session idle waiters。

回归测试 `packages/coding-agent/test/suite/regressions/6363-agent-settled-event.test.ts:29-90` 固定了两种链路：

- 第一个 run 因 overloaded error 自动重试，共有两个 `agent_end`，只有一个 `agent_settled`；
- 扩展在首个 `agent_end` 排入 follow-up，session 续跑后才 settled。

### 失败路径与设计取舍

扩展的 `agent_settled` handler 被 await，公开 `AgentSession.subscribe()` listener 则是同步 `void` 回调。session idle promise 在两类通知之后 resolve，但公开 listener 不能通过返回 promise 延长 settlement。同步 listener 若抛错，`_emitAgentSettled()` 的 finally 仍会释放 idle waiter，当前 `session.prompt()` 则会收到该异常。

`agent_settled` 表示当前自动链路已经结束，不是对未来永久空闲的承诺。某个 handler 仍可能主动启动新操作。官方扩展注释也只保证 handler 执行时通常 `ctx.isIdle()` 为 true。

## 10. 三种等待与中止 API 的选择

| 调用 | 覆盖范围 | 返回时可确认什么 | 不保证什么 |
| --- | --- | --- | --- |
| `agent.abort()` | 当前 core run | 只确认 signal 已触发 | 工具已停止、事件已闭合、队列已消费 |
| `agent.waitForIdle()` | 调用时的当前 core run | 该 run 与 awaited Agent 订阅器结束 | session 不会自动 retry/continue |
| `session.abort()` | 当前 core run 和 retry，并等待 session 链收尾 | 返回时 session idle | 压缩一定被取消、已产生的副作用被回滚 |
| `session.waitForIdle()` | 当前 session 自动链路 | retry、compaction、queued continuation 已 settled | 以后不会有新 prompt |
| `agent_end` 事件 | 一个 core run | 不会再有该 run 的 loop 事件 | Agent 已 idle、session 已 settled |
| `agent_settled` 事件 | 一次 session prompt 链 | 当前自动续跑链结束 | 未来不会由外部再启动任务 |

`packages/coding-agent/test/suite/regressions/6363-agent-settled-event.test.ts:92-156` 让扩展命令在一个工具仍运行时调用 `ctx.waitForIdle()`。命令直到工具释放、下一轮模型完成和 session 发出 settled 后才返回，返回时 `ctx.isIdle()` 为 true。

## 11. 一次中止的完整状态流

假设模型正在流式输出，调用方执行 `await session.abort()`：

```text
AgentSession.abort()
  abortRetry()
  Agent.abort()
    activeRun.abortController.abort()
    signal.aborted = true

provider 观察 signal
  返回 aborted AssistantMessage

Agent.processEvents
  message_end(aborted assistant)
    streamingMessage = undefined
    messages 追加 aborted assistant
  turn_end
    errorMessage = assistant.errorMessage（若存在）
  agent_end
    await Agent 订阅器

Agent.finishRun
  isStreaming = false
  pendingToolCalls = empty
  resolve Agent.waitForIdle()
  activeRun = undefined

AgentSession._handlePostAgentRun
  aborted 不进入正常自动重试
  无 compaction、无 queued continuation

AgentSession._emitAgentSettled
  _isAgentRunActive = false
  await extension agent_settled handlers
  emit public agent_settled
  resolve AgentSession.waitForIdle()

session.abort() 返回
```

中止结果仍是一条会话消息。这样恢复 session 时可以看见任务被取消的位置，provider 下一轮也能接收到一致的历史，而不是遇到一个凭空缺失的 assistant turn。

## 12. 常见误解

### 收到 `agent_end` 后可以立即调用 `agent.prompt()` 吗？

不可以依赖这一点。`agent_end` listener 本身还属于 activeRun，回调执行期间 `isStreaming` 为 true，新 prompt 会被拒绝。应等待当前 `prompt()` 或 `agent.waitForIdle()` 返回。

### `waitForIdle()` 正常返回是否说明模型成功？

不说明。error 和 aborted 都会被归一化成消息并正常完成生命周期。需要检查最后一条 assistant message 的 `stopReason` 和 `errorMessage`。

### queued message 入队后已经属于会话历史吗？

不属于。只有循环 drain 后发出 `message_end`，它才进入 `AgentState.messages` 并由 coding-agent 持久化。

### abort 会撤销已经执行的工具吗？

不会。AbortSignal 是协作式取消，已完成的副作用不会回滚。需要事务或补偿操作时，必须在具体工具层实现。

### `Agent.waitForIdle()` 和 `AgentSession.waitForIdle()` 可以互换吗？

不能。前者封闭一个低层 run；后者封闭 coding-agent 的自动重试、压缩和续跑链。会话切换、关闭和扩展命令通常需要 session 级等待。
