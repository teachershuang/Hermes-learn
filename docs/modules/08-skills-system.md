# Skills System：Hermes 怎么把“做事方法”沉淀下来

## 这一讲先解决什么问题

Memory 记的是稳定事实。Skills 记的是做事方法。

这句话看起来简单，但工程上差别很大。事实适合短、小、稳定，所以可以每轮塞进 system prompt。方法通常更长，包含触发条件、命令、坑点、模板、脚本和参考资料。如果把所有方法正文都塞进 system prompt，模型每轮都会背着一大堆无关流程走路，成本高，干扰也大。

Hermes 的 Skills System 解决的是这个问题：

```text
让模型知道“有哪些专门方法”
  -> 但不把所有方法正文一次性塞给模型
  -> 需要时用 skill_view 加载完整内容
  -> 发现方法过时后用 skill_manage 修改
  -> 后台 review 把新经验沉淀成 skill
  -> curator 和 usage 负责长期维护
```

这一讲重点讲运行时主链。Skills Hub 的目录来源、网页索引和安装生态会在 CLI / 插件 / 生态章节里再展开。

## 功能 1：Skill 不是普通 Markdown，而是可被 agent 发现的程序化记忆

一个标准 skill 是一个目录，核心文件是 `SKILL.md`：

```text
my-skill/
  SKILL.md
  references/
  templates/
  assets/
  scripts/
```

`SKILL.md` 前面有 YAML frontmatter：

```yaml
---
name: skill-name
description: Brief description
platforms: [linux, macos]
metadata:
  hermes:
    tags: [debugging, workflow]
    related_skills: [systematic-debugging]
---

# Skill Title

Full instructions here...
```

这个格式让 Hermes 能做两件事。第一，只读 frontmatter 就能构建技能索引，不必把正文全塞进 prompt。第二，正文旁边可以带 `references/`、`templates/`、`scripts/`，让 skill 不只是“提示词”，而是一个小型任务包。

源码入口在 `agent/skill_utils.py`：

```python
def parse_frontmatter(content: str) -> Tuple[Dict[str, Any], str]:
    frontmatter: Dict[str, Any] = {}
    body = content

    if not content.startswith("---"):
        return frontmatter, body

    end_match = re.search(r"\n---\s*\n", content[3:])
    if not end_match:
        return frontmatter, body

    yaml_content = content[3 : end_match.start() + 3]
    body = content[end_match.end() + 3 :]
```

注意这里有一个很实用的设计：没有 frontmatter 时不报错，直接返回空 metadata 和原正文。Hermes 需要兼容老格式、手写文件和外部导入的 skill。严格到一碰就坏，系统就不好维护。

但“能兼容”不等于“随便写”。`skill_manage(create/edit)` 写入新 skill 时会校验 frontmatter，安装外部 skill 时还会走安全扫描。读路径宽容，写路径严格，这是 Agent 系统里常见的工程取舍。

## 功能 2：system prompt 只放技能索引，不放全部技能正文

Skills 第一次进入模型视野，不是通过 `skill_view`，而是通过 system prompt 里的技能索引。

入口在 `agent/prompt_builder.py`：

```python
def build_skills_system_prompt(
    available_tools: "set[str] | None" = None,
    available_toolsets: "set[str] | None" = None,
    compact_categories: "frozenset[str] | None" = None,
) -> str:
    """Build a compact skill index for the system prompt."""
    skills_dir = get_skills_dir()
    external_dirs = get_all_skills_dirs()[1:]
```

它做的不是“拼一大段技能说明”，而是扫描 skill 文件，解析 metadata，按分类生成一个 `<available_skills>` 索引。模型看到的是类似这样的信息：

```text
## Skills (mandatory)
Before replying, scan the skills below...

<available_skills>
  software-development:
    - systematic-debugging: ...
    - test-driven-development: ...
</available_skills>
```

真正重要的是这段强制行为指导：

```python
"Before replying, scan the skills below. If a skill matches or is even partially relevant "
"to your task, you MUST load it with skill_view(name) and follow its instructions. "
...
"If a skill has issues, fix it with skill_manage(action='patch').\n"
```

这就是 Hermes 的渐进式披露：

```text
system prompt：技能名 + 描述 + 分类
skill_view(name)：完整 SKILL.md
skill_view(name, file_path)：支持文件、模板、脚本
```

为什么不直接把所有 skill 正文塞进去？原因有三个。

第一，token 成本会失控。一个技能库可能有几十到上百个 skill，每个 skill 都可能有几百到几千字。

