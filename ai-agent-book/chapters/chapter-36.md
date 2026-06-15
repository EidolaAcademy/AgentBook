# 第 36 章：Agent memory、skills preload 与长期协作能力

到目前为止，我们讲的 Agent 大多围绕“当前任务”。它能读代码、调用工具、启动子 Agent、创建 teammate、在 worktree 里修改文件。这些能力已经足以完成很多复杂工作。

但真实工程协作还有一个问题：Agent 能不能记住长期经验？

例如：

- 这个项目的测试必须连真实数据库，不能用 mock。
- 这个团队更喜欢小 PR，而不是大重构。
- 某个 Agent 做安全审查时，总是需要先看 `docs/security.md`。
- 某个项目的测试命令不是 `pytest`，而是 `uv run pytest tests/web`。
- 某类任务应该使用某个 skill。

如果每次会话都从零开始，Agent 会重复犯错、重复搜索、重复询问。长期协作能力就是为了解决这个问题。

Claude Code 中有两类重要机制：

1. Agent memory：让特定 Agent 类型拥有长期记忆。
2. skills preload：让 Agent 启动时预加载某些技能说明。

这章会讲它们的区别、实现方式、安全边界和新手实现路径。

## 36.1 memory 和 skill 的区别

memory 和 skill 很容易混淆。

memory 是“长期事实或偏好”。

例如：

```txt
这个项目的 API 集成测试必须连接真实数据库。
用户偏好回答简短，不要长总结。
auth 模块重构是合规要求，不是技术债清理。
```

skill 是“如何做某类任务的操作说明”。

例如：

```txt
如何创建功能规格文档。
如何审查 React 性能问题。
如何生成发布说明。
如何做数据库迁移审查。
```

一个简单判断：

```txt
以后需要记住的事实 -> memory
以后需要重复执行的方法 -> skill
当前任务的一次性计划 -> task 或 plan
```

不要把所有东西都塞进 memory。memory 越多，越容易污染上下文。

## 36.2 为什么 Agent memory 要按 Agent 类型隔离

不同 Agent 需要不同记忆。

`security-reviewer` 需要记住：

- 项目的安全边界。
- 常见风险点。
- 合规要求。

`test-fixer` 需要记住：

- 测试命令。
- 测试环境要求。
- flaky test 处理方式。

`frontend-designer` 需要记住：

- 设计系统。
- UI 规范。
- 组件库约束。

如果所有 Agent 共用一份 memory，会出现两个问题。

第一，噪声太多。test-fixer 不需要读一堆 UI 设计偏好。

第二，边界不清。某些安全审查记忆不一定适合普通总结 Agent 使用。

所以 Claude Code 中 memory 是按 agentType 存储的：

```txt
agent-memory/<agentType>/MEMORY.md
```

教学版也建议这样设计：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

## 36.3 memory scope：user、project、local

Claude Code 的 Agent memory 有三个 scope：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

它们对应不同用途。

`user` scope：跨项目的用户级记忆。

适合：

- 用户偏好。
- 用户技能背景。
- 用户常用表达方式。
- 适用于所有项目的习惯。

例如：

```txt
用户是后端工程师，对 React 不熟悉，解释前端概念时可以类比后端。
```

`project` scope：项目级、可团队共享的记忆。

适合：

- 项目架构约定。
- 测试策略。
- 发布流程。
- 团队共识。

例如：

```txt
本项目的 integration tests 必须连接真实数据库，不能使用 mock。
```

`local` scope：本机本项目私有记忆。

适合：

- 本机路径。
- 本地开发环境。
- 未提交配置。
- 个人调试习惯。

例如：

```txt
本机 Redis 测试实例运行在 localhost:6380。
```

这个三层 scope 非常实用。因为不是所有记忆都应该共享，也不是所有记忆都应该跨项目。

## 36.4 memory 目录设计

教学版可以这样设计：

```txt
~/.agent/agent-memory/<agentType>/MEMORY.md
项目/.agent/agent-memory/<agentType>/MEMORY.md
项目/.agent/agent-memory-local/<agentType>/MEMORY.md
```

对应函数：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

注意 agentType 要 sanitize。插件 Agent 可能叫：

```txt
my-plugin:security-reviewer
```

冒号在某些系统路径里不合适，可以替换成 `-`：

```python
def sanitizeAgentTypeForPath(agentType: str):
    return agentType.replace(/:/g, '-')
```

## 36.5 memory prompt 如何注入系统提示词

Agent 启动时，如果定义中有：

```yaml
---
name: security-reviewer
memory: project
---
```

