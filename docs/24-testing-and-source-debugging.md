# 第 24 讲：测试分层与源码调试

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的测试不是一套从“小”到“大”均匀扩张的金字塔。不同层使用不同替身，分别固定 provider 输出、Agent 状态、文件系统和终端行为。源码调试的要点是先找出错误跨过了哪条边界，再选择能观察那条边界的最小测试。

```text
协议转换与纯函数
  │ 固定输入，直接断言返回值
  ▼
Agent loop
  │ 自定义 EventStream，断言事件、工具执行与消息顺序
  ▼
Agent + faux provider
  │ 可编程响应队列，模拟流式 delta、错误、abort、usage 与 cache
  ▼
AgentSession suite
  │ 内存会话/设置/鉴权 + 临时目录 + 扩展 + faux provider
  ▼
序列化 fixture / TUI 虚拟终端
  │ 真实 JSONL 兼容性 / ANSI、光标、宽字符和 viewport
  ▼
真实 provider e2e
  │ 网络、凭据、供应商协议与计费模型
```

上层测试覆盖更多组件，也引入更多失败来源。一个 provider 请求失败，可能来自网络、凭据、模型下线或服务端变化；同样的状态机若能用 faux provider 复现，就应该先在本地确定 Pi 自己的行为。

## 1. 测试入口并不统一

### workspace 调度

根 `package.json` 的 `test` 脚本执行 `npm run test --workspaces --if-present`，把控制权交给各 package。四个主要包的 runner 有差异：

| package | 默认测试命令 | 主要用途 |
|---|---|---|
| `packages/ai` | `vitest --run` | provider 转换、stream、重试、鉴权和可选线上测试 |
| `packages/agent` | `vitest --run` | Agent 状态与 agent loop；另有 harness 专用配置 |
| `packages/coding-agent` | `vitest --run` | AgentSession、会话、资源、工具、扩展和运行模式 |
| `packages/tui` | `node --test ... test/*.test.ts` | 终端组件、ANSI、布局和差分渲染 |

`packages/tui/vitest.config.ts` 只包含 `wrap-ansi.test.ts`，但 package 的默认脚本使用 Node test runner。看到仓库里有 Vitest 配置，不能据此认为所有测试都由 Vitest 执行。

### 测试直接引用源码

`packages/agent/vitest.config.ts:1-23` 将 `@earendil-works/pi-ai` 映射到相邻 workspace 的 `src/index.ts`；`packages/coding-agent/vitest.config.ts:1-35` 同样把 ai、agent、tui 依赖映射到源码。定向测试不依赖先构建 `dist`，跨包修改会直接进入被测调用链。

这个设计缩短了反馈路径，也带来一个判断边界：源码测试通过说明 TypeScript 源码组合可运行，不等于发布包的 exports、复制资产和二进制构建已经正确。发布问题仍需要 build 或 release smoke test。

### `check` 与 `test` 的职责

根 `npm run check` 执行 Biome、依赖固定检查、相对 import 检查、两类发布锁检查、TypeScript 类型检查和浏览器 smoke。它不运行测试。代码修改通常至少需要：

```text
定向测试：验证改动的运行时行为
    +
npm run check：验证格式、类型、依赖和生成物约束
```

只跑其中一个会留下明显空缺。类型检查无法证明事件顺序，测试也可能没有覆盖错误的 import 或发布锁漂移。

## 2. faux provider 固定模型输出

### 解决的问题

Agent 测试需要模型连续返回文本、thinking、tool call、错误和中止结果。如果使用真实模型，同一 prompt 可能产生不同工具参数，网络和计费也进入故障范围。`packages/ai/src/providers/faux.ts` 实现了一个使用正式 `Model`、`AssistantMessageEventStream` 和 provider dispatch 接口的可编程 provider。

它不是简单地一次返回字符串。测试可以配置多个模型、响应队列、异步 response factory、token chunk 大小和发送速率；每次调用仍产生与真实 provider 同形的 stream 事件。

### 响应队列与可观察状态

`createFauxCore()` 位于 `packages/ai/src/providers/faux.ts:346-474`。调用 stream 时先从队列取一个 step，再增加 `callCount`。step 可以是固定 `AssistantMessage`，也可以是读取本次 `Context`、`StreamOptions`、调用次数和模型的函数：

