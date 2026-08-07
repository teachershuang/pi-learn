# 第 04 讲：工具调用从模型输出到执行结果

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 不会看到 `toolCall` 就立即执行。模型响应必须先完整结束，agent-loop 才从最终 assistant message 中提取调用，依次完成参数兼容、schema 校验、执行前拦截、工具执行、结果拦截和消息归一化。

```text
最终 AssistantMessage
  │ 提取 toolCall content block
  ▼
选择整个批次的执行策略
  │
  ├── prepareArguments        兼容旧参数或模型偏差
  ├── validateToolArguments  克隆、类型转换、schema 校验
  ├── beforeToolCall         修改参数或阻止执行
  ├── tool.execute           执行业务动作并报告进度
  ├── afterToolCall          替换结果字段
  └── ToolResultMessage      写入 transcript，交给下一轮模型
```

这条链上有三种不同形态的数据：模型生成的原始 `ToolCall`、工具真正收到的已校验参数、最终写入会话的 `ToolResultMessage`。把它们都叫“工具参数”或“工具结果”，很容易漏掉中间的修改点。

## 1. `AgentTool` 把模型声明、运行代码和控制提示放在同一份契约里

### 解决的问题

provider 只需要工具名称、说明和参数 schema；agent-loop 还需要执行函数、进度回调、结果详情以及并发策略。`AgentTool` 在 `pi-ai` 的通用 `Tool` 之上补齐运行期能力。

### 源码入口与关键代码

模型输出的调用块定义在 `packages/ai/src/types.ts:349-355`，只有 id、名称和原始 arguments。运行时工具定义在 `packages/agent/src/types.ts:372-395`：

```ts
/** Tool definition used by the agent runtime. */
export interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
	/** Human-readable label for UI display. */
	label: string;
	/**
	 * Optional compatibility shim for raw tool-call arguments before schema validation.
	 * Must return an object that matches `TParameters`.
	 */
	prepareArguments?: (args: unknown) => Static<TParameters>;
	/** Execute the tool call. Throw on failure instead of encoding errors in `content`. */
	execute: (
		toolCallId: string,
		params: Static<TParameters>,
		signal?: AbortSignal,
		onUpdate?: AgentToolUpdateCallback<TDetails>,
	) => Promise<AgentToolResult<TDetails>>;
	/**
	 * Per-tool execution mode override.
	 * - "sequential": this tool must execute one at a time with other tool calls.
	 * - "parallel": this tool can execute concurrently with other tool calls.
	 *
	 * If omitted, the default execution mode applies.
	 */
	executionMode?: ToolExecutionMode;
}
```

工具返回的 `AgentToolResult` 定义在 `packages/agent/src/types.ts:349-370`：

```ts
/** Final or partial result produced by a tool. */
export interface AgentToolResult<T> {
	/** Text or image content returned to the model. */
	content: (TextContent | ImageContent)[];
	/** Arbitrary structured details for logs or UI rendering. */
	details: T;
	/** Names of tools introduced by this result and available from this transcript point onward. */
	addedToolNames?: string[];
	/**
	 * Hint that the agent should stop after the current tool batch.
	 * Early termination only happens when every finalized tool result in the batch sets this to true.
	 */
	terminate?: boolean;
}

/**
 * Callback used by tools to stream partial execution updates.
 *
 * The callback is scoped to the current `execute()` invocation. Calls made after
 * the tool promise settles are ignored.
 */
export type AgentToolUpdateCallback<T = any> = (partialResult: AgentToolResult<T>) => void;
```

### 运行流程与状态变化

`content` 是返回给模型的文本或图片；`details` 主要供日志和 UI 渲染。两者可以表达不同内容。例如文件读取工具可以把文本放进 `content`，把行数、截断信息放进 `details`。

`addedToolNames` 表示从当前 transcript 位置起新出现的工具。`terminate` 只参与本批次的循环决策，不会作为字段写入 `ToolResultMessage`。进度回调使用同一个结果形状，但它产生的是事件，不会直接进入 transcript。

