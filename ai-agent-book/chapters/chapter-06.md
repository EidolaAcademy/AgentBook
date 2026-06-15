# 第 2 卷：最小 Agent 实现

## 第 6 章：Tool 接口与第一个 Read 工具

### 6.1 本章目标

从这一章开始，我们的小程序会真正“动手”。第 5 章里，FakeModelClient 已经能返回一个 `tool_use` block，但 AgentEngine 还不会执行它。现在我们要补上工具系统的第一块拼图：统一 Tool 接口。

读完本章，你会完成：

1. 理解为什么所有工具都应该遵守同一个接口。
2. 学会用 Zod 定义工具参数 schema。
3. 实现 `Tool` 类型。
4. 实现工具注册表。
5. 实现 `runToolUse()`。
6. 实现第一个真实工具：`read_file`。
7. 把工具结果作为 `tool_result` 回到消息历史。
8. 对照 Claude Code 的 `src/tool.py` 和 `src/mini_agent/tools/FileReadTool/FileReadtool.py` 理解工程级设计。

这一章是 Agent 从“会说话”到“会观察环境”的第一步。只要 `read_file` 跑通，后面的 `bash`、`write_file`、`grep`、`web_fetch` 本质上都是同一模式的扩展。

### 6.2 为什么需要统一 Tool 接口

先想一个最朴素的实现。模型返回：

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

你当然可以直接写：

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

这在只有一个工具时能工作。但第二个工具来了怎么办？

```python
if toolUse.name === "read_file":
    // ...
    } else if (toolUse.name == "bash"):
        // ...
        } else if (toolUse.name == "write_file"):
            // ...
            } else if (toolUse.name == "grep"):
                // ...
```

很快你会遇到一堆重复问题：

1. 每个工具都要校验参数。
2. 每个工具都要处理错误。
3. 每个工具都要决定是否只读。
4. 每个工具都要决定是否能并发。
5. 每个工具都要生成给模型看的描述。
6. 每个工具都要把输出转换成字符串。
7. 每个工具都要被注册进工具列表。

如果没有统一接口，工具越多，代码越乱。最终你会得到一个巨大的 `switch`，里面混合了参数校验、权限、安全、执行、格式化和日志。

统一 Tool 接口的目标是：让 AgentEngine 不关心工具内部怎么实现。它只需要知道：

```text
根据名字找到工具 -> 校验 input -> 调用 tool.call -> 得到结果 -> 生成 tool_result
```

这和 Claude Code 的设计一致。源码里的 `src/tool.py` 定义了非常完整的 Tool 类型，而 `src/services/tools/toolExecution.py` 只依赖这个统一接口执行工具。

### 6.3 一个工具应该回答哪些问题

新手版工具先回答六个问题：

1. 我叫什么？
2. 我给模型看的描述是什么？
3. 我的参数 schema 是什么？
4. 我是不是只读？
5. 我能不能和其他工具并发？
6. 调用我时怎么执行？

对应 Python 类型：

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

这里 `Input` 是工具参数类型，`Output` 是工具内部返回类型。

为什么还需要 `formatResult`？因为工具内部返回的数据结构不一定适合直接给模型。例如读取文件时，我们可能希望内部返回：

```python

    "path": "/abs/path/README.md"
    "content": "..."
    "lineCount": 120
```

但给模型的内容可以格式化成：

```text
File: /abs/path/README.md
Lines: 120

...
```

把执行和格式化拆开，会让后面处理大输出、截断、结构化结果更容易。

### 6.4 Claude Code 的 Tool 接口更复杂在哪里

Claude Code 的 `src/tool.py` 里，Tool 接口远比我们的教学版复杂。它除了 `call`、`inputSchema`、`isReadOnly` 这些核心字段，还包含：

- `prompt()`：给模型看的工具说明。
- `description()`：给用户或权限弹窗看的描述。
- `checkPermissions()`：工具自己的权限逻辑。
- `validateInput()`：schema 之外的业务校验。
- `mapToolResultToToolResultBlockParam()`：把工具输出转为 API tool_result。
- `renderToolUseMessage()`：终端 UI 渲染工具调用。
- `renderToolResultMessage()`：终端 UI 渲染工具结果。
- `getToolUseSummary()`：紧凑视图里的摘要。
- `isDestructive()`：是否破坏性操作。
- `isOpenWorld()`：是否访问开放外部世界。
- `preparePermissionMatcher()`：为 hook/权限规则准备匹配器。
- `toAutoClassifierInput()`：给安全分类器看的紧凑输入。

