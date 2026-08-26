# 第 16 讲：AgentSession 的协调职责

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

`AgentSession` 不是另一个 agent loop。低层 `Agent` 已经负责模型调用、工具循环、steering、follow-up 和中止；`AgentSession` 处理的是应用层一致性：把资源和配置编译成 Agent 当前可用的模型、工具与系统提示，把 Agent 事件依次交给扩展、运行模式和会话存储，再在一轮结束后决定重试、压缩或真正结算。

```text
              创建阶段
SettingsManager ─┐
ResourceLoader ──┼─► createAgentSession() ─► Agent + AgentSession
ModelRuntime ────┤                              │
SessionManager ──┘                              │
                                                 ▼
              运行阶段                  AgentSession.prompt()
                                                 │
                ┌────────────────────────────────┼──────────────────────────┐
                ▼                                ▼                          ▼
       ExtensionRunner                    Agent.prompt()             SessionManager
   输入、事件、工具与命令            模型流、工具循环、队列       append-only 历史与分支投影
                ▲                                │                          ▲
                └──────── 事件结算与后续策略 ◄───┴──────────────────────────┘
```

跨 cwd 切换 session 时，还要在外层重建 cwd-bound services。这个职责属于 `AgentSessionRuntime`，不是 `AgentSession.reload()`。两个名字相近，生命周期却不同。

## 1. 六个对象分别拥有什么状态

### 解决的问题

Pi 同时有当前模型、活动消息、配置、扩展、资源和 JSONL 历史。如果这些数据都由一个类保存，reload、session resume 和嵌入式 SDK 很难判断哪些对象应复用，哪些必须重建。当前实现把状态拆给六个对象，再让 `AgentSession` 负责跨对象更新顺序。

| 对象 | 拥有的状态 | 不负责的部分 |
|---|---|---|
| `Agent` | 当前 `model/systemPrompt/tools/messages`、一次 active run、队列、AbortSignal | JSONL 落盘、资源发现、应用级重试和压缩 |
| `SessionManager` | session id、cwd、JSONL entries、leaf 与 branch 投影 | 模型请求和扩展调度 |
| `SettingsManager` | global/project settings 的合并视图及写回 | 把设置应用到正在运行的 Agent |
| `ResourceLoader` | extensions、skills、prompts、themes、context files、SYSTEM/APPEND_SYSTEM | 当前活动工具和 Agent message state |
| `ModelRuntime` | provider 注册、凭据、模型目录、认证与请求分派 | agent 循环和会话历史 |
| `ExtensionRunner` | 本次扩展实例的注册表、事件 reducer、运行时 context | session 的最终状态变更和持久化 |

`AgentSession` 自身保存的主要是协调态：`_isAgentRunActive`、`_lastAssistantMessage`、retry attempt、压缩和 bash 的 AbortController、待提交的 bash/custom messages、tool registry、基础 system prompt，以及连接各对象的回调。

`AgentSessionConfig` 直接接收这些依赖，位于 `packages/coding-agent/src/core/agent-session.ts:177-207`。它不在构造函数里自行搜索配置目录，也不自行打开 session 文件。这样 SDK 可以替换 SessionManager、ResourceLoader 或 tool definitions，测试也能用 in-memory manager 与 faux provider 固定边界。

### 状态不能只看一个副本

同一概念可能有几个生命周期不同的表示：

- `agent.state.messages` 是下一次模型调用使用的活动 transcript；
- `SessionManager` entries 是已发生事件的审计历史；
- `buildSessionContext().messages` 是从当前 branch 重新投影出的 transcript；
- `_steeringMessages/_followUpMessages` 只供应用层显示队列，真正待投递消息还在 Agent 队列；
- `_baseSystemPrompt` 来自资源和活动工具，`_systemPromptOverride` 只在当前 prompt 的续接 turn 内有效。

协调代码经常同时更新两个副本，但目的不同。把 retry error 从 Agent messages 删除，是为了不把失败响应再发给模型；错误已经在 `SessionManager` 中保留，审计历史不会随之删除。

