# 第 25 讲：AgentHarness 的状态边界与未完成迁移

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

`AgentHarness` 试图把 Agent 运行时的几个时间尺度分开：应用看到的是可随时修改的最新配置，正在发送的 provider 请求使用冻结快照，会话写入则按消息结束和 turn 结束的边界排队落盘。它已经直接驱动 `runAgentLoop()`，但尚未接替 coding-agent 中的 `AgentSession`。

```text
应用配置（可在运行中更新）
        │ createTurnState()
        ▼
turn snapshot（供一次 provider 请求使用）
        │ assistant + tool results 完成
        ▼
save point（落盘 pending writes，生成下一份快照）
        │ agent_end
        ▼
settled / idle

Session ──操作一棵会话树
   │
SessionStorage ──读写一个会话
   │
SessionRepo ──创建、查找、删除和 fork 多个会话
```

这套拆分解决的是并发更新和持久化顺序，不等于完整的断点恢复。当前实现还没有持久化运行队列、pending write 接受记录和未完成 operation，也没有 coding-agent 已有的自动压缩与应用级自动重试。

## 1. 它没有替换现行 AgentSession

### 两种协调器的边界

`AgentSession` 是 coding-agent 当前的应用协调器。它同时持有低层 `Agent`、旧 `SessionManager`、设置、资源加载器、模型运行时和扩展 runner，还管理自动压缩、自动重试、bash 任务与 UI 所需事件。它的状态字段集中在 `packages/coding-agent/src/core/agent-session.ts:284-354`。

`AgentHarness` 位于更低的 `packages/agent`。它不再持有 `Agent`，而是在 `executeTurn()` 中直接调用 `runAgentLoop()`。它拥有的是一次 agent run 所需的配置、会话写入、队列、操作锁和事件边界，见 `packages/agent/src/harness/agent-harness.ts:157-206`、`531-605`。

| 问题 | 现行 `AgentSession` | 新 `AgentHarness` |
| --- | --- | --- |
| 低层循环 | 持有 `Agent`，调用 `agent.prompt()` / `continue()` | 直接调用 `runAgentLoop()` |
| 会话 | 旧 `SessionManager`，同步文件接口 | 新 `Session` + 可替换 storage/repo，异步接口 |
| 扩展 | 完整 `ExtensionRunner`、命令、UI、资源与 provider 注册 | 少量 `subscribe()` / `on()` 事件；通用 hooks 仍在设计 |
| 自动压缩 | 已实现阈值和 overflow 恢复 | 只有显式 `compact()` |
| 应用级重试 | 已实现 assistant error 判定与指数退避 | 未实现；只有 provider 请求参数可配置重试 |
| coding-agent 接入 | 正在使用 | 尚未使用 |

官方的模型迁移说明直接标明 coding-agent 仍由 `AgentSession` 驱动，`AgentSession -> AgentHarness` 被列在 Pi 2.0 阶段，见 `packages/agent/docs/models.md:790-795`、`936-940`。因此，两套 Session 也是并存关系：新抽象不是旧 JSONL 会话实现已经完成的内部重构。

### 为什么先放在 agent 包

`AgentHarness` 只依赖模型集合、执行环境、会话和 agent loop，不承担 CLI、TUI、扩展命令或用户设置。这样可以先固定通用运行语义，再迁移 coding-agent 的应用能力。代价是当前不能把 harness 测试通过理解成 coding-agent 已获得同样行为；迁移还要处理旧扩展事件、资源来源、UI 状态和恢复兼容。

## 2. Turn snapshot 冻结一次请求

### 解决的问题

工具执行期间，监听器可能切换模型、thinking level、工具或系统提示。如果运行中的 provider 请求直接读取这些可变字段，一次请求的 model、tools 和 prompt 可能来自不同时间点。`AgentHarnessTurnState` 将一次模型调用需要的一组值放进同一份快照：

```ts
interface AgentHarnessTurnState<
	TSkill extends Skill = Skill,
	TPromptTemplate extends PromptTemplate = PromptTemplate,
	TTool extends AgentTool = AgentTool,
> {
	messages: AgentMessage[];
	resources: AgentHarnessResources<TSkill, TPromptTemplate>;
	streamOptions: AgentHarnessStreamOptions;
	sessionId: string;
	systemPrompt: string;
	model: Model<any>;
	thinkingLevel: ThinkingLevel;
	tools: TTool[];
	activeTools: TTool[];
}
```

