# 第 03 讲：`agentLoop` 的双层循环

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的一次执行不能简单理解成“一问一答”。模型可能先返回工具调用，工具结果又触发下一次模型请求；用户可以在工具运行时补充方向，也可以把新任务排到当前任务之后；coding-agent 还可能在底层循环结束后自动重试、压缩上下文或处理扩展追加的消息。

先固定三个边界：

```text
AgentSession 的一次 prompt
└── 一个或多个 Agent run
    ├── agent_start
    ├── 一个或多个 turn
    │   ├── turn_start
    │   ├── 一次 assistant response
    │   ├── 这次响应产生的零个或多个 tool result
    │   └── turn_end
    └── agent_end
```

`packages/agent/src/types.ts:411-421` 对 turn 的定义很窄：一次 assistant response，加上由这次响应引起的工具调用与结果。它不是一整段用户任务。run 从 `agent_start` 到 `agent_end`；session 则可以把多个 run 串在一次上层 `prompt()` 中。

## 1. `prompt()` 和 `continue()` 从不同的上下文边界启动

### 解决的问题

新输入和已有 transcript 的续跑需要不同语义。新输入应当先写入活动上下文并产生用户消息事件；续跑应直接从已经存在的尾部消息发起模型请求，不能再伪造一条用户消息。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:95-118` 是新 prompt 的入口：

```ts
export async function runAgentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal?: AbortSignal,
	streamFn?: StreamFn,
): Promise<AgentMessage[]> {
	const newMessages: AgentMessage[] = [...prompts];
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

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
	return newMessages;
}
```

`packages/agent/src/agent-loop.ts:120-142` 是续跑入口：

```ts
export async function runAgentLoopContinue(
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal?: AbortSignal,
	streamFn?: StreamFn,
): Promise<AgentMessage[]> {
	if (context.messages.length === 0) {
		throw new Error("Cannot continue: no messages in context");
	}

	if (context.messages[context.messages.length - 1].role === "assistant") {
		throw new Error("Cannot continue from message role: assistant");
	}

	const newMessages: AgentMessage[] = [];
	const currentContext: AgentContext = { ...context };

	await emit({ type: "agent_start" });
	await emit({ type: "turn_start" });

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
	return newMessages;
}
```

### 运行流程与状态变化

`runAgentLoop()` 创建两个集合：

- `currentContext.messages` 是本次 run 内不断增长的工作上下文，初始值为旧 transcript 加本次 prompts；
- `newMessages` 只收集本次 run 新增的消息，最终放进 `agent_end.messages`，也作为流的结果返回。

prompt 本身在模型请求前发出 `message_start/message_end`。`Agent.processEvents()` 收到 `message_end` 后，才把这些消息加入长期持有的 `AgentState.messages`。

`runAgentLoopContinue()` 的 `newMessages` 从空数组开始。已有尾部消息已经属于 transcript，所以不会重复发消息事件；测试 `packages/agent/test/agent-loop.test.ts:1322-1361` 验证了续跑结果只有新 assistant message。

上层 `Agent.prompt()` 和 `Agent.continue()` 都禁止与活动 run 并发。运行中出现新输入时，调用者必须选择 `steer()` 或 `followUp()`，而不是再开一个共享同一状态的循环。

### 失败路径与设计取舍

低层 continuation 至少要求上下文非空，且尾消息不能是 assistant。user、tool result 或自定义 role 能否被模型接受，由调用者提供的 `convertToLlm()` 负责。测试允许自定义尾消息，明确把转换责任留给调用方。

`Agent.continue()` 还处理了一个上层特例：如果 transcript 以 assistant 结尾，但队列里有 steering 或 follow-up，它会先取出队列消息，再按新 prompt 启动。两种入口因此不是“是否保留上下文”的区别，它们都会使用现有 transcript；真正的区别是本次是否引入并发出新的输入消息。

## 2. 内层循环回答“当前工作是否还没做完”

### 解决的问题

一次 assistant response 可能只给出 tool call，不能在工具执行后直接结束。另一方面，用户在当前 turn 运行期间提交的 steering，必须等本轮工具批次完整结束，再进入下一次模型请求，避免把一个 assistant message 的工具结果拆散。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:163-224` 是内层循环的主体：

