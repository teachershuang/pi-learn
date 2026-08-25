# 第 15 讲：扩展系统

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的扩展不是一个统一的 middleware 栈。它由三层组成：资源层决定哪些脚本有资格执行，加载层把 TypeScript/JavaScript 模块运行成注册表，`ExtensionRunner` 再按不同事件的协议调度这些注册项。工具、命令和 UI 是注册表中的不同能力；观察、改写、接管、阻断与取消则是不同事件各自定义的控制语义。

```text
settings / package / CLI extension paths
                 │  project trust 过滤、去重和排序
                 ▼
          DefaultResourceLoader
                 │  jiti.import() + factory(pi)
                 ▼
Extension[] ── handlers / tools / commands / shortcuts / flags / renderers
                 │
                 ▼
          ExtensionRunner
       ┌─────────┼───────────┐
       │         │           │
    事件调度    注册表合并    ExtensionContext
       │         │           │
       └─────────┴───── AgentSession 绑定运行时动作与 UI
```

这套结构的关键分界是：扩展 factory 负责声明能力，`ExtensionRunner` 负责解释冲突和事件返回值，`AgentSession` 才拥有模型、工具、会话与运行模式的真实状态。

## 1. 发现扩展之前，先决定项目代码能否执行

### 解决的问题

扩展是进程内代码，不是静态配置。项目目录中的扩展可以读写文件、访问环境变量和网络，也可以注册工具供模型调用。Pi 因此不能在询问项目是否可信之前就执行项目扩展。

主路径在 `packages/coding-agent/src/core/resource-loader.ts:331-404`。需要解析项目信任时，`DefaultResourceLoader.reload()` 先强制设置 `projectTrusted=false`，只装载用户级、全局级和 CLI 临时扩展；这些扩展可以处理 `project_trust`。得到信任结论后，settings 与 package resources 才按最终状态重载，项目扩展才可能进入最终集合。

```ts
this.settingsManager.setProjectTrusted(false);
await this.settingsManager.reload();
return this.loadCurrentExtensionSet({ includeInlineFactories: true });

// reload 中的第二阶段
preTrustExtensions = await this.loadProjectTrustExtensions();
const projectTrusted = await options.resolveProjectTrust({
	extensionsResult: preTrustExtensions,
});
this.settingsManager.setProjectTrusted(projectTrusted);

await this.settingsManager.reload();
const resolvedPaths = await this.packageManager.resolve();
```

`project_trust` 不是投票。`ExtensionRunner.emitProjectTrustEvent()` 按扩展及其 handler 的注册顺序执行，`undecided` 表示交给后续处理者；第一个明确的 `yes` 或 `no` 立即结束扩展决策。handler 抛错会被记录，后续 handler 仍可决定。若所有扩展都不决定，才回到内置的信任解析流程。

### 路径来源与顺序

最终路径不是由 `loader.ts` 自己扫描整个工作区得来。`DefaultResourceLoader` 先让 `PackageManager` 解析已启用的包与自动资源，再解析 CLI 附加路径，随后用 `mergePaths(cliEnabledExtensions, enabledExtensions)` 合并。显式 CLI 扩展排在已发现扩展之前，物理路径会规范化并去重。

预信任阶段已经装载成功的扩展不会在同一轮信任解析后再执行一次 factory。`loadFinalExtensionSet()` 用 resolved path 复用这些实例，只加载剩余的项目扩展，最后再按最终 `extensionPaths` 恢复顺序，见 `packages/coding-agent/src/core/resource-loader.ts:517-567`。这避免了全局扩展在一次启动中重复注册或重复创建资源。

### 目录发现的边界

`discoverExtensions()` 位于 `packages/coding-agent/src/core/extensions/loader.ts:548-607`，它只处理传入的 extensions 目录：

- 目录下的 `.ts`、`.js` 文件直接成为入口；
- 一级子目录若有 `package.json` 的 `pi.extensions`，使用清单列出的一个或多个入口；
- 否则尝试子目录的 `index.ts`，再尝试 `index.js`；
- 不递归到第二层；清单指向的路径不存在时跳过。