源码入口是 `packages/agent/src/harness/agent-harness.ts:141-155`。`createTurnState()` 在一个位置读取会话上下文、资源、session id、工具和实时配置，并且只调用一次动态 system prompt provider：

```ts
private async createTurnState(): Promise<AgentHarnessTurnState<TSkill, TPromptTemplate, TTool>> {
	const context = await this.session.buildContext();
	const resources = this.getResources();
	const sessionMetadata = await this.session.getMetadata();
	const tools = [...this.tools.values()];
	const activeTools = this.activeToolNames
		.map((name) => this.tools.get(name))
		.filter((tool): tool is TTool => tool !== undefined);
	let systemPrompt = "You are a helpful assistant.";
	if (typeof this.systemPrompt === "string") {
		systemPrompt = this.systemPrompt;
	} else if (this.systemPrompt) {
		systemPrompt = await this.systemPrompt({
			env: this.env,
			session: this.session,
			model: this.model,
			thinkingLevel: this.thinkingLevel,
			activeTools,
			resources,
		});
	}
	return {
		messages: context.messages,
		resources,
		streamOptions: cloneStreamOptions(this.streamOptions),
		sessionId: sessionMetadata.id,
		systemPrompt,
		model: this.model,
		thinkingLevel: this.thinkingLevel,
		tools,
		activeTools,
	};
}
```

见 `packages/agent/src/harness/agent-harness.ts:314-346`。

### 状态怎样变化

`setModel()`、`setThinkingLevel()`、`setTools()`、`setResources()` 和 `setStreamOptions()` 修改 harness 的最新配置。getter 也读取这份最新配置，而不是当前请求的快照。正在流式传输的请求继续使用旧 `activeTurnState`；tool batch 结束后，save point 生成新快照，下一次 provider 请求才读取新值。

`packages/agent/test/harness/agent-harness.test.ts:293-356` 在第一次响应产生 tool call 后，同时修改模型、thinking、资源、系统提示和 active tools。断言第一次请求仍使用旧组合，第二次请求完整切到新组合。`packages/agent/test/harness/agent-harness-stream.test.ts:141-179` 对 stream options 固定了同样的边界。

### 快照不是深拷贝

资源数组、工具数组、headers 和 metadata map 会浅拷贝，对象内部值不做递归复制，见 `packages/agent/docs/agent-harness.md:58-76`。应用若原地修改 skill 或 metadata 内部对象，仍可能绕过边界。这里选择浅拷贝是为了避免复制任意工具实现和资源对象的成本；相应约束是调用方应把这些对象当成不可变值，用 setter 替换集合。

另一个边界是 `Models` 集合本身没有进入快照。具体 `Model` 被冻结，但 provider collection 仍由 harness 共享，见 `packages/agent/docs/models.md:763-771`。运行时替换同名 provider 的影响并未由 turn snapshot 隔离。

## 3. Operation phase 阻止结构操作重叠

### 状态模型

公开类型定义了五种 phase：

```ts
type AgentHarnessPhase =
	| "idle"
	| "turn"
	| "compaction"
	| "branch_summary"
	| "retry";
```

见 `packages/agent/src/harness/types.ts:494`。`prompt()`、`skill()`、`promptFromTemplate()`、`compact()` 和 `navigateTree()` 都先同步检查 `idle`，再在第一个 `await` 之前切换 phase。第二个结构操作不会排队，而是以 `AgentHarnessError` 的 `busy` code 拒绝。

这一顺序很重要。若先读取会话再标记 busy，两次同时进入的方法都可能通过检查，随后各自在同一 leaf 上追加记录。

### 允许在运行中发生的操作

运行中的 `steer()` 和 `followUp()` 进入低层 loop 的安全队列；`nextTurn()` 留给下一次用户发起的 turn。配置 setter 立即修改未来快照。`appendMessage()` 则转成 pending write。它们不是不受限制的并发写入，而是各自有明确的消费边界。

