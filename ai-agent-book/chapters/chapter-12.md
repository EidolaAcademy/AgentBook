# 第 2 卷：最小 Agent 实现

## 第 12 章：接入真实模型 API

### 12.1 本章目标

到目前为止，我们一直使用 FakeModel。它帮助我们搭建了 Agent 骨架，但它不具备真实语言理解能力。真正的 Agent 需要接入模型 API，让模型自己根据用户目标和工具结果决定下一步。

本章目标：

1. 理解 ModelClient 抽象的价值。
2. 实现一个真实模型客户端。
3. 把内部 Message 转成 API messages。
4. 把 Tool 转成 API tools。
5. 处理模型返回的 text 和 tool_use。
6. 理解非流式和流式的区别。
7. 对照 Claude Code 的 `services/api/claude.py`。

### 12.2 为什么先有 ModelClient 抽象

我们很早就定义了：

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

当时看起来有点多余。现在它的价值出现了。

FakeModelClient 和真实模型客户端都实现同一个接口：

```text
AgentEngine -> ModelClient
```

AgentEngine 不需要知道背后是真模型还是假模型。

这带来几个好处：

1. 测试时用 FakeModel。
2. 开发时可以切换模型供应商。
3. 失败时可以记录请求和响应。
4. 未来可以加入重试、fallback、成本统计，而不改 AgentEngine。

Claude Code 的 `src/query/deps.ts` 就是同样思想。`query()` 不直接写死所有依赖，而是可以注入 `callModel`、`microcompact` 等。

### 12.3 选择非流式先实现

真实产品通常使用 streaming，因为用户体验更好，工具调用也能更早开始。

但教学项目先用非流式。

原因：

1. 代码更短。
2. 更容易理解。
3. 工具调用块一次性返回，不需要拼接 partial input。
4. 方便调试。

等非流式跑通后，再学习流式。Claude Code 使用复杂的 streaming 逻辑，是为了性能和体验；新手不应该第一步就跳进去。

### 12.4 API 消息格式和内部消息格式

我们的内部 Message：

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

API 也有 messages，但具体字段可能略有不同。我们需要写 mapper。

原则：

```text
内部结构不要直接等于外部 API 结构
```

为什么？

第一，外部 API 可能变化。

第二，你可能支持多个模型供应商。

第三，内部消息还有 UI、进度、系统事件等，不一定都发给模型。

Claude Code 有大量 mapper，例如 `src/mini_agent/utils/messages/mappers.js`、`normalizeMessagesForAPI` 等，就是为了处理内部消息和 API 消息之间的差异。

### 12.5 转换 text block

先写简单转换：

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

再转换 message：

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

system message 可以单独放到 API 的 `system` 参数里。

### 12.6 Tool 转 API Schema

我们的 Tool 目前只有 Zod schema，没有 JSON Schema。教学项目可以先给 Tool 增加一个字段：

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

更新 Tool：

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

然后每个工具手写。

例如 read_file：

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

转换：

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

手写 JSON Schema 有重复，但对新手清晰。后面可以用自动转换。

### 12.7 AnthropicModelClient 示例

下面是概念代码。实际 SDK 版本可能略有差异，按你安装的 SDK 文档调整。

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

这里用了 `as any`，是为了避免教学代码被 SDK 具体类型淹没。真实项目应该认真对齐 SDK 类型。

### 12.8 fromApiMessage

把 API 响应转回内部 Message：

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

assistant 响应通常不会包含 `tool_result`，因为 tool_result 是客户端执行工具后再发回模型的。

### 12.9 system prompt 第一版

先写一个简短 system prompt：

```python
import asyncio
from dataclasses import dataclass
from pathlib import Path

@dataclass
class CommandResult:
    command: str
    exit_code: int | None
    stdout: str
    stderr: str
    timed_out: bool = False

async def run_command(command: str, cwd: Path, timeout: float = 30.0) -> CommandResult:
    process = await asyncio.create_subprocess_shell(
        command,
        cwd=str(cwd),
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    try:
        stdout, stderr = await asyncio.wait_for(process.communicate(), timeout=timeout)
    except asyncio.TimeoutError:
        process.kill()
        return CommandResult(command, None, "", "命令超时", True)
    return CommandResult(command, process.returncode, stdout.decode(), stderr.decode())
```