这只是目录到入口文件的发现算法。现代启动主链仍由 `PackageManager` 和 `DefaultResourceLoader` 决定应扫描哪些位置、哪些资源已启用以及项目是否可信。把 `discoverAndLoadExtensions()` 当成完整启动入口，会漏掉 package、CLI、trust 和资源元数据这些上层规则。

## 2. 加载结果不是“模块对象”，而是一组注册项

### TypeScript 如何执行

`loadExtensionModule()` 使用 `jiti` 动态导入 `.ts` 或 `.js`，见 `packages/coding-agent/src/core/extensions/loader.ts:163-220`。`moduleCache:false` 关闭 jiti 自身的长期模块缓存；源码运行、Node 开发环境与 Bun 单文件构建则分别使用虚拟模块或 alias，让扩展都能导入公开的 coding-agent、agent、ai、tui 包名。旧的 `@mariozechner/*` 名称也有兼容 alias。

```ts
const jiti = createJiti(import.meta.url, {
	moduleCache: false,
	fsCache: false,
	interopDefault: true,
	tryNative: false,
	alias: createJitiAliases(),
});

const module = await jiti.import(resolvedPath, { default: true });
const factory = extractFactory(module);
if (!factory) {
	throw new Error(`Extension must export a default function: ${extensionPath}`);
}
```

默认导出必须是 `ExtensionFactory`。`loadExtension()` 先创建空的 `Extension` 数据结构，再执行 `await factory(api)`。factory 调用 `pi.on()`、`registerTool()`、`registerCommand()` 等方法，把 handler 和定义写入该扩展自己的 Map；factory 的返回值不承担运行时协议。

### 缓存缓存了什么

ResourceLoader 主路径使用 `loadExtensionsCached()`。缓存键是解析后的扩展路径，条目还记录 cwd 和 generation；命中时复用模块的 factory 函数，但每次仍创建新的 `Extension` 并重新执行 factory。直接调用 `loadExtensions()` 则不使用这层 factory cache。

reload 时，如果 ResourceLoader 已经加载过，会先调用 `clearExtensionCache()`：清空 factory cache 并增加 generation。`packages/coding-agent/test/suite/regressions/extension-factory-cache.test.ts:69-125` 分别验证了同 cwd 缓存、直接加载不缓存、reload 重新加载模块，以及缓存不会跨 cwd 复用。

这里有两种不同状态：

- 模块顶层状态存在于本次模块求值结果中；reload 后模块重新求值；
- factory 闭包和注册表属于本次扩展实例；即使 factory 来自缓存，factory 也会再次执行。

### 初始化失败

模块语法错误、没有默认函数、依赖解析失败或 factory 抛错，都在 `loadExtension()` 的同一个 `try/catch` 中变成 `{path, error}`。失败扩展不进入 `extensions`，其他入口继续加载。这里的隔离单位是“一个入口文件”，不是入口内的某一项注册：factory 注册了一半再抛错时，整个扩展实例仍被丢弃。

factory 执行时，扩展尚未与 session 绑定。注册方法可用，provider 注册会先排队；依赖真实 session 的动作最初是抛错 stub。扩展如果在模块初始化阶段立即调用 `sendMessage()`、`setModel()` 等动作，会加载失败。这个限制让“声明能力”和“使用当前会话”分开：前者在 factory 中完成，后者应在事件、命令或工具回调中完成。

## 3. 注册表保留来源，冲突规则按能力分别定义

### 每个扩展先拥有独立 Map

`createExtensionAPI()` 位于 `packages/coding-agent/src/core/extensions/loader.ts:295-453`。每次注册都会写进当前 `Extension` 的 Map，并附带由入口路径生成的 `sourceInfo`。所有 API 先执行 `runtime.assertActive()`，从而给被作废的 runtime 留出统一拒绝入口。

`ExtensionRunner` 再遍历有序的 `Extension[]` 合并结果。冲突不是一个全局的“后者覆盖前者”规则：

