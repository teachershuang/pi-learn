# 第 20 讲：上下文溢出与恢复

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 把上下文溢出和普通瞬时错误分开处理。溢出意味着同一份上下文再次发送仍会失败，恢复动作必须先压缩；429、5xx、连接中断等错误通常不需要改写上下文，可以等待后重放 assistant turn。

两条恢复路径都发生在一次低层 Agent run 已经结束之后。失败 assistant message 会先进入内存状态和 JSONL，`AgentSession` 再判断是压缩还是退避，并从当前内存上下文移除失败 message。这里存在两份不同用途的状态：JSONL 保留失败证据，运行态为下一次请求准备可发送上下文。

```text
provider / transport
  → 最终 AssistantMessage(stopReason, errorMessage, usage)
  → message_end：扩展可归一化；随后写入 JSONL
  → turn_end → agent_end
  → AgentSession._handlePostAgentRun()
       ├─ 瞬时错误：移除运行态 error → backoff → agent.continue()
       ├─ 上下文溢出：移除运行态 error → compact → agent.continue()
       └─ 不可恢复：保留最终结果，结束
```

## 1. provider 错误先变成 AssistantMessage

### 解决的问题

上层恢复器不能依赖每家 SDK 的异常类型。provider 可能在建连时抛错，也可能在 SSE/WebSocket 中途失败；有些服务不报错，只在 usage 或 finish reason 上暴露上下文已满。

`packages/agent/src/agent-loop.ts:281-372` 负责消费 provider stream。正常结束和流错误都会取 `response.result()`，发出同一组 `message_end` 事件。若 stream 函数直接抛异常，`Agent.runWithLifecycle()` 会在 `packages/agent/src/agent.ts:469-509` 构造失败 assistant message：

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

恢复分类的输入因此总是最终 `AssistantMessage`，而不是原始 HTTP status、SDK error 或 stream event。provider 适配器仍有一项责任：把有用的错误文本保留到 `errorMessage`。文本丢失后，上层只能看到泛化的 `stopReason: "error"`，无法区分溢出、限流和认证失败。

## 2. 溢出识别有三条证据路径

入口是 `packages/ai/src/utils/overflow.ts:129-159`：

```ts
export function isContextOverflow(message: AssistantMessage, contextWindow?: number): boolean {
	if (message.stopReason === "error" && message.errorMessage) {
		const isNonOverflow = NON_OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!));
		if (!isNonOverflow && OVERFLOW_PATTERNS.some((p) => p.test(message.errorMessage!))) {
			return true;
		}
	}

	if (contextWindow && message.stopReason === "stop") {
		const inputTokens = message.usage.input + message.usage.cacheRead;
		if (inputTokens > contextWindow) return true;
	}

	if (contextWindow && message.stopReason === "length" && message.usage.output === 0) {
		const inputTokens = message.usage.input + message.usage.cacheRead;
		if (inputTokens >= contextWindow * 0.99) return true;
	}
	return false;
}
```

三条证据对应不同 provider 行为：

| 信号 | 典型行为 | 判断 |
|---|---|---|
| `stopReason: "error"` + 已知文本 | Anthropic、OpenAI、Google、OpenRouter 等明确拒绝请求 | 匹配 provider 或通用 overflow 正则 |
| `stopReason: "stop"` + input usage 超过窗口 | 服务接受了超长输入并报告真实用量 | 视为 silent overflow，但已得到完整 assistant answer |
| `stopReason: "length"` + output 为 0 + 输入占满 99% 窗口 | 服务截断输入，已无输出空间 | 视为 length-stop overflow |

错误文本规则在 `packages/ai/src/utils/overflow.ts:36-80`。通用的 `too many tokens` 容易误伤 Bedrock 限流，因此代码先排除 throttling、service unavailable、rate limit 和 too many requests。`packages/ai/test/overflow.test.ts:33-138` 同时验证多家溢出文案和这些反例。

### 识别不是 token 预计算

这套函数判断 provider 已经给出的结果，不会在请求前精确计算 prompt token。第 19 讲的阈值压缩负责提前规避；`isContextOverflow()` 是失败后的补救。Ollama 一类服务若静默截断输入，又不报告超窗 usage，Pi 没有足够证据恢复。

