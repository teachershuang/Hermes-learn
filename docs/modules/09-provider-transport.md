# Provider / Transport：Hermes 怎么接住不同模型供应商

## 目标与范围

Agent 看起来是在“调用一个模型”，但工程上并不是这样。

不同供应商的差异很多：

```text
OpenAI Chat Completions 用 messages
OpenAI Responses 用 input 和 response item
Anthropic Messages 把 system、messages、tools 拆成另一套结构
Bedrock Converse 又是 AWS 的协议
OpenRouter、Nous、Qwen、Kimi、NVIDIA 虽然都说 OpenAI-compatible，但字段支持并不完全一样
```

如果主循环里到处写：

```python
if provider == "...":
    ...
elif api_mode == "...":
    ...
```

代码会很快失控。每加一个 provider，都要改消息转换、工具 schema、max_tokens、reasoning、缓存统计、错误恢复、模型列表、认证方式。

Hermes 的做法是把问题拆成两层：

```text
ProviderProfile：这个 provider 是谁，有什么 endpoint、认证方式和请求怪癖
ProviderTransport：这个 api_mode 怎么把 Hermes 内部消息转成供应商请求，再把响应转回统一形状
```

再加上 runtime provider 解析、fallback 和 credential pool，完整链路是：

```text
配置 / CLI / OAuth / credential pool
  -> resolve_runtime_provider
  -> AIAgent 初始化 provider / model / base_url / api_mode
  -> get_transport(api_mode)
  -> transport.build_kwargs(...)
  -> provider SDK / HTTP call
  -> transport.normalize_response(...)
  -> Hermes 主循环继续处理文本和 tool calls
  -> 失败时 credential pool 或 fallback 接管
```

本章沿这条链区分 `provider`、`model`、`base_url`、`api_mode`、`transport`、`profile` 和 `credential pool` 的职责。

## 功能 1：Provider 和 Transport 不是一回事

首先区分两个概念。

`provider` 是供应商身份，比如 `openrouter`、`anthropic`、`nous`、`bedrock`、`xai`、`azure-foundry`。

`api_mode` 是 API 协议路径，比如：

```text
chat_completions
codex_responses
anthropic_messages
bedrock_converse
codex_app_server
```

一个 provider 会选择一个 api_mode。多个 provider 可以共享同一个 api_mode。比如 OpenRouter、Nous、DeepSeek、NVIDIA、Kimi 等大量供应商都走 `chat_completions`，但它们的细节并不完全相同。

所以 Hermes 把它拆成两层：

| 层 | 解决什么问题 |
| --- | --- |
| `ProviderProfile` | 这个供应商的身份、endpoint、认证方式、默认模型、请求怪癖 |
| `ProviderTransport` | 这个 API 协议如何转换消息、工具、参数和响应 |

这些对象之间的关系如下：

```text
provider 多而常变
transport 少而稳定
```

新增一个 OpenAI-compatible provider，不应该新增一套 transport。通常只需要新增一个 `ProviderProfile`，让它复用 `chat_completions` transport。

## 功能 2：ProviderProfile 把供应商信息声明出来

源码入口是 `providers/base.py`：

```python
@dataclass
class ProviderProfile:
    name: str
    api_mode: str = "chat_completions"
    aliases: tuple = ()

    display_name: str = ""
    description: str = ""
    signup_url: str = ""

    env_vars: tuple = ()
    base_url: str = ""
    models_url: str = ""
    auth_type: str = "api_key"
    supports_health_check: bool = True

    fallback_models: tuple = ()
    hostname: str = ""

    default_headers: dict[str, str] = field(default_factory=dict)
    fixed_temperature: Any = None
    default_max_tokens: int | None = None
    default_aux_model: str = ""
```

这个类是声明式的。源码开头说得很清楚：

```python
"""Provider profiles are DECLARATIVE — they describe the provider's behavior.
They do NOT own client construction, credential rotation, or streaming.
Those stay on AIAgent.
"""
```

也就是说，profile 不负责真正发请求。它只告诉系统：

```text
这个 provider 叫什么
别名有哪些
默认走哪个 api_mode
base_url 是什么
需要哪些环境变量
models endpoint 在哪里
默认 max_tokens 怎么处理
是否支持 vision / tool-result images
是否要加特殊 header / extra_body
```

