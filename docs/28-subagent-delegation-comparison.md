# 第 28 讲：父 Agent 如何委派子任务

> Pi 上游仓库：`earendil-works/pi`<br>
> Pi 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> Pi 版本描述：`v0.80.7-13-g5d9fedf73`<br>
> 外部项目核对日期：2026-09-09

比较对象限定为一个工程问题：**父 Agent 如何把一个自包含任务交给独立执行单元，并控制子上下文、工具权限、结果回流和取消失败**。上下文压缩、会话恢复和团队任务表只在直接影响这条委派链时出现。

本文使用四种证据标签：

- **Pi 源码事实**：可在锁定的 Pi commit 中复核；
- **开源实现**：可在 Hermes Agent 或 Codex 的指定 commit 中复核；
- **官方公开行为**：来自 Codex、Claude Code 当前官方文档，不推断未公开内部实现；
- **推断**：根据公开接口得出的设计判断，不写成产品事实。

外部项目仍在演进。本文记录的是核对日的行为；Pi 的判断始终以锁定 commit 为准。

## 1. 同一个问题，四种边界

| 维度 | Pi | Hermes Agent | Codex | Claude Code |
| --- | --- | --- | --- | --- |
| 能力所在层 | 可选扩展示例，不在核心 | 框架内置 `delegate_task` | 产品与开源运行时内置协作工具 | 产品内置 subagent；另有 agent teams |
| 子上下文起点 | 任务文本 + agent system prompt；独立无持久化 session | goal + context 生成新提示，跳过普通 context files 与 memory | 可新建子线程，也支持历史 fork；配置从父 turn 快照派生 | 普通 subagent 使用独立上下文；fork 是另一种显式模式 |
| 工具控制 | agent 文件的工具名列表；无列表时不额外收窄 | 不得扩大父工具集，并按角色扣除危险协作工具 | 继承父运行时策略，可由 custom agent 再收窄 | 工具、禁用工具、permission mode、hook 与 MCP 可按 agent 配置 |
| 并发与嵌套 | parallel 最多 8 项、同时 4 个；示例无深度状态 | 并发与深度都有配置，叶子默认失去委派工具 | 多子线程、消息、follow-up、等待、停止与关闭均有运行时对象 | foreground/background；当前文档支持受深度限制的嵌套 |
| 工作目录 | 默认共享父 cwd，也可逐项指定 cwd | 默认共享；可选 Git worktree，默认关闭 | 官方警告并行写冲突；sandbox 是权限边界，不等于工作副本隔离 | 可设置 `isolation: worktree`，否则从主会话 cwd 启动 |
| 结果回流 | JSON 事件收集为工具结果；parallel 每项模型可见输出上限 50 KiB | 父 Agent 只接收摘要；按父上下文余量动态截断并保存完整文本 | 主线程收集结果并汇总，用户可检查各子线程 | 普通 subagent 返回摘要；后台结果通过完成通知回到后续 turn |
| 中止与失败 | AbortSignal → SIGTERM，5 秒后尝试 SIGKILL；chain 遇错即停 | 父中止递归传播；超时、异常、未送达 steer 有独立状态 | 可 steer、interrupt、wait、close；无审批通道时需新审批的动作失败 | 可停止任务；权限请求回到主会话；API 截断有有限次续写 |
| 持久化 | 子进程固定 `--no-session`，没有可恢复 child task | child session 与 live log 可记录；另有异步委派状态 | 每个 child 是可寻址线程，可继续发送任务或关闭 | 普通 subagent 可 resume；agent teams 另存任务表和 mailbox |

表中最容易混淆的是“Pi 没有内置 subagent”和“Pi 完全不能运行 subagent”。前者是官方设计原则，后者不成立：源码仓库提供了完整的扩展示例，只是没有把它升级成核心协议。

## 2. Pi 把委派留给扩展

### 2.1 解决的问题