```ts
let firstTurn = true;
let pendingMessages = (await config.getSteeringMessages?.()) || [];

while (true) {
	let hasMoreToolCalls = true;

	while (hasMoreToolCalls || pendingMessages.length > 0) {
		if (!firstTurn) await emit({ type: "turn_start" });
		else firstTurn = false;

		for (const message of pendingMessages) {
			await emit({ type: "message_start", message });
			await emit({ type: "message_end", message });
			currentContext.messages.push(message);
			newMessages.push(message);
		}
		pendingMessages = [];

		const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFn);
		newMessages.push(message);
		if (message.stopReason === "error" || message.stopReason === "aborted") {
			await emit({ type: "turn_end", message, toolResults: [] });
			await emit({ type: "agent_end", messages: newMessages });
			return;
		}
		// ...提取并执行本轮 tool calls
		await emit({ type: "turn_end", message, toolResults });
	}
	// ...外层 follow-up 判断
}
```

片段压缩了空队列判断，并把工具执行分支留到下一节，循环条件和事件顺序未改动。

### 运行流程与状态变化

内层条件由两个信号组成：

| 信号 | 决策者 | 为真时的含义 |
| --- | --- | --- |
| `hasMoreToolCalls` | 工具批次执行结果 | 工具结果还需要交给模型形成下一次 assistant response |
| `pendingMessages.length > 0` | steering 队列 | 下一次模型请求前要注入用户补充消息 |

`hasMoreToolCalls` 初始为 `true`，保证没有工具、没有队列消息时也会产生第一条 assistant response。此后每轮重新计算。assistant message 在流式接收时已经写入 `currentContext.messages`；工具结果随后按 tool call 顺序追加，所以下一轮模型能看到完整的“assistant tool call → tool result”配对。

steering 的轮询发生在当前 assistant response 和整批工具执行之后。`packages/agent/test/agent-loop.test.ts:620-723` 构造了一个包含两个工具调用的响应，即使第一项执行后队列已有 steering，第二项仍会先完成，steering 才被注入下一轮上下文。

这种边界保护了消息协议的一致性。若 steering 能插进同一 assistant message 的两个 tool result 之间，部分 provider 会看到不完整的工具批次，应用也很难判断用户是在修改哪一轮工作。

### 失败路径与设计取舍

assistant response 的 `stopReason` 为 `error` 或 `aborted` 时，循环不再执行其中可能存在的工具调用，也不再读取 steering/follow-up 队列，直接用空 `toolResults` 结束 turn 和 run。队列没有被本轮消费，仍可由之后的显式续跑处理。

`stopReason === "length"` 且响应里含 tool call 时，参数可能在 token 边界被截断。Pi 为每个调用生成错误 tool result，不执行工具，然后继续下一轮，让模型有机会重新发起调用。这里把一次额外模型请求换成了不执行残缺参数的安全边界。

## 3. 工具批次决定是否自然进入下一轮

### 解决的问题

常规工具结果需要返回给模型继续推理；某些工具已经完成整个控制流，继续请求模型反而多余。并行批次中又可能只有部分结果要求终止，不能让其中一个结果提前吞掉其他结果。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:202-220, 542-585` 把工具批次归约成消息和一个布尔值：

```ts
const toolResults: ToolResultMessage[] = [];
hasMoreToolCalls = false;
if (toolCalls.length > 0) {
	const executedToolBatch =
		message.stopReason === "length"
			? await failToolCallsFromTruncatedMessage(toolCalls, emit)
			: await executeToolCalls(currentContext, message, config, signal, emit);

	toolResults.push(...executedToolBatch.messages);
	hasMoreToolCalls = !executedToolBatch.terminate;

	for (const result of toolResults) {
		currentContext.messages.push(result);
		newMessages.push(result);
	}
}

