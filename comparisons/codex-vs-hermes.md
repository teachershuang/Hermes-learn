# 第 16 讲：Codex 与 Hermes 的工程架构对比

比较 Agent 框架最容易犯的错误，是拿功能清单排高低：谁有 memory，谁有 MCP，谁能调用终端，谁就更“完整”。这种比较解释不了源码，也无法指导自己的工程设计。

这一讲换一个角度：Codex 和 Hermes 分别把哪些问题放进核心运行时？它们如何表示一次用户输入、一次 turn、一次工具调用和一次审批？为什么两边都有 Agent loop、skills、memory、MCP、subagent，源码组织却明显不同？

本文基于以下源码快照：

- `NousResearch/hermes-agent@590a19332e898fc9bda55a31999926572d8fbc26`
- `openai/codex@2f7d89b1419bf7064346855b0acde23514b1ebc5`

Codex 变化很快，版本号必须保留。本文讨论的是这个快照中能从源码和官方文档验证的实现，不把旧版本印象当作当前结论。

## 这篇先解决什么问题

读完这一讲，应该能回答这些问题：

- Hermes 的 `AIAgent.run_conversation` 和 Codex 的 `Submission -> Event` 有什么结构差异。
- Codex 的 thread/turn 与 Hermes 的 session/conversation 是不是同一组概念。
- 两边都支持 tool call，为什么 Codex 使用强类型 Router/Registry，Hermes 使用动态注册和 toolset。
- Codex 的 OS sandbox 与 Hermes 的危险命令审批、Docker backend 是什么关系。
- 两边怎样加载 `AGENTS.md`，为什么不能只说“都是把文件拼进 prompt”。
- Codex 的 rollout JSONL + SQLite 索引与 Hermes 的 SessionDB 有什么取舍。
- 两边都支持 steer、interrupt、subagent 和 memory，真正不同的地方在哪里。

## 先给结论：核心边界不同

Codex 开源核心首先是一套面向工作区任务的 Agent runtime。它把下面这些东西做成强约束：

- thread 和 turn 的协议对象。
- submission/event 双向队列。
- 工作区、命令、patch 与 OS sandbox。
- approval policy 与 sandbox policy 的组合。
- 可重放的 rollout 事件流。
- 供 TUI、IDE 和其他客户端使用的 App Server 协议。

Hermes 首先是一套可长期运行的通用 Agent 产品。它的核心压力来自：

- 多 Provider 和兼容不同模型 API。
- 大量内置工具、插件、MCP、skills 和动态 toolset。
- CLI、TUI、Desktop、消息平台、cron、API server 等入口。
- 跨 session memory、用户画像和会话搜索。
- Telegram、Discord 等平台的身份隔离、投递与并发输入。

两边正在互相靠近。Codex 已经有 skills、plugins、memory、subagent、automations 和远程 App Server；Hermes 也有 coding context、worktree、Kanban 和沙箱化终端。真正的差别不是“有没有”，而是**哪一层被当作系统不可绕过的主干**。

## 功能 1：Agent 内核如何接收输入和输出事件

### Codex：核心就是一对异步队列

Codex 在 `codex-rs/core/src/session/mod.rs` 中直接说明了自己的高层接口：

```rust
/// The high-level interface to the Codex system.
/// It operates as a queue pair where you send submissions and receive events.
pub struct Codex {
    pub(crate) tx_sub: Sender<Submission>,
    pub(crate) rx_event: Receiver<Event>,
    pub(crate) agent_status: watch::Receiver<AgentStatus>,
    pub(crate) session: Arc<Session>,
    pub(crate) session_loop_termination: SessionLoopTermination,
}
```

客户端向 `tx_sub` 发送 `Submission`，核心从 `rx_event` 返回 `Event`。`Submission` 和 `Event` 都带关联 id：

```rust
pub struct Submission {
    pub id: String,
    pub op: Op,
    pub client_user_message_id: Option<String>,
    pub trace: Option<W3cTraceContext>,
}

pub struct Event {
    pub id: String,
    pub msg: EventMsg,
}
```

`Op` 不是“用户消息”的别名。它是控制 Agent session 的命令枚举，包含：

- `UserInput`：启动普通用户 turn。
- `Interrupt`：中止当前 turn，但不默认杀死后台终端进程。
- `CleanBackgroundTerminals`：显式清理长期运行的 shell。
- `ExecApproval`、`PatchApproval`：把人的审批结果送回当前状态机。
- `ThreadSettings`：按队列顺序修改 thread 设置。
- MCP elicitation、实时会话、inter-agent communication 等控制操作。

