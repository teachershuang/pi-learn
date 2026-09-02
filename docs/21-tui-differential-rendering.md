# 第 21 讲：TUI 的差分渲染

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的交互界面不是一组直接向 stdout 打印内容的业务回调。`InteractiveMode` 先把 session 事件投影为组件状态，`Component.render(width)` 再把状态投影成逻辑行，最后 `TUI` 把逻辑行与上一次结果比较，生成终端控制序列。

```text
AgentSession event
  → InteractiveMode 增删或更新组件
  → requestRender() 合并高频刷新请求
  → Component.render(width) 生成 string[]
  → overlay 合成、光标标记提取、ANSI 行尾复位
  → 与 previousLines 比较
  → 全量重绘或改写变化区间
  → Terminal.write() 一次写入同步输出块
```

这条链上有三类状态，不能混为一谈：

- session 和 agent 保存对话、工具结果及运行状态；
- 组件保存展示所需的局部状态，例如编辑器光标、流式消息内容和工具卡片展开状态；
- `TUI` 保存上次渲染行、终端光标位置、viewport 和图片 ID，用于计算下一次写入。

界面重建不会改写 session，session 事件也不会直接决定某一行终端字符怎样移动。

## 1. Component 契约只描述逻辑屏幕

### 解决的问题

业务组件不应各自处理光标上移、清行和终端 resize。否则两个组件连续更新时，会同时假设自己拥有 stdout，无法可靠合成界面。

`packages/tui/src/tui.ts:61-87` 把组件压缩成三个职责：

```ts
export interface Component {
	/** Render the component to lines for the given viewport width */
	render(width: number): string[];

	/** Optional handler for keyboard input when component has focus */
	handleInput?(data: string): void;

	/** If true, component receives key release events (Kitty protocol). */
	wantsKeyRelease?: boolean;

	/** Invalidate any cached rendering state. */
	invalidate(): void;
}
```

`render()` 的结果是逻辑行，不是写入指令。每行可含颜色、超链接或图片协议的 escape sequence，但可见宽度不能超过参数 `width`。`invalidate()` 只要求组件丢弃失效缓存；真正刷新仍由 `requestRender()` 触发。

`Container` 在 `packages/tui/src/tui.ts:256-289` 纵向拼接子组件。它不知道消息、工具或 footer 的含义，也不做布局协商：同一层的 children 按顺序贡献行。这种契约很小，扩展容易实现；代价是复杂组件必须自己处理换行、截断、缓存和内部布局。

### 失败路径

普通差分更新时，`TUI.doRender()` 会检查非图片行的 `visibleWidth(line)`。一旦超过终端宽度，它先停止 TUI、恢复终端状态，再抛错并提示自定义组件使用 `visibleWidth()` 与 `truncateToWidth()`，见 `packages/tui/src/tui.ts:1519-1547`。

这个检查没有覆盖所有入口。overlay 会在合成前防御性截断，首次全量渲染也不会走同一段越界检查。因此“组件必须遵守宽度”仍是公开契约，运行时检查只是部分防线，不是类型层保证。

## 2. 焦点决定输入所有者，光标位置由渲染结果声明

键盘输入和硬件光标是两件事。`handleInput()` 决定谁处理按键；`Focusable.focused` 决定组件是否在输出中标出输入法光标位置。

`packages/tui/src/tui.ts:98-120` 定义了 `Focusable` 和零宽 `CURSOR_MARKER`。`setFocusInternal()` 在切换时先把旧组件的 `focused` 设为 `false`，再设置新组件并把它标为 `true`，见 `packages/tui/src/tui.ts:370-433`。

```ts
if (isFocusable(this.focusedComponent)) {
	this.focusedComponent.focused = false;
}

this.focusedComponent = nextFocus;

if (isFocusable(nextFocus)) {
	nextFocus.focused = true;
}
```