## 2. 创建顺序先固定基础设施，再恢复 session 状态

### 服务与 session 分两步创建

CLI 主链先调用 `createAgentSessionServices()`，再调用 `createAgentSessionFromServices()`，入口分别在 `packages/coding-agent/src/core/agent-session-services.ts:134-178` 和 `packages/coding-agent/src/core/agent-session-services.ts:188-206`。

第一步创建与 cwd 绑定的基础设施：

1. 规范化 cwd 和 agentDir；
2. 创建或复用 ModelRuntime；
3. 创建 SettingsManager；
4. 创建 DefaultResourceLoader 并执行 reload；
5. 把扩展 factory 初始化阶段排队的 provider 注册到 ModelRuntime；
6. 在不访问网络的条件下刷新模型目录；
7. 校验并应用扩展 CLI flag。

这一步只返回 `AgentSessionServices` 和 diagnostics，不创建 Agent。原因是模型范围、工具选项和恢复状态需要以目标 cwd 的 settings、providers 和 resources 为依据。

第二步把明确的 SessionManager、model、thinking level 和 tool options 传给 `createAgentSession()`。该函数位于 `packages/coding-agent/src/core/sdk.ts:164-385`。它先调用 `sessionManager.buildSessionContext()`，已有历史时优先恢复 session 记录的模型与 thinking level；恢复模型不存在或没有认证时，再按 settings 和可用模型选择 fallback。

```ts
const existingSession = sessionManager.buildSessionContext();
const hasExistingSession = existingSession.messages.length > 0;

if (!model && hasExistingSession && existingSession.model) {
	const restoredModel = modelRuntime.getModel(
		existingSession.model.provider,
		existingSession.model.modelId,
	);
	if (restoredModel && modelRuntime.hasConfiguredAuth(restoredModel.provider)) {
		model = restoredModel;
	}
}
```

新 session 会立即 append 初始 model change 和 thinking level；恢复 session 则把 branch 投影直接装入 `agent.state.messages`。如果旧 session 没有 thinking change entry，创建过程补写一条当前默认值，使以后恢复不再依赖当时的 settings。

### Agent 通过闭包读取动态服务

`createAgentSession()` 创建 Agent 时没有把 provider SDK 写死。它给 Agent 注入 `streamFn`、`transformContext`、`onPayload` 和 `onResponse`：

- `streamFn` 从 ModelRuntime 取请求实现，并在请求前读取超时、provider retry 和 attribution headers；
- `transformContext` 把每次 LLM 调用前的 messages 交给当前 ExtensionRunner；
- `onPayload` 和 `onResponse` 接入 provider request/response 扩展事件；
- `convertToLlm` 在每次转换时读取 `blockImages`，所以设置改变后不用重建 converter。

这些闭包通过 `extensionRunnerRef.current` 找当前 runner。reload 替换 runner 后，Agent 的 hook 不必重新安装，也不会继续把 provider 事件发给旧扩展实例。

### 构造函数只接线，不发 session_start

`AgentSession` 构造函数在 `packages/coding-agent/src/core/agent-session.ts:356-381` 完成三件事：订阅 Agent 事件，安装 tool/next-turn hook，构建 ExtensionRunner、工具注册表和基础 system prompt。

`session_start` 要等运行模式调用 `bindExtensions()` 后才发出。此时 TUI、RPC、print 或 JSON 已经提供各自的 UI、abort、shutdown 和 command context bindings。扩展若在 `session_start` 注册动态工具，registerTool 会触发 registry refresh，测试 `packages/coding-agent/test/agent-session-dynamic-tools.test.ts:20-99` 验证了新工具、来源信息和 system prompt 会在 bind 后一起更新。

## 3. AgentSession 把资源编译成每轮可用的运行快照

### 工具变化必须同步 system prompt