第二，模型会被无关流程干扰。用户只是问一个简单问题，模型却同时看到部署、渗透测试、图像生成、代码审查、区块链操作的完整步骤，很容易过度行动。

第三，prefix cache 会变差。system prompt 越大、越频繁变化，provider 的缓存命中越差。Hermes 为 skills index 做了两层缓存，说明它非常在意这块成本。

## 功能 3：技能索引为什么要做两层缓存

`build_skills_system_prompt` 的注释把缓存说得很清楚：

```python
"""Build a compact skill index for the system prompt.

Two-layer cache:
  1. In-process LRU dict keyed by (skills_dir, tools, toolsets, hidden)
  2. Disk snapshot (``.skills_prompt_snapshot.json``) validated by
     mtime/size manifest — survives process restarts
"""
```

第一层是进程内 LRU：

```python
cache_key = (
    str(skills_dir.resolve()),
    tuple(str(d) for d in external_dirs),
    tuple(sorted(str(t) for t in (available_tools or set()))),
    tuple(sorted(str(ts) for ts in (available_toolsets or set()))),
    _platform_hint,
    tuple(sorted(disabled)),
    tuple(sorted(compact_categories or ())),
)
```

这个 key 很值得看。它不只看 skills 目录，还看工具、工具集、平台、禁用技能、压缩分类。因为同一个技能库，在不同平台或不同工具集下，能显示的 skill 可能不一样。

第二层是磁盘快照：

```python
snapshot = _load_skills_snapshot(skills_dir)
```

如果快照有效，就直接使用预解析 metadata。无效时才全量扫描 `SKILL.md`，然后写回 `.skills_prompt_snapshot.json`。这不是过早优化。Hermes 的 skill 数量一多，冷启动时反复读 YAML、解析 category、检查平台，成本会很明显。

这里还体现了一个原则：system prompt 不是“每次随手拼字符串”。它是运行时性能、缓存命中、安全过滤和工具可用性的交汇点。Skills index 是 prompt 的一部分，所以它也要像工程模块一样被缓存、校验和失效。

## 功能 4：Skill 是否出现在索引里，不是只看文件存在

Hermes 不会把所有 `SKILL.md` 都无条件展示给模型。过滤主要有四类。

第一类是平台过滤。源码在 `agent/skill_utils.py`：

```python
def skill_matches_platform(frontmatter: Dict[str, Any]) -> bool:
    platforms = frontmatter.get("platforms")
    if not platforms:
        return True
    if not isinstance(platforms, list):
        platforms = [platforms]
    current = sys.platform
    for platform in platforms:
        normalized = str(platform).lower().strip()
        mapped = PLATFORM_MAP.get(normalized, normalized)
        if current.startswith(mapped):
            return True
    return False
```

没有 `platforms` 字段就默认兼容所有平台。写了 `platforms: [linux, macos]`，Windows 上就不会出现在索引里。这样可以避免模型在 Windows 会话里主动加载只适合 Linux 的技能。

第二类是环境过滤：

```python
def skill_matches_environment(frontmatter: Dict[str, Any]) -> bool:
    environments = frontmatter.get("environments")
    if not environments:
        return True
    ...
    if _detect_environment(normalized):
        return True
    return False
```

这里的注释很关键：environment 是 offer-time filter。它只控制“是否主动展示在索引里”，不阻止显式 `skill_view`。比如一个 kanban-only skill 平时不显示，用户明确加载时仍然可以加载。这叫“降低噪音，但尊重显式意图”。

第三类是工具条件。源码在 `agent/prompt_builder.py`：

```python
def _skill_should_show(conditions, available_tools, available_toolsets) -> bool:
    for ts in conditions.get("fallback_for_toolsets", []):
        if ts in ats:
            return False
    for t in conditions.get("fallback_for_tools", []):
        if t in at:
            return False

    for ts in conditions.get("requires_toolsets", []):
        if ts not in ats:
            return False
    for t in conditions.get("requires_tools", []):
        if t not in at:
            return False

    return True
```

这让 skill 可以表达“只有某工具可用时才展示”，也可以表达“主工具可用时隐藏这个 fallback skill”。这比静态技能列表更接近真实产品：不同 profile、不同 gateway、不同工具权限下，模型应该看到不同的能力边界。

第四类是禁用列表。`get_disabled_skill_names` 会读配置里的全局禁用和平台级禁用：

