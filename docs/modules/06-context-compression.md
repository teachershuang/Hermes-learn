# Context Compression：上下文太长以后，Hermes 怎么继续工作

## 这一讲先解决什么问题

Agent 的上下文不是无限的。对话一长，里面会堆积用户需求、assistant 回复、工具调用、文件内容、终端输出、图片占位、错误日志。继续把所有历史原样塞给模型，迟早会遇到两个问题：

第一，超过 provider 的上下文窗口，请求直接失败。

第二，还没失败，模型也已经被旧任务、旧工具输出、旧计划拖慢甚至带偏。

Hermes 的 Context Compression 解决的不是“省 token”这么简单。它要在丢掉一部分原始上下文的同时，保留继续工作所需的信息，并且不能破坏 tool call 消息序列、SessionDB 历史、memory 提取、gateway 会话路由。

这就是这一讲的主线：

```text
判断是否该压缩
  -> 选择哪些内容保留原文
  -> 裁剪旧工具输出
  -> 总结中间窗口
  -> 组装压缩后的 messages
  -> 更新 system prompt
  -> 写回 SessionDB
  -> 继续下一轮模型调用
```

## 功能 1：Hermes 的压缩不是一个函数，而是一套上下文引擎

入口类在 `agent/context_compressor.py`：

```python
class ContextCompressor(ContextEngine):
    """Default context engine — compresses conversation context via lossy summarization."""

    @property
    def name(self) -> str:
        return "compressor"
```

这里继承的是 `ContextEngine`。这说明 Hermes 把“上下文管理”抽成了一个可替换的引擎接口。默认实现是 compressor，但后续也可以挂其他 context engine，比如更复杂的本地上下文图谱、长期检索引擎、插件式压缩器。

`agent/context_engine.py` 对这个抽象的描述很清楚：

```python
class ContextEngine:
    """Interface for context management engines.

    Default is "compressor" (the built-in). Only one engine is active.
    """

    def should_compress(self, prompt_tokens: int = None) -> bool:
        return False

    def compress(self, messages, current_tokens=None, focus_topic=None, force=False):
        return messages
```

这带来一个重要结论：Hermes 的压缩不是散落在主循环里的临时代码，而是一个 runtime 子系统。主循环只问它两个问题：

1. 现在该不该压缩？
2. 如果该压缩，请给我一份新的 messages。

至于怎么压缩、用哪个辅助模型、保护哪些消息、失败时怎么办，都封装在 context engine 里。

## 功能 2：压缩参数从哪里来

压缩配置在 agent 初始化时读取。源码在 `agent/agent_init.py`：

```python
_compression_cfg = _agent_cfg.get("compression", {})
compression_threshold = float(_compression_cfg.get("threshold", 0.50))
compression_enabled = str(_compression_cfg.get("enabled", True)).lower() in {"true", "1", "yes"}
compression_target_ratio = float(_compression_cfg.get("target_ratio", 0.20))
compression_protect_last = int(_compression_cfg.get("protect_last_n", 20))
compression_protect_first = max(
    0, int(_compression_cfg.get("protect_first_n", 3))
)
compression_in_place = is_truthy_value(
    _compression_cfg.get("in_place"), default=False
)
```

这些配置分别控制不同层面：

| 配置 | 作用 |
| --- | --- |
| `enabled` | 是否启用自动压缩 |
| `threshold` | 上下文达到窗口多少比例时触发压缩，默认 0.50 |
| `target_ratio` | 压缩后摘要和尾部预算的大致比例 |
| `protect_first_n` | 开头保留多少条非 system 消息 |
| `protect_last_n` | 尾部至少保护多少条近期消息 |
| `abort_on_summary_failure` | 摘要失败时是否直接放弃压缩 |
| `in_place` | 压缩后是否保持同一个 session id |

`protect_first_n` 和 `protect_last_n` 很好理解，但它们不是唯一规则。Hermes 后面还会按 token budget 保护尾部，避免“最后 20 条消息里有一个巨大工具输出”把压缩搞失效。

初始化 compressor 时，Hermes 会传入主模型、provider、base_url、max_tokens、辅助压缩模型配置等信息：

```python
ContextCompressor(
    model=agent.model,
    threshold_percent=compression_threshold,
    protect_first_n=compression_protect_first,
    protect_last_n=compression_protect_last,
    summary_target_ratio=compression_target_ratio,
    provider=agent.provider,
    max_tokens=agent.max_tokens,
)
```

