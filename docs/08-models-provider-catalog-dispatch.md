# 第 08 讲：`Models`、Provider 与模型目录

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

第 07 讲里的 `Model<Api>` 只是一份可序列化的模型说明。它不会自己寻找凭据，也不知道该调用哪个函数。真正把模型说明变成一次请求的，是 `Models`、`Provider` 和 API 实现三层协作：

```text
Models
  │ 按 provider 标识注册、查目录、检查认证、发起请求
  ▼
Provider
  │ 拥有一家供应商的目录、认证策略和 API 选择规则
  ▼
API 实现
  │ 把统一 Context 转成具体协议，并解析响应流
  ▼
AssistantMessageEventStream
```

这三层不是“供应商适配器”的三个名字。`Models` 是运行时注册表，`Provider` 是供应商边界，API 实现是传输协议边界。OpenRouter 可以作为一个 provider 复用 OpenAI 协议；GitHub Copilot 则能在同一个 provider 内按模型选择 Anthropic、OpenAI Completions 或 OpenAI Responses。

## 1. `Provider` 是运行单元，`Models` 是组合根

### 解决的问题

应用需要同时容纳内置供应商、扩展注册的供应商和用户覆盖的供应商。若目录、认证、请求函数都挂在全局分支语句上，新增 provider 必须修改中央调度器，测试也很难只替换一家供应商。

Pi 把一家供应商需要承担的职责收进一个 `Provider` 对象，再由 `Models` 保存这些对象。`Models` 不解释 Anthropic 或 OpenAI 的业务规则，只依据 provider 标识找到决策者。

### 源码入口与关键代码

`packages/ai/src/models.ts:62-116` 定义 provider 契约：

```ts
export interface Provider<TApi extends Api = Api> {
	readonly id: string;
	readonly name: string;
	readonly baseUrl?: string;
	readonly headers?: ProviderHeaders;
	readonly auth: ProviderAuth;

	getModels(): readonly Model<TApi>[];
	refreshModels?(context: RefreshModelsContext): Promise<void>;
	filterModels?(
		models: readonly Model<TApi>[],
		credential: Credential | undefined,
	): readonly Model<TApi>[];

	stream<T extends TApi>(
		model: Model<T>,
		context: Context,
		options?: ApiStreamOptions<T>,
	): AssistantMessageEventStream;
	streamSimple(
		model: Model<TApi>,
		context: Context,
		options?: SimpleStreamOptions,
	): AssistantMessageEventStream;
}
```

`packages/ai/src/models.ts:214-243` 显示注册表的状态、写入和读取方式：

```ts
class ModelsImpl implements MutableModels {
	private providers = new Map<string, Provider>();
	private credentials: CredentialStore;
	private modelsStore: ModelsStore;
	private authContext: AuthContext;

	constructor(options?: CreateModelsOptions) {
		this.credentials = options?.credentials ?? new InMemoryCredentialStore();
		this.modelsStore = options?.modelsStore ?? new InMemoryModelsStore();
		this.authContext = options?.authContext ?? defaultAuthContext();
	}

	setProvider(provider: Provider): void {
		this.providers.set(provider.id, provider);
	}

	deleteProvider(id: string): void {
		this.providers.delete(id);
	}

	clearProviders(): void {
		this.providers.clear();
	}

	getProviders(): readonly Provider[] {
		return Array.from(this.providers.values());
	}

	getProvider(id: string): Provider | undefined {
		return this.providers.get(id);
	}
```

### 运行流程与状态变化

`setProvider()` 以 `provider.id` 为键写入 `Map`。同一 id 再注册一次，不会并存两个对象，而是原位替换旧 provider。替换后，下一次目录读取、认证检查和模型请求都会走新对象；已经取得的旧 `Model` 值本身不会被追溯修改。

内置组合发生在 `packages/ai/src/providers/all.ts:78-125`：`builtinProviders()` 构造所有 provider，`builtinModels(options)` 创建 `Models`，随后逐个调用 `setProvider()`。核心入口 `packages/ai/src/index.ts:3-8` 刻意不导入生成目录和 provider 工厂；调用方需要从 `@earendil-works/pi-ai/providers/*` 或 `providers/all` 显式组合。这使核心类型和运行时保持无副作用，也让打包器有机会只装入实际使用的实现。