组件聚焦后，在自己的文本光标处输出 `CURSOR_MARKER`。`TUI.extractCursorPosition()` 只扫描可见 viewport，从标记前文本的可见宽度算出列号，随后删除标记。最后 `positionHardwareCursor()` 移动真实终端光标；是否显示它由设置控制。相关入口是 `packages/tui/src/tui.ts:1234-1252` 和 `packages/tui/src/tui.ts:1627-1657`。

这种设计同时保留了反色字符绘制的“假光标”和输入法候选框需要的硬件位置。组件若嵌套了 `Input` 或 `Editor`，容器还必须把 `focused` 传给内部输入组件；官方说明位于 `packages/coding-agent/docs/tui.md:31-85`。

### 输入分派顺序

`ProcessTerminal` 开启 raw mode、bracketed paste 和键盘协议，把 stdin 切成单个输入序列，见 `packages/tui/src/terminal.ts:134-205`。进入 `TUI.handleInput()` 后，顺序是：

1. 消费等待中的终端颜色响应；
2. 让全局 input listener 消费或改写输入；
3. 识别图片 cell size 响应；
4. 处理全局 debug 键；
5. 修正因 resize 或 overlay 可见性变化而失效的焦点；
6. 过滤未订阅的 Kitty key-release；
7. 调用当前焦点组件的 `handleInput()`，然后请求渲染。

入口见 `packages/tui/src/tui.ts:761-834`。Ctrl+C 也不会由 `TUI` 强制退出，而是交给焦点组件判断。这样 selector、loader 和编辑器可以给同一按键不同语义；同时也意味着焦点丢失会直接表现为“按键没有接收者”。

## 3. 编辑器把文本状态与视觉布局分开

`Editor` 的主状态只有逻辑行和 UTF-16 字符索引形式的光标：

```ts
interface EditorState {
	lines: string[];
	cursorLine: number;
	cursorCol: number;
}

export class Editor implements Component, Focusable {
	private state: EditorState = {
		lines: [""],
		cursorLine: 0,
		cursorCol: 0,
	};
	focused: boolean = false;
}
```

源码入口是 `packages/tui/src/components/editor.ts:209-260`。换行后的视觉行、滚动位置、反色光标和 autocomplete 列表都由 `render(width)` 临时计算，不写回 `state.lines`。

`render()` 先扣除水平 padding 和末尾光标预留列，再按 terminal height 的 30% 决定最多显示多少行；随后把当前光标所在的视觉行滚进窗口。聚焦时，编辑器在反色光标前插入 `CURSOR_MARKER`，见 `packages/tui/src/components/editor.ts:464-588`。

按键处理直接修改编辑器状态。Enter、换行、历史、撤销、autocomplete 和普通字符都在 `handleInput()` 分支中决策，见 `packages/tui/src/components/editor.ts:591-879`。提交时，编辑器先展开大段粘贴的占位符、清空自己的编辑状态，再调用 `onSubmit`：

```ts
private submitValue(): void {
	this.cancelAutocomplete();
	const result = this.expandPasteMarkers(this.state.lines.join("\n")).trim();

	this.state = { lines: [""], cursorLine: 0, cursorCol: 0 };
	this.pastes.clear();
	this.pasteCounter = 0;
	this.exitHistoryBrowsing();
	this.scrollOffset = 0;
	this.undoStack.clear();
	this.lastAction = null;

	if (this.onChange) this.onChange("");
	if (this.onSubmit) this.onSubmit(result);
}
```

见 `packages/tui/src/components/editor.ts:1248-1262`。`Editor` 只产出文本；coding-agent 的 submit handler 再决定它是 slash command、bash、steering、follow-up 还是普通 prompt。

### 大段粘贴为什么不是直接塞进可见文本

bracketed paste 超过 10 行或 1000 字符时，编辑器把原文放进 `pastes` map，在可见文本中插入 `[paste #N ...]`。提交时才恢复原文，见 `packages/tui/src/components/editor.ts:1144-1210`。