真正消费这些操作的是 `codex-rs/core/src/session/handlers.rs::submission_loop`：

```text
Codex::submit(Op)
  -> tx_sub.send(Submission)
  -> submission_loop 接收
  -> match sub.op
  -> 创建/控制 active turn
  -> 发送 EventMsg
  -> Codex::next_event()
```

因此 Codex 的“Agent loop”不只是一段 `while model wants tool`。外层还有一个 session control loop，负责审批、中断、设置变更和 turn 生命周期；模型-工具循环只是其中一种 task。

### Hermes：Python 外观类加一次对话调用

Hermes 的主入口更接近普通应用服务：

```text
AIAgent.run_conversation(user_message, ...)
  -> agent/conversation_loop.py::run_conversation
  -> 构建 turn context
  -> provider 请求
  -> tool call / tool result
  -> 继续迭代
  -> 返回结构化 result
```

`run_agent.py::AIAgent` 是外观类，真正的长循环在 `agent/conversation_loop.py`。不同入口通过回调、transport 或 Gateway 把流式状态转成 UI/平台事件。

Hermes 不是没有控制协议。第 15 讲已经看到，`tui_gateway/server.py` 提供 `session.create`、`prompt.submit`、`session.interrupt`、`session.steer`；消息 Gateway 还有 queue/steer/interrupt。区别是这些协议位于入口层，`AIAgent` 的公开心智模型仍然是“运行一次 conversation”。

### 为什么会形成这种差异

Codex 的 UI、IDE、App Server 和核心需要稳定交换细粒度事件。把 `Op`、`EventMsg` 和 approval 都做成 Rust enum，可以获得：

- 编译期类型检查。
- 统一的事件关联 id。
- TUI、App Server 和测试可以消费同一协议。
- 中断与审批不是临时 callback，而是正式控制消息。

Hermes 要适配更多动态平台和 Python 插件。直接让 `AIAgent` 暴露回调和结构化结果，接入新平台更快；代价是同一状态可能在 Agent、CLI、TUI Gateway 和消息 Gateway 之间各有一层表达，需要额外防止语义漂移。

## 功能 2：thread/turn 与 session/conversation 的关系

### Codex 对 thread 和 turn 分得很清楚

在 App Server 中：

- `thread/start` 创建持久对话容器。
- `thread/resume` 恢复已有容器。
- `thread/fork` 从历史位置复制出新 thread。
- `turn/start` 在某个 thread 中启动一次 Agent 运行。
- `turn/steer` 向当前 active turn 注入新输入，不创建新 turn。
- `turn/interrupt` 中断指定 turn。

