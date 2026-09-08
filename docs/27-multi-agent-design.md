# 第 27 讲：从 Pi 现有边界推演多 Agent 协调层

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 已有的多实例能力解决了“怎样启动并寻址多个 Agent 进程”，还没有回答“谁负责拆任务、怎样交付上下文、何时算完成、失败后如何处理”。缺失的主体是任务协调层。它应当拥有任务状态、依赖关系、工作区租约和结果验收；Agent 只执行被分配的一次尝试。

本文把源码事实和设计方案分开：

- 标有“源码事实”的内容可以在指定 commit 中验证；
- 标有“设计推演”的类型、状态机和流程是基于现有接口提出的方案，Pi 当前没有这些实现。

## 1. 现有四层各自拥有哪种状态

> 源码事实

| 层 | 已有职责 | 自己拥有的状态 | 没有的状态 |
| --- | --- | --- | --- |
| `Agent` / `agentLoop` | 单实例模型调用、工具循环、队列和中止 | messages、streaming、pending tool calls、steer/follow-up | 跨 Agent 任务、依赖、工作区所有权 |
| `Session` / `SessionManager` | 保存一条 Agent 的历史树并构建上下文 | entry、parentId、leaf、model/thinking 变化 | 任务状态、执行租约、汇总结论 |
| coding-agent RPC | 远程控制一个 `AgentSession` | command response、session event、扩展 UI 请求 | durable event、任务完成协议、跨实例事务 |
| `orchestrator` | 启停 RPC 子进程并按 instanceId 转发 | 进程状态、cwd、sessionId、订阅者 | 目标拆分、依赖图、预算、结果验收 |

`InstanceRecord` 足以描述进程目录，却无法表示任务：

```ts
export interface InstanceRecord {
	id: string;
	status: InstanceStatus;
	cwd: string;
	createdAt: string;
	lastSeenAt?: string;
	label?: string;
	sessionId?: string;
	sessionFile?: string;
	radiusPiId?: string;
}
```

见 `packages/orchestrator/src/types.ts:15-25`。这里没有 objective、parent task、dependencies、attempt、budget、result 或 acceptance。把 `online` 解释成“任务正在运行”，或把进程 `error` 直接解释成“任务失败”，都会混淆两套生命周期。

## 2. 协调层的概念模型

> 设计推演

最小可用设计需要任务实体和一组协调组件：

```text
Goal
  └── Task DAG
        ├── Task A ── Attempt A1 ── Worker instance A
        ├── Task B ── Attempt B1 ── Worker instance B
        └── Task C ── Attempt C1 ── Worker instance C

TaskStore       保存任务、依赖、attempt 与状态转换
Scheduler       判断哪些任务可运行，把 attempt 租给 worker
WorkerAdapter   用 RPC 或 AgentHarness 驱动一个 Pi 实例
WorkspaceManager 分配只读目录、worktree 或写锁
ArtifactStore   保存任务包、结构化结果、补丁、日志与验收记录
```

Goal 是用户目标；Task 是可调度工作；Attempt 是 Task 的一次执行；Worker instance 是承载 Attempt 的进程。Task 失败后可以创建新 Attempt，原 Attempt 仍保留，审计时才能看出失败发生在模型、工具、进程还是验收阶段。

这层不应塞进 `AgentSession`。`AgentSession` 协调一个会话内部的模型、工具、压缩和重试；跨实例依赖与工作区冲突需要看到所有任务，只能由更外层的对象决策。

## 3. 任务分派需要独立状态机

### 解决的问题

进程在线不表示任务可执行。依赖尚未完成、写工作区被占用、预算不足或结果正在验收时，worker 都可能在线，但 task 不能进入下一步。

> 设计推演

