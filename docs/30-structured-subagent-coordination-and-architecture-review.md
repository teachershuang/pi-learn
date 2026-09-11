# 第 30 讲：并行子 Agent 的结构化收尾与架构复盘

## 改造范围

本课仍以 Pi 源码 commit `5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf` 为基线。源码练习分支是 `hongkaigu/lesson-30-subagent-coordination`，其中保留了第 29 讲尚未提交的子进程中止补丁。本次新增部分只处理 parallel 模式的任务调度：父级中止或子任务抛出意外异常后，不再启动排队任务；已经启动的任务全部收尾后，调度器才向上层传播错误。

源码分支没有提交，也没有推送到上游。这份改造是本地练习，不代表 `earendil-works/pi` 已经采用该实现。

## 原调度器的问题

parallel 模式最多接收 8 个任务，同时运行 4 个。基线中的 `mapWithConcurrencyLimit()` 用多个 worker 共享 `nextIndex`，每个 worker 完成一项后继续领取下一项：

```ts
const results: TOut[] = new Array(items.length);
let nextIndex = 0;
const workers = new Array(limit).fill(null).map(async () => {
	while (true) {
		const current = nextIndex++;
		if (current >= items.length) return;
		results[current] = await fn(items[current], current);
	}
});
await Promise.all(workers);
return results;
```

正常路径没有问题：并发数受限，结果仍按输入顺序保存。麻烦出现在 `fn()` 抛错时。

`Promise.all(workers)` 会在第一个 worker 拒绝后立即拒绝，但其他 worker 不会自动取消。它们仍在执行，完成当前任务后还会继续读取 `nextIndex`，于是父工具已经进入失败处理，排队的子 Agent 却可能刚刚启动。这里破坏的是生命周期边界：创建子任务的父调用已经结束，却不再拥有那些仍在运行的任务。

用户中止会稳定触发这个问题。`runSingleAgent()` 收到 `AbortSignal` 后终止子进程，并在进程关闭后抛出 `Subagent was aborted`。最先关闭的子进程使 `Promise.all()` 立即拒绝，其他活跃进程仍在清理；没有共享停止条件的 worker 还可能领取后续任务。由于同一个 signal 已经中止，新进程通常会刚启动就收到终止信号。这会浪费进程创建成本。上层工具执行器在 `execute()` 结束后把 `acceptingUpdates` 设为 false，迟到的进度不会再进入事件流，但后台任务仍然存在，见 `packages/agent/src/agent-loop.ts:659-704`。

## 调度器拥有的状态

改造后的状态集中在 `packages/coding-agent/examples/extensions/subagent/concurrency.ts:6-45`。

| 状态 | 写入者 | 读取者 | 含义 |
| --- | --- | --- | --- |
| `nextIndex` | 成功领取任务的 worker | 所有 worker | 下一项尚未领取的输入位置 |
| `results[index]` | 执行该位置的 worker | 调度器调用方 | 保持输入顺序的成功结果 |
| `hasError` | 第一个捕获异常的 worker | 所有 worker | 禁止继续领取排队任务 |
| `firstError` | 第一个捕获异常的 worker | 调度器 | 所有活跃任务结束后要传播的原始错误 |
| `signal.aborted` | 父调用的 `AbortController` | 所有 worker 与调度器 | 父级已经撤销本次并行调用 |

`nextIndex` 不需要锁。JavaScript 在单个事件循环中执行同步片段，检查停止条件、读取索引和自增之间没有 `await`。worker 只有进入 `await fn(...)` 后才让出执行权。

## 两阶段失败：先停止派发，再等待收尾

新的 worker 循环把“发现失败”和“向父级报告失败”分成两个阶段：

```ts
const workers = Array.from({ length: workerCount }, async () => {
	while (!hasError && !options.signal?.aborted) {
		const index = nextIndex++;
		if (index >= items.length) return;

		try {
			results[index] = await fn(items[index], index);
		} catch (error) {
			if (!hasError) {
				hasError = true;
				firstError = error;
			}
		}
	}
});

await Promise.all(workers);

if (hasError) throw firstError;
if (options.signal?.aborted) {
	const reason = options.signal.reason;
	throw reason instanceof Error ? reason : new Error("Parallel subagent execution aborted");
}
```