```python
global_disabled = _normalize_string_set(skills_cfg.get("disabled"))
if resolved_platform:
    platform_disabled = (skills_cfg.get("platform_disabled") or {}).get(
        resolved_platform
    )
    if platform_disabled is not None:
        return global_disabled | _normalize_string_set(platform_disabled)
return global_disabled
```

所以“技能存在”和“模型能看见”是两回事。能不能出现在 prompt 里，要同时过目录扫描、frontmatter 解析、平台、环境、工具条件和禁用配置。

## 功能 5：`skills_list` 和 `skill_view` 分别解决什么问题

Skills 暴露给模型的主要工具是两个：

```text
skills_list：列出技能 metadata
skill_view：读取完整技能内容或支持文件
```

`skills_list` 在 `tools/skills_tool.py`：

```python
def skills_list(category: str = None, task_id: str = None) -> str:
    all_skills = _find_all_skills()

    if category:
        all_skills = [s for s in all_skills if s.get("category") == category]

    all_skills = _sort_skills(all_skills)
    categories = sorted(
        {s.get("category") for s in all_skills if s.get("category")}
    )

    return json.dumps({
        "success": True,
        "skills": all_skills,
        "categories": categories,
        "count": len(all_skills),
        "hint": "Use skill_view(name) to see full content, tags, and linked files",
    })
```

它返回的是轻量列表。适合模型不确定有没有合适 skill 时查一下，但多数情况下，system prompt 里的 `<available_skills>` 已经够模型判断。

`skill_view` 才是真正加载技能：

```python
def skill_view(
    name: str,
    file_path: str = None,
    task_id: str = None,
    preprocess: bool = True,
) -> str:
```

它要处理的边界比“读一个文件”多得多：

```text
校验 name，防止路径穿越
  -> 如果是 plugin:skill，走插件技能注册表
  -> 扫描本地 skills 目录和 external_dirs
  -> 检查重名冲突
  -> 解析 frontmatter
  -> 检查平台和禁用状态
  -> 如果 file_path 存在，读取支持文件
  -> 否则返回 SKILL.md + linked_files + tags + setup 状态
```

重名冲突这段很有意思：

```python
if len(candidates) > 1:
    return json.dumps({
        "success": False,
        "error": (
            f"Ambiguous skill name '{name}': {len(candidates)} skills "
            "match across your local skills dir and external_dirs. "
            "Refusing to guess — load one explicitly by its categorized path."
        ),
        "matches": paths,
    })
```

Hermes 没有让本地 skill 悄悄覆盖外部 skill，也没有随便选第一个。它直接失败，让模型或用户用更具体的路径加载。对 Agent 来说，“猜错一个技能”比“多问一步”更危险，因为后续所有步骤都会被错误技能带偏。

## 功能 6：`skill_view` 的 linked files 才是技能包的重点

当 `skill_view(name)` 返回主文件时，它还会列出支持文件：

```python
linked_files = {}
if reference_files:
    linked_files["references"] = reference_files
if template_files:
    linked_files["templates"] = template_files
if asset_files:
    linked_files["assets"] = asset_files
if script_files:
    linked_files["scripts"] = script_files
```

返回里还有提示：

```python
"usage_hint": "To view linked files, call skill_view(name, file_path) where file_path is e.g. 'references/api.md' or 'assets/config.yaml'"
```

这就是渐进式披露的第二层。一个好的 skill 不应该把所有东西塞进 `SKILL.md`。主文件负责告诉模型什么时候用、按什么步骤做、有哪些坑。细节材料放到：

| 目录 | 适合放什么 |
| --- | --- |
| `references/` | API 摘要、错误案例、排障记录、领域知识 |
| `templates/` | 可复制修改的配置、脚手架、报告模板 |
| `scripts/` | 可重复执行的检查脚本、生成脚本、探针 |
| `assets/` | 补充资源，按 skill 自己的需要组织 |

这和普通 prompt library 很不一样。普通 prompt library 多半只是“把一段提示词复制到上下文”。Hermes 的 skill 更像一个轻量包管理单元：有 manifest、有主说明、有支持文件、有脚本、有安装和安全边界。

## 功能 7：技能可以声明密钥和运行条件

`skill_view` 还会处理运行前提。比如一个 skill 在 frontmatter 里声明需要某个环境变量，Hermes 会在加载时检查：

```python
required_env_vars = _get_required_environment_variables(
    frontmatter, legacy_env_vars
)
missing_required_env_vars = [
    e
    for e in required_env_vars
    if not e.get("optional")
    and not _is_env_var_persisted(e["name"], env_snapshot)
]
```

