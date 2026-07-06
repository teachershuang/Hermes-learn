# Agent Loop：一次对话如何推进

## 这篇先解决什么问题

这一篇只讲一件事：用户发来一条消息后，Hermes 的主循环怎样推进。

先不要把它想成“调用一次模型”。Hermes 的一轮对话更像一个小型状态机：

```text
准备上下文
-> 调模型
-> 如果模型要工具，执行工具，把结果放回消息列表
-> 再调模型
-> 直到模型给最终回答、预算用完、用户中断，或失败分支终止
```

这就是 `agent/conversation_loop.py` 里的 `run_conversation`。

## 功能 1：每一轮先构建 turn context

### 先讲人话

`run_conversation` 一进来，并不会马上请求模型。它先把这次用户输入整理成“本轮上下文”。

这个阶段要解决的问题很多：

- 用户消息是否需要清洗。
- 有没有旧的 conversation history。
- system prompt 是复用还是重建。
- 当前 task_id 是什么。
- 当前 turn_id 是什么。
- memory 或插件有没有临时上下文要注入。
- session 和写入来源如何绑定。

所以 Hermes 把这段准备逻辑放进 `build_turn_context`。主循环只拿它返回的结果继续跑。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 对话主循环 | `agent/conversation_loop.py` 中的 `run_conversation` | 一次 turn 的主入口 |
| turn 准备 | `agent/turn_context.py` 中的 `build_turn_context` | 构建本轮 messages、prompt、task_id 等 |
| system prompt 恢复 | `agent/conversation_loop.py` 中的 `_restore_or_build_system_prompt` | 让系统提示保持稳定 |

### 关键源码

`run_conversation` 开头先调用 `build_turn_context`：

```python
_ctx = build_turn_context(
    agent,
    user_message,
    system_message,
    conversation_history,
    task_id,
    stream_callback,
    persist_user_message,
    persist_user_timestamp,
    restore_or_build_system_prompt=_restore_or_build_system_prompt,
    install_safe_stdio=_install_safe_stdio,
    sanitize_surrogates=_sanitize_surrogates,
    summarize_user_message_for_log=_summarize_user_message_for_log,
    set_session_context=set_session_context,
    set_current_write_origin=set_current_write_origin,
    ra=_ra,
)

messages = _ctx.messages
active_system_prompt = _ctx.active_system_prompt
effective_task_id = _ctx.effective_task_id
turn_id = _ctx.turn_id
```

这里的重点不是参数多，而是职责划分：本轮的准备工作先集中做完，然后主循环只围绕 `messages`、`active_system_prompt`、`effective_task_id` 这些变量推进。

### 工程实现

`build_turn_context` 的结果可以理解成一包运行时材料：

| 字段 | 用途 |
| --- | --- |
| `messages` | 本轮要继续追加的消息列表 |
| `conversation_history` | 外部传入或恢复出的历史 |
| `active_system_prompt` | 当前 API 调用要用的 system prompt |
| `effective_task_id` | 工具隔离、回调、日志里用的任务标识 |
| `turn_id` | 当前 turn 的追踪标识 |
| `current_turn_user_idx` | 本轮用户消息在 messages 中的位置，后续注入临时上下文要用 |
| `_ext_prefetch_cache` | 外部记忆或 recall 结果 |
| `_plugin_user_context` | 插件在本轮注入的用户上下文 |

Hermes 这样做有两个好处。

第一，主循环不会被前置准备逻辑淹没。第二，prompt cache 更容易保持稳定。特别是 system prompt，Hermes 会尽量复用稳定前缀，不把临时 memory recall 随便塞进去。

### 失败路径

这个阶段常见问题不是模型失败，而是上下文污染：

- 插件注入内容如果直接改 system prompt，会破坏 prompt cache。
- 历史消息如果 role 顺序坏了，后面 API 可能返回空内容或 400。
- 用户消息里有异常字符，严格 provider 可能拒绝。
- session 上下文没绑定好，工具写入和记忆更新可能落到错误会话。

所以 Hermes 在主循环前后都有消息修复和清洗逻辑。下一节会看到它在 API 调用前再次 repair message sequence。

## 功能 2：主循环靠预算和中断控制

### 先讲人话

Agent 不能无限跑。模型可能一直调用工具，工具可能一直失败，用户也可能中途打断。所以主循环必须有停止边界。

Hermes 用两个东西控制：