worker 捕获异常后不立即向外抛，而是设置共享停止状态并结束自己的循环。其他 worker 完成正在执行的任务后也会看到 `hasError`，不会领取下一项。因为 worker 自身没有拒绝，外层 `Promise.all(workers)` 会等到所有活跃 worker 结束。

等候结束后，调度器重新抛出 `firstError`。这保留了原始异常对象和堆栈，也保证父调用收到错误时，调度器已经没有已知的活跃任务。

父级中止走同一条收尾路径，但停止信号来自 `AbortSignal`。如果活跃任务正常返回，调度器在循环条件处停止；如果活跃任务因中止而抛错，首个错误会被保存。最终优先传播实际的 worker 错误，否则传播 `signal.reason`。调用方因此能保留 `AbortController.abort(reason)` 提供的错误对象。

## 与子进程中止协议如何配合

调度器本身不能结束进程。它只决定是否继续派发，以及何时允许父 Promise 结算。直接子进程仍由第 29 讲增加的 `bindAbortSignal()` 管理：

```text
父工具收到 abort
    │
    ├─ concurrency.ts：禁止 worker 领取排队任务
    │
    └─ process-control.ts：向每个活跃子进程发送 SIGTERM
                              │
                              ├─ 子进程退出：清理计时器
                              └─ 5 秒未退出：发送 SIGKILL

所有活跃 runSingleAgent() 结束
    │
    └─ concurrency.ts：传播首个错误或 signal.reason
```

入口在 `packages/coding-agent/examples/extensions/subagent/index.ts:599-630`。`runSingleAgent()` 获得同一个父 signal，调度器也获得这个 signal：

```ts
const results = await mapWithConcurrencyLimit(
	params.tasks,
	{ concurrency: MAX_CONCURRENCY, signal },
	async (t, index) => {
		const result = await runSingleAgent(
			ctx.cwd,
			agents,
			t.agent,
			t.task,
			t.cwd,
			undefined,
			signal,
			(partial) => {
				if (partial.details?.results[0]) {
					allResults[index] = partial.details.results[0];
					emitParallelUpdate();
				}
			},
			makeDetails("parallel"),
		);
		allResults[index] = result;
		emitParallelUpdate();
		return result;
	},
);
```

这里形成了清楚的职责分配：

- `index.ts` 解释工具参数，创建子进程任务并汇总结果；
- `concurrency.ts` 拥有排队索引、并发上限和父级结算时机；
- `process-control.ts` 把一个 `AbortSignal` 转换为操作系统进程信号，并负责计时器清理。

调度器没有导入 Agent、SessionManager 或 TUI。进程控制模块也不知道并行队列。依赖方向保持从入口组合层指向两个小型机制模块，没有形成反向调用。

## 哪些失败会停止整批任务

parallel 模式要区分“任务结果失败”和“调度过程失败”。二者的处理故意不同。

子 Agent 返回非零退出码、`stopReason: "error"` 或模型错误时，`runSingleAgent()` 通常仍会返回 `SingleResult`。parallel 模式把它记为某一项失败，其他任务继续运行，最终给父模型一份逐项汇总。这是批处理语义：一个侦察任务失败，不必取消其他独立侦察任务。

以下情况会以异常穿过 worker 边界：

- 用户中止后，`runSingleAgent()` 抛出 `Subagent was aborted`；
- 临时文件、进程创建或入口内部出现未转换的异常；
- 将来传入调度器的任务函数主动抛出致命错误。

这类错误意味着并行调用本身无法按约定完成。调度器停止排队任务，等活跃任务收尾，再原样抛出第一项异常。已经开始的任务不会被调度器强制取消；它们是否快速结束，取决于任务函数是否响应同一个 signal。

还有一个硬边界：如果操作系统拒绝终止某个进程，或活跃任务完全不响应中止且永不返回，`Promise.all(workers)` 会一直等待。五秒计时器只负责把直接子进程从 `SIGTERM` 升级为 `SIGKILL`，不是调度器总超时。要提供总超时，需要新增独立策略并定义超时后是否允许父级放弃所有权；本次没有偷偷加入这个决定。

