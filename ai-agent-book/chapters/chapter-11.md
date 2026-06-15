# 第 2 卷：最小 Agent 实现

## 第 11 章：把工具串成完整任务闭环

### 11.1 本章目标

前面几章我们逐个实现了工具：

```text
read_file
glob_files
grep_files
shell
write_file
edit_file
```

如果只看单个工具，它们都不复杂。Agent 真正的价值在于把工具串起来，完成一个目标。

例如：

```text
用户：把 README 里的端口 3000 改成 4000
  ↓
Agent 搜索 README
  ↓
Agent 读取 README
  ↓
Agent 编辑 README
  ↓
Agent 报告完成
```

再复杂一点：

```text
用户：修复 login 测试
  ↓
Agent 搜索 login 测试
  ↓
Agent 读取测试和源码
  ↓
Agent 编辑源码
  ↓
Agent 运行测试
  ↓
Agent 根据结果继续修或报告完成
```

本章目标：

1. 理解完整 Agent 任务闭环。
2. 设计多轮工具调用的消息历史。
3. 改进 `AgentEngine`，支持多轮工具调用和 maxTurns。
4. 理解工具结果如何影响下一步。
5. 用 FakeModel 模拟一个完整任务。
6. 对照 Claude Code 的 `query.py` 主循环。

### 11.2 完整闭环是什么

完整闭环不是“调用一个工具”，而是：

```text
目标
  ↓
观察
  ↓
判断
  ↓
行动
  ↓
验证
  ↓
继续或结束
```

对 coding Agent 来说：

观察可能是：

- 搜索文件。
- 读取文件。
- 查看 git diff。
- 运行测试。

行动可能是：

- 写文件。
- 编辑文件。
- 运行格式化命令。

验证可能是：

- 再次读取文件。
- 运行测试。
- 运行 lint。
- 查看 diff。

Agent 的主循环必须允许模型在每一步根据结果决定下一步。

### 11.3 消息历史示例

以“修改 README 端口”为例，消息历史可能是：

```text
user:
  把 README 里的端口 3000 改成 4000

assistant:
  我先查找 README 文件。
  tool_use glob_files {"pattern":"**/README*"}

user:
  tool_result glob_files:
  README.md

assistant:
  我读取 README.md。
  tool_use read_file {"path":"README.md"}

user:
  tool_result read_file:
  ... port 3000 ...

assistant:
  我将替换端口。
  tool_use edit_file {"path":"README.md","oldString":"port 3000","newString":"port 4000"}

user:
  tool_result edit_file:
  File edited.

assistant:
  已完成修改。
```

这里每个工具结果都会成为下一次模型调用的输入。模型不是靠隐藏状态记住工具返回，而是靠 messages 看到结果。

### 11.4 AgentEngine 当前的问题

我们之前的 Engine 已经有 while 循环，但还比较粗糙：

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

它还缺几个东西：

1. maxTurns。
2. 对连续工具调用的日志。
3. 对权限 prompter 的传递。
4. 对最终消息的渲染。
5. 对工具结果顺序的清晰保证。

先加 maxTurns 和 prompter。

### 11.5 改进 AgentEngine 配置

定义：

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

Engine：

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

这里有一个设计选择：达到 maxTurns 后返回 assistantText，而不是 throw。

为什么？

因为 maxTurns 是用户任务层面的停止，不一定是程序 bug。返回一条消息给用户更友好。

生产系统可能会更细：

- SDK 返回 structured status。
- UI 显示“已达到最大轮数”。
- transcript 记录停止原因。

Claude Code 的 `query.py` 会返回 terminal reason，并且有更多预算控制。

### 11.6 工具调用顺序

当前 Engine 对多个工具调用串行执行：

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

这保证了顺序简单、上下文清晰。

如果模型一次请求：

```text
read_file A
read_file B
```

串行执行没问题，只是慢一点。

如果模型一次请求：

```text
edit_file A
read_file A
```

串行顺序就很重要。先编辑再读取，和先读取再编辑结果不同。

所以新手版先全部串行是合理的。后面讲并发时，再根据 `isConcurrencySafe` 分批。

Claude Code 的 `toolOrchestration.py` 就是做这个事情：连续并发安全工具可以并行，不安全工具串行。

### 11.7 FakeModel 模拟完整任务