```ts
type TaskState =
	| "created"
	| "ready"
	| "running"
	| "reviewing"
	| "succeeded"
	| "failed"
	| "cancel_requested"
	| "cancelled"
	| "blocked";

type AttemptState =
	| "leased"
	| "running"
	| "reconciling"
	| "reviewing"
	| "succeeded"
	| "failed"
	| "cancel_requested"
	| "cancelled";

interface TaskRecord {
	id: string;
	goalId: string;
	parentTaskId?: string;
	dependsOn: string[];
	objective: string;
	inputArtifactIds: string[];
	capabilityProfileId: string;
	workspaceRequest: WorkspaceRequest;
	state: TaskState;
	activeAttemptId?: string;
	acceptedAttemptId?: string;
}

interface AttemptRecord {
	id: string;
	taskId: string;
	state: AttemptState;
	workerInstanceId: string;
	leaseToken: string;
	resultArtifactId?: string;
}
```

建议的状态流是：

```text
Task:    created → ready → running → reviewing → succeeded
                         │            │
                         │            └──退回并重试→ ready
                         └──不可重试──────────→ failed / blocked

Attempt: leased → running → reviewing → succeeded
                      └──失联→ reconciling → 恢复观察 / failed

Task 或 Attempt 的非终态 → cancel_requested → cancelled
```

状态转换由协调层写入 TaskStore。worker 只能报告事件和候选结果，不能自行把任务改成 `succeeded`。验收者可以是确定性检查、专门的 reviewer Agent 或用户；谁做决定必须记录在 acceptance 事件中。

失败路径包括重复调度、依赖循环、租约过期和晚到结果。每次下发都带 `taskId + attemptId + leaseToken`。TaskStore 以 compare-and-set 接受状态转换；旧 attempt 或旧 token 的结果保存为诊断材料，不改变当前任务。

Task 与 Attempt 分开后，一次执行失败不必抹掉任务，也不会覆盖旧的诊断信息。这套状态比“收到 `agent_end` 就完成”繁琐，却能区分执行结束、结果可用和结果已被接受。

## 4. RPC 负责控制，不负责判定任务完成

### 源码入口与现有流程

> 源码事实

RPC 的 `prompt` 响应只证明 preflight 成功或请求已进入队列：

```ts
case "prompt": {
	// Start prompt handling immediately, but emit the authoritative response only after
	// prompt preflight succeeds. Queued and immediately handled prompts also count as success.
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

见 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:393-415`。测试 `packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:193-287` 分别固定了 preflight 失败、成功和 streaming 中排队三种响应。它没有把 command response 当作最终文本。

RPC 事件也不是持久队列。`RpcProcessInstance` 收到 child stdout 后，response 按 id 唤醒 Promise，其余对象直接广播给当前监听器，见 `packages/orchestrator/src/rpc-process.ts:101-127`。断线期间的事件不会补发。

### 分派调用链

> 设计推演

```text
Scheduler 选中 ready task
  → WorkspaceManager 取得 workspace lease
  → WorkerAdapter 取得或创建 Pi instance
  → 建立 rpc_stream，再发送带 attemptId 的 prompt
  → prompt response 成功：attempt 改为 running
  → 持续接收 session event，并写入协调层事件日志
  → agent_end：只表示本次 Agent loop settled 的候选信号
  → 查询 get_state / get_messages / get_entries，收集最终证据
  → worker 调用 submit_result，或 adapter 从约定产物构造结果
  → attempt 进入 reviewing
  → 验收通过后 task 才进入 succeeded
```

先订阅再发 prompt 可以减少丢失首批事件的窗口，但不能获得 exactly-once 语义。协调层仍需用 attemptId 去重，并在重连后用查询命令重建当前快照。

如果连接断开，任务暂时进入 `reconciling` 子状态或保留 `running` 并标记 transport unknown。直接重发 prompt 有副作用重复风险；应先检查进程、session 和已登记 artifact。

## 5. 上下文隔离以任务包为边界

### 源码中可复用的能力

> 源码事实

新 Harness 的 `SessionRepo` 已把会话创建、打开、列出、删除和 fork 抽成接口：

