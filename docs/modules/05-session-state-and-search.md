# SessionDB 与 session_search：Hermes 怎么记住一次会话

## 目标与范围

本章分析一轮对话产生的消息怎样持久化，以及后续 turn、resume 和 `session_search` 怎样重新取得这些上下文。

首先区分三个概念：

| 概念 | 它保存什么 | 典型用途 |
| --- | --- | --- |
| `SessionDB` | 原始会话记录、消息、工具调用、模型配置、system prompt、压缩关系 | 恢复会话、WebUI 展示历史、跨会话检索 |
| `session_search` | 一个工具，从 `SessionDB` 里检索过去的会话消息 | 用户提到“上次”“之前做过”，模型需要找旧记录 |
| Memory | 从对话中提炼出来的稳定事实或偏好 | 长期个性化、跨会话保留较稳定的信息 |

所以，`SessionDB` 不是 memory。它更像 Hermes 的会话账本。Memory 是提炼后的事实，`session_search` 是翻账本的工具。

## 功能 1：SessionDB 是 Hermes 的持久化会话层

Hermes 没有把会话历史写成一组 JSONL 文件，而是在 `hermes_state.py` 中实现 SQLite 会话库。SQLite 不需要单独部署服务，FTS5 提供全文检索，WAL 模式适合“多个读者、一个写者”的桌面与 Gateway 场景。

源码入口在 `hermes_state.py`：

```python
class SessionDB:
    """
    SQLite-backed session storage with FTS5 search.

    Thread-safe for the common gateway pattern (multiple reader threads,
    single writer via WAL mode). Each method opens its own cursor.
    """
```

这段注释已经暴露了它的定位：不是缓存，也不是 prompt builder 的附属品，而是一个持久化状态库。

### sessions 表保存会话级元数据

`sessions` 表记录的是“这次会话是谁、从哪里来、用什么模型、有没有父会话、system prompt 是什么”。这些字段看起来杂，但它们解决的是同一个问题：恢复会话时不能只恢复消息，还要恢复消息所在的运行环境。

恢复链重点依赖以下字段：

| 字段 | 工程意义 |
| --- | --- |
| `id` | 当前会话的主键，工具、gateway、WebUI 都围绕它定位会话 |
| `source` | 会话来源，例如 CLI、gateway、cron、subagent，用来过滤和排序历史 |
| `model` / `model_config` | 恢复时判断模型配置是否一致，也能保存分支、代理、压缩相关标记 |
| `system_prompt` | 第一次构建后的系统提示词快照，后续恢复同一 session 时可以复用 |
| `parent_session_id` | 压缩续接、分支、子代理等场景的父子关系 |
| `message_count` / `tool_call_count` | 会话摘要级统计，避免每次都扫描 messages 表 |
| `cwd` / `git_branch` / `git_repo_root` | 让会话和项目工作区绑定，便于恢复上下文 |

`parent_session_id` 用于处理压缩后的续接、分支会话和子代理会话。`resolve_resume_session_id` 不能无条件沿父子关系向下查找，否则恢复目标可能落到不相关的 subagent 会话。

### messages 表保存真实消息流

`messages` 表才是对话内容主体。它保存用户消息、assistant 消息、tool result，还会保存工具调用相关字段。

关键字段大致是这些：

| 字段 | 工程意义 |
| --- | --- |
| `role` | `user`、`assistant`、`tool` 等消息角色 |
| `content` | 文本内容，也可能是序列化后的多模态内容 |
| `tool_calls` | assistant 发起的工具调用 JSON |
| `tool_call_id` | tool result 和 assistant tool call 的配对 ID |
| `tool_name` | 工具结果来自哪个工具 |
| `reasoning*` / `codex_*` | 保存部分模型或运行时的 reasoning / message item 结构 |
| `active` | 当前是否进入 live context |
| `compacted` | 是否是被压缩归档的旧消息 |

`active` 和 `compacted` 是理解压缩的钥匙。live context 默认只加载 `active=1` 的消息；但被压缩归档的旧消息会变成 `active=0, compacted=1`，它们不会再喂给模型，却仍然可以被 `session_search` 找到。

## 功能 2：一条消息是怎么写进 SessionDB 的

从运行时看，Hermes 的对话消息先在内存里的 `messages` / `conversation_history` 中流转。到合适时机，agent 会把还没有入库的消息刷进 `SessionDB`。

入口在 `run_agent.py`：