phase 没有公开 getter，外部只能从调用结果和事件判断状态。`waitForIdle()` 等待 `runPromise`，并不等待 compaction 或 tree navigation，因为这两种结构操作没有调用 `startRunPromise()`。这是当前实现边界，不应把方法名理解为“等待所有 phase 回到 idle”。官方文档也把 phase/settlement 语义标为仍需审计，见 `packages/agent/docs/agent-harness.md:94-120`、`314-325`。

`"retry"` 目前只存在于联合类型中；`agent-harness.ts` 没有把 phase 设置为该值。它是预留状态，不是已实现的重试阶段。

## 4. Pending session writes 保护记录顺序

### 为什么不能立即写

低层 loop 在 `message_end` 产生正式消息。如果 assistant 的 `message_end` 监听器又调用 `harness.appendMessage()`，直接写盘会让监听器消息与 loop 消息竞争 leaf。harness 的规则是先持久化 agent 消息，再通知监听器；监听器发起的写入在 busy 期间只进入 `pendingSessionWrites`。

```ts
private async handleAgentEvent(event: AgentEvent, signal?: AbortSignal): Promise<void> {
	if (event.type === "message_end") {
		await this.session.appendMessage(event.message);
		await this.emitAny(event, signal);
		return;
	}
	if (event.type === "turn_end") {
		let eventError: unknown;
		try {
			await this.emitAny(event, signal);
		} catch (error) {
			eventError = error;
		}
		const hadPendingMutations = this.pendingSessionWrites.length > 0;
		await this.flushPendingSessionWrites();
		if (eventError) throw eventError;
		await this.emitOwn({ type: "save_point", hadPendingMutations });
		return;
	}
	if (event.type === "agent_end") {
		await this.flushPendingSessionWrites();
		this.phase = "idle";
		await this.emitAny(event, signal);
		await this.emitOwn({ type: "settled", nextTurnCount: this.nextTurnQueue.length }, signal);
		return;
	}
	await this.emitAny(event, signal);
}
```

见 `packages/agent/src/harness/agent-harness.ts:488-515`。`packages/agent/test/harness/agent-harness.test.ts:358-387` 让 assistant `message_end` 监听器追加 custom message，最后固定落盘角色顺序为 `user -> assistant -> custom`。

### 写入失败时不丢队首

`flushPendingSessionWrites()` 总是读取数组第一个元素，等对应 Session append 成功后才 `shift()`，见 `packages/agent/src/harness/agent-harness.ts:462-486`。如果 storage 抛错，失败项与后续项仍留在内存队列。`executeTurn()` 的 `finally` 还会再次尝试 flush，并清理 abort controller。

这保证了进程存活期间的顺序和失败保留，但还不是 durable queue。busy 状态下的 `appendMessage()` 在把对象推入内存数组后就返回，接受记录没有先写入 JSONL；此时进程退出会丢失该项。官方 durability 设计要求“public API 返回前先持久化接受的 mutation”，当前源码尚未达到，见 `packages/agent/docs/durable-harness.md:83-116`。

## 5. Save point 是运行内屏障

低层 loop 的顺序位于 `packages/agent/src/agent-loop.ts:192-245`：provider 响应结束，执行工具并产生 tool result，然后发出 `turn_end`，最后调用 `prepareNextTurn()`。harness 在这两个回调中完成两部分工作：

1. `turn_end` 处理器先 flush pending writes，再发出 `save_point`。
2. `prepareNextTurn()` 再构造新 turn state，并把 context、model 和 thinking level 交还给低层 loop。

```ts
prepareNextTurn: async () => {
	await this.flushPendingSessionWrites();
	const nextTurnState = await this.createTurnState();
	setTurnState(nextTurnState);
	return {
		context: this.createContext(nextTurnState),
		model: nextTurnState.model,
		thinkingLevel: nextTurnState.thinkingLevel,
	};
},
getSteeringMessages: async () =>
	this.drainQueuedMessages(this.steerQueue, this.steeringQueueMode),
getFollowUpMessages: async () =>
	this.drainQueuedMessages(this.followUpQueue, this.followUpQueueMode),
```

见 `packages/agent/src/harness/agent-harness.ts:435-447`。