复杂 provider 可以覆写 hook：

```python
def prepare_messages(self, messages): ...

def build_extra_body(self, *, session_id=None, **context): ...

def build_api_kwargs_extras(self, *, reasoning_config=None, **context):
    return {}, {}

def get_max_tokens(self, model: str | None) -> int | None:
    return self.default_max_tokens
```

这比在主循环里加一堆 `if provider == "kimi"`、`if provider == "openrouter"` 要干净。provider 的怪癖留在 provider 自己的 profile 里。

## 功能 3：Provider 插件注册表让新增供应商不必改核心代码

Provider profile 不是硬编码在一个大列表里。注册表在 `providers/__init__.py`：

```python
_REGISTRY: dict[str, ProviderProfile] = {}
_ALIASES: dict[str, str] = {}
_discovered = False

def register_provider(profile: ProviderProfile) -> None:
    _REGISTRY[profile.name] = profile
    for alias in profile.aliases:
        _ALIASES[alias] = profile.name
```

查询时才惰性发现：

```python
def get_provider_profile(name: str) -> ProviderProfile | None:
    if not _discovered:
        _discover_providers()
    canonical = _ALIASES.get(name, name)
    return _REGISTRY.get(canonical)
```

发现顺序是：

```text
仓库自带 plugins/model-providers/<name>/
  -> 用户自己的 $HERMES_HOME/plugins/model-providers/<name>/
  -> 旧式 providers/<name>.py
```

源码里允许 user plugins 覆盖 bundled profiles：

```python
"""Later registrations with the same name replace earlier ones — so user
plugins under ``$HERMES_HOME/plugins/model-providers/`` can override
bundled profiles without editing repo code.
"""
```

这让 provider 集成从“改核心代码”变成“新增插件目录”。目前仓库里能看到很多 provider 插件，例如：

```text
plugins/model-providers/anthropic
plugins/model-providers/openrouter
plugins/model-providers/nvidia
plugins/model-providers/azure-foundry
plugins/model-providers/bedrock
plugins/model-providers/kimi-coding
plugins/model-providers/novita
```

模型供应商接口变化频繁。如果每个 Provider 都直接修改 `run_agent.py`，主循环会逐步变成供应商兼容层。Hermes 将这部分移到 plugin/profile 层，主循环只读取已经解析的 Runtime 信息。

## 功能 4：Transport 把 API 协议差异压进一个类

Transport 抽象在 `agent/transports/base.py`：

```python
class ProviderTransport(ABC):
    @property
    @abstractmethod
    def api_mode(self) -> str:
        ...

    @abstractmethod
    def convert_messages(self, messages: List[Dict[str, Any]], **kwargs) -> Any:
        ...

    @abstractmethod
    def convert_tools(self, tools: List[Dict[str, Any]]) -> Any:
        ...

    @abstractmethod
    def build_kwargs(self, model, messages, tools=None, **params) -> Dict[str, Any]:
        ...

    @abstractmethod
    def normalize_response(self, response: Any, **kwargs) -> NormalizedResponse:
        ...
```

它的职责很窄：

```text
把 Hermes 内部消息格式转成 provider 请求
把 Hermes 工具 schema 转成 provider 工具格式
组装 SDK / HTTP 调用参数
把 provider 原始响应规范化
```

它不负责：

```text
client 生命周期
streaming 控制
认证刷新
credential rotation
fallback
interrupt
prompt cache 策略
```

这些逻辑仍在 `AIAgent` 和相关 Runtime helper 中。Transport 只负责数据格式转换，不是供应商客户端管理器。

## 功能 5：Transport 注册表按 api_mode 惰性加载

注册表在 `agent/transports/__init__.py`：

```python
_REGISTRY: dict = {}
_discovered: bool = False

def register_transport(api_mode: str, transport_cls: type) -> None:
    _REGISTRY[api_mode] = transport_cls

def get_transport(api_mode: str):
    global _discovered
    if not _discovered:
        _discover_transports()
    cls = _REGISTRY.get(api_mode)
    if cls is None:
        _discover_transports()
        cls = _REGISTRY.get(api_mode)
    if cls is None:
        return None
    return cls()
```

发现函数导入各 transport 模块：

