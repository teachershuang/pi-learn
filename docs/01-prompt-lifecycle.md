# 第 01 讲：一条 prompt 的端到端主链

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 处理一条 prompt，不是从输入框直接调用模型。当前 CLI 主链分成四段：启动层选择运行模式，`AgentSession` 做应用级预处理，`Agent` 驱动低层循环，模型运行时再把统一请求交给 provider。返回方向则依靠事件：同一批事件会更新 Agent 内存状态、写入会话，并被 TUI、JSON 或 RPC 消费。

```text
用户输入
  │
  ▼
cli.ts → main() → interactive / print / json / RPC
                         │
                         ▼
                  AgentSession.prompt()
                         │ 预处理、校验、扩展
                         ▼
                    Agent.prompt()
                         │ 状态快照
                         ▼
                    runAgentLoop()
                         │ Context
                         ▼
              ModelRuntime.streamSimple()
                         │ 认证、模型与 provider 分派
                         ▼
                  Provider event stream
                         │
                         ▼
             AgentEvent → 状态 / 会话 / 界面
```

这张图故意没有展开工具执行。模型若返回 tool call，低层循环会执行工具并再次请求模型；这是第 04 讲的主题。本课只确认 prompt 如何进入循环，以及结果怎样返回调用者。

## 1. 启动层只决定“由谁提交和消费 prompt”

### 解决的问题

同一个编码代理要支持交互终端、一次性文本输出、JSON 事件流和双向 RPC。四种模式不应各自实现模型调用，否则认证、工具、会话和错误语义会分叉。

### 源码入口与关键代码

`packages/coding-agent/src/cli.ts:8-20` 是可执行程序的源码入口。它完成进程级初始化后，把参数交给 `main()`。真正的模式分流位于 `packages/coding-agent/src/main.ts:809-855`：

```ts
if (appMode === "rpc") {
	printTimings();
	await runRpcMode(runtime);
} else if (appMode === "interactive") {
	const interactiveMode = new InteractiveMode(runtime, {
		migratedProviders,
		modelFallbackMessage,
		autoTrustOnReloadCwd,
		initialMessage,
		initialImages,
		initialMessages: parsed.messages,
		verbose: parsed.verbose,
	});
	printTimings();
	await interactiveMode.run();
} else {
	const exitCode = await runPrintMode(runtime, {
		mode: toPrintOutputMode(appMode),
		messages: parsed.messages,
		initialMessage,
		initialImages,
	});
	// ...恢复 stdout，并设置退出码
}
```

### 运行流程与状态变化

`main()` 在分流前已经解析参数、选定或新建 `SessionManager`、确定会话 cwd、加载项目资源并创建 `AgentSessionRuntime`。模式层拿到的是同一个运行时对象：

- interactive 从编辑器接收输入，调用 `session.prompt()`，再把 session 事件投影为终端组件；
- print 依次 `await session.prompt()`，最后从 `session.state.messages` 取最终 assistant 文本；
- json 订阅 session 事件，每个事件写成一行 JSON；
- RPC 把命令转为 session 操作，并把事件写回协议输出。

运行模式改变输入输出协议，不改变 Agent 循环。

### 失败路径与设计取舍

模式确定后，非交互模式会接管 stdout，避免扩展或日志破坏 JSON/RPC 协议。运行时诊断中只要有 error，`main()` 会在提交 prompt 前退出。非交互模式没有可用模型也会直接报错；interactive 可以继续进入界面，让用户完成模型选择或登录。

另一种实现是让每种模式自行创建模型客户端。这会省掉一层运行时抽象，却会复制会话恢复和资源加载逻辑。Pi 选择共享 `AgentSessionRuntime`，模式层只处理交互差异。

## 2. `createAgentSession()` 把分层对象接成可运行实例

### 解决的问题

低层 `Agent` 不认识配置文件、项目资源、会话树和 provider 注册。coding-agent 需要把这些对象接入 Agent，同时保留低层循环的独立性。

### 源码入口与关键代码

`main()` 先用 `createAgentSessionServices()` 创建 cwd 绑定的 `ModelRuntime`、`SettingsManager` 和 `ResourceLoader`，再由 `createAgentSessionFromServices()` 进入 `packages/coding-agent/src/core/sdk.ts:createAgentSession()`。Agent 的模型调用函数在这里注入：