这里的 save point 表示“之前的 agent 消息和监听器写入已经按序进入 Session，可以为下一次请求重建上下文”。它没有记录 operation id、turn id、provider request 或 tool call 的开始/结束，也不能恢复传到一半的 stream。把它称为 checkpoint 容易高估现状。

动态 system prompt provider 若在首次 `createTurnState()` 抛错，`prompt()` 会恢复 idle 并拒绝；若在 save point 的新快照中抛错，异常进入低层 run 的失败处理，harness 生成并持久化 assistant error message。两者留下的会话结果不同。

## 6. Session、Storage 与 Repo 各管一层

### Session 管会话语义

新 `Session` 不自己决定文件放在哪里。它负责给 entry 生成 `id/parentId/timestamp`，执行标签目标和 move target 校验，并把 active branch 投影为模型上下文。`buildContext()` 先沿 leaf 取得完整分支，从完整分支归约 model、thinking 和 active tools，再应用 compaction transform 与自定义 transform，最后投影消息，见 `packages/agent/src/harness/session/session.ts:37-135`、`137-187`。

完整分支与模型上下文没有合并为同一对象。label、session info 和默认 custom entry 可以留在审计树里而不进入模型消息；compaction entry 会替换早期消息的投影视图。

### Storage 管一个会话的物理读写

`SessionStorage` 的接口位于 `packages/agent/src/harness/types.ts:441-455`：

```ts
export interface SessionStorage<TMetadata extends SessionMetadata = SessionMetadata> {
	getMetadata(): Promise<TMetadata>;
	getLeafId(): Promise<string | null>;
	setLeafId(leafId: string | null): Promise<void>;
	createEntryId(): Promise<string>;
	appendEntry(entry: SessionTreeEntry): Promise<void>;
	getEntry(id: string): Promise<SessionTreeEntry | undefined>;
	findEntries<TType extends SessionTreeEntry["type"]>(
		type: TType,
	): Promise<Array<Extract<SessionTreeEntry, { type: TType }>>>;
	getLabel(id: string): Promise<string | undefined>;
	getPathToRoot(leafId: string | null): Promise<SessionTreeEntry[]>;
	getEntries(): Promise<SessionTreeEntry[]>;
}
```

`InMemorySessionStorage` 适合测试和短生命周期宿主。`JsonlSessionStorage` 使用 version 3 header 和逐行 entry；追加时先调用文件系统 `appendFile()`，成功后才更新内存 entries、索引、label cache 和 current leaf，见 `packages/agent/src/harness/session/jsonl-storage.ts:271-280`。因此磁盘追加失败不会制造“内存声称成功、文件没有记录”的状态。

leaf 移动本身也会追加一条 `leaf` entry，而不是只改内存游标。重新打开 JSONL 时按记录顺序计算最后 leaf，见 `packages/agent/src/harness/session/jsonl-storage.ts:155-177`、`247-265`。这使 tree navigation 能跨重启保留。

### Repo 管多个会话的生命周期

`SessionRepo` 提供 `create/open/list/delete/fork`，见 `packages/agent/src/harness/types.ts:469-479`。它不参与单次 append，也不生成模型上下文。

`InMemorySessionRepo` 用 id 到 `Session` 的 map；`JsonlSessionRepo` 按 cwd 编码子目录，读取每个文件第一行列出 metadata，并把 fork 的分支复制进新 JSONL，见 `packages/agent/src/harness/session/memory-repo.ts:5-49`、`packages/agent/src/harness/session/jsonl-repo.ts:38-160`。fork 复制已有 entry id，新的 session header 记录 `parentSessionPath`；它是分支快照副本，不是两个文件共享可变节点。

分层后的替代方案是继续让 `SessionManager` 同时承担树语义、同步 Node 文件 I/O、目录查找和 fork。旧实现对 coding-agent 很直接，但难以在浏览器、远程文件系统或测试内存后端中复用。新接口增加了异步调用和对象数量，换来执行环境与存储后端可替换。

## 7. 失败、settlement 与事件回调

### 高层错误统一分类

