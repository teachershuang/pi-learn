# 第 07 讲：`pi-ai` 的供应商无关协议

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

`pi-ai` 不负责规划任务，也不执行工具。它解决的是更靠近模型的一层问题：应用如何用同一套数据结构描述模型请求，又怎样把 Anthropic、OpenAI、Google 及兼容服务的流式响应变成统一事件。

```text
Model<Api> + Context + StreamOptions
                 │
                 ▼
        provider / API implementation
                 │ 各家 SDK、HTTP、SSE、WebSocket
                 ▼
      AssistantMessageEventStream
                 │
        ┌────────┴────────┐
        ▼                 ▼
  流式事件供 UI       result() 得到最终
  增量显示            AssistantMessage
```

这层协议由三组普通数据和一个流对象组成：`Model<Api>` 描述要调用什么，`Context` 描述要发送什么，`AssistantMessage` 描述最终得到了什么，`AssistantMessageEventStream` 描述生成过程。provider 的认证和模型目录在第 08、09 讲展开；本讲只看它们共同遵守的边界。

## 1. `api` 与 `provider` 是两条不同坐标

### 解决的问题

模型供应商和传输协议并非一一对应。NVIDIA、OpenRouter 或本地兼容服务可以使用 OpenAI-compatible API；同一家供应商也可能同时提供 completions 和 responses 两种协议。如果只保存一个 `provider` 字段，运行时不知道该交给哪个消息转换器和流解析器。

### 源码入口与关键代码

`packages/ai/src/types.ts:16-32` 把已知协议列成字面量，同时允许扩展字符串：

```ts
export type KnownApi =
	| "openai-completions"
	| "mistral-conversations"
	| "openai-responses"
	| "azure-openai-responses"
	| "openai-codex-responses"
	| "anthropic-messages"
	| "bedrock-converse-stream"
	| "google-generative-ai"
	| "google-vertex"
	| "pi-messages";

export type Api = KnownApi | (string & {});

export type KnownImagesApi = "openrouter-images";
export type ImagesApi = KnownImagesApi | (string & {});
```

模型定义在 `packages/ai/src/types.ts:703-729`：

```ts
export interface Model<TApi extends Api> {
	id: string;
	name: string;
	api: TApi;
	provider: ProviderId;
	baseUrl: string;
	reasoning: boolean;
	thinkingLevelMap?: ThinkingLevelMap;
	input: ("text" | "image")[];
	cost: ModelCost;
	contextWindow: number;
	maxTokens: number;
	headers?: Record<string, string>;
	compat?: TApi extends "openai-completions"
		? OpenAICompletionsCompat
		: TApi extends "openai-responses" | "openai-codex-responses"
			? OpenAIResponsesCompat
			: TApi extends "anthropic-messages"
				? AnthropicMessagesCompat
				: never;
}
```

### 运行流程与状态变化

`provider` 确定商业或部署主体，关联模型目录、base URL 和认证；`api` 确定请求格式、响应解析和专属选项。源码中的两个 provider 工厂能直接说明差别：

- `packages/ai/src/providers/openai.ts:6-14` 返回 `Provider<"openai-responses">`；
- `packages/ai/src/providers/nvidia.ts:6-14` 返回 `Provider<"openai-completions">`，API 实现复用 `openAICompletionsApi()`。

因此 `provider: "nvidia"` 不要求实现一套 NVIDIA 专属消息协议。只要端点遵守 OpenAI completions 语义，就能复用同一个 API 适配器。

`Model` 没有 SDK 客户端或执行函数。README 的 Context Serialization 章节明确把模型视为普通可序列化数据；真正的实现由 `api` 和 provider 注册关系在运行时找到。

### 失败路径与设计取舍

`Api` 允许任意字符串，第三方协议不会被封闭联合类型挡住。代价是自定义 API 无法自动获得内置协议的精确 options 类型，需要提供自己的类型封装或接受通用 `Record<string, unknown>`。

`compat` 使用条件类型限制不同协议的兼容开关。`Model<"anthropic-messages">` 不能在类型层误用 OpenAI completions 的兼容字段。这是编译期约束，不替代运行时校验；从 JSON 反序列化的错误对象仍可能绕过 TypeScript。

## 2. `Model<Api>` 把协议选择传递给请求选项类型

