# 第 14 讲：Cron 和自动化任务

这一讲讲 Hermes 的无人值守任务系统。它看起来像一个 `cronjob` 工具，但源码里不是简单的“延迟执行一段 prompt”。Hermes 把自动化拆成了几层：任务记录、时间计算、触发器、执行体、平台投递、安全边界。理解这几层，才能回答一个关键问题：为什么 Hermes 能让 Agent 在没有用户在线追问的情况下，周期性地做事，并且还能尽量不乱跑。

## 这篇先解决什么问题

读完这一讲，应该能讲清楚四件事：

- 模型怎么通过 `cronjob` 工具创建和管理定时任务。
- Hermes 怎么把自然时间、cron 表达式、一次性任务统一成 `next_run_at`。
- 内置 ticker 和 Chronos provider 的区别是什么。
- 一个 cron job 真正运行时，为什么是“新 session + 自动投递”，不是继续当前聊天。

## 功能 1：模型如何创建一个 cron job

### 先讲人话

用户可以说：“每天早上 9 点帮我总结新闻并发到 Telegram。”模型不会自己睡到明天再执行，而是调用 `cronjob(action="create", ...)`，把这件事登记成一条持久任务。之后，调度器按时间触发这条任务。

这个设计的重点是：模型只负责“定义任务”，调度器负责“按时执行”。两者不能混在一起，否则 Agent 一旦结束当前会话，任务也就丢了。

### 源码入口

| 角色 | 文件/符号 | 说明 |
| --- | --- | --- |
| 模型可见工具 | `tools/cronjob_tools.py` / `cronjob` | `create/list/update/pause/resume/remove/run` 的统一入口 |
| 任务持久化 | `cron/jobs.py` / `create_job` | 生成 job id，写入 schedule、prompt、delivery、skills 等字段 |
| 调度执行 | `cron/scheduler.py` / `tick`、`run_one_job` | 找到 due job，执行、保存输出、投递、更新状态 |
| 触发器抽象 | `cron/scheduler_provider.py` / `CronScheduler` | 决定“什么时候 fire”，不决定“怎么执行” |

### 关键源码

先看模型入口。`cronjob` 的 `create` 分支会做参数校验、安全扫描、脚本路径校验、`context_from` 引用校验，然后才写入 job：

```python
if normalized == "create":
    if not schedule:
        return tool_error("schedule is required for create", success=False)

    canonical_skills = _canonical_skills(skill, skills)
    _no_agent = bool(no_agent)

    if _no_agent:
        if not script:
            return tool_error(
                "create with no_agent=True requires a script; "
                "the script is the job.",
                success=False,
            )
    elif not prompt and not canonical_skills:
        return tool_error("create requires either prompt or at least one skill", success=False)

    if prompt:
        scan_error = _scan_cron_prompt(prompt)
        if scan_error:
            return tool_error(scan_error, success=False)
```

这段在 `tools/cronjob_tools.py`。它说明一个细节：cron job 不是“任何字符串都能登记”。普通 LLM 任务必须有 `prompt` 或 `skills`；`no_agent=True` 时必须有 `script`，因为这时脚本就是任务本身。

真正写入 job 的地方在 `cron/jobs.py`：

```python
job = {
    "id": job_id,
    "name": name or label_source[:50].strip(),
    "prompt": prompt_text,
    "skills": normalized_skills,
    "model": normalized_model,
    "provider": normalized_provider,
    "script": normalized_script,
    "no_agent": normalized_no_agent,
    "context_from": context_from,
    "schedule": parsed_schedule,
    "schedule_display": parsed_schedule.get("display", schedule),
    "next_run_at": compute_next_run(parsed_schedule),
    "last_run_at": None,
    "last_status": None,
    "deliver": deliver,
    "origin": origin,
    "enabled_toolsets": normalized_toolsets,
    "workdir": normalized_workdir,
}
```

这里最好把 job 当成一个“未来会话的启动配置”。它不只保存 prompt，也保存运行模型、投递目标、工作目录、上游任务引用、技能列表和工具集限制。后面 scheduler 运行时，就是靠这条记录重新拼出一个可执行环境。

### 工程实现

`cronjob(action="create")` 会经过几个关口：

