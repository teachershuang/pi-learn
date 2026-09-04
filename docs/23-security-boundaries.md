# 第 23 讲：安全边界、信任决策与供应链

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的安全模型建立在一个直接的前提上：进程拥有启动用户的权限，内置工具和扩展默认也在这个权限范围内运行。`project trust`、工具调用前的确认、容器隔离和发布链校验解决的是四类不同问题，不能用其中一层代替另一层。

```text
仓库与配置文件
  │ project trust：决定项目资源是否进入进程
  ▼
Pi 进程与扩展
  │ tool_call hook：决定某次 Agent 工具调用是否继续
  ▼
文件、子进程、网络、凭据
  │ OS / 容器 / VM 策略：限制进程实际能触达的资源
  ▼
外部系统

依赖锁、摘要、CI、provenance 位于另一条轴线上：
它们回答“运行的产物从哪里来、是否发生漂移”，不负责限制产物运行后的权限。
```

这个模型先区分三个容易混淆的词：

| 机制 | 决策对象 | 决策者 | 拒绝后的结果 | 没有解决的问题 |
|---|---|---|---|---|
| project trust | 项目级设置、扩展、技能、包等启动输入 | CLI 参数、已保存决定、全局默认、扩展或用户 | 跳过受保护的项目资源 | 不限制内置工具和已加载代码的权限 |
| approval / guard | 一次工具调用或会话动作 | 扩展策略或用户 | 返回错误工具结果或取消动作 | 不改变进程权限，也覆盖不了扩展直接调用 Node API |
| sandbox / isolation | 进程能访问的文件、网络、凭据和子进程 | 操作系统、容器、VM 或外部策略执行器 | 系统层拒绝访问 | 不判断操作在业务上是否合理 |

## 1. `project trust` 是启动输入门，不是运行沙箱

### 解决的问题

进入陌生仓库时，项目可能携带 `.pi/settings.json`、扩展代码、技能、提示模板、主题和包声明。若 Pi 在询问前加载这些资源，信任确认本身已经太迟。源码采用两阶段加载：先只加载用户级与命令行指定资源，由这些已在当前信任域中的扩展参与信任判断；取得决定后，再用最终信任状态重载设置和资源。

官方边界写在 `packages/coding-agent/docs/security.md:5-35`。其中最关键的限制是：project trust 只管项目资源加载，不会在启动后收紧 read、write、edit、bash 的权限。

### 判断优先级

入口是 `packages/coding-agent/src/core/project-trust.ts:44-96` 的 `resolveProjectTrusted()`：

```ts
export async function resolveProjectTrusted(options: ResolveProjectTrustOptions): Promise<boolean> {
	const { cwd, explicitTrust, trustStore, settingsManager, preTrustExtensions, createContext } = options;

	if (explicitTrust !== undefined) return explicitTrust;

	if (!hasTrustRequiringProjectResources(cwd)) return true;

	if (preTrustExtensions.extensions.length > 0) {
		const ctx = createContext();
		const { result, errors } = await emitProjectTrustEvent(
			preTrustExtensions,
			{ type: "project_trust", cwd },
			ctx,
		);
		for (const error of errors) ctx.ui.notify(`Project trust hook failed: ${error.error}`, "warning");
		if (result?.trusted && result.trusted !== "undecided") {
			const trusted = result.trusted === "yes";
			if (result.remember) trustStore.setDecision(cwd, trusted);
			return trusted;
		}
	}

	const storedDecision = trustStore.getDecision(cwd);
	if (storedDecision !== undefined) return storedDecision;

	const defaultPolicy = settingsManager.getProjectTrustDefault();
	if (defaultPolicy === "always") return true;
	if (defaultPolicy === "never") return false;

	const ctx = createContext();
	if (!ctx.hasUI) return false;
	return (await promptForProjectTrust(ctx, cwd, trustStore)) ?? false;
}
```

决策顺序是显式运行参数、预信任扩展、保存记录、全局默认、交互询问。非交互模式没有 UI 且策略仍为 `ask` 时返回 `false`，属于默认拒绝。`--approve` / `--no-approve` 只覆盖本次 project trust，不是逐工具审批开关。