系统会把 memory prompt 追加到这个 Agent 的 system prompt 中。

教学版：

```python
def getSystemPrompt(agent: AgentDefinition):
    prompt = agent.basePrompt

    def if(self, agent.memory):
        prompt += '\n\n' + loadAgentMemoryPrompt(
        agent.agentType
        agent.memory
        )

    return prompt
```

`loadAgentMemoryPrompt` 做三件事：

1. 计算 memory 目录。
2. 确保目录存在。
3. 构造一段告诉 Agent 如何使用 memory 的提示词。

```python
from dataclasses import dataclass
from typing import Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionRule:
    source: str
    behavior: Decision
    tool_name: str
    rule_content: str = ""

def format_rule(rule: PermissionRule) -> str:
    content = f"({rule.rule_content})" if rule.rule_content else ""
    return f"{rule.source}:{rule.behavior}:{rule.tool_name}{content}"

def deny_message(rule: PermissionRule) -> dict[str, str]:
    return {
        "behavior": "deny",
        "message": f"Denied by {format_rule(rule)}",
        "reason": "Matched deny rule",
    }
```

这里的重点是：memory prompt 不一定直接把所有记忆内容塞进上下文。更好的方式是告诉 Agent memory 在哪里、什么时候读、什么时候写。

## 36.6 memory-enabled Agent 为什么要自动补 Read/Edit/Write

如果一个 Agent 有 memory，但它的工具列表没有 Read/Edit/Write，它就无法读取或更新 memory。

Claude Code 中有一个设计：当 memory 启用时，如果 Agent 显式声明了 tools，就把 Read/Edit/Write 注入工具列表。

教学版：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

注意只在 tools 显式存在时处理。若 tools undefined 表示允许默认工具集合，就不用特别加。

## 36.7 memory 不是任务列表

源码中的 memory prompt 明确区分 memory、plan 和 task。这一点非常重要。

不要把下面内容写入 memory：

```txt
今天要修复登录 bug。
我刚刚运行了 pytest。
下一步要读 src/auth/token.py。
```

这些属于当前任务状态，应该放到 todo、plan 或当前上下文。

适合 memory 的内容是：

```txt
本项目登录测试需要先启动 test-db，否则 auth integration tests 会失败。
```

判断标准：

```txt
未来其他会话是否仍然有价值？
```

如果答案是否定的，就不要保存为 memory。

## 36.8 memory 的安全边界

memory 很强，也很危险。

风险一：错误记忆长期污染。

如果 Agent 错误记住“测试不需要真实数据库”，以后可能反复做错。

风险二：敏感信息泄露。

不要把 token、密码、私钥写入 project memory，尤其 project memory 可能进入版本控制。

风险三：用户临时要求被长期化。

用户说“这次先跳过测试”，不代表以后都跳过测试。

风险四：跨项目污染。

user memory 如果记录了某个项目专有约定，会影响其他项目。

所以 memory prompt 应该包含规则：

```txt
只保存经过验证、未来仍有用的信息。
不要保存秘密。
不要保存一次性任务状态。
不要把项目专有内容保存到 user scope。
```

## 36.9 memory snapshot

Claude Code 中还有 memory snapshot 机制。它允许项目提供一份 memory snapshot，首次初始化时复制到本地 memory。

为什么需要 snapshot？

比如团队希望 `security-reviewer` 一开始就知道项目安全边界，但又不想直接覆盖每个人本地改过的 memory。

目录可以是：

```txt
.agent/agent-memory-snapshots/security-reviewer/
  snapshot.json
  MEMORY.md
```

`snapshot.json`：

```json
{
  "updatedAt": "2026-06-15T12:00:00Z"
}
```

检查逻辑：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

第一次没有本地 memory 时，可以自动初始化。已有本地 memory 且 snapshot 更新时，不要直接覆盖，而是提示或标记 pending update。

## 36.10 为什么不能随便覆盖本地 memory

本地 memory 可能包含用户和 Agent 长期积累的经验。项目 snapshot 更新后，如果系统直接覆盖本地 memory，可能丢失重要内容。

所以更安全的策略是：

- 本地为空：自动复制 snapshot。
- 本地已有：提示有更新。
- 用户确认后再替换或合并。

教学版：

```python
def if(self, !hasLocalMemory):
    await initializeFromSnapshot(agentType, scope)
    } else if (snapshotIsNewer):
        agent.pendingSnapshotUpdate =:
            "snapshotTimestamp": snapshot.updatedAt
```

