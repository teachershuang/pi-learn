# 第 14 讲：内置编码工具

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的内置工具不是一组直接调用 Node API 的零散函数。每个工具先定义参数 schema、模型可见说明、执行函数和 TUI renderer，再由 `AgentSession` 选择活动工具，包装成 `AgentTool` 交给低层 agent loop。

```text
模型 tool call
      │ schema 校验、prepareArguments
      ▼
AgentTool.execute(signal, onUpdate)
      │
      ├── read             文件或图片 → 头部截断
      ├── bash             shell 子进程 → 流式快照 → 尾部截断
      ├── edit / write     同文件修改队列 → 文件系统
      └── grep / find      rg / fd → 结果上限 → 头部截断
      │
      ▼
ToolResult.content + details
      │
      ├── 模型上下文与 session 记录
      └── TUI 按 definition renderer 显示
```

`read`、`bash`、`edit`、`write` 默认启用。`grep`、`find`、`ls` 也有完整定义，但需要通过工具选项启用。这个区别来自运行时的活动列表，不是因为后三者属于扩展。

## 1. 定义注册表与活动工具是两层状态

### 解决的问题

系统需要知道所有可用工具，才能支持 `--tools`、运行中切换和扩展同名覆盖；模型请求却只应携带当前允许使用的工具。Pi 因而同时维护 definition registry 和 active tool list。

`packages/coding-agent/src/core/tools/index.ts:156-173` 每次都创建七个内置定义：

```ts
export function createAllToolDefinitions(cwd: string, options?: ToolsOptions): Record<ToolName, ToolDef> {
	return {
		read: createReadToolDefinition(cwd, options?.read),
		bash: createBashToolDefinition(cwd, options?.bash),
		edit: createEditToolDefinition(cwd, options?.edit),
		write: createWriteToolDefinition(cwd, options?.write),
		grep: createGrepToolDefinition(cwd, options?.grep),
		find: createFindToolDefinition(cwd, options?.find),
		ls: createLsToolDefinition(cwd, options?.ls),
	};
}
```

`AgentSession._buildRuntime()` 随后把默认活动列表定为四项，见 `packages/coding-agent/src/core/agent-session.ts:2531-2574`。`--tools` 是 allowlist，`--exclude-tools` 再排除指定名称，`--no-builtin-tools` 只关掉内置项而保留扩展工具。扩展注册同名工具时，definition registry 和执行 registry 中的内置项都会被替换。

状态分工如下：

- definition 保存 schema、description、prompt snippet、执行器和 renderer；
- registry 保存当前名称实际指向哪个 definition；
- active list 决定哪些 `AgentTool` 发给模型，也决定系统提示包含哪些工具说明。

仅仅在 registry 中找到 `grep`，不能据此断言模型当前能调用它。

## 2. cwd 是路径解析基准，不是访问边界

### 统一解析

文件类工具最终都走 `resolveToCwd()`。底层 `normalizePath()` 和 `resolvePath()` 位于 `packages/coding-agent/src/utils/paths.ts:57-84`：

```ts
export function resolvePath(input: string, baseDir: string = process.cwd(), options: PathInputOptions = {}): string {
	const normalized = normalizePath(input, options);
	const normalizedBaseDir = normalizePath(baseDir);
	return isAbsolute(normalized) ? nodeResolvePath(normalized) : nodeResolvePath(normalizedBaseDir, normalized);
}
```

这层会展开 `~`，接受 `file://`，工具路径还会去掉开头的 `@`、把多种 Unicode 空格换成普通空格。相对路径以 session cwd 为基准；绝对路径直接保留，含 `..` 的路径也可走出 cwd。

当前实现没有检查“解析后的路径必须位于工作目录内”。read、edit、write、grep、find 也没有内置 approval。换句话说，cwd 解决的是相对路径含义，不提供文件系统沙箱。进程账户能访问什么，工具就可能访问什么。

### read 的路径兼容补偿

`read` 比修改工具多走一步 `resolveReadPathAsync()`，见 `packages/coding-agent/src/core/tools/path-utils.ts:76-118`。普通路径不存在时，它依次尝试 macOS 截图时间中的窄不换行空格、NFD 文件名、弯单引号以及组合形式。edit 和 write 只做普通解析，不会用这些候选路径。

这个差异能让模型读到常见的 macOS 截图文件，却也意味着“同一段原始路径能够 read”不保证 edit 会命中同一个兼容候选。

## 3. read：文本保留开头，图片成为内容块

### 文本流程

入口是 `createReadToolDefinition()`，位于 `packages/coding-agent/src/core/tools/read.ts:203-347`。执行顺序是：