```ts
const stream: StreamFunction<string, StreamOptions> = (requestModel, context, streamOptions) => {
	const outer = createAssistantMessageEventStream();
	const step = pendingResponses.shift();
	state.callCount++;

	queueMicrotask(async () => {
		try {
			await streamOptions?.onResponse?.({ status: 200, headers: {} }, requestModel);
			if (!step) {
				let message = createErrorMessage(
					new Error("No more faux responses queued"),
					api,
					provider,
					requestModel.id,
				);
				message = withUsageEstimate(message, context, streamOptions, promptCache);
				outer.push({ type: "error", reason: "error", error: message });
				outer.end(message);
				return;
			}

			const resolved = typeof step === "function"
				? await step(context, streamOptions, state, requestModel)
				: step;
			let message = cloneMessage(resolved, api, provider, requestModel.id);
			message = withUsageEstimate(message, context, streamOptions, promptCache);
			await streamWithDeltas(outer, message, minTokenSize, maxTokenSize, tokensPerSecond, streamOptions?.signal);
		} catch (error) {
			const message = createErrorMessage(error, api, provider, requestModel.id);
			outer.push({ type: "error", reason: "error", error: message });
			outer.end(message);
		}
	});

	return outer;
};
```

队列耗尽会产生明确的 error message。旧 coding-agent harness 会循环复用响应，测试漏写一次 provider 返回也可能侥幸通过；新 faux provider 的耗尽错误把额外调用暴露出来。`packages/ai/test/faux-provider.test.ts:117-159` 固定了顺序消费、替换、追加和耗尽语义。

response factory 可以检查模型实际收到的上下文。例如工具测试的第二个响应可以统计 `toolResult`，从而确认工具结果已在下一次模型调用前进入 context。这比在测试外部猜测“第二次应该调用了”更接近真实边界。

### 流式状态与失败注入

`streamWithDeltas()` 在 `packages/ai/src/providers/faux.ts:229-344` 按内容块发出 start、delta、end 和 terminal event。tool call 参数先按 JSON 字符串产生 delta，最后才替换为完整对象；设置 `tokensPerSecond` 后可以在中途触发 abort。

faux provider 可以稳定制造这些路径：

- `stopReason: "toolUse"` 后继续下一次模型调用；
- `stopReason: "error"` 的 terminal error；
- 请求开始前或 text、thinking、tool call 中途 abort；
- 多个 tool call 的固定 id 和参数；
- response factory 抛错；
- 每个 session 独立的 prompt cache usage。

`packages/ai/test/faux-provider.test.ts:329-586` 不只断言最终文本，还逐项断言事件顺序和 abort 后缺少对应 end event。失败时能够判断问题发生在事件组装、Agent 消费还是更上层的会话协调。

### 两种注册方式

`fauxProvider()` 返回普通 `Provider`，适合显式 `Models` 集合；`registerFauxProvider()` 在兼容层的 provider registry 注册唯一 api，并返回 `unregister()`。后者见 `packages/ai/src/compat.ts:160-175`，coding-agent suite 用它驱动仍经过兼容 dispatch 的 `AgentSession`。

注册表是进程级状态。每个测试完成后必须 unregister，否则后续测试可能命中残留 provider。新 suite 的 cleanup 同时 dispose session、注销 faux provider 并删除临时目录。

## 3. Agent loop 测试直接控制事件源

### 为什么不用完整 AgentSession

`agentLoop()` 的职责是消费模型流、执行工具、插入 steering/follow-up、维持消息顺序并决定是否继续。若为这些规则创建完整 session，会把资源加载、鉴权和持久化带进测试。`packages/agent/test/agent-loop.test.ts` 因此自建 `MockAssistantStream` 和 stream function，只保留 loop 所需的 `Context`、`Model`、tool 与 config。

`MockAssistantStream` 复用正式 `EventStream` 的 terminal 判断：

```ts
class MockAssistantStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
	constructor() {
		super(
			(event) => event.type === "done" || event.type === "error",
			(event) => {
				if (event.type === "done") return event.message;
				if (event.type === "error") return event.error;
				throw new Error("Unexpected event type");
			},
		);
	}
}
```