```python
def _discover_transports() -> None:
    try:
        import agent.transports.anthropic
    except ImportError:
        pass
    try:
        import agent.transports.codex
    except ImportError:
        pass
    try:
        import agent.transports.chat_completions
    except ImportError:
        pass
    try:
        import agent.transports.bedrock
    except ImportError:
        pass
```

这里有三个工程细节。

第一，transport 是按需 import，不在启动时强行加载所有 SDK 和适配器。

第二，每个 import 都用 `try/except ImportError` 包住。某个 provider 相关依赖缺失，不应该让其他 provider 不能用。

第三，`get_transport` miss 时会再 discover 一次，避免测试或乱序 import 导致注册表只加载了一半。

`AIAgent` 侧还有一层实例缓存：

```python
def _get_transport(self, api_mode: str = None):
    mode = api_mode or self.api_mode
    cache = getattr(self, "_transport_cache", None)
    if cache is None:
        cache = {}
        self._transport_cache = cache
    t = cache.get(mode)
    if t is None:
        from agent.transports import get_transport
        t = get_transport(mode)
        cache[mode] = t
    return t
```

这说明 transport 是轻量对象，但仍然按 `api_mode` 缓存。fallback 或 model switch 改变 `api_mode` 时，Hermes 会清掉 `_transport_cache`，避免旧 transport 被误用。

## 功能 6：NormalizedResponse 是主循环的统一响应形状

不同 provider 返回的响应结构不一样。Hermes 不希望主循环每个地方都知道 Anthropic、Codex、Bedrock 的原始结构，于是 transport 会返回 `NormalizedResponse`。

源码在 `agent/transports/types.py`：

```python
@dataclass
class NormalizedResponse:
    content: str | None
    tool_calls: list[ToolCall] | None
    finish_reason: str
    reasoning: str | None = None
    usage: Usage | None = None
    provider_data: dict[str, Any] | None = field(default=None, repr=False)
```

工具调用也统一成 `ToolCall`：

```python
@dataclass
class ToolCall:
    id: str | None
    name: str
    arguments: str
    provider_data: dict[str, Any] | None = field(default=None, repr=False)
```

`provider_data` 是保留协议细节的地方。比如：

```text
Codex: call_id、response_item_id
Gemini: thought_signature
Anthropic: reasoning_details / content_blocks
```

这是一种很实用的折中。主循环可以读统一字段：`content`、`tool_calls`、`finish_reason`。但协议特有信息不会丢，它被放进 `provider_data`，只给真正懂该协议的路径读取。

## 功能 7：api_mode 在 agent 初始化时被确定

`AIAgent` 初始化时会设置 `agent.provider` 和 `agent.api_mode`。源码在 `agent/agent_init.py`：

```python
provider_name = provider.strip().lower() if isinstance(provider, str) and provider.strip() else None
agent.provider = provider_name or ""

if api_mode in {"chat_completions", "codex_responses", "anthropic_messages", "bedrock_converse", "codex_app_server"}:
    agent.api_mode = api_mode
elif agent.provider == "openai-codex":
    agent.api_mode = "codex_responses"
elif agent.provider in {"xai", "xai-oauth"}:
    agent.api_mode = "codex_responses"
elif agent.provider == "anthropic":
    agent.api_mode = "anthropic_messages"
elif agent.provider == "bedrock":
    agent.api_mode = "bedrock_converse"
else:
    agent.api_mode = "chat_completions"
```

这只是第一层。后面还有模型级修正：

```python
if (
    api_mode is None
    and agent.api_mode == "chat_completions"
    and not agent._is_azure_openai_url()
    and (
        agent._is_direct_openai_url()
        or agent._provider_model_requires_responses_api(agent.model, provider=agent.provider)
    )
):
    agent.api_mode = "codex_responses"
```

这里的重点是：`api_mode` 不是只由 provider 决定。有时同一个 provider 下，不同模型需要不同 API 路径。比如一些 GPT-5 系列模型需要 Responses API，但 Azure OpenAI 是例外，仍然走 chat completions。

所以 Hermes 的判断顺序是：

```text
用户显式 api_mode 优先
  -> provider 固有 api_mode
  -> base_url 形态推断
  -> model 特性修正
  -> provider-specific exception
```

源码中存在大量分支，因为现实 Provider 的接口和兼容行为并不统一。

## 功能 8：运行时 provider 解析把配置、凭证和 endpoint 合并

