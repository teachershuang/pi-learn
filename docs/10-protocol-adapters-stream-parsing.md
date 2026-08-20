# 第 10 讲：协议适配与流解析

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

`pi-ai` 对上提供统一的 `Context`、`AssistantMessage` 和流式事件，对下却要接住结构差异很大的供应商协议。适配器不是改几个字段名的序列化器，它同时承担两次翻译：请求前把 Pi 历史投影成供应商允许的输入；响应时把供应商事件还原为带稳定内容索引的 Pi 消息。

```text
Context
  │
  ├── 消息转换：role、content、tool result、reasoning signature
  │
  ├── 请求构造：model、tools、thinking、cache、timeout
  │
  ▼
供应商流
  │
  ├── 协议解析：SSE / SDK AsyncIterable / WebSocket
  ├── 块状态机：start → delta → end
  ├── 终止判定：stop / length / toolUse / error / aborted
  ▼
AssistantMessageEventStream + 持续变更的 AssistantMessage
```

统一层保留了文本、thinking、tool call 的语义，但没有抹平供应商的协议约束。thinking signature、OpenAI output item id、Anthropic tool result 分组和 Google `Part` 的归属，都要在各自适配器内处理。

## 1. 适配器拥有一次请求的可变输出

### 解决的问题

调用方既要立即收到增量事件，也要在流结束后拿到完整的 assistant message。若事件和最终消息分别构造，二者很容易在中止、工具参数半截或 usage 晚到时出现分歧。

### 源码入口与关键代码

三个入口分别是：

- `packages/ai/src/api/anthropic-messages.ts:484-559` 的 `stream()`；
- `packages/ai/src/api/openai-responses.ts:96-172` 的 `stream()`；
- `packages/ai/src/api/google-generative-ai.ts:52-92` 的 `stream()`。

OpenAI Responses 的入口清楚展示了共同骨架，`packages/ai/src/api/openai-responses.ts:105-166`：

```ts
const output: AssistantMessage = {
	role: "assistant",
	content: [],
	api: model.api as Api,
	provider: model.provider,
	model: model.id,
	usage: {
		input: 0,
		output: 0,
		cacheRead: 0,
		cacheWrite: 0,
		totalTokens: 0,
		cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
	},
	stopReason: "stop",
	timestamp: Date.now(),
};

try {
	// 构造 client 和 params
	const { data: openaiStream, response } =
		await client.responses.create(params, requestOptions).withResponse();
	stream.push({ type: "start", partial: output });
	await processResponsesStream(openaiStream, output, stream, model, options);
	stream.push({ type: "done", reason: output.stopReason, message: output });
	stream.end();
} catch (error) {
	output.stopReason = options?.signal?.aborted ? "aborted" : "error";
	output.errorMessage = formatOpenAIResponsesError(error);
	stream.push({ type: "error", reason: output.stopReason, error: output });
	stream.end();
}
```

### 运行流程与状态变化

`output` 在请求开始前建立，后续所有增量都原地修改它。`start`、各种 delta 以及最终 `done/error` 中的 `partial`、`message` 或 `error` 指向同一条正在形成的逻辑消息。

请求参数完成后才发 `start`。认证失败、payload hook 抛错、建立 HTTP 请求失败时，消费者只会看到终结的 `error`；连接已经建立后，消费者先看到 `start`，再看到已经到达的内容和最终错误。

### 失败路径与设计取舍

catch 不会丢弃已经收到的 `content`。失败消息仍能说明流中止前产生了哪些文本或 thinking，但内部定位字段和工具 JSON 缓冲会被删除。这样做保留诊断价值，也避免把解析器的半成品当成会话格式。

另一种做法是把每个 delta 处理成不可变消息快照，语义更纯，但流长时会反复复制整条消息。Pi 选择单个可变 accumulator，事件消费者因而不能把 `partial` 当作历史快照长期保存；需要快照时必须自行复制。

