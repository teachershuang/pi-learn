# 第 06 讲：失败如何进入事件、消息与状态

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 没有把所有异常都交给同一个全局 `catch`。工具参数错误需要反馈给模型，让模型有机会重发；provider 失败要结束当前 turn，但仍应留下 assistant 消息；同一 Agent 上的并发 prompt 属于调用方违约，应在运行开始前拒绝。监听器抛错更特殊：它发生在控制面，既可能被包装成一次 run 失败，也可能破坏包装失败所需的事件序列。

判断一个失败时，至少要同时检查四个观察面：

| 观察面 | 要找的结果 |
| --- | --- |
| 事件 | 是否出现 `tool_execution_end`、`turn_end`、`agent_end` |
| transcript | 写入了 `toolResult.isError`，还是 assistant error message |
| `AgentState` | `pendingToolCalls`、`errorMessage`、`isStreaming` 最终如何变化 |
| 调用 Promise | `prompt()` 正常返回，还是 reject |

同一句“调用失败”，在这四处留下的结果可能完全不同。

```text
模型或宿主输入
  │
  ├─ 工具级可恢复失败 ──> error toolResult ──> 下一次模型请求
  │
  ├─ turn 级终止失败 ──> assistant(error/aborted) ──> agent_end
  │
  ├─ run 前置条件失败 ──> prompt/continue 直接 reject，不产生事件
  │
  └─ 控制面异常 ──────> Agent 尝试合成失败生命周期
                         └─ 若失败处理也抛错，Promise reject
```

工具级失败的详细正常链路见[第 04 讲](./04-tool-call-pipeline.md)，运行结束与 idle 边界见[第 05 讲](./05-runtime-state-abort-settlement.md)。本讲只讨论失败怎样改变这些链路。

## 1. 失败的归属层决定是否继续调用模型

### 解决的问题

工具名或参数错误通常来自模型输出。若直接让整个 run reject，模型看不到错误，也无法修正调用。provider 已经失败时则不能继续当前 turn；再发请求需要宿主的重试策略。并发 prompt 又不是模型可以处理的问题，把它写进 transcript 反而会污染上下文。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:192-224` 先接收最终 assistant message，再决定停止还是处理工具：

```ts
const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFn);
newMessages.push(message);

if (message.stopReason === "error" || message.stopReason === "aborted") {
	await emit({ type: "turn_end", message, toolResults: [] });
	await emit({ type: "agent_end", messages: newMessages });
	return;
}

const toolCalls = message.content.filter((c) => c.type === "toolCall");
const toolResults: ToolResultMessage[] = [];
hasMoreToolCalls = false;
if (toolCalls.length > 0) {
	const executedToolBatch =
		message.stopReason === "length"
			? await failToolCallsFromTruncatedMessage(toolCalls, emit)
			: await executeToolCalls(currentContext, message, config, signal, emit);
	toolResults.push(...executedToolBatch.messages);
	hasMoreToolCalls = !executedToolBatch.terminate;
	// 将 tool results 追加到本轮上下文
}

await emit({ type: "turn_end", message, toolResults });
```

### 运行流程与状态变化

这段分支形成两种协议内失败：

- assistant 的 `stopReason` 已是 `error` 或 `aborted`：当前 turn 结束，不检查也不执行其中的工具块；
- assistant 正常结束但工具调用有问题：生成一个 `role: "toolResult"`、`isError: true` 的消息，当前 run 通常继续，让模型读取错误。

`AgentState.errorMessage` 只在 `turn_end.message.errorMessage` 存在时更新。工具错误没有写进 assistant message，因此不会自动进入 `state.errorMessage`；它存在于 tool result、工具结束事件和当前 turn 的 `toolResults` 中。

### 失败路径与设计取舍

Pi 以“谁能修复”划分失败边界。模型可能修复工具名和参数，所以这些错误进入 transcript；模型无法修复宿主对同一状态对象的并发写入，所以并发 prompt 直接拒绝；provider error 是否重试由更高层决定。

这不是按异常类型机械分类。同一个 `Error` 对象从 `tool.execute()` 抛出时会变成 tool result，从事件监听器抛出时却可能终止整个 run。异常发生的位置比 JavaScript 类型更重要。

## 2. 长度截断会让同一 assistant message 中的全部工具调用失败

### 解决的问题

流式 JSON 的补救解析器可能把被截断的参数整理成一个语法合法、甚至通过 schema 的对象。例如模型原本要输出一个完整路径，token 上限把字符串截在中间；单靠校验无法知道这个短字符串是不是模型本意。执行它可能写错文件或运行错误命令。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:383-407` 不尝试判断哪个工具块完整，而是拒绝该 assistant message 中的整批调用：