## 测试如何证明结算顺序

`packages/coding-agent/test/subagent-concurrency.test.ts:18-108` 使用可手动释放的 Promise 控制 worker 完成顺序，没有启动真实模型或 CLI。

第一个用例同时放行两个 worker，确认活跃数不超过 2。它先完成索引 1，再完成索引 0，最终结果仍是 `[0, 10, 20, 30]`。这约束了并发上限和输入顺序。

第二个用例让索引 0 立即抛错，索引 1 保持运行。此时返回 Promise 仍未结算，索引 2、3 也没有启动。释放索引 1 后，调度器才以索引 0 的原始错误拒绝。

第三个用例在索引 0、1 运行时中止父 signal。索引 0 结束后，调度器仍等待索引 1；排队项没有启动。索引 1 结束后，返回 Promise 以 `signal.reason` 拒绝。

并发调度测试与进程中止测试合并运行，结果是 2 个测试文件、6 个用例全部通过。之后执行仓库规定的 `npm run check`，Biome、依赖固定检查、TypeScript、shrinkwrap、安装锁和浏览器 smoke check 全部通过。完整测试套件没有运行，遵守上游仓库禁止直接执行全量 Vitest 的规则。

## 文档是示例的分发清单

subagent README 采用逐文件软链接安装。增加 `concurrency.ts` 后，必须把它写入目录结构与安装命令，否则独立安装的 `index.ts` 会导入缺失模块。`packages/coding-agent/examples/extensions/subagent/README.md:13-44` 已同步这项变化，错误处理部分也写明 parallel 模式的停止与等待语义。`packages/coding-agent/CHANGELOG.md` 的 Unreleased/Fixed 记录了行为变化。

这个例子说明，模块拆分会改变可部署单元。仓库内 TypeScript 能解析导入，只能证明工作区完整；README 的安装清单决定用户复制出去的扩展是否完整。

## 回到 Pi 的整体模块边界

这次修改落在 coding-agent 的扩展示例里，没有把并行调度塞进 `packages/agent`。原因是两处并行解决的问题不同。

`packages/agent/src/agent-loop.ts:413-554` 管理一次模型响应产生的工具调用。它要保证工具结果按模型给出的 tool call 顺序回写，并处理 schema、hook、terminate 与事件顺序。subagent parallel 模式则在一个扩展工具内部限制多个 CLI 子进程，拥有独立的任务数量、流式详情和进程生命周期。把后者下沉到 Agent 循环会迫使通用运行时理解某个扩展的进程模型。

完成全部课程后，可以用四个问题快速定位修改位置：

1. 改的是供应商协议、模型目录、凭据或流事件，状态属于 `packages/ai`。
2. 改的是 turn、工具批次、steering、follow-up 或运行中止，先看 `packages/agent`。
3. 改的是 CLI、会话树、压缩、扩展、资源发现或编码工具，组合责任在 `packages/coding-agent`。
4. 改的是终端组件和差分重绘，属于 `packages/tui`；跨进程实例监督才进入实验性的 `packages/orchestrator`。

这套定位法不能只看目录名。还要追问状态由谁创建、谁能修改、失败后由谁结算。本次的 `AbortSignal` 来自父工具执行，coding-agent 扩展把它交给调度器和子进程控制器；调度器决定停止派发与等待，进程控制器决定信号升级，上层 Agent 负责把最终异常归一化为工具结果。每层只处理自己能观察到的事实。

## 最后的工程判断

Pi 的核心不是一个塞满策略的 Agent 类，而是一组边界清楚、状态寿命不同的层：模型请求快照、Agent 运行时、会话追加日志、应用协调状态、TUI 派生视图和外部子进程。修改源码时最容易犯的错，是拿某一层的状态代替另一层的事实。第 29 讲把“信号发出”误当成“进程退出”，本课处理的是另一种越界：父 Promise 已经失败，却没有等自己启动的任务结束。

源码阅读最终要落在这种判断上。函数位置可以搜索，真正费功夫的是确认所有权和结算点：谁启动工作，谁就要定义它何时算结束；谁向上报告失败，谁就要保证报告时没有失去管理的后台活动。