```python
def _flush_messages_to_session_db(
    self,
    messages: List[Dict],
    conversation_history: List[Dict] = None,
):
    ...
    if msg.get(_DB_PERSISTED_MARKER):
        ...
    self._session_db.append_message(...)
    msg[_DB_PERSISTED_MARKER] = True
```

这里的 `_DB_PERSISTED_MARKER` 是去重标记。Hermes 的消息会在多个地方被观察和刷库：主循环、工具执行、Codex runtime、压缩逻辑都可能触发持久化。如果没有这个标记，同一条用户消息或 assistant tool call 很容易重复写入。

真正落库的是 `hermes_state.py` 的 `append_message`：

```python
def append_message(
    self,
    session_id: str,
    role: str,
    content: str = None,
    tool_name: str = None,
    tool_calls: Any = None,
    tool_call_id: str = None,
    ...
) -> int:
    tool_calls_json = json.dumps(tool_calls) if tool_calls else None
    stored_content = self._encode_content(content)

    cursor = conn.execute(
        """INSERT INTO messages (session_id, role, content, tool_call_id,
           tool_calls, tool_name, timestamp, ...)
           VALUES (?, ?, ?, ?, ?, ?, ?, ...)""",
        (...),
    )
```

这段代码做了几件重要的小事：

第一，结构化字段先序列化。`tool_calls`、reasoning details、多模态 content 都不能原样塞进 SQLite，所以要变成 JSON。

第二，tool call 和 tool result 都会落库。assistant 带 `tool_calls` 的消息必须和后续 `role="tool"` 结果配对。SessionDB 保存这对关系，恢复会话时才能重新构造 Provider 接受的消息序列。

第三，写入以后会更新 `message_count` 和 `tool_call_count`。这不是为了好看，而是为了列表页、检索页、恢复判断不用反复聚合 messages 表。

一个细节很值得学：Hermes 会跳过一些临时脚手架消息。比如恢复用的提示、内部修复用的临时内容，不应该成为用户真实会话历史。否则下次恢复时，模型会看到一堆“运行时为自己准备的说明”，上下文会被污染。

## 功能 3：“继续”到底由谁管理

在 Codex 里经常说“继续”，Hermes 里这个上下文管理在哪里？

这里要分两层。

如果用户在当前会话里直接说“继续”，这只是一个普通 user message。它会被追加到当前内存消息流里，然后模型根据当前 live context 继续回答。这个场景主要由 agent loop 和 prompt builder 管。

如果用户关闭后重新打开，或者用 `--resume`、WebUI、gateway 恢复一个旧会话，那就进入 `SessionDB` 的职责范围。Hermes 会从数据库加载历史消息，重新组装成 provider 能接受的 conversation 格式。

源码在 `hermes_state.py`：

```python
def get_messages_as_conversation(
    self,
    session_id: str,
    include_ancestors: bool = False,
    include_inactive: bool = False,
) -> List[Dict[str, Any]]:
    rows = self._conn.execute(
        "SELECT role, content, tool_call_id, tool_calls, tool_name, ..."
        "FROM messages WHERE session_id IN (...)"
        f"{active_clause} ORDER BY id",
        tuple(session_ids),
    ).fetchall()
```

注意最后的 `ORDER BY id`。它没有按 timestamp 排序。

这不是随手写的。时间戳可能因为系统时间跳变、虚拟机恢复、NTP 调整而不单调。如果按 timestamp 排序，assistant 的 `tool_calls` 可能被排到对应 tool result 后面。对很多 provider 来说，这会直接导致下一次 API 请求失败。自增 ID 更接近真实插入顺序，所以 Hermes 用它恢复消息序列。

恢复时，Hermes 还会把这些字段还原出来：

```python
if row["tool_call_id"]:
    msg["tool_call_id"] = row["tool_call_id"]
if row["tool_name"]:
    msg["tool_name"] = row["tool_name"]
if row["tool_calls"]:
    msg["tool_calls"] = json.loads(row["tool_calls"])
```

这说明恢复会话不是“把文本拼回去”。它要恢复的是一条可继续运行的消息协议流。工具调用 ID、工具名、assistant tool_calls 的 JSON 结构都要保留。

## 功能 4：压缩以后，恢复应该恢复到哪里

长对话会超出上下文窗口。Hermes 的上下文压缩会把旧消息总结成更短的 compacted transcript。问题来了：压缩之后，原始消息应该消失吗？

Hermes 的答案是：不能直接消失。

`agent/conversation_compression.py` 里有两种模式。

