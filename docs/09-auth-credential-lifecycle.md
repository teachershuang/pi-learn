# 第 09 讲：认证与凭据生命周期

> 上游仓库：`earendil-works/pi`<br>
> 源码基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`<br>
> 版本描述：`v0.80.7-13-g5d9fedf73`

Pi 的认证代码要处理交互登录、环境变量、AWS 或 Google 的环境凭据、OAuth token 轮换、多个并发请求和持久化失败，模型协议层最终只接收一份简单请求参数。

```text
登录路径
用户界面 ──AuthInteraction──► ProviderAuth.login()
                                  │ 返回 Credential
                                  ▼
                         CredentialStore.modify()

请求路径
Model ──► Models.getAuth() ──► ProviderAuth.resolve() / OAuth refresh
                                  │ 返回 AuthResult
                                  ▼
                      合并 apiKey、headers、env、baseUrl
                                  ▼
                            Provider.stream*()
```

两条路径共享 provider 和凭据存储，但触发位置不同。登录路径允许询问用户并改变持久化状态；请求路径不能弹出登录界面，只能解析已有凭据、刷新过期 OAuth，或者明确失败。

## 1. `Credential`、`ModelAuth` 和状态信息不是同一个对象

### 解决的问题

认证数据有不同生命周期。refresh token 应该持久化，临时覆盖的 header 不应写入磁盘；状态界面需要显示“由 OAuth 配置”，但不能为了列状态而暴露 access token。若用一个宽泛对象贯穿所有阶段，调用方很容易把请求参数当作可持久化账号，或者在枚举账号时读取秘密。

### 源码入口与关键代码

`packages/ai/src/auth/types.ts:3-43` 定义了四种用途：

```ts
export interface ModelAuth {
	apiKey?: string;
	headers?: ProviderHeaders;
	baseUrl?: string;
}

export interface ApiKeyCredential {
	type: "api_key";
	key?: string;
	env?: ProviderEnv;
}

export interface OAuthCredentials {
	refresh: string;
	access: string;
	expires: number;
	[key: string]: unknown;
}

export interface OAuthCredential extends OAuthCredentials {
	type: "oauth";
}

export type Credential = ApiKeyCredential | OAuthCredential;