## 2. 请求转换不是无损序列化

### 解决的问题

Pi 的历史可以混合 user、assistant、tool result、图片、thinking 和多个供应商产生的 assistant message。目标 API 对 role、内容块、工具结果位置、ID 格式和 reasoning replay 各有约束。原样发送通常会被 API 拒绝，错误地保留签名还可能把一个模型的私有状态交给另一个模型。

### 共同入口

三个适配器都会先经过消息变换，再进入各自格式：

- Anthropic：`buildParams()` 调用 `transformMessages()` 和本文件的 `convertMessages()`，见 `packages/ai/src/api/anthropic-messages.ts:920-954`；
- OpenAI Responses：`buildParams()` 调用 `convertResponsesMessages()`，见 `packages/ai/src/api/openai-responses.ts:233-248`；
- Google：`buildParams()` 调用 `convertMessages()`，见 `packages/ai/src/api/google-generative-ai.ts:343-362`。

这一步的输入仍是同一个 `Context`，输出已经是供应商协议对象。转换不会改写会话里的原消息。

### Anthropic：role 内嵌内容块，连续工具结果合并

`packages/ai/src/api/anthropic-messages.ts:1141-1200` 把 assistant content 转成 `text`、`thinking`、`redacted_thinking` 和 `tool_use`。thinking signature 存在时才能作为原生 thinking 回放；普通 Anthropic 模型遇到没有签名的 thinking，会降级为文本。这样可以继续携带可见内容，又不会伪造供应商要求的签名。

工具结果的转换位于 `convertMessages()` 的后半段，`packages/ai/src/api/anthropic-messages.ts:1200-1235`。它收集连续的 `toolResult`，合并成一个 user turn，以满足兼容端点对相邻工具结果消息形状的约束。

工具调用 ID 还要经过 `normalizeToolCallId()`，只保留字母、数字、下划线和连字符，并截到 64 个字符，见 `packages/ai/src/api/anthropic-messages.ts:1049-1052`。

### OpenAI Responses：历史被展开成 input items

Responses API 不是简单的 chat message 数组。assistant 的一个 Pi message 可能展开成 reasoning item、output message 和多个 function call。`packages/ai/src/api/openai-responses-shared.ts:170-228` 的核心转换如下：

```ts
for (const block of msg.content) {
	if (block.type === "thinking") {
		if (block.thinkingSignature) {
			const reasoningItem = JSON.parse(block.thinkingSignature);
			output.push(reasoningItem);
		}
	} else if (block.type === "text") {
		output.push({
			type: "message",
			role: "assistant",
			content: [{ type: "output_text", text: sanitizeSurrogates(block.text), annotations: [] }],
			status: "completed",
			id: msgId,
			phase: parsedSignature?.phase,
		});
	} else if (block.type === "toolCall") {
		const [callId, itemIdRaw] = block.id.split("|");
		output.push({
			type: "function_call",
			id: itemIdRaw,
			call_id: callId,
			name: block.name,
			arguments: JSON.stringify(block.arguments),
		});
	}
}
```

Pi 把 Responses 的 `call_id` 和 output item `id` 组合进一个工具调用 ID，形式是 `callId|itemId`。回放时再拆开。跨模型或跨供应商消息会重写或省略不安全的 item id，避免触发 Responses 对 reasoning item 与 function call 配对关系的校验，见 `packages/ai/src/api/openai-responses-shared.ts:105-128`、`170-224`。

工具结果转换为 `function_call_output` item；图片能力允许时，输出可以由 `input_text` 和 `input_image` 组成，见 `packages/ai/src/api/openai-responses-shared.ts:229-268`。

### Google：`Content[]` 与 `Part[]`

Google 使用 `user/model` role。文本、图片、thinking、function call 都是 `Part`。`packages/ai/src/api/google-shared.ts:126-175` 只在 assistant message 来自同一 provider 且同一 model 时保留原生 thought signature：