ResourceLoader 只知道有哪些扩展和资源。`AgentSession._refreshToolRegistry()` 把内置 definition、扩展工具和 SDK custom tools 合并，应用 allowlist/denylist，再生成 `AgentTool[]` 写入 `agent.state.tools`。

`setActiveToolsByName()` 不只替换工具数组，还会按活动工具重新构建基础 system prompt，见 `packages/coding-agent/src/core/agent-session.ts:914-929`：

```ts
this.agent.state.tools = tools;

this._baseSystemPrompt = this._rebuildSystemPrompt(validToolNames);
this.agent.state.systemPrompt =
	this._systemPromptOverride ?? this._baseSystemPrompt;
```

原因很直接：模型可调用的 tool schemas 和 system prompt 中的工具说明必须是同一集合。只改 `agent.state.tools` 会留下过期说明；只改 prompt 则会告诉模型一个实际不存在的工具。

### turn 之间重新读取可变状态

低层 Agent 在一次包含多个 tool round 的 run 中会生成下一 turn snapshot。`AgentSession._installAgentNextTurnRefresh()` 位于 `packages/coding-agent/src/core/agent-session.ts:499-519`，它保留调用者已有的 `prepareNextTurn`，随后用当前值覆盖 system prompt、tools、model 和 thinking level。

这解决了扩展工具在 execute 中动态增减、扩展在 handler 中切换模型，以及 per-prompt system prompt override 跨 tool round 继续生效的问题。一次 provider 请求已经发出后不会被中途改写；变化从下一 turn 起生效。

### ExtensionRunner 只发意图，AgentSession 落实状态

`_bindExtensionCore()` 把扩展 API 映射为真实动作，见 `packages/coding-agent/src/core/agent-session.ts:2311-2427`。例如：

- `setActiveTools` 调用 AgentSession registry；
- `appendEntry` 调用 SessionManager，并额外发出 `entry_appended`；
- `setModel` 先由 ModelRuntime 检查认证，再同时更新 Agent、session 和 settings；
- `registerProvider` 修改 ModelRuntime，并刷新 Agent 当前持有的 model 对象；
- `compact` 启动 AgentSession 的压缩状态机；
- `abort` 优先交给运行模式清理 UI 队列，否则直接中止 session。

ExtensionRunner 因而不拥有这些动作的最终状态。它只校验 runtime 生命周期并调用已绑定函数；跨对象的一致性顺序仍由 AgentSession 决定。

## 4. 事件结算顺序决定什么能够落盘

### 一条 Agent 事件的应用层路径

构造函数把 `_handleAgentEvent` 订阅到 Agent。低层 Agent 会等待异步订阅者，因此同一个 run 内的下一步不能越过尚未完成的扩展事件与持久化。

`_handleAgentEvent()` 的顺序位于 `packages/coding-agent/src/core/agent-session.ts:573-645`：

1. user `message_start` 若来自 steer/follow-up 队列，先从 UI 镜像队列移除；
2. `await _emitExtensionEvent(event)`；
3. 同步通知 AgentSession 的公开订阅者，即 TUI、JSON、RPC 或 SDK；
4. 对 `message_end` 调用 SessionManager append；
5. assistant message 还会更新 `_lastAssistantMessage`，供 post-run 策略使用。

扩展先于公开订阅者和持久化。`message_end` reducer 若替换消息，AgentSession 会原地修改 Agent 已保存的 message 对象，使随后 `turn_end/agent_end`、公开 listener 与 SessionManager 看到同一个版本。`packages/coding-agent/test/suite/agent-session-runtime.test.ts:123-162` 用修改 assistant cost 的扩展验证内存与持久化结果一致。

### 为什么 tool_call 能看到 assistant tool-use 已落盘

低层 Agent 发出 assistant `message_end` 后，必须等待 AgentSession 的异步 handler 完成，才开始工具 preflight。即使扩展的 `message_end` handler 主动 yield，assistant tool-use entry 仍会先 append，然后 `tool_call` 才触发。