Pi 核心只负责一次 Agent 循环、工具调用、session 和扩展注册。`packages/coding-agent/docs/usage.md:303-309` 明确说明：sub-agent、MCP、permission popup、plan mode 和 todo 都不属于内置功能，可以由扩展或外部工具提供。

仓库中的 `packages/coding-agent/examples/extensions/subagent/` 正是这种边界的示范。安装后，它向父 Agent 注册一个普通工具；父模型决定调用工具，扩展再创建独立 `pi` 子进程。它不是 `Agent`、`AgentHarness` 或 orchestrator 的隐藏分支。

### 2.2 源码入口与关键短代码

入口是 `packages/coding-agent/examples/extensions/subagent/index.ts:461-528`。工具接收 single、parallel、chain 三种互斥输入，并在使用项目级 agent 定义前进行一次 UI 确认：

```ts
pi.registerTool({
	name: "subagent",
	description: [
		"Delegate tasks to specialized subagents with isolated context.",
		"Modes: single (agent + task), parallel (tasks array), chain (sequential with {previous} placeholder).",
	].join(" "),
	parameters: SubagentParams,

	async execute(_toolCallId, params, signal, onUpdate, ctx) {
		const agentScope = params.agentScope ?? "user";
		const discovery = discoverAgents(ctx.cwd, agentScope);
		const agents = discovery.agents;
		const confirmProjectAgents = params.confirmProjectAgents ?? true;

		// single、parallel、chain 必须恰好选择一种
		// 项目级定义只在显式选择 project/both 后参与发现
		// 交互模式下，真正执行前再询问用户
	}
});
```

agent 定义来自用户目录或离 cwd 最近的 `.pi/agents`。`agents.ts:26-74,85-116` 只读取 `name`、`description`、`tools`、`model` 和 Markdown 正文。项目定义与用户定义同名时，`both` 模式让项目定义覆盖用户定义。

### 2.3 调用链与上下文变化

一次 single 委派经过以下路径：

```text
父模型生成 subagent tool call
  → 扩展按 scope 重新发现 agent 文件
  → 选出 model、tools 和 system prompt
  → 把 system prompt 写入 0600 临时文件
  → spawn 新 pi 进程：--mode json -p --no-session
  → 追加 --model / --tools / --append-system-prompt
  → 只把 “Task: <task>” 作为用户输入
  → 逐行解析子进程 JSON 事件
  → 提取最后一条 assistant 文本作为结果
  → 清理临时提示文件
```

`runSingleAgent()` 的进程边界很直接：

```ts
const args: string[] = ["--mode", "json", "-p", "--no-session"];
if (agent.model) args.push("--model", agent.model);
if (agent.tools?.length) args.push("--tools", agent.tools.join(","));

if (agent.systemPrompt.trim()) {
	const tmp = await writePromptToTempFile(agent.name, agent.systemPrompt);
	args.push("--append-system-prompt", tmp.filePath);
}

args.push(`Task: ${task}`);
const proc = spawn(invocation.command, invocation.args, {
	cwd: cwd ?? defaultCwd,
	shell: false,
	stdio: ["ignore", "pipe", "pipe"],
});
```

见 `packages/coding-agent/examples/extensions/subagent/index.ts:294-339`。

这里的“独立上下文”由新进程和 `--no-session` 实现，不是从父 session 投影出来的分支。父会话正文、父工具调用历史和父压缩摘要不会自动进入子模型；传入内容只有父模型写出的 task、agent 定义的 system prompt，以及新进程按正常启动流程加载的项目资源。

状态也分为两份：父进程只有一次工具调用；子进程运行自己的 Agent loop，但不保存 session。进程退出后，`SingleResult` 作为父工具结果的 `details` 快照进入父 session，完整 child 消息可以随父历史保存；它仍不是可继续运行的 child session，不能用 child id 恢复或追加任务。

### 2.4 结果合并不是任务协议

parallel 模式允许最多 8 项，`mapWithConcurrencyLimit()` 同时运行 4 个子进程。每项成功与否由 `exitCode`、`stopReason === "error"` 或 `"aborted"` 判定；所有项结束后再按输入顺序拼成父模型可见文本。每项超过 50 KiB 时只截断模型可见内容，完整消息仍在工具结果 details 中，并可随父 session 保存。见 `index.ts:33-36,219-235,583-663`。