初学者不需要一开始实现这些。你只要明白一件事：复杂接口不是为了炫技，而是因为真实产品里工具同时服务于模型、用户、权限系统、UI、日志、测试、MCP 和子 Agent。

我们先实现最小接口，后面每章逐步加能力。

### 6.5 为什么需要 Zod

模型生成的工具参数不可信。这里的“不可信”不是说模型恶意，而是说它可能犯错。

例如工具 schema 要求：

```json
{
  "path": "README.md",
  "offset": 10,
  "limit": 20
}
```

模型可能生成：

```json
{
  "file": "README.md",
  "offset": "10",
  "limit": "twenty"
}
```

也可能生成：

```json
{
  "path": ["README.md"]
}
```

如果你直接把这些参数传给工具，工具内部会出现奇怪错误。更糟的是，对 Bash 或写文件工具来说，错误参数可能带来安全风险。

Zod 的作用是把“运行时输入”变成“经过验证的类型”。

比如：

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

校验：

```python
parsed = ReadFileInputSchema.safeParse(toolUse.input)

def if(self, !parsed.success):
    return toolError(toolUse.id, parsed.error.message)

input = parsed.data
```

从 `parsed.data` 开始，Python 知道 `input.path` 一定是 string，`offset` 如果存在就一定是非负整数。

Claude Code 中也使用 Zod。`src/tool.py` 的 Tool 类型中有 `inputSchema`，具体工具例如 `FileReadTool`、`BashTool` 都会定义自己的 schema。`src/services/tools/toolExecution.py` 在执行工具前会先 `safeParse`，失败后返回 `InputValidationError` 给模型。

这就是工程级 Agent 的基本纪律：模型参数先校验，再执行。

### 6.6 创建 tool.py

在我们的 mini-agent 中，创建 `src/mini_agent/tools/tool.py`：

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

这里的 `AnyTool` 是为了工具注册表方便。严格来说，`any` 会损失类型精度，但工具注册表要存放不同 Input 类型的工具，很难用一个简单数组保持所有泛型。真实大型项目会用更复杂的类型技巧，Claude Code 的 `buildTool` 就做了很多类型层面的处理。

新手项目里，`AnyTool` 可以接受。关键是单个工具内部仍然有清晰 schema。

### 6.7 buildTool：给工具默认值

Claude Code 的 `buildTool` 很值得学习。它会给工具补上默认方法，例如默认 `isEnabled` 为 true，默认 `isConcurrencySafe` 为 false，默认 `isReadOnly` 为 false。

我们也可以做一个简化版：

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

为什么默认 `isReadOnly` 和 `isConcurrencySafe` 都是 false？

因为安全默认值要保守。如果一个工具没有明确声明自己只读，就把它当成可能写入。如果一个工具没有明确声明自己可并发，就串行执行。这样可能慢一点，但不容易出事故。

这和 Claude Code 的原则一致：不确定时 fail closed。

### 6.8 工具注册表

创建 `src/mini_agent/tools/registry.py`：

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

这个注册表做三件事：

1. 注册工具。
2. 根据名字查工具。
3. 列出所有工具给模型。

Claude Code 的 `src/tools.py` 是更复杂的注册表。它不是简单 map，而是根据 feature flag、权限模式、MCP、环境变量、agent 类型动态组装工具池。但本质还是“当前会话有哪些工具可用”。

### 6.9 read_file 工具的需求

现在实现第一个工具。最小需求：

1. 输入 path。
2. 可选 offset 和 limit。
3. 只能读取当前工作目录下的文件。
4. 如果文件不存在，返回错误。
5. 如果是目录，返回错误。
6. 支持按行截取。
7. 返回文件路径、行数和内容。

为什么要限制当前工作目录？因为 Agent 不应该默认读取整个系统。即便是读操作，也可能读到密钥、浏览器数据、SSH key、系统配置。

Claude Code 的 FileReadTool 更复杂，它支持：

- 图片读取和压缩。
- PDF 页码范围。
- Notebook。
- 二进制检测。
- 大文件 token 限制。
- 设备文件阻止。
- 相似路径建议。
- 文件读取监听器。
- 权限规则。
- 技能触发。

我们先实现文本文件读取。