这个 prompt 传达了几个重要规则：

1. 不要猜文件内容。
2. 搜索、读取、shell、写入分别何时用。
3. 截断时缩小范围。

Claude Code 的 system prompt 远比这复杂，会动态加入上下文、工具说明、模式、环境信息、项目记忆等。但核心目标一样：指导模型如何安全有效地使用工具。

### 12.10 环境变量

CLI 中读取 API key：

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

然后：

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

模型名请根据当前可用模型调整。真实项目里应该把模型放进配置，而不是写死。

Claude Code 有复杂模型选择逻辑，包括默认模型、用户指定模型、fallback model、fast mode、thinking config 等。mini-agent 先写死一个模型即可。

### 12.11 第一次真实运行

你可以试：

```text
总结 README
```

理想流程：

```text
assistant -> glob_files 或 read_file
tool -> 返回内容
assistant -> 总结
```

再试：

```text
搜索 login
```

理想流程：

```text
assistant -> grep_files
tool -> 返回命中
assistant -> 解释结果或继续 read_file
```

再试：

```text
创建 hello.txt
```

理想流程：

```text
assistant -> write_file
permission prompt -> y/N
tool -> 写入
assistant -> 报告完成
```

如果模型不调用工具，而是直接编造文件内容，说明 system prompt 或工具描述不够明确。

### 12.12 常见接入问题

问题一：模型不调用工具。

可能原因：

- tools 没传给 API。
- tool schema 格式不对。
- system prompt 没强调使用工具。
- 用户请求不需要工具。

问题二：模型调用工具但参数错误。

这是正常现象。schema 校验会返回错误 tool_result，模型下一轮可能修正。

问题三：tool_use 和 tool_result 配对错误。

检查 tool_use id 是否保存，tool_result 是否使用同一个 id。

问题四：API 报 messages 格式错误。

通常是 content block 格式不符合 SDK 类型，或 tool_result 放错 role。

问题五：上下文太长。

现在 mini-agent 还没有压缩。长任务可能很快遇到限制。后面会讲上下文工程。

### 12.13 和 Claude Code API 层对照

Claude Code 的 `src/services/api/claude.ts` 做的事情包括：

1. 内部消息转 API 消息。
2. 工具转 API schema。
3. 处理 prompt caching。
4. 处理 thinking。
5. 处理 beta headers。
6. 处理 streaming。
7. 处理 retry 和 fallback。
8. 处理 usage、cost、quota。
9. 处理 max output tokens。
10. 处理工具搜索和 deferred tools。

我们的 mini-agent 只做了最小 API 调用。但主线一样：

```text
internal messages + tools -> API request -> assistant content blocks -> internal message
```

理解这个主线后，再看 Claude Code 的 API 层，会发现复杂度来自产品需求，而不是核心概念不同。

### 12.14 本章练习

练习一：给 Tool 增加 `inputJsonSchema`。

为所有工具手写 JSON Schema。

练习二：实现 `toApiMessages()`。

支持：

- text
- tool_use
- tool_result

练习三：实现 `toApiTools()`。

把 ToolRegistry 转成 API tools。

练习四：实现真实 ModelClient。

用你选择的模型 SDK 完成一次非流式调用。

练习五：测试真实工具调用。

让模型完成：

```text
总结 README
```

确认它会调用 `read_file` 或 `glob_files`。

### 12.15 本章小结

本章我们把 mini-agent 从 FakeModel 推向真实模型。

你应该理解：

1. ModelClient 抽象让假模型和真模型可替换。
2. 内部消息需要映射到 API 消息。
3. Tool 需要映射到 API schema。
4. system prompt 会强烈影响模型是否正确使用工具。
5. 非流式先跑通，流式后优化。

下一章我们会加入流式响应，让用户不用等完整模型回复结束才看到输出。

---