| 步骤 | 负责什么 | 为什么需要 |
| --- | --- | --- |
| 参数形态校验 | 判断 `schedule`、`prompt/skills/script` 是否满足当前模式 | 防止登记一条未来必失败的任务 |
| prompt injection 扫描 | `_scan_cron_prompt(prompt)` | cron 是无人值守任务，危险 prompt 不能等到运行时再发现 |
| 脚本路径校验 | `_validate_cron_script_path(script)` | 防止模型把任意敏感路径塞进定时任务 |
| `base_url` 校验 | `_validate_cron_base_url(provider, base_url)` | 防止把命名 provider 的凭据导向恶意 endpoint |
| `context_from` 校验 | 创建时确认上游 job 存在 | 防止未来运行时引用一个不存在的上游输出 |
| `create_job` 持久化 | 生成 job 记录和 `next_run_at` | 让任务跨会话、跨进程、跨 gateway 重启继续存在 |

注意这个边界：模型可以请求创建任务，但它不能跳过 `cronjob` 工具里的校验。Hermes 把风险拦在工具 handler 和 job store 之间，而不是寄希望于模型“自觉不要写危险任务”。

## 功能 2：schedule 怎么变成 next_run_at

### 先讲人话

用户输入的时间可能是 `30m`、`every 2h`、`0 9 * * *`，也可能是一个 ISO 时间戳。Hermes 不会把这些原样存在数据库里等到运行时猜，而是在创建/更新时解析成结构化 schedule，再计算 `next_run_at`。

### 关键源码

`cron/jobs.py` 里的 `parse_schedule` 把四类时间统一成 `kind`：

```python
if schedule_lower.startswith("every "):
    duration_str = schedule[6:].strip()
    minutes = parse_duration(duration_str)
    return {
        "kind": "interval",
        "minutes": minutes,
        "display": f"every {minutes}m"
    }

parts = schedule.split()
if len(parts) >= 5 and all(re.match(r'^[\d\*\-,/]+$', p) for p in parts[:5]):
    croniter(schedule)
    return {
        "kind": "cron",
        "expr": schedule,
        "display": schedule
    }
```

时间格式大致分成：

| 输入 | 解析结果 | 运行语义 |
| --- | --- | --- |
| `30m`、`2h`、`1d` | `kind="once"` | 从现在起延迟一次 |
| `every 30m` | `kind="interval"` | 固定间隔重复 |
| `0 9 * * *` | `kind="cron"` | 用 cron 表达式算下一次 |
| `2026-02-03T14:00` | `kind="once"` | 指定时间运行一次 |

### 工程实现

Hermes 这里的技术选型很保守：解析后统一用 `next_run_at` 驱动调度。这样 scheduler 不需要每次重新理解用户输入，只需要问一句：`next_run_at <= now` 吗？

还有一个容易忽略的点：Hermes 会把 naive timestamp 绑定到配置里的 Hermes 时区，而不是直接用服务器本地时区。定时任务最怕“用户以为是早上 9 点，服务器按 UTC 9 点跑”。这类 bug 不一定立刻炸，但会让无人值守任务变得不可信。

## 功能 3：谁负责触发 cron job

### 先讲人话

Hermes 把 cron 拆成两个轴：

- Axis A：任务执行。也就是创建 Agent、跑 prompt、保存输出、投递结果。
- Axis B：任务触发。也就是谁来决定这个 job 到点了。

这是一处很重要的工程分层。默认情况下，Hermes 进程里有一个 60 秒 ticker。托管场景下，可以用 Chronos：外部服务到点后回调 Hermes。无论哪种触发方式，最后都走同一个 `run_one_job`。

### 关键源码

`cron/scheduler_provider.py` 直接把边界写在注释和接口里：

```python
class CronScheduler(ABC):
    @abstractmethod
    def start(self, stop_event, *, adapters=None, loop=None, interval=60) -> None:
        ...

    def on_jobs_changed(self) -> None:
        return None

    def fire_due(self, job_id: str, *, adapters=None, loop=None) -> bool:
        from cron.jobs import claim_job_for_fire, get_job
        from cron.scheduler import run_one_job

        if not claim_job_for_fire(job_id):
            return False
        job = get_job(job_id)
        if job is None:
            return False
        return run_one_job(job, adapters=adapters, loop=loop)
```

这段代码值得记住：provider 的 `fire_due` 只做 claim 和调用共享执行体。它不重新实现 Agent，不重新实现投递。

默认 provider 是 `InProcessCronScheduler`：