```ts
async function failToolCallsFromTruncatedMessage(
	toolCalls: AgentToolCall[],
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
	const messages: ToolResultMessage[] = [];
	for (const toolCall of toolCalls) {
		await emit({
			type: "tool_execution_start",
			toolCallId: toolCall.id,
			toolName: toolCall.name,
			args: toolCall.arguments,
		});
		const finalized: FinalizedToolCallOutcome = {
			toolCall,
			result: createErrorToolResult(
				`Tool call "${toolCall.name}" was not executed: the response hit the output token limit, so its arguments may be truncated. Re-issue the tool call with complete arguments.`,
			),
			isError: true,
		};
		await emitToolExecutionEnd(finalized, emit);
		const toolResultMessage = createToolResultMessage(finalized);
		await emitToolResultMessage(toolResultMessage, emit);
		messages.push(toolResultMessage);
	}
	return { messages, terminate: false };
}
```

### 运行流程与状态变化

每个调用仍产生完整的可观察外壳：

```text
tool_execution_start
tool_execution_end(isError=true)
message_start(toolResult)
message_end(toolResult)
```

真正的 `tool.execute()`、`beforeToolCall` 和 `afterToolCall` 都不会运行。`Agent` 在 start 时把 id 加入 `pendingToolCalls`，在 end 时移除；最终 transcript 中保留与每个 tool call id 配对的错误结果。批次返回 `terminate: false`，下一次模型请求可以重发完整调用。

测试 `packages/agent/test/agent-loop.test.ts:310-381` 构造了一个 `stopReason: "length"` 且参数仍能通过 schema 的 tool call，验证执行数组保持为空、工具结束事件标为错误，并且 agent-loop 确实发起第二次模型调用。

### 失败路径与设计取舍

批次中即使有一个看起来完整的调用，也会一起失败。这牺牲了部分可用结果，换取“不执行来源不完整的动作”。对于读操作，这可能显得保守；对于文件写入、命令执行等有副作用的工具，逐个猜测完整性风险更高。

这里的“截断”专指 assistant `stopReason === "length"`，不是工具输出为了控制上下文而做的展示截断。后者已经执行完成，只是结果内容被裁剪，两者不能混为一谈。

## 3. 未知工具、参数错误和执行前异常都变成即时错误结果

### 解决的问题

模型可能调用当前上下文不存在的工具，旧会话中的参数也可能不再符合新版 schema。`prepareArguments`、schema 校验和 `beforeToolCall` 还可能自己抛错。它们都发生在业务工具启动之前，不应伪装成已经执行过的操作。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:602-665` 把执行前阶段收束为 `PreparedToolCall` 或 `ImmediateToolCallOutcome`：

```ts
const tool = currentContext.tools?.find((t) => t.name === toolCall.name);
if (!tool) {
	return {
		kind: "immediate",
		result: createErrorToolResult(`Tool ${toolCall.name} not found`),
		isError: true,
	};
}