### 失败路径与设计取舍

provider 标识没有全局唯一性检查。后注册对象覆盖先注册对象，这是扩展和用户定制能够接管内置 provider 的基础，也意味着注册顺序是配置语义的一部分。

另一种方案是不可变注册表：重复 id 直接报错，覆盖时必须创建新的 `Models`。它更容易追踪配置来源，但不适合运行中加载扩展、重载配置或替换测试替身。Pi 选择可变注册表，并把冲突治理留给上层组合代码。

## 2. 目录读取采用“最后已知快照”

### 解决的问题

模型选择器需要同步渲染，不能每次搜索或打开列表都等待远端 `/models`。另一方面，某些目录来自构建期数据，另一些目录只有登录后才能从服务端得到。单一的“每次读取都请求网络”无法同时满足启动速度、离线使用和动态更新。

Pi 把目录分成两个动作：

- `getModels()` 同步返回当前内存快照；
- `refreshModels()` 异步尝试更新快照。

### 静态目录

多数内置 provider 使用生成的静态目录。`packages/ai/src/providers/anthropic.ts:7-20` 只负责把认证、目录和协议实现组装起来：

```ts
export function anthropicProvider(): Provider<"anthropic-messages"> {
	return createProvider({
		id: "anthropic",
		name: "Anthropic",
		baseUrl: "https://api.anthropic.com",
		auth: {
			// ANTHROPIC_OAUTH_TOKEN takes precedence over ANTHROPIC_API_KEY
			apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_OAUTH_TOKEN", "ANTHROPIC_API_KEY"]),
			oauth: lazyOAuth({ name: "Anthropic (Claude Pro/Max)", load: loadAnthropicOAuth }),
		},
		models: Object.values(ANTHROPIC_MODELS),
		api: anthropicMessagesApi(),
	});
}
```

`ANTHROPIC_MODELS` 来自自动生成文件。`packages/ai/scripts/generate-models.ts:2213-2224` 先按 provider 和 model id 分组、去重；`packages/ai/scripts/generate-models.ts:2271-2303` 删除旧的 `*.models.ts`，再为每家供应商生成目录及总聚合文件 `models.generated.ts`。

生成目录不是运行时缓存。它随 npm 包发布，变化需要重新运行生成脚本、构建和发布。好处是首次启动即可同步列出模型，代价是目录可能落后于服务端。

### 动态目录与覆盖规则

`createProvider()` 同时支持静态基线和动态覆盖。`packages/ai/src/models.ts:532-561` 的核心合并规则如下：

```ts
const baselineModels = input.models;
	let dynamicModels: readonly Model<TApi>[] = [];

	let inflightRefresh: Promise<void> | undefined;
	const fetchModels = input.fetchModels;
	const currentModels = (): readonly Model<TApi>[] => {
		const merged = [...baselineModels];
		for (const model of dynamicModels) {
			const index = merged.findIndex((entry) => entry.id === model.id);
			if (index >= 0) merged[index] = model;
			else merged.push(model);
		}
		return merged;
	};
```

同 id 的动态模型覆盖静态基线；动态目录新增的 id 会追加；动态结果没有提到的静态模型仍保留。一次成功刷新替换的是动态覆盖层，静态基线继续作为保底目录。

这个规则适合“包内目录作保底，远端信息修正或扩充”的 provider。如果远端目录必须是唯一真相，工厂函数可以不传 `models`，让基线为空。`radiusProvider()` 就是一种纯动态实现，它不出现在生成目录中，但仍由 `builtinProviders()` 注册。

### 读取失败

`Models.getModels()` 会捕获单个 provider 的 `getModels()` 异常，并把该 provider 当作空目录处理，其他 provider 仍能返回。目录聚合因此是一份尽力返回的视图，不能当作健康检查。

