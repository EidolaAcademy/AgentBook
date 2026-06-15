# 第 34 章：多 Agent 任务编排、取消、恢复与冲突处理

前几章我们已经建立了多 Agent 的关键组件：

- `AgentTool` 负责委派任务。
- 子 Agent 上下文隔离负责消息、权限和工具状态。
- worktree isolation 负责文件系统隔离。

但当多个子 Agent 真正并行跑起来以后，一个新的问题出现了：谁来管理这些任务？

如果你只是这样写：

```python
void runChildAgent(prompt)
```

那这个子 Agent 就像被扔进空气里的一段 Promise。它可能成功，可能失败，可能卡住，可能被用户取消，可能产生结果，也可能留下后台命令。父 Agent、用户界面、SDK、日志系统都很难知道它到底处于什么状态。

成熟的多 Agent 系统不能这样做。它必须把子 Agent 当成 task，也就是一个有身份、有状态、有进度、有输出、有取消控制、有生命周期清理的运行对象。

本章会讲：

1. 为什么后台 Agent 必须进入任务系统。
2. 如何设计任务状态机。
3. 如何注册、更新、完成、失败、取消任务。
4. 如何把前台 Agent 切到后台。
5. 如何给父 Agent 和用户发送任务通知。
6. 如何保存和查询任务输出。
7. 如何恢复后台 Agent。
8. 多 Agent 结果冲突时如何处理。

这章对应 Claude Code 源码中的 `LocalAgentTask`、`agentToolUtils`、任务框架、后台通知和进度跟踪逻辑。我们会把这些设计讲成一个新手可以实现的版本。

## 34.1 为什么需要任务系统

在单 Agent 系统里，执行流程很简单：

```txt
用户输入
  |
  v
Agent 思考
  |
  v
调用工具
  |
  v
返回结果
```

主循环一直等待当前任务完成。状态天然存在于调用栈里。

但后台子 Agent 不一样。父 Agent 调用它以后，可能立即继续工作：

```txt
父 Agent 启动子 Agent A
父 Agent 启动子 Agent B
父 Agent 继续回答用户

Agent A 后台运行
Agent B 后台运行
```

这时调用栈已经不够用了。你需要一个任务注册表：

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

每个后台 Agent 都注册成一个任务：

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

有了任务系统，用户界面可以显示任务列表，父 Agent 可以收到完成通知，SDK 可以订阅进度，取消按钮可以找到对应 abort controller，恢复逻辑也可以通过 taskId 找到 transcript。

任务系统是多 Agent 编排的中心。

## 34.2 任务状态机

任务状态机越清楚，系统越稳定。最小状态可以是：

```txt
running -> completed
running -> failed
running -> killed
```

也可以加上：

```txt
pending -> running -> completed
                 |-> failed
                 |-> killed
```

教学版先用四个状态：

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

状态转换规则：

```txt
running 可以变成 completed、failed、killed
completed 不能再变
failed 不能再变
killed 不能再变
```

代码：

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

这能避免一个常见 bug：任务已经完成后，又被迟到的失败回调改成 failed。

## 34.3 注册后台 Agent

后台 Agent 启动时，第一件事不是跑模型，而是注册任务。

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

注册越早越好。因为 Agent 可能在启动后很快失败。如果失败发生在注册前，UI 和父 Agent 可能完全看不到它。

Claude Code 中 `registerAsyncAgent` 会创建 abort controller，初始化输出文件 symlink，创建 task state，注册 cleanup handler，然后放进 AppState。

这些步骤说明一个原则：后台任务必须先变成可管理对象，再开始跑。

## 34.4 更新任务状态要用原子 updater

任务状态可能被多个异步路径更新：

- Agent 正常完成。
- 用户点击取消。
- 自动清理触发。
- 恢复逻辑重新注册。
- 进度事件更新。

所以不要直接读改写：

```python
task = state.tasks[id]
task.status = 'completed'
```

更好的方式是 updater：

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

这个设计有两个好处：

第一，每次更新都基于最新状态，而不是异步函数里捕获的旧状态。

第二，如果 updater 返回原对象，可以避免不必要 UI 刷新。

## 34.5 进度跟踪：工具次数、token 和最近活动

用户最怕后台任务像黑盒一样沉默。任务系统应该记录进度。

一个实用的进度结构：

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

源码中的 progress tracker 会分开记录 input tokens 和 output tokens。原因是模型 API 的 input tokens 往往是当前回合累计上下文，而 output tokens 是每轮新增输出。如果简单相加每轮 input tokens，就会重复计数。

教学版：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

每次收到 assistant message：

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

UI 可以显示：

```txt
Agent 正在搜索 src/auth
已使用 7 个工具
约 42k tokens
```

这会显著提升用户信任。

