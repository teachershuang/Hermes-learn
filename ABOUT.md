# 关于 Hermes-learn

`Hermes-learn` 记录我阅读 Hermes Agent 源码时整理出的架构笔记。仓库中的结论来自具体源码、测试和公开协议，重点是运行时怎样维护状态、限制副作用并处理失败，而不是罗列 Agent 领域的流行术语。

## 为什么研究 Hermes

Hermes 在一个公开仓库中同时实现了多 Provider 接入、工具调用、SessionDB、上下文压缩、长期记忆、Skills、MCP、插件、多 Agent、自动化任务和多种交互入口。这些模块共用同一套 Agent Runtime，适合用来研究一个 Agent 项目从单次模型调用扩展到长期运行服务后出现的工程问题。

Hermes 不是这些问题的唯一解法。笔记在相同问题上对照 Codex 与 Claude Code 的公开源码或协议，比较状态归属、扩展接口和安全边界。无法从公开材料确认的内部实现不会写成事实。

## 整理方法

我按功能和调用链组织内容。每个主题先确定输入与输出，再定位源码入口，追踪中间状态、持久化位置和失败分支。代码只保留说明控制流所需的片段，完整实现仍以对应的上游源码快照为准。

学习过程中产生的问题不会原样搬进文档。只有能澄清源码边界、解释设计取舍或纠正常见误解的问题，才会整理到相应章节。

## 当前范围

第一版编号从 00 到 20，共 21 篇，主要对应 Hermes 源码快照 [`590a193`](https://github.com/NousResearch/hermes-agent/commit/590a19332e898fc9bda55a31999926572d8fbc26)。覆盖范围包括：

- Agent Loop、Prompt Builder、Provider Transport 和工具执行链。
- SessionDB、session_search、上下文压缩、Memory 与 Skills。
- 安全控制、MCP、插件、浏览器与终端工具。
- 多 Agent、Kanban、自动化任务和各类产品入口。
- Codex、Claude Code 对照，以及相关论文与 Hermes 实现之间的关系。

完整目录见 [README.md](README.md)。

## 这套笔记解决什么问题

这套笔记用于回答具体的实现问题：一次用户输入怎样经过多次模型迭代，tool call 为什么要在副作用发生前持久化，临时 Memory 为什么只进入单次 Provider 请求，finalizer 为什么承担协议闭合和资源清理，以及后台 review 怎样避免污染主 session。

这些问题都可以继续落到函数、数据结构和数据库记录上。脱离源码快照的框架印象不属于本仓库的结论。

## 声明

这是非官方学习项目。Hermes、Codex、Claude Code 及相关名称归各自权利方所有。Hermes 后续版本可能调整函数位置、配置项和运行行为，引用笔记中的实现结论时应同时保留源码快照信息。