`api_mode` 进入 `AIAgent` 前，通常已经经过 runtime provider 解析。入口在 `hermes_cli/runtime_provider.py`：

```python
def resolve_runtime_provider(
    *,
    requested: Optional[str] = None,
    explicit_api_key: Optional[str] = None,
    explicit_base_url: Optional[str] = None,
    target_model: Optional[str] = None,
) -> Dict[str, Any]:
    """Resolve runtime provider credentials for agent execution."""
```

它返回的不是一个简单 provider 名，而是一组运行时参数：

```text
provider
api_mode
base_url
api_key
source
requested_provider
model override
headers / request overrides
```

里面有很多特殊分支。比如 Azure Foundry：

```python
if requested_provider == "azure-foundry":
    azure_runtime = _resolve_azure_foundry_runtime(...)
    return azure_runtime
```

比如 Anthropic：

```python
if provider == "anthropic":
    ...
    return {
        "provider": "anthropic",
        "api_mode": "anthropic_messages",
        "base_url": base_url,
        "api_key": api_key,
    }
```

比如 Nous：

```python
if provider == "nous":
    ...
    return {
        "provider": "nous",
        "api_mode": "chat_completions",
        ...
    }
```

这层存在的意义是把“用户想用什么”和“本轮实际怎么调用”分开。用户可能只写了：

```yaml
model:
  provider: openrouter
  default: anthropic/claude-sonnet-4.5
```

但运行时需要知道真实 base_url、key 来源、是否 OAuth、是否使用 credential pool、api_mode、header、模型名是否要 normalize。`resolve_runtime_provider` 就是把这些信息合成一个可执行配置。

## 功能 9：ChatCompletionsTransport 处理“OpenAI-compatible 但不完全兼容”的现实

`chat_completions` 是默认路径，覆盖大量 OpenAI-compatible provider。源码在 `agent/transports/chat_completions.py`。

它的 `convert_messages` 不是简单原样返回。它会剥掉严格 provider 不接受的内部字段：

```python
def convert_messages(self, messages, **kwargs) -> list[dict[str, Any]]:
    """Messages are already in OpenAI format — strip internal fields
    that strict chat-completions providers reject...
    """
```

它处理的字段包括：

```text
codex_reasoning_items
codex_message_items
tool_name
timestamp
_empty_recovery_synthetic 等内部标记
call_id / response_item_id
Gemini extra_content
```

这里有一个很好的例子：Gemini thinking models 需要 `thought_signature` 回放，但其他严格 OpenAI-compatible provider 会拒绝这个字段。所以 Hermes 只在目标模型还是 Gemini/Gemma 时保留：

```python
strip_extra_content = not _model_consumes_thought_signature(
    kwargs.get("model")
)
```

这就是 transport 的价值。Hermes 内部消息要保留丰富状态，供应商 API 又只接受自己的 schema。Transport 是两者之间的清洗和转换层。

## 功能 10：ProviderProfile 接管 per-provider quirk

在 chat completions 路径里，如果 provider 有 profile，Hermes 会走 profile 路径：

```python
from providers import get_provider_profile
_profile = get_provider_profile(agent.provider)

if _profile:
    return _ct.build_kwargs(
        model=agent.model,
        messages=api_messages,
        tools=tools_for_api,
        provider_profile=_profile,
        ...
    )
```

`ChatCompletionsTransport` 内部会调用：

```python
def _build_kwargs_from_profile(self, profile, model, sanitized, tools, params):
    sanitized = profile.prepare_messages(sanitized)
    ...
    if profile.fixed_temperature is OMIT_TEMPERATURE:
        pass
    elif profile.fixed_temperature is not None:
        api_kwargs["temperature"] = profile.fixed_temperature

    profile_max = profile.get_max_tokens(model)
    ...
    extra_body_from_profile, top_level_from_profile = (
        profile.build_api_kwargs_extras(...)
    )

    profile_body = profile.build_extra_body(...)
```

这说明 provider quirk 不再靠几十个 bool flag 传进来，而是从 profile hook 里取。

比如：

```text
某些 provider 不允许 temperature
某些 provider 要把 reasoning 放进 extra_body
某些 provider 要把 reasoning_effort 放顶层
某些 provider 有默认 max_tokens
某些 provider 要加 provider preferences
```