- `max_iterations`：外层最大 API 调用次数。
- `IterationBudget`：线程安全预算计数器，支持 consume/refund。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 主循环 | `agent/conversation_loop.py` 中的 `while` | 每一次循环代表一次模型 API 调用机会 |
| 预算类 | `agent/iteration_budget.py` 中的 `IterationBudget` | 控制 agent turn 的迭代预算 |
| 中断标志 | `agent._interrupt_requested` | 用户打断或平台要求停止时使用 |

### 关键源码

主循环的条件很直接：

```python
while (
    api_call_count < agent.max_iterations
    and agent.iteration_budget.remaining > 0
) or agent._budget_grace_call:
    if agent._interrupt_requested:
        interrupted = True
        _turn_exit_reason = "interrupted_by_user"
        break

    api_call_count += 1
    agent._api_call_count = api_call_count

    if agent._budget_grace_call:
        agent._budget_grace_call = False
    elif not agent.iteration_budget.consume():
        _turn_exit_reason = "budget_exhausted"
        break
```

`IterationBudget` 本身很小：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self) -> None:
        with self._lock:
            if self._used > 0:
                self._used -= 1
```

注意 `refund`。Hermes 认为某些程序化工具调用不应该消耗预算，比如 `execute_code` 这种 RPC 风格的调用。

### 工程实现

一次循环大致代表：

```text
准备 API messages
-> 调用模型
-> 解析 assistant message
-> 如果有 tool_calls，执行工具并 continue
-> 如果没有 tool_calls，生成 final_response 并 return
```

预算控制的是“模型轮次”，不是工具内部步骤。一个模型响应里可以有多个 tool call。工具执行完后，如果还需要模型继续判断，就进入下一次 API 调用。

### 失败路径

预算和中断主要防这些问题：

- 模型重复调用同一个工具。
- 工具失败后模型不换策略。
- 上下文压缩或 provider retry 导致任务拖太久。
- 用户已经发送新指令，但旧任务还在跑。

Hermes 的策略不是简单 kill。它会尽量把中断变成可读的结果，关闭不完整的工具序列，并把已发生的消息持久化，避免 session 断在半路。

### 和 Codex / Claude Code 的差异

Codex 和 Claude Code 也需要预算和中断，但它们多围绕单个开发任务和当前工作区。

Hermes 还要服务 Gateway 和后台任务。用户可能在 Telegram 上发一条新消息打断旧任务，也可能有 cron 或子 Agent 在跑。所以 Hermes 的中断和预算不仅影响 CLI 体验，还影响跨平台会话一致性。

## 功能 3：API 调用前会重新整理 messages

### 先讲人话

模型 API 对消息格式很挑剔。尤其是 tool call 场景，assistant message、tool message、user message 的顺序必须正确。历史里如果混入孤儿 tool result，或者 tool call arguments 损坏，provider 可能直接报错，也可能返回空内容。

所以 Hermes 在每次 API 调用前都会修一遍消息。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 参数修复 | `agent._sanitize_tool_call_arguments` | 修复损坏的 tool call arguments |
| role 顺序修复 | `agent_runtime_helpers.repair_message_sequence_with_cursor` | 修复消息 role 交替问题 |
| API 消息清洗 | `agent._sanitize_api_messages` | 去掉孤儿 tool result 或补 stub |

### 关键源码

调用模型前，Hermes 先修 tool call arguments：

```python
repaired_tool_calls = agent._sanitize_tool_call_arguments(
    messages,
    logger=request_logger,
    session_id=agent.session_id,
)
```

再修消息序列：

```python
from agent.agent_runtime_helpers import repair_message_sequence_with_cursor

repaired_seq = repair_message_sequence_with_cursor(agent, messages)
```

最后构造 `api_messages`，并把 system prompt 放在最前面：

```python
effective_system = active_system_prompt or ""
if agent.ephemeral_system_prompt:
    effective_system = (
        effective_system + "\n\n" + agent.ephemeral_system_prompt
    ).strip()

if effective_system:
    api_messages = [{"role": "system", "content": effective_system}] + api_messages