| 注册项 | 扩展之间的冲突 | 与内置能力的关系 |
|---|---|---|
| tool | 同名时第一个扩展注册项保留 | 进入 AgentSession 执行 registry 后，扩展同名工具覆盖内置工具 |
| command | 都保留，重名者变成 `name:1`、`name:2` | 扩展命令在 prompt 入口先于 skill/template 展开检查 |
| flag | 同名时第一个保留 | 解析后的值存放在共享 runtime |
| renderer | 按扩展顺序取第一个匹配 custom type 的 renderer | 无默认 renderer 时回退到通用显示 |
| shortcut | 禁止覆盖受保护的内置 action；可覆盖项会告警 | 多扩展冲突时后加载者生效并告警 |

ResourceLoader 还会把 tool、command、flag 的扩展间重名写成 diagnostics，但不会因为冲突卸载扩展。诊断与实际 precedence 分属两层：前者提醒配置问题，后者保证运行结果确定。

### 工具注册后的两次合并

`ExtensionRunner.getAllRegisteredTools()` 首先处理扩展之间的同名项，只保留第一个。`AgentSession._refreshToolRegistry()` 随后先放内置工具，再逐项 `set()` 扩展工具，因此胜出的扩展工具可以替换同名内置工具，见 `packages/coding-agent/src/core/agent-session.ts:2492-2520`。

工具是否注册和是否活动仍是两件事。registerTool 在 runtime 已绑定后会触发 refresh；新工具进入 registry，但下一轮发给模型的集合仍受 allowlist、exclude list 和 active tool names 控制。工具执行包装器还比较执行前后的活动列表：若执行期间只新增了工具，会把 `addedToolNames` 带回 agent loop，支持原生 deferred tool loading；如果同时移除了工具，就不把这次变化表示成单纯新增。

### 命令不是模型工具

扩展命令由 `AgentSession._tryExecuteExtensionCommand()` 直接查找并执行，入口位于 `packages/coding-agent/src/core/agent-session.ts:1258-1281`。prompt 以 `/` 开头且命中扩展命令时，handler 在 skill/template expansion 和 `input` 事件之前运行；即使 agent 正在 streaming，也不会把这段文本排进 steer/follow-up 队列。

handler 收到的是 `ExtensionCommandContext`，比普通事件和工具使用的 `ExtensionContext` 多出 `waitForIdle()`、`newSession()`、`fork()`、`switchSession()`、`navigateTree()` 和 `reload()`。命令异常会被报告为 extension error，并被视为已经处理：原始 `/command` 不会继续发给模型。

## 4. ExtensionContext 是延迟读取的运行时视图

### 为什么不直接传 AgentSession

事件、工具和快捷键需要 cwd、model、abort signal、UI 与少量会话动作，但不应直接依赖整个 `AgentSession`。`ExtensionRunner.createContext()` 返回带 getter 和闭包方法的受限视图，见 `packages/coding-agent/src/core/extensions/runner.ts:636-708`。

这些值在访问时才读取，不在 context 创建时拍快照。例如 `ctx.model` 读取 runner 当前绑定的 model；`ctx.signal` 返回当前 agent run 的 AbortSignal；`ctx.getSystemPrompt()` 在 `before_agent_start` reducer 中能看到前一个 handler 已修改的 system prompt。

`createCommandContext()` 特意用 property descriptors 复制 getter，而不是对象展开。对象展开会立即读取 getter，把旧值冻结到新对象，并绕过之后的 stale 检查。这个细节说明 context 的主要职责不只是少暴露几个方法，还要维持当前 runtime 的生命周期边界。

### UI 由运行模式绑定

扩展 API 只定义 `ExtensionUIContext`，实际能力由运行模式注入：

- interactive/TUI 绑定完整终端 UI，可显示 select、confirm、input、editor、widget、status、自定义 component 与 editor；
- RPC 通过协议桥提供对话和通知，但 TUI `custom()` 不可用；
- JSON 与 print 模式使用 no-op UI，`hasUI=false`，选择和输入返回 `undefined`，confirm 返回 `false`，notify/status/widget 不产生可见效果。

