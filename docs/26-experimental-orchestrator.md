# 第 26 讲：实验性 Orchestrator 与多实例进程管理

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的 `orchestrator` 不是负责拆任务和汇总答案的 LLM supervisor。它是一个本机进程主管：维护一条 IPC socket，为每个实例启动独立的 coding-agent RPC 子进程，把命令转给指定实例，再把响应、AgentSession 事件和扩展 UI 请求送回客户端。

```text
orchestrator CLI / 其他本机客户端
                 │
                 │ 外层 JSONL：spawn / list / status / stop / rpc / rpc_stream
                 ▼
        本机 IPC socket server
                 │
                 ▼
       OrchestratorSupervisor
          │              │
          │              └── instances.json / 可选 Radius presence
          │
          ├── RpcProcessInstance A ── stdin/stdout JSONL ── pi RPC 子进程 A
          ├── RpcProcessInstance B ── stdin/stdout JSONL ── pi RPC 子进程 B
          └── RpcProcessInstance C ── stdin/stdout JSONL ── pi RPC 子进程 C
```

多实例在这里表示多个相互隔离的进程和会话。源码没有 supervisor agent、任务图、共享上下文、Agent 间消息或结果合并协议。

## 1. 包入口先声明了实验边界

`packages/orchestrator/README.md:1-5` 只给出一条实质性说明：包处于实验阶段，CLI、API 和行为都不稳定，甚至可能被移除。`package.json` 依赖 `@earendil-works/pi-coding-agent`，构建目标包含 `src/cli.ts`，但当前发布配置没有 `bin` 字段，见 `packages/orchestrator/package.json:1-44`。因此源码中的 `orchestrator` 命令不能直接等同于 npm 安装后已有的全局命令。

程序有两个入口面：

- `src/index.ts` 导出 supervisor、IPC、storage 和 Radius 等内部能力；
- `src/cli.ts` 提供 `serve/list/spawn/status/stop/rpc/rpc-stream` 命令。

`serve` 是常驻进程，其余命令是 socket 客户端。单次命令建立连接、发送一条请求、读取第一条响应后关闭；`rpc-stream` 保持连接并双向转发 JSONL，见 `packages/orchestrator/src/ipc/client.ts:5-62`、`packages/orchestrator/src/cli.ts:37-77`。

## 2. Supervisor 管进程，不理解 Agent 任务

### 内存状态与持久记录

每个在线实例在 supervisor 中有一份 `LiveInstance`：

```ts
interface LiveInstanceResources {
	rpcProcess?: RpcProcessInstance;
	radiusPiId?: string;
	sessionId?: string;
}

interface LiveInstance {
	record: InstanceRecord;
	resources: LiveInstanceResources;
	subscribers: Set<AgentSessionEventListener>;
	onUiRequest?: (request: RpcExtensionUIRequest) => void;
	unsubscribeEvents?: () => void;
	unsubscribeExit?: () => void;
}
```

见 `packages/orchestrator/src/supervisor.ts:15-28`。`record` 是可以写入 `instances.json` 的摘要；`resources`、事件订阅和子进程句柄只在当前 orchestrator 进程内有效。

`InstanceRecord` 只有 id、status、cwd、时间、label、session 和 Radius id。它不记录当前 prompt、工具调用、队列、模型、token 预算或任务依赖，见 `packages/orchestrator/src/types.ts:1-25`。

### 一个实例怎样启动

`spawnInstance()` 的状态流转是：

```ts
async spawnInstance(options: { cwd: string; label?: string }): Promise<InstanceRecord> {
	const now = new Date().toISOString();
	const live: LiveInstance = {
		record: {
			id: randomUUID(),
			status: "starting",
			cwd: options.cwd,
			createdAt: now,
			lastSeenAt: now,
			label: options.label,
		},
		resources: {},
		subscribers: new Set(),
	};
	this.liveInstances.set(live.record.id, live);
	upsertInstance(live.record);

	try {
		const rpcProcess = createRpcProcessInstance({ cwd: options.cwd });
		this.bindRpcProcess(live, rpcProcess);
		await this.syncInstanceRecord(live);
		const registeredRecord = await radiusPresence.registerPi(live.record);
		this.updateRecord(live, { radiusPiId: registeredRecord.radiusPiId });
		this.setStatus(live, "online");
		return cloneInstance(live.record);
	} catch (error) {
		return await this.failSpawn(live, error);
	}
}
```