## 34.6 活动描述应该由工具提供

直接显示工具名通常不够友好：

```txt
Running Grep
Running Read
Running Bash
```

更好的显示：

```txt
Searching for refreshToken
Reading src/auth/token.py
Running pytest auth
```

可以让工具提供 `getActivityDescription`：

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

进度更新时：

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

这样任务系统不需要理解每个工具的输入格式，工具自己知道如何描述自己的活动。

## 34.7 完成任务：先改变状态，再做附加工作

后台 Agent 完成后，系统通常要做很多事情：

- 汇总结果。
- 更新任务状态。
- 运行 handoff 安全检查。
- 清理 worktree。
- 发送通知。
- 写输出文件。
- 停止总结服务。

这些操作里，有些可能很慢。比如 handoff classifier 可能要调用模型，worktree cleanup 可能要执行 Git 命令。如果你等这些都做完再把任务状态改成 completed，等待任务结果的工具可能一直卡住。

源码里有一个非常重要的顺序：先把任务标记为 completed，让等待者立即解锁，然后再做通知增强和清理。

教学版：

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

`completeTask`：

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

这个顺序非常重要。记住一句话：状态转换是主路径，通知和清理是附加路径。附加路径不能阻塞主路径。

## 34.8 失败任务

失败也一样。先把任务标记为 failed，再做清理和通知。

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

生命周期：

```python
try:
    result = await runAgent()
    completeTask(taskId, result, setState)
    await cleanupAndNotifyCompleted()
    } catch (error):
        message = error instanceof Error ? error.message : String(error)
        failTask(taskId, message, setState)
        await cleanupAndNotifyFailed(message)
```

失败通知应该包含：

- taskId。
- description。
- error。
- outputFile。
- 如果有 worktree 修改，也要包含 worktreePath。

即使任务失败，它也可能产生了有用修改或部分分析，不能简单丢掉。

## 34.9 取消任务

取消不是失败。用户主动停止任务，状态应该是 `killed` 或 `cancelled`。

教学版：

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

为什么要用 `killed` 变量？因为状态更新可能发现任务已经不是 running。只有真正发生取消时，才需要发送后续通知或清理输出。

取消时还应该尽量提取部分结果。源码里有类似 `extractPartialResult` 的函数，从已有 assistant messages 中倒着找最后一段文本。

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

取消通知可以写：

```txt
Agent "review auth module" was stopped.
Partial result:
已检查 token.py，尚未检查 session.py。
```

部分结果能帮助父 Agent 和用户决定是否继续。

## 34.10 前台 Agent 转后台

一个很好的用户体验是：子 Agent 刚开始同步运行，如果超过几秒还没结束，就提示用户可以把它放到后台。

流程：

```txt
父 Agent 调用子 Agent
        |
        v
注册为 foreground task
        |
        v
继续同步等待
        |
        v
超过阈值，显示 background hint
        |
        v
用户选择后台
        |
        v
当前工具调用返回 async_launched
        |
        v
子 Agent 继续后台跑
```

教学版可以用一个 Promise 作为 background signal：

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

运行时 race：

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

如果赢的是 background，就停止前台等待，把剩下的 Agent 循环交给后台生命周期继续跑。

这个设计很细，但用户体验很好。短任务同步完成，长任务可后台化。

## 34.11 自动后台化

有些场景下系统可以自动把长任务放到后台。例如超过 120 秒仍未完成，就自动后台运行。

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

任务完成前要取消 timer：

```python
cancelAutoBackground = registerAutoBackground(...)

try:
    await runAgent()
    } finally:
        cancelAutoBackground()
```

注意，不要在每一轮循环里重复注册 timer 或 promise，否则会积累回调。源码里也特别避免了这种问题：background race promise 要在循环外创建。

## 34.12 任务通知

后台任务完成后，父 Agent 怎么知道？一种方式是把通知放进消息队列，让主循环下一轮读到。

通知可以是 XML 或 JSON。教学版用 JSON：

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

入队：

```python
import json

def enqueueAgentNotification(notification: TaskNotification):
    messageQueue.append(:
        "mode": 'task-notification'
        "value": json.dumps(notification)
```

主循环读取后，可以把它作为 attachment 交给父 Agent：

```txt
后台 Agent 已完成。输出文件：...
结果摘要：...
```

通知里包含 worktree 信息很重要。如果子 Agent 在隔离 worktree 里产生了修改，父 Agent 要知道去哪里看。

## 34.13 防止重复通知

异步系统里重复通知很常见：

- 正常完成路径发一次。
- 清理路径又发一次。
- 用户手动查询触发一次。
- 恢复逻辑重放一次。

任务状态里可以放一个 `notified` 字段。

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

发送通知前：

```python
def if(self, !markNotifiedOnce(taskId, setState)):
    return

enqueueAgentNotification(...)
```