这减少了数千字符参与逐键导航和渲染的成本，但引入了“一个可见标记对应一段隐藏文本”的双重状态。标记必须作为原子 segment 参与移动、删除和换行；即使标记本身比终端还宽，也要继续拆分。`packages/tui/test/editor.test.ts:3737-3793` 专门固定了窄终端和光标落在标记上的回归路径。

## 4. 一次刷新先合并，再限频

流式 token、spinner 和工具输出都可能高频调用 `requestRender()`。如果每个事件立即重绘，终端 I/O 会反过来拖慢事件处理。

`packages/tui/src/tui.ts:712-759` 用两个状态合并请求：`renderRequested` 表示已有待处理刷新，`renderTimer` 表示限频定时器已存在。普通刷新通过 next tick 调度，并把两次实际渲染的最小间隔限制为 16ms。渲染期间又收到请求时，结束后继续安排下一轮。

`requestRender(true)` 是另一条路径。它清空 previous lines、宽高、光标和 viewport 记忆，在 next tick 强制执行；外部编辑器返回或 terminal 状态可能被其他程序改写时会使用它。

### 设计取舍

16ms 合并意味着组件状态可以连续变化多次，而终端只看到最后一次快照。TUI 保证的是最终投影和事件循环内的顺序，不保证每个 token 都对应一帧。这个策略适合流式文本，也让测试必须等待 render pipeline settle，不能在 `requestRender()` 后立即读取屏幕。

## 5. 差分单位是完整逻辑行

`TUI.doRender()` 的前半段固定为：

```ts
const width = this.terminal.columns;
const height = this.terminal.rows;

let newLines = this.render(width);
if (this.overlayStack.length > 0) {
	newLines = this.compositeOverlays(newLines, width, height);
}

const cursorPos = this.extractCursorPosition(newLines, height);
newLines = this.applyLineResets(newLines);
```

见 `packages/tui/src/tui.ts:1254-1281`。overlay、光标标记和行尾 reset 都在比较之前处理，所以 `previousLines` 保存的是最终逻辑屏幕，不是各组件的原始输出。

比较从第 0 行扫描到新旧数组的较长者，记录 `firstChanged` 与 `lastChanged`。普通路径把光标移到第一条变化行，逐行清除并重写这个闭区间；两条不相邻的变化会连同中间未变行一起写。代码位于 `packages/tui/src/tui.ts:1367-1394` 和 `packages/tui/src/tui.ts:1461-1549`。

所有输出用 synchronized output 的 `CSI ? 2026 h/l` 包起来并一次 `terminal.write()`。终端若支持该协议，会把一组清行、移动和文本显示为同一帧。它减少撕裂，但不能修正错误的宽度计算或不支持同步输出的终端。

### 何时放弃差分

以下情况会进入全量路径，清屏并重建 `previousLines`、光标、viewport 和图片集合：

- terminal width 改变，所有组件都可能重新换行；
- terminal height 改变，viewport 需要重新对齐，Termux 软件键盘场景例外；
- 第一条变化位于上一次 viewport 之上，差分无法触及 scrollback；
- 删除行会迫使 viewport 向上移动；
- Kitty 图片预清理可能触发滚屏；
- 显式 `requestRender(true)`；
- 开启 `clearOnShrink` 后内容低于历史工作区高水位，且当前没有 overlay。

首次渲染会完整写出，但默认不清屏。相关分支分布在 `packages/tui/src/tui.ts:1283-1365`、`packages/tui/src/tui.ts:1404-1458` 和 `packages/tui/src/tui.ts:1493-1505`。

包 README 把它概括为首次、宽度/viewport 失效、普通更新三种策略，见 `packages/tui/README.md:591-599`。源码中的 resize、shrink、图片和删除分支是这三类的具体化，不能只用“三种策略”替代失败条件。

### 差分渲染持有的状态