```ts
export interface SessionRepo<
	TMetadata extends SessionMetadata = SessionMetadata,
	TCreateOptions extends SessionCreateOptions = SessionCreateOptions,
	TListOptions = void,
> {
	create(options: TCreateOptions): Promise<Session<TMetadata>>;
	open(metadata: TMetadata): Promise<Session<TMetadata>>;
	list(options?: TListOptions): Promise<TMetadata[]>;
	delete(metadata: TMetadata): Promise<void>;
	fork(source: TMetadata, options: SessionForkOptions & TCreateOptions): Promise<Session<TMetadata>>;
}
```

见 `packages/agent/src/harness/types.ts:469-479`。`JsonlSessionRepo.fork()` 复制选定路径上的 entry，写入新的 session 文件，并用 `parentSessionPath` 记录来源，见 `packages/agent/src/harness/session/jsonl-repo.ts:134-160`。

每个 Harness turn 从自己的 Session 构建上下文快照：

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

见 `packages/agent/src/harness/agent-harness.ts:314-346`。这说明独立 Session 可以形成独立模型上下文。它并未提供任务级信息流控制。

### 任务包的内容

> 设计推演

协调层不应默认把父 Agent 的完整 session fork 给每个 worker。完整 fork 会复制无关对话、工具输出和秘密，还会消耗各 worker 的上下文预算。任务包应由明确字段组成：

```ts
interface TaskEnvelope {
	taskId: string;
	attemptId: string;
	objective: string;
	acceptanceCriteria: string[];
	constraints: string[];
	inputArtifacts: ArtifactRef[];
	allowedPaths: string[];
	forbiddenPaths: string[];
	upstreamResults: ResultRef[];
	resultSchemaVersion: number;
}
```

新 worker session 只接收 TaskEnvelope 和必要 artifact。需要保留共同推理背景时，可以从父 session 的一个稳定 leaf fork；协调层仍要生成“实际复制了哪些 entry”的 provenance。共享背景和共享可写 session 是两件事，多个 worker 不应同时 append 同一个 JSONL 文件。

当前 `JsonlSessionStorage` 把 entry append 到文件后更新各实例自己的内存索引，见 `packages/agent/src/harness/session/jsonl-storage.ts:180-203,271-280`。两个 storage 对象并发写同一文件时，彼此不会更新对方的 `byId/currentLeafId`，源码也没有跨进程锁。单 session 单 writer 应成为协调层不变量。

## 6. 工具调用可承载委派命令，但任务不能只存在于 tool result

### 已有工具边界

> 源码事实

`agentLoop` 在执行前查工具、准备并校验参数，再调用 `beforeToolCall`。阻断会生成错误 tool result；真正执行时把同一个 AbortSignal 传给工具，见 `packages/agent/src/agent-loop.ts:602-705`。Harness 又把 `tool_call`、`tool_result` 暴露成可修改的 hook，见 `packages/agent/src/harness/agent-harness.ts:399-447`。

所以应用层可以注册 `delegate_task`、`await_task` 和 `submit_result` 一类工具。不过这些名字在指定 commit 中不存在。

### 建议协议

> 设计推演

`delegate_task` 只负责创建任务并返回稳定 taskId，不在一次工具调用里阻塞等待整个子任务：

```ts
delegate_task({
	objective,
	acceptanceCriteria,
	inputs,
	capabilityProfile,
	workspaceRequest,
	dependsOn
}) -> { taskId, state: "created" | "ready" }

await_tasks({ taskIds, returnWhen: "all" | "first_terminal" })
  -> { tasks: TaskSnapshot[] }

submit_result({ taskId, attemptId, result })
  -> { acceptedForReview: boolean, reason?: string }
```

任务先写 TaskStore，再向父 Agent 返回 taskId。这样父 turn、RPC 连接或 supervisor 进程退出后，任务仍有独立身份。`await_tasks` 可以被 abort；取消等待不自动取消子任务，除非调用者明确请求 cascade cancellation。