这些都属于“供应商特性”，不属于主循环逻辑。

没有 profile 的未知 provider 仍然能走 legacy flag path。这是兼容性设计：已知 provider 用 profile，未知 custom endpoint 尽量还能跑。

## 功能 11：Fallback 切换不是只换 model 字符串

主模型失败后，Hermes 可以切到 fallback。关键逻辑在 `agent/chat_completion_helpers.py`。

fallback 激活时会重新判断 api_mode：

```python
fb_api_mode = "chat_completions"
fb_base_url = str(fb_client.base_url)

if fb_provider == "openai-codex":
    fb_api_mode = "codex_responses"
elif fb_provider == "anthropic" or fb_base_url.rstrip("/").lower().endswith("/anthropic"):
    fb_api_mode = "anthropic_messages"
elif agent._is_direct_openai_url(fb_base_url):
    fb_api_mode = "codex_responses"
elif agent._provider_model_requires_responses_api(fb_model, provider=fb_provider):
    fb_api_mode = "codex_responses"
elif fb_provider == "bedrock":
    fb_api_mode = "bedrock_converse"
```

然后更新 agent 状态：

```python
agent.model = fb_model
agent.provider = fb_provider
agent.base_url = fb_base_url
agent.api_mode = fb_api_mode
if hasattr(agent, "_transport_cache"):
    agent._transport_cache.clear()
agent._fallback_activated = True
```

fallback 不能只修改 `model`，至少还要同步：

```text
model
provider
base_url
api_mode
client
credential pool
prompt caching policy
context compressor 的 context_length
system prompt 里的模型身份
```

否则就会出现很隐蔽的问题：模型切到 32K fallback，但压缩器仍按 200K 主模型判断；或者 provider 换了，transport cache 还拿旧 api_mode；或者 prompt 里还自称主模型。

Hermes 在 fallback 后会更新 context compressor：

```python
agent.context_compressor.update_model(
    model=agent.model,
    context_length=fb_context_length,
    base_url=agent.base_url,
    api_key=getattr(agent, "api_key", ""),
    provider=agent.provider,
    api_mode=agent.api_mode,
)
```

这就是生产级 agent 和 demo 的差别。demo 可以“失败了换个模型再试”。生产系统必须把所有依赖模型特性的子系统一起切过去。

## 功能 12：Credential Pool 负责同一 provider 内的凭证恢复

Fallback 是跨模型或跨 provider 的切换。Credential pool 解决的是另一类问题：同一个 provider 有多个可用凭证时，某个 key 失败了怎么办。

核心数据结构在 `agent/credential_pool.py`：

```python
@dataclass
class PooledCredential:
    provider: str
    id: str
    label: str
    auth_type: str
    priority: int
    source: str
    access_token: str
    refresh_token: Optional[str] = None
    last_status: Optional[str] = None
    last_error_code: Optional[int] = None
    base_url: Optional[str] = None
```

`CredentialPool` 支持四种策略：

```text
fill_first
round_robin
random
least_used
```

失败后会标记并轮换：

```python
def mark_exhausted_and_rotate(
    self,
    *,
    status_code: Optional[int],
    error_context: Optional[Dict[str, Any]] = None,
    api_key_hint: Optional[str] = None,
) -> Optional[PooledCredential]:
    ...
    self._mark_exhausted(entry, status_code, error_context)
    self._current_id = None
    next_entry = self._select_unlocked()
    return next_entry
```

不同错误的处理策略不同：

```text
401：先尝试刷新 OAuth token，失败再轮换
402：通常表示额度 / 账单耗尽，标记耗尽并轮换
429：第一次可以重试，重复限速后轮换
```

`AIAgent` 真正换凭证时调用 `_swap_credential`：

```python
def _swap_credential(self, entry) -> None:
    runtime_key = getattr(entry, "runtime_api_key", None) or getattr(entry, "access_token", "")
    runtime_base = getattr(entry, "runtime_base_url", None) or getattr(entry, "base_url", None) or self.base_url

    if self.api_mode == "anthropic_messages":
        self._anthropic_client.close()
        self._anthropic_api_key = runtime_key
        self._anthropic_base_url = runtime_base
        self._anthropic_client = build_anthropic_client(...)
        self.api_key = runtime_key
        self.base_url = runtime_base
        return

    self.api_key = runtime_key
    self.base_url = runtime_base.rstrip()
    self._client_kwargs["api_key"] = self.api_key
    self._client_kwargs["base_url"] = self.base_url
    self._replace_primary_openai_client(reason="credential_rotation")
```