export interface CredentialInfo {
	providerId: string;
	type: Credential["type"];
}
```

`AuthResult` 和 `AuthCheck` 位于 `packages/ai/src/auth/types.ts:97-109`。前者补充 provider 解析出的环境配置和人类可读来源，后者只保留认证类型和来源。它们都不是新的凭据。

### 运行流程与状态变化

- `Credential` 以 provider 标识为键进入 `CredentialStore`。API key credential 可以保存 key，也可以只保存 provider 专属的环境配置；OAuth credential 保存 access、refresh、expires 及供应商附加字段。
- `CredentialInfo` 供账号列表和状态界面使用。它没有 key、access 或 refresh 字段。
- `AuthResult` 是一次解析结果。`auth` 字段会进入请求，`env` 交给具体 API 实现，`source` 只用于状态说明。
- `ModelAuth` 的 `baseUrl` 允许凭据改变本次请求端点。GitHub Copilot 的企业账号就是这个用法。

持久化 credential 不直接等于最终 Authorization header。OAuth access token 还要经过 `toAuth()`；AWS ambient credential 甚至可以返回空的 `ModelAuth`，让 AWS SDK 自己从凭据链取认证材料。

### 失败路径与设计取舍

`Credential` 使用 `type` 作为判别字段。一家 provider 当前只保存一份 credential，API key 与 OAuth 不会并存于同一个 store 项。这个模型简单，但切换认证方式会替换旧项，而不是保留多个可选账号。

另一种方案是把每个账号都保存为独立记录，请求时再选 active account。Pi 当前没有这层账号模型；provider 标识就是凭据存储的唯一键。扩展多账号能力时，不能只给 `OAuthCredential` 加字段，还要改变 store 键和选择流程。

## 2. API key 解析有明确优先级，ambient credential 不进入存储

### 解决的问题

同一家 provider 可能同时存在命令行临时 key、`auth.json` 中的 key、环境变量和云平台默认凭据。请求必须稳定选择一个来源。特别是 OAuth 刷新失败时，如果系统悄悄回退到环境 key，用户看到的账号和真正计费账号可能已经不同。

### 通用 API key provider

`packages/ai/src/auth/helpers.ts:9-24` 是普通环境变量 provider 使用的工厂：

```ts
export function envApiKeyAuth(name: string, envVars: readonly string[]): ApiKeyAuth {
	return {
		name,
		login: async (interaction) => {
			const key = await interaction.prompt({ type: "secret", message: `Enter ${name}` });
			return { type: "api_key", key };
		},
		resolve: async ({ ctx, credential }) => {
			if (credential?.key) return { auth: { apiKey: credential.key }, source: "stored credential" };
			for (const envVar of envVars) {
				const value = await ctx.env(envVar);
				if (value) return { auth: { apiKey: value }, source: envVar };
			}
			return undefined;
		},
	};
}
```

登录得到的 key 优先于环境变量。若 store 中没有 credential，resolver 才检查传入的环境变量名。环境变量只是请求期输入，不会被复制到 credential store。

### 总解析顺序

`packages/ai/src/auth/resolve.ts:37-68` 是 API key 和 OAuth 共用的入口：

```ts
export async function resolveProviderAuth(
	provider: { id: string; auth: ProviderAuth },
	credentials: CredentialStore,
	authContext: AuthContext,
	overrides?: AuthResolutionOverrides,
): Promise<AuthResult | undefined> {
	const requestAuthContext = overrides?.env ? overlayEnvAuthContext(authContext, overrides.env) : authContext;

	if (overrides?.apiKey !== undefined && provider.auth.apiKey) {
		return resolveApiKey(requestAuthContext, provider.auth.apiKey, provider.id, {
			type: "api_key",
			key: overrides.apiKey,
			env: overrides.env,
		});
	}

	const stored = await readCredential(credentials, provider.id);
	if (stored) {
		if (stored.type === "oauth" && provider.auth.oauth) {
			return resolveStoredOAuth(credentials, provider.id, provider.auth.oauth, stored);
		}
		if (stored.type === "api_key" && provider.auth.apiKey) {
			const credential = overrides?.env ? { ...stored, env: { ...stored.env, ...overrides.env } } : stored;
			return resolveApiKey(requestAuthContext, provider.auth.apiKey, provider.id, credential);
		}
		return undefined;
	}

	return provider.auth.apiKey
		? resolveApiKey(requestAuthContext, provider.auth.apiKey, provider.id, undefined)
		: undefined;
}
```

决策顺序是：

1. 请求显式传入的 `apiKey`；
2. store 中该 provider 的 credential；
3. provider 自己解析的环境变量、配置文件或 ambient credential。

请求级 `env` 会覆盖默认 `AuthContext.env()`，也会合并进已存 API key credential 的 `env`。它是本次解析的输入，不改变持久化数据。

### “stored credential owns the provider”

只要 store 中存在 credential，ambient fallback 就停止。即使存的是 OAuth credential，而 provider 当前只注册了 API key handler，结果也是 `undefined`，不会改用环境 key。`packages/ai/test/models-runtime.test.ts:366-394` 和 `447-455` 分别固定了正常优先级与错误类型阻断回退的行为。

这条规则让账号选择可预测。代价是陈旧或类型不匹配的存储项会让一个原本可由环境变量调用的 provider 显示为未配置。修复方式是重新登录或删除旧 credential，不是继续增加隐式回退。

### ambient credential 不一定是字符串 key

`packages/ai/src/providers/amazon-bedrock.ts:52-69` 会检查 bearer token、AWS profile、access key 对、ECS role 和 web identity。profile 或 IAM 路径返回 `{ auth: {}, source: ... }`，真正签名由 Bedrock API 实现中的 AWS SDK 完成。

Vertex 的 `ApiKeyAuth.resolve()` 也可以检查 ADC 文件、project 和 location，入口在 `packages/ai/src/providers/google-vertex.ts:62-83`。`ApiKeyAuth` 这个名字表示 provider 的非 OAuth 认证分支，不代表所有实现最终都产生一个 API key 字符串。

## 3. 登录是 provider 驱动的交互协议

### 解决的问题

Anthropic 需要授权 URL 和手工 code，GitHub Copilot 使用 device code，Bedrock 允许选择 bearer token、profile 或 credential chain。`pi-ai` 不能依赖某个 TUI 组件，但 provider 又必须能发出提示、显示链接和报告进度。

### 交互边界

`packages/ai/src/auth/types.ts:131-155` 把登录界面压缩为两个回调：

```ts
export type AuthEvent =
	| { type: "info"; message: string; links?: readonly AuthInfoLink[] }
	| { type: "auth_url"; url: string; instructions?: string }
	| {
			type: "device_code";
			userCode: string;
			verificationUri: string;
			intervalSeconds?: number;
			expiresInSeconds?: number;
	  }
	| { type: "progress"; message: string };