这体现了一个原则：长期记忆属于用户和团队资产，不能静默覆盖。

## 36.11 skills 是什么

skill 是可复用的任务说明。它通常是一个 Markdown 文件，告诉 Agent 如何完成某类工作。

例如 `feature-spec` skill：

```md
# Feature Spec Skill

Use this skill when the user asks for a product feature specification.

Steps:
1. Clarify user goal.
2. Identify personas.
3. Write requirements.
4. Define edge cases.
5. Produce acceptance criteria.
```

skill 和普通 prompt 的区别是：

- skill 可以被发现。
- skill 可以有描述、whenToUse、allowedTools。
- skill 可以注册 hooks。
- skill 可以由 Agent 预加载。
- skill 可以来自用户目录、项目目录、插件或内置包。

## 36.12 Agent frontmatter 中声明 skills

Agent 定义可以写：

```yaml
---
name: product-planner
description: Plan product features
skills:
  - feature-spec
  - user-story-map
---

你是一个产品规划 Agent。
```

启动该 Agent 时，系统预加载这些 skills。

源码中的流程是：

```txt
读取 agentDefinition.skills
        |
        v
加载所有可用 skills
        |
        v
解析 skill 名称
        |
        v
过滤不存在或非 prompt skill
        |
        v
并发读取 skill prompt
        |
        v
作为 meta user message 加入 initialMessages
```

教学版：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, cwd: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=cwd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

## 36.13 skill 名称解析

插件 skills 可能带命名空间：

```txt
product-management:feature-spec
```

但 Agent frontmatter 里可能只写：

```yaml
skills:
  - feature-spec
```

源码中解析顺序大致是：

1. 精确匹配。
2. 如果 Agent 来自插件，尝试加插件前缀。
3. 后缀匹配 `:skillName`。

教学版：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, cwd: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=cwd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

这让插件 Agent 可以更自然地引用同插件下的 skill。

## 36.14 skill preload 和 SkillTool 调用的区别

skill preload 是 Agent 启动时自动加载。

SkillTool 调用是模型运行过程中主动请求某个 skill。

preload 适合：

- 这个 Agent 总是需要的能力。
- Agent 角色定义的一部分。
- 成本可接受的核心指南。

SkillTool 适合：

- 任务中偶尔需要。
- skill 很长，不想每次都加载。
- 模型根据上下文选择是否使用。

不要把所有 skill 都 preload。这样会让初始上下文变大。应该只 preload 角色必需的技能。

## 36.15 skill loading metadata

源码在加载 skill 时会插入 metadata，让 UI 和消息系统知道这是 skill loading。

教学版可以简单：

```python
import asyncio
from dataclasses import dataclass

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(command, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
        return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
```

然后：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

这样 transcript 中能看出某段内容来自 skill，而不是普通用户输入。

## 36.16 skill hooks

skill 可以有 hooks。例如加载某个 skill 时注册额外检查。

但 hooks 有安全风险。用户 skill、项目 skill、插件 skill 的信任级别不同。

源码里有一个策略：如果系统处于 plugin-only policy，就只允许 admin trusted 来源注册 hooks。

教学版可以定义：

```python
def canRegisterSkillHooks(skill: SkillCommand):
    if (not policy.strictPluginOnlyHooks) return True
    return skill.source == 'plugin'  or  skill.source == 'policy'
```

原则是：内容可以读，hook 执行要更谨慎。因为 hook 是代码或命令，风险高于文本 prompt。

## 36.17 skills 与 compaction

当会话 compact 后，系统可能丢掉旧消息。如果 skill 内容丢失，Agent 后续可能忘记正在使用的 skill。

源码会记录 invoked skills，并按 agentId 作用域保存，避免不同 Agent 的 skill 混在一起。

教学版：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

compact 时：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

重点是 agentId。没有它，子 Agent A 的 skill 可能被恢复到子 Agent B 的上下文里，造成交叉污染。

## 36.18 memory 与 skills 如何配合

一个长期协作 Agent 可以同时使用 memory 和 skills。

例如 `security-reviewer`：

```yaml
---
name: security-reviewer
description: Review security risks
memory: project
skills:
  - threat-model-review
  - secure-code-review
tools:
  - Read
  - Grep
---
```

memory 让它知道项目长期安全约定。

skills 让它知道如何执行威胁建模和代码审查。

它们的关系是：

```txt
memory：这个项目有哪些长期事实？
skill：这类任务应该怎么做？
prompt：这次具体要做什么？
```

三者结合，Agent 才会既有背景，又有方法，又有当前目标。