`api_mode` 也参与凭证切换。Anthropic client 和 OpenAI-compatible client 的重建方式不同，不能只替换一个凭证字符串。

## 功能 13：Fallback 和 Credential Pool 的边界

这两个机制容易混。

| 机制 | 处理什么失败 | 改变什么 |
| --- | --- | --- |
| Credential Pool | 同一 provider 下某个凭证 401/402/429 | 换 key / token / base_url，尽量不换 provider |
| Fallback Model | 主 provider 或模型整体不可用 | 换 model / provider / api_mode / client / context length |

一个例子：

```text
OpenRouter 某个 key 429，但池里还有另一个 key
```

优先用 credential pool 轮换，不必换模型。

另一个例子：

```text
主 provider 整体失败，或者 fallback chain 明确配置了备用 provider
```

这时才进入 fallback，切换 provider/model/api_mode。

Hermes 在 fallback 时还会处理 credential pool 污染问题：

```python
if _existing_pool is not None:
    _pool_provider = (getattr(_existing_pool, "provider", "") or "").strip().lower()
    if _pool_provider and _pool_provider != fb_provider:
        agent._credential_pool = None
```

原因很直接：如果从 OpenRouter fallback 到 Anthropic，还带着 OpenRouter 的 pool，后续 401/429 恢复可能会把错误的 key/base_url 写回去。这类 bug 很难排查，所以切 provider 时必须重新绑定或清空 pool。

## 功能 14：Codex App Server 是运行时路径，不是普通 transport

除了四个标准 transport，Hermes 还有 `codex_app_server`。

它不是 `ProviderTransport` 子类，而是一条独立运行时路径。文档里把它列成可选 opt-in，因为它会把回合交给 `codex app-server` 子进程。

运行时在 `agent/codex_runtime.py`，相关文件有：

```text
agent/transports/codex_app_server.py
agent/transports/codex_app_server_session.py
agent/transports/codex_event_projector.py
agent/transports/hermes_tools_mcp_server.py
```

普通 transport 的模式是：

```text
Hermes 构造请求
  -> 调 provider API
  -> normalize response
  -> Hermes 执行 tool calls
```

Codex app-server 路径更像：

```text
Hermes 启动 / 复用 codex app-server 子进程
  -> 把 Hermes 工具暴露成 MCP
  -> Codex 子进程驱动一轮
  -> Hermes 把 Codex 事件投影回自己的 messages
```

所以它和 `chat_completions`、`anthropic_messages` 不是同一种抽象。它是运行时替换，而不是单纯请求格式转换。

Hermes 的 Provider 层由两类扩展组成：

```text
Transport 扩展：同一个 agent loop，换 API 数据路径
Runtime 扩展：把整轮执行交给另一个 agent runtime，再投影回 Hermes
```

## 功能 15：辅助模型也复用 provider / transport 思路

Hermes 里不只有主对话调用模型。压缩、标题、背景 review、memory flush、vision 等辅助任务也会用模型。

相关设计在 `agent/auxiliary_client.py`，文档称它为 Auxiliary Client。它的原则是：辅助任务不要各自实现一套 provider fallback。

辅助客户端会：

```text
解析任务级 provider / model
  -> 如果 auto，优先考虑主模型或 provider chain
  -> 按 fallback_chain 尝试
  -> 必要时回到 main agent model
  -> 复用 transport 做请求格式和响应处理
```

这说明 Provider / Transport 不是只服务主循环。它是整个 Hermes “调用 LLM” 的基础设施。否则每个辅助功能都自己写 provider 兼容，很快就会出现同一个 provider 在不同路径表现不一致的问题。

## 和 Codex、Claude Code 的对比

Codex 和 Claude Code 的供应商边界更集中：它们主要服务自家模型和自家运行时，再通过工具、MCP、插件扩展能力。用户不需要经常考虑 `api_mode`、`base_url`、provider profile、credential pool 这些细节。