因此“扩展加载成功”不等于“扩展可以弹窗”。扩展必须根据 `ctx.hasUI` 和 `ctx.mode` 决定是否走交互路径，并给无 UI 模式提供明确的非交互结果。UI renderer 也只负责显示：custom entry 是否进入模型上下文，由 entry/message 类型本身决定，不由 renderer 决定。

## 5. 事件不是同一种 hook：逐项看控制权

所有 handler 都按扩展加载顺序、再按该扩展内注册顺序串行调用。相同的执行顺序之上，Pi 定义了六类返回语义。

### 5.1 观察事件：返回值没有控制作用

`session_start`、`session_info_changed`、`session_compact`、`session_shutdown`、`session_tree`、`agent_start`、`agent_end`、`agent_settled`、`turn_start`、`turn_end`、`message_start`、`message_update`、`tool_execution_start/update/end`、`model_select`、`thinking_level_select` 和 `after_provider_response` 属于观察事件。

它们暴露已经发生或正在发生的阶段，适合统计、清理和 UI 同步。handler 抛错时，通用 `emit()` 记录 extension path、event、message 和 stack，然后继续调用后续 handler；异常不会撤销已经发生的状态，也不会停止 agent 主链。

`session_shutdown` 值得单独理解：它给扩展释放 timer、文件 watcher、socket 等长期资源的机会，但不是 veto 点。需要取消会话切换或压缩，应使用对应的 `session_before_*` 事件。

### 5.2 reducer：前一个结果成为后一个输入

以下事件支持串联改写：

- `context`：在每次 LLM 调用前改写 `AgentMessage[]`；初值先 `structuredClone()`，避免直接改写 Agent 持有的消息数组；
- `before_provider_request`：前一个 handler 返回的 payload 交给后一个；返回 `undefined` 表示保留当前值；
- `before_agent_start`：system prompt 逐项替换，返回的 custom message 逐项累积；
- `message_end`：替换最终消息，但角色必须与原消息相同；角色不符只报错，不采用结果；
- `tool_result`：按字段合并 `content`、`details`、`isError`，省略字段保留当前值；
- `input`：`transform` 后的 text/images 交给后一个 handler，未提供 images 时保留现值。

`before_provider_headers` 是一个特殊的原地 reducer。所有 handler 共享同一个 headers 对象，通过写属性修改 header，以 `null` 删除 header；返回值被忽略。

这些 reducer 的 handler 异常都采用隔离继续：保留异常发生前已经得到的 current value，记录错误，然后交给后续 handler。它们不是事务；前面的修改不会因为后面失败而回滚。

### 5.3 聚合：收集所有扩展的贡献

`resources_discover` 收集所有 handler 返回的 `skillPaths`、`promptPaths` 和 `themePaths`，而不是让后者覆盖前者。AgentSession 在 `session_start` 后发出该事件，把结果作为 temporary source 扩展到 ResourceLoader，然后重建 system prompt。handler 失败只丢失该 handler 的贡献。

### 5.4 首个接管：第一个明确结果结束选择

`project_trust` 的第一个 `yes/no` 获得决定权；`user_bash` 的第一个非空结果获得 `!`/`!!` 执行权。后者可以提供自定义 `BashOperations`，也可以直接提供完整 `BashResult`。抛错会记录后继续寻找下一个接管者。

`input` 的 `{action:"handled"}` 也会短路后续 handler，但它与 `user_bash` 略有不同：在 handled 之前，各个 transform 已经形成新的 current input；handled 表示扩展已经完成反馈，整个 skill/template expansion 与 agent prompt 都被跳过。

### 5.5 阻断：只阻止本次工具执行

`tool_call` 在工具参数 schema 校验之后、工具 execute 之前触发。handler 可以原地修改 `event.input`，后续 handler 和实际执行看到修改后的对象；Pi 不会在修改后再次校验 schema。返回 `{block:true, reason}` 会立即停止余下的 `tool_call` handler，并为本次调用生成 `isError=true` 的 tool result。

