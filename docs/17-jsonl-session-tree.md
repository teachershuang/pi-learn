# 第 17 讲：JSONL 会话账本与 append-only tree

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的 session 文件不是“下一轮要发给模型的 messages 数组”，也不是每次保存一份完整状态快照。它是一份 JSONL 事件账本：第一行声明 session，后续每行记录一个不可变 entry；`id` 和 `parentId` 把这些 entry 连接成树，当前 `leaf` 决定从树中选择哪条路径。真正恢复给 Agent 的消息，是对这条路径再次投影得到的结果。

```text
JSONL 文件顺序             parentId 关系                 当前投影

header
A: user                    A ─► B ─► C                  A → B → D
B: assistant                     ├► C                   （leaf = D）
C: user                          └► D
D: user（后来追加）

文件保留 C；模型上下文不包含 C。
```

这里有三种不同的“顺序”：文件行的追加顺序、树上的父子顺序、当前 leaf 到根的路径顺序。线性对话里三者恰好一致；一旦回到旧节点产生分支，只按文件末尾读取 messages 就会得到错误上下文。

## 1. Header 确定文件身份，不参与会话树

### 解决的问题

恢复 session 前，Pi 需要知道文件格式版本、session 身份、创建时间和原工作目录。跨项目 fork 还要保留来源关系。这些是整份文件的元数据，不应伪装成一条对话事件。

`SessionHeader` 定义在 `packages/coding-agent/src/core/session-manager.ts:30-44`：

```ts
export const CURRENT_SESSION_VERSION = 3;

export interface SessionHeader {
	type: "session";
	version?: number;
	id: string;
	timestamp: string;
	cwd: string;
	parentSession?: string;
}

export interface NewSessionOptions {
	id?: string;
	parentSession?: string;
}
```

Header 必须是第一个可解析的 JSON 对象。它没有 entry 的 `parentId`，因此不属于树，也不会进入模型上下文。字段分别承担以下职责：

| 字段 | 含义 | 恢复时的作用 |
|---|---|---|
| `version` | 文件格式版本；缺失按 v1 处理 | 决定是否执行迁移 |
| `id` | session 身份，默认由 UUIDv7 生成 | CLI resume 与 Agent 的 session id |
| `timestamp` | session 创建时间 | session 列表的创建时间与兜底活动时间 |
| `cwd` | 创建 session 时的工作目录 | 恢复 cwd-bound 资源和默认 session 目录 |
| `parentSession` | fork 来源文件 | 记录派生关系，不自动合并父文件 |

`parentSession` 只是 provenance。恢复子 session 时，Pi 不会再打开父文件补齐历史，因为 fork 时选中的 entries 已复制到新文件。

新建 session 先在内存中创建 header，但不会立刻创建文件。`newSession()` 位于 `packages/coding-agent/src/core/session-manager.ts:861-886`，它同时清空 entry 索引、label 索引和 leaf，并为持久化模式计算目标文件名。文件名由时间戳和 session id 构成，身份仍以 header 的 `id` 为准。

## 2. Entry 是事件；共同骨架把不同状态放进同一棵树

### 解决的问题

对话恢复不只有 user/assistant message。模型切换、thinking level、压缩、分支摘要、扩展状态和标签也需要与“发生在对话的哪个位置”绑定。Pi 让所有事件共享 `SessionEntryBase`，避免另建几套互不一致的时间线。

共同字段位于 `packages/coding-agent/src/core/session-manager.ts:46-51`：

```ts
export interface SessionEntryBase {
	type: string;
	id: string;
	parentId: string | null;
	timestamp: string;
}
```

- `id` 是 entry 身份。正常新 entry 使用经过碰撞检查的 8 位 UUID 前缀；100 次都碰撞时退回完整 UUID，见 `session-manager.ts:216-224`。
- `parentId` 指向这个事件发生前的 leaf。`null` 表示根节点。
- `timestamp` 用于展示和同级子节点排序，但不决定当前 branch。
- `type` 决定 payload 结构以及是否进入模型上下文。

当前版本的 entry 类型定义集中在 `session-manager.ts:53-149`：

| 类型 | 保存的事实 | 是否投影为 AgentMessage |
|---|---|---|
| `message` | user、assistant、toolResult、bashExecution、custom 等消息 | 是，保留原消息 |
| `model_change` | provider 与 model id | 否；用于恢复当前模型 |
| `thinking_level_change` | thinking level | 否；用于恢复思考级别 |
| `compaction` | 摘要、保留起点、压缩前 token 数 | 是，转为 compaction summary |
| `branch_summary` | 被放弃分支的摘要及来源 | 是，转为 branch summary |
| `custom` | 扩展自己的持久化数据 | 否 |
| `custom_message` | 扩展注入的内容、显示策略和私有 details | 是，转为 role=`custom` 的消息 |
| `label` | 某个 target entry 的标签变更 | 否 |
| `session_info` | 当前 session 名称变更 | 否 |

