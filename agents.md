# Hermes-learn 协作说明

这个项目不是为了把 Hermes 文档重新抄一遍。目标很直接：把 Hermes 当前版本的设计和工程实现学明白，能讲清楚，能定位源码，能回答别人追问，也能把里面的做法迁移到自己的 Agent 项目里。

学习材料有三份：

- 架构文档：`E:\coding\Hermes-Wiki`
- Hermes 源码：`E:\coding\hermes-agent`
- 学习笔记：`E:\coding\Hermes-learn`

学习笔记默认使用中文。每次生成或修改笔记，都要按 `humanizer-zh` 的要求收一遍文风：少套话，少宣传腔，少三段式排比，少空泛总结。技术判断要直接，源码解释要具体。不要为了显得完整而写废话。

## 第一目标

第一目标是教会我理解 Hermes，不是为了写笔记而写笔记。

所以每篇笔记都要服务一个学习任务：

- 这个功能解决什么问题？
- Hermes 为什么这样设计？
- 文档里怎么说？
- 源码里怎么实现？
- 失败时会发生什么？
- 和 Codex、Claude Code 这类 Agent 工程相比，它的处理方式有什么不同？
- 我应该如何记住它，如何回答相关问题？

如果一段内容不能帮助理解、定位、解释或复现，就不要写。

## 笔记结构

笔记按“功能”组织，而不是机械按源码目录组织。一篇笔记可以讲多个相关功能，但每个功能都要紧跟源码和实现解释。

推荐结构如下：

```md
# 主题标题

## 这篇先解决什么问题

用几句话说明本篇要学会什么。不要写成宣传简介。

## 功能 1：功能名

### 先讲人话

这个功能是做什么的，为什么 Agent 系统需要它。

### 文档设计

对应 Hermes-Wiki 的文档路径。只提炼设计意图，不长篇搬运。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |

### 关键源码

只贴必要片段。源码片段要短，贴完马上解释。

```python
# 必要时在代码中加中文注释
```

### 工程实现

按调用链讲清楚：输入是什么，经过哪些函数，状态怎么变化，输出是什么。

### 失败路径

说明异常、重试、降级、截断、安全拦截、上下文超限等边界。

### 和 Codex / Claude Code 的差异

只讲同一问题上的差异，不泛泛比较。

### 小实验

给一个可运行或可检查的实验，例如查 registry、模拟 tool call、读 SQLite、开关 toolset。

### 问答

把学习中出现的追问整理成 QA。

## 功能 2：功能名

同上。

## 本篇要记住的东西

压缩成真正有用的几条，不写空泛总结。
```

## 源码引用规则

笔记要包含源码，但不能变成源码搬运。

规则：

- 只贴关键片段，通常控制在 10 到 40 行。
- 大段逻辑用“路径 + 函数名 + 行号 + 调用链”说明。
- 如果源码片段较难读，可以在 Markdown 代码块里加中文注释。
- 解释优先于摘录。读者应该通过笔记理解源码，而不是被迫重新读一遍完整文件。
- 对外开源时，避免复制过长源码，降低版权和维护风险。

## 学习方式

每个功能按这个顺序走：

1. 先用白话讲问题。
2. 再看 Hermes-Wiki 的设计。
3. 到源码里找真实实现。
4. 画出调用链。
5. 解释关键数据结构和状态变化。
6. 看失败路径和保护逻辑。
7. 对比 Codex、Claude Code。
8. 补一个小实验。
9. 把学习中的问题整理成 QA。

我会在讲解时不断提醒哪些是“必须懂的主干”，哪些是“以后深入再看”。Hermes 很大，不能一开始就铺满所有细节。

## 用户偏好

已确认：

- 笔记要兼顾“能改源码”和“能讲清架构”。
- 对比框架重点关注 Codex 和 Claude Code。
- 避免超长源码搬运。
- 每章可以加入可运行实验。
- 学习过程中提出的问题，可以优化语言后写进文档的 QA。
- 总计划要放在本文件末尾，用勾选标记进度。每次完成后更新勾选状态。

## 当前已确认的 Hermes 源码地图

| 模块 | 主要路径 | 学习意义 |
| --- | --- | --- |
| Agent 入口 | `run_agent.py` | `AIAgent` 外观类，连接初始化、对话循环、工具调用、状态回调 |
| Agent 内核 | `agent/` | prompt、conversation loop、压缩、provider 适配、运行时修复 |
| 工具注册 | `tools/registry.py` | 工具自注册、schema 暴露、handler 调度 |
| 工具编排 | `model_tools.py` | 工具发现、tool definitions、同步/异步桥接、tool call 执行 |
| 工具集 | `toolsets.py` | 工具分组、递归解析、启用/禁用策略 |
| 会话状态 | `hermes_state.py` | SQLite、FTS5、session、message、压缩、分支、搜索 |
| Provider | `providers/`、`agent/transports/` | 多模型供应商、API 差异、认证、fallback |
| CLI | `cli.py`、`hermes_cli/` | 命令层、配置、doctor、setup、gateway 管理 |
| Gateway | `gateway/` | Telegram、Discord、Slack、WhatsApp、API server 等入口 |
| 技能 | `skills/`、`optional-skills/`、`tools/skills_*` | procedural memory、自我改进、技能生态 |
| 插件/MCP | `plugins/`、`tools/mcp_tool.py` | 外部能力扩展、工具动态发现、OAuth、插件 hook |
| 多 Agent | `tools/delegate_tool.py`、`tools/async_delegation.py`、`hermes_cli/kanban_db.py` | 子 Agent、后台任务、Kanban、worktree 隔离 |
| 安全 | `tools/approval.py`、`tools/url_safety.py`、`tools/path_security.py` | 命令审批、路径防护、SSRF、注入和敏感信息处理 |