### 失败路径与设计取舍

接口要求工具在失败时抛异常，而不是把错误藏在普通 `content` 中。agent-loop 负责把异常统一变成 `isError: true` 的 tool result。这样 provider 总能收到协议完整的工具结果，UI 也能用明确的错误标志渲染。

`details` 使用泛型，核心层不解释它的结构。这保留了工具自己的渲染数据，但也意味着扩展、导出器和 UI 不能假设所有结果都有同一种 details schema。

## 2. 工具只从最终 assistant message 中启动

### 解决的问题

流式 tool call 的参数会分成多个 delta 到达。在 JSON 尚未闭合时执行，可能得到缺字段、截断字符串或之后还会被覆盖的值。

### 源码入口与关键代码

`streamAssistantResponse()` 等到 provider stream 给出最终结果后，才把 assistant message 返回给 `runLoop()`。随后 `packages/agent/src/agent-loop.ts:202-220` 提取 tool call：

```ts
// Check for tool calls
const toolCalls = message.content.filter((c) => c.type === "toolCall");

const toolResults: ToolResultMessage[] = [];
hasMoreToolCalls = false;
if (toolCalls.length > 0) {
	// A "length" stop means the output was cut off by the token limit, so
	// every tool call in the message may carry truncated arguments. Fail
	// them all instead of executing potentially borked calls.
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
```

### 运行流程与状态变化

tool call 是 assistant message 的 content block，不是独立的顶层消息。agent-loop 保留 assistant message，再把每个执行结果作为 `role: "toolResult"` 的消息追加到上下文。下一次模型请求因此看到完整的调用与结果配对。

流式阶段的 `toolcall_start/toolcall_delta/toolcall_end` 只更新 assistant message 和界面。真实执行发生在整个响应完成之后。

### 失败路径与设计取舍

`stopReason === "length"` 时，已经被补救解析器拼成合法 JSON 的 arguments 仍可能语义残缺。Pi 不执行该 assistant message 中的任何工具，而是给每个调用生成错误结果，让模型下一轮重新发起完整调用。

这个判断覆盖整个 assistant message，不逐项猜测哪个调用可能完整。它牺牲了部分可执行调用，换来不让截断参数触发文件修改、命令执行等副作用。

## 3. 参数准备发生在 schema 校验之前

### 解决的问题

模型或旧会话可能使用已经废弃但仍可无歧义转换的参数形状。直接放宽当前 schema 会让工具长期承担多套输入；直接拒绝又会增加不必要的失败轮次。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:588-665` 先调用工具自己的兼容函数，再校验返回值：

```ts
function prepareToolCallArguments(tool: AgentTool<any>, toolCall: AgentToolCall): AgentToolCall {
	if (!tool.prepareArguments) {
		return toolCall;
	}
	const preparedArguments = tool.prepareArguments(toolCall.arguments);
	if (preparedArguments === toolCall.arguments) {
		return toolCall;
	}
	return {
		...toolCall,
		arguments: preparedArguments as Record<string, any>,
	};
}