```python
class InProcessCronScheduler(CronScheduler):
    def start(self, stop_event, *, adapters=None, loop=None, interval=60):
        from cron.scheduler import tick as cron_tick
        from cron.jobs import record_ticker_heartbeat

        record_ticker_heartbeat()
        while not stop_event.is_set():
            ok = False
            try:
                cron_tick(verbose=False, adapters=adapters, loop=loop, sync=False)
                ok = True
            except BaseException as e:
                logger.error("Cron tick error: %s", e, exc_info=True)
            record_ticker_heartbeat(success=ok)
            stop_event.wait(interval)
```

它每 60 秒 tick 一次。出错时不会让 ticker 线程静默死亡，而是记录 heartbeat 和错误。这对长期运行的自动化系统很实际：一次异常不应该让所有后续定时任务都失效。

### Chronos：托管场景的 scale-to-zero

Chronos 是另一个 `CronScheduler` provider。它的设计目标不是“更准的 cron 表达式”，而是让 hosted gateway 闲置时可以 scale to zero。

`plugins/cron_providers/chronos/__init__.py` 的核心逻辑是：

```python
def start(self, stop_event, *, adapters=None, loop=None, interval=60):
    try:
        self.reconcile()
    except Exception as e:
        logger.warning("Chronos start() reconcile failed: %s", e)
    # Intentionally return: no loop, no periodic wake.
```

Chronos 启动时不进入 60 秒循环，而是把下一次 fire 注册给外部服务。到点后，外部服务回调 Hermes 的 `/api/cron/fire`，Hermes 验证 token，再调用 provider 的 `fire_due`。

这里的取舍很清楚：

| 方案 | 优点 | 代价 |
| --- | --- | --- |
| 内置 ticker | 简单，完全本地，默认可用 | 进程必须常驻 |
| Chronos | 空闲时可以停机，到点再唤醒 | 需要外部托管服务、回调 URL、JWT 验证 |

### at-most-once：为什么要 claim

在多 gateway 或外部回调重试场景里，同一个 job 可能被多个进程同时看到。Hermes 用 `claim_job_for_fire` 做存储级抢占：

```python
if existing:
    claimed_at = _ensure_aware(datetime.fromisoformat(existing["at"]))
    if (now - claimed_at).total_seconds() < claim_ttl_seconds:
        return False

job["fire_claim"] = {"at": now.isoformat(), "by": _machine_id()}
if kind in {"cron", "interval"}:
    job["next_run_at"] = compute_next_run(job["schedule"], now.isoformat())
save_jobs(jobs)
return True
```

这不是为了追求“绝对只执行一次”。现实里网络和进程崩溃做不到绝对保证。这里实现的是更实用的 at-most-once fire：抢到 claim 的进程执行，没抢到的跳过；如果抢到的机器崩了，TTL 过后允许恢复。

## 功能 4：一个 cron job 真正运行时发生什么

### 先讲人话

cron job 到点后，Hermes 不是把它塞回原聊天继续跑。它会创建一个新的 cron session。这个 session 没有当前聊天历史，但可以加载 job 自己保存的 prompt、skills、script output、`context_from` 输出和 workdir 下的项目上下文。

这是无人值守任务和交互式对话最大的区别。

### 关键源码：构建有效 prompt

`cron/scheduler.py` 的 `_build_job_prompt` 会把脚本输出、上游任务输出、cron 执行提示和 skills 拼在一起：

```python
if script_path:
    success, script_output = _run_job_script(script_path)
    if success and script_output:
        prompt = (
            "## Script Output\n"
            "The following data was collected by a pre-run script. "
            "Use it as context for your analysis.\n\n"
            f"```\n{script_output}\n```\n\n"
            f"{prompt}"
        )
        has_injected_data = True

context_from = job.get("context_from")
if context_from:
    latest_output = output_files[0].read_text(encoding="utf-8").strip()
    if len(latest_output) > _MAX_CONTEXT_CHARS:
        latest_output = latest_output[:_MAX_CONTEXT_CHARS] + "\n\n[... output truncated ...]"
```

`context_from` 的语义是“把上游 job 最近一次输出注入当前 job”。它不是实时等待上游完成，也不是 DAG 调度器。源码里明确读取的是 output 目录里最新的 `.md` 文件，并截断到 8000 字符。也就是说，Hermes 支持简单流水线，但不是 Airflow 那种严格依赖编排系统。