如果缺少变量，结果里会标出：

```python
"missing_required_environment_variables": remaining_missing_required_envs,
"setup_needed": setup_needed,
"readiness_status": SkillReadinessStatus.SETUP_NEEDED.value
```

这很重要。模型加载一个 skill 后，不只是拿到文字说明，还能知道“这个 skill 当前能不能跑”。在 CLI 环境里，Hermes 还可以通过回调收集 secret；在 Gateway 环境里，则给用户配置提示。

这里的设计思路是：skill 的“可用性”不应该只靠模型读说明猜。运行时应该把缺少的 env、credential file、setup hint 结构化返回，模型再据此决定是继续、提示用户配置，还是换方案。

## 功能 8：`skill_manage` 让模型能写入和维护技能

如果说 `skill_view` 是读取技能，`skill_manage` 就是写入技能。源码在 `tools/skill_manager_tool.py`。

它支持这些动作：

```text
create
edit
patch
delete
write_file
remove_file
```

入口是：

```python
def skill_manage(
    action: str,
    name: str,
    content: str = None,
    category: str = None,
    file_path: str = None,
    file_content: str = None,
    old_string: str = None,
    new_string: str = None,
    replace_all: bool = False,
    absorbed_into: str = None,
) -> str:
```

写入前有 approval gate：

```python
gate_result = _apply_skill_write_gate(
    action, name, content=content, category=category,
    file_path=file_path, file_content=file_content,
    old_string=old_string, new_string=new_string,
    replace_all=replace_all, absorbed_into=absorbed_into,
)
if gate_result is not None:
    return gate_result
```

这说明 skill 写入被当成高影响操作。原因很直接：skill 会影响未来很多次任务，不是当前 turn 的临时输出。一个错误 skill 可能让 agent 以后长期重复错误流程。

创建 skill 的路径是：

```python
def _create_skill(name: str, content: str, category: str = None) -> Dict[str, Any]:
    err = _validate_name(name)
    ...
    err = _validate_frontmatter(content)
    ...
    existing = _find_skill(name)
    if existing:
        return {"success": False, "error": f"A skill named '{name}' already exists..."}

    skill_dir = _resolve_skill_dir(name, category)
    skill_dir.mkdir(parents=True, exist_ok=True)

    skill_md = skill_dir / "SKILL.md"
    _atomic_write_text(skill_md, content)

    scan_error = _security_scan_skill(skill_dir)
    if scan_error:
        shutil.rmtree(skill_dir, ignore_errors=True)
        return {"success": False, "error": scan_error}
```

几个细节要记住：

第一，先校验名称和 frontmatter。写入不合格的 `SKILL.md`，后续索引和 `skill_view` 都会变得不可预测。

第二，写文件使用 atomic write，避免半截文件留在技能库。

第三，写完立刻安全扫描。扫描失败就回滚新建目录。

第四，成功后会清理 skills prompt cache：

```python
from agent.prompt_builder import clear_skills_system_prompt_cache
clear_skills_system_prompt_cache(clear_snapshot=True)
```

否则模型下一轮还可能看到旧的技能索引。

## 功能 9：为什么 patch 优先于 edit

`skill_manage` 同时支持 `edit` 和 `patch`。但 schema 里的描述明确建议：小修改用 `patch`，大改才用 `edit`。

`patch` 的实现不是简单 `str.replace`：

```python
from tools.fuzzy_match import fuzzy_find_and_replace

new_content, match_count, _strategy, match_error = fuzzy_find_and_replace(
    content, old_string, new_string, replace_all
)
```

这复用了 Hermes 的 fuzzy matching engine，能容忍一些空白、缩进、转义差异。对模型来说，这比“完整重写一个大文件”可靠得多。

失败时，工具会给模型返回文件预览和修正提示：

```python
return {
    "success": False,
    "error": err_msg,
    "file_preview": preview,
}
```

这是非常典型的 agent 工程设计：不要只告诉模型“失败了”，要给它足够信息让它自己修正下一次 tool call。

`edit` 是全量替换，风险更高。它适合结构性改写，但必须先读原 skill。后台 review 场景里，Hermes 还有 read-before-write guard，防止 review agent 没看文件就乱改。

## 功能 10：后台 Skill Review 什么时候介入

前面问过 Memory / Skill review 的介入时机。Memory 已经在上一讲讲过，这里补 skill 侧。

主循环每次工具迭代都会累加 `_iters_since_skill`。源码在 `agent/conversation_loop.py`：