## 36.19 initialPrompt

Agent 定义还可以有 `initialPrompt`。它和 memory、skills 又不同。

`initialPrompt` 是每次 Agent 启动时追加到第一轮任务前的提示。它适合放：

- 启动仪式。
- 固定检查步骤。
- 必须先做的动作。

例如：

```yaml
initialPrompt: "开始前先读取 docs/testing.md，确认测试命令。"
```

不要把很长的 skill 内容放进 initialPrompt。那应该是 skill。

## 36.20 长期能力的成本控制

memory、skills、initialPrompt 都会增加上下文。

控制原则：

1. memory 按 Agent 类型隔离。
2. memory 按 scope 隔离。
3. skill 不要全部 preload。
4. 大 skill 让模型通过 SkillTool 按需加载。
5. compact 后只恢复当前 Agent 需要的 skill。
6. memory 中不要写长日志。

可以设置预算：

```python
MAX_PRELOADED_SKILL_CHARS = 30_000
MAX_MEMORY_PROMPT_CHARS = 20_000
```

超出时：

```txt
Skill too large to preload. Use SkillTool when needed.
```

## 36.21 新手实现路线

建议按这个顺序做：

1. AgentDefinition 增加 `memory?: 'user' | 'project' | 'local'`。
2. 实现 memory 目录路径。
3. 实现 `loadAgentMemoryPrompt`。
4. 启动 Agent 时追加 memory prompt。
5. memory-enabled Agent 自动获得 Read/Edit/Write。
6. AgentDefinition 增加 `skills?: string[]`。
7. 加载 skill registry。
8. 实现 skill name resolution。
9. 启动 Agent 时 preload skills。
10. skill invocation 按 agentId 记录。
11. compact 后恢复当前 Agent 的 skills。
12. 增加 snapshot 初始化。

不要一开始就做 memory 自动总结或自动写入。先让 Agent 明确读写 MEMORY.md，这更可控。

## 36.22 测试 memory

测试一：路径隔离。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试二：scope 隔离。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试三：plugin agentType sanitize。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试四：memory tool injection。

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if root != target and root not in target.parents:
        raise ValueError(f"Path is outside current workspace: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str) -> str:
    return resolve_inside_workspace(workspace, user_path).read_text(encoding="utf-8")
```

## 36.23 测试 skills

测试一：精确匹配。

```python
expect(resolveSkillName('feature-spec', skills, agent))
.toBe('feature-spec')
```

测试二：插件前缀匹配。

```python
agent = { agentType: 'pm:planner' }
expect(resolveSkillName('feature-spec', skills, agent))
.toBe('pm:feature-spec')
```

测试三：不存在 skill 不应崩溃。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

测试四：非 prompt skill 不 preload。

```python
expect(preloaded).not.toContain(nonPromptSkill)
```

测试五：skill 记录带 agentId。

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

## 36.24 常见错误

错误一：把所有历史都写 memory。

memory 不是聊天记录。它只保存未来仍有价值的知识。

错误二：project memory 存秘密。

project memory 可能进入版本控制。秘密应该在 local memory 或更安全的 secret store。

错误三：所有 Agent 共用 memory。

这会导致噪声和误用。按 agentType 隔离。

错误四：所有 skills 都 preload。

这会让每次启动都很贵。只 preload 角色必需技能。

错误五：compact 后恢复所有 skills。

应该按 agentId 恢复当前 Agent 的 skills，避免跨 Agent 污染。

## 36.25 本章小结

这一章我们讲了长期协作能力。

memory 解决“Agent 应该记住什么长期事实”，skills 解决“Agent 应该如何执行某类任务”。memory 按 agentType 和 scope 隔离，skills 可以按 Agent 定义预加载，也可以运行时通过 SkillTool 按需加载。

关键原则：

1. memory 不是任务列表。
2. memory 要按 user、project、local 分 scope。
3. memory 要按 agentType 分目录。
4. memory-enabled Agent 需要 Read/Edit/Write 能力。
5. snapshot 可以初始化 memory，但不要静默覆盖本地积累。
6. skills 是操作说明，不是长期事实。
7. skills preload 只用于角色必需能力。
8. skill hooks 要受信任策略约束。
9. invoked skills 要按 agentId 记录，避免 compaction 后跨 Agent 泄漏。
10. memory、skill、prompt 三者分别回答“长期事实”“方法”和“本次目标”。

下一章会继续讲自定义 Agent、插件 Agent 与分发机制。到那里，我们会把一个本地 Agent 系统变成可扩展、可分享、可治理的生态。