这是一个很小但很关键的幂等设计。

## 34.14 输出文件：后台任务不能只存在内存里

后台任务结果最好写到磁盘，原因有三个：

第一，结果可能很大，不适合直接塞进父 Agent 上下文。

第二，进程重启后可以恢复。

第三，父 Agent 可以按需读取。

目录设计：

```txt
.agent/tasks/
  <taskId>/
    output.md
    events.jsonl
    metadata.json
```

任务注册时：

```python
def getTaskOutputPath(taskId: str):
    return ".agent/tasks/{taskId}/output.md"
```

写输出：

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

通知里返回：

```json
{
  "taskId": "agent_123",
  "outputFile": ".agent/tasks/agent_123/output.md"
}
```

父 Agent 不需要立刻读取全文。它可以先看 summary，如果需要再读 outputFile。

## 34.15 增量输出与 outputOffset

后台任务运行时，用户可能想看到新增输出。不要每次都把整个 outputFile 读一遍。

可以记录 offset：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

读取增量：

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

生成 attachment 后更新 offset：

```python
task.outputOffset = delta.newOffset
```

源码里有类似逻辑：生成任务 attachment 时读取新输出，并只应用 offset patch，避免用旧任务快照覆盖任务刚刚完成的状态。

这又是一个异步竞态细节。读取文件时任务可能已经完成，所以更新 offset 时要重新检查 fresh state。

## 34.16 恢复任务

后台 Agent 的恢复分两种。

第一种，任务已经完成，只需要恢复结果和 transcript。

第二种，任务运行到一半，进程重启后想继续。

继续运行比较复杂，需要：

- agentId。
- agentType。
- 原始 prompt。
- sidechain transcript。
- content replacement state。
- worktreePath。
- abort controller。
- 工具池和权限上下文。

教学版可以先实现“恢复可查看状态”，不实现继续运行。

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

高级恢复再考虑继续执行：

```python
async def resumeAgentTask(taskId: str):
    task = await loadAgentTask(taskId)
    context = await rebuildSubagentContext(task)
    return runAgentFromTranscript(context)
```

恢复是大工程，不要一开始就做满。先保证任务可查看、结果不丢。

## 34.17 pendingMessages：给运行中的 Agent 继续发消息

多 Agent 团队模式里，父 Agent 或用户可能想给正在运行的 Agent 发消息。例如：

```txt
先别改 session.py，重点看 token.py。
```

如果 Agent 正在工具执行中，不能立刻把消息插入模型上下文。更安全的方式是排队：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

发送：

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

Agent 循环在工具轮次边界 drain：

```python
def drainPendingMessages(taskId: str):
    task = getTask(taskId)
    messages = task.pendingMessages
    task.pendingMessages = []
    return messages
```

为什么不立即插？因为工具调用和工具结果必须保持顺序。中途插入用户消息可能破坏模型 API 的消息结构。

## 34.18 多 Agent 冲突处理：任务层与文件层

冲突有两类。

第一类是任务层冲突。

例如：

- 两个 Agent 都在解决同一个问题。
- 一个 Agent 说应该重构，另一个说应该最小修复。
- 两个 Agent 的结论互相矛盾。

第二类是文件层冲突。

例如：

- 两个 Agent 都修改 `src/auth/token.py`。
- 一个 Agent 修改接口，另一个 Agent 修改调用方。
- 两个 patch 无法同时 apply。

任务系统应该记录每个 Agent 的改动文件：

```python
from dataclasses import dataclass
from typing import Any, Protocol

@dataclass
class AgentTaskState:
    changedFiles: list[str]
    worktreePath: str
    worktreeBranch: str
```

合并前先检查文件重叠：

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

如果有冲突，父 Agent 应该先比较，而不是盲目合并。

## 34.19 冲突仲裁 Agent

当多个子 Agent 输出互相冲突，可以启动一个专门的仲裁 Agent。

它的输入不是整个 transcript，而是结构化摘要：

```txt
你是冲突仲裁 Agent。

任务：
比较下面两个子 Agent 的方案，判断是否冲突、哪个方案更适合合并。

Agent A:
- 修改文件：...
- 方案摘要：...
- 测试结果：...

Agent B:
- 修改文件：...
- 方案摘要：...
- 测试结果：...

输出：
1. 是否冲突
2. 冲突文件
3. 推荐方案
4. 需要人工确认的点
```

仲裁 Agent 应该只读。它不应该直接改文件。它的任务是帮助父 Agent 决策。

## 34.20 编排模式一：串行委派

最简单的编排是串行：

```txt
Explore Agent 找文件
        |
        v
Review Agent 找问题
        |
        v
Fix Agent 修改
        |
        v
Test Agent 验证
```

优点：