测试在 microtask 中 push terminal event，随后同时观察三类结果：`stream.result()` 返回的新增消息、异步迭代得到的 `AgentEvent[]`、工具实现记录的副作用。单看最终文本会漏掉事件顺序和重复执行。

### 状态变化如何被断言

`packages/agent/test/agent-loop.test.ts:239-308` 的工具主链准备两次 provider 响应：第一次返回 tool call，第二次返回结束文本。断言不仅检查工具确实执行，还检查 `tool_execution_start/end`。

更容易出错的是并发顺序。`packages/agent/test/agent-loop.test.ts:525-968` 用受控 Promise 卡住一个工具，验证：

- execution event 按实际完成顺序出现；
- 写入 context 的 tool results 保持模型原始 tool call 顺序；
- 任一工具声明 sequential 时整批串行；
- 全部允许 parallel 时才能观察到并行。

这里不能用固定 sleep 猜先后。测试保留 `releaseFirst` 回调，先确认第二个工具已到达指定状态，再释放第一个工具。同步点把调度条件变成可证明的状态。

### 失败路径

同一文件把失败变成模型可观察结果：未知工具、参数校验失败、`beforeToolCall` 阻断和工具异常都应生成 error tool result。`stopReason: "length"` 中即使修复出的 tool 参数通过 schema，也不能执行，因为参数可能在 token 边界被截断，见 `packages/agent/test/agent-loop.test.ts:310-381`。

这一层最适合修改 `packages/agent/src/agent-loop.ts` 后定位问题。它不验证 SessionManager 是否及时 append，也不验证扩展 handler 何时看到持久化状态；这些属于下一层。

## 4. AgentSession suite 验证组件之间的接缝

### harness 组装了什么

新的 suite 规则写在 `packages/coding-agent/test/suite/README.md:1-16`：测试必须使用 `test/suite/harness.ts` 和 faux provider，不接真实 API、key、网络或付费 token；通用生命周期测试放 suite 根目录，issue 回归放 `regressions/<issue>-<slug>.test.ts`。

`createHarness()` 在 `packages/coding-agent/test/suite/harness.ts:90-204` 组装：

```text
临时 cwd
  + SessionManager.inMemory()
  + SettingsManager.inMemory()
  + AuthStorage.inMemory()
  + in-memory ModelRegistry
  + registerFauxProvider()
  + 可选 tools / resource loader / inline extensions
  → Agent
  → AgentSession
  → 统一收集 AgentSessionEvent[]
```

默认鉴权值是 `faux-key`，不会发到网络。测试可用 `withConfiguredAuth: false` 验证缺少 credential 的预检失败。`baseToolsOverride`、allowlist、exclude list 和扩展工厂允许只替换当前问题需要的边界，其余部分保持正式实现。

cleanup 是测试正确性的一部分：它 dispose session、unregister provider 并递归删除临时目录。定时器、watcher、全局 registry 或未释放的 session 残留，都会造成“单独通过、整组失败”的污染。

### 从 prompt 到 settle

`packages/coding-agent/test/suite/agent-session-prompt.test.ts:30-116` 先覆盖简单 prompt，再覆盖一次 tool call 和同一 assistant response 中的多个 tool call。典型用法是：

```ts
harness.setResponses([
	fauxAssistantMessage(fauxToolCall("echo", { text: "hello" }), { stopReason: "toolUse" }),
	fauxAssistantMessage("done"),
]);

await harness.session.prompt("start");

expect(toolRuns).toEqual(["hello"]);
expect(harness.session.messages.map((message) => message.role)).toEqual([
	"user",
	"assistant",
	"toolResult",
	"assistant",
]);
```

这里 `await prompt()` 的完成点是 session-level settle：工具已完成，后续模型响应也已收束。测试若只等待 `message_end`，可能在自动重试、follow-up 或持久化完成前开始断言。

### 接缝问题为什么需要 suite

回归 `packages/coding-agent/test/suite/regressions/1717-2113-agent-session-event-settlement.test.ts:20-96` 给扩展的 `message_end` handler 人为加入异步让步，再检查 SessionManager 中的角色顺序。第二个测试在 `tool_call` hook 内读取 branch，要求此时已经存在 user 和 assistant tool-use message。