worker 不能通过自由文本声称“已经完成”来修改状态。`submit_result` 要校验 attempt、lease、schema 和 artifact 引用。它只把 attempt 推到 reviewing，最终 acceptance 仍由协调层处理。

## 7. 工作区隔离决定多实例能否安全写代码

### 现有边界

> 源码事实

orchestrator 创建 child 时只设置 cwd，并原样继承环境变量：

```ts
this.process = spawn(rpcCommand.command, rpcCommand.args, {
	cwd: options.cwd,
	env: process.env,
	stdio: ["pipe", "pipe", "pipe"],
});
```

见 `packages/orchestrator/src/rpc-process.ts:37-43`。多个实例指向同一 cwd 时可以同时修改同一文件。

Harness 的 `ExecutionEnv` 是可替换接口，覆盖文件系统和 shell，见 `packages/agent/src/harness/types.ts:252-332`。默认 `NodeExecutionEnv` 允许绝对路径，并把相对路径解析到 cwd，见 `packages/agent/src/harness/env/nodejs.ts:47-49,394-455`。抽象接口提供替换点，默认实现不构成 sandbox。

### 工作区分配规则

> 设计推演

| 任务类型 | 工作区策略 | 并发规则 | 交付方式 |
| --- | --- | --- | --- |
| 纯分析 | 共享只读 checkout | 可并发 | 结构化报告与引用 |
| 独立代码改动 | 每 attempt 独立 Git worktree | 可并发 | commit 或 patch artifact |
| 必须写共享状态 | 路径或资源独占租约 | 串行 | 写集清单与验收记录 |
| 外部系统操作 | 独立 credential/policy profile | 按目标系统限流 | 操作回执与幂等键 |

WorkspaceManager 在 task 进入 leased 前完成分配。租约包含 canonical path、base commit、允许写入的路径、到期时间和 fencing token。worker 每次提交结果都附 token；过期 attempt 的写入不能自动合并。

仅在提示词里写“不要修改其他文件”不够。读写边界应由只读挂载、worktree、容器用户或代理文件系统执行。共享 checkout 上的 Git status 只能发现冲突，不能阻止冲突。

## 8. 权限是每个 attempt 的能力配置

### 现有 project trust 不等于任务授权

> 源码事实

coding-agent 的 project trust 决定是否加载项目设置、资源、包和扩展。无 UI 且没有其他决定时返回 false，见 `packages/coding-agent/src/core/project-trust.ts:46-95`。它没有给每个 task 分配不同的文件、网络和工具权限。

Harness 构造器允许应用传入 `env`、`tools`、`activeToolNames`、model 和 resources，见 `packages/agent/src/harness/types.ts:800-836`。这适合表达能力配置，但安全性取决于应用传入的实现。若同一进程中的扩展仍可直接访问 Node.js，工具 allowlist 无法限制扩展自身的文件或网络调用。

### CapabilityProfile

> 设计推演

```ts
interface CapabilityProfile {
	id: string;
	toolNames: string[];
	readPaths: string[];
	writePaths: string[];
	commandPolicy: CommandPolicy;
	networkPolicy: NetworkPolicy;
	credentialRefs: string[];
	modelPolicy: ModelPolicy;
	maxWallTimeMs: number;
	maxAttempts: number;
}
```

profile 由协调层解析成具体执行边界：

- WorkerAdapter 只注册允许的工具；
- ExecutionEnv 或外部 sandbox 执行路径、进程与网络限制；
- credential broker 只向指定 attempt 发放短期凭据；
- scheduler 执行模型、token、费用和墙钟预算；
- policy 需要人工批准时，task 进入 blocked，记录请求与决定。

worker 的系统提示可以描述限制，真正的决策者仍是 policy enforcement。若限制只写在 prompt 里，它属于行为引导，不属于权限控制。

## 9. 结果合并需要证据对象和唯一决策者

### 问题