### 关键源码：运行 no_agent job

`no_agent=True` 时，Hermes 不创建 Agent，不花模型 token：

```python
if job.get("no_agent"):
    script_path = job.get("script")
    if not script_path:
        err = "no_agent=True but no script is set for this job"
        return False, "", "", err

    ok, output = _run_job_script(script_path)
    if not ok:
        return False, doc, alert, output
    if not _parse_wake_gate(output):
        return True, silent_doc, SILENT_MARKER, None
    if not output.strip():
        return True, silent_doc, SILENT_MARKER, None
    return True, doc, output, None
```

这个模式很像传统 watchdog：脚本检查系统状态，有输出就发给用户，没输出就安静结束。它适合“服务器异常就通知我”这类任务，不适合需要推理和工具调用的任务。

### 关键源码：运行 LLM job

普通 cron job 会创建 `AIAgent`。但创建前后有很多环境处理：

```python
os.environ["HERMES_CRON_SESSION"] = "1"

_ctx_tokens = set_session_vars(
    platform="",
    chat_id="",
    chat_name="",
)

if _job_workdir:
    os.environ["TERMINAL_CWD"] = _job_workdir
```

这里有几个工程边界：

| 边界 | 作用 | 为什么需要 |
| --- | --- | --- |
| `HERMES_CRON_SESSION=1` | 告诉审批和工具系统这是 cron 场景 | cron 没有在线用户，不能弹交互式确认框 |
| 清空普通 session vars | 避免工具误以为 origin chat 正在驱动本轮执行 | origin 是投递元数据，不是当前用户输入 |
| `HERMES_CRON_AUTO_DELIVER_*` | 给自动投递链路保存目标 | final response 由系统投递，不让模型自己调用 `send_message` |
| `TERMINAL_CWD` | workdir job 的项目上下文和工具 cwd | 让定时任务能在指定 repo 里跑 |
| workdir 锁 | 防止多个 job 争用进程级 cwd/env | `TERMINAL_CWD` 是进程级状态，不能并发乱改 |

这个地方能看出 Hermes 的产品目标：它不是只做“本地 CLI 对话”，而是把 Agent 当长期在线服务运行。所以它要处理 env、session、gateway、tool cwd、投递状态之间的互相污染。

## 功能 5：结果怎么自动投递

### 先讲人话

cron job 的 final response 不应该由模型自己调用 `send_message` 发出去。Hermes 会在 prompt 里告诉模型：你只管输出最终内容，系统会自动投递。这样可以把“生成内容”和“发送到哪里”分离。

### 关键源码

`_build_job_prompt` 会加一个 cron hint：

```python
cron_hint = (
    "[IMPORTANT: You are running as a scheduled cron job. "
    "DELIVERY: Your final response will be automatically delivered "
    "to the user; do NOT use send_message or try to deliver "
    "the output yourself. Just produce your report/output as your "
    "final response and the system handles the rest. "
    "SILENT: If there is genuinely nothing new to report, respond "
    "with exactly \"[SILENT]\" ..."
)
```

最终投递发生在 `run_one_job`：

```python
deliver_content = final_response if success else _summarize_cron_failure_for_delivery(job, error)
should_deliver = bool(deliver_content.strip())

if should_deliver and success and _is_cron_silence_response(deliver_content):
    should_deliver = False

if should_deliver:
    delivery_error = _deliver_result(job, deliver_content, adapters=adapters, loop=loop)
```

Hermes 区分三种结果：

| 结果 | 处理方式 |
| --- | --- |
| 正常 final response | 保存输出并投递 |
| `[SILENT]` 或空输出 | 保存运行记录，但不发消息 |
| job 执行失败 | 生成失败摘要并尝试投递，让用户知道自动化坏了 |

### 平台投递的两条路径

`_deliver_result` 会先解析投递目标，再尝试用 live adapter 发送。live adapter 不可用时，再走 standalone send path。这个设计是为了覆盖 Matrix 这类 E2EE 平台：有些平台必须通过活着的 gateway adapter 才能正确加密或路由。

投递目标也不是只有一个 `chat_id`。Hermes 还要处理：

- `origin`：发回创建任务的那条会话。
- `local`：只保存输出，不发送。
- `all`：发到所有配置好的 home channel。
- `platform:chat_id:thread_id`：发到明确平台、聊天和话题。
- `attach_to_session`：把 cron 输出写回目标会话 transcript，让用户后续回复时能接着聊。