第一种是 in-place compaction。会话 ID 不变，旧 active 消息软归档，新 compacted 消息作为 active 消息插入：

```python
if in_place:
    agent.commit_memory_session(messages)
    agent._session_db.archive_and_compact(agent.session_id, compressed)
    agent._flushed_db_message_ids = set()
    compacted_in_place = True
```

对应的数据库实现是 `archive_and_compact`：

```python
def archive_and_compact(self, session_id, compacted_messages):
    conn.execute(
        "UPDATE messages SET active = 0, compacted = 1 "
        "WHERE session_id = ? AND active = 1",
        (session_id,),
    )
    inserted, tool_calls_total = self._insert_message_rows(
        conn, session_id, compacted_messages
    )
```

这是一种非破坏式压缩。模型恢复 live context 时只看到 `active=1` 的 compacted 消息；但原始旧消息仍在数据库里，仍能被 `session_search` 检索。

第二种是 rotation。旧 session 被结束，新建一个 child session，`parent_session_id` 指向旧 session：

```python
agent._session_db.end_session(agent.session_id, "compression")
old_session_id = agent.session_id
agent.session_id = f"{...}"
agent._session_db.create_session(
    session_id=agent.session_id,
    ...,
    parent_session_id=old_session_id,
)
```

rotation 的好处是边界清楚：压缩前后是两个 session。代价是恢复时要知道“真正的续接点”在哪个 child session。

这就是 `resolve_resume_session_id` 的作用：

```python
def resolve_resume_session_id(self, session_id: str) -> str:
    tip = self.get_compression_tip(session_id)
    if tip and tip != session_id:
        session_id = tip

    ...
    child_row = self._conn.execute(
        "SELECT id FROM sessions "
        "WHERE parent_session_id = ? "
        "  AND json_extract(..., '$._branched_from') IS NULL "
        "  AND json_extract(..., '$._delegate_from') IS NULL "
        "  AND COALESCE(source, '') != 'tool' "
        "ORDER BY started_at DESC, id DESC LIMIT 1",
        (current,),
    ).fetchone()
```

这里有一个很细的判断：它会排除 branch、delegate/subagent、tool child。因为这些 session 也可能有 `parent_session_id`，但它们不是“压缩后的继续会话”。如果恢复时误入 subagent session，用户看到的就不是自己原来那条主线。

所以，“继续”这件事在 Hermes 里不是一句提示词能解决的。它依赖三个层次：

| 场景 | 负责模块 |
| --- | --- |
| 当前窗口里用户说“继续” | 当前内存消息流和 agent loop |
| 重新打开旧会话继续 | `SessionDB.get_messages_as_conversation` |
| 压缩后继续 | `resolve_resume_session_id` 和 `parent_session_id` 规则 |

## 功能 5：FTS5 索引让旧会话可以被检索

`session_search` 之所以不是全表扫，是因为 `messages` 写入后会被 FTS5 索引。

源码在 `hermes_state.py`：

```python
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content
);

CREATE TRIGGER IF NOT EXISTS messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (
        new.id,
        COALESCE(new.content, '') || ' ' ||
        COALESCE(new.tool_name, '') || ' ' ||
        COALESCE(new.tool_calls, '')
    );
END;
```

这段设计有两个点。

第一，FTS 表只索引一个 `content` 字段，但它拼入了 `content`、`tool_name` 和 `tool_calls`。搜索因此同时覆盖用户文本、assistant 文本、工具名和工具调用参数。过去使用某个工具修改文件的会话，也可能通过工具调用内容被检索出来。

第二，索引更新靠 trigger。`messages` 插入、删除、更新时，FTS 表自动同步。业务代码不需要每次写消息后再手动维护搜索索引，少一个出错点。

Hermes 还额外建了一个 trigram FTS 表：

```python
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts_trigram USING fts5(
    content,
    tokenize='trigram'
);
```

这是为中文、日文、泰文等脚本准备的。默认 `unicode61` 分词器对 CJK 不友好，短语搜索容易被拆坏。trigram 把文本切成重叠片段，更适合做中文子串检索。这个细节直接影响中文会话，因为旧记录检索经常使用中文关键词。

真正搜索在 `search_messages`：

```python
def search_messages(
    self,
    query: str,
    source_filter: List[str] = None,
    role_filter: List[str] = None,
    ...
    include_inactive: bool = False,
):
    if not self._fts_enabled:
        return []

    where_clauses = ["messages_fts MATCH ?"]
    if not include_inactive:
        where_clauses.append("(m.active = 1 OR m.compacted = 1)")
```