Hermes 不一样。它更像一个多供应商 agent 产品，要同时接 OpenAI-compatible、Anthropic Messages、OpenAI Responses、Bedrock、OAuth 订阅、聚合器、本地模型和自定义端点。

| 问题 | Hermes | Codex / Claude Code 常见形态 |
| --- | --- | --- |
| 供应商数量 | 多 provider / custom endpoint / aggregator | 以自家模型和官方路径为主 |
| API 协议差异 | `api_mode` + `ProviderTransport` | 用户通常不直接感知 |
| provider 元信息 | `ProviderProfile` 插件化 | 多由产品内部配置 |
| 失败恢复 | credential pool + fallback chain + client rebuild | 多由平台或宿主运行时处理 |
| 自定义接入 | `plugins/model-providers`、custom providers | 通过 API 配置、MCP、外部工具等方式扩展 |

Hermes 的复杂性来自它想当“多模型代理运行时”。这让它更灵活，也让工程边界更重要。

## 常见问题

### 1. provider、model、base_url、api_mode 到底怎么区分？

`provider` 是供应商身份，比如 `openrouter`。  
`model` 是模型名，比如 `anthropic/claude-sonnet-4.5`。  
`base_url` 是实际请求 endpoint。  
`api_mode` 是请求协议路径，比如 `chat_completions` 或 `anthropic_messages`。

同一个 provider 可以有多个 model。不同 provider 可以共享同一个 api_mode。同一个 model 在不同 provider 上也可能需要不同处理。

### 2. 为什么 OpenAI-compatible provider 还要 ProviderProfile？

因为“兼容”通常只表示大体接口相似，不表示所有字段都一样。

有的 provider 不支持 `temperature`，有的要求 `max_tokens`，有的把 reasoning 放在 `extra_body`，有的拒绝内部字段，有的 models endpoint 不标准。`ProviderProfile` 把这些差异放到供应商自己的声明和 hook 里，避免污染主循环。

### 3. Transport 和旧 adapter 是什么关系？

Transport 是新的统一边界。旧 adapter 里的部分转换逻辑仍可能被 transport 内部委托使用，但主调用路径应该面向 transport：`build_kwargs`、`normalize_response`、`extract_cache_stats`。

可以这样理解：adapter 是历史实现和底层辅助函数，transport 是现在主循环看的抽象边界。

### 4. 为什么 fallback 后要更新 context compressor？

因为不同模型上下文窗口不同。主模型可能是 200K，fallback 可能只有 32K。如果压缩器还按主模型判断，fallback 请求可能直接超上下文。

所以 fallback 后要更新 model、provider、base_url、api_key、api_mode 和 context_length。这个动作不是优化，是正确性要求。

### 5. Credential pool 和 fallback 谁先发生？

通常先尝试同 provider 内的恢复：刷新 token、换同 provider 的另一个凭证。这样用户仍然留在同一个模型/供应商路径上。

如果 provider 或模型整体不可用，或者 fallback chain 被触发，才切到 fallback model/provider。

### 6. 为什么 Codex app-server 不做成普通 transport？

因为它不是“请求格式不同”，而是“整轮运行时不同”。

普通 transport 只是把 Hermes 的 messages/tools 转成供应商 API，然后 Hermes 自己执行 tool calls。Codex app-server 路径会启动子进程，把 Hermes 工具暴露给它，再把子进程事件投影回 Hermes。它替换的是执行方式，不只是数据格式。

## 实现要点

Hermes Provider 层的职责分工是：`ProviderProfile` 管理供应商身份和兼容特征，`ProviderTransport` 负责 API 协议转换。

`api_mode` 是连接两者的关键字段。它决定走 `chat_completions`、`codex_responses`、`anthropic_messages`、`bedrock_converse`，还是独立的 `codex_app_server` 运行时。

主循环不应该直接理解每个 provider 的原始响应。Transport 把响应统一成 `NormalizedResponse`，主循环只处理 `content`、`tool_calls`、`finish_reason` 等通用字段。

fallback 不是只换模型名。它必须同步 provider、base_url、api_mode、client、transport cache、credential pool、prompt cache、context compressor 和 system prompt identity。

credential pool 是同 provider 内的恢复机制，fallback 是跨模型或跨 provider 的恢复机制。两者边界清楚，系统才不会在失败恢复时把错误 key、错误 endpoint 或错误 context window 混在一起。