- 容易控制。
- 成本可预测。
- 结果依赖清楚。

缺点：

- 慢。
- 前面 Agent 错了，后面全受影响。

串行适合新手实现，也适合高风险任务。

## 34.21 编排模式二：并行探索

并行探索适合信息收集：

```txt
Explore frontend
Explore backend
Explore tests
```

它们都只读，所以冲突风险低。

父 Agent 收集结果后合并：

```python
results = await Promise.all([
delegate('explore frontend')
delegate('explore backend')
delegate('explore tests')

merged = mergeExplorationResults(results)
```

并行探索是最适合先实现的多 Agent 模式，因为不涉及文件修改冲突。

## 34.22 编排模式三：并行方案试验

并行方案试验适合复杂修复：

```txt
Fix Agent A: 最小改动方案
Fix Agent B: 重构方案
Fix Agent C: 测试优先方案
```

每个 Agent 必须用 worktree isolation。完成后父 Agent 比较 diff 和测试结果。

这种模式成本高，但在复杂问题上很有价值。

## 34.23 编排模式四：监督者模式

监督者模式中，父 Agent 不亲自做细节工作，而是：

- 分解任务。
- 启动子 Agent。
- 监控进度。
- 处理冲突。
- 汇总结果。

它更像项目经理加技术负责人。

监督者提示词可以写：

```txt
你负责协调多个子 Agent。
不要启动没有明确产出的 Agent。
每个子任务必须有范围、工具限制和完成标准。
收到结果后要验证关键证据。
如果两个 Agent 结果冲突，先标记冲突并收集证据，不要盲目合并。
```

监督者模式强大，但也最容易失控。必须配合预算、最大并发数和最大委派深度。

## 34.24 最大并发数

不要允许父 Agent 一次启动无限子 Agent。

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

超过限制时：

```txt
Too many agents are already running. Wait for one to finish before launching another.
```

最大并发数保护的是：

- 成本。
- CPU。
- 文件系统。
- 用户注意力。
- 模型 API 限额。

## 34.25 任务 GC

任务完成后不能永远留在内存里。

可以设置：

```python
from dataclasses import dataclass
from typing import Any, Protocol

@dataclass
class AgentTaskState:
    evictAfter: int
    retain: bool
```

如果用户正在查看某个任务，就 `retain = true`，不要清理。否则完成后 30 秒清理：

```python
def maybeEvictTask(task: AgentTaskState):
    if (task.retain) return False
    if (not isTerminalStatus(task.status)) return False
    if (not task.notified) return False
    return time.time() > (task.evictAfter  or  0)
```

注意，清理内存状态不代表删除磁盘 transcript。内存任务可以 GC，审计记录仍然保留。

## 34.26 SDK 事件与 UI 事件分开

一个任务进度可能要发给：

- TUI。
- VS Code 插件。
- SDK 调用者。
- 日志系统。
- 父 Agent 消息队列。

不要把所有事件混在一起。比如给 SDK 的 task_progress 不一定要注入 LLM 上下文；给父 Agent 的 task_notification 才需要进入消息队列。

可以分成：

```python
emitSdkEvent(...)
enqueueAgentNotification(...)
updateAppState(...)
logEvent(...)
```

这样不同消费者可以各取所需。

## 34.27 编排测试

测试一：完成任务只转换一次。

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

测试二：取消触发 abort。

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

测试三：通知只发送一次。

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

测试四：进度不会覆盖 summary。

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

测试五：任务输出 offset 不覆盖完成状态。

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

这些测试看起来细，但它们覆盖的是多 Agent 系统最常见的竞态。

## 34.28 本章小结

这一章我们讲了多 Agent 的任务编排。

核心思想是：后台 Agent 必须被注册为 task。task 有状态、有进度、有输出、有取消控制、有通知和清理。不要把后台 Agent 当成一个无人管理的 Promise。

几个关键原则：

1. 先注册任务，再启动 Agent。
2. 状态转换必须原子化。
3. completed、failed、killed 是终态，不能被迟到回调覆盖。
4. 任务完成或失败时，先改变状态，再做 classifier、worktree cleanup、通知等附加工作。
5. 取消要提取部分结果。
6. 前台 Agent 可以在运行中转后台，但 background signal 要设计成一次性。
7. 通知要幂等，避免重复发送。
8. 输出要写磁盘，支持增量读取和恢复。
9. 多 Agent 冲突要先检测文件重叠，再考虑仲裁。
10. 编排必须有最大并发、预算和 GC。

下一章我们会进入 teammate、agent swarms 与团队式协作。那一章会讲从“一次性子 Agent”到“可寻址协作者”的升级：如何给 Agent 命名，如何发送消息，如何维护团队上下文，以及为什么团队式协作比普通子任务更难治理。
