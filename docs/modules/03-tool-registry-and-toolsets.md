# Tool Registry 与 Toolsets：工具是怎么被模型看见的

## 目标与范围

Prompt Builder 决定模型看到哪些上下文，工具系统则决定模型看到哪些能力。本章追踪工具从注册、分组和可用性检查，到 Schema 暴露与 Python handler 分发的完整路径。

这几个问题不能混在一起。Hermes 把它们拆成四层：

```text
工具文件自注册 -> Tool Registry
工具分组选择 -> Toolsets
模型可见 schema -> model_tools.get_tool_definitions
真实工具执行 -> model_tools.handle_function_call -> registry.dispatch
```

这套设计的重点不是“把函数放进一个 dict”。它解决的是 Agent 产品里的几个硬问题：工具很多、平台很多、插件很多、权限不同、依赖不同、还有 MCP 这种运行时动态变化的工具。

## 功能 1：工具文件自己注册到中央 Registry

### 设计目的

Hermes 的工具不是在 `model_tools.py` 里手写一个巨大列表。每个工具文件在被 import 时，自己调用 `registry.register(...)`，把工具名、所属 toolset、schema、handler、可用性检查函数一起登记进去。

比如 `tools/todo_tool.py`：

```python
registry.register(
    name="todo",
    toolset="todo",
    schema=TODO_SCHEMA,
    handler=lambda args, **kw: todo_tool(
        todos=args.get("todos"), merge=args.get("merge", False), store=kw.get("store")),
    check_fn=check_todo_requirements,
    emoji="...",
)
```

