# Tool Call 分支与失败处理：模型发起工具调用以后发生了什么

## 目标与范围

工具注册和 Schema 暴露完成后，模型才能提交 tool call。本章分析 Hermes 怎样把这段不可信协议输入转成受约束的真实工具执行。

先把几个概念拆开：

| 概念 | 含义 |
| --- | --- |
| Tool | 一个可执行能力，例如 `read_file`、`terminal`、`web_search` |
| Tool call | 模型在某一轮 assistant message 里提出的一次工具调用请求 |
| Tool calls | 同一个 assistant message 里可以有多次工具调用请求 |
| Tool result | Hermes 执行工具后追加回 `messages` 的 `role="tool"` 消息 |

所以“单个工具”和“tool call”不是一回事。

```text
工具是能力定义
tool call 是模型某一轮决定使用这个能力的一次请求
```

一个工具可以被多次调用；一个 assistant message 也可以同时带多个 tool calls。

## 功能 1：tool call 分支从哪里开始

### 设计目的

模型返回 assistant message 后，Hermes 会检查它有没有 `tool_calls`。如果没有，就走普通最终回复路径。如果有，就进入 tool call 分支。

源码入口在 `agent/conversation_loop.py`：

```python
if assistant_message.tool_calls:
    ...
```

这一分支做的事情很多，但主线可以简化成：

```text
检查工具名
检查 arguments JSON
限制危险/重复调用
把 assistant(tool_calls) 先写入 messages
执行工具
把每个 tool result 追加回 messages
回到 while 循环，继续请求模型
```

注意最后一步。执行完工具以后，Hermes 不是直接回答用户，而是把工具结果交回模型，让模型基于结果继续推理。

## 功能 2：为什么先校验工具名

### 设计目的

模型可能会幻觉出不存在的工具名。Hermes 不能直接执行，也不能让消息序列断掉。

所以它先尝试修复工具名：

```python
for tc in assistant_message.tool_calls:
    if tc.function.name not in agent.valid_tool_names:
        repaired = agent._repair_tool_call(tc.function.name)
        if repaired:
            tc.function.name = repaired
```

如果修不好，就把错误作为 tool result 返回给模型：

```python
messages.append({
    "role": "tool",
    "name": tc.function.name,
    "tool_call_id": tc.id,
    "content": content,
})
```

这一步有两个目的：

1. 让模型知道自己调用错了，有机会下一轮改正。
2. 保持 assistant tool call 和 tool result 的配对关系。

很多 provider 对 tool call 消息序列要求很严格：assistant 发出了几个 `tool_call_id`，后面就必须有对应数量的 tool result。少一个都可能导致下一次 API 调用失败。

### 工具名错误为什么不直接终止

因为这类错误经常是模型自己能修正的。例如：

```text
ReadFile -> read_file
TodoTool_tool -> todo
```

Hermes 的策略是：先修复，修不好就把错误喂回模型，最多重试几次。只有连续失败超过限制，才返回 partial。

## 功能 3：为什么要校验 arguments JSON

tool call 的参数必须是 JSON。模型有时会输出坏 JSON，例如少了右括号、字符串没闭合、或者 provider 截断了输出。

Hermes 先把非字符串参数转成字符串：

```python
if isinstance(args, (dict, list)):
    tc.function.arguments = json.dumps(args)
```

空参数会变成 `{}`：

```python
if not args or not args.strip():
    tc.function.arguments = "{}"
```

然后用 `json.loads` 检查合法性。

如果判断是截断，Hermes 不执行工具，直接返回 partial：

```python
return {
    "final_response": "Response truncated due to output length limit",
    "completed": False,
    "partial": True,
}
```

如果只是普通 JSON 错误，Hermes 会先重试 API。重试仍失败后，会注入 tool error results，让模型下一轮修正。

这和工具名错误一样，核心都是：不要破坏消息序列。

## 功能 4：为什么要先把 assistant tool-call turn 持久化

这一点很重要。

