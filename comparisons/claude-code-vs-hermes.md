# 第 17 讲：Claude Code 与 Hermes 的工程架构对比

Claude Code 和 Hermes 都能阅读代码、调用工具、延续会话、加载项目指令并派生子代理。功能名称相近，但状态和扩展边界不同：Claude Code 围绕本地开发会话组织能力，Hermes 围绕可长期运行、可接入多个平台的通用 Agent 组织能力。

本文基于以下公开快照：

- `NousResearch/hermes-agent@590a19332e898fc9bda55a31999926572d8fbc26`
- `anthropics/claude-code@d4d8fbbb333c627d8fe2c1c583a5ccc26fdb1aed`
- `anthropics/claude-agent-sdk-python@528265fa09da954f0a0da1bf31e16db32b510138`

证据边界需要单独说明。`anthropics/claude-code` 是公开仓库，但主要内容是插件、示例配置、运维脚本、变更记录和问题跟踪，不包含 Claude Code 核心运行时的完整源码。Claude Agent SDK 的 Python 部分有源码，不过 SDK 最终会启动 Claude Code CLI。它能证明公开接口怎样调用 CLI，不能证明 CLI 内部怎样实现 Agent loop。

所以本文使用三种证据，并明确区分：

1. Claude Code 官方文档，用来确认产品行为与公开协议。
2. `anthropics/claude-code` 中的插件源码，用来观察扩展机制的真实用法。
3. Claude Agent SDK 源码，用来确认应用程序如何配置、启动和控制 Claude Code。

Hermes 可以沿调用链读到 Agent loop、工具调度、会话存储、Gateway 和安全检查。两边的可见性不同，因此比较内部实现时必须使用不同证据强度。

## 对比范围

对比集中在以下问题：

- `CLAUDE.md` 和 Hermes 支持的 `AGENTS.md`、`.hermes.md` 有什么差别。
- 用户输入“继续”时，Claude Code 与 Hermes 分别从哪里恢复上下文。
- Claude Code 的 permissions、sandbox、Hooks 是不是同一层机制。
- Hermes 为什么同时需要 Tool Registry、工具调用管线、approval 和 plugin middleware。
- Claude Code 的 Skills、plugins、MCP 与 Hermes 的对应设计怎样分工。
- 两边的子代理如何隔离上下文、限制工具、处理后台任务。
- Claude Code Remote Control/Channels 与 Hermes Gateway 为什么不能简单画等号。
- 哪些 Claude Code 结论有源码证据，哪些只能确认外部行为。

## 架构边界

Claude Code 的产品中心是一个开发者会话。项目目录、文件修改、Shell、权限、上下文窗口、检查点和 IDE/远程控制都围绕这条会话展开。它的公开配置也很贴近开发工作流：

- `CLAUDE.md` 保存项目约束。
- `.claude/skills/` 保存可按需加载的流程。
- `.claude/agents/` 定义专用子代理。
- `.claude/settings.json` 管理权限、Hooks、MCP 和沙箱。
- 本地 transcript 保存可恢复的对话。

Hermes 的产品中心是一个能够长期驻留的 Agent 实例。开发目录只是其中一种运行环境，它还要处理：

- 多模型 Provider 与不同 API 兼容层。
- CLI、TUI、Desktop、API、cron 和消息平台入口。
- Telegram、Discord、Slack、WhatsApp 等来源的身份与会话隔离。
- 跨会话 memory、用户画像、session search。
- 内置工具、插件工具、MCP 工具和动态 Tool Search。
- 后台子代理、投递、重启恢复和会话并发输入。

Claude Code 的公开产品边界首先是 coding agent，Hermes 的源码边界首先是通用 Agent Runtime 与多入口产品。两边都在扩展能力，但主干仍受各自入口和状态模型影响。

## 功能 1：一次会话怎样被创建、继续和恢复

### Claude Code：会话是项目目录下持续写入的 transcript