见 `packages/orchestrator/src/supervisor.ts:270-298`。

先写 `starting` 记录，再启动子进程。子进程收到 `get_state` 后，supervisor 保存 `sessionId/sessionFile`；可选 Radius 注册完成，状态才变成 `online`。这里没有专门的 ready event，第一次 `get_state` 响应就是启动探针。

外层 `SpawnRequest` 类型还声明了 `provider` 和 `model`，见 `packages/orchestrator/src/ipc/protocol.ts:10-16`，但 `handleIpcRequest()` 只把 `cwd` 和 `label` 传给 supervisor，见 `packages/orchestrator/src/handler.ts:57-68`。在这个基线上，给 spawn 请求填写 provider/model 不会改变子进程配置。

### 进程隔离的实际范围

`RpcProcessInstance` 用 `spawn()` 创建 coding-agent 的 `rpc-entry`，工作目录使用实例 cwd，环境变量原样继承 orchestrator，stdio 全部设为 pipe，见 `packages/orchestrator/src/rpc-process.ts:25-60`。

这会隔离 JavaScript 堆、AgentSession 内存状态和进程生命周期；不会隔离文件系统、网络、凭据或操作系统权限。多个实例若使用同一 cwd，可以同时修改同一文件。orchestrator 没有 sandbox、目录锁、Git worktree 分配或写冲突仲裁。

## 3. 两层 JSONL 协议怎样连接

### 外层 IPC 协议

socket 请求分为管理操作和 RPC 桥接：

| 请求 | 输入 | 决策者 | 输出 |
| --- | --- | --- | --- |
| `spawn` | cwd、label；类型中还有未使用的 provider/model | supervisor | 新实例摘要 |
| `list` | 无 | storage | 所有持久记录 |
| `status` | instanceId | live map 优先，随后 storage | 一个实例摘要 |
| `stop` | instanceId | supervisor | 停止结果 |
| `rpc` | instanceId + `RpcCommand` | 指定 RPC 子进程 | 包装后的一个 `RpcResponse` |
| `rpc_stream` | instanceId | IPC server + supervisor | 长连接 ready、response、event 与 UI 消息 |

请求与响应联合类型在 `packages/orchestrator/src/ipc/protocol.ts:10-128`。编码只做 `JSON.stringify(message) + "\n"`，解析只做 `JSON.parse()` 后类型断言，没有运行时 schema 校验，见同文件 `130-142`。

普通 `rpc` 适合查询或等待一个明确响应。若发送 `prompt`，coding-agent 在 preflight 成功后就返回接受响应，模型输出随后以 AgentSession event 产生；单次 IPC 客户端已经关闭，看不到这些事件。需要完整流时必须使用 `rpc_stream`。

### 内层 coding-agent RPC

子进程运行 `packages/coding-agent/src/rpc-entry.ts:1-13`，最终进入 `runRpcMode()`。其 stdin 接收 `RpcCommand` 或 `extension_ui_response`，stdout 混合输出 `RpcResponse`、AgentSession event 和 `extension_ui_request`。命令集合覆盖 prompt、steer、abort、模型、thinking、压缩、bash 和会话操作，见 `packages/coding-agent/src/modes/rpc/rpc-types.ts:20-72`。

`RpcProcessInstance.send()` 为没有 id 的命令生成关联 id，把 resolver 存进 map，再写入子进程 stdin：

```ts
send(command: RpcCommand): Promise<RpcResponse> {
	if (this.exited) {
		throw new Error(`RPC process is not running. Stderr: ${this.stderrBuffer}`);
	}
	const id = command.id ?? `orchestrator_${++this.nextRequestId}_${randomUUID()}`;
	const fullCommand = { ...command, id };
	return new Promise<RpcResponse>((resolve, reject) => {
		this.pendingRequests.set(id, { resolve, reject });
		this.process.stdin?.write(`${JSON.stringify(fullCommand)}\n`, (error) => {
			if (!error) {
				return;
			}
			this.pendingRequests.delete(id);
			reject(toError(error));
		});
	});
}
```

见 `packages/orchestrator/src/rpc-process.ts:143-159`。stdout 中 `type: "response"` 的消息按 id 唤醒对应 Promise，`extension_ui_request` 交给 UI handler，其余消息一律按 `AgentSessionEvent` 广播，见同文件 `63-128`。