真实模型会自己根据工具结果决定下一步。FakeModel 没有智能，所以我们写一个小状态机模拟。

目标：用户输入：

```text
把 README 里的 3000 改成 4000
```

FakeModel 流程：

1. 如果还没有 glob 结果，调用 `glob_files`。
2. 如果有 glob 结果但还没 read，调用 `read_file`。
3. 如果有 read 结果但还没 edit，调用 `edit_file`。
4. 如果 edit 成功，输出完成。

为了实现这个，我们需要根据历史判断已经做过什么。

示例辅助函数：

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

然后：

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

这个 FakeModel 很笨，但它模拟了多轮工具调用。

### 11.8 FakeModel 的局限

你可能已经发现一个问题：上面的 FakeModel 硬编码了 `README.md`。如果 glob 结果是 `docs/README.md`，它仍然会读 `README.md`。

真实模型会从 tool_result 里读取路径，再决定下一步。FakeModel 要做到这一点也可以，但会越来越像一个手写规则系统。

这提醒我们：FakeModel 的目的不是替代真实模型，而是测试 AgentEngine 的控制流。

它适合测试：

- tool_use 是否被执行。
- tool_result 是否追加。
- maxTurns 是否生效。
- 权限询问是否触发。

不适合测试：

- 模型是否能智能选择文件。
- 模型是否能正确修 bug。
- 模型是否能理解复杂错误。

### 11.9 真实模型接入时要注意什么

当你接真实模型时，最重要的是把工具 schema 传给模型。

模型需要知道：

- 工具名。
- 工具描述。
- 参数 JSON Schema。

否则它不知道该怎么调用。

教学项目目前的 Tool 使用 Zod schema。你可以：

1. 手写每个工具的 JSON Schema。
2. 使用 pydantic-to-json-schema 之类工具转换。
3. 为 Tool 增加 `inputJsonSchema` 字段。

Claude Code 的 Tool 支持 `inputSchema` 和 `inputJSONSchema`，MCP 工具可以直接提供 JSON Schema。

### 11.10 工具描述对模型行为的影响

工具描述不是文档装饰，而是模型决策的一部分。

不好的描述：

```text
Run stuff.
```

好的描述：

```text
Read a text file from the current workspace. Use this when you know the file path and need its contents.
```

再比如 `grep_files`：

```text
Search for a plain text pattern in workspace files. Returns file path, line number, and matching line. Use this before read_file when you know a symbol or string but not the file path.
```

模型会根据描述决定何时使用工具。

Claude Code 的工具都有 prompt/description，并且有些工具还有 searchHint，用于工具搜索。

### 11.11 完整任务中的权限体验

当完整任务进入写入阶段时，用户会看到：

```text
Allow editing file: README.md [y/N]
```

这时候用户输入 `y`，Agent 才执行 edit。

这就是人与 Agent 协作的关键时刻。Agent 不应该悄悄改文件；用户也不应该被每个只读操作打断。好的权限体验是：

```text
观察类操作自动执行
修改类操作请求确认
危险操作明确警告
```

Claude Code 在这方面做了大量 UI 工作，包括 diff 展示、权限弹窗、计划模式等。

### 11.12 本章练习

练习一：给 AgentEngine 加 maxTurns。

要求：

- 默认 10。
- 达到上限后返回 assistant 消息。
- 不要无限循环。

练习二：写一个 FakeModel 多步任务。

模拟：

```text
glob_files -> read_file -> edit_file -> final answer
```

练习三：测试权限。

在 default 模式下，确认 edit_file 会询问用户。

练习四：测试 plan 模式。

在 plan 模式下，同样任务应该在 edit_file 时被拒绝。

练习五：测试 maxTurns。

让 FakeModel 永远返回同一个 tool_use，确认 Engine 会停止。

### 11.13 本章小结

本章我们把工具串成了完整任务闭环。

你现在应该理解：

1. Agent 的价值来自多轮工具调用，而不是单个工具。
2. 消息历史是模型理解前一步工具结果的唯一依据。
3. maxTurns 是必要的安全预算。
4. FakeModel 适合测试控制流，不适合测试智能。
5. 工具描述会影响真实模型如何选择工具。

下一章我们会开始接入真实模型 API，让 mini-agent 从 FakeModel 走向真正的 LLM Agent。

---