function shouldTerminateToolBatch(finalizedCalls: FinalizedToolCallOutcome[]): boolean {
	return finalizedCalls.length > 0 && finalizedCalls.every(
		(finalized) => finalized.result.terminate === true,
	);
}
```

片段由工具分支和批次终止函数两处短代码拼接，省略了并行、串行执行细节。

### 运行流程与状态变化

没有 tool call 时，`hasMoreToolCalls` 保持 `false`。有 tool call 时：

- 所有结果都带 `terminate: true`，工具链本身不再要求下一次模型请求；
- 只要有一个结果未终止，整批结果仍要交回模型；
- 长度截断生成的失败结果固定返回 `terminate: false`，允许模型修正调用。

并行执行只改变工具如何调度，不改变归约规则。`packages/agent/test/agent-loop.test.ts:1140-1254` 分别验证了“全部终止时只有一次 LLM 调用”和“并行批次未全部终止时仍有第二次 LLM 调用”。

### 失败路径与设计取舍

`terminate` 只关闭“工具结果要求续跑”这个条件，不是不可覆盖的全局退出。当前 turn 之后若取得 steering，内层循环仍会继续；外层也仍可能取得 follow-up。需要无条件在本轮结束后停止时，应由 `shouldStopAfterTurn` 决策。

用“全体为真”归约并行批次，避免单个工具替其他工具决定整个 run 的命运。代价是工具作者必须清楚 `terminate` 的批次语义，不能把它理解为立即中断其他并行任务。

## 4. steering 和 follow-up 分属两层循环

### 解决的问题

运行中追加的消息有两种意图：一种要尽快影响下一轮推理，另一种要等当前任务自然完成后再开始。只用一个 FIFO 队列无法表达这种优先级。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:224-274` 在每个 turn 结束后按固定顺序决策：

```ts
await emit({ type: "turn_end", message, toolResults });

const nextTurnSnapshot = await config.prepareNextTurn?.({
	message,
	toolResults,
	context: currentContext,
	newMessages,
});
if (nextTurnSnapshot) {
	currentContext = nextTurnSnapshot.context ?? currentContext;
	// model 与 thinking level 也可在这里替换
}

if (await config.shouldStopAfterTurn?.({ message, toolResults, context: currentContext, newMessages })) {
	await emit({ type: "agent_end", messages: newMessages });
	return;
}

pendingMessages = (await config.getSteeringMessages?.()) || [];
}

const followUpMessages = (await config.getFollowUpMessages?.()) || [];
if (followUpMessages.length > 0) {
	pendingMessages = followUpMessages;
	continue;
}

await emit({ type: "agent_end", messages: newMessages });
```

片段省略了 snapshot 中 model、thinking level 的赋值，保留了四个决策点的先后关系。

### 运行流程与状态变化

顺序不能交换：

1. `turn_end` 表示 assistant response 和它的工具结果已经完整结束。
2. `prepareNextTurn` 可以替换下一轮使用的 context、model 或 thinking level。
3. `shouldStopAfterTurn` 是硬停止点。返回 `true` 后不再轮询两类队列。
4. steering 在每个 turn 后轮询，取得后进入内层下一轮。
5. 只有工具链和 steering 都不再要求续跑，才轮询 follow-up。
6. follow-up 被放回 `pendingMessages`，外层循环重新进入内层。

循环启动前还有一次 steering 轮询。它覆盖“用户消息已入队，但 run 刚好还没开始”的时间窗，使这条消息能进入第一次模型请求。

`Agent` 的两个 `PendingMessageQueue` 支持 `one-at-a-time` 和 `all`。前者每次 drain 一条，因此两条 steering 会触发两次 assistant response；后者一次取出全部消息，合并进入同一次请求。`packages/coding-agent/test/suite/agent-session-queue.test.ts:158-203` 固定了两类消息在逐条模式下的顺序。

