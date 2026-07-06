# Prompt Builder：模型到底看到了什么

## 这篇先解决什么问题

上一课讲 `run_conversation`：Hermes 如何推进一轮对话。这一课讲它的前置问题：模型在每次 API 调用时到底看到了什么。

Agent 工程里，prompt 不是一段“系统人设”。它更像运行时合约。Hermes 要把这些内容放进模型上下文：

- Agent 身份和行为规则。
- 当前模型、平台、运行环境。
- 工具使用规则。
- Skills 索引。
- 项目上下文文件。
- 记忆和用户画像。
- 当前会话的时间、session、model/provider 信息。
- 插件和外部记忆临时注入的内容。

这些内容不能随便拼。放错位置会带来实际问题：prompt cache 失效、注入风险变高、跨 provider 兼容性变差、模型看到的上下文过期，或者长期记忆被临时任务污染。

## 功能 1：Hermes 把 system prompt 分成三层

### 先讲人话

Hermes 的 system prompt 不是一个大字符串从头写到尾。它先拆成三层，再拼起来：

```text
stable   稳定层：身份、工具规则、skills 索引、平台/环境提示
context  上下文层：用户传入的 system_message、项目上下文文件
volatile 易变层：memory、USER profile、外部记忆、日期、model/provider
```

这个分层是一个技术选型。它不是为了代码好看，而是为了控制“哪些内容可以长期稳定，哪些内容可能随项目或会话变化，哪些内容最容易变”。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| system prompt 拼装 | `agent/system_prompt.py` 中的 `build_system_prompt_parts` | 把 prompt 拆成 stable/context/volatile |
| 最终拼接 | `agent/system_prompt.py` 中的 `build_system_prompt` | 把三层拼成最终 system prompt |
| prompt 组件 | `agent/prompt_builder.py` | 提供 identity、skills、context files、environment hints 等构件 |
| 缓存恢复 | `agent/conversation_loop.py` 中的 `_restore_or_build_system_prompt` | 复用或重建系统提示 |

### 关键源码

`agent/system_prompt.py` 文件头直接说明了这个设计：

```python
"""System-prompt assembly for :class:`AIAgent`.

The agent's system prompt is built once per session and reused across all
turns -- only context compression triggers a rebuild. This keeps the
upstream prefix cache warm.

Three tiers are joined with ``\n\n``:

* ``stable``   -- identity, tool guidance, skills prompt, environment hints...
* ``context``  -- caller-supplied ``system_message`` plus context files...
* ``volatile`` -- memory snapshot, USER.md profile, timestamp/session/model...
"""
```

最后由 `build_system_prompt` 拼起来：

```python
def build_system_prompt(agent: Any, system_message: Optional[str] = None) -> str:
    parts = build_system_prompt_parts(agent, system_message=system_message)
    joined = "\n\n".join(
        p for p in (parts["stable"], parts["context"], parts["volatile"]) if p
    )
    return joined
```

这段看着简单，但背后是 Hermes prompt 体系的核心：先按稳定性分层，再一次性拼成缓存友好的前缀。

### 工程实现

三层分别解决不同问题。

| 层 | 典型内容 | 为什么放这里 |
| --- | --- | --- |
| `stable` | 身份、工具规则、Skills 索引、环境提示、平台提示、模型族操作建议 | 多数 turn 不变，适合做 prompt cache 前缀 |
| `context` | `system_message`、`.hermes.md`、`AGENTS.md`、`CLAUDE.md`、`.cursorrules` | 和项目/调用方有关，通常一个 session 内稳定 |
| `volatile` | Memory、USER profile、外部记忆、日期、session/model/provider 行 | 更容易变，但仍随本次系统提示一起缓存和持久化 |

这里有个细节：虽然叫 `volatile`，它也被拼进 `_cached_system_prompt`。Hermes 的策略不是每一轮都重渲染 volatile，而是“本 session 的 system prompt 建好后复用”。只有压缩等边界事件才重建。

