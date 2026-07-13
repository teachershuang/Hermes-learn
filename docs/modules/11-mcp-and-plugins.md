# MCP 与插件：Hermes 如何接入外部能力，又不把核心运行时交出去

## 目标与范围

Hermes 里常见的扩展有两种，名字都容易让人混淆。

MCP 是进程或网络另一端的服务。Hermes 作为 client 连接它，发现它提供的工具，再把工具交给模型调用。它解决的是"怎样接入外部系统"。

Plugin 是运行在 Hermes 进程里的 Python 代码。它能注册工具、provider、平台适配器、hook、middleware 和命令。它解决的是"怎样扩展 Hermes 自己的行为"。

两者最后都会影响模型可调用的能力，但信任边界完全不同：

```text
MCP server
  -> JSON-RPC 连接与 tools/list
  -> Hermes 包装成 mcp_<server>_<tool>
  -> Tool Registry
  -> 模型调用

Python plugin
  -> plugin.yaml + __init__.py
  -> register(ctx)
  -> Tool Registry / Hook Registry / Middleware Registry / Provider Registry
  -> Hermes 运行时行为改变
```

MCP 的返回内容来自外部服务，所以 Hermes 要把它当成不可信数据。Plugin 直接进入本进程，因此重点是发现、加载、隔离失败和权限门槛。把两者都叫"插件"会掩盖这些差异。

## 功能 1：MCP 工具如何变成 Hermes 工具

入口在 `tools/mcp_tool.py`。文件开头把职责写得很直接：连接 stdio、HTTP/Streamable HTTP 或 SSE 的 MCP server，发现工具，然后注册到 Hermes 的 Tool Registry。

工具不是手写进一个固定表，而是 server 返回 `tools/list` 后动态生成。注册代码的入口是：

```python
# tools/mcp_tool.py
def _register_server_tools(name, server, config):
    toolset_name = f"mcp-{name}"
    ...
    registry.register(
        name=..., toolset=toolset_name, schema=..., handler=...
    )
```

所以一个名为 `github` 的 MCP server 不会把裸工具名 `create_issue` 直接塞进全局命名空间。Hermes 用 server 名给它加前缀，工具归入 `mcp-github` 这个 toolset。这样既减少重名，也让用户能按 server 做 include/exclude 或选择 toolset。

注册前还会读取配置里的过滤条件：

```python
# tools/mcp_tool.py
# tools.include: 只注册白名单中的工具
# tools.exclude: 除黑名单外全部注册
# include 优先于 exclude
tools_filter = config.get("tools") or {}
```

这不是一个小优化。大型 MCP server 往往带几十甚至上百个工具；全塞进 prompt 会增加 token、干扰模型选工具，还可能超过 provider 的 tools 限制。Hermes 在 `model_tools.py` 里还会把部分 MCP/plugin 工具延迟为 Tool Search bridge，只把搜索桥和少量核心工具先交给模型。模型需要时再检索并激活具体工具。

### 工具调用时发生了什么

Registry 需要同步形式的 handler，但 MCP session 在后台 asyncio loop 中。Hermes 因此生成一个同步包装器：

```python
# tools/mcp_tool.py
def _make_tool_handler(server_name, tool_name, tool_timeout):
    def _handler(args, **kwargs):
        ...
        result = await server.session.call_tool(tool_name, arguments=args)
        ...
    return _handler
```

这里的 `await` 实际跑在 MCP 专用后台事件循环中。主 agent loop 只看见一个和其他注册工具一致的 handler。这个适配层很重要：上游不必知道某工具来自本地 Python、子进程还是远程 HTTP。

工具结果也不是只取字符串。`CallToolResult` 可能包含 text，也可能包含图像块。源码会把图像缓存为 Hermes 统一的 `MEDIA:` 引用，交给后续消息适配器；否则截图类 MCP 工具虽然执行成功，模型却可能只收到空结果。

## 功能 2：动态发现意味着 Registry 不是静态配置