| 字段 | 用途 | 失真后的表现 |
|---|---|---|
| `previousLines` | 找变化区间 | 旧字符残留或重复输出 |
| `previousWidth/Height` | 判断 resize | 按旧换行结果局部更新 |
| `cursorRow` | 逻辑内容末尾 | viewport 计算错误 |
| `hardwareCursorRow` | 当前真实光标行 | 下一次相对移动写错位置 |
| `previousViewportTop` | 判断变化是否可触及 | 把 scrollback 当成可更新区域 |
| `maxLinesRendered` | shrink 时清理工作区 | 留下旧行或频繁全量清屏 |
| `previousKittyImageIds` | 删除旧图片资源 | 图片残影或资源泄漏 |

`packages/tui/test/tui-render.test.ts:485-627` 用 headless terminal 验证中间行 spinner、首尾行和不相邻行变化后的可见结果。它没有只断言字符串 buffer，而是读取终端执行 ANSI 后的 viewport，因此能发现光标追踪错误。

## 6. 终端宽度不是 JavaScript 字符串长度

颜色代码没有可见宽度，CJK 与多数 emoji 常占两列，组合字符可能由多个 code point 组成却只占一个 grapheme。直接用 `string.length` 会让差分光标逐帧漂移，最终触发终端自动换行。

`visibleWidth()` 在 `packages/tui/src/utils.ts:162-270` 先去掉已支持的 CSI、OSC 和 APC 序列，再用 `Intl.Segmenter` 按 grapheme 计数。省去 ASCII 快路径和宽度 cache 后，核心段落位于 `packages/tui/src/utils.ts:232-259`：

```ts
let clean = str;
if (str.includes("\t")) {
	clean = clean.replace(/\t/g, "   ");
}
if (clean.includes("\x1b")) {
	let stripped = "";
	let i = 0;
	while (i < clean.length) {
		const ansi = extractAnsiCode(clean, i);
		if (ansi) {
			i += ansi.length;
			continue;
		}
		stripped += clean[i];
		i++;
	}
	clean = stripped;
}

let width = 0;
for (const { segment } of graphemeSegmenter.segment(clean)) {
	width += graphemeWidth(segment);
}
```

上面省略了源码中的 ASCII 快路径和 cache，实际入口仍以所引行号为准。宽度规则把 RGI emoji 和单个 regional indicator 视为两列，零宽 cluster 记为 0，其他字符使用 East Asian Width。孤立 regional indicator 的保守宽度是为流式 flag emoji 的中间状态服务，见 `packages/tui/src/utils.ts:167-210`。

`wrapTextWithAnsi()` 不只是切字符串。它追踪当前 SGR 属性和 OSC 8 hyperlink，在新视觉行重新打开有效样式，并在换行边界关闭需要关闭的状态，见 `packages/tui/src/utils.ts:388-624` 与 `packages/tui/src/utils.ts:705-818`。

最后，`applyLineResets()` 给每个非图片行追加完整 SGR reset 和 OSC 8 close。overlay 切片也会在三段之间插入 reset，避免底层行的颜色或超链接穿过覆盖区域。`packages/tui/test/tui-overlay-style-leak.test.ts:51-79` 验证了样式 reset 被切到可见列之外时，下一行仍不会继承 italic。

### 边界失败

overlay 若恰好从双宽字符的第二列开始，不能保留这个字符的一半。`compositeLineAt()` 用严格列切片丢弃跨边界 grapheme，再用空格补齐目标列，最后把整行截到 terminal width，见 `packages/tui/src/tui.ts:1175-1224`。`packages/tui/test/regression-overlay-cjk-boundary.test.ts:28-67` 用“让”字固定了这条规则。

## 7. Overlay 是对最终行的覆盖，不是第二块屏幕

`showOverlay()` 把组件和 options 压入 stack，同时记录 `preFocus`。capturing overlay 可见时自动获得焦点；`nonCapturing` overlay 只显示，不抢输入。返回的 handle 可以 hide、临时隐藏、focus、unfocus 和查询状态，见 `packages/tui/src/tui.ts:493-585`。