这项取舍保护模型选择界面：一家扩展 provider 损坏，不会让全部模型消失。代价是调用方只看聚合结果时无法区分“确实没有模型”和“读取目录失败”。需要诊断精确错误时，应直接检查 provider 或刷新结果，不能把空数组当成无故障证明。

## 3. 刷新并发执行，错误按供应商隔离

### 解决的问题

动态目录更新涉及网络、凭据、缓存和取消。任何一家供应商超时都不应阻断其他目录；多个界面同时刷新时，也不应对同一端点发出重复请求。

### `Models.refresh()` 的调度职责

`packages/ai/src/models.ts:272-317` 先筛出实现了 `refreshModels` 的 provider，再并发刷新。下面保留并发分支和失败恢复部分：

```ts
await Promise.all(
	refreshable.map(async (provider) => {
		if (options.signal?.aborted) return;
		const store: ProviderModelsStore = {
			read: () => this.modelsStore.read(provider.id),
			write: (entry) => this.modelsStore.write(provider.id, entry),
			delete: () => this.modelsStore.delete(provider.id),
		};
		let stored: Credential | undefined;
		try {
			stored = await this.readCredential(provider.id);
			const credential = await this.resolveRefreshCredential(provider, stored, allowNetwork, options.signal);
			if (!credential) return;
			await provider.refreshModels({ credential, store, allowNetwork, signal: options.signal });
		} catch (error) {
			if (!options.signal?.aborted) {
				errors.set(
					provider.id,
					error instanceof Error
						? error
						: new ModelsError("model_source", `Model refresh failed for ${provider.id}`, { cause: error }),
				);
			}
			try {
				await provider.refreshModels({
					credential: stored,
					store,
					allowNetwork: false,
					signal: options.signal,
				});
			} catch {
				// Preserve the original auth/network error; cache restoration is best-effort here.
			}
		}
	}),
);
```

调度者是 `Models`，输入是当前 provider 集合、凭据存储、模型目录存储和可选中止信号。每个 provider 得到自己的 `ProviderModelsStore`，再自行决定如何恢复缓存和请求远端。

状态变化分三类：

1. 未配置凭据：跳过该 provider，目录保持原状；
2. 刷新成功：provider 替换自己的动态快照并写缓存；
3. 刷新失败：错误记入 `errors`，随后尝试只从缓存恢复，不回滚其他 provider 的成功结果。

`Promise.all` 等待所有分支完成，但分支内部吞下并记录普通错误，因此返回的是每家供应商的结果集合。取消单独体现在 `aborted`，不会被包装成某家供应商的 `model_source` 错误。

### 缓存为什么必须按 provider 限权

`packages/ai/src/models-store.ts:9-21` 把全局存储裁成 provider 专属视图：

```ts
export interface ModelsStore {
	read(providerId: string): Promise<ModelsStoreEntry | undefined>;
	write(providerId: string, entry: ModelsStoreEntry): Promise<void>;
	delete(providerId: string): Promise<void>;
}

export interface ProviderModelsStore {
	read(): Promise<ModelsStoreEntry | undefined>;
	write(entry: ModelsStoreEntry): Promise<void>;
	delete(): Promise<void>;
}
```

provider 看不到 `providerId` 参数，无法误读或覆盖别家的目录。`pi-ai` 提供的默认实现 `InMemoryModelsStore` 只在当前进程保存数据；需要跨进程恢复时，由宿主注入持久化实现。Coding Agent 的文件存储位于 `packages/coding-agent/src/core/models-store.ts`，这说明持久化策略属于应用层，不是 provider 工厂的硬编码职责。

### 同一 provider 内的并发去重

`createProvider()` 在闭包中保存 `inflightRefresh`。第一次刷新先恢复缓存，再在允许网络时调用 `fetchModels`；并发调用复用同一个 Promise。请求成功后更新动态快照并写入 `{ models, checkedAt }`，失败则保留上一次已知状态。Promise 结束后清除进行中引用，后续调用可以重试。