该层没有响应超时。子进程仍存活但不再响应时，pending Promise 会一直等待；只有 stdin 写失败、子进程退出或显式 dispose 才会拒绝。调用方若重复使用相同 command id，后一次会覆盖 map 中前一次的 resolver，这也是当前协议没有防护的边界。

## 4. rpc_stream 把一个 socket 变成双向桥

IPC server 收到 `rpc_stream` 后先确认实例，再撤掉普通 data listener，建立 stream handle。后续每条输入按照 Promise 链串行处理：

```ts
let rpcRequestQueue = Promise.resolve();
socket.on("data", (rpcChunk: Buffer | string) => {
	buffer += rpcChunk.toString();
	for (;;) {
		const rpcNewlineIndex = buffer.indexOf("\n");
		if (rpcNewlineIndex === -1) {
			break;
		}
		const rpcLine = buffer.slice(0, rpcNewlineIndex).trim();
		buffer = buffer.slice(rpcNewlineIndex + 1);
		if (!rpcLine) {
			continue;
		}
		rpcRequestQueue = rpcRequestQueue.then(async () => {
			try {
				await rpcStream.handleRequest(JSON.parse(rpcLine));
			} catch (rpcError: unknown) {
				socket.write(encodeMessage({
					type: "error",
					ok: false,
					error: rpcError instanceof Error ? rpcError.message : String(rpcError),
				}));
			}
		});
	}
});
```

见 `packages/orchestrator/src/ipc/server.ts:95-134`。

这里的串行只覆盖同一条外层 stream 连接。多个 stream 客户端或多个单次 `rpc` socket 仍可同时向同一个子进程发送命令，最终由 coding-agent 的 AgentSession 处理运行中 prompt、steer、follow-up 和 busy 条件。

返回消息共享一条无额外 envelope 的 JSONL 流。客户端依靠 `type` 和 command id 区分 ready、RPC response、session event、UI request 与 error。orchestrator 不保存 event 历史，断线后只能用 `get_state/get_entries/get_messages` 重新查询当前子进程，不能补发已经错过的流事件。

## 5. 事件可以多播，扩展 UI 只有一个接收者

`bindRpcProcess()` 给每个子进程绑定一个事件入口和一个退出入口。session event 遍历 `live.subscribers`，因此多个 `rpc_stream` 客户端都能收到事件。退出事件转给 `handleUnexpectedRpcExit()`，见 `packages/orchestrator/src/supervisor.ts:90-134`。

扩展 UI 不是多播。`LiveInstance` 只有一个 `onUiRequest` 字段；每次打开 stream 都会用新回调覆盖旧回调。关闭连接时，只有仍是当前回调的连接才能清除它，见 `packages/orchestrator/src/supervisor.ts:197-233`。

结果是：

- AgentSession event 可以有多个观察者；
- RPC response 只回到发出该 command 的 stream；
- extension UI request 交给最后连接的 stream；
- 源码没有 UI 所有权租约、抢占通知或 request 到客户端的持久映射。

若最后一个 UI 客户端断开，coding-agent 内部等待的 dialog 要靠自身 timeout 或 abort 结束；orchestrator 的 `close()` 只移除回调和事件订阅，不会自动为未回答的 UI request 生成 cancelled response。

## 6. InstanceRecord 是目录，不是运行快照

### 写入方式

`instances.json` 是一个完整 JSON 数组。`upsertInstance()` 每次都读取全文件、替换或追加一项，再用 `writeFileSync()` 覆盖，见 `packages/orchestrator/src/storage.ts:35-69`。没有临时文件加原子 rename、文件锁、schema 版本或损坏恢复。

`status` 先查 `liveInstances`，查不到再读持久记录；`list` 直接读取持久文件，见 `packages/orchestrator/src/supervisor.ts:235-268`。正常状态变更都会同步写盘，所以两者通常一致，但持久记录只表示最近一次已知状态，不是一次实时 liveness probe。

supervisor 只在这些命令后额外发送 `get_state`：`new_session`、`switch_session`、`fork`、`clone`、`set_session_name` 和 `prompt`，见 `packages/orchestrator/src/supervisor.ts:34-52`。其他 RPC 即使改变 transient 状态，也不会刷新 record。实例的模型、thinking、队列和流式状态本来也不在 record 中。

### session 仍由子进程拥有

