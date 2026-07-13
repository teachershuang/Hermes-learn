# 浏览器、网页、终端与代码执行：模型的四种“动手方式”

## 目标与范围

用户说“查一下网页，再改一下项目里的代码”，模型可能有四种动作：

```text
web_search   找候选链接和摘要
web_extract  取一个或多个页面的正文
browser_*    打开真实页面，读取界面，点击、输入、截图
terminal     在选定的执行环境里运行 shell 命令
execute_code 写一段 Python，把多步低层工具调用放进脚本
```

它们不是能力越来越强的一条直线。

`web_search` 和 `web_extract` 面向信息获取，后端是搜索或内容提取服务。`browser_*` 面向交互网站和已登录页面。`terminal` 是持久工作环境中的 shell。`execute_code` 适合把重复、多步骤、对中间结果不感兴趣的工作压缩到一个脚本里。

选择错工具会带来真实成本。用 browser 抓一篇静态文档，慢且容易触发反爬；用 `web_extract` 完成登录后点击，根本做不到；把两次 grep 写成 `execute_code`，反而增加审批、调试和失败面。

## 功能 1：Web 工具是“搜索”和“正文提取”，不是浏览器替身

入口在 `tools/web_tools.py`。当前源码公开的核心工具只有：

```python
def web_search_tool(query: str, limit: int = 5) -> str:
    ...

async def web_extract_tool(urls: List[str], format=None, char_limit=None) -> str:
    ...
```

`web_search_tool()` 只返回标题、URL、描述和排序位置。源码在函数注释里明确要求：需要正文时再调用 `web_extract_tool()`。这让模型先做“找哪些页面值得读”，再做“读哪几页”，而不是一次把搜索页和网页正文都塞进上下文。

### provider 按能力拆分，不按品牌绑定

Web provider 的抽象类是 `agent/web_search_provider.py` 里的 `WebSearchProvider`。它要求 provider 至少支持 search 或 extract 之一，并把能力声明成：

```python
def supports_search(self) -> bool: ...
def supports_extract(self) -> bool: ...
```

`agent/web_search_registry.py` 按 capability 选 provider。解析顺序值得逐条理解：

1. 用户显式指定 `web.search_backend` 或 `web.extract_backend` 时，优先返回这个 provider，即使它当前不可用。
2. 只剩一个可用 provider 时，直接使用它。
3. 否则按既有偏好顺序，在可用 provider 中选择。

第一条看起来反直觉，源码却写得很清楚：显式配置但缺 API key 时，应该返回“这个 provider 没配置好”的准确错误，而不是悄悄换到另一个服务。静默降级会让用户难以判断流量、费用和数据实际上去了哪里。

```python
# agent/web_search_registry.py
if configured:
    provider = snapshot.get(configured)
    if provider is not None and _capable(provider):
        return provider
```

这个 registry 与第 11 讲的 Plugin 系统连接在一起。`web_search_tool()` 先确保 web provider plugins 已加载，再从 registry 找当前 provider；工具包装器不需要知道 Firecrawl、Tavily、Exa 或自定义 provider 的厂商差异。

### `web_extract` 的输出为何不是“全文”

网页正文可能非常大。`web_extract_tool()` 不再用 LLM 先总结网页，而是采用 truncate-and-store：保留头尾片段，正文写入缓存，并在返回内容的 footer 中给出后续读取路径。

```python
# tools/web_tools.py
model_text, truncated = _truncate_with_footer(clean, url, effective_char_limit)
result["content"] = model_text
```

这是一种上下文管理选择。模型先得到文档开头、结尾和“中间被省略”的明确信号；只有确实需要细节时才读缓存文件。它比自动摘要更可审计，也避免摘要模型把网页中的关键条件悄悄删掉。

URL 在送给第三方 extract provider 前还要经过几道检查：URL 中的 token/API key、看起来像凭证的 query parameter、私有网络地址。私有地址会以单个结果的错误形式合并回响应，而不是让同批公共 URL 也失效。

```python
# tools/web_tools.py
if not await async_is_safe_url(url):
    ssrf_blocked.append({
        "url": url,
        "error": "Blocked: URL targets a private or internal network address",
    })
```