```ts
const isSameProviderAndModel =
	msg.provider === model.provider && msg.model === model.id;

for (const block of msg.content) {
	if (block.type === "text") {
		const thoughtSignature = resolveThoughtSignature(
			isSameProviderAndModel,
			block.textSignature,
		);
		parts.push({ text: sanitizeSurrogates(block.text), ...(thoughtSignature && { thoughtSignature }) });
	} else if (block.type === "thinking") {
		if (isSameProviderAndModel) {
			parts.push({
				thought: true,
				text: sanitizeSurrogates(block.thinking),
				...(thoughtSignature && { thoughtSignature }),
			});
		} else {
			parts.push({ text: sanitizeSurrogates(block.thinking) });
		}
	}
}
```

signature 还必须通过 base64 形状检查，见 `packages/ai/src/api/google-shared.ts:51-65`。跨模型 thinking 会变成普通文本，不会继续标记为 `thought: true`。

Google 的 `functionResponse` 归入 user turn。连续工具结果会并入同一个带 function response 的 user content；Gemini 3 以前的图片工具结果还可能拆成额外 user message，见 `packages/ai/src/api/google-shared.ts:176-230`。

### 状态变化、失败路径与取舍

消息转换只产生请求局部对象，不改变 `Context.messages`。转换阶段可能因损坏的 OpenAI reasoning signature 执行 `JSON.parse()` 而失败，也可能在 schema、空内容或供应商限制下丢弃不合法块。失败发生在发出 `start` 前，最终仍由 provider wrapper 归一成 error assistant message。

这些转换是有意的有损投影。跨供应商时保留自然语言和工具语义，供应商私有的签名、配对 ID 和缓存信息只在确认兼容时回放。若强求完全无损，只能把各供应商原始对象长期暴露到公共消息类型中，结果会让 agent loop 和会话层依赖具体 API。

## 3. Anthropic：SSE 事件与 content block 一一对齐

### 解决的问题

Anthropic 的流有两层边界：HTTP body 先按 SSE 行组成事件，事件的 JSON 再描述 message 或 content block。代理或兼容端点可能切分任意字节、使用 CRLF、发送未知事件，甚至在 JSON 字符串中留下非法反斜杠或原始制表符。

### SSE 解码与协议校验

`iterateSseMessages()` 在 `packages/ai/src/api/anthropic-messages.ts:384-440` 用 `TextDecoder` 保留跨 chunk 的字符状态，用 buffer 保留未完成行；空行才 flush 一个 SSE event，多条 `data:` 会用换行合并。

`iterateAnthropicEvents()` 在 `packages/ai/src/api/anthropic-messages.ts:443-480` 继续做协议级判断：

```ts
for await (const sse of iterateSseMessages(response.body, signal)) {
	if (sse.event === "error") {
		throw new Error(sse.data);
	}

	if (!ANTHROPIC_MESSAGE_EVENTS.has(sse.event ?? "")) {
		continue;
	}

	try {
		const event = parseJsonWithRepair<RawMessageStreamEvent>(sse.data);
		if (event.type === "message_start") sawMessageStart = true;
		else if (event.type === "message_stop") sawMessageEnd = true;
		yield event;
	} catch (error) {
		throw new Error(
			`Could not parse Anthropic SSE event ${sse.event}: ${message}; data=${sse.data}`,
		);
	}
}

if (sawMessageStart && !sawMessageEnd) {
	throw new Error("Anthropic stream ended before message_stop");
}
```

未知 SSE event 被忽略；明确的 `error` event 立即失败；已经见到 `message_start` 却没有 `message_stop`，即使网络层正常 EOF，也算协议失败。这个终端标记防止半条回复被误报为成功。

### content block 状态机

`packages/ai/src/api/anthropic-messages.ts:576-695` 按供应商的 `event.index` 维护 block：

