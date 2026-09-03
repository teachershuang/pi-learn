# 第 22 讲：SDK、JSON 与 RPC 的宿主边界

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

SDK、JSON 模式和 RPC 模式最终都驱动 `AgentSession`，差别不在模型能力，而在宿主如何跨过边界：SDK 传递对象和 Promise，JSON 模式只向外投递一次运行的事件，RPC 则让另一个进程持续发送命令并接收响应。边界一变，状态所有权、完成信号、UI 能力和故障范围也随之改变。

```text
进程内 SDK                    单向 JSON                     双向 RPC
宿主代码                       调用方进程                    RPC 客户端
  │ 对象调用 / 事件回调          │ 启动参数                    │ stdin：command / UI response
  ▼                            ▼                            ▼
AgentSession                runPrintMode(json)          runRpcMode
  │ 直接暴露活动状态             │ 订阅活动 session             │ command handler
  ▼                            │ stdout：header + event       │ stdout：response + event + UI request
SessionManager                ▼                            ▼
                           JSONL 消费者                   客户端派生状态
```

三种形式中的权威对话状态仍在 `AgentSession` 与 `SessionManager`。JSON 消费者和 RPC 客户端拿到的是事件或快照，不会因为在本地保存了一份消息数组就取得会话所有权。

## 1. 共同内核与三种宿主契约

### 解决的问题

同一个编码 Agent 既要嵌入 Node.js 程序，也要能被 shell 管道消费，还要允许其他语言或 IDE 长期控制。若每种入口各自实现 prompt、工具、压缩和持久化，状态机很快会分叉。Pi 把这些能力留在 `AgentSession`，运行模式只负责接入和退出。

`packages/coding-agent/src/main.ts:798-855` 完成模式分派：RPC 进入不会正常返回的 `runRpcMode()`；JSON 与 text 共用 `runPrintMode()`；SDK 调用者则绕过 CLI，直接使用 `createAgentSession()` 或 `createAgentSessionRuntime()`。

| 边界 | 输入 | 输出 | 权威活动状态 | 调用方看到的完成 |
|---|---|---|---|---|
| SDK | 方法参数、对象引用 | Promise、事件回调、直接属性 | 当前进程中的 `AgentSession` | 取决于所 await 的方法；`session.prompt()` 等到整轮 settle |
| JSON | 启动参数中的初始消息和后续消息列表 | session header 与 session events | 子进程中的 `AgentSession` | 进程退出；没有逐命令 response |
| RPC | stdin 上连续的 command | stdout 上交错的 response、event、UI request | RPC 子进程中的当前 session | command response 只确认命令结果；运行完成看 `agent_settled` |

这里的 RPC 是 Pi 自己定义的 JSONL 协议，不是 JSON-RPC 2.0：消息由 `type` 区分，命令结构是 TypeScript 联合类型，没有 `jsonrpc`、`method`、`params` 那套信封。

## 2. SDK：宿主持有对象，也承担生命周期

### 源码入口与状态变化

`createAgentSession()` 位于 `packages/coding-agent/src/core/sdk.ts:164-392`。它解析 cwd 和 agent 目录，创建或采用调用者传入的 `SessionManager`、`SettingsManager`、`ModelRuntime` 与 `ResourceLoader`，恢复 session 中的模型、thinking 和消息，再组装 `Agent` 与 `AgentSession`。

调用者拿到的不是远端句柄，而是活动对象本身：可以读取 `session.agent.state`，订阅事件，替换工具或注入 in-memory session manager。`packages/coding-agent/src/core/sdk.ts:357-392` 的收尾体现了这层关系：

```ts
if (existingSession) {
	agent.state.messages = sessionContext.messages;
} else {
	if (selectedModel) {
		sessionManager.appendModelChange(selectedModel.provider, selectedModel.id);
	}
	sessionManager.appendThinkingLevelChange(thinkingLevel);
}

const session = new AgentSession({
	agent,
	sessionManager,
	settingsManager,
	cwd,
	modelRuntime,
	resourceLoader,
	customTools,
	toolDefinitionRegistry,
	extensionRunnerRef,
	initialActiveToolNames,
	sessionStartEvent: options.sessionStartEvent,
});

return { session, extensionsResult, modelFallbackMessage };
```

