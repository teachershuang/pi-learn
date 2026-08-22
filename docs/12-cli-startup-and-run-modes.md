# 第 12 讲：CLI 启动与运行模式

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的四种运行模式共用同一套会话和 Agent 状态。CLI 先确定有效工作目录、会话和 cwd 绑定的服务，再给 `AgentSessionRuntime` 接上不同的输入输出端：

```text
argv + TTY 状态 + stdin
          │
          ▼
       main()
          ├── 选择 SessionManager 与有效 cwd
          ├── 创建 settings / resources / models
          ├── 创建 AgentSessionRuntime
          ▼
   ┌─────────────┬─────────────┬─────────────┐
interactive    print/text    print/json      RPC
 TUI 循环       最终文本       事件 JSONL    命令/响应/事件 JSONL
```

这里容易出现两个概念混淆。源码中的 CLI `Mode` 只有 `text | json | rpc`，应用层 `AppMode` 才有 `interactive | print | json | rpc`。另外，JSON 与 RPC 都使用 JSONL，但 JSON 是单向事件输出，RPC 会持续读取命令。

## 1. 进程入口只做进程级初始化

### 解决的问题

网络 SDK 和 CLI 主流程加载前，需要先设置进程标识、全局 HTTP dispatcher 和警告策略。这些动作属于整个进程，不能等到某个 session 创建后才执行。

### 源码入口与状态变化

可执行入口是 `packages/coding-agent/src/cli.ts:1-20`：

```ts
import { APP_NAME } from "./config.ts";
import { configureHttpDispatcher } from "./core/http-dispatcher.ts";
import { main } from "./main.ts";

process.title = APP_NAME;
process.env.PI_CODING_AGENT = "true";
process.emitWarning = (() => {}) as typeof process.emitWarning;

configureHttpDispatcher();

main(process.argv.slice(2));
```

入口没有创建 Agent。它先标记当前进程属于 coding-agent，屏蔽 Node warning 输出，并在 provider SDK 发请求前装好 undici dispatcher。`main()` 读取 settings 后还会再次应用代理与 idle timeout，位置在 `packages/coding-agent/src/main.ts:474-487` 和 `734-737`。

### 设计取舍

将 `main()` 写成接收 `string[]` 和可选 extension factories 的函数，CLI 包装层很薄，测试或嵌入代码可直接传参。进程级副作用仍留在 `cli.ts`，避免导入 `main.ts` 就修改标题或全局 warning handler。

## 2. 参数解析先保留信息，语义校验随后进行

### 解析规则

`packages/coding-agent/src/cli/args.ts:63-210` 的 `parseArgs()` 做一次线性扫描。已知长参数写入 `Args`，普通字符串进入 `messages`，`@path` 进入 `fileArgs`。未知长参数暂存到 `unknownFlags`，因为扩展可能在资源加载后才注册它；未知短参数没有扩展命名空间，直接形成 error diagnostic。

```ts
} else if (arg.startsWith("@")) {
	result.fileArgs.push(arg.slice(1));
} else if (arg.startsWith("--")) {
	const eqIndex = arg.indexOf("=");
	if (eqIndex !== -1) {
		result.unknownFlags.set(arg.slice(2, eqIndex), arg.slice(eqIndex + 1));
	} else {
		const flagName = arg.slice(2);
		const next = args[i + 1];
		if (next !== undefined && !next.startsWith("-") && !next.startsWith("@")) {
			result.unknownFlags.set(flagName, next);
			i++;
		} else {
			result.unknownFlags.set(flagName, true);
		}
	}
} else if (arg.startsWith("-") && !arg.startsWith("--")) {
	result.diagnostics.push({ type: "error", message: `Unknown option: ${arg}` });
}
```

代码位于 `packages/coding-agent/src/cli/args.ts:181-205`。`--thinking` 的非法值只产生 warning，不会终止启动；缺失 `--name` 值和未知短参数属于 error。`main()` 在 `packages/coding-agent/src/main.ts:508-520` 统一打印 diagnostics，只要其中有 error 就以状态 1 退出。

`-p` 有一条专门规则：后面的普通值可立即作为 prompt，但下一个 option 不能被吃掉。以 `---` 开头的 YAML frontmatter 仍按 prompt 处理。`packages/coding-agent/test/args.test.ts:25-51` 固定了这几个边界。