### 6.10 路径安全：不要相信模型给的 path

假设当前项目目录是：

```text
/Users/me/project
```

模型请求：

```json
{ "path": "../../.ssh/id_rsa" }
```

如果你直接 `readFile(path)`，就可能读取项目外的敏感文件。

因此要把路径解析为绝对路径，并检查它是否仍在 cwd 下：

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

这个函数的意思是：

1. 用 `path.resolve` 得到绝对路径。
2. 计算它相对 cwd 的路径。
3. 如果相对路径以 `..` 开头，说明跑到 cwd 外面了。
4. 如果相对路径本身是绝对路径，也拒绝。

真实产品还要考虑 symlink、大小写文件系统、Windows 路径、额外授权目录等。Claude Code 的文件系统权限系统比这复杂得多。本章先掌握基本边界。

### 6.11 实现 read_file

创建 `src/mini_agent/tools/readFiletool.py`：

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

这个工具已经有了几个好习惯：

1. 用 schema 校验输入。
2. 限制路径在 cwd 内。
3. 区分目录和文件。
4. 支持 offset/limit。
5. 声明自己只读。
6. 声明自己可并发。
7. 内部输出和模型输出分离。

### 6.12 实现 runToolUse

现在我们需要把 `tool_use` block 变成 `tool_result` message。

创建 `src/mini_agent/tools/runToolUse.py`：

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

注意这段代码的错误处理策略：

1. 未知工具 -> 错误 tool_result。
2. 参数校验失败 -> 错误 tool_result。
3. 工具执行抛错 -> 错误 tool_result。

这些都是任务级错误，返回给模型，而不是让整个程序崩溃。

这正是 Claude Code 的 `toolExecution.py` 思路。它会在找不到工具、Zod 校验失败、validateInput 失败、权限拒绝、call 抛错时生成模型可读的错误结果。

### 6.13 把工具注册进 CLI

在 `cli.py` 或一个单独的 `createRegistry.py` 中注册：

```python

def createToolRegistry():
    registry = ToolRegistry()
    registry.register(readFileTool)
    return registry
```

然后启动 Engine 时传进去：

```python
registry = createToolRegistry()
engine = AgentEngine(:
    "model": FakeModelClient()
    "tools": registry
    "cwd": process.cwd()
```

这意味着 `AgentEngine` 构造函数需要升级。

### 6.14 升级 AgentEngine

上一章的 Engine 只做：

```text
用户消息 -> 模型回复 -> 返回
```

现在要做：

```text
用户消息 -> 模型回复 -> 提取工具调用 -> 执行工具 -> 追加结果 -> 再次调用模型
```

先实现最简单版本：

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

这个循环已经是真正的 Agent 循环了。它会一直运行，直到模型返回不含 tool_use 的 assistant 消息。

### 6.15 防止无限循环

上面的循环有一个严重问题：如果模型一直调用工具，程序会永远跑下去。

比如 FakeModelClient 每次看到 tool_result 后又返回同一个 tool_use，就会无限循环。

所以必须加 `maxTurns`：

```python
from typing import Any

def example(context: dict[str, Any]) -> dict[str, Any]:
    return {"ok": True, "context": context}
```

但注意，这里 throw 是系统级保护。它不是某个工具失败，而是 Agent 循环失控。真实产品会更温和地返回一条 assistant error message 或中断状态。

Claude Code 的 `query.py` 也支持 `maxTurns`。工程级 Agent 一定要有上限，包括：

- 最大轮数。
- 最大 token。
- 最大费用。
- 最大工具并发。
- 最大工具输出。
- 最大命令运行时间。

没有预算的 Agent 不是自动化助手，而是一台可能无限花钱和无限运行的机器。

### 6.16 让 FakeModel 完成一次读取流程

现在 FakeModel 要更聪明一点：

1. 用户说“读文件”时，返回 tool_use。
2. 收到 tool_result 后，返回最终文本。

示例：

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

这不是完美模型，但它能模拟真实 Agent 的关键流程：

```text
用户请求 -> 模型发工具调用 -> 程序执行工具 -> 模型读结果 -> 模型回复
```

### 6.17 工具描述如何给模型

我们目前只是本地注册了工具，但真实模型需要知道工具有哪些、参数是什么、描述是什么。

在真实 API 请求中，你会把工具 schema 发给模型：