这也解释了一个实际选择：带有临时签名、身份令牌的 URL 不应随手交给云端网页提取服务。源码会拒绝这类 URL，并提示在确实获授权时转用本地 browser/CDP 会话。

## 功能 2：浏览器工具维护的是“页面会话”，不是 URL 字符串

浏览器入口是 `tools/browser_tool.py`。常用动作包括 `browser_navigate`、`browser_snapshot`、`browser_click`、`browser_type` 和 `browser_scroll`。

`browser_navigate()` 接受 `task_id`，并以它组织会话。一个任务连续调用 navigate、snapshot、click 时，cookie、页面状态和元素引用留在同一会话里；不同子 agent 的 `task_id` 不应抢同一个浏览器。

```python
# tools/browser_tool.py
session_info = _get_session_info(nav_session_key)
result = _run_browser_command(
    nav_session_key, "open", [url], timeout=...
)
```

这里的 `nav_session_key` 不只是 task id 的原样副本。Hermes 还会处理 hybrid routing：公共网页可走云端 browser provider，私有 URL 在配置允许时改由本地 Chromium sidecar 访问。这样云 provider 不会收到内部地址，但浏览器仍能在开发环境完成本地服务的调试。

### snapshot 为什么是可访问性树，而不是截图

模型点页面需要可定位的元素。截图对人直观，对模型却很难稳定地引用某个按钮。`browser_snapshot()` 优先返回文本化的 accessibility tree，并带可供 `browser_click(ref=...)` 使用的 refs：

```python
# tools/browser_tool.py
result = _run_browser_command(effective_task_id, "snapshot", args)
snapshot_text = data.get("snapshot", "")
refs = data.get("refs", {})
```

因此典型链路是：

```text
navigate
  -> 返回精简 snapshot 和 refs
  -> 模型挑选一个 ref
  -> click/type
  -> 再 snapshot 检查页面状态
```

这比“让模型看图猜坐标”稳定，也减少了误点。截图仍有用途，例如视觉布局、图表和 canvas，但不是网页表单操作的首选状态表示。

大型 snapshot 同样不能直接无限回填。源码会按阈值截断；调用者提供 `user_task` 时，`_extract_relevant_content()` 会尝试挑出与任务相关的片段。浏览器工具在这一点上与 Web 工具一致：工具输出必须服务于下一次推理，而不能把整个上下文窗口占满。

### 浏览器的 SSRF 要检查两次

`browser_navigate()` 先检查输入 URL，再检查最终 URL。第二次检查针对重定向：

```python
# tools/browser_tool.py
if final_url and final_url != url and not _is_safe_url(final_url):
    _run_browser_command(nav_session_key, "open", ["about:blank"], timeout=10)
    return json.dumps({"success": False, "error": "Blocked: redirect landed on a private/internal address"})
```

只在打开前检查不够。攻击者可以给出公共 URL，再 302 到 `127.0.0.1`、云 metadata endpoint 或内网控制台。Hermes 在发现危险最终地址后导航到 `about:blank`，避免下一次 snapshot 把内部页面内容读回模型。

还有一个更细的分支：如果 `browser_console` 中的 JavaScript 修改了 `location.href`，`browser_snapshot()` 会重新读取当前页面 URL，再判断是否允许返回页面内容。也就是说，安全检查不能只相信“入口函数已经验证过”。

浏览器后端本身通过 `agent/browser_provider.py` 抽象。provider 必须实现 `is_available()`、`create_session()`、`close_session()` 和 `emergency_cleanup()`。`is_available()` 被要求只做本地廉价检查，例如环境变量、令牌文件或可选依赖；它不能每次工具注册都发网络请求。

## 功能 3：terminal 的核心对象是执行环境

终端实现在 `tools/terminal_tool.py`。它支持 local、Docker、Singularity、SSH、Modal、Daytona 等 backend。模型看到的是一个 `terminal` 工具，但它真正操作的是某个 task 对应的 environment。

环境的创建和复用遵循这条链路：

```text
读取 TERMINAL_ENV 与 backend 配置
  -> 根据 task_id 找已有 environment
  -> 没有时取得该 task 的创建锁
  -> 再次检查，避免并发重复创建
  -> 创建 backend 实例并缓存
  -> 在该环境执行命令
```

源码使用“先查、加锁、再查”的模式：

