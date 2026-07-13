# Hermes-learn

一套面向 Hermes Agent 当前源码的中文工程笔记。

我按真实运行链整理 Prompt、工具调用、会话状态、上下文压缩、长期记忆、安全控制、多 Agent 协作和产品入口，并在相同问题上对照 Codex 与 Claude Code。文档不按源码目录逐项复述，也不把 Agent 简化成“模型加几个工具”。

课程状态：第一版已经完成，编号从 00 到 20，共 21 篇。主要分析的 Hermes 源码快照是 [`NousResearch/hermes-agent@590a193`](https://github.com/NousResearch/hermes-agent/commit/590a19332e898fc9bda55a31999926572d8fbc26)。

## 适合谁读

- 已经了解大模型基础概念，想继续学习 Agent Runtime 工程实现的开发者。
- 正在设计工具系统、记忆系统、会话恢复或多 Agent 调度的人。
- 需要讲清 Hermes 架构，并能继续回答源码追问的人。
- 想比较 Hermes、Codex、Claude Code 工程取舍的人。

## 整理方式

每篇笔记从一个具体工程问题出发，再定位 Hermes 的源码入口。源码只摘取理解控制流所需的短片段，后面记录状态变化、设计原因和失败路径。

笔记主要回答四类问题：

1. 输入由谁接收，下一步由谁决定。
2. 哪些状态只活在当前调用中，哪些状态会跨 turn 或跨 session 保存。
3. 模型输出怎样变成受约束的真实副作用。
4. Provider、工具或后台任务失败后，系统怎样恢复和收尾。

源码锚点全部使用 Hermes 仓库相对路径，可以直接在对应源码快照中追踪定义和调用方。

## 完整阅读顺序

建议按编号阅读。00 先建立全局地图，01 到 15 拆解主要模块，16 到 18 补框架对比和论文背景，19 回到源码目录，20 再把所有模块串成一次完整对话。

### 起点

- [第 00 讲：Hermes 全局架构地图](docs/stage-0/00-全局架构地图.md)：先分清宿主入口、Agent Runtime、Provider、工具、状态存储和扩展系统。

### 核心运行时

- [第 01 讲：Agent Loop](docs/modules/01-agent-loop.md)：一次用户输入如何经过多次模型迭代，直到得到最终回复。
- [第 02 讲：Prompt Builder](docs/modules/02-prompt-builder.md)：system prompt、项目上下文、Memory、Skills 和平台提示怎样进入模型请求。
- [第 03 讲：Tool Registry 与 Toolsets](docs/modules/03-tool-registry-and-toolsets.md)：工具怎样注册、分组、筛选并暴露给模型。
- [第 04 讲：Tool Call 分支与失败处理](docs/modules/04-tool-dispatch-and-failure.md)：tool call 从协议输入变成真实 handler 调用前，要经过哪些检查。
- [第 05 讲：SessionDB 与 session_search](docs/modules/05-session-state-and-search.md)：会话怎样保存、恢复、搜索和维护父子关系。

### 上下文、记忆与模型接入

- [第 06 讲：Context Compression](docs/modules/06-context-compression.md)：上下文超限前怎样生成摘要、保留尾部并接回原会话。
- [第 07 讲：Memory System](docs/modules/07-memory-system.md)：用户偏好和长期事实怎样写入、扫描并重新进入 system prompt。
- [第 08 讲：Skills System](docs/modules/08-skills-system.md)：可复用做事方法怎样发现、加载和沉淀。
- [第 09 讲：Provider 与 Transport](docs/modules/09-provider-transport.md)：不同模型供应商的请求格式、认证、流式响应和 fallback 怎样被统一。
- [第 10 讲：Security 与 Sandbox](docs/modules/10-security-and-sandbox.md)：审批、路径安全、URL 安全、提示词注入和敏感信息分别在哪一层处理。

### 扩展能力与产品形态

- [第 11 讲：MCP 与插件](docs/modules/11-mcp-and-plugins.md)：外部工具、OAuth 和生命周期 Hook 怎样接入核心运行时。
- [第 12 讲：浏览器、网页、终端与代码执行](docs/modules/12-browser-web-terminal-tools.md)：四类高频工具的职责边界、执行方式和风险控制。
- [第 13 讲：多 Agent 与 Kanban](docs/modules/13-multi-agent-and-kanban.md)：短委派、后台任务、持久队列和 worktree 隔离怎样配合。
- [第 14 讲：Cron 和自动化任务](docs/modules/14-cron-and-automation.md)：定时任务怎样创建、触发、执行并投递结果。
- [第 15 讲：CLI、TUI、Gateway、Dashboard 与 Desktop](docs/modules/15-gateway-cli-tui-dashboard.md)：不同入口如何复用同一个 Agent 内核，又各自维护交互状态。

### 对比、理论与总复盘

- [第 16 讲：Codex 与 Hermes](comparisons/codex-vs-hermes.md)：比较会话模型、工具协议、安全边界、上下文管理和多 Agent 能力。
- [第 17 讲：Claude Code 与 Hermes](comparisons/claude-code-vs-hermes.md)：比较项目上下文、Hooks、权限、Skills、MCP 和会话压缩。
- [第 18 讲：从论文概念到 Hermes 工程实现](references/papers-and-concepts.md)：把 ReAct、Toolformer、Reflexion、Voyager、MemGPT 和 RAG 放回具体代码。
- [第 19 讲：Hermes 源码目录导读](docs/stage-0/01-源码目录导读.md)：按运行职责阅读仓库，而不是按文件夹名称猜用途。
- [第 20 讲：一次对话从输入到工具调用，再到最终回复](docs/stage-0/02-一次对话从输入到工具调用再到回复.md)：沿一条完整时序复盘前面所有模块，并解释持久化和异常收尾。

## 阅读方式

第一遍按功能主线确认输入、状态、输出和失败路径，不需要记住所有函数名。

第二遍按源码锚点追踪定义、调用方和持久化位置。只存在于函数局部变量中的状态通常只影响当前模型迭代；进入 Agent 实例、SessionDB 或 Memory 后，生命周期和恢复方式都会改变。

问题最好落到具体状态转换上。例如：“tool call 已经写入 SessionDB，但工具还没有执行，此时进程退出会怎样？”这类问题可以直接追踪持久化和副作用边界，也适合整理成可检索的问答。

## 仓库结构

```text
Hermes-learn/
├── docs/
│   ├── stage-0/       # 全局地图、源码导读、端到端复盘
│   └── modules/       # 15 个工程模块
├── comparisons/       # Codex、Claude Code 对比
├── references/        # 论文与工程概念
├── ABOUT.md           # 项目背景和内容边界
└── README.md          # 阅读入口
```

## 内容边界

这是非官方学习仓库，不替代 [Hermes Agent 源码](https://github.com/NousResearch/hermes-agent) 和官方文档。Hermes 仍在更新，函数位置、配置项和行为可能在新版本中变化。阅读具体结论时，请同时核对每篇标注的源码快照。

笔记中的 Hermes 源码只保留必要短片段，相关代码的权利和许可归原项目及贡献者所有。Codex 与 Claude Code 的对比只使用公开代码、公开协议或官方资料；无法从公开信息确认的内部实现不会当作事实描述。

## 参与修订

发现源码版本变化、锚点失效或解释错误，可以在 [Issues](https://github.com/teachershuang/Hermes-learn/issues) 中给出对应文件、源码版本和具体问题。学习过程中的提问也可以进入笔记，但会先整理成能解释源码边界、设计取舍或常见误解的问题。

项目背景和写作取向见 [ABOUT.md](ABOUT.md)。