async function prepareToolCall(
	currentContext: AgentContext,
	toolCall: AgentToolCall,
	// ...assistantMessage、config、signal
) {
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
		// ...进入 beforeToolCall，成功后返回 prepared 结果
	} catch (error) {
		return {
			kind: "immediate",
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
		};
	}
}
```

片段省略了下一节单独展开的 before hook，并保留了参数准备、校验和异常归一化的顺序。

真正的校验器位于 `packages/ai/src/utils/validation.ts:278-309`。它先 `structuredClone()`，再用 TypeBox 的 `Value.Convert()` 做类型转换；若 schema 是序列化后的普通 JSON Schema，还会执行兼容的基本类型 coercion，最后才检查约束并生成带字段路径的错误信息。

coding-agent 的 edit 工具是实际用例。`packages/coding-agent/src/core/tools/edit.ts:94-117` 会把字符串形式的 `edits` 解析成数组，也会把旧版 `oldText/newText` 转成当前的 `edits[]`，随后仍由当前 schema 判定是否合法。

### 运行流程与状态变化

这一阶段的顺序是：按名称找工具，调用 `prepareArguments`，克隆并转换参数，执行 schema 校验。成功后产生 `validatedArgs`；失败则产生 `ImmediateToolCallOutcome`，不会进入真实 `execute()`。

校验可能做类型转换。测试 `packages/ai/test/validation.test.ts:35-115` 固定了部分行为，例如字符串 `"42"` 可以转成 number，字符串 `"1"` 不能转成 boolean。schema 校验不是简单的 `typeof` 比较。

### 失败路径与设计取舍

工具不存在、兼容函数抛错或 schema 校验失败，都会被归一化为错误 tool result。assistant 仍能在下一轮看到具体失败原因并改正调用，整个 agent run 不会因为一次参数错误直接崩溃。

`prepareArguments` 是窄范围兼容层，不是第二套校验器。它返回的内容仍必须通过当前 schema。若兼容规则有多种解释，自动改写会掩盖模型错误，此时保留校验失败更安全。

## 4. `beforeToolCall` 是执行前的受信任控制点

### 解决的问题

schema 只能判断参数形状，无法决定本次操作是否被允许。宿主或扩展需要结合当前策略检查命令、路径和操作类型，也可能修正已经校验的参数。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:621-658` 在校验后调用 hook：

```ts
if (config.beforeToolCall) {
	const beforeResult = await config.beforeToolCall(
		{
			assistantMessage,
			toolCall,
			args: validatedArgs,
			context: currentContext,
		},
		signal,
	);
	if (signal?.aborted) {
		return {
			kind: "immediate",
			result: createErrorToolResult("Operation aborted"),
			isError: true,
		};
	}
	if (beforeResult?.block) {
		return {
			kind: "immediate",
			result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
			isError: true,
		};
	}
}
if (signal?.aborted) {
	return {
		kind: "immediate",
		result: createErrorToolResult("Operation aborted"),
		isError: true,
	};
}
return {
	kind: "prepared",
	toolCall,
	tool,
	args: validatedArgs,
};
```

### 运行流程与状态变化

hook 的输入同时包含两份参数信息：`toolCall` 是 assistant message 中的原始调用块，`args` 是兼容处理和 schema 校验后的对象。工具 `execute()` 最终收到后者。

返回 `{ block: true, reason }` 时，工具不会执行。agent-loop 生成 `isError: true` 的 tool result，把 reason 返回给模型。阻止执行属于一次可恢复的工具失败，不会直接结束 run。

hook 也可以原地修改 `args`。修改后的对象直接交给工具，不再做第二次 schema 校验。`packages/agent/test/agent-loop.test.ts:383-443` 把原本符合 string schema 的值改成 number，工具实际收到的就是 number。

### 失败路径与设计取舍

hook 抛错、返回 block，或者执行 hook 期间 AbortSignal 被触发，都会走 immediate error 分支。`afterToolCall` 不会处理这些结果，因为工具从未进入执行阶段。

“校验后允许修改且不重验”让扩展可以做强力的参数重写，也打破了 TypeScript 泛型在运行时的保证。工具不能把第三方 `beforeToolCall` 当作普通观察者。修改参数的扩展属于受信任代码，应自行维持 schema 和安全约束。

## 5. 工具异常和进度更新在执行层归一化

### 解决的问题

