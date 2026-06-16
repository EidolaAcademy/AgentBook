# 第 3 卷：工具系统工程化

## 第 20 章：并发工具调度，什么时候可以一起跑

### 20.1 本章目标

到现在为止，mini-agent 执行多个工具时是串行的：

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

这很简单，也很安全。但它不总是高效。

如果模型一次请求：

```text
read_file("a.py")
read_file("b.py")
grep_files("login")
```

这些都是只读工具，可以并发执行。

但如果模型请求：

```text
edit_file("a.py")
read_file("a.py")
```

就不能随便并发。读取必须在编辑之后，或者编辑必须在读取之后，顺序影响结果。

本章目标：

1. 理解为什么需要工具调度。
2. 区分并发安全和不安全工具。
3. 实现批处理调度。
4. 理解只读不一定等于并发安全。
5. 处理并发结果顺序。
6. 对照 Claude Code 的 `toolOrchestration.ts` 和 `StreamingToolExecutor`。

### 20.2 并发为什么有价值

Agent 任务中有很多等待：

- 读文件要等磁盘。
- 搜索要遍历文件。
- Web fetch 要等网络。
- MCP 工具要等外部服务。

如果三个只读工具可以同时执行，总耗时可能从：

```text
1s + 1s + 2s = 4s
```

变成：

```text
max(1s, 1s, 2s) = 2s
```

这对交互体验很重要。

Claude Code 里不仅支持并发工具执行，还支持“模型流式输出工具调用时提前执行工具”，进一步减少等待。

### 20.3 并发的风险

并发不是免费午餐。

风险一：写写冲突。

两个工具同时编辑同一个文件，后写的可能覆盖先写的。

风险二：读写顺序不确定。

一个工具读取文件，另一个工具同时编辑文件，读取结果可能是旧的，也可能是新的。

风险三：共享上下文修改。

有些工具执行后会修改 ToolContext，例如改变 cwd、更新缓存、注册后台任务。并发时修改顺序会变复杂。

风险四：输出顺序混乱。

模型生成 tool_use 的顺序是 A、B、C，但并发执行可能 B 先完成。消息历史里如何放结果？

风险五：错误传播。

并发工具中一个失败，其他工具是否继续？Shell 失败是否取消其他 shell？

所以并发必须有规则。

### 20.4 isReadOnly 和 isConcurrencySafe

我们 Tool 接口已有：

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

它们不是完全一样。

`isReadOnly` 表示工具不会修改外部状态。

`isConcurrencySafe` 表示工具可以和其他并发安全工具一起跑。

大多数只读工具可以并发：

- read_file
- glob_files
- grep_files

但有些只读工具也可能不适合并发：

- 读取并更新内部 LRU cache 的工具。
- 访问速率限制很严格的外部 API。
- 需要独占设备的工具。

所以接口保留两个方法，而不是用一个 `isReadOnly` 代替全部。

Claude Code 也是这样。很多工具的 `isConcurrencySafe` 会返回 `isReadOnly`，但接口上仍然分开。

### 20.5 分批策略

最简单安全策略：

```text
连续并发安全工具 -> 放进同一批并发执行
非并发安全工具 -> 单独一批串行执行
```

例如：

```text
Read A
Read B
Edit C
Read D
Grep E
Shell pytest
Read F
```

分批：

```text
Batch 1 concurrent: Read A, Read B
Batch 2 serial: Edit C
Batch 3 concurrent: Read D, Grep E
Batch 4 serial: Shell pytest
Batch 5 concurrent: Read F
```

为什么不是把所有 read 都提到前面并发？

因为工具顺序可能有语义。模型生成顺序是它的计划。调度器只能合并连续安全工具，不应该跨越写入工具重排。

Claude Code 的 `toolOrchestration.ts` 采用类似思想：把工具调用分成连续的并发安全批次和单个非安全批次。

### 20.6 实现 partitionToolUses

先定义：

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

实现：

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

注意：如果工具不存在或参数无法解析，就当成不并发安全。保守策略。

### 20.7 并发执行一批工具

并发批次：

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

`Promise.all` 返回结果顺序和输入数组顺序一致，即使实际完成顺序不同。

这很好，因为消息历史里可以按 tool_use 顺序追加 tool_result。

