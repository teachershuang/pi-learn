# Pi Agent 源码学习

这是一份围绕 [`earendil-works/pi`](https://github.com/earendil-works/pi) 编写的中文源码笔记。

仓库关心的是 Pi 怎么运行：一次 prompt 如何进入 Agent 循环，模型输出怎样变成工具调用，会话如何保存和恢复，上下文何时压缩，扩展与多实例又把边界推到了哪里。遇到文档与代码不一致的地方，以指定 commit 的源码和测试为准。

目前共 31 讲，已经全部完成。最后两讲包含源码修改练习；相关改动是本地练习分支上的实现，不是上游已经合并的功能。

## 从哪里开始

第一次接触 Pi，可以按编号顺序阅读。已经熟悉 Agent 工程时，下面几个入口更省时间：

- 想先看完整调用链：[第 01 讲：一条 prompt 的端到端主链](docs/01-prompt-lifecycle.md)
- 想弄清工具执行：[第 04 讲：工具调用从模型输出到执行结果](docs/04-tool-call-pipeline.md)
- 想研究模型接入：[第 08 讲：Models、Provider 与模型目录](docs/08-models-provider-catalog-dispatch.md)
- 想理解会话与上下文：[第 17 讲](docs/17-jsonl-session-tree.md)至[第 20 讲](docs/20-context-overflow-and-recovery.md)
- 想看安全边界：[第 23 讲：安全边界](docs/23-security-boundaries.md)
- 想研究多 Agent：[第 26 讲](docs/26-experimental-orchestrator.md)至[第 30 讲](docs/30-structured-subagent-coordination-and-architecture-review.md)

## 源码基线

笔记对应的上游源码版本固定为：

```text
repository: earendil-works/pi
branch:     main
commit:     5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf
describe:   v0.80.7-13-g5d9fedf73
```

固定 commit 是为了让路径、符号、调用顺序和测试结论可以复查。上游更新后，旧笔记不会自动适用于新版本。准备对照阅读时，可以先取出同一份源码：

```bash
git clone https://github.com/earendil-works/pi.git
cd pi
git checkout 5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf
```

仓库身份、workspace、构建入口与测试入口记录在[第 00 讲](docs/00-project-map.md)。

## 课程目录

### 一、工程地图

- [第 00 讲：项目身份、源码基线与工程地图](docs/00-project-map.md)
- [第 01 讲：一条 prompt 的端到端主链](docs/01-prompt-lifecycle.md)
- [第 02 讲：消息、上下文与状态不是同一个对象](docs/02-message-context-state.md)

### 二、低层 Agent 循环

- [第 03 讲：agentLoop 的双层循环](docs/03-agent-loop-control-flow.md)
- [第 04 讲：工具调用从模型输出到执行结果](docs/04-tool-call-pipeline.md)
- [第 05 讲：运行时状态、队列、中止与 settlement](docs/05-runtime-state-abort-settlement.md)
- [第 06 讲：失败路径与可观察结果](docs/06-failure-paths-and-observable-results.md)

### 三、模型与供应商接入

- [第 07 讲：pi-ai 的协议层](docs/07-pi-ai-protocol-layer.md)
- [第 08 讲：Models、Provider 与模型目录](docs/08-models-provider-catalog-dispatch.md)
- [第 09 讲：认证与凭据生命周期](docs/09-auth-credential-lifecycle.md)
- [第 10 讲：协议适配与流解析](docs/10-protocol-adapters-stream-parsing.md)
- [第 11 讲：模型选择、reasoning 与上下文预算](docs/11-model-selection-reasoning-context-budget.md)

### 四、Coding Agent 应用层

- [第 12 讲：CLI 启动与运行模式](docs/12-cli-startup-and-run-modes.md)
- [第 13 讲：资源发现与系统提示组装](docs/13-resource-discovery-system-prompt.md)
- [第 14 讲：内置编码工具](docs/14-builtin-coding-tools.md)
- [第 15 讲：扩展系统](docs/15-extension-system.md)
- [第 16 讲：AgentSession 的协调职责](docs/16-agent-session-coordination.md)

### 五、会话树与上下文管理

- [第 17 讲：JSONL 会话格式与 append-only tree](docs/17-jsonl-session-tree.md)
- [第 18 讲：从完整历史构建模型上下文](docs/18-context-projection.md)
- [第 19 讲：压缩与分支摘要](docs/19-compaction-and-branch-summary.md)
- [第 20 讲：上下文溢出与恢复](docs/20-context-overflow-and-recovery.md)

### 六、界面、嵌入与安全

- [第 21 讲：TUI 的差分渲染](docs/21-tui-differential-rendering.md)
- [第 22 讲：SDK、JSON 与 RPC](docs/22-sdk-json-rpc.md)
- [第 23 讲：安全边界](docs/23-security-boundaries.md)

### 七、演进方向、多 Agent 与源码修改

- [第 24 讲：测试体系与源码调试](docs/24-testing-and-source-debugging.md)
- [第 25 讲：新 AgentHarness](docs/25-agent-harness.md)
- [第 26 讲：实验性 Orchestrator 与多实例](docs/26-experimental-orchestrator.md)
- [第 27 讲：多 Agent 设计推演](docs/27-multi-agent-design.md)
- [第 28 讲：子 Agent 委派的同题比较与论文核对](docs/28-subagent-delegation-comparison.md)
- [第 29 讲：第一次完整源码修改](docs/29-first-source-change.md)
- [第 30 讲：并行子 Agent 的结构化收尾与架构复盘](docs/30-structured-subagent-coordination-and-architecture-review.md)

## 笔记怎么写

每篇笔记按功能组织，不照着目录逐文件复述。一个功能通常会交代它解决的问题、源码入口、短代码片段、运行流程、状态变化、失败路径和设计取舍。

文中会区分几类信息：

- 源码和测试能直接验证的行为；
- README、官方文档或论文给出的公开说明；
- 尚未落地的设计，以及根据现有边界作出的推断。

涉及 Hermes、Codex 或 Claude Code 的比较时，只比较同一个工程问题。论文也按原问题核对，不会因为概念相似就写成 Pi 已经实现。

## 适合什么读者

这些内容默认读者能阅读 TypeScript，知道大模型会流式返回文本或工具调用。无需提前熟悉 Pi 的代码结构。

如果目标只是安装和使用 Pi，上游仓库的 README 更直接。这里更适合需要追调用链、排查状态问题、接入模型，或者准备修改 Agent 工程的人。

## 参与

欢迎通过 Issue 或 Pull Request 纠正路径、符号、行号和行为判断。提交问题时，最好附上使用的上游 commit；如果结论来自运行结果，也请写明输入、观察到的事件或对应测试。

新增内容应保留事实边界。设计建议可以大胆，但要明确写成建议；没有源码或公开资料支撑的判断，应标为推断。

## 与上游的关系

这是社区学习仓库，不是 Pi 官方文档，也不代表上游维护者的观点。Pi 的安装、版本与发布信息请以上游仓库为准。