orchestrator 只保存 `sessionId/sessionFile` 字符串，不读写会话内容。消息落盘、分支、压缩、重试和工具状态仍由每个 coding-agent `AgentSession` 与 `SessionManager` 管理。多实例之间没有共享 SessionManager；若两个实例显式打开同一个 session 文件，orchestrator 也没有冲突检测。

## 7. 三种退出路径留下不同结果

### 正常 stop

`stopInstance()` 先持久化 `stopping`，解绑事件，注销 Radius Pi，再向子进程发送 `SIGTERM` 并等待 exit。finally 中把内存 record 改为 `stopped`，随后从 live map 和 `instances.json` 删除，见 `packages/orchestrator/src/supervisor.ts:300-319`。

调用者会收到一个成功 stop 响应，但此实例不会作为 stopped 历史继续出现在 list 中。`RpcProcessInstance.dispose()` 没有超时和升级到强制终止；子进程忽略 `SIGTERM` 时，stop 和 orchestrator shutdown 都可能一直等待，见 `packages/orchestrator/src/rpc-process.ts:186-196`。

### 子进程意外退出

child 的 `error` 或 `exit` 会把全部 pending RPC 以包含 stderr 的 Error 拒绝，再通知 supervisor。supervisor 将状态持久化为 `error`、解绑 listener、尝试注销 Radius、删除 live map 项，但保留 `instances.json` 中的 error 记录，见 `packages/orchestrator/src/rpc-process.ts:86-98`、`130-141` 和 `packages/orchestrator/src/supervisor.ts:115-134`。

没有自动重启、重试次数、退避或 session 续接。此后 `status` 还能读到 error 摘要，`rpc/stop/rpc_stream` 因没有 live process 而返回 unknown instance。错误记录与可操作实例是两回事。

### Orchestrator 自身重启

`serve()` 先启动 socket server，再调用 `recoverAfterRestart()`，然后才启动可选 Radius heartbeat，见 `packages/orchestrator/src/serve.ts:9-35`。所谓 recovery 的实现很短：

```ts
async recoverAfterRestart(): Promise<void> {
	const recoveredAt = new Date().toISOString();
	const instances = loadInstances().map((instance) => ({
		...instance,
		status: instance.status === "online" || instance.status === "starting" ? "stopped" : instance.status,
		lastSeenAt: recoveredAt,
	}));
	for (const instance of instances) {
		await radiusPresence.disconnectPi(instance);
	}
	saveInstances(instances);
}
```

见 `packages/orchestrator/src/supervisor.ts:244-255`。

它不会查找旧 pid、重新连接子进程、重新启动实例或恢复未完成 RPC。旧 `online/starting` 只被改成 `stopped`；旧 `stopping` 和 `error` 保持原状态。每条记录都尝试注销 Radius presence，失败会使 serve 初始化失败。记录中的 `radiusPiId` 没有在这个过程里清空。

所以重启恢复的含义是清理陈旧在线声明，不是恢复 Agent 执行。

## 8. Radius 只发布 presence

Radius 集成是可选的。凭据来自 coding-agent 保存的 Radius OAuth credential 或 `RADIUS_API_KEY`。orchestrator 注册 machine，再为每个 live instance 注册 Pi，周期性发送 heartbeat，见 `packages/orchestrator/src/radius.ts:107-209`。

注册 payload 明确声明当前能力：

```ts
const registered = await post<RegisterPiResponse>("pis/register", {
	machineId: machine.id,
	label: instance.label,
	cwd: instance.cwd,
	hostname: hostname(),
	pid: process.pid,
	transport: "local-rpc",
	capabilities: { rpc: true, relay: false, iroh: false },
	sessionId: instance.sessionId,
});
```

见 `packages/orchestrator/src/radius.ts:189-209`。Radius 获得的是机器和实例在线信息；实际 RPC 仍走本机 socket。`relay:false` 和 `iroh:false` 表明当前代码没有通过 Radius 远程中继命令。

heartbeat 遇到普通网络错误使用带 jitter 的指数退避；连续三次 404 后重新注册 machine 或 Pi，见 `packages/orchestrator/src/radius.ts:303-436`。这套 retry 只维护 presence，不会重试 Agent prompt、工具调用或子进程启动。

## 9. 失败从哪一层返回