这是一个取舍。优点是 prompt cache 命中更稳，缺点是 mid-session 的某些变化不会自动反映在 system prompt 里。Hermes 对这种变化更多使用 tool result、用户消息注入、session search、memory tool 等路径处理。

### 技术选型

为什么不每一轮都重新拼 system prompt？

因为大模型 provider 的 prefix cache 对长会话很重要。Hermes 的 system prompt 很重：工具规则、skills、context files、memory、平台提示都可能占很多 tokens。如果每轮都变，provider 无法复用前缀，成本和延迟都会上升。

Hermes 选择：

```text
system prompt 尽量稳定
临时上下文放到 user message 或 tool result
压缩边界才重建 system prompt
```

这就是 Prompt Builder 的第一条主线。

### 和 Codex / Claude Code 的差异

Codex 和 Claude Code 也需要项目上下文和系统规则，但它们更偏开发者工作区。Hermes 的 prompt 要覆盖更多产品场景：CLI、Gateway、cron、Kanban worker、skills、memory、插件平台、不同 provider。

所以 Hermes 的 prompt 更像“运行时协议”，不是单纯“助手人设”。这也是它更重的原因。

## 功能 2：system prompt 只构建一次，并持久化到 SessionDB

### 先讲人话

如果每个 turn 都新建一个 `AIAgent`，但 system prompt 每次都重新生成，prompt cache 就会经常错过。Gateway 场景尤其容易这样：每条平台消息都可能创建新的 agent 实例。

Hermes 的办法是：第一次构建 system prompt 后，把它存到 SessionDB。后续继续同一个 session 时，优先从数据库取出原来的 prompt，能复用就原样复用。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 恢复或构建 | `agent/conversation_loop.py` 中的 `_restore_or_build_system_prompt` | 从 SessionDB 恢复，或新建并写回 |
| 缓存字段 | `agent._cached_system_prompt` | 当前 agent 实例内复用的 system prompt |
| 持久化 | `SessionDB.update_system_prompt` | 把系统提示存到会话行 |

### 关键源码

恢复逻辑先查 SessionDB：

```python
if conversation_history and agent._session_db:
    session_row = agent._session_db.get_session(agent.session_id)
    if session_row is not None:
        raw_prompt = session_row.get("system_prompt")
        if raw_prompt:
            stored_prompt = raw_prompt
            stored_state = "present"
```

如果存储的 prompt 和当前模型/provider 匹配，就直接复用：

```python
if stored_prompt and _stored_prompt_matches_runtime(agent, stored_prompt):
    agent._cached_system_prompt = stored_prompt
    return
```

否则重新构建，并写回数据库：

```python
agent._cached_system_prompt = agent._build_system_prompt(system_message)

if agent._session_db:
    agent._session_db.update_system_prompt(
        agent.session_id,
        agent._cached_system_prompt,
    )
```

### 工程实现

这段逻辑解决了三个问题。

第一，Gateway 不必因为每次新建 agent 实例而丢掉 prompt cache。只要 session row 里有 prompt，就能恢复。

第二，系统提示和模型/provider 绑定。如果用户中途切模型，旧 prompt 里的 `Model:` 或 `Provider:` 行可能过期，Hermes 会检测并重建。

第三，持久化失败要显式 warning。因为这会导致后续 turn 反复重建 prompt，成本会上升，但用户不一定能从表面行为看出来。

### 技术选型

这里的选型是“把 system prompt 当作 session 状态”，而不是当作临时请求参数。

这很符合 Hermes 的产品目标。它不是短命脚本，而是长生命周期 Agent。一个 session 的系统约束、项目上下文、技能索引和记忆快照，是会话语义的一部分。把它落进 SessionDB，恢复时才有一致性。

## 功能 3：为什么临时上下文不进 system prompt

### 先讲人话

并不是所有上下文都应该进 system prompt。比如插件 `pre_llm_call` 临时返回的内容、外部记忆 prefetch 的内容，只对当前 turn 有意义。如果把它们拼进 system prompt，会破坏稳定前缀，也可能让临时信息变成长期系统约束。

