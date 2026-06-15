# 第 2 卷：最小 Agent 实现

## 第 16 章：测试 Agent，不要相信“看起来能跑”

### 16.1 本章目标

Agent 项目很容易出现一种错觉：你手动试了几个问题，模型回答不错，于是觉得系统没问题。

这很危险。

Agent 系统里有很多结构性约束：

- tool_use 和 tool_result 必须配对。
- 工具参数必须校验。
- 权限拒绝必须反馈给模型。
- plan 模式不能写文件。
- 输出截断必须提示模型。
- 会话恢复不能留下坏消息。
- maxTurns 必须阻止无限循环。

这些不应该靠手动测试。

本章目标：

1. 理解 Agent 测试和普通应用测试的区别。
2. 学会测试 Tool。
3. 学会测试 runToolUse。
4. 学会用 FakeModel 测试 AgentEngine。
5. 学会测试权限模式。
6. 学会测试上下文裁剪和会话恢复。
7. 对照 Claude Code 中依赖注入和测试友好设计。

### 16.2 Agent 测试的难点

普通函数测试通常是：

```text
输入 -> 输出
```

Agent 测试有几类特殊难点。

第一，模型不稳定。

真实模型同一个输入可能输出不同结果。不能让所有测试依赖真实模型。

第二，工具有副作用。

写文件、运行 shell、改目录都可能影响测试环境。

第三，消息历史复杂。

测试不只看最终回答，还要看中间是否调用了正确工具。

第四，权限有交互。

测试不能真的等人输入，需要 fake prompter。

第五，上下文裁剪可能破坏协议。

这类 bug 不一定立刻显现，但下一次 API 会失败。

所以测试策略要分层。

### 16.3 测试分层

建议分五层：

第一层：纯函数测试。

例如：

- `globToRegExp()`
- `estimateTokens()`
- `resolveInsideCwd()`
- `countOccurrences()`

第二层：工具测试。

例如：

- `read_file` 能读文件。
- `edit_file` oldString 不存在时报错。
- `shell` 超时能停止。

第三层：工具执行层测试。

例如：

- 未知工具返回错误 tool_result。
- 参数 schema 错误返回 InputValidationError。
- 权限 deny 返回错误 tool_result。

第四层：AgentEngine 测试。

用 FakeModel 模拟 tool_use，确认 Engine 会执行工具并追加结果。

第五层：持久化和恢复测试。

确认 transcript 能保存、加载，并检测坏消息。

### 16.4 测试工具：临时目录

文件工具测试不要在真实项目目录乱写。用临时目录。

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

测试结束可以删除。注意删除命令要谨慎，确保路径是测试创建的临时目录。

### 16.5 read_file 测试

示例：

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

再测路径逃逸：

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

这类测试非常重要。路径边界不应该靠人工记忆。

### 16.6 edit_file 测试

测试 oldString 不存在：

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

测试 oldString 多次出现：

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

这能防止误改多处。

### 16.7 runToolUse 测试

runToolUse 是协议关键层。

测试未知工具：

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

测试 schema 错误：

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

这些测试保证模型生成坏参数时，Agent 不会崩。

### 16.8 FakePrompter

权限测试需要 fake prompter：

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

测试用户允许：

```python
permissionPrompter = FakePermissionPrompter(True)
```

测试用户拒绝：

```python
permissionPrompter = FakePermissionPrompter(False)
```

这样测试不会卡住等待 stdin。

### 16.9 AgentEngine 测试

用 FakeModel 模拟：

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

测试：

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

这个测试不需要真实模型，却能验证 Agent 循环。

### 16.10 maxTurns 测试

模型永远返回工具调用：

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

测试 Engine 会停：

```python
engine = AgentEngine(:
    "model": InfiniteToolModel()
    "tools": registry
    cwd
    "maxTurns": 3

final = await engine.submit("loop")
expect(renderMessage(final)).toContain("maxTurns")
```

这能防止无限循环回归。

### 16.11 上下文裁剪测试

构造一组消息：

```python
messages = [
userText("start")

    "role": "assistant"
    "content": [
    { type: "tool_use", id: "1", name: "read_file", input: { path: "a" } }
toolResult("1", "content")
]
```

测试裁剪后不留下 unresolved tool_use。

这类测试很容易被忽视，但非常关键。上下文裁剪 bug 往往不是当场报错，而是在下一次 API 请求时爆炸。

### 16.12 Transcript 测试

测试保存和恢复：

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

再测试坏行：

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

### 16.13 不要把真实模型作为单元测试依赖

真实模型测试可以有，但不要放在普通单元测试里。

原因：

1. 慢。
2. 花钱。
3. 不稳定。
4. 需要 API key。
5. 输出不完全可预测。

可以把真实模型测试放到单独命令：

```bash
pytest tests/integration
```

并且默认不在 CI 跑。

Claude Code 这类项目也会通过依赖注入、VCR、fake deps 等方式避免所有测试都打真实 API。

### 16.14 本章练习

练习一：为纯函数写测试。

至少测试：

- `globToRegExp`
- `resolveInsideCwd`
- `countOccurrences`

练习二：为 read/edit/write 工具写测试。

覆盖成功和失败。

练习三：为 runToolUse 写测试。

覆盖：

- unknown tool
- schema error
- permission denied
- tool success

练习四：为 AgentEngine 写 FakeModel 测试。

覆盖多轮工具调用和 maxTurns。

练习五：为 transcript 写测试。

覆盖保存、恢复、坏行跳过。

### 16.15 本章小结

本章我们建立了 mini-agent 的测试意识。

你应该理解：

1. Agent 测试不能依赖“手动试一试”。
2. FakeModel 是测试控制流的关键。
3. FakePrompter 是测试权限的关键。
4. 工具和协议层必须有单元测试。
5. 真实模型测试应该单独放在集成测试里。

下一章我们会回到 Claude Code 源码，把 mini-agent 已经实现的模块和 Claude Code 的对应模块做一次系统对照。

---