```ts
agent = new Agent({
	initialState: {
		systemPrompt: "",
		model,
		thinkingLevel,
		tools: [],
	},
	convertToLlm: convertToLlmWithBlockImages,
	streamFn: async (model, context, options) => {
		const providerRetrySettings = settingsManager.getProviderRetrySettings();
		const httpIdleTimeoutMs = settingsManager.getHttpIdleTimeoutMs();
		const effectiveTimeoutMs = httpIdleTimeoutMs === 0 ? 2147483647 : httpIdleTimeoutMs;
		const timeoutMs = options?.timeoutMs ?? providerRetrySettings.timeoutMs ?? effectiveTimeoutMs;
		return modelRuntime.streamSimple(model, context, {
			...options,
			timeoutMs,
			maxRetries: options?.maxRetries ?? providerRetrySettings.maxRetries,
			// ...websocket timeout、retry delay 与 header transform
		});
	},
	transformContext: async (messages) => {
		const runner = extensionRunnerRef.current;
		return runner ? runner.emitContext(messages) : messages;
	},
});
```

代码位置：`packages/coding-agent/src/core/sdk.ts:289-355`。片段省略了 websocket timeout、provider header 扩展和响应观察回调。

### 运行流程与状态变化

创建过程先从会话恢复模型、thinking level 和消息；新会话则把初始模型与 thinking level 写入 `SessionManager`。随后 `AgentSession` 接管这些依赖，并订阅 Agent 事件。

这里出现了一个重要的控制反转：`packages/agent` 只要求一个 `streamFn(model, context, options)`，不知道请求最终由哪个 provider 发出。coding-agent 注入 `ModelRuntime.streamSimple()`，同时把配置中的超时、重试、传输方式和扩展回调带入请求。

### 失败路径与设计取舍

资源加载、扩展注册和模型选择的错误会先形成 runtime diagnostics。恢复旧会话时若原模型不可用，代码尝试选择可用模型，并保留 fallback 提示。无法选择模型时仍可创建 session，但真正提交 prompt 会失败。

依赖注入让 agent-loop 可以用 faux stream 做单测，也允许 SDK 调用者替换服务。代价是调用链不能只靠函数名阅读：要回到 `new Agent({...})` 才能知道 `streamFn`、上下文转换和认证由谁提供。

## 3. `AgentSession.prompt()` 是应用级闸门

### 解决的问题

用户输入在进入低层循环前，还可能是扩展命令、skill 命令或 prompt template；运行中的第二条输入也不能被当成新的并发 run。模型和认证必须在消耗 provider 请求前完成检查。

### 源码入口与关键代码

`packages/coding-agent/src/core/agent-session.ts:1102-1253` 依次处理命令、input 扩展、skill/template 展开、流式期间排队、模型与认证校验、压缩检查和消息构造。正常路径的末段如下：

```ts
if (!this.model) {
	throw new Error(formatNoModelSelectedMessage());
}

const hasConfiguredAuth =
	this._modelRuntime.hasConfiguredAuth(this.model.provider) ||
	(await this._modelRuntime.checkAuth(this.model.provider)) !== undefined;
if (!hasConfiguredAuth) {
	const isOAuth = this._modelRuntime.isUsingOAuth(this.model.provider);
	if (isOAuth) {
		throw new Error(
			`Authentication failed for "${this.model.provider}". ` +
				`Credentials may have expired or network is unavailable. ` +
				`Run '/login ${this.model.provider}' to re-authenticate.`,
		);
	}
	throw new Error(formatNoApiKeyFoundMessage(this.model.provider));
}

// ...发送前的 compaction 检查
messages = [];
const userContent: (TextContent | ImageContent)[] = [
	{ type: "text", text: expandedText },
];
if (currentImages) userContent.push(...currentImages);
messages.push({
	role: "user",
	content: userContent,
	timestamp: Date.now(),
});

preflightResult?.(true);
await this._runAgentPrompt(messages);
```

### 运行流程与状态变化

普通文本会变成一条带时间戳的 `AgentMessage`。图片与文本放在同一 user content 数组中。`before_agent_start` 扩展还可以补充 custom message 或临时替换本轮 system prompt，然后 `_runAgentPrompt()` 才调用低层 `Agent.prompt()`。

如果 Agent 正在运行，调用者必须声明 `streamingBehavior`：`steer` 在当前 assistant turn 和工具批次结束后、下一次模型调用前注入；`followUp` 等当前循环本来要停止时再开始后续输入。没有声明就抛错，不会静默制造第二个并发循环。

### 失败路径与设计取舍

扩展命令如果被识别，会直接执行并返回，不消耗 provider response。skill 文件读取失败会报告扩展错误，并把原始文本继续送入；缺少模型或认证则在构造正式 run 前抛错。预处理失败不会追加 user message。

把预处理放在 `AgentSession` 而不是 `Agent`，使低层包不依赖 coding-agent 的命令和资源系统。相应地，直接使用 `Agent.prompt()` 的调用者不会自动获得 template、skill、会话压缩和认证预检。

## 4. `Agent` 固定一次 run 的状态和完成边界

### 解决的问题

agent-loop 接收的是一次运行的快照；对外暴露的 `Agent` 则要防止并发 prompt，维护流式消息、最终消息和中止信号，并确保监听器看到一致的事件顺序。