Hermes 的做法是：临时上下文追加到当前用户消息，而不是 system prompt。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 临时上下文来源 | `agent/turn_context.py` 中 `pre_llm_call` 和 external-memory prefetch | 本轮产生的临时上下文 |
| 用户消息注入 | `agent/conversation_loop.py` 构造 `api_messages` 的循环 | 把临时上下文附加到当前 user message |
| memory block 格式 | `agent.memory_manager.build_memory_context_block` | 把外部记忆包成边界清楚的块 |

### 关键源码

`conversation_loop.py` 在构造 `api_messages` 时，只对当前用户消息注入临时上下文：

```python
if idx == current_turn_user_idx and msg.get("role") == "user":
    _injections = []
    if _ext_prefetch_cache:
        _fenced = build_memory_context_block(_ext_prefetch_cache)
        if _fenced:
            _injections.append(_fenced)
    if _plugin_user_context:
        _injections.append(_plugin_user_context)
    if _injections:
        _base = api_msg.get("content", "")
        if isinstance(_base, str):
            api_msg["content"] = _base + "\n\n" + "\n\n".join(_injections)
```

紧接着，源码注释说明为什么不放 system prompt：

```python
# Plugin context from pre_llm_call hooks is injected into the
# user message, NOT the system prompt.
# This is intentional -- system prompt modifications break the prompt
# cache prefix. The system prompt is reserved for Hermes internals.
```

### 工程实现

这个设计可以理解成两条通道：

```text
长期规则 / 会话稳定上下文 -> system prompt
当前 turn 临时材料 -> user message 附加块
```

这能避免两个常见错误。

第一，临时搜索结果或插件提示不会污染长期系统提示。第二，system prompt 的字节前缀更稳定，provider prefix cache 更容易命中。

### 技术选型

很多 Agent 框架会把所有“额外上下文”都塞进 system prompt，因为实现简单。但一旦系统变大，这个做法会有问题：

- system prompt 每轮变化，缓存失效。
- 临时信息被模型误解为长期规则。
- 插件内容权限过高，增加 prompt injection 风险。
- 调试困难，不知道某条指令到底来自系统层还是临时 recall。

Hermes 选择分通道注入，是更重但更稳的工程做法。

## 功能 4：Skills 不是直接塞全文，而是塞索引

### 先讲人话

Hermes 有 Skills 系统，但它不会把所有技能全文塞进 prompt。那会非常贵，也会干扰模型。

它在 system prompt 里放的是一个“技能索引”：有哪些技能、每个技能大概做什么、什么时候应该加载。真正需要某个 skill 时，模型再用 `skill_view` 加载。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| Skills 索引构建 | `agent/prompt_builder.py` 中的 `build_skills_system_prompt` | 扫描技能目录，生成 compact index |
| Skills 工具判断 | `agent/system_prompt.py` 中 `has_skills_tools` | 只有 skills 工具存在时才注入索引 |
| Skills 缓存 | `_SKILLS_PROMPT_CACHE` 和 `.skills_prompt_snapshot.json` | 进程内 + 磁盘双层缓存 |

### 关键源码

`build_skills_system_prompt` 的 docstring 说得很清楚：

```python
def build_skills_system_prompt(...):
    """Build a compact skill index for the system prompt.

    Two-layer cache:
      1. In-process LRU dict keyed by (skills_dir, tools, toolsets, hidden)
      2. Disk snapshot (``.skills_prompt_snapshot.json``) validated by
         mtime/size manifest -- survives process restarts

    Falls back to a full filesystem scan when both layers miss.
    """
```

`agent/system_prompt.py` 里只有在 skills 工具可用时才注入：

```python
has_skills_tools = any(
    name in agent.valid_tool_names
    for name in ["skills_list", "skill_view", "skill_manage"]
)

if has_skills_tools:
    skills_prompt = _r.build_skills_system_prompt(
        available_tools=agent.valid_tool_names,
        available_toolsets=avail_toolsets,
        compact_categories=_compact_cats or None,
    )
```

### 工程实现

Skills prompt 的技术选型是“索引优先，按需加载”。

它解决的问题很实际：

