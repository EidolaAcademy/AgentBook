# 第 35 章：teammate、agent swarms 与团队式协作

前面几章我们一直在讲“子 Agent”。子 Agent 的典型形态是一次性任务：父 Agent 给它一个 prompt，它完成后返回结果。即使它后台运行，本质上仍然是一个任务。

这一章进入另一个层级：teammate，也就是团队成员式 Agent。

teammate 和普通子 Agent 最大的区别是：它不是只执行一次任务后消失，而是可以被命名、被寻址、接收后续消息、保持自己的对话状态，并和其他 teammate 一起形成一个团队。Claude Code 中把这类能力放在 agent swarms / teammate 体系里。

如果普通子 Agent 像“派一个人去查资料”，teammate 更像“把一个人拉进项目组”。它有名字，有角色，有团队归属，有通信方式，有生命周期，也可能有自己的 UI 面板或独立进程。

这一章会讲：

1. teammate 和普通子 Agent 的区别。
2. team file 是什么，为什么需要团队配置文件。
3. teammate 如何生成身份和名字。
4. SendMessage 如何路由消息。
5. mailbox 协议如何让不同 Agent 通信。
6. in-process teammate 和 tmux teammate 有什么不同。
7. teammate 的任务状态、空闲状态和关闭流程。
8. 团队式协作的治理边界。
9. 新手如何实现一个最小团队系统。

## 35.1 从一次性子 Agent 到 teammate

普通子 Agent 的流程是：

```txt
父 Agent -> AgentTool(prompt) -> 子 Agent 运行 -> 返回结果
```

它像函数调用：

```python
result = await runSubAgent(prompt)
```

teammate 的流程更像启动一个长期协作者：

```txt
父 Agent 创建 team
父 Agent spawn teammate: researcher
父 Agent 给 researcher 发消息
researcher 工作
父 Agent 再给 researcher 发消息
researcher 回复或更新任务
父 Agent 关闭 teammate
```

它不是函数调用，而是一个实体：

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class ChildAgentRequest:
    agent_id: str
    task: str
    allowed_tools: list[str] = field(default_factory=list)
    max_turns: int = 8

@dataclass
class ChildAgentResult:
    agent_id: str
    summary: str
    messages: list[dict[str, Any]] = field(default_factory=list)

async def run_child_agent(request: ChildAgentRequest, model: Any, tools: dict[str, Any]) -> ChildAgentResult:
    messages = [{"role": "user", "content": request.task}]
    for _ in range(request.max_turns):
        response = await model.complete(messages, tools)
        messages.append(response)
        if not response.get("tool_uses"):
            return ChildAgentResult(request.agent_id, str(response.get("content", "")), messages)
    return ChildAgentResult(request.agent_id, "子 Agent 达到最大轮数", messages)
```

这个变化非常大。因为一旦 Agent 成为“实体”，系统就需要处理身份、通信、生命周期、团队状态、权限继承、UI 展示和资源清理。

## 35.2 teammate 的适用场景

不是所有任务都应该用 teammate。

适合 teammate 的场景：

- 长时间协作。
- 需要多轮沟通。
- 多个角色同时工作。
- 用户想观察每个成员的进展。
- 任务之间有相互依赖。
- 需要一个 Agent 等待后续指令。

例如：

```txt
team-lead：总体协调。
researcher：搜索代码和文档。
tester：运行测试并汇报失败。
implementer：根据确认后的方案修改代码。
reviewer：审查最终 diff。
```

不适合 teammate 的场景：

- 一次性文件搜索。
- 小范围总结。
- 单个函数修复。
- 不需要后续沟通的后台任务。

这些用普通子 Agent 更简单、更便宜。

一个好的原则是：如果你只需要“结果”，用子 Agent；如果你需要“持续协作关系”，用 teammate。

## 35.3 team file：团队状态的共享账本

团队式协作需要一个共享账本。Claude Code 中有 team file，里面记录团队名、leader、成员、pane、cwd、订阅、权限等信息。

教学版可以设计：

```python
import json
from pathlib import Path
from typing import Any

def read_json(path: Path) -> dict[str, Any]:
    return json.loads(path.read_text(encoding="utf-8"))