```python
if (agent._skill_nudge_interval > 0
        and "skill_manage" in agent.valid_tool_names):
    agent._iters_since_skill += 1
```

turn 结束时检查是否达到阈值。源码在 `agent/turn_finalizer.py`：

```python
_should_review_skills = False
if (agent._skill_nudge_interval > 0
        and agent._iters_since_skill >= agent._skill_nudge_interval
        and "skill_manage" in agent.valid_tool_names):
    _should_review_skills = True
    agent._iters_since_skill = 0
```

配置来自初始化：

```python
agent._skill_nudge_interval = int(
    skills_config.get("creation_nudge_interval", 10)
)
```

所以它不是“每轮都 review”，也不是“模型自己想起来就 review”。它的触发条件是：

```text
工具迭代累计达到阈值
  + skill_manage 工具可用
  + 本轮有 final_response
  + 本轮没有被 interrupt
```

触发后，`agent._spawn_background_review(...)` 会在后台线程里 fork 一个 review agent：

```python
if final_response and not interrupted and (_should_review_memory or _should_review_skills):
    agent._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

注意顺序：先给用户返回 final response，再做 review。这样不会拖慢主任务。

## 功能 11：Skill Review 的 prompt 在教模型沉淀“类级别方法”

Skill review 的核心提示词在 `agent/background_review.py`：

```python
_SKILL_REVIEW_PROMPT = (
    "Review the conversation above and update the skill library. Be "
    "ACTIVE — most sessions produce at least one skill update, even if "
    "small. ...\n\n"
    "Target shape of the library: CLASS-LEVEL skills, each with a rich "
    "SKILL.md and a `references/` directory for session-specific detail. "
    "Not a long flat list of narrow one-session-one-skill entries. ..."
)
```

这里有一个很重要的判断：Hermes 不希望每次任务都生成一个小 skill。它希望 skill 是 class-level umbrella。也就是“这一类任务的方法”，而不是“今天这个 PR 的记录”。

提示词给了优先级：

```text
1. 先更新当前会话已加载的 skill
2. 再找已有 umbrella skill 更新
3. 再给已有 umbrella 添加 references/templates/scripts
4. 最后才创建新的 class-level skill
```

这套顺序是为了防止技能库碎片化。否则后台 review 很快会把 skill 目录变成几百个一次性小文件。模型未来看到索引时，反而更难选。

Skill review 还明确说，用户对工作方式的纠正不只是 memory：

```text
Memory 说“用户是谁、当前事实是什么”
Skills 说“这类任务以后应该怎么做”
```

比如用户说“以后写学习笔记不要放本机路径”。这可以成为用户偏好，但如果它属于“写开源学习笔记”的工作方法，也应该进入对应 skill 或项目指导。只写 memory，模型可能知道偏好；写进 skill，模型在执行这类任务时会得到具体方法约束。

## 功能 12：后台 review fork 为什么要严格隔离

后台 review 不是在主 agent 里直接继续对话，而是 fork 一个新的 `AIAgent`。源码在 `agent/background_review.py`：

```python
review_agent = AIAgent(
    model=_rt.get("model") or agent.model,
    max_iterations=16,
    quiet_mode=True,
    platform=agent.platform,
    provider=_rt.get("provider") or agent.provider,
    ...
    skip_memory=True,
)
```

随后有一组关键隔离设置：

```python
review_agent._memory_store = agent._memory_store
review_agent._persist_disabled = True
review_agent._session_db = None
review_agent._session_json_enabled = False
review_agent.suppress_status_output = True
review_agent.compression_enabled = False
```

这些不是多余防御。

`_persist_disabled=True` 防止 review prompt 被写入用户真实 SessionDB。否则用户下一次继续对话时，模型可能把“Review the conversation above and update the skill library”当成真实历史上下文。

`compression_enabled=False` 防止 review fork 和主会话抢压缩，造成 session parent/child 关系混乱。

`skip_memory=True` 防止外部 memory provider 把 review harness 当成真实用户对话同步出去。

工具白名单也很严格：

```python
review_toolsets = ["skills"]
if review_agent._memory_enabled or review_agent._user_profile_enabled:
    review_toolsets.insert(0, "memory")
```

review agent 可以看完整会话快照，但只能调用 memory / skills 相关工具。它不能趁机跑终端、读文件、发网络请求。后台自我改进必须被限制在“沉淀经验”这个范围里。

## 功能 13：Skills Guard 负责安装和写入后的安全扫描

技能能影响 future prompt，所以安全边界比普通文档更高。Hermes 有专门的 `tools/skills_guard.py`。

主入口：

```python
def scan_skill(skill_path: Path, source: str = "community") -> ScanResult:
    """
    Scan all files in a skill directory for security threats.

    Performs:
    1. Structural checks
    2. Regex pattern matching on all text files
    3. Invisible unicode character detection
    """