SDK 中的直接访问很方便，也意味着调用者能破坏抽象。例如直接改 `agent.state.messages` 可以绕过 `SessionManager` 的 append 记录；直接保留旧 session 引用，又可能在 runtime 换会话后操作失效对象。类型安全只约束 TypeScript 形状，不等于状态修改自动合法。

### `prompt()` 的 Promise 是运行边界

`AgentSession.prompt()` 的 `preflightResult` 会在输入被接受、入队或被扩展命令处理时回调，但公开 Promise 仍等到重试、自动压缩和队列续跑全部结束。这个区分在 `packages/coding-agent/docs/sdk.md:180-234` 有明确契约。

因此进程内代码可以写：

```ts
const events: AgentSessionEvent[] = [];
const unsubscribe = session.subscribe((event) => events.push(event));

try {
	await session.prompt("检查当前改动");
	// 此处本轮 session-level run 已经 settled。
} finally {
	unsubscribe();
	session.dispose();
}
```

但 Promise 正常返回不等于模型成功。provider 错误通常会收束成 `stopReason: "error"` 的 assistant message；宿主仍需检查最终消息或事件。同步预检错误、扩展回调异常等路径则可能 reject。

### 会话替换属于 `AgentSessionRuntime`

单个 `AgentSession` 负责当前会话。new、resume、fork、import 会改变 cwd-bound services 或 session manager，因此由 `AgentSessionRuntime` 替换整个活动 runtime，而不是在旧对象上改几个字段。

`packages/coding-agent/src/core/agent-session-runtime.ts:167-191` 先关闭旧 session，再安装新结果并重新绑定宿主：

```ts
private async teardownCurrent(
	reason: SessionShutdownEvent["reason"],
	targetSessionFile?: string,
): Promise<void> {
	await emitSessionShutdownEvent(this.session.extensionRunner, {
		type: "session_shutdown",
		reason,
		targetSessionFile,
	});
	this.beforeSessionInvalidate?.();
	this.session.dispose();
}

private apply(result: CreateAgentSessionRuntimeResult): void {
	this._session = result.session;
	this._services = result.services;
	this._diagnostics = result.diagnostics;
	this._modelFallbackMessage = result.modelFallbackMessage;
}
```

先销毁再创建让旧扩展、watcher 和 UI 不会与新 runtime 并存，代价是创建新 runtime 失败时没有可自动回滚的旧 session。SDK 宿主必须接住异常，并在替换后重新获取 `runtime.session`、重新订阅事件和绑定扩展。

## 3. SDK 的扩展 UI 默认不存在

### 触发位置与输入输出

`createAgentSession()` 只创建核心对象，不知道宿主有没有界面。`ExtensionRunner` 初始使用 `noOpUIContext`；select/input/editor 返回 `undefined`，confirm 返回 `false`，通知与状态更新不做事，见 `packages/coding-agent/src/core/extensions/runner.ts:233-264`。

宿主调用 `session.bindExtensions()` 才把 UI、模式、命令动作、abort 和 shutdown 处理器写入 runner。入口在 `packages/coding-agent/src/core/agent-session.ts:2209-2232`：

```ts
async bindExtensions(bindings: ExtensionBindings): Promise<void> {
	if (bindings.uiContext !== undefined) {
		this._extensionUIContext = bindings.uiContext;
	}
	if (bindings.mode !== undefined) {
		this._extensionMode = bindings.mode;
	}
	if (bindings.shutdownHandler !== undefined) {
		this._extensionShutdownHandler = bindings.shutdownHandler;
	}

	this._applyExtensionBindings(this._extensionRunner);
	await this._extensionRunner.emit(this._sessionStartEvent);
	await this.extendResourcesFromExtensions(
		this._sessionStartEvent.reason === "reload" ? "reload" : "startup",
	);
}
```

这意味着“SDK 支持自定义 UI”的准确含义是：宿主可以实现并注入 `ExtensionUIContext`。仅调用 `createAgentSession()` 不会凭空获得对话框。绑定还是 `session_start` 的触发点；漏掉它不只影响 UI，也会漏掉扩展启动事件和动态资源扩展。

## 4. JSON 模式：一次运行的只读事件投影