export interface AuthInteraction {
	signal?: AbortSignal;
	prompt(prompt: AuthPrompt): Promise<string>;
	notify(event: AuthEvent): void;
}
```

provider 决定何时 prompt、提示类型是什么、收到字符串后如何换 token。调用方决定怎样显示输入框、是否打开浏览器、怎样取消。`AuthPrompt.signal` 还能单独取消某个手工输入：若本地 callback server 先收到授权结果，等待 code 的输入框可以结束，而整个登录流程继续完成。

### 凭据何时写入

`packages/ai/src/models.ts:421-443` 负责选择登录方法和持久化：

```ts
async login(providerId: string, type: AuthType, interaction: AuthInteraction): Promise<Credential> {
	const provider = this.providers.get(providerId);
	if (!provider) throw new ModelsError("provider", `Unknown provider: ${providerId}`);
	const method = type === "oauth" ? provider.auth.oauth : provider.auth.apiKey;
	if (!method?.login) {
		throw new ModelsError("auth", `${provider.name} does not support ${type} login`);
	}
	const credential = await method.login(interaction);
	try {
		await this.credentials.modify(providerId, async () => credential);
	} catch (error) {
		throw new ModelsError("auth", `Credential store modify failed for ${providerId}`, { cause: error });
	}
	return credential;
}

async logout(providerId: string): Promise<void> {
	try {
		await this.credentials.delete(providerId);
	} catch (error) {
		throw new ModelsError("auth", `Credential store delete failed for ${providerId}`, { cause: error });
	}
}
```

登录方法完整返回 credential 后，`Models` 才调用 `modify()`。用户取消、授权服务器报错或 token exchange 失败时，不会写入半成品。若外部授权已经成功，但本地存储失败，调用方收到 `auth` 错误；这次 credential 没有成为本地持久状态。

`logout()` 只删除本地 credential。源码没有调用 provider 的 revoke endpoint，因此它不是远端 token 吊销。需要彻底撤销授权时，还要到供应商账号侧操作。

### Coding Agent 的 UI 只实现回调

`packages/coding-agent/src/modes/interactive/interactive-mode.ts:5178-5224` 把 `AuthPrompt` 投影为选择框、手工输入或普通输入，把 `AuthEvent` 投影为 URL、device code、info 或 progress。最后 `loginProvider()` 只把这两个回调交给 `ModelRuntime.login()`。

请求路径不会进入这些组件。`ModelRuntime.login()` 在 `packages/coding-agent/src/core/model-runtime.ts:483-494` 委托 `Models.login()`，成功后刷新模型目录和可用性快照；普通 `stream()` 只做认证解析，不会在缺少 credential 时自动弹框。

## 4. `CredentialStore.modify()` 是凭据一致性的核心

### 解决的问题

OAuth provider 经常轮换 refresh token。若两个请求同时发现 access token 过期，各自刷新并写回，其中一个可能覆盖另一个刚得到的新 refresh token。进程内 Promise 去重不够：两个 Pi 进程可能同时读写同一个 `auth.json`。

### Store 契约

`packages/ai/src/auth/types.ts:60-88` 把所有写入收口到 `modify()` 和 `delete()`：

```ts
export interface CredentialStore {
	read(providerId: string): Promise<Credential | undefined>;

	list(): Promise<readonly CredentialInfo[]>;

	modify(
		providerId: string,
		fn: (current: Credential | undefined) => Promise<Credential | undefined>,
	): Promise<Credential | undefined>;