chain 模式更简单：上一步最后一条 assistant 文本替换下一步 task 中的 `{previous}`。任何一步失败，链立即停止。它没有结果 schema、验收者、重试策略或 durable task id，因此只能证明“文本被串起来”，不能证明工作已经验收。

### 2.5 权限与失败路径

工具列表控制模型能看到哪些 Pi 工具，不是操作系统权限。`spawn()` 没有设置独立 sandbox 或工作区，默认使用父 cwd；也没有显式清除环境变量。**推断**：按 Node 子进程默认行为，子进程会继承父进程环境，因此工具 allowlist 不能替代凭据隔离。

项目级 agent 文件可能包含仓库控制的提示。默认 `agentScope` 为 `user`，这是第一层保护；显式启用 project/both 后，交互模式还会确认。没有 UI 时该确认分支不执行，所以自动化调用方必须自己决定是否允许项目定义。见 `index.ts:472-528` 和 `README.md:55-65`。

中止路径由父工具的 `AbortSignal` 驱动：先向子进程发送 SIGTERM，5 秒后若仍未标记 killed，再尝试 SIGKILL。见 `index.ts:399-408`。未知 agent 返回 `exitCode: 1`；子进程非零退出、模型 error/aborted 都是失败；parallel 保留其他项结果，chain 停在首个失败步骤。

这个示例没有专门的测试文件。仓库测试能覆盖扩展注册、CLI 参数和通用中止信号，却没有构造 subagent 的 single/parallel/chain、项目 agent 确认、50 KiB 截断或子进程清理。README 中的行为仍需把扩展示例本身纳入测试后，才能获得回归保证。

## 3. Hermes Agent 把委派做成框架内状态

> 开源实现基线：`NousResearch/hermes-agent@9d865810b66b15a811eb62e0095f8cb66f6c032d`

### 3.1 入口与子上下文

Hermes 的入口是 `tools/delegate_tool.py::delegate_task()`。它不启动另一个 CLI，而是在框架内构造新的 `AIAgent`。`_build_child_agent()` 为 child 计算深度和角色，用 goal、context 与 workspace hint 生成临时 system prompt，再创建独立 session：

```py
child_depth = getattr(parent_agent, "_delegate_depth", 0) + 1
max_spawn = _get_max_spawn_depth()
effective_role = (
    "orchestrator"
    if _get_orchestrator_enabled() and child_depth < max_spawn
    else "leaf"
)
child_toolsets, child_disabled_toolsets = _resolve_child_toolsets(
    parent_agent, toolsets, effective_role
)
child_prompt = _build_child_system_prompt(goal, context, ...)

child = AIAgent(
    **rt,
    enabled_toolsets=child_toolsets,
    disabled_toolsets=child_disabled_toolsets,
    ephemeral_system_prompt=child_prompt,
    skip_context_files=True,
    skip_memory=True,
    session_db=child_session_db,
    parent_session_id=parent_sid,
    iteration_budget=None,
)
```