这里有两个边界。第一，“持久化”不等于“发给模型”；`custom`、label 和 session name 都能落盘，却不会污染上下文。第二，“不是 message entry”也不等于“不进上下文”；压缩和分支摘要会在投影时构造成专用 AgentMessage。

### `custom` 与 `custom_message` 不是同一个扩展接口

`custom` 用来重建扩展内部状态。扩展恢复时扫描相同 `customType` 的 entry，自行解释 `data`。SessionManager 不认识 payload，也不会替扩展执行 reducer。

`custom_message` 则明确进入上下文。投影函数位于 `session-manager.ts:379-404`：

```ts
export function sessionEntryToContextMessages(entry: SessionEntry): AgentMessage[] {
	if (entry.type === "message") {
		const message = entry.message;
		if (
			(message.role === "user" || message.role === "assistant" || message.role === "toolResult") &&
			message.content == null
		) {
			return [{ ...message, content: [] }];
		}
		return [message];
	}
	if (entry.type === "custom_message") {
		return [createCustomMessage(entry.customType, entry.content ?? [], entry.display, entry.details, entry.timestamp)];
	}
	if (entry.type === "branch_summary" && entry.summary) {
		return [createBranchSummaryMessage(entry.summary, entry.fromId, entry.timestamp)];
	}
	if (entry.type === "compaction") {
		return [createCompactionSummaryMessage(entry.summary, entry.tokensBefore, entry.timestamp)];
	}
	return [];
}
```

`display` 只控制 TUI 是否显示 custom message，不控制其上下文资格；`details` 会留在 AgentMessage 供扩展使用，后续转换到模型协议时不作为正文发送。普通 `custom` 则在这里直接落入空数组。

## 3. append 同时推进文件、索引和 leaf

### 运行流程与状态变化

各种 `appendXxx()` 的差别主要是 payload。共同动作都收敛到 `_appendEntry()`，入口在 `session-manager.ts:975-980`：

```ts
private _appendEntry(entry: SessionEntry): void {
	this.fileEntries.push(entry);
	this.byId.set(entry.id, entry);
	this.leafId = entry.id;
	this._persist(entry);
}
```

以 `appendMessage()` 为例，调用链是：

1. 读取当前 `leafId`，写入新 entry 的 `parentId`；
2. 生成未与现有 `byId` 冲突的 id；
3. 把 entry 追加到 `fileEntries`；
4. 更新按 id 查找的 `byId`；
5. 把 leaf 移到新 entry；
6. 按持久化策略写 JSONL。

状态变化可以写成：

```text
append(payload)
  输入：leaf = P
  生成：entry E { id = E, parentId = P }
  内存：entries += E, byId[E] = E, leaf = E
  文件：首次满足落盘条件时写全量；之后 append E 一行
```

`parentId` 在写入后不再改变。分支不是把旧数组裁短，而是让下一条 entry 指向一个更早的父节点。

### 为什么第一条 assistant message 才触发建文件

`_persist()` 位于 `session-manager.ts:946-973`。新 session 在还没有 assistant message 时只积累内存 entry；第一条 assistant message 到来后，用排他创建标志 `wx` 一次写入 header 和此前所有 entries，之后才逐行追加。

```ts
const hasAssistant = this.fileEntries.some(
	(e) => e.type === "message" && e.message.role === "assistant",
);

if (!hasAssistant) {
	if (this.flushed) {
		appendFileSync(this.sessionFile, `${JSON.stringify(entry)}\n`);
	} else {
		this.flushed = false;
	}
	return;
}

if (!this.flushed) {
	const fd = openSync(this.sessionFile, "wx");
	try {
		for (const e of this.fileEntries) {
			writeFileSync(fd, `${JSON.stringify(e)}\n`);
		}
	} finally {
		closeSync(fd);
	}
	this.flushed = true;
} else {
	appendFileSync(this.sessionFile, `${JSON.stringify(entry)}\n`);
}
```

这项策略避免仅输入一条问题、尚未得到响应的临时 session 出现在历史列表里，也保证首轮 assistant 写入时，header、初始 model/thinking 记录和 user message 一并存在。