Hermes 在真正执行工具之前，会先把 assistant 的 tool_calls 追加到 `messages`，并写入 SessionDB：

```python
messages.append(assistant_msg)
agent._flush_messages_to_session_db(messages, conversation_history)
```

源码注释说明了原因：如果某个工具有副作用，甚至导致 Hermes 重启或中断，恢复时仍然能看到“模型已经发出过这个 tool call”。

举个例子：

```text
assistant 发出 terminal tool call
Hermes 先把这个 tool call 写入 SessionDB
terminal 执行中进程中断
用户 resume
SessionDB 里仍然保留那次 assistant(tool_calls)
```

否则恢复后会出现一个更危险的问题：工具副作用可能已经发生了，但会话历史里没有记录模型为什么执行它。

这是 Hermes 在工具执行链里很工程化的一点：先记录意图，再执行副作用。

## 功能 5：一个 assistant message 可以有多个 tool calls

模型可能一次返回：

```text
assistant.tool_calls = [
  read_file(...),
  search_files(...),
  session_search(...)
]
```

Hermes 会把这一批 tool calls 当作一个批次处理。执行之前还会做两个限制：

```python
assistant_message.tool_calls = agent._cap_delegate_task_calls(...)
assistant_message.tool_calls = agent._deduplicate_tool_calls(...)
```

`_cap_delegate_task_calls` 是限制 `delegate_task` 这类子 Agent 调用数量，避免模型一轮里开太多并发子任务。

`_deduplicate_tool_calls` 是去掉同一轮里重复的 `(tool_name, arguments)`，避免模型重复调用同一个工具浪费成本。

## 功能 6：顺序执行和并发执行怎么选

Hermes 的 `_execute_tool_calls` 不总是顺序跑，也不总是并发跑。

入口在 `run_agent.py`：

```python
if not _should_parallelize_tool_batch(tool_calls):
    return self._execute_tool_calls_sequential(...)

return self._execute_tool_calls_concurrent(...)
```

源码注释给出的原则是：

```text
看起来独立的批次可以并发
只读工具通常可以并发
文件读写只有目标路径不重叠时才并发
其他交互性或有副作用的工具走顺序
```

为什么不能全部并发？

因为工具可能有副作用。例如两个 `patch` 同时改同一个文件，或者一个 `terminal` 命令依赖上一个命令的输出，乱并发会制造竞态。

为什么也不能全部顺序？

因为一批独立只读工具顺序执行会慢。例如同时读取几个文件、并行搜索几个来源，这些适合并发。

Hermes 的选择是保守并发：能证明相对独立才并发，不能证明就顺序。

## 功能 7：顺序执行路径

顺序执行在 `agent/tool_executor.py` 的 `execute_tool_calls_sequential`。

它按顺序遍历：

```python
for i, tool_call in enumerate(assistant_message.tool_calls, 1):
    ...
```

每个 tool call 会经历这些步骤：

```text
检查用户 interrupt
解析 arguments
展开 Tool Search bridge
应用 tool request middleware
检查 plugin block
检查 tool guardrail
执行特殊 agent tools
或调用 model_tools.handle_function_call
追加 tool result
flush SessionDB
```

这里有一类特殊工具不直接走通用 registry dispatch，例如：

```text
todo
session_search
memory
clarify
read_terminal
delegate_task
```

原因是它们需要访问 agent 内部状态，例如 todo store、SessionDB、memory store、clarify callback、父 agent 信息。通用 registry handler 不一定能拿到这些上下文。

普通工具则走：

```python
function_result = handle_function_call(...)
```

最终都会追加 tool result：

```python
messages.append(make_tool_result_message(function_name, _tool_content, tool_call.id))
```

## 功能 8：并发执行路径

并发执行在 `execute_tool_calls_concurrent`。

它会并发跑多个工具，但结果仍然按原始 tool call 顺序追加回 `messages`：

```python
Results are collected in the original tool-call order and appended to
messages so the API sees them in the expected sequence.
```

