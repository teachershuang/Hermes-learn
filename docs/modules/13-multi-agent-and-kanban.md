# 多 Agent 与 Kanban：短委派、持久队列和工作区隔离

## 先把三个概念分开

Hermes 的多 Agent 不只有一种形态。至少要区分三件事：

```text
delegate_task
  父 agent 同步等待一个短任务的结果

Kanban
  SQLite 中持久化的任务队列与状态机，worker 可以在父会话消失后继续工作

git worktree
  给并行编码任务隔离文件系统工作区和分支的 Git 机制
```

把它们混在一起，会出现两种常见误用：把需要人工介入、跨天重试的工作塞进一次 `delegate_task`；或让多个 agent 在同一份 checkout 里同时改文件，再指望模型自己避开冲突。

这节课的结论先说在前面：`delegate_task` 像函数调用，Kanban 像工作队列，worktree 像并行写代码时的文件隔离。三者可以组合，但不互相替代。

## 功能 1：`delegate_task` 是同步的子 Agent 调用

源码入口是 `tools/delegate_tool.py`。文件顶部给出了非常准确的设计说明：它会创建拥有独立上下文、受限 toolset 和独立 terminal session 的子 `AIAgent`；父 agent 在子 agent 完成前阻塞等待。

```python
# tools/delegate_tool.py
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",  # 防止默认情况下递归委派
    "clarify",        # 子 agent 不能直接向用户追问
    "memory",         # 不能写入共享长期 memory
    "send_message",   # 不能产生跨平台副作用
    "execute_code",   # 强制子 agent 逐步推理和调用工具
])
```

这段列表并非“功能缺失”。它是在定义一个短生命周期 worker 的职责边界：接收一个目标，完成可验证的小任务，把结论交回父 agent。它不应该再开新一层团队、卡在等用户回答、污染父 agent 的长期记忆，或把执行轨迹藏进大段脚本里。

### 子 agent 看见什么，父 agent 看见什么

委派时，子 agent 不是把父 agent 的完整 messages 列表复制一份。它拿到的是：

```text
委派目标 goal
补充 context
聚焦后的 system prompt
独立 task_id
显式允许的 toolsets
```

父上下文只保留一次 delegation 调用与最终 summary，不保留子 agent 的中间 tool call 和思维轨迹。这在两个方面有价值。

第一，父 agent 的 context 不会被子任务里的检索、报错和多轮工具输出撑爆。第二，子 agent 不会被父对话中已经过期的试探、偏题和情绪性文字带偏。

代价也很明确：子 agent 不天然知道父 agent 已经做过哪些调查。`context` 必须写清楚已有结论、文件位置、约束和交付格式。委派提示如果只写“研究一下这个问题”，通常只是在把模糊任务平移给另一个模型。

### 并发与深度为何都要限制

`delegate_task` 可以接收批量任务，并用线程池并发执行。但它受 `delegation.max_concurrent_children` 限制，默认值是 3。超过上限时，工具返回明确错误，不会悄悄截掉任务。

还存在 `max_spawn_depth`，范围被限制为 1 到 3。默认深度为 1：父 agent 只能创建 leaf child，child 不能继续委派。开启 `role="orchestrator"` 并提高深度后，child 才能再创建叶子 worker。

这两个限制是在控制乘法效应：

```text
深度 3，单层并发 3
最多可能形成 3 × 3 × 3 = 27 个叶子 worker
```

真正稀缺的不只是模型请求额度。并发 agent 还会争抢终端环境、网络配额、文件路径和人的注意力。源码把“是否允许继续分叉”做成运行时规则，比靠 system prompt 里一句“不要过度委派”可靠得多。

### 子 agent 的 provider 能与父 agent 不同

委派默认继承父 agent 的 provider 和 model，也允许在 `delegation` 配置里覆盖 provider、model、base URL 和 wire protocol。配置优先级是：自定义 `base_url`，然后是 delegation provider，最后才继承父 provider。

这支持一种实用分工：主 agent 用较强模型做任务拆解、整合与代码判断；叶子 agent 用较快、较便宜的模型完成边界明确的搜索、文件盘点或测试归因。但不要把“便宜模型”理解为免费并行。子 agent 的失败、重复和低质量总结仍会在父 agent 那里产生返工成本。

## 功能 2：Kanban 解决的是“委派返回以后怎么办”

Kanban 的设计文档和实现围绕 `hermes_cli.kanban_db` 展开。它把每张 board 放进 SQLite：任务、依赖、评论、事件和每次运行记录都写入数据库。worker 是独立 OS process，有自己的 profile 和生命周期。