### 解决的问题

统一入口既要提供各家都懂的 `temperature`、abort signal 等选项，也要保留 `reasoningSummary`、Anthropic thinking budget 之类的协议专属能力。若入口只接受一个宽泛对象，调用方会失去字段检查和自动补全。

### 源码入口与关键代码

`packages/ai/src/types.ts:193-229` 建立 API 到 options 的类型映射：

```ts
export interface ApiOptionsMap {
	"anthropic-messages": AnthropicOptions;
	"openai-completions": OpenAICompletionsOptions;
	"openai-responses": OpenAIResponsesOptions;
	"openai-codex-responses": OpenAICodexResponsesOptions;
	"azure-openai-responses": AzureOpenAIResponsesOptions;
	"google-generative-ai": GoogleOptions;
	"google-vertex": GoogleVertexOptions;
	"mistral-conversations": MistralOptions;
	"bedrock-converse-stream": BedrockOptions;
	"pi-messages": PiMessagesOptions;
}

export type ApiStreamOptions<TApi extends Api> = TApi extends keyof ApiOptionsMap
	? ApiOptionsMap[TApi]
	: StreamOptions & Record<string, unknown>;

export interface ProviderStreams {
	stream(model: Model<Api>, context: Context, options?: StreamOptions): AssistantMessageEventStream;
	streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

### 运行流程与状态变化

`Models.stream<TApi>()` 接收 `Model<TApi>`，options 随 `TApi` 解析成对应类型。直接持有 `Model<"openai-responses">` 时，TypeScript 能检查 OpenAI Responses 专属字段。通过运行时目录查到的模型是 `Model<Api>`，需要用 `hasApi()` 恢复精确类型：

```ts
export function hasApi<TApi extends Api>(model: Model<Api>, api: TApi): model is Model<TApi> {
	return model.api === api;
}
```

该守卫位于 `packages/ai/src/models.ts:615-627`。它只比较字符串，但比较结果把运行时事实带回类型系统。

底层 `ProviderStreams` 故意收窄成统一分派形状。每个 `src/api/*` 模块都能作为同一种值被 lazy wrapper 和 provider 工厂持有；精确 options 类型保留在公开 API 模块和 `Provider<TApi>.stream()` 边界。

### 失败路径与设计取舍

`streamSimple()` 只接受供应商无关的 `SimpleStreamOptions`，其中 reasoning 是统一等级；`stream()` 暴露协议专属 options。前者方便 Agent 跨模型切换，后者保留供应商能力。统一接口无法保证每个等级在所有模型上语义完全相同，映射和降级仍由模型元数据及 API 实现决定。

类型映射只在编译期工作。JavaScript 调用者、类型断言和反序列化输入仍能传入无效字段；provider 通常忽略不认识的通用字段，专属字段是否报错取决于适配器和上游 API。

## 3. `Context` 是模型请求边界，不是 Agent 运行状态

### 解决的问题

provider 需要 system prompt、对话消息和工具声明，却不需要知道队列、订阅者、正在执行的工具或 session 树。把 Agent 全部状态传给 provider，会让协议适配器和上层编排耦合。

### 源码入口与关键代码

`packages/ai/src/types.ts:442-454` 的输入结构很小：

```ts
export interface Tool<TParameters extends TSchema = TSchema> {
	name: string;
	description: string;
	parameters: TParameters;
}

export interface Context {
	systemPrompt?: string;
	messages: Message[];
	tools?: Tool[];
}
```

### 运行流程与状态变化

`Context.messages` 是某次模型请求要看到的 transcript。`pi-ai` 的 `models.complete()` 不会自动把返回消息追加进去；README Quick Start 在 `await s.result()` 后显式执行 `context.messages.push(finalMessage)`。低层库因此不会偷偷改变调用方的对话状态。

公共 `Message` 联合中没有 system role。system prompt 单独放在 `Context.systemPrompt`，再由各 API 适配器映射成目标协议需要的 system、developer 或独立 instruction 字段。这样不会把某一家供应商的系统消息表示法写进 transcript。

`Tool` 只有名称、描述和 TypeBox 参数 schema。执行函数属于 `packages/agent` 的 `AgentTool`，不进入 provider 请求。这个边界让 `Context` 可以跨线程、进程或服务传递，也避免把不可序列化函数发给模型网关。

TypeBox schema 可以表示为 JSON Schema。工具参数仍要在收到完整 tool call 后调用 `validateToolCall()` 或由 agent-loop 校验；schema 随请求发送不代表本地已经验证模型输出。

### 失败路径与设计取舍

`Context` 是普通可变对象。`pi-ai` 不负责 transcript 的并发控制，也不保证调用后自动保存消息。忘记追加最终 assistant message，下一次请求就缺失这一轮；追加 tool call 却没有匹配的 tool result，又可能被各 API 的转换层过滤或触发上游协议错误。

图片以 base64 存在 content block 中。Context 可以 JSON 序列化，但大图片也会原样进入序列化结果，持久化成本由宿主承担。

## 4. 消息角色固定，内容用带标签的 block 联合扩展

### 解决的问题

不同 provider 对文本、思考、图片和工具调用使用不同字段。上层若直接保存各家原始响应，换模型、导出会话和实现统一 UI 都要重复判断供应商格式。

### 源码入口与关键代码

`packages/ai/src/types.ts:327-355` 把 assistant 内容归一为带 `type` 的 block：

```ts
export interface TextContent {
	type: "text";
	text: string;
	textSignature?: string;
}

export interface ThinkingContent {
	type: "thinking";
	thinking: string;
	thinkingSignature?: string;
	redacted?: boolean;
}

export interface ImageContent {
	type: "image";
	data: string;
	mimeType: string;
}

export interface ToolCall {
	type: "toolCall";
	id: string;
	name: string;
	arguments: Record<string, any>;
	thoughtSignature?: string;
}
```

消息角色定义在 `packages/ai/src/types.ts:382-419`：

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

### 运行流程与状态变化

assistant 始终返回 block 数组，文本、thinking 和 tool call 可以按原始顺序共存。UI 根据 `type` 渲染；provider 转换器根据目标 API 决定怎样编码这些块。`toolCall.id` 与 `toolResult.toolCallId` 形成配对，`isError` 把失败从普通文本中分离出来。

`api`、`provider` 和 `model` 写在 assistant message 上，而不是只依赖当前选择器。会话切换模型后，历史消息仍保留来源信息。跨 provider 发送时，适配器可以据此决定哪些签名可原样回放，哪些 thinking 要降级成文本。

`textSignature`、`thinkingSignature` 和 `thoughtSignature` 是少数进入公共协议的供应商痕迹。核心层不解释其内容，只负责保存，以维持多轮推理或工具调用的上游连续性。

### 失败路径与设计取舍

统一 block 没有抹平全部差异。Google 不支持真正的函数参数增量流；某些 provider 要求 opaque thinking signature；跨 provider 时签名不能随意复用。公共结构保留最小必要元数据，实际转换规则仍在各 API 适配器中。

`details` 只属于本地 tool result 的扩展数据，不保证发给模型。provider 主要消费 `content`、id、名称和错误标志；UI 不能把某个工具的 details 结构当成全局协议。

## 5. usage 是统一账本，reasoning 已包含在 output 中

### 解决的问题

供应商对 input、缓存、reasoning 和价格的返回方式不同。上层需要一致字段显示上下文消耗和费用，又不能把 reasoning token 重复计入总量。

### 源码入口与关键代码

`packages/ai/src/types.ts:357-378` 定义归一化 usage：

```ts
export interface Usage {
	input: number;
	output: number;
	cacheRead: number;
	cacheWrite: number;
	cacheWrite1h?: number;
	/**
	 * Reasoning/thinking tokens, when the provider reports them. This is a subset of
	 * `output`: `output` already includes these tokens.
	 */
	reasoning?: number;
	totalTokens: number;
	cost: {
		input: number;
		output: number;
		cacheRead: number;
		cacheWrite: number;
		total: number;
	};
}
```

### 运行流程与状态变化

API 适配器先把原始 usage 分到四个互斥桶：非缓存 input、output、cache read、cache write。`totalTokens` 通常是四者之和；`reasoning` 只是 output 的明细，不能再加一次。`packages/ai/src/api/openai-completions.ts:1139-1151` 明确执行了这套计算。

费用不直接信任 provider 的总价。`packages/ai/src/models.ts:629-648` 根据 `Model.cost` 的每百万 token 费率和可选阶梯重新计算，并对 Anthropic 1 小时 cache write 使用单独规则。最终 cost 跟随 assistant message 进入 transcript。

测试 provider `packages/ai/test/faux-provider.test.ts:30-68` 检查基本 input/output/totalTokens，并验证 text、thinking、tool call 能共存。faux provider 的缓存路径也按 `input + output + cacheRead + cacheWrite` 生成总量。

### 失败路径与设计取舍

provider 没有提供 reasoning 明细时，该字段是 `undefined`，不能推断为零。错误或中止消息也可能只有部分 usage；异步加载在请求前失败时，`lazyStream` 合成的 usage 全为零。

费用取自本地 `Model.cost`，不是 provider 返回的账单。如果模型目录中的费率与实际计价不同，计算结果也会不同；usage 本身还受供应商返回口径限制。这些字段适合成本显示、预算控制和诊断，不应未经核对直接作为财务结算凭证。

## 6. stop reason 把结束原因从文本中独立出来

### 解决的问题

“输出结束”可能表示正常完成、达到 token 上限、等待工具结果、请求失败或用户中止。只看 assistant content 无法决定后续动作。

### 源码入口与关键代码

`packages/ai/src/types.ts:380-400` 将原因限制为五种：

```ts
export type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";

