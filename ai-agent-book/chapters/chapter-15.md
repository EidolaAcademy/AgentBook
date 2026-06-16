# 第 2 卷：最小 Agent 实现

## 第 15 章：会话持久化与恢复

### 15.1 本章目标

到目前为止，mini-agent 的消息历史都在内存里。程序一退出，历史就消失。

这对玩具项目没问题，但对 coding Agent 不够。真实任务可能持续很久，用户可能中途关闭终端，或者希望回看 Agent 做过什么。我们需要把会话保存下来。

本章目标：

1. 理解 transcript 的作用。
2. 设计会话文件格式。
3. 实现消息保存。
4. 实现会话恢复。
5. 理解持久化和上下文压缩的关系。
6. 对照 Claude Code 的 session storage。

### 15.2 为什么需要 transcript

Transcript 可以理解为“会话记录”。

它记录：

- 用户说了什么。
- 模型回答了什么。
- 模型调用了哪些工具。
- 工具返回了什么。
- 哪些权限被允许或拒绝。
- 哪些文件被修改。

它有几个用途。

第一，恢复会话。

用户可以第二天继续昨天的任务。

第二，调试问题。

如果 Agent 做错了事，可以回看它为什么做出那个决定。

第三，审计安全。

写文件、运行命令、权限决定都应该可追溯。

第四，生成总结。

长任务可以从 transcript 中提取摘要。

Claude Code 会把会话保存在项目相关目录中，并支持 `/resume`、session recovery、agent transcript 等能力。

### 15.3 JSONL 格式

会话文件可以用 JSONL：一行一个 JSON 对象。

示例：

```jsonl
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"总结 README"}]}}
{"type":"message","message":{"role":"assistant","content":[{"type":"tool_use","id":"1","name":"read_file","input":{"path":"README.md"}}]}}
{"type":"message","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"1","content":"..."}]}}
```

为什么用 JSONL？

1. 追加方便。
2. 文件损坏时可以恢复前面完整行。
3. 大会话不用每次重写整个 JSON 数组。
4. 很适合日志。

也可以用 JSON 数组，但每次追加都要读写整个文件，不适合长会话。

### 15.4 定义 TranscriptEvent

创建 `src/session/transcript.py`：

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

先记录 message 和 permission。后面可以加入：

- tool_progress
- compact_boundary
- error
- cost
- model_request

Claude Code 的消息类型更丰富，也会记录 API usage、tool result replacement、agent metadata 等。

### 15.5 保存消息

创建 `TranscriptStore`：

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

AgentEngine 每次 push message 时，也 append：

```python
self.messages.append(message)
await self.options.transcript.appendMessage(message)
```

为了避免到处手写，可以封装：

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

然后所有地方都用 `addMessage()`。

### 15.6 恢复消息

读取 JSONL：

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

这里遇到坏行直接跳过。JSONL 的好处是，一行坏了不影响整个文件。

启动时：

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

Engine 构造：

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

### 15.7 会话路径设计

可以把会话存在：

```text
.mini-agent/sessions/<timestamp>.jsonl
```

例如：

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

也可以支持：

```bash
mini-agent --resume .mini-agent/sessions/xxx.jsonl
```

教学项目先手写路径即可。

真实产品要考虑：

- 多项目隔离。
- session id。
- 自定义标题。
- 最近会话列表。
- 删除和清理。
- 隐私和权限。

Claude Code 有项目路径映射、session id、resume conversation、agent sidechain transcript 等复杂机制。

### 15.8 持久化与 compact 的关系

一个重要设计：保存到 transcript 的应该是完整事件，发给模型的可以是压缩视图。

也就是说：

```text
transcript: 尽量完整，方便审计和恢复
model context: 预算化、压缩、裁剪
```

不要因为上下文压缩就把原始 transcript 删掉。否则用户无法回看完整过程。

Claude Code 也区分 UI/history/transcript 和 API context projection。压缩是为了发给模型，不等于历史消失。

### 15.9 恢复后的注意事项

恢复会话后，工具配对仍然要合法。

如果 transcript 最后停在：

```text
assistant: tool_use ...
```

但没有 tool_result，说明程序可能在工具执行中崩溃。

恢复时可以：

1. 自动补一个错误 tool_result。
2. 丢弃最后未完成 assistant message。
3. 提示用户会话不完整。

教学项目可以先检测：

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

Claude Code 有更完整的 conversation recovery 逻辑，会处理不完整消息、工具结果、compact boundary 等。

### 15.10 权限记录

权限决定也应该记录。

当用户允许：

```text
Allow shell command: pytest
```

记录：

```json
{
  "type": "permission",
  "toolName": "shell",
  "decision": "allow",
  "reason": "Allow shell command: pytest",
  "timestamp": "..."
}
```

这对审计很有用。

本书当前 mini-agent 可以先只记录 message。后面安全章节再完善权限日志。

### 15.11 本章练习

练习一：实现 TranscriptStore。

要求：

- append JSONL。
- 自动创建目录。

练习二：Engine 使用 `addMessage()`。

要求所有消息写入内存时也写 transcript。

练习三：实现 `loadMessages()`。

要求：

- 读取 JSONL。
- 跳过坏行。
- 只恢复 message event。

练习四：实现 `--resume`。

从指定 session 文件恢复。

练习五：检测未完成工具调用。

恢复后如果存在 unresolved tool_use，打印 warning。

### 15.12 本章小结

本章我们让 mini-agent 有了记忆文件。

你应该理解：

1. Transcript 是 Agent 的审计日志。
2. JSONL 适合长会话追加。
3. 恢复会话要检查消息合法性。
4. Transcript 应该尽量完整，模型上下文可以压缩。
5. Claude Code 的 session storage 是同一思想的产品级版本。

下一章我们会讲测试。Agent 系统如果没有测试，很容易在工具协议、权限、上下文裁剪中悄悄出错。

---