这里再次出现了 `active` 和 `compacted`。默认搜索会包含两类消息：

1. `active=1`：当前 live context 中还存在的消息。
2. `compacted=1`：已经被压缩移出 live context，但仍属于历史记录的消息。

它会排除 `active=0, compacted=0` 的消息。这通常表示用户撤回、rewind 或不应再被使用的历史。Hermes 因此区分“被总结掉”和“被拿掉”：前者仍可检索，后者默认不参与检索。

## 功能 6：session_search 是给模型用的翻历史工具

`session_search` 定义在 `tools/session_search_tool.py`。它不是一个普通搜索接口，而是注册进 Hermes 工具系统的工具。模型可以在合适时机调用它。

注册代码很直接：

```python
registry.register(
    name="session_search",
    toolset="session_search",
    schema=SESSION_SEARCH_SCHEMA,
    handler=lambda **kwargs: session_search(**kwargs),
    check_fn=check_session_search_requirements,
)
```

`check_fn` 会检查本地状态库目录是否存在。也就是说，工具可见性不是只看 schema，还要看运行环境是否满足要求。

### session_search 的四种调用形态

`session_search` 没有显式 `mode` 参数，而是根据传参推断模式：

```python
if session_id and around_message_id is not None:
    return _scroll(...)

if session_id:
    return _read_session(...)

if not query:
    return _list_recent_sessions(...)

return _discover(...)
```

四种形态分别是：

| 形态 | 参数 | 用途 |
| --- | --- | --- |
| discovery | `query` | 用关键词找过去相关会话 |
| scroll | `session_id + around_message_id` | 围绕某条命中消息向前后翻更多上下文 |
| read | `session_id` | 读取指定 session 的内容 |
| browse | 无参数 | 查看最近会话 |

这种设计对模型比较友好。模型不需要先决定一个枚举 mode，只要按自己掌握的信息填参数。知道关键词就 discovery，知道 session id 就 read，知道消息锚点就 scroll。

### discovery 为什么不直接返回整段会话

`_discover` 的核心流程是：

```python
raw_results = db.search_messages(
    query=query,
    role_filter=role_list,
    exclude_sources=list(_HIDDEN_SESSION_SOURCES),
    limit=_DISCOVER_SCAN_LIMIT,
    sort=sort,
)

raw_results = _order_for_recall(raw_results)
...
view = db.get_anchored_view(hit_sid, msg_id, window=5, bookend=3)
```

它没有把整段会话一次性塞回来，而是返回三块内容：

| 返回内容 | 为什么需要 |
| --- | --- |
| `snippet` | 告诉模型关键词命中了哪里 |
| `messages` | 命中点前后几条消息，恢复局部语境 |
| `bookend_start` / `bookend_end` | 会话开头和结尾，帮助判断目标和结论 |

这个设计很实用。长会话可能几百上千条消息，直接读全量会浪费上下文，也容易把模型带偏。命中窗口加首尾摘要，能让模型先判断“这是不是我要找的那段”。如果还不够，再用 scroll 继续翻。

### 为什么要按 lineage 去重

`_discover` 里有一段：

```python
resolved_sid = _resolve_to_parent(db, raw_sid)
if resolved_sid not in seen_sessions:
    row = dict(r)
    row["_lineage_root"] = resolved_sid
    seen_sessions[resolved_sid] = row
```

同一条长会话经过压缩、rotation、续接后，可能形成多个 session row。如果不按 lineage 去重，搜索结果前几项可能全是同一条会话的不同片段。Hermes 把它们归到同一个 lineage root，优先给模型更多不同会话的候选。

### 为什么要降低 cron 的排序

`tools/session_search_tool.py` 里还有一个容易被忽略的设计：

```python
_DEMOTED_SESSION_SOURCES = ("cron",)

def _order_for_recall(raw_results):
    return sorted(
        raw_results,
        key=lambda r: 1 if (r.get("source") or "") in _DEMOTED_SESSION_SOURCES else 0,
    )
```

cron 这类自动任务会产生大量重复词汇。如果完全按 BM25 排序，它们可能淹没用户真实交互会话。Hermes 没有把 cron 排除，因为自动任务历史有时也有价值；它只是把 cron 降权，让交互会话先出现。这是检索系统里很典型的工程权衡：不是所有匹配都同等重要。

### source-first 限制

`SESSION_SEARCH_SCHEMA` 里明确写了一个限制：`session_search` 搜的是 Hermes 对话历史，不是外部事实来源。

