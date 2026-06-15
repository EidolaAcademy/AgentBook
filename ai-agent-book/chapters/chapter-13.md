# 第 2 卷：最小 Agent 实现

## 第 13 章：流式响应，让 Agent 更像实时助手

### 13.1 本章目标

上一章我们实现了非流式模型调用。非流式的体验是：

```text
用户输入
  ↓
等待模型完整生成
  ↓
一次性显示结果
```

如果模型回答很短，这没问题。但 coding Agent 经常会：

- 思考较长时间。
- 输出较长解释。
- 生成工具调用。
- 等待工具执行。
- 再继续生成。

用户如果一直看不到反馈，会以为程序卡住了。

流式响应可以让模型边生成边显示：

```text
用户输入
  ↓
模型开始输出 token
  ↓
CLI 实时显示
  ↓
如果出现 tool_use，执行工具
  ↓
继续下一轮
```

本章目标：

1. 理解流式响应的价值。
2. 实现文本流式显示。
3. 理解工具调用流式输出为什么复杂。
4. 设计 mini-agent 的简化流式接口。
5. 对照 Claude Code 的 streaming 实现。

### 13.2 非流式的问题

非流式最大的问题是“沉默”。

例如用户说：

```text
分析这个项目的架构。
```

模型可能需要几秒甚至几十秒。如果 CLI 没有任何输出，用户不知道：

- 模型是否收到请求。
- 网络是否卡住。
- 程序是否崩溃。
- Agent 是否正在调用工具。

流式输出可以缓解这个问题。即使只显示：

```text
我先查看项目结构...
```

用户也会感觉系统活着。

Claude Code 作为终端工具，非常重视流式体验。它不仅流式显示文本，还流式显示工具进度、Bash 输出、权限状态、spinner、后台任务等。

### 13.3 最简单的流式接口

先定义模型流事件：

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

ModelClient 增加：

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

为什么用 AsyncGenerator？

因为模型输出是一串异步事件：

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

这和 Claude Code 的 `query()` 很像。`query()` 本身就是 async generator，会不断 yield stream event、message、tool result summary 等。

### 13.4 CLI 显示文本 delta

AgentEngine 可以先支持纯文本流：

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

这已经能让用户看到模型实时输出。

但是工具调用还没处理。我们需要在 message_complete 后提取 tool_use，再执行工具。也就是说第一版流式仍然是：

```text
流式显示文本
  ↓
等完整 assistant message
  ↓
执行工具
```

这比非流式体验好，但还不是最高性能。

### 13.5 为什么工具调用流式更复杂

模型生成 tool_use 时，input JSON 可能不是一次到达。

流式事件可能类似：

```text
content_block_start: tool_use name=read_file id=...
input_json_delta: {"pa
input_json_delta: th":"READ
input_json_delta: ME.md"}
content_block_stop
```

客户端要把这些 delta 拼起来，等 JSON 完整后才能执行工具。

更复杂的是，有些系统会在工具 input 完整时就提前执行工具，而不是等整个 assistant 消息结束。Claude Code 的 `StreamingToolExecutor` 就是做这个优化。

它要处理：

- 工具块流式到达。
- 参数是否完整。
- 并发安全。
- 非并发工具排队。
- 工具进度立即 yield。
- streaming fallback 时丢弃旧结果。
- 用户中断时取消工具。

所以教学项目先不做“流式工具提前执行”。我们先做“文本流式 + message complete 后执行工具”。

### 13.6 实现 stream 后的 AgentEngine

可以给 Engine 增加一个方法：

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

`onTextDelta` 可以由 CLI 传入：

```python
def onTextDelta(self, text):
    process.stdout.write(text)
```

### 13.7 流式事件和最终消息都要保留

一个常见错误是：流式打印了文本，但没有保存完整 assistant message。

这会导致下一轮模型看不到上一轮回答，也看不到 tool_use。

正确做法：

```text
delta 用于 UI
完整 message 用于历史
```

Claude Code 也区分 stream event 和 message。UI 可以实时显示 partial，但 transcript 和下一轮 API 需要规范化后的完整消息。

### 13.8 工具进度也应该流式

模型文本可以流式，工具也应该流式。

例如 shell 命令：

```bash
pytest
```

可能持续输出：

```text
running test A
running test B
failed test C
```

如果工具要等全部结束才显示，用户体验仍然不好。

我们的 `execCommand()` 现在内部监听 stdout/stderr，但没有向外报告进度。可以扩展：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

然后：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

Tool 接口也可以支持 progress callback。

Claude Code 的 Tool `call()` 支持 `onProgress`，工具执行层会把 progress message yield 给 UI。BashTool、AgentTool 等都会利用这个能力。

### 13.9 本章先不实现的高级能力

为了保持 mini-agent 可理解，本章不实现：

1. partial tool input JSON 拼接。
2. 工具提前执行。
3. thinking block 流式处理。
4. streaming fallback。
5. tombstone partial message。
6. 多工具流式并发。

这些都是 Claude Code 级别的能力。我们会在后面工程化章节讲原理。

### 13.10 和 Claude Code streaming 对照

Claude Code 的流式链路大致是：

```text
query.py
  ↓
services/api/claude.py queryModelWithStreaming
  ↓
Anthropic streaming API
  ↓
stream events
  ↓
assistant messages / tool_use blocks
  ↓
StreamingToolExecutor
  ↓
tool progress / tool_result
```

其中 `StreamingToolExecutor` 是关键优化。它让工具在模型输出期间就可以开始执行，从而减少总耗时。

举例：

```text
模型生成 tool_use read_file README.md
  ↓
客户端发现 tool_use 完整
  ↓
立刻读取文件
  ↓
模型可能还在输出后续文本
```

这对快速工具很有价值。但复杂度也很高，所以 mini-agent 先不做。

### 13.11 本章练习

练习一：定义 `ModelStreamEvent`。

支持：

- text_delta
- message_complete

练习二：给 ModelClient 增加 `stream()`。

FakeModel 可以模拟流式：把字符串拆成多个字符或词逐个 yield。

练习三：实现 `submitStreaming()`。

要求：

- delta 实时输出。
- complete message 保存进历史。
- message_complete 后执行工具。

练习四：给 shell exec 加 `onOutput`。

先只在 CLI 打印输出，不必进入消息历史。

练习五：比较非流式和流式体验。

运行同一个长回答，看用户等待感有什么不同。

### 13.12 本章小结

本章我们让 mini-agent 具备了实时响应的雏形。

你应该理解：

1. streaming 主要改善用户体验和延迟。
2. delta 用于 UI，完整 message 用于历史。
3. 工具调用流式比文本流式复杂得多。
4. 生产级 Agent 会流式显示模型输出和工具进度。
5. Claude Code 的 StreamingToolExecutor 是性能优化层，不是 Agent 最小必需层。

下一章我们会处理一个越来越明显的问题：上下文会不断增长。Agent 工作越久，messages 越长，最终一定需要压缩和预算。

---