这类错误在 agent loop 单测里看不见：loop 只保证事件顺序，不拥有 JSONL append；单测 SessionManager 也看不见扩展何时被调用。suite 的价值就在于固定两个组件之间的先后关系。

### 旧 harness 与新 suite

`packages/coding-agent/test/test-harness.ts` 是旧路径，自行实现 faux stream、落盘 auth 与 SettingsManager。它的 response sequence 会循环复用。新规则明确要求不要继续扩展旧 harness，除非新 harness 缺少必要能力。

新增回归时混用两套 harness 会产生不同语义：队列耗尽、usage、cache、abort 和 provider 注册方式都不完全一致。迁移测试前应先确认它依赖的是业务行为，还是旧替身的偶然行为。

## 5. 会话 fixture 验证磁盘格式与迁移

### 内存对象覆盖不了什么

直接构造 `SessionEntry[]` 适合验证 tree 和 context 算法，却绕过 JSONL 解析、旧版本迁移、损坏行处理以及真实大消息。`packages/coding-agent/test/fixtures/large-session.jsonl` 保存了一段旧版本长会话；`packages/coding-agent/test/compaction.test.ts:34-40` 先调用 `parseSessionEntries()`，再执行 `migrateSessionEntries()`：

```ts
function loadLargeSessionEntries(): SessionEntry[] {
	const sessionPath = join(__dirname, "fixtures/large-session.jsonl");
	const content = readFileSync(sessionPath, "utf-8");
	const entries = parseSessionEntries(content);
	migrateSessionEntries(entries);
	return entries.filter((e): e is SessionEntry => e.type !== "session");
}
```

`packages/coding-agent/test/compaction.test.ts:512-538` 用它验证三件事：旧会话仍能解析；cut point 落在合法 message；迁移后的会话能重建消息与模型状态。后面的真实摘要测试带 `ANTHROPIC_OAUTH_TOKEN` 才启用，说明同一个 fixture 既能服务纯本地算法，也能用于显式线上检查。

### fixture 的选择

| 要验证的行为 | 合适输入 |
|---|---|
| 新 entry 的 parentId、分支和 label | 测试内构造最小 entry |
| 老版本 header 或字段迁移 | 保留旧格式 JSONL fixture |
| malformed line、空文件、损坏 header | 临时文件写入精确字节 |
| 大会话 cut point、token 估算和恢复 | 脱敏后的代表性长会话 fixture |
| 当前 session append 顺序 | `SessionManager.inMemory()` + suite 事件 |

fixture 不是随手保存的现场数据。提交前要去掉凭据、绝对路径和无关长文本，并让测试断言其格式价值。若只断言“行数大于 100”，fixture 内容发生误删时未必能定位具体契约；迁移版本、关键 entry 类型和目标状态应有针对性断言。

## 6. TUI 测试的是终端状态机

### 虚拟终端不是字符串快照

TUI 输出包含 cursor move、clear line、style reset、宽字符占位和 scrollback。把 `terminal.write()` 的字符串拼起来只能确认发送过哪些 escape sequence，无法确认终端应用这些序列后的屏幕。

`packages/tui/test/virtual-terminal.ts:1-218` 用 `@xterm/headless` 实现正式 `Terminal` 接口。`write()` 把 ANSI 交给 xterm；`getViewport()` 和 `getScrollBuffer()` 从解析后的 buffer 读取内容；`resize()` 会触发正式 resize callback；`sendInput()` 模拟键盘输入。

写入 xterm 是异步的，因此断言前需要 flush。`waitForRender()` 还等待 TUI 的节流渲染：

```ts
/** Wait for TUI's throttled render pipeline to settle. */
async waitForRender(): Promise<void> {
	await new Promise<void>((resolve) => process.nextTick(resolve));
	await new Promise<void>((resolve) => setTimeout(resolve, 20));
	await this.flush();
}
```

少一次等待会形成时序假失败：实现没有错，断言读到的是上一帧。

### 两种观察方式

`packages/tui/test/tui-render.test.ts` 同时使用 xterm 最终状态和 `LoggingVirtualTerminal` 的原始 writes：

- viewport 断言用户最终看见的字符、光标位置和清除后的空行；
- raw writes 断言是否走了差分更新、全量清屏或 Kitty image 删除协议。

