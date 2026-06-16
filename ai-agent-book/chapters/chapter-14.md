# 第 2 卷：最小 Agent 实现

## 第 14 章：上下文预算，避免 Agent 被历史淹没

### 14.1 本章目标

Agent 每一轮都会把新消息加入历史：

```text
user message
assistant message
tool_result
assistant message
tool_result
...
```

短任务没问题。长任务中，messages 会越来越长。工具结果尤其危险：一次 `read_file` 可能返回几百行，一次 `shell` 可能返回几千行，一次 `grep_files` 可能返回几百条命中。

最终会出现几个问题：

1. API 请求超过模型上下文限制。
2. 成本越来越高。
3. 延迟越来越大。
4. 模型被大量旧信息干扰。
5. 工具结果重复占用上下文。

本章目标：

1. 理解上下文窗口是什么。
2. 理解 token 预算为什么重要。
3. 实现粗略 token 估算。
4. 实现消息裁剪。
5. 实现工具结果截断和替换。
6. 理解自动压缩的基本思路。
7. 对照 Claude Code 的 compact、microcompact 和 tool result budget。

### 14.2 上下文窗口是什么

模型不是无限记忆。每次 API 调用都有上下文窗口限制。你发给模型的 system prompt、messages、tools、tool results，都要放进这个窗口。

如果超过限制，API 可能报错：

```text
prompt is too long
```

或者模型表现变差。

上下文窗口可以理解为模型本次“能看到的材料总量”。Agent 的挑战是：既要让模型看到足够信息，又不能把所有历史都无限塞进去。

### 14.3 哪些内容占上下文

Agent 请求里通常包含：

1. system prompt。
2. 工具 schema。
3. 用户消息。
4. assistant 消息。
5. tool_use。
6. tool_result。
7. 项目上下文。
8. 记忆。
9. 压缩摘要。

初学者常常只关注用户消息，忽略工具 schema 和工具结果。实际上工具结果可能是最大来源。

比如：

```text
read_file package-lock.json
```

如果工具返回整个 lockfile，可能一下子占掉大量上下文。

所以 FileReadTool、BashTool、GrepTool 都必须限制输出。

### 14.4 粗略 token 估算

精确 token 计算需要 tokenizer。教学项目先用粗略估算：

```python
from dataclasses import dataclass

@dataclass
class Message:
    role: str
    content: str

def estimate_tokens(text: str) -> int:
    return max(1, len(text) // 4)

def compact_messages(messages: list[Message], max_tokens: int, keep_recent: int = 6) -> list[Message]:
    total = sum(estimate_tokens(message.content) for message in messages)
    if total <= max_tokens:
        return messages
    old = messages[:-keep_recent]
    recent = messages[-keep_recent:]
    summary = "\n".join(f"- {message.role}: {message.content[:120]}" for message in old)
    return [Message(role="system", content=f"历史摘要:\n{summary}"), *recent]
```

这是非常粗略的英文估算。中文、代码、JSON 都会有偏差。但作为预算保护，比完全没有好。

估算消息：

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

真实项目应该使用模型对应 tokenizer 或 API token count。Claude Code 里有 token estimation 服务和上下文 warning 逻辑。

### 14.5 最简单的裁剪策略

第一版策略：

```text
如果消息超过预算，只保留最近 N 条。
```

代码：

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

这个策略简单，但有危险：

1. 可能删掉任务目标。
2. 可能删掉工具调用对应结果。
3. 可能删掉重要文件内容。
4. 可能让模型失去上下文。

所以只能作为临时保护。

### 14.6 不能拆散 tool_use 和 tool_result

裁剪消息时，最重要的是不要留下半截工具调用。

错误历史：

```text
assistant: tool_use read_file id=1
```

但对应的：

```text
user: tool_result id=1
```

被裁掉了。

这会导致下一次 API 请求不合法或模型困惑。

所以裁剪时应该从边界开始，确保保留区域内没有未配对 tool_use。

简化策略：

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

裁剪后如果有 unresolved，就继续往前包含更多消息，直到配对完整。

Claude Code 中相关逻辑更复杂，因为还要处理 thinking blocks、compact boundary、tombstone 等。

### 14.7 工具结果预算

比裁剪整条消息更好的办法是限制工具结果。

比如一个 shell 结果：

```text
STDOUT:
... 20000 行 ...
```

我们可以保留前一部分，并提示：

```text
Tool result was truncated. Use a more specific command.
```

我们之前已经在各工具中做了局部截断。但长会话中，即便每次工具结果都不大，累计起来也会很大。

可以在每次模型调用前统一处理：

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

这会损失信息，但能保护上下文。