`ProjectTrustStore` 将规范化目录写入用户 agent 目录下的 `trust.json`。`packages/coding-agent/src/core/trust-manager.ts:39-95` 允许当前目录继承最近父目录的决定；选择只信任当前目录会写子项，选择信任父目录则删除多余的子项。读取和写入都用文件锁，避免多个 Pi 进程覆盖彼此的决定。

### 哪些文件触发询问

`hasTrustRequiringProjectResources()` 在 `packages/coding-agent/src/core/trust-manager.ts:177-206` 检查：

- 当前项目 `.pi/settings.json`；
- `.pi/extensions`、`.pi/skills`、`.pi/prompts`、`.pi/themes`；
- `.pi/SYSTEM.md`、`.pi/APPEND_SYSTEM.md`；
- 当前目录到仓库根之间的 `.agents/skills`。

只有一个空的 `.pi` 目录不会触发询问。测试 `packages/coding-agent/test/trust-manager.test.ts:11-67` 验证了父目录继承、子目录覆盖、空目录和用户主目录 `.agents/skills` 的排除规则。

### 两阶段加载中的状态变化

`packages/coding-agent/src/core/resource-loader.ts:330-460` 的启动顺序可以压缩为：

```text
SettingsManager(projectTrusted=false)
  → bootstrap load：用户扩展 + CLI -e 扩展
  → project_trust 事件 / trust store / 用户选择
  → SettingsManager.setProjectTrusted(result)
  → 重新读取设置
  → 解析并按需安装包
  → final load：加入获准的项目资源
```

`packages/coding-agent/src/main.ts:600-705` 还按规范化 cwd 缓存本次进程中的信任结果。会话切换到另一个 cwd 时会单独解析；同一 cwd 不重复弹窗。

拒绝信任后，`packages/coding-agent/src/core/resource-loader.ts:330-460` 与 `packages/coding-agent/src/core/package-manager.ts:2303-2419` 跳过项目设置、`.pi` 资源、项目包和项目 `.agents/skills`。项目包存储也有二次检查：`PackageManager.assertProjectTrustedForScope()` 在未信任时拒绝访问 `.pi/npm` 和 `.pi/git`。

### 项目上下文是例外

`AGENTS.md` 和 `CLAUDE.md` 不在上述受保护列表。`packages/coding-agent/test/resource-loader.test.ts:348-424` 明确验证：`projectTrusted: false` 时，项目 `SYSTEM.md`、扩展、技能、提示和主题被跳过，项目 `AGENTS.md` 仍进入上下文。

这是有意的产品取舍。项目上下文用于告诉模型如何理解和操作仓库；若未信任时完全不读，Pi 会失去项目约定。代价是陌生仓库仍然拥有提示注入面。它不能直接成为 Node.js 代码，却可能诱导模型调用仍有完整用户权限的内置工具。顶层 `SECURITY.md:11-22` 因此把用户可写的工作区、shell 环境、配置和主目录视作同一个信任边界，并把仓库中的指令注入列为使用者需要承担的风险。

## 2. 工具审批只能拦住经过调用链的动作

### 触发位置与失败结果

Pi core 提供 `beforeToolCall`，coding-agent 把扩展的 `tool_call` 事件接到这里。`packages/coding-agent/src/core/agent-session.ts:441-469` 在每次工具参数完成 schema 校验后调用扩展 runner；`packages/coding-agent/src/core/extensions/runner.ts:885-905` 按扩展和 handler 顺序执行，遇到第一个 `{ block: true }` 就停止。

`packages/agent/src/agent-loop.ts:618-664` 决定阻断如何反馈给模型：

```ts
const preparedToolCall = prepareToolCallArguments(tool, toolCall);
const validatedArgs = validateToolArguments(tool, preparedToolCall);
if (config.beforeToolCall) {
	const beforeResult = await config.beforeToolCall(
		{
			assistantMessage,
			toolCall,
			args: validatedArgs,
			context: currentContext,
		},
		signal,
	);
	if (signal?.aborted) {
		return { kind: "immediate", result: createErrorToolResult("Operation aborted"), isError: true };
	}
	if (beforeResult?.block) {
		return {
			kind: "immediate",
			result: createErrorToolResult(beforeResult.reason || "Tool execution was blocked"),
			isError: true,
		};
	}
}
```

