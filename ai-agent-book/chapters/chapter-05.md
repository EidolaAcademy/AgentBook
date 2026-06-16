# 第 1 卷：入门与总览

## 第 5 章：消息协议与工具调用

### 5.1 本章目标

本章是从聊天机器人进入 Agent 的分水岭。上一章我们写了一个只会把用户输入发给模型的小 CLI。本章要给它引入工具调用协议。

读完本章，你应该能理解：

1. 为什么普通字符串消息不够用。
2. 什么是 content block。
3. 什么是 `tool_use`。
4. 什么是 `tool_result`。
5. 为什么工具调用和工具结果必须配对。
6. 工具失败时应该如何告诉模型。
7. mini-agent 的消息类型应该如何升级。
8. Claude Code 源码中哪些地方处理这些协议细节。

这章非常关键。很多 Agent 项目失败，不是因为模型不够强，而是因为消息协议处理错了。工具结果放错角色、工具 ID 对不上、错误直接 throw 出去、历史中留下半截工具调用，这些问题都会让 Agent 在多轮任务中变得不稳定。

### 5.2 为什么字符串消息不够用

上一章的消息类型是：

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class ChatMessage:
    role: str
    content: str


messages: list[ChatMessage] = [
    ChatMessage(role="system", content="你是一个代码助手。"),
    ChatMessage(role="user", content="解释什么是闭包。"),
]
```

这对普通聊天足够，但对 Agent 不够。因为 Agent 消息里不只有文本，还会有结构化动作。

模型可能输出：

```text
我需要读取 README.md。
```

这只是文本。程序无法可靠知道它是否真的要执行读取，也不知道路径字段在哪里。

Agent 需要模型输出结构化块：

```json
{
  "type": "tool_use",
  "id": "toolu_001",
  "name": "read_file",
  "input": {
    "path": "README.md"
  }
}
```

程序看到 `type: "tool_use"`，就知道这不是普通回答，而是一个需要执行的动作。

执行完成后，程序再追加：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "content": "README 文件内容..."
}
```

模型看到这个结果后，再继续生成总结。

这就是为什么 content 不能只是字符串。它需要支持多种 block。

### 5.3 Content Block 的概念

Content block 可以理解为“消息内容里的积木”。一条消息不再只有一段文本，而是由多个块组成。

例如 assistant 消息可以是：

```json
[
  {
    "type": "text",
    "text": "我先读取 README。"
  },
  {
    "type": "tool_use",
    "id": "toolu_001",
    "name": "read_file",
    "input": {
      "path": "README.md"
    }
  }
]
```

user 消息也可以是：

```json
[
  {
    "type": "tool_result",
    "tool_use_id": "toolu_001",
    "content": "# 项目说明..."
  }
]
```

注意这里的 `user` 不一定代表真人输入。工具结果在 Messages API 里通常作为 user role 的内容返回给模型。这一点初学者很容易困惑：明明工具是程序执行的，为什么 role 是 user？

可以这样理解：assistant 发起工具请求，外部环境把观察结果“交回”给 assistant。这个外部环境在对话协议里站在 user 侧，所以 tool_result 放在 user 消息中。

### 5.4 tool_use 的字段

一个工具调用块通常包含：

```python
from typing import Any

ContentBlock = dict[str, Any]


def text_block(text: str) -> ContentBlock:
    return {"type": "text", "text": text}


def tool_use_block(tool_use_id: str, name: str, tool_input: dict[str, Any]) -> ContentBlock:
    return {
        "type": "tool_use",
        "id": tool_use_id,
        "name": name,
        "input": tool_input,
    }


def tool_result_block(tool_use_id: str, content: str, is_error: bool = False) -> ContentBlock:
    return {
        "type": "tool_result",
        "tool_use_id": tool_use_id,
        "content": content,
        "is_error": is_error,
    }
```

字段含义：

`type` 表示这是工具调用。

`id` 是本次工具调用的唯一编号。后面的 tool_result 必须用它来配对。

`name` 是工具名，例如 `read_file`、`bash`、`edit_file`。

`input` 是工具参数。它必须符合工具 schema。

最重要的是 `id`。没有 ID，就无法在多个工具调用中知道哪个结果对应哪个请求。

例如模型同时请求两个文件：

```json
[
  {
    "type": "tool_use",
    "id": "toolu_read_a",
    "name": "read_file",
    "input": { "path": "a.py" }
  },
  {
    "type": "tool_use",
    "id": "toolu_read_b",
    "name": "read_file",
    "input": { "path": "b.py" }
  }
]
```