### 延迟校验的代价

扩展 flags 只有加载扩展后才能核对。`packages/coding-agent/src/core/agent-session-services.ts:76-120` 将 CLI 暂存值匹配到扩展注册表；未知 flag 或 string flag 缺值会成为 runtime error diagnostic。这样能让扩展扩充 CLI，又意味着拼错的未知长参数要到服务创建阶段才报错。

## 3. TTY 状态决定默认运行模式

### 决策表

`packages/coding-agent/src/main.ts:99-110` 的顺序很短，却决定了后续 stdin 与 stdout 的所有权：

```ts
function resolveAppMode(parsed: Args, stdinIsTTY: boolean, stdoutIsTTY: boolean): AppMode {
	if (parsed.mode === "rpc") return "rpc";
	if (parsed.mode === "json") return "json";
	if (parsed.print || !stdinIsTTY || !stdoutIsTTY) return "print";
	return "interactive";
}
```

具体结果如下：

| 条件 | AppMode | 输入 | 标准输出 |
|---|---|---|---|
| `--mode rpc` | RPC | stdin JSONL 命令 | 响应、事件 JSONL |
| `--mode json` | JSON | CLI prompt 或管道文本 | session header 与事件 JSONL |
| `-p` / `--print` | print | CLI prompt 或管道文本 | 最后一条 assistant 的文本块 |
| stdin 或 stdout 不是 TTY | print | 管道或参数 | 最终文本 |
| 其余情况 | interactive | TUI 编辑器 | TUI 渲染 |

`--mode text` 没有强制 print；在双 TTY 环境中仍会进入 interactive。它只在已经由 `-p` 或非 TTY 条件选中 print 时表示文本输出。这个行为来自当前分支判断，不宜把帮助文本中的“text default”理解成独立的第五种启动模式。

### stdin 的所有权

`packages/coding-agent/src/main.ts:55-72` 只在 stdin 非 TTY 时读完整管道。RPC 跳过这一步，因为 stdin 必须留给命令协议；同理，`@file` 在 RPC 启动时被拒绝，见 `packages/coding-agent/src/main.ts:552-555`。

`packages/coding-agent/src/cli/initial-message.ts:21-43` 按顺序合并管道内容、`@file` 文本和第一个 CLI message。剩余 messages 之后逐条提交。合并使用空字符串连接，文件处理器自行在包装文本中加入换行；调用者不能假设这里会自动插入段落分隔符。

图片文件成为 `initialImages`，文本文件包进 `<file name="...">`。文件不存在、读取失败或处理图片失败时，决策者是文件处理层：前两者直接退出，图片处理失败则把失败说明作为文件文本交给模型。入口在 `packages/coding-agent/src/cli/file-processor.ts:25-87`。

## 4. 会话选择先于 cwd 绑定服务

### 为什么先选会话

一个 session header 记录创建时的 cwd。恢复其他项目的会话后，project settings、扩展、模型注册和系统提示都应从目标 cwd 重新发现；若启动时先用 `process.cwd()` 创建服务，再打开 session，模型上下文和工具执行目录会分裂。

`packages/coding-agent/src/main.ts:577-616` 先解析 `sessionDir`，创建 `SessionManager`，再读取 `sessionManager.getCwd()`。运行时服务直到目标 cwd 确定后才创建。

### 会话参数的优先级

`packages/coding-agent/src/main.ts:256-354` 的 `createSessionManager()` 按以下顺序决策：

1. `--no-session`、help、list-models 使用内存 session；只读命令不会预占 session ID；
2. `--fork` 查找源 session，并在当前 cwd 建新会话；
3. `--session` 打开明确路径或匹配 ID；全局匹配到其他项目时询问是否 fork 到当前 cwd；
4. `--resume` 打开会话选择器选中的文件；
5. `--continue` 恢复当前项目最近会话；
6. `--session-id` 精确打开当前项目同 ID 会话，找不到则告警后创建；
7. 没有选择参数时创建新 session。

`--fork` 不能与 `--session`、`--continue`、`--resume` 或 `--no-session` 合用。`--session-id` 不能与前三种恢复参数合用，但可作为 fork 的目标 ID；目标已存在会失败。校验位于 `packages/coding-agent/src/main.ts:192-249`，行为测试见 `packages/coding-agent/test/session-id-readonly.test.ts:82-154`。