| 失败位置 | 可观察结果 | 保留下来的状态 |
| --- | --- | --- |
| socket 不存在或提前关闭 | CLI 的 IPC Promise reject | instances 文件不变 |
| 请求 JSON 无法解析 | 外层 `error` response 后关闭连接 | 实例不变 |
| unknown instance | `ok:false` 的外层 error | 旧持久记录可能仍存在 |
| 子进程 stdin 写失败 | 对应 pending RPC reject | supervisor 随后可能收到 exit |
| 子进程正常存活但不响应 | Promise 无期限等待 | pending map 与实例保持在线 |
| child stdout 不是合法 JSON | `handleLine()` 抛错；serve 的全局异常路径可能关闭整个 orchestrator | 是否完成清理由异常时机决定 |
| spawn 中任一步失败 | record 依次写为 `error`、`stopped`，资源清理，live map 删除 | stopped 记录保留 |
| 正常 stop | 子进程和 Radius 清理，记录删除 | coding-agent 自己已落盘的 session 仍在 |
| orchestrator 崩溃后重启 | 旧 online/starting 改为 stopped | 不恢复子进程和进行中的命令 |

协议对象只经过 TypeScript 类型约束，没有运行时验证。来自 socket 或 child stdout 的合法 JSON 仍可能形状错误。错误消息还可能包含子进程累计 stderr；如果 IPC socket 可被不受信任的本机用户访问，这会扩大诊断信息暴露面。源码没有为 socket 设置显式访问控制。

## 10. 测试能证明到哪里

`packages/orchestrator` 没有 test 目录，`package.json` 也没有 test script。supervisor、IPC socket、storage、Radius heartbeat、异常退出和 restart cleanup 在该包内没有自动化测试。

下游 coding-agent 有 RPC 测试，但覆盖的是内层协议：

- `packages/coding-agent/test/rpc-jsonl.test.ts:5-65` 固定 LF/CRLF、Unicode 分隔符和末行行为；
- `packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:187-288` 固定 prompt preflight 只返回一次成功或失败，以及运行中排队 prompt 的接受响应；
- `packages/coding-agent/test/rpc-client-process-exit.test.ts:23-38` 固定 coding-agent `RpcClient` 在自己的 child 退出后拒绝请求。

这些测试没有经过 orchestrator 的 socket server、`RpcProcessInstance` 或 supervisor。它们能说明被桥接协议的部分行为，不能证明多实例生命周期已经可靠。当前最明显的测试缺口包括：重复 command id、多个 rpc_stream 客户端、UI owner 断线、spawn 失败清理、子进程挂起、SIGTERM 超时、instances.json 损坏以及 Radius 重注册。

## 11. 已解决的是实例寻址，尚未提供协作语义

源码已经解决四个具体问题：

- 一个常驻本机服务可以创建和停止多个 headless Pi 进程；
- 每个实例有独立 cwd、进程内状态和 session 标识；
- 客户端可用 instanceId 定向发送 coding-agent RPC；
- 长连接可以观察事件，并转发扩展 UI 的请求与回答。

下面这些能力没有源码支撑：

- supervisor agent 根据目标拆分任务；
- 子 Agent 之间发送消息或共享上下文；
- 依赖图、并发上限、预算和完成条件；
- 结果汇总、冲突裁决或质量复核；
- 每个实例不同的权限、sandbox 或 worktree；
- 子进程崩溃后的自动重启与幂等重放；
- 跨机器 RPC relay。

把 `spawn` 后的多个实例称作“多 Agent”在进程数量上没有错，但它只提供运行容器与通信管道。谁分配工作、发什么上下文、怎样判断完成、失败后是否重派，都要由外部客户端实现。

## 12. 设计取舍

每个实例使用独立进程，故障和内存状态不会直接串到其他实例；同时复用了 coding-agent 已有的 RPC 模式，orchestrator 不必再次实现 prompt、工具、压缩和会话操作。代价是每个实例都要承担完整 coding-agent 启动成本，并继承相同环境与权限。

JSONL 适合把 response、event 和 UI request 放在同一条流里，关联 id 也允许多个请求在途。协议缺少 runtime schema、timeout 和流事件游标后，错误 JSON、无响应 child 和断线补偿只能由调用方处理。

`instances.json` 让 list/status 在 child 消失后仍有线索，但它不是进程注册表或执行日志。重启时选择把旧实例降为 stopped，是保守且可预测的清理策略；它也说明当前 orchestrator 没有 durability。若后续要承载真正的任务编排，首先需要稳定实例生命周期和恢复契约，再在其上定义任务、消息、权限与结果，而不是把这些语义塞进现有的 spawn/list/rpc 接口。