`packages/coding-agent/test/suite/regressions/1717-2113-agent-session-event-settlement.test.ts:29-95` 构造了延迟 20ms 的 message_end handler 和两个工具调用，确认持久化顺序始终是 user、assistant、两个 toolResult、最终 assistant；tool_call handler 读 branch 时已经能看到 user 和 assistant。

如果 AgentSession 只是把事件转发给 UI，不参与这条 await 链，异步 extension handler 会让 tool result 抢先写进 JSONL，恢复时就无法重建合法的 tool use/result 配对。

### 公开 listener 的失败边界

`AgentSession._emit()` 同步遍历 listeners，没有 `try/catch`。listener 抛错会中断当前 `_handleAgentEvent`；如果发生在 message_end，后面的 SessionManager append 可能没有执行，错误继续进入低层 Agent 的订阅者失败路径。

扩展事件大多由 ExtensionRunner 隔离错误，公开 listener 则被当成运行模式的一部分，不是旁路日志。这是当前源码行为。新增 SDK listener 时应自行捕获非关键错误；不能假设 listener 失败只影响显示。

## 5. prompt 是应用级 preflight，不是简单转调

`AgentSession.prompt()` 位于 `packages/coding-agent/src/core/agent-session.ts:1102-1253`。真正调用 `Agent.prompt()` 之前，它按以下顺序处理：

```text
扩展 command
    │未命中
input reducer
    │continue / transform
skill 与 prompt template 展开
    │
    ├─ Agent 正在运行：必须明确 steer 或 followUp，然后入队
    │
    └─ Agent 空闲：flush bash → 校验 model/auth → 检查旧上下文
                       → 组装 user/nextTurn messages
                       → before_agent_start
                       → _runAgentPrompt()
```

preflight 失败发生在新 user message 交给 Agent 之前。没有模型、没有认证、streaming 时未指定队列方式都会直接抛错，不产生新的模型消息。扩展 command 或 input `handled` 则是正常消费，返回前不会启动 Agent。

`before_agent_start` 可以追加 custom messages，并给这一 prompt 设置 system prompt override。`_runAgentPrompt()` 的 `finally` 会清除 override，所以它能跨越本次 run 的多个 tool turn，却不会泄漏到下一条独立 prompt。

### isStreaming 是应用级 settlement 锁

Agent 自己的 active run 在一次 `agent_end` 后已经结束；AgentSession 可能还在等待 retry backoff、执行压缩，或者准备下一次 `continue()`。因此 `AgentSession.isStreaming` 实际读取 `_isAgentRunActive`，直到全部 post-run continuation 完成才变回 false。

这也是 `AgentSession.waitForIdle()` 与低层 `Agent.waitForIdle()` 的区别。前者等待 retry、compaction 和 extension 在 agent_end 后排入的 continuation；后者只知道当前 Agent run。

## 6. post-run 循环把多个 Agent run 收束成一次 settled

### 单一控制点

`_runAgentPrompt()` 位于 `packages/coding-agent/src/core/agent-session.ts:1049-1091`：

```ts
this._isAgentRunActive = true;
try {
	await this.agent.prompt(messages);
	while (await this._handlePostAgentRun()) {
		await this.agent.continue();
	}
} finally {
	this._systemPromptOverride = undefined;
	this._flushPendingBashMessages();
	await this._emitAgentSettled();
}
```

每次 Agent run 完成后，`_handlePostAgentRun()` 取已经通过事件链结算的最后一条 assistant message，按固定优先级决策：

1. 可重试 provider error：准备 retry；
2. retry 已耗尽：发出失败结束事件并清零计数；
3. context overflow 或 threshold：检查并执行 auto-compaction；
4. agent_end 扩展刚排入消息：继续一次以交付队列；
5. 都没有：退出循环并发出 `agent_settled`。

普通 retry 明确排在 compaction 前面，但 `_isRetryableError()` 先排除 context overflow，所以溢出不会占用 provider retry 次数。这个顺序把网络/限流恢复与上下文恢复分成两套预算。