1. 解析路径并检查可读权限；
2. 通过文件内容探测图片 MIME，而不是相信扩展名；
3. 普通文件按 UTF-8 解码；
4. `offset` 从 1 开始换算数组下标，`limit` 先选行；
5. `truncateHead()` 再执行 2000 行和 50KB 两个上限；
6. 返回正文、下一次读取的 offset 提示以及结构化 truncation details。

```ts
const startLine = offset ? Math.max(0, offset - 1) : 0;
const startLineDisplay = startLine + 1;
if (startLine >= allLines.length) {
	throw new Error(`Offset ${offset} is beyond end of file (${allLines.length} lines total)`);
}

const truncation = truncateHead(selectedContent);
```

代码位于 `packages/coding-agent/src/core/tools/read.ts:267-288`。read 保留开头，因为源码、配置和文档通常需要从声明处向后理解。被截断的内容不会另存临时文件，模型必须按提示提高 offset 继续读。

一行本身超过 50KB 时，`truncateHead()` 不返回半行，而是给出 bash `sed | head` 的替代命令。默认截断函数的边界在 `packages/coding-agent/src/core/tools/truncate.ts:64-143`。

### 图片流程

探测到支持的图片后，read 读取二进制并调用 `processImage()`。成功结果含一个说明文本块和一个 `ImageContent`；BMP 等格式可能先转为 PNG，过大图片默认缩放。当前模型不支持图像时，结果仍附带图片块，同时文本说明标记该图片会从请求中省略。

`packages/coding-agent/test/tools.test.ts:188-237` 用“PNG 内容写进 `.txt`”和“伪 PNG 文本”证明判断依据是文件 magic。图片处理失败不会伪装成成功附件，只返回包含失败原因的文本。

### 中止与失败

开始前或异步步骤之间观察到 abort，read 会拒绝 `Operation aborted`。文件不存在、无读取权限、offset 越界和底层读取失败也都走 error tool result。read 不改文件，失败后没有持久状态；图片处理过程可能消耗内存，但不会写回原文件。

## 4. bash：一个 shell 进程，一条合并输出流

### shell 选择与 Windows 差异

`createLocalBashOperations()` 位于 `packages/coding-agent/src/core/tools/bash.ts:82-147`。它不调用当前平台的默认命令解释器，而会显式寻找 bash：

- 用户设置的 `shellPath` 优先；路径不存在直接报错；
- Windows 依次找 Git Bash 和 PATH 中的 `bash.exe`；找不到就失败，不回退到 PowerShell 或 cmd；
- Unix 优先 `/bin/bash`，再找 PATH，最后回退 `sh`。

具体选择在 `packages/coding-agent/src/utils/shell.ts:67-127`。常规 bash 使用 `-c`；旧式 Windows WSL `bash.exe` 改用 `-s`，从 stdin 送入命令，避免复杂命令在 argv 传递时被破坏。Windows 创建子进程时 `windowsHide: true`，且不使用 detached process group；Unix 则以 detached 进程组运行。

settings 中的 `shellCommandPrefix` 会与模型命令用换行拼接，在同一个 shell 中执行。它适合装载 alias 或环境初始化，也意味着 prefix 失败、输出或副作用都属于同一次工具调用。

### 流式输出与最终结果

stdout 和 stderr 都注册到同一个 `onData`，Pi 按到达顺序合并字节流，不在结果中保留来源通道。`OutputAccumulator` 使用流式 `TextDecoder`，能正确处理一个 UTF-8 字符被拆到两个 chunk 的情况。

`packages/coding-agent/src/core/tools/bash.ts:310-368` 把高频输出合并成最多每 100ms 一次的 partial update：

```ts
const handleData = (data: Buffer) => {
	if (!acceptingOutput) return;
	output.append(data);
	scheduleOutputUpdate();
};

const finishOutput = async () => {
	acceptingOutput = false;
	output.finish();
	clearUpdateTimer();
	emitOutputUpdate();
	const snapshot = output.snapshot({ persistIfTruncated: true });
	await output.closeTempFile();
	return snapshot;
};
```

partial update 只更新当前 tool execution 的展示状态；低层 agent loop 最终写入消息和 session 的仍是 settled result。operations 已经返回后才到达的 callback 会被 `acceptingOutput` 丢弃，回归测试 `packages/coding-agent/test/suite/regressions/5208-late-bash-output.test.ts:13-29` 固定了这条边界。

### 尾部截断与完整输出