1. `content_block_start` 建立 text、thinking 或 tool call，并发出对应 `*_start`；
2. `text_delta`、`thinking_delta` 追加字符串；
3. `input_json_delta` 追加到工具块的 `partialJson`，同时尽力解析当前 arguments；
4. `signature_delta` 累积 thinking signature，但不产生可见文本事件；
5. `content_block_stop` 发出 `*_end`，删除内部 index 和工具 JSON 缓冲。

Anthropic 的 event index 不等同于 Pi `content` 下标。代码先用 `findIndex()` 找到对应 block，再把得到的 Pi 下标写入 `contentIndex`。这能容忍适配器忽略某种上游块后，两套索引不再相等。

### JSON 修复、状态变化与失败

`parseJsonWithRepair()` 和 `parseStreamingJson()` 位于 `packages/ai/src/utils/json-parse.ts:27-123`。前者只在标准 `JSON.parse()` 失败后修复字符串内的原始控制字符和非法 escape；后者还会尝试 `partial-json`，完全无法解析时返回空对象。

测试 `packages/ai/test/anthropic-sse-parsing.test.ts:81-166` 构造了包含 `A\H` 和原始制表符的畸形工具参数，确认最终恢复为合法字符串。修复是窄范围容错，不会为任意结构错误猜测字段。

工具调用在 `toolcall_delta` 阶段可能只有部分 arguments。执行边界仍是 `toolcall_end`，后面的 agent loop 还会用工具 schema 校验。流在 block stop 前失败时，已解析的 arguments 可以用于诊断，但 `partialJson` 会在 catch 中移除，不进入持久消息。

## 4. OpenAI Responses：用 output slot 处理交错 item

### 解决的问题

Responses 的事件以 output item 为中心。reasoning、message 和 function call 可以拥有不同 `output_index`；文本 delta 还带 content index。若适配器只维护一个“当前块”，交错事件会把 thinking 写进文本，或把一个工具调用的参数拼到另一个调用上。

### slot 的建立

`processResponsesStream()` 位于 `packages/ai/src/api/openai-responses-shared.ts:331-592`。它用 `Map<number, ResponsesOutputSlot>` 把上游 `output_index` 映射到 Pi 内容块及其稳定下标：

```ts
const outputSlots = new Map<number, ResponsesOutputSlot>();

const createSlot = (outputIndex: number, item: ResponseOutputItem) => {
	if (item.type === "message") {
		const block: TextContent = { type: "text", text: "" };
		output.content.push(block);
		const slot = { type: "text", block, contentIndex: output.content.length - 1 };
		outputSlots.set(outputIndex, slot);
		stream.push({ type: "text_start", contentIndex: slot.contentIndex, partial: output });
		return slot;
	}
	if (item.type === "function_call") {
		const block = {
			type: "toolCall",
			id: `${item.call_id}|${item.id}`,
			name: item.name,
			arguments: {},
			partialJson: item.arguments || "",
		};
		// 写入 output、建立 slot、发出 toolcall_start
	}
};
```

reasoning item 也建立独立 slot。`response.output_item.added` 通常创建它；若供应商只发 `output_item.done`，`getOrCreateSlot()` 仍能补建，避免丢失终态 item。

### delta 与 done 的不同职责

`packages/ai/src/api/openai-responses-shared.ts:449-572` 把事件分成两类：

- reasoning summary/text、output text、refusal 和 function arguments delta 负责增量更新；
- `response.output_item.done` 用供应商给出的完整 item 覆盖最终文本或 arguments，保存 signature，然后发出块结束事件并删除 slot。

`response.function_call_arguments.done` 有时会带上 delta 阶段缺失的尾部。代码只有在完整字符串以前缀形式包含已收内容时，才把差额补发成 `toolcall_delta`；不满足前缀关系时只更新最终 arguments，避免向消费者伪造一段错误增量。