def read_jsonl(path: Path) -> list[dict[str, Any]]:
    if not path.exists():
        return []
    return [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines() if line.strip()]

def load_agent_task(task_id: str, root: Path) -> dict[str, Any]:
    metadata = read_json(root / ".agent" / "tasks" / task_id / "metadata.json")
    transcript = read_jsonl(root / ".agent" / "subagents" / f"{task_id}.jsonl")
    return {"id": task_id, "metadata": metadata, "transcript": transcript}
```

这个文件可以放在：

```txt
.agent/teams/<team-name>/config.json
```

为什么要有 team file？

第一，给 SendMessage 找人。模型只知道要发给 `researcher`，系统需要知道 `researcher` 对应哪个 agentId、哪个 mailbox、哪个任务。

第二，给 teammate 发现团队成员。一个 teammate 启动后，可以读取 team file，知道团队里还有谁。

第三，给 UI 展示团队结构。

第四，给恢复和清理提供依据。

第五，给权限协作提供基础。比如某些路径是团队允许编辑的。

## 35.4 创建团队

创建团队的最小流程：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass
class Message:
    role: str
    content: list[dict[str, Any]] | str

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

如果父 Agent 没有显式创建 team，也可以在第一次 spawn teammate 时自动补一个默认 team。但对新手系统，建议让团队创建显式一些，这样状态更清楚。

## 35.5 teammate 身份：name、agentId、teamName

teammate 必须有稳定身份。名字给模型和用户使用，agentId 给系统使用。

例如：

```txt
name: researcher
teamName: auth-fix
agentId: researcher@auth-fix
```

源码里会对名字做 sanitize，避免特殊字符破坏 agentId 或路径。比如 `@` 不能随便出现在 teammate name 中，因为 `name@team` 这种格式会变得模糊。

教学版：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

还要处理重名。如果团队里已经有 `researcher`，新成员可以叫：

```txt
researcher-2
researcher-3
```

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

## 35.6 spawn teammate 的输入

teammate spawn 需要的信息比普通子 Agent 多。

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

`name` 是可寻址名字。

`prompt` 是初始任务。

`team_name` 指定加入哪个团队。如果省略，可以继承当前 team context。

`agent_type` 指定角色定义。

`model` 指定模型。

`cwd` 指定工作目录。

`plan_mode_required` 表示该 teammate 需要先提出计划并获得批准。

这和普通 `AgentTool` 的区别是：普通子 Agent 不一定有名字，teammate 必须有名字。

## 35.7 spawn teammate 的核心流程

最小流程：

```txt
校验 name 和 prompt
        |
        v
确定 teamName
        |
        v
生成唯一 name
        |
        v
生成 teammateId
        |
        v
分配颜色
        |
        v
选择运行后端
        |
        v
启动 teammate
        |
        v
写 AppState teamContext
        |
        v
写 team file
        |
        v
发送初始 prompt
```

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

## 35.8 in-process teammate

in-process teammate 运行在同一个 Python 进程里。它用 AsyncLocalStorage 或类似机制保存自己的 teammate context。

优点：

- 启动快。
- 不需要 tmux。
- 更容易和当前 UI 集成。
- 共享 AppState，消息和任务状态更容易展示。

缺点：

- 同一进程资源竞争更明显。
- 隔离程度低于独立进程。
- 如果进程崩溃，所有 teammate 都受影响。
- 必须非常小心上下文隔离。

源码中的 in-process teammate task 有这些字段：

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

这里有两个值得学习的点。

第一，`abortController` 和 `currentWorkAbortController` 分开。一个用于关闭整个 teammate，一个用于取消当前工作回合。

第二，`isIdle` 很重要。teammate 不是普通后台任务，普通任务完成后结束；teammate 可以空闲等待下一条消息。

## 35.9 tmux teammate

tmux teammate 是另一个进程，运行在 tmux session 或 pane 中。

优点：

- 进程隔离更强。
- 每个 teammate 有自己的终端。
- 崩溃影响范围更小。
- 用户可以直接观察或进入 pane。

缺点：

- 依赖 tmux 或终端后端。
- 启动和管理更复杂。
- 消息传递需要 mailbox。
- 跨进程状态同步更麻烦。

源码中会构造 CLI 命令：

```txt
claude --agent-id ... --agent-name ... --team-name ...
```

并把权限模式、模型、settings、plugin-dir 等参数传给 teammate。这里的设计原则是：teammate 是一个新的 Claude Code 实例，但它带着团队身份启动。

教学版如果不想依赖 tmux，可以先只做 in-process teammate。等机制跑通后，再支持 out-of-process。

## 35.10 后端选择与 fallback

生产系统不能假设所有环境都有 tmux 或 iTerm2。后端选择大致是：

```txt
如果启用 in-process：用 in-process
否则检测 pane backend
  - iTerm2
  - tmux