被拒绝的调用不会执行工具，但仍生成 `isError: true` 的 tool result。模型能看到拒绝原因，可以改用安全参数、解释需要或结束当前方案。扩展 handler 自身抛错时，`AgentSession` 也按失败关闭处理，将异常变成阻断，而不是继续执行原工具。

### approval 由扩展实现

默认安装并不等于每次 bash、write 都会询问。`packages/coding-agent/examples/extensions/permission-gate.ts:10-34` 只是一个可选扩展示例：

```ts
export default function (pi: ExtensionAPI) {
	const dangerousPatterns = [/\brm\s+(-rf?|--recursive)/i, /\bsudo\b/i, /\b(chmod|chown)\b.*777/i];

	pi.on("tool_call", async (event, ctx) => {
		if (event.toolName !== "bash") return undefined;
		const command = event.input.command as string;
		if (!dangerousPatterns.some((pattern) => pattern.test(command))) return undefined;

		if (!ctx.hasUI) {
			return { block: true, reason: "Dangerous command blocked (no UI for confirmation)" };
		}
		const choice = await ctx.ui.select(`Dangerous command:\n\n${command}\n\nAllow?`, ["Yes", "No"]);
		return choice === "Yes" ? undefined : { block: true, reason: "Blocked by user" };
	});
}
```

它只识别三组正则，只拦 bash 工具，并且依赖扩展保持加载。命令换一种写法、改走其他工具或调用扩展自带能力，都可能不经过这条规则。会话 clear、switch、fork 的确认则使用另一组 `session_before_*` 事件，见 `packages/coding-agent/examples/extensions/confirm-destructive.ts:10-58`。这两类确认的触发点和取消结果都不同。

### 为什么这不是 sandbox

扩展通过 `jiti` 在 Pi 进程内被 import，导出的工厂函数立即执行，见 `packages/coding-agent/src/core/extensions/loader.ts:390-466`。扩展 API 还直接提供 `execCommand`，也允许注册工具、provider、命令和事件处理器。加载后的扩展拥有 Node.js 进程能够取得的文件、环境变量、网络和子进程能力。

因此 `tool_call` guard 只覆盖 Agent loop 发起的工具调用。恶意或有缺陷的扩展可以直接使用 `node:fs`、`node:child_process` 或网络 API，不需要先制造一个工具调用。guard 与被保护代码同进程、同权限，也不是独立的策略执行器。

内置文件工具同样没有工作区边界。`packages/coding-agent/src/core/tools/path-utils.ts:44-55` 明确接受绝对路径；`packages/coding-agent/src/core/tools/write.ts:195-224` 将路径解析后交给 `fs.writeFile`。能够写到哪里由启动用户和操作系统决定，不由 cwd 决定。

## 3. 凭据保护的是落盘形式，不是进程内秘密

### 存储与解析

`AuthStorage` 在用户 agent 目录保存 `auth.json`。`packages/coding-agent/src/core/auth-storage.ts:81-169` 创建父目录时使用 `0700`，写入临时文件和目标文件时使用 `0600`，并通过文件锁串行化并发修改。`RuntimeAuthStorage` 的命令行覆盖只存在内存中，不会写回磁盘，见 `packages/coding-agent/src/core/runtime-credentials.ts:9-48`。

这些措施减少其他本机账号读取文件和并发写坏 JSON 的风险，但不能防止同一用户权限下的 Pi、扩展、调试器或被攻陷的本机进程读取秘密。

API key 支持三种值：字面量、环境变量名、以 `!` 开头的命令。`packages/coding-agent/src/core/resolve-config-value.ts:178-253` 的命令型解析会启动 shell，等待最多 10 秒，并把 stdout 去空白后作为秘密：