多个 Agent 即使各自回答正确，也可能修改同一段代码、基于不同 commit、重复执行测试，或对完成条件使用不同解释。把多段 assistant text 拼接起来无法处理这些冲突。

> 设计推演

```ts
interface TaskResult {
	schemaVersion: number;
	taskId: string;
	attemptId: string;
	baseRevision?: string;
	summary: string;
	claims: EvidenceBackedClaim[];
	artifacts: ArtifactRef[];
	changedPaths: string[];
	verification: VerificationRecord[];
	remainingRisks: string[];
	status: "candidate" | "blocked" | "failed";
}
```

合并流程按 artifact 类型区分：

```text
事实报告
  → 核对引用与样本范围
  → 去重或保留分歧
  → reviewer 生成带来源的汇总

代码改动
  → 核对 baseRevision 与 changedPaths
  → 在集成 worktree 应用 commit/patch
  → 运行目标测试和回归测试
  → 验收通过后记录 acceptedAttemptId
```

代码合并由 WorkspaceManager 或专门的 integrator 执行。普通 worker 不应直接写目标分支，也不应由“最后返回的 worker”获胜。两个候选都通过局部测试时，验收仍要检查组合后的结果。

结果保留 `taskId/attemptId/baseRevision`，可以判断晚到结果属于哪次尝试。自由文本保留作解释，状态转换依赖结构化字段和独立验证。

## 10. 取消分成等待、运行和进程三层

### Pi 已有的取消语义

> 源码事实

`Agent.abort()` 只触发当前 run 的 AbortController；`waitForIdle()` 等到运行及所有被 await 的 listener 收尾，见 `packages/agent/src/agent.ts:304-320,469-518`。工具是否及时停止取决于它是否遵守收到的 AbortSignal。

Harness 的 `abort()` 会清空 steer/follow-up，保留 next-turn 队列，触发 signal，再等待 idle，见 `packages/agent/src/harness/agent-harness.ts:970-1001`。测试 `packages/agent/test/harness/agent-harness.test.ts:164-210` 固定了这组差异。

RPC `abort` 等待 `session.abort()` 后才返回成功，见 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:427-430`。orchestrator 的 `stop` 则结束整个 child，不等同于一次 task abort。

### 取消流程

> 设计推演

1. TaskStore 先记录 `cancel_requested`，scheduler 不再创建新 attempt。
2. WorkerAdapter 向当前实例发 RPC `abort`，等待 settle，并设置截止时间。
3. 超时后终止 worker 进程；写任务的 workspace 标记为 `needs_inspection`。
4. 收集已落盘 session、artifact 和外部操作回执。
5. fencing token 失效，晚到结果不能覆盖 cancelled 状态。
6. 父任务选择是否级联取消子任务；共享子任务只有在没有其他消费者时才停止。

`cancel_requested` 与 `cancelled` 不能合成一次写入。工具可能已经完成外部副作用，进程也可能尚未退出。终态表示协调层完成了收尾与状态核对，不表示现实世界的副作用被回滚。

## 11. 故障恢复以 attempt 对账为核心

### 当前恢复能力

> 源码事实

子进程意外退出时，orchestrator 把实例记为 `error`、移除 live handle，不自动重启，见 `packages/orchestrator/src/supervisor.ts:115-134`。orchestrator 自身重启后只把旧 `online/starting` 改为 `stopped` 并清理 presence，见同文件 `244-255`。

Session 能保存消息和工具结果，Harness 在 `turn_end` 刷新 pending session writes 并发出 `save_point`，见 `packages/agent/src/harness/agent-harness.ts:488-512`。save point 是会话已写入的边界，不是 task checkpoint；它不知道外部副作用、验收进度或 workspace lease。

### 恢复协议

> 设计推演

协调层采用 append-only task event log，并定期生成快照。重启后的 reconciler 逐项对账：

```text
读取非终态 task / attempt
  → 检查 lease 是否过期
  → 查询 instance 与 session 是否存在
  → 检查 result/artifact 是否已经提交
  → 检查 workspace base、dirty state 和外部幂等回执
  → 能确认继续：重新连接并恢复观察
  → 不能确认继续：关闭旧 attempt，创建新 attempt 或转 blocked