```text
delegate_task:
parent -> child -> 等待 -> summary 回到 parent context

Kanban:
create task -> SQLite row -> dispatcher claim -> worker process
  -> comment / heartbeat / complete / block
  -> SQLite event + run history -> 后续 worker、人或 dashboard 继续处理
```

这不是“把聊天记录做成看板”。它是一台小型任务状态机。

### task、run、comment 和 link 各自负责什么

一个 task 是逻辑工作单元，状态通常经过：

```text
triage -> todo -> ready -> running -> done
                         \-> blocked -> ready
```

task 可以有 assignee、priority、tenant、workspace、最大运行时间和重试预算。

`task_runs` 记录每一次尝试。这个拆分很必要：同一个“修复限流器”任务可能第一次因缺少业务决定而 blocked，第二次因测试失败而结束，第三次才完成。把所有结果覆写在 task 一行里，事后就无法回答“这次成功以前到底发生过什么”。

comment 是 agent 与人之间的持久交接协议。新 worker 启动时会读任务正文、父任务结果、既往尝试和评论线程。link 则是 parent -> child 的依赖边；所有父任务完成后，dispatcher 才把 child 从 `todo` 推进到 `ready`。

这使得下面这种图可以不用某个“总控 agent”一直在线：

```text
research-a ─┐
            ├-> reviewer -> writer
research-b ─┘
```

worker 可以消失，gateway 可以重启，依赖关系仍留在数据库里。

## 功能 3：dispatcher 不是简单轮询器

Kanban dispatcher 通常嵌入 gateway，并按配置周期扫描 board。它要做的工作比“看到 ready 就启动”多得多：

```text
回收过期 claim
检查 worker PID 是否已经消失
推进所有依赖已完成的 task
原子 claim 一个 ready task
准备 workspace
按 assignee profile 启动 worker
在反复启动失败后自动 block，避免无意义重试
```

任务被 claim 后，会生成对应的 run row。worker 在长任务中调用 `kanban_heartbeat()` 更新存活信号。若任务运行过久、又长期没有 heartbeat，dispatcher 会把它视作 stale，终止本机 worker 并重新放回 `ready`。这不是把“长任务”判错，而是在处理进程崩溃、机器休眠、模型卡死或 worker 忘记收尾时的恢复问题。

失败也分类型。claim 超时、PID 消失、超过最大运行时间、启动失败、worker 主动 block 的后续策略不同。特别是连续 spawn failure 会触发 `failure_limit`，自动 block task 并留下原因。否则一个不存在的 profile、无法挂载的 workspace 或持续 429 会每个调度周期都重新拉起进程，变成无声的资源风暴。

### worker 为什么不用 shell 调 `hermes kanban complete`

Kanban worker 拿到的不是 CLI，而是 `kanban_*` 工具集：

```text
kanban_show        读取当前任务和 worker_context
kanban_heartbeat   汇报存活
kanban_comment     追加持久说明
kanban_complete    写入总结和结构化 metadata
kanban_block       请求人或其他角色介入
kanban_create/link 编排型 profile 创建子任务和依赖
```

原因有三个，且都与工程可用性有关。

远程 terminal backend 中不一定安装 Hermes CLI，也未必挂载 Kanban 数据库。结构化工具参数避免 shell quoting 把 JSON metadata 弄坏。最后，工具直接返回 JSON，模型不必从 stderr 猜测操作是否成功。

正常对话不会默认携带这些工具 schema。只有 dispatcher 启动的 worker，或显式启用 Kanban toolset 的 orchestrator profile，才会得到它们。这避免每个普通聊天会话都背上不相关的工具描述。

### 一个可靠 handoff 应留下什么

`kanban_complete()` 不应只写“已完成”。Hermes 的建议 metadata 形式很值得借鉴：

```json
{
  "changed_files": ["src/limiter.py"],
  "verification": ["pytest tests/test_limiter.py -q"],
  "dependencies": ["parent task id"],
  "retry_notes": "上次失败原因和修复方式",
  "residual_risk": ["未覆盖的边界"]
}
```

它的目的不是把所有信息结构化，而是让下一个人或 agent 快速判断：改了什么、如何验证、失败后如何继续、还遗留什么风险。凭一段漂亮的自然语言总结，后续 worker 很难可靠接手。

## 功能 4：worktree 处理的是并行写入冲突

Kanban task 的 workspace 可以是临时 scratch、受信任的共享目录或 Git worktree。涉及代码变更时，`worktree` 通常是更安全的选择。