### 运行流程

JSON 模式没有独立 Agent 实现。`runPrintMode()` 以 `mode: "json"` 绑定扩展，先输出已有 session header，再订阅并原样序列化后续 `AgentSessionEvent`。初始消息和 `messages` 数组依次 await，所有输入处理完便退出。

`packages/coding-agent/src/modes/print-mode.ts:100-127` 是它的核心：

```ts
if (mode === "json") {
	const header = session.sessionManager.getHeader();
	if (header) {
		writeRawStdout(`${JSON.stringify(header)}\n`);
	}

	unsubscribe = session.subscribe((event) => {
		writeRawStdout(`${JSON.stringify(event)}\n`);
	});
}

try {
	if (initialMessage) {
		await session.prompt(initialMessage, { images: initialImages });
	}
	for (const message of messages) {
		await session.prompt(message);
	}
	// ...text 模式才挑出最终 assistant 文本
	return 0;
} catch (error) {
	console.error(chalk.red(`Error: ${error instanceof Error ? error.message : "Unknown error"}`));
	return 1;
}
```

JSON 消费者能据事件构建进度、消息和工具执行视图，却不能在同一通道发送 abort、steer、切换模型或回答扩展对话框。即使启动时传入多个 prompt，它们也是进程内预先确定的顺序调用，不是外部请求流。

### 状态与失败边界

首行 header 给出 session 身份，后续 event 描述运行中的变化。它们适合日志、管道和只读 UI，但不是可恢复检查点：某个消费者漏掉事件后，协议没有 `get_state` 或 `get_entries` 命令供它补拉。

JSON 与 text 模式共用清理路径，但不共用最终消息的退出码判断。只有 text 分支检查末条 assistant 的 `stopReason` 并把 error/aborted 改为 1；JSON 分支已把失败 message/event 写入协议流，`session.prompt()` 正常收束时仍返回 0。两种模式遇到会 reject 的同步异常才统一返回 1。`finally` 始终取消订阅、移除信号处理器、调用 `runtimeHost.dispose()` 并刷新协议输出。SIGTERM/SIGHUP 则在清理 detached 子进程与 runtime 后使用 143/129 退出，相关实现位于 `packages/coding-agent/src/modes/print-mode.ts:111-158`。

JSONL stdout 是协议面，不能混入扩展或依赖的普通日志。`takeOverStdout()` 把此后的常规 stdout 写入重定向到 stderr，只有 `writeRawStdout()` 仍写协议流，见 `packages/coding-agent/src/core/output-guard.ts:45-93`。

## 5. RPC：命令响应与运行事件是两条时间线

### 协议帧与相关性

RPC 进程长期读取 stdin。每个 `RpcCommand` 可带 `id`，相应 `response` 回显该 ID；agent events 不带请求 ID，因为一次运行可能含重试、压缩和多个排队输入，不能可靠归属于单个同步调用。

记录边界严格使用 LF。`packages/coding-agent/src/modes/rpc/jsonl.ts:4-58` 用 `StringDecoder` 处理被 chunk 切开的 UTF-8，只在 `\n` 处分帧，并兼容删除 LF 前的 `\r`。不用 Node `readline` 是为了保留 JSON 字符串中合法的 U+2028/U+2029。

解析失败会返回 `command: "parse"` 的失败响应；未知命令也返回失败，并保留原请求 ID。它们没有改变 session。命令 handler 抛错时，由最外层 dispatch catch 归一化为当前 command 的失败 response。

### 为什么 `prompt` 不能等待执行完成再回包

普通 RPC 命令由 handler await 后返回。`prompt` 是例外：handler 启动 `session.prompt()`，但把 `preflightResult(true)` 当作唯一成功响应时点。`packages/coding-agent/src/modes/rpc/rpc-mode.ts:393-415` 的实现如下：

```ts
case "prompt": {
	let preflightSucceeded = false;
	void session
		.prompt(command.message, {
			images: command.images,
			streamingBehavior: command.streamingBehavior,
			source: "rpc",
			preflightResult: (didSucceed) => {
				if (didSucceed) {
					preflightSucceeded = true;
					output(success(id, "prompt"));
				}
			},
		})
		.catch((e) => {
			if (!preflightSucceeded) {
				output(error(id, "prompt", e.message));
			}
		});
	return undefined;
}
```

