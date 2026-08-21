# 第 11 讲：模型选择、reasoning 与上下文预算

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 把“用哪个模型”“让模型思考到什么程度”“这次还能生成多少”“已经花了多少”拆成四类状态。它们会在一次请求中汇合，却不是同一个配置项：

```text
CLI / scoped models / session / settings / provider defaults
                         │
                         ▼
                  Model<Api> 实例
                         │
              semantic thinking level
                         │
                         ▼
                 provider 请求参数
                         │
        context estimate ├── output maxTokens
                         │
                         ▼
               provider usage → cost
```

`Model<Api>` 是决策中心。模型条目不只含名称，还声明 API 协议、认证供应商、reasoning 能力、上下文窗口、最大输出和计价方式。CLI 先选出这个模型，Agent 再保存当前模型和 thinking level；协议适配器最后才把统一语义翻译成供应商参数。

## 1. 初始模型是一条有条件的优先级链

### 解决的问题

新会话要尊重命令行和用户默认值，恢复旧会话又不能悄悄换模型。`--models` 还兼有两个用途：限制交互界面的模型轮换范围，并在新会话没有 `--model` 时提供初始候选。

### 源码入口与关键代码

参数入口是 `packages/coding-agent/src/cli/args.ts:89-90`、`114-115` 和 `130-139`。实际决策分成两段：

- `packages/coding-agent/src/main.ts:356-433` 的 `buildSessionOptions()` 处理 CLI 与 scoped models；
- `packages/coding-agent/src/core/sdk.ts:182-238` 的 `createAgentSession()` 恢复会话，必要时再调用 `packages/coding-agent/src/core/model-resolver.ts:537-630` 的 `findInitialModel()`。

恢复路径的关键代码位于 `packages/coding-agent/src/core/sdk.ts:195-225`：

```ts
let model = options.model;

if (!model && hasExistingSession && sessionContext.model) {
	const restoredModel = modelRuntime.getModel(
		sessionContext.model.provider,
		sessionContext.model.modelId,
	);
	if (restoredModel && modelRuntime.hasConfiguredAuth(restoredModel.provider)) {
		model = restoredModel;
	}
}

if (!model) {
	const resolved = await findInitialModel({
		scopedModels: options.scopedModels ?? [],
		isContinuing: hasExistingSession,
		defaultProvider: settings.getDefaultProvider(),
		defaultModelId: settings.getDefaultModel(),
		modelRuntime,
	});
	model = resolved.model;
}
```

合并两段实现后，初始模型的真实顺序是：

1. 明确的 `--model`，包括与 `--provider` 组合使用的模型；
2. 仅对空白新会话，从 scoped models 中选择：保存的默认模型若在 scope 内则优先，否则取 scope 第一项；
3. 恢复会话活动分支记录的模型；
4. 设置中保存且当前已配置认证的默认模型；
5. 在已认证可用模型中，按 provider 默认表寻找候选；
6. 第一个已认证可用模型；
7. 没有可用模型时保持 `undefined`，由上层提示配置认证。

这里有两个容易误判的边界。第一，`--models` 不会在恢复已有消息的会话时取代会话模型，它仍会限制后续的 Ctrl+P 轮换。第二，保存的默认模型不是无条件可信：对应 provider 没有认证时会被跳过。`packages/coding-agent/test/model-resolver.test.ts:652-689` 验证了同 ID 的已认证本地 provider 可以接替未认证默认 provider。

### 失败路径与设计取舍

恢复模型还在目录中、但认证已经失效时，Pi 不会带着它发出必败请求，而是进入默认选择链。代价是一次恢复可能换模型；这比保留不可调用模型更可用，但也说明“会话记录了模型”不等于“本机一定能恢复它”。

把 scope 只用于新会话初选，保住了会话复现性。若 scope 永远高于 session restore，修改全局 `enabledModels` 就会改变旧会话的下一次回答。

## 2. 模型模式解析不等于简单字符串查找

### 源码入口与运行流程