bash 保留最后 2000 行或最后 50KB。日志结尾通常含退出原因、测试摘要和错误堆栈，这正是 `truncateTail()` 与 read 相反的理由。发生截断时，`OutputAccumulator` 把原始字节流写进临时文件，并在 result text 与 details 中返回 `fullOutputPath`。

只有一条超长末行时允许保留该行末尾的部分字节。截断算法不会切坏 UTF-8 字符边界。`packages/coding-agent/src/core/tools/output-accumulator.ts:35-222` 还把内存中的滚动尾部限制在显示上限的约两倍，避免先无限积累再截断。

### 退出、超时与中止

非零 exit code 会把已有输出附在错误前，再拒绝 `Command exited with code N`。timeout 和 abort 都杀整个进程树：Windows 调用 `taskkill /F /T /PID`，Unix 杀负 PID 对应的进程组，代码在 `packages/coding-agent/src/utils/shell.ts:200-221`。

子 shell 退出后，后代进程可能仍持有 stdout/stderr。`waitForChildProcess()` 不无限等 `close`：输出仍在到达时不断重置 100ms idle timer；管道安静后结束等待。`packages/coding-agent/src/utils/child-process.ts:49-132` 与 Windows 专项测试 `packages/coding-agent/test/bash-close-hang-windows.test.ts:66-125` 覆盖了“不漏晚到输出”和“不被继承句柄永久挂住”之间的取舍。

## 5. edit：先计算整组替换，再写一次文件

### 匹配模型

edit 接受同一文件的一组 `{ oldText, newText }`。所有 oldText 都对调用开始时读到的原文件匹配，不会让第二项去匹配第一项改完后的内容。任一项为空、找不到、不唯一或与另一项重叠，整组替换在写文件前失败。

虽然公开说明强调 exact text，`fuzzyFindText()` 会在精确匹配失败后做受限归一化：

- 去掉每行尾部空白并统一换行；
- 用 NFKC 处理全角字符和兼容形式；
- 把弯引号、若干 Unicode 横线和特殊空格映射到 ASCII。

`applyEditsToNormalizedContent()` 在 `packages/coding-agent/src/core/tools/edit-diff.ts:304-365` 先确定全部匹配和唯一性，再按位置逆序替换：

```ts
const initialMatches = normalizedEdits.map((edit) => fuzzyFindText(normalizedContent, edit.oldText));
const usedFuzzyMatch = initialMatches.some((match) => match.usedFuzzyMatch);
const replacementBaseContent = usedFuzzyMatch ? normalizeForFuzzyMatch(normalizedContent) : normalizedContent;

for (let i = 0; i < normalizedEdits.length; i++) {
	const matchResult = fuzzyFindText(replacementBaseContent, normalizedEdits[i].oldText);
	if (!matchResult.found) throw getNotFoundError(path, i, normalizedEdits.length);
}
```

模糊匹配发生时，只重写实际触及的行，未修改行继续使用原始字节表示。这样不会因为一次弯引号匹配顺带清除全文件的尾部空格。测试 `packages/coding-agent/test/tools.test.ts:894-1118` 覆盖 Unicode 归一化、归一化后重复项和未触及行保留。

### 文件格式与返回状态

`createEditToolDefinition()` 的执行段位于 `packages/coding-agent/src/core/tools/edit.ts:287-365`。它去掉 UTF-8 BOM 后匹配，写回时恢复 BOM，并按文件首次出现的换行风格恢复 LF 或 CRLF。成功结果包含：

- 给模型的简短成功文本；
- TUI 使用的 display diff；
- 可应用的 unified patch；
- 新文件中第一处变化的行号。

TUI 在参数流式生成期间可以异步计算 preview，但 preview 不是写入依据。真正 execute 时会重新读取文件、重新匹配，再执行写入。

### 不是事务写入

整组替换先验证后只调用一次 `writeFile`，所以某一项匹配失败不会留下前几项已经生效的状态。这里仍没有临时文件加原子 rename，也没有失败回滚；底层写入失败不能被解释为事务保证。

## 6. write：创建或整文件覆盖

write 不读取旧内容，也不计算 diff。它解析路径、递归创建父目录，然后用 UTF-8 覆盖目标文件，入口在 `packages/coding-agent/src/core/tools/write.ts:181-250`。

```ts
return withFileMutationQueue(absolutePath, async () => {
	throwIfAborted();
	await ops.mkdir(dir);
	throwIfAborted();
	await ops.writeFile(absolutePath, content);
	throwIfAborted();
	return {
		content: [{ type: "text", text: `Successfully wrote ${content.length} bytes to ${path}` }],
		details: undefined,
	};
});
```