去重放在 provider 内，而不是 `Models.refresh()` 的全局层，因为只有 provider 知道哪些请求代表同一目录源。它也让调用方绕过 `Models` 直接调用 `provider.refreshModels()` 时仍能获得并发保护。

### README 与当前源码的差异

`packages/ai/README.md:311-318` 仍示例化 `models.refresh("llamacpp")`，并描述单 provider 刷新失败会使 Promise 被拒绝。当前 `Models` 接口在 `packages/ai/src/models.ts:142-149` 接受的是 `ModelsRefreshOptions`，公开实现只刷新全部动态 provider，返回 `{ aborted, errors }`。

在本课程固定的 commit 上，应以接口、实现和 `packages/ai/test/models-runtime.test.ts:219-364` 的测试为准。README 这段属于尚未同步的旧说明。图片模型的 `ImagesModels.refresh(provider?)` 仍支持单 provider 参数，但它是另一套运行时，不能反推文本模型 `Models.refresh()` 也支持。

## 4. “目录存在”“认证可用”“请求成功”是三个判断

### 解决的问题

静态目录可以列出尚未配置密钥的模型；OAuth 凭据可能已经过期；某些账号只获准访问 provider 目录的一部分。如果模型选择器把目录条目直接等同于可调用模型，就会展示大量当前用户无法使用的选项。

### `checkAuth()` 只做轻量检查

`packages/ai/src/models.ts:354-399` 将目录可用性拆成两步：

```ts
async checkAuth(providerId: string): Promise<AuthCheck | undefined> {
	const provider = this.providers.get(providerId);
	if (!provider) return undefined;
	return this.checkProviderAuth(provider, await this.readCredential(providerId));
}

async getAvailable(providerId?: string): Promise<readonly Model<Api>[]> {
	const providers = providerId
		? [this.providers.get(providerId)].filter((entry) => entry !== undefined)
		: this.getProviders();
	const checks = await Promise.all(
		providers.map(async (provider) => {
			const credential = await this.readCredential(provider.id);
			return { provider, credential, auth: await this.checkProviderAuth(provider, credential) };
		}),
	);
	return checks.flatMap(({ provider, credential, auth }) => {
		if (!auth) return [];
		const models = provider.getModels();
		return provider.filterModels?.(models, credential) ?? models;
	});
}
```

`checkAuth()` 的决策输入包括凭据存储、环境变量和 provider 的认证声明。它只回答“当前是否存在一种认证来源”，不会刷新过期 OAuth。`packages/ai/test/models-runtime.test.ts:395-427` 专门放入一份已过期凭据，断言 OAuth refresh 次数仍为零，同时该 provider 被列入可用集合。

这是刻意的读路径约束。打开模型选择器不应悄悄发起刷新 token 的网络请求，也不应改写凭据存储。真正发请求时，认证解析器才在锁内刷新过期凭据；这部分生命周期在第 09 讲展开。

### 凭据相关的模型过滤

GitHub Copilot 的 OAuth 凭据可以携带 `availableModelIds`。`packages/ai/src/providers/github-copilot.ts:18-27` 只在该字段结构有效时按 id 过滤；API key 用户或旧凭据仍看到完整目录。决策者是 provider，因为只有它理解凭据中的供应商专属字段。

`filterModels()` 不修改 `getModels()` 的完整同步快照，只影响 `Models.getAvailable()`。这样诊断工具仍能检查 provider 声明的全部模型，面向当前用户的选择界面则显示账号允许的子集。

### 失败路径与结论边界

`getAvailable()` 捕获不到 `provider.getModels()` 或 `filterModels()` 的异常；与 `getModels()` 的尽力聚合不同，这条异步可用性查询可能使 Promise 被拒绝。认证检查出错也会向上传递。源码因此区分了“尽量渲染目录”和“基于认证给出可信可用列表”。

模型出现在 `getAvailable()` 中仍不证明请求一定成功。它只证明读取时发现认证来源，并通过了 provider 的目录过滤。网络可达性、token 是否能刷新、服务端权限是否变化、请求参数是否合法，都要到实际 stream 调用才有结果。