```ts
const preparedToolCall = prepareToolCallArguments(tool, toolCall);
const validatedArgs = validateToolArguments(tool, preparedToolCall);
const beforeResult = await config.beforeToolCall({
	assistantMessage,
	toolCall,
	args: validatedArgs,
	context: currentContext,
}, signal);
if (beforeResult?.block) {
	return {
		kind: "immediate",
		result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
		isError: true,
	};
}
```

低层位置在 `packages/agent/src/agent-loop.ts:618-664`。注意校验发生在扩展修改之前；这给扩展修补命令或路径的能力，也把“修改后仍符合工具约定”的责任交给扩展。

`ExtensionRunner.emitToolCall()` 是异常策略的例外：它没有包裹 handler 的 `try/catch`。异常穿过 AgentSession 的 beforeToolCall 适配层，最后由低层 `prepareToolCall()` 捕获并转换为失败 tool result。结果仍是“本次工具不执行”，但 reason 来自异常消息。其他 sibling 工具是否继续，仍由 agent loop 的批量执行策略决定；这不是取消整个 agent run。

### 5.6 取消：阻止尚未提交的会话操作

`session_before_switch`、`session_before_fork`、`session_before_compact` 和 `session_before_tree` 可以返回 `cancel:true`。通用 `emit()` 遇到 cancel 会立即停止后续 handler，调用方据此不执行对应操作。

不取消时，后一个非空结果会覆盖前一个结果对象，而不是逐字段合并。因此多个扩展同时定制 compact/tree 时，最后返回的完整结果决定调用方看到的 customization；扩展不能假设自己的局部字段会与另一个扩展自动组合。compact 可直接提供 `CompactionResult`，tree 可提供 summary、instructions、replaceInstructions 和 label，fork 还可决定是否恢复 conversation。

阻断与取消的共同点是短路，边界不同：`tool_call` 阻断生成一个可见的失败工具结果，agent 仍可继续；`session_before_*` 取消的是尚未发生的会话状态变更，不会制造 tool result。

## 6. 两条关键调用链

### 用户输入到 agent

`AgentSession.prompt()` 中的扩展顺序是：

```text
/extension-command ? ──命中──► command handler，原输入结束
          │未命中
          ▼
input handlers ── transform ──► 下一个 input handler
          │handled                    │continue
          ▼                           ▼
        结束              skill / prompt template expansion
                                      │
                                      ▼
                         before_agent_start reducers
                                      │ custom messages 写入 session
                                      │ per-turn system prompt override
                                      ▼
                                  Agent.prompt()
```

`before_agent_start` 不是 `context` 的别名。前者每个用户 prompt 触发一次，可以为这一轮改 system prompt 并追加 custom message；后者在 agent loop 每次请求模型前触发，一轮内经过工具调用后可能再次执行，只改发往模型的消息投影。

### 模型工具调用到结果

```text
assistant tool call
  │
  ├─ tool_execution_start        观察
  ├─ schema validation
  ├─ tool_call                   修改 input / 阻断；异常也阻断
  ├─ tool.execute                可发 streaming update
  ├─ tool_result                 串联修改 content/details/isError
  ├─ tool_execution_end          观察最终状态
  └─ message_start/end           最终 ToolResultMessage
```

在默认并行模式中，同一 assistant message 的 sibling tool calls 先顺序完成 preflight，再并发 execute。`tool_call` 可以看到前一个 sibling 的参数修改决定，却不能依赖另一个 sibling 的执行结果已经写入 session。完成顺序可以影响 `tool_result`/`tool_execution_end` 的实时交错，最终 toolResult message 仍按 assistant 源顺序提交。并发层属于低层 agent loop，ExtensionRunner 本身没有把所有工具 handler 并行化。

## 7. 错误隔离不是“扩展错误都忽略”

扩展系统有四个不同的失败边界：

1. **加载失败**：当前入口不进入扩展集合，其他入口继续；错误作为 resource diagnostic 展示。
2. **普通事件/reducer 失败**：记录错误，保留当前状态，继续后续 handler 与主流程。
3. **命令失败**：记录错误，命令仍算已消费，不把命令文本交给模型。
4. **`tool_call` 失败**：失败关闭本次工具调用，产生 error tool result；工具 execute 不运行。