```ts
export async function resolveConfigValueUncached(value: string, env?: NodeJS.ProcessEnv): Promise<string | undefined> {
	if (!value.startsWith("!")) return resolveLiteralOrEnv(value, env);

	const command = value.slice(1);
	if (!command.trim()) return undefined;

	try {
		const stdout = await executeCommand(command, env);
		return stdout.trim() || undefined;
	} catch {
		return undefined;
	}
}

export function resolveConfigValue(value: string, env?: NodeJS.ProcessEnv): Promise<string | undefined> {
	let promise = credentialCache.get(value);
	if (!promise) {
		promise = resolveConfigValueUncached(value, env);
		credentialCache.set(value, promise);
	}
	return promise;
}
```

常规解析按进程缓存，避免每次请求都执行密码管理器命令；部分需要动态刷新的 provider 配置会使用 uncached 版本。命令失败、超时或输出为空时返回 `undefined`，随后由模型鉴权路径表现为缺少 credential，而不是把 shell stderr 当作 key。

命令型 credential 是便利机制，也扩大了配置文件的执行语义。它不受 project trust 控制，因为 `auth.json` 属于用户配置域。若该文件已经被同一用户权限下的其他程序篡改，Pi 的模型启动过程会执行其中的命令；这正是顶层 `SECURITY.md` 将用户可写本地状态划入同一信任边界的原因。

### 凭据会在哪里出现

- provider 请求必须在进程内取得 API key 或 OAuth token；已加载扩展也与这些请求代码处于同一进程信任域。
- 把整个 Pi 放进 Docker 时，provider key 必须进入容器；容器成为新的秘密暴露面。
- 只把工具委派给隔离环境时，Pi 和 auth 留在宿主，能减少工具进程接触 provider key 的机会。
- 将宿主 agent 目录直接挂进容器会同时暴露 `auth.json`、session 和其他用户级配置，不宜作为默认做法。

文件 mode 不是加密，环境变量也不是隔离。真正的最小暴露需要从进程边界、挂载、网络策略和短期凭据共同设计。

## 4. 隔离必须落到执行边界

`packages/coding-agent/docs/containerization.md:1-111` 给出三种部署形态。它们改变的不是模型提示，而是工具或 Pi 进程实际运行的位置。

### Gondolin：宿主保留 Pi，委派内置工具

`packages/coding-agent/examples/extensions/gondolin` 扩展覆盖内置工具和 `!` 命令，将执行委派给隔离环境。Pi、会话和 provider auth 仍在宿主。项目目录挂载为 `/workspace` 后，隔离环境内的写入会反映到宿主项目；这是一条明确授权的写通道，并非只读保护。

边界还取决于覆盖范围：Gondolin 处理 Pi 内置工具与用户 bash，其他扩展注册的工具默认仍在宿主执行，除非它们主动委派。判断“已沙箱化”时必须逐条核对所有执行入口，不能只看 bash 的落点。

### Docker：整个进程进入容器

整进程容器化把 Pi、扩展和内置工具放进同一容器。约束来自容器用户、capability、seccomp、网络和挂载。若以当前用户可写方式 bind mount 工作区，Agent 仍可修改工作区；它只是难以越过容器边界触达未挂载的宿主资源。

这种方案需要把模型凭据注入容器。使用独立 named volume 保存 agent home，可以避免直接暴露宿主 agent 目录；代价是认证、会话和配置要在容器侧单独管理。

### OpenShell：策略化沙箱

OpenShell 模式将完整 Pi 放进本地或远程沙箱，并对文件、进程、网络、凭据和推理访问应用策略。远程模式会复制项目文件，写入不会天然回流到原工作区；这与 bind mount 的状态语义不同。

三种方案都需要继续回答：哪些目录可写、能否联网、允许启动哪些进程、凭据在哪里注入、扩展工具是否经过隔离。容器名称本身不是答案。

## 5. 运行时下载与第三方扩展是独立供应链

### `fd` / `rg` 自动下载

交互模式启动时，`packages/coding-agent/src/modes/interactive/interactive-mode.ts:675-683` 并行调用 `ensureTool("fd")` 和 `ensureTool("rg")`。`packages/coding-agent/src/utils/tools-manager.ts:324-369` 先查用户 bin 目录和 PATH；缺失且未设置 `PI_OFFLINE` 时，从 GitHub release 下载。