如果检测失败：fallback 到 in-process 或报错
```

教学版：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

注意，fallback 不是随便做的。如果用户显式要求 tmux，而 tmux 不可用，应该给出清楚错误，而不是悄悄换成 in-process。

## 35.11 SendMessage：团队通信工具

有了 teammate，必须有通信工具。Claude Code 中是 `SendMessage`。

最基本输入：

```python
from dataclasses import dataclass, field
from typing import Literal

@dataclass
class Finding:
    file_path: str
    message: str
    severity: Literal["low", "medium", "high"]

@dataclass
class AgentResult:
    summary: str
    findings: list[Finding] = field(default_factory=list)
    files_read: list[str] = field(default_factory=list)
    commands_run: list[str] = field(default_factory=list)
    changed_files: list[str] = field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"

def format_agent_result(result: AgentResult) -> str:
    parts = [f"Summary:\n{result.summary}"]
    if result.findings:
        findings = "\n".join(f"- {item.severity}: {item.file_path}: {item.message}" for item in result.findings)
        parts.append(f"Findings:\n{findings}")
    if result.changed_files:
        parts.append("Changed files:\n" + "\n".join(result.changed_files))
    parts.append(f"Confidence: {result.confidence}")
    return "\n\n".join(parts)
```

`to` 可以是：

- teammate name，例如 `researcher`
- `*`，表示广播
- 某些扩展地址，例如跨会话 bridge 或 UDS

教学版先支持 teammate name 和 `*`。

发送给单个 teammate：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

广播：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

## 35.12 为什么 message 需要 summary

源码中对普通字符串消息要求 `summary`。这是一个很实用的设计。

原因是消息正文可能很长，UI、日志、工具结果不应该总是显示全文。summary 可以作为短标题：

```txt
"focus on refresh token" -> researcher
```

教学版也建议保留：

```python
from dataclasses import dataclass, field
from typing import Literal

@dataclass
class Finding:
    file_path: str
    message: str
    severity: Literal["low", "medium", "high"]

@dataclass
class AgentResult:
    summary: str
    findings: list[Finding] = field(default_factory=list)
    files_read: list[str] = field(default_factory=list)
    commands_run: list[str] = field(default_factory=list)
    changed_files: list[str] = field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"

def format_agent_result(result: AgentResult) -> str:
    parts = [f"Summary:\n{result.summary}"]
    if result.findings:
        findings = "\n".join(f"- {item.severity}: {item.file_path}: {item.message}" for item in result.findings)
        parts.append(f"Findings:\n{findings}")
    if result.changed_files:
        parts.append("Changed files:\n" + "\n".join(result.changed_files))
    parts.append(f"Confidence: {result.confidence}")
    return "\n\n".join(parts)
```

这和前面 `description` / `prompt` 分离是同一个思想：短描述和完整内容不要混成一个字段。

## 35.13 SendMessage 的路由顺序

SendMessage 的路由比看起来复杂。源码中有一个很重要的路径：先尝试把 `to` 当成已注册的后台 Agent name 或 agentId，如果找到，就把消息加入 pending queue 或恢复后台 Agent；如果找不到，再走团队 mailbox 路由。

简化流程：

```txt
SendMessage(to, message)
        |
        v
如果 to 是后台 Agent 名称或 agentId
        |
        v
running：queuePendingMessage
stopped：resumeAgentBackground
        |
        v
否则按 team teammate 路由
        |
        v