### 源码入口与关键代码

`packages/agent/src/agent.ts:337-409` 把输入标准化，然后把状态快照、循环配置和事件 sink 交给 `runAgentLoop()`：

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

private async runPromptMessages(
	messages: AgentMessage[],
	options: { skipInitialSteeringPoll?: boolean } = {},
): Promise<void> {
	await this.runWithLifecycle(async (signal) => {
		await runAgentLoop(
			messages,
			this.createContextSnapshot(),
			this.createLoopConfig(options),
			(event) => this.processEvents(event),
			signal,
			this.streamFn,
		);
	});
}
```

### 运行流程与状态变化

`runWithLifecycle()` 创建本次 run 的 `AbortController`，将 `isStreaming` 设为 true，并清空上次错误。`processEvents()` 先归约内部状态，再逐个 `await` Agent 监听器：

- `message_start/update` 更新 `streamingMessage`；
- `message_end` 清空流式对象，并把最终消息加入 `state.messages`；
- tool start/end 更新 `pendingToolCalls`；
- `turn_end` 记录 assistant error；
- `agent_end` 表示循环不再产生新事件，但此时 run 尚未立刻变成 idle。

所有 `agent_end` 监听器完成后，`finishRun()` 才清空运行态并解析 `waitForIdle()` 使用的 promise。

### 失败路径与设计取舍

若 `streamFn` 或循环内部直接抛出异常，`Agent` 不让事件序列半途消失。它合成一条 `stopReason: "error"` 或 `"aborted"` 的 assistant message，再补发 `message_start → message_end → turn_end → agent_end`。因此 `Agent.prompt()` 对这类运行失败通常正常 resolve，错误作为消息状态返回；“并发 prompt”一类前置条件错误仍会 reject。

监听器被 await 可以保证 AgentSession 的扩展处理和会话写入完成后再结束 run，但慢监听器会直接拖慢 prompt。Pi 在这里选择了确定顺序，而不是无等待的事件广播。

## 5. provider stream 被翻译成 Agent 事件

### 解决的问题

provider 返回的是增量文本、thinking、tool call 和结束事件。Agent 上层需要稳定的 `message_start/update/end`，不能让每个界面理解各家的流协议。

### 源码入口与关键代码

`packages/agent/src/agent-loop.ts:281-373` 的 `streamAssistantResponse()` 先转换上下文，再消费统一流：

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

const streamFunction = streamFn || streamSimple;
const resolvedApiKey =
	(config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) ||
	config.apiKey;

const response = await streamFunction(config.model, llmContext, {
	...config,
	apiKey: resolvedApiKey,
	signal,
});
```

### 运行流程与状态变化

一轮无工具响应的完整事件序列是：

```text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant)
message_update(assistant) × N
message_end(assistant)
turn_end
agent_end
```

`runAgentLoop()` 先把 user message 加入本次 context，再请求 assistant。若 assistant 包含 tool call，循环执行工具、追加 tool result，然后进入下一次 `turn_start` 和模型请求，直到没有工具、没有 steering/follow-up，或遇到终止条件。

在当前 coding-agent 主链中，`ModelRuntime.streamSimple()` 先准备认证、模型和 provider options，再调用 `provider.streamSimple()`。`packages/ai/src/api/lazy.ts:41-60` 的 `lazyStream()` 会立即返回外层流，异步完成认证或模块加载；准备失败会变成一个 error stream 和最终 error message，而不是在创建流时同步抛出。

### 失败路径与设计取舍

流以 `error` 事件结束时，`streamAssistantResponse()` 仍调用 `response.result()` 取得结构化 assistant error message。低层循环看到 `stopReason` 为 error 或 aborted 后发出 `turn_end`、`agent_end` 并停止。若 provider 代码在统一流之外直接抛错，则由上一节的 `handleRunFailure()` 补齐事件。

统一事件流让 agent-loop 不依赖 SSE、WebSocket 或某个 SDK 的回调格式。代价是错误有两条归一化入口：能进入 `AssistantMessageEventStream` 的失败由流结束，意外抛出的失败由 Agent 合成消息。调用者不能只用 `try/catch` 判断模型调用是否成功，还要检查最终 assistant message 的 `stopReason`。

## 6. 同一事件如何进入内存、会话和界面

### 解决的问题

流式界面需要立即看到增量，会话文件只应保存完成的消息，Agent 内存状态又要在监听器执行前保持最新。三个消费者对时机的要求不同。

### 源码入口与关键代码

`Agent.processEvents()` 先修改内存，再 await 它的监听器。`AgentSession` 在构造时注册 `_handleAgentEvent()`；该处理器先运行扩展事件，接着同步通知 session 订阅者，并在 `message_end` 时落盘：