提前回包允许同一客户端在模型运行时继续发送 steer、follow-up、abort 或查询命令。若 response 要等整轮完成，单连接客户端很容易把“命令调用”误当成串行任务队列，也无法及时控制正在运行的 Agent。

代价是两种成功必须分开：

- `response(success: true)`：输入已接受、已入队，或扩展命令已经处理；
- `agent_settled`：session-level run 不再有自动重试、压缩重试或队列续跑。

接受后的 provider 失败只出现在 message/event 流中，不会为同一个 ID 再发第二个失败 response。`packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:187-287` 分别固定了预检失败、立即接受和流式入队三条路径，每条都只有一次 response。

## 6. RpcClient 保存请求账本，不保存权威会话

`RpcClient` 是 TypeScript 的子进程包装器。`send()` 分配 `req_N`，把 resolve/reject 放入 `pendingRequests`，写入 stdin，并设置 30 秒 response 超时。stdout 收到带匹配 ID 的 response 时结算对应 Promise；其余对象都交给 event listeners。

`packages/coding-agent/src/modes/rpc/rpc-client.ts:499-529` 展示了这层分流：

```ts
private handleLine(line: string): void {
	try {
		const data = JSON.parse(line);

		if (data.type === "response" && data.id && this.pendingRequests.has(data.id)) {
			const pending = this.pendingRequests.get(data.id)!;
			this.pendingRequests.delete(data.id);
			pending.resolve(data as RpcResponse);
			return;
		}

		for (const listener of this.eventListeners) {
			listener(data as AgentSessionEvent);
		}
	} catch {
		// Ignore non-JSON lines
	}
}
```

这里的 map 只拥有请求相关性。模型、messages、队列、当前 leaf 与 session manager 仍在子进程。客户端需要权威快照时调用 `get_state`、`get_entries` 或 `get_tree`；本地 event reducer 只是缓存或展示模型。

`client.prompt()` 只 await 接受响应。`promptAndWait()` 则先订阅事件，再发送 prompt，最后等 `agent_settled`。先订阅很重要：如果先 await prompt 再开始监听，极快的运行可能已经发出结束事件。即便如此，共享 RPC 连接上同时存在其他运行时，`agent_settled` 也没有 request ID；需要严格的任务级相关性时，现协议还要由上层限制并发或另加业务标识。

### 失败路径

请求 30 秒未收到 response 会 reject，但子进程可能仍在执行；超时不会自动发送 abort。非 JSON stdout 行被客户端静默忽略，所以服务端通过 stdout takeover 防止日志污染十分关键。子进程退出、spawn 错误或 stdin 写失败会保存 `exitError` 并拒绝全部 pending requests，避免 Promise 永久悬挂；`packages/coding-agent/test/rpc-client-process-exit.test.ts:23-37` 用退出码 43 固定了这项保证。

## 7. RPC 的会话替换必须重建桥

`new_session`、`switch_session`、`fork` 和 `clone` 可能让 `runtimeHost.session` 指向新对象。原订阅、扩展 runner 与 UI binding 都属于旧 session，不能继续使用。

`runRpcMode()` 把 `rebindSession()` 注册给 runtime。它先取得新的 `runtimeHost.session`，绑定 RPC UI、命令动作和 shutdown handler，再取消旧订阅并订阅新 session，见 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:312-363`。

固定基线还有一处重复绑定：`AgentSessionRuntime.finishSessionReplacement()` 已经调用注册的 rebind callback；RPC 的 `new_session`、`switch_session`、`fork` 和 `clone` handler 在替换成功后又显式调用一次 `rebindSession()`，例如 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:432-438`。第二次绑定会再次发出 `session_start`，并重新安装订阅。现有 RPC 单元测试多用不会执行已注册 callback 的 mock runtime，尚未覆盖真实 runtime 下的调用次数。这是源码可验证的实现缺口；理想修复是只保留 runtime callback 这一条重绑定入口，并增加真实 `AgentSessionRuntime` 的集成测试。