```

扫描分两层。

第一层是结构检查：

```python
def _check_structure(skill_dir: Path, ignore=None) -> List[Finding]:
    """
    Check the skill directory for structural anomalies:
    - Too many files
    - Suspiciously large total size
    - Binary files
    - Symlinks pointing outside the skill directory
    - Individual files that are too large
    """
```

第二层是内容检查。它会扫描 prompt injection、外泄、危险命令、持久化后门、混淆等模式，还会检查不可见 Unicode。

安装决策由 `should_allow_install` 决定：

```python
policy = INSTALL_POLICY.get(result.trust_level, INSTALL_POLICY["community"])
vi = VERDICT_INDEX.get(result.verdict, 2)
decision = policy[vi]
```

这里有 trust level：builtin、trusted、community、agent-created。不同来源命中同样问题时，处理策略不一样。比如社区来源的 dangerous verdict 不能用 `--force` 轻易绕过。

Hermes 还支持 `.skillignore` / `.clawhubignore`：

```python
_SKILL_IGNORE_FILENAMES = (".skillignore", ".clawhubignore")
_NEVER_IGNORABLE = {"SKILL.md"}
```

这解决的是误报问题。一个 skill 包里可能带开发笔记、测试材料、引用文档，其中有些文字会触发规则。允许 ignore 可以减少误报，但 `SKILL.md` 永远不能被忽略。主指令文件必须被扫描。

这和上一讲 memory 的 threat scanner 思路相同：高风险内容进入 prompt 前，先过确定性 guardrail。区别是 skill 是一个目录包，所以还要查结构、文件数量、二进制、符号链接和来源信任。

## 功能 14：usage telemetry 和 curator 是技能库的长期维护层

Skill 创建出来不是结束。长期运行后会出现几个问题：

```text
有些 skill 从没被用过
有些 skill 重叠
有些 skill 过时
有些 skill 被创建得太细
有些 skill 需要合并到 umbrella
```

Hermes 用 `tools/skill_usage.py` 记录使用侧信息：

```python
def bump_view(skill_name: str) -> None:
    """Bump view_count and last_viewed_at. Called from skill_view()."""
    ...

def bump_use(skill_name: str) -> None:
    """Bump use_count and last_used_at."""
    ...

def bump_patch(skill_name: str) -> None:
    """Bump patch_count and last_patched_at. Called from skill_manage."""
```

这里有一个重要修正：usage telemetry 跟 curator eligibility 解耦。源码注释说得很直接：

```python
"""Tracks every skill regardless of provenance — built-ins and hub skills included.
Usage telemetry is observability, not a curation signal.
"""
```

也就是说，所有 skill 都可以统计使用情况，但不是所有 skill 都能被自动整理、删除或归档。hub-installed、bundled、agent-created 的生命周期不同。

自动归档走 `archive_skill`：

```python
def archive_skill(skill_name: str) -> Tuple[bool, str]:
    """Move a curator-eligible skill directory to ~/.hermes/skills/.archive/."""

    if not is_curation_eligible(skill_name, local_skill_dir):
        if is_protected_builtin(skill_name):
            return False, "..."
        if is_hub_installed(skill_name):
            return False, f"skill '{skill_name}' is hub-installed; never archive"
```

这说明 curator 不是“LLM 想删就删”。它有 provenance、pinned、hub-installed、bundled built-in 等边界。后台维护可以自动化，但不能没有所有权边界。

## 功能 15：Pinned skill 保护的是删除，不是修改

`skill_manager_tool.py` 里有 `_pinned_guard`：

```python
def _pinned_guard(name: str) -> Optional[str]:
    ...
    return (
        f"Skill '{name}' is pinned and cannot be deleted by "
        f"skill_manage. Ask the user to run "
        f"`hermes curator unpin {name}` first."
    )