这说明压缩策略和当前模型强绑定。不同模型的 context window 不同，`max_tokens` 还会占用同一个窗口的输出空间。Hermes 不能只看“总上下文长度”，还要估算“输入实际可用空间”。

## 功能 3：什么时候触发压缩

Hermes 里压缩有三条主要入口。

### 入口一：turn 开始前的 preflight

源码在 `agent/turn_context.py`：

```python
if agent.compression_enabled and _should_run_preflight_estimate(...):
    _preflight_tokens = estimate_request_tokens_rough(
        messages,
        system_prompt=active_system_prompt or "",
        tools=agent.tools or None,
    )
    ...
    elif _compressor.should_compress(_preflight_tokens):
        messages, active_system_prompt = agent._compress_context(
            messages,
            system_message,
            approx_tokens=_preflight_tokens,
            task_id=effective_task_id,
        )
```

preflight 的作用是在真正请求模型前做一次粗估。如果估算已经接近阈值，就先压缩，再发请求。

注意估算时包含了 `tools=agent.tools`。这很关键。工具 schema 本身可能很大，几十个工具会贡献大量 token。只估 messages 会低估请求体大小，导致等 provider 报错后才发现超了。

### 入口二：provider 报 payload/context 错误后的 recovery

源码在 `agent/conversation_loop.py`。当请求失败并被分类为 payload too large 或 context overflow 时，Hermes 会压缩后重试：

```python
if is_payload_too_large:
    messages, active_system_prompt = agent._compress_context(
        messages,
        system_message,
        approx_tokens=approx_tokens,
        task_id=effective_task_id,
    )
    conversation_history = conversation_history_after_compression(agent, messages)
    _retry.restart_with_compressed_messages = True
```

另一个分支是 context-length error：

```python
if is_context_length_error:
    new_ctx = get_context_length_from_provider_error(error_msg, old_ctx)
    if new_ctx is not None:
        compressor.update_model(context_length=new_ctx, ...)
    messages, active_system_prompt = agent._compress_context(...)
```

这里有一个工程细节：Hermes 会区分“输入太长”和“输出 cap 太大”。如果 provider 的错误只是说 `max_tokens` 超过输出上限，压缩历史没有意义，应该降低输出 token cap。把所有 400 都扔进压缩循环，会进入死循环。

### 入口三：一轮 tool call 之后的自动检查

当 assistant 发起工具调用、工具结果回到消息流后，下一轮请求前上下文可能突然变大。Hermes 会根据 provider 返回的真实 prompt token 或粗估值，再决定是否压缩：

```python
_compressor = agent.context_compressor
if _compressor.last_prompt_tokens > 0:
    _real_tokens = _compressor.last_prompt_tokens
else:
    _real_tokens = estimate_request_tokens_rough(
        messages,
        tools=agent.tools or None,
    )

if agent.compression_enabled and _compressor.should_compress(_real_tokens):
    messages, active_system_prompt = agent._compress_context(...)
```

这条路径很常见。比如模型读了一个大文件、跑了一个长日志命令，工具结果进了上下文，压缩压力马上上来。

## 功能 4：should_compress 不只是比较 token 数

`ContextCompressor.should_compress` 表面上看很简单：

```python
def should_compress(self, prompt_tokens: int = None) -> bool:
    tokens = prompt_tokens if prompt_tokens is not None else self.last_prompt_tokens
    if tokens < self.threshold_tokens:
        return False
```

但它后面还有两类保护。

第一类是失败冷却。如果摘要模型刚失败过，自动压缩会暂时停止：

```python
_cooldown_remaining = self._summary_failure_cooldown_until - time.monotonic()
if _cooldown_remaining > 0:
    return False
```

这避免了一个很糟糕的循环：摘要模型失败，压缩没有真正降低 token；下一轮又超过阈值；系统又尝试摘要；又失败。没有冷却，CLI 或 gateway 看起来就像卡住了。

第二类是 anti-thrashing。如果最近几次压缩收益很小，也会跳过：

```python
if self._ineffective_compression_count >= 2:
    return False
```

这说明 Hermes 把压缩看成一种有成本、有风险的操作。压缩不是越频繁越好。每压一次，摘要都会丢信息；如果连续几次只省下一点点，就该提示用户 `/new` 或手动指定 focus，而不是一直自动压。