程序执行后必须返回：

```json
[
  {
    "type": "tool_result",
    "tool_use_id": "toolu_read_a",
    "content": "a.py 内容"
  },
  {
    "type": "tool_result",
    "tool_use_id": "toolu_read_b",
    "content": "b.py 内容"
  }
]
```

如果 ID 对错了，模型会把文件内容理解反。写代码时这类错误很隐蔽，必须通过类型和测试防住。

### 5.5 tool_result 的字段

工具结果块可以定义为：

```python
from dataclasses import dataclass

@dataclass
class ToolResultBlock:
    tool_use_id: str
    content: str
    is_error: bool = False
    type: str = "tool_result"


result = ToolResultBlock(
    tool_use_id="toolu_001",
    content="README 文件内容...",
)
```

字段含义：

`type` 表示这是工具结果。

`tool_use_id` 对应前面的 `tool_use.id`。

`content` 是工具返回给模型的内容。

`is_error` 表示工具是否失败。

工具失败时，不应该总是直接终止程序。应该把错误作为工具结果交回模型。例如：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_001",
  "is_error": true,
  "content": "Error: file README.md not found"
}
```

模型看到这个错误后，可能会搜索文件、换路径、询问用户，或者解释无法完成。

### 5.6 为什么错误也要进入消息历史

新手常犯的错误是：

```python
from typing import Any


async def run_tool_and_crash(tool: Any, tool_input: dict[str, Any], context: Any) -> str:
    # 这是错误示范：工具抛出的异常会直接打断整个 Agent loop。
    return await tool.call(tool_input, context)
```

这样做会让整个 Agent 循环中断。对系统错误来说可以，但对工具执行错误来说通常不理想。

比如文件不存在：

```text
Error: ENOENT README.md
```

这不是 Agent 必须崩溃的理由。模型可以运行搜索工具找真实文件名，也可以告诉用户项目没有 README。

比如测试失败：

```text
Exit code 1
Expected 1 but received 2
```

这更不是崩溃理由。用户本来就是让 Agent 修测试，测试失败是有价值的观察。

所以我们要区分两类错误：

第一类，系统级错误。比如模型 API 认证失败、程序代码 bug、消息格式非法。这类可能需要终止。

第二类，任务级错误。比如文件不存在、命令退出码非零、权限拒绝。这类应该作为 `tool_result` 返回给模型。

Claude Code 的 `src/services/tools/toolExecution.ts` 就大量处理这种情况：工具不存在、schema 错误、validateInput 失败、权限拒绝、工具调用异常，都会被包装成模型可理解的工具结果。

### 5.7 工具调用配对规则

配对规则可以总结为：

```text
每个 assistant 的 tool_use，都必须有后续 user 的 tool_result。
```

更具体一点：

1. `tool_result.tool_use_id` 必须等于某个 `tool_use.id`。
2. 一个 `tool_use.id` 不应该对应多个成功结果。
3. 如果工具被取消，也要返回取消结果。
4. 如果工具失败，也要返回错误结果。
5. 不要在历史里留下没有结果的工具调用。

为什么这么严格？因为下一次 API 调用时，模型服务会验证消息结构。如果有 assistant 要求工具但没有结果，服务可能拒绝请求，或者模型会在逻辑上困惑。

Claude Code 在 `src/query.ts` 里有处理 missing tool result blocks 的逻辑。如果某些 assistant message 中有 tool_use 但缺少结果，它会生成中断消息补齐，避免下一轮上下文不合法。

教学版可以先写一个检查函数：

```python
from typing import Any

def iter_content_blocks(message: dict[str, Any]) -> list[dict[str, Any]]:
    content = message.get("content", [])
    if isinstance(content, str):
        return [{"type": "text", "text": content}]
    return [block for block in content if isinstance(block, dict)]


def validate_tool_pairs(messages: list[dict[str, Any]]) -> list[str]:
    pending: dict[str, str] = {}
    errors: list[str] = []

    for index, message in enumerate(messages):
        for block in iter_content_blocks(message):
            block_type = block.get("type")

            if block_type == "tool_use":
                tool_use_id = str(block.get("id", ""))
                if not tool_use_id:
                    errors.append(f"message[{index}] has tool_use without id")
                    continue
                pending[tool_use_id] = str(block.get("name", "unknown"))

            if block_type == "tool_result":
                tool_use_id = str(block.get("tool_use_id", ""))
                if tool_use_id not in pending:
                    errors.append(f"message[{index}] has orphan tool_result: {tool_use_id}")
                    continue
                del pending[tool_use_id]

    for tool_use_id, tool_name in pending.items():
        errors.append(f"missing tool_result for {tool_name}: {tool_use_id}")

    return errors