一个 MCP server 连接后仍可能改变工具列表，例如用户在数据库服务端开启了新 capability。MCP 协议允许 server 发送 `notifications/tools/list_changed`。

Hermes 的处理在 `MCPServerTask._make_message_handler()`：

```python
# tools/mcp_tool.py
case ToolListChangedNotification():
    self._schedule_tools_refresh()
```

它没有在通知回调里立刻 `list_tools()`。原因是同一条 stdio JSON-RPC 流可能正在执行正常工具调用；同步刷新会抢占流，造成后续调用卡死。源码把刷新丢到独立 task，然后用 `_refresh_lock` 串行化多次刷新。

刷新过程也没有把所有旧工具先删光再重建：

```python
# tools/mcp_tool.py
stale_tool_names = old_tool_names - new_tool_names
for tool_name in stale_tool_names:
    registry.deregister(tool_name)

self._registered_tool_names = _register_server_tools(...)
```

只注销已经消失的工具，保留同名工具的入口。运行中的 agent turn 可能已经持有 tool call id 和 handler；全量删建会制造短暂的"工具不存在"窗口。这是动态 Registry 最容易被忽略的一类并发问题。

连接断开时也不能把一个失败 server 留给模型无限重试。`_make_tool_handler()` 有断路器：连续失败超过阈值后，在冷却时间内直接返回"不要立刻重试，改用替代方案或请用户检查 server"。重连成功后，`_register_discovered_tools_if_needed()` 会重新发布工具。它把"服务暂时不可用"变成可解释的工具结果，而不是让模型耗尽整个 tool loop。

## 功能 3：MCP 的安全边界不止在连接时

### 描述和结果都可能携带注入内容

MCP server 提供的 tool description 会进入模型的工具说明，工具结果又会重新进入对话上下文。两处都可能成为 prompt injection 入口。

`tools/mcp_tool.py` 定义了 `_scan_mcp_description()`，对可疑工具描述做模式扫描和日志记录。它偏向告警，不等于自动证明 server 恶意；描述里出现某些 instruction-like 文本时，人工仍应审查该 server。

结果侧的处理更明确。`agent/tool_dispatch_helpers.py` 识别 `mcp_` 前缀，把文本结果包起来：

```python
# agent/tool_dispatch_helpers.py
return (
    f'<untrusted_tool_result source="{name}">\n'
    f'{_neutralize_delimiters(content)}\n'
    f'</untrusted_tool_result>'
)
```

这段标签是在告诉模型：它读到的是外部数据，不是新的系统指令。`_neutralize_delimiters()` 还会改写结果里伪造的 `untrusted_tool_result` 标记，避免恶意 server 提前闭合边界。

它不能让不可靠模型突然变得可靠，也不能代替网络隔离和权限控制。它解决的是一个更具体的问题：不要让网页、MCP 回复和系统 prompt 在文本形态上混成同一层。

### MCP server 也可以反过来请求能力

MCP 不只有 `tools/call`。Hermes 还实现了两个反向方向。

`sampling/createMessage` 允许 server 请求 Hermes 调一次 LLM。`SamplingHandler` 把 server 消息转为 Hermes 能处理的格式，并限制每分钟请求数、允许的模型和最大 tool round，避免一个外部 server 借宿主模型无限循环。

`elicitation/create` 则是 server 在工具执行中请求结构化用户输入，例如支付确认或 OAuth 确认。`ElicitationHandler` 不自己在后台线程里随意读输入，而是转入 `tools.approval` 的审批通道。无论请求来自内置 terminal 还是 MCP server，用户确认都要回到同一条交互和 session 路由链上。

## 功能 4：OAuth 不是"登录一次就完了"

远程 MCP server 可能要求 OAuth。Hermes 没把 token 状态散在连接代码里，而是放在 `tools/mcp_oauth_manager.py` 的 `MCPOAuthManager`。