Claude Code 官方的 [Sessions](https://code.claude.com/docs/en/sessions) 文档把 session 定义为与项目目录绑定的已保存对话。CLI 在工作过程中持续写本地 transcript，因此退出程序不等于删除上下文。

常见入口是：

```text
claude --continue          恢复当前目录最近一次会话
claude --resume            打开会话选择器
claude --resume <id/name>  恢复指定会话
/resume                    在运行中的 CLI 里切换会话
/branch                    从当前历史分出新会话
```

这里需要区分三个动作：

- `continue`：按当前项目目录寻找最近会话。
- `resume`：按 session id 或名称寻找某一条既有 transcript。
- `fork/branch`：复制已有历史，生成新的 session id，原分支不再被后续消息修改。

用户输入“继续”只是普通自然语言时，模型会根据当前上下文理解它；`--continue` 则是 CLI 的会话恢复命令。前者发生在模型层，后者发生在运行时层，不能混为一谈。

Claude Agent SDK 把这些能力映射成配置字段。源码锚点：

- `anthropics/claude-agent-sdk-python/src/claude_agent_sdk/types.py::ClaudeAgentOptions`
- `anthropics/claude-agent-sdk-python/src/claude_agent_sdk/_internal/transport/subprocess_cli.py::_build_command`

```python
@dataclass
class ClaudeAgentOptions:
    continue_conversation: bool = False
    resume: str | None = None
    session_id: str | None = None
    fork_session: bool = False
```

SDK 并没有在 Python 里重新实现恢复算法。它把选项翻译成 CLI 参数：

```python
if self._options.continue_conversation:
    cmd.append("--continue")

if self._options.resume:
    cmd.extend(["--resume", self._options.resume])

if self._options.fork_session:
    cmd.append("--fork-session")
```

这个片段能确认 SDK 的职责是“配置和传输”，却不能证明 Claude Code 内部使用某个 Python 类保存消息。

### Hermes：SessionDB 保存对话，Gateway 另外维护来源到 session 的路由

Hermes 的会话有两层身份：

1. `session_id` 指向 SessionDB 中的持久对话。
2. Gateway 的 `session_key` 把平台、聊天、话题、参与者和 profile 映射到某个 session。

源码锚点：

- `NousResearch/hermes-agent/gateway/session.py::build_session_key`
- `NousResearch/hermes-agent/gateway/session.py::SessionStore`
- `NousResearch/hermes-agent/hermes_state.py::SessionDB`

```python
def build_session_key(
    source: SessionSource,
    group_sessions_per_user: bool = True,
    thread_sessions_per_user: bool = False,
    profile: Optional[str] = None,
) -> str:
    ...
    return ":".join(key_parts)
```

`build_session_key` 不是简单拼一个 `platform:chat_id`。它还处理：

- DM 是否有独立 `chat_id`。
- 群聊是否按用户隔离。
- thread/topic 是否共享一条会话。
- WhatsApp JID/LID 等身份别名归一化。
- 多 profile 是否需要独立命名空间。

`SessionStore` 再把这个确定性 key 映射到 SessionDB 的会话记录。这样 Gateway 重启后，同一个来源仍能找回原 session。

```text
消息平台事件
  -> SessionSource
  -> build_session_key
  -> SessionStore 查路由
  -> session_id
  -> SessionDB 读取历史
  -> AIAgent.run_conversation
```

### “继续”到底由谁理解

如果 Hermes 当前 session 的历史仍在模型上下文里，用户说“继续”，模型会把它理解为继续最近未完成的话题。运行时并没有专门匹配“继续”两个字。

如果进程已经重启，Gateway/CLI 先根据 session id 恢复消息，再把可用历史送给模型。历史过长时，ContextCompressor 可能把中段换成摘要。此时模型看到的是“恢复后的有效上下文”，不是数据库里的所有原始行。

因此判断链是：

```text
运行时决定恢复哪条 session
  -> 上下文引擎决定哪些历史进入请求
  -> 模型根据当前可见内容解释“继续”
```

Claude Code 也是同一原则。CLI 决定恢复哪条 transcript，模型决定自然语言“继续”指向什么。

## 功能 2：项目指令怎样进入上下文

### Claude Code：层级 CLAUDE.md 会合并，不是覆盖

Claude Code 的 [Memory](https://code.claude.com/docs/en/memory) 文档说明，项目指令可以来自组织、用户、项目根目录、当前目录和本地私有文件。对于目录树中的 `CLAUDE.md` 与 `CLAUDE.local.md`，Claude Code 会按从上到下的路径顺序合并，离工作目录更近的内容后出现。

例如从 `repo/apps/web/` 启动时，可能形成：

```text
用户级 CLAUDE.md
  -> repo/CLAUDE.md
  -> repo/apps/CLAUDE.md
  -> repo/apps/web/CLAUDE.md
  -> 同层 CLAUDE.local.md
```

子目录里的 `CLAUDE.md` 不必全部在启动时加载。Claude Code 读到该子目录中的文件时，可以再按需发现。这个设计有两个目的：

- monorepo 中每个包可以保存自己的约束。
- 与当前工作无关的子目录规则不占用上下文。

官方文档还给出一个容易被忽略的细节：`CLAUDE.md` 内容作为 system prompt 之后的用户消息进入上下文，不是直接并入 system prompt。它仍然是高优先级项目指令，但从消息角色上说与真正的 system prompt 不同。

### Hermes：启动时第一类命中，运行时再懒加载子目录提示

Hermes 的启动上下文由 `build_context_files_prompt` 负责。源码锚点：

- `NousResearch/hermes-agent/agent/prompt_builder.py::build_context_files_prompt`

```python
project_context = (
    _load_hermes_md(cwd_path, context_length)
    or _load_agents_md(cwd_path, context_length)
    or _load_claude_md(cwd_path, context_length)
    or _load_cursorrules(cwd_path, context_length)
)
```

优先级是：

1. `.hermes.md` / `HERMES.md`，向上寻找至 Git 根目录。
2. `AGENTS.md` / `agents.md`，只看当前工作目录。
3. `CLAUDE.md` / `claude.md`，只看当前工作目录。
4. `.cursorrules` / `.cursor/rules/*.mdc`，只看当前工作目录。

这里使用 Python 的 `or`，意味着项目上下文类型是 first-match-wins。假设当前目录同时有 `AGENTS.md` 和 `CLAUDE.md`，启动阶段只会采用 `AGENTS.md`，不会像 Claude Code 那样把两者都合并。

`SOUL.md` 不参与这组竞争。它来自 Hermes home，单独进入身份槽位。

Hermes 还有第二条路径：`SubdirectoryHintTracker` 在工具访问新目录时，检查该目录及有限层级祖先中的提示文件。源码锚点：

- `NousResearch/hermes-agent/agent/subdirectory_hints.py::SubdirectoryHintTracker`

```python
hints = tracker.check_tool_call(
    "read_file", {"path": "backend/src/main.py"}
)
if hints:
    tool_result += hints
```

发现到的内容追加在 tool result 后面，而不是修改 system prompt。这样做能保留 system prompt 前缀缓存，又能让模型在进入子目录的那一刻拿到局部规则。

### 两种设计的关键差异

| 问题 | Claude Code | Hermes |
| --- | --- | --- |
| 主文件 | `CLAUDE.md` | `.hermes.md`，并兼容 `AGENTS.md`、`CLAUDE.md`、Cursor rules |
| 父目录规则 | 层级合并 | `.hermes.md` 向上寻找；其他类型启动时只读 cwd |
| 同目录多种规则 | Claude 自己的规则体系合并 | 项目上下文类型 first-match-wins |
| 子目录规则 | 访问相关目录时懒加载 | 依据工具参数发现，追加进 tool result |
| 消息位置 | 官方文档称其位于 system prompt 后的用户消息 | `build_context_files_prompt` 生成 system prompt 的 Project Context 部分 |
| 缓存考虑 | 根规则可在 compact 后重注入 | 子目录提示不改 system prompt，保留前缀缓存 |

这项差异会直接影响指令优先级、上下文生命周期、缓存稳定性和 monorepo 的作用域。

## 功能 3：长期记忆与项目指令怎样分工

### Claude Code：CLAUDE.md 是显式规则，auto memory 是模型维护的经验

Claude Code 把两类内容分开：

- `CLAUDE.md`：团队或用户显式维护的稳定指令。
- auto memory：Claude 在工作中写入的经验、约定和调试线索。

auto memory 目录有一个 `MEMORY.md` 入口和若干主题文件。启动时只自动加载 `MEMORY.md` 的前 200 行或 25KB，详细主题文件按需读取。它是普通 Markdown，用户可以检查和删除。

这套设计把 memory 当成“可审计的文件知识库”。模型负责选择何时写入，但运行时规定存储位置、启动加载上限和作用域。

### Hermes：MEMORY.md 与 USER.md 进入冻结的 system prompt 快照

Hermes 内置 memory tool 有两个目标：

- `target="memory"`：持久事实、偏好、长期上下文。
- `target="user"`：用户身份、角色、表达偏好等画像。

源码锚点：

- `NousResearch/hermes-agent/tools/memory_tool.py::MemoryStore`

```python
class MemoryStore:
    def __init__(self, memory_char_limit=2200, user_char_limit=1375):
        self.memory_entries = []
        self.user_entries = []
        self._system_prompt_snapshot = {"memory": "", "user": ""}
```

加载时，Hermes 同时维护两份状态：

```text
磁盘 MEMORY.md / USER.md
  -> live entries：工具调用后可修改并落盘
  -> frozen snapshot：本 session 构建 prompt 时固定
```

为什么不让新写入的 memory 立即改 system prompt？因为 system prompt 前缀一旦改变，Provider 的 prompt cache 很可能失效；而且模型在同一 turn 里写入一条 memory 后又把它当成更高优先级指令，会造成难以审计的自我强化。

Hermes 还会在写入前和加载快照时扫描 prompt injection 模式：

```python
scan_error = _scan_memory_content(content)
if scan_error:
    return {"success": False, "error": scan_error}
```

这一步由确定性规则库执行，不是主模型临时判断。模型只看到工具返回的允许或拒绝结果。

### 取舍

Claude Code 的 auto memory 更像模型维护的项目笔记目录，允许分主题扩展；Hermes 默认 memory 是有严格字符预算的精选事实，并把用户画像单独建模。前者容量和组织方式更开放，后者更强调 system prompt 成本、稳定快照和跨平台用户一致性。

## 功能 4：模型怎样看到工具，运行时怎样执行工具

### 先回答共同问题：调用哪个工具主要由模型决定

无论 Claude Code 还是 Hermes，运行时都会把工具名称、描述和参数 schema 放进模型请求。模型生成结构化 tool use，运行时再验证权限、执行工具并把结果送回模型。

```mermaid
flowchart LR
    A["运行时提供工具 schema"] --> B["模型选择工具并生成参数"]
    B --> C["运行时做权限与安全检查"]
    C --> D["执行工具"]
    D --> E["tool result 返回模型"]
    E --> F["模型继续推理或回答"]
```

模型是否选择 `session_search`、`Read`、`Grep` 或 MCP 工具，通常取决于工具描述、当前问题和模型能力。运行时可以通过工具可见性、提示词、强制规则和 Hook 缩小选择空间，但不会替模型完成语义决策。

### Claude Code：公开接口区分“可见工具”和“自动批准工具”

Claude Agent SDK 的 `ClaudeAgentOptions` 有几个很容易混淆的字段：

```python
tools: list[str] | ToolsPreset | None
allowed_tools: list[str]
disallowed_tools: list[str]
permission_mode: PermissionMode | None
can_use_tool: CanUseTool | None
```

含义分别是：

- `tools`：决定哪些内置工具出现在可用工具集合。
- `allowed_tools`：匹配的工具不再询问用户，不等于“只有这些工具可见”。
- `disallowed_tools`：从可用集合中移除工具。
- `permission_mode`：定义未被规则直接处理的调用怎样审批。
- `can_use_tool`：SDK 收到 ask 型权限请求时，由应用程序返回允许或拒绝。

SDK 启动 CLI 时使用 stream-json：

```python
cmd = [self._cli_path, "--output-format", "stream-json", "--verbose"]
```

在双向模式中，CLI 可以向 SDK 发 `control_request`。SDK 的处理路径会识别 `can_use_tool`，调用用户回调，再返回结构化决定。源码锚点：

- `anthropics/claude-agent-sdk-python/src/claude_agent_sdk/_internal/query.py::_handle_control_request`

这证明了 SDK 与 CLI 之间有一套控制协议，但 CLI 内部如何从模型 tool use 走到 permission request，公开 SDK 源码并没有展开。

### Hermes：Registry 只负责注册和最终派发，前面还有完整管线

Hermes 的 `ToolRegistry` 保存名称、toolset、schema、handler、可用性检查和结果上限。源码锚点：

- `NousResearch/hermes-agent/tools/registry.py::ToolRegistry`

```python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    entry = self.get_entry(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        return json.dumps({"error": sanitized})
```

`registry.dispatch` 不是工具调用的全部。真正进入它之前，`agent/tool_executor.py` 还要处理：

1. 解析模型给出的工具名与 JSON 参数。
2. Tool Search scope gate，防止模型直接调用尚未暴露的延迟工具。
3. tool request middleware，允许插件在安全检查前重写参数。
4. `pre_tool_call` Hook，允许插件阻断调用。
5. tool-loop guardrail，识别同一失败调用反复重试。
6. 特殊内置工具路径，例如 memory、session search、delegate。
7. tool execution middleware，用 `next_call` 包住真实执行。
8. 记录耗时、错误、取消和观测事件。
9. 限制或外置过大的结果。
10. 根据工具参数发现子目录上下文文件。
11. 将 tool result 追加到消息列表，等待下一次模型请求。

源码锚点：

- `NousResearch/hermes-agent/agent/tool_executor.py::execute_tool_calls_sequential`
- `NousResearch/hermes-agent/agent/tool_executor.py::execute_tool_calls_concurrent`
- `NousResearch/hermes-agent/agent/tool_guardrails.py::ToolCallGuardrailController`
- `NousResearch/hermes-agent/hermes_cli/middleware.py`

这解释了为什么“单个工具”和“tool call”不是同一个概念：

- 单个工具是 Registry 中的一项能力定义。
- tool call 是某一次模型请求产生的调用实例，带参数、tool call id、状态、耗时和结果。

同一个工具可以在一个 session 里产生很多次 tool call；并行响应还可能一次产生多个 tool call。

## 功能 5：权限、审批和沙箱怎样配合

### Claude Code：三层机制各做一件事

Claude Code 的安全路径可以拆成三层。

第一层是 permission rules。规则使用 allow、ask、deny，优先级是 deny、ask、allow。deny 命中后不会因为某条 allow 规则而放行。

第二层是 permission mode。官方 [Permissions](https://code.claude.com/docs/en/permissions) 文档列出了 `default`、`acceptEdits`、`plan`、`auto`、`dontAsk`、`bypassPermissions` 等模式。mode 是默认策略，具体规则仍可缩小权限。

第三层是 OS sandbox。官方 [Sandboxing](https://code.claude.com/docs/en/sandboxing) 文档说明，Bash 子进程可通过 macOS Seatbelt、Linux/WSL2 bubblewrap 和网络代理限制文件与网络访问。它只覆盖 Bash 及子进程；Read、Edit、Write 等内置工具仍由 permissions 约束。

因此：

```text
permission：允不允许调用某个工具或某类参数
sandbox：即使调用了 Bash，操作系统允许它碰到什么
Hook：在生命周期节点追加组织自己的判断或处理
```

把三者都叫“审批”会失去设计重点。permission 是策略判定，sandbox 是执行边界，Hook 是扩展点。

### Hermes：审批以危险操作识别为主，隔离环境是可选 backend

Hermes 的终端安全主要在 `tools/approval.py`。它包含：

- 硬阻断的敏感目标和用户自定义 deny glob。
- 危险命令模式，例如远程脚本直接 pipe 到 shell。
- `manual`、`smart`、`off` 审批模式。
- CLI、Gateway、ACP 等不同界面的审批回调。
- session 级一次允许、会话允许、永久允许。
- cron 的非交互策略。
- `turn_id`、`tool_call_id`、`session_key` 的关联信息。

smart approval 可以调用辅助模型评估风险，但硬阻断和规则扫描不依赖模型。即使启用 yolo，也有不能被绕过的保护项。

Hermes 的终端还可以选择 Local、SSH、Docker、Singularity、Modal、Daytona 等 environment backend。Docker backend 会使用容器能力限制、资源限制和可选挂载；如果挂载了宿主路径，审批层不会再把它视为完全隔离。

源码锚点：

- `NousResearch/hermes-agent/tools/approval.py`
- `NousResearch/hermes-agent/tools/environments/base.py::BaseEnvironment`
- `NousResearch/hermes-agent/tools/environments/docker.py::DockerEnvironment`

### 安全模型的差别

Claude Code 把工作区内 coding 操作做成统一 permissions，并为 Bash 提供原生 OS sandbox。Hermes 的主线更偏“识别危险命令并按运行界面审批”，执行环境可以从本机换到容器或远程机器。

这会带来不同的风险：

- Claude Code 的规则系统更统一，但用户若放宽 sandbox 路径或网络域，边界也会随之变大。
- Hermes backend 很灵活，但 Local、Docker 挂载、SSH 的信任边界完全不同，部署者必须理解当前 backend。
- 两边都不能只靠模型承诺“我会小心”。真正的限制必须在模型之外执行。

## 功能 6：Hooks 与 middleware 为什么不是一回事

### Claude Code Hooks：生命周期事件上的外部处理器

Claude Code 的 [Hooks](https://code.claude.com/docs/en/hooks) 可以在 `PreToolUse`、`PostToolUse`、`Stop`、`SessionStart`、`PreCompact` 等事件上运行 command、HTTP、prompt 或 agent handler。

例如 `PreToolUse` 可以返回 allow、deny、ask、defer，也可以修改工具参数。多个结果冲突时，deny 的优先级最高。

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "command violates project policy"
  }
}
```

Hook 判断可以有两种来源：

- command/HTTP：确定性代码或外部策略服务。
- prompt/agent：把事件交给另一个模型判断。

因此“Hook 是否使用模型”没有统一答案。看 Hook 类型。文件名检查、命令正则、审计日志通常适合代码；模糊的完成度评估可以使用 prompt Hook，但成本和不确定性更高。

公开 `anthropics/claude-code` 仓库里的 Ralph Wiggum 插件提供了真实例子。源码锚点：

- `anthropics/claude-code/plugins/ralph-wiggum/hooks/hooks.json`

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh"
          }
        ]
      }
    ]
  }
}
```

脚本检查循环状态。如果任务还未满足结束条件，就阻止 Stop，让主循环继续。这里能看到插件怎样使用公开 Hook 协议，但仍看不到 Claude Code 核心怎样派发这个事件。

### Hermes Hooks：事件通知与少量阻断点

Hermes 的 `VALID_HOOKS` 包含 tool、LLM、API、session、approval、Gateway、subagent 和 Kanban 生命周期。源码锚点：

- `NousResearch/hermes-agent/hermes_cli/plugins.py::VALID_HOOKS`

其中常见的几类是：

- `pre_tool_call` / `post_tool_call`：工具执行前后。
- `pre_llm_call` / `post_llm_call`：每轮模型调用附近。
- `pre_api_request` / `post_api_request` / `api_request_error`：Provider 请求边界。
- `on_session_start` / `on_session_end` / `on_session_reset`：会话生命周期。
- `pre_approval_request` / `post_approval_response`：审批观测。
- `subagent_start` / `subagent_stop`：委派生命周期。
- `pre_gateway_dispatch`：外部消息进入 Gateway 后、正式分派前。

并不是所有 Hook 都能修改控制流。比如 approval hooks 是 observer，返回值会被忽略；想阻断工具，应使用 `pre_tool_call`。

### Hermes middleware：包住请求或真实执行

Hermes 另有四种 middleware：

```python
TOOL_REQUEST_MIDDLEWARE = "tool_request"
TOOL_EXECUTION_MIDDLEWARE = "tool_execution"
LLM_REQUEST_MIDDLEWARE = "llm_request"
LLM_EXECUTION_MIDDLEWARE = "llm_execution"
```

request middleware 修改将要发送的 payload。例如 tool request middleware 可以重写参数，后面的 Hook、guardrail 和审批看到的都是修改后版本。

execution middleware 使用 `next_call` 包住真实调用，适合重试、缓存、计时、熔断或统一错误处理。

```text
request middleware：改“要执行什么”
pre Hook：决定“是否继续”或记录事件
execution middleware：包住“怎样执行”
post Hook：观察“执行结果是什么”
```

Claude Code 的 Hooks 覆盖面很大，部分能力相当于 Hermes 的 Hook 加 middleware。Hermes 把 observer、request transform 和 execution wrapper 分开，插件作者需要更准确地选择扩展点。

## 功能 7：Skills、plugins 与 MCP 怎样分工

### Claude Code：Skills 是按需进入上下文的流程

Claude Code 的 [Skills](https://code.claude.com/docs/en/skills) 使用 `SKILL.md`。启动时模型只看到可调用 skill 的描述，完整正文在调用后才进入上下文。

这就是 progressive disclosure：

```text
启动：skill name + description
命中任务：加载 SKILL.md 正文
需要细节：再读取 reference、examples 或运行 scripts
```

Skill frontmatter 可以控制：

- 模型能否自动调用。
- 用户能否从 `/` 菜单调用。
- skill 激活时预批准或禁用哪些工具。
- 临时使用哪个模型和 effort。
- 在主上下文运行，还是 `context: fork` 到子代理。
- 只对哪些路径生效。
- skill 生命周期内启用哪些 Hooks。

这已经不是一段静态 prompt，而是“带元数据、作用域和资源目录的可执行工作流说明”。

### Claude Code plugins：分发容器

Plugin 可以打包：

```text
plugin/
  .claude-plugin/plugin.json
  skills/
  agents/
  hooks/
  .mcp.json
  .lsp.json
  monitors/
```

Plugin 自身不是一种新的推理机制。它负责把 skills、agents、Hooks、MCP、LSP 等扩展一起安装、命名和更新。

### MCP：能力协议，不是 prompt 文件

MCP server 对外暴露工具和资源。Claude Code 可以连接 stdio、HTTP 等 MCP，插件也可以自带 `.mcp.json`。模型看到的是 MCP 工具 schema，调用后由 MCP 客户端和 server 通信。

所以三者的边界是：

- Skill 告诉模型一项任务应该怎样做。
- Plugin 负责分发一组扩展。
- MCP 把外部系统能力接成工具或资源。

### Hermes 的对应设计

Hermes 同样有 skills、plugins 和 MCP，但还多一层 Tool Registry/toolset：

```text
Skill：按需加载的知识与流程
Plugin：注册 Hook、middleware、工具、平台或 memory provider
MCP：把外部 server 的工具动态注册进 Registry
Toolset：决定一组工具是否向当前模型暴露
Tool Search：工具太多时，先搜索再延迟暴露
```

源码锚点：

- `NousResearch/hermes-agent/tools/skills_tool.py`
- `NousResearch/hermes-agent/tools/skill_manager_tool.py`
- `NousResearch/hermes-agent/hermes_cli/plugins.py`
- `NousResearch/hermes-agent/tools/mcp_tool.py`
- `NousResearch/hermes-agent/tools/registry.py`
- `NousResearch/hermes-agent/tools/tool_search.py`

Hermes 面临的特殊压力是工具数量和 Provider 兼容性。不是每个模型都能稳定处理上百个 schema，因此它需要 toolset、动态可用性检查、Tool Search scope gate 和 schema sanitizer。

## 功能 8：子代理怎样隔离上下文与权限

### Claude Code：子代理是一份可配置的独立运行环境

Claude Code 的 `.claude/agents/<name>.md` 用 YAML frontmatter 定义名称、描述、工具、模型、权限、skills、MCP、memory、后台模式和 worktree 隔离。Markdown 正文是子代理自己的 system prompt。

官方 [Subagents](https://code.claude.com/docs/en/sub-agents) 文档说明，普通子代理从新的上下文窗口开始，不自动继承主会话的完整消息历史。主代理会生成一条 delegation message。子代理完成后，只把结果摘要和元数据送回主会话。

```text
主会话
  -> 根据 description 选择 Agent
  -> 生成 delegation message
  -> 子代理独立读取文件、调用工具
  -> 返回结果
  -> 主会话继续
```

当前版本还支持：

- background subagent。
- 恢复已有子代理历史。
- 子代理按固定深度继续派生子代理。
- `isolation: worktree` 创建隔离工作树。
- 子代理自己的 memory scope。
- 通过 `tools`、`disallowedTools` 和 permissionMode 限制能力。

公开仓库中的 `feature-dev` 插件给出了子代理定义示例。源码锚点：

- `anthropics/claude-code/plugins/feature-dev/agents/code-explorer.md`

```yaml
---
name: code-explorer
description: Deeply analyzes existing codebase features...
tools: Glob, Grep, LS, Read, NotebookRead, WebFetch, WebSearch
model: sonnet
---
```

这段文件可以证明 Agent 定义格式和插件用法，不能证明核心调度算法。

### Hermes：delegate_task 在主 Agent 内部创建 AIAgent 子实例

Hermes 的入口是 `delegate_task`。源码锚点：

- `NousResearch/hermes-agent/tools/delegate_tool.py::delegate_task`
- `NousResearch/hermes-agent/tools/async_delegation.py`

```python
def delegate_task(
    goal=None,
    context=None,
    tasks=None,
    role=None,
    background=None,
    parent_agent=None,
) -> str:
    ...
```

它支持单任务与批量 fan-out，并在运行时执行这些约束：

- `delegation.max_concurrent_children` 限制并发数量。
- `delegation.max_spawn_depth` 限制树深。
- 模型传入的 `max_iterations` 不覆盖用户配置预算。
- 子代理默认继承父 Agent 的 toolsets，模型不能随意扩大工具集合。
- `leaf` 不能继续委派，`orchestrator` 可以在深度限制内派生 worker。
- 顶层模型触发的委派默认进入后台，完成结果通过队列重新进入主对话。
- `/stop` 和 Gateway shutdown 可以取消后台委派。

Hermes 的子代理仍然使用 `AIAgent`，因此可以继承 Provider、上下文引擎、工具系统和插件生命周期。Kanban 与 worktree 则提供另一条面向长期任务的编排路径。

### 设计差异

Claude Code 把“子代理定义文件”做成一等产品配置，用户可以细粒度指定工具、模型、memory、Hooks 和 worktree。Hermes 的 delegate 更偏运行时工具和统一配置，角色主要是 leaf/orchestrator，工具继承策略更集中。

设计 coding agent 时，Claude Code 的 per-agent 权限和 worktree isolation 是直接相关的参考；设计跨 Provider、跨平台的通用 Agent 时，Hermes 的统一 `AIAgent` 子实例和后台回流机制更容易复用既有运行时。

## 功能 9：上下文压缩、检查点与恢复不是同一件事

### Claude Code：compact 管上下文，checkpoint 管文件和会话回退

Claude Code 的 `/compact` 会用摘要替换上下文中的长历史。官方 [Context window](https://code.claude.com/docs/en/context-window) 文档说明，压缩后不同内容的处理不同：

- system prompt 保持不变。
- 项目根 `CLAUDE.md` 和 auto memory 从磁盘重新注入。
- 子目录 `CLAUDE.md` 等待再次读取相关文件时恢复。
- 已调用 skill 的正文按预算重新附加。
- Hooks 是代码，不属于上下文消息。

Checkpoint 则记录 Claude 文件编辑工具造成的修改。`/rewind` 可以恢复代码、恢复对话，或从某个位置定向摘要。Bash 命令改过的文件不在文件 checkpoint 的完整保护范围内。

因此：

```text
compact：减少送给模型的 token
checkpoint：保存可回退状态
resume：重新打开持久 session
fork：从历史复制新分支
```

### Hermes：ContextCompressor 保护头尾，摘要中间

Hermes 默认 `ContextCompressor` 的算法写在类注释中。源码锚点：

- `NousResearch/hermes-agent/agent/context_compressor.py::ContextCompressor`

```python
class ContextCompressor(ContextEngine):
    """Default context engine.

    1. Prune old tool results
    2. Protect head messages
    3. Protect recent tail by token budget
    4. Summarize middle turns
    5. Update the previous summary on later compactions
    """
```

它还处理：

- tool call/tool result 配对，防止压缩后产生孤儿 id。
- 摘要失败 cooldown，避免每轮反复触发同一个失败。
- 手动 `/compress` 强制重试。
- 按 `focus_topic` 引导摘要保留某类信息。
- session 结束时清理 `_previous_summary`，防止摘要跨会话污染。
- 在 SessionDB 中保存与恢复压缩失败状态。

Hermes 也有 checkpoint manager、session branch 和 undo 路径，但它们没有被收敛成 Claude Code `/rewind` 那样统一的交互入口。两边底层问题相同：历史、模型上下文和工作区文件是三种不同状态，不能只恢复其中一种却假装全部回到了过去。

## 功能 10：远程入口和消息平台怎样接入正在运行的 Agent

### Claude Code Remote Control：远程界面控制本地会话

官方 [Remote Control](https://code.claude.com/docs/en/remote-control) 文档说明，Claude Code 进程仍在本机运行，Web 或手机只是连接到该本地 session。文件、MCP、工具和项目配置都留在本机。连接由本地主动发起出站 HTTPS，不需要开放入站端口。

它解决的是：人离开终端后，继续操作同一条本地开发会话。

### Claude Code Channels：外部事件推入正在运行的 session

官方 [Channels](https://code.claude.com/docs/en/channels-reference) 把 channel 定义为带特殊能力的 MCP server。Channel server 在本机作为子进程运行，通过 stdio 与 Claude Code 通信，再从 Telegram、Discord、iMessage 或 webhook 接收事件。

```text
外部聊天/CI/webhook
  -> Channel MCP server
  -> notification 推入当前 Claude Code session
  -> Claude 处理
  -> 可选 reply tool 回复外部系统
```

标准 MCP 通常由模型主动查询；Channel 允许外部事件主动推入。它还需要 sender gating，避免任何人都能向高权限本地 Agent 注入指令。

### Hermes Gateway：平台入口、会话路由和运行时调度是一体的

Hermes 的 Gateway 不是给某一条 CLI session 加远程窗口。它本身就是长期服务入口，负责：

- 加载平台 adapter。
- 鉴权、pairing、allowlist 和群聊策略。
- 构建确定性 session key。
- 创建或恢复 AIAgent。
- 处理同 session 的 queue、steer、interrupt。
- 把流式事件转换成平台消息编辑、分段和最终回复。
- 路由危险命令审批。
- 管理 `/reset`、`/resume`、`/model`、`/compress`、`/stop` 等命令。
- 重启后恢复 session routing。

源码锚点：

- `NousResearch/hermes-agent/gateway/run.py`
- `NousResearch/hermes-agent/gateway/session.py`
- `NousResearch/hermes-agent/gateway/stream_consumer.py`
- `NousResearch/hermes-agent/gateway/slash_commands.py`
- `NousResearch/hermes-agent/plugins/platforms/`

### 三者不要混淆

| 能力 | Claude Remote Control | Claude Channels | Hermes Gateway |
| --- | --- | --- | --- |
| 主要目的 | 人远程控制本地 coding session | 外部事件推入运行中 session | 长期承载多个平台和多个 session |
| 运行位置 | Claude Code 仍在本机 | Claude Code 与 Channel server 在本机 | Gateway 服务所在机器 |
| 会话路由 | 连接已有本地 session | 绑定当前运行 session | 平台/聊天/thread/用户到 session 的正式路由 |
| 外部回复 | Web/移动端同步会话 | reply tool 可回外部渠道 | adapter 原生发送、编辑、上传和审批 |
| 典型生命周期 | 用户显式开启 | 插件/MCP 子进程随 session | 常驻服务，可跨进程恢复 |

## 功能 11：开源程度怎样影响学习方式

Claude Code 的外部协议已经覆盖多个控制点。从公开资料可以确认：

- 配置文件和作用域。
- tool permission 语义。
- Hook 输入输出协议。
- Skills、plugins、subagents、MCP 的组合方式。
- session、compact、checkpoint、Remote Control 和 Channels 的产品行为。
- Agent SDK 怎样把选项翻译成 CLI 参数，怎样处理 stream-json 控制消息。

但下面这些问题不能从公开仓库得出确定答案：

- Claude Code 核心 Agent loop 的具体类和状态机。
- 内置工具真正的 dispatch 数据结构。
- permissions 内部匹配器和 sandbox 调度的完整调用链。
- compact 摘要 prompt 的全部实现。
- Remote Control 服务端协议的内部实现。

Hermes 则可以继续向下追：

```text
run_agent.py::AIAgent
  -> agent/conversation_loop.py
  -> agent/tool_executor.py
  -> tools/registry.py
  -> 具体 handler
  -> SessionDB / Gateway / plugin hooks
```

学习 Claude Code 时，应把重点放在公开契约与产品抽象；学习 Hermes 时，可以检验抽象在真实源码中怎样落地。把 Claude Agent SDK 当成 Claude Code 核心源码，会得到错误的架构图。

## 一张总表

| 维度 | Claude Code | Hermes |
| --- | --- | --- |
| 第一使用场景 | 本地与云端 coding agent | 多入口通用 Agent |
| 核心源码 | 未完整公开 | 主运行时开源 |
| 项目指令 | `CLAUDE.md` 层级与 rules | `.hermes.md` 优先，兼容多种规则文件 |
| 长期记忆 | auto memory 文件目录 | 有界 `MEMORY.md` + `USER.md` 冻结快照，可插拔 memory provider |
| 工具扩展 | 内置工具 + MCP | Registry + toolset + plugin + MCP + Tool Search |
| 权限 | allow/ask/deny + permission mode | 危险模式、deny、manual/smart/off、界面审批 |
| 沙箱 | Bash 的 OS 级文件/网络隔离 | 可选 Local/SSH/Docker/Modal 等环境 backend |
| 扩展事件 | Hooks | Hooks + request/execution middleware |
| Skills | Agent Skills，正文按需加载 | Skills，结合 toolset 与 skill 工具管理 |
| 子代理 | Markdown 定义，细粒度工具/模型/memory/worktree | `delegate_task` 创建 AIAgent 子实例，统一预算、深度与后台回流 |
| 会话 | 项目本地 transcript、resume/branch | SessionDB + Gateway routing |
| 压缩 | auto compact、`/compact`、定向 summarize | ContextCompressor 保护头尾、迭代摘要中段 |
| 回退 | checkpoint + `/rewind` | checkpoint、undo、session branch 等多条路径 |
| 远程与消息 | Remote Control + Channels | Gateway + platform adapters |

## 怎样把这两套设计用到自己的 Agent

如果要做 coding agent，可以优先吸收 Claude Code 的这些做法：

- 把项目指令、Skills、子代理和权限做成可提交的项目配置。
- 区分工具可见性、自动批准和 OS 沙箱。
- 给文件编辑建立 checkpoint，不把聊天历史当成唯一状态。
- 子代理使用独立上下文和可选 worktree，减少主上下文污染。
- 把 compact 后哪些内容重注入写成明确契约。

如果要做长期在线的通用 Agent，可以优先吸收 Hermes 的这些做法：

- 会话身份由来源数据确定，不依赖模型猜测用户是谁。
- Gateway 把平台协议、session routing 和 Agent runtime 分层。
- memory 使用冻结快照，工具写入不在同一 session 中偷偷改变 system prompt。
- Tool Registry 之外保留统一执行管线，集中处理 Hook、guardrail、耗时和结果限制。
- Provider、工具、平台和 memory provider 都允许替换，不把产品绑死在单一模型 API。

更完整的实现往往会混合两者：Claude Code 式的工作区权限与 checkpoint，加上 Hermes 式的多平台会话路由、动态工具系统和长期记忆。

## 常见问答

### Q1：Claude Code 是开源的吗？

公开仓库不等于核心运行时完整开源。`anthropics/claude-code` 提供插件、示例、脚本、变更记录和 issue；Claude Agent SDK 的宿主代码有源码；Claude Code CLI 核心实现不能像 Hermes 一样完整沿源码阅读。

### Q2：`CLAUDE.md` 就是 system prompt 吗？

不是。它是项目指令。Claude Code 官方文档明确说明，`CLAUDE.md` 作为 system prompt 后的用户消息加载。Hermes 则会把命中的项目上下文组装进 system prompt 的 Project Context 区域。两者都能强烈影响模型，但消息角色和生命周期不同。

### Q3：用了 Hook 后，还需要 permission 吗？

需要。Hook 是扩展判断，permission 是统一授权策略。`PreToolUse` 可以增加或收紧决策，但不应把组织安全全部押在一段容易失效的 Hook 脚本上。OS sandbox 又是第三层，它约束命令真正能访问的资源。

### Q4：Skills 和 `CLAUDE.md` 应该怎样分工？

稳定且普遍适用的项目事实放 `CLAUDE.md`。只有执行某类任务时才需要的多步骤流程放 Skill。Claude Code 的 Skill 正文按需加载；Hermes 也通过 skill 工具避免把全部流程永久塞入 system prompt。

### Q5：模型什么时候会调用某个 Skill 或工具？

模型根据 description、当前任务和上下文决定。运行时通过 schema、可见性和调用限制影响选择。必须强制执行的规则要放在 permissions、Hook、guardrail 或 sandbox 中，不能只写“请记得调用”。

### Q6：Claude Code Channels 和 Hermes 的 Telegram adapter 是一回事吗？

不是。Channel 是把外部事件推入当前 Claude Code session 的 MCP 扩展；Hermes adapter 属于常驻 Gateway 的正式平台层，还参与身份、会话路由、并发策略、消息编辑和重启恢复。

### Q7：为什么 Hermes 的项目上下文要做 injection 扫描？

因为 Hermes 把项目上下文和 memory 放在 system prompt 的敏感位置。扫描由确定性 threat pattern 执行，能阻止一部分常见注入与外泄指令。它不是完整安全证明，所以仍要配合工具权限、URL/path 安全和执行隔离。

### Q8：哪一套子代理设计更好？

要看产品。Claude Code 的 per-agent Markdown 配置、memory 和 worktree 很适合团队 coding 流程；Hermes 的统一 AIAgent 子实例、Provider 继承和后台回流适合跨平台常驻 Agent。评价时应看上下文隔离、权限继承、预算、取消和结果回流，不要只看“能否并行”。

## 读源码时应留下的判断框架

同一组问题也适用于其他 Agent 框架：

1. 会话、一次运行和一次 tool call 是否有不同 id。
2. 项目指令进入哪种消息角色，何时加载，压缩后怎样恢复。
3. memory 是模型随意写的文本，还是有预算、作用域、审计和写入保护。
4. 工具的可见性、授权、执行和结果处理是否分层。
5. Hook 是观察器、阻断器、参数变换器，还是执行 wrapper。
6. sandbox 是提示词承诺、应用层检查，还是 OS/容器强制边界。
7. 子代理是否隔离上下文，怎样继承工具与权限，失败后怎样回流。
8. 外部消息如何绑定用户与 session，重启后是否仍能恢复。
9. compact、resume、fork、checkpoint 分别修改哪一种状态。
10. 哪些结论来自源码，哪些只来自文档或黑盒行为。

这组问题比功能清单更有用。功能名经常相同，状态边界和失败路径才决定一个 Agent 是否可维护。

## 参考资料

Claude Code 官方文档：

- [Manage sessions](https://code.claude.com/docs/en/sessions)
- [How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [Explore the context window](https://code.claude.com/docs/en/context-window)
- [Configure permissions](https://code.claude.com/docs/en/permissions)
- [Sandboxing](https://code.claude.com/docs/en/sandboxing)
- [Hooks reference](https://code.claude.com/docs/en/hooks)
- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing)
- [Remote Control](https://code.claude.com/docs/en/remote-control)
- [Channels reference](https://code.claude.com/docs/en/channels-reference)

公开仓库：

- [anthropics/claude-code](https://github.com/anthropics/claude-code)
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