api_messages = agent._sanitize_api_messages(api_messages)
```

### 工程实现

可以把这里看成一层“出门前检查”：

```text
内部 messages
-> 修 tool call arguments
-> 修 role 顺序
-> 注入本轮临时上下文
-> 插入 system prompt
-> provider 兼容性清洗
-> 发给模型
```

内部 messages 可以带一些 Hermes 自己需要的字段，比如 reasoning、持久化标记、Codex Responses 字段。真正发给 provider 前，必须删掉严格 API 不认识的字段。

### 失败路径

这一步如果做不好，会出现很难查的问题：

- provider 返回空内容，但根因是消息序列坏了。
- Mistral、Fireworks 等严格 API 拒绝未知字段。
- Anthropic 不接受最终 assistant block 是 thinking。
- tool result 没有对应 tool call，形成 orphan。
- assistant 有 tool call，但缺少后续 tool result。

Hermes 的代码里有大量“看起来啰嗦”的清洗逻辑，就是为了让不同 provider 都能吃下同一套内部消息历史。

## 功能 4：模型返回 tool call 后怎么处理

### 先讲人话

模型返回 tool call，并不代表工具马上执行。Hermes 还要先检查：

- 工具名是不是真的存在。
- 工具名能不能自动修复。
- 参数是不是合法 JSON。
- 是否需要去重或限制 delegate_task。
- 是否要先持久化 assistant tool-call turn。

这些检查通过后，才会进入工具执行。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| tool call 分支 | `agent/conversation_loop.py` 中 `if assistant_message.tool_calls` | 模型要求使用工具后的主分支 |
| 工具执行入口 | `run_agent.py` 中 `_execute_tool_calls` | 决定顺序执行还是并发执行 |
| 单工具执行 | `agent/agent_runtime_helpers.py` 中 `invoke_tool` | 执行内置 agent 工具或 registry 工具 |
| 具体执行器 | `agent/tool_executor.py` | 顺序/并发工具执行实现 |

### 关键源码

tool call 分支的前半段先验证工具名：

```python
if assistant_message.tool_calls:
    for tc in assistant_message.tool_calls:
        if tc.function.name not in agent.valid_tool_names:
            repaired = agent._repair_tool_call(tc.function.name)
            if repaired:
                tc.function.name = repaired

    invalid_tool_calls = [
        tc.function.name for tc in assistant_message.tool_calls
        if tc.function.name not in agent.valid_tool_names
    ]
```

然后检查参数是不是 JSON：

```python
for tc in assistant_message.tool_calls:
    args = tc.function.arguments
    if not args or not args.strip():
        tc.function.arguments = "{}"
        continue
    try:
        json.loads(args)
    except json.JSONDecodeError as e:
        invalid_json_args.append((tc.function.name, str(e)))
```

真正执行工具前，Hermes 会先把 assistant tool-call message 加入 messages，并尝试写入 SessionDB：

```python
messages.append(assistant_msg)
agent._emit_interim_assistant_message(assistant_msg)

agent._flush_messages_to_session_db(messages, conversation_history)

agent._execute_tool_calls(
    assistant_message,
    messages,
    effective_task_id,
    api_call_count,
)
```

这个顺序很讲究。先持久化 assistant tool-call turn，再执行工具。这样即使某个 destructive tool 导致进程重启，恢复时也知道“模型当时已经要求执行了哪个工具”。

### 工程实现

tool call 路径可以压成这条链：

```text
assistant_message.tool_calls
-> 校验工具名
-> 修复工具名
-> 校验 JSON 参数
-> 去重/限制特殊工具
-> append assistant tool-call message
-> flush 到 SessionDB
-> _execute_tool_calls
-> tool result append 到 messages
-> continue 下一次模型调用
```

`_execute_tool_calls` 自己不直接干所有事，它先判断能不能并发：

```python
def _execute_tool_calls(...):
    tool_calls = assistant_message.tool_calls
    self._executing_tools = True
    try:
        if not _should_parallelize_tool_batch(tool_calls):
            return self._execute_tool_calls_sequential(...)

        return self._execute_tool_calls_concurrent(...)
    finally:
        self._executing_tools = False