如果用户给出文件、URL、GitHub issue 或应用线程，Agent 应优先检查原始来源，而不是只搜索历史。历史只能说明过去讨论过什么，不能证明外部对象的当前状态。

这个边界非常重要。很多 agent 出错，不是因为不会检索，而是把旧对话当成现实世界的证据。

## 功能 7：SessionDB、Memory、Prompt Builder 的边界

Prompt Builder 与 SessionDB 之间的边界可以整理为：

Prompt Builder 负责构建模型当前能看到的 system prompt。它会组合系统规则、项目上下文、用户 profile、memory、日期、模型信息、工具说明等内容。

SessionDB 保存的是历史会话。它不等于每次都被完整塞进 prompt。只有恢复会话时，历史消息才会通过 `get_messages_as_conversation` 回到 conversation；只有模型主动调用 `session_search` 时，旧会话检索结果才会作为 tool result 回到当前上下文。

Memory 则是另一条线。压缩前，`conversation_compression.py` 会调用：

```python
agent.commit_memory_session(messages)
```

这说明 Hermes 会在旧 transcript 被总结前尝试提取长期记忆。这里的含义是：原始历史进 SessionDB，稳定事实进 Memory，压缩摘要进入 live context。三者都和“记住”有关，但生命周期和用途不同。

可以用一条链路记住：

```text
当前消息流
  -> _flush_messages_to_session_db
  -> messages 表和 FTS 索引
  -> 恢复时 get_messages_as_conversation
  -> 检索时 session_search

压缩边界
  -> commit_memory_session 提炼长期事实
  -> archive_and_compact 或 rotation
  -> live context 变短，原始历史仍可找回
```

## 和 Codex、Claude Code 的对比

这里不讨论它们的内部私有实现，只比较公开可观察的工程形态。

Codex 这类编码 agent 很重视 thread、工作区和工具执行轨迹。用户在同一线程里说“继续”，通常依赖当前线程上下文和宿主应用保存的历史状态。它的重点是让开发任务在一个 thread 里连续推进，工具执行结果、文件变更、终端输出都会成为协作上下文的一部分。

Hermes 的特色是把会话历史做成一个模型可调用的工具：`session_search`。这让“找过去说过什么”从宿主 UI 能力变成 agent runtime 的一部分。模型可以在当前推理中决定是否检索旧会话，检索结果也会以 tool result 的形式进入消息流。

Claude Code 也有项目上下文和会话恢复能力，用户常见的 `CLAUDE.md` 更接近项目级指导文件。它解决的是“这个项目里 agent 应该遵守什么约定”。Hermes 的 `system_prompt` 持久化和 prompt builder 分层也能覆盖类似需求，但它更强调把运行时上下文、memory、工具说明、session state 分层组装，而不是只依赖一个项目说明文件。

可以粗略这样比较：

| 问题 | Hermes 的做法 | Codex / Claude Code 常见形态 |
| --- | --- | --- |
| 项目指导 | prompt builder 分层合成，可能包含项目上下文文件和用户配置 | Codex 有线程和工作区上下文；Claude Code 常见 `CLAUDE.md` |
| 会话恢复 | `SessionDB` 还原消息协议流 | 通常由宿主 thread / session 机制恢复 |
| 查找旧对话 | `session_search` 作为工具暴露给模型 | 更多依赖 UI、线程历史或宿主检索能力 |
| 压缩后保留旧记录 | `active/compacted` 和 lineage 规则 | 常见做法是 compact / summarize，但实现细节不一定暴露给模型 |

Hermes 这个设计对学习 agent runtime 很有价值。它把“上下文太长怎么办”“历史怎么恢复”“旧会话怎么检索”“压缩后原文怎么保留”放在同一个持久化层里处理，而不是每个功能各写一套临时逻辑。

## 常见问题

### 1. 当前 session 的模型能不能看见过去的全部 SessionDB？

不能。模型默认只能看见当前 prompt 和当前 conversation 里已经放进去的内容。SessionDB 只是本地数据库，不会自动全部进入上下文。

模型能看到旧会话，通常有两种方式：

第一，恢复旧 session 时，Hermes 用 `get_messages_as_conversation` 把该 session 的消息重新加载进 conversation。

第二，模型调用 `session_search`，搜索结果作为 tool result 回到当前上下文。

所以“能不能看见”不是由 SQLite 决定的，而是由恢复流程、工具可见性、模型是否调用工具，以及 tool result 是否被追加进消息流共同决定的。