渲染时，`compositeOverlays()`：

1. 过滤当前不可见 overlay，按 `focusOrder` 从后到前排列；
2. 先根据 terminal width/height、margin、anchor、百分比和上限求布局；
3. 用求出的 width 调用 overlay component 的 `render()`；
4. 把 base lines 至少补到一个 viewport 高；
5. 按列把 overlay 行覆盖进 base line。

入口是 `packages/tui/src/tui.ts:897-1091`。overlay 的视觉顺序跟 `focusOrder` 绑定，显式 `focus()` 会把较早创建的 overlay 提到最前。焦点恢复还记录 visible、blocked 和 resume 状态，处理 overlay 临时打开普通组件再返回的场景。

### 为什么焦点恢复比一个栈顶指针复杂

可见性回调会随 resize 改变，non-capturing overlay 不应成为输入 fallback，overlay 内又可能临时切换到普通 selector。简单地在关闭时恢复“创建前焦点”会恢复到已经卸载的组件，或抢走临时界面的关闭按键。

`getTopmostVisibleOverlay()` 只从可见且 capturing 的项中按最近 focus 次序选择；移除 overlay 时，依赖它的 `preFocus` 还会重定向到更早目标。`packages/tui/test/overlay-non-capturing.test.ts` 覆盖了创建、隐藏、嵌套替换、显式 unfocus 和恢复次序，这组测试说明 overlay stack 同时承担视觉层级与输入所有权，但两者不是同一个“栈顶”。

## 8. 图片既是 escape sequence，也是占行组件

图片不能当作零宽 ANSI 文本处理。Kitty/iTerm2 序列会让终端在若干 cell 上绘制像素，而逻辑行数组仍必须为它预留高度。

`Image.render()` 的路径见 `packages/tui/src/components/image.ts:60-124`：

```ts
if (caps.images) {
	if (caps.images === "kitty" && this.imageId === undefined) {
		this.imageId = allocateImageId();
	}
	const result = renderImage(this.base64Data, this.dimensions, {
		maxWidthCells: maxWidth,
		maxHeightCells: maxHeight,
		imageId: this.imageId,
		moveCursor: false,
	});

	if (result) {
		if (result.imageId) {
			this.imageId = result.imageId;
		}

		if (caps.images === "kitty") {
			lines = [result.sequence];
			for (let i = 0; i < result.rows - 1; i++) {
				lines.push("");
			}
	}
```

能力检测依据 terminal 环境变量，tmux 和 screen 走保守路径；未知 terminal 默认不用图片与 hyperlink，见 `packages/tui/src/terminal-image.ts:65-124`。这不是运行时握手验证，环境标识错误时仍可能误判。

TUI 启动后，只有检测到图片协议才发送 `CSI 16 t` 查询 cell 像素尺寸。合法响应更新全局 cell dimensions、invalidate 所有组件并重绘；其他输入继续发给焦点组件。`packages/tui/test/tui-cell-size-input.test.ts:44-80` 验证裸 Escape 不会被查询逻辑吞掉。

Kitty 图片会分配 ID。差分范围碰到图片占用的空白行时，范围必须扩到整个图片块，并先删除旧 ID，再画新位置。全量清屏也会释放 previous image IDs。iTerm2 没有同样的 ID 清理路径，组件通过光标上移序列把图片放进预留行。

`packages/tui/test/terminal-image.test.ts:369-465` 固定了不让协议自行移动光标、按 cell size 限制宽高，以及“首行序列 + 后续空行”的布局。`packages/tui/test/tui-render.test.ts:74-322` 则验证图片移动、删除、viewport 越界和全量重绘。

## 9. VirtualTerminal 验证的是 ANSI 执行结果