## 功能 5：压缩真正做了什么

核心方法是 `ContextCompressor.compress`：

```python
def compress(self, messages, current_tokens=None, focus_topic=None, force=False):
    messages, pruned_count = self._prune_old_tool_results(...)

    compress_start = self._protect_head_size(messages)
    compress_start = self._align_boundary_forward(messages, compress_start)

    compress_end = self._find_tail_cut_by_tokens(messages, compress_start)

    turns_to_summarize = messages[compress_start:compress_end]
    summary = self._generate_summary(turns_to_summarize, focus_topic=summary_focus_topic)
```

可以拆成五步。

### 第一步：先裁剪旧工具输出

工具结果往往是上下文里的大头。比如 `read_file` 读了几千行，`terminal` 打出长日志，`search_files` 返回大量匹配。如果这些工具结果已经很旧，不一定需要原文。

Hermes 先做一个不需要 LLM 的廉价预处理：

```python
messages, pruned_count = self._prune_old_tool_results(
    messages,
    protect_tail_count=self.protect_last_n,
    protect_tail_tokens=self.tail_token_budget,
)
```

这一步会把旧工具输出替换成占位或短摘要，减少送给 summarizer 的输入。它的价值很大：如果直接把巨大工具输出交给摘要模型，摘要调用本身也可能超上下文。

### 第二步：保护开头

`_protect_head_size` 决定开头保留多少消息。system prompt 会隐式保护，前几条用户和 assistant 交换也通常保留，因为它们包含任务起点、项目约束、用户最初意图。

但第二次、第三次压缩时，Hermes 会更激进。原因很现实：如果每次都保留开头，长会话压缩多次后，开头会越来越“贵”，真正能压缩的中间区域越来越少。

### 第三步：按 token budget 保护尾部

`_find_tail_cut_by_tokens` 从消息尾部往前走，累积 token，直到达到尾部预算：

```python
def _find_tail_cut_by_tokens(self, messages, head_end, token_budget=None):
    if token_budget is None:
        token_budget = self.tail_token_budget
    ...
    for i in range(n - 1, head_end - 1, -1):
        msg_tokens = _estimate_msg_budget_tokens(msg)
        if accumulated + msg_tokens > soft_ceiling and (n - i) >= min_tail:
            break
```

这比“保留最后 N 条消息”更可靠。最后 N 条里可能有一条巨大工具输出；也可能都是短消息。Hermes 用 token budget 做主规则，再用最小消息数做兜底。

它还会保证最近的 user message 和 assistant message 留在尾部。这样压缩后，模型不会把当前任务的最后一句用户请求总结掉，也不会突然丢掉刚才已经可见的 assistant 回复。

### 第四步：对中间窗口做结构化摘要

Hermes 不是让摘要模型随便写“上文摘要”。`_generate_summary` 要求固定结构：

```text
## Historical Task Snapshot
## Completed Actions
## Active State
## Historical In-Progress State
## Blocked
## Key Decisions
## Resolved Questions
## Historical Pending User Asks
## Relevant Files
## Historical Remaining Work
## Critical Context
```

这些标题有明显的设计意图。

`Completed Actions` 保存已经做完的事，避免模型重复劳动。

`Active State` 保存当前工作区状态，例如文件、分支、测试结果、运行进程。

`Key Decisions` 保存为什么这样做，避免后续模型只看到结论看不到取舍。

`Resolved Questions` 保存用户已经问过并得到回答的问题，避免压缩后模型又重复解释。

`Historical Pending User Asks` 和 `Historical Remaining Work` 的标题里故意带 `Historical`。这是为了提醒后续模型：这些是历史参考，不是当前指令。真正要做什么，仍然由压缩后最新的 user message 决定。

### 第五步：组装 summary、head、tail，并清理消息协议

压缩后的消息不是只有一条 summary。它通常由三部分组成：

```text
protected head
context summary
protected tail
```

Hermes 还会处理几个协议问题。

第一，summary 有明确前缀：

```python
SUMMARY_PREFIX = (
    "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted ..."
)
```

这段前缀反复强调：summary 是参考，不是当前指令；只响应 summary 之后的最新 user message。

第二，summary 末尾有结束标记：

```python
_SUMMARY_END_MARKER = (
    "--- END OF CONTEXT SUMMARY — respond to the message below, not the summary above ---"
)
```