reasoning 的 `thinkingSignature` 保存完整 reasoning item 的 JSON，而不是只存 encrypted content。请求回放时可以原样恢复 item。Azure 有时只在 terminal response 的 `output` 中给 encrypted content，`backfillReasoningSignatures()` 会在结束时补回，见 `packages/ai/src/api/openai-responses-shared.ts:392-408`。

### 终端事件决定请求是否完整

`response.completed` 和 `response.incomplete` 会更新 response id、usage、费用和 stop reason。存在工具调用且原 stop reason 为 `stop` 时，Pi 改为 `toolUse`。`response.failed` 提取 error 或 incomplete details 后抛错；普通 `error` event 也抛错。

循环结束后还会检查是否见过 terminal response event：

```ts
if (!sawTerminalResponseEvent) {
	throw new Error("OpenAI Responses stream ended before a terminal response event");
}
```

测试 `packages/ai/test/openai-responses-terminal-event.test.ts:156-231` 分别固定了提前 EOF、completed、incomplete 和 failed 的结果。提前 EOF 即使已经产生 reasoning delta，也必须得到 `stopReason: "error"`；incomplete 则是完整收到供应商终态后的 `length`，二者不能混为一类。

`packages/ai/test/openai-responses-partial-json-cleanup.test.ts:67-105` 还确认 `partialJson` 在 `output_item.done` 后不存在于持久工具块，`toolcall_end.toolCall` 与最终消息中的块是同一个已清理对象。

## 5. Google：根据连续 Part 重建块边界

### 解决的问题

Google SDK 返回 `AsyncIterable<GenerateContentResponse>`，每个 chunk 中是 candidate 和 `Part[]`。文本 Part 没有 Anthropic 那样独立的 block start/stop 事件；function call 通常已经带完整 arguments。适配器需要自己推导 Pi 块何时开始、结束和切换。

### 文本与 thinking 的切换

`packages/ai/src/api/google-generative-ai.ts:93-159` 用 `currentBlock` 保存当前 text 或 thinking：

```ts
for await (const chunk of googleStream) {
	const candidate = chunk.candidates?.[0];
	for (const part of candidate?.content?.parts ?? []) {
		if (part.text !== undefined) {
			const isThinking = isThinkingPart(part);
			if (
				!currentBlock ||
				(isThinking && currentBlock.type !== "thinking") ||
				(!isThinking && currentBlock.type !== "text")
			) {
				// 结束旧块，创建新块，发送 *_start
			}

			if (currentBlock.type === "thinking") {
				currentBlock.thinking += part.text;
				currentBlock.thinkingSignature = retainThoughtSignature(
					currentBlock.thinkingSignature,
					part.thoughtSignature,
				);
			} else {
				currentBlock.text += part.text;
			}
		}
	}
}
```

`isThinkingPart()` 只认 `part.thought === true`。`thoughtSignature` 可能附在文本或 function call 上，只表示后续回放需要保留的私有状态，不能据此把 part 分类为 thinking。测试 `packages/ai/test/google-thinking-signature.test.ts:4-37` 同时约束了分类和“后续 delta 缺少 signature 时保留旧值”的行为。

### function call 是原子块

遇到 `part.functionCall` 时，适配器先结束 `currentBlock`，再生成工具调用。缺少 ID 或 ID 重复时会生成进程内唯一 ID；arguments 直接来自 `functionCall.args`。随后连续发送 `toolcall_start`、一个包含完整 JSON 的 `toolcall_delta` 和 `toolcall_end`，见 `packages/ai/src/api/google-generative-ai.ts:161-205`。

这三个事件虽然紧邻，仍维持统一事件契约。消费者不必为 Google 增加“无 delta 的工具调用”分支；同时也不能假设每家 provider 都会把 arguments 拆成多个增量。

### usage、终止与失败