	delete(providerId: string): Promise<void>;
}
```

`modify()` 不是普通 setter。回调必须在串行化的读改写区间内看到当前值；返回新 credential 才写入，返回 `undefined` 表示保持当前项。删除必须走 `delete()`，避免“无修改”和“删除”共享同一个返回值。

`InMemoryCredentialStore` 在 `packages/ai/src/auth/credential-store.ts:8-50` 为每个 provider 保存一条 Promise 链。同一 provider 的写操作串行，不同 provider 可以并发。这个保证只覆盖当前进程。

### Coding Agent 的持久化实现

Coding Agent 默认在 `ModelRuntime.create()` 中用 `AuthStorage` 包装 `auth.json`，再套一层 `RuntimeCredentials`，入口位于 `packages/coding-agent/src/core/model-runtime.ts:130-164`。

`AuthStorage.modify()` 在文件锁内重新解析磁盘内容，然后只替换目标 provider，源码位于 `packages/coding-agent/src/core/auth-storage.ts:217-254`：

```ts
async read(provider: string): Promise<Credential | undefined> {
	const credential = this.data[provider];
	if (credential?.type !== "api_key") return credential;
	if (credential.key === undefined) return credential;
	return { ...credential, key: resolveConfigValue(credential.key, credential.env) };
}

async modify(
	provider: string,
	fn: (current: Credential | undefined) => Promise<Credential | undefined>,
): Promise<Credential | undefined> {
	return this.storage.withLockAsync(async (content) => {
		const currentData = this.parseStorageData(content);
		const next = await fn(currentData[provider]);
		if (next === undefined) {
			this.data = currentData;
			return { result: currentData[provider] };
		}

		const merged: AuthStorageData = { ...currentData, [provider]: next };
		this.data = merged;
		return { result: next, next: JSON.stringify(merged, null, 2) };
	});
}
```

每次写入都以锁内重新读取的文件为基准，别的进程刚写入的 provider 不会被旧内存快照覆盖。文件初始化和写入使用 `mode: 0o600`，并调用 `chmodSync()`，见 `packages/coding-agent/src/core/auth-storage.ts:21-46`、`72-89` 和 `97-139`。

测试 `packages/coding-agent/test/auth-storage.test.ts:73-128` 覆盖了外部修改保留、并发写入和定点删除；`140-216` 还确认获取锁失败时不写文件、锁失效后允许重试、畸形 JSON 不会被新内容覆盖。

### 命令型 key 与枚举边界

Coding Agent 允许 `auth.json` 的 key 写成 `$ENV_VAR` 或 `!command`。`AuthStorage.read()` 调用 `resolveConfigValue()`，后者会展开环境变量或执行命令，具体行为在 `packages/coding-agent/src/core/resolve-config-value.ts:138-180`。

因此状态枚举必须调用 `list()`，不能为了显示 provider 列表逐个 `read()`。`list()` 只从当前数据快照返回 `{ providerId, type }`，不会解析 key，也不会执行命令。命令型配置的信任和权限边界留到第 23 讲；这里先确认一个源码事实：读取有效 credential 可能有执行行为，枚举 metadata 没有。

### 运行时 key 不落盘

`RuntimeCredentials` 在持久 store 前放一层内存覆盖。`packages/coding-agent/src/core/runtime-credentials.ts:12-35` 的读取逻辑如下：

```ts
setRuntimeApiKey(providerId: string, apiKey: string): void {
	this.overrides.set(providerId, apiKey);
}

removeRuntimeApiKey(providerId: string): void {
	this.overrides.delete(providerId);
}

hasRuntimeApiKey(providerId: string): boolean {
	return this.overrides.has(providerId);
}

async read(providerId: string): Promise<Credential | undefined> {
	const override = this.overrides.get(providerId);
	return override ? { type: "api_key", key: override } : this.store.read(providerId);
}