自定义 provider 可以在 `message_end` 扩展中规范化 `errorMessage`。扩展先于公开监听器和 session 写入执行，相关顺序见 `packages/coding-agent/src/core/agent-session.ts:700-759`。官方 `packages/coding-agent/docs/custom-provider.md:533-580` 建议把自定义溢出文案改写为带 `context_length_exceeded` 的错误；必须限制 provider 范围，避免把其他模型的错误误分类。

## 3. 写入顺序决定失败后还剩什么

`Agent.processEvents()` 先把 `message_end` 放进 `agent.state.messages`，再按注册顺序等待监听器，见 `packages/agent/src/agent.ts:527-568`。`AgentSession` 的监听器执行顺序是：

```text
message_end
  → message_end 扩展，可原位替换最终 message
  → AgentSession 公共事件订阅者
  → SessionManager.appendMessage()
  → 记录 _lastAssistantMessage
turn_end
agent_end
  → 本次低层 run 完成
  → _handlePostAgentRun() 才开始恢复判断
```

持久化代码位于 `packages/coding-agent/src/core/agent-session.ts:574-645`。所以压缩或普通重试开始前，失败 assistant 已经成为 append-only tree 的 leaf。后续代码中的 `slice(0, -1)` 只替换 `agent.state.messages`，不删除 JSONL entry。

这个顺序有两个直接结果：

- 进程崩溃在恢复判断之前，session 仍能说明上次请求怎样失败；
- “重试时不把失败 message 发回 provider”是运行态规则，不是历史清理规则。

## 4. 一次 run 结束后怎样选择恢复器

`packages/coding-agent/src/core/agent-session.ts:1049-1090` 把一个用户 prompt 包在外层循环中：每次 `agent.prompt()` 或 `agent.continue()` 返回后，都运行 `_handlePostAgentRun()`。后者先检查普通 retry，再检查 compaction：

```ts
private async _handlePostAgentRun(): Promise<boolean> {
	const msg = this._lastAssistantMessage;
	this._lastAssistantMessage = undefined;
	if (!msg) return false;

	if (this._isRetryableError(msg) && (await this._prepareRetry(msg))) {
		return true;
	}

	if (msg.stopReason === "error" && this._retryAttempt > 0) {
		this._emit({
			type: "auto_retry_end",
			success: false,
			attempt: this._retryAttempt,
			finalError: msg.errorMessage,
		});
		this._retryAttempt = 0;
	}

	if (await this._checkCompaction(msg)) return true;
	return this.agent.hasQueuedMessages();
}
```

“普通 retry 在前”不意味着 overflow 会先被原样重发。`_isRetryableError()` 在 `packages/coding-agent/src/core/agent-session.ts:2606-2614` 先调用 `isContextOverflow()`，命中后直接返回 `false`，把它留给 compaction。否则，一条同时包含 413 和 `server error` 的消息可能在未缩短上下文的情况下反复发送。

返回 `true` 只表示外层应该再调用一次 `agent.continue()`。真正的续跑入口在 `packages/agent/src/agent.ts:348-370`：最后一条运行态消息必须能从 user 或 tool result 继续；失败 assistant 必须在此之前移除。

## 5. 明确报错的 overflow：压缩后重放一次

核心分支位于 `packages/coding-agent/src/core/agent-session.ts:1935-1993`：

```ts
if (sameModel && isContextOverflow(assistantMessage, contextWindow)) {
	const willRetry = assistantMessage.stopReason !== "stop";

	if (!willRetry) {
		return await this._runAutoCompaction("overflow", false);
	}

	if (this._overflowRecoveryAttempted) {
		this._emit({
			type: "compaction_end",
			reason: "overflow",
			result: undefined,
			aborted: false,
			willRetry: false,
			errorMessage: "Context overflow recovery failed after one compact-and-retry attempt. " +
				"Try reducing context or switching to a larger-context model.",
		});
		return false;
	}

	this._overflowRecoveryAttempted = true;
	const messages = this.agent.state.messages;
	if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
		this.agent.state.messages = messages.slice(0, -1);
	}
	return await this._runAutoCompaction("overflow", willRetry);
}
```