try {
	const preparedToolCall = prepareToolCallArguments(tool, toolCall);
	const validatedArgs = validateToolArguments(tool, preparedToolCall);
	if (config.beforeToolCall) {
		const beforeResult = await config.beforeToolCall(
			{ assistantMessage, toolCall, args: validatedArgs, context: currentContext },
			signal,
		);
		if (beforeResult?.block) {
			return {
				kind: "immediate",
				result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
				isError: true,
			};
		}
	}
	return { kind: "prepared", toolCall, tool, args: validatedArgs };
} catch (error) {
	return {
		kind: "immediate",
		result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
		isError: true,
	};
}
```

### 运行流程与状态变化

三类失败的共同结果是：已发出 `tool_execution_start`，没有调用 `tool.execute()`，随后发出错误 `tool_execution_end` 和 error tool result。差异在于停止位置：

| 失败 | 最后经过的决策点 | 不会调用 |
| --- | --- | --- |
| 未知工具 | 按名称查找工具 | 参数准备、校验、两个 hook、execute |
| `prepareArguments`/校验失败 | `try` 中抛错 | `beforeToolCall`、execute、`afterToolCall` |
| `beforeToolCall` 阻止或抛错 | 执行前策略 | execute、`afterToolCall` |
| preflight 期间收到 abort | signal 检查 | execute、`afterToolCall` |

`validateToolArguments()` 位于 `packages/ai/src/utils/validation.ts:278-309`。它先 `structuredClone()` 原始参数，再按 schema 做类型转换和校验；失败信息包含字段路径与收到的参数。测试 `packages/ai/test/validation.test.ts:35-115` 既覆盖字符串到数字等兼容转换，也确认无法转换的值抛出 `Validation failed`。

### 失败路径与设计取舍

`afterToolCall` 只处理已经进入执行阶段的工具。未知工具、参数错误和 preflight 阻止不会进入它，因此不能把 `afterToolCall` 当成所有工具结果的统一错误出口。

并行模式也先按 assistant 中的顺序逐个 preflight。某个调用即时失败不会自动取消同批次中已经允许的其他工具；它们仍可并发执行，最后所有 tool result 按模型给出的调用顺序写入 transcript。

参数校验允许有限类型转换，这增加了模型输出的容错性，也意味着“通过校验”不等于“原始 JSON 类型完全正确”。需要严格区分时，应在 schema 或 preflight 策略中明确约束，而不是只检查 execute 收到的值。

## 4. 工具异常进入 `afterToolCall`，后处理异常会覆盖原结果

### 解决的问题

工具运行可能抛错，异步进度事件也可能处理失败。核心循环需要等已经接受的进度事件收束，再产生唯一的最终结果；扩展还可能需要给错误补充可读信息或决定是否结束批次。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:668-708` 负责执行和进度窗口：

```ts
const updateEvents: Promise<void>[] = [];
let acceptingUpdates = true;

try {
	const result = await prepared.tool.execute(
		prepared.toolCall.id,
		prepared.args as never,
		signal,
		(partialResult) => {
			if (!acceptingUpdates) return;
			updateEvents.push(Promise.resolve(emit({
				type: "tool_execution_update",
				toolCallId: prepared.toolCall.id,
				toolName: prepared.toolCall.name,
				args: prepared.toolCall.arguments,
				partialResult,
			})));
		},
	);
	acceptingUpdates = false;
	await Promise.all(updateEvents);
	return { result, isError: false };
} catch (error) {
	acceptingUpdates = false;
	await Promise.all(updateEvents);
	return {
		result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
		isError: true,
	};
} finally {
	acceptingUpdates = false;
}
```

### 运行流程与状态变化

`execute()` resolve 或 reject 后，`acceptingUpdates` 立即关闭。此前已经进入 `updateEvents` 的事件会被 await；此后调用保存下来的 `onUpdate` 函数只会直接返回。测试 `packages/agent/test/agent.test.ts:268-407` 覆盖两种晚到更新：整个 prompt 结束后调用，以及并行批次中另一个工具尚未完成时调用。两者都不会增加事件，也不会造成未处理的 Promise rejection。

执行异常先变成 `{ result: errorResult, isError: true }`，随后仍会进入 `afterToolCall`。`packages/agent/src/agent-loop.ts:719-747` 允许 hook 替换 `content`、`details`、`terminate` 或 `isError`。因此宿主可以补充错误展示，甚至明确把错误改成成功。

### 失败路径与设计取舍

`afterToolCall` 自己抛错时，源码会用它的错误消息替换当前 result，并把 `isError` 设为 true。若原工具已经失败，原始错误可能因此丢失；若工具已经成功，成功结果也会被后处理错误覆盖。这让 hook 保持对最终结果的决定权，但 hook 应在需要时把原始错误作为 cause 或 details 保留下来。

还有一个不直观的边界：`tool_execution_update` 的 `emit()` Promise 也放在 `try` 中。若 Agent 订阅者处理进度事件时抛错，第一次 `Promise.all(updateEvents)` 会进入 `catch`，但 `catch` 又会 await 同一组 update Promise，拒绝会再次向外传播。它不会稳定地收束为工具错误，而会进入 Agent 的 run 失败处理；若工具同时抛错，进度监听器的拒绝还可能遮住工具异常。当前相关测试只验证更新收束和晚到更新，没有专门覆盖“进度监听器抛错”。

## 5. provider error 是终止事件，不是 rejected stream Promise

### 解决的问题

网络失败、鉴权失败、限流和用户中止都可能发生在流建立前或读取中。若每个 provider 选择不同的 throw/reject 方式，agent-loop 很难保证总能产生最终消息并关闭事件流。