代价是 `getSessionFile()` 返回路径不代表文件已经存在。调用方若在首个 assistant message 前依赖该文件，必须理解延迟落盘契约。创建分支文件也沿用同一规则：选中路径包含 assistant 时立即写文件，否则等新分支的第一条 assistant message。

## 4. 文件是追加日志，树由引用关系解释

### branch 只移动内存 leaf

`branch()` 位于 `session-manager.ts:1283-1294`。它只检查目标 id 存在，然后修改 `leafId`：

```ts
branch(branchFromId: string): void {
	if (!this.byId.has(branchFromId)) {
		throw new Error(`Entry ${branchFromId} not found`);
	}
	this.leafId = branchFromId;
}
```

假设已有 `A → B → C`，调用 `branch(B)` 后并不写文件，也不删除 C。下一次 append D 才把 `{ parentId: B }` 追加到文件，于是 B 有 C、D 两个孩子，leaf 前进到 D。

```text
文件行：A, B, C, D

树：A ─► B ─► C
          └──► D  ← leaf
```

因此，“当前选中了哪个旧节点”本身不是持久化字段。若只调用 `branch(B)` 随即退出，重新打开文件时 `_buildIndex()` 会把最后一个文件 entry 当 leaf，选择不会保留。只要随后追加 D，`parentId=B` 就把新的 branch 永久编码进树，而文件末尾 D 也自然成为恢复后的 leaf。

`resetLeaf()` 同理：它把 leaf 设为 `null`，下一条 append 会产生另一个根。这个内存动作单独也不落盘。

### 当前路径由 leaf 反向追父节点

路径构造位于 `session-manager.ts:330-356`。算法不扫描“最近一段连续行”，而是从 leaf 反复查询 `byId[parentId]`，最后反转：

```ts
const index = buildEntryIndex(entries, byId);
let leaf = leafId ? index.get(leafId) : undefined;
leaf ??= entries[entries.length - 1];

const path: SessionEntry[] = [];
let current = leaf;
while (current) {
	path.push(current);
	current = current.parentId ? index.get(current.parentId) : undefined;
}
path.reverse();
return path;
```

这个结构让分支成本很低：旧路径完全保留，新分支只增加节点。回看、导出树和从任意节点派生 session 都不需要复制整个历史。

代价是引用完整性没有数据库约束。父节点缺失时，路径在缺口处停止；`getTree()` 会把 orphan 当作另一个 root，见 `session-manager.ts:1239-1276`。重复 id 也没有显式报错，构建 Map 时后出现的 entry 会覆盖按 id 索引。正常 API 会避免这些情况，手工修改或损坏文件则可能得到“仍能加载、语义已退化”的结果。

## 5. label 和 session name 用追加事件表达修改

### 解决的问题

append-only 历史不能原地给旧 message 增加标签，也不能覆盖 header 中的名称。Pi 把“修改”记录为新 entry，再从日志求当前值。

`appendLabelChange()` 位于 `session-manager.ts:1157-1181`：它先确认 `targetId` 存在，随后把 label entry 接到当前 leaf。label 非空时更新 `labelsById`，为空时删除索引中的当前值。旧 label entry 都保留，因此同一目标采用“文件顺序中的最后一次变更生效”。

`session_info` 也是相同思路。`appendSessionInfo()` 会去掉换行并 trim；`getSessionName()` 从全部 entries 末尾反向寻找最新一条，空名称表示显式清除，见 `session-manager.ts:1064-1090`。

这两类元数据有一个细微差异：

- label 的目标是某个 entry，恢复时 `_buildIndex()` 顺序重放所有 label change，建立派生索引；
- session name 是整份 session 的属性，查询时直接取物理文件中最后一条 `session_info`，不按当前 branch 过滤。

二者都不会进入 LLM messages，但它们本身仍是树节点。后续 entry 可能以 label 为父节点。抽取单条 branch 时，Pi 会过滤原 label entries、重新串接保留路径，再把路径上目标 entry 的最终 label 追加到末尾，避免因移除 label 产生 orphan。实现位于 `session-manager.ts:1334-1412`，对应测试在 `packages/coding-agent/test/session-manager/labels.test.ts:98-193`。

## 6. “append-only”有明确适用范围

正常对话写入不修改旧 entry，这使 session 具有审计历史和天然分支。但源码里存在三类非追加操作：

1. 打开 v1/v2 文件并迁移时，内存迁移完成后 `_rewriteFile()` 覆盖原文件；
2. 打开一个真正的空文件时，Pi 写入新 header；
3. `createBranchedSession()` 创建新 session 文件，只保留指定 leaf 的祖先路径，并重建 label entries。