官方 [Codex App Server](https://developers.openai.com/codex/app-server) 文档把调用顺序写成：

```text
initialize
  -> thread/start 或 thread/resume
  -> turn/start
  -> item/started、delta、tool event
  -> turn/completed
```

源码定义在：

- `codex-rs/app-server-protocol/src/protocol/common.rs`
- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/core/src/codex_thread.rs`

`threadId` 表示一条可恢复的对话分支，`turnId` 表示一次运行。当前版本还区分 `sessionId`：根 thread 用自己的 id，fork 出来的 thread 可以保留根 session tree 的 id。这个结构是为了表达“同一会话树里的多个分支”，不能把三个 id 全叫 session id。

### Hermes 的 session 更偏持久会话，turn 边界较隐式

Hermes 的 `SessionDB` 用 session row 和 message row 保存历史。一次 `run_conversation` 通常对应一轮用户输入以及随后的多次模型/工具迭代。工具调用有 turn/tool-call correlation，但公开 API 更常围绕 `session_id` 和 `prompt.submit` 组织。

两边可以这样对照：

| 语义 | Codex | Hermes |
| --- | --- | --- |
| 持久对话 | thread | session |
| 一次用户驱动运行 | turn | 一次 `run_conversation` / `prompt.submit` |
| 运行中的补充输入 | `turn/steer` | `session.steer` 或 Gateway steer |
| 停止当前运行 | `turn/interrupt` / `Op::Interrupt` | `agent.interrupt` / `session.interrupt` |
| 历史分支 | `thread/fork` | SessionDB parent/branch、`session.branch` |

### 两边现在都支持 steer，但保护方式不同

Codex 的 `turn/steer` 要求 `expectedTurnId` 与当前 active turn 匹配。源码最终进入 `Session::steer_input` 和 `session/input_queue.rs`，把输入追加到 turn-local pending input：

```rust
pub(crate) enum InputQueueActivity {
    Mailbox,
    Steer,
}

pub(crate) struct TurnInputQueue {
    items: Vec<TurnInput>,
}
```

`expectedTurnId` 防止客户端拿着过期界面状态去修改一个已经结束或已经更换的 turn。

Hermes 消息 Gateway 没有把所有平台输入都暴露成 `turnId` API，而是用 session active state、run generation 和 pending event 处理竞态。它还允许为消息平台选择 queue、steer 或 interrupt，并在压缩、子 Agent 等阶段降级。

两边解决的是同一个问题：旧输入不能误伤新运行。Codex 更偏显式协议 id，Hermes 更偏 Gateway 内部代际与事件队列。

## 功能 3：项目指令怎样进入 prompt

### Codex 的 AGENTS.md 是层级指令链

Codex 的实现位于：

- `codex-rs/core/src/agents_md.rs`
- `codex-rs/core/src/agents_md_manager.rs`
- `codex-rs/core/src/config/mod.rs`

核心发现逻辑从项目根目录走到当前 cwd，每一级最多选择一个文件：

```rust
fn candidate_filenames(config: &Config) -> Vec<&str> {
    let mut names = Vec::new();
    names.push("AGENTS.override.md");
    names.push("AGENTS.md");
    for candidate in &config.project_doc_fallback_filenames {
        ...
    }
    names
}
```

目录顺序是 root 到 cwd，越靠近 cwd 的文件越晚进入指令链，因此作用域更具体。每个目录优先 `AGENTS.override.md`，再找 `AGENTS.md`，再找用户配置的 fallback 文件。

[AGENTS.md 官方说明](https://developers.openai.com/codex/guides/agents-md) 还给出两个边界：

- 默认总大小上限是 32 KiB，由 `project_doc_max_bytes` 控制。
- 指令链通常在一次 CLI run 或 TUI session 启动时重建，不会每个模型迭代都扫描文件系统。

### Hermes 支持多种上下文文件，还会运行时补充子目录提示

Hermes 的入口在 `agent/prompt_builder.py::build_context_files_prompt`。初始项目上下文采用“类型优先、首个命中”的方式：

```text
.hermes.md
  -> AGENTS.md / agents.md
  -> CLAUDE.md / claude.md
  -> .cursorrules
```

它不是把同一目录下所有规则文件一起加载。选择一种项目上下文格式，是为了避免同一仓库同时存在 Codex、Claude 和 Cursor 规则时重复或冲突。

Hermes 还有 `agent/subdirectory_hints.py`。当 Agent 通过 read/search 等工具进入子目录时，它可以逐步发现该目录附近的 `AGENTS.md`、`CLAUDE.md` 或 `.cursorrules`，并把作用域提示注入后续模型调用。这和 Codex 启动时从 root 走到 cwd 的策略不同：

| 问题 | Codex | Hermes |
| --- | --- | --- |
| 初始扫描 | root 到当前 cwd，每层一个文件 | cwd/root 选择一种主要上下文文件 |
| 多种文件名 | `AGENTS.md` 为主，可配置 fallback | 原生识别 `.hermes.md`、`AGENTS.md`、`CLAUDE.md`、Cursor 规则 |
| 子目录规则 | 启动 cwd 决定初始链 | 工具导航时可渐进发现 subdirectory hints |
| 其他 prompt 层 | base/developer、skills、权限、环境状态 | system message、context files、skills、memory、profile、provider/platform hint |

### 为什么这不只是字符串拼接

两边最后都要生成文本或 Responses API items，但工程问题在拼接之前：

- 谁发现文件。
- 哪个目录拥有作用域。
- 多个文件冲突时谁优先。
- 什么时候重建。
- 多大以后截断。
- 是否安全扫描。
- 压缩或恢复后怎样重新注入。

Codex 更强调可预测的项目规则继承。Hermes 的 Prompt Builder 还要处理用户画像、跨 session memory、skill index 和平台身份，因此 volatile prompt 层更厚。Hermes 还会把首次构建的 system prompt 与 session 持久化，恢复时尽量复用；Codex 则把 instruction sources、turn context 和 rollout items 纳入 thread 生命周期。

## 功能 4：工具注册和 tool call 路由

### Codex：模型可见 spec 与运行时 handler 分开

Codex 的主要入口是：

- `codex-rs/core/src/tools/router.rs::ToolRouter`
- `codex-rs/core/src/tools/registry.rs::ToolRegistry`
- `codex-rs/core/src/tools/spec_plan.rs`
- `codex-rs/core/src/tools/handlers/`

`ToolRouter` 同时持有 registry 和模型可见 specs：

```rust
pub struct ToolRouter {
    registry: ToolRegistry,
    model_visible_specs: Vec<ToolSpec>,
}
```

它先把 Responses API 的不同 item 解析成统一 `ToolCall`：

```text
ResponseItem::FunctionCall
ResponseItem::CustomToolCall
ResponseItem::ToolSearchCall
  -> ToolRouter::build_tool_call
  -> ToolCall { tool_name, call_id, payload }
```

然后 `ToolRegistry` 按 `ToolName` 找到实现。重复注册时会报错或在测试中 panic，不允许后来的 handler 悄悄覆盖前一个：

```rust
for tool in tools {
    let name = tool.tool_name();
    if tools_by_name.contains_key(&name) {
        error_or_panic(format!("tool {name} already registered"));
        continue;
    }
    tools_by_name.insert(name, tool);
}
```

每个 runtime 还声明是否支持 parallel tool calls、是否等待 cancellation、怎样产生 telemetry tags。Rust trait 和 enum 让“工具能力”成为类型契约。

### Hermes：动态注册、toolset 和多来源工具

Hermes 的对应入口是：

- `tools/registry.py::ToolRegistry`
- `toolsets.py`
- `model_tools.py`
- `agent/conversation_loop.py` 的 tool-call 分支

工具通常通过 Python 装饰器或注册函数加入全局 registry。`toolsets.py` 再按场景展开工具组，MCP 和插件可以在运行时注册/撤销工具。模型看到的列表还会经过 Tool Search、provider 能力和当前入口策略筛选。

这种设计适合 Hermes 的工具来源：

- 内置 terminal、browser、web、memory、skills。
- 插件工具。
- 动态 MCP server。
- cron、Gateway、delegate、Kanban 等产品工具。

Codex 也支持 MCP、dynamic tools、extensions 和 tool search，不能再概括成“Codex 静态、Hermes 动态”。更准确的差别是：Codex 用统一 Rust runtime trait 包住不同工具来源；Hermes 以 Python registry 和 toolset 为中心，让动态来源更自然地进入同一表。

### tool call 失败怎样回给模型

Codex 的 `ToolRouter::build_tool_call` 会在 payload 不匹配时生成 `FunctionCallError::RespondToModel`，让模型看到可修正的参数错误。注册表 dispatch 还统一记录 sandbox、permission profile、call id 和 terminal outcome。

Hermes 会在 dispatch 前后做参数 coercion、Tool Search bridge、middleware、plugin hook、审批、截断和循环守卫，最后把字符串化 tool result 回填 provider 消息。

可以这样记：

- Codex 倾向把错误做成协议和类型系统中的 variant。
- Hermes 倾向把错误做成跨工具兼容的结构化字典/文本，并由 dispatch pipeline 修复差异。

这不是语言风格的小差别。它决定了新增工具时，错误处理逻辑是由编译器推动补齐，还是依赖注册约定和集成测试守住。

## 功能 5：sandbox 与 approval 为什么必须分开比较

### Codex：sandbox 是执行能力，approval 是决策策略

Codex 在 `codex-rs/protocol/src/protocol.rs` 中把两者定义为不同 enum。

`SandboxPolicy` 决定技术上允许什么：

```rust
pub enum SandboxPolicy {
    DangerFullAccess,
    ReadOnly { network_access: bool },
    ExternalSandbox { network_access: NetworkAccess },
    WorkspaceWrite {
        writable_roots: Vec<AbsolutePathBuf>,
        ...
    },
}
```

`AskForApproval` 决定何时询问用户：

```rust
pub enum AskForApproval {
    UnlessTrusted,
    OnRequest,
    Granular(GranularApprovalConfig),
    Never,
}
```

官方 [Agent approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security) 将两者概括为：sandbox 决定 Agent 技术上能做什么，approval policy 决定什么时候必须停下来问人。CLI/IDE 默认使用 OS 级机制限制写入范围和网络；macOS、Linux、Windows 分别有对应实现。

`codex-rs/core/src/tools/sandboxing.rs` 再把一次命令归约成三种结果：

```rust
enum ExecApprovalRequirement {
    Skip { bypass_sandbox: bool, ... },
    NeedsApproval { reason: Option<String>, ... },
    Forbidden { reason: String },
}
```

这三种结果很重要：`Never` 不是“危险命令全部允许”，而是“不要弹审批，失败直接回模型”。sandbox 仍然可以拒绝操作。反过来，用户批准一次越界操作，也不意味着后续所有命令都自动获得完全权限。

### Hermes：工具侧安全层更厚，OS sandbox 取决于 backend

Hermes 的危险命令审批集中在 `tools/approval.py`，并配合：

- `tools/path_security.py` 的路径限制。
- `tools/url_safety.py` 的 SSRF 与 URL 检查。
- `tools/threat_patterns.py` 的 prompt injection 规则。
- memory、skills、cron 写入审批。
- Gateway sender allowlist 和平台 pairing。
- terminal backend 的 local、Docker、SSH 等运行方式。

Hermes 可以把 terminal backend 配成 Docker，使一个 session 的命令在容器中运行。源码也明确把 local terminal 视为 unsandboxed；API server 暴露本地执行能力时会给出风险提示。

所以不能说“Hermes 没有 sandbox”，也不能说“两边 sandbox 一样”：

| 边界 | Codex | Hermes |
| --- | --- | --- |
| 默认 coding 执行模型 | OS sandbox 是核心契约 | backend 可选，本地与 Docker/SSH 语义不同 |
| 命令审批 | 与 sandbox policy 正交，类型化 | 规则检测、deny、session approval、smart approval、平台回调 |
| 文件范围 | sandbox writable/readable roots | 文件工具路径校验 + terminal backend 边界 |
| 网络 | sandbox/network policy、proxy 规则 | URL 安全、工具级策略、容器/运行环境网络 |
| 非终端风险 | MCP/app side-effect approval | memory/skill/cron/context/platform 多层专用 guardrail |

Codex 的优势是执行边界统一、可由操作系统强制。Hermes 覆盖的产品风险更多，除了 shell 越界，还要处理长期 memory 污染、cron prompt、消息平台身份和投递目标。构建自己的 Agent 时，这两类保护通常都需要，不能二选一。

## 功能 6：上下文与压缩怎样建模

### Codex 的 ContextManager 保存 Responses API item

`codex-rs/core/src/context_manager/history.rs` 中的 `ContextManager` 不是简单的消息数组：

```rust
pub(crate) struct ContextManager {
    items: Vec<ResponseItem>,
    history_version: u64,
    token_info: Option<TokenUsageInfo>,
    reference_context_item: Option<TurnContextItem>,
    world_state_baseline: Option<WorldStateSnapshot>,
}
```

它负责：

- 只记录适合 API history 的 item。
- 按 policy 截断超长 function/tool output。
- 在不支持图像的模型上移除不适合的 image item。
- 保存 token usage。
- 在 rollback、compaction 后递增 history version。
- 维护 reference context 和 world state baseline，生成后续差量上下文。

Codex 的历史单位直接贴近 Responses API item，所以 function call、custom tool call、reasoning、message、compaction item 能用统一 enum 表示。

### Codex 同时支持本地和远程 compaction

入口位于：

- `codex-rs/core/src/compact.rs`
- `codex-rs/core/src/compact_remote.rs`
- `codex-rs/core/src/compact_remote_v2.rs`

provider 支持远程压缩时可以走远程任务，否则用本地 summarization prompt。压缩还区分：

- 手动压缩还是自动压缩。
- turn 前压缩还是 turn 中压缩。
- 是否需要把 initial context 插回 summary 前。
- pre/post compact hook 是否允许继续。

压缩成功后，Codex 会构造新 history、保留必要用户消息、写入 compacted item，并更新 auto-compact window。它还会提醒长 thread 和多次压缩可能降低准确性。

### Hermes 的压缩与 system prompt 恢复更紧密

Hermes 的主要入口是：

- `agent/context_compressor.py`
- `agent/conversation_compression.py`
- `agent/conversation_loop.py`
- `agent/system_prompt.py`

Hermes 压缩 message history 后，会重新构建或恢复 system prompt，把 context files、memory、skills 等层重新放回模型上下文。它还要维护 SessionDB 中的 summary、parent chain 和可搜索历史。

两边的共同原则是：压缩只替换可压缩历史，不能让权限、项目规则和当前工作环境一起消失。差别在于 Codex 更接近 Responses API item/world-state reducer；Hermes 更接近 message list + system prompt layers + SessionDB summary。

### rollout 文件压缩不等于上下文压缩

Codex 的 `codex-rs/rollout/src/compression.rs` 会把冷的 `.jsonl` 文件压成 `.jsonl.zst`。这是磁盘存储优化，不会直接减少模型 token。

同一个项目里“compression”至少有两种：

- context compaction：减少模型输入。
- rollout compression：减少磁盘占用。

读源码时必须看对象是什么。Hermes 中也要区分 context compression、tool result truncation 和数据库整理，不能看到“压缩”就认为是同一层。

## 功能 7：会话持久化为何选择不同主存储

### Codex：append-only rollout 是可重放事实，SQLite 负责索引

Codex 的 `codex-rs/rollout/src/recorder.rs` 把 thread 事件追加到 rollout JSONL。文件中可以包含：

- SessionMeta。
- ResponseItem。
- EventMsg。
- TurnContext。
- Compacted item。

追加式事件流便于：

- 按原始顺序回放一次 Agent 运行。
- 调试某个 tool call 前后发生了什么。
- 在 schema 演进时保留原事件。
- fork/resume 时重建 history。

`codex-rs/state` 的 SQLite 则保存 thread metadata、rollout path、标题、cwd、model、sandbox policy、token usage、父子关系等，用于快速列表、筛选和后台 memory job。它不是简单替代 rollout 文件。

### Hermes：SQLite 直接承载 session 和 message

`hermes_state.py::SessionDB` 把 sessions、messages、compression、parent chain 等直接存入 SQLite，并建立 FTS5/Trigram 索引。`session_search` 可以直接查询消息内容和 session 元数据。

这两种设计可以概括为：

| 问题 | Codex | Hermes |
| --- | --- | --- |
| 主要运行轨迹 | append-only rollout JSONL | SQLite messages/session rows |
| 快速列表和元数据 | SQLite state index | 同一个 SessionDB |
| 全文搜索 | rollout 搜索 + state 能力 | FTS5/Trigram 是一等能力 |
| 回放调试 | 原始事件保留得更自然 | 需要从 message/tool metadata 重建 |
| 事务查询 | 跨事件分析要经过 index/reducer | SQL 查询直接，关系更集中 |

Codex 偏 event sourcing：先保留发生过的事实，再从事实提取状态。Hermes 偏关系状态模型：session 和 message 就是主要事实。两边现在都不是“纯 JSONL”或“纯 SQLite”，只是主次不同。

## 功能 8：App Server 与 UI Gateway 的相似和差别

### 相似点：都把 Agent runtime 与 UI 解耦

Codex App Server 和 Hermes `tui_gateway` 都支持双向 JSON-RPC，都有：

- 创建/恢复会话。
- 启动 turn/prompt。
- 流式模型和工具事件。
- 审批请求与响应。
- steer 和 interrupt。
- 配置或 session 设置。

Codex App Server 默认使用换行分隔 JSON，WebSocket 仍属于实验性 transport。官方协议要求连接先 `initialize`，再发送 `initialized`，未完成握手的请求会被拒绝。WebSocket 入口还使用有界队列，过载时返回 `-32001`，客户端应该指数退避。

Hermes stdio TUI Gateway 启动后先发送 `gateway.ready`，随后由 `server.dispatch` 处理方法。Web 后端的 `/api/ws` 复用同一 dispatch；Dashboard 另外还有 REST、PTY 和事件广播通道。

### 差别：Codex 协议围绕 thread/turn，Hermes 围绕产品 session

| 维度 | Codex App Server | Hermes UI Gateway |
| --- | --- | --- |
| 核心对象 | thread、turn、item | session、prompt、event |
| 协议 schema | Rust 类型生成，多版本兼容 | Python method registry，前端共享 TS client |
| 客户端 | TUI、IDE、远程客户端等 | Ink TUI、Dashboard、Desktop |
| 工作区策略 | cwd、sandbox、approval 是 turn/thread 参数 | cwd、profile、model、toolsets 与 Agent 构建绑定 |
| 消息平台 | App Server 本身不负责 chat adapter | 另有 `gateway/` 常驻平台运行时 |

Codex App Server 适合“把 Codex 编码 Agent 嵌进另一个客户端”。Hermes UI Gateway 适合“给 Hermes 自己的多个界面提供统一 Agent 后端”。两个协议都能扩展，但初始产品边界不同。

## 功能 9：多 Agent、memory 和长期任务不能再用旧印象比较

### 多 Agent

Codex 当前源码有 `codex-rs/core/src/agent/control.rs::AgentControl`。subagent 使用独立 `ThreadId`，记录 parent/child、agent path、状态、rollout budget，并能 spawn、resume、interrupt 和发送 inter-agent communication。

Hermes 的多 Agent 分成：

- 同步 delegate。
- async delegation。
- Kanban task/worker。
- worktree 隔离。
- Gateway completion event。

Codex 更倾向把子 Agent 视为 thread tree 中的受控 session；Hermes 还把后台任务、看板调度和消息平台通知作为产品流程。

### Memory

Codex 当前仓库已经有 `codex-rs/memories`、state memory jobs 和 thread memory mode。它会从合适的 rollout 中提取、聚合可复用记忆，并区分 enabled、disabled、polluted 等状态。因此“Codex 没有长期记忆”已经是错误结论。

Hermes 的 memory 更直接进入日常 prompt：`MemoryStore` 管理长期条目，MemoryProvider 和 USER profile 在 system prompt 的 volatile layer 中加载，还有写入审批、注入扫描和 review 流程。

真正区别是：Codex memory 与 thread rollout、后台提取管线结合更紧；Hermes memory 与用户画像、多平台长期 Agent 身份结合更紧。

### 长期任务

Codex 产品当前有 goals、automations、cloud tasks 和长期工作表面，不能说它只运行一次 CLI 命令。Hermes 则在开源仓库中直接实现 cron job、Gateway 常驻进程、平台 delivery 和恢复逻辑。

比较时要区分两个问题：

1. 产品是否提供这项能力。
2. 当前开源仓库能否看到完整调度和投递实现。

前者回答用户能不能用，后者回答工程师能从哪里学习实现。

## 从工程实践看，应该向谁学什么

### 构建编码 Agent 时优先学习 Codex

值得直接借鉴的设计包括：

- `Submission/Event` 控制协议。
- thread/turn/item 的清晰层次。
- sandbox policy 与 approval policy 正交。
- OS 级 sandbox 作为默认执行边界。
- typed tool runtime 与统一 telemetry。
- append-only rollout 和状态 reducer。
- App Server 的初始化、能力协商、有界队列和过载错误。

这些设计适合高风险的本地代码修改，因为“模型生成的命令在哪里运行”比“模型是否记得用户喜好”更先决定系统是否可靠。

### 构建长期个人 Agent 或消息 Agent 时优先学习 Hermes

值得直接借鉴的设计包括：

- 多 Provider profile 和 failover。
- toolset、插件、MCP、skills 的动态组合。
- SessionDB、FTS 和 `session_search`。
- memory/user profile 的 prompt 集成。
- Telegram/Discord 等平台 adapter 与 session key。
- queue/steer/interrupt 的平台输入策略。
- cron、delivery、Gateway restart recovery。

这些设计适合 Agent 长时间在线、从多个入口接收任务的场景，因为身份、投递、恢复和跨 session 状态会成为主要复杂度。

### 更完整的实现需要组合两边

一个既能长期在线又能修改代码的 Agent，合理的组合是：

```text
Hermes 式平台与会话层
  + Codex 式 thread/turn 控制协议
  + Codex 式 OS sandbox
  + Hermes 式 memory / profile / delivery
  + 两边共同的 tool registry / MCP / skills
  + append-only audit trail 与可查询状态库
```

这里不是建议把两个仓库拼在一起，而是说明模块边界：平台身份不能由 coding sandbox 解决，OS sandbox 也不能由 prompt injection 正则替代。

## 常见误解与问答

### Codex 是否只是一个调用 OpenAI 模型的 CLI？

不是。开源核心有 session control loop、工具路由、sandbox、approval、rollout、context manager、App Server 和 subagent。CLI/TUI 只是客户端之一。把它看成“一个 Rust 写的聊天循环”会漏掉最值得学习的协议与执行边界。

### Hermes 是否比 Codex 更通用，所以架构一定更复杂？

不一定。Hermes 的产品入口更多，动态集成更复杂；Codex 的 sandbox、协议类型、rollout、跨客户端兼容和工作区权限也很复杂。复杂度只是落在不同位置，不能用目录数量判断。

### 两边都有 ToolRegistry，能否认为实现相同？

不能。Codex Registry 保存实现 runtime，Router 还维护模型可见 spec，并用 Rust trait/enum 约束 payload、并发和取消。Hermes Registry 与 toolset、插件和 MCP 的动态注册关系更紧，dispatch 过程包含更多 Python middleware 和兼容修复。

### Codex 的 `approvalPolicy="never"` 是否等于完全放权？

不是。它表示不向用户弹审批，失败直接回模型。sandbox policy 仍然决定进程能写哪里、能否联网。只有 `danger-full-access` 这类 sandbox 设置才改变技术执行范围，两者必须分开看。

### Hermes 的危险命令检测能否替代 OS sandbox？

不能。规则检测可以识别常见危险命令，但 shell 语法、解释器、二进制程序和间接写入方式太多，不可能只靠正则覆盖。Docker backend 或其他隔离环境提供技术边界；approval 和 deny 规则提供策略边界。

### Codex 和 Hermes 谁的会话恢复更可靠？

不能脱离失败类型回答。Codex 的 append-only rollout 对事件回放和调试很有优势；Hermes 的 SQLite session/message 对搜索、关系查询和平台长期会话更直接。可靠性还取决于 tool tail 修复、幂等性、持久化时机和外部副作用，不能只看存储格式。

### 为什么 Codex 用 Rust，Hermes 用 Python？

Codex 需要高并发事件、跨平台 sandbox、强协议类型和单二进制分发，Rust 的类型与系统能力很合适。Hermes 需要快速接入 Provider、平台 SDK、Python 工具和研究生态，Python 的动态性更合适。语言选择和产品边界一致，不是单纯的性能排名。

## 本篇要记住的东西

Codex 把编码 Agent 的执行契约放在核心：`Submission -> Session -> active turn -> tool runtime -> Event`，再用 OS sandbox、approval 和 rollout 守住工作区任务。

Hermes 把长期 Agent 的产品契约放在核心：`入口事件 -> session/profile -> prompt/memory/skills -> conversation loop -> tool ecosystem -> 平台投递`，再用 Gateway、SessionDB、cron 和多层 guardrail 维持长期运行。

两边都已经拥有 steer、interrupt、skills、memory、MCP 和 subagent。判断差异时不要再问“有没有”，而要问：

- 它位于哪个模块？
- 是核心协议还是外围能力？
- 状态由谁持久化？
- 失败后由谁恢复？
- 安全边界由提示词、规则、容器还是操作系统强制？

能回答这五个问题，才算真正读懂一个 Agent 工程。

## 参考资料

### Hermes 源码

- `run_agent.py`
- `agent/conversation_loop.py`
- `agent/prompt_builder.py`
- `agent/subdirectory_hints.py`
- `agent/context_compressor.py`
- `tools/registry.py`
- `tools/approval.py`
- `toolsets.py`
- `model_tools.py`
- `hermes_state.py`
- `tui_gateway/server.py`
- `gateway/run.py`

### Codex 源码

- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/session/handlers.rs`
- `codex-rs/core/src/session/input_queue.rs`
- `codex-rs/core/src/context_manager/history.rs`
- `codex-rs/core/src/compact.rs`
- `codex-rs/core/src/agents_md.rs`
- `codex-rs/core/src/tools/router.rs`
- `codex-rs/core/src/tools/registry.rs`
- `codex-rs/core/src/tools/sandboxing.rs`
- `codex-rs/core/src/agent/control.rs`
- `codex-rs/rollout/src/recorder.rs`
- `codex-rs/state/src/runtime.rs`
- `codex-rs/app-server/src/message_processor.rs`
- `codex-rs/app-server-protocol/src/protocol/common.rs`

### 官方文档

- OpenAI：[Codex App Server](https://developers.openai.com/codex/app-server)
- OpenAI：[Codex CLI](https://developers.openai.com/codex/cli/features)
- OpenAI：[AGENTS.md](https://developers.openai.com/codex/guides/agents-md)
- OpenAI：[Agent approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security)
- OpenAI：[Codex 开源仓库](https://github.com/openai/codex)
- Nous Research：[Hermes Agent 开源仓库](https://github.com/NousResearch/hermes-agent)