`packages/coding-agent/src/core/model-resolver.ts:63-154` 负责匹配，`270-339` 展开 scoped patterns。模式可写成裸 ID、`provider/id`、glob，或附加 `:thinking`。处理过程是：

1. 显式 provider 限定候选集合；
2. 优先完整 ID 和规范 ID 精确命中；
3. 裸 ID 若跨 provider 重名则拒绝猜测；
4. 部分匹配优先稳定别名，再在 dated variants 中选择较新项；
5. scope 展开后去重，并报告没有命中的 pattern。

OpenRouter 一类模型 ID 本身含 `/` 或 `:`，因此解析器不能把第一个斜杠、最后一个冒号机械地当分隔符。`packages/coding-agent/test/model-resolver.test.ts:260-335` 覆盖 provider 前缀、模糊匹配和带冒号的 OpenRouter ID；`211-258` 覆盖 scope 诊断与 thinking 后缀。

`--model sonnet:high` 的 `high` 只有在它是合法 thinking level、且没有显式 `--thinking` 时才会被剥离。自定义模型的回退测试位于 `packages/coding-agent/test/model-resolver.test.ts:464-587`。若用户已经写了 `--thinking medium`，解析器把 `:high` 留在模型 ID 中。这允许 provider 真正拥有带该后缀的自定义 ID，也意味着混用两种写法可能导致模型查找失败。

### 设计取舍

模糊匹配适合命令行交互，但不是稳定标识。会话持久化写入 provider 与完整 model ID，而不是保存用户当时输入的 pattern；目录增加新模型后，旧会话的含义不会随模糊匹配结果漂移。

## 3. 会话保存的是模型事实和 thinking 变化

### 状态变化

`packages/coding-agent/src/core/session-manager.ts:358-372` 在活动分支上扫描状态：

- `thinking_level_change` 更新当前 thinking level；
- `model_change` 更新模型；
- 后续 assistant message 会用实际响应中的 provider/model 再次更新模型。

最后一条 assistant message 更接近已经发生的调用事实，所以它可以覆盖先前的切换意图。`packages/coding-agent/test/session-manager/build-context.test.ts:95-123` 明确验证了这一点。

运行中切换模型走 `packages/coding-agent/src/core/agent-session.ts:1547-1653`。`setModel()` 先检查认证，再同步 Agent 状态、追加 `model_change`、保存默认模型，并把 thinking level 收敛到新模型支持的范围。模型轮换会过滤未认证 provider；scoped model 明确指定 thinking 时采用该值，没有指定时继承当前偏好。

thinking 切换位于 `packages/coding-agent/src/core/agent-session.ts:1660-1733`。只有有效 level 真正变化时才追加 `thinking_level_change`。设置同时写入默认值，供新会话或从非 reasoning 模型切回时恢复偏好。

### 恢复规则

`packages/coding-agent/src/core/sdk.ts:226-238` 的 thinking 优先级是：

1. 本次创建显式传入的 level；
2. 已有会话活动分支中的 `thinking_level_change`；
3. 老会话若没有 thinking entry，使用设置默认值；
4. 新会话使用设置默认值；
5. 最后回退到 `packages/coding-agent/src/core/defaults.ts:3` 的 `medium`。

选出 level 后还要按当前模型能力 clamp。恢复消息后，老会话缺少 thinking entry 时会补写一条，代码在 `packages/coding-agent/src/core/sdk.ts:357-369`。这样后续恢复不再依赖可能变化的全局设置。

压缩只改变送给模型的消息投影，不删除分支上的配置状态。`packages/coding-agent/test/session-manager/build-context.test.ts:185-211` 验证压缩后仍保留 `high`；`packages/coding-agent/test/suite/agent-session-runtime.test.ts:522-596` 验证切换到另一会话会恢复其模型与 thinking。

## 4. thinking level 是语义偏好，不是统一 token 数

### 能力模型

`packages/ai/src/types.ts:686-719` 中，模型用三个字段描述这一层能力：