to="*"：broadcast
to=name：writeToMailbox(name)
```

为什么要这样？因为系统里有两类“可继续沟通的 Agent”：

第一类是普通后台 Agent。它可能通过 `AgentTool` 启动，并注册到了 `agentNameRegistry`。

第二类是 teammate。它存在 team file 和 mailbox 中。

这两类都能被 SendMessage 触达，但机制不同。

## 35.14 给运行中的后台 Agent 发消息

如果目标是普通后台 Agent，而且还在 running，不能立即把消息插入上下文。应该排队到下一轮工具边界。

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

返回：

```txt
Message queued for delivery to researcher at its next tool round.
```

这个机制和 teammate mailbox 不同。它是 task 内部的 pending message queue。

## 35.15 给 stopped Agent 发消息：自动恢复

如果 Agent 已经 stopped，但有 transcript，可以恢复它：

```python
import json
from dataclasses import asdict, dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: dict[str, Any]
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())

def append_transcript(path: Path, entry: TranscriptEntry) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("a", encoding="utf-8") as file:
        file.write(json.dumps(asdict(entry), ensure_ascii=False) + "\n")

def load_transcript(path: Path) -> list[dict[str, Any]]:
    if not path.exists():
        return []
    return [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines() if line.strip()]
```

这说明“SendMessage”不只是即时通信工具，也是一种继续后台 Agent 的入口。

但恢复必须谨慎：

- transcript 是否存在？
- worktreePath 是否还存在？
- agentType 是否能恢复？
- 权限上下文是否仍然允许？
- 原工具池是否还能重建？

恢复失败要返回清楚错误，而不是假装消息已送达。

## 35.16 teammate mailbox

tmux teammate 是独立进程，不能直接访问父进程内存里的 pending queue。所以需要 mailbox。

mailbox 可以是文件：

```txt
.agent/teams/auth-fix/mailboxes/researcher.jsonl
```

写入：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass
class Message:
    role: str
    content: list[dict[str, Any]] | str

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

teammate 自己定期读取 mailbox，把新消息注入自己的 Agent 循环。

in-process teammate 可以直接收到初始 prompt，不需要通过 mailbox，否则会重复收到同一条初始消息。源码中特别强调了这一点。

## 35.17 teammate 系统提示词

teammate 必须知道：它写在自己普通回答里的文字，不会自动被其他队友看到。它必须用 SendMessage。

系统提示词可以加入：

```txt
你正在作为团队中的一个 Agent 运行。
如果要和团队成员沟通，必须使用 SendMessage。
只是写普通回复不会被其他 teammate 看见。
用户主要和 team lead 交互。
你的工作通过任务系统和 teammate messaging 协调。
```

这段提示很重要。否则 teammate 可能以为自己在聊天窗口里说一句话，其他人就能看到。但在系统架构上，每个 Agent 有自己的上下文，不使用通信工具就不会跨上下文传播。

## 35.18 广播要谨慎

`SendMessage(to: "*")` 很方便，但容易制造噪声。

广播适合：

- 全体停止。
- 全体汇报进度。
- 全体同步新约束。
- 任务阶段切换。

不适合：

- 给某个 Agent 的具体细节。
- 长篇代码片段。
- 高频状态更新。

可以在 prompt 中约束：

```txt
只有当消息确实影响所有 teammate 时才使用广播。
普通协作应发送给具体 teammate。
```

还可以在程序层限制广播频率。

## 35.19 结构化消息

除了普通文本，SendMessage 还可以支持结构化消息，例如：

- shutdown_request
- shutdown_response
- plan_approval_response

教学版：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Any

@dataclass
class Message:
    role: str
    content: list[dict[str, Any]] | str

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

结构化消息的好处是可验证、可渲染、可触发状态变化。

比如关闭请求：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

teammate 可以回复 approve 或 reject。

## 35.20 plan mode approval

有些 teammate 不能直接执行修改，而必须先提出计划并等待批准。这就是 plan mode。

适用场景：

- 高风险修改。
- 大规模重构。
- 需要用户确认的操作。
- team lead 想控制实现方向。

状态：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

流程：

```txt
teammate 提出计划
        |
        v
awaitingPlanApproval = true
        |
        v
team lead 审核
        |
        v
SendMessage(plan_approval_response)
        |
        v