### cwd 缺失

保存的 cwd 已被删除时，`packages/coding-agent/src/main.ts:589-605` 按模式处理：interactive 给出“使用当前 cwd 继续”或取消的选择；print、JSON、RPC 没有可交互确认界面，直接报告 `MissingSessionCwdError` 并退出。`packages/coding-agent/test/session-cwd.test.ts:34-91` 还验证 runtime 创建前必须拦截这个错误。

这项检查不能只在工具执行前补救。settings、资源和 provider 注册本身都依赖 cwd，错误目录下创建出的 runtime 已经不可信。

## 5. 服务创建分成基础设施与会话两步

### 调用链

确定 session cwd 后，`main()` 构造一个可重复使用的 runtime factory。核心链条是：

```text
createAgentSessionServices(target cwd)
  ├── SettingsManager
  ├── ResourceLoader.reload()
  ├── 扩展 provider 注册
  └── ModelRuntime.refresh()

resolve model / thinking / tools against those services

createAgentSessionFromServices()
  └── createAgentSession()

createAgentSessionRuntime()
  └── AgentSessionRuntime(session + services + factory)
```

实现分别位于 `packages/coding-agent/src/core/agent-session-services.ts:122-207`、`packages/coding-agent/src/main.ts:649-731` 和 `packages/coding-agent/src/core/agent-session-runtime.ts:409-433`。

服务层收集 diagnostics，不自行打印或退出。应用层在 `packages/coding-agent/src/main.ts:758-768` 打印结果；任何 error 都会终止启动，扩展加载错误还会提示 `pi -ne`。这种边界让 SDK 调用者可以采用自己的错误呈现方式。

### 固定输入与 cwd 输入

CLI 传入的 extension、skill、prompt template 和 theme 路径在启动 cwd 下先解析成绝对路径，见 `packages/coding-agent/src/main.ts:628-631`。后续切换到另一个 session cwd 时，这些显式路径不会被重新解释；项目自动发现的资源则跟随新 cwd 重建。

这是一个有意的区分：命令行路径表达调用者当时指定的对象，项目资源表达当前 session 所属项目。两者都跟随 cwd 会让相同命令在 resume 后指向另一个文件。

## 6. `AgentSessionRuntime` 是四种模式共享的可替换状态

### 状态所有权

`packages/coding-agent/src/core/agent-session-runtime.ts:59-115` 保存当前 `AgentSession`、与其 cwd 一致的 services、创建 factory、diagnostics 和模型回退说明。模式层拿到的是 runtime host，不直接把初始 session 永久缓存。

切换会话的主线位于 `packages/coding-agent/src/core/agent-session-runtime.ts:155-191`：

```ts
const sessionManager = SessionManager.open(sessionPath, undefined, options?.cwdOverride);
assertSessionCwdExists(sessionManager, this.cwd);
await this.teardownCurrent("resume", sessionManager.getSessionFile());
this.apply(
	await this.createRuntime({
		cwd: sessionManager.getCwd(),
		agentDir: this.services.agentDir,
		sessionManager,
		sessionStartEvent: { type: "session_start", reason: "resume", previousSessionFile },
	}),
);
await this.finishSessionReplacement(options?.withSession);
```

旧 session 先发 `session_shutdown` 并 dispose，新 runtime 按目标 cwd 重建，最后调用模式注册的 rebind callback。interactive 在构造函数中注册重绑，print 与 RPC 在启动时注册。重绑会重新绑定扩展命令上下文、替换事件订阅，并让模式引用新的 session。

`packages/coding-agent/test/suite/agent-session-runtime.test.ts:451-520` 验证跨 cwd 切换后 `runtime.cwd` 与 session manager 同步变化。

### 失败路径

源码明确采用“先拆旧 runtime，再建新 runtime”。新 runtime 创建失败时，异常交给模式层处理，没有自动回滚到旧 session。interactive 将这类错误视为 fatal runtime error；RPC/print 返回协议错误或退出失败。替代方案是先并行建好新 runtime 再销毁旧对象，但两个 cwd 的扩展、终端 UI 和进程资源会短暂共存，清理顺序更难保证。