### 源码入口与关键代码

`packages/ai/src/types.ts:301-313` 定义 provider stream 契约：请求、模型和运行失败应编码在 `AssistantMessageEventStream` 中，终点是 `error` 事件及一条 `stopReason: "error" | "aborted"` 的 assistant message。

异步鉴权、provider 查找和模块加载由 `packages/ai/src/api/lazy.ts:46-60` 包装：

```ts
export function lazyStream(
	model: Model<Api>,
	setup: () => Promise<AsyncIterable<AssistantMessageEvent>>,
): AssistantMessageEventStream {
	const outer = new AssistantMessageEventStream();

	setup()
		.then((inner) => forwardStream(outer, inner))
		.catch((error) => {
			const message = createSetupErrorMessage(model, error);
			outer.push({ type: "error", reason: "error", error: message });
			outer.end(message);
		});

	return outer;
}
```

### 运行流程与状态变化

agent-loop 对 `done` 和 `error` 终端事件采用同一条取最终消息路径：`response.result()` 返回 assistant message，随后发出 `message_end`。`runLoop()` 看到其 stop reason 后发出 `turn_end`、`agent_end` 并返回。

以 `Agent` 为宿主时，标准 provider error 的外部表现是：

```text
assistant message 写入 transcript，stopReason=error/aborted
turn_end.toolResults = []
AgentState.errorMessage = assistant.errorMessage
agent_end 发出并等待监听器
prompt() 正常 resolve
```

`prompt()` 正常返回并不表示模型成功，只表示这次运行按 Agent 协议收束。业务层必须检查最终 assistant message、`state.errorMessage` 或上层 session 事件，不能只依赖 `try/catch`。

### 失败路径与设计取舍

自定义 `StreamFn` 的类型注释也要求“不抛异常”。不过 `Agent` 仍有兜底：`runWithLifecycle()` 捕获异常并调用 `handleRunFailure()`，合成一条空 content 的 assistant error message，再补发 `message_start/end`、`turn_end` 和 `agent_end`。测试 `packages/agent/test/agent.test.ts:126-155` 让 `streamFn` 直接抛出 `provider exploded`，验证完整失败生命周期和 `AgentState.errorMessage`。

低层 `agentLoop()` 没有同样的异常修复。`packages/agent/src/agent-loop.ts:31-53` 以 fire-and-forget 方式调用 `runAgentLoop(...).then(stream.end)`，没有 rejection handler。若 `convertToLlm`、`transformContext` 或低层 event sink 违反契约并抛错，返回的 `EventStream` 可能等不到 `agent_end`。这是直接使用低层 API 时必须承担的边界；需要状态归约和失败生命周期时，应使用 `Agent` 或直接调用可 await 的 `runAgentLoop()` 并自行捕获异常。

## 6. 监听器不是旁路日志，它处在运行结算链上

### 解决的问题

`Agent.subscribe()` 常被用于持久化和扩展处理。后续工具 preflight 可能依赖 assistant 消息已经保存，因此监听器需要成为事件屏障。但一旦把外部代码放进主链，它的失败也会影响 Agent。

### 源码入口与关键代码

`packages/agent/src/agent.ts:527-573` 先归约状态，再按注册顺序 await 监听器：

```ts
private async processEvents(event: AgentEvent): Promise<void> {
	switch (event.type) {
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
		// 其他事件归约
	}

	const signal = this.activeRun?.abortController.signal;
	if (!signal) throw new Error("Agent listener invoked outside active run");
	for (const listener of this.listeners) {
		await listener(event, signal);
	}
}
```

### 运行流程与状态变化

监听器抛错前，当前事件对应的状态已经提交，不会自动回滚。抛错的监听器之后，排在后面的监听器不会收到该事件。异常通常回到 `runWithLifecycle()`，Agent 随后尝试合成 assistant failure message。

若监听器只在某个正常事件上失败，而能正常处理补发的失败事件，run 最终仍可能以 error assistant message 收束，`prompt()` 正常返回。若它对补发的 `message_start`、`message_end` 或 `agent_end` 继续抛错，`handleRunFailure()` 也会 reject；`finally` 仍执行 `finishRun()`，清除运行态并解除 `waitForIdle()`，但 `prompt()` 会 reject。

### 失败路径与设计取舍