所以更精确的表述是：**正常 session 事件是 append-only；格式迁移会重写原文件，分支抽取会产生一份新的投影文件。**

原地 `branch()` 与 `createBranchedSession()` 也不是同一种操作：

| 操作 | 原历史 | session id | 文件 | `parentSession` |
|---|---|---|---|---|
| `branch(id)` | 全部保留 | 不变 | 后续仍追加原文件 | 不变 |
| `branchWithSummary(id, ...)` | 全部保留，并新增 summary | 不变 | 原文件追加 | 不变 |
| `createBranchedSession(id)` | manager 切换到所选路径 | 新 id | 新文件或延迟创建 | 指向原 session 文件 |
| `forkFrom(source, targetCwd)` | 复制来源文件的全部非 header entries | 新 id | 目标 cwd 下新文件 | 指向 source 文件 |

`branchWithSummary()` 会先把 leaf 移到分叉点，再立即追加 `branch_summary`。与裸 `branch()` 相比，它不只持久化结构选择，还把被放弃路径的重要信息压成一条会进入上下文的消息。

## 7. 加载、校验与迁移的失败边界

### JSONL 读取

`loadEntriesFromFile()` 位于 `session-manager.ts:487-542`。它用 1 MiB buffer 和 `StringDecoder` 流式拆行，不把整份文件读成一个 JavaScript 字符串；这允许打开超过 Node 单字符串长度限制的 session。

每行独立 `JSON.parse`。空行和解析失败的行会被跳过，最后再验证第一条成功解析的对象必须是带字符串 id 的 session header：

```ts
const entry = parseSessionEntryLine(line);
if (entry) entries.push(entry);

if (entries.length === 0) return entries;
const header = entries[0];
if (header.type !== "session" || typeof header.id !== "string") {
	return [];
}
return entries;
```

这是一种“尽量恢复”策略。中间一行损坏不会使整个 session 不可读，但那一行若恰好是父节点，后代会成为 orphan；Pi 不会凭 timestamp 猜测并修补链条。

显式打开文件时，`setSessionFile()` 在 `session-manager.ts:825-858` 区分两种空结果：

- 文件大小为 0：把它当作待初始化 session，写入有效 header；
- 文件非空但没有有效 header：抛出错误并保留原内容，不把任意 JSONL 覆盖成 Pi session。

不存在的显式路径会创建内存中的新 session，并保留该路径，等落盘条件满足后创建文件。文件创建使用 `wx`；若目标在此期间被其他进程创建，写入会失败，而不是静默覆盖。

### 版本迁移

迁移入口在 `session-manager.ts:226-291`：

- v1 → v2：按原线性顺序生成 `id/parentId`；compaction 的数组索引引用转换成 entry id；
- v2 → v3：把 message 中旧的 `hookMessage` role 改成 `custom`；
- header 没有 version 时按 v1 处理；达到当前版本后不重复迁移。

打开旧 session 会立即重写文件。这使后续代码只处理当前结构，代价是“读取历史”带有磁盘修改副作用；同时，重写不是事务式临时文件替换，进程在写到一半时中止可能留下截断文件。

## 8. 恢复不是反序列化整个 Agent

### SessionManager 先恢复树

打开已有文件时，`setSessionFile()` 执行：

```text
loadEntriesFromFile
  → 取得 header.id
  → migrateToCurrentVersion（必要时重写）
  → _buildIndex
       byId[id] = entry
       顺序重放 label change
       leaf = 最后一条 entry.id
```

这里恢复的是账本和派生索引。`_buildIndex()` 不寻找“没有孩子的最新节点”；它直接把物理文件最后一条 entry 设为 leaf。这与正常 append 的写入方式相配，也说明外部重排行、手工在末尾追加旧 branch 元数据都会改变默认活动分支。

### SDK 再恢复可运行状态

`createAgentSession()` 位于 `packages/coding-agent/src/core/sdk.ts:164-238` 和 `sdk.ts:357-369`。它调用 `buildSessionContext()`，从当前路径恢复三项状态：

1. `messages`：路径上的 message、custom message、branch summary，以及压缩后的有效上下文；
2. `model`：路径中最后一次 model change，或最后一条 assistant message 的 provider/model；
3. `thinkingLevel`：路径中最后一次 thinking level change，缺省为 `off`。

如果调用方没有显式指定 model，SDK 才尝试使用 session 中的 provider/model；模型仍在目录中且认证可用才恢复，否则记录 fallback 提示并选择当前可用模型。thinking level 也要经过当前模型能力的 clamp。因此，JSONL 保存的是“希望恢复的选择”，最终运行状态仍受当前模型目录、凭据和能力约束。