## 5. 请求先按 provider 定位，再按 `model.api` 选择协议

### 解决的问题

一个 `Model` 同时携带 `provider` 和 `api`。运行时需要先找到谁负责认证和端点，再选择哪套协议序列化请求。若直接按 `api` 全局分派，会绕过 provider 的 OAuth、headers、base URL 和账号策略。

### 第一段：provider 与认证

`packages/ai/src/models.ts:445-477` 的 `applyAuth()` 先按 `model.provider` 找 provider，再解析认证：

```ts
private async applyAuth<TOptions extends StreamOptions & ModelsStreamTransforms>(
	model: Model<Api>,
	options: TOptions | undefined,
): Promise<{ requestModel: Model<Api>; requestOptions: StreamOptions | undefined }> {
	this.requireProvider(model);
	const resolution = await this.getAuth(model, {
		apiKey: options?.apiKey,
		env: options?.env,
	});
	if (!resolution) {
		throw new ModelsError("auth", `Provider is not configured: ${model.provider}`);
	}
	const auth = resolution.auth;
	const apiKey = options?.apiKey ?? auth.apiKey;
	let headers = mergeHeaders(auth.headers, options?.headers);
	if (options?.transformHeaders) headers = await options.transformHeaders(headers ?? {});
	const env = resolution.env || options?.env
		? { ...(resolution.env ?? {}), ...(options?.env ?? {}) }
		: undefined;
	const requestModel = auth.baseUrl ? { ...model, baseUrl: auth.baseUrl } : model;
	const { transformHeaders: _transformHeaders, ...providerOptions } = options ?? {};
	const requestOptions = { ...providerOptions, apiKey, headers, env } as StreamOptions;
	return { requestModel, requestOptions };
}
```

认证结果可以为本次请求替换 base URL，并参与 header 合并。这里产生的是请求局部的 model 副本，不会改写目录里的原始 `Model`。显式 options 中的 API key 和 headers 具有更高优先级，详细合并次序留到第 09 讲。

### 第二段：provider 内选择 API 实现

普通 provider 可以只配置一个 API 实现；混合 provider 则提供以 `model.api` 为键的映射。`packages/ai/src/providers/github-copilot.ts:28-32` 的注册如下：

```ts
api: {
	"anthropic-messages": anthropicMessagesApi(),
	"openai-completions": openAICompletionsApi(),
	"openai-responses": openAIResponsesApi(),
},
```

`packages/ai/src/models.ts:563-585` 在 provider 内完成选择：

```ts
const single =
	typeof (input.api as ProviderStreams).stream === "function"
		? (input.api as ProviderStreams)
		: undefined;
const byApi = single ? undefined : (input.api as Partial<Record<string, ProviderStreams>>);
const apiFor = (model: Model<Api>): ProviderStreams | undefined => single ?? byApi?.[model.api];

const dispatch = (
	model: Model<Api>,
	run: (streams: ProviderStreams) => AssistantMessageEventStream,
): AssistantMessageEventStream => {
	const streams = apiFor(model);
	if (!streams) {
		return lazyStream(model, async () => {
			throw new ModelsError("stream", `Provider ${input.id} has no API implementation for "${model.api}"`);
		});
	}
	return run(streams);
};
```

因此完整决策链是：

```text
Model.provider ──► Models.requireProvider()
                         │
                         ├─► 解析该 provider 的认证和端点
                         ▼
                   provider.stream()
                         │
                 Model.api 选择实现
                         ▼
              implementation.stream()
```

`Models.stream()` 本身也包在 `lazyStream()` 中。未知 provider、未配置凭据、缺少 API 实现都会在消费流后变成统一的 assistant error 结果，不会在调用 `stream()` 的那一刻同步抛出。第 07 讲讨论过这种延迟流语义；本讲需要补上的边界是：错误由哪一层发现。未知 provider 和认证问题由 `Models` 发现，API 映射缺口由 provider 发现，HTTP 或协议错误由 API 实现发现。

## 6. 新增 provider 的改动地图