这是为了防止模型把 summary 里的历史问题当成当前问题。

第三，压缩后会清理孤立的 tool call / tool result。因为压缩切掉中间窗口时，可能切掉 assistant tool_calls 的一边，只留下 tool result，或者反过来。很多 provider 对工具调用配对要求严格，残缺序列会导致下一次请求失败。

第四，压缩后的消息会去掉 `_db_persisted` 标记：

```python
def _strip_persistence_markers(messages):
    for msg in messages:
        if isinstance(msg, dict):
            msg.pop(_DB_PERSISTED_MARKER, None)
```

这是为了避免压缩后的新 transcript 写入 SessionDB 时被误认为“已经持久化过”。这个细节和第五讲连得很紧。

## 功能 6：摘要失败时，Hermes 怎么避免越修越坏

摘要失败是高概率事件。辅助模型可能没配置、超时、返回空内容、401/403、代理返回非 JSON、流中断。Hermes 对这些错误做了分层处理。

`_generate_summary` 成功时，会保存 `_previous_summary`，下次压缩走迭代更新：

```python
summary = redact_sensitive_text(content.strip())
self._previous_summary = summary
self._clear_compression_failure_cooldown()
return self._with_summary_prefix(summary)
```

失败时，先看能不能降级到主模型。比如配置的 summary model 不可用，但主模型还能跑，Hermes 会尝试用主模型总结。

如果还是失败，就会记录冷却：

```python
self._record_compression_failure_cooldown(_transient_cooldown, err_text)
self._last_summary_error = err_text
return None
```

`compress` 收到空 summary 后，会根据错误类型和配置决定：

```python
if not summary and (
    self.abort_on_summary_failure
    or self._last_summary_auth_failure
    or self._last_summary_network_failure
):
    self._last_compress_aborted = True
    return messages
```

这里的原则很清楚：如果是认证错误或网络中断，不应该把中间上下文丢掉再塞一个“摘要不可用”的占位。因为这不是模型能力问题，而是外部条件坏了。保留原消息，等待用户修复配置或稍后重试，损失更小。

如果配置允许 fallback，Hermes 会插入一个确定性的 fallback summary，但会记录 `_last_summary_fallback_used` 和 dropped count，让上层能提示用户。这属于“尽量继续工作”的取舍。

## 功能 7：compress_context 负责把压缩接回 Hermes runtime

`ContextCompressor.compress` 只负责把 messages 压成新的 messages。真正和 Hermes runtime 接起来的是 `agent/conversation_compression.py` 的 `compress_context`：

```python
def compress_context(agent, messages, system_message, approx_tokens=None, task_id="default", focus_topic=None, force=False):
    check_compression_model_feasibility(agent)
    in_place = bool(getattr(agent, "compression_in_place", False))
    agent._emit_status(COMPACTION_STATUS)
    ...
    compressed = agent.context_compressor.compress(...)
```

这个函数做了很多压缩器不该管的事。

### 压缩锁

它会先尝试拿 SessionDB 里的 compression lock：

```python
_lock_acquired = _lock_db.try_acquire_compression_lock(
    _lock_sid,
    _lock_holder,
    ttl_seconds=_lock_ttl,
)
```

为什么需要锁？因为同一个 session 可能被多个 agent 实例共享。比如主会话和 background review fork 同时看到上下文太长，如果它们都压缩，就可能从同一个父 session 分叉出两个 child session。gateway 只认其中一个，另一个变成孤儿。

锁的粒度是旧 session id。拿不到锁时，当前路径直接返回原 messages，让已经拿到锁的路径完成压缩。

### 压缩前通知 memory provider

压缩会把旧 transcript 从 live context 移走。移走前，Hermes 给 memory provider 一个机会：

```python
if agent._memory_manager:
    agent._memory_manager.on_pre_compress(messages)
```

后面还会调用：

```python
agent.commit_memory_session(messages)
```

这说明压缩边界也是 memory 提取边界。原始消息还在 SessionDB，但 live context 要变短，稳定信息要尽量在边界前被提炼出来。

### 重建 system prompt

压缩后，Hermes 会让 prompt builder 重新构建 system prompt：

```python
agent._invalidate_system_prompt()
new_system_prompt = agent._build_system_prompt(system_message)
agent._cached_system_prompt = new_system_prompt
```