`finally` 保证 prompt 失败、中止或 continuation 失败时也会清理 per-turn prompt、提交延迟的 bash message，并解除 session idle barrier。`agent_settled` 才是应用层“不会自动再发请求”的事件。

## 7. 自动重试保留失败历史，只清理下一次请求上下文

### 决策与状态变化

`_prepareRetry()` 位于 `packages/coding-agent/src/core/agent-session.ts:2620-2670`。它每次增加 `_retryAttempt`，超过 settings 的 `maxRetries` 就拒绝继续；未超过时发出 `auto_retry_start`，用 `baseDelayMs * 2^(attempt-1)` 计算退避并等待可中止的 sleep。

在 sleep 前，最后一条 error assistant 已经经过 message_end 并写入 SessionManager。AgentSession 随后只从 `agent.state.messages` 删除它。下一次 `agent.continue()` 从上一个有效上下文恢复，而 JSONL 仍记录每次失败。

重试成功时，下一条非 error assistant 的 message_end 发出 `auto_retry_end{success:true}` 并清零计数。达到上限、错误不可重试或 sleep 被取消时都不再 `continue()`；取消还会给 UI 一个 `finalError:"Retry cancelled"`。

`packages/coding-agent/test/suite/agent-session-retry-events.test.ts:33-201` 固定 faux responses，验证了一次和多次恢复、最大次数、禁用、不可重试错误、取消 sleep，以及恢复响应继续产生工具调用时 `prompt()` 仍等待到最终答案。

### 失败路径

retry 的“成功”指后续得到非 error assistant，不代表工具或业务目标成功。反过来，`prompt()` 没抛异常也不代表 provider 首次请求成功；错误以 assistant message 和 retry events 留在历史中。调用方应观察最终 assistant、`auto_retry_end` 与 `agent_settled`，不能只看 Promise 是否 fulfilled。

## 8. 自动压缩先写 compaction entry，再重建 Agent messages

### 两种触发时机

AgentSession 在两个位置检查压缩：

- 一个 Agent run 结束后检查刚产生的 assistant；
- 新 prompt 提交前检查历史中最后一个 assistant，覆盖之前被 abort 或 length stop 留下的高占用上下文。

`_checkCompaction()` 位于 `packages/coding-agent/src/core/agent-session.ts:1935-2024`。它区分：

- context overflow：错误响应可以压缩后自动 `continue()`；成功响应虽然 usage 超窗，只压缩，不重发已经完成的答案；
- threshold：接近窗口时压缩，默认不自动请求模型；如果此时队列仍有消息，继续一次交付队列。

overflow 恢复每条新 user message 最多尝试一次。`_overflowRecoveryAttempted` 在 user `message_start` 时重置；第二次仍 overflow 会发出明确错误，不形成无限“压缩—重试”循环。

### 提交顺序

`_runAutoCompaction()` 的成功路径是：

1. 从 SessionManager 当前 branch 计算 preparation；
2. 允许 `session_before_compact` 取消或提供完整 CompactionResult；
3. 否则用当前 model、auth、thinking level 和 streamFn 生成摘要；
4. `sessionManager.appendCompaction(...)`；
5. 再调用 `buildSessionContext()`，把新投影写入 `agent.state.messages`；
6. 发 `session_compact` 和 `compaction_end`；
7. 根据 `willRetry` 或 queued messages 决定是否让 post-run loop `continue()`。

先持久化边界，再重建内存，意味着恢复来源仍是 session tree。压缩不是直接就地删除 Agent 数组中的旧消息；Agent messages 是压缩 entry 产生后的派生视图。

### 失败与过期 usage

自动压缩失败不会让整个 prompt Promise rejected。它发出带 `errorMessage` 的 `compaction_end` 并返回 false，post-run loop 随后结算。手动 `compact()` 则会把错误重新抛给调用者。