## 7. interactive：事件驱动的 TUI 长循环

### 初始化与运行

`packages/coding-agent/src/modes/interactive/interactive-mode.ts:441-485` 的构造函数建立 TUI 容器并注册 session rebind。`init()` 位于 `672-823`，先启动终端 UI，再绑定扩展；这保证 `session_start` 扩展可以使用交互式对话框。随后才渲染恢复消息，加载资源说明不会被历史消息顶到前面。

`run()` 位于 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:825-906`。它处理初始 prompts，然后进入无限输入循环：

```ts
while (true) {
	const userInput = await this.getUserInput();
	try {
		await this.session.prompt(userInput);
	} catch (error: unknown) {
		const errorMessage = error instanceof Error ? error.message : "Unknown error occurred";
		this.showError(errorMessage);
	}
}
```

模式订阅 `AgentSessionEvent`，入口在 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2807-2815`。UI 只是状态投影：AgentSession 仍拥有消息、队列、模型、压缩和工具执行状态。prompt 失败会显示错误并回到输入循环，不会像单次命令那样自动以非零状态退出。

### TUI 专属能力

扩展绑定使用 mode `tui`，可获得组件、编辑器、主题和对话框能力，见 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:1605-1681`。这些能力属于交互宿主，不是 AgentSession 的固有能力。

## 8. print 与 JSON：同一单次执行器的两种输出

### 共同流程

`packages/coding-agent/src/modes/print-mode.ts:30-159` 的 `runPrintMode()` 依次提交 initial message 和剩余 messages，最后 dispose runtime 并刷新受控 stdout。运行中发生 session replacement 时，它重新绑定扩展和事件订阅。

text 模式只检查所有 prompt 完成后的最后一条消息，并输出其中的 text blocks。thinking、tool call 和中间轮次不会写到 stdout。最后消息为 error/aborted 时输出错误并返回状态 1；抛出的异常也归一为状态 1。

JSON 模式先输出 session header，再把每个 session event 序列化为一行 JSON。它不会读取后续命令，也不会等待客户端响应。`packages/coding-agent/test/print-mode.test.ts:93-142` 验证两种输出都会在 finally 路径触发 `session_shutdown`，assistant error 会返回非零状态。

### 多条 message 的语义

多个 CLI messages 是连续多轮 prompt，而不是一次 prompt 的多段文本。只有第一条会与 stdin、`@file` 合成 initial message；后面的字符串逐条调用 `session.prompt()`。text 最终只打印最后一轮回答，JSON 则能观察全部轮次事件。

## 9. RPC：命令响应与 Agent 事件并行

### 协议边界

`packages/coding-agent/src/modes/rpc/rpc-mode.ts:43-795` 启动一个不自行结束的进程。stdin 按 JSONL 拆行，stdout 包含三类记录：

- 带可选 `id` 的 command response；
- `AgentSessionEvent`；
- extension UI request。

`packages/coding-agent/src/modes/rpc/jsonl.ts:1-58` 只用 LF 分帧，兼容 CRLF，并会处理没有末尾换行的最后一条。Unicode U+2028/U+2029 保留在 JSON 字符串内部，不作为行边界，测试见 `packages/coding-agent/test/rpc-jsonl.test.ts:1-58`。

### prompt response 不表示生成完成

`packages/coding-agent/src/modes/rpc/rpc-mode.ts:385-415` 启动 prompt 后，在 preflight 成功时立即回一条 success response。模型生成、工具执行和 settlement 继续通过事件流报告。认证等 preflight 失败则只回一条 failure response。这样客户端能区分“命令已被接受”和“整个 Agent turn 已结束”。

流式期间以 `streamingBehavior` 提交的 prompt 可以入队，同样算命令成功。`packages/coding-agent/test/rpc-prompt-response-semantics.test.ts:187-280` 验证 preflight 失败、立即执行和排队三条路径都只产生一次相关 response。

其他命令由 `handleCommand()` 调用当前 session 或 runtime host，包括模型切换、压缩、bash、session switch 和 fork。未知命令、JSON 解析错误、目标 entry 不存在都会变成结构化 failure response；进程不会因单条坏命令退出。

### 扩展 UI 与背压

RPC 把 select、confirm、input、editor 等请求转成 `extension_ui_request`，客户端再发 `extension_ui_response`。TUI 组件、footer、header、主题切换等能力无法跨这个协议复用，会返回不支持或空实现，源码在 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:91-288`。