恢复过程不是“捕获异常后再调一次 provider”，而是：

1. 验证失败 message 的 `provider/model` 与当前模型相同。用户已经换到别的模型时，不用旧窗口的错误压缩新上下文。
2. 设置 `_overflowRecoveryAttempted`，阻止明确 error 路径无限 compact-and-retry。
3. 从运行态移除失败 assistant；JSONL 不变。
4. 准备并生成 compaction summary。
5. 追加 compaction entry，从 SessionManager 重建上下文。
6. 再次移除重建后位于末尾的 error assistant。
7. 返回 `true`，外层调用 `agent.continue()`，重新生成这个 assistant turn。

第 4 至 6 步位于 `packages/coding-agent/src/core/agent-session.ts:2029-2175`。第 6 步不可省：压缩以 session branch 为材料，而失败 entry 已经持久化；`buildSessionContext()` 会把近期保留区里的失败 message 再投影回来。

### 重放上限怎样复位

`_overflowRecoveryAttempted` 在新的 user message 开始时复位，也会在任意非 error assistant 完成时复位，见 `packages/coding-agent/src/core/agent-session.ts:574-635`。对明确的 `stopReason: "error"` overflow，第一次失败不会复位标记；压缩重试仍溢出时，第二次检查停止恢复。`packages/coding-agent/test/agent-session-auto-compaction-queue.test.ts:136-189` 验证 `_runAutoCompaction()` 只调用一次，并检查最终错误事件。

## 6. silent overflow 不重放

如果 provider 返回 `stopReason: "stop"`，但 usage 已超过当前模型窗口，Pi 仍压缩，不过 `willRetry` 为 `false`。完整 answer 已经写入 session，`Agent.continue()` 也不允许从 assistant message 无条件续跑。此时压缩为下一次用户输入腾出空间，不重做已经成功的回答。

这是结果语义优先于 overflow 标签：同样被 `isContextOverflow()` 命中，`stop` 代表“本轮已有可用回答”，`error` 代表“本轮没有回答”。二者不能共用自动重放策略。

## 7. length-stop overflow 在固定 commit 中没有闭合

`isContextOverflow()` 把“`stopReason: "length"`、output 为 0、输入占满窗口”认作 overflow。`_checkCompaction()` 又把所有非 `stop` 结果设为 `willRetry: true`，所以 length 分支准备压缩后重放。

但 `_runAutoCompaction()` 重建上下文后只删除 `stopReason: "error"` 的末尾 assistant，见 `packages/coding-agent/src/core/agent-session.ts:2162-2170`。length assistant 会留在运行态；随后 `Agent.continue()` 在 `packages/agent/src/agent.ts:353-370` 拒绝从 assistant message 继续。另一个缺口是 message_end 会把所有非 error 结果视为成功，从而提前复位 `_overflowRecoveryAttempted`。

这不是推测出的 provider 行为，而是固定 commit 上可以顺着条件分支得到的控制流结论。现有 `packages/ai/test/overflow.test.ts:125-138` 只验证 length-stop 分类，没有覆盖 `AgentSession` 的 compact-and-continue。按本课的失败分类，它属于上游源码缺陷及测试缺口；公开 README 中“overflow 会恢复并重试”的描述对明确 error 路径成立，对该 length 路径不成立。

## 8. 普通瞬时错误走指数退避

### 哪些错误可重试

`packages/ai/src/utils/retry.ts:7-101` 只分类，不决定次数和等待。可重试文本包括 overloaded、429、5xx、service unavailable、连接断开、fetch/socket/WebSocket 错误、timeout、提前结束的 stream，以及 provider 明确给出的 retry 指示。

quota、budget、billing、月度或免费额度用尽先被排除。它们可能也带 429，但等待几秒不会恢复。测试 `packages/ai/test/retry.test.ts:14-55` 检查显式 retry 文案、socket 断开、普通服务错误和 `quota exceeded` 反例。