approve：继续执行
reject：按反馈修改计划
```

这比让 teammate 直接执行更安全。

## 35.21 teammate 不能无限创建 teammate

源码里有一个限制：teammate 不能再 spawn teammate。原因是 team roster 是扁平的，leader 是唯一协调者。如果 teammate 可以无限创建 teammate，会出现：

- 团队层级混乱。
- 谁负责谁不清楚。
- 消息路由复杂化。
- 成本不可控。
- UI 很难展示。

所以规则是：

```txt
teammate 可以启动普通同步 subagent
teammate 不能创建新的 teammate
```

教学版可以这样检查：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

这体现了一个重要治理原则：团队结构要保持简单。

## 35.22 in-process teammate 不能随便启动后台 Agent

源码里还有一个限制：in-process teammate 不能启动后台 Agent。原因是它的生命周期绑定在 leader 进程和 task 状态里，后台嵌套会让取消、清理和 UI 状态变得复杂。

教学版：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

这不是能力不足，而是为了避免生命周期嵌套失控。

## 35.23 teammate task 与普通 agent task 的区别

普通后台 Agent：

```txt
running -> completed / failed / killed
```

teammate：

```txt
running(active) <-> running(idle)
running -> killed
```

teammate 可以长时间 idle。它不是 completed，因为它还可以继续接收消息。

任务字段中需要：

```python
from dataclasses import dataclass
from typing import Any, Literal

Decision = Literal["allow", "deny", "ask"]

@dataclass
class PermissionDecision:
    decision: Decision
    reason: str = ""
    rule: str | None = None

def check_tool_permission(tool_name: str, tool_input: dict[str, Any]) -> PermissionDecision:
    command = str(tool_input.get("command", ""))
    if tool_name == "Bash" and command.startswith("rm -rf"):
        return PermissionDecision("deny", "危险删除命令", "Bash(rm -rf*)")
    if tool_name in {"Read", "Grep"}:
        return PermissionDecision("allow")
    return PermissionDecision("ask", f"需要用户确认: {tool_name}")
```

用户发消息时：

```python
injectUserMessageToTeammate(taskId, message)
```

这个函数即使 teammate idle 也应该接受消息；只在 terminal state 下拒绝。

## 35.24 UI 消息镜像要限量

in-process teammate 的完整会话可能很长。如果 UI task state 里保存所有 messages，会造成巨大内存占用。

源码中对 teammate UI messages 做了 cap，例如只保留最近 50 条。完整 transcript 仍然在本地 allMessages 或磁盘里，UI 只是镜像。

教学版：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

这是一个很现实的工程优化。多 Agent 系统里，内存不是抽象问题。几十个 teammate 每个保存完整历史，会很快膨胀。

## 35.25 颜色和可视化

Claude Code 会给 teammate 分配颜色。颜色看似 UI 细节，其实对多 Agent 协作很重要。

当有多个 Agent 同时输出时，用户需要快速知道：

- 谁在工作？
- 谁发了消息？
- 谁等待计划批准？
- 谁完成了任务？

教学版可以实现：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

颜色不是核心逻辑，但它让复杂系统可观察。

## 35.26 团队任务列表

teammate 不应该只靠聊天协作。它们还需要任务列表。

一个团队任务可以是：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

team lead 可以创建任务：

```txt
task-1: researcher 检查 auth 相关文件
task-2: tester 运行 auth 测试
task-3: reviewer 审查最终 diff
```

teammate 定期查看任务列表，领取或更新任务。

这比纯消息流更稳定。消息适合沟通，任务列表适合状态。

## 35.27 teammate 的关闭流程

不要直接杀掉 teammate。更好的流程是请求关闭。

```txt
team lead -> shutdown_request -> teammate
teammate 保存状态、汇报结果
teammate -> shutdown_response approve
系统移除 teammate
```

如果 teammate 拒绝：

```txt
shutdown_response reject: 我还在运行测试，请等待。
```

教学版可以这样：

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

然后通过 SendMessage 处理 approve / reject。

## 35.28 team cleanup

团队结束后要清理：

- team file。
- mailbox 文件。
- teammate tasks。
- tmux windows 或 panes。
- worktrees。
- 临时任务目录。

清理前必须确认没有 active members。

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

不要在 teammate 还活着时删除 mailbox，否则消息可能丢失。

## 35.29 最小可实现团队系统

新手可以按这个顺序实现：

1. TeamFile 读写。
2. spawnTeam。
3. spawnTeammate，只支持 in-process。
4. teammateId = name@team。
5. SendMessage(to=name)。
6. teammate pendingUserMessages。
7. teammate idle/active 状态。
8. shutdown_request / shutdown_response。
9. UI 最近消息 cap。
10. team cleanup。

暂时不要做：

- tmux 后端。
- 跨机器 bridge。
- 复杂 plan mode。
- 自动任务分配。
- teammate 再委派。

先把身份和通信跑通。

## 35.30 一个完整示例

父 Agent 创建团队：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

创建 researcher：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

创建 tester：

```python
from pathlib import Path

