# 第 00 讲：项目身份、源码基线与工程地图

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

“Pi Agent”不足以定位研究对象。同名项目、早期仓库名、发布包名和源码目录名并不一致。后续所有架构判断都依赖同一份源码基线，否则函数位置、包边界和测试结论很容易串版本。

本课程研究的是 `earendil-works/pi`。根 README 将它称为 Pi Agent Harness，根 `package.json` 的工程名仍是 `pi-monorepo`，发布到 npm 的核心包则使用 `@earendil-works` scope。这三个名字分别对应项目称呼、monorepo 工程名和发布包名，不是三个框架。

## 1. 源码基线解决什么问题

源码学习常见的误差不是“函数看错了”，而是拿不同时间的材料拼出一套不存在的实现。例如，当前源码已经有 `packages/orchestrator`，但根 README 的 All Packages 表格只列出 ai、agent、coding-agent 和 tui；`packages/agent/docs` 还写有若干 planned 或 in progress 的 Harness 设计。文档能说明公开意图，却不能单独证明某项能力已经接入主调用链。

因此，每篇笔记需要同时保留三类定位信息：

- 仓库身份：`earendil-works/pi`
- 完整 commit：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`
- 涉及的相对路径和符号名

tag 或版本号适合人阅读，但不能代替 commit。`v0.80.7-13-g5d9fedf73` 表示当前提交位于 `v0.80.7` 之后 13 个提交，最后的短哈希也不足以排除跨仓库歧义。

源码更新不会修改旧笔记的事实基线。新 commit 出现后，应先比较本课涉及的文件，再补充迁移说明或另写新版分析。

## 2. 这不是一个包，而是五层工程

根 `package.json:5-24` 定义了 npm workspaces 和仓库级命令：

```json
"workspaces": [
  "packages/*",
  "packages/coding-agent/examples/extensions/with-deps",
  "packages/coding-agent/examples/extensions/custom-provider-anthropic",
  "packages/coding-agent/examples/extensions/custom-provider-gitlab-duo",
  "packages/coding-agent/examples/extensions/sandbox",
  "packages/coding-agent/examples/extensions/gondolin"
],
"scripts": {
  "clean": "npm run clean --workspaces",
  "build": "cd packages/tui && npm run build && cd ../ai && npm run build && cd ../agent && npm run build && cd ../coding-agent && npm run build && cd ../orchestrator && npm run build",
  "check": "biome check --write --error-on-warnings . && npm run check:pinned-deps && npm run check:ts-imports && npm run check:shrinkwrap && npm run check:install-lock:coding-agent && tsgo --noEmit && npm run check:browser-smoke",
  "test": "npm run test --workspaces --if-present"
}
```

`packages/*` 当前展开为五个包。它们不是地位相同的功能集合，而是一条有方向的依赖链。

| 包 | 解决的问题 | 主要入口 | 边界 |
| --- | --- | --- | --- |
| `packages/ai` | 统一模型、消息、认证和供应商流式协议 | `packages/ai/src/index.ts`、`packages/ai/src/models.ts`、`packages/ai/src/api/*` | 不执行 agent 循环，不管理编码会话 |
| `packages/agent` | 运行模型与工具的循环，维护 Agent 内存状态并发出事件 | `packages/agent/src/agent.ts`、`packages/agent/src/agent-loop.ts` | 不处理 CLI、项目资源和 TUI |
| `packages/coding-agent` | 把 Agent 组合成可使用的编码代理 | `packages/coding-agent/src/cli.ts`、`packages/coding-agent/src/main.ts`、`packages/coding-agent/src/core/sdk.ts` | 依赖 ai、agent 和 tui，不实现供应商底层协议 |
| `packages/tui` | 终端输入、组件和差分渲染 | `packages/tui/src/index.ts`、`packages/tui/src/tui.ts`、`packages/tui/src/terminal.ts` | 不决定模型调用和会话语义 |
| `packages/orchestrator` | 监督多个 RPC 模式的 Pi 子进程 | `packages/orchestrator/src/cli.ts`、`packages/orchestrator/src/supervisor.ts` | 仍是实验包，不等于完整多 Agent runtime |

根构建脚本给出的顺序是 `tui → ai → agent → coding-agent → orchestrator`。这不是完整的架构证明，但能暴露构建依赖：coding-agent 需要前面三个包，orchestrator 又依赖 coding-agent。真正的运行时依赖仍要到各包 `package.json` 和 import 关系中确认。

仓库级目录承担另外几类工作：

- `.pi/` 保存本仓库自用的扩展、prompt 和 skill。
- `.github/` 保存 CI、发布、贡献者门禁和依赖审计工作流。
- `scripts/` 保存模型生成、依赖锁校验、发布和统计脚本。
- `packages/*/docs` 是公开使用文档，`packages/agent/docs` 还包含尚未全部落地的设计稿。

基线快照共有 1042 个受 Git 跟踪的文件。代码和测试主要集中在 `packages`，不能根据根目录文件少就判断工程简单。

## 3. 入口有三种含义

阅读“入口代码”时，需要先说明是哪一种入口。

### 发布入口

`packages/coding-agent/package.json:1-21` 同时暴露命令行和库入口：

```json
{
  "name": "@earendil-works/pi-coding-agent",
  "version": "0.80.7",
  "description": "Coding agent CLI with read, bash, edit, write tools and session management",
  "type": "module",
  "piConfig": {
    "configDir": ".pi"
  },
  "bin": {
    "pi": "dist/cli.js"
  },
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js"
    },
    "./rpc-entry": {
      "import": "./dist/rpc-entry.js"
    }
  }
}
```

安装后执行 `pi`，进入的是构建产物 `dist/cli.js`。其他 Node.js 程序 import 这个包时，进入 `dist/index.js`；orchestrator 等外部进程需要 RPC 模式时，还可以使用单独的 `rpc-entry`。

### 源码入口

构建产物对应的源码入口是 `packages/coding-agent/src/cli.ts`。它设置进程名称和环境标记、配置 HTTP dispatcher，然后调用 `main(process.argv.slice(2))`。`main()` 才负责参数解析、会话选择、资源和模型服务创建，以及 interactive、print、json、RPC 模式分流。

从源码运行的 `pi-test.sh`、`pi-test.ps1` 和 `pi-test.bat` 是开发入口。它们让 CLI 使用仓库源码，但不会改变调用链的所有权：业务组合仍从 coding-agent 开始。

### 库入口

`packages/ai/src/index.ts`、`packages/agent/src/index.ts` 和 `packages/coding-agent/src/index.ts` 是公共 API 的汇总面。源码内部文件存在，不代表发布包承诺用户可以直接 import；判断公共能力要再看 `package.json` 的 `exports`。

这一区分会贯穿后续课程。源码路径解释实现，package exports 解释外部契约，CLI bin 解释程序启动位置。

## 4. 当前主链与实验分支

基线 commit 的 coding-agent 主链可以先压缩成下面一行：

```text
cli.ts → main() → createAgentSession() → AgentSession → Agent → agentLoop → ModelRuntime/Models → Provider
```

这只是导航图，不在第 00 讲展开内部步骤。现在需要确认的是状态所有权的大致分层：

- `Agent` 持有一次进程中的消息、模型、工具、队列和流式状态。
- `AgentSession` 把 Agent 与会话、设置、资源、扩展、压缩和重试组合起来。
- `SessionManager` 把会话写入 append-only JSONL tree。
- `ModelRuntime/Models` 解析模型和认证，再把请求交给具体 Provider。
- 运行模式订阅事件，将同一套 AgentSession 投影为 TUI、普通文本、JSON 或 RPC。

`packages/agent/src/harness/agent-harness.ts` 代表另一条正在推进的实现路径。它不是当前 CLI 的顶层对象。包内设计文档明确列出 planned 和 in progress 项，因此课程会在第 25 讲单独比较两套实现，前面的主链分析不会把 Harness 的目标状态套到 AgentSession 上。

`packages/orchestrator/README.md:1-3` 也给出了明确限制：包处于实验阶段，CLI、API 和行为可能变化或移除。当前实现通过子进程和 RPC 管理多个 Pi 实例，先把它理解为多实例监督层更准确。

## 5. 构建、检查和测试不是同一件事

根脚本已经把三类动作分开：

- `npm run build` 生成各包的 `dist`，检查构建顺序和产物生成。
- `npm run check` 会运行格式化、依赖约束、TypeScript 检查、shrinkwrap/install-lock 校验和 browser smoke。
- 测试验证具体运行行为；仓库规则要求常规非 e2e 测试通过根 `test.sh` 运行。

`test.sh:1-23` 先建立隔离边界：

```bash
#!/usr/bin/env bash
set -e