两者不能互相替代。相同 viewport 可能由一次小范围更新或昂贵的全屏重绘得到；只看最终画面发现不了性能回归。相反，escape sequence 顺序正确也不保证宽字符边界在终端 buffer 中正确。

coding-agent 的 `edit-tool-no-full-redraw.test.ts:68-242` 把大 diff 的 `ToolExecutionComponent` 放进 TUI，记录 `fullRedraws` 和 clear-screen 次数，再把 tool call preview 结算为 result。它验证应用组件更新不会触发整屏闪烁，是跨 package 的渲染接缝测试。

### 平台差异

终端宽度、Windows 子进程关闭、剪贴板和原生图片依赖都有平台分支。能够用 VirtualTerminal 表达的行为应保持平台无关；必须触发真实系统 API 时，测试需要明确平台条件和 cleanup。把 Windows 专有故障硬塞进一个假 POSIX terminal，得到的只是错误的替身。

## 7. 真实 provider e2e 必须显式隔离

### 为什么不能直接跑全量 Vitest

`packages/ai/test` 中不少文件把纯转换测试和真实 API 测试放在同一个 suite，线上部分通过 `describe.skipIf(!process.env.*_API_KEY)` 或 OAuth token 判断是否启用。机器环境里只要已有 key，全量 `vitest` 就可能发起网络请求并产生费用。

根 `test.sh:1-81` 在运行 workspace tests 前：

- 临时移走用户 `auth.json`，并用 `trap` 在退出时恢复；
- 设置 `PI_NO_LOCAL_LLM=1`；
- unset 各 provider key、云平台 ambient credential、gateway 和 GitHub token；
- 最后才执行 `npm test`。

所以它的目标不是测试真实 provider，而是让常规全量测试保持离线、无凭据。源码根 `AGENTS.md:30-39` 也规定：非 e2e 测试走 `./test.sh`，其余情况只运行 package 下的指定文件，不直接启动完整 Vitest suite。

### e2e 能证明什么

真实 e2e 用来回答替身无法回答的问题：供应商当前是否接受 payload、SSE/WebSocket 是否按预期终止、OAuth 或 ambient auth 是否可用、某模型的 thinking/tool 协议是否改变。它不适合证明确定性的 Agent loop 顺序。

线上测试常带 timeout 和 retry，这是为网络抖动留余量。retry 后通过不能自动归为本地稳定；若修改只涉及本地转换，仍应有不依赖网络的回归测试保存精确响应结构。

文件名也不能代替源码判断。`packages/agent/test/e2e.test.ts` 当前全部使用 faux provider，验证的是 Agent 集成路径，不接真实模型；是否联网要看 provider 创建和环境门控。

## 8. 从故障到最小回归测试

### 先确定状态所有者

调试时先问“错误状态由谁拥有”：

| 现象 | 首选层 | 主要观察值 |
|---|---|---|
| provider payload 或流事件错误 | `packages/ai` 转换/stream 测试 | 请求体、delta、terminal message |
| 工具未执行、重复执行、顺序错误 | `packages/agent/test/agent-loop.test.ts` | tool side effect、event、result messages |
| prompt、retry、compaction、扩展与持久化先后错误 | coding-agent suite | session events、messages、branch entries、faux callCount |
| 老会话打不开或迁移后状态错 | session-manager 测试 + JSONL fixture | parsed entries、version、leaf、projected context |
| 终端残影、光标错、整屏闪烁 | TUI VirtualTerminal / raw writes | viewport、cursor、scrollback、full redraw count |
| 仅某供应商线上失败 | 显式 provider e2e | HTTP 状态、响应体、真实 terminal event |

把复现放在状态所有者所在层，失败信息最短。然后才向上加一层，确认调用方是否正确消费该状态。

### 构造可控时间

并发和中止测试应使用 Promise 门闩、事件订阅或 fake timer，把“何时继续”交给测试。固定 `sleep(100)` 同时有两种坏结果：慢机器偶发失败，快机器平白等待。TUI 自身有节流周期时可以等待其公开 settle 方法，但断言业务并发仍应等待具体事件。

### 断言三个平面

一次 Agent 故障通常同时改变：