工具可能运行很久，需要持续更新 UI；也可能抛出任意类型的异常。进度不能污染 transcript，异常又必须形成 provider 能理解的 tool result。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:673-708` 包住一次真实执行：

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
			updateEvents.push(
				Promise.resolve(
					emit({
						type: "tool_execution_update",
						toolCallId: prepared.toolCall.id,
						toolName: prepared.toolCall.name,
						args: prepared.toolCall.arguments,
						partialResult,
					}),
			),
			);
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

每次 `onUpdate(partialResult)` 产生 `tool_execution_update`。agent-loop 保存这些 emit promise，并在工具结束后等待已经发出的更新处理完，再进入最终结果阶段。工具 promise 一旦 settled，`acceptingUpdates` 变为 false；之后误用旧回调不会产生迟到事件。

`packages/agent/test/agent.test.ts:268-406` 验证了两种迟到更新：整个 prompt 已结束后调用旧回调，以及并行批次中该工具已结束、另一工具仍运行时调用旧回调。两者都被忽略。

`tool_execution_start` 的 `args` 是参数准备前的原始值。update 事件读取原 tool call 对象上的 arguments，不使用 `validatedArgs`；若 `prepareArguments` 曾原地修改该对象，update 会看到修改后的值。`execute()` 收到的是已校验且可能被 hook 修改的 `prepared.args`。UI 不能仅根据 start 或 update 事件推定实际执行参数。

### 失败路径与设计取舍

`execute()` 抛出的 `Error` 使用 message，其他抛出值使用 `String(error)`，随后包装成文本错误结果。这个错误还会进入 `afterToolCall`，因此宿主可以统一修改错误文本、details 或 `isError`。

进度事件不写入 `AgentState.messages`。它们适合瞬时 UI，不适合作为下轮模型上下文；最终返回值才会生成持久化的 tool result message。

## 6. `afterToolCall` 修改最终结果，不重跑工具

### 解决的问题

工具已经执行后，宿主可能需要脱敏输出、补充渲染信息、改变错误标志，或者声明本批次不必再次请求模型。此时重新包装整个工具会重复执行适配逻辑。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:719-754` 对字段做浅层覆盖：

```ts
let result = executed.result;
let isError = executed.isError;

if (config.afterToolCall) {
	try {
		const afterResult = await config.afterToolCall(
			{
				assistantMessage,
				toolCall: prepared.toolCall,
				args: prepared.args,
				result,
				isError,
				context: currentContext,
			},
			signal,
		);
		if (afterResult) {
			result = {
				...result,
				content: afterResult.content ?? result.content,
				details: afterResult.details ?? result.details,
				terminate: afterResult.terminate ?? result.terminate,
			};
			isError = afterResult.isError ?? isError;
		}
	} catch (error) {
		result = createErrorToolResult(error instanceof Error ? error.message : String(error));
		isError = true;
	}
}

return {
	toolCall: prepared.toolCall,
	result,
	isError,
};
```

### 运行流程与状态变化

hook 看见工具执行后的原始 result 和当前 `isError`。它可以分别替换 `content`、`details`、`isError`、`terminate`；未提供的字段保留原值。`content` 和 `details` 都是整体替换，没有深合并。

`afterToolCall` 在 `tool_execution_end` 之前运行，所以 end 事件、后续 tool result message 和批次终止判断看到的都是修改后结果。它不会重新调用工具。

### 失败路径与设计取舍

工具执行抛错时仍会进入 after hook。hook 可以把底层异常改成适合模型阅读的错误，也可以改变 `isError`。若 hook 自己抛错，原工具结果会被丢弃，替换成 hook 异常对应的错误结果。

这种覆盖能力适合统一脱敏和策略收尾，也意味着 hook 可能把真实工具失败标成成功。审计工具链时，不能只看 `execute()` 的返回值，还要检查 `afterToolCall` 的安装者。

## 7. 并行模式分开了预检顺序、完成顺序和消息顺序

### 解决的问题