见 [`tools/delegate_tool.py:154-264`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/delegate_tool.py#L154-L264)。child 有自己的 task/session 标识、terminal 状态和文件操作缓存；父 Agent 不接收 child 的中间 tool call 与 reasoning，只接收最终摘要。

Pi 示例和 Hermes 都用“任务包而非父全文”隔离上下文，差别在于 Hermes 把 parent/child 身份、深度和 session 关系纳入框架状态，Pi 示例只保留一次工具调用的临时结果。

### 3.2 工具决策者与安全边界

Hermes 先从父工具集推导 child 工具集，调用方要求的 toolset 只能与父能力取交集。随后无条件扣除 `clarify`、`memory`、`send_message`、`cronjob_manage` 等工具；达到深度上限的 leaf 也失去 `delegate_task`。可继续委派的 orchestrator child 才重新获得 delegation toolset。见 [`tools/delegate_tool_toolsets.py:13-114`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/delegate_tool_toolsets.py#L13-L114)。

这比 Pi 示例的工具名列表多两条不变量：child 不能扩大父权限；协作副作用工具按角色强制扣除。Hermes 仍需后端、terminal approval callback 和凭据配置共同提供真实隔离，工具名过滤本身也不是 sandbox。

### 3.3 汇总、取消与工作区

同步 batch 使用 daemon worker 并行运行 child。父中止后，运行时向活跃 child 传播 cooperative interrupt，把未完成 future 记为 `interrupted`，已完成与异常结果按 task index 排序。见 [`tools/delegate_tool_dispatch.py:97-163`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/delegate_tool_dispatch.py#L97-L163)。

摘要不是任意长文本。默认静态上限为 24,000 字符，同时根据父上下文剩余 token、压缩输出预留和 child 数量计算动态上限，取两者较小值。超限时保留头尾，并把完整摘要写入缓存文件。见 [`tools/delegate_tool_results.py:158-223,239-290`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/delegate_tool_results.py#L158-L290)。这解决的是“并行结果反过来压爆父上下文”，Pi 示例的固定 50 KiB 上限没有感知父上下文余量。

Hermes 可为 child 创建 Git worktree，但 `delegation.worktree_isolation` 默认关闭；非 Git 或创建失败时会静默降级到共享工作目录。保留下来的结果包含 path、branch、commit 数和 dirty 状态；探测失败时标记为未知，不把默认的零/干净误当事实。见 [`tools/subagent_worktree.py:75-155`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/subagent_worktree.py#L75-L155)。

## 4. Codex 把 child 作为可寻址线程

> 开源实现基线：`openai/codex@c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1`

### 4.1 公开行为

Codex 官方文档把子任务编排描述为：spawn 子 Agent、发送 follow-up、等待结果、停止或关闭线程，最后由主线程合并结果。CLI 的 `/agent` 可以切换到 child thread 检查进行中的内容；应用界面也能打开每个 child。参见 [Codex Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)。

这不是“函数返回一段文本”模型。child 有稳定 thread id，父 Agent 可在首轮完成后继续发新任务，也可在运行中 steer 或 interrupt。中间历史留在 child thread，父线程主要消费回流结果，用户仍能检查原始线程。

### 4.2 配置快照与权限

开源实现的 `build_agent_spawn_config()` 从父 turn 的有效配置建立 child 配置。源码不是简单复制启动时配置，而是重新覆盖当前模型、provider、reasoning、developer instructions、cwd、approval policy 和 permission profile：

```rs
fn build_agent_shared_config(turn: &TurnContext) -> Result<Config, FunctionCallError> {
    let base_config = turn.config.clone();
    let mut config = (*base_config).clone();
    config.token_budget = turn.configured_token_budget.clone();
    config.model = Some(turn.model_info().slug.clone());
    config.model_provider = turn.provider.info().clone();
    config.model_reasoning_effort = turn.reasoning_effort()
        .or(turn.model_info().default_reasoning_level.as_ref())
        .cloned();
    config.developer_instructions = turn.developer_instructions.clone();
    apply_spawn_agent_runtime_overrides(&mut config, turn)?;
    Ok(config)
}
```

见 [`multi_agents_common.rs:158-248`](https://github.com/openai/codex/blob/c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1/codex-rs/core/src/tools/handlers/multi_agents_common.rs#L158-L248)。role 或显式 model/reasoning override 在这份快照之后叠加，并验证模型与 reasoning 是否兼容。

官方公开行为进一步规定：subagent 继承父 turn 的当前 sandbox 和 permission mode；交互 CLI 可以把 child 的审批请求显示给用户，无法呈现新审批的非交互流程则让动作失败并把错误返回父流程。custom agent 可再配置成只读。sandbox/approval 的决策者是运行时和用户，不是 child 的 system prompt。

### 4.3 线程控制与失败

工具规格不仅包含 `spawn_agent`，还有发送消息、触发 follow-up、等待、恢复与中止等操作。`send_message` 只排队消息，不自动启动新 turn；`followup_task` 在 idle child 上触发新 turn，在运行中则于消息边界或工具结束后交付。见 [`multi_agents_spec.rs:131-237`](https://github.com/openai/codex/blob/c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L131-L237)。

运行时把 thread not found、closed child、manager unavailable 和 spawn failure 转换为可返回模型的错误。见 [`multi_agents_common.rs:76-99`](https://github.com/openai/codex/blob/c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1/codex-rs/core/src/tools/handlers/multi_agents_common.rs#L76-L99)。这使“子任务失败”和“协作运行时不可用”保留为不同原因。

官方文档提醒并行写密集型任务可能互相冲突。**推断**：Codex 的线程隔离、sandbox 继承和工作副本隔离是三个不同问题；看到独立 child thread，不能据此假设它自动获得独立 Git worktree。

## 5. Claude Code 区分 subagent 与 agent team

> 本节只陈述 [Claude Code subagents](https://code.claude.com/docs/en/sub-agents) 和 [agent teams](https://code.claude.com/docs/en/agent-teams) 的官方公开行为，不声称掌握未公开内部源码。

### 5.1 普通 subagent

普通 subagent 用于把高噪声支线移出主上下文。每个 child 有独立 context window、system prompt、工具与权限，结束后把摘要返回主会话。agent 文件可设置 tools、disallowedTools、model、permissionMode、MCP、hooks、maxTurns、skills、memory、background 与 worktree isolation 等字段。

触发者可以是 Claude 根据 description 自动选择，也可以由用户用自然语言或 @mention 明确指定。@mention 保证选中的 subagent 会运行，但完整用户消息仍先到主会话，由主 Agent 生成 child task prompt。

默认工作目录是主会话 cwd；`isolation: worktree` 才创建独立工作副本。foreground 会阻塞主会话，background 与主会话并行。当前文档说明 background 的权限请求会显示在主会话，用户可以批准或拒绝单次工具调用。

child 的工具来自主会话能力并经过固定过滤和 agent 配置收窄。达到深度限制时 `Agent` 工具被移除或调用报错；`AskUserQuestion`、结束主会话和调度类工具也不能由普通 child 任意使用。这里的嵌套是受深度和工具过滤控制的，不应沿用旧版本“subagent 永远不能再 spawn”的结论。

失败方面，后台 child 成功后通过完成通知回到后续 turn；失败或人工停止会保留短期任务状态。若 API 中途截断且已有文本但没有 tool call，Claude Code 会有限次要求 child 续写，次数用尽才结束为错误。

### 5.2 agent teams 是另一种协作语义

Agent teams 不是普通 subagent 的别名。官方公开架构包含 team lead、独立 Claude Code teammate、共享 task list 和 mailbox。teammate 可以彼此发消息、认领带依赖的任务；任务和团队配置保存在本地目录。mailbox 写入失败时发送方收到错误，不会报告为已发送。

这更接近第 27 讲设计推演中的协调层，但仍不能据此说 Pi 已实现相同机制。Pi 的 orchestrator 只有实例与 RPC 进程状态，subagent 示例只有一次工具调用结果，都没有共享任务表和 agent mailbox。

## 6. 用一次委派对齐状态变化

目标：父 Agent 让 child 检查认证模块，只允许读取，最后返回可核对的结论。

### Pi 扩展示例

```text
父 tool call(running)
  → child process(running, --no-session, tools=read/grep/...)
  → JSON message/tool events 累积在 SingleResult
  → child exit
  → tool result(success/error)
```

决策者是父模型和扩展代码。失败留在当次工具结果；没有 child id、恢复状态或 durable task。

### Hermes Agent

```text
delegate_task
  → child identity + depth + role
  → child AIAgent/session running
  → progress/live log
  → completed | error | timeout | interrupted
  → 预算化 summary 返回父 Agent
```

决策者是 delegate runtime。父权限上界、强制禁用工具、深度和摘要预算由框架执行。

### Codex

```text
spawn_agent
  → child thread running
  ↔ send_message / followup / steer
  → wait result or interrupt
  → child thread idle/completed/closed
  → parent consolidated response
```

决策者分为主 Agent、协作运行时与审批用户。child thread 可继续寻址，失败按 spawn、lookup、closed、permission 等原因返回。

### Claude Code subagent

```text
自动匹配或显式指定
  → foreground/background child
  → 独立 context + filtered tools + permission mode
  → permission request 回主会话
  → completion notification / failure / stop
  → summary 进入主会话
```

agent teams 会再增加 task claim、dependency 与 mailbox，不应混入普通 subagent 流程。

## 7. 对 Pi 的设计结论

Pi 示例证明扩展 API、JSON 模式和独立进程足以搭出轻量委派。它适合临时分析、并行搜索和固定文本链，不适合作为可恢复的任务协调层。缺口不是再加一个 `spawn()`，而是把下面这些状态变成协议：

- child/task 的稳定身份与父子关系；
- 进入 child 的 context manifest，而不只是任意 task 字符串；
- 父权限上界、每项 capability 与执行环境；
- 结果 schema、来源、验收状态和摘要预算；
- cancel requested、interrupted、unknown outcome 与恢复；
- 写工作区租约、worktree 信息与合并责任。

Hermes 提供了“框架内 child 生命周期”的参考，Codex 提供了“可寻址线程与运行时策略继承”的参考，Claude Code 则明确拆开 subagent 和 agent team。**推断**：若 Pi 继续坚持小核心，较自然的演进不是把整套调度器塞进 `Agent`，而是把稳定 child/task 协议放在独立包中，由扩展或 orchestrator 适配；单 Agent 核心仍只处理自己的 turn 与工具。

## 8. 论文核对：相似不等于实现

### AutoGen

[AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation](https://arxiv.org/abs/2308.08155) 原本解决的是通用多 Agent 应用编程：Agent 可以组合 LLM、人和工具，开发者用自然语言或代码定义多方会话模式。

Pi 没有实现 AutoGen 的 conversable-agent 抽象或通用多方 conversation runtime。Pi subagent 示例的父工具调用和文本回流，与“Agent 通过消息协作”只有思想相近；其 single/parallel/chain 是扩展自行写死的三种流程，不能据此建立实现继承关系。

### MetaGPT

[MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework](https://arxiv.org/abs/2308.00352) 针对朴素串联 LLM 时的逻辑不一致和级联幻觉，把标准作业流程编码为 prompt 序列，让不同角色生产并核对中间成果。

Pi 仓库的示例确实有 scout、planner、worker、reviewer 角色和链式 prompt，但没有 MetaGPT 的 SOP runtime、统一中间产物协议或论文所述的软件团队装配线。能从源码确认的是“样例角色名称和顺序相似”；“Pi 实现了 MetaGPT”没有依据。

两篇论文都不能证明并行 Agent 比单 Agent 更可靠。并行只是增加候选工作；结果是否正确仍取决于任务拆分、权限、证据回流和验收。

## 9. 官方设计文档与当前实现的边界

`packages/agent/docs/durable-harness.md` 是长期设计说明，不是现行能力清单。它提出 session 作为 durable append-only state tree，恢复从持久边界开始；未完成 provider stream 不可续传，非幂等 tool call 默认不可自动重放。文档中的 future builder、durable queue entry、operation marker 和 recovery policy 在锁定 commit 仍属于目标设计。

这份设计与 child 委派的关系是约束而非实现：如果以后要恢复 child task，必须记录已接受任务、已消费队列、未完成工具副作用和运行时依赖版本。Pi subagent 示例使用 `--no-session`，没有接上这些 durable 设计；orchestrator 的实例表也不是 task journal。

## 10. 测试证据与未验证边界

Hermes 的 `tests/tools/test_delegate.py` 覆盖 child prompt、禁止工具、并发、spawn depth、失败摘要和角色行为；`tests/tools/test_subagent_worktree.py` 明确验证 worktree isolation 默认关闭；`tests/tools/test_delegate_timeout_cleanup.py` 验证超时后要等 worker 展开完成再关闭 child，避免与 session 最终写入竞争。这里引用的是指定 commit 的测试意图，未在 Pi 工作区执行 Hermes 测试。

Codex 开源仓库的 multi-agent 规格与 handler 测试可核对配置继承和工具错误语义；Claude Code 部分没有源码证据，只能以官方文档为边界。

Pi 工作区没有 subagent 示例的专门测试。本次尝试运行 coding-agent 的参数与并发相关测试，Vitest 在加载测试文件前因依赖未安装而退出。这是环境问题，没有形成断言结果，不能归为文档错误或上游缺陷。

## 11. 常见边界问题

### Pi 已经有 subagent 扩展，为什么第 27 讲仍说协调层缺失？

扩展解决“启动 child 并收回文本”。协调层还要保存任务身份、依赖、attempt、工作区租约、验收与恢复。两者不是同一层。第 27 讲的结论应收窄为“Pi 核心和 orchestrator 没有 durable coordination protocol”，不能表述为“仓库没有任何 subagent 实现”。

### 新进程是否自动意味着安全隔离？

不是。新进程提供地址空间和上下文分离；默认共享 cwd 与环境时，文件和凭据仍可能共享。sandbox、工具过滤、凭据代理和 worktree 分别控制不同边界。

### 摘要返回是否等于上下文隔离？

只解决结果进入父上下文的体积。child 启动时加载了什么、能读什么、完整结果保存在哪里，是另外三项决策。Hermes 明确跳过普通 context files/memory；Pi 示例会按新 Pi 进程的正常资源发现，重新加载获准使用的项目资源。

### agent role 是否等于权限？

不是。role 或 system prompt 描述希望模型怎样行动；工具 allowlist、permission profile、sandbox 和工作区才执行能力边界。Hermes、Codex 与 Claude Code 都有运行时收窄机制，Pi 示例主要依赖工具名列表和项目 agent 确认。

## 12. 源码与资料锚点

### Pi 固定基线

- 核心设计原则：`packages/coding-agent/docs/usage.md:303-309`
- subagent 入口、模式与项目 agent 确认：`packages/coding-agent/examples/extensions/subagent/index.ts:433-528`
- child 进程、JSON 事件、中止：`packages/coding-agent/examples/extensions/subagent/index.ts:267-421`
- chain 与 parallel 合并：`packages/coding-agent/examples/extensions/subagent/index.ts:530-683`
- agent 文件发现与覆盖：`packages/coding-agent/examples/extensions/subagent/agents.ts:26-116`
- 示例安全说明与限制：`packages/coding-agent/examples/extensions/subagent/README.md:55-65,111-160`
- durable harness 目标设计：`packages/agent/docs/durable-harness.md:7-24,60-80,116-180`

### 外部开源实现与官方行为

- [Hermes Agent `delegate_tool.py`](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/tools/delegate_tool.py)
- [Hermes Agent delegation 文档](https://github.com/NousResearch/hermes-agent/blob/9d865810b66b15a811eb62e0095f8cb66f6c032d/website/docs/user-guide/features/delegation.md)
- [Codex `multi_agents_common.rs`](https://github.com/openai/codex/blob/c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1/codex-rs/core/src/tools/handlers/multi_agents_common.rs)
- [Codex `multi_agents_spec.rs`](https://github.com/openai/codex/blob/c3eeaae9a3d401f7e7ade0817b7b27299af7d2a1/codex-rs/core/src/tools/handlers/multi_agents_spec.rs)
- [Codex Subagents 官方文档](https://learn.chatgpt.com/docs/agent-configuration/subagents)
- [Claude Code Subagents 官方文档](https://code.claude.com/docs/en/sub-agents)
- [Claude Code Agent Teams 官方文档](https://code.claude.com/docs/en/agent-teams)