这里没有事务。一个 `message_end` 监听器抛错时，消息已进入 `AgentState.messages`；失败处理又可能追加一条 assistant error message。宿主若同时写数据库，需要自行保证幂等和部分提交后的恢复。

串行、可 await 的监听器提供确定顺序，也放大了监听器故障。用于纯遥测的订阅器应在内部隔离错误；参与持久化的监听器应明确失败策略，不能假设 Agent 会回滚状态。官方 `packages/agent/docs/observability.md` 提出的被动 observability subscriber 反而要求吞掉或隔离错误，这与控制面 hook/Agent subscriber 的语义不同；该文档描述的是观测设计，不代表现有 `Agent.subscribe()` 已经具备隔离。

## 7. 并发 prompt 在任何消息或事件产生前被拒绝

### 解决的问题

两个 run 同时追加同一个 transcript、修改 `streamingMessage` 和 `pendingToolCalls`，会让事件顺序失去意义。Pi 不为单个 Agent 加可重入调度器，而是用 `activeRun` 作为互斥边界。

### 源码入口与关键代码

`packages/agent/src/agent.ts:334-374` 在规范化输入和建立新 run 之前检查 `activeRun`：

```ts
async prompt(input: string | AgentMessage | AgentMessage[], images?: ImageContent[]): Promise<void> {
	if (this.activeRun) {
		throw new Error(
			"Agent is already processing a prompt. Use steer() or followUp() to queue messages, or wait for completion.",
		);
	}
	const messages = this.normalizePromptInput(input, images);
	await this.runPromptMessages(messages);
}

async continue(): Promise<void> {
	if (this.activeRun) {
		throw new Error("Agent is already processing. Wait for completion before continuing.");
	}
	const lastMessage = this._state.messages[this._state.messages.length - 1];
	if (!lastMessage) throw new Error("No messages to continue from");
	// 检查末尾角色和队列后再运行
	await this.runContinuation();
}
```

### 运行流程与状态变化

第二次 `prompt()` 返回 rejected Promise，不产生 `agent_start`，不把第二条 user message 写进 transcript，也不改变第一次 run 的 error 状态。第一次运行继续执行。测试 `packages/agent/test/agent.test.ts:475-548` 分别验证活动期间的第二次 `prompt()` 和 `continue()` 被拒绝。

想改变当前工作应调用 `steer()`；想排在当前工作完成后应调用 `followUp()`；需要严格的调用方顺序则先 await 前一次 `prompt()` 或 `waitForIdle()`。这三种选择的注入时机不同，不能用捕获并发错误后盲目重试替代。

### 失败路径与设计取舍

锁的生命周期覆盖 `agent_end` 监听器。已经看到 `agent_end` 事件却立即调用新的 `prompt()`，仍可能收到 “already processing”；只有监听器全部完成、`finishRun()` 清掉 `activeRun` 后才可启动新 run。

`continue()` 还有独立前置条件：没有消息，或末尾仍是 assistant 且没有可排出的 steering/follow-up 时，会直接拒绝。这些错误也不进入 transcript，因为它们说明宿主选择了错误的恢复入口。

## 8. 用可观察结果定位失败，而不是只看错误字符串

同一段错误文本可能来自不同层。下面的矩阵可以直接用于调试：

| 场景 | 工具是否执行 | transcript 新增 | `state.errorMessage` | `prompt()` |
| --- | --- | --- | --- | --- |
| `stopReason=length` 且含 tool call | 否，整批拒绝 | assistant + error toolResult；随后通常还有新 assistant | 通常不变 | 正常收束 |
| 未知工具 | 否 | error toolResult | 不变 | 正常收束 |
| 参数校验/preflight 失败 | 否 | error toolResult | 不变 | 正常收束 |
| `tool.execute()` 抛错 | 已进入执行 | 经 `afterToolCall` 后的 error toolResult | 不变 | 正常收束 |
| provider 协议内 error | 否 | assistant error message | 写入错误文本 | 正常收束 |
| 自定义 `StreamFn` 直接抛错，经 `Agent` 运行 | 否 | 合成的 assistant error message | 写入错误文本 | 通常正常收束 |
| 活动期间第二次 `prompt()` | 第二个 run 未启动 | 无 | 不变 | 第二个 Promise reject |
| 监听器持续抛错 | 取决于事件位置 | 可能部分写入 | 可能写入，也可能来不及 | 可能 reject |
| 工具结束后的晚到 update | 工具已结束 | 无 | 不变 | 不受影响 |