### 2. 模型怎么知道什么时候该调用 session_search？

不是完全靠“模型聪明”，也不是 Hermes 写了一个硬编码规则：只要用户说“上次”就自动搜索。Hermes 的做法更接近现代 agent runtime 的常见模式：runtime 把工具能力和使用说明交给模型，模型决定是否发起 tool call，runtime 再负责校验和执行。

流程可以拆成这样：

```text
工具注册
  -> toolset 和 check_fn 过滤出当前可用工具
  -> session_search 的 schema 进入本轮 tools 列表
  -> provider 请求携带 tools schema
  -> 模型根据用户问题和工具描述决定是否调用
  -> Hermes 校验 tool call
  -> 执行 session_search
  -> 检索结果作为 tool result 回到当前消息流
```

`tools/session_search_tool.py` 里的 schema 描述不只是参数说明，它还写了使用边界：这个工具搜索的是 Hermes 会话历史；如果用户给了文件、URL、issue、应用线程等直接来源，应该先看原始来源，再把历史记录当作辅助上下文。模型能不能想到调用它，很大程度上取决于这段 schema 描述是否清楚。

所以这里的责任是分开的：

| 角色 | 负责什么 |
| --- | --- |
| Tool Registry | 决定 `session_search` 当前是否可见、是否满足运行条件 |
| Tool Schema | 告诉模型这个工具能做什么、什么时候该用、有哪些边界 |
| 模型 | 根据当前用户问题决定是否发起 tool call |
| Tool Dispatch | 校验工具名、参数和权限，然后执行工具 |
| Conversation Loop | 把 tool result 放回消息流，让模型继续回答 |

这也是为什么 `session_search` 不会自动把过去所有历史塞进 prompt。Hermes 给模型一把“查旧会话”的钥匙，但不替模型每次都打开这扇门。这样做的好处是上下文更干净；代价是如果模型判断失误，它可能该查时没查，或者不该查时查了。Hermes 用 schema、source-first 限制、当前 session lineage 排除、toolset 过滤来降低这种错误，但最后的选择仍然是模型行为的一部分。

### 3. 压缩后的旧消息为什么还能搜到？

因为 Hermes 的 in-place compaction 不会删除旧消息，而是把旧消息标成 `active=0, compacted=1`。live context 默认只加载 `active=1`，所以模型不会继续背着完整旧 transcript。但 `search_messages` 默认包含 `compacted=1`，所以 `session_search` 仍然能找回压缩前的原文片段。

这比直接覆盖旧消息更稳。直接覆盖会让检索只能搜到摘要，细节丢了；完全不压缩又会让 live context 太长。Hermes 选择了“模型上下文变短，数据库历史保留”。

### 4. session_search 算不算 memory？

不算。`session_search` 是检索过去对话的工具，返回的是历史消息。Memory 是从对话中提炼出来、准备长期复用的事实或偏好。

例如，用户曾在某次会话中说“我喜欢用中文写学习笔记”。原文保存在 SessionDB；Hermes 将它提炼为长期偏好后，才会进入 Memory。后续模型可以直接读取该偏好，也可以通过 `session_search` 找回原始上下文。

### 5. 为什么不把所有历史都塞进 prompt？

因为上下文窗口有限，而且旧历史不一定相关。把所有历史塞进去会增加 token 成本，也会让模型被无关旧任务干扰。

Hermes 的策略是：当前任务需要的 live context 放在 conversation；稳定偏好放进 memory；不确定是否相关的旧会话，通过 `session_search` 按需召回。这比“越多越好”的上下文策略更可控。

## 实现要点

SessionDB 是 Hermes 的会话事实来源。它保存原始消息、工具调用、模型配置、system prompt 和压缩关系。

恢复会话时，Hermes 不是拼接文本，而是还原 provider 能继续接受的消息协议流。工具调用 ID、工具名、tool_calls JSON、消息顺序都必须保留。

压缩不是删除历史。Hermes 通过 `active/compacted` 和 `parent_session_id` 区分 live context、归档历史和续接会话。

`session_search` 是模型可调用的跨会话检索工具。它通过 FTS5 和 trigram 索引查找历史消息，再用命中窗口、首尾 bookends、lineage 去重控制返回内容。

Hermes 的上下文管理不只存在于 Prompt Builder 或 Agent Loop。内存消息流负责当前 turn，SessionDB 保存可恢复事实，压缩机制控制 live context，`session_search` 按需召回旧记录。这四部分共同决定“继续”时模型实际取得哪些上下文。