```python
# tools/terminal_tool.py
with _env_lock:
    if effective_task_id in _active_environments:
        env = _active_environments[effective_task_id]
        needs_creation = False
    else:
        needs_creation = True
```

随后用 task 专属创建锁做 double check。没有这一步，两个并发 tool call 都可能判断“环境不存在”，各自启动 Docker 或 Modal sandbox，结果既浪费资源，也让同一会话的工作目录和状态分叉。

### 终端持久性是 backend 属性，不是工具保证

`terminal` 的工具描述会说工作目录和导出的环境变量可在调用间保留，但这成立的前提是当前 backend 的 environment 被复用。local persistent shell、SSH persistent shell 和 persistent container 的实现不同；短生命周期或配置为非持久的 backend 则会丢失进程状态。

所以工程上不应让模型把“上一次 `cd` 了某目录”当成唯一事实。关键工作目录、文件路径和命令产物应明确写入后续命令或由 environment 状态管理器复核。

终端在创建环境后才进入命令审批。第 10 讲的 `check_all_command_guards()` 在执行前被调用：

```python
# tools/terminal_tool.py
approval = _check_all_guards(
    command, env_type,
    has_host_access=_docker_has_host_access(config),
)
```

被阻断时，工具返回 `status: "blocked"`；gateway 的 ask 模式则可以返回 `pending_approval`。这两种状态不能混为一谈：前者是当前调用不能执行，后者是执行权还在等待用户决定。

另外，前台命令超过上限会被拒绝，并要求模型转为后台受管任务；长时间 server/watch 命令也会收到类似引导。原因不是“不能运行长任务”，而是前台 tool call 没有适合管理它的生命周期、输出和取消语义。

## 功能 4：`execute_code` 是一个小型编排器

`tools/code_execution_tool.py` 的 `execute_code()` 运行 Python 脚本，但它的目标不是取代 terminal。它适合这类工作：脚本内部要多次调用搜索、读取、过滤、写入等工具，而模型只需要最后的汇总结果。

```text
普通工具循环：模型 -> 工具 -> 中间结果回填 -> 模型 -> 工具

execute_code：模型写 Python -> 子进程执行和内部 RPC -> 最终 stdout 回填
```

中间结果不进入主对话，能减少 token 和模型往返次数。代价是脚本中的错误更隐蔽，且脚本本身是任意 Python，所以必须有额外限制。

### 沙箱脚本只能得到受限的工具门面

Hermes 会生成 `hermes_tools.py`，让脚本通过 RPC 调回宿主进程。可调用工具不是 Registry 的全部，而是 `SANDBOX_ALLOWED_TOOLS` 与当前 session enabled tools 的交集：

```python
# tools/code_execution_tool.py
session_tools = set(enabled_tools) if enabled_tools else set()
sandbox_tools = frozenset(SANDBOX_ALLOWED_TOOLS & session_tools)
```

这意味着 agent 即使知道某个高权限工具名，也不能仅靠在 Python 中 import 一个 stub 来调用它。脚本能调用什么由父进程生成的门面决定。

本地 backend 使用 Unix domain socket 做 RPC；远程 backend 使用文件式 RPC。后者的脚本在 Docker、SSH、Modal 等环境里，写请求文件，宿主轮询后执行真实工具并写响应文件。两种 transport 的目的相同：让脚本可以复用 Hermes 工具，同时不把宿主的 Registry、认证对象和进程内状态直接暴露给子进程。

### 为什么它仍需要单独审批

即使脚本在子进程里，Python 仍可以 `subprocess.run()`、`os.remove()` 或调用原生库。它绕开了 `terminal()` 对 shell 字符串的危险命令扫描。

```python
# tools/code_execution_tool.py
_guard = check_execute_code_guard(
    code, env_type,
    has_host_access=_docker_has_host_access(_env_config),
)
if not _guard.get("approved", False):
    return json.dumps({"status": "error", "error": ...})
```

Docker 有 host bind mount 时也不能被当成完全隔离。源码明确把这种情况传给 guard，因为容器里的脚本仍可能改到宿主工作区。这正是“运行在容器里”与“没有宿主影响”之间常被忽略的差别。

### 输出处理决定模型能否正确恢复

`execute_code` 对 stdout 做 head+tail 截断，剥离 ANSI 转义符，并在回填前做 secret redaction：