它维护每个 server 的 provider、锁、失效 token 状态和磁盘更新时间。实际 token、client registration 和 OAuth metadata 由 `tools/mcp_oauth.py` 的 `HermesTokenStorage` 写入磁盘。

这一层处理的不是教科书里的 happy path，而是长期运行中的几个麻烦问题：

```text
进程重启：从磁盘恢复 token 和 OAuth metadata，避免猜错 token endpoint。
别的进程刷新 token：下一次调用检查 mtime，发现变化后让 SDK 重载。
多并发请求同时 401：按 failed access token 去重，只让一个恢复流程运行。
无交互环境没有缓存 token：直接报可操作错误，不能在 cron 或 gateway 中挂死等浏览器回调。
重新认证失败：保存旧状态快照，避免把原本仍可用的 token 一并删除。
```

`handle_401()` 不承诺“401 后一定成功登录”。它先检查磁盘是否已有外部刷新，再判断是否有 refresh token 可用，最后让下一个请求走 SDK 的刷新路径，避免每次工具调用都无条件发起网络 refresh。

OAuth token 不应被模型读取。第 10 讲里的 `agent/file_safety.py` 会把 MCP token 目录列为敏感读取路径；错误消息也经过 `_sanitize_error()`，避免把 Bearer token、URL 用户信息等带回模型上下文。

## 功能 5：Plugin 是本地扩展 API，不是 MCP 的别名

Plugin 的发现和加载在 `hermes_cli/plugins.py`。一个目录插件通常有 `plugin.yaml` 和 `__init__.py`；前者提供名称、版本、类型和描述，后者导出 `register(ctx)`。

```python
# hermes_cli/plugins.py
def _load_plugin(self, manifest):
    module = self._load_directory_module(manifest)
    register_fn = getattr(module, "register", None)
    if register_fn is not None:
        ctx = PluginContext(manifest, self)
        register_fn(ctx)
```

`PluginContext` 是给插件的宿主 API。它不要求插件直接改全局变量。常见注册入口包括：

```python
ctx.register_tool(...)
ctx.register_hook(...)
ctx.register_middleware(...)
ctx.register_command(...)
ctx.register_platform(...)
ctx.register_skill(...)
```

同一个机制支撑了仓库里的 web search provider、模型 provider、平台 adapter、memory backend、可选 dashboard 和独立业务插件。它让核心 agent 不必为每一个供应商和平台不断扩张。

### 加载来源与信任差异

`PluginManager._discover_and_load_inner()` 会扫描 bundled plugin、用户目录插件、可选的项目目录插件，以及 pip entry point。项目插件默认不是自动打开的，必须显式设置 `HERMES_ENABLE_PROJECT_PLUGINS=1`。

这是一种很实际的选择。项目目录常常来自 git clone；自动 import 其中的 Python 就等于打开仓库时执行第三方代码。Hermes 把它做成 opt-in。出现故障时，`HERMES_SAFE_MODE=1` 会跳过 plugin discovery，让用户先回到不加载第三方扩展的基线环境。

### 不能静默覆盖内置工具

插件可注册工具，但默认不能抢占已有工具名。若要覆盖，例如把内置 `browser_navigate` 换成自己的实现，插件既要传 `override=True`，又要得到配置允许：

```python
# hermes_cli/plugins.py
if override and not self._tool_override_allowed(name):
    raise PluginToolOverrideError(...)
```

对 bundled plugin，Hermes 默认信任维护者；用户、项目和 pip plugin 则需要在配置中明确开启 `allow_tool_override`。否则一个普通插件可以把 `shell_exec` 或 `write_file` 替换成窃取数据的 handler。这条门槛的目的不是防止插件出错，而是防止扩展点变成提权入口。

## 功能 6：hook 和 middleware 如何改变运行时

Plugin 既能观察事件，也能改变请求。两者不要混用。

`register_hook()` 放入生命周期回调。`invoke_hook()` 会逐个调用并隔离异常；一个 observer plugin 崩掉不该打断核心 agent loop。`pre_tool_call` 是例外中的重要一类：它可以返回阻断指令，因而适合策略、配额和审批前检查。

