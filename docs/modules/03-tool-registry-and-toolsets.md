# Tool Registry 与 Toolsets：工具是怎么被模型看见的

## 这一课先解决什么问题

上一课讲 Prompt Builder 时，我们一直在说“模型能看到哪些内容”。这节课换到工具系统：模型能看到哪些工具？这些工具从哪里来？为什么有些工具注册了，但当前 session 里模型看不到？模型发起 tool call 后，又是谁把它派发到真实 Python 函数？

这几个问题不能混在一起。Hermes 把它们拆成四层：

```text
工具文件自注册 -> Tool Registry
工具分组选择 -> Toolsets
模型可见 schema -> model_tools.get_tool_definitions
真实工具执行 -> model_tools.handle_function_call -> registry.dispatch
```

这套设计的重点不是“把函数放进一个 dict”。它解决的是 Agent 产品里的几个硬问题：工具很多、平台很多、插件很多、权限不同、依赖不同、还有 MCP 这种运行时动态变化的工具。

## 功能 1：工具文件自己注册到中央 Registry

### 先讲人话

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

### 先讲人话

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

### 先讲人话

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

### 先讲人话

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

## 功能 6：`handle_function_call` 是执行前的关口

### 先讲人话

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

这体现了 Hermes 的安全取舍：扩展能力要开放，但覆盖内置能力必须可审计。

## 和 Codex / Claude Code 的对比

Codex 也有工具暴露和执行边界，但它的工具集更多由当前运行环境和宿主协议控制。用户看到的是一组已经由 Codex runtime 暴露出来的工具，模型并不需要理解一个庞大的插件式 registry。

Claude Code 更强调本地开发工作流，工具面围绕读写文件、执行命令、编辑审批、MCP 扩展展开。它也有项目上下文和工具权限，但 Hermes 的 toolsets 更像一个产品级路由层：同一套 agent runtime 要覆盖 CLI、Gateway、Cron、Webhook、消息平台、Kanban worker 和插件平台。

Hermes 的复杂度主要来自“多入口 + 多平台 + 长生命周期”。Tool Registry 和 Toolsets 就是为了让这些入口共享同一套工具定义，又能按场景收紧能力。

## 本篇要记住的东西

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