### 失败路径与设计取舍

`prepareNextTurn`、`shouldStopAfterTurn` 和队列回调的类型契约都要求调用者自行兜底，不应抛出异常。若回调仍然抛错，低层 `runLoop()` 无法产生正常的闭合事件序列；`Agent.runWithLifecycle()` 会捕获异常，补出 error assistant message、`turn_end` 和 `agent_end`，再清理运行态。

双队列让“修正当前工作”和“排队下一项工作”有了源码级边界。它没有抢占正在执行的工具。steering 的“尽快”指当前工具批次结束后，而不是取消当前批次。

## 5. `shouldStopAfterTurn` 是显式停止裁决点

### 解决的问题

有些停止理由来自模型与工具之外，例如上下文即将溢出、宿主要求在当前安全边界暂停。直接 abort 会把当前响应标记为中止，也可能切断工具；等自然退出又可能多发一次模型请求。

### 源码入口与关键代码

`packages/agent/src/types.ts:203-213` 把停止回调约束在完整 turn 之后：

```ts
/**
 * Called after each turn fully completes and `turn_end` has been emitted.
 *
 * If it returns true, the loop emits `agent_end` and exits before polling steering or follow-up queues,
 * without starting another LLM call. The current assistant response and any tool executions finish normally.
 *
 * Use this to request a graceful stop after the current turn, e.g. before context gets too full.
 *
 * Contract: must not throw or reject. Throwing interrupts the low-level agent loop without producing a normal event sequence.
 */
shouldStopAfterTurn?: (context: ShouldStopAfterTurnContext) => boolean | Promise<boolean>;
```

### 运行流程、状态变化与失败结果

`shouldStopAfterTurn` 收到已完成的 assistant message、完整 tool results、更新后的 context 和本次新增消息。它在 `prepareNextTurn` 之后执行，因此能看到替换后的上下文；在队列轮询之前执行，因此返回 `true` 时 steering 和 follow-up 都保留在队列中。

测试 `packages/agent/test/agent-loop.test.ts:1043-1122` 让第一轮产生 tool call，并让停止回调返回 `true`。结果是工具正常执行、上下文已有 `user → assistant → toolResult`，LLM 只调用一次，follow-up 完全没有被轮询。

这与几种其他停止路径不同：

| 条件 | 当前工具是否执行 | 是否再看 steering | 是否再看 follow-up | 结果 |
| --- | --- | --- | --- | --- |
| assistant `error/aborted` | 否 | 否 | 否 | 立即 `turn_end → agent_end` |
| tool batch 全部 `terminate` | 是 | 是 | 内层结束后是 | 不再因工具结果续轮 |
| `shouldStopAfterTurn === true` | 是 | 否 | 否 | 当前 turn 完整结束后强制结束 run |
| 无 tool call、无 steering | 不适用 | 已轮询 | 是 | 有 follow-up 则开新 turn，否则结束 |
| `length` 中含 tool call | 不执行真实工具，生成错误结果 | 是 | 暂不进入 | 默认再请求模型修正 |

`shouldStopAfterTurn` 选择了 turn 边界作为安全停止点。它不能在耗时工具运行中即时止损；即时停止仍由 AbortSignal 负责，两者处理的是不同问题。

## 6. `Agent.continue()` 处理 transcript 尾部，`AgentSession` 处理 run 之后的工作

### 解决的问题

底层 run 已经结束，并不代表 coding-agent 可以立刻向界面报告 settled。自动重试、上下文压缩和 `agent_end` 扩展处理器都可能在结束事件附近制造新的续跑条件。

### 源码入口与关键代码