candidate 的 `finishReason` 经 `mapStopReason()` 归一。`STOP` 是 `stop`，`MAX_TOKENS` 是 `length`，安全、背诵、畸形 function call 和 unexpected tool call 等原因都是 `error`，见 `packages/ai/src/api/google-shared.ts:306-335`。只要输出含工具调用，最终 stop reason 改为 `toolUse`。

usage metadata 可能晚到，适配器每个 chunk 都以最新完整统计覆盖 `output.usage`。Google 的 candidates token 与 thoughts token 相加成为 output，thoughts 同时单列为 reasoning，见 `packages/ai/src/api/google-generative-ai.ts:217-235`。

流自然结束时，仍在打开的 text/thinking 块会补发结束事件。abort 或 error 会保留已经形成的内容，写入归一化后的 provider error，并以 `error` 事件结束。

## 6. SSE、WebSocket 和统一事件不是同一层

### 解决的问题

传输层决定字节怎样到达，provider 事件决定字节表示什么，Pi 事件决定上层怎样消费。把三层混为一谈，会误以为换成 WebSocket 就必须改 agent loop，或误以为使用 SDK 就不再处理协议完整性。

### 当前三条代表路径

| API 实现 | 本课路径 | 传输/迭代入口 | Pi 块状态来源 |
| --- | --- | --- | --- |
| `anthropic-messages` | Anthropic 原始消息流 | 自行读取 HTTP body 并解 SSE | `content_block_start/delta/stop` |
| `openai-responses` | OpenAI SDK Responses stream | SDK 返回的 `AsyncIterable<ResponseStreamEvent>` | `output_index` 与 output item 生命周期 |
| `google-generative-ai` | Google GenAI SDK stream | SDK 返回的 response chunks | 相邻 `Part` 的类型切换 |

普通 `openai-responses` 在本基线中没有 WebSocket 选择项。WebSocket 位于独立的 `openai-codex-responses` API 实现。`packages/ai/src/api/openai-codex-responses.ts:260-339` 的 `transport` 可取 SSE、WebSocket 或 auto；auto 的 WebSocket 在尚未发出消息流事件前失败，可以退回 SSE，已经开始发事件后则直接失败，防止回退后重复文本或工具参数。

Codex WebSocket 和 SSE 最后都进入共享的 `processResponsesStream()`，见 `packages/ai/src/api/openai-codex-responses.ts:590-605` 及 WebSocket 处理路径。传输可以替换，Pi 事件合同不变；回退安全却取决于“是否已经发出事件”这个边界。

### 状态与取舍

Codex WebSocket 连接按 session 缓存，并记录 busy、创建时间和 continuation；传输失败会把该 session 标为后续走 SSE，相关状态位于 `packages/ai/src/api/openai-codex-responses.ts:756-803`。这是 Codex API 的传输优化，不能推广成所有 OpenAI Responses provider 的能力。

共享一套传输抽象可以减少代码，但无法消除不同 API 的握手、header、重试和 continuation 语义。Pi 只把解析后的 Responses event 处理提取为共享函数，传输状态仍留在 Codex 适配器中。

## 7. 重试分成 SDK 请求重试、Codex 请求重试和上层整轮重试

### 解决的问题

重试的位置决定是否可能重复输出。尚未收到响应体时重新发 HTTP 请求，与已经向用户发出文本后重新启动整个 assistant turn，风险完全不同。

### 三种机制

Anthropic 和普通 OpenAI Responses 把 `options.maxRetries ?? 0` 交给各自 SDK，请求代码见 `packages/ai/src/api/anthropic-messages.ts:550-555`、`packages/ai/src/api/openai-responses.ts:134-140`。Pi 默认把 SDK retry 设为零；调用方明确传入后，具体重试阶段由 SDK 实现负责，本仓库适配器没有自行复制已发出的 Pi delta。

Google Generative AI 的这条 `stream()` 没有读取 `maxRetries`，也没有适配器内重试循环。不能因为另外两条 API 接受同一个基础 options，就认为 Google 路径行为相同。