```python
# tools/code_execution_tool.py
stdout_text = strip_ansi(stdout_text)
stdout_text = redact_sensitive_text(stdout_text, code_file=True)
```

超时时，返回值里不只放 error，还把超时消息写入 `output`。源码注释解释了原因：只返回空 output 时，模型容易把结果误判为“没有发生任何事”，再给用户一段空泛成功回复。

远程 RPC 中还禁止 sandbox 脚本给 terminal 传 `background`、`pty`、`notify_on_complete`、`watch_patterns` 等参数。这些参数会把短命脚本变成长期会话管理器，超出 `execute_code` 的生命周期模型。

## 工具选择表

| 任务 | 首选 | 原因 | 不宜优先选 |
| --- | --- | --- | --- |
| 找公开资料、比较多个来源 | `web_search` | 结果小，先筛选链接 | browser |
| 读取静态文章或文档正文 | `web_extract` | 直接提取，自动控制正文体积 | browser |
| 登录、表单、点击、SPA 状态 | `browser_*` | 有 session、元素 ref 和页面状态 | web_extract |
| 构建、测试、git、项目命令 | terminal | 直接使用工作环境 | execute_code |
| 脚本化多步检索/处理，模型只要最终结果 | `execute_code` | 中间结果不膨胀上下文 | 多轮普通 tool call |

## 与 Codex、Claude Code 的对照

Codex 的本地开发工作流更接近 Hermes 的 terminal 一侧：读写工作区、运行命令、用审批模式控制自主程度。OpenAI 的公开说明把 Full Auto 描述为当前目录范围内、网络默认关闭的 sandbox；这是一种围绕本地代码仓库划定执行边界的设计。[Codex CLI 说明](https://help.openai.com/en/articles/11096431)

Claude Code 也把工具许可直接暴露给 CLI，例如 `--allowedTools`、`--disallowedTools` 和 permission mode；其 MCP server 可通过 `claude mcp` 管理。[Claude Code CLI 参考](https://docs.anthropic.com/en/docs/claude-code/cli-usage) [Claude Code MCP 文档](https://docs.anthropic.com/en/docs/mcp)

Hermes 的不同处不在于“有一个 shell 工具”，而在于同一套 tool loop 要覆盖 CLI、gateway、消息平台、cron 和多种远程 backend。因此它把 environment 生命周期、远程脚本 RPC、browser provider、网页 provider 和审批上下文都拆成可替换的组件。代价是代码路径更长；收益是一个工具能在不同运行面复用，而不是只为单机 coding session 服务。

## 常见误解

### `web_extract` 能不能代替 browser？

不能。它负责抓取 URL 内容，没有浏览器中的交互状态、登录态和元素操作。静态文档优先 extract；需要点击、输入、执行页面 JavaScript 或检查实际页面流程时，才进入 browser。

### `browser_snapshot` 是不是截图？

不是默认意义上的截图。它主要是可访问性树的文本表示，包含可引用的元素 ref。模型以 ref 点击和输入，比用像素坐标更稳定。截图是另一类视觉信息，用于布局或图像内容检查。

### `execute_code` 里的工具调用是否绕过正常 guardrail？

不会自动绕过。脚本经父进程 RPC 调用允许的 Hermes 工具，仍会进入对应工具的审批和安全逻辑；脚本本身的任意 Python 风险则由 `check_execute_code_guard()` 单独处理。两层都需要，缺一层都有绕过空间。

### 为什么网页正文不直接全文塞给模型？

全文可能占满上下文，也会掩盖真正需要的段落。Hermes 返回头尾和明确的截断标记，把完整内容留在缓存中，让模型在必要时定向读取。这是把上下文预算当作运行时资源，而不是无限字符串。

### terminal 的 container backend 是不是天然安全？

不是。容器是否隔离取决于挂载、网络、运行用户和 host access。尤其 bind mount 会让容器里的写入影响宿主目录。Hermes 会把 Docker 是否有 host access 传进审批判断，但部署者仍要决定容器的真实权限。

## 核对清单

对于“查资料、登录后台、修改代码、跑测试”这类任务，工具选择顺序通常是：web tools 负责资料检索，browser 负责登录态和交互，terminal 负责修改与验证；只有中间步骤多且不需要逐步推理时，才考虑 `execute_code`。