async list(): Promise<readonly CredentialInfo[]> {
	const entries = new Map((await this.store.list()).map((entry) => [entry.providerId, entry]));
	for (const providerId of this.overrides.keys()) {
		entries.set(providerId, { providerId, type: "api_key" });
	}
	return [...entries.values()];
}
```

运行时 key 会遮住同 provider 的持久 credential，但不会改写它。移除覆盖后，原 credential 再次可见。`delete()` 同时删除覆盖和底层项，因此经 `ModelRuntime.logout()` 退出时，两层状态都会清除。

## 5. OAuth 刷新使用双重检查锁

### 解决的问题

access token 有有效期，refresh token 还可能在刷新时轮换。快速路径不应为每个请求获取文件锁；过期路径又必须保证所有进程只有一个刷新者，并让等待者使用它写回的新 token。

### OAuth provider 的职责

`packages/ai/src/auth/types.ts:184-207` 把 OAuth 分成三个动作：

- `login(interaction)` 与用户和授权服务器交互，返回可存储 credential；
- `refresh(credential)` 访问 token endpoint，返回轮换后的 credential；
- `toAuth(credential)` 无副作用地生成本次请求的 `ModelAuth`。

分开 `refresh()` 和 `toAuth()` 后，锁由 `Models` 控制，provider 不需要知道 credential 存在内存、文件还是数据库。GitHub Copilot 的 `toAuth()` 还会从 credential 推导账号专属 base URL，源码在 `packages/ai/src/auth/oauth/github-copilot.ts:361-379`。

### 过期请求的状态流转

`packages/ai/src/auth/resolve.ts:84-118` 实现双重检查：

```ts
async function resolveStoredOAuth(
	credentials: CredentialStore,
	providerId: string,
	oauth: OAuthAuth,
	stored: OAuthCredential,
): Promise<AuthResult | undefined> {
	let credential = stored;

	if (Date.now() >= credential.expires) {
		let post: Credential | undefined;
		try {
			post = await credentials.modify(providerId, async (current) => {
				if (current?.type !== "oauth") return undefined;
				if (Date.now() < current.expires) return undefined;
				try {
					return await oauth.refresh(current);
				} catch (error) {
					throw new ModelsError("oauth", `OAuth refresh failed for ${providerId}`, { cause: error });
				}
			});
		} catch (error) {
			if (error instanceof ModelsError) throw error;
			throw new ModelsError("auth", `Credential store modify failed for ${providerId}`, { cause: error });
		}
		if (post?.type !== "oauth") return undefined;
		credential = post;
	}

	try {
		return { auth: await oauth.toAuth(credential), source: "OAuth" };
	} catch (error) {
		throw new ModelsError("oauth", `OAuth auth derivation failed for ${providerId}`, { cause: error });
	}
}
```

第一次 expiry check 是无锁快速判断。token 过期后进入 `modify()`，回调重新读取当前 credential：

- 用户已在等待期间 logout 或改成 API key：保持当前状态，解析结果为未配置；
- 另一个请求已经刷新：不再访问 token endpoint，直接使用 store 返回的新 credential；
- 仍是过期 OAuth：当前调用者刷新并在释放锁前写回。

有效 token 不调用 `modify()`。`packages/ai/test/models-runtime.test.ts:492-537` 用两个并发 `getAuth()` 断言只刷新一次，也验证有效 token 走无锁路径。

不少 provider 会在写入 `expires` 时预留提前量。例如 Anthropic 在 `packages/ai/src/auth/oauth/anthropic.ts:335-350` 保存服务端有效期时减去五分钟。通用 resolver 只比较 `Date.now()` 与 `expires`，不认识各家安全窗口；窗口属于 provider 的 token 解释职责。

### 刷新失败留下什么

`oauth.refresh()` 在 `modify()` 回调内抛错时，新值没有产生，store 保留旧 credential。错误以 `ModelsError("oauth")` 向上返回，不会删除 credential，也不会尝试环境变量。用户可以稍后重试；若 refresh token 已失效，则需要重新登录覆盖旧项。

这与“清掉旧 token 再刷新”相比更利于诊断，也避免临时网络故障直接丢失 refresh token。代价是同一个坏 credential 会持续触发失败，直到用户重新登录或 logout。

## 6. 请求认证最后组装，header 按名称大小写无关地覆盖

### 解决的问题

认证 resolver 可能产生 API key、Authorization header、provider 环境配置或动态 base URL；模型目录还可以声明 model headers；调用方又可能为单次请求传入 header。合并必须只有一个确定顺序，且不能同时保留 `Authorization` 和 `authorization` 两个逻辑重复字段。

### provider 级与 model 级解析

`packages/ai/src/models.ts:401-419` 的两个 `getAuth()` overload 只差 model headers：

```ts
getAuth(providerId: string, overrides?: AuthResolutionOverrides): Promise<AuthResult | undefined>;
getAuth(model: Model<Api>, overrides?: AuthResolutionOverrides): Promise<AuthResult | undefined>;
async getAuth(
	providerOrModel: string | Model<Api>,
	overrides?: AuthResolutionOverrides,
): Promise<AuthResult | undefined> {
	const providerId = typeof providerOrModel === "string" ? providerOrModel : providerOrModel.provider;
	const provider = this.providers.get(providerId);
	if (!provider) return undefined;
	const result = await resolveProviderAuth(provider, this.credentials, this.authContext, overrides);
	if (!result || typeof providerOrModel === "string" || !providerOrModel.headers) return result;
	return {
		...result,
		auth: {
			...result.auth,
			headers: mergeHeaders(result.auth.headers, providerOrModel.headers),
		},
	};
}
```

`getAuth(providerId)` 适合状态和 provider 级诊断；`getAuth(model)` 再加上该模型的静态 headers。调用方若先 `getAuth(model)` 再调用 `stream()`，认证会解析两次。README 建议直接使用 stream 的 options 和 `transformHeaders`。

### 最终合并顺序

`packages/ai/src/models.ts:453-476` 在 provider dispatch 前组装请求：

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
	const env = resolution.env || options?.env ? { ...(resolution.env ?? {}), ...(options?.env ?? {}) } : undefined;
	const requestModel = auth.baseUrl ? { ...model, baseUrl: auth.baseUrl } : model;
	const { transformHeaders: _transformHeaders, ...providerOptions } = options ?? {};
	const requestOptions = { ...providerOptions, apiKey, headers, env } as StreamOptions;

	return { requestModel, requestOptions };
}
```