再看 `tools/terminal_tool.py`：

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=_handle_terminal,
    check_fn=check_terminal_requirements,
    max_result_size_chars=100_000,
)
```

这里已经能看出一个工具的基本结构：

| 字段 | 含义 |
| --- | --- |
| `name` | 模型 tool call 里会使用的工具名 |
| `toolset` | 这个工具属于哪个工具组 |
| `schema` | 给模型看的函数参数定义 |
| `handler` | 真正执行工具的 Python 函数 |
| `check_fn` | 当前环境是否允许暴露这个工具 |
| `max_result_size_chars` | 工具结果最大返回长度 |

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 中央注册表 | `tools/registry.py` 中的 `ToolRegistry` | 保存工具元数据和执行入口 |
| 工具发现 | `tools/registry.py` 中的 `discover_builtin_tools` | 扫描内置工具文件并 import |
| 工具注册 | 各个 `tools/*.py` 末尾的 `registry.register(...)` | 工具模块导入时完成注册 |
| 上层编排 | `model_tools.py` | 触发发现、生成 schema、执行派发 |

### 关键源码

`tools/registry.py` 的文件头已经把依赖方向写得很清楚：

```python
"""
Each tool file calls ``registry.register()`` at module level to declare its
schema, handler, toolset membership, and availability check.  ``model_tools.py``
queries the registry instead of maintaining its own parallel data structures.
"""
```

导入链是：

```text
tools/registry.py
    <- tools/*.py
    <- model_tools.py
    <- run_agent.py / cli.py / batch_runner.py
```

这个方向很重要。`registry.py` 不导入具体工具，具体工具只导入 registry。`model_tools.py` 是唯一负责触发工具发现的地方。这样可以避免循环导入。

## 功能 2：Hermes 用 AST 过滤工具文件，再 import 触发注册

### 设计目的

Hermes 不会粗暴 import `tools/` 下面所有 Python 文件。它先用 AST 看文件顶层有没有 `registry.register(...)`，只有命中的文件才 import。

源码在 `discover_builtin_tools`：

```python
module_names = [
    f"tools.{path.stem}"
    for path in sorted(tools_path.glob("*.py"))
    if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
    and _module_registers_tools(path)
]

for mod_name in module_names:
    importlib.import_module(mod_name)
```

`_module_registers_tools` 只检查模块顶层：

```python
return any(_is_registry_register_call(stmt) for stmt in tree.body)
```

这是一种很务实的工程选择。工具目录里会有辅助文件，辅助文件可能也引用 registry，但不应该被当成工具模块自动 import。只认顶层 `registry.register(...)`，可以减少误导入。

### 为什么不直接维护一个 import 列表

早期工具少时，手写 import 列表没问题。工具多了以后，问题会变明显：

- 新增工具要改工具文件，还要改 `model_tools.py`。
- 删除工具容易留下旧 import。
- 插件和 MCP 动态工具很难统一。
- 工具元数据可能散落在多个平行 dict 里。

现在的设计是：工具文件自己声明，registry 统一保存，model_tools 只查询。新增内置工具时，核心路径少改一处。

## 功能 3：Registry 只说明“有哪些工具”，Toolsets 决定“当前要给模型哪些工具”

### 设计目的

工具注册进 registry，不代表模型这一轮一定能看见它。模型能看见什么，还要经过 toolset 选择。

`toolsets.py` 定义了很多工具组，例如：

```python
"web": {
    "description": "Web research and content extraction tools",
    "tools": ["web_search", "web_extract"],
    "includes": []
}
```

还有平台级 toolset：

```python
"hermes-cli": {
    "description": "Full interactive CLI toolset - all default tools plus cronjob management",
    "tools": _HERMES_CORE_TOOLS,
    "includes": []
}
```

核心工具列表 `_HERMES_CORE_TOOLS` 里包含 web、terminal、file、skills、browser、memory、session_search、clarify、execute_code、delegate_task 等常用工具。

### toolset 是权限边界，不只是分类

看起来 toolset 像分类，其实它也是权限边界。

例如 webhook 场景经常来自第三方不可信内容，所以 Hermes 有一个更收紧的 `_HERMES_WEBHOOK_SAFE_TOOLS`：

```python
_HERMES_WEBHOOK_SAFE_TOOLS = [
    "web_search",
    "web_extract",
    "vision_analyze",
    "clarify",
]
```

这说明 Hermes 不会把所有入口都默认给 shell、文件写入、代码执行能力。不同平台看到的工具面不同。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 静态工具组 | `toolsets.py` 中的 `TOOLSETS` | 定义内置工具组 |
| 核心工具 | `toolsets.py` 中的 `_HERMES_CORE_TOOLS` | CLI 和多数平台共享的默认能力 |
| 解析工具组 | `toolsets.py` 中的 `resolve_toolset` | 把 toolset 展开成工具名列表 |
| 校验工具组 | `toolsets.py` 中的 `validate_toolset` | 判断 toolset 名称是否有效 |

### 关键源码

`resolve_toolset` 支持组合和递归展开：

```python
tools = set(toolset.get("tools", []))

for included_name in toolset.get("includes", []):
    included_tools = resolve_toolset(included_name, visited, include_registry=include_registry)
    tools.update(included_tools)

return sorted(tools)
```

所以 toolset 可以直接列工具，也可以 include 其他 toolset。它不是简单数组，而是一个可组合的能力声明。

## 功能 4：`get_tool_definitions` 把 toolset 变成模型能看的 schema

### 设计目的

模型 API 需要的是工具 schema，不是 Python 函数。Hermes 要先把启用的 toolsets 解析成工具名，再去 registry 取 schema。

入口是 `model_tools.py` 的 `get_tool_definitions`：

```python
def get_tool_definitions(
    enabled_toolsets=None,
    disabled_toolsets=None,
    quiet_mode=False,
    skip_tool_search_assembly=False,
):
```

真正计算在 `_compute_tool_definitions`：

```python
if enabled_toolsets is not None:
    for toolset_name in effective_enabled_toolsets:
        resolved = resolve_toolset(toolset_name)
        tools_to_include.update(resolved)
else:
    for ts_name in get_all_toolsets():
        tools_to_include.update(resolve_toolset(ts_name))
```

然后再处理禁用工具组：

```python
if disabled_toolsets:
    for toolset_name in disabled_toolsets:
        resolved = resolve_toolset(toolset_name)
        tools_to_include.difference_update(resolved)
```

最后问 registry 要 schema：

```python
filtered_tools = registry.get_definitions(tools_to_include, quiet=quiet_mode)
```

这一层输出的才是模型会收到的 `tools` 数组。

### 为什么需要 `check_fn`

即使某个工具在 toolset 里，也不代表当前机器可用。比如 terminal、browser、Home Assistant、x_search、video_gen 都可能依赖环境变量、二进制、外部服务或用户配置。

`registry.get_definitions` 会执行 `check_fn`：

```python
if entry.check_fn:
    if entry.check_fn not in check_results:
        check_results[entry.check_fn] = _check_fn_cached(entry.check_fn)
    if not check_results[entry.check_fn]:
        continue
```

这一步很关键。模型不会看到当前不可用的工具。否则模型就会调用一个环境里根本跑不起来的工具，下一轮又要解释失败，浪费上下文和时间。

Hermes 还会缓存 `check_fn` 结果，避免每次生成工具 schema 都探测环境。工具可用性检查可能很贵，例如检查浏览器、容器、远端服务。

## 功能 5：注册、暴露、执行是三件事

这一点最容易混。

| 阶段 | 问题 | 关键代码 |
| --- | --- | --- |
| 注册 | Hermes 知不知道有这个工具？ | `registry.register(...)` |
| 暴露 | 当前 session 的模型能不能看见它？ | `get_tool_definitions(...)` |
| 执行 | 模型已经调用它，谁来跑 handler？ | `handle_function_call(...) -> registry.dispatch(...)` |

举个例子。

`terminal` 在 `tools/terminal_tool.py` 注册了：

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=_handle_terminal,
    check_fn=check_terminal_requirements,
)
```

但如果当前 session 没有启用包含 `terminal` 的 toolset，模型看不到它。

如果当前 session 启用了，但 `check_terminal_requirements` 返回 False，模型也看不到它。

只有当 toolset 允许、环境检查通过、schema 被放进 API 请求后，模型才可能发起：

```json
{"name": "terminal", "arguments": {"cmd": "..."}}
```

这时才进入执行派发。

### 工具可见性的完整判定链

“当前 session 的模型能不能看见某个工具”，主要由 `model_tools.get_tool_definitions` 这条链判断：

```text
enabled_toolsets / disabled_toolsets
        ↓
toolsets.resolve_toolset(...)
        ↓
tools_to_include
        ↓
registry.get_definitions(...)
        ↓
check_fn 过滤
        ↓
dynamic schema 修正
        ↓
schema sanitizer
        ↓
Tool Search progressive disclosure
        ↓
最终传给模型的 tools 数组
```

具体判断分几步。

第一步，看当前 session 启用了哪些 toolsets。源码在 `model_tools.py`：

```python
if enabled_toolsets is not None:
    for toolset_name in effective_enabled_toolsets:
        resolved = resolve_toolset(toolset_name)
        tools_to_include.update(resolved)
else:
    for ts_name in get_all_toolsets():
        tools_to_include.update(resolve_toolset(ts_name))
```

如果显式传了 `enabled_toolsets`，就只从这些 toolsets 里取工具。否则走默认的全局 toolset 解析。

第二步，再减去 `disabled_toolsets`：

```python
if disabled_toolsets:
    for toolset_name in disabled_toolsets:
        resolved = resolve_toolset(toolset_name)
        tools_to_include.difference_update(resolved)
```

这一步很重要。即使某个工具被平台 bundle 带进来了，也可以通过 disabled toolsets 做减法。

第三步，交给 registry 取 schema：

```python
filtered_tools = registry.get_definitions(tools_to_include, quiet=quiet_mode)
```

`registry.get_definitions` 不是简单返回 schema。它会检查每个工具自己的 `check_fn`：

```python
if entry.check_fn:
    if entry.check_fn not in check_results:
        check_results[entry.check_fn] = _check_fn_cached(entry.check_fn)
    if not check_results[entry.check_fn]:
        continue
```

所以工具可见性不是只看名字是否在 toolset 里。它至少要同时满足：

```text
工具已经注册到 registry
工具名被当前 enabled toolsets 展开出来
没有被 disabled toolsets 移除
工具自己的 check_fn 通过
schema 没有在动态修正中被移除
没有被 Tool Search 延迟披露
```

第四步，Hermes 还会根据真正可用的工具修正部分 schema。

例如 `execute_code` 的 schema 会只列出当前真的可用的 sandbox tools：

```python
available_tool_names = {t["function"]["name"] for t in filtered_tools}
if "execute_code" in available_tool_names:
    sandbox_enabled = SANDBOX_ALLOWED_TOOLS & available_tool_names
    dynamic_schema = build_execute_code_schema(sandbox_enabled, mode=_get_execution_mode())
```

这解决的是“描述里说某工具可用，但模型实际不能调用”的问题。否则模型会被 schema 误导。

第五步，Tool Search 可能把一部分 MCP / plugin 工具从完整 tools 数组里拿掉，换成 `tool_search / tool_describe / tool_call` 三个桥接工具。此时工具并不是彻底不可用，而是变成“需要先搜索，再描述，再桥接调用”。

所以更准确地说，模型能不能看见工具有两种状态：

| 状态 | 含义 |
| --- | --- |
| 直接可见 | 工具 schema 直接在 `tools` 数组里 |
| 延迟可见 | 工具被 Tool Search 收起，需要先通过 `tool_search` 发现 |

如果一个工具既不在直接 tools 数组，也不在当前 session 的 Tool Search 范围里，那模型就不应该能调用它。

## 功能 6：`handle_function_call` 是执行前的关口

### 设计目的

`registry.dispatch` 会执行 handler，但 Hermes 不会让模型 tool call 直接打到 dispatch。中间还有 `model_tools.handle_function_call`。

它负责做这些事情：

- 参数类型 coercion。
- Tool Search bridge 的特殊处理。
- tool request middleware。
- 插件 `pre_tool_call` 阻断。
- ACP / Zed 编辑审批。
- read/search loop 计数重置。
- 执行时间统计。
- tool execution middleware。
- 最后再调用 `registry.dispatch`。

关键源码：

```python
function_args = coerce_tool_args(function_name, function_args)
```

执行前插件可以阻断：

```python
block_message = get_pre_tool_call_block_message(...)
if block_message is not None:
    return json.dumps({"error": block_message}, ensure_ascii=False)
```

真正派发在这里：

```python
return registry.dispatch(
    function_name, next_args,
    task_id=task_id,
    session_id=session_id,
    user_task=user_task,
)
```

这说明 Registry 是最终执行路由，但不是全部安全边界。真正的执行前控制大多在 `handle_function_call` 和更外层的 agent loop。

### 执行前关口逐项解释

`handle_function_call` 里那串步骤看起来杂，其实可以分成四类：修正模型输出、限制能力边界、接入插件扩展、保存可观测性。

| 步骤 | 作用 | 为什么需要 |
| --- | --- | --- |
| 参数类型 coercion | 按工具 JSON Schema 修正参数类型 | 模型经常把 `42` 写成 `"42"`，把数组写成字符串 |
| Tool Search bridge | 处理 `tool_search / tool_describe / tool_call` | 工具太多时渐进式披露，避免一次塞爆 tools 数组 |
| tool request middleware | 在执行前改写工具参数 | 给插件或宿主一个“修改请求”的机会 |
| `pre_tool_call` 阻断 | 插件可以拒绝某次工具调用 | 做安全策略、审计策略或平台限制 |
| ACP / Zed 编辑审批 | 文件修改前请求宿主批准 | 编辑类工具不能绕过 IDE / ACP 的权限控制 |
| read/search loop 计数重置 | 非读工具执行后重置连续读计数 | 防止模型陷入重复读文件 / 搜索的循环 |
| 执行时间统计 | 记录工具耗时 | 给 post hook、日志、性能诊断使用 |
| tool execution middleware | 包裹真实执行过程 | 插件可以做 tracing、重试、包装结果 |
| `registry.dispatch` | 找到 handler 并执行 | 最后一跳，真正调用工具实现 |

#### 参数类型 coercion

模型输出的 tool arguments 不总是严格符合 schema。比如工具需要：

```json
{"limit": 5, "merge": true, "urls": ["https://example.com"]}
```

模型可能给：

```json
{"limit": "5", "merge": "true", "urls": "https://example.com"}
```

`coerce_tool_args` 会根据 registry 里的 schema 做安全修正：

```python
schema = registry.get_schema(tool_name)
...
coerced = _coerce_value(value, expected, schema=prop_schema)
```

它还会把 schema 期待数组、但模型给了单个值的情况包成单元素数组。这样能减少很多无意义的工具失败。

#### Tool Search bridge

Tool Search 是给 MCP / plugin 工具准备的渐进式披露机制。工具太多时，Hermes 不把所有工具 schema 直接给模型，而是给三个桥接工具：

```text
tool_search    搜索可用工具
tool_describe  查看某个工具参数
tool_call      调用被延迟披露的工具
```

但这里有一个安全点：`tool_call` 不能绕过当前 session 的 toolsets。

源码里会重新用当前 `enabled_toolsets / disabled_toolsets` 生成 scoped catalog：

```python
current_defs = get_tool_definitions(
    enabled_toolsets=enabled_toolsets,
    disabled_toolsets=disabled_toolsets,
    quiet_mode=True,
    skip_tool_search_assembly=True,
)
```

然后检查 underlying tool 是否在当前 session 允许的延迟工具集合里：

```python
if underlying_name not in _scoped_deferrable:
    return json.dumps({"error": ...})
```

这一步防止受限 session 通过 `tool_call` 偷调全局 registry 里的工具。

#### tool request middleware

`tool_request` middleware 发生在工具真正执行前，用来改写参数：

```python
_tool_request_mw = apply_tool_request_middleware(...)
function_args = _tool_request_mw.payload
```

它和 observer hook 不一样。observer 只是看见事件；middleware 可以改变事件。比如插件可以给某个工具参数补默认值、替换目标环境、加审计标记，或者把不合规参数改成安全版本。

#### `pre_tool_call` 阻断

`pre_tool_call` 是插件阻断点：

```python
block_message = get_pre_tool_call_block_message(...)
if block_message is not None:
    return json.dumps({"error": block_message}, ensure_ascii=False)
```

如果插件返回 block message，这次工具调用不会继续执行。它适合做策略阻断，例如某个平台不允许执行 shell，或者某类 URL / 路径需要拒绝。

#### ACP / Zed 编辑审批

这一步只在 ACP / Zed 这类宿主绑定了 requester 时生效：

```python
edit_block_message = maybe_require_edit_approval(function_name, function_args)
if edit_block_message is not None:
    return edit_block_message
```

如果工具是 `write_file`、`patch` 这类会改文件的操作，宿主可以先展示 diff，让用户或 IDE 批准。没有 requester 时，这一步直接跳过。

#### read/search loop 计数重置

Hermes 有读文件 / 搜索文件的循环防护。如果模型连续读同一类内容，系统需要识别它是不是陷入了“读了又读”的循环。

但如果中间执行了其他工具，比如 patch、terminal、write_file，说明模型已经不只是重复读。于是执行非 read/search 工具时会重置计数：

```python
if function_name not in _READ_SEARCH_TOOLS:
    notify_other_tool_call(task_id or "default")
```

这不是安全审批，而是行为模式纠偏。

#### 执行时间统计和 post hook

Hermes 会记录工具耗时：

```python
_dispatch_start = time.monotonic()
...
duration_ms = int((time.monotonic() - _dispatch_start) * 1000)
```

耗时会传给后续 hook，用于日志、插件监控、性能诊断。比如某个 web 工具突然变慢，插件可以基于 duration 做告警或统计。

#### tool execution middleware

真正执行工具前，Hermes 还会把 dispatch 包进 execution middleware：

```python
result = run_tool_execution_middleware(
    function_name,
    function_args,
    _dispatch,
    original_args=_tool_original_args,
)
```

这和 request middleware 的区别是：

```text
tool_request middleware：改参数
tool_execution middleware：包住执行过程
```

execution middleware 可以做 tracing、超时包装、重试、结果加工，最后再调用 `next_call(args)` 进入真实 dispatch。

#### 最后一跳：`registry.dispatch`

前面的关口都过了，才到：

```python
return registry.dispatch(function_name, next_args, ...)
```

所以不要把 `registry.dispatch` 理解成整个工具系统。它只是最后一跳。真正复杂的工程边界在 dispatch 之前。

## 功能 7：`registry.dispatch` 只做最后一跳

`registry.dispatch` 的逻辑很克制：

```python
entry = self.get_entry(name)
if not entry:
    return json.dumps({"error": f"Unknown tool: {name}"})

if entry.is_async:
    return _run_async(entry.handler(args, **kwargs))
return entry.handler(args, **kwargs)
```

它只负责：

- 根据工具名找到 `ToolEntry`。
- 判断同步还是异步 handler。
- 调用 handler。
- 捕获异常并返回 JSON error。

这就是一个好的 registry 该做的事：少做业务判断，只做路由和元数据查询。权限、平台、插件 hook、审批、参数修复这些更复杂的事情，放在外层。

## 功能 8：为什么 Hermes 要支持动态注销和覆盖

`ToolRegistry` 还有 `deregister` 和 `override=True`。这不是为了炫技，而是为了插件和 MCP。

MCP server 的工具列表可能运行时变化。服务器发来 `tools/list_changed` 后，Hermes 需要把旧工具删掉，再注册新工具。

插件也可能想替换某个内置工具。但这很危险，所以源码里要求显式 `override=True`，而且插件覆盖内置工具还要 operator opt-in：

```python
elif override:
    _owner = self._plugin_owner_of(handler)
    if _owner is not None and not self._plugin_override_policy.get(_owner, False):
        raise PermissionError(...)
```

这里的安全边界很明确：扩展可以注册新能力，覆盖内置能力则必须留下可审计记录。

## 和 Codex / Claude Code 的对比

Codex 也有工具暴露和执行边界，但它的工具集更多由当前运行环境和宿主协议控制。用户看到的是一组已经由 Codex runtime 暴露出来的工具，模型并不需要理解一个庞大的插件式 registry。

Claude Code 更强调本地开发工作流，工具面围绕读写文件、执行命令、编辑审批、MCP 扩展展开。它也有项目上下文和工具权限，但 Hermes 的 toolsets 更像一个产品级路由层：同一套 agent runtime 要覆盖 CLI、Gateway、Cron、Webhook、消息平台、Kanban worker 和插件平台。

Hermes 的复杂度主要来自“多入口 + 多平台 + 长生命周期”。Tool Registry 和 Toolsets 就是为了让这些入口共享同一套工具定义，又能按场景收紧能力。

## 实现要点

1. `registry.register(...)` 只是让 Hermes 知道有这个工具。
2. Toolset 决定当前 session 理论上允许哪些工具。
3. `check_fn` 决定当前环境下工具是否真的可暴露。
4. `get_tool_definitions` 输出的是模型实际看到的 tool schemas。
5. `handle_function_call` 是执行前的主要关口。
6. `registry.dispatch` 只是最后一跳：找到 handler，执行它，返回结果。
7. 插件和 MCP 让 registry 必须支持动态注册、注销和可审计覆盖。

## QA

### 为什么工具注册了，模型还是看不到？

因为注册、暴露、执行是三件事。工具注册到 registry，只说明 Hermes 知道它存在。模型能不能看到，还要看当前 session 的 enabled toolsets、disabled toolsets，以及工具自己的 `check_fn` 是否通过。

### Toolset 是不是只是分类？

不是。Toolset 同时是能力组合和权限边界。`webhook` 这类不可信入口就会使用更窄的工具面，避免第三方内容诱导模型执行 shell、写文件或跑代码。

### 为什么不把所有工具都给模型？

工具太多会浪费 token，也会增加误调用概率。更重要的是，不同入口的信任等级不同。CLI 里可用的工具，不一定应该出现在 webhook 或受限 worker 里。

### Registry 为什么不直接处理权限？

因为 registry 应该保持简单：保存工具元数据，提供查询，执行最后一跳。权限和平台策略变化更快，放在 `toolsets.py`、`model_tools.py`、agent loop、插件 hook 和审批系统里更合适。

### Tool Search 和 Tool Registry 是什么关系？

Tool Registry 是全量工具目录。Tool Search 是渐进式披露机制，用来在工具很多时先隐藏部分工具，只让模型通过搜索发现和调用。即使走 Tool Search bridge，也必须受到当前 session toolsets 的范围限制，不能绕过权限边界。

### 用户说“继续”时，会不会触发某个工具？

通常不会。“继续”首先是一条普通 user message，不是 tool call，也不是 registry 里的某个工具。

它能让模型接着做，是因为当前 turn 的 `messages` 里已经带着前面的对话历史。`agent/turn_context.py` 会先复制 `conversation_history`，再把本轮用户消息追加进去：

```python
messages = list(conversation_history) if conversation_history else []
...
messages.append({"role": "user", "content": user_message})
```

所以模型看到的不是孤零零的“继续”，而是：

```text
前面的用户请求
前面的 assistant 回复
前面的 tool call / tool result
用户：继续
```

这也解释了为什么它不属于 Tool Registry 的职责。Registry 负责“工具是否存在、是否暴露、如何派发”；“继续”属于对话上下文管理，主要依赖 `conversation_history`、SessionDB、resume 和 context compression。

如果用户退出后再用 `hermes --continue` 或 `hermes --resume <session_id>`，Hermes 会从 SessionDB 恢复历史消息，填回 `conversation_history`。下一轮仍然走同一条路径：历史消息先进入 `messages`，新的“继续”再追加到尾部。

当历史太长时，Context Compression 会把中间内容压成摘要，同时保留最近尾部。此时“继续”的语义来自：

```text
压缩摘要 + 最近几轮原文 + 当前用户消息
```

这个边界要记清楚：

| 能力 | 负责什么 |
| --- | --- |
| Tool Registry | 工具注册、查询、最后派发 |
| Toolsets | 当前 session 允许哪些工具暴露给模型 |
| conversation_history | 当前会话的直接上下文 |
| SessionDB | 会话持久化、resume、session_search 的数据来源 |
| Context Compression | 历史太长时保留可继续工作的摘要和尾部 |
| Memory | 长期偏好和稳定事实，不应该主要保存“任务做到哪一步” |