“通常”不是含糊处理，而是指出监听器和自定义 hook 能改变外层结算。只要宿主把代码插入 await 链，就要把它纳入失败分析。

### 调试顺序

出现异常时，先找最后一个完整生命周期事件：

1. 有 `tool_execution_start`，没有 end：检查工具是否悬挂、事件监听器是否在 start/update/end 阶段抛错。
2. 有 `tool_execution_end(isError=true)`：查看对应 tool result，判断失败在 preflight、execute 还是 `afterToolCall`。
3. 有 `turn_end.message.stopReason=error`：查看 assistant `errorMessage`，不要再把它当工具失败。
4. 完全没有 `agent_start`：检查并发 prompt、`continue()` 前置条件或更高层参数校验。
5. 有 `agent_end` 但调用尚未返回：检查 awaited `agent_end` 监听器和更高层 session settlement。

事件只能证明运行到达过某个边界，不能单独证明持久化成功。例如状态归约发生在监听器之前，`message_end` 已发出而持久化监听器仍可能失败。

## 9. 测试给出的保证与尚未覆盖的边界

本讲依赖的关键测试形成了以下证据：

| 测试 | 已验证的保证 |
| --- | --- |
| `packages/agent/test/agent-loop.test.ts:310-381` | 长度截断的 tool call 不执行，并生成错误后继续模型循环 |
| `packages/ai/test/validation.test.ts:35-115` | 参数转换的兼容范围，以及无法转换时抛出验证错误 |
| `packages/agent/test/agent.test.ts:126-155` | `Agent` 把直接抛出的 stream 失败补成完整 error 生命周期 |
| `packages/agent/test/agent.test.ts:268-407` | tool promise 结束后的更新被忽略，包括并行批次仍活动时 |
| `packages/agent/test/agent.test.ts:475-548` | 活动 run 期间拒绝第二次 prompt/continue |

在当前基线中，没有专门测试以下组合：未知工具和有效工具处于同一并行批次、`afterToolCall` 抛错后原错误如何保留、Agent 订阅者在每一种事件上抛错、低层 `agentLoop()` 内部 Promise reject 后消费者如何退出。这些行为可以沿源码控制流推出，但不能写成“已有测试保证”。其中低层 EventStream 缺少 rejection 终点尤其值得在修改源码时补回归测试。

## QA：容易混淆的边界

### `prompt()` 没抛异常，是否说明运行成功？

不是。provider error 和大部分工具错误都按协议收束，`prompt()` 可以正常返回。应检查最后的 assistant/tool result、`stopReason`、`isError` 和 session 层的 settled/retry 结果。

### 为什么未知工具不直接抛错？

工具名来自模型输出，模型有机会在看到 `Tool ... not found` 后换用现有工具。把它变成 error tool result，既保持 tool call/result 配对，也保留自我修复路径。

### 参数校验失败后会调用 `afterToolCall` 统一格式吗？

不会。校验失败属于 immediate outcome，没有进入 execute/finalize 路径。需要统一所有工具错误展示时，应处理 `tool_execution_end` 或 tool result 事件，而不能只依赖 `afterToolCall`。

### provider error message 中如果残留 tool call，会执行吗？

不会。`runLoop()` 先检查 assistant `stopReason`，遇到 `error` 或 `aborted` 立即结束 turn，工具提取在这个分支之后。

### 监听器抛错后，刚收到的消息会回滚吗？

不会。`processEvents()` 先更新 AgentState，再调用监听器。数据库或外部状态是否写入则取决于监听器失败的位置，需要宿主自己的事务或幂等策略。

### 为什么晚到的工具更新直接丢弃，而不是报错？

最终结果已经确定，再接受进度会造成“完成后又回到运行中”的倒序事件。忽略迟到回调能维持单调生命周期，也避免工具保留回调后制造未处理 rejection。代价是工具作者不会从回调返回值获知更新被丢弃。

## 本讲主线

Pi 的失败处理不是一条异常冒泡链，而是按可恢复范围分层：工具级错误进入 tool result，让模型继续；provider error 进入 assistant message，结束当前 run；调用前置条件直接 reject，不污染 transcript；监听器和自定义 hook 位于控制面，可能改变正常结算。

阅读失败路径时，错误字符串只能说明表象。事件是否闭合、消息写入了哪种角色、AgentState 是否恢复、外层 Promise 如何结束，四者合起来才是这次失败的真实工程结果。