顺序可以写成：

```text
provider 解析出的 auth headers
        ↓
model.headers
        ↓
显式 options.headers
        ↓
transformHeaders() 最后改写
        ↓
Provider.stream*()
```

`mergeHeaders()` 在 `packages/ai/src/models.ts:198-212` 逐项进行大小写无关比较。后来的 `authorization` 会先删除已有 `Authorization`，再以新名字写入。显式 header 因此覆盖认证和模型 header；transform 得到完整结果，拥有最后修改权。

`transformHeaders` 是 `Models` 专属选项。调用 provider 前会从 options 中移除，provider 只收到普通 headers。`packages/ai/test/models-runtime.test.ts:603-663` 验证了显式 API key、动态 base URL、大小写无关覆盖、model header 和 transform 只运行一次。

认证得到的 base URL 不会改写目录中的 `Model`。`applyAuth()` 为本次请求创建 model 副本，这样 GitHub Copilot 账号端点或其他凭据派生端点不会污染共享模型快照。

## 7. 失败语义按阶段区分

### 直接调用 `getAuth()`

`getAuth()` 返回 `undefined` 表示 provider 未知或当前没有可用认证配置。它在真实故障时 reject：

| 阶段 | `ModelsError.code` | 留下的状态 |
| --- | --- | --- |
| credential store 读取、锁或写入失败 | `auth` | 原存储项保持；具体取决于 store 是否完成写入 |
| API key resolver 抛错 | `auth` | credential 不变 |
| OAuth refresh 失败 | `oauth` | 旧 OAuth credential 保留 |
| OAuth `toAuth()` 失败 | `oauth` | credential 已是当前值，不做删除 |
| provider 未注册 | 返回 `undefined` | 无状态变化 |

登录时未知 provider 使用 `provider` 错误；认证方式没有 `login()` 使用 `auth` 错误。logout 的 delete 失败也包装为 `auth`。

### 请求路径

`stream()` 和 `streamSimple()` 在 `lazyStream()` 内调用 `applyAuth()`。未配置 provider 会产生 `auth` 错误，未知 provider 产生 `provider` 错误，OAuth 和 store 错误保留原 code；这些错误最终表现为 `stopReason: "error"` 的 assistant message。调用 `stream()` 本身不会同步抛出。

`complete()` 和 `completeSimple()` 等待流的 `result()`，得到同一条错误 assistant message。上层 agent loop 因此可以把认证失败纳入统一模型响应失败路径，而登录和 `getAuth()` 仍以 Promise rejection 报告操作失败。

### `checkAuth()` 的特殊边界

第 08 讲已经确认，`checkAuth()` 只判断配置来源，不刷新过期 OAuth。它适合模型选择器和状态显示；真正请求或显式 `getAuth()` 才执行 token refresh。把 `checkAuth()` 的成功当作“token 仍有效”，会高估当前请求能力。

## 8. 测试固定的行为