```

重试策略按副作用分类：

| 最近阶段 | 自动重试条件 | 不能自动重试的情况 |
| --- | --- | --- |
| 只读分析 | 输入 artifact 与 base revision 未变 | 外部数据源已变化且未保存快照 |
| 模型生成、尚未执行工具 | 可新建 attempt | 无法确认是否已经发出写操作 |
| 独立 worktree 写入 | worktree 可检查且未集成 | 已向共享分支或外部系统提交 |
| 外部系统操作 | 操作有幂等键并可查询结果 | 非幂等调用结果未知 |

恢复不应重放旧 RPC command。prompt 和工具调用没有通用 exactly-once 保证。安全做法是保存输入快照，创建新的 attempt，并让验收层识别重复产物。

## 12. Agent 间通信走任务存储，不做进程直连

> 设计推演

最小实现不需要 worker A 直接持有 worker B 的 RPC 地址。通信统一写成 task event 或 artifact：

```text
父 Agent 创建子任务
  → TaskStore 保存依赖与任务包
  → 子 Agent 读取任务包并提交结果
  → 父 Agent 通过 await_tasks 或 follow-up 收到结果引用
```

这种结构让权限、取消、重试和审计都经过同一决策点。直连消息虽然延迟更低，却会绕过任务状态，重启后也难以判断消息是否送达、是否处理。

需要增量协作时，可以增加带序号的 `TaskMessage`，字段至少包含 senderAttemptId、recipientTaskId、kind、artifact refs 和 idempotency key。消息是持久事件，worker 在线只是消费条件。

Pi 的 steer/follow-up 仍有价值：协调层可以把已持久化的 TaskMessage 投递到正在运行的 worker。`agentLoop` 在当前 assistant turn 结束后取 steering，在本应停止时取 follow-up，见 `packages/agent/src/agent-loop.ts:155-274`。队列是单 Agent 内存机制，不能替代 TaskMessage 存储。

## 13. 放在哪一层实现

> 设计推演

建议新增独立应用层，例如 `packages/multi-agent`，通过接口依赖现有包：

```text
multi-agent coordinator
  ├── TaskStore / ArtifactStore / PolicyEngine
  ├── Scheduler / Reconciler / WorkspaceManager
  └── WorkerAdapter
          ├── RpcWorkerAdapter ──> orchestrator ──> coding-agent RPC
          └── HarnessWorkerAdapter ──> AgentHarness