`packages/coding-agent/src/core/agent-session.ts:1049-1090` 在 session 层包住底层 Agent：

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
	if (!msg) return false;
	if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) return true;
	// ...结束失败的重试状态
	if (await this._checkCompaction(msg)) return true;
	return this.agent.hasQueuedMessages();
}
```

片段省略了失败重试计数的收尾分支，不影响 retry、compaction、queued message 的判断次序。

### 运行流程与状态变化

初次调用使用 `agent.prompt(messages)`。每个 run 完成后，session 依次检查：

1. 最后一条 assistant message 是否属于可重试错误，并且重试准备成功；
2. 是否需要自动压缩，压缩逻辑是否已准备好新的上下文；
3. `agent_end` 的异步扩展处理器是否又排入消息。

任一条件成立，session 调用 `agent.continue()` 开始新的 run。直到这些条件全部消失，才清除临时 system prompt、刷出待处理 bash message，并发出 settled 事件。

`agent_end` 本身也不是 `Agent` 已空闲的瞬间。`Agent.processEvents()` 会等待订阅者处理完事件，之后 `finishRun()` 才清理 `activeRun` 和 `isStreaming`。这保证 `agent_end` 扩展处理器能够被 session 纳入续跑判断，也解释了为什么“收到结束事件”与“可以再次调用 prompt”之间仍有一小段生命周期。

### 失败路径与设计取舍

`Agent.continue()` 在 transcript 以 assistant 结尾时，优先 drain steering，再 drain follow-up；两者都没有才报错。处理 steering 时传入 `skipInitialSteeringPoll`，避免刚刚 drain 出来的首批消息又被循环启动阶段的 steering 轮询打乱。剩余队列仍按配置的逐条或批量模式继续处理。

session 把自动恢复放在多个独立 run 之间，而不是塞进 `runLoop()`，使低层循环保持通用，也让 coding-agent 可以插入压缩和扩展语义。代价是观察生命周期时必须标明层级：一次 `agent_end` 能结束一个 run，却未必结束一次 session prompt。

## 7. 一条完整调用链

假设模型第一次响应包含两个工具调用；工具执行期间用户各提交一条 steering 和 follow-up：

```text
AgentSession.prompt("检查项目")
  Agent.prompt()
    runAgentLoop()
      agent_start
      turn 1
        写入 user prompt
        LLM response: tool A + tool B
        执行 A、B，写入两个 toolResult
        轮询 steering，取得“先看配置文件”
      turn 2
        写入 steering
        LLM response: 最终回答，无 tool call
        轮询 steering，无消息
      内层退出
      轮询 follow-up，取得“再检查测试”
      turn 3
        写入 follow-up
        LLM response: 最终回答，无 tool call
      agent_end
  AgentSession 检查 retry / compaction / agent_end 新队列
  无续跑条件，agent_settled
```

这条链里有一次 session prompt、一次 Agent run、三个 turn 和三次 assistant response。若 `agent_end` 扩展处理器又排入 follow-up，最外层会再出现一个 Agent run，session 仍要等它结束后才 settled。

## 8. 常见误解

### steering 会中断正在执行的工具吗？

不会。源码在整批工具处理完并发出 `turn_end` 后才轮询 steering。需要即时取消时应触发 abort；steering 表达的是下一轮推理方向。

### follow-up 会创建新的 Agent run 吗？

通常不会。只要它在 `runLoop()` 自然退出前已进入 follow-up 队列，外层循环会在同一 run 内处理。若消息由异步 `agent_end` 处理器加入，此时低层循环已经结束，`AgentSession` 才会用 `continue()` 创建下一个 run。

### `continue()` 是把最后一条 assistant response 再发一次吗？

不是。它从现有 transcript 发起新的模型请求，不重复写入旧消息。若 transcript 以 assistant 结尾，必须先有排队消息把对话重新推进到可请求状态；否则直接报错。

### 一个 turn 能包含多次模型响应吗？

不能。Pi 的事件类型把 turn 定义为一次 assistant response 及其工具调用和结果。工具结果触发的下一次模型请求属于下一个 turn。