- 技能可能很多，不能全量塞进上下文。
- 技能是否可用和当前 toolsets 有关。
- Gateway 不同平台可能禁用不同技能。
- 外部技能目录也要参与索引。
- 扫描技能目录不能每次都冷启动。

所以 Hermes 做了两层缓存：

```text
进程内 LRU cache：同一进程内快速复用
磁盘 snapshot：进程重启后也不用每次全量解析
```

并且 cache key 里包含 available tools、toolsets、platform、disabled skills、compact categories。这个细节很重要：skills 索引不是固定文本，它依赖当前 agent 的工具能力和平台。

### 和 Codex / Claude Code 的差异

Codex 和 Claude Code 更常见的是“项目指令 + 工具能力 + 当前任务上下文”。Hermes 的 Skills 更像 procedural memory：不是一次性的项目规则，而是可被创建、修补、归档、复用的工作流知识。

所以 Hermes 不能只靠静态 instruction 文件。它需要一个索引，让模型知道“有这些可加载技能”，但又不能把所有技能全文塞爆上下文。

## 功能 5：项目上下文文件有优先级和安全扫描

### 先讲人话

项目上下文文件很有用，但也危险。比如 `AGENTS.md`、`CLAUDE.md`、`.cursorrules` 可能来自仓库。它们会进入 system prompt，权限很高。如果文件里有 prompt injection，模型会很容易被带偏。

Hermes 的做法是：

- 只加载一类项目上下文，按优先级 first found wins。
- 加载前做 threat scan。
- 内容过长时截断，并提示需要完整内容时用工具读取。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 上下文文件加载 | `agent/prompt_builder.py` 中的 `build_context_files_prompt` | 发现并加载项目上下文 |
| 注入扫描 | `_scan_context_content` | 检测 prompt injection / promptware |
| 文件优先级 | `_load_hermes_md`、`_load_agents_md`、`_load_claude_md`、`_load_cursorrules` | 按顺序加载 |
| 内容截断 | `_truncate_content` | 控制 context file 进入 prompt 的大小 |

### 关键源码

优先级写在 `build_context_files_prompt` 里：

```python
project_context = (
    _load_hermes_md(cwd_path, context_length)
    or _load_agents_md(cwd_path, context_length)
    or _load_claude_md(cwd_path, context_length)
    or _load_cursorrules(cwd_path, context_length)
)
```

扫描逻辑在 `_scan_context_content`：

```python
def _scan_context_content(content: str, filename: str) -> str:
    findings = _scan_for_threats(content, scope="context")
    if findings:
        return (
            f"[BLOCKED: {filename} contained potential prompt injection "
            f"({', '.join(findings)}). Content not loaded.]"
        )
    return content
```

### 工程实现

这个设计体现了几个取舍。

第一，为什么 first found wins？因为多个项目规则文件同时生效会互相冲突。Hermes 优先 `.hermes.md / HERMES.md`，再到 `AGENTS.md`，再到 `CLAUDE.md`，最后 `.cursorrules`。这让 Hermes 自己的项目指令优先。

第二，为什么要扫描？因为这些文件可能来自不可信仓库，而它们会被放进 system prompt。普通文件内容出现在 tool result 里还可以隔离，但 context file 是高权限注入点，必须更保守。

第三，为什么要截断？因为上下文文件和 system prompt、工具定义、skills、memory 都共享 token 预算。把超长规则文件完整塞进去，会挤掉当前任务真正需要的内容。

## 功能 6：Memory 在 prompt 里，但不是所有记忆都应该进 prompt

### 先讲人话

Hermes 会把长期记忆和用户画像放进 system prompt，但它对 memory 的定位很清楚：Memory 不是任务日志，也不是 TODO，也不是“我刚刚做完了什么”。

Memory 应该保存稳定、可复用、未来仍有价值的信息。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| Memory 指导 | `agent/prompt_builder.py` 中的 `MEMORY_GUIDANCE` | 告诉模型什么该存、什么不该存 |
| Memory 注入 | `agent/system_prompt.py` 中 volatile tier | 把 memory 和 USER profile 格式化进 system prompt |
| 外部 Memory | `agent._memory_manager.build_system_prompt()` | 外部记忆 provider 的 system prompt block |