## 重点对比对象

### Codex

重点比较这些问题：

- 工具调用如何暴露给模型。
- sandbox 和权限边界在哪里。
- 上下文压缩如何触发。
- 文件编辑如何验证。
- 多轮工具调用失败后如何恢复。
- session 和任务状态由谁管理。

Codex 更偏软件工程任务环境，很多能力由宿主运行时和工具面提供。Hermes 更像一个长期在线的 Agent 产品，自己维护更多状态、平台和扩展能力。

### Claude Code

重点比较这些问题：

- CLI 交互和项目上下文如何进入 prompt。
- 文件读写、命令执行、审批、安全边界如何处理。
- 长任务里如何压缩上下文和恢复状态。
- skills、memory、hooks、MCP 这类扩展点如何组织。

Claude Code 更偏开发者本地工作流。Hermes 同时覆盖本地 CLI、远程网关、定时任务、多 Agent 协作和跨平台消息投递。

## 文风要求

每篇笔记都按 `humanizer-zh` 的思路检查：

- 不写“标志着、体现了、至关重要、深刻、赋能、生态闭环”这类空泛词。
- 不写“这不仅是……更是……”这种模板句。
- 不用三段式排比假装完整。
- 不在结尾写万能总结。
- 多写具体函数、具体路径、具体状态变化。
- 可以承认复杂性。看不确定的地方就标出来，后面补证据。
- 讲源码时像人在带读代码，不像百科条目。

## QA 规则

学习过程中你的问题会进入文档，但要先做轻微整理：

- 把口语问题改成更适合检索的标题。
- 保留原问题的意思，不改变关注点。
- 答案要回到源码和设计，不只给概念解释。
- 如果问题能引出对比，就补 Codex / Claude Code 的处理差异。

示例：

原问题：`为什么 Hermes 这里工具搞得这么绕？`

整理为：

```md
### 为什么 Hermes 不把所有工具写在一个静态列表里？

答：……
```

## 小实验规则

每个重要模块尽量有一个小实验。实验不追求复杂，但要能帮助理解。

可选形式：

- 用 `rg` 定位某个函数的调用链。
- 打印当前 registry 里有哪些工具。
- 启用/禁用某个 toolset，看 tool definitions 的变化。
- 构造一个假的 tool call，观察 `handle_function_call` 怎么分发。
- 打开 `state.db` 看 sessions/messages 的结构。
- 模拟工具结果过长，看 Hermes 如何截断或持久化。
- 模拟 provider 报错，看 fallback 或错误分类。

实验要写清楚：运行什么、预期看到什么、这个现象说明什么。

## 总计划

- [x] 建立 `E:\coding\Hermes-learn` 项目目录。
- [x] 阅读并盘点 `E:\coding\Hermes-Wiki` 的文档结构。
- [x] 阅读并盘点 `E:\coding\hermes-agent` 的源码结构。
- [x] 写入第一版 `agents.md`。
- [x] 写入 `docs/stage-0/00-全局架构地图.md`。
- [ ] 重写 `docs/stage-0/00-全局架构地图.md`，按最终功能式结构调整。
- [ ] 编写 `docs/stage-0/01-源码目录导读.md`。
- [ ] 编写 `docs/stage-0/02-一次对话从输入到工具调用再到回复.md`。
- [ ] 编写 `docs/modules/01-agent-loop.md`：对话循环、iteration budget、停止条件、空响应恢复。
- [ ] 编写 `docs/modules/02-prompt-builder.md`：system prompt、context files、skills、memory、platform hints。
- [ ] 编写 `docs/modules/03-tool-registry-and-toolsets.md`：工具自注册、toolset 解析、动态可用性。
- [ ] 编写 `docs/modules/04-tool-dispatch-and-failure.md`：tool call 执行、异步桥接、失败守卫、结果截断。
- [ ] 编写 `docs/modules/05-session-state-and-search.md`：SessionDB、SQLite、FTS5、session parent chain。
- [ ] 编写 `docs/modules/06-context-compression.md`：自动压缩、手动压缩、summary、prompt cache。
- [ ] 编写 `docs/modules/07-memory-system.md`：长期记忆、MemoryStore、MemoryProvider、用户画像。
- [ ] 编写 `docs/modules/08-skills-system.md`：技能发现、技能使用、技能沉淀、自我改进。
- [ ] 编写 `docs/modules/09-provider-transport.md`：ProviderProfile、API 差异、fallback、credential pool。
- [ ] 编写 `docs/modules/10-security-and-sandbox.md`：审批、路径安全、URL 安全、prompt injection、敏感信息。
- [ ] 编写 `docs/modules/11-mcp-and-plugins.md`：MCP 工具、插件 hook、OAuth、动态发现。
- [ ] 编写 `docs/modules/12-browser-web-terminal-tools.md`：浏览器、Web、终端、代码执行工具。
- [ ] 编写 `docs/modules/13-multi-agent-and-kanban.md`：delegate、async delegation、Kanban、worktree。
- [ ] 编写 `docs/modules/14-cron-and-automation.md`：调度、平台投递、无人值守任务。
- [ ] 编写 `docs/modules/15-gateway-cli-tui-dashboard.md`：CLI、TUI、Gateway、Dashboard、Desktop。
- [ ] 编写 `comparisons/codex-vs-hermes.md`。
- [ ] 编写 `comparisons/claude-code-vs-hermes.md`。
- [ ] 编写 `references/papers-and-concepts.md`：ReAct、Toolformer、Reflexion、Voyager、MemGPT、RAG、context compression、function calling、sandboxing、multi-agent。
- [ ] 每完成一篇笔记，回到本计划打勾，并补充新的 QA。
