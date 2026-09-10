# 第 29 讲：一次完整的源码修改——修正子 Agent 的中止升级

## 修改范围

本次修改基于 Pi 源码 commit `5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`，工作分支为 `hongkaigu/lesson-29-subagent-abort`。改动限定在 coding-agent 的 subagent 扩展示例：修正子进程中止逻辑，补充定向测试、示例说明和变更记录。Agent 循环、会话管理、编排器与依赖版本均未调整。

这不是上游已经合并的行为，而是一份本地源码练习补丁。它要解决的工程问题很小，却完整经过了问题定位、边界设计、实现、测试与文档同步。

## 问题：发出信号不等于进程已经退出

subagent 扩展通过 `spawn()` 启动一个独立的 `pi --mode json` 子进程。用户中止工具调用时，原实现先发送 `SIGTERM`，五秒后检查 `proc.killed`，如果它仍为假才发送 `SIGKILL`：

```ts
let wasAborted = false;

if (signal) {
	const killProc = () => {
		wasAborted = true;
		proc.kill("SIGTERM");
		setTimeout(() => {
			if (!proc.killed) proc.kill("SIGKILL");
		}, 5000);
	};
	if (signal.aborted) killProc();
	else signal.addEventListener("abort", killProc, { once: true });
}
```

问题不在五秒这个数字，而在状态含义。Node.js 对 `subprocess.killed` 的定义是“是否成功向子进程发送过信号”，并明确说明它不表示子进程已经终止。[Node.js `subprocess.killed` 文档](https://nodejs.org/api/child_process.html#subprocesskilled)

因此，第一次 `proc.kill("SIGTERM")` 成功后，`proc.killed` 通常已经变为 `true`。即使子进程忽略 `SIGTERM`、卡在清理阶段或迟迟没有退出，定时器也不会再发送 `SIGKILL`。原代码把两个不同事实压进了同一个布尔值：

- 控制面事实：父进程已经请求中止，并成功发出过信号；
- 运行时事实：子进程是否真的结束。

中止升级必须依据第二个事实决策。

## 状态模型

这段逻辑涉及四类状态，各自有明确的所有者：

| 状态 | 所有者 | 读写方式 | 用途 |
| --- | --- | --- | --- |
| 用户是否请求中止 | `AbortSignal` | 监听 `abort` | 触发中止流程 |
| 本次调用是否因中止结束 | `ChildProcessAbortHandle.aborted` | 中止处理器写，调用方读 | 把进程结果转换成工具错误 |
| 子进程是否退出 | Node 子进程对象 | 读取 `exitCode`、`signalCode`，监听 `exit` | 决定是否需要强制终止 |
| 宽限期是否仍有效 | 中止处理器 | 保存并清理定时器 | 防止退出后误发 `SIGKILL` |

状态流如下：

```text
AbortSignal 触发
    │
    ├─ 子进程已退出 ────────────────> 不发送信号，aborted=false
    │
    └─ 子进程仍运行
         ├─ 标记 aborted=true
         ├─ 发送 SIGTERM
         └─ 启动 5 秒宽限期
              ├─ 宽限期内 exit ────> 取消定时器并解绑监听
              └─ 到期仍未 exit ────> 发送 SIGKILL
```

`aborted` 描述调用原因，`exitCode` 和 `signalCode` 描述进程结果。两者不能互相替代：一个子进程可能收到中止请求后自行正常退出，也可能被信号终止；上层都应把这次工具调用报告为“已中止”。

## 把进程控制从工具实现中分离

新增的 `process-control.ts` 只依赖五项能力：退出码、终止信号、发送信号、注册一次性退出监听、移除退出监听。它没有接收完整的 Agent 配置或工具上下文。

`packages/coding-agent/examples/extensions/subagent/process-control.ts:27-53` 的核心逻辑是：

```ts
const hasExited = () => proc.exitCode !== null || proc.signalCode !== null;

const dispose = () => {
	if (disposed) return;
	disposed = true;
	if (forceKillTimer) clearTimeout(forceKillTimer);
	signal?.removeEventListener("abort", onAbort);
	proc.off("exit", onExit);
};

const onExit = () => dispose();
const onAbort = () => {
	if (disposed || aborted || hasExited()) return;
	aborted = true;
	if (!proc.kill("SIGTERM")) return;

	forceKillTimer = setTimeout(() => {
		forceKillTimer = undefined;
		if (!hasExited()) proc.kill("SIGKILL");
	}, forceKillDelayMs);
	forceKillTimer.unref();
};

if (signal) {
	proc.once("exit", onExit);
	if (signal.aborted) onAbort();
	else signal.addEventListener("abort", onAbort, { once: true });
}
```

这里有三项容易遗漏的处理。

第一，退出判断读取 `exitCode` 和 `signalCode`。正常退出会设置前者，被信号终止会设置后者；任一不为 `null` 都说明无需升级。

第二，子进程一旦发出 `exit` 事件，立即取消定时器并解绑监听。否则定时器仍会持有闭包，五秒后还可能对已经结束的进程执行一次无意义的 `kill()`。

第三，定时器调用 `unref()`。强制终止计时器不应单独阻止父进程退出；如果事件循环已经没有其他工作，Node 可以直接结束。

为便于测试，模块定义了窄接口 `AbortableChildProcess`，而不是伪造完整的 `ChildProcess`。这种接口不是业务抽象，它是一条测试边界：实现只声明自己真正使用的进程能力，测试替身也不必模拟标准库对象的全部重载。

## 接回 subagent 调用链

`runSingleAgent()` 仍负责启动进程、解析 JSONL、累计 usage 和生成工具结果。它只把进程中止交给新模块：

```ts
let wasAborted = false;
let abortHandle: ChildProcessAbortHandle | undefined;

const exitCode = await new Promise<number>((resolve) => {
	const invocation = getPiInvocation(args);
	const proc = spawn(invocation.command, invocation.args, {
		cwd,
		stdio: ["ignore", "pipe", "pipe"],
	});

	// stdout、stderr、close 与 error 处理保持不变
	abortHandle = bindAbortSignal(proc, signal);
});

wasAborted = abortHandle?.aborted ?? false;
abortHandle?.dispose();
currentResult.exitCode = exitCode;
if (wasAborted) throw new Error("Subagent was aborted");
```

入口位于 `packages/coding-agent/examples/extensions/subagent/index.ts:330-407`。调用链没有改变：工具执行器调用 `runSingleAgent()`，后者创建 CLI 子进程，子进程通过 JSONL 返回消息。变化只发生在中止分支：原先由入口函数直接管理信号和定时器，现在由 `bindAbortSignal()` 返回一个可查询、可释放的句柄。

这也保留了原有失败语义。子进程退出仍先使 Promise 完成并记录 `exitCode`；如果期间发生过有效的中止请求，调用方随后抛出 `Subagent was aborted`。中止不会被伪装成普通的非零退出码。

## 定向测试如何锁定失败路径

测试没有启动真实的 `pi` 进程，而是使用一个能记录信号、发出 `exit` 事件的窄替身。真实进程会引入启动时间、平台信号行为和模型配置，使五秒升级逻辑难以稳定复现；此处需要验证的是状态机，而不是 CLI 集成。

`packages/coding-agent/test/subagent-process-control.test.ts:30-68` 覆盖三条路径：

1. 中止后立即收到 `SIGTERM`，五秒内没有退出，再收到 `SIGKILL`；
2. 宽限期内收到 `exit`，定时器被清理，不再发送 `SIGKILL`；
3. 绑定时进程已经退出，即使 `AbortSignal` 已中止也不再发送信号，`aborted` 保持为假。

定向测试最终结果为 1 个测试文件、3 个用例全部通过。第一次从仓库根目录直接调用 `node_modules/vitest/dist/cli.js` 时未找到模块；该依赖安装在 `packages/coding-agent/node_modules`，改由包目录内的实际入口执行后通过。这是依赖布局造成的命令路径问题，不是文档错误或上游源码缺陷。

第一次运行 `npm run check` 时，测试替身被强制断言为过宽的标准库类型，TypeScript 拒绝了不兼容的事件重载。这个失败属于本次代码的类型设计错误。改为窄接口后，定向测试再次通过，仓库完整检查也通过，包括 Biome、依赖固定检查、TypeScript、shrinkwrap、安装锁和浏览器 smoke check。

## 文档和分发也属于修改的一部分

subagent 示例的 README 给出了逐文件软链接安装方式。新增模块后，如果只改入口代码而不补软链接命令，用户安装得到的 `index.ts` 会导入一个不存在的文件。因此 `packages/coding-agent/examples/extensions/subagent/README.md:12-42` 同时更新了能力说明、目录结构与安装命令；`packages/coding-agent/CHANGELOG.md:28` 记录了用户可感知的修复。

这说明“代码能通过测试”不是修改的全部边界。源码的模块拆分会改变示例的分发单元，文档必须随之调整，否则仓库内运行正确，复制到用户目录后仍会失败。

## 设计取舍与未覆盖边界

把逻辑抽成独立模块增加了一个文件和一条安装命令，代价是示例结构稍微复杂。收益是中止状态机可以脱离模型调用和 JSONL 解析进行确定性测试，也避免继续扩大 `runSingleAgent()` 的职责。

当前实现只控制 `spawn()` 返回的直接子进程，没有建立独立进程组，也没有遍历其后代。若子进程再启动长期运行的孙进程，直接发送 `SIGTERM` 或 `SIGKILL` 不保证整棵进程树都结束。这个边界可由源码确认；是否需要进程组、平台专用终止策略或外部进程树库，要根据扩展的部署平台另行设计。

五秒宽限期仍是固定策略，没有做成用户配置。对交互式工具而言，它提供了简单且可预测的上限；如果未来出现需要长时间落盘的子 Agent，固定宽限期可能过短。那时应把“退出协议”提升为扩展配置，而不是继续在进程控制模块中堆叠特例。