“新增 provider”至少有三种情况。先判断目录来自哪里、是否复用现有协议，再决定修改面。直接复制另一家供应商的全部文件，往往会把无关的 OAuth、动态目录或兼容开关一起带进来。

### 情况 A：静态目录，复用现有 API

典型形式是一个 OpenAI-compatible 服务。最小运行时闭环包括：

1. 在 `packages/ai/src/types.ts` 的 `KnownProvider` 加入内置 id。第三方运行时注册不强制修改该联合类型；进入内置集合才需要。
2. 在 `packages/ai/scripts/generate-models.ts` 的数据获取、转换或 provider 映射中生成正确的 `Model`。不要手改 `*.models.ts`，脚本下次运行会先删除这些文件。
3. 运行生成脚本，得到 `packages/ai/src/providers/<id>.models.ts`，并让 `models.generated.ts` 聚合它。
4. 新建 `packages/ai/src/providers/<id>.ts` 工厂函数，声明认证、base URL、静态 models 和已有的延迟加载 API 实现。
5. 在 `packages/ai/src/providers/all.ts` 导入工厂函数，并加入 `builtinProviders()`。
6. 若 Coding Agent 要把它作为默认候选，在 `packages/coding-agent/src/core/model-resolver.ts` 的 `defaultModelPerProvider` 指定真实存在的默认 model id。
7. 补 provider 组装、认证、目录元数据、请求分派测试，并更新对应 README 与 changelog。

这里通常不需要修改 `packages/ai/src/api/`。provider 名称不同不等于协议不同；端点行为能由现有 compatibility metadata 准确描述时，应复用已有实现。

### 情况 B：动态目录，复用现有 API

除上述工厂函数和注册外，重点变为 `fetchModels`：它接收已解析凭据、中止信号和 provider 专属存储，返回规范化的 `Model[]`。若需要离线启动，工厂函数先从存储恢复；若使用 `createProvider()`，通用的恢复、覆盖、写缓存和并发去重已经封装好。

纯动态 provider 不必出现在 `models.generated.ts`。但若希望它成为内置 provider，仍要加入 `KnownProvider`、`builtinProviders()`，并检查 Coding Agent 默认选择、认证 UI 和文档是否认识它。Radius 证明“内置”与“拥有静态生成目录”是两件事。

### 情况 C：引入新的传输协议

除了 provider 工作，还要补齐 API 层：

1. 在 `packages/ai/src/types.ts` 增加 `KnownApi` 和 `ApiOptionsMap` 映射；
2. 在 `packages/ai/src/api/<api>.ts` 实现统一的 `stream`、`streamSimple` 契约；
3. 提供延迟加载封装，避免 provider 工厂在导入时加载较大的 SDK；
4. 从 `packages/ai/src/index.ts` 导出必要的 options 类型；包当前已有 `./api/*` 通配子路径导出，不需要为每个 API 单独增加 `package.json` 条目；
5. 补消息转换、工具调用、thinking、usage、结束原因、取消、重试和错误流测试；
6. 工厂函数把新实现作为单 API 对象，或加入按 `model.api` 选择的映射。

这时修改面变大的原因是统一协议层新增了一种语义。若所谓“新 API”只是 OpenAI-compatible 端点的一两个字段差异，优先判断现有 `compat` 是否能够精确表达；硬开新协议会复制大量流解析和失败处理。

### 一张可执行的判断表

| 问题 | 是 | 否 |
| --- | --- | --- |
| 要进入 Pi 内置 provider 集合吗 | 改 `KnownProvider`、`providers/all.ts`，检查 Coding Agent 默认模型 | 运行时 `setProvider()` 即可 |
| 模型目录能在构建期确定吗 | 接入生成脚本和 `*.models.ts` | 实现 `refreshModels` 或 `fetchModels` |
| 要跨进程保留动态目录吗 | 宿主注入持久化 `ModelsStore` | `InMemoryModelsStore` 足够 |
| 能准确复用现有协议吗 | 复用延迟加载 API，并设置必要 compat | 新增 API 实现与类型映射 |
| 同一 provider 有多种协议吗 | `api` 传映射，由 `model.api` 决策 | `api` 直接传单一实现 |
| 可见模型取决于账号权限吗 | 实现 `filterModels()` | 保持完整目录即可 |