这也是为什么 cron 不能只做成一个“后台 print”。对多平台 Agent 来说，投递本身就是一层复杂系统。

## 功能 6：无人值守任务的安全边界

cron 的风险比普通对话更高，因为它可能在凌晨自动运行，没人盯着确认每一步。Hermes 做了几类保护。

### 1. 创建和更新时扫描 prompt

`tools/cronjob_tools.py` 在 create/update 时调用 `_scan_cron_prompt`。这和 memory 那讲类似，不是调用模型判断，而是规则扫描。cron prompt 如果包含明显的越权指令、数据外泄意图、隐藏 Unicode 等，就直接拒绝。

运行时还会扫描 assembled prompt，因为 skills 和脚本输出是在运行时加入的。源码里把扫描分成 strict 和 loose 两档：裸 prompt 用 strict；包含 skill 或脚本数据时用 loose，避免安全文档、日志、bug 报告里出现命令字符串就误杀任务。

### 2. 脚本只能来自 scripts 目录

`_run_job_script` 会把脚本路径解析到 `scripts` 目录下，并用 `relative_to` 防路径穿越：

```python
scripts_dir = _get_hermes_home() / "scripts"
scripts_dir_resolved = scripts_dir.resolve()

raw = Path(script_path).expanduser()
path = raw.resolve() if raw.is_absolute() else (scripts_dir / raw).resolve()

try:
    path.relative_to(scripts_dir_resolved)
except ValueError:
    return False, "Blocked: script path resolves outside the scripts directory"
```

这挡住的是“模型把 `/etc/passwd`、`../../danger.py`、符号链接逃逸路径塞进 cron 脚本”的风险。

### 3. 子进程环境会清理凭据

cron script 运行时会调用 `_sanitize_subprocess_env(os.environ.copy())`。原因很直接：gateway 进程里可能有 provider key、OAuth token、平台密钥。用户脚本不应该默认继承所有这些秘密。

### 4. Chronos fire 需要 JWT 验证

Chronos 的 `/api/cron/fire` 是远程触发入口。如果这个入口没有鉴权，攻击者就能让 agent 运行任意已有 job。`plugins/cron_providers/chronos/verify.py` 用 PyJWT 校验签名、audience、issuer、过期时间和 `purpose="cron_fire"`：

```python
claims = jwt.decode(token, signing_key, **decode_kwargs)

if claims.get("purpose") != _FIRE_PURPOSE:
    return None
```

这里不手写 JWT 验证，也不允许没有 key 时“先解码看看”。认证失败就返回 `None`，handler 回 401。

### 5. 自动化不应该递归制造自动化

Hermes 在 cron 执行环境里禁用 `cronjob` 工具集，防止一个定时任务不断创建更多定时任务。这个点很容易被忽略：普通 Agent loop 出错最多卡住当前 session；cron 如果能递归创建任务，会把错误持久化成系统级负担。

## 功能 7：并发、锁和失败路径

### tick 锁

`tick` 开始时会拿 `~/.hermes/cron/.tick.lock`。这防止多个进程同时跑同一轮 due job。拿不到锁就跳过本轮。

### job store 锁

`cron/jobs.py` 对 `jobs.json` 做文件锁和原子写。创建、更新、claim、mark run 都要经过这层。它解决的是另一个问题：不是“一个 tick 内别重复跑”，而是“多个进程别把 jobs.json 写坏”。

### 并发执行

`tick` 会把 due jobs 拆成两类：

| 类型 | 执行策略 | 原因 |
| --- | --- | --- |
| 有 `workdir` 的 job | 单线程队列顺序执行 | 它会改 `TERMINAL_CWD` 这类进程级状态 |
| 无 `workdir` 的 job | 线程池并发执行 | 不需要独占项目 cwd |

同时，Hermes 维护 `_running_job_ids`，防止上一次 tick 还在跑的 job 被下一次 tick 再提交一次。

### 失败记录

`run_one_job` 会把失败分成 execution error 和 delivery error。一个 job 可能“成功生成了报告，但 Telegram 暂时发不出去”。这两类状态分开记录，后续排查时才知道是 Agent 坏了，还是平台投递坏了。

## 和 Codex / Claude Code 的差异

### Codex