write 适合新文件或明确的整文件重写。已有文件会直接覆盖，没有“必须先 read”或内容版本检查。返回文本里的 `${content.length} bytes` 实际使用 JavaScript 字符串长度；含中文或非 BMP 字符时它不是 UTF-8 字节数。这是当前实现的计数口径缺陷，不影响写入内容。

## 7. edit 与 write 共用进程内文件修改队列

### 为什么需要队列

低层 agent loop 可以并行执行工具调用。两个 edit 若同时读到同一旧文件，再各自写回，后写者会覆盖先写者。`withFileMutationQueue()` 在 `packages/coding-agent/src/core/tools/file-mutation-queue.ts:22-61` 按目标文件建立 Promise 链：

- 已存在文件尽量用 `realpath` 作为 key，因此符号链接别名进入同一队列；
- 不存在的文件使用规范化绝对路径；
- 同一 key 串行，不同 key 仍可并行；
- finally 释放下一项，并在队尾结束后删除 Map 条目。

`packages/coding-agent/test/file-mutation-queue.test.ts:37-238` 证明了同文件 edit 不丢变更、edit 与 write 共享顺序、符号链接别名串行，以及不同文件保持并发。

### abort 不等于回滚

edit 和 write 没在 abort 监听器里立即 reject，而是在每次 await 之后检查 signal。原因是立即释放队列时，尚未完成的底层写入仍可能晚到并覆盖下一个调用。

因此会出现一种状态：abort 发生在 `writeFile` 已经修改磁盘之后。当前调用随后以 `Operation aborted` 失败，但文件变化仍存在；队列会等这次写入 settled 后才放行后续修改。队列保证顺序，不承诺取消回滚，也不跨进程协调。

## 8. grep：rg 负责匹配，Pi 负责结果形状

### 调用链

`createGrepToolDefinition()` 位于 `packages/coding-agent/src/core/tools/grep.ts:123-381`。默认执行先通过 `ensureTool("rg", true)` 找到 ripgrep，再启动：

```ts
const args: string[] = ["--json", "--line-number", "--color=never", "--hidden"];
if (ignoreCase) args.push("--ignore-case");
if (literal) args.push("--fixed-strings");
if (glob) args.push("--glob", glob);
args.push("--", pattern, searchPath);
```

代码位于 `packages/coding-agent/src/core/tools/grep.ts:215-220`。`--` 把后面的 pattern 固定为搜索文本，避免 `--pre=...` 之类输入变成 rg 参数。rg 的 JSON event 只提取 match；exit code 1 表示没有匹配，不算工具错误。

### 状态与截断

默认最多收集 100 个 match。到达上限后杀掉 rg，并在结果里告诉模型提高 limit 或缩小 pattern。没有 context 时直接使用 rg event 中的行文本；要求上下文时，Pi 在 rg 结束后重新读取文件并拼出前后行。

每条匹配行最多 500 个 JavaScript 字符，整个结果再按 50KB 保留头部。grep 不保存被截断的完整结果，也不发 partial update。`details` 分别记录 match limit、整体 byte truncation 和长行截断，调用者可区分三种原因。

## 9. find：fd 搜路径，输出统一成相对 POSIX 路径

`find` 使用 glob，不搜索文件内容。默认实现位于 `packages/coding-agent/src/core/tools/find.ts:109-370`：

1. 以 cwd 解析搜索根；
2. 找到或取得 `fd`；
3. 启用 hidden，交给 fd 处理 `.gitignore`；
4. 路径型 glob 加 `--full-path`，必要时补 `**/`；
5. 用 `--` 隔开 pattern 与参数；
6. 把绝对结果转成相对搜索根的 `/` 分隔路径；
7. 应用默认 1000 项和 50KB 上限。

```ts
let effectivePattern = pattern;
if (pattern.includes("/")) {
	args.push("--full-path");
	if (!pattern.startsWith("/") && !pattern.startsWith("**/") && pattern !== "**") {
		effectivePattern = `**/${pattern}`;
	}
}
args.push("--", effectivePattern, searchPath);
```

代码位于 `packages/coding-agent/src/core/tools/find.ts:246-253`。回归测试 `packages/coding-agent/test/suite/regressions/3302-find-path-glob.test.ts:34-72` 固定了 `src/**/*.spec.ts` 等带目录 glob；`packages/coding-agent/test/suite/regressions/3303-find-nested-gitignore.test.ts:30-83` 确认子目录 `.gitignore` 不会泄漏到兄弟目录。

在 Git 仓库内，fd 使用仓库感知的默认规则，使嵌套仓库形成 ignore 边界；仓库外才加 `--no-require-git`。无结果返回正常文本，glob 解析错误、fd 非正常退出、路径或依赖问题才形成 error result。