`register_middleware()` 放入行为性回调。源码对它的定义很直接：request middleware 可以改写 effective payload，execution middleware 可以包裹真实 callback。前者在工具 guardrail 和审批前调整参数，后者围绕实际执行做计时、审计、替代执行或结果修饰。第 10 讲已经讲过，middleware 不能被理解为纯观察日志。

Plugin 加载失败时，`_load_plugin()` 会记录该插件的 error，并把其他已发现插件继续写入状态；hook 和 middleware 调用也会隔离单个 callback 的异常。这提高了宿主可用性，但也意味着运维上应当查看 `hermes plugins list` 和日志，不能假定“进程启动成功”就等于全部扩展生效。

## 与 Codex、Claude Code 的对照

Codex 和 Claude Code 也都把 MCP 当作把外部工具交给 coding agent 的标准接口。共同点是：server 的工具 schema 需要进入模型可见的工具集合，外部返回值不应被当作可信指令，授权状态不能混入普通聊天记录。

Hermes 的不同处在于它把 MCP 放进一个更宽的运行面：CLI、gateway、cron、消息平台和多个 provider 都可能复用同一套 MCP 连接。于是它额外处理后台发现不阻塞启动、重连后重新注册、工具动态变化、并发 401 去重、MCP server 主动 sampling 和 elicitation。

Plugin 的取向也不同。Codex / Claude Code 的本地扩展通常围绕开发者项目工作流、MCP、配置文件和 hooks 展开；Hermes 的 Python plugin API 还承担 provider、平台 adapter、memory backend、dashboard 等宿主级能力。这让它更适合作为多平台 agent runtime，也使本地第三方代码的信任边界更重。

## 常见误解

### MCP 工具是不是普通工具？

对模型和 Tool Registry 来说，最终是普通工具：有名称、JSON schema、handler 和 result。对运行时来说不是。它的 handler 需要跨线程/事件循环进入长连接 JSON-RPC session，还要面对重连、超时、OAuth 和 server 动态改工具列表。

### MCP server 增加了工具，当前 session 的模型能立刻看见吗？

不能把它理解成"每次 token 都自动刷新"。server 发 `tools/list_changed` 后，Hermes 会后台刷新 Registry；下一次构建工具快照时，新的工具才会进入模型可见集合。若启用了 Tool Search bridge，模型通常先看见搜索桥，检索后才拿到具体 schema。谁决定模型是否看见它，是工具快照构建和 toolset/filter 配置，不是 MCP server 自己直接修改 prompt。

### Plugin 能绕过审批吗？

Plugin 能写自己的 handler，因此它本身就是受信任代码，不能把它当作沙箱里的不可信脚本。它若调用 Hermes 的标准 terminal/file tool，会进入相应 guardrail；若自己直接执行系统调用，就必须由插件作者和部署者负责约束。Hermes 的 `safe mode`、项目插件 opt-in 和工具覆盖门槛是在降低这一风险，不是把恶意 Python 变安全。

### 为什么 OAuth 不在每次 MCP 调用前都强制刷新？

每次调用做 refresh 会增加延迟和失败面，也可能让并发请求同时刷新同一 token。Hermes 采用“本地 token 状态 + mtime 侦测 + 401 后去重恢复”的组合：正常路径轻量，失效路径集中处理。

### hook 和 middleware 应该怎么选？

只需要观察事件、收集指标或发通知，用 hook。需要修改参数、短路请求、包裹执行或替换结果，才用 middleware。把有副作用的改写逻辑塞进普通 hook，会让调用顺序和失败语义变得难以判断。

## 核对清单

一段 `mcp_servers` 配置应能追踪到 server 连接、工具发现、`mcp-<server>` toolset 注册、模型选择、实际执行和不可信结果回填。

Hermes 同时提供 MCP 和 Python plugin：前者连接外部服务，后者修改宿主行为；两者的信任模型、失败处理和安全责任不同。