`packages/tui/test/virtual-terminal.ts:11-218` 用 `@xterm/headless` 实现同一个 `Terminal` 接口。它保存 input 与 resize handler，允许测试发送按键、改变尺寸、等待异步写入，然后读取 viewport、scroll buffer 和 cursor position。

```ts
sendInput(data: string): void {
	this.inputHandler?.(data);
}

resize(columns: number, rows: number): void {
	this._columns = columns;
	this._rows = rows;
	this.xterm.resize(columns, rows);
	this.resizeHandler?.();
}

async waitForRender(): Promise<void> {
	await new Promise<void>((resolve) => process.nextTick(resolve));
	await new Promise<void>((resolve) => setTimeout(resolve, 20));
	await this.flush();
}
```

这种测试能发现“输出序列看似合理，执行后光标却错了一行”的问题。它仍不是所有真实 terminal 的等价物：图片协议、IME、Windows console mode、tmux 转发和不同 Unicode 宽度实现需要独立边界测试或真实终端复现。

## 10. InteractiveMode 把 session 事件投影成组件

coding-agent 启动时创建一个 `TUI` 和多个纵向容器：header、资源提示、chat、pending messages、status、editor 上下 widgets、editor、下方 widgets、footer。它把 editor 设为初始焦点后才启动 terminal，见 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:441-488` 与 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:700-719`。

`subscribeToAgent()` 订阅 `AgentSessionEvent`，唯一入口 `handleEvent()` 先 invalidate footer，再按事件更新组件，见 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2807-3110`。

| session 事件 | 组件投影 | 局部状态变化 | 失败或结束结果 |
|---|---|---|---|
| `agent_start` | working status、terminal progress | 清空 `pendingTools` | 旧 retry escape handler 被恢复 |
| `message_start(user)` | 新建 user message component | pending queue display 更新 | 不创建 streaming component |
| `message_start(assistant)` | 新建 `AssistantMessageComponent` | 设置 `streamingMessage` | 组件立即加入 chat |
| `message_update` | 更新 assistant 内容；发现 toolCall 时新建或更新 tool component | `pendingTools[id]` 建立映射 | 尚未执行的工具先显示参数流 |
| `message_end` | 固化 assistant component | 清掉 streaming 引用 | aborted/error 把所有待执行工具标为错误 |
| `tool_execution_start` | 复用或补建 tool component | 标记执行开始 | 缺少先前 toolCall UI 也能补建 |
| `tool_execution_update` | 更新 partial result | 保持 pending 映射 | 未找到 ID 时忽略该次 UI 更新 |
| `tool_execution_end` | 写入最终 result | 从 `pendingTools` 删除 | `isError` 进入工具卡片 |
| `compaction_end` | 成功时清空并重建 chat | footer、队列与摘要更新 | abort、error 分别显示状态或错误 |
| `agent_end` | 移除残留 streaming component、关闭 progress | 清空临时映射 | 不删除 session 中已完成消息 |

这里最容易误判的是 tool UI。模型流中一出现 `toolCall`，卡片就可能建立；真正的执行事件随后复用同一个 `toolCallId`。所以“工具卡片已出现”不等于工具已开始，更不等于结果已持久化。

### 初始加载和流式更新是两条投影路径

恢复 session 时没有历史事件重放。`renderInitialMessages()` 读取 compaction-aware entries，`renderSessionEntries()` 先用 `sessionEntryToContextMessages()` 转成可展示 item，再调用 `renderSessionItems()` 重建 assistant、tool call 与 tool result 的对应关系，见 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3269-3370` 和 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3402-3417`。

流式路径维护 `streamingComponent` 与 `pendingTools`；重建路径用局部 `renderedPendingTools` 扫描完整 item 序列，最后才把没有结果的工具放回 `pendingTools`。两条路径必须得到同样的可见结构，但状态来源不同：前者来自事件增量，后者来自 session 快照。

### 持久化顺序造成的短暂差异

`message_end` 事件到达 interactive mode 时，对应 session entry 可能尚未 append。源码在 cache miss notice 处明确说明这一点，见 `packages/coding-agent/src/modes/interactive/interactive-mode.ts:3378-3383`。界面可以先显示最终 assistant message，稍后持久化完成；若此时强行从 entries 重建，不能假设刚结束的 message 已存在。

因此组件树是事件驱动的派生视图，不是 session 的权威副本。需要恢复、分支或压缩时，代码回到 `SessionManager` 重建，而不是序列化 `chatContainer.children`。

## 11. 状态变化与失败结果总表

```text
用户按键
  → ProcessTerminal / StdinBuffer
  → TUI 终端响应与全局 listener
  → focusedComponent.handleInput()
  → EditorState 改变或 onSubmit(text)
  → InteractiveMode 决定 prompt / command / queue
  → AgentSession 产生事件
  → InteractiveMode 更新组件树
  → requestRender()
  → TUI previousLines 差分
  → Terminal.write(ANSI)