这一步容易被忽略。第二讲讲过，system prompt 会持久化和复用，但压缩是一个明确的重建点。因为压缩可能改变 memory、context engine 状态、todo snapshot、session state，继续复用旧 system prompt 会不一致。

### 写入 SessionDB

这部分第五讲已经讲过，这里只把它放回压缩链路：

```python
if in_place:
    agent._session_db.archive_and_compact(agent.session_id, compressed)
else:
    agent._session_db.end_session(agent.session_id, "compression")
    agent.session_id = f"{...}"
    agent._session_db.create_session(
        session_id=agent.session_id,
        parent_session_id=old_session_id,
    )
```

in-place 保持同一个 session id，用 `active=0, compacted=1` 归档旧消息，再插入新 active transcript。

rotation 会结束旧 session，创建 child session。它需要迁移 goal、title、contextvar、日志 session context，并更新 `parent_session_id`。

这就是为什么压缩不是“summary = summarize(messages)”这么简单。它改变了 live context，还可能改变 session identity。

### 清理 file-read dedup

压缩后，Hermes 会清理文件读取去重缓存：

```python
from tools.file_tools import reset_file_dedup
reset_file_dedup(task_id)
```

原因很直接：压缩前模型可能已经看过某个文件，所以后续再读时工具可能返回“文件没变”的短结果。压缩后原文已经不在 live context 里，如果还返回短结果，模型就拿不到文件内容。清理 dedup 后，模型重新读文件时能拿到完整内容。

## 功能 8：自动压缩和手动 /compress 的区别

自动压缩由阈值、provider 错误、工具结果膨胀触发。用户不直接指定压缩目标。

手动 `/compress` 走 slash command 路径。gateway 里会创建临时压缩 agent，然后调用：

```python
tmp_agent._compress_context(
    head,
    "",
    approx_tokens=approx_tokens,
    focus_topic=focus_topic,
    force=True,
)
```

`force=True` 很重要。它会清掉失败冷却，让用户可以在自动压缩失败后手动重试。

Hermes 还支持 focus topic。`/compress <focus>` 会把 focus 传给 summarizer：

```python
if focus_topic:
    prompt += f"""
FOCUS TOPIC: "{focus_topic}"
This compaction should PRIORITISE preserving all information related to the focus topic above...
"""
```

这和 Claude Code 的 `/compact <focus>` 思路相近：用户告诉压缩器“这次最该保留什么”。比如长会话里既聊了架构、又修了 bug、又讨论部署，你可以让压缩重点保留“部署问题”，其它内容更激进地总结。

Hermes 还有 partial compression。`hermes_cli/partial_compress.py` 负责解析类似 `/compress here`、`/compress here 4`、`--keep 4` 的参数，把历史切成 head 和 tail：

```python
head, tail = split_history_for_partial_compress(msgs, keep_last)
compressed = rejoin_compressed_head_and_tail(compressed, tail)
```

partial compression 的意义是：用户可以决定压缩边界，保留最近几轮原文，把更早的部分压掉。这比自动压缩更像“人工整理上下文”。

## 功能 9：为什么压缩摘要要反复强调“不是当前指令”

这是 agent 压缩里最容易出事故的地方。

如果 summary 里写：

```text
用户要求修复登录 bug。下一步：修改 auth.py。
```

后续模型可能把它当成当前任务，哪怕用户最新消息已经换成“别修了，先解释架构”。这就是 stale instruction 问题。

Hermes 的 `SUMMARY_PREFIX` 用了很长一段话防止这个问题：

```text
Earlier turns were compacted into the summary below.
treat it as background reference, NOT as active instructions.
Respond ONLY to the latest user message that appears AFTER this summary.
```

而且标题也从早期的 `Active Task` 这类容易误导的名字，改成了 `Historical Task Snapshot`、`Historical Remaining Work`。这不是文案洁癖，而是模型行为控制。压缩摘要一旦被模型误读成新指令，agent 就会“复活”旧任务。

所以，好的上下文压缩不是把旧内容写短，而是要把旧内容降级成背景材料。

## 和 Codex、Claude Code 的对比

Claude Code 的 `/compact` 是开发者比较熟悉的参照物。它的核心体验是：用户可以在长会话中手动 compact，并可带 focus，让后续对话继续在压缩后的上下文上推进。Hermes 的 `/compress <focus>` 和 partial compression 明显吸收了同类思路，但 Hermes 更强调和 SessionDB、memory provider、tool call 协议、gateway session 的衔接。