export interface AssistantMessage {
	role: "assistant";
	content: (TextContent | ThinkingContent | ToolCall)[];
	api: Api;
	provider: ProviderId;
	model: string;
	responseModel?: string;
	responseId?: string;
	diagnostics?: AssistantMessageDiagnostic[];
	usage: Usage;
	stopReason: StopReason;
	errorMessage?: string;
	timestamp: number;
}
```

### 运行流程与状态变化

各 API 适配器把供应商自己的 finish reason 映射到该联合类型。上层据此决策：

| stop reason | 含义 | 常见后续 |
| --- | --- | --- |
| `stop` | 正常结束 | 保存消息，等待下一次用户输入 |
| `length` | 输出达到上限 | 保留部分内容；Agent 对其中工具调用采取保守失败策略 |
| `toolUse` | assistant 请求工具 | 执行并追加 tool result |
| `error` | 请求或解析失败 | 显示错误，由上层决定重试 |
| `aborted` | AbortSignal 中止 | 保存部分内容或放弃本轮 |

错误不是 rejected `result()`。协议要求 provider 发出 terminal `error` 事件，并让 `result()` 返回带 `stopReason` 和 `errorMessage` 的 assistant message。`packages/ai/test/faux-provider.test.ts:413-460` 验证 error 与 aborted 都保留已生成的部分文本，再以 `error` 事件结束。

### 失败路径与设计取舍

`done` 事件的 reason 只能是 `stop | length | toolUse`，`error` 事件的 reason 只能是 `error | aborted`。这种类型拆分让消费者在 switch 分支中拿到正确的最终字段。

“请求不抛错”只覆盖符合 `StreamFunction` 契约的请求、模型和运行失败。调用方在 stream 建立前传入完全错误的对象，或自定义实现违反协议，仍可能触发普通异常或无法结算的流。第 06 讲已经分析了 Agent 对这类违约的兜底边界。

## 7. 事件流同时服务增量 UI 和最终持久化

### 解决的问题

流式 UI 需要增量 delta，持久化和下一轮请求需要一条完整 assistant message。若消费者自行拼接，每个应用都要重复处理 block 索引、错误终点和部分 JSON。

### 源码入口与关键代码

`packages/ai/src/types.ts:456-476` 给出事件联合：

```ts
export type AssistantMessageEvent =
	| { type: "start"; partial: AssistantMessage }
	| { type: "text_start"; contentIndex: number; partial: AssistantMessage }
	| { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
	| { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
	| { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
	| { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
	| { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
	| { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
	| { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
	| { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
	| { type: "done"; reason: Extract<StopReason, "stop" | "length" | "toolUse">; message: AssistantMessage }
	| { type: "error"; reason: Extract<StopReason, "aborted" | "error">; error: AssistantMessage };
```

### 运行流程与状态变化

`partial` 是截至当前事件累计形成的 assistant message，`delta` 是本次新增片段。`contentIndex` 指向 content 数组中的 block。不同 block 的事件可能交错，README `packages/ai/README.md:605-624` 明确要求消费者按 index 关联，不能假定一次 text start 到 end 之间不会出现 tool call 或 thinking 事件。

工具参数在 `toolcall_delta` 阶段只经过尽力解析，字段可能缺失、字符串可能截断。`toolcall_end` 表示参数流组装结束，仍未经过工具 schema 校验。README 在 `packages/ai/README.md:554-568` 也是这样说明的。

测试 `packages/ai/test/faux-provider.test.ts:329-411` 验证 thinking、text、tool call 的 start/delta/end 序列、参数 delta 的拼接结果以及单条消息中的多个工具调用。

### 失败路径与设计取舍

README 的 Complete Event Reference 表在 `packages/ai/README.md:618-622` 把 `toolcall_end.toolCall` 写成 “Complete validated tool call”，这与同一 README 的参数校验章节及实现边界不一致。这里应理解为“流式组装完成”，不是“通过 TypeBox 校验”。直接使用 `pi-ai` 时，执行前仍要调用 `validateToolCall()`；`packages/agent` 的 agent-loop 会代为完成。

provider 实现通常持续修改同一个累计 message 对象，并把它放进 `partial`。例如 `packages/ai/src/api/anthropic-messages.ts:557-584` 在同一个 `output` 上更新 usage 和 content，再把该对象送入事件。需要保存历史帧的消费者应自行复制所需字段；只保留引用，之后可能看到对象已被后续 delta 更新。

## 8. `EventStream` 用一个终端事件同时关闭迭代和兑现结果

### 解决的问题

调用者既要 `for await` 读取事件，又要在任意位置等待最终消息。两个通道若分别维护完成状态，容易出现迭代结束而 final Promise 未完成，或者错误事件丢失。

### 源码入口与关键代码

`packages/ai/src/utils/event-stream.ts:21-66` 是通用队列实现：

```ts
push(event: T): void {
	if (this.done) return;

	if (this.isComplete(event)) {
		this.done = true;
		this.resolveFinalResult(this.extractResult(event));
	}

	const waiter = this.waiting.shift();
	if (waiter) {
		waiter({ value: event, done: false });
	} else {
		this.queue.push(event);
	}
}

async *[Symbol.asyncIterator](): AsyncIterator<T> {
	while (true) {
		if (this.queue.length > 0) {
			yield this.queue.shift()!;
		} else if (this.done) {
			return;
		} else {
			const result = await new Promise<IteratorResult<T>>((resolve) => this.waiting.push(resolve));
			if (result.done) return;
			yield result.value;
		}
	}
}

result(): Promise<R> {
	return this.finalResultPromise;
}
```

`AssistantMessageEventStream` 在 `packages/ai/src/utils/event-stream.ts:69-87` 指定 `done` 和 `error` 都是终点：前者提取 `message`，后者提取 `error`。

### 运行流程与状态变化

事件先交给正在等待的消费者，没有消费者时进入内存队列。终端事件到达时先把 `done` 设为 true、resolve `result()`，再把终端事件本身交给迭代器。队列排空后，下一次迭代返回完成。终点之后的 `push()` 被忽略。

`Models.complete()` 没有第二套非流式请求。`packages/ai/src/models.ts:479-515` 显示 `complete()` 和 `completeSimple()` 都只是调用对应 stream 后等待 `.result()`。所以修复流解析也会同时修复 complete 路径。

### 失败路径与设计取舍

`end(result?)` 可以结束迭代，但只有传入 result 时才兑现 final Promise。如果自定义 producer 没有发送 `done/error`，又调用无参数 `end()`，`for await` 会结束，`result()` 却会一直 pending。内置 provider 的协议要求终端事件，第三方实现也必须遵守。

这个 EventStream 没有背压上限。消费者比 provider 慢时，事件会继续积累在内存队列。大多数 delta 很小，但超长响应和不消费事件的调用方式仍需要关注；只要最终会调用 `result()`，producer 并不会等待 UI 处理每个事件。

## 9. lazy stream 把异步装配失败也变成协议内错误

### 解决的问题

provider 选择、认证和 SDK 动态 import 都是异步操作，但公开 `stream()` 需要立即返回一个可迭代对象。如果先 await 完成装配，调用形态会与普通 provider 不一致；如果让装配异常直接 reject，又破坏“错误通过流返回”的约定。

### 源码入口与关键代码

`packages/ai/src/api/lazy.ts:46-60` 先创建外层流，再在后台装配内层流：

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

成功时，`forwardStream()` 把内层事件逐个推到外层，并转交最终 result。失败时，wrapper 合成 content 为空、usage 为零、`stopReason: "error"` 的 assistant message，发出 error 终点。未知 provider、认证失败和模块加载失败因此都能通过同一种协议到达消费者。

`lazyApi()` 再把动态 import 包成统一的 `ProviderStreams`。根入口只导出类型和轻量核心；真正发起某个 API 请求时才加载对应 SDK。`packages/ai/test/lazy-module-load.test.ts:65-118` 验证导入根模块、全部 provider 或 compat 入口不会加载五个重型 SDK，而 Anthropic 请求只加载 Anthropic SDK。

### 失败路径与设计取舍

lazy wrapper 隐藏了 setup Promise，错误只能从流中观察。调用者若创建 stream 后既不迭代也不 await `result()`，错误消息不会显示，但流仍会自行结算。

动态加载降低初始体积和启动成本，也把部分缺依赖错误推迟到第一次请求。诊断时应区分“provider 已注册”与“对应 SDK 已成功加载”。

## 10. 协议层的模块边界

`packages/ai/src/index.ts:1-45` 明确把根入口保持为无副作用核心：

```text
pi-ai root
  types.ts                 公共数据与类型协议
  utils/event-stream.ts    事件队列和最终结果
  models.ts                provider collection 与分派入口
  api/lazy.ts              异步装配和按需加载

pi-ai/api/<api-id>
  某种线上协议的请求转换与流解析

pi-ai/providers/<id>
  provider 身份、认证、模型目录、base URL 与 API 实现组合

pi-ai/compat
  旧式全局注册和兼容入口
```

这条边界解释了为什么 `pi-ai` 可以独立用于普通聊天程序，而 `packages/agent` 在它之上添加循环、工具执行和事件归约。模型协议不依赖 Agent；Agent 依赖模型协议。

## QA：容易混淆的协议边界

### `Model` 是不是一个已经初始化好的 SDK client？

不是。它是模型 id、协议、provider、能力、上下文窗口和价格等普通数据。SDK client 在具体 API 实现中按请求建立或复用。

### `provider` 与 `api` 为什么不能合成一个字段？

因为多个 provider 可以复用同一 API，同一 provider 也可能支持多个 API。前者决定认证与端点归属，后者决定线上协议和 options 类型。

### `toolcall_end` 是否意味着参数安全可执行？

不意味着。它只表示该工具调用的流式参数已经组装结束。参数仍可能缺字段、类型错误或不满足 schema，必须经过 `validateToolCall()` 或 agent-loop 的校验。

### 为什么 error 也通过 `result()` 返回，而不是 reject？

错误前可能已有文本、thinking、usage 和响应 id。把失败保留为 AssistantMessage，调用者能保存部分结果，并用 stop reason 统一判断结束原因。自定义 producer 的编程错误不在这项保证内。

### `event.partial` 和 `event.delta` 应该用哪个？

delta 适合增量追加，partial 适合用当前累计状态刷新界面。多个 block 会交错，必须结合 `contentIndex`；需要长期保存某一帧时应复制，而不是长期持有 provider 正在修改的对象引用。

### `Usage.reasoning` 要不要加进 totalTokens？

不要。reasoning 是 output 的子集。总量按 input、output、cache read 和 cache write 计算。

## 本讲主线

`pi-ai` 的统一不在于让所有供应商拥有完全相同的能力，而在于固定请求和响应的公共形状：模型用 `api` 指向协议实现，Context 只携带模型所需数据，内容块保留文本、thinking、图片和工具调用，usage 与 stop reason 给出统一结算语义，事件流同时满足实时显示和最终持久化。

供应商差异仍然存在，但被压在 API 实现、模型元数据和少量 opaque signature 中。上层 Agent 因此可以围绕稳定的 `AssistantMessage` 与事件协议工作，不必直接依赖每一家 SDK。