```

| 失败位置 | 决策者 | 留下的状态 | 用户可见结果 |
|---|---|---|---|
| 组件行超宽 | `TUI.doRender()` | previous state 未完成本轮更新 | TUI stop 后抛错；差分路径写 crash log |
| width/viewport 无法安全差分 | `TUI` | 旧屏状态被替换 | 清屏并全量重绘 |
| overlay 超宽 | overlay compositor | 组件原状态不变 | 当前帧被按声明宽度截断 |
| 终端不支持图片 | capability layer | 原图片数据仍在 component | 显示 MIME、尺寸和文件名 fallback |
| 图片 cell size 响应非法 | `TUI` | 保留旧 cell dimensions | 响应被消费，不触发新布局 |
| assistant aborted/error | `InteractiveMode` | session 仍保存最终 message；UI 清临时引用 | assistant 显示错误，未完成工具统一标错 |
| tool update 找不到 ID | `InteractiveMode` | 不创建新映射 | update/end 被忽略；只有后续 start 能补建组件 |
| 组件 cache 未 invalidate | 组件实现 | session 不受影响 | 主题或内容显示旧值 |

## 12. 设计边界与替代方案

### 为什么按行差分，而不是字符级 diff

字符级 diff 能减少几个字节，却必须完整模拟宽字符、组合字符、样式状态、图片和 terminal 自动换行。Pi 选择“定位到首个变化行，清行，再写变化区间”，把正确性建立在较少的 terminal 状态上。spinner 更新仍只重写一行，收益已经覆盖主要场景。

替代方案是全屏 alternate buffer 或 React/Ink 一类 reconciler。alternate buffer 更容易管理 viewport，但退出后不保留普通 scrollback；通用 reconciler能提供更丰富的布局抽象，也会增加依赖和状态层。Pi 当前实现保留终端历史，代价是必须自己处理 shrink、scrollback 与光标高水位。

### 为什么组件返回字符串而不是 cell matrix

字符串让 ANSI 样式、OSC hyperlink 和图片协议直接通过，也让扩展组件门槛很低。cell matrix 可以在类型层防止超宽和样式泄漏，却需要先把所有 escape protocol 解析成中间表示，图片也要成为特殊 cell block。

Pi 的选择把复杂度集中到 `visibleWidth()`、切片、行尾 reset 和回归测试。自定义组件获得了表达能力，也承担遵守宽度和正确 invalidate 的责任。

### 为什么 InteractiveMode 不直接写终端

事件处理器若直接打印，message update、tool update、footer、overlay 和 editor 会争夺光标。现在每个处理器只更新组件状态并请求渲染，`TUI` 是唯一 stdout 布局决策者。session 层也因此可以复用于 print、JSON 和 RPC 模式，不依赖终端组件。

这条边界同样限制了 TUI：它不知道一个 tool result 是否已持久化，也不能据界面恢复会话。状态所有权始终从 session 向组件、再向终端单向投影；需要重建时回到 session，而不是反读屏幕。