独立的 Codex Responses SSE 路径实现了显式 retry。`packages/ai/src/api/openai-codex-responses.ts:350-428` 只在获得成功 response 之前循环：

- HTTP 状态和 error body 先由 `isRetryableError()` 分类；
- `Retry-After` 或指数退避决定等待时间；
- 网络错误也可重试，usage limit 不重试；
- abort 会打断等待；
- 成功得到 response 后退出循环，再开始解析 SSE。

因此这段重试不会重复已经发送的 Pi 内容。WebSocket auto 回退也只允许发生在 `websocketStarted === false` 时，理由相同。

`packages/ai/src/utils/retry.ts:87-100` 的 `isRetryableAssistantError()` 是另一层：它只根据最终 assistant error message 分类瞬时错误和额度错误，不执行 sleep，也不拥有重试次数。注释明确要求调用方先处理 context overflow，再决定预算、退避和是否重启整轮。真正的 Coding Agent 自动重试属于后续 `AgentSession` 课程。

### 失败路径与设计取舍

上层整轮重试可能面对已经产生部分内容的 error message。是否从界面移除旧内容、是否写入 session、是否重新发送完整上下文，不属于 provider adapter 的决策。适配器只保证失败是可观察的完整 assistant message，并尽量保留部分内容。

把所有重试统一塞进 provider 层会让 provider 在不知道会话写入和 UI 状态的情况下重启整轮，容易制造重复工具调用。Pi 将“请求建立前的安全重试”和“已有失败消息后的策略重试”分开。

## 8. 错误归一化保留阶段信息

### 供应商错误体

Anthropic 原始 SSE 的 `event: error` 把 data 作为错误；SSE JSON 无法解析时，错误包含 event、data 和原始行；缺少 `message_stop` 使用单独错误。这里保留的是协议阶段。

OpenAI Responses 的普通 `error` event包含 code/message；`response.failed` 优先使用 response error，退而使用 incomplete details。没有 terminal event 则报告提前结束。`openai-responses.ts` 外层再用 `formatOpenAIResponsesError()` 处理 SDK 抛出的错误对象，见 `packages/ai/src/api/openai-responses.ts:81-84`、`158-167`。

Google 外层通过 `normalizeProviderError()` 和 `formatProviderError()` 处理 SDK 异常，见 `packages/ai/src/api/google-generative-ai.ts:267-277`。finish reason 映射为 `error` 时，当前代码只在随后抛出 `"An unknown error occurred"`，不会把具体 safety reason 写入 `errorMessage`。这是当前实现的信息损失，不应在上层文档中宣称能看到完整 Google 拒绝原因。

### 中止与普通错误

三个 wrapper 都以 `options.signal?.aborted` 作为最后判定：signal 已中止时是 `stopReason: "aborted"`，否则是 `error`。这避免底层 SDK 各自不同的 AbortError 文案泄漏成不一致 stop reason。

失败事件携带的是 `error: output`，不是裸异常。agent loop 因而仍能用统一 message 类型记录 provider、model、timestamp、partial content 和 usage。裸异常信息被折叠进 `errorMessage`。

### 设计取舍

Pi 对 stop reason 做的是面向 agent loop 的归类，不是供应商原因的完整枚举。`toolUse`、`length` 和 `aborted` 足以决定下一步控制流；诊断细节依赖 `errorMessage`。Google finish reason 的细节目前没有完整保存，若要改进，应增加结构化诊断或明确错误文本，而不是不断扩张公共 `StopReason`。

## 9. 测试固定的协议边界

本课相关测试不只检查最终文本，还刻意构造流的不完整和兼容性异常：

