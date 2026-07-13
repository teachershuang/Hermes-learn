# 关于 Hermes-learn

`Hermes-learn` 是一套已经完成第一版的中文 Hermes Agent 源码课程。它关注的不是“Agent 有哪些热门概念”，而是这些概念进入真实工程后由哪些模块承担、状态保存在哪里、失败时怎样恢复。

## 为什么选择 Hermes

Hermes 把多 Provider、工具调用、SessionDB、上下文压缩、长期记忆、Skills、MCP、插件、多 Agent、自动化任务和多平台入口放在同一个公开仓库里。沿着它的运行链阅读，可以看到一个 Agent 从模型调用样例走向长期运行系统后，多出了哪些协议、状态和保护逻辑。

这套课程不把 Hermes 当作唯一答案。每个模块都尽量说明它解决了什么问题，再对比 Codex 或 Claude Code 的公开实现与协议。比较只围绕同一个工程问题展开，不用产品定位代替源码证据。

## 写作方法

笔记按功能组织。一个功能先用普通语言解释，再给出源码入口和短代码片段，随后追踪输入、状态变化、输出与失败路径。源码摘录保持克制，读者需要回到上游仓库继续阅读完整实现。

课程中的学习问题不会逐字保存。只有能澄清源码边界、解释设计取舍或纠正常见误解的问题，才会整理进对应笔记。

## 当前范围

第一版课程编号从 00 到 20，共 21 篇，主要对应 Hermes 源码快照 [`590a193`](https://github.com/NousResearch/hermes-agent/commit/590a19332e898fc9bda55a31999926572d8fbc26)。内容覆盖：

- Agent Loop、Prompt Builder、Provider Transport 和工具执行链。
- SessionDB、session_search、上下文压缩、Memory 与 Skills。
- 安全控制、MCP、插件、浏览器与终端工具。
- 多 Agent、Kanban、自动化任务和多种产品入口。
- Codex、Claude Code 对比，以及相关论文到工程实现的映射。

完整目录和建议阅读顺序见 [README.md](README.md)。

## 读完后应该具备的能力

读者应该能沿一次对话解释 Hermes 的主要控制流，区分用户 turn、模型迭代、API retry 和 tool call，并能指出 Prompt、工具结果、短期会话与长期记忆分别由谁管理。

遇到工程追问时，答案应能回到具体函数和状态。例如：为什么 tool call 要在副作用发生前持久化，为什么临时 Memory 只进入本次 Provider 请求，为什么 finalizer 不能只写成一个 `finally`，以及后台 review 为什么不能污染主 session。

## 声明

这是非官方学习项目。Hermes、Codex、Claude Code 及相关名称归各自权利方所有。笔记对当前源码行为的描述不保证适用于后续版本，引用时应保留源码快照信息。