```

这个函数不完美，但可以帮你理解配对思想。

### 5.8 升级 mini-agent 的消息类型

现在我们把教学版 mini-agent 的 `agent/messages.py` 升级。

第一步，定义 block：

```python
from typing import Any, Literal, NotRequired, TypedDict


class TextBlock(TypedDict):
    type: Literal["text"]
    text: str


class ToolUseBlock(TypedDict):
    type: Literal["tool_use"]
    id: str
    name: str
    input: dict[str, Any]


class ToolResultBlock(TypedDict):
    type: Literal["tool_result"]
    tool_use_id: str
    content: str
    is_error: NotRequired[bool]


ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock
```

第二步，定义消息：

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
from uuid import uuid4

Role = str
Content = list[ContentBlock] | str

@dataclass
class Message:
    role: Role
    content: Content
    uuid: str = field(default_factory=lambda: str(uuid4()))

@dataclass
class TranscriptEntry:
    kind: str
    session_id: str
    message: Message
    uuid: str
    parent_uuid: str | None = None
    created_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())
```

第三步，写辅助函数：

```python
from typing import Any
from uuid import uuid4


def user_message(text: str) -> Message:
    return Message(role="user", content=[text_block(text)])


def assistant_text(text: str) -> Message:
    return Message(role="assistant", content=[text_block(text)])


def assistant_tool_use(name: str, tool_input: dict[str, Any]) -> Message:
    tool_use_id = f"toolu_{uuid4().hex[:12]}"
    return Message(
        role="assistant",
        content=[
            text_block(f"我需要调用工具 {name}。"),
            tool_use_block(tool_use_id, name, tool_input),
        ],
    )


def tool_result_message(tool_use_id: str, content: str, is_error: bool = False) -> Message:
    return Message(
        role="user",
        content=[tool_result_block(tool_use_id, content, is_error)],
    )
```

这样写的好处是，其他代码不需要手动拼对象，减少格式错误。

### 5.9 更新 FakeModelClient

上一章的 fake client 返回字符串，现在要返回 block 消息：

```python
from uuid import uuid4


class FakeModelClient:
    async def complete(self, messages: list[Message]) -> Message:
        last_user = next((m for m in reversed(messages) if m.role == "user"), None)
        user_text = render_message(last_user) if last_user else ""

        if "读文件" in user_text or "read" in user_text.lower():
            tool_use_id = f"toolu_{uuid4().hex[:12]}"
            return Message(
                role="assistant",
                content=[
                    text_block("我需要读取 README.md。"),
                    tool_use_block(tool_use_id, "Read", {"file_path": "README.md"}),
                ],
            )

        return Message(
            role="assistant",
            content=[text_block(f"收到：{user_text}")],
        )
```

这一步看似麻烦，但很必要。因为从现在开始，模型回复可能不只有文本。我们必须养成遍历 content blocks 的习惯。

### 5.10 更新 CLI 输出

CLI 打印 assistant 消息时，也不能直接 `message.content`。要提取文本块：

```python
def render_message(message: Message | None) -> str:
    if message is None:
        return ""
    if isinstance(message.content, str):
        return message.content

    lines: list[str] = []
    for block in message.content:
        block_type = block.get("type")
        if block_type == "text":
            lines.append(str(block.get("text", "")))
        elif block_type == "tool_use":
            lines.append(f"[tool_use: {block.get('name')}]")
        elif block_type == "tool_result":
            marker = "error" if block.get("is_error") else "result"
            lines.append(f"[tool_{marker}: {block.get('tool_use_id')}]")
            lines.append(str(block.get("content", "")))
    return "\n".join(line for line in lines if line)
```

然后：

```python
message = await engine.submit(text)
print(render_message(message))
```

真实产品会把不同 block 渲染成不同 UI。Claude Code 使用 React Ink 组件渲染工具调用、工具结果、进度、diff、权限弹窗等。我们先用纯文本显示。

### 5.11 从模型响应中提取工具调用

写一个工具函数：

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class ToolUse:
    id: str
    name: str
    input: dict[str, Any]