## 7. 测试给出的行为边界

`packages/ai/test/models-runtime.test.ts` 和 `packages/ai/test/providers.test.ts` 不是只在验证“函数能调用”。它们固定了以下设计语义：

- 注册同 id provider 后，目录切换到新对象；删除后不再可见；一家 `getModels()` 抛错不影响其余目录。
- 多家动态 provider 并发刷新，一家失败时成功结果仍保留，失败以 provider 标识存入 `errors`。
- 有缓存、无网络的恢复路径能重建目录；成功网络刷新会写回目录和检查时间。
- 未配置凭据的 provider 不发目录请求；需要 OAuth 的 provider 在刷新目录前可以刷新过期 token。
- `checkAuth()` 不刷新 OAuth；`getAvailable()` 只返回找到认证来源的 provider。
- 显式 API key、环境认证、provider/model/options headers 按既定优先级合并。
- 混合 provider 根据 `model.api` 进入不同实现；缺少映射时得到 error stream。
- 并发刷新同一动态 provider 只调用一次 `fetchModels`。

这些测试把三类失败分开了：目录源失败进入 refresh 结果，目录聚合失败被降级为空，实际请求失败进入 assistant error stream。修改 provider 运行时时，不能用同一种 `throw` 方式粗暴替换三条路径，否则上层 UI、刷新状态和 agent loop 会得到不同的可观察结果。

## 8. 常见问题

### Provider 已注册，为什么模型列表仍为空？

注册只让 `Models` 知道 provider 对象。纯动态 provider 在恢复缓存或成功刷新之前，`getModels()` 可以为空；`getAvailable()` 还会排除没有认证来源的 provider。诊断时要依次区分注册状态、目录快照和认证可用性。

### 模型出现在静态目录里，是否说明现在就能调用？

不能。静态目录证明构建时收录过这个模型。`getAvailable()` 进一步证明当前读取到了认证来源，并通过账号过滤；只有一次实际请求才能验证 token 刷新、网络、服务端权限和协议参数。

### 动态刷新会把静态模型全部删掉吗？

使用 `createProvider()` 时不会。动态层只覆盖同 id 静态项，并追加新项；动态结果未覆盖的静态项仍保留。若工厂函数没有静态基线，刷新结果才构成完整目录。

### 为什么 `Models.refresh()` 不提供单 provider 参数？

固定 commit 的公开接口只接受 options，并发刷新全部动态 provider。README 中的单 provider 示例与当前代码不同。需要只刷新一家时，可以取得 provider 后调用其 `refreshModels()`，但调用方必须自行提供 `RefreshModelsContext`，这不是与 `Models.refresh()` 等价的便捷 API。

### 为什么目录缓存不直接写在 provider 文件里？

存储位置、权限和生命周期由宿主环境决定。库本身默认内存实现，Coding Agent 可以选择文件存储，浏览器或其他宿主也可以提供自己的后端。provider 只获得限权后的 `ProviderModelsStore`，既不依赖文件系统，也不能碰其他供应商的数据。

### 新增 OpenAI-compatible provider，是否一定要新增 API 实现？

不一定。只要消息格式、流事件和错误语义能由已有 OpenAI 实现及 compat metadata 准确表达，就只需新增 provider、目录和认证组合。只有现有协议抽象无法表达真实行为时，才应扩展 API 层。

## 结语

`Models` 保存“当前装入了哪些供应商”，`Provider` 保存“一家供应商怎样列模型、判断认证并选择协议”，API 实现负责“怎样在网络上传输统一请求”。目录读取采用最后已知快照，刷新按 provider 并发且隔离错误，实际请求再做完整认证解析。

这组边界解释了两类常见现象：模型列表可用并不等于请求可用；新增供应商也不必复制一套协议。第 09 讲继续沿认证分支进入凭据存储、API key、OAuth、短期 token 和请求 header 的生命周期。