```ts
export interface Model<TApi extends Api> {
	id: string;
	name: string;
	api: TApi;
	provider: Provider;
	reasoning: boolean;
	thinkingLevelMap?: Partial<Record<ThinkingLevel, string | null>>;
	contextWindow: number;
	maxTokens: number;
	cost: ModelCost;
}
```

统一 level 为 `off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`。`packages/ai/src/models.ts:651-683` 计算可用集合：

- 非 reasoning 模型只支持 `off`；
- 普通 reasoning 模型默认支持到 `high`；
- `xhigh` 和 `max` 必须由模型映射显式启用；
- 映射值为 `null` 表示明确不支持；
- 请求 level 不可用时，clamp 先向更高档寻找，再向更低档寻找。

最后一条会产生看似反直觉的结果：模型若声明 `xhigh: null`、`max: "max"`，请求 `xhigh` 会升到 `max`，而不是降到 `high`。`packages/ai/test/max-thinking.test.ts:14-68` 固定了该行为。

### 进入 Agent 循环

`packages/agent/src/agent.ts:424-442` 把状态转换为流参数：`off` 变成 `reasoning: undefined`，其他 level 原样传给 `pi-ai`，同时传入可选的自定义 `thinkingBudgets`。Agent 不解释供应商字段；协议适配器才拥有最终决定权。

这种设计让 UI、会话和扩展只处理稳定语义，不必知道 Anthropic 的 budget、OpenAI 的 effort 或 Gemini 的 thinking config。它没有承诺不同 provider 的 `high` 具有相同计算量、延迟或成本。

## 5. 三类 provider 对同一 level 的解释不同

### Anthropic：adaptive effort 与 token budget 并存

`packages/ai/src/api/anthropic-messages.ts:760-825` 先判断模型是否支持 adaptive thinking。支持时把统一 level 映射为 effort，`minimal` 与 `low` 会合并为较低档；旧模型仍使用 token budget。

预算式模型调用 `adjustMaxTokensForThinking()`。thinking budget 会加入输出上限，同时受模型 `maxTokens` 约束，并至少给正常回答留下 1024 tokens。高 thinking 因而可能挤压可见回答长度，而不是额外获得无限输出空间。

### OpenAI Responses：reasoning effort 加模型映射

`packages/ai/src/api/openai-responses.ts:174-188` 先 clamp level，`270-284` 再应用 `thinkingLevelMap`，形成 `reasoning.effort`。映射表可以把 Pi 的 level 改成 provider 自己接受的字符串，也可以用 `null` 禁止某档。

Responses 请求还可要求返回 encrypted reasoning，供后续轮次复用推理上下文。这是协议状态，不改变 Pi 会话仍以统一 thinking content 和 assistant message 为边界。

### Google：类别与预算按模型家族分流

`packages/ai/src/api/google-generative-ai.ts:284-320`、`403-508` 区分多类模型：

- Gemini 3、Gemma 4 采用类别型 thinking level；
- 较早的 Gemini 2.5 采用 token budget，且各型号上下限不同；
- 某些模型不能真正关闭内部 thinking，`off` 只能降到最低隐藏思考，并关闭 thought 输出；
- Gemini 3 Pro 会把多档 Pi level 折叠成较少的供应商档位。

因此，Pi 的 `off` 表示“不请求可见或可控 reasoning”的统一意图，并不保证供应商模型内部完全不进行推理。

## 6. 上下文窗口、当前占用和输出上限是三件事

### 三个量的边界

`contextWindow` 来自模型目录，描述一次请求可容纳的总体 token 容量。`maxTokens` 是本次生成的输出上限。当前上下文占用则是根据消息、系统提示、工具定义和最近 usage 推出的动态值。

`packages/ai/src/utils/estimate.ts:14-60` 的纯估算采用约 4 字符/token，图片按 4800 字符计入，并计算 assistant text、thinking、工具名和 JSON 参数。这是跨 tokenizer 的低成本近似，不适合充当账单数据。

更可靠的路径位于 `packages/ai/src/utils/estimate.ts:63-142`：