1. 对外事件；
2. 内存中的 messages / pending state；
3. SessionManager 持久化分支。

不是每个测试都要机械断言三遍。若 bug 正好发生在两个平面不同步的接缝，就必须同时观察。例如 settlement 回归既检查 extension hook 的触发时刻，也检查 branch 中 assistant 与 toolResult 的顺序。

### 保留失败路径

回归测试的响应队列要覆盖导致 bug 的失败与恢复，不要只保留修复后的成功响应。网络重试应先给出 error assistant message 再给 recovered response；overflow 应明确区分第一次恢复和第二次仍溢出；abort 应确认 terminal reason、残留 partial content 和后续是否继续。

## 9. 修改后的最小验证矩阵

| 修改范围 | 必跑测试 | 需要追加的验证 |
|---|---|---|
| `packages/ai` 纯转换、usage、stream parser | `packages/ai/test` 下的对应功能测试 | 只有真实服务契约存疑时跑单个 e2e case |
| `packages/agent/src/agent-loop.ts` | `agent-loop.test.ts` + 新回归 | 若影响 `Agent` 状态，再加 `agent.test.ts` 或 faux 集成测试 |
| `AgentSession`、队列、retry、compaction | 对应 `test/suite/agent-session-*.test.ts` | issue 回归放 suite/regressions；涉及磁盘再加 SessionManager |
| `SessionManager` 格式或迁移 | 对应 `test/session-manager/*.test.ts` | 添加最小旧格式 fixture；必要时跑 compaction 大 fixture |
| 内置工具 | 工具自己的 test | 涉及消息持久化或渲染时再加 suite / TUI 接缝测试 |
| `packages/tui` 布局和渲染 | `node --test test/<feature>.test.ts` | 宽字符、resize、raw writes 与 viewport 按问题选择 |
| CLI / RPC / 进程退出 | 对应 print、RPC、process test | 需要时启动子进程；不要用 in-process mock 替代退出码 |
| 依赖、类型或跨包 API | 上述行为测试 + `npm run check` | 发布 exports/资产变化再做 build 或 local release smoke |

Vitest 定向运行从目标 package 根执行：

```bash
node ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts
```

TUI 使用 Node runner：

```bash
node --test test/specific.test.ts
```

新增或修改测试文件时，应先让该文件稳定通过，再扩大到同层相关文件。常规全量非 e2e 验证使用根 `./test.sh`，由它清理 credential 环境。代码改动最后运行 `npm run check`；文档改动不需要用全仓 build 冒充文档验证。

## 10. 失败分类

### 环境问题

测试 runner 或依赖不存在、Node 版本不符、可选原生模块无法加载、平台命令缺失，测试尚未到达断言。这种结果只能写“未执行”，不能写通过或上游缺陷。

### 测试或文档错误

源码行为与固定基线一致，但测试替身建模错误、等待了错误事件、复用了旧 harness 的循环语义，属于测试错误。笔记中的路径、符号、行号或结论与固定 commit 不符，则是文档错误。两者都应修改自己的产物，不能归咎于环境。

### 实现缺陷

测试到达目标断言，输入满足前置条件，实际状态违反源码声明或已有契约，才有实现缺陷的证据。平台差异和外部服务变化仍要先排除。真实 e2e 失败只能证明端到端契约当前不成立，不能单凭一次 HTTP 错误判断 Pi 实现有 bug。

## 11. 设计取舍

Pi 的 faux provider 复用了正式流接口，代价是测试替身本身有 500 多行，也需要独立测试。它比 `mockResolvedValue("ok")` 更重，却能复现 delta、tool call、abort 和 cache usage；这些正是 Agent 状态机最容易出错的部分。

AgentSession suite 大量使用内存 store，但仍保留临时 cwd 和可选真实文件。这让大多数测试快速、无网络，同时没有假装文件系统不存在。序列化格式和 TUI 又各自使用 JSONL fixture 与 headless terminal，避免一个万能 harness 吞掉所有边界。

真实模型测试被保留在仓库中，但常规测试入口主动移除凭据。这样的默认值适合开发主循环：本地回归可重复，线上兼容性由显式 e2e 负责。源码修改时，选择哪一层测试，本身就是对模块边界和状态所有权的一次判断。