这点很重要。即使内部并发，外部消息序列仍然要稳定。模型看到的是：

```text
assistant(tool_calls=[A, B, C])
tool_result(A)
tool_result(B)
tool_result(C)
```

而不是哪个工具先跑完就先追加哪个。否则下一轮模型可能把结果和 tool_call_id 对不上。

## 功能 9：Tool Search bridge 在执行时还要再验一次权限

Tool Search 会延迟披露一部分 MCP 或 plugin 工具。模型先调用 `tool_search`，再通过 `tool_call` bridge 调用底层工具。

执行阶段还要再做一次 session 范围校验：

```python
if _underlying in _tool_search_scoped_names(agent):
    function_name = _underlying
    function_args = _underlying_args
else:
    _ts_scope_block = json.dumps({"error": ...})
```

为什么不能只在暴露阶段校验？

因为 `tool_call` 是一个桥。模型表面上调用的是 `tool_call`，里面的参数指定了真正要调用的底层工具。如果执行阶段不再校验，受限 session 可能通过桥绕过 toolset 边界。

所以 Hermes 在两个地方都守边界：

```text
生成 tools schema 时，Tool Search catalog 受 toolsets 限制
执行 tool_call bridge 时，underlying tool 再受 scoped names 限制
```

## 功能 10：工具结果怎么回填

Hermes 不直接把工具结果塞成普通 assistant 文本，而是构造成 `role="tool"` 消息。

入口是 `make_tool_result_message`：

```python
return {
    "role": "tool",
    "name": name,
    "tool_name": name,
    "content": wrapped,
    "tool_call_id": tool_call_id,
}
```

`tool_call_id` 是关键。它告诉模型：这条结果对应前面哪一个 tool call。

### 为什么要包裹不可信工具结果

`make_tool_result_message` 还会对高风险工具结果做语义包裹，例如：

```text
web_search
web_extract
browser_*
mcp_*
```

这些工具可能返回网页、GitHub issue、MCP response 里的外部内容。外部内容里可能藏着 prompt injection，例如“忽略之前所有指令”。

Hermes 的做法不是靠正则去猜每一种攻击，而是把高风险结果包成：

```xml
<untrusted_tool_result source="web_extract">
...
</untrusted_tool_result>
```

这是一种结构性防御：告诉模型这段是数据，不是新指令。

## 功能 11：工具循环护栏

工具失败后，模型可能会反复调用同一个工具、同一组参数，形成死循环。

Hermes 用 `agent/tool_guardrails.py` 做 per-turn 工具调用模式检测。这个模块本身是纯逻辑：

```python
The controller in this module is intentionally side-effect free:
it tracks per-turn tool-call observations and returns decisions.
```

它会返回几类 decision：

```text
allow
warn
block
halt
```

默认是 warning-first。也就是说，发现可疑循环时先给模型警告，不马上硬停。只有配置启用 hard stop，才会阻断或终止。

典型检测包括：

| 模式 | 含义 |
| --- | --- |
| repeated exact failure | 同一个工具、同一参数连续失败 |
| same tool failure | 同一工具多次失败 |
| idempotent no progress | 只读工具反复返回同样结果 |

当 guardrail 阻断某个工具时，Hermes 不会直接丢掉这个 tool call，而是合成一个 tool result：

```text
Tool blocked by guardrail policy
```

这样消息序列仍然完整，模型下一轮可以换策略。

## 功能 12：中断和剩余工具怎么处理

如果用户在工具执行中发出 interrupt，Hermes 不会继续启动剩余工具。

顺序路径里会对剩下的 tool calls 追加取消结果：

```python
messages.append(make_tool_result_message(
    skipped_name,
    "[Tool execution skipped ...]",
    skipped_tc.id,
))
```

这和前面的原则一致：每个 assistant tool call 都必须有对应 tool result。哪怕工具没执行，也要给模型一个结构化结果，说明它被跳过了。

## 功能 13：异常时补齐缺失 tool results