工具自身 `execute()` 抛错不由 ExtensionRunner 吞掉。低层 agent loop 将它归一化为 `isError=true` 的 tool result，再允许 `tool_result` handler 查看或改写。扩展甚至可以把 `isError` 改为 false，但这只改变交给后续系统和模型的结果语义，不能撤销已经发生的副作用。

UI 失败还要区分“无 UI 能力”和“UI handler 抛错”。print/json 模式的 no-op 返回值是正常模式行为，不应报告成扩展故障。真正的 handler 异常才进入 extension error listener。

## 8. reload 重建的是扩展 runtime，不重建会话历史

`AgentSession.reload()` 位于 `packages/coding-agent/src/core/agent-session.ts:2577-2599`：

```ts
const previousFlagValues = this._extensionRunner.getFlagValues();
await emitSessionShutdownEvent(this._extensionRunner, {
	type: "session_shutdown",
	reason: "reload",
});
await this.settingsManager.reload();
this.syncQueueModesFromSettings();
resetApiProviders();
await this._resourceLoader.reload();
this._buildRuntime({
	activeToolNames: this.getActiveToolNames(),
	flagValues: previousFlagValues,
	includeAllExtensionTools: true,
});
```

之后，已有运行模式 bindings 会重新绑定到新 runner，再发出 `session_start{reason:"reload"}` 和 `resources_discover{reason:"reload"}`。交互模式还在新 `session_start` 之前重建聊天显示，避免新扩展的通知插到旧资源视图前面。

reload 前后状态分为三类：

- **保留**：SessionManager 中的 JSONL 历史、当前模型与 agent messages、当前活动工具名称、已解析 flag 值；
- **重建**：settings/resources、扩展模块、factory 闭包、handlers/tools/commands/renderers、provider 注册、工具 registry、扩展贡献的 skills/prompts/themes；
- **由扩展自行持久化**：希望跨 reload 保留的业务状态，应写入 custom session entry 或外部存储，不能依赖模块变量。

`ctx.reload()` 在旧 command handler 的 call frame 内等待上述流程。await 返回后的代码仍是旧版本闭包，所以官方扩展文档要求把 reload 当成 handler 的终点：`await ctx.reload(); return;`。新事件、命令和工具会走新 runner。

### 旧上下文失效的源码边界

`ExtensionRunner.invalidate()` 和 `runtime.assertActive()` 提供 stale guard。会话被 new/resume/fork 替换时，旧 `AgentSession.dispose()` 明确调用 invalidate；回归测试也验证捕获的旧 `pi`/`ctx` 会抛错。

固定基线中，直接 `AgentSession.reload()` 会替换 `_extensionRunner`，但这段方法和 `_buildRuntime()` 没有显式对旧 runner 调用 `invalidate()`。因此能从源码确认的边界是：未来事件路由已经切到新 runner，官方契约也要求 reload 后不再使用旧对象；但“reload 后每个捕获的旧 `pi`/`ctx` 一定被 stale guard 拒绝”在这条直接路径上缺少与 session replacement 相同的实现与测试证据。这是当前基线的实现缺口判断，不应把错误信息中的概括文字当成已经验证的安全保证。

## 9. 设计取舍

### 进程内扩展换来完整能力，也扩大信任边界

扩展可以注册真正的 AgentTool、访问 SessionManager、执行进程命令并自定义 UI，能力远高于声明式插件。代价是没有进程隔离，也没有每项能力授权。project trust 只决定项目动态代码是否装载，不限制已获准扩展的文件、网络或凭据访问。sandbox 需要在进程、容器或工具实现层另行提供。

### 有序串行调度换来可解释的组合

reducer 按加载顺序串行运行，后一个总能看到前一个结果，诊断和复现都比较直接。代价是慢 handler 会增加关键路径延迟，尤其 `context`、provider request 和 `tool_call`。Pi 没有为 handler 统一设置超时；扩展应使用 `ctx.signal` 让自身的异步工作可中止。