AUTH_FILE="$HOME/.pi/agent/auth.json"
AUTH_BACKUP="$HOME/.pi/agent/auth.json.bak"

cleanup() {
    if [[ -f "$AUTH_BACKUP" ]]; then
        mv "$AUTH_BACKUP" "$AUTH_FILE"
        echo "Restored auth.json"
    fi
}
trap cleanup EXIT

if [[ -f "$AUTH_FILE" ]]; then
    mv "$AUTH_FILE" "$AUTH_BACKUP"
    echo "Moved auth.json to backup"
fi

export PI_NO_LOCAL_LLM=1
```

脚本随后清除各供应商 API key，再执行 workspace tests。原因不是“测试不需要模型”这么简单，而是仓库里存在会在认证信息可用时启用的 e2e 路径。常规测试必须主动隔离真实凭据，避免一次本地验证变成付费网络调用。

基线共有 316 个 `*.test.ts` 或 `*.spec.ts`：agent 16 个、ai 98 个、coding-agent 175 个、tui 27 个。orchestrator 当前没有包内测试。不同包使用的测试方式也不完全相同：agent、ai、coding-agent 主要用 Vitest，tui 的包脚本使用 Node test runner。

测试失败需要先按位置分类：

| 失败位置 | 首先检查 | 可能含义 |
| --- | --- | --- |
| build | TypeScript 编译、资源复制、包构建顺序 | 产物无法生成或导出配置有误 |
| check | 格式、类型、依赖固定、生成文件一致性 | 工程约束被破坏，不一定是运行逻辑错误 |
| 定向单测 | 输入、事件顺序、状态或持久化结果 | 实现行为与测试契约不一致 |
| e2e | 凭据、网络、供应商行为、真实模型 | 可能是环境或外部服务问题 |

只看到“测试失败”还不能判断是上游缺陷。后续修改源码时，会根据改动面选择定向测试，再决定是否扩大验证范围。

## 6. 第 00 讲留下的工程模型

Pi 可以先看成三层主干和两层外壳：

```text
供应商协议层：packages/ai
Agent 运行层：packages/agent
编码代理应用层：packages/coding-agent