Session、compaction 和 branch summary 的异常会被包装成 `AgentHarnessError`，原异常保留在 `cause`，见 `packages/agent/src/harness/agent-harness.ts:128-139`。结构方法使用 throw/reject，而文件系统等低层 capability 使用 `Result`；调用者不能忽略一次会话写入失败。

provider、context hook 或 loop 内部异常会进入 `emitRunFailure()`，生成一条 `stopReason: "error"` 的 assistant message，并人工走完 `message_start -> message_end -> turn_end -> agent_end`。测试 `packages/agent/test/harness/agent-harness.test.ts:262-291` 确认 context hook 抛错后 error message 已持久化，harness 也能接受下一次 prompt。

### 事件在提交后失败不会回滚

`message_end` 先写 Session，后通知订阅者。订阅者抛错时，已写消息不会撤销。`turn_end` listener 抛错会先被暂存，pending writes 仍执行 flush，然后异常才继续传播。状态变化与通知失败采用“提交后报错”，而不是事务回滚。

`agent_end` 将 phase 设回 idle，再 await `agent_end` subscriber 和 `settled` subscriber。此时结构操作的 busy 锁已经解除，上一轮 `prompt()` 却还没有返回；这正是官方待办中“settled 是否过早”的风险。`runPromise` 直到整个 `prompt()` 的 finally 才解除，所以外部 `waitForIdle()` 会等待这些回调。若某个 active-run callback 反过来 await 同一个 `waitForIdle()`，会形成自等待；官方文档把未来的 `runWhenIdle()` facade 作为解决方向，当前尚无该 API，见 `packages/agent/docs/agent-harness.md:7-22`、`314-325`。

## 8. 当前 on() 不是最终 hooks 架构

现有 harness 已支持两类回调：`subscribe()` 观察全部 agent/harness 事件，`on(type)` 为部分事件返回变换或阻断结果。provider request patch 和 provider payload 已按注册顺序链式传递，测试位于 `packages/agent/test/harness/agent-harness-stream.test.ts:89-139`、`181-212`。

但是通用 `emitHook()` 只保存最后一个非 `undefined` 返回值，各 handler 收到同一个原始 event，见 `packages/agent/src/harness/agent-harness.ts:232-249`。这意味着多个 `tool_result` handler 还没有“后一个看到前一个 patch”的 reducer 语义。hook context、来源 metadata、cleanup、facade 和可配置错误策略也没有实现。

`packages/agent/docs/hooks.md` 描述的是目标设计，不是可调用 API。设计中的 `HookEvent` phantom result、`AgentHarnessHooks`、`observe()`、`setContext()`、`clear()` 和 `dispose()` 在当前 `src/harness` 中不存在。文档还要求：

- context 和 provider payload 按前一处理器结果继续变换；
- tool call 遇到 block 提前结束；
- tool result 逐次累积 patch；
- session-before 事件遇到 cancel 提前结束；
- 注册项携带扩展来源，hook context 暴露受控 facade。

这些语义要等 reducer 与 facade 落地后才能视为保证。当前能确认的是少量特制事件路径，而不是完整的扩展系统替代品。

## 9. JSONL 持久化不等于 durable harness

### 已经能恢复的状态

当前新 Session 可以从 JSONL 恢复：

- 消息和 custom entry；
- model、thinking level、active tools 的变更 entry；
- compaction、branch summary、label 和 session name；
- 最后一次持久化 leaf。

不过 `AgentHarness` 构造器使用显式传入的 model、thinking 和 active tools。`createTurnState()` 虽然调用 `Session.buildContext()`，却只采用其中的 `messages`，没有用归约出的 `context.model`、`thinkingLevel` 和 `activeToolNames` 覆盖构造参数。也就是说，entry 已能保存这些状态，harness 自动 restore 仍未实现。

### 崩溃时仍会丢什么

以下状态只存在于 harness 内存字段：

- steer、follow-up 和 next-turn 队列；
- 尚未 flush 的 pending session writes；
- 当前 phase、operation 和 turn 进度；
- 已开始但未完成的 provider 请求；
- 已开始但结果未落盘的工具调用。

`packages/agent/docs/durable-harness.md` 提出的半持久化方案会增加 queue enqueue/consume、pending write enqueue/applied、operation、turn、provider request 和 tool call 等 entry，再从 session log 归约恢复状态。该方案还没有对应类型和 reducer。