```

但 schema 描述里强调：

```python
"Pinned skills are protected from deletion only — skill_manage(action='delete') "
"will refuse ... Patches and edits go through on pinned skills so you can still improve them..."
```

这个边界很细，但很重要。pin 的目的不是把 skill 冻结成只读，而是防止 curator 或模型把重要 skill 删掉、归档掉。一个 pinned skill 仍然可能有错误命令、过时步骤、缺少坑点，所以 patch/edit 应该允许。

如果把 pinned 理解为“完全不能改”，用户就会陷入一个坏状态：最重要的 skill 反而不能修。Hermes 把删除保护和内容改进分开，这是比较成熟的生命周期设计。

## 功能 16：插件技能为什么不进系统提示索引

Hermes 支持插件注册命名空间技能，例如 `myops:deploy`。`skill_view` 会检测 `:`：

```python
if ":" in name:
    from agent.skill_utils import is_valid_namespace, parse_qualified_name
    from hermes_cli.plugins import discover_plugins, get_plugin_manager

    namespace, bare = parse_qualified_name(name)
    ...
    plugin_skill_md = pm.find_plugin_skill(name)
```

插件 skill 会通过 `_serve_plugin_skill` 加载：

```python
return json.dumps({
    "success": True,
    "name": f"{namespace}:{bare}",
    "content": f"{banner}{rendered_content}" if banner else rendered_content,
    "description": description,
})
```

但插件技能不进入 `<available_skills>` 主索引。原因很实际：

```text
第三方插件数量可能很多
插件启停会频繁改变 prompt
system prompt 已经很大
用户安装插件不等于模型应该自动感知所有插件技能
```

所以插件技能是显式 opt-in。模型必须知道名字，或者从插件文档、用户指令中获得名字，再调用 `skill_view("plugin:skill")`。

这和本地 skills 的默认索引不同。内置/本地 skill 是 agent 的常规程序化记忆；插件 skill 更像外部能力包，不自动污染主 prompt。

## 功能 17：Memory、Skills、Session Search 的边界再收束一次

上一讲已经讲过这张边界表，这一讲换成 skills 视角再看：

| 系统 | 存什么 | 怎么进入模型 |
| --- | --- | --- |
| Memory | 稳定事实、偏好、环境事实 | 每轮自动进入 system prompt |
| Skills | 可复用流程、任务方法、坑点、模板 | 索引进 system prompt，正文按需 `skill_view` |
| Session Search | 过去对话原文和摘要 | 模型需要时主动检索 |

判断一条信息应该去哪里，可以这样问：

```text
这是短事实吗？
  -> Memory

这是以后做同类任务的方法吗？
  -> Skill

这是过去某次会话的细节吗？
  -> Session Search