每个 session event 写入 raw stdout 后，Agent 的异步订阅者会等待输出队列背压，见 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:350-363`。慢客户端因此会拖慢事件推进，而不是无限积累未写出的 JSON；这是以吞吐换内存和事件顺序的选择。

## 10. stdout 保护把协议输出与运行日志分开

### 触发位置

非 interactive 模式在迁移、资源加载和扩展初始化前调用 `takeOverStdout()`，位置在 `packages/coding-agent/src/main.ts:546-550`。普通 `--help`、`--list-models` 和提前返回的 `--version` 保留传统 stdout；显式 `--mode json --help` 或 `-p --help` 则启用保护。

### 实现

`packages/coding-agent/src/core/output-guard.ts:39-74` 保存原始 stdout writer，然后把全局 `process.stdout.write` 改接 stderr：

```ts
const rawStdoutWrite = process.stdout.write.bind(process.stdout);
const rawStderrWrite = process.stderr.write.bind(process.stderr);

process.stdout.write = ((chunk, encodingOrCallback, callback): boolean => {
	if (typeof encodingOrCallback === "function") {
		return rawStderrWrite(String(chunk), encodingOrCallback);
	}
	return rawStderrWrite(String(chunk), callback);
}) as typeof process.stdout.write;
```

模式层只有通过 `writeRawStdout()` 才能写回真正 stdout。该函数把写操作串在 promise tail 上，保持记录顺序；遇到 `ENOBUFS`、`EAGAIN` 或 `EWOULDBLOCK` 会短暂重试。其他写错误最终以状态 1 终止进程，代码在 `packages/coding-agent/src/core/output-guard.ts:17-37` 和 `85-108`。

这道保护针对的不是 console API 本身。扩展、包安装器或依赖即使直接 `console.log()`，内容也会转到 stderr，避免破坏 JSONL 或供 shell 捕获的最终文本。`packages/coding-agent/test/stdout-cleanliness.test.ts:68-108` 验证 JSON/help 和 print/help 的启动杂音不会进入 stdout。

## 11. 退出与清理

print 在 `finally` 中移除 signal handler、dispose runtime，并等待 raw stdout 清空。assistant error 与普通异常返回 1；SIGTERM 和非 Windows 的 SIGHUP 会先清理分离子进程，再以 143 或 129 退出，见 `packages/coding-agent/src/modes/print-mode.ts:39-64`。

RPC 收到 stdin end 会正常 shutdown；SIGTERM/SIGHUP 也会清理子进程和 runtime。shutdown 会取消订阅、发 session shutdown、暂停 stdin，并在适合的信号路径刷新 stdout，入口在 `packages/coding-agent/src/modes/rpc/rpc-mode.ts:697-789`。客户端子进程意外退出时，正在等待的请求不能永久悬挂，`packages/coding-agent/test/rpc-client-process-exit.test.ts:24-38` 验证它会以退出码拒绝 promise。

interactive 退出还要恢复终端 raw mode、光标和 TUI 组件，因此由 InteractiveMode 自己排序清理。本讲只保留这个边界；终端状态和差分渲染放到第 21 讲。

## 12. 本讲形成的边界

- CLI 先确定 session 与有效 cwd，再创建 project-local services；`process.cwd()` 只是起点。
- interactive、print、JSON、RPC 共享 `AgentSessionRuntime`，差异主要在输入、输出、扩展 UI 和进程寿命。
- JSON 是单向的 print 事件流；RPC 是双向命令协议，prompt response 只确认 preflight。
- session replacement 会重建 cwd 绑定服务，模式必须重新绑定当前 session。
- stdout takeover 将任意启动日志导向 stderr，raw writer 独占机器可读输出。
- 单次模式以退出码表达失败；interactive 把可恢复错误留在 TUI；RPC 把单条命令错误留在结构化响应中。

下一讲沿着 runtime 创建过程进入资源发现：`AGENTS.md`、skills、prompt templates、themes 和 `.pi` 配置怎样合并，以及 reload 后哪些状态会变化。