### 谁决定次数和等待

AgentSession 默认最多重试 3 次，base delay 为 2000ms；延迟按 `baseDelayMs × 2^(attempt-1)` 计算，即 2s、4s、8s。配置入口见 `packages/coding-agent/src/core/settings-manager.ts:813-818`。

```ts
this._retryAttempt++;
if (this._retryAttempt > settings.maxRetries) {
	this._retryAttempt--;
	return false;
}

const delayMs = settings.baseDelayMs * 2 ** (this._retryAttempt - 1);
this._emit({
	type: "auto_retry_start",
	attempt: this._retryAttempt,
	maxAttempts: settings.maxRetries,
	delayMs,
	errorMessage: message.errorMessage || "Unknown error",
});

const messages = this.agent.state.messages;
if (messages.length > 0 && messages[messages.length - 1].role === "assistant") {
	this.agent.state.messages = messages.slice(0, -1);
}
```

完整实现位于 `packages/coding-agent/src/core/agent-session.ts:2620-2669`。这里没有 jitter，也不读取该 assistant error 中的 Retry-After；它是 AgentSession 自己的可见退避策略。等待结束后返回 `true`，外层用原 user/tool-result 上下文重新生成 assistant。

成功判断发生在后续 assistant 的 `message_end`。只要 `stopReason !== "error"`，代码发出 `auto_retry_end(success: true)` 并清零 attempt。这意味着 `aborted` 或 `length` 也会结束普通 retry 计数；它们是否代表业务成功，要由各自后续逻辑判断。

### provider 内部 retry 是另一层

`retry.provider.maxRetries` 会在 `packages/coding-agent/src/core/sdk.ts:298-313` 直接传给 provider SDK/adapter。该层发生在一个 assistant stream 尚未最终结束时，通常不会产生多条 `message_end` 或多条 session entry。AgentSession 的 `retry.maxRetries` 则在最终 error assistant 已经落盘后重跑整个 assistant turn，并发出 `auto_retry_*` 事件。

官方 `packages/coding-agent/docs/settings.md:134-162` 把 provider retry 默认写为 0，建议只有明确需要时才开启；长时间额度等待若被 SDK 截留，上层无法及时显示和取消。`maxRetryDelayMs` 默认 60 秒，用于限制 provider 要求的超长等待，并不限制 AgentSession 的 2s/4s/8s 退避。

## 9. abort 在三个阶段留下不同状态

| 用户中止位置 | 实际控制器 | 已经持久化什么 | 是否继续请求 |
|---|---|---|---|
| provider 正在流式响应 | Agent run AbortController | user 已落盘；最终 aborted assistant 也在 `message_end` 落盘 | 本次 run 结束，不做普通 retry |
| AgentSession 正在 backoff | `_retryAbortController` | 触发 retry 的 error assistant 已落盘，且已从运行态移除 | sleep 被拒绝，不调用 `agent.continue()` |
| 自动 compaction 正在生成摘要 | `_autoCompactionAbortController` | overflow assistant 已落盘；尚无 compaction entry | 不提交摘要，不重放 turn |