```

例子：

```text
用户偏好中文输出。
```

适合 Memory。

```text
写开源学习笔记时，不要放本机路径；每个功能要跟源码锚点和工程实现。
```

如果只是用户偏好，可以进 Memory。如果它是“写这类学习笔记”的稳定流程，更应该进项目指导或 skill。

```text
第七讲补充了 threat_patterns.py 的 strict scope。
```

这是项目进度，适合文档和 git 历史，不适合 Memory 或 Skill。

## 和 Codex、Claude Code 的对比

Claude Code 常见的长期项目行为约束来自 `CLAUDE.md`、命令、MCP、hooks 和 settings。它也有 skills 生态，但在日常使用里，项目说明文件承担了很多“这个项目该怎么做”的角色。Claude Code 更偏本地开发者工作流，技能和项目上下文通常围绕当前 repo 展开。

Codex 更强调 thread、workspace、工具执行和本地项目指导。它也有 skills 机制，但具体加载由宿主运行时和当前任务触发，很多能力还来自 Codex 自己的工具面和插件。对于软件工程任务，Codex 更像“在一个工作区里持续协作的工程代理”。

Hermes 的特点是把 Skills 做成了长期在线 agent 的程序化记忆层：

| 问题 | Hermes | Codex / Claude Code 常见形态 |
| --- | --- | --- |
| 模型怎么知道有哪些方法 | system prompt 中的 `<available_skills>` 索引 | 项目文件、技能列表、宿主工具或显式加载 |
| 方法正文怎么加载 | `skill_view` 渐进式读取 | 通常按技能机制、项目文件或命令加载 |
| 方法怎么被更新 | `skill_manage`，后台 review 可主动 patch/create | 多由用户编辑、命令、插件或 agent 按任务修改 |
| 长期维护 | usage telemetry + curator + archive/restore | 取决于宿主产品和项目工作流 |
| 安全扫描 | `skills_guard.py` 扫内容和结构 | 通常依赖平台安全边界、安装信任和文件权限 |

Hermes 更激进的地方是“自我沉淀”：它不只使用 skill，还会在任务结束后判断要不要更新 skill。这个能力很强，也很危险，所以它配套了后台 fork 隔离、工具白名单、read-before-write、写入审批、安全扫描、pinned 保护和 curator 边界。

## 常见问题

### 1. Skill 和普通 prompt 模板有什么区别？

普通 prompt 模板通常是一段要复制进上下文的文本。Skill 是一个可发现、可按需加载、可维护的任务包。

Hermes 的 skill 至少包含这些工程属性：frontmatter metadata、平台/环境条件、主说明、支持文件、脚本、密钥要求、使用统计、安全扫描、写入工具和生命周期管理。它不是“把提示词放进文件”这么简单。

### 2. 模型什么时候知道该用哪个 skill？

第一层靠 system prompt 的 `<available_skills>` 索引。模型每轮都会看到 skill 名、分类和描述。Prompt 里明确要求：如果 skill 匹配或部分相关，就必须 `skill_view(name)`。

第二层靠 `skills_list`。当模型不确定名字或类别时，可以查询完整 metadata 列表。

第三层靠用户显式指令。用户可以要求加载某个 skill，或者任务本身触发特定 skill 的描述。

这仍然依赖模型判断，但不是“全靠智力”。Hermes 把可选项、触发描述和强制行为规则放进 prompt，再用工具把完整内容按需拿进来。

### 3. 为什么 skill 正文不直接进 system prompt？

因为 skill 正文太长，而且大多数时候无关。system prompt 只放索引，可以让模型知道“有哪些方法”，同时避免无关正文干扰推理。

真正需要时再 `skill_view`。这样既省 token，也降低 prompt 污染，还能保持 provider prefix cache 更稳定。

### 4. Skill Review 会不会乱写一堆小 skill？

Hermes 用 prompt 和工具保护尽量避免这个问题。review prompt 要求优先更新已加载 skill，其次更新已有 umbrella，再添加支持文件，最后才创建新的 class-level skill。

但这不是绝对保证。后台 review 仍然是 LLM 判断，所以 Hermes 又加了 curator、usage telemetry、archive/restore、pinned guard 和 protected skills 边界。模型负责提出更新，运行时负责限制破坏半径。

### 5. `skill_manage` 写入失败时模型怎么恢复？

不同失败会返回不同结构化信息。比如 patch 找不到文本时，会返回 `file_preview` 和匹配失败提示；安全扫描失败会回滚并返回错误；重名会返回冲突信息；删除 pinned skill 会提示用户先 unpin。

这和 Hermes 其他工具的设计一致：工具失败不能只是“失败”，要把下一步能怎么修告诉模型。

### 6. Skill 适合保存用户偏好吗？

看偏好类型。

“用户喜欢中文”适合 Memory。
“写开源学习笔记时，每个功能后面必须跟源码和工程实现”更适合 skill 或项目指导，因为它是某类任务的方法。

如果一个偏好会改变某类任务的执行步骤，就应该进入对应 skill。否则模型知道用户偏好，但具体做任务时仍可能按旧流程走。

### 7. Skills Guard 和上一讲的 memory threat scanner 是一回事吗？

不是同一个入口，但思路相通。

Memory 写入用 `tools/threat_patterns.py` 的 strict 规则，扫描一段将进入 system prompt 的文本。

Skill 安装和写入用 `tools/skills_guard.py`，扫描的是一个技能目录。它不仅看 prompt injection 和外泄模式，还看文件结构、总大小、二进制、符号链接、`.skillignore`、来源信任和安装策略。

Memory 是一段长期事实。Skill 是一个长期方法包。后者的攻击面更大。

## 本讲要带走的主线

Hermes 的 Skills System 是程序化记忆，不是简单提示词模板。

模型每轮先看到技能索引，而不是所有正文。索引由 `build_skills_system_prompt` 构建，并经过平台、环境、工具条件、禁用列表和缓存处理。

`skill_view` 负责按需加载完整技能和支持文件。它处理插件命名空间、外部目录、重名冲突、平台检查、密钥前提和 linked files。

`skill_manage` 负责创建、修改、删除和写支持文件。写入路径有审批、frontmatter 校验、atomic write、安全扫描、cache invalidation 和 usage telemetry。

后台 Skill Review 在工具迭代达到阈值后、用户收到回复之后运行。它 fork 一个隔离 agent，只允许 memory/skill 工具，用来把非平凡流程沉淀到技能库。

长期维护交给 usage telemetry 和 curator。Hermes 不只是会创建 skill，还要知道哪些 skill 被看过、被用过、被修过，哪些可以归档，哪些因为来源或 pinned 状态不能动。