版本与安装流程位于 `packages/coding-agent/src/utils/tools-manager.ts:245-315`：大多数平台先请求 latest release，拼接资产 URL，下载并解压，找到同名二进制后移入 agent bin 目录。源码没有读取 release checksum、签名或固定摘要；安装后的本地二进制再次启动时也只按路径复用。网络超时、平台不支持、下载或解压失败会返回 `undefined` 并显示警告，Pi 继续运行，但依赖这些工具的补全或搜索能力会降级。

这条路径与 Pi 自己的 npm lockfile 无关。若部署环境要求可复现和离线审计，应预装经过组织校验的 `fd` / `rg`，固定 PATH，并设置 `PI_OFFLINE=1`，避免运行时获取 latest 资产。

### 扩展包安装

project trust 允许加载项目包，也允许安装缺失包。`packages/coding-agent/src/core/package-manager.ts:1240-1298` 依次解析 npm、git 和本地来源；缺失时由调用方选择 install、skip 或 error。安装后，包中声明的扩展仍会在 Pi 进程内执行。

`packages/coding-agent/src/core/package-manager.ts:1730-1801` 的 npm 安装参数用于处理 peer dependency，并没有统一添加 `--ignore-scripts`。git 来源在 clone/checkout 后若存在 `package.json`，也会执行 package manager install，见同文件 `1820-1888`。这意味着扩展包的生命周期脚本可能在扩展模块加载之前执行。project trust 是对整个项目来源的授权，不是对每个包、每个版本或每条 install script 的单独审查。

固定扩展版本或 git commit 可以降低无意漂移，但不能把有缺陷或恶意代码变成安全代码。第三方扩展需要同时评估来源、版本、安装脚本、运行权限和更新策略。

## 6. Pi 自身的依赖锁与发布链

上游仓库对自身发布产物采用了比运行时第三方扩展更严格的流程。这些机制主要保证可复现、可审查和发布来源。

### 精确依赖与两类发布锁

根 `package.json` 使用精确外部依赖版本。`scripts/check-pinned-deps.mjs:1-63` 递归检查各 workspace 的 registry 依赖，发现 `^`、`~` 或范围版本就失败；内部 workspace 和非 registry 来源另行处理。

coding-agent 有两份生成物：

- `scripts/generate-coding-agent-shrinkwrap.mjs` 从根 lockfile 提取发布包依赖闭包，保留 `resolved` 和 `integrity`，拒绝本地 link、缺失依赖和未审核的 install script；
- `scripts/generate-coding-agent-install-lock.mjs:1-19,260-432` 为二进制安装器生成独立根包与 lockfile，检查内部包版本一致、依赖可解析、平台可选包齐全，并只允许两个写明理由的 install-script 包。

allowlist 既检查新增脚本，也检查陈旧条目：依赖不再存在时要求移除对应豁免。`--check` 模式逐字比较生成结果，避免手工修改生成锁而不更新来源。

根 lockfile 的 pre-commit 检查位于 `scripts/check-lockfile-commit.mjs:1-120`。它允许仅 workspace 元数据变化，其他锁文件变化需要显式设置 `PI_ALLOW_LOCKFILE_CHANGE`，并提示审查 resolved、integrity、版本和 install scripts。环境变量是带审计意图的逃生口，不是密码学强制。

### 发布顺序

`scripts/release.mjs:1-208` 要求工作区干净，更新版本与 changelog，重生成模型目录和两类 lock，运行 check 与测试，然后提交、打 tag 并推送。tag 触发 `.github/workflows/build-binaries.yml`：

```text
固定提交 SHA 的 Actions
  → npm ci --ignore-scripts
  → build / check / test
  → 构建各平台二进制与安装锁
  → 生成并复核 SHA256SUMS
  → 暂存 draft GitHub Release
  → npm trusted publishing + --provenance
  → npm 成功后公开 GitHub Release
  → 任一步失败则清理 draft
```

发布 workflow 默认 `permissions: {}`，各 job 只申请需要的权限；npm job 使用 `id-token: write`，`scripts/publish.mjs:98-124` 以 `npm publish --provenance --ignore-scripts` 发布四个同版本 workspace 包。公开 GitHub Release 放在 npm 发布成功之后，避免公开一套只完成一半的资产。