`AgentSession.abort()` 会先 `abortRetry()`，再 `agent.abort()`，并等待整个外层 run settle，见 `packages/coding-agent/src/core/agent-session.ts:1530-1541`。TUI 在 retry 倒计时时把 Escape 绑定到 `abortRetry()`；压缩期间则绑定 `abortCompaction()`，入口在 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3023-3108`。

低层 Agent 对 aborted assistant 仍发出 `message_end`、`turn_end` 和 `agent_end`。`_checkCompaction()` 在 run 后跳过 aborted message；发送下一条 prompt 前会用 `skipAbortedCheck: false` 再检查一次阈值，见 `packages/coding-agent/src/core/agent-session.ts:1185-1190`。所以中止不会自动重放，但下一次输入前仍可能先压缩旧上下文。

`packages/coding-agent/test/suite/agent-session-retry-events.test.ts:146-167` 验证取消 backoff 后没有第二次 provider 调用；同文件 `337-360` 验证流式中止仍持久化 aborted assistant。

## 10. 重试期间的输入不会挤进错误位置

自动 compaction 期间，interactive mode 保持编辑器可用，但把普通输入放进 `compactionQueuedMessages`。`compaction_end` 到达后：

- 若 `willRetry` 为真，steering/follow-up 被排入即将重放的 turn；
- 若不重放，第一条普通消息启动新 prompt，其余消息进入相应队列；
- 发送失败时恢复整个临时队列。

提交拦截位于 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2771-2791`，队列冲刷位于 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3992-4070`。它解决的是 TUI 交互顺序，不属于 AgentSession 的持久化队列；进程在 compaction 中途退出时，这批消息不会写入 JSONL。

普通 backoff 仍处在 `_runAgentPrompt()` 的外层运行中，`isStreaming` 为真。新输入沿既有 steering/follow-up 队列处理，直到 `agent.continue()` 启动下一次低层 run。

## 11. JSONL 与运行态只暂时分离

一次普通 retry 成功后，状态大致是：

```text
JSONL branch：user → failed assistant → recovered assistant
运行态上下文：user → recovered assistant
```

overflow recovery 还会在两者之间追加 compaction entry。失败 assistant 留在 append-only tree，下一次请求的运行态副本不含它。

这里有一个持久化边界：`sessionEntryToContextMessages()` 对普通 message 原样投影，不过滤 `stopReason: "error"` 或 `"aborted"`，见 `packages/coding-agent/src/core/session-manager.ts:379-390`。只要失败 assistant 仍在当前 branch 的投影窗口内，session 重新加载后，它就会重新出现在 `buildSessionContext()` 的结果中。当前代码所说的“keep in session for history, remove from agent state”只保证当前进程里的立即重试，不是可持久恢复的排除标记。

现有 retry 测试验证当前运行中的第二次调用、事件和工具循环，没有构造“失败后成功、关闭 session、重新加载、检查 provider 上下文”的场景。这是源码可验证的持久化语义和测试缺口。若要让排除跨重启保持，可以增加显式 retry-link/tombstone entry，或让上下文投影识别被后续成功 attempt 替代的失败节点；仅在内存数组上 `slice()` 无法表达这个事实。

## 12. 失败结果矩阵

| 输入或失败点 | 决策 | 运行态结果 | JSONL 结果 |
|---|---|---|---|
| 已识别 overflow，compaction 成功 | 压缩并重放一次 | error assistant 被移除；从摘要和原 user/tool result 继续 | error + compaction + 后续结果均保留 |
| 第二次明确 overflow | 停止自动恢复 | 最后 error 留在运行态 | 两次 error 与一次 compaction 均保留 |
| silent `stop` overflow | 只压缩 | 完整 answer 保留，不 continue | answer + compaction 保留 |
| length-stop overflow | 代码试图压缩后 continue，但末尾仍是 assistant | `Agent.continue()` 拒绝；恢复链未闭合 | length assistant 与 compaction 保留 |
| overflow 未被识别 | 可能落入普通 retry，也可能直接结束 | 取决于 retry 文本分类 | error 保留 |
| compaction 关闭、无认证、无可压缩材料或摘要失败 | 不重放 overflow | 首次检查已移除明确 error 时，当前运行态不含它 | error 保留；无 compaction entry |
| 瞬时错误，后续成功 | 指数退避后 continue | 失败 attempts 从当前上下文移除 | 每次失败和成功都保留 |
| retry 次数耗尽 | 发出 `auto_retry_end(success: false)` | 最后一条 error 保留 | 所有 attempts 保留 |
| backoff 被取消 | 停止等待 | 触发 retry 的 error 已从当前运行态移除 | error 保留 |

自动恢复的判定依赖三件事：provider 给出的最终 message 是否足够准确，当前模型与失败 message 是否一致，以及压缩/重试提交前 AbortSignal 是否仍有效。排查“为什么没有继续”时，应按这个顺序查看 `stopReason/errorMessage/usage`、分类函数、`compaction_end` 或 `auto_retry_*` 事件，再对照运行态消息和 session branch；单看界面最后一条文本无法还原恢复决策。