### 关键源码

Memory guidance 明确排除了任务进度：

```python
MEMORY_GUIDANCE = (
    "You have persistent memory across sessions. Save durable facts using the memory "
    "tool: user preferences, environment details, tool quirks, and stable conventions. "
    ...
    "Do NOT save task progress, session outcomes, completed-work logs, or temporary TODO "
    "state to memory; use session_search to recall those from past transcripts. "
)
```

Memory 注入在 volatile tier：

```python
if agent._memory_store:
    if agent._memory_enabled:
        mem_block = agent._memory_store.format_for_system_prompt("memory")
        if mem_block:
            volatile_parts.append(mem_block)

    if agent._user_profile_enabled:
        user_block = agent._memory_store.format_for_system_prompt("user")
        if user_block:
            volatile_parts.append(user_block)
```

### 工程实现

这里要区分三个概念：

| 概念 | 作用 |
| --- | --- |
| Memory | 长期事实和偏好 |
| Session history | 过去对话和任务日志 |
| Session search | 需要过去任务细节时再查 |

这就是为什么 `MEMORY_GUIDANCE` 明确说 PR 号、issue 号、commit SHA、完成了某阶段这类信息不该进 memory。这些东西容易过期，应该留在 session history 里，需要时用 `session_search` 查。

### 技术选型

Hermes 没把所有历史都当 memory，是对的。长期记忆如果失控，会污染每一轮 prompt。一个坏 memory 比一条坏历史消息更危险，因为它会长期影响模型行为。

所以 Hermes 采用：

```text
稳定偏好/事实 -> memory
过去任务细节 -> session_search
具体流程经验 -> skill
```

这个边界后面讲 Memory 和 Skills 时还会继续展开。

## 本篇要记住的东西

1. Prompt Builder 的核心不是“写提示词”，而是管理上下文的权限、稳定性和成本。
2. Hermes 把 system prompt 分成 `stable / context / volatile` 三层。
3. system prompt 建好后会缓存并持久化到 SessionDB，目的是复用 prefix cache。
4. 临时上下文不进 system prompt，而是附加到当前 user message，避免污染稳定前缀。
5. Skills 进 prompt 的是索引，不是全文；需要时再用 `skill_view` 加载。
6. 项目上下文文件进入 system prompt 前要做优先级选择、截断和 prompt injection 扫描。
7. Memory 只应该放稳定长期信息，任务过程要靠 session history / session_search。

## QA

### 为什么 Hermes 不每轮重新生成 system prompt？

因为 system prompt 很重，而且 provider 的 prefix cache 依赖前缀稳定。每轮重新生成，哪怕只改了时间或临时上下文，也可能让缓存失效。Hermes 选择把 system prompt 作为 session 状态缓存并持久化，换取更稳定的成本和延迟表现。

### 为什么插件和外部记忆临时内容要放进 user message？

因为这些内容只对当前 turn 有意义。如果放进 system prompt，会把临时材料提升成高权限长期规则，还会破坏 prompt cache。放进当前 user message，既能让模型看到，又不会污染系统层。

### 为什么 Skills 只放索引？

Skills 是 procedural memory，数量可能很多。如果全量塞进 prompt，会浪费 token，也会让模型被无关技能干扰。索引让模型知道有哪些技能可用，真正需要时再调用 `skill_view` 读取完整内容。

### `AGENTS.md`、`CLAUDE.md` 这类文件为什么要扫描？

它们会进入 system prompt，权限比普通文件内容高。如果仓库里有人放了 prompt injection，模型可能把它当作系统规则。Hermes 在加载前做 threat scan，是把这类文件当作半可信输入处理。

### Memory 和 Skill 的边界是什么？

Memory 保存稳定事实和偏好，Skill 保存可复用流程。比如“用户喜欢中文技术讲解”适合 memory；“这个项目发布前要跑哪些命令”更适合 skill。任务日志、PR 号、commit SHA 不适合 memory，应该靠 session history 和 session_search。