```text
同一 checkout：
agent A 改 src/a.py，agent B 运行 formatter 或 reset，状态互相干扰

独立 worktree：
agent A -> branch feature/a + directory A
agent B -> branch feature/b + directory B
```

每个 worktree 有独立工作目录和 branch，也有按 worktree 路径隔离的 checkpoint history。一个 agent 的 `/rollback` 不应回滚另一个 agent 的修改。

Hermes 可以通过 `hermes -w` 自动创建临时 worktree，也可以让 Kanban task 用 `worktree` workspace。完成任务时 scratch workspace 会删除，worktree 则保留，等待审阅、测试和合并。这个区别要提前想清楚：研究临时文件适合 scratch；需要保留 diff 的编码任务应使用 worktree。

worktree 不是并行协作的全部答案。两个 agent 即使在不同分支修改同一模块，最终合并仍可能冲突。它解决的是工作时互相踩文件，不解决设计冲突、重复实现和验证缺失。Kanban 的依赖、review task 和结构化 handoff 才是在处理后面的协作问题。

## 一个当前版本边界：不要把 async delegation 写成既有能力

你可能会看到 `delegate_task_async`、`check_task`、`collect_task`、`steer_task` 这类名称。公开 issue 把它们描述为非阻塞后台委派的提案，用于让父 agent 在 child 运行时继续工作。

但当前主线源码与配置文档明确描述的 `delegate_task` 仍是同步调用。课程以当前实现为准：

```text
短而需要立即结果：delegate_task
长、可恢复、需要协作或人工介入：Kanban
```

未来若 async delegation 合入，它会填补二者之间“同一 session 的非阻塞后台子任务”这一位置，但不能自动替代 Kanban 的持久审计、依赖图和跨 profile 交接。

## 与 Codex、Claude Code 的对照

很多 coding agent 都能开多个会话或配合 Git worktree 并行工作，但“多个会话”本身不等于多 Agent 系统。缺少持久任务状态、claim、失败回收、依赖关系和 handoff 记录时，协调责任仍落在用户或外部项目管理工具上。

Hermes 的取向是把这层协调收进 runtime：短任务用 RPC 式委派，长任务用 SQLite board。它的代价是需要维护 dispatcher、profile、workspace 和状态机；它的好处是任务不依赖某段对话还留在 context 内。

对需要隔离改动的编码任务，Codex、Claude Code 和 Hermes 最终都会受同一个 Git 事实约束：独立 branch/worktree 降低同时写同一目录的风险，但最终仍需要测试、review 和合并决策。Hermes 把这些步骤表达成可依赖、可重试的 task，适合更长的工程流水线。

## 常见误解

### `delegate_task` 是否等于“多 Agent 并行开发”？

不等于。它能并行执行独立子任务，但父 agent 会等待结果，子 agent 也没有持久身份和协作记录。它更接近一次同步 RPC 调用。

### Kanban task 完成后，为什么还要保留 run history？

task 描述“要完成什么”，run 描述“某一次尝试发生了什么”。重试、超时、被 reclaim、人工 block 和多轮 review 都是 run 级事实。没有 run history，系统只能知道最后状态，无法做失败归因。

### 为什么 worker 要 heartbeat？

dispatcher 不能只看 PID。进程可能还在，但模型请求、网络调用或工具执行已经卡死。heartbeat 是 worker 主动说明“我仍在推进”的证据；长期没有它，任务可以被安全回收并重新派发。

### Kanban 的 SQLite board 能否跨机器共享？

当前设计是单机 board。worker PID 检测、SQLite 文件和 dispatcher 都假设在同一主机。跨机器要使用独立 board 加消息队列或其他外部协调层，不能把同一 SQLite 文件放到共享盘上赌并发正确性。

### worktree 是否能消除 merge conflict？

不能。worktree 隔离的是进行中的文件系统状态和 branch。两个分支修改同一区域，合并时仍会冲突；任务拆分和 review 仍然是必要的。

## 读完这一讲应能回答

当任务是“快速审查三个文件并把结论交给当前对话”，应选 `delegate_task`。当任务是“研究、实现、review、反复修改并保留人工决策”，应创建 Kanban task 和依赖图。多个 coding worker 需要同时修改代码时，应为每个 worker 使用独立 worktree，并把验证结果写入 handoff metadata。

## 参考源码与文档

- [`tools/delegate_tool.py`](https://github.com/NousResearch/hermes-agent/blob/main/tools/delegate_tool.py)
- [`Kanban 功能文档`](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/kanban.md)
- [`delegation 配置`](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/configuration.md)
- [`Git worktree 指南`](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/git-worktrees.md)