```json
{
  "name": "read_file",
  "description": "Read a text file from the current workspace",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string" },
      "offset": { "type": "number" },
      "limit": { "type": "number" }
    },
    "required": ["path"]
  }
}
```

我们的 `Tool` 里已经有 `name`、`description` 和 `inputSchema`，但 Zod schema 不能直接发给模型。后面接真实 API 时，我们需要把 Zod 转成 JSON Schema，或者手写 JSON schema。

Claude Code 的 `src/mini_agent/utils/api.js` 中有 `toolToAPISchema` 一类逻辑，负责把内部 Tool 转成 API tool schema。MCP 工具则可能本身就提供 JSON Schema。

教学项目可以先手写：

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

后面再逐步自动化。

### 6.18 与 Claude Code FileReadTool 对照

我们的 `read_file` 做了文本读取、路径限制、offset/limit。Claude Code 的 `FileReadTool` 做得更多。

它的输入 schema 包括：

- `file_path`：绝对路径。
- `offset`：起始行。
- `limit`：读取行数。
- `pages`：PDF 页码范围。

它还处理：

1. 图片文件：读取后转成模型可接收的 image block。
2. PDF：抽取指定页。
3. Notebook：按 cell 映射。
4. 二进制文件：避免按文本读取乱码。
5. 设备文件：阻止 `/dev/random` 等会卡住的路径。
6. token 限制：文件太大时要求 offset/limit。
7. 文件修改时间：辅助判断缓存是否过期。
8. 相似路径：文件不存在时给建议。
9. 权限检查：结合 filesystem permission rules。
10. 技能触发：读取特定文件后激活相关 skill。

为什么这么复杂？因为“读文件”在演示项目里是一行代码，在真实 coding Agent 里是高风险高频操作。

读文件可能带来：

- 隐私风险。
- 上下文爆炸。
- 程序卡死。
- 编码错误。
- 图片/PDF/Notebook 特殊格式。
- 路径误判。
- 权限越界。

这就是本书反复强调的：Agent 工程的复杂度来自边界情况。

### 6.19 常见坑

坑一：不校验路径范围。

模型可能生成 `../../secret`。即便模型不是恶意，用户也可能误导它。必须限制工作区。

坑二：直接读取超大文件。

一个 50MB 日志文件足以拖慢程序、撑爆上下文、浪费费用。后面我们会加入文件大小和 token 限制。

坑三：把工具内部错误当系统崩溃。

文件不存在应该作为 tool_result 返回给模型，而不是让 CLI 崩。

坑四：没有 maxTurns。

Agent 循环必须有上限。

坑五：把所有工具都当可并发。

Read 可以并发，但写文件、改目录、运行某些 shell 命令不一定可以。默认不要并发。

### 6.20 本章练习

练习一：实现 `Tool` 类型。

要求包含：

- `name`
- `description`
- `inputSchema`
- `call`
- `formatResult`
- `isReadOnly`
- `isConcurrencySafe`

练习二：实现 `buildTool()`。

要求：

- `formatResult` 默认 `String(output)`。
- `isReadOnly` 默认 false。
- `isConcurrencySafe` 默认 false。

练习三：实现 `ToolRegistry`。

要求：

- 支持注册工具。
- 支持按名字查找。
- 重复注册时报错。

练习四：实现 `read_file`。

要求：

- 只能读取 cwd 内的文本文件。
- 支持 offset/limit。
- 目录时报错。
- 文件不存在时返回错误 tool_result。

练习五：升级 `AgentEngine`。

要求：

- 提取 tool_use。
- 调用 `runToolUse()`。
- 把 tool_result 加入消息历史。
- 循环直到模型不再调用工具。
- 加 `maxTurns`。

### 6.21 本章小结

本章我们完成了 mini-agent 的第一个真实工具系统。现在它已经不只是聊天机器人，而是能通过工具读取工作区文件的 Agent。

你应该掌握：

1. 统一 Tool 接口能避免工具逻辑散落。
2. Zod 参数校验是工具执行前的必要步骤。
3. 工具失败应该以错误 tool_result 返回模型。
4. `read_file` 虽然简单，但已经涉及路径安全、文件类型、输出格式和上下文预算。
5. Claude Code 的 FileReadTool 是同一思想的产品级版本。

下一章我们会实现搜索工具。读取文件让 Agent 能看一个点，搜索工具让 Agent 能探索整个项目。

---