底层 agent 订阅器还会 await `waitForRawStdoutBackpressure()`。这让 event listener 成为输出屏障：协议写队列排空前，低层 run 不会越过相应事件边界。它降低高速流把 stdout 缓冲打爆的风险，也把客户端读速纳入运行时延迟。

状态切换失败不是事务回滚。`AgentSessionRuntime` 先 teardown 旧 session 再创建新 session；新建失败会作为 RPC command failure 返回，此时旧 session 已失效。客户端不应在失败后继续相信切换前缓存，而应查询状态或重启进程。

## 8. 扩展 UI 是嵌套在 RPC 里的反向请求

扩展原本运行在 Agent 进程中，却可能需要外部界面回答 select、confirm、input 或 editor。RPC 因而在正常的“客户端请求、服务端响应”之外，增加一条反向链：

```text
extension ctx.ui.confirm(...)
  → server 建立 pendingExtensionUIRequests[id]
  → stdout: extension_ui_request
  → client 展示自己的 UI
  → stdin: extension_ui_response(id, confirmed/cancelled)
  → server resolve 原 Promise
  → extension 继续执行
```

`packages/coding-agent/src/modes/rpc/rpc-mode.ts:89-130` 创建 dialog Promise，响应值、AbortSignal 或 timeout 都可结算并清理 pending map。select/input 取消时得到 `undefined`，confirm 得到 `false`。notify、status、widget、title 与 editor text 只发 request，不等待回答。

RPC UI 不是 TUI 的远程镜像。`custom()`、footer/header component、working indicator、autocomplete 和 tool expansion 等需要直接组件或终端对象的能力会降级为 no-op 或固定返回值。`ctx.hasUI` 仍为 true，因为对话框与通知可用；扩展若需要真实终端能力，应检查 `ctx.mode === "tui"`，而不是只检查 `hasUI`。

另一个细节是 `editor()` 在固定基线中单独创建 pending Promise，没有复用带 AbortSignal 和 timeout 清理的 `createDialogPromise()`。官方协议把 editor 归入 dialog，但其服务端实现没有同等的超时/中止回收路径；客户端若不回应，该扩展调用会一直等待。这是源码可验证的实现差异，不应把其他 dialog 的 timeout 保证外推给 editor。

JSON 和默认 SDK 不具备这条桥。JSON 的 print binding 使用 no-op UI；SDK 只有在宿主显式实现并传入 `uiContext` 后才能提供等价交互。

## 9. 进程退出是协议的一部分

### JSON / print

输入列表处理完即返回退出码。text 模式把末条 assistant error/aborted 映射为 1；JSON 模式不做这次检查，调用方必须解释事件中的最终消息。会让 `session.prompt()` reject 的异常在两种模式中都返回 1。无论成功失败，`finally` 都执行 extension `session_shutdown`、session dispose 和 stdout flush。CLI 只在非零时设置 `process.exitCode`，让事件循环自然排空。

### RPC 服务端

RPC 没有“最后一条命令”。stdin EOF 是宿主关闭连接的正常信号，会触发 shutdown。SIGTERM/SIGHUP 先杀 detached 子进程，再 dispose runtime，并分别以 143/129 退出。扩展调用 shutdown 时只设置请求标记：若 Agent 正在运行，服务端等到 `agent_settled` 再退出，避免在扩展回调或工具执行中途销毁 session。

`packages/coding-agent/src/modes/rpc/rpc-mode.ts:696-789` 的 shutdown 会取消输入与事件订阅、移除信号监听、dispose runtime、暂停 stdin，并在非 SIGTERM 路径刷新 stdout。SIGTERM 特意跳过 flush，说明信号退出更偏向及时终止；客户端不能假设最后几个已排队帧一定送达。

### RpcClient

`RpcClient.stop()` 停止读取 stdout，发送 SIGTERM，最多等一秒后 SIGKILL。它不是协议内的 graceful-close command，也不会等待某个最终事件。若需要先让 Agent 停止，应先发送 `abort()` 并等状态收束，再 stop；如果只调用 stop，进程隔离保证了资源最终可回收，但不能保证业务操作完整。

## 10. 失败结果总表