```

这个判断避免了危险并发。读文件、查状态这类独立工具可以并发；涉及写文件、重叠路径、未知 MCP 并发安全的调用就要谨慎。

### 失败路径

tool call 分支很容易失败，Hermes 分了几类处理：

| 失败 | 处理 |
| --- | --- |
| 工具名不存在 | 尝试修复；修不了就把错误作为 tool result 给模型自我纠正 |
| 工具参数不是 JSON | 少量重试；仍失败则注入错误 tool result |
| 参数被截断 | 不执行工具，直接按输出截断处理 |
| 重复调用无进展 | tool guardrail 警告或停止 |
| 工具执行异常 | 工具结果里返回错误，下一轮让模型换策略 |

这里体现了一个重要设计：Hermes 尽量把可恢复错误反馈给模型，而不是立刻崩掉。只有截断、预算耗尽、安全拦截这类情况才更倾向停止。

### 和 Codex / Claude Code 的差异

Codex 和 Claude Code 也会做工具名、参数和权限检查。区别在于 Hermes 的工具来源更杂：内置工具、插件、MCP、Gateway 相关工具、memory、skills、delegate、kanban 都可能出现在同一个系统里。

所以 Hermes 在 tool call 前后的校验更厚。它要防的不只是模型写错参数，还要防插件阻断、MCP 工具不可用、delegate 失控、工具结果污染长期上下文。

## 功能 5：没有 tool call 时如何结束

### 先讲人话

如果模型没有返回 tool call，Hermes 就把 assistant content 当作最终回答。但它仍然要做一些收尾：

- 去掉内部 think block。
- 处理空响应恢复。
- 输出给 stream callback。
- 持久化 session。
- 触发后续 memory/skill review。
- 清理 task 资源。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| final response 分支 | `agent/conversation_loop.py` 中 tool call 分支之后 | 处理最终文本 |
| 持久化 | `run_agent.py` 中 `_flush_messages_to_session_db` | 把新消息写入 SessionDB |
| turn 收尾 | `agent/conversation_loop.py` 尾部收尾逻辑 | 清理、回调、review |

### 工程实现

主循环最后会返回一个结构化结果，而不是只返回字符串：

```text
{
  "final_response": "...",
  "messages": [...],
  "api_calls": N,
  "completed": true/false,
  ...
}
```

这对 CLI 可能有点多，但对 Gateway、API server、测试、轨迹保存很有用。不同入口可以根据这些字段决定如何展示、是否重试、是否标记失败。

## 本篇要记住的东西

1. `run_conversation` 不是简单请求模型，它是一个带预算、修复、工具执行、持久化的状态机。
2. 每一轮先 `build_turn_context`，再进入模型/工具循环。
3. `IterationBudget` 控制模型调用轮次，某些工具调用可以 refund。
4. API 调用前会修 messages，因为 provider 对 role 顺序和字段很敏感。
5. tool call 执行前要校验工具名和 JSON 参数。
6. Hermes 会先持久化 assistant tool-call turn，再执行工具，避免中途崩溃后丢上下文。
7. 工具执行后不是直接结束，而是把结果回填 messages，再让模型继续判断。

## 小实验

在 Hermes 源码仓库里运行：

```powershell
rg -n "def run_conversation|build_turn_context|while \\(|assistant_message.tool_calls|_execute_tool_calls|IterationBudget" agent run_agent.py
```

你要能对应上：

- `build_turn_context`：本轮准备。
- `while (...)`：主循环边界。
- `IterationBudget`：预算消耗。
- `assistant_message.tool_calls`：工具分支。
- `_execute_tool_calls`：工具执行入口。

再定位工具执行：

```powershell
rg -n "def _execute_tool_calls|def invoke_tool|handle_function_call" run_agent.py agent model_tools.py
```

你会看到 tool call 从 agent loop 进入 runtime helper，再进入 `model_tools.handle_function_call`。

## QA

### 为什么 Hermes 不直接在模型返回 tool call 后执行？

因为模型输出不可信。它可能写错工具名、参数不是 JSON、参数被截断，也可能重复调用一个没有进展的工具。Hermes 必须先验证和修复，不能把模型输出直接当命令执行。

### 为什么执行工具前要先 flush 到 SessionDB？

因为工具可能有副作用。比如写文件、重启进程、启动子 Agent。如果副作用已经发生，但 assistant tool-call turn 没有持久化，恢复时就不知道这个副作用从哪里来。先写入 tool-call turn，可以保留可恢复的执行轨迹。

### 为什么 `execute_code` 可以 refund 预算？

`execute_code` 更像程序化工具调用的 RPC 通道。它可能在一次模型计划中被频繁使用。如果每次都吃掉一个完整模型迭代预算，会让预算失真。Hermes 对这类调用做 refund，是为了把预算留给真正的模型推理轮次。

### 这和普通 ReAct loop 有什么关系？

普通 ReAct loop 是“思考 -> 行动 -> 观察 -> 再思考”。Hermes 的主循环也符合这个形态，但工程上多了很多保护：消息修复、provider 适配、工具校验、持久化、预算、中断、压缩和 guardrails。论文里的 loop 是思想模型，Hermes 这里是生产 runtime。