Codex 这类编码 agent 也会面对长线程和上下文预算问题。它的用户体验常常是 thread 继续、任务摘要、工具轨迹和工作区状态共同维持连续性。Hermes 把这一块做得更显式：有 `ContextEngine` 抽象、有压缩阈值、有 SessionDB compression lock、有 in-place/rotation 两种持久化策略。

可以这样看：

| 问题 | Hermes 的做法 | Claude Code / Codex 常见形态 |
| --- | --- | --- |
| 手动压缩 | `/compress`，支持 focus 和 partial compression | Claude Code 常见 `/compact`，可带关注点 |
| 自动触发 | preflight、provider error、tool call 后检查 | 多数框架也会在长上下文时摘要或截断，但内部细节不一定暴露 |
| 压缩持久化 | `archive_and_compact` 或 rotation child session | 通常由宿主 thread/session 管理 |
| 工具调用协议 | 压缩后清理 orphan tool call/result | 成熟 agent 都必须处理，但 Hermes 源码里很显式 |
| 旧历史检索 | 原始消息仍进 SessionDB，可由 `session_search` 找回 | 常见是 thread 历史或 UI 检索，是否模型可调用取决于实现 |

Hermes 值得学习的地方在于，它没有把压缩当成“提示词技巧”。它把压缩做成了 agent runtime 的状态迁移：有触发条件，有消息重写，有数据库边界，有失败恢复，有用户可控入口。

## 常见问题

### 1. 压缩后模型是不是只能看到摘要？

不是。压缩后的 live context 通常包含 protected head、summary、protected tail。最近的用户请求、最近的 assistant 回复、一些尾部工具结果会尽量原样保留。被压缩掉的中间窗口变成 summary。

如果使用 in-place compaction，旧原文还在 SessionDB，只是 `active=0, compacted=1`，不会默认进入 live context，但可以被 `session_search` 找回。

### 2. 为什么不直接删除最早的消息？

直接删除最早消息会丢任务起点、关键约束和已经做过的决策。很多编码任务的“为什么这样做”就在早期对话里。

Hermes 默认保留一部分 head，再总结中间，保留尾部。这个结构比简单 FIFO 删除更适合长任务。

### 3. 压缩为什么会重建 system prompt？

因为压缩边界可能改变 memory、todo snapshot、context engine 状态和 session state。继续复用旧 system prompt，会让模型看到的系统层和新的 messages 不一致。

`compress_context` 会调用 `_invalidate_system_prompt()` 和 `_build_system_prompt()`，然后把新 prompt 写回 SessionDB。

### 4. 压缩会不会丢工具调用？

会丢一部分旧工具调用的原文，但不能留下残缺协议。Hermes 会避免切断 assistant tool_calls 和 tool results 的配对，并清理 orphan 消息。否则 provider 可能因为消息序列不合法而拒绝请求。

这就是为什么压缩不能只按 token 字符串切片。它必须理解聊天消息结构。

### 5. 压缩失败时为什么不硬着头皮继续？

因为最危险的失败不是“不压缩”，而是“摘要失败后仍然丢掉中间上下文”。如果是认证错误、网络中断或空响应，Hermes 会尽量 abort，保留原 messages，让用户修复配置或手动重试。

在可接受的情况下，它也可以插入 fallback marker，但会记录 warning。这里的取舍是：继续运行和保留真实上下文之间要优先保护后者。

## 本讲要带走的主线

Context Compression 是 Hermes 的上下文状态迁移机制。它不是单纯的 prompt 工程，也不是简单摘要。

触发层负责判断什么时候压缩：turn preflight、provider 错误恢复、tool call 后检查。

压缩层负责决定保留什么、总结什么、裁剪什么：保护 head，保护 tail，裁剪旧工具输出，总结中间窗口。

runtime 层负责把压缩接回系统：重建 system prompt，写入 SessionDB，迁移 session，通知 memory provider，清理 file dedup，释放压缩锁。

学完这一讲，你应该能回答一个核心问题：为什么 Hermes 的压缩代码横跨 `context_compressor.py`、`conversation_compression.py`、`turn_context.py`、`conversation_loop.py` 和 `hermes_state.py`？因为压缩不是一个局部文本操作，而是一次会话生命周期事件。