工具调用尤其不能在重启后盲目重跑。外部写操作可能已经成功，只是结果尚未记入会话。设计文档的保守策略是标记 interrupted，只有工具明确声明 idempotent/retry-safe 才自动重试，见 `packages/agent/docs/durable-harness.md:118-180`。这是设计结论，不是当前运行行为。

## 10. “重试”实际分成三层

| 层次 | 当前状态 | 触发者与结果 |
| --- | --- | --- |
| provider transport retry | 已接通 | `streamOptions.maxRetries/maxRetryDelayMs` 传给 `Models.streamSimple()`，由 provider 请求层决定 |
| assistant error 后自动 retry | harness 未实现 | 现行 `AgentSession._prepareRetry()` 判断 retryable error、指数退避并 `agent.continue()` |
| 崩溃恢复后 retry | 仅设计 | durable 文档要求从持久边界恢复，非幂等工具默认不重跑 |

现行 `AgentSession` 的应用级 retry 位于 `packages/coding-agent/src/core/agent-session.ts:1049-1091`、`2602-2693`。它保留错误消息到旧 SessionManager，但从 Agent 内存 context 移除最后一条 error，再等待可中止的指数退避。`AgentHarness` 没有 `_isRetryableError()`、retry counter 或 backoff controller，也没有 auto-compaction decision point，见 `packages/agent/docs/agent-harness.md:210-218`。

因此，设置 `maxRetries` 不能替代 `AgentSession` 的自动重试：前者处理单次 provider 请求的传输失败，后者读取完成后的 assistant error message 并决定是否继续整个 agent run。

## 11. 测试固定了哪些边界

新 harness 测试使用 faux provider，不依赖真实网络。关键断言分布如下：

| 行为 | 测试锚点 |
| --- | --- |
| steering / follow-up 每次安全点消费一条 | `packages/agent/test/harness/agent-harness.test.ts:89-130`、`219-260` |
| abort 清空 steer/follow-up，保留 nextTurn | `packages/agent/test/harness/agent-harness.test.ts:164-217` |
| save point 刷新模型、thinking、资源、prompt 和工具 | `packages/agent/test/harness/agent-harness.test.ts:293-356` |
| agent 消息先于 listener pending write | `packages/agent/test/harness/agent-harness.test.ts:358-387` |
| `waitForIdle()` 等待异步 listener | `packages/agent/test/harness/agent-harness.test.ts:389-419` |
| stream option 当前请求冻结、下一请求刷新 | `packages/agent/test/harness/agent-harness-stream.test.ts:141-179` |
| 同一 Session 语义在内存和 JSONL storage 上运行 | `packages/agent/test/harness/session.test.ts:19-203` |
| repo 的 create/open/list/delete/fork | `packages/agent/test/harness/repo.test.ts:8-92` |

这些测试证明的是正常进程生命周期内的顺序、快照和后端一致性。它们没有模拟进程在 queue enqueue、pending flush、provider stream 或 tool call 中途退出，也没有覆盖通用 hooks reducer、应用级 retry 或 coding-agent 迁移。

## 12. 设计取舍与当前判断

`AgentHarness` 的主要进展是明确状态所有权：实时配置属于 harness，单次请求属于 turn snapshot，持久会话树属于 Session，物理读写属于 storage，会话集合生命周期属于 repo。save point 把这些层连接起来，使运行中更新可以在下一次 provider 请求生效，又不污染当前请求。

这套结构比 `AgentSession` 的集中协调更容易复用和测试，也把写入顺序表达得更明确。当前成本是两套会话系统并存，应用能力尚未迁移，部分 API 名称比实际保证更强：`waitForIdle()` 不覆盖所有结构 phase，save point 不能做崩溃恢复，pending write 也不是 durable queue。

在固定源码基线上，可以把 `AgentHarness` 视为已实现的通用运行内核原型；不能把它视为 coding-agent 的现行入口、完整扩展宿主或耐崩溃执行器。后续迁移是否成立，取决于 hooks reducer/facade、异步 restore、durable operation journal、auto-compaction、retry 和 lifecycle hardening tests 是否真正进入源码。