- `packages/ai/test/anthropic-sse-parsing.test.ts:81-166`：SSE JSON 与工具 JSON 含非法 escape/控制字符时可修复；后续用例还覆盖 refusal details 和 `message_stop` 后的未知事件。
- `packages/ai/test/openai-responses-terminal-event.test.ts:156-231`：提前 EOF 是 error，completed 是 stop，incomplete 是 length，failed 保留 provider error。
- `packages/ai/test/openai-responses-partial-json-cleanup.test.ts:67-105`：最终工具块和 `toolcall_end` 都不能带内部 `partialJson`。
- `packages/ai/test/google-thinking-signature.test.ts:4-37`：`thought` 决定 thinking 分类，signature 只负责回放，并在后续 delta 缺失时保留。
- `packages/ai/test/retry.test.ts:13-58`：过载、网络断开和明确 retry 指引可重试；quota/billing 型 429 不可重试。

修改适配器时，至少要验证请求投影、事件顺序、提前 EOF、工具参数终结、signature 回放和错误 stop reason。只检查 `complete()` 返回了一段文本，无法发现内容索引错位、半截流误报成功或 scratch state 被持久化。

## 10. 常见问题

### 为什么 provider adapter 不能共用一个通用流解析器？

共同部分已经下沉到 `AssistantMessageEventStream`、公共消息类型、JSON 解析工具和 Responses event processor。Anthropic 的 block index、Responses 的 output item slot、Google 的相邻 Part 切换属于三种不同状态机。继续强行合并会把大量 provider 分支塞进一个解析器，反而模糊终端事件和块边界。

### `toolcall_delta` 中的 arguments 能直接执行吗？

不能。Anthropic 和 OpenAI 在 delta 阶段调用 `parseStreamingJson()`，结果可能是部分对象，也可能暂时是空对象。适配器在 `toolcall_end` 前不会把调用交给 agent loop；agent loop 随后还会进行工具 schema 校验。

### 为什么每个 delta 都有 `contentIndex`？

一条 assistant message 可以交错出现 thinking、文本和多个工具调用。消费者不能只把 delta 追加到“最后一个块”。`contentIndex` 把事件绑定到 `output.content` 中稳定的位置，OpenAI 的 slot map 正是为此存在。

### thinking signature 是可读推理文本的哈希吗？

不是。源码把它当作供应商私有、需要回放的 opaque state。Anthropic 使用 signature 或 redacted payload；OpenAI 保存整个 reasoning item JSON；Google 的 thought signature 可能附在任何 Part 上。它们格式不同，也不能跨模型随意复用。

### 普通 OpenAI Responses 会自动使用 WebSocket 吗？

不会。本基线中 transport 选择属于 `openai-codex-responses`，普通 `openai-responses` 直接使用 OpenAI SDK 的 stream。二者共享 Responses event 处理函数，不共享传输实现。

### 流中已经出现文本后断线，provider adapter 会自动重发吗？

三条代表路径的 wrapper 都不会自行重启已经输出的 Pi 流。它们返回带 partial content 的 error message。Codex WebSocket 只有在尚未开始发事件时才允许退回 SSE。是否整轮重试由更上层决定。

### 为什么 incomplete 是 `length`，提前 EOF 却是 `error`？

incomplete 是供应商明确发出的终端状态，说明响应完整结束但达到限制；提前 EOF 没有终端证明，无法确认内容、usage 和工具参数是否完整。两者对能否安全继续的含义不同。

## 结语

Pi 的协议适配分成请求投影、传输读取、供应商事件状态机和公共事件输出四层。统一类型让 agent loop 不必理解 Anthropic、OpenAI 或 Google 的 wire format；适配器仍保留每家协议的终端条件、签名和工具调用约束。

三条路径最值得记住的差别是状态定位方式：Anthropic 用 content block index，OpenAI Responses 用 output slot，Google 用当前 Part 类型。它们最后都形成同一个 `AssistantMessage`，但到达这个结果的失败条件并不相同。第 11 讲将在这个边界之上继续追踪模型选择、reasoning 参数映射、上下文预算、usage 和费用统计。