`.github/workflows/npm-audit.yml` 每日用 `npm ci --ignore-scripts` 后运行生产依赖 audit 和 registry signature audit。它能发现已知漏洞或签名问题，不能证明依赖在业务逻辑上无恶意，也不能覆盖运行时另行下载的扩展与 `fd` / `rg`。

### 这些防线的边界

| 防线 | 能发现或限制 | 仍需另行处理 |
|---|---|---|
| 精确版本与 lock integrity | 依赖版本漂移、tarball 与锁不一致 | 已锁定版本本身的恶意逻辑 |
| install-script allowlist | 发布闭包中新出现的安装脚本 | 模块 import 后的任意运行时代码 |
| CI check / test | 已编码的不变量和回归 | 测试未覆盖的行为 |
| SHA256SUMS | workflow 阶段间资产损坏或替换 | 用户是否从可信渠道取得摘要 |
| npm provenance | npm 包与构建身份、来源的关联 | 包运行时权限和业务安全性 |
| project trust | 陌生项目的主动资源加载 | `AGENTS.md` 提示注入、内置工具权限 |

## 7. 失败路径与安全含义

### 拒绝项目信任

项目资源被跳过，用户级资源和命令行显式扩展仍可运行，项目 `AGENTS.md` 仍可进入上下文。用户可以继续使用内置工具，且工具保持启动用户权限。拒绝不是只读模式。

### 信任 hook 抛错

`emitProjectTrustEvent()` 收集扩展错误并继续寻找其他决定；错误通过 UI warning 呈现。没有扩展给出决定时，流程继续查保存记录和默认策略，不会把 hook 异常自动当作信任。

### 工具 guard 抛错或拒绝

拒绝会生成错误 tool result；handler 抛错也阻断当前调用。会话没有因此获得新的 OS 限制，后续调用仍按同一扩展策略重新判断。

### credential 命令失败

超时、非零退出或空输出解析为 `undefined`，模型接入层随后报告缺少鉴权。错误不会回退成执行 stderr，也不会把失败输出写入 auth 文件。

### 自动下载失败

`ensureTool()` 返回 `undefined`，交互模式继续启动。失败属于能力降级。若安全策略要求禁止网络下载，不能依赖下载恰好失败，应设置 offline 并预装工具。

### 发布链中途失败

release 脚本和 workflow 大多采用失败即停止。GitHub 资产先作为 draft 暂存；npm 或后续发布失败时 cleanup job 删除 draft。npm 多包发布仍是逐包操作，若某包已发布而后续包失败，重跑时 `publish.mjs` 会跳过已存在版本并继续验证；这减少重复发布错误，但不能提供 registry 级原子事务。

## 8. 从源码验证安全边界

本讲对应的定向测试分成四组：

- `trust-manager.test.ts`：目录继承与触发资源；
- `settings-manager.test.ts`、`resource-loader.test.ts`：未信任时跳过项目设置和主动资源，同时保留项目上下文；
- `extensions-runner.test.ts` 与 agent loop 测试：首个信任决定、工具调用阻断及错误结果；
- `auth-storage.test.ts`、`resolve-config-value.test.ts`：权限、并发写、环境变量与命令型 credential 的缓存和失败。

这些测试验证的是实现契约，不证明当前机器已经形成隔离环境。容器挂载、网络、用户身份、凭据注入和外部策略必须在实际部署中单独验收。

## 9. 设计取舍

Pi 选择“高扩展能力、低内建策略”的内核：工具接受绝对路径，扩展与主进程共享能力，用户可用 hook 自定义确认，也可把工具或整个进程放进外部沙箱。这样便于接入 SSH、容器、自定义 provider 和任意开发工作流，核心无需预设一套无法覆盖所有环境的权限语言。

代价同样清楚：默认安全性高度依赖启动位置、用户权限和所加载代码。project trust 只把最早的项目代码执行点推迟到用户决定之后；approval 只在选定调用点增加决策；依赖锁只约束取得哪份代码。真正面对不可信仓库或不可信生成内容时，最终边界仍要由独立用户、容器、VM、网络策略和最小凭据来执行。