认证相关测试同时检查返回值、状态变化和并发语义：

- `packages/ai/test/models-runtime.test.ts:366-455`：stored credential 阻断 ambient fallback，登录写入 store，logout 删除本地项，类型不匹配时不偷偷换认证来源。
- `packages/ai/test/models-runtime.test.ts:457-537`：过期 token 刷新后持久化；刷新失败保留旧项；两个并发请求只刷新一次；有效 token 不获取修改锁。
- `packages/ai/test/models-runtime.test.ts:539-577`：store 和 API key resolver 的异常统一包装为 `auth`。
- `packages/ai/test/models-runtime.test.ts:579-663`：显式 key/env 的优先级、header 合并、base URL 副本和 transform 消费位置。
- `packages/ai/test/oauth-auth.test.ts:25-126`：Anthropic、OpenAI Codex 和 GitHub Copilot 的 `toAuth()`、refresh token 交换及 Copilot 动态端点。
- `packages/coding-agent/test/auth-storage.test.ts:73-216`：跨实例写入、文件锁失败、锁失效和畸形 JSON 的保留策略。
- `packages/coding-agent/test/runtime-credentials.test.ts:6-39`：运行时 key 遮住持久项但不落盘，枚举不泄露 key，logout 同时清除两层。

修改认证层时，至少要分别验证登录写入、请求解析、并发刷新和 header 合并。只跑某个 OAuth provider 的 token exchange 测试，无法证明 credential store 的锁和请求失败语义仍然正确。

## 9. 常见问题

### `auth.json` 中有 credential，为什么环境变量反而不生效？

stored credential 拥有该 provider。只要存储项存在，resolver 就不再走 ambient fallback；类型不匹配也一样。删除旧项或重新登录后，环境变量才重新进入候选路径。请求显式传入 `apiKey` 是更高优先级的例外。

### OAuth token 过期时，每个并发请求都会刷新吗？

不会。所有请求可能同时通过第一次过期判断，但只有拿到 `CredentialStore.modify()` 锁后仍看到过期 credential 的调用者会刷新。其余调用者读到新 token 后直接使用。跨进程是否成立，取决于 store 是否提供跨进程锁；默认内存 store 只保证单进程。

### `models.login()` 与 `models.getAuth()` 有什么根本区别？

`login()` 允许用户交互，成功后写 credential store；`getAuth()` 只解析已有来源，必要时刷新过期 OAuth，并返回请求材料。请求缺少 credential 时不会自动调用 `login()`。

### `logout()` 会让供应商服务器上的 token 失效吗？

不会。当前实现只调用 `CredentialStore.delete()`。它清理 Pi 本地状态，没有远端 revoke 调用。远端撤销需要使用供应商提供的账号或授权管理入口。

### 为什么 `list()` 和 `read()` 要分开？

`list()` 只返回 provider 标识和 credential 类型，适合状态枚举。`read()` 返回秘密，Coding Agent 还可能解析 `$ENV_VAR` 或执行 `!command`。界面只想显示“哪些 provider 已存凭据”时调用 `list()`，可以避免不必要的秘密解析和命令执行。

### 为什么 OAuth 的 `toAuth()` 不直接在 refresh 中返回 header？

refresh 的输出需要持久化，`toAuth()` 的输出只属于请求。分离后，store 保存结构稳定的 credential，request-time base URL 或 header 可以每次从当前 credential 推导；`Models` 也能在锁内统一处理 token 轮换。

### 能否把所有认证都简化成 `apiKey: string`？

Bedrock 的 IAM credential chain、Vertex ADC、GitHub Copilot 的账号专属端点都需要 key 之外的信息。`ModelAuth` 只保留协议请求直接需要的 apiKey、headers 和 baseUrl，其余 provider 配置通过 `AuthResult.env` 传递。

## 结语

Pi 把认证分成三个时间点：登录产生并保存 credential，请求前把 credential 解析成 `AuthResult`，OAuth 过期时在 store 的串行化读改写区间内刷新。环境凭据只在没有存储项时参与，显式请求参数拥有最高优先级。

这套实现的重点是状态所有权：provider 解释认证语义，应用拥有存储，`Models` 拥有解析、刷新锁和请求合并。第 10 讲将从已经组装好的请求参数继续进入 Anthropic、OpenAI Responses 和 Google 的消息转换与流解析。