压缩后的 kept assistant 仍带有压缩前 usage。`_checkCompaction()` 会用 compaction timestamp 排除旧 usage；error/zero-usage 响应只有找到压缩后有效 usage 时才用估算值。`packages/coding-agent/test/suite/agent-session-compaction.test.ts:233-445` 固定了单次 overflow、成功 overflow 不重试、旧 usage 不重复压缩，以及无有效 usage 时不擅自触发 threshold。

## 9. reload 是同一个 session 内的部分重建

### 重载顺序

`AgentSession.reload()` 位于 `packages/coding-agent/src/core/agent-session.ts:2577-2599`：

1. 保存扩展 flag values；
2. 对旧 runner 发 `session_shutdown{reason:"reload"}`；
3. `settingsManager.reload()`；
4. 把 steering/follow-up mode 同步到 Agent；
5. 重置 provider 注册并让 ResourceLoader reload；
6. 重建内置工具 definition、ExtensionRunner、provider bindings、tool registry 和 system prompt；
7. 恢复活动工具名称与 flag values；
8. 运行模式已绑定时，发新的 `session_start` 和 `resources_discover`。

reload 保留 Agent、SessionManager、ModelRuntime 和当前 transcript，不改变 cwd，不打开新的 session 文件。`packages/coding-agent/test/suite/agent-session-model-extension.test.ts:352-371` 固定了 `start:startup → shutdown:reload → start:reload`；`packages/coding-agent/test/suite/regressions/2753-reload-stale-resource-settings.test.ts:25-105` 则修改顶层 prompt settings，确认 reload 后旧 prompt template 从当前资源集合移除。

### 哪些设置能立即生效

配置刷新不是全量替换 Agent：

- retry、compaction、shell command、认证检查等值在执行对应动作时读取；
- provider timeout 与 `blockImages` 在每次请求/消息转换时读取；
- queue mode、工具 definition、资源、扩展和 system prompt 由 reload 显式同步或重建；
- model/thinking/tool 的公开 setter 会同时更新运行内存及其应写入的 session/settings；
- Agent 构造时捕获的 `transport`、`thinkingBudgets` 等字段不会仅因 `SettingsManager.reload()` 自动全部替换。交互模式修改 transport 时会同时写 settings 和 `session.agent.transport`。

判断某项设置是否支持运行中刷新，必须追到消费位置：getter 出现在请求路径、reload 同步函数，还是只出现在 Agent 构造参数。看到 settings 文件已经重读，不能推断所有派生状态都已更新。

## 10. 跨 session 与跨 cwd 由 AgentSessionRuntime 更换整套服务

### 为什么 reload 不够

Session header 保存自己的 cwd。resume 到另一个项目时，project settings、resources、extensions 和 trust 结论都应以目标 cwd 重新计算；继续复用旧 AgentSession 只替换 SessionManager 会把两个项目的运行环境混在一起。

`AgentSessionRuntime` 位于 `packages/coding-agent/src/core/agent-session-runtime.ts:68-399`。它持有“当前 AgentSession + 当前 AgentSessionServices”，new/resume/fork/import 的共同顺序是：

```text
session_before_switch / session_before_fork
          │ cancel 时原 runtime 不动
          ▼
解析并校验目标 SessionManager/cwd
          ▼
旧扩展 session_shutdown
          ▼
宿主同步拆除扩展 UI → old session.dispose()/invalidate
          ▼
createRuntime(target cwd, target SessionManager)
          │ 新 Settings/Resources/Extensions/AgentSession
          ▼
apply → rebind run mode → withSession(fresh context)
```

`packages/coding-agent/test/suite/agent-session-runtime.test.ts:450-520` 创建两个不同 cwd 的 runtime，再 switch 到第二个 session，确认 runtime cwd 与新 SessionManager cwd 同步改变。`packages/coding-agent/test/suite/agent-session-runtime.test.ts:522-590` 还验证目标 session 的 model 与 thinking state 会被恢复，而不是沿用来源 session 的内存值。

### replacement 失败没有事务回滚