如果想实时显示完成顺序，可以额外发 progress event；但最终 messages 建议保持稳定顺序。

### 20.8 串行执行批次

串行批次：

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

实际上按我们的 partition，非并发安全批次通常只有一个工具。但写成通用形式更清楚。

### 20.9 runToolUses

整合：

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

AgentEngine 中替换：

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

### 20.10 并发和权限提示

并发工具如果都需要 ask，会发生什么？

例如模型同时请求：

```text
edit_file A
edit_file B
```

它们不是并发安全，会串行询问。

如果两个并发安全工具却需要 ask，这种情况比较少，因为并发安全通常是只读，默认 allow。

但 MCP 工具可能是并发安全却仍需要权限。此时并发弹多个 prompt 体验很差。

简化策略：

```text
需要 ask 的工具不要并发。
```

如何实现？

分批前很难知道权限，因为权限判断可能异步。生产系统会更复杂。

mini-agent 暂时不处理这个边界。只要我们把写入工具标记为非并发安全，就已经避免大部分并发 prompt。

### 20.11 并发错误处理

如果并发批次中一个工具失败，其他工具怎么办？

对于只读工具，通常可以继续。

例如：

```text
read_file A 成功
read_file B 文件不存在
grep_files C 成功
```

返回三个 tool_result，其中 B 是 error。

不要因为 B 失败就丢掉 A/C 的结果。

对于 shell 并发，策略可能不同。Claude Code 的 StreamingToolExecutor 中，如果并发 Bash 工具出错，可能会取消 sibling subprocess，避免无意义继续运行。

mini-agent 的并发批次暂时只包含只读工具，所以继续即可。

### 20.12 contextModifier 问题

Claude Code 的 ToolResult 支持 `contextModifier`。有些工具执行后会修改 ToolUseContext。

例如：

- 更新 readFile cache。
- 更新任务状态。
- 注册后台任务。

并发时 contextModifier 不能随便同时应用。Claude Code 会收集并按工具顺序应用，避免竞态。

mini-agent 目前没有 contextModifier。读者只需要知道：一旦工具能修改共享上下文，并发调度就必须更谨慎。

### 20.13 流式工具执行和普通并发的区别

本章讲的是普通并发：

```text
模型完整返回 assistant message
  ↓
拿到所有 tool_use
  ↓
分批执行
```

Claude Code 的 StreamingToolExecutor 更进一步：

```text
模型正在流式输出
  ↓
某个 tool_use input 完整
  ↓
立刻加入执行队列
  ↓
模型可能还在继续输出其他内容
```

这能减少延迟，但要处理更多复杂问题：

- partial tool input。
- fallback。
- 用户中断。
- 工具结果缓冲。
- 结果按原顺序 yield。
- 非并发工具阻塞后续工具。

mini-agent 可以先不做。普通并发已经能解决很多等待问题。

### 20.14 并发测试

测试 partition：

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

测试结果顺序：

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

这个测试很重要。并发执行时 UI 可以乱序显示进度，但最终 tool_result 顺序最好稳定。

### 20.15 本章练习

练习一：实现 `partitionToolUses()`。

要求：

- 连续并发安全工具合批。
- 非安全工具单独批次。
- 参数解析失败时当作非安全。

练习二：实现 `runToolUses()`。

要求：

- 并发批次用 Promise.all。
- 串行批次逐个 await。
- 返回结果顺序稳定。

练习三：改造 AgentEngine。

从逐个 runToolUse 改为 runToolUses。

练习四：测试并发。

创建两个 fake read 工具，一个延迟 100ms，一个延迟 10ms，确认结果顺序仍按 tool_use 顺序。

练习五：思考 StreamingToolExecutor。

为什么流式工具执行比普通并发难？列出至少三个原因。

### 20.16 本章小结

本章我们让 mini-agent 的工具执行更高效。

你应该理解：

1. 并发可以减少等待，但会引入顺序和状态风险。
2. 只读通常可以并发，但不等于永远安全。
3. 写入工具必须串行。
4. 分批调度比全局重排更安全。
5. 并发结果应该保持 tool_use 顺序。
6. Claude Code 的 StreamingToolExecutor 是普通并发的进一步优化。

下一章我们会深入工具结果预算：当工具输出非常大时，如何保存、预览、替换，避免上下文被撑爆。