Claude Code 的 `toolResultStorage` 更高级：过大的工具结果会保存到磁盘，模型看到预览和文件路径。如果需要完整内容，可以再用 Read 工具读取。

### 14.8 自动摘要压缩

更高级的做法是摘要旧消息。

流程：

```text
旧 messages
  ↓
调用小模型或当前模型总结
  ↓
生成 summary message
  ↓
保留 summary + 最近消息
```

示例 summary：

```text
Conversation summary:
- User asked to fix login tests.
- We found src/auth/login.py and tests/login.test.py.
- Test failed because expired tokens were treated as valid.
- We edited login.py to reject expired tokens.
- Need to rerun pytest.
```

然后消息变成：

```text
user: Conversation summary...
最近 10 条消息
```

这比简单裁剪好，因为保留了任务脉络。

但摘要也有风险：

1. 摘要可能遗漏细节。
2. 摘要可能写错。
3. 摘要会消耗额外 API 成本。
4. 摘要不能破坏工具配对。

Claude Code 的 autoCompact 就是这类机制，但更复杂。它会根据 token warning state 触发，生成 summary messages，并处理 compact boundary。

### 14.9 mini-agent 的第一版 compact

先实现一个接口：

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

FakeSummarizer：

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

真实 Summarizer 可以调用模型。

compact：

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

这个版本不完美，但能展示结构。

### 14.10 compact 应该在什么时候运行

应该在调用模型前运行：

```python
from dataclasses import dataclass

@dataclass
class Message:
    role: str
    content: str

def estimate_tokens(text: str) -> int:
    return max(1, len(text) // 4)

def compact_messages(messages: list[Message], max_tokens: int, keep_recent: int = 6) -> list[Message]:
    total = sum(estimate_tokens(message.content) for message in messages)
    if total <= max_tokens:
        return messages
    old = messages[:-keep_recent]
    recent = messages[-keep_recent:]
    summary = "\n".join(f"- {message.role}: {message.content[:120]}" for message in old)
    return [Message(role="system", content=f"历史摘要:\n{summary}"), *recent]
```

注意：内部完整 transcript 可以保留，发给模型的是 compact 后的 view。

Claude Code 也有类似区分：UI/会话可能保留更完整历史，但 API 请求前会投影、压缩、预算化。

### 14.11 上下文工程的原则

原则一：工具输出先限制。

不要等上下文爆了才处理。每个工具都应该控制输出。

原则二：历史请求前再预算。

每次 API 调用前检查当前 messages。

原则三：不要破坏协议。

tool_use/tool_result、thinking blocks、compact boundary 都要小心。

原则四：摘要不是事实源。

如果需要精确代码内容，应该重新读文件，而不是相信摘要。

原则五：告诉模型信息被截断或压缩。

模型要知道自己看到的不是完整历史。

### 14.12 和 Claude Code 对照

Claude Code 中上下文工程相关模块包括：

```text
src/query.ts
src/services/compact/autoCompact.ts
src/services/compact/microCompact.ts
src/services/compact/compact.ts
src/mini_agent/utils/toolResultStorage.py
src/services/tokenEstimation.ts
src/memdir/
src/services/SessionMemory/
```

它的层次大致是：

1. 工具自己限制输出。
2. 大工具结果可持久化到磁盘。
3. query 前应用 tool result budget。
4. microcompact 做局部压缩。
5. autoCompact 在接近上下文限制时总结历史。
6. memory 系统保存长期信息。

这是工程级 Agent 必须有的能力。没有上下文工程，Agent 越工作越笨，越工作越贵，最后直接失败。

### 14.13 本章练习

练习一：实现 token 粗略估算。

用字符数除以 4。

练习二：实现 `shrinkToolResults()`。

要求：

- tool_result 超过上限时截断。
- 加入提示。

练习三：实现 `compactMessagesIfNeeded()`。

要求：

- 超过预算时总结旧消息。
- 保留最近 N 条。

练习四：确保不拆散工具配对。

写测试构造 tool_use/tool_result，检查裁剪后仍配对。

练习五：观察上下文增长。

打印每轮 estimated tokens，看长任务中增长趋势。

### 14.14 本章小结

本章我们给 mini-agent 加入了上下文预算意识。

你应该理解：

1. Agent 历史会无限增长，必须控制。
2. 工具结果是上下文膨胀的主要来源。
3. 简单裁剪有风险，不能破坏工具配对。
4. 摘要压缩能保留任务脉络，但可能遗漏细节。
5. Claude Code 的 compact 系统是产品级上下文工程。

下一章我们会讲会话持久化：如何把 messages 保存下来，让 Agent 能恢复之前的工作。

---