实现先 teardown 旧 session，再调用异步 `createRuntime()`。新服务或 session 创建失败时，异常交给运行模式处理；代码没有恢复已经 dispose 的旧 runtime。类注释也明确把 user-facing error handling 交给 caller。

这种顺序保证旧扩展先释放 watcher/UI，并避免两个 runtime 同时活动，代价是切换不是可回滚事务。需要更强可用性的替代方案是先在隔离状态创建并验证新 runtime，再原子交换；但它会让新旧扩展在一段时间内并存，也要处理 provider 全局注册和 UI 资源冲突。当前实现选择生命周期清晰，而不是失败后自动回退。

## 11. 设计取舍与修改检查点

### AgentSession 是协调器，也是当前实现的集中风险点

集中协调让更新顺序可从一个类追踪：扩展修改何时落盘、重试何时删除活动错误、压缩何时重建 transcript，都有单一入口。代价是类已经同时承担 prompt preflight、事件 settlement、模型切换、工具 registry、压缩、retry、bash、tree navigation 和 export，任何新增状态都容易形成漏同步。

修改 `AgentSession` 时，最有用的检查不是“方法能调用”，而是下面几组不变量：

- Agent message、公开事件和 SessionManager entry 是否仍按同一顺序结算；
- model/tool/system prompt 的运行快照是否在下一 turn 一起更新；
- retry/compaction 删除的是活动上下文还是审计历史；
- `agent_end` 与 `agent_settled` 是否仍代表不同完成边界；
- reload 后哪些对象保留，哪些重建，旧回调是否还可能访问旧 runtime；
- session replacement 在失败前已经改变了哪些外部状态。

对应的最小测试面包括：

- prompt 与工具续轮：`packages/coding-agent/test/suite/agent-session-prompt.test.ts`；
- retry 和 settlement：`packages/coding-agent/test/suite/agent-session-retry-events.test.ts`；
- message 顺序回归：`packages/coding-agent/test/suite/regressions/1717-2113-agent-session-event-settlement.test.ts`；
- 自动/手动压缩：`packages/coding-agent/test/suite/agent-session-compaction.test.ts`；
- 扩展、模型和 reload：`packages/coding-agent/test/suite/agent-session-model-extension.test.ts`；
- session/cwd replacement：`packages/coding-agent/test/suite/agent-session-runtime.test.ts`。

## 12. 常见边界问题

### AgentSession 是状态唯一来源吗？

不是。它协调多个状态所有者。当前模型和活动 transcript 在 Agent；完整历史和 branch 在 SessionManager；资源在 ResourceLoader；配置在 SettingsManager。AgentSession 保存少量协调态，并规定跨对象更新顺序。

### 为什么 provider error 已写入 JSONL，重试时却看不到它？

message_end 先完成持久化，post-run 策略随后从 `agent.state.messages` 删除 error assistant。下一请求不携带失败响应，历史文件仍能解释发生过几次失败。

### 收到 `agent_end` 后能否认为不会再请求模型？

不能。AgentSession 可能退避后 retry、overflow compact 后 continue，或交付 agent_end 扩展刚排入的消息。`agent_settled` 才表示这些自动 continuation 都结束。

### reload 与 resume 有什么本质区别？

reload 在同一个 AgentSession 内重读 settings/resources/extensions，保留 cwd、SessionManager 和 Agent transcript。resume 由 AgentSessionRuntime 销毁旧 session，并按目标 session 的 cwd 重建 services 与 AgentSession。

### SettingsManager.reload() 是否保证所有运行配置生效？

不保证。SettingsManager 只更新配置视图。消费方若在每次动作时调用 getter，或 AgentSession.reload 显式同步，变化才进入运行态；只在 Agent 构造时读取的设置需要对应 setter 或重建 session。

### 压缩是在 Agent messages 上删除旧记录吗？

不是。压缩先向 SessionManager append compaction entry，再从当前 branch 构建新的 session context，最后替换 Agent messages。完整 entries 仍保留，模型上下文只是其派生投影。