只读或互不影响的工具可以并发执行，但写文件、修改共享状态的工具需要串行。即使并发，写入 transcript 的 tool result 仍需匹配 assistant message 中的调用顺序。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:413-427` 对整个批次选择一种策略：

```ts
async function executeToolCalls(
	currentContext: AgentContext,
	assistantMessage: AssistantMessage,
	config: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
	const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
	const hasSequentialToolCall = toolCalls.some(
		(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
	);
	if (config.toolExecution === "sequential" || hasSequentialToolCall) {
		return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
	}
	return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
}
```

并行分支在 `packages/agent/src/agent-loop.ts:491-555` 等全部预检完成后启动可执行项，再按原顺序生成消息：

```ts
for (const toolCall of toolCalls) {
	// ...按源码顺序发出 tool_execution_start
	const preparation = await prepareToolCall(currentContext, assistantMessage, toolCall, config, signal);
	if (preparation.kind === "immediate") {
		// ...立即发出 error end，并把结果放回 finalizedCalls
		continue;
	}
	finalizedCalls.push(async () => {
		const executed = await executePreparedToolCall(preparation, signal, emit);
		const finalized = await finalizeExecutedToolCall(
			currentContext,
			assistantMessage,
			preparation,
			executed,
			config,
			signal,
		);
		await emitToolExecutionEnd(finalized, emit);
		return finalized;
	});
}

const orderedFinalizedCalls = await Promise.all(
	finalizedCalls.map((entry) => (typeof entry === "function" ? entry() : Promise.resolve(entry))),
);
const messages: ToolResultMessage[] = [];
for (const finalized of orderedFinalizedCalls) {
	const toolResultMessage = createToolResultMessage(finalized);
	await emitToolResultMessage(toolResultMessage, emit);
	messages.push(toolResultMessage);
}

return {
	messages,
	terminate: shouldTerminateToolBatch(orderedFinalizedCalls),
};
```

片段折叠了 immediate error 分支，只保留“预检收集任务、并发等待、按原顺序发消息”的结构。

### 运行流程与状态变化

默认模式是 parallel，但“并行”有三段顺序：

1. agent-loop 按 assistant message 中的顺序发出 start 事件并逐个完成工具查找、参数校验和 `beforeToolCall`。
2. 通过预检的执行函数交给 `Promise.all()` 并发运行。`tool_execution_end` 在各工具真正完成时发出，所以顺序可以变化。
3. `Promise.all()` 返回值保持输入顺序。tool result 的 `message_start/message_end` 和 `turn_end.toolResults` 因此仍按模型调用顺序排列。

测试 `packages/agent/test/agent-loop.test.ts:525-618` 让第二个工具先结束，观察到 end 事件顺序是 `tool-2, tool-1`，持久化结果顺序仍是 `tool-1, tool-2`。

只要批次中有一个目标工具声明 `executionMode: "sequential"`，整个批次就走串行分支；Pi 没有在同一批里建立“这个工具串行，其他工具互相并行”的调度图。`packages/agent/test/agent-loop.test.ts:726-968` 覆盖了全局默认、单工具覆盖和全部允许并行三种情况。

### 失败路径与设计取舍

串行分支会在每个工具结束后检查 abort，并停止启动后续调用。并行分支把同一个 AbortSignal 传给所有已启动工具，工具实现仍要主动响应 signal。

批次级降级为串行很保守，却避免了隐式推断工具之间的读写依赖。更细的并发需要显式资源锁或依赖图；当前实现把这类协调留给工具本身，例如 edit 工具还会按文件路径使用 mutation queue。

## 8. 事件更新运行态，tool result 才进入 transcript

### 解决的问题

UI 需要知道哪些工具正在运行，模型需要稳定的最终结果。瞬时运行态和会话历史不能靠同一组对象表达。

### 源码入口与关键代码

`packages/agent/src/agent.ts:537-553` 根据事件维护 `pendingToolCalls`：

```ts
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
```

最终消息由 `packages/agent/src/agent-loop.ts:774-791` 创建：

```ts
function createToolResultMessage(finalized: FinalizedToolCallOutcome): ToolResultMessage {
	return {
		role: "toolResult",
		toolCallId: finalized.toolCall.id,
		toolName: finalized.toolCall.name,
		// Untyped tools (JS extensions) can return results without content; normalize
		// so the null never enters session history or provider payloads.
		content: finalized.result.content ?? [],
		details: finalized.result.details,
		...(finalized.result.addedToolNames?.length ? { addedToolNames: finalized.result.addedToolNames } : {}),
		isError: finalized.isError,
		timestamp: Date.now(),
	};
}

async function emitToolResultMessage(toolResultMessage: ToolResultMessage, emit: AgentEventSink): Promise<void> {
	await emit({ type: "message_start", message: toolResultMessage });
	await emit({ type: "message_end", message: toolResultMessage });
}
```

### 运行流程与状态变化

单个成功工具的主要事件顺序是：

```text
tool_execution_start       pendingToolCalls 加入 id
tool_execution_update*     UI 获得零个或多个进度结果
tool_execution_end         pendingToolCalls 删除 id
message_start(toolResult)
message_end(toolResult)    AgentState.messages 追加最终消息
```

`tool_execution_start` 在工具查找和参数校验之前发出。因此未知工具、参数错误和被 hook 阻止的调用也会短暂进入 pending 集合，随后用 error end 事件移除。

tool result message 保留调用 id、工具名、content、details、`addedToolNames` 和 `isError`，并添加时间戳。未类型化的 JavaScript 扩展若错误地省略 content，边界会将它归一化成空数组，避免 null 进入 session 或 provider payload。

### 失败路径与设计取舍

`terminate` 不写入 transcript。它是 agent-loop 的控制提示，不是要交给模型解释的业务结果。下一轮是否发生由当前批次立即决定；以后从 session 恢复时不会根据历史 tool result 再执行一次终止逻辑。

start/end 事件与 tool result message 分开，使并行 UI 能按完成时间刷新，同时保持 provider 上下文的调用顺序。消费事件的代码不能拿 `tool_execution_end` 顺序重建 transcript。

## 9. coding-agent 用同一对 hook 接入扩展

### 解决的问题

扩展需要拦截内置工具和扩展工具。如果每种工具包装器各自实现拦截，重载扩展时容易出现行为不一致，普通 `AgentTool` override 也可能绕过策略。

### 源码入口与关键代码

`packages/coding-agent/src/core/agent-session.ts:449-469` 把扩展的 `tool_call` 事件安装为 Agent 的 before hook：

```ts
this.agent.beforeToolCall = async ({ toolCall, args }) => {
	const runner = this._extensionRunner;
	if (!runner.hasHandlers("tool_call")) {
		return undefined;
	}

	try {
		return await runner.emitToolCall({
			type: "tool_call",
			toolName: toolCall.name,
			toolCallId: toolCall.id,
			input: args as Record<string, unknown>,
		});
	} catch (err) {
		if (err instanceof Error) {
			throw err;
		}
		throw new Error(`Extension failed, blocking execution: ${String(err)}`);
	}
};
```

同一方法在 `packages/coding-agent/src/core/agent-session.ts:471-496` 把 `tool_result` 事件接到 after hook，允许扩展替换 content、details 和 `isError`。

### 运行流程与状态变化

扩展按照注册顺序处理 `tool_call`。后面的处理器能看到前面对 `event.input` 的原地修改；任一处理器返回 block，runner 立即停止并把结果交给 agent-loop。扩展类型在 `packages/coding-agent/src/core/extensions/types.ts:885-899` 明确说明参数修改后不再校验。

`tool_result` 处理器也是依次运行，后面的处理器基于前面修改后的结果继续处理。它的公开返回类型只有 content、details 和 `isError`；core `afterToolCall` 支持的 `terminate` 没有暴露给 coding-agent 的扩展事件。

### 失败路径与设计取舍

`tool_call` 处理器抛错会向上传到 before hook，最终阻止工具并生成错误结果。`tool_result` 处理器抛错则由 ExtensionRunner 记录扩展错误并继续其他处理器，不会覆盖已经执行成功的工具结果。两类异常策略不同：执行前偏向禁止不确定操作，执行后偏向保留已有结果。

集中安装 hook 后，内置工具、扩展工具和外部传入的 `AgentTool` 走同一拦截点。hook 在运行时读取当前 ExtensionRunner，扩展重载不需要重新包装所有工具。

## 10. 失败归一化表

| 失败位置 | `execute()` 是否运行 | `afterToolCall` 是否运行 | 写入模型上下文的结果 |
| --- | --- | --- | --- |
| assistant response 因 length 截断 | 否 | 否 | 参数可能截断的错误 tool result |
| 工具名不存在 | 否 | 否 | tool not found 错误 |
| `prepareArguments` 抛错 | 否 | 否 | 异常文本错误 |
| schema 校验失败 | 否 | 否 | 带字段路径和原始参数的错误 |
| `beforeToolCall` 返回 block | 否 | 否 | block reason 或默认阻止文本 |
| before hook 抛错 | 否 | 否 | hook 异常文本错误 |
| 执行前 signal 已 aborted | 否 | 否 | `Operation aborted` |
| `execute()` 抛错 | 是 | 是 | 执行异常，可被 after hook 改写 |
| `afterToolCall` 抛错 | 已运行 | 已进入但失败 | after hook 异常替换原结果 |
| 未类型化工具省略 content | 是 | 是 | content 归一化为空数组 |

这些失败通常不会作为未捕获异常离开 agent-loop。它们会形成正常的 `tool_execution_end`、tool result 消息和 `turn_end`，让模型在下一轮修正。事件订阅器或循环配置回调自身破坏契约时，才会进入更外层的 run failure 处理。

## 11. 一次双工具调用的完整状态流

假设 assistant message 依次请求 `read` 和一个声明为 sequential 的 `edit` 工具：

```text
assistant message_end
  content = [toolCall(read), toolCall(edit)]

executeToolCalls()
  发现 edit.executionMode = sequential
  整批改走串行

read
  start(raw args)                    pending = {read}
  prepareArguments
  schema validation
  beforeToolCall(validated args)
  execute(onUpdate*)
  afterToolCall
  end(final result)                  pending = {}
  message_end(toolResult: read)      transcript 追加 read 结果

edit
  start(raw args)                    pending = {edit}
  prepareEditArguments
  schema validation
  beforeToolCall(validated args)
    扩展可以 block 或原地修改
  execute
  afterToolCall
  end(final result)                  pending = {}
  message_end(toolResult: edit)      transcript 追加 edit 结果

turn_end(toolResults = [read, edit])
下一次 LLM 请求读取两个结果
```

这个例子中，read 本来可以并行，仍被 edit 的 sequential 声明拖入串行批次。执行策略以整个 assistant response 中的工具集合为单位，不是逐个工具独立决定。

## 12. 常见误解

### schema 校验通过后，工具参数就不会再变吗？

会变。`beforeToolCall` 和 coding-agent 的 `tool_call` 扩展拿到校验后的可变对象，可以原地修改，而且不会重新校验。这是明确的扩展能力，也是需要审计的信任边界。

### parallel 是否表示所有步骤都并行？

不是。工具查找、参数准备、校验和 before hook 按调用顺序完成；真正的 `execute()` 才并发。end 事件按完成顺序，tool result 消息按模型调用顺序。

### 工具返回错误文本是否等于工具失败？

不一定。核心层依据 `isError` 判断错误。普通工具通过抛异常得到 `isError: true`；after hook 还能修改这个标志。只看 content 中是否出现“error”无法可靠判断。

### 被策略阻止的工具会让 Agent 直接停止吗？

不会。阻止结果被编码成 error tool result，模型可以读取 reason 并调整方案。若策略要求本轮结束后不再请求模型，还需要配合 `terminate` 或 `shouldStopAfterTurn` 的控制语义。