恢复后，SDK 把投影结果赋给 `agent.state.messages`。旧 session 没有 thinking entry 时，会按当前默认值补一条，避免下一次继续依赖外部 settings。

### 真正持久化与没有持久化的状态

| 状态 | 是否由 session JSONL 恢复 | 说明 |
|---|---|---|
| 对话消息和 tool result | 是 | 位于活动 leaf 路径且未被压缩投影排除时进入 Agent |
| 模型选择 | 是，但可能 fallback | 需当前 ModelRuntime 仍有模型和认证 |
| thinking level | 是，但会 clamp | 受当前模型能力限制 |
| session id、cwd、来源 | 是 | 来自 header |
| session name、labels | 是 | 由追加事件求最新值 |
| 扩展 custom data | 原始数据是 | 扩展必须扫描并自行重建内部状态 |
| 活动 branch | 间接持久化 | 重开默认取文件最后一条 entry；裸 leaf 移动不落盘 |
| Agent 正在运行的请求、AbortSignal | 否 | 进程内瞬时状态 |
| steering/follow-up 待处理队列 | 否 | 只有已经成为 message entry 的结果会保留 |
| retry attempt、retry timer | 否 | 由新运行重新计算 |
| 当前活动工具集合 | 否 | 由启动选项、settings 和资源重新建立 |
| system prompt、已加载扩展实例 | 否 | 根据当前 cwd 的资源重新加载 |
| provider 凭据 | 否 | 来自 auth 与运行环境，不写入 session |

Session 文件因此不是进程 checkpoint。它足以重建一条可继续的语义历史，却不承诺恢复中断瞬间的执行栈、未完成网络请求或工具进程。

## 9. 关键测试固定了哪些契约

测试不是对实现的机械复述，它们明确了几个容易被改坏的边界：

- `packages/coding-agent/test/session-manager/tree-traversal.test.ts:10-122` 验证所有 append 类型共享 parent 链，并在每次追加后推进 leaf；
- `tree-traversal.test.ts:192-311` 验证旧节点分叉后，旧孩子与新孩子是 sibling，当前上下文只走新 branch；
- `tree-traversal.test.ts:411-532` 验证分支抽取只保留祖先路径，以及没有 assistant 时延迟建文件的契约；
- `packages/coding-agent/test/session-manager/labels.test.ts:5-55` 验证 label 是最后一次变更生效，而不是修改目标 entry；
- `labels.test.ts:145-204` 验证抽取 branch 时重接因移除 label 断开的父子链，并确认 label 不进上下文；
- `packages/coding-agent/test/session-manager/save-entry.test.ts:4-45` 验证 custom data 留在树中但不增加上下文消息数；
- `packages/coding-agent/test/session-manager/migration.test.ts:4-75` 验证 v1 线性历史转换为 v3 树，且已有 id/parentId 不被改写；
- `packages/coding-agent/test/session-manager/file-operations.test.ts:10-104` 验证流式读取、跳过坏行和 header 校验；同文件 `file-operations.test.ts:207-277` 验证空文件可初始化，非空非法文件必须报错且内容不变。

这些测试共同说明了设计取舍：Pi 偏向保留历史、局部容错和便宜分支，而不是用强 schema 校验拒绝任何不完整文件。对应风险也很直接——跳过坏行可能让 session 表面可打开，却丢失一个事件或截断父链。对重要 session 做外部备份，比假设 JSONL 加载成功就代表历史完整更可靠。

## 10. 一条恢复主线

完整恢复链如下：

```text
CLI / SDK 选择 session
  → SessionManager.open() / continueRecent()
  → loadEntriesFromFile()
       流式逐行解析，坏行跳过，首条必须是 header
  → setSessionFile()
       必要时迁移并重写
  → _buildIndex()
       建 byId、重放 labels、最后一条 entry 成为 leaf
  → buildSessionContext()
       leaf 反向追 parentId，选择当前路径
       解析 model / thinking level
       把有上下文语义的 entries 投影为 AgentMessage
  → createAgentSession()
       校验当前模型与认证，必要时 fallback
       clamp thinking level
       写入 agent.state.messages
```

JSONL 保存“发生过什么”，树回答“当前沿哪条历史继续”，上下文投影回答“这一轮模型实际看到什么”。这三个问题分别由日志、`id/parentId + leaf`、`buildSessionContext()` 处理。下一讲会继续拆解最后一步：compaction、branch summary 和各种消息怎样共同决定最终上下文。