## 10. rg 和 fd 缺失时可能触发网络安装

`ensureTool()` 位于 `packages/coding-agent/src/utils/tools-manager.ts:326-365`。顺序是 Pi 工具目录、系统 PATH、按 GitHub latest release 下载。`PI_OFFLINE=1` 会禁止下载；Android/Termux 要求用户通过包管理器安装。

下载发生在第一次执行 grep/find 时，不是定义注册阶段。它会写 Pi 的工具目录，并根据平台用 tar、Windows `tar.exe` / PowerShell `Expand-Archive` 或 Unix unzip 解包。下载失败后 grep/find 报“不可用且无法下载”，不会偷偷改用 bash 命令。

这是一项执行期外部状态变化：一次只读搜索可能安装二进制。版本选择、下载信任和发布链放到第 23 讲的安全边界中继续分析。

## 11. 三种“截断”不要混为一谈

| 位置 | 何时发生 | 保留方向 | 完整内容去向 |
|---|---|---|---|
| read | 生成 tool result 前 | 头部 | 不保存；用 offset 续读 |
| bash | 流式累积与 settled result | 尾部 | 临时文件，details 带路径 |
| grep / find | 搜索结束后 | 头部 | 不保存；提高 limit 或收窄条件 |
| TUI 折叠 | 渲染时 | 屏幕预览 | tool result 本身未改变 |
| 会话压缩 | 历史进入模型上下文前 | 摘要替代旧历史 | JSONL 历史仍是另一层状态 |

工具截断发生在结果进入 agent message 之前，模型和 session 接收到的是截断后的正文与 notices。TUI 的 Ctrl+O 只改变显示行数，不会给模型恢复被工具截掉的内容。会话压缩更晚发生，解决的是累计历史过长，不是单次工具输出过大。

## 12. 失败路径与落盘结果

| 场景 | 决策者 | tool result | 外部状态 |
|---|---|---|---|
| read 文件不存在或 offset 越界 | read | error | 无文件变化 |
| bash 非零退出 | bash | error，附已有输出 | 命令副作用不会回滚 |
| bash timeout / abort | shell backend | error，杀进程树 | 已发生的命令副作用保留 |
| edit 任一 oldText 失败或重叠 | edit matcher | error | 写入前失败，文件不变 |
| edit/write 在写入后观察到 abort | 修改工具 | error | 文件可能已经改变 |
| write 父目录创建或写入失败 | filesystem | error | 目录或目标文件可能已有部分状态 |
| grep 无匹配 | grep | 正常文本 `No matches found` | 可能已安装 rg |
| find 无匹配 | find | 正常文本 `No files found matching pattern` | 可能已安装 fd |
| rg/fd 不可用且下载失败 | dependency manager | error | 可能留下随后清理的下载临时状态 |

Pi 对工具错误的统一包装已经在第 04、06 讲展开。这里的重点是文件或进程状态不会随 error result 自动回滚。模型看到“失败”时，仍要根据具体阶段判断磁盘和子进程已经发生了什么。

## 13. 常见边界问答

### 内置工具只能操作当前项目吗？

不能这样理解。相对路径以 cwd 为基准，但绝对路径和 `..` 都被接受。当前代码没有 cwd containment check，也没有内置文件 approval。

### grep 和 find 为什么不默认启用？

默认四工具中的 bash 已能运行 `rg`、`find` 等命令。独立 grep/find 提供更窄的 schema、ignore 规则、结果上限和注入边界，但会扩大模型的工具选择面。Pi 把它们留给显式 tool allowlist。

### edit 到底是不是精确替换？

先做精确匹配；失败后允许尾部空白、Unicode 兼容字符、引号、横线和特殊空格的归一化匹配。匹配结果仍必须唯一，多项编辑不能重叠。它不是任意相似度或语义匹配。

### abort 后能否认定文件没改？

不能。edit/write 会等当前文件系统 await 完成后再观察 abort，以免过早释放同文件队列。abort 若在写入期间发生，工具可能报错而磁盘已经变化。

### bash 的 stdout 和 stderr 能否分别分析？

内置 bash 把两个 stream 都交给同一个 accumulator，最终结果没有通道标签。需要可靠区分时，命令本身应重定向到不同文件，或由扩展替换执行后端和结果结构。

### 截断的 tool result 能否靠展开 TUI 恢复？

不能。TUI 展开只显示 result 中已有的内容。bash 会给完整输出临时文件；read 需要 offset 续读；grep/find 需要改变搜索条件或 limit 后重新执行。