```ts
const lastUsageIndex = findLastValidUsageIndex(messages);

if (lastUsageIndex >= 0 && !hasNewerInsertedPrefix(messages, lastUsageIndex)) {
	tokens = usageTotal(messages[lastUsageIndex]);
	for (const message of messages.slice(lastUsageIndex + 1)) {
		tokens += estimateMessageTokens(message);
	}
} else {
	tokens = estimateAllMessages(messages);
}

if (lastUsageIndex < 0) {
	tokens += estimateSystemAndTools(systemPrompt, tools);
} else {
	tokens += estimateToolsAddedAfterUsage(tools, messages[lastUsageIndex]);
}
```

provider usage 代表它看到的请求前缀，通常比字符估算可靠；后续新消息再用启发式补齐。压缩摘要或其他消息被插入到该响应之前时，旧 usage 已不能描述新前缀，代码会放弃它，避免“精确数字套在错误上下文上”。动态新增的工具也只补算 usage 之后出现的部分，防止重复计算。

### 本次输出上限

`packages/ai/src/api/simple-options.ts:12-19` 计算通用输出上限：

```ts
const maxTokens = Math.min(
	requestedMaxTokens ?? model.maxTokens,
	model.maxTokens,
	model.contextWindow - estimatedContextTokens - 4096,
);

return Math.max(1, maxTokens);
```

4096 是为估算误差和协议开销留下的安全区。这个函数只限制本次输出，不负责触发会话压缩。压缩判断位于 coding-agent 的 `AgentSession`，主要入口是 `packages/coding-agent/src/core/agent-session.ts:1935-2023`。

预算式 reasoning API 还会在这个基础上调整：`packages/ai/src/api/simple-options.ts:47-76` 默认把 `minimal/low/medium/high` 映射为 1024/2048/8192/16384 thinking tokens，`xhigh/max` 在这一通用预算函数中收敛到 `high`。这些数字只是预算式适配器的默认值，不是 Pi 对所有 provider 的定义。

### 压缩后的未知状态

`packages/coding-agent/src/core/agent-session.ts:3111-3155` 给 UI 计算当前上下文占用。刚压缩完成、尚未拿到压缩后第一条 assistant usage 时，旧 usage 明显失效，接口返回 `tokens: null, percent: null`。`packages/coding-agent/test/agent-session-stats.test.ts:80-169` 验证了从有效占用、压缩后未知，到新响应恢复精确占用的过程。

返回未知比展示一个平滑但虚假的百分比更诚实。字符估算仍用于请求保护和溢出判断，UI 则明确表达当前缺少可信基线。

## 7. usage 是协议事实，cost 是本地模型目录计算

### token 口径

`packages/ai/src/types.ts:355-380` 的 `Usage` 分为 `input`、`output`、`cacheRead`、`cacheWrite` 和可选 `reasoning`。`reasoning` 是 `output` 的子集，不能再加一次：

```ts
export interface Usage {
	input: number;
	output: number;
	cacheRead: number;
	cacheWrite: number;
	cacheWrite1h?: number;
	reasoning?: number;
	totalTokens: number;
	cost: {
		input: number;
		output: number;
		cacheRead: number;
		cacheWrite: number;
		total: number;
	};
}
```

各适配器先把供应商口径归一化。Google 会把 thought tokens 纳入 output 并单列 reasoning；OpenAI 也把 reasoning 作为 output breakdown；不提供分项的 provider 则保留 `undefined`。所以跨 provider 可以比较总 input/output，却不能假定每家都能报告 reasoning 明细。

### 费用计算

`packages/ai/src/models.ts:629-648` 的 `calculateCost()` 使用模型目录中每百万 token 的单价。计价 tier 按 `input + cacheRead + cacheWrite` 选择，超过阈值后整次请求采用该 tier，而不是只给超出部分换价。`packages/ai/test/models-runtime.test.ts:126-160` 验证阈值比较是严格大于。

Anthropic 的一小时 cache write 是特例：若协议报告 `cacheWrite1h`，这部分按 input 单价的两倍计算，其余 cache write 按普通写缓存价。`packages/ai/test/anthropic-cache-write-1h-cost.test.ts:49-86` 覆盖分拆计价与没有 breakdown 时的回退。