```ts
await this._emitExtensionEvent(event);

this._emit(event.type === "agent_end" ? { ...event, willRetry: this._willRetryAfterAgentEnd(event) } : event);

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

代码位置：`packages/coding-agent/src/core/agent-session.ts:597-644`。

### 运行流程与状态变化

一条 assistant delta 只更新 Agent 的 `streamingMessage`，并通知界面刷新，不写 JSONL。到了 `message_end`：

1. Agent 把最终 message 放入内存 transcript；
2. Agent await `AgentSession._handleAgentEvent()`；
3. AgentSession 让扩展观察事件，通知模式层，然后由 `SessionManager.appendMessage()` 持久化最终对象；
4. `agent_end` 之后，AgentSession 还可能自动重试、自动压缩或处理 extension 新排入的消息；这些都结束后才发 `agent_settled`。

TUI 在 `message_start` 创建 assistant 组件，在 `message_update` 替换其内容。print text 模式不打印 delta，而是在 `session.prompt()` 返回后读取最后一条 assistant message。json 和 RPC 则输出事件流。

### 失败路径与设计取舍

`AgentSession._emit()` 对模式层监听器是同步调用，但不 await 监听器返回的 promise。会话持久化本身仍位于被 Agent await 的 `_handleAgentEvent()` 内。RPC 为 stdout 背压另外订阅底层 `session.agent`，把等待写入 Agent 的完成边界；这说明“session 订阅者”和“Agent 订阅者”不能混为一类。

只在 `message_end` 落盘避免了每个 token 都写会话文件，也意味着进程在流式中途被强杀时，屏幕上已出现的 partial assistant message 可能没有对应的完成记录。这是减少写放大与保存中间态之间的取舍。

## 7. `prompt()` 何时算完成

`Agent.prompt()` 完成，表示本次低层循环已经发出 `agent_end`，Agent 的异步监听器也已完成。`AgentSession.prompt()` 的边界更晚：`_runAgentPrompt()` 还会处理自动重试、自动压缩和 extension 在结束阶段加入的后续消息，最后发出 `agent_settled`。

因此，“模型停止生成”“Agent 发出 agent_end”“AgentSession settled”是三个时刻。一次性 print 模式依赖最晚的边界，才能安全读取最终消息并退出。交互模式无需等到 settled，事件到达时就会刷新界面。

## 8. 测试怎样约束这条主链

`packages/coding-agent/test/suite/agent-session-prompt.test.ts:30-78` 用 faux provider 固定两种输入：普通文本响应得到 `[user, assistant]`；tool call 响应得到 `[user, assistant, toolResult, assistant]`，并确认 `await session.prompt()` 会等到工具后的第二次模型响应结束。

同一测试文件的 `337-396` 构造一个被 promise 阻塞的工具，确认运行期间再次调用 `session.prompt()` 且未声明 `streamingBehavior` 会失败；它也验证缺少模型和认证时在 provider 调用前抛错。

`packages/agent/test/agent.test.ts:126-193` 约束两件事：直接抛出的 provider 错误必须产生完整生命周期事件和 error assistant message；异步 `agent_end` 监听器未完成时，`Agent.prompt()` 不能提前 resolve，`state.isStreaming` 也仍为 true。

这些测试覆盖的是控制流和状态语义，不需要真实 API key。它们比“CLI 看起来能回答”更能定位回归：界面能显示文本，不代表会话已经保存，也不代表 prompt 的完成边界没有被缩短。

## QA

### 一条 user message 是谁加入 Agent 内存的？

`AgentSession.prompt()` 只构造消息。`runAgentLoop()` 为它发出 `message_start` 和 `message_end`；`Agent.processEvents()` 在收到 `message_end` 时才把它 push 到 `state.messages`。会话文件也在同一个 `message_end` 经 AgentSession 写入。

### `Agent.prompt()` 为什么不直接返回 assistant message？

一次 prompt 可能包含多个 assistant message、tool result、steering 和 follow-up。`void` 返回值迫使调用者从 Agent 状态或事件读取完整结果，避免把“最后一条 assistant”误当成整次运行。若只需要单次 provider 完成结果，`pi-ai` 的 `completeSimple()` 才是对应抽象。

### provider 失败后，为什么 `await session.prompt()` 可能不抛异常？

模型请求失败属于会话可观察结果。统一流或 Agent 的异常归一化会生成 `stopReason: "error"` 的 assistant message，并让生命周期正常收尾。参数、模型、认证和并发调用等前置错误仍会抛出。调用方需要同时处理 reject 和最终消息的 `stopReason`。

### 当前 CLI 是否已经改用 `AgentHarness`？

没有。固定基线的 `main()` 通过 `createAgentSessionFromServices()` 进入 `createAgentSession()`，构造的是 `AgentSession + Agent`。`packages/agent/src/harness/agent-harness.ts` 是另一条仍在演进的实现，本课主链没有经过它。
