# Hermes-learn

这是一个面向 Hermes Agent 工程实现的中文学习笔记仓库。

目标不是复述文档，也不是整理一份源码目录说明，而是沿着 Hermes 的真实工程路径，把一个 Agent 框架如何组织 prompt、上下文、工具、记忆、技能、会话状态、安全边界和多 Agent 协作讲清楚。

## 这个仓库适合谁

- 想系统理解 Hermes Agent 设计的人。
- 想学习 Agent 工程实现，而不只停留在 ReAct、Function Calling 概念层的人。
- 想比较 Hermes、Codex、Claude Code 等 Agent 框架差异的人。
- 想以后自己实现 Agent runtime、工具系统、记忆系统或技能系统的人。

## 学习方式

每篇笔记围绕一个或几个功能点展开：

1. 先讲这个功能解决什么问题。
2. 再定位到 Hermes 源码里的关键入口。
3. 选取必要源码片段，不大段搬运。
4. 解释代码背后的工程取舍。
5. 对比 Codex、Claude Code 或其他 Agent 框架在同类问题上的处理方式。
6. 补充读者容易误解的问题。

笔记会尽量避免“源码逐行翻译”。真正重要的是理解：为什么这里要这样拆层，为什么这个状态要落库，为什么某些上下文不能进入 system prompt，为什么工具调用要先写入会话历史，再执行工具。

## 当前目录

```text
docs/
  stage-0/
    00-全局架构地图.md
  modules/
    01-agent-loop.md
    02-prompt-builder.md
    03-tool-registry-and-toolsets.md
    04-tool-dispatch-and-failure.md
    05-session-state-and-search.md
    06-context-compression.md
```

## 已完成内容

- 全局架构地图：先建立 Hermes 的主线认知。
- Agent Loop：一轮对话如何从用户输入推进到模型调用、工具调用和最终回复。
- Prompt Builder：模型到底看到了什么，system prompt 如何分层、缓存、持久化。
- Tool Registry 与 Toolsets：工具如何注册、分组、暴露给模型。
- Tool Call 分支与失败处理：模型发起工具调用以后，Hermes 如何校验、执行、截断和回收结果。
- SessionDB 与 session_search：会话如何持久化、恢复、压缩，并被模型按需检索。
- Context Compression：上下文太长以后，Hermes 如何触发压缩、生成摘要、保留尾部并接回会话状态。

## 后续主题

后续笔记会继续覆盖：

- Tool Registry 与 Toolsets
- Tool Call 分支与失败处理
- SessionDB 与 session_search
- Context Compression
- Memory System
- Skills System
- Provider / Transport
- Security 与 Sandbox
- MCP 与 Plugins
- Browser / Web / Terminal Tools
- Multi-Agent 与 Kanban
- Cron / Automation
- CLI / TUI / Gateway / Dashboard

## 写作原则

- 中文讲解，面向学习者。
- 引用源码使用仓库相对路径。
- 不暴露本机路径。
- 不复制大段源码，只保留理解设计所需的关键片段。
- 不把文档写成聊天记录，只沉淀真正有技术价值的问题。
- 不写空泛总结，尽量回到具体函数、数据结构和运行流程。

## 说明

这个仓库是非官方学习笔记。Hermes Agent 的实现可能继续变化，笔记会尽量跟随当前源码整理，但不替代官方文档和源码本身。