### 每类事件单独定义控制语义

统一 middleware 接口看起来简单，却容易让“日志观察者是否能阻止工具”“结果修改失败是否应停止 agent”等问题变得含糊。Pi 让工具安全检查 fail-closed，让普通观察与展示 fail-open，让会话变更在提交前可取消，并让 payload/result 采用 reducer。这增加了 API 学习成本，但控制边界能够从事件类型和 result type 中读出来。

### 注册定义与运行时动作分离

factory 阶段只适合注册，绑定后回调才适合操作会话。这让同一个扩展定义能在 TUI、RPC、JSON、print 和 SDK 环境复用，也避免初始化顺序依赖尚未存在的 AgentSession。代价是扩展作者必须理解 factory 闭包、session 生命周期与 reload 之间的关系，长期资源还必须响应 `session_shutdown`。

## 10. 常见边界问题

### 扩展与 hook 是一回事吗？

不是。扩展是装载和注册单元；hook 是扩展可注册的一类能力。一个扩展还可以注册工具、命令、快捷键、flag、provider 和 renderer。事件本身也不存在统一 hook 语义，必须查看该事件的 result type 与 Runner dispatch 实现。

### `tool_call` 能修改参数，为什么还要自定义工具？

`tool_call` 适合审计、阻断或对已有工具做局部修补，不负责声明 schema、description、renderer 和 execute。新能力应注册工具；横切安全规则才适合拦截工具调用。参数修改后不再校验，也使大规模改写更容易制造运行时错误。

### `context` 修改会写回会话吗？

不会。Runner 先克隆 messages，改写结果只进入即将发生的模型请求。要留下可恢复的持久状态，应通过 message/session API 写入；要改变当前轮 system prompt，应使用 `before_agent_start`；三者生命周期不同。

### `tool_result` 把 `isError` 改成 false，工具就回滚成功了吗？

不会。它只改后续观察到的结果。工具已经执行，文件、子进程或网络副作用不会由结果 reducer 回滚。反过来，把成功结果标成错误也不会撤销副作用。

### reload 会重新加载代码，为什么还要 `session_shutdown`？

模块重载只让新 runtime 使用新代码，不会自动停止旧闭包创建的 watcher、timer 或 socket。旧扩展必须在 shutdown handler 中释放资源，否则这些对象仍可能存活并与新实例重复工作。

### 注册同名 command 和 tool，谁覆盖谁？

它们属于不同命名空间，不互相覆盖。command 由用户 `/name` 直接触发，重名 command 用后缀区分；tool 由模型按 tool schema 调用，同名扩展工具只保留加载顺序中的第一个，随后可以覆盖同名内置工具。

## 11. 从源码修改扩展系统时的验证面

扩展改动至少要按受影响语义选择测试，而不是只验证“脚本能加载”：

- 发现与初始化：`packages/coding-agent/test/extensions-discovery.test.ts`；
- 注册表冲突、context、reducer 与错误隔离：`packages/coding-agent/test/extensions-runner.test.ts`；
- input 的 transform/handled 短路：`packages/coding-agent/test/extensions-input-event.test.ts`；
- trust、最终排序、资源扩展和冲突 diagnostics：`packages/coding-agent/test/resource-loader.test.ts`；
- module/factory cache 与 reload：`packages/coding-agent/test/suite/regressions/extension-factory-cache.test.ts`；
- AgentSession 生命周期与 session replacement：`packages/coding-agent/test/agent-session-runtime-events.test.ts`；
- 动态工具进入下一轮：`packages/coding-agent/test/agent-session-dynamic-tools.test.ts`。

最容易漏测的是事件返回值的组合顺序。新增 hook 时，需要同时固定：触发位置、handler 顺序、输入是否复制、返回值如何组合、何时短路、异常是 fail-open 还是 fail-closed，以及状态变更已经发生还是尚未提交。只有事件名字和 TypeScript overload，不足以构成稳定的工程协议。