交互外壳：packages/tui
多实例外壳：packages/orchestrator（实验性）
```

这个模型故意省略了许多细节，但依赖方向没有颠倒。模型供应商不知道会话树，agent loop 不知道终端如何重绘，TUI 也不负责决定工具是否继续执行。coding-agent 位于组合位置，所以多数端到端问题最终会跨过它。

下一讲从 `packages/coding-agent/src/cli.ts` 出发，只追踪一条 prompt 如何走到 provider，再沿事件回到会话和界面。届时才展开各层之间真正传递的数据。

## QA

### `pi-monorepo`、Pi Agent Harness 和 `@earendil-works/pi-coding-agent` 是什么关系？

它们分别是根工程名、项目称呼和 npm 发布包名。判断源码身份时以 Git 远端和 commit 为准，判断安装入口时看包名与 `bin`，判断代码组织时看 workspace。

### 为什么现在不直接从 `AgentHarness` 开始？

因为基线 commit 的 coding-agent 主链实例化的是 `Agent` 和 `AgentSession`。`AgentHarness` 有独立测试和设计文档，但仍有明确未完成项。先读 Harness 会把未来设计误认成当前 CLI 行为。

### 根 `npm test` 为什么不是默认验证命令？

它会递归运行 workspace tests，而仓库含有可能受本机凭据和 endpoint 环境变量影响的 e2e 测试。根 `test.sh` 先隔离认证和本地模型环境，再调用测试，是常规非 e2e 验证入口。