```

第一阶段宜使用 `RpcWorkerAdapter`。coding-agent 主链已经支持完整工具、扩展、资源与 session，能复用当前行为。Adapter 要补 task event 持久化、完成判定和超时，不能假设 orchestrator 已经提供。

`HarnessWorkerAdapter` 适合后续收敛。Harness 的 SessionRepo、ExecutionEnv、turn snapshot、hook 和 save point 边界更清晰，也更容易做内存级测试；指定 commit 中 coding-agent 主链尚未迁移到 Harness。现在直接替换会同时承担多 Agent 与运行时迁移两类风险。

orchestrator 继续负责进程控制。可以扩展 spawn profile、健康检查和 restart primitive，但不让它拥有 Task DAG。这样单机进程实现将来替换成容器或远端 worker 时，任务语义不会跟着重写。

## 14. 最小实现顺序

> 设计推演

### 阶段一：单机、只读、显式任务

实现 TaskStore、Scheduler、RpcWorkerAdapter 和结构化 TaskResult。所有 worker 共享只读 checkout，不允许代码写入；并发上限固定。目标是验证任务状态、上下文隔离、事件重连和结果验收。

### 阶段二：独立 worktree 与取消

加入 WorkspaceManager、lease/fencing token、代码 artifact、集成验收和 cancel timeout。每个写 attempt 使用独立 worktree，禁止直接写目标分支。

### 阶段三：恢复与策略

加入 reconciler、幂等键、短期 credential、外部 sandbox、预算和人工 approval。此时才允许带外部副作用的任务。

每个阶段都能独立验收。先实现“Agent 自由互聊”会掩盖任务状态缺失，也很难补上可靠恢复。

## 15. 测试要固定跨层不变量

> 设计推演

| 测试 | 构造的输入 | 必须观察的结果 |
| --- | --- | --- |
| 依赖调度 | B 依赖 A，两个 worker 空闲 | A 未验收前 B 不得 leased |
| 单次租约 | 两个 scheduler 同时领取一个 ready task | 只有一个 attempt 获得有效 token |
| 上下文隔离 | 两个任务包含不同秘密标记 | provider payload 不出现对方标记 |
| session 单 writer | 两个 worker 请求同一 session | 第二个被拒绝或获得独立 fork |
| 写冲突 | 两个 task 修改同一文件 | 在独立 worktree 运行，集成时显式冲突 |
| 取消 | 工具运行中取消并让工具延迟退出 | 先 cancel_requested，settle 后才 cancelled |
| 进程崩溃 | prompt 接受后 kill child | attempt 进入对账，不直接标 task failed |
| 协调层重启 | running task 已提交 artifact、尚未验收 | reconciler 找到 artifact，避免重跑 |
| 晚到结果 | A1 租约过期后 A2 成功，随后 A1 返回 | A1 结果保留但不能覆盖 acceptedAttemptId |
| 权限 | 只读 profile 请求 write/network | enforcement 拒绝，task result 记录失败证据 |

现有测试只能固定可复用原语：

- `packages/agent/test/harness/repo.test.ts:8-91` 验证 SessionRepo 创建、打开、fork 和 metadata；
- `packages/agent/test/harness/agent-harness.test.ts:89-210` 验证队列 drain 与 abort 差异；
- `packages/agent/test/harness/agent-harness.test.ts:293-419` 验证 save point 刷新、写入顺序和 settlement；
- `packages/coding-agent/test/agent-session-concurrent.test.ts:130-181` 验证单 session 忙时拒绝第二个普通 prompt，但允许 steer/follow-up；
- `packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:187-287` 验证 RPC prompt response 的接受语义。

`packages/orchestrator` 在指定 commit 中没有包内测试。这些已有测试都没有构造 Task DAG、worker lease、workspace 冲突、结果验收或协调层恢复，不能作为多 Agent 可靠性的证据。

本课尝试运行 `packages/agent/test/harness/agent-harness.test.ts` 和 `packages/agent/test/harness/repo.test.ts`。测试进程在加载测试文件前退出，原因是工作区没有安装根目录的 Vitest 模块。这是环境问题；没有产生测试断言结果，不能归为文档错误或上游源码缺陷。

## 16. 一次完整状态流

> 设计推演

目标：“分析两个独立模块并提交一份经过核对的修改”。

```text
1. coordinator 创建 T1(模块 A 分析)、T2(模块 B 分析)、T3(集成修改)
2. T3.dependsOn = [T1, T2]
3. scheduler 为 T1/T2 分配只读 workspace 与不同 worker
4. 两个 worker 各自获得精简 TaskEnvelope 和独立 session
5. prompt response 成功，A1/B1 进入 running
6. worker 提交引用源码锚点的 TaskResult，任务进入 reviewing
7. reviewer 核对证据；T1/T2 succeeded
8. T3 变为 ready，WorkspaceManager 创建独立 worktree
9. integrator 只读取 T1/T2 的 accepted result，不读取两个完整 session
10. T3 提交 commit artifact 和测试记录
11. 验收在集成 worktree 运行，成功后接受 attempt
12. 释放 lease，保存 goal 汇总与 provenance
```

若步骤 5 后某个 worker 崩溃，只有对应 attempt 进入 reconciliation；另一个任务继续。若步骤 10 的测试失败，T3 attempt 失败，T1/T2 的已验收事实不回滚。任务层的价值就在这种局部故障隔离。

## 17. 设计取舍

中心 TaskStore 带来明确状态和恢复点，也形成单一协调依赖。它需要事务或条件写，不能沿用 orchestrator 当前的整文件覆盖式 `instances.json`。

独立 session 降低上下文串扰，却要求用 artifact 明确传递结果。完整 session fork 省去摘要工作，但会复制无关信息并放大 token 与秘密暴露。默认使用任务包，fork 只用于确实需要共同推理历史的任务。

worktree 比共享目录占用更多磁盘，换来可检查的写边界和确定的集成入口。纯分析任务可共享只读 checkout，不必为统一形式支付创建成本。

中央结果验收会增加延迟。没有验收时，多 Agent 只提高候选产出速度，无法保证组合结果正确。验收规则应按任务类型选择确定性检查、reviewer 或人工决定，不能全部交给生成结果的同一个 worker。

## 18. 常见边界问题

### 多个 Pi 进程是否已经构成多 Agent 系统？

它们构成了多个 Agent 执行单元。任务分派、信息流、权限、依赖和结果验收仍需外层实现，因此还不是完整的协作运行时。

### 能否让父 Agent 通过一个工具直接等待所有子 Agent？

可以实现，但长期等待会占住父 Agent 的 tool call 和运行生命周期。任务先持久化，工具返回 taskId，再通过 `await_tasks` 或后续消息取得结果，取消和重连更清楚。

### session fork 是否等同于创建子 Agent？

fork 复制一段会话历史，解决上下文起点问题。它不会创建进程、分配工具、限制权限、建立任务依赖或定义完成条件。

### `agent_end` 能否作为 task success？

不能。它表示一次 Agent loop 不再发出事件；结果可能是 provider error、aborted、错误 tool result 或不符合验收条件的普通文本。task success 由结果 schema 和 acceptance 决定。

### worker 崩溃后能否自动重放 prompt？

只读、输入已冻结且没有外部副作用时，可以创建新 attempt。普通工具调用和外部系统操作没有 exactly-once 保证，状态不明时必须先对账，必要时转人工处理。

### 多 Agent 的安全性是否只需给每个 worker 不同系统提示？

系统提示只能影响模型选择。文件、shell、网络、凭据和目标分支权限需要由 ExecutionEnv、sandbox、credential broker 与 WorkspaceManager 执行。

## 19. 源码锚点

- 单 Agent 循环与队列：`packages/agent/src/agent-loop.ts:155-310`、`packages/agent/src/agent.ts:169-320`
- 工具执行与中止信号：`packages/agent/src/agent-loop.ts:491-705`
- Harness 状态、快照与 save point：`packages/agent/src/harness/agent-harness.ts:157-205,294-447,488-620,970-1001`
- Harness session 与 repo 抽象：`packages/agent/src/harness/types.ts:422-500`、`packages/agent/src/harness/session/session.ts:137-337`
- JSONL repo 与 storage：`packages/agent/src/harness/session/jsonl-repo.ts:38-160`、`packages/agent/src/harness/session/jsonl-storage.ts:180-280`
- coding-agent RPC 命令与状态：`packages/coding-agent/src/modes/rpc/rpc-types.ts:20-107`
- RPC prompt、abort 与输入处理：`packages/coding-agent/src/modes/rpc/rpc-mode.ts:385-445,702-770`
- orchestrator 实例与进程状态：`packages/orchestrator/src/types.ts:1-25`、`packages/orchestrator/src/supervisor.ts:63-155,197-339`
- orchestrator child 与事件转发：`packages/orchestrator/src/rpc-process.ts:25-170`
- project trust：`packages/coding-agent/src/core/project-trust.ts:12-95`