如果工具执行过程中出现非预期异常，Hermes 会扫描最近的 assistant tool_calls，给没有结果的 tool_call_id 补上 error tool result：

```python
if msg.get("role") == "assistant" and msg.get("tool_calls"):
    answered_ids = {...}
    for tc in msg["tool_calls"]:
        if tc["id"] not in answered_ids:
            messages.append({
                "role": "tool",
                "tool_call_id": tc["id"],
                "content": f"Error executing tool: {error_msg}",
            })
```

这个设计很细，但非常必要。因为 provider 要求：

```text
assistant(tool_calls) 后面必须跟对应 tool results
```

如果异常导致某个 tool result 缺失，下一轮 API 请求可能直接因为消息格式非法失败。补齐 error result，是为了让会话还能继续。

## 和 Codex / Claude Code 的对比

Codex 也很重视工具调用前后的消息完整性，尤其是本地文件编辑、命令执行、审批、sandbox 和补丁应用。它的工具执行更多被宿主运行时控制，用户看到的是一组已由环境提供的工具。

Claude Code 的工具链同样围绕本地开发任务展开，重点是文件读写、命令执行、MCP、hook、审批和上下文压缩。它也会处理 tool use / tool result 的配对关系。

Hermes 的特点是：同一套 tool call 分支要同时服务 CLI、Gateway、Cron、Webhook、Kanban worker、插件、MCP 和长生命周期 session。因此它对恢复、持久化、toolset 范围、Tool Search bridge、外部不可信结果包裹做得更重。

## 实现要点

1. Tool 是能力；tool call 是模型某一轮发出的调用请求。
2. 一个 assistant message 可以带多个 tool calls。
3. Hermes 先验证工具名和 JSON 参数，再执行工具。
4. assistant(tool_calls) 会先写入 SessionDB，再执行工具副作用。
5. 顺序还是并发，由 tool call 批次是否独立决定。
6. 每个 tool_call_id 都必须得到一个 tool result，哪怕是错误、跳过或阻断。
7. 高风险工具结果会被包成不可信数据，降低间接 prompt injection 风险。
8. tool guardrail 默认 warning-first，必要时合成 tool result 阻断循环。
9. 工具异常后补齐缺失 tool results，是为了保持下一轮 API 消息合法。

## QA

### tool call 和工具到底有什么区别？

工具是注册好的能力，例如 `read_file`。tool call 是模型在某一轮里提出“我要调用 `read_file` 并传这些参数”的请求。一个工具可以被多次调用，一个 assistant message 也可以带多个 tool calls。

### 为什么不直接执行模型给的 tool call？

因为模型可能给错工具名、坏 JSON、重复调用、越过 toolset 边界，或者被外部内容诱导。Hermes 必须先做校验、修复、范围检查和 guardrail，再执行。

### 为什么 invalid tool call 要作为 tool result 返回，而不是普通 assistant 文本？

因为 provider 的消息协议要求 assistant 的每个 tool_call_id 都有对应 tool result。把错误写成 tool result，既能保持消息序列合法，也能让模型下一轮看到错误并自我修正。

### 为什么工具执行前要先 flush SessionDB？

因为工具可能有副作用。如果副作用执行了，但 assistant tool call 没写入历史，resume 后就看不到模型为什么执行了这个动作。先持久化 tool-call turn，可以让恢复路径保留真实执行意图。

### 并发执行会不会打乱 tool result 顺序？

不会。Hermes 可以并发执行独立工具，但会按原始 tool call 顺序追加结果。这样模型看到的 `tool_call_id` 和结果顺序仍然稳定。

### Memory / Skill review 是不是在这里触发？

模型在 tool call 分支里调用 `memory` 或 `skill_manage` 时，它们属于普通工具调用。

后台 memory review 与 skill review 在 turn 收尾后运行，用于判断是否沉淀长期记忆或技能。它们不属于当前 tool call 批次，具体状态与隔离方式见 Memory 和 Skills 章节。