| 失败位置 | 决策者 | 协议结果 | 状态后果 |
|---|---|---|---|
| SDK prompt 预检失败 | `AgentSession` | Promise reject，`preflightResult(false)` | 未开始新的 provider run |
| SDK provider 在接受后失败 | agent/session | assistant error message 与事件，Promise通常正常收束 | 失败消息进入活动状态及持久化路径 |
| JSON assistant error/aborted | agent/session | 失败 message/event 已输出；prompt 正常收束时进程仍可退出 0 | finally 关闭当前 runtime；消费者必须读事件判断业务失败 |
| JSON 消费者漏事件 | 外部消费者 | 无补拉命令 | 只能从持久化 session 或重新运行恢复 |
| RPC JSON 解析失败 | RPC dispatcher | `command: parse` 失败 response | session 不变 |
| RPC 未知命令 | RPC dispatcher | 保留 ID 的失败 response | session 不变 |
| RPC prompt 预检失败 | `AgentSession` | 该 ID 一次失败 response | 没有后续 run |
| RPC prompt 接受后失败 | agent/session | 已有成功 response；失败只走 events/messages | 客户端须等并解释运行事件 |
| RPC response 超时 | `RpcClient` | 对应 Promise reject | 服务端任务可能仍运行，不自动 abort |
| RPC 子进程退出 | `RpcClient` | 全部 pending Promise reject | 子进程内活动状态丢失；已落盘 session 可另行恢复 |
| runtime 换会话创建失败 | `AgentSessionRuntime` | SDK throw 或 RPC 失败 response | 旧 session 已 teardown，不能假设自动回滚 |
| RPC dialog 无回应 | RPC UI bridge | timeout/abort 时给默认值；editor 可能持续等待 | pending UI Promise 阻塞扩展调用 |

## 11. 设计取舍与选择依据

SDK 的优势不只是少一次序列化。它让宿主直接注入工具、资源和模型运行时，也能把事件处理 Promise 放进同一调用链。代价是故障隔离弱，错误的状态修改、未 dispose 的 watcher 或阻塞 listener 都发生在宿主进程内。

JSON 模式把集成面压到“启动参数 + stdout JSONL”，最容易接进 shell、日志系统和一次性流水线。它没有反向控制和断线补偿，因此不适合需要持续会话控制的 IDE。若为它增加 stdin 命令、ID 和状态查询，它就会逐步变成 RPC。

RPC 用序列化、请求账本和生命周期处理换取语言无关与进程隔离。服务端崩溃不会直接破坏宿主内存，但正在等待的 UI、请求和未持久化状态都必须显式失败。命令 response 与 event 分离让控制保持响应，也要求客户端建立两套状态：请求是否被接受，以及 Agent 实际运行到哪里。

三种边界没有能力高低的固定排序。一次性 CI 采集适合 JSON；同进程深度定制适合 SDK；IDE、桌面应用或非 Node 宿主适合 RPC。真正的选择标准是状态应由谁持有、故障需要隔离到哪里、调用方是否必须在运行中继续发命令，以及扩展 UI 要到什么程度。

## 12. 源码测试锁定的协议保证

- `packages/coding-agent/test/sdk-session-manager.test.ts:28-94`：SDK 默认持久化路径使用 agentDir，显式 manager 不被替换，省略 cwd 时可从 manager 推导。
- `packages/coding-agent/test/print-mode.test.ts:93-142`：text/JSON 成功路径都会触发 session shutdown；覆盖的 assistant error 用例属于 text 模式并返回非零。
- `packages/coding-agent/test/rpc-jsonl.test.ts:5-65`：只以 LF 分帧，保留 U+2028/U+2029，兼容 CRLF 与无结尾 LF 的最后一帧。
- `packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:187-287`：prompt 预检失败、接受和流式入队都恰好返回一次相关 response。
- `packages/coding-agent/test/suite/regressions/5868-rpc-unknown-command-id.test.ts:83-112`：未知命令的失败响应保留请求 ID。
- `packages/coding-agent/test/rpc-client-process-exit.test.ts:23-37`：RPC 子进程意外退出会拒绝正在等待的客户端请求。

这些测试分别约束了对象注入、一次性清理、协议分帧、响应时点和进程故障。它们没有证明外部客户端保存的派生状态能自动恢复，也没有给 RPC event 增加 request 相关性；这两项仍是宿主层职责。