Codex 更偏“用户发起的开发任务循环”。官方 Codex CLI 文档强调一个聚焦的 terminal loop，用权限和 sandbox 控制文件编辑、命令执行、自动化和 delegation。它适合一次任务里自动读写代码、跑测试、提交结果。

Hermes 的 cron 是长期在线服务能力：任务被持久化，gateway 或 Chronos 到点触发，执行结果再路由回平台会话。它处理的是“用户不在场时，Agent 怎么继续工作并通知用户”。

可以这样记：

- Codex 的自动化核心更像“把一次开发任务跑到底”。
- Hermes 的 cron 更像“把未来的 Agent 会话登记进系统”。

### Claude Code

Claude Code 的 hooks 和 permissions 很强。官方文档里，hooks 可以在工具生命周期点执行，permissions 决定 allow / ask / deny。它们适合给本地开发 Agent 加规则：比如 Bash 前检查、写文件前拦截、命令后记录日志。

Hermes cron 解决的是另一个问题：任务调度、无人值守运行、跨平台投递、可继续会话、外部 provider 触发。Claude Code 可以用 shell cron 或 CI 包起来跑，但“cron job 作为 Agent 内建对象”不是它的核心抽象。

所以同一个问题“每天自动检查仓库并通知我”：

| 框架 | 常见处理方式 |
| --- | --- |
| Codex | 更适合用户发起一次 coding task，或外部 CI/脚本定时拉起 |
| Claude Code | 可用外部 cron/CI 拉起 CLI，再用 hooks/permissions 控制工具行为 |
| Hermes | 内建 `cronjob` 工具登记任务，由 gateway/Chronos 调度并投递到平台 |

## 常见误解

### cron job 会继续当前聊天上下文吗？

不会。Hermes 会创建新的 cron session。它会使用 job 自己保存的 prompt、skills、script output、`context_from`、workdir 项目上下文，但不会继承当前聊天的完整历史。这样做是为了让无人值守任务可重复、可恢复，不依赖某次聊天刚好还在上下文里。

### `context_from` 是不是严格 DAG？

不是。它只是读取上游 job 最近一次保存的输出，并注入当前 prompt。它不会等待上游 job 本轮跑完，也不会做复杂依赖调度。想象成“拿最近一次日报作为今天分析的输入”，不要想象成 Airflow 的 DAG executor。

### `no_agent=True` 是不是更安全？

它跳过 LLM，所以没有模型幻觉和 tool loop 成本。但它仍然执行脚本。安全性取决于脚本来源、脚本目录限制、环境变量清理和平台投递控制。它更适合确定性检查，不适合需要推理的任务。

### 为什么 Hermes 不让模型自己调用 `send_message` 投递 cron 结果？

因为自动化任务的输出目标应该由 job 配置决定，而不是由模型在运行时决定。模型只生成内容，系统负责投递。这样可以避免重复发送、发错平台、绕过 topic/thread 路由，也能统一处理 `[SILENT]`、失败摘要和 delivery error。

## 本篇要记住的东西

Hermes cron 的核心不是“定时执行 prompt”，而是“把未来的一次 Agent 运行持久化”。`tools/cronjob_tools.py` 负责模型入口和创建校验，`cron/jobs.py` 负责 job store 和时间计算，`cron/scheduler.py` 负责执行、保存、投递，`cron/scheduler_provider.py` 负责触发器抽象。内置 ticker 和 Chronos 的差异只在触发层，真正执行都走 `run_one_job`。

记住这句话就够用了：cron job 是一个可恢复的未来 session，不是当前 session 的延迟消息。

## 参考资料

- Hermes 源码：`tools/cronjob_tools.py`
- Hermes 源码：`cron/jobs.py`
- Hermes 源码：`cron/scheduler.py`
- Hermes 源码：`cron/scheduler_provider.py`
- Hermes 源码：`plugins/cron_providers/chronos/__init__.py`
- Hermes 源码：`plugins/cron_providers/chronos/verify.py`
- Hermes 文档：[Cron Internals](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/developer-guide/cron-internals.md)
- Hermes 文档：[Scheduled Tasks (Cron)](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/cron.md)
- OpenAI 文档：[Codex CLI features](https://developers.openai.com/codex/cli/features)
- Anthropic 文档：[Claude Code permissions](https://code.claude.com/docs/en/permissions)
- Anthropic 文档：[Claude Code hooks](https://code.claude.com/docs/en/hooks)