def extract_tool_uses(message: Message) -> list[ToolUse]:
    if isinstance(message.content, str):
        return []

    tool_uses: list[ToolUse] = []
    for block in message.content:
        if block.get("type") != "tool_use":
            continue
        tool_uses.append(
            ToolUse(
                id=str(block["id"]),
                name=str(block["name"]),
                input=dict(block.get("input", {})),
            )
        )
    return tool_uses
```

这个函数后面会进入 AgentEngine：

```python
async def run_agent_step(model: FakeModelClient, messages: list[Message]) -> list[ToolUse]:
    assistant_message = await model.complete(messages)
    messages.append(assistant_message)
    return extract_tool_uses(assistant_message)
```

现在我们还没有工具执行层，所以先只提取，不执行。

### 5.12 让 FakeModel 模拟工具调用

为了测试协议，可以让 fake model 在用户输入包含“读文件”时返回 tool_use：

```python
async def demo_fake_tool_call() -> None:
    model = FakeModelClient()
    messages = [user_message("请读文件 README.md")]

    tool_uses = await run_agent_step(model, messages)

    assert len(tool_uses) == 1
    assert tool_uses[0].name == "Read"
    assert tool_uses[0].input == {"file_path": "README.md"}
```

运行后，如果用户输入：

```text
请读文件
```

CLI 可以显示：

```text
我需要读取 README.md。
[tool_use: read_file]
```

下一章我们会真正执行这个工具。

### 5.13 消息协议和 Claude Code 的对应关系

Claude Code 中和消息协议相关的模块很多，主要包括：

```text
src/utils/messages.ts
src/utils/messages/mappers.ts
src/utils/messages/systemInit.ts
src/services/api/claude.ts
src/services/tools/toolExecution.ts
src/entrypoints/sdk/coreSchemas.ts
src/query.ts
```

`src/utils/messages.ts` 和 `src/utils/messages/mappers.ts` 负责创建、规范化、转换各种消息。

`src/utils/messages/systemInit.ts` 负责构造会话初始化类消息。

`src/services/api/claude.ts` 在发 API 前会把内部消息转换成 API 需要的格式，并确保工具结果配对。

`src/query.ts` 在 Agent 循环中收集 assistant message、tool_use block 和工具结果。

`src/services/tools/toolExecution.ts` 会把工具输出映射成 tool_result block。

这说明消息协议不是一个边缘细节，而是贯穿整个 Agent 系统的主干。

### 5.14 常见坑

坑一：把 `tool_result` 放进 assistant 消息。

工具结果应该作为 user role 内容返回。assistant 是请求工具的一方，不是报告工具执行结果的一方。

坑二：丢失 `tool_use.id`。

如果你执行工具时没有保存 ID，就无法生成正确 `tool_result.tool_use_id`。

坑三：工具失败时 throw。

任务级失败应该变成 `is_error: true` 的 tool_result。

坑四：把所有 content 当字符串。

一旦引入工具调用，content 就是 block 数组。渲染、日志、模型适配都要处理 block。

坑五：多个工具调用并发后结果顺序混乱。

即便并发执行，也要能正确按 ID 配对。Claude Code 的并发工具执行层会特别处理结果顺序和上下文修改。

### 5.15 本章练习

练习一：升级 `Message` 类型。

把上一章的字符串 content 改成 content blocks，并修复所有类型错误。

练习二：实现 `render_message()`。

要求：

- text block 打印文本。
- tool_use block 打印 `[tool_use: 工具名]`。
- tool_result block 打印 `[tool_result: ID]`。

练习三：实现 `extract_tool_uses()`。

要求只从 assistant 消息中提取工具调用。

练习四：让 FakeModelClient 返回工具调用。

当用户输入包含“读文件”时，返回 `read_file` 工具调用。

练习五：实现配对检查。

写一个函数找出没有结果的 tool_use。暂时只打印 warning，不中断程序。

### 5.16 本章小结

本章我们完成了从普通聊天消息到 Agent 消息协议的升级。你现在应该理解：

1. Agent 消息需要 content blocks。
2. `tool_use` 是 assistant 发出的动作请求。
3. `tool_result` 是外部环境返回给 assistant 的观察结果。
4. 两者必须通过 ID 配对。
5. 工具失败也应该作为 tool_result 回传。
6. mini-agent 已经准备好进入真正的工具执行阶段。

下一章我们会定义统一 Tool 接口，并实现第一个真正可执行的工具：读取文件。

---

# 第 2 卷：最小 Agent 实现
