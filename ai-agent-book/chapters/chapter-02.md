# 第 1 卷：入门与总览

## 第 2 章：从一次模型调用到 Agent 循环

### 2.1 本章目标

本章要把 Agent 的核心循环讲透。读完之后，你应该能自己写出一个最小 Agent runner，并能理解 Claude Code 源码中 `query()` 为什么是整个系统的心脏。

我们会先从普通模型调用开始，再加入 messages，再加入 tools，再加入 tool_result，最后形成完整循环。

### 2.2 一次普通模型调用

最简单的模型调用只有一个 prompt：

```python
answer = await model.generate("解释什么是闭包")
print(answer)
```

这类调用的问题是没有历史。用户下一句说“再举个例子”，模型不知道“再”指什么。于是我们引入消息历史：

```python
messages = [
{ role: "user", content: "解释什么是闭包" }
{ role: "assistant", content: "闭包是..." }
{ role: "user", content: "再举个例子" }
]
```

模型看到的是一段对话，而不是孤立问题。

### 2.3 System Prompt 的作用

除了用户消息，我们还需要 system prompt。它定义模型的身份、规则和工作方式。

例如：

```text
You are a careful coding assistant.
When you need information from the filesystem, use tools instead of guessing.
Do not modify files unless the user asks you to.
```

对 Agent 来说，system prompt 尤其重要，因为它要告诉模型：

1. 有哪些工具。
2. 什么时候该用工具。
3. 不要猜测外部世界。
4. 遇到权限限制时如何处理。
5. 输出格式和工作风格。

Claude Code 的 system prompt 并不是一个简单字符串，而是由多个部分拼接而成。源码中与 prompt 和 context 相关的逻辑分布在：

- `src/constants/prompts.js`
- `src/context.py`
- `src/mini_agent/utils/queryContext.js`
- `src/mini_agent/utils/systemPrompt.py`
- `src/query.py`

这说明大型 Agent 的 prompt 是动态生成的：不同模型、不同模式、不同权限、不同项目上下文，都会影响最终 prompt。

### 2.4 Tools：让模型拥有行动选项

工具是结构化的。模型不是输出“我想读 README”，而是输出一个工具调用块：

```json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "Read",
  "input": {
    "file_path": "/project/README.md"
  }
}
```

程序看到这个块后，执行本地工具：

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

然后把结果包装成：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "content": "README 文件内容..."
}
```

再发回模型。模型读到工具结果，才能继续生成最终回答。

### 2.5 工具调用为什么必须结构化

初学者很容易想：让模型输出一段 JSON，然后我自己解析不就行了吗？

不够。

工具调用协议要解决的不只是 JSON 格式，还有：

1. 工具名称是否合法。
2. 参数 schema 是否正确。
3. 工具调用和工具结果如何配对。
4. 多个工具调用如何排序。
5. 工具失败如何表达。
6. 工具结果如何进入下一轮上下文。
7. 模型是否知道哪些工具可用。

如果没有正式协议，Agent 很快会出现奇怪问题。例如模型输出：

```text
CALL_TOOL read_file path=README.md
```

你可以写正则解析。但下一次它可能输出：

```text
I will read README.md now.
```

再下一次它可能输出：

```json
{"tool": "ReadFile", "filename": "README.md"}
```

这就变成了脆弱的文本解析。结构化工具调用让模型和程序之间有明确合同。

### 2.6 最小循环的状态机

Agent 循环可以画成状态机：

```text
[等待用户输入]
      ↓
[调用模型]
      ↓
模型是否请求工具？
      ↓ 否
[输出最终回答] -> [等待用户输入]
      ↓ 是
[执行工具]
      ↓
[追加工具结果]
      ↓
[调用模型]
```

注意，执行工具之后不是直接输出给用户，而是回到调用模型。因为工具结果往往只是原材料，模型还要解释、整合、决定下一步。

比如用户说“修复测试”。工具结果可能是：

```text
FAIL src/parser.test.py
Expected "abc" but received "ab"
```

这个结果不是最终答案。模型要读它，判断失败原因，读取相关源文件，修改代码，再运行测试。

### 2.7 教学版 Agent Runner

下面是一个教学版 runner。它省略了真实 SDK 细节，但保留核心结构：

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

这段代码已经能表达 Agent 的核心。但是它还不适合真实项目，因为缺少：

- 参数校验。
- 权限检查。
- 并发控制。
- 超时取消。
- 上下文压缩。
- 流式响应。
- 工具结果大小限制。
- 日志和错误恢复。

Claude Code 的 `src/query.py` 就是在这个最小循环上不断加工程层。

### 2.8 Claude Code 中的对应位置

核心函数是：

- `src/query.py` 中的 `query()`
- `src/query.py` 中的 `queryLoop()`

`query()` 是外层生成器，负责启动查询并在正常结束时标记命令生命周期完成。真正复杂逻辑在 `queryLoop()`。

`queryLoop()` 内部每轮都会做这些事：

1. 准备消息。
2. 做 memory prefetch。
3. 做 skill discovery prefetch。
4. 做 tool result budget。
5. 做 snip/microcompact/context collapse/autocompact。
6. 生成完整 system prompt。
7. 选择当前模型。
8. 调用 streaming API。
9. 收集 assistant messages 和 tool_use blocks。
10. 执行工具。
11. 将结果加入消息。
12. 判断是否继续。

你可以把它理解成一个加强版：

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

### 2.9 新手常见误区

误区一：把工具结果直接展示给用户。

工具结果应该先回给模型。用户真正需要的是模型基于工具结果整理后的答案。当然 UI 可以同时展示工具执行过程，但对话状态里必须有 tool_result。

误区二：工具失败就终止整个 Agent。

很多工具失败是可恢复的。文件不存在，模型可以换路径；测试失败，模型可以继续修；权限拒绝，模型可以解释原因或请求用户。

误区三：不保存完整消息历史。

没有历史，模型无法知道自己刚才调用了哪个工具。至少在一轮 Agent 执行中，assistant 的 tool_use 和 user 的 tool_result 必须保留。

误区四：工具参数不校验。

模型输出不等于可信输入。必须校验类型、必填字段、路径范围、安全约束。

### 2.10 本章小结

本章我们从一次普通模型调用出发，构造了 Agent 循环。你现在应该理解：

1. Messages 保存对话状态。
2. System prompt 定义规则。
3. Tools 定义行动空间。
4. Tool use 和 tool result 必须配对。
5. Agent 循环的核心是“模型 -> 工具 -> 模型”。
6. Claude Code 的 `query.py` 是这个循环的工程级实现。

下一章我们进入源码地图，看看这套项目如何把入口、循环、工具、权限、MCP 和子 Agent 组织起来。

---