def resolve_inside_workspace(workspace: Path, user_path: str) -> Path:
    root = workspace.resolve()
    target = (root / user_path).resolve()
    if target != root and root not in target.parents:
        raise ValueError(f"路径越界: {user_path}")
    return target

def read_text_file(workspace: Path, user_path: str, limit: int = 200) -> str:
    file_path = resolve_inside_workspace(workspace, user_path)
    lines = file_path.read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:limit])
```

给 researcher 发后续消息：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

广播阶段切换：

```python
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]

async def run_agent_step(model: Any, messages: list[dict[str, Any]], tools: dict[str, Any]) -> dict[str, Any]:
    response = await model.complete(messages, tools)
    messages.append(response)
    return response
```

最后关闭：

```python
from dataclasses import dataclass

@dataclass
class SearchResult:
    tool_name: str
    score: int
    description: str

def search_tools(query: str, tools: list[dict[str, str]], limit: int = 5) -> list[SearchResult]:
    terms = [term.lower() for term in query.split() if term.strip()]
    results: list[SearchResult] = []
    for tool in tools:
        haystack = f"{tool.get('name', '')} {tool.get('description', '')}".lower()
        score = sum(1 for term in terms if term in haystack)
        if score:
            results.append(SearchResult(tool.get("name", ""), score, tool.get("description", "")))
    return sorted(results, key=lambda item: item.score, reverse=True)[:limit]
```

这就是一个最小团队协作闭环。

## 35.31 团队式协作的风险

teammate 很强，但风险也更大。

第一，成本更难控制。多个 teammate 可以同时运行、等待、接收消息、继续工作。

第二，消息噪声会增加。如果每个 Agent 都频繁广播，父 Agent 上下文会被通信淹没。

第三，权限边界更复杂。teammate 是否继承 leader 权限？是否能申请权限？是否能写文件？

第四，生命周期更复杂。teammate idle 时算不算运行？关闭前是否要等待它完成当前工具？

第五，用户理解成本更高。用户需要知道团队里有哪些成员，各自负责什么。

所以团队式协作必须有治理策略：

- 最大 teammate 数量。
- 广播频率限制。
- 明确角色和任务。
- 默认只读。
- 写操作用 worktree。
- 关闭流程要温和。
- 所有通信可审计。

## 35.32 本章小结

这一章我们从普通子 Agent 走到了 teammate 和 agent swarms。

普通子 Agent 是一次性任务；teammate 是长期协作者。teammate 有名字、团队、身份、消息通道、任务状态和生命周期。它可以通过 SendMessage 接收消息，也可以通过 team file 和 mailbox 发现团队成员。

关键设计包括：

1. 用 team file 记录团队成员和共享状态。
2. 用 name@teamName 形成稳定 agentId。
3. 用 SendMessage 做显式通信。
4. 用 mailbox 支持跨进程 teammate。
5. in-process teammate 依赖上下文隔离和任务状态。
6. tmux teammate 依赖进程和终端后端。
7. teammate 可以 idle，普通后台 Agent 通常不会。
8. teammate 关闭应该走 shutdown 协议。
9. UI 消息镜像要限量，完整 transcript 另存。
10. 团队式协作必须有预算、权限和生命周期治理。

下一章我们会讲 Agent memory、skills preload 与长期协作能力。那一章会解释：当 Agent 不只是一次会话里的临时工具，而需要长期积累项目知识、预加载技能、在不同任务间复用经验时，系统应该怎样设计。