价格表来自本地模型目录，不是 provider 返回的账单。模型价格变更而本地目录未更新时，token usage 仍可能正确，cost 估算却会过时；最终结算仍应以 provider 账单为准。

### 当前上下文与历史累计

`packages/coding-agent/src/core/agent-session.ts:3070-3108` 的 session stats 会累加会话树中每条 assistant entry 的 token 和费用，表示会话历史总消耗。当前 context usage 只描述下一次请求附近仍在上下文中的内容。压缩能降低后者，不会抹掉前者；把两者混为一个“token 使用量”，就会错误地认为压缩可以退回已经发生的费用。

## 8. 从选择到计费的完整状态流

一次新请求经过以下决策：

1. 启动层根据 CLI、scope、session、settings 和认证状态选出 `Model<Api>`；失败结果是没有可调用模型，而不是猜一个未认证 provider。
2. 会话层恢复或选择 thinking level，并按模型能力 clamp；不支持 reasoning 的模型最终只有 `off`。
3. Agent 把 model、semantic level、消息、工具和可选 thinking budgets 交给 `pi-ai`。
4. 适配器把 level 映射为 effort、类别或 token budget；映射后的输出预算仍受模型最大输出和剩余上下文约束。
5. provider 流返回 usage；适配器把供应商字段归一化，并用模型目录计算 cost。
6. assistant message 携带实际 provider/model/usage 写入会话。它既是历史消耗证据，也成为后续恢复模型和估算上下文的基线。
7. 压缩改变下一轮上下文投影；会话累计费用、模型变更与 thinking 变更仍留在 append-only 历史中。

这条链上有三个决策者：启动层决定“选谁”，模型能力与适配器共同决定“thinking 怎样落到协议”，provider usage 加本地价格表决定“这次如何统计”。把三者塞进一个 settings 对象会更省代码，却会模糊恢复、能力校验和账单证据的责任边界。

## 9. 常见问题

### `medium` 是否表示 8192 个 thinking tokens？

不是。8192 只是通用预算式 API 的默认映射。OpenAI 可能收到 effort 字符串，Anthropic adaptive 模型使用 effort，Google 不同家族可能使用类别或不同预算。`medium` 表达的是相对强度偏好。

### `--models` 会覆盖恢复会话的模型吗？

不会。已有消息的会话优先恢复其活动分支模型；scope 用于后续轮换。明确的 `--model` 才会覆盖恢复模型。

### reasoning token 为什么不能加到 output 上？

因为 `Usage.reasoning` 被定义为 `output` 的子集。再次相加会同时夸大 total tokens 和费用。

### `maxTokens` 是不是上下文窗口剩余量？

不是。它是输出上限，还要同时受模型目录上限、用户请求值、当前上下文估算和安全余量约束。输入上下文占用与输出配额共同争用 `contextWindow`。

### 压缩后 token 百分比为什么会暂时为空？

压缩改变了消息前缀，旧 provider usage 已不再对应当前上下文。Pi 等待压缩后的新响应提供可信 usage；在此之前不把旧数字伪装成当前占用。

## 10. 本讲形成的边界

- 模型选择的稳定身份是 provider + 完整 model ID；pattern 只属于交互解析。
- session restore 的目标是复现活动分支，但认证状态仍决定模型能否实际恢复。
- thinking level 是跨 provider 的语义层，能力表和适配器有权收敛、折叠或转成预算。
- 上下文估算服务于请求保护与压缩，provider usage 才是一次调用的主要 token 证据。
- 当前 context usage 与 session 累计 usage 分别回答“下一次还装得下多少”和“历史已经消耗多少”。
- cost 是依据本地模型目录对 usage 的估算，不能替代 provider 最终账单。

下一讲进入 coding-agent 应用层，从 CLI 参数解析、会话选择和服务创建开始，比较 interactive、print、JSON 与 RPC 如何共享同一个 `AgentSession`。
